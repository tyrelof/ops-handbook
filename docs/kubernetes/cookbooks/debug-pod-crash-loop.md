# Cookbook: Debugging Pod Crash Loops

> [!NOTE]
> A pod stuck in `CrashLoopBackOff` means the container starts, fails, and restarts repeatedly.
> This guide walks through systematic root-cause analysis.

## 1. Quick Status Check

```bash
# See pod status and restart count
kubectl get pods -n <namespace> -o wide

# Get detailed pod info and recent events
kubectl describe pod <pod-name> -n <namespace>
```

**Look for**:
- `RestartCount`: How many times has this crashed?
- `LastState.Terminated.Reason`: Why did it exit? (ExitCode, Signal, Message)
- `Events` section: Shows init failures, OOM kills, or explicit errors

## 2. Check the Logs (Previous Run)

```bash
# Logs from the LAST failed container instance
kubectl logs <pod-name> -n <namespace> --previous

# If pod won't start at all, logs may be minimal
kubectl logs <pod-name> -n <namespace>
```

**Common exit codes**:
- `Exit 1, 127, 137`: App crashed or missing dependency
- `137`: OOM (Out of Memory) kill
- `139`: Segmentation fault (C/C++ crash)
- `Exit 0`: Pod exited normally but probes expect it running

## 3. Check Resource Constraints

```bash
# View actual resource usage and limits
kubectl top pod <pod-name> -n <namespace>

# Check pod definition for resource requests/limits
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].resources}'
```

**If memory is exhausted**:
```yaml
# Fix in deployment/values.yaml
resources:
  requests:
    memory: "512Mi"  # ← Increase initial request
  limits:
    memory: "1Gi"    # ← Increase hard limit
```

## 4. Check Init Containers & Environment

```bash
# Show all init containers
kubectl describe pod <pod-name> -n <namespace> | grep -A 20 "Init Containers"

# Check all environment variables actually set
kubectl exec <pod-name> -n <namespace> -- env | sort
```

**Common issues**:
- Init container fails (e.g., config not ready, sidecar injection fails)
- Missing `ConfigMap` or `Secret` the app needs
- Wrong image tag or missing image

## 5. Manual Pod Inspection

### Start a shell in the pod (if it stays running)
```bash
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh
# Inside pod:
ls -la /app
ps aux
cat /proc/1/status  # PID 1 process status
```

### Dry-run with debug image
```bash
# If the image is broken, spawn a helper container from the same image
kubectl run debug-pod -it --image=<pod-image> --rm -- /bin/sh
```

## 6. Check Probes (Liveness/Readiness)

```bash
# View probe configuration
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].livenessProbe}'

# Check probe logs (often hidden, but events show failures)
kubectl describe pod <pod-name> -n <namespace> | grep -i "probe\|readiness\|liveness"
```

**Probe fix**:
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 30  # ← Give app time to start!
  periodSeconds: 10
  failureThreshold: 3
```

## 7. Deployment Manifest Check

```bash
# Review the full pod spec
kubectl get deployment <deployment-name> -n <namespace> -o yaml

# Check for:
# - Wrong image tag
# - Missing env vars
# - Volume mount paths
# - Service account permissions
```

## 8. Common Fixes Checklist

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| Immediate exit (RestartCount high) | App is exiting too quickly | Check logs, fix app code, increase `initialDelaySeconds` |
| `CreateContainerConfigError` | ConfigMap/Secret missing | Verify ConfigMap/Secret exists and is named correctly |
| `ImagePullBackOff` | Image doesn't exist or no auth | Check image name, registry credentials |
| Memory exhaustion | App memory leak or too large | Increase memory limit, check for leaks with `top` |
| DNS/network fail | Service/DNS not ready | Ensure dependency services are running, check service name |

## 9. Drill Down: Enable Debug Logging

```bash
# For apps with log levels
kubectl set env deployment/<name> -n <namespace> LOG_LEVEL=debug

# Or patch to add debug flag
kubectl patch deployment <name> -n <namespace> -p '{"spec":{"template":{"spec":{"containers":[{"name":"<container>","args":["--debug"]}]}}}}'
```

## 10. Nuclear Option: Full Pod Debug Container

```bash
# Attach a sidecar that won't crash
kubectl patch deployment <name> -n <namespace> --type='json' -p='[{
  "op": "add",
  "path": "/spec/template/spec/containers/1",
  "value": {
    "name": "debug",
    "image": "busybox:latest",
    "command": ["sleep", "3600"]
  }
}]'

# Now exec into the debug container
kubectl exec -it <pod-name> -c debug -n <namespace> -- /bin/sh
```

---

**Next steps**:
- Fix the identified issue
- Redeploy: `kubectl rollout restart deployment/<name> -n <namespace>`
- Monitor: `kubectl get pods -n <namespace> -w` (watch mode)
- View full logs: `kubectl logs <pod-name> -n <namespace> -f`
