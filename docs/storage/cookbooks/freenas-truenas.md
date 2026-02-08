# FreeNAS/TrueNAS - Storage & NAS Management

Complete guide to deploying, managing, and monitoring FreeNAS/TrueNAS for enterprise storage, backups, and disaster recovery.

## Table of Contents
1. [Fundamentals](#fundamentals)
2. [Installation & Setup](#installation--setup)
3. [ZFS & RAID](#zfs--raid)
4. [Storage Pools](#storage-pools)
5. [NFS Sharing](#nfs-sharing)
6. [iSCSI Targets](#iscsi-targets)
7. [Snapshots & Replication](#snapshots--replication)
8. [Backups](#backups)
9. [Monitoring & Alerts](#monitoring--alerts)
10. [Troubleshooting](#troubleshooting)

---

## Fundamentals

FreeNAS (now TrueNAS Community Edition) is an open-source NAS built on FreeBSD with ZFS, providing advanced storage features like snapshots, replication, and high availability.

**Key Features**:
- ZFS (Copy-on-Write, snapshots, checksums)
- RAID-Z (single/double/triple parity, like RAID 5/6)
- NFS, CIFS/SMB, iSCSI, S3
- Block-level replication
- Continuous data protection (CDP)
- Web-based management

**Hardware Minimum**:
- 8 GB RAM (16+ recommended for 10+ TB pools)
- 1+ dedicted HDD per vdev (data protection)
- ECC memory (highly recommended)
- Gigabit+ network

---

## Installation & Setup

### Download & Install

```bash
# Download TrueNAS Community (free)
curl -O https://download.truenas.com/TrueNAS-CORE-13.0-U6.2-x64.iso

# Create VM: 4+ vCPU, 16+ GB RAM, 20 GB root, dedicated storage drives
# Attach ISO and boot → Install to first disk
```

### Initial Configuration

1. **Network Setup**: Assign IP to data network (e.g., 10.0.10.20/24)
2. **Create Storage Pool**: Identify available disks
3. **Hostname/Domain**: Set FQDN
4. **NTP**: Configure time sync
5. **Alerts**: Set email for notifications

Access WebUI at `https://10.0.10.20`

---

## ZFS & RAID

### RAID-Z Levels

```
RAID-Z1: 1 parity drive (like RAID 5)
  Min: 3 drives, Max: 8 recommended
  Capacity: (N-1) * drive_size
  Rebuild time: high (use in labs/non-critical)

RAID-Z2: 2 parity drives (like RAID 6)
  Min: 4 drives, Max: 8 recommended
  Capacity: (N-2) * drive_size
  Rebuild time: medium (best for production)

RAID-Z3: 3 parity drives
  Min: 5 drives, Max: 8 recommended
  Capacity: (N-3) * drive_size
  Rebuild time: faster recovery, more overhead
```

### Vdev Recommendations

For storage pool with 10+ TB of usable capacity:

```
Single 8-disk RAID-Z2 pool:
  8 x 4TB drives → 24 TB usable (6 TB parity)
  
Multi-vdev (RAID-Z2 + RAID-Z2):
  2 x RAID-Z2 (4 drives each) → 24 TB usable
  Pros: Better parallelism; each vdev can rebuild separately
  Cons: More complex
```

---

## Storage Pools

### Create Pool via WebUI

**Storage → Pools → Add**:

1. **Name**: `tank` (conventional)
2. **Disk Selection**: Choose 4+ drives for RAID-Z2
3. **Encryption**: Enable (AES-256) for sensitive data
4. **Deduplication**: Disable (uses extra RAM/CPU for marginal gains)
5. **Review and Create**

### Create via CLI

```bash
# SSH to TrueNAS
ssh root@10.0.10.20

# Create RAID-Z2 pool (4 drives)
zpool create tank raidz2 /dev/da0 /dev/da1 /dev/da2 /dev/da3

# Check status
zpool status
zpool list

# View disk usage
df -h /tank
```

### Add Spare Drive

```bash
# Hot spare (automatically replaces failed drive)
zpool add tank spare /dev/da4

# Check
zpool status
```

---

## NFS Sharing

### Create NFS Share

**Storage → Pools → tank → Add Dataset**:

```
Name: backups
Comments: VM & server backups
Compression: LZ4
Recordsize: 128K (default for general use)
Sync: Disabled (unless NFSv4 is used)
Encryption: Inherit (from pool)
Quota: None (or set limit)
```

### Configure NFS Export

**Sharing → NFS**:

1. **Enable NFS**: checked
2. **Add Share**:
   - Dataset: `/tank/backups`
   - Maproot User: root
   - Maproot Group: wheel
   - Networks: `10.0.0.0/8` (allow internal only)

### Mount on Client

```bash
# Linux/Unix client
sudo mount -t nfs -o vers=4,rw 10.0.10.20:/tank/backups /mnt/backups

# Persistent mount (/etc/fstab)
echo "10.0.10.20:/tank/backups /mnt/backups nfs vers=4,rw,_netdev 0 0" | sudo tee -a /etc/fstab
sudo mount -a

# Test write
touch /mnt/backups/test.txt
```

---

## iSCSI Targets

### Create iSCSI Dataset & Target

**Storage → Pools → tank → Add Zvol** (block device):

```
Name: iscsi-disk-01
Size: 500 GB
Compression: LZ4
```

**Sharing → iSCSI**:

1. **Global Tunables**: Leave defaults
2. **Add Portal**:
   - Listen Address: 10.0.10.20 (NAS IP)
   - Port: 3260
3. **Add Target**:
   - Name: `iqn.2024-01.com.example:storage:iscsi-disk-01`
   - Target Alias: `Production Storage`
4. **Add LUN**:
   - Target: select created target
   - LUN ID: 0
   - Extent: select created zvol
   - Block Size: 512B (standard)

### Connect from Initiator (Linux)

```bash
# Install iSCSI initiator
sudo apt install open-iscsi

# Discovery
sudo iscsiadm -m discovery -t st -p 10.0.10.20:3260

# Login
sudo iscsiadm -m node -T iqn.2024-01.com.example:storage:iscsi-disk-01 \
  -p 10.0.10.20:3260 -l

# Check
lsblk  # should see new block device

# Format and mount
sudo mkfs.ext4 /dev/sdb
sudo mkdir /mnt/iscsi-storage
sudo mount /dev/sdb /mnt/iscsi-storage

# Make persistent (/etc/iscsid.conf)
# Set: node.startup = automatic
sudo systemctl restart iscsid
```

---

## Snapshots & Replication

### Create Snapshot

**Storage → Snapshots**:

```
Pool/Dataset: tank/backups
Naming Pattern: auto-%Y%m%d_%H%M%S
Hourly Snapshot: checked
Snapshot Lifetime: 30 days (auto-delete old)
```

Automatic snapshots start immediately.

### Manual Snapshot

```bash
ssh root@10.0.10.20
zfs snapshot tank/backups@manual-20240207
zfs list -t snapshot
```

### Replication to Remote NAS

**Storage → Replication Tasks**:

```
Source Dataset: tank/backups
Destination Dataset: backup-nas.example.com:/tank/replicas
Frequency: Daily at 2 AM
Snapshot Lifetime: 30 days
Encryption: enabled
```

Create API key on backup NAS for authentication.

Verify on remote:
```bash
ssh root@backup-nas.example.com
zfs list -t snapshot tank/replicas
```

---

## Backups

### Automated Backups to External USB

**Tasks → Rsync Tasks**:

```
Direction: Push to remote machine
Source Path: /mnt/tank/backups
Remote Host: 10.0.10.100 (backup server)
Remote Path: /backups/freenas
Frequency: Weekly (Sunday 2 AM)
Preserve Permissions: checked
Delete Files: unchecked
```

### Manual Backup

```bash
# On TrueNAS
rsync -avz --delete /mnt/tank/backups/ backup-server:/backups/freenas/

# Verify
ssh backup-server "du -sh /backups/freenas"
```

---

## Monitoring & Alerts

### Email Alerts

**System → Alert Services → Email**:

```
Mail Server: smtp.gmail.com
Port: 587
TLS: enable
Username: alerts@example.com
Password: app_password
From Address: nas-alerts@example.com
```

**System → Alerts**:

Check:
- Pool Status (failed disks)
- Replication Status
- SMART errors
- Email on failure

### Disk Health Monitoring

**Storage → Disks**:

- SMART tests: Enable short weekly, long monthly
- Monitor: S.M.A.R.T. attributes for wear/errors
- Set alerts: On SMART failures

### Prometheus Integration

**System → Advanced → Reporting**:

- Enable reporting for metrics export
- Scrape endpoint: `http://10.0.10.20:8080/metrics` (if Prometheus agent installed)

---

## Troubleshooting

### Disk Replacement

```bash
# Identify failed disk (zpool status shows DEGRADED)
zpool status

# Offline disk
zpool offline tank /dev/da0

# Physically replace
# (After replacement boots with new disk)

# Online new disk
zpool online tank /dev/da0

# Resilver (rebuild parity)
watch zpool status tank
```

### Snapshot Space Issues

```bash
# High snapshot usage?
zfs list -t snapshot

# Delete old snapshots (or extend lifetime in WebUI)
zfs destroy tank/backups@old-snapshot

# Check dataset size with snapshots
zfs get usedbysnapshots tank/backups
```

### Replication Failures

```bash
# Check replication task status in WebUI
# Logs: System → Event → Alert

# Manual trigger (if needed)
# WebUI: Storage → Replication → Run Now
```

---

## Best Practices

1. **Hardware**: Use ECC RAM + 3.5" drives for NAS (not desktop drives)
2. **Capacity**: Plan for 70% full; leave headroom for snapshots
3. **Redundancy**: Never use RAID-Z1; use RAID-Z2+ for 4+ TB drives
4. **Snapshots**: Enable hourly/daily for protection; set lifetime
5. **Replication**: Backup to remote site; test recovery
6. **Monitoring**: Enable SMART tests; set email alerts
7. **Updates**: Apply patches during maintenance window; test first
8. **Documentation**: Record pool config, vdev layout, IP addresses

---

*FreeNAS/TrueNAS storage guide for backups, replication, and disaster recovery.*
