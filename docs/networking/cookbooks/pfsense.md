# pfSense - Open-Source Firewall & Router

Complete guide to deploying and managing pfSense for network security, routing, VPN, and advanced networking on bare metal or virtual machines.

## Table of Contents
1. [Fundamentals](#fundamentals)
2. [Installation](#installation)
3. [Basic Network Setup](#basic-network-setup)
4. [Firewall Rules](#firewall-rules)
5. [NAT Configuration](#nat-configuration)
6. [DHCP & DNS](#dhcp--dns)
7. [VPN (IPSec & OpenVPN)](#vpn-ipsec--openvpn)
8. [High Availability (CARP)](#high-availability-carp)
9. [Advanced Routing](#advanced-routing)
10. [Monitoring & Logging](#monitoring--logging)
11. [Troubleshooting](#troubleshooting)

---

## Fundamentals

pfSense is a stateful firewall and router based on FreeBSD. It's cost-effective for SMBs and ideal for labs.

**Key Features**:
- Multi-WAN failover
- Layer 7 filtering (DPI)
- VPN termination (IPSec, OpenVPN, WireGuard)
- High availability with CARP
- Captive portal & bandwidth management
- Transparent bridging
- IPv6 support

**Minimum Requirements**:
- 512 MB RAM (2 GB+ recommended)
- 4 GB disk space
- 2+ network interfaces (for LAN + WAN separation)

---

## Installation

### Download & Boot

```bash
# Download pfSense ISO
curl -O https://mirror.example.com/pfsense/CE_2.7.0-RELEASE-amd64.iso

# Create VM with 2+ vNICs, 2+ GB RAM, 20 GB disk
# Attach ISO and boot
```

### Setup Wizard

During first boot:

```
1. Accept license
2. Partition disk → Auto (UFS) → Finish
3. Install → Reboot
4. Configure interfaces:
   WAN = em0 (DHCP or static to ISP)
   LAN = em1 (10.0.10.0/24)
5. Set hostname/domain
6. Configure WAN (DHCP or static)
7. Configure LAN (10.0.10.1/24)
8. Set admin password
9. Reboot
```

After reboot, access WebUI at `https://10.0.10.1` (LAN IP).

---

## Basic Network Setup

### Interface Configuration

Via WebUI: **Interfaces → Assignments**

```
WAN:  em0  (DHCP from ISP, or static public IP)
LAN:  em1  (10.0.10.0/24, gateway 10.0.10.1)
OPT1: em2  (10.0.50.0/24, for DMZ)
OPT2: em3  (10.0.99.0/24, for guest network)
```

### Static IP Example (WAN)

**Interfaces → WAN**:
- IPv4 Configuration Type: Static IPv4
- IPv4 Address: 203.0.113.10 / 29
- IPv4 Gateway: 203.0.113.9
- IPv4 Upstream Gateway: 203.0.113.9
- DNS Servers: 8.8.8.8, 1.1.1.1

### LAN Interface (Internal)

**Interfaces → LAN**:
- IPv4 Configuration Type: Static IPv4
- IPv4 Address: 10.0.10.1 / 24
- IPv6: disable (unless needed)

---

## Firewall Rules

### Default Policy

**Firewall → Rules → WAN tab**:
- Default: Block all inbound (implicit deny)
- Outbound: Allow all (from LAN to WAN)

### Add Allow Rules

Example: Allow SSH from external to internal:

**Firewall → Rules → WAN → Add**:
```
Action: Pass
Interface: WAN
Direction: in
TCP/IP Version: IPv4
Protocol: TCP
Source: any
Destination: LAN address (10.0.10.0/24)
Destination Port: 22
```

### Anti-Spoofing

**Firewall → Rules → WAN → Add**:
```
Action: Drop
Interface: WAN
Direction: in
Source: 10.0.0.0/8   (Block RFC1918 from WAN)
Protocol: any
Log: checked
```

### VLAN Isolation

Create rules to block east-west traffic between sensitive VLANs:

```
Action: Drop
Interface: LAN
Direction: in/out
Source: 10.0.40.0/24 (VoIP VLAN)
Destination: 10.0.20.0/24 (User VLAN)
Log: checked
```

---

## NAT Configuration

### Dynamic NAT (Outbound)

**Firewall → NAT → Outbound tab**:

Set to: Hybrid Outbound NAT mode

Add rule:
```
Interface: WAN
Source: 10.0.10.0/24
Address: Interface Address (WAN IP)
Description: Outbound NAT for LAN
```

### Static NAT (Inbound)

Forward external IP to internal server:

**Firewall → NAT → Port Forward**:
```
Interface: WAN
Protocol: TCP
External Port: 443
NAT IP: 10.0.50.10 (DMZ web server)
NAT Port: 443
Description: HTTPS to web server
```

### 1:1 NAT

Map entire internal subnet to external range:

**Firewall → NAT → 1:1 tab**:
```
Interface: WAN
External Subnet: 198.51.100.0/24
Internal Subnet: 10.0.50.0/24
Destination: Any
```

---

## DHCP & DNS

### DHCP Server

**Services → DHCP Server → LAN tab**:
```
Enable DHCP: checked
Range: 10.0.10.50 to 10.0.10.254
Gateway: 10.0.10.1
DNS Servers: 10.0.10.1, 8.8.8.8
Domain Name: example.com
NTP Servers: time.example.com
```

### DNS Resolver

**Services → DNS Resolver**:
```
Enable: checked
Listen Port: 53
Interfaces: LAN
Outgoing Port: random
Enable DNSSEC: checked (optional)
Register DHCP leases: checked
```

### Static DNS Entries

**Services → DNS Resolver → Host Overrides**:
```
Host: web
Domain: example.com
IP: 10.0.50.10
Description: Internal web server
```

---

## VPN (IPSec & OpenVPN)

### IPSec Site-to-Site

**VPN → IPSec → Tunnels**:

Phase 1 (IKE):
```
Disabled: unchecked
Key Exchange Version: IKEv2
Internet Protocol: IPv4
Interface: WAN
Remote Gateway: 203.0.113.5 (peer public IP)
```

Phase 2 (ESP):
```
Protocol: ESP
Encryption: AES-256
Integrity: SHA256
PFS Key Group: 14
```

Local/Remote Networks:
```
Local: 10.0.0.0/8
Remote: 10.1.0.0/8 (peer internal network)
```

### OpenVPN Server (Remote Access)

**VPN → OpenVPN → Servers**:

```
Protocol: UDP
Port: 1194
Cipher: AES-256-GCM
Compression: LZ4-V2
HMAC: SHA256
```

Generate certificates:
**System → Cert Manager → CA → Create self-signed CA**

Create certificate:
**System → Cert Manager → Certificates → Create**

Configure client network:
```
Tunnel Network: 10.8.0.0/24
Redirect Gateway: checked
```

### OpenVPN Client

**VPN → OpenVPN → Clients**:

```
Disabled: unchecked
Protocol: UDP
Address: vpn.example.com:1194
Auth: Username/Password + Certs
```

---

## High Availability (CARP)

Deploy two pfSense instances with virtual IP (VIP) failover.

### Primary Node

**System → High Avail. Sync**:
```
Synchronize States: checked
Synchronize Peer IP: 10.0.99.2 (secondary pfSense IP)
Username: admin
Password: sync_password
Sync Config: check all
```

### Configure CARP VIP

**Interfaces → Virtual IPs**:
```
Type: CARP
Interface: LAN
Address: 10.0.10.254 / 24
Virtual Password: carp_secret
VHID: 1
Advertising Frequency: 1 second
```

Clients use 10.0.10.254 as gateway; traffic auto-fails over if primary dies.

### Monitoring

**Status → Monitoring**:
- View sync status and packet loss
- Check CARP status: green = active, red = passive

---

## Advanced Routing

### Static Routes

**System → Routing → Static Routes**:

```
Destination: 10.1.0.0/8 (remote network)
Gateway: 10.0.10.254 (next-hop)
Description: Route to branch office
```

### Dynamic Routing (OSPF)

**System → Routing → Routing Protocol Settings** (requires routing package):

Install via **System → Package Manager**.

Enable OSPF:
```
Area: 0.0.0.0
Router ID: 10.0.99.1
```

Add interface:
```
Interface: LAN
Area: 0.0.0.0
Network: 10.0.10.0/24
```

---

## Monitoring & Logging

### Real-time Stats

**Status → Traffic Graph**: Monitor bandwidth by interface

### Firewall Logs

**Status → System Logs → Firewall tab**:
- View blocked/allowed connections
- Set to CSV export for analysis

### Syslog Forwarding

**System → Logging → Remote**:
```
Enable Remote Logging: checked
IP: 10.0.30.10 (syslog server)
Port: 514
Protocol: UDP
Facility: Local 5
```

### Backup & Restore

**Diagnostics → Backup & Restore**:
- Backup: Download all configs (encrypted)
- Restore: Upload backup after hardware failure

---

## VPN Advanced: Multi-Site Mesh

### Site-to-Site VPN Mesh (3 Sites)

Create secure encrypted tunnels between HQ, Remote Office 1, and Remote Office 2:

```
HQ (10.0.10.0/24)
  |
  +---IPSec---+
  |           |
  v           v
Remote-1    Remote-2
(10.0.20/24) (10.0.30/24)
  |___________| (direct IPSec tunnel)
```

**HQ to Remote-1 IPSec**:
- IKE Phase 1: 10.0.10.1 ← → 203.0.113.20
- IKE Phase 2: Encrypt traffic 10.0.10.0/24 ← → 10.0.20.0/24
- Each site must create TWO tunnels for full mesh (HQ↔R1, HQ↔R2, R1↔R2)

**Advanced Setup**:
```
VPN → IPSec Tunnels → Add Tunnel
  Phase 1:
    - My identifier: HQ_Site1
    - Peer identifier: Remote_Office_1
    - DPD Enable: Yes
    - Lifetime: 28800s (aggressive rekey prevents old connections)
  
  Phase 2:
    - Local subnet: 10.0.10.0/24
    - Remote subnet: 10.0.20.0/24
    - Protocol: ESP
    - Encryption: AES-256
    - Hash: SHA512
    - PFS Group: 14
```

Add firewall rules for VPN traffic:
```
Rules → IPsec → Add
  Action: Pass
  Protocol: Any
  Source: 10.0.10.0/24
  Destination: 10.0.20.0/24
  Gateway: Default
```

---

## Dynamic DNS & Multi-WAN

### DDNS for Sites with Dynamic IPs

When remote office has ISP-assigned (not static) public IP:

```
Services → Dynamic DNS
  Service: No-IP / Freedns / Cloudflare
  Interface: WAN (Remote office uplink)
  Hostname: remoteofficevpn.no-ip.com
  TTL: 300s
```

Then in other sites' IPSec config, replace static IP with hostname.

### Multi-WAN Failover

If main ISP fails, automatically use backup ISP:

```
System → Gateways → Gateways
  Gateway 1: ISP1 (default 203.0.113.1)
    Monitor IP: 8.8.8.8
    Alert: Down when 5 pings fail
  
  Gateway 2: ISP2 (backup 203.0.113.200)
    Monitor IP: 8.8.8.8
    Tier: 2 (used if Tier 1 down)

Firewall → Rules → LAN
  Add rule: Route via ISP1
  Add rule: Route via ISP2 when ISP1 down
```

Test failover:
```
Disable WAN interface → Watch traffic reroute to backup
```

---

## Troubleshooting

### Connectivity Issues

```bash
# SSH to pfSense (enable SSH in Web UI first)
ssh admin@10.0.10.1

# View interface status
ifconfig

# Check routing table
netstat -rn

# Test connectivity
ping 8.8.8.8

# Inspect firewall rules
pfctl -s rules
pfctl -s labels

# View NAT translations
pfctl -s nat
```

### Performance

- **CPU high**: Check for DPI/L7 filters; consider disabling
- **Memory full**: Increase state table in **System → Advanced → Firewall**
- **Packet loss**: Check interface errors with `ifconfig`; may indicate hardware issue

### Common Issues

| Issue | Solution |
|-------|----------|
| WAN no internet | Check Firewall → Rules → WAN (default deny?); verify NAT |
| DHCP not working | Ensure DHCP is enabled; check IP range; review syslog |
| VPN won't connect | Check firewall rules on WAN; verify certificates; check logs |
| High latency | Disable DPI; check QoS rules; monitor CPU/memory |

---

## Best Practices

1. **Segmentation**: Create separate VLANs (users, servers, guest, VoIP, IoT)
2. **Redundancy**: Deploy CARP pair for HA; multi-WAN for failover
3. **Logging**: Enable syslog export to central server for forensics
4. **Updates**: Apply patches regularly; test in lab first
5. **Backups**: Export config weekly and store securely
6. **Documentation**: Label rules with clear descriptions
7. **Monitoring**: Use Telegraf + InfluxDB + Grafana for long-term metrics
8. **Security**: Change default password; disable unnecessary services; use SSH keys

---

*pfSense firewall, router, and VPN guide for SMBs and labs using 10.x addressing and enterprise patterns.*
