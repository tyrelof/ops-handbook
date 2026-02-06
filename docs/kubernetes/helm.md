# Helm Reality Check

**Never trust values.yaml.**

## Render
```bash
helm template <release> ./chart -f values.yaml > rendered.yaml
```

## Compare
```bash
kubectl get <resource> -o yaml
```
