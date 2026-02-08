# Quick Reference Index

## 🆘 Troubleshooting by Symptom

### Application Issues

| Symptom | Likely Cause | Cookbook | Commands |
|---------|-------------|----------|----------|
| Pod stuck in CrashLoopBackOff | App crash, config error, OOM | [Crash Loops](docs/kubernetes/cookbooks/debug-pod-crash-loop.md) | `kubectl describe pod`, `kubectl logs --previous` |
| 502 Bad Gateway | Upstream unavailable | [Nginx Proxy](docs/docker/cookbooks/nginx-reverse-proxy.md) | `curl upstream:port`, check health endpoint |
| High latency | Slow backend, network issue | [Monitoring](docs/linux/cookbooks/monitoring-performance.md) | `kubectl top pods`, `iotop`, `tcpdump` |
| Connection refused | Service not running | [SSH](docs/linux/cookbooks/ssh-management.md), [Services](docs/windows/cookbooks/server-management-powershell.md) | `ss -tpan`, `netstat`, `Get-Service` |
| Out of memory | Memory leak, too many pods | [Monitoring](docs/linux/cookbooks/monitoring-performance.md) | `top`, `kubectl top`, `Get-Process` |

### Deployment Issues

| Symptom | Solution | Cookbook |
|---------|----------|----------|
| Deployment stuck rolling out | Check pod status, events | [Crash Loops](docs/kubernetes/cookbooks/debug-pod-crash-loop.md) |
| Database migration failed | Rollback, fix migration | [Migrations](docs/kubernetes/cookbooks/database-migration.md) |
| Chart won't install | Validate syntax, check values | [Helm Charts](docs/kubernetes/cookbooks/helm-charts.md) |

### Infrastructure Issues

| Symptom | Solution | Cookbook |
|---------|----------|----------|
| Disk full | Clean logs, temp files | [Monitoring](docs/linux/cookbooks/monitoring-performance.md) or [PowerShell](docs/windows/cookbooks/server-management-powershell.md) |
| High CPU | Identify process, optimize | [Monitoring](docs/linux/cookbooks/monitoring-performance.md) |
| SSH connection issues | Check keys, permissions, firewall | [SSH Management](docs/linux/cookbooks/ssh-management.md) |
| Port already in use | Kill process, restart service | [Windows](docs/windows/cookbooks/server-management-powershell.md) or `lsof -i :port` |

### Network Issues

| Symptom | Check | Cookbook |
|---------|-------|----------|
| DNS not resolving | /etc/resolv.conf, nslookup | [kubectl](docs/kubernetes/kubectl.md) or [Linux Networking](docs/linux/networking.md) |
| No external connectivity | Ingress rules, network policies | [Ingress](docs/kubernetes/ingress.md) |
| High packet loss | traceroute, mtu, bandwidth | [Monitoring](docs/linux/cookbooks/monitoring-performance.md) |

---

## 📋 Common Tasks Checklist

### Deploying to Production

- [ ] Test in staging first
- [ ] Back up database (if needed) - [Migrations](docs/kubernetes/cookbooks/database-migration.md)
- [ ] Run migrations - [Migrations](docs/kubernetes/cookbooks/database-migration.md)
- [ ] Use blue-green or canary deployment
- [ ] Monitor error rates
- [ ] Keep rollback plan ready
- [ ] Update runbook with changes

### Setting Up New Server

- [ ] Configure SSH keys - [SSH Management](docs/linux/cookbooks/ssh-management.md)
- [ ] Harden SSH - [SSH Management](docs/linux/cookbooks/ssh-management.md)
- [ ] Set up monitoring - [Monitoring](docs/linux/cookbooks/monitoring-performance.md)
- [ ] Configure firewall - [Windows](docs/windows/cookbooks/server-management-powershell.md)
- [ ] Set up log rotation - [Linux Logs](docs/linux/logs.md)
- [ ] Create backup plan - [Storage](docs/kubernetes/storage.md)

### Debugging Performance Issue

1. Check CPU - [Monitoring](docs/linux/cookbooks/monitoring-performance.md) - `top`, `sar`
2. Check Memory - [Monitoring](docs/linux/cookbooks/monitoring-performance.md) - `free`, `ps`
3. Check Disk I/O - [Monitoring](docs/linux/cookbooks/monitoring-performance.md) - `iostat`, `iotop`
4. Check Network - [Monitoring](docs/linux/cookbooks/monitoring-performance.md) - `netstat`, `ss`, `tcpdump`
5. Check Logs - [Linux Logs](docs/linux/logs.md) - `journalctl`, `/var/log/`
6. Check Application - [Crash Loops](docs/kubernetes/cookbooks/debug-pod-crash-loop.md) - `strace`, profiler

### Onboarding Developer Access

- [ ] Generate SSH key - [SSH Management](docs/linux/cookbooks/ssh-management.md)
- [ ] Add to authorized_keys - [SSH Management](docs/linux/cookbooks/ssh-management.md)
- [ ] Configure kubectl access - [kubectl](docs/kubernetes/kubectl.md)
- [ ] Grant IAM permissions - [AWS EKS](docs/kubernetes/aws-eks.md)
- [ ] Share relevant cookbooks
- [ ] Review security policies

---

## 🔗 Essential Commands Reference

### Kubernetes Diagnostics
```bash
# Pod status
kubectl get pods -n <ns> -o wide

# Recent events
kubectl get events -n <ns> --sort-by='.lastTimestamp'

# Pod logs (last 100 lines, follow)
kubectl logs <pod> -n <ns> --tail=100 -f

# Previous run (crashed pod)
kubectl logs <pod> -n <ns> --previous

# Pod details
kubectl describe pod <pod> -n <ns>

# Check which node
kubectl get pods -n <ns> -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
```

### Linux Diagnostics
```bash
# Top processes by CPU
ps aux --sort=-%cpu | head -10

# Top processes by memory
ps aux --sort=-%mem | head -10

# Disk usage
df -h

# Memory stats
free -h

# Active connections
ss -tpan | grep ESTAB

# System load
uptime

# Last errors
journalctl -xe

# Port listener
ss -tpan | grep :3000
```

### Windows Diagnostics (PowerShell)
```powershell
# Service status
Get-Service -Name "ServiceName" | Format-List *

# Top processes by memory
Get-Process | Sort-Object WS -Descending | Select-Object -First 10

# Disk space
Get-Volume

# Network stats
Get-NetTCPConnection | Group-Object State

# Event log errors
Get-EventLog -LogName System -EntryType Error -Newest 10

# Find process on port
Get-NetTCPConnection -LocalPort 3000
```

---

## 📚 Documentation by Type

### Getting Started
- [Kubernetes Navigation](docs/kubernetes/navigation.md)
- [Linux Navigation](docs/linux/navigation.md)
- [PowerShell Basics](docs/windows/core-powershell.md)
- [Docker Lifecycle](docs/docker/lifecycle.md)

### Step-by-Step Guides (Cookbooks)
- [Complete Docker Stack](docs/docker/cookbooks/complete-stack.md)
- [Nginx Proxy](docs/docker/cookbooks/nginx-reverse-proxy.md)
- [Pod Crash Loop Debug](docs/kubernetes/cookbooks/debug-pod-crash-loop.md)
- [Database Migrations](docs/kubernetes/cookbooks/database-migration.md)
- [Helm Charts](docs/kubernetes/cookbooks/helm-charts.md)
- [SSH Setup](docs/linux/cookbooks/ssh-management.md)
- [Performance Monitoring](docs/linux/cookbooks/monitoring-performance.md)
- [Nginx from Scratch](docs/linux/cookbooks/web-server-nginx.md)
- [Server Hardening](docs/linux/cookbooks/security-hardening.md)
- [Windows Server Mgmt](docs/windows/cookbooks/server-management-powershell.md)
- [IIS Web Server](docs/windows/cookbooks/iis-web-server.md)

### Reference Guides (Lookup)
- [kubectl Commands](docs/kubernetes/kubectl.md)
- [Ingress Setup](docs/kubernetes/ingress.md)
- [Services & Endpoints](docs/kubernetes/services.md)
- [Pod Configuration](docs/kubernetes/pods.md)
- [Helm Guide](docs/kubernetes/helm.md)
- [ArgoCD GitOps](docs/kubernetes/argocd.md)
- [Secrets Management](docs/kubernetes/secrets.md)
- [Storage & Backups](docs/kubernetes/storage.md)
- [RDS Operations](docs/kubernetes/rds.md)
- [Redis Diagnostics](docs/kubernetes/redis.md)
- [Linux Text Processing](docs/linux/text-processing.md)
- [System Management](docs/linux/system.md)
- [Docker Networking](docs/docker/networking.md)
- [Docker Volumes](docs/docker/volumes.md)

### Troubleshooting Guides
- [Kubernetes Triage](docs/kubernetes/triage.md)
- [Pod Crash Loops](docs/kubernetes/cookbooks/debug-pod-crash-loop.md)
- [Linux Troubleshooting](docs/linux/troubleshooting.md)
- [Docker Debugging](docs/docker/debugging.md)
- [Windows Troubleshooting](docs/windows/troubleshooting.md)

### Incident & Operational Guides
- [Kubernetes Incidents](docs/kubernetes/incidents.md)
- [Kubernetes Postmortems](docs/kubernetes/postmortem.md)
- [Disaster Recovery](docs/kubernetes/dr.md)
- [Observability](docs/kubernetes/observability.md)
- [Guardrails](docs/kubernetes/guardrails.md)

---

## 🎓 Learning Paths

### Path 1: Kubernetes Operator
1. [Navigation](docs/kubernetes/navigation.md)
2. [Triage Checklist](docs/kubernetes/triage.md)
3. [kubectl Reference](docs/kubernetes/kubectl.md)
4. [Pod Crash Loops](docs/kubernetes/cookbooks/debug-pod-crash-loop.md)
5. [Ingress & Services](docs/kubernetes/ingress.md)
6. [Database Migrations](docs/kubernetes/cookbooks/database-migration.md)
7. [Helm Charts](docs/kubernetes/cookbooks/helm-charts.md)

### Path 2: Linux System Administrator
1. [Navigation](docs/linux/navigation.md)
2. [System Management](docs/linux/system.md)
3. [Logs & Journald](docs/linux/logs.md)
4. [SSH Management](docs/linux/cookbooks/ssh-management.md)
5. [Performance Monitoring](docs/linux/cookbooks/monitoring-performance.md)
6. [Nginx Setup](docs/linux/cookbooks/web-server-nginx.md)
7. [Security Hardening](docs/linux/cookbooks/security-hardening.md)

### Path 3: Docker Developer
1. [Docker Lifecycle](docs/docker/lifecycle.md)
2. [Images & Builds](docs/docker/images.md)
3. [Docker Compose](docs/docker/compose.md)
4. [Full Stack Setup](docs/docker/cookbooks/complete-stack.md)
5. [Nginx Proxy](docs/docker/cookbooks/nginx-reverse-proxy.md)
6. [Networking](docs/docker/networking.md)

### Path 4: Windows Administrator
1. [PowerShell Basics](docs/windows/core-powershell.md)
2. [Services & Processes](docs/windows/services.md)
3. [Event Logs](docs/windows/event-logs.md)
4. [Server Management](docs/windows/cookbooks/server-management-powershell.md)
5. [IIS Web Server](docs/windows/cookbooks/iis-web-server.md)
6. [Networking & Firewall](docs/windows/networking.md)

---

## 🔐 Security Checklists

### SSH Hardening
- [ ] Use Ed25519 keys
- [ ] Disable password auth
- [ ] Disable root login
- [ ] Change default port (optional)
- [ ] Use key rotation
- See: [SSH Management](docs/linux/cookbooks/ssh-management.md)

### Kubernetes Security
- [ ] Enable RBAC
- [ ] Set pod security policies
- [ ] Use network policies
- [ ] Encrypt secrets at rest
- [ ] Regular security scans
- See: [Secrets](docs/kubernetes/secrets.md), [Guardrails](docs/kubernetes/guardrails.md)

### Server Hardening
- [ ] Firewall rules
- [ ] Disable unused services
- [ ] Regular patching
- [ ] Audit logging
- [ ] SSH key-based auth
- See: [SSH Management](docs/linux/cookbooks/ssh-management.md), [Server Hardening](docs/linux/cookbooks/security-hardening.md)

---

## 📞 Emergency Procedures

### Service Down (5 min response)
1. Check service status
2. Check recent logs for errors
3. Check resource usage (CPU/memory/disk)
4. Restart if needed
5. If restart fails → investigate further

See: [Triage](docs/kubernetes/triage.md), [Troubleshooting](docs/linux/troubleshooting.md)

### Database Issue (15 min response)
1. Check database connection
2. Check recent errors in logs
3. Check disk space (if full)
4. Check query performance
5. Consider failover or restore

See: [RDS Operations](docs/kubernetes/rds.md), [Database Migrations](docs/kubernetes/cookbooks/database-migration.md)

### Deployment Issue (10 min response)
1. Check pod status
2. Check recent events
3. Check resource requests/limits
4. Check health probes
5. Consider rollback

See: [Pod Crash Loops](docs/kubernetes/cookbooks/debug-pod-crash-loop.md), [kubectl](docs/kubernetes/kubectl.md)

---

**Last Updated**: February 2026  
**Maintainer**: Tyrel Orde Fecha
