# Networking Documentation

Advanced networking, firewall configuration, routing, and network design patterns.

## Overview

This section covers enterprise network infrastructure including edge firewalls, routing protocols, VPN, and load balancing for production environments.

## Cookbooks

### [pfSense](cookbooks/pfsense.md)
Open-source firewall, router, and VPN solution. Covers installation, interface configuration, firewall rules, NAT, DHCP/DNS, IPSec/OpenVPN VPN setup, CARP high availability, and dynamic routing with OSPF.

**Topics**:
- Firewall rules and NAT (dynamic, static, 1:1 mapping)
- DHCP/DNS resolver
- IPSec site-to-site and OpenVPN remote access VPN
- CARP virtual IP for HA failover
- OSPF dynamic routing
- Monitoring and troubleshooting

**Use Cases**:
- SMB edge firewall
- Lab VPN gateway
- Site-to-site VPN termination
- High availability failover with CARP

---

## Related Sections

- [Cisco Networking](../cisco/cookbooks/network-design.md) - Enterprise L2/L3 design, OSPF/BGP, ASA firewall
- [Load Balancing](../load-balancing/) - HAProxy, session persistence, health checks
- [VPN & Remote Access](../kubernetes/cheatsheet.md) - IPSec, OpenVPN, WireGuard

---

*Last updated: 2024 | Community contributions welcome*
