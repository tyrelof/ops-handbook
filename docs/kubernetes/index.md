# EKS / Kubernetes / AWS Platform Runbook

**Audience**: Platform Engineers, DevOps, SRE
**Intent**: This document is a single source of truth for operating, debugging, and stabilizing workloads on AWS EKS.

## Core Mental Model

Always reason **outside → inside**:

`Browser → DNS → AWS LB (ALB/NLB) → Ingress Controller → Kubernetes Service → EndpointSlice → Pod → Container → App → Dependency (DB/Redis/S3/External API)`

If something is broken, only one hop is broken. Find it.

## Contents

- [Basic Kubernetes Navigation](navigation.md)
- [Quick Triage Checklist](triage.md)
- [Safe kubectl Patterns](kubectl.md)
- [Ingress & Load Balancing](ingress.md)
- [Service → EndpointSlice](services.md)
- [Pods, Probes, Containers](pods.md)
- [Helm Reality Check](helm.md)
- [ArgoCD / GitOps](argocd.md)
- [Secrets & Encryption](secrets.md)
- [WebSockets / Reverb](websockets.md)
- [EKS / AWS Specifics](aws-eks.md)
- [Storage & Backups](storage.md)
- [RDS (DB triage & restore)](rds.md)
- [Redis Diagnostics](redis.md)
- [Observability & Alerts](observability.md)
- [Platform Guardrails](guardrails.md)
- [Incident Appendix](incidents.md)
- [Disaster Recovery Playbooks](dr.md)
- [Postmortem & Templates](postmortem.md)
- [Tooling Cheatsheet](cheatsheet.md)
- [Glossary](glossary.md)
