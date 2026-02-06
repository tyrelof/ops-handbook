# Ingress & Load Balancing

## Inspect ingress
```bash
kubectl get ingress -o yaml
kubectl describe ingress <name>
```

## Inspect routing truth
```bash
kubectl get ingress <name> -o yaml | yq '.spec.rules[].http.paths'
```

## Ingress controller logs
```bash
kubectl -n ingress-nginx logs deploy/ingress-nginx-controller --tail=200
```

## Common failures
- Catch-all / overrides specific paths
- Helm template ignores per-path backends
- Service name mismatch
