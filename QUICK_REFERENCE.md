# Quick Reference

Use this page when you need a fast next step during an incident or deployment.

## Symptom → First Document

| Symptom | Start Here |
|---|---|
| Pod crashing / CrashLoopBackOff | [docs/kubernetes/cookbooks/debug-pod-crash-loop.md](docs/kubernetes/cookbooks/debug-pod-crash-loop.md) |
| Deployment stuck or failing rollout | [docs/kubernetes/triage.md](docs/kubernetes/triage.md) |
| Migration failed | [docs/kubernetes/cookbooks/database-migration.md](docs/kubernetes/cookbooks/database-migration.md) |
| 502 / reverse proxy issue | [docs/docker/cookbooks/nginx-reverse-proxy.md](docs/docker/cookbooks/nginx-reverse-proxy.md) |
| Host is slow / high CPU / high memory | [docs/linux/cookbooks/monitoring-performance.md](docs/linux/cookbooks/monitoring-performance.md) |
| SSH access failing | [docs/linux/cookbooks/ssh-management.md](docs/linux/cookbooks/ssh-management.md) |
| Windows service/host issue | [docs/windows/cookbooks/server-management-powershell.md](docs/windows/cookbooks/server-management-powershell.md) |
| AD / Group Policy issue | [docs/windows/cookbooks/active-directory-group-policy.md](docs/windows/cookbooks/active-directory-group-policy.md) |

## 10-Minute Triage Checklists

### Kubernetes
- Check pod status: `kubectl get pods -n <ns> -o wide`
- Check events: `kubectl get events -n <ns> --sort-by='.lastTimestamp'`
- Check logs: `kubectl logs <pod> -n <ns> --tail=100 --previous`
- Check details: `kubectl describe pod <pod> -n <ns>`

### Linux
- Top CPU processes: `ps aux --sort=-%cpu | head -10`
- Top memory processes: `ps aux --sort=-%mem | head -10`
- Disk usage: `df -h`
- Recent system errors: `journalctl -xe`
- Port check: `ss -tpan | grep :<port>`

### Windows (PowerShell)
- Service details: `Get-Service -Name "ServiceName" | Format-List *`
- Top memory processes: `Get-Process | Sort-Object WS -Descending | Select-Object -First 10`
- Disk usage: `Get-Volume`
- Recent errors: `Get-EventLog -LogName System -EntryType Error -Newest 10`
- Port owner: `Get-NetTCPConnection -LocalPort <port>`

## Common Operational Flows

### Production Deploy
- Validate in staging.
- Back up stateful systems if required.
- Run migration plan from [docs/kubernetes/cookbooks/database-migration.md](docs/kubernetes/cookbooks/database-migration.md).
- Monitor rollout and error rates.
- Keep rollback command/path ready.

### New Server Baseline
- SSH/key setup: [docs/linux/cookbooks/ssh-management.md](docs/linux/cookbooks/ssh-management.md)
- Hardening baseline: [docs/linux/cookbooks/security-hardening.md](docs/linux/cookbooks/security-hardening.md)
- Monitoring baseline: [docs/linux/cookbooks/monitoring-performance.md](docs/linux/cookbooks/monitoring-performance.md)
- Logging baseline: [docs/linux/logs.md](docs/linux/logs.md)

### Developer Access Onboarding
- SSH key onboarding: [docs/linux/cookbooks/ssh-management.md](docs/linux/cookbooks/ssh-management.md)
- Cluster access model: [docs/kubernetes/kubectl.md](docs/kubernetes/kubectl.md)
- Cloud/EKS access: [docs/kubernetes/aws-eks.md](docs/kubernetes/aws-eks.md)

## High-Value References

- Master docs index: [docs/index.md](docs/index.md)
- Kubernetes command reference: [docs/kubernetes/kubectl.md](docs/kubernetes/kubectl.md)
- Kubernetes incidents: [docs/kubernetes/incidents.md](docs/kubernetes/incidents.md)
- Linux troubleshooting: [docs/linux/troubleshooting.md](docs/linux/troubleshooting.md)
- Docker debugging: [docs/docker/debugging.md](docs/docker/debugging.md)
- Windows troubleshooting: [docs/windows/troubleshooting.md](docs/windows/troubleshooting.md)

---

If this page doesn’t match your issue, go to [docs/index.md](docs/index.md) and pick the closest domain index.
