# Database Administration Documentation

Database systems, replication strategies, backup and recovery, performance tuning, and connection management.

## Overview

This section covers ACID databases including PostgreSQL and MySQL with emphasis on replication, backups, disaster recovery, and production operations.

## Cookbooks

### [PostgreSQL & MySQL](cookbooks/postgresql-mysql.md)
Enterprise database administration guide. Covers installation, user and database management, tablespaces, streaming replication (primary/replica), backup strategies (logical dumps, continuous archiving, point-in-time recovery), performance tuning, and connection pooling.

**Topics**:
- PostgreSQL installation and configuration
- User roles and privilege management
- Database and schema management
- Tablespace creation and allocation
- Streaming replication setup (primary at 10.0.30.1, replica at 10.0.30.2)
- Base backup procedure and recovery
- Backup strategies (pg_dump, WAL-E for S3 archiving, PITR)
- Performance tuning (shared_buffers, effective_cache_size, work_mem, WAL settings)
- Query optimization and indexing
- VACUUM and ANALYZE operations
- PgBouncer connection pooling (transaction mode for web apps)
- Prometheus exporter integration for metrics
- MySQL quick reference with master/slave replication

**Use Cases**:
- Relational database for applications
- Streaming replication for read scaling
- Continuous archiving for point-in-time recovery
- Connection pooling for web applications
- Metrics collection for monitoring stack

---

## Related Sections

- [Monitoring](../monitoring/) - Prometheus metrics for database health
- [Storage](../storage/) - Backup destination and replication targets
- [Kubernetes](../kubernetes/) - Database in containers and persistent volumes
- [Linux](../linux/) - System tuning for database performance

---

*Last updated: 2024 | Community contributions welcome*
