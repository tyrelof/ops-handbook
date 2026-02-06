# Tooling Cheatsheet & Debug Recipes

## Common commands
```bash
kubectl get all -n <ns>
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> --tail=100
```

## Useful scripts
- **Restart deployment**:
  ```bash
  kubectl rollout restart deploy/<deploy> -n <ns>
  ```
- **Scale deployment**:
  ```bash
  kubectl scale deploy/<deploy> --replicas=<num> -n <ns>
  ```
- **Get pod shell**:
  ```bash
  kubectl exec -it <pod> -- /bin/sh
  ```
