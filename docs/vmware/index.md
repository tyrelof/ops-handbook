# VMware Infrastructure

Virtual infrastructure management using VMware vSphere, ESXi, and vCenter Server. Complete lifecycle management for virtual machines and cluster operations.

## 📚 Core Topics

- [vSphere & ESXi Operations](cookbooks/vsphere-esxi.md) - ESXi hosts, vCenter, VMs, storage, clustering, disaster recovery

## Quick Links

| Topic | Use Case |
|-------|----------|
| ESXi Setup | Host deployment and configuration |
| vCenter Management | Multi-host cluster management |
| VM Lifecycle | Create, clone, migrate, snapshot VMs |
| vMotion | Live VM migration |
| Resource Pools | CPU/memory management |
| Storage Configuration | Datastore setup (NFS, iSCSI, VMFS) |
| Networking | Virtual switches, port groups, VLANs |
| High Availability | HA and FT for VMs |
| Disaster Recovery | Replication, failover procedures |
| Performance Tuning | Monitoring and optimization |

## Cookbooks

### 🖥️ vSphere & ESXi Operations
**[Full Cookbook](cookbooks/vsphere-esxi.md)**

Complete guide to managing virtual infrastructure at scale (50-500+ VMs):
- ESXi host deployment and initialization
- vSphere cluster configuration with HA and DRS
- VM lifecycle management (create, power, snapshot, migrate)
- vMotion for live migration
- Resource pools and performance optimization
- Datastore management (NFS, iSCSI, VMFS)
- VM templates and linked clones
- Virtual networking (vSwitches, port groups, VLANs)
- Replication and disaster recovery
- Monitoring and metrics analysis
- Troubleshooting common issues
- PowerCLI automation examples

**Contains**: 150+ PowerCLI commands, best practices, production patterns

---

## 🎯 Use Cases

- Building private cloud infrastructure
- Managing physical-to-virtual migrations
- Setting up disaster recovery
- Automating VM provisioning
- Performance optimization
- Capacity planning
- High availability setup

## Related Documentation

- [Linux System Management](../linux/system.md)
- [Windows System Management](../windows/services.md)
- [Kubernetes Cluster Management](../kubernetes/index.md)
- [Docker Container Orchestration](../docker/index.md)
- [Ansible Infrastructure Automation](../docker/cookbooks/ansible-automation.md)

---

*VMware vSphere and ESXi infrastructure guide*
