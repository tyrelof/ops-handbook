# Cookbook: VMware vSphere & ESXi - Enterprise Virtualization

> [!NOTE]
> Enterprise virtualization management at scale.
> Covers: ESXi host management, vSphere cluster operations, VM lifecycle, storage, networking, and disaster recovery.

## 1. ESXi Host Fundamentals

### Initial ESXi Setup
```bash
# SSH into ESXi host
ssh root@192.168.1.50

# Check ESXi version
vmware -v

# Check system info
esxcli system info get

# List available networks
esxcli network nic list

# Configure management IP (DHCP)
esxcli network ip interface ipv4 set -i vmk0 -t dhcp

# Or static IP
esxcli network ip interface ipv4 set -i vmk0 -t static -I 192.168.1.50 -N 255.255.255.0 -g 192.168.1.1

# Configure DNS
esxcli network ip dns server add --server=8.8.8.8
esxcli network ip config set --hostname=esx01.example.com
```

### vSphere Client Connection
```bash
# Via vSphere Client (GUI):
# 1. Open vSphere Client
# 2. Enter ESXi host IP or vCenter
# 3. Authenticate with root or domain user

# Via PowerCLI (command line)
# PowerCLI installed on Windows
Connect-VIServer -Server 192.168.1.50 -User root -Password "password"

# Check connection
Get-VMHost
Get-Cluster
Get-Datastore
```

## 2. Host Management

### Host Operations
```powershell
# PowerCLI Commands

# Get all hosts in cluster
Get-Cluster "Cluster-01" | Get-VMHost

# Get host details
Get-VMHost "esx01.example.com" | Select-Object Name, Version, Build, MemoryGB, ProcessorCount

# Enable SSH on host (for troubleshooting)
Get-VMHost "esx01" | Get-VMHostService | Where {$_.Key -eq "TSM-SSH"} | Start-VMHostService -Confirm:$false

# Put host in maintenance mode (before updates/reboots)
Get-VMHost "esx01" | Set-VMHost -State Maintenance -Confirm:$false

# Exit maintenance mode
Get-VMHost "esx01" | Set-VMHost -State Connected -Confirm:$false

# Reboot host (safely, with VMs migrating)
Get-VMHost "esx01" | Restart-VMHost -Confirm:$false

# Check host health
Get-VMHost "esx01" | Get-VMHostHardware | Select-Object *

# Monitor CPU/Memory
Get-VMHost "esx01" | Select-Object @{N="CPU (%)";E={[math]::Round($_.CpuUsageMHz / $_.CpuTotalMHz * 100, 2)}}, @{N="Mem (%)";E={[math]::Round($_.MemoryUsageGB / $_.MemoryTotalGB * 100, 2)}}
```

### Host Networking
```powershell
# Get vSwitches
Get-VMHost "esx01" | Get-VirtualSwitch

# Get port groups (networks)
Get-VMHost "esx01" | Get-VirtualPortGroup

# Get physical NICs
Get-VMHost "esx01" | Get-VMHostNetworkAdapter | Format-Table Name, IP, SubnetMask, Speed

# Create new vSwitch (for VM network)
New-VirtualSwitch -VMHost "esx01" -Name "vSwitch1"

# Create port group (VM network)
New-VirtualPortGroup -VirtualSwitch "vSwitch1" -Name "VM-Network"

# Add physical NIC to vSwitch (for redundancy)
Get-VMHost "esx01" | Get-VirtualSwitch "vSwitch1" | Add-VirtualSwitchPhysicalNetworkAdapter -VMHostPhysicalNic (Get-VMHostNetworkAdapter -VMHost "esx01" -Name "vmnic1") -Confirm:$false
```

### Host Updates & Patches
```powershell
# Download ESXi patch from VMware
# Then upload to vCenter

# Check available updates
Get-VMHost "esx01" | Get-Baseline -Baseline (Get-Baseline -TargetType Host | Where {$_.Name -match "Critical"})

# Apply patches (requires maintenance mode)
Get-VMHost "esx01" | Set-VMHost -State Maintenance
Get-VMHost "esx01" | Update-VMHostProfile
Get-VMHost "esx01" | Set-VMHost -State Connected

# Or via CLI using esxcli
# esxcli software vib update -d /vmfs/volumes/datastore1/patch.zip
```

## 3. Virtual Machine Lifecycle

### Create VM
```powershell
# New VM from template
New-VM -Name "WebServer-01" -Template "Linux-Template" `
    -ResourcePool "Production" -VMHost "esx01"

# Create VM from scratch
New-VM -Name "WebServer-02" -VMHost "esx01" `
    -MemoryGB 8 -NumCpu 4 -DiskGB 50 `
    -NetworkName "VM-Network" -GuestId "rhel7_64Guest"

# Configure VM hardware
Get-VM "WebServer-01" | Set-VM -MemoryGB 16 -NumCpu 8 -Confirm:$false

# Add disk to VM
New-HardDisk -VM "WebServer-01" -CapacityGB 100 -Datastore "DS-01"

# Add network adapter to VM
New-NetworkAdapter -VM "WebServer-01" -NetworkName "VM-Network" -Confirm:$false
```

### Power Management
```powershell
# Power on
Get-VM "WebServer-01" | Start-VM -Confirm:$false

# Wait for VM to be ready
Get-VM "WebServer-01" | Wait-Tools

# Power off gracefully
Get-VM "WebServer-01" | Shutdown-Guest -Confirm:$false

# Force power off (if hung)
Get-VM "WebServer-01" | Stop-VM -Confirm:$false

# Reboot gracefully
Get-VM "WebServer-01" | Restart-Guest -Confirm:$false

# Get VM status
Get-VM "WebServer-01" | Select-Object Name, PowerState, CpuUsageMhz, MemoryUsageMB
```

### VM Snapshots
```powershell
# Create snapshot before changes
Get-VM "WebServer-01" | New-Snapshot -Name "Before-Update" -Description "Pre-update snapshot"

# List snapshots
Get-VM "WebServer-01" | Get-Snapshot

# Revert to snapshot
Get-VM "WebServer-01" | Get-Snapshot -Name "Before-Update" | Set-Snapshot -SnapshotConfiguration -PowerOn:$false | Restore-Snapshot -Confirm:$false

# Remove snapshot (consolidate disk)
Get-VM "WebServer-01" | Get-Snapshot | Remove-Snapshot -Confirm:$false

# Remove all snapshots
Get-VM "WebServer-01" | Get-Snapshot | Remove-Snapshot -RemoveChildren -Confirm:$false

# Check VM disk consolidation
Get-VM "WebServer-01" | Get-View | Select-Object @{N="ConsolidationNeeded";E={$_.Runtime.ConsolidationNeeded}}
```

### VM Migration (vMotion)
```powershell
# Migrate VM to different host (zero downtime)
Get-VM "WebServer-01" | Move-VM -Destination (Get-VMHost "esx02")

# Migrate with resource pool
Get-VM "WebServer-01" | Move-VM -Destination (Get-ResourcePool "Production")

# Migrate storage (Storage vMotion)
Get-VM "WebServer-01" | Move-VM -Datastore (Get-Datastore "DS-SSD")

# Check VM for vMotion compatibility
Get-VM "WebServer-01" | Select-Object Name, Version, PowerState
```

### Delete VM
```powershell
# Power off first
Get-VM "WebServer-01" | Stop-VM -Confirm:$false

# Remove VM
Get-VM "WebServer-01" | Remove-VM -Confirm:$false -DeletePermanently

# Note: Disks may still exist if not deleted with VM
Get-Datastore "DS-01" | Find-DatastoreFile -Name "WebServer-01*" | Remove-DatastoreFile -Confirm:$false
```

## 4. Resource Management

### vSphere Cluster Setup
```powershell
# Create cluster
New-Cluster -Name "Production" -Location (Get-Datacenter "DataCenter-01") `
    -DrsEnabled -VsanEnabled -Ha Enabled

# Add hosts to cluster
Get-VMHost "esx01" | Move-VMHost -Destination (Get-Cluster "Production")
Get-VMHost "esx02" | Move-VMHost -Destination (Get-Cluster "Production")

# Configure cluster resources
Get-Cluster "Production" | Set-Cluster -DrsAutomationLevel FullyAutomated

# Create resource pool
New-ResourcePool -Name "Production" -Location (Get-Cluster "Production") `
    -MemReservationMB 0 -MemLimitMB 102400 -CpuReservationMhz 0
```

### CPU & Memory Reservation
```powershell
# Set VM resource limits
Get-VM "WebServer-01" | Set-VM -MemoryGB 8 -NumCpu 4 -Confirm:$false

# Set resource reservation
Get-VM "WebServer-01" | New-ResourcePool -MemReservationMB 4096 -CpuReservationMhz 2000

# Get current utilization
Get-VMHost "esx01" | Get-VMHostResource | Select-Object MemoryGB, MemoryUsageMB

# Monitor resource contention
Get-VMHost "esx01" | Select-Object @{N="CPU (%)";E={[math]::Round($_.CpuUsageMHz / $_.CpuTotalMHz * 100, 2)}}, @{N="Mem (%)";E={[math]::Round($_.MemoryUsageGB / $_.MemoryTotalGB * 100, 2)}}
```

## 5. Storage Management

### Datastores
```powershell
# List datastores
Get-Datastore | Select-Object Name, CapacityGB, FreeSpaceGB, @{N="% Used";E={[math]::Round((($_.CapacityGB - $_.FreeSpaceGB) / $_.CapacityGB * 100), 2)}}

# Get datastore details
Get-Datastore "DS-01" | Select-Object *

# Add NFS datastore
New-Datastore -VMHost "esx01" -Name "NFS-Storage" -NfsHost "192.168.1.100" -NfsPath "/mnt/storage"

# Add VMFS datastore (iSCSI)
Get-VMHostDisk -VMHost "esx01" | Where {$_.Size -gt 100GB} | New-Datastore -Name "VMFS-01"

# Monitor datastore space
Get-Datastore | Where {$_.FreeSpaceGB / $_.CapacityGB -lt 0.1} | Select-Object Name, FreeSpaceGB, CapacityGB

# Get disk provisioning model
Get-VM "WebServer-01" | Get-HardDisk | Select-Object Name, CapacityGB, StorageFormat
```

### VM Templates & Cloning
```powershell
# Create template from VM
Get-VM "WebServer-Golden" | New-Template -Name "Linux-Web-Template" -Location (Get-Folder "Templates")

# Clone from template
New-VM -Name "WebServer-03" -Template "Linux-Web-Template" `
    -ResourcePool "Production" -Datastore "DS-01"

# Clone VM (full copy)
New-VM -Name "WebServer-Clone" -VM "WebServer-01" `
    -ResourcePool "Production" -Datastore "DS-01"

# Linked clone (uses snapshot, saves space)
# Not directly supported in PowerCLI, use vSphere Client
```

## 6. Networking

### Virtual Network Configuration
```powershell
# Create vSwitch with failover
New-VirtualSwitch -VMHost "esx01" -Name "Production-vSwitch" -NumPorts 2048

# Add physical NICs (bonding/teaming)
Get-VMHost "esx01" | Get-VirtualSwitch "Production-vSwitch" | 
    Add-VirtualSwitchPhysicalNetworkAdapter -VMHostPhysicalNic @(
        Get-VMHostNetworkAdapter -VMHost "esx01" -Name "vmnic0",
        Get-VMHostNetworkAdapter -VMHost "esx01" -Name "vmnic1"
    ) -Confirm:$false

# Create distributed switch (for multi-host setup)
New-VDSwitch -Name "DVS-Production" -Location (Get-Datacenter "DataCenter-01")

# Create port group on distributed switch
New-VDPortgroup -VDSwitch "DVS-Production" -Name "VM-Network"

# Configure VLAN on port group
Get-VDPortgroup "VM-Network" | Set-VDVlanConfiguration -VlanId 100 -Confirm:$false
```

### VM Network Configuration
```powershell
# Get VM network adapters
Get-VM "WebServer-01" | Get-NetworkAdapter

# Connect/disconnect network adapter
Get-VM "WebServer-01" | Get-NetworkAdapter | Connect-NetworkAdapter -Confirm:$false
Get-VM "WebServer-01" | Get-NetworkAdapter | Disconnect-NetworkAdapter -Confirm:$false

# Change network
Get-VM "WebServer-01" | Get-NetworkAdapter | Set-NetworkAdapter -NetworkName "DMZ-Network" -Confirm:$false

# Add secondary network (multi-homed)
New-NetworkAdapter -VM "WebServer-01" -NetworkName "Internal-Network" -Confirm:$false
```

## 7. Backup & Disaster Recovery

### VM Backups
```powershell
# Native vSphere backup (snapshots)
Get-VM "WebServer-01" | New-Snapshot -Name "Daily-Backup-$(Get-Date -f 'yyyy-MM-dd')" `
    -Description "Automated daily backup"

# Retention: delete old snapshots
Get-VM "WebServer-01" | Get-Snapshot | Where {$_.Created -lt (Get-Date).AddDays(-7)} | Remove-Snapshot -Confirm:$false

# Third-party backup (Veeam, CommVault, etc.)
# Configure agent in VM and connect to backup server
```

### VM Replication
```powershell
# VMware Site Recovery Manager (SRM) replication
# Configure replication policy in vCenter

# vSphere Replication
Get-VM "WebServer-01" | New-Replication -ReplicationServer "Replica-Host" `
    -Datastore (Get-Datastore "DS-Replica")

# Monitor replication
Get-VM "WebServer-01" | Get-Replication | Select-Object VM, State, RpoAchievements
```

### Disaster Recovery Procedure
```powershell
# 1. Detect failure
Get-VMHost "esx01" | Select-Object Name, State, ConnectionState

# 2. If host is down, trigger failover
Get-Cluster "Production" | Set-Cluster -HAEnabled:$true -HAAdmissionControl:$true

# HA restarts VMs on other hosts automatically

# 3. If site down, fail over using SRM
# In vCenter: Site Recovery > Failover

# 4. Verify VM functionality
Get-VM -Location (Get-Cluster "Production") | Select-Object Name, PowerState, VMHost
```

## 8. Monitoring & Performance

### Performance Monitoring
```powershell
# Get host performance stats
Get-VMHost "esx01" | Get-Stat -Stat "cpu.usage.average", "mem.usage.average" `
    -Start (Get-Date).AddDays(-7) -Interval 300 | Export-CSV "perf-report.csv"

# Get VM performance
Get-VM "WebServer-01" | Get-Stat -Stat "cpu.usagemhz.average", "mem.usagemb.average" `
    -Start (Get-Date).AddHours(-1) -Interval 20

# Real-time CPU usage
Get-VMHost "esx01" | Get-Stat -Stat "cpu.usagemhz.latest" -Start (Get-Date).AddMinutes(-10) -Interval 20

# Storage performance
Get-Datastore "DS-01" | Get-Stat -Stat "datastore.read.average", "datastore.write.average" `
    -Start (Get-Date).AddHours(-1) -Interval 300
```

### VM Event Monitoring
```powershell
# Get recent VM events
Get-VM "WebServer-01" | Get-VIEvent | Select-Object CreatedTime, EventType, FullFormattedMessage | Sort-Object CreatedTime -Descending | Select-Object -First 10

# Get VM task history
Get-VM "WebServer-01" | Get-Task | Select-Object Name, State, Result, StartTime, FinishTime

# Alert on VM CPU/Memory
Get-VM | Where {(Get-Stat -Entity $_ -Stat "cpu.usagemhz.average" -Realtime).Value -gt 80000} | Select-Object Name
```

## 9. vCenter Management

### vCenter Basics
```powershell
# Connect to vCenter
Connect-VIServer -Server vcenter.example.com -User administrator@example.com -Password "password"

# Get inventory
Get-Datacenter | Get-Cluster | Get-VMHost
Get-VM | Select-Object Name, PowerState, VMHost, MemoryGB, NumCpu

# Create folder structure
New-Folder -Name "Production" -Location (Get-Datacenter "DataCenter-01")

# Organize VMs by folder
Get-VM "WebServer-*" | Move-VM -Destination (Get-Folder "Production")

# Set VM tags
Get-VM "WebServer-01" | New-Tag -Name "Production" -Category "Environment"
Get-VM "WebServer-01" | New-Tag -Name "High-Availability" -Category "Requirements"
```

## 10. Troubleshooting

### Host Issues
```powershell
# Host not responding
Get-VMHost "esx01" | Select-Object Name, State, ConnectionState

# Restart management agents
Get-VMHost "esx01" | Restart-VMHostService -ServiceName pcscd -Confirm:$false

# Host storage issues
Get-VMHost "esx01" | Get-HBA | Select-Object Name, Device, Status

# Check datastore accessibility
Get-VMHost "esx01" | Get-Datastore | Select-Object Name, State, Accessibility
```

### VM Issues
```powershell
# VM won't start
Get-VM "WebServer-01" | Select-Object Name, PowerState, HostConnectionState

# Check compatibility
Get-VM "WebServer-01" | Select-Object @{N="Version";E={$_.Version}}, VMHost

# VM tools issue
Get-VM "WebServer-01" | Get-Guest | Select-Object ToolsRunningStatus, ToolsVersion

# Disk consolidation needed
Get-VM "WebServer-01" | Get-View | Select-Object @{N="ConsolidationNeeded";E={$_.Runtime.ConsolidationNeeded}}
```

### Network Issues
```powershell
# Port group not accessible
Get-PortGroup "VM-Network" | Select-Object Name, State

# VLAN tag verification
Get-VDPortgroup "VM-Network" | Get-VDVlanConfiguration

# Physical NIC status
Get-VMHost "esx01" | Get-VMHostNetworkAdapter | Select-Object Name, Mac, IP, Speed, Connected
```

## 11. Best Practices

| Practice | Implementation | Benefit |
|----------|----------------|---------|
| Snapshots | Auto-delete after 7 days | Prevent disk space issues |
| Resource pools | Reserve memory per tier | Prevent starvation |
| vMotion | Distributed switches | Seamless migration |
| HA/FT | Enable per cluster | High availability |
| DRS | Automatic load balancing | Optimal performance |
| Storage | RAID 10 or SSD | Performance & resilience |
| Backups | Daily snapshots + replicas | Disaster recovery |
| Monitoring | Performance baselines | Detect issues early |

## 12. Common Commands Reference

```powershell
# Host
Get-VMHost | Get-VMHostService | Start-VMHostService
Set-VMHost -State Maintenance | Connected
Get-VMHostNetwork | Get-VMHostNetworkAdapter

# VMs
Get-VM | Start-VM | Stop-VM | Restart-VM
Get-VM | Move-VM -Destination <host/pool/datastore>
Get-VM | New-Snapshot | Get-Snapshot | Remove-Snapshot

# Storage
Get-Datastore | New-Datastore
Get-VM | Get-HardDisk | New-HardDisk

# Cluster
Get-Cluster | Set-Cluster -DrsAutomationLevel <level>
Get-Cluster | Get-ResourcePool
```

---

**This forms the foundation of enterprise virtualization infrastructure at scale.**
