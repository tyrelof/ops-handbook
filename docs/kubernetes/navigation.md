# Basic Kubernetes Navigation

## Namespaces
```bash
kubectl get ns
kubectl config set-context --current --namespace=<ns>
kubectl config view --minify --output 'jsonpath={..namespace}'
```

## Core resources
```bash
kubectl get pods
kubectl get svc
kubectl get deploy
kubectl get ingress
kubectl get cm
kubectl get secret
```

## Wide view
```bash
kubectl get pods -o wide
kubectl get nodes -o wide
```
