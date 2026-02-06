# Secrets & Encryption (ExternalSecrets, KMS)

## Status
```bash
kubectl get externalsecret
kubectl describe externalsecret <name>
```

## Generated secret
```bash
kubectl get secret <name> -o yaml
```

## Decode
```bash
kubectl get secret <name> -o jsonpath='{.data.KEY}' | base64 -d
```

## Force refresh
```bash
kubectl annotate externalsecret <name> force-sync="$(date +%s)" --overwrite
```
