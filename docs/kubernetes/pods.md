# Pods, Probes, Containers

## Pod health
```bash
kubectl get pods
kubectl describe pod <pod>
```

## Logs
```bash
kubectl logs <pod>
kubectl logs <pod> -c <container>
kubectl logs deploy/<deploy>
```

## Check ports
```bash
kubectl exec -it <pod> -- ss -ltnp
```

## Probe rules
- **Startup probe** must allow cold boot.
- **Readiness probe** controls traffic (should fail if app is alive but not ready).
- **Liveness probe** restarts only if wedged (deadlocked).
