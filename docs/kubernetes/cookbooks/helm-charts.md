# Cookbook: Helm Charts - Packaging & Deployment

> [!NOTE]
> Practical guide to creating and deploying Helm charts for Kubernetes applications.
> Covers: Chart structure, templating, values, and production patterns.

## 1. Helm Chart Structure

```
my-app-chart/
├── Chart.yaml                 # Chart metadata
├── values.yaml               # Default values
├── values-prod.yaml          # Production overrides
├── values-staging.yaml       # Staging overrides
├── templates/
│   ├── deployment.yaml       # Deployment resource
│   ├── service.yaml          # Service resource
│   ├── ingress.yaml          # Ingress resource
│   ├── configmap.yaml        # Config
│   ├── secret.yaml           # Secrets
│   ├── _helpers.tpl          # Template helpers
│   ├── NOTES.txt             # Post-install notes
│   └── tests/
│       └── test-connection.yaml  # Helm tests
├── .helmignore               # Files to exclude
└── README.md                 # Chart documentation
```

## 2. Chart.yaml (Metadata)

```yaml
apiVersion: v2
name: my-app
description: My application Helm chart
type: application
version: 1.0.0               # Chart version
appVersion: "1.2.3"          # App version
keywords:
  - web
  - api
home: https://github.com/example/my-app
sources:
  - https://github.com/example/my-app
maintainers:
  - name: Your Name
    email: your-email@example.com
```

## 3. values.yaml (Configuration)

```yaml
# Default values for my-app

# Global config
global:
  environment: production
  domain: example.com

# Replica count
replicaCount: 3

# Image configuration
image:
  repository: my-registry.azurecr.io/my-app
  pullPolicy: IfNotPresent
  tag: "1.2.3"                    # Overrides appVersion

imagePullSecrets:
  - name: acr-credentials

# Annotations and labels
nameOverride: ""
fullnameOverride: ""

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "3000"

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL

# Service configuration
service:
  type: ClusterIP
  port: 80
  targetPort: 3000
  annotations: {}

# Ingress configuration
ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: app-tls
      hosts:
        - app.example.com

# Resource limits
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

# Autoscaling
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80

# Pod Disruption Budget
podDisruptionBudget:
  enabled: true
  minAvailable: 1

# Health checks
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5

# Environment variables
env:
  LOG_LEVEL: info
  NODE_ENV: production

# Secrets as environment variables
envFrom:
  - secretRef:
      name: app-secrets
  - configMapRef:
      name: app-config

# Volume mounts
volumeMounts:
  - name: tmp
    mountPath: /tmp

volumes:
  - name: tmp
    emptyDir: {}

# Affinity rules (pod scheduling)
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - my-app
          topologyKey: kubernetes.io/hostname

# Node selector
nodeSelector: {}

# Tolerations
tolerations: []

# Service Account
serviceAccount:
  create: true
  annotations: {}
  name: ""
```

## 4. Template Files

### deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "my-app.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
      - name: {{ .Chart.Name }}
        securityContext:
          {{- toYaml .Values.securityContext | nindent 12 }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: 3000
          protocol: TCP
        livenessProbe:
          {{- toYaml .Values.livenessProbe | nindent 12 }}
        readinessProbe:
          {{- toYaml .Values.readinessProbe | nindent 12 }}
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
        {{- with .Values.env }}
        env:
          {{- range $key, $value := . }}
          - name: {{ $key }}
            value: {{ $value | quote }}
          {{- end }}
        {{- end }}
        {{- with .Values.envFrom }}
        envFrom:
          {{- toYaml . | nindent 12 }}
        {{- end }}
        {{- with .Values.volumeMounts }}
        volumeMounts:
          {{- toYaml . | nindent 12 }}
        {{- end }}
      {{- with .Values.volumes }}
      volumes:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "my-app.selectorLabels" . | nindent 4 }}
```

### ingress.yaml
```yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "my-app.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```

### _helpers.tpl
```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "my-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "my-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "my-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-app.labels" -}}
helm.sh/chart: {{ include "my-app.chart" . }}
{{ include "my-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "my-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "my-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

## 5. Deployment Commands

### Test Chart
```bash
# Validate syntax
helm lint ./my-app-chart

# Dry-run (see what gets created)
helm install my-release ./my-app-chart --dry-run --debug

# Render templates locally
helm template my-release ./my-app-chart
```

### Install Chart
```bash
# Install from local directory
helm install my-release ./my-app-chart -n my-namespace --create-namespace

# Install from repository
helm repo add myrepo https://charts.example.com
helm install my-release myrepo/my-app -n my-namespace

# Install with custom values
helm install my-release ./my-app-chart \
  -n my-namespace \
  -f values-prod.yaml

# Override specific values
helm install my-release ./my-app-chart \
  -n my-namespace \
  --set image.tag=1.2.3 \
  --set replicaCount=5 \
  --set domain=prod.example.com
```

### Upgrade Chart
```bash
# Simple upgrade
helm upgrade my-release ./my-app-chart -n my-namespace

# Upgrade with values
helm upgrade my-release ./my-app-chart \
  -n my-namespace \
  -f values-prod.yaml \
  --set image.tag=1.3.0

# Upgrade and rollback if fails
helm upgrade my-release ./my-app-chart \
  -n my-namespace \
  --wait  # Wait for deployment to be ready
```

### Rollback
```bash
# View history
helm history my-release -n my-namespace

# Rollback to previous version
helm rollback my-release -n my-namespace

# Rollback to specific revision
helm rollback my-release 3 -n my-namespace
```

### Uninstall
```bash
helm uninstall my-release -n my-namespace
```

## 6. Environment-Specific Values

### values-prod.yaml
```yaml
replicaCount: 5

image:
  tag: "1.2.3"

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

autoscaling:
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
```

### Deploy to Prod
```bash
helm upgrade --install my-app ./my-app-chart \
  -n production \
  -f values.yaml \
  -f values-prod.yaml \
  --wait
```

## 7. Helm Tests

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "my-app.fullname" . }}-test-connection"
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['{{ include "my-app.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

Run tests:
```bash
helm test my-release -n my-namespace
```

## 8. Chart Packaging & Distribution

### Package Chart
```bash
# Create tarball
helm package ./my-app-chart

# Result: my-app-1.0.0.tgz
```

### Push to Repository
```bash
# Using Helm Repo (requires setup)
# For GitHub Pages:
# 1. Generate index.yaml
helm repo index ./charts/

# 2. Commit and push
git add . && git commit -m "Add chart" && git push

# 3. Add to local repo
helm repo add myrepo https://example.github.io/charts
helm repo update
helm search repo myapp
```

## 9. Best Practices

| Practice | Why | How |
|----------|-----|-----|
| Use environment-specific values files | Different configs for dev/staging/prod | Create `values-{env}.yaml` files |
| Set resource requests/limits | Prevents resource starvation | Define in deployment template |
| Use health checks | Kubernetes restarts failed pods | Set liveness and readiness probes |
| Run as non-root | Security hardening | Set `securityContext.runAsNonRoot: true` |
| Use autoscaling | Handle traffic spikes | Enable HPA in values |
| Test charts | Catch errors before deploy | Run `helm lint` and `helm test` |
| Version charts | Track changes | Bump `version` in Chart.yaml |
| Document values | Others can understand config | Add comments to values.yaml |

## 10. Common Commands Reference

```bash
# Search for charts
helm search repo <keyword>

# Show chart values
helm show values <chart>

# Show all chart info
helm show all <chart>

# Verify chart structure
helm lint <chart-directory>

# Get deployed release status
helm status <release> -n <namespace>

# Get release values
helm get values <release> -n <namespace>

# Compare versions
helm diff upgrade <release> <chart> -n <namespace>

# Clean up
helm uninstall <release> -n <namespace>
```

---

**Helm Lifecycle**:
1. Create Chart.yaml and values.yaml
2. Create templates/ directory with K8s manifests
3. Lint and test locally
4. Install to development cluster
5. Package for distribution
6. Store in Helm repository
7. Deploy to production via GitOps pipeline
