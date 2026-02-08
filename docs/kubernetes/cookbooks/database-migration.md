# Cookbook: Database Migration on Kubernetes

> [!NOTE]
> How to safely run database migrations when deploying on EKS.
> Covers: pre-deployment checks, transaction handling, rollback strategies, and monitoring.

## 1. Pre-Migration Checklist

```bash
# Backup the database FIRST
kubectl exec -it <rds-pod> -n <namespace> -- pg_dump -U <user> <db> > backup.sql

# Check current app version
kubectl get deployment <app> -n <namespace> -o jsonpath='{.spec.template.spec.containers[0].image}'

# Verify migration scripts exist in image
kubectl run debug --image=<image>:new-tag --rm -it -- /bin/sh
  # Inside pod: ls -la /app/migrations/
```

## 2. Strategy 1: Blue-Green (Safest)

Run two versions of the app simultaneously, then switch traffic.

### Step 1: Deploy new version (without traffic)
```bash
# Create a new deployment (green)
kubectl create deployment <app>-v2 -n <namespace> --image=<image>:new-tag --replicas=0

# Or manually edit and set replicas: 0
kubectl scale deployment <app>-v2 -n <namespace> --replicas=0
```

### Step 2: Run migration as a Job
```bash
# Create a one-time job to run migrations
kubectl create job migrate-v2 --image=<image>:new-tag -n <namespace> -- migrate up
```

### Step 3: Wait for migration to complete
```bash
# Watch the job
kubectl get jobs -n <namespace> -w
kubectl logs job/migrate-v2 -n <namespace> -f

# Check if successful
kubectl get job migrate-v2 -n <namespace> -o jsonpath='{.status.succeeded}'
```

### Step 4: Scale up green, then flip traffic
```bash
# Scale green deployment
kubectl scale deployment <app>-v2 -n <namespace> --replicas=3

# Verify readiness
kubectl get pods -l app=<app>-v2 -n <namespace>

# Update service selector (or ingress) to point to v2
kubectl patch service <app> -n <namespace> -p '{"spec":{"selector":{"version":"v2"}}}'

# Or with Ingress:
kubectl patch ingress <app> -n <namespace> --type='json' -p='[{"op":"replace","path":"/spec/rules/0/http/paths/0/backend/service/name","value":"<app>-v2"}]'
```

### Step 5: Monitor, then clean up old
```bash
# Keep old deployment running for 30 minutes (rollback window)
# Monitor error rates
kubectl top nodes
kubectl logs -l app=<app>-v2 -n <namespace> --tail=50 -f

# After 30 min, delete old deployment
kubectl delete deployment <app> -n <namespace>
```

## 3. Strategy 2: Rolling Update (Backward-Compatible Migrations)

Use if all migrations are backward compatible with old code.

```bash
# Just deploy normally
kubectl set image deployment/<app> <container>=<image>:new-tag -n <namespace>

# Monitor rollout
kubectl rollout status deployment/<app> -n <namespace>

# Or slow it down
kubectl patch deployment <app> -n <namespace> -p '{"spec":{"strategy":{"rollingUpdate":{"maxUnavailable":1,"maxSurge":1}}}}'
```

**Critical**: The OLD version must work with NEW database schema during transition.

## 4. Strategy 3: Helm Hooks (Automatic Pre-Deployment)

If using Helm, run migrations automatically before deployment.

```yaml
# helm/templates/migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-migrate
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      serviceAccountName: {{ .Release.Name }}-sa
      containers:
      - name: migrate
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command:
          - /bin/sh
          - -c
          - "npm run migrate:up"
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
      restartPolicy: Never
  backoffLimit: 3
```

Deploy with:
```bash
helm upgrade <release> ./chart -n <namespace> --wait
```

## 5. Handling Failing Migrations

### If migration fails mid-way

```bash
# Check job logs
kubectl logs job/migrate-v2 -n <namespace>

# Check database state (may be partially migrated)
kubectl exec -it <rds-pod> -n <namespace> -- psql -U <user> <db>
  # SELECT * FROM schema_migrations;  -- Check what ran

# Rollback the migration
kubectl create job migrate-rollback --image=<image>:new-tag -n <namespace> -- migrate down

# Or manually in the DB
kubectl exec -it <rds-pod> -n <namespace> -- psql -U <user> <db> << EOF
  BEGIN;
  -- Manually undo the changes
  ROLLBACK;
EOF
```

### If app is broken after migration

```bash
# Quick rollback traffic to old version
kubectl patch service <app> -n <namespace> -p '{"spec":{"selector":{"version":"v1"}}}'

# Delete broken deployment
kubectl delete deployment <app>-v2 -n <namespace>

# Re-investigate migration in staging
```

## 6. Zero-Downtime: Expand Then Contract

For very safe large migrations:

### Phase 1: Expand (add new columns/tables)
```bash
# Can run with old code running
kubectl create job expand --image=<image>:migration-expand -n <namespace> -- migrate expand
```

### Phase 2: Deploy new code (reads/writes both)
```bash
# Code handles both old and new schema
kubectl set image deployment/<app> <container>=<image>:new-tag -n <namespace>
```

### Phase 3: Populate (copy data from old to new)
```bash
# Backfill new columns with data from old
kubectl create job backfill --image=<image>:migration-backfill -n <namespace> -- migrate backfill
```

### Phase 4: Contract (drop old columns)
```bash
# Remove old columns
kubectl create job contract --image=<image>:migration-contract -n <namespace> -- migrate contract
```

## 7. Monitoring During Migration

```bash
# Pod status
kubectl get pods -n <namespace> -w

# Database connections
kubectl exec -it <rds-pod> -n <namespace> -- psql -U <user> <db>
  # SELECT count(*) FROM pg_stat_activity;

# Disk space
kubectl exec -it <rds-pod> -n <namespace> -- df -h

# Query slow queries (if issue arises)
kubectl exec -it <rds-pod> -n <namespace> -- psql -U <user> <db> << EOF
  SELECT query, calls, mean_time FROM pg_stat_statements
  WHERE mean_time > 1000 ORDER BY mean_time DESC;
EOF
```

## 8. Verification After Migration

```bash
# Check database schema matches expected version
kubectl exec -it <rds-pod> -n <namespace> -- psql -U <user> <db> << EOF
  SELECT version FROM schema_migrations ORDER BY version DESC LIMIT 5;
EOF

# Run smoke tests
kubectl run smoke-test --image=<image>:new-tag --rm -it -n <namespace> -- npm test

# Monitor app health
kubectl logs -l app=<app> -n <namespace> --tail=100 -f
```

## 9. Rollback Plan (If Everything Breaks)

```bash
# 1. Switch traffic back to old version
kubectl patch service <app> -n <namespace> -p '{"spec":{"selector":{"version":"v1"}}}'

# 2. Revert database (if needed)
kubectl create job rollback-db --image=<migration-image>:v1 -n <namespace> -- migrate down

# 3. Or restore from backup
# Restore from RDS snapshot (takes 5-10 min)
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier <app>-db-restored \
  --db-snapshot-identifier <snapshot-id> \
  --region us-east-1

# 4. Update connection string if using new DB
kubectl set env deployment/<app> -n <namespace> DATABASE_URL=<new-url>
```

## 10. Automation: GitOps Migration Flow

```yaml
# .github/workflows/deploy-migration.yml
name: Deploy with Migration

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build image
        run: |
          docker build -t $REGISTRY/$IMAGE:$GITHUB_SHA .
          docker push $REGISTRY/$IMAGE:$GITHUB_SHA
      
      - name: Create migration job
        run: |
          kubectl set image job/migrate-job -n production \
            migration=$REGISTRY/$IMAGE:$GITHUB_SHA --record
      
      - name: Wait for migration
        run: |
          kubectl wait --for=condition=complete job/migrate-job -n production --timeout=300s
      
      - name: Check migration status
        run: |
          kubectl get job/migrate-job -n production -o wide
          if [ $(kubectl get job/migrate-job -n production -o jsonpath='{.status.failed}') -gt 0 ]; then
            kubectl logs job/migrate-job -n production
            exit 1
          fi
      
      - name: Deploy app
        run: |
          helm upgrade $RELEASE ./chart -n production \
            --set image.tag=$GITHUB_SHA --wait
```

---

**Key Takeaways**:
- **Always backup** before migrations
- **Test in staging** first
- **Blue-green is safest** for production
- **Monitor closely** during and after
- **Keep rollback plan ready**
