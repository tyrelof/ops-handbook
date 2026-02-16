# Infrastructure & Platform Reference Book

A practical operations handbook for day-to-day infrastructure work.

## Author
**Tyrel Orde Fecha**
*SysAdmin | Network Admin | Cloud Engineer | DevOps | Platform Engineer*

## Entry Points

Start with one of these:

- **[Documentation Hub](docs/index.md)**: Canonical index for all domains and cookbooks.
- **[Start Here](START_HERE.md)**: Role-based onboarding path.
- **[Quick Reference](QUICK_REFERENCE.md)**: Incident-first troubleshooting and checklists.

Primary technical sections:

- **[Kubernetes / EKS](docs/kubernetes/index.md)**
- **[Linux](docs/linux/index.md)**
- **[Docker](docs/docker/index.md)**
- **[Windows](docs/windows/index.md)**

Additional domains are available from **[docs/index.md](docs/index.md)** (Cisco, VMware, Databases, Monitoring, Networking, Security, Storage, Automation).

## How to Use

1. Open **[docs/index.md](docs/index.md)**.
2. Select the closest domain section.
3. Use reference pages for lookup and cookbooks for procedures.
4. For active incidents, start from **[QUICK_REFERENCE.md](QUICK_REFERENCE.md)**.

## Root File Policy

Keep the repository root minimal and navigation-focused.

- Keep only **README.md**, **START_HERE.md**, **QUICK_REFERENCE.md**, and **docs/** at root.
- Add all domain content under **docs/** and its section folders.
- Store one-time rollout, expansion, or migration notes in **docs/history/**.
- Treat **[docs/index.md](docs/index.md)** as the canonical navigation source.

Legacy expansion snapshots are archived in **[docs/history/README.md](docs/history/README.md)**.

---
Personal Infrastructure Knowledge Base  
Versioned and continuously refined through production experience.
