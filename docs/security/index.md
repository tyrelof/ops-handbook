# Security & VPN Documentation

VPN protocols, encryption, secure remote access, and network security.

## Overview

This section covers modern VPN solutions including WireGuard for site-to-site and remote access VPN deployment.

## Cookbooks

### [WireGuard VPN](cookbooks/wireguard-vpn.md)
Modern VPN protocol for fast, secure connections with minimal code complexity. Covers installation, basic client-server setup, site-to-site VPN, mesh networks, and comparison to IPSec/OpenVPN.

**Topics**:
- Key generation and management
- Client-server VPN configuration
- Site-to-site VPN between offices (10.0.10.x ↔ 10.0.20.x)
- Mesh network topologies
- Mobile/remote access with split tunneling
- Performance tuning (kernel parameters, UDP buffers)
- Monitoring with Prometheus
- Troubleshooting connection issues

**Use Cases**:
- Remote employee VPN access
- Site-to-site office connectivity
- Mesh network for multiple offices
- High-performance VPN gateway
- Mobile device VPN

---

## Related Sections

- [pfSense Networking](../networking/cookbooks/pfsense.md) - Alternative firewall VPN (IPSec/OpenVPN)
- [Linux Networking](../linux/networking.md) - Network configuration and firewall rules
- [Kubernetes Security](../kubernetes/guardrails.md) - Pod network security policies

---

*Last updated: 2024 | Community contributions welcome*
