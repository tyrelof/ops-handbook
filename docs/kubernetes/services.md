# Service → EndpointSlice

## Service exists?
```bash
kubectl get svc <svc>
```

## Endpoints exist?
```bash
kubectl get endpointslice -l kubernetes.io/service-name=<svc>
```

## Selector match?
```bash
kubectl get svc <svc> -o jsonpath='{.spec.selector}'
kubectl get pods --show-labels
```
