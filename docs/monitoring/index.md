# Monitoring & Observability Documentation

Metrics collection, visualization, alerting, and observability best practices for production systems.

## Overview

This section covers modern observability stacks including metrics (Prometheus), visualization (Grafana), alerting, and troubleshooting techniques.

## Cookbooks

### [Prometheus & Grafana](cookbooks/prometheus-grafana.md)
Complete monitoring and visualization stack. Covers Prometheus installation and configuration (static and dynamic service discovery), Grafana dashboards, custom queries with PromQL, alerting with Alertmanager, and data retention policies.

**Topics**:
- Prometheus metrics collection and scraping
- Service discovery (static targets, file-based, dynamic)
- PromQL query language and operators
- Grafana data sources and dashboard import
- Official dashboard library (Node Exporter, PostgreSQL, Nginx, etc.)
- Custom dashboard creation
- Alertmanager configuration and routing
- Alert rules (CPU, memory, disk, service health)
- Data retention and remote storage backends

**Use Cases**:
- Infrastructure monitoring (servers, networks, storage)
- Application performance monitoring (APM)
- Kubernetes cluster observability
- Database and service health monitoring
- SLA tracking and alerting

---

## Related Sections

- [Kubernetes Observability](../kubernetes/observability.md) - Pod metrics, cluster health
- [Docker Monitoring](../docker/debugging.md) - Container metrics, logging
- [Linux System Monitoring](../linux/system.md) - System metrics, performance analysis
- [Databases Monitoring](../databases/) - Query metrics, replication lag

---

*Last updated: 2024 | Community contributions welcome*
