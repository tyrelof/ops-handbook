# EKS / AWS Specifics

## Nodes
```bash
kubectl get nodes
kubectl describe node <node>
```

## IRSA (IAM Roles for Service Accounts)
```bash
kubectl describe sa <sa>
aws sts get-caller-identity
```

## Load balancers
```bash
aws elbv2 describe-load-balancers
```
