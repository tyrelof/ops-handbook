# Safe kubectl Patterns

- Always use `--namespace=<ns>` for clarity (or set context contextually).
- Use `kubectl get ... -o wide` to reveal more info (IPs, Nodes).
- Prefer `kubectl describe` over `kubectl edit` for diagnostics.
- Use `kubectl logs -f` for live log tailing.
- For resource diffs, use `kubectl diff -f <file>`.

## Essential kubectl Commands

### View Resources
```bash
# List all pods in a namespace
kubectl get pods -n <namespace>

# Show more details
kubectl get pods -n <namespace> -o wide

# JSON output (for scripting)
kubectl get pods -n <namespace> -o json

# YAML output
kubectl get pods -n <namespace> -o yaml

# Custom columns
kubectl get pods -n <namespace> -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP

# Watch resources in real-time
kubectl get pods -n <namespace> -w
```

### Describe Resources
```bash
# Full details and recent events
kubectl describe pod <pod-name> -n <namespace>

# Describe deployment
kubectl describe deployment <deployment-name> -n <namespace>

# Describe service
kubectl describe svc <service-name> -n <namespace>
```

### Logs
```bash
# Stream logs
kubectl logs <pod-name> -n <namespace> -f

# Last 100 lines
kubectl logs <pod-name> -n <namespace> --tail=100

# From previous run (crashed pod)
kubectl logs <pod-name> -n <namespace> --previous

# All containers in pod
kubectl logs <pod-name> -n <namespace> --all-containers=true

# Specific container
kubectl logs <pod-name> -c <container-name> -n <namespace>

# Timestamps
kubectl logs <pod-name> -n <namespace> --timestamps=true
```

### Exec (Run Commands in Pod)
```bash
# Interactive shell
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# Run single command
kubectl exec <pod-name> -n <namespace> -- ps aux

# Specific container
kubectl exec -it <pod-name> -c <container-name> -n <namespace> -- /bin/sh

# As specific user
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh -c "whoami"
```

### Port Forwarding
```bash
# Forward local port to pod
kubectl port-forward <pod-name> 8080:3000 -n <namespace>

# Forward to service
kubectl port-forward svc/<service-name> 8080:80 -n <namespace>

# All interfaces (dangerous)
kubectl port-forward <pod-name> 8080:3000 --address=0.0.0.0
```

### Copy Files
```bash
# Copy from pod to local
kubectl cp <namespace>/<pod-name>:/app/file.txt ./local-file.txt

# Copy from local to pod
kubectl cp ./local-file.txt <namespace>/<pod-name>:/app/file.txt

# Use specific container
kubectl cp ./file.txt <namespace>/<pod-name>:/tmp/ -c <container-name>
```

### Apply & Update
```bash
# Apply resource (idempotent)
kubectl apply -f manifest.yaml -n <namespace>

# Apply directory
kubectl apply -f ./manifests/ -n <namespace>

# Dry-run to see what would happen
kubectl apply -f manifest.yaml --dry-run=client

# Validate yaml
kubectl apply -f manifest.yaml --dry-run=client -o yaml

# Show what changed
kubectl diff -f manifest.yaml -n <namespace>
```

### Delete Resources
```bash
# Delete pod
kubectl delete pod <pod-name> -n <namespace>

# Delete by label
kubectl delete pods -l app=my-app -n <namespace>

# Delete deployment (cascades to pods)
kubectl delete deployment <deployment-name> -n <namespace>

# Force delete (ungraceful)
kubectl delete pod <pod-name> -n <namespace> --grace-period=0 --force

# Delete all in namespace (careful!)
kubectl delete all -n <namespace>
```

### Labels & Selectors
```bash
# Show labels
kubectl get pods -n <namespace> --show-labels

# Filter by label
kubectl get pods -l app=my-app -n <namespace>

# Multiple labels (AND)
kubectl get pods -l app=my-app,env=prod -n <namespace>

# Add label to pod
kubectl label pod <pod-name> env=prod -n <namespace>

# Update label
kubectl label pod <pod-name> env=staging --overwrite -n <namespace>

# Remove label
kubectl label pod <pod-name> env- -n <namespace>
```

### Rollout & Deployment Control
```bash
# View rollout history
kubectl rollout history deployment/<deployment-name> -n <namespace>

# See details of specific revision
kubectl rollout history deployment/<deployment-name> --revision=2 -n <namespace>

# Rollback to previous version
kubectl rollout undo deployment/<deployment-name> -n <namespace>

# Rollback to specific revision
kubectl rollout undo deployment/<deployment-name> --to-revision=2 -n <namespace>

# Pause rollout (useful for canary deployments)
kubectl rollout pause deployment/<deployment-name> -n <namespace>

# Resume rollout
kubectl rollout resume deployment/<deployment-name> -n <namespace>

# Watch rollout status
kubectl rollout status deployment/<deployment-name> -n <namespace>

# Restart deployment (rolling restart)
kubectl rollout restart deployment/<deployment-name> -n <namespace>
```

### Scale Resources
```bash
# Scale deployment
kubectl scale deployment/<deployment-name> --replicas=5 -n <namespace>

# Scale statefulset
kubectl scale statefulset/<statefulset-name> --replicas=3 -n <namespace>
```

### Edit Resources
```bash
# Edit resource in default editor
kubectl edit deployment/<deployment-name> -n <namespace>

# Edit with specific editor
KUBE_EDITOR=nano kubectl edit deployment/<deployment-name> -n <namespace>

# Patch resource (careful, not recommended for large changes)
kubectl patch deployment/<deployment-name> -n <namespace> -p '{"spec":{"replicas":5}}'

# Patch with JSON merge
kubectl patch pod <pod-name> -n <namespace> --type merge -p '{"metadata":{"labels":{"env":"prod"}}}'
```

### Events & Troubleshooting
```bash
# Show recent events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Watch events live
kubectl get events -n <namespace> -w

# Describe pod (includes events)
kubectl describe pod <pod-name> -n <namespace>

# Check node status
kubectl get nodes

# Describe node
kubectl describe node <node-name>

# Get pod conditions
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.conditions[*]}'
```

### Context & Namespace
```bash
# Get current context
kubectl config current-context

# List contexts
kubectl config get-contexts

# Switch context
kubectl config use-context <context-name>

# Get current namespace
kubectl config view --minify -o jsonpath='{.context.namespace}'

# Set default namespace for context
kubectl config set-context --current --namespace=<namespace>

# Get all namespaces
kubectl get ns

# Create namespace
kubectl create namespace <namespace>
```

## Advanced Patterns

### Bulk Operations
```bash
# Get all resources across all namespaces
kubectl get all -A

# Apply to all namespaces
kubectl apply -f manifest.yaml -n $(kubectl get ns -o jsonpath='{.items[*].metadata.name}')

# Delete all failed pods
kubectl delete pods --all-namespaces --field-selector status.phase=Failed
```

### JSONPath Queries
```bash
# Get pod IPs
kubectl get pods -n <namespace> -o jsonpath='{.items[*].status.podIP}'

# Get pod names
kubectl get pods -n <namespace> -o jsonpath='{.items[*].metadata.name}'

# Get specific field
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.phase}'

# Complex query
kubectl get pods -n <namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

### Useful Aliases
```bash
# Add to ~/.bashrc or ~/.zshrc
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgd='kubectl get deployment'
alias kgs='kubectl get svc'
alias kl='kubectl logs'
alias ke='kubectl exec -it'
alias kd='kubectl describe'
alias ka='kubectl apply -f'
alias kdel='kubectl delete'
alias kctx='kubectl config current-context'
```

---

**Golden Rules**:
- Always specify namespace explicitly in scripts
- Use `--dry-run=client` before applying changes
- Prefer `kubectl apply` over `kubectl create`
- Use labels for filtering, not manual selection
- Check events and logs first when troubleshooting
- Test in non-production clusters first
