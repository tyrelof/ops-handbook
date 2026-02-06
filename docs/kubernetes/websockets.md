# WebSockets / Reverb (Patterns)

## Correct pattern
- Browser → 443
- Ingress → reverb:6001

## Check routing
```bash
kubectl get ingress <name> -o yaml | yq '.spec.rules[].http.paths'
```

## Check reverb
```bash
kubectl logs deploy/<app>-reverb
kubectl exec -it <reverb-pod> -- ss -ltnp | grep 6001
```

> [!WARNING]
> Frontend must **NOT** use `:6001`.
