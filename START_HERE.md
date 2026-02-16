# Start Here

This handbook is organized for day-to-day operations, not long-form reading.

## 1) Use the Right Entry Point

- Full navigation: [docs/index.md](docs/index.md)
- Fast problem lookup: [QUICK_REFERENCE.md](QUICK_REFERENCE.md)
- Repository overview: [README.md](README.md)

## 2) Pick Your Domain

- Kubernetes: [docs/kubernetes/index.md](docs/kubernetes/index.md)
- Linux: [docs/linux/index.md](docs/linux/index.md)
- Docker: [docs/docker/index.md](docs/docker/index.md)
- Windows: [docs/windows/index.md](docs/windows/index.md)
- Automation: [docs/automation/index.md](docs/automation/index.md)
- Enterprise networking: [docs/cisco/index.md](docs/cisco/index.md)
- Virtualization: [docs/vmware/index.md](docs/vmware/index.md)

## 3) Recommended First Reads by Role

### Kubernetes Operator
1. [docs/kubernetes/navigation.md](docs/kubernetes/navigation.md)
2. [docs/kubernetes/triage.md](docs/kubernetes/triage.md)
3. [docs/kubernetes/kubectl.md](docs/kubernetes/kubectl.md)
4. [docs/kubernetes/cookbooks/debug-pod-crash-loop.md](docs/kubernetes/cookbooks/debug-pod-crash-loop.md)

### Linux Administrator
1. [docs/linux/navigation.md](docs/linux/navigation.md)
2. [docs/linux/system.md](docs/linux/system.md)
3. [docs/linux/logs.md](docs/linux/logs.md)
4. [docs/linux/cookbooks/monitoring-performance.md](docs/linux/cookbooks/monitoring-performance.md)

### Windows Administrator
1. [docs/windows/core-powershell.md](docs/windows/core-powershell.md)
2. [docs/windows/services.md](docs/windows/services.md)
3. [docs/windows/event-logs.md](docs/windows/event-logs.md)
4. [docs/windows/cookbooks/server-management-powershell.md](docs/windows/cookbooks/server-management-powershell.md)

### DevOps / Platform Engineer
1. [docs/docker/index.md](docs/docker/index.md)
2. [docs/kubernetes/index.md](docs/kubernetes/index.md)
3. [docs/automation/index.md](docs/automation/index.md)
4. [docs/monitoring/index.md](docs/monitoring/index.md)

## 4) If You Are Troubleshooting Right Now

1. Open [QUICK_REFERENCE.md](QUICK_REFERENCE.md).
2. Find the closest symptom.
3. Jump to the linked cookbook/runbook.
4. Follow that guide end-to-end before branching.

## 5) How to Add New Content

- Add new technical content under the correct folder in `docs/`.
- Prefer cookbook format for step-by-step operational tasks.
- Add or update links in the relevant `index.md`.
- Keep root files minimal (see policy in [README.md](README.md)).
- Put one-time project notes in [docs/history/README.md](docs/history/README.md).

## 6) Maintenance Cadence

- Weekly: add notes when you resolve recurring incidents.
- Monthly: promote repeated fixes into cookbook sections.
- Quarterly: remove stale procedures and broken links.

---

For the current table of contents, always start from [docs/index.md](docs/index.md).
