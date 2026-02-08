# MinIO - S3-Compatible Object Storage

Complete guide to MinIO for S3-compatible object storage, bucket policies, replication, and disaster recovery.

## Table of Contents
1. [Installation](#installation)
2. [Single-Server Setup](#single-server-setup)
3. [Distributed Cluster](#distributed-cluster)
4. [Bucket Management](#bucket-management)
5. [Access Control & Policies](#access-control--policies)
6. [Replication](#replication)
7. [Lifecycle & Tiering](#lifecycle--tiering)
8. [Monitoring & Metrics](#monitoring--metrics)
9. [Backup & Disaster Recovery](#backup--disaster-recovery)
10. [Troubleshooting](#troubleshooting)

---

## Installation

### Single-Server (Development)

```bash
# Download MinIO binary
cd /opt
sudo wget https://dl.min.io/server/minio/release/linux-amd64/minio
sudo chmod +x /opt/minio

# Create data directory
sudo mkdir -p /mnt/minio-data

# Run MinIO
/opt/minio server /mnt/minio-data

# Output:
# Endpoint:  http://192.168.1.100:9000
# RootUser: minioadmin
# RootPass: minioadmin
```

### Docker Single-Server

```bash
docker run -d \
  --name minio \
  -p 9000:9000 \
  -p 9001:9001 \
  -e MINIO_ROOT_USER=admin \
  -e MINIO_ROOT_PASSWORD=secure_password_123 \
  -v minio-data:/data \
  minio/minio:latest server /data --console-address ":9001"

# Access:
# API:     http://localhost:9000
# Console: http://localhost:9001
```

### Systemd Service

```bash
sudo cat > /etc/systemd/system/minio.service <<EOF
[Unit]
Description=MinIO Object Storage
After=network.target

[Service]
Type=simple
User=minio
Group=minio
ExecStart=/opt/minio server /mnt/minio-data
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo useradd -r minio
sudo chown -R minio:minio /mnt/minio-data

sudo systemctl daemon-reload
sudo systemctl enable minio
sudo systemctl start minio
```

---

## Single-Server Setup

### Access MinIO

```bash
# Install MinIO CLI
sudo curl -o /usr/local/bin/mc https://dl.min.io/client/mc/release/linux-amd64/mc
sudo chmod +x /usr/local/bin/mc

# Add MinIO server alias
mc alias set minio http://localhost:9000 admin secure_password_123

# Test connection
mc ls minio
```

### Web Console

Access at `http://localhost:9001` (or port from output):
- Username: minioadmin (or MINIO_ROOT_USER)
- Password: minioadmin (or MINIO_ROOT_PASSWORD)

---

## Distributed Cluster

### 4-Node Cluster Setup (10.0.30.20-23)

High availability with automatic failover.

**Install on all 4 nodes**:
```bash
# Create erasure-coded cluster (requires minimum 4 nodes)
# Each node hosts 1 disk

# Node 1 (10.0.30.20):
/opt/minio server \
  http://10.0.30.20:9000/mnt/minio-data \
  http://10.0.30.21:9000/mnt/minio-data \
  http://10.0.30.22:9000/mnt/minio-data \
  http://10.0.30.23:9000/mnt/minio-data

# Node 2-4: Run same command (all participate in cluster)
```

All nodes must know about all other nodes.

### Verify Cluster

```bash
mc admin info minio

# Output:
# Uptime: 2 hours
# Version: RELEASE.2025-02-07
# Network: 4 nodes, all online
# Erasure Configuration: 4-2 (4 data drives, 2 parity drives per set)
```

### Load Balancer in Front

```
Client
  |
  v
HAProxy (10.0.10.254:9000)
  |
  +---> MinIO-1 (10.0.30.20:9000)
  +---> MinIO-2 (10.0.30.21:9000)
  +---> MinIO-3 (10.0.30.22:9000)
  +---> MinIO-4 (10.0.30.23:9000)
```

HAProxy config:
```
frontend minio_api
  bind *:9000
  default_backend minio_nodes

backend minio_nodes
  balance roundrobin
  option httpchk GET /minio/health/live
  server node1 10.0.30.20:9000 check
  server node2 10.0.30.21:9000 check
  server node3 10.0.30.22:9000 check
  server node4 10.0.30.23:9000 check
```

---

## Bucket Management

### Create & List Buckets

```bash
# Create bucket
mc mb minio/backups

# List buckets
mc ls minio

# Output:
# [2025-02-07 10:00:00 UTC]     0B backups/
# [2025-02-07 10:01:00 UTC]     0B databases/
# [2025-02-07 10:02:00 UTC]     0B logs/
```

### Upload & Download Files

```bash
# Upload file
mc cp /var/backups/db.sql minio/backups/db_2025-02-07.sql

# Upload directory
mc cp --recursive /var/log/ minio/logs/

# Download file
mc cp minio/backups/db_2025-02-07.sql /tmp/restore/

# List bucket contents
mc ls minio/backups --recursive

# Get file info
mc stat minio/backups/db_2025-02-07.sql
```

### Bucket Versioning

Keep multiple versions of objects:

```bash
# Enable versioning
mc version enable minio/backups

# Upload same file twice (creates new version)
mc cp /var/backups/db.sql minio/backups/db.sql
mc cp /var/backups/db_new.sql minio/backups/db.sql

# List all versions
mc ls minio/backups --versions

# Restore specific version
mc cp minio/backups/db.sql --version-id abc123 /tmp/restore/
```

---

## Access Control & Policies

### Service Accounts

```bash
# Create service account for application
mc admin user svcacct add minio admin app-backup-service

# Output:
# Access Key: AKIAIOSFODNN7EXAMPLE
# Secret Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# List service accounts
mc admin user svcacct list minio admin

# Revoke service account
mc admin user svcacct rm minio admin app-backup-service
```

### Bucket Policies

Restrict access to specific buckets/prefixes:

```bash
# Create policy file (policy.json)
cat > /tmp/policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::000000000000:user/app-backup-service"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::backups/*"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::000000000000:user/app-backup-service"
      },
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::backups"
    }
  ]
}
EOF

# Apply policy to bucket
mc policy set-json /tmp/policy.json minio/backups
```

### Public Read Access

```bash
# Allow anonymous read access to bucket
cat > /tmp/public-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::public/*"
    }
  ]
}
EOF

mc policy set-json /tmp/public-policy.json minio/public
```

---

## Replication

### Bucket-to-Bucket Replication

Replicate objects between MinIO instances (disaster recovery):

```bash
# Add remote MinIO server alias
mc alias set minio-dr http://10.0.30.30:9000 admin password

# Enable replication rule on source bucket
mc replicate rule add \
  --priority 1 \
  --source-bucket backups \
  --destination-bucket backups \
  --target-endpoint http://10.0.30.30:9000 \
  --target-access-key backup_user \
  --target-secret-key backup_password \
  minio

# Verify replication
mc replicate rule ls minio
```

### Continuous Replication to S3

```bash
# For backup to AWS S3
mc mirror --watch minio/backups/recent s3/my-bucket/backups/

# Or automated with replication policy pointing to S3
```

---

## Lifecycle & Tiering

### Object Lifecycle Rules

Automatically delete or transition old objects:

```bash
# Create lifecycle policy (lifecycle.json)
cat > /tmp/lifecycle.json <<'EOF'
{
  "Rules": [
    {
      "ID": "delete-old-backups",
      "Filter": {
        "Prefix": "backups/"
      },
      "Expiration": {
        "Days": 90
      },
      "Status": "Enabled"
    },
    {
      "ID": "archive-logs",
      "Filter": {
        "Prefix": "logs/"
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "GLACIER"
        }
      ],
      "Status": "Enabled"
    }
  ]
}
EOF

# Apply lifecycle policy
mc ilm import minio/backups < /tmp/lifecycle.json

# View lifecycle configuration
mc ilm rule ls minio/backups
```

---

## Monitoring & Metrics

### Health & Status

```bash
# Check cluster health
mc admin health info minio

# Check drive status
mc admin disk info minio

# Get usage statistics
mc du minio/backups

# Output:
# 256GB backups/db_2025-02/
# 128GB backups/db_2025-01/
# ...
```

### Prometheus Metrics

```bash
# MinIO exports metrics at /minio/v2/metrics/cluster

# Add to prometheus.yml
scrape_configs:
  - job_name: 'minio'
    static_configs:
      - targets: ['10.0.30.20:9000']
    metrics_path: '/minio/v2/metrics/cluster'
    basic_auth:
      username: 'admin'
      password: 'secure_password_123'
```

### Object Count & Size

```bash
# Count objects in bucket
mc find minio/backups --type f | wc -l

# Find large files
mc find minio/backups --type f --larger 1gb

# Find recent uploads (last 24 hours)
mc find minio/backups --newer 24h
```

---

## Backup & Disaster Recovery

### Export Configuration

```bash
# Backup MinIO configuration
mc admin config get minio > /tmp/minio-config.json

# Backup IAM policies
mc admin policy list minio | xargs -I {} mc admin policy export minio {} > /tmp/policies.json

# Backup user accounts
mc admin user list minio > /tmp/users.json
```

### Full Data Export to External Storage

```bash
# Mirror entire bucket to NFS/backup server
mc mirror --recursive --watch minio/backups /backup/nfs-mount/backups/

# Mirror to another MinIO instance
mc mirror --recursive minio/backups minio-dr/backups-restore/
```

### Point-in-Time Recovery

```bash
# List object versions (if versioning enabled)
mc ls minio/backups --versions --recursive

# Restore specific version
mc cp minio/backups/file.txt --version-id abc123 /restore/

# Restore entire bucket to point-in-time
# Use object metadata timestamps to identify backup point
```

---

## Troubleshooting

### High Disk Usage

```bash
# Identify largest buckets
mc du minio --recursive

# Find large objects
mc find minio --larger 1gb --recursive

# Check object count per bucket
for bucket in $(mc ls minio | awk '{print $NF}'); do
  count=$(mc find minio/$bucket --type f | wc -l)
  echo "$bucket: $count objects"
done

# Solutions:
# 1. Enable lifecycle rules to delete old objects
# 2. Archive cold data to cheaper storage
# 3. Add more drives to cluster
```

### Replication Failures

```bash
# Check replication status
mc replicate status minio/backups

# View replication metrics
curl -s http://10.0.30.20:9000/minio/v2/metrics/cluster | grep replicate

# Troubleshoot:
# 1. Verify network connectivity to destination
# 2. Check service account permissions
# 3. Review replication rules with: mc replicate rule ls minio
```

### Slow Performance

```bash
# Check cluster latency
mc admin health info minio

# Identify slow nodes
mc admin disk info minio | grep -E "latency|slow"

# Check network between cluster nodes
# From node1 to node2:
ping -c 5 10.0.30.21

# Monitor CPU/memory on all nodes
ssh 10.0.30.20 'top -b -n 1 | head -n 12'

# Solutions:
# 1. Replace slow disk
# 2. Upgrade network hardware
# 3. Distribute load with load balancer
```

### Authorization Failures

```bash
# Check MinIO logs
sudo tail -f /var/log/minio.log | grep -i "denied\|forbidden"

# Verify service account
mc admin user svcacct info minio admin app-backup-service

# Test S3 CLI with credentials
AWS_ACCESS_KEY_ID=AKIA... AWS_SECRET_ACCESS_KEY=... \
  aws s3 ls --endpoint-url http://10.0.30.20:9000
```

---

## Best Practices

1. **Erasure Coding**: Use distributed cluster for redundancy; minimum 4 nodes recommended
2. **Replication**: Enable bucket replication for disaster recovery to second MinIO cluster or S3
3. **Policies**: Create service accounts with least-privilege policies; never share root credentials
4. **Monitoring**: Export metrics to Prometheus; alert on disk usage, replication lag, errors
5. **Versioning**: Enable for critical buckets to prevent accidental deletions
6. **Lifecycle**: Implement lifecycle rules to archive/delete old objects automatically
7. **Backups**: Regular backups of IAM configuration and critical bucket policies
8. **Consistency**: Use versioning + replication for strong consistency guarantees
9. **Encryption**: Use TLS for transport; enable object encryption for sensitive data
10. **Load Balancing**: Use HAProxy/load balancer in front of cluster for high availability

---

*MinIO S3-compatible object storage guide for backup, archival, and data protection.*
