# Quick Triage Checklist (First 10 minutes)

1. **Check pod status**:
   ```bash
   kubectl get pods -n <ns>
   ```
2. **Check ingress**:
   ```bash
   kubectl get ingress -n <ns> -o wide
   ```
3. **Check service endpoints**:
   ```bash
   kubectl get endpointslice -n <ns>
   ```
4. **Check recent events**:
   ```bash
   kubectl get events -n <ns> --sort-by='.metadata.creationTimestamp'
   ```
5. **Check logs of affected pods**:
   ```bash
   kubectl logs <pod> -n <ns> --tail=100
   ```
