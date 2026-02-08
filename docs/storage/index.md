# Storage Documentation

Storage systems, data protection, replication, and backup strategies for enterprise environments.

## Overview

This section covers NAS/storage systems, RAID strategies, backup and disaster recovery, and data protection methodologies.

## Cookbooks

### [FreeNAS/TrueNAS](cookbooks/freenas-truenas.md)
Enterprise NAS platform with ZFS filesystem. Covers pool creation, RAID-Z strategies, NFS/iSCSI exports, snapshots, replication, and backup automation.

**Topics**:
- ZFS fundamentals and RAID-Z levels (Z1, Z2, Z3)
- Pool and dataset management
- NFS and iSCSI configuration
- Snapshot scheduling and retention
- Replication to remote NAS
- Backup strategies (rsync, snapshots)
- Disk health monitoring and SMART testing

**Use Cases**:
- Backup destination for virtualization environments
- NFS datastore for Kubernetes persistent volumes
- iSCSI target for block storage applications
- Disaster recovery replication

---

## Related Sections

- [Linux Storage](../linux/) - LVM, mount points, filesystem operations
- [Kubernetes Storage](../kubernetes/storage.md) - Persistent volumes, storage classes
- [Databases](../databases/) - Database backup and replication

---

*Last updated: 2024 | Community contributions welcome*
