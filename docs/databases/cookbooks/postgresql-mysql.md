# PostgreSQL & MySQL - Enterprise Databases

Complete guide to installing, managing, replicating, and optimizing PostgreSQL and MySQL for production workloads.

## Table of Contents
1. [PostgreSQL Fundamentals](#postgresql-fundamentals)
2. [Installation & Setup](#installation--setup)
3. [Database Management](#database-management)
4. [Replication (Streaming)](#replication-streaming)
5. [Backup & Recovery](#backup--recovery)
6. [Performance Tuning](#performance-tuning)
7. [Connection Pooling](#connection-pooling)
8. [MySQL Quick Reference](#mysql-quick-reference)
9. [Monitoring](#monitoring)
10. [Troubleshooting](#troubleshooting)

---

## PostgreSQL Fundamentals

PostgreSQL is a robust, open-source relational database with ACID compliance, advanced data types, and powerful replication.

**Key Features**:
- ACID transactions
- Streaming replication (physical & logical)
- Tablespaces for storage management
- Full-text search
- JSON/JSONB native support
- Row-level security
- Extensions (PostGIS, pg_stat_statements, etc.)

---

## Installation & Setup

### Ubuntu 20.04

```bash
# Add PostgreSQL repository
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt focal-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# Install PostgreSQL 14
sudo apt update
sudo apt install postgresql-14 postgresql-contrib-14

# Start and enable
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Verify
sudo -u postgres psql --version
```

### Initial Configuration

```bash
# Connect as postgres user
sudo -u postgres psql

# Set password for postgres user
ALTER USER postgres WITH PASSWORD 'SecurePassword123';

# Create admin role
CREATE ROLE admin WITH LOGIN SUPERUSER PASSWORD 'AdminPassword123';

# Create application database
CREATE DATABASE appdb OWNER admin;

# Grant permissions
GRANT CONNECT ON DATABASE appdb TO admin;
\c appdb
GRANT ALL ON SCHEMA public TO admin;

# Exit
\q
```

### postgresql.conf Tuning

Edit `/etc/postgresql/14/main/postgresql.conf`:

```ini
# Connections
max_connections = 200
reserved_connections = 10

# Memory
shared_buffers = 256MB        # 25% of system RAM
effective_cache_size = 1GB    # 50-75% of system RAM
work_mem = 4MB                # shared_buffers / max_connections
maintenance_work_mem = 64MB

# WAL (for replication)
wal_level = replica
max_wal_senders = 5
wal_keep_segments = 64

# Checkpoints
checkpoint_completion_target = 0.9
wal_buffers = 16MB

# Logging
logging_collector = on
log_directory = 'pg_log'
log_filename = 'postgresql-%a.log'
log_statement = 'all'
log_duration = on

# Replication slots (for HA)
max_replication_slots = 5
```

Reload config:
```bash
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

---

## Database Management

### Create Database & Users

```sql
-- Create application database
CREATE DATABASE myapp
  OWNER appuser
  ENCODING 'UTF8'
  TEMPLATE template0;

-- Create application user with password
CREATE USER appuser WITH ENCRYPTED PASSWORD 'AppPassword123';

-- Grant permissions
GRANT CONNECT ON DATABASE myapp TO appuser;
\c myapp
GRANT USAGE ON SCHEMA public TO appuser;
GRANT CREATE ON SCHEMA public TO appuser;
GRANT ALL ON ALL TABLES IN SCHEMA public TO appuser;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public TO appuser;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA public TO appuser;

-- Set permissions for future objects
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO appuser;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO appuser;
```

### List Databases & Users

```sql
-- List databases
\l

-- List users/roles
\du

-- Describe table
\d+ table_name

-- Show database size
SELECT datname, pg_size_pretty(pg_database_size(datname)) 
FROM pg_database ORDER BY pg_database_size(datname) DESC;
```

### Tablespaces (Separate Storage)

```sql
-- Create tablespace on fast SSD
CREATE TABLESPACE ssd_space LOCATION '/mnt/ssd/pgdata';

-- Create table on specific tablespace
CREATE TABLE performance_critical (
  id BIGSERIAL PRIMARY KEY,
  data TEXT
) TABLESPACE ssd_space;
```

---

## Replication (Streaming)

### Primary Node Setup

On primary (10.0.30.1):

```bash
# Edit /etc/postgresql/14/main/postgresql.conf
wal_level = replica
max_wal_senders = 5
wal_keep_segments = 64

# Edit /etc/postgresql/14/main/pg_hba.conf
# Add at end:
host    replication     repuser     10.0.30.2/32    md5

# Reload
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

Create replication user:
```sql
CREATE USER repuser WITH REPLICATION ENCRYPTED PASSWORD 'RepPassword123';
```

---

### Replica Node Setup

On replica (10.0.30.2):

```bash
# Stop PostgreSQL if running
sudo systemctl stop postgresql

# Take base backup from primary
sudo -u postgres pg_basebackup \
  -h 10.0.30.1 \
  -U repuser \
  -D /var/lib/postgresql/14/main \
  -Pv \
  -W

# Create recovery.conf (PostgreSQL < 12) or recovery.signal (>= 12)
# PostgreSQL 12+:
sudo -u postgres touch /var/lib/postgresql/14/main/recovery.signal

# Edit postgresql.conf on replica
echo "primary_conninfo = 'host=10.0.30.1 port=5432 user=repuser password=RepPassword123'" | \
  sudo tee -a /etc/postgresql/14/main/postgresql.conf

# Start PostgreSQL
sudo systemctl start postgresql

# Verify replication status (on primary)
sudo -u postgres psql -c "SELECT client_addr, usename, state FROM pg_stat_replication;"
```

---

## Backup & Recovery

### pg_dump (Logical Backup)

```bash
# Dump single database
sudo -u postgres pg_dump myapp > myapp_backup.sql

# Dump with custom format (more compact)
sudo -u postgres pg_dump -Fc myapp > myapp_backup.dump

# Compress
sudo -u postgres pg_dump myapp | gzip > myapp_backup.sql.gz

# Restore
sudo -u postgres psql myapp < myapp_backup.sql

# Restore custom format
sudo -u postgres pg_restore -d myapp myapp_backup.dump
```

### WAL-E (Continuous Archiving)

For point-in-time recovery (PITR), archive WAL segments to S3:

```bash
# Install WAL-E
sudo apt install wal-e

# Configure /etc/postgresql/14/main/postgresql.conf
archive_mode = on
archive_command = 'wal-e wal-push %p'
archive_timeout = 300

# Set AWS credentials
export AWS_ACCESS_KEY_ID=your_key
export AWS_SECRET_ACCESS_KEY=your_secret
export WALE_S3_PREFIX=s3://backup-bucket/postgres

# Test
sudo -u postgres wal-e backup-push
```

---

## Performance Tuning

### Query Analysis

```sql
-- Enable query planning info
EXPLAIN ANALYZE SELECT * FROM large_table WHERE id > 1000000;

-- Check slow query log
SELECT query, mean_exec_time, calls 
FROM pg_stat_statements 
ORDER BY mean_exec_time DESC LIMIT 10;
```

### Index Optimization

```sql
-- Create index on frequently searched column
CREATE INDEX idx_users_email ON users(email);

-- Composite index for range queries
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);

-- Partial index (smaller, faster)
CREATE INDEX idx_active_orders ON orders(id) WHERE status = 'active';

-- Check index usage
SELECT schemaname, tablename, indexname, idx_scan 
FROM pg_stat_user_indexes 
ORDER BY idx_scan DESC;
```

### Vacuum & Analyze

```bash
# Run in cron (daily)
0 2 * * * sudo -u postgres vacuumdb -d myapp -z

# Manual vacuum
sudo -u postgres psql myapp -c "VACUUM ANALYZE;"
```

---

## Connection Pooling

### PgBouncer

Reduce connection overhead for high-concurrency applications:

```bash
# Install
sudo apt install pgbouncer

# Configure /etc/pgbouncer/pgbouncer.ini
[databases]
myapp = host=10.0.30.1 port=5432 dbname=myapp

[pgbouncer]
listen_port = 6432
listen_addr = 0.0.0.0
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
pool_mode = transaction  # or session
log_connections = 1
log_disconnections = 1

# Start
sudo systemctl start pgbouncer
sudo systemctl enable pgbouncer

# Connect via PgBouncer (port 6432)
psql -h 10.0.30.1 -p 6432 -d myapp -U appuser
```

---

## MySQL Quick Reference

### Installation

```bash
sudo apt install mysql-server

# Secure installation
sudo mysql_secure_installation

# Start
sudo systemctl start mysql
sudo systemctl enable mysql
```

### User & Database

```sql
-- Create database
CREATE DATABASE myapp;

-- Create user
CREATE USER 'appuser'@'10.0.30.%' IDENTIFIED BY 'Password123';

-- Grant permissions
GRANT ALL PRIVILEGES ON myapp.* TO 'appuser'@'10.0.30.%';
FLUSH PRIVILEGES;

-- Verify
SHOW GRANTS FOR 'appuser'@'10.0.30.%';
```

### Replication (Master/Slave)

**Master** (`/etc/mysql/conf.d/mysqld.cnf`):
```ini
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
binlog_format = row
```

**Slave** (`/etc/mysql/conf.d/mysqld.cnf`):
```ini
server-id = 2
relay-log = /var/log/mysql/mysql-relay-bin
relay-log-index = /var/log/mysql/mysql-relay-bin.index
```

On master:
```sql
SHOW MASTER STATUS;
```

On slave:
```sql
CHANGE MASTER TO
  MASTER_HOST='10.0.30.1',
  MASTER_USER='repl_user',
  MASTER_PASSWORD='password',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=154;

START SLAVE;
SHOW SLAVE STATUS\G
```

---

## Monitoring

### Key Metrics

```sql
-- Connections
SELECT count(*) as active_connections FROM pg_stat_activity;

-- Replication lag (on replica)
SELECT now() - pg_last_xact_replay_timestamp() as replication_lag;

-- Table sizes
SELECT schemaname, tablename, pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_tables ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC LIMIT 10;

-- Cache hit ratio (should be > 99%)
SELECT 
  sum(blks_hit) / (sum(blks_hit) + sum(blks_read)) as cache_hit_ratio 
FROM pg_stat_database;
```

### Integration with Prometheus

```yaml
# prometheus.yml
scrape_configs:
  - job_name: postgres
    static_configs:
      - targets: ['10.0.30.1:9187']
```

Install postgres_exporter:
```bash
sudo useradd -rs /bin/false postgres_exporter
sudo wget -O /usr/local/bin/postgres_exporter \
  https://github.com/prometheusio/postgres_exporter/releases/download/v0.11.0/postgres_exporter-0.11.0.linux-amd64
sudo chmod +x /usr/local/bin/postgres_exporter
```

---

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| Connection refused | Check `pg_hba.conf`; verify firewall; check listen_addresses |
| High memory usage | Adjust `shared_buffers`, `work_mem`; check for slow queries |
| Replication lag | Increase `wal_level`; check network; review wal_senders |
| Slow queries | Run EXPLAIN ANALYZE; create missing indexes; vacuum table |
| Disk full | Archive/delete old WAL; increase tablespace disk; add more storage |

### Debug Commands

```bash
# Connect and check config
sudo -u postgres psql -c "SHOW all;"

# View PostgreSQL logs
sudo tail -f /var/log/postgresql/postgresql-14-main.log

# Check system resources
vmstat 1 5
iostat -x 1 5
```

---

## Advanced Replication: Cascading Replicas

Chain replicas for better resource usage (Primary → Replica1 → Replica2):

```
Primary (10.0.30.1)
    |
    v
Replica-1 (10.0.30.2) - Receives WAL from Primary
    |
    v
Replica-2 (10.0.30.3) - Receives WAL from Replica-1 (not Primary)
```

**On Replica-1** (`/etc/postgresql/14/main/postgresql.conf`):
```
wal_level = replica
hot_standby = on
hot_standby_feedback = on

# Allow Replica-2 to connect and stream
max_wal_senders = 5
```

**On Replica-2** (`/etc/postgresql/14/main/postgresql.conf`):
```
primary_conninfo = 'host=10.0.30.2 port=5432 user=replication password=...'
```

Benefits:
- Reduces load on primary (Replica-2 doesn't consume primary's wal_sender slot)
- Useful for geographically distributed replicas
- Monitoring: Check `pg_stat_replication` on each relay node

---

## Logical Replication (Selective Tables)

Replicate only specific tables to separate database (useful for analytics/ETL):

**On Publisher (Primary)**:
```sql
-- Enable logical replication
ALTER SYSTEM SET wal_level = logical;
SELECT pg_reload_conf();

-- Create publication for specific tables
CREATE PUBLICATION analytics_pub FOR TABLE orders, customers, products;
```

**On Subscriber (Analytics DB)**:
```sql
-- Create tables (same schema as publisher)
CREATE TABLE orders (...);
CREATE TABLE customers (...);
CREATE TABLE products (...);

-- Create subscription
CREATE SUBSCRIPTION analytics_sub 
CONNECTION 'host=10.0.30.1 port=5432 user=replication password=...'
PUBLICATION analytics_pub;

-- Verify
SELECT * FROM pg_stat_subscription;
```

Use Case: Real-time analytics database receiving only order/customer/product tables, not sensitive data.

---

## Best Practices

1. **Backups**: Daily pg_dump + WAL archiving for PITR
2. **Replication**: Always use streaming for HA; monitor lag closely
3. **Monitoring**: Export metrics to Prometheus; alert on high lag/memory
4. **Tuning**: Match config to available RAM; use dedicated tablespaces for hot data
5. **Security**: Use strong passwords; enable SSL; limit network access
6. **Updates**: Test patches in lab; apply during maintenance window
7. **Connection Pooling**: Use PgBouncer in transaction mode for web apps
8. **Cascading Replicas**: Use for distributed replication to reduce primary load
9. **Logical Replication**: Selective table replication for analytics/ETL pipelines
10. **Monitoring**: Use `pg_stat_replication` to track replica health

---

*PostgreSQL and MySQL guide for production databases, replication, and optimization.*
