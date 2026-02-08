# WireGuard - Modern VPN Protocol

Complete guide to WireGuard for fast, secure VPN connections with minimal code complexity and modern cryptography.

## Table of Contents
1. [Installation](#installation)
2. [Basic VPN Setup](#basic-vpn-setup)
3. [Site-to-Site VPN](#site-to-site-vpn)
4. [VPN Mesh Network](#vpn-mesh-network)
5. [Comparison to Other Protocols](#comparison-to-other-protocols)
6. [Mobile/Remote Access](#mobileremote-access)
7. [Performance Tuning](#performance-tuning)
8. [Monitoring](#monitoring)
9. [Troubleshooting](#troubleshooting)

---

## Installation

### Ubuntu/Debian

```bash
sudo apt update
sudo apt install -y wireguard wireguard-tools

# Verify
wg --version
ip link add wg0 type wireguard  # Test interface creation
ip link del wg0

# Enable IP forwarding (for VPN gateway)
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### CentOS/RHEL 8+

```bash
sudo dnf install -y wireguard-tools

# Enable IP forwarding
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Docker

```bash
docker run -it --rm --cap-add=NET_ADMIN --cap-add=SYS_MODULE \
  -v /lib/modules:/lib/modules linuxserver/wireguard

# Or for persistent VPN gateway container
```

---

## Basic VPN Setup

### Generate Keys

```bash
# Generate private and public key
wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey

# Secure permissions
sudo chmod 600 /etc/wireguard/privatekey
sudo chmod 644 /etc/wireguard/publickey

# View keys
cat /etc/wireguard/privatekey
cat /etc/wireguard/publickey
```

### Server Configuration (VPN Gateway at 10.0.99.1)

Create `/etc/wireguard/wg0.conf`:

```ini
[Interface]
PrivateKey = <server_private_key>
Address = 10.0.99.1/24
ListenPort = 51820
SaveConfig = false

# Allow IPv4 forwarding for VPN traffic
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# Peer 1: Laptop (client)
[Peer]
PublicKey = <laptop_public_key>
AllowedIPs = 10.0.99.2/32

# Peer 2: Phone (mobile)
[Peer]
PublicKey = <phone_public_key>
AllowedIPs = 10.0.99.3/32

# Peer 3: Remote office site-to-site
[Peer]
PublicKey = <remote_office_public_key>
AllowedIPs = 10.0.99.4/32, 10.0.20.0/24  # VPN IP + office subnet
```

Enable and start:
```bash
sudo chmod 600 /etc/wireguard/wg0.conf
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0

# Verify
ip addr show wg0
wg show
```

### Client Configuration (Laptop at 10.0.99.2)

Create `/etc/wireguard/wg-client.conf`:

```ini
[Interface]
PrivateKey = <client_private_key>
Address = 10.0.99.2/32
DNS = 8.8.8.8, 8.8.4.4

[Peer]
PublicKey = <server_public_key>
Endpoint = vpn-gateway.example.com:51820
AllowedIPs = 10.0.99.0/24, 10.0.0.0/8
PersistentKeepalive = 25
```

Connect:
```bash
sudo systemctl start wg-quick@wg-client

# Verify connection
ping 10.0.99.1
```

---

## Site-to-Site VPN

### Headquarters to Remote Office

**HQ VPN Gateway (10.0.10.1) Config**:
```ini
[Interface]
PrivateKey = <hq_private_key>
Address = 10.0.99.1/24
ListenPort = 51820

PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# Remote office gateway peer
[Peer]
PublicKey = <remote_public_key>
AllowedIPs = 10.0.99.2/32, 10.0.20.0/24  # Remote VPN IP + remote subnet
Endpoint = remote-vpn.example.com:51820
PersistentKeepalive = 25
```

**Remote Office VPN Gateway (10.0.20.1) Config**:
```ini
[Interface]
PrivateKey = <remote_private_key>
Address = 10.0.99.2/24
ListenPort = 51820

PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# HQ gateway peer
[Peer]
PublicKey = <hq_public_key>
AllowedIPs = 10.0.99.1/32, 10.0.10.0/24  # HQ VPN IP + HQ subnet
Endpoint = hq-vpn.example.com:51820
PersistentKeepalive = 25
```

Result:
```
HQ Subnet (10.0.10.0/24)
    |
    v
HQ VPN Gateway (10.0.10.1) <-- WireGuard Tunnel --> Remote VPN Gateway (10.0.20.1)
                                   (10.0.99.1/2)
    
    v
Remote Office Subnet (10.0.20.0/24)

Hosts on 10.0.10.x can ping 10.0.20.x automatically
```

---

## VPN Mesh Network

### 3-Node Mesh (Each node connects to all others)

Generate keys for 3 nodes:
```bash
for i in 1 2 3; do
  wg genkey | tee node$i.key | wg pubkey > node$i.pub
done
```

**Node 1 (10.0.99.1) Config**:
```ini
[Interface]
PrivateKey = <node1_private_key>
Address = 10.0.99.1/24
ListenPort = 51820

PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT

[Peer]
PublicKey = <node2_public_key>
AllowedIPs = 10.0.99.2/32
Endpoint = node2.example.com:51820
PersistentKeepalive = 25

[Peer]
PublicKey = <node3_public_key>
AllowedIPs = 10.0.99.3/32
Endpoint = node3.example.com:51820
PersistentKeepalive = 25
```

**Node 2 (10.0.99.2) Config**:
```ini
[Interface]
PrivateKey = <node2_private_key>
Address = 10.0.99.2/24
ListenPort = 51820

[Peer]
PublicKey = <node1_public_key>
AllowedIPs = 10.0.99.1/32
Endpoint = node1.example.com:51820
PersistentKeepalive = 25

[Peer]
PublicKey = <node3_public_key>
AllowedIPs = 10.0.99.3/32
Endpoint = node3.example.com:51820
PersistentKeepalive = 25
```

**Node 3** follows same pattern with endpoints to nodes 1 and 2.

Topology:
```
    Node-1 (10.0.99.1)
       /  \
      /    \
  Node-2   Node-3
(10.0.99.2) (10.0.99.3)

All nodes can ping each other directly
```

---

## Comparison to Other Protocols

### WireGuard vs IPSec vs OpenVPN

| Feature | WireGuard | IPSec | OpenVPN |
|---------|-----------|-------|---------|
| **Code Size** | ~4,000 lines | ~100,000 lines | ~100,000 lines |
| **Cryptography** | Modern (Curve25519, ChaCha20) | Older but proven | Good (AES-256) |
| **Speed** | Fastest (~10-50% faster) | Medium | Slower (50-100Mbps) |
| **CPU Usage** | Lowest | Medium | High |
| **Setup Complexity** | Simple (5 minutes) | Complex (30+ minutes) | Medium (15 minutes) |
| **Key Rotation** | Automatic | Manual | Manual |
| **UDP Only** | Yes (stateless) | Both TCP/UDP | Both TCP/UDP |
| **NAT Traversal** | Good (PersistentKeepalive) | Excellent | Good |
| **Mobile Friendly** | Excellent | Poor | Medium |
| **Kernel Support** | Linux 5.6+, iOS, Android | All OSes | All OSes |

**Use WireGuard for**:
- Mobile VPN
- Site-to-site fast links
- Mesh networks
- Modern deployments

**Use IPSec/OpenVPN for**:
- Enterprise legacy systems
- Complex security policies
- Older OS compatibility

---

## Mobile/Remote Access

### Pre-Shared Keys (PSK) for Extra Security

```bash
# Generate PSK
wg genpsk > /tmp/preshared.key

# Add to config
[Peer]
PublicKey = <client_public_key>
AllowedIPs = 10.0.99.2/32
PresharedKey = <preshared_key_content>
```

### Dynamic IP Clients (Laptops)

```ini
[Interface]
PrivateKey = <client_key>
Address = 10.0.99.2/32
DNS = 10.0.30.10  # Internal DNS

[Peer]
PublicKey = <server_key>
Endpoint = vpn.example.com:51820
AllowedIPs = 10.0.99.0/24, 10.0.0.0/8, 192.168.0.0/16
PersistentKeepalive = 25
```

### Full Tunnel vs Split Tunnel

```ini
# Full tunnel (all traffic through VPN)
AllowedIPs = 0.0.0.0/0, ::/0

# Split tunnel (only internal subnets through VPN)
AllowedIPs = 10.0.0.0/8
```

---

## Performance Tuning

### Kernel Parameters

```bash
# For high-throughput VPN gateway
echo "net.core.rmem_max = 134217728" | sudo tee -a /etc/sysctl.conf
echo "net.core.wmem_max = 134217728" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_rmem = 4096 87380 67108864" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_wmem = 4096 65536 67108864" | sudo tee -a /etc/sysctl.conf

sudo sysctl -p
```

### UDP Buffer Sizes

```bash
# Increase UDP receive buffer
echo "net.core.rmem_max = 262142592" | sudo tee -a /etc/sysctl.conf
echo "net.core.rmem_default = 262142592" | sudo tee -a /etc/sysctl.conf

# Check current buffer
cat /proc/sys/net/core/rmem_max
```

### Connection Tracking

```bash
# For heavy connection tracking at firewall
echo "net.netfilter.nf_conntrack_max = 2000000" | sudo tee -a /etc/sysctl.conf
echo "net.netfilter.nf_conntrack_tcp_timeout_established = 600" | sudo tee -a /etc/sysctl.conf

sudo sysctl -p
```

---

## Monitoring

### Check VPN Status

```bash
# Show WireGuard interface details
sudo wg show

# Output:
# interface: wg0
#   public key: abc123...
#   private key: (hidden)
#   listening port: 51820
# 
# peer: xyz789...
#   endpoint: 192.168.1.100:51820
#   allowed ips: 10.0.99.2/32
#   latest handshake: 2 minutes, 30 seconds ago
#   transfer: 1.2 MB received, 3.5 MB sent

# Get peer transfer stats
sudo wg show wg0 transfer

# Monitor in real-time
watch -n 1 'wg show wg0 transfer'
```

### Prometheus Metrics

```bash
# Install wireguard_exporter
docker run -d -p 9586:9586 \
  -v /etc/wireguard:/etc/wireguard:ro \
  MindFlavor/prometheus_wireguard_exporter:latest

# Add to prometheus.yml
scrape_configs:
  - job_name: 'wireguard'
    static_configs:
      - targets: ['localhost:9586']
```

### Latency & Packet Loss

```bash
# From client to server
ping -c 10 10.0.99.1

# Monitor handshake frequency
sudo wg show all handshakes

# Low handshake interval = network instability
```

---

## Troubleshooting

### Connection Refused

```bash
# Verify WireGuard is running
sudo wg show
# If empty, not running

# Start interface
sudo systemctl start wg-quick@wg0

# Check systemd logs
sudo systemctl status wg-quick@wg0
sudo journalctl -u wg-quick@wg0 -n 20
```

### Can't Connect to Peer

```bash
# Check firewall allows 51820/UDP
sudo ufw allow 51820/udp
sudo firewall-cmd --add-port=51820/udp --permanent

# Verify endpoint is reachable
nc -u -z <endpoint_ip> 51820

# Check for NAT issues (if behind NAT)
# Add to server peer config:
PersistentKeepalive = 25  # Sends keepalive every 25s
```

### No Traffic After Connection

```bash
# Verify routing
ip route show

# Should see VPN subnet routes, e.g.:
# 10.0.99.0/24 dev wg0 proto kernel scope link src 10.0.99.2

# If missing, check config AllowedIPs

# Check iptables rules (on gateway)
sudo iptables -t nat -L -n -v | grep -i wireguard

# Check IP forwarding
cat /proc/sys/net/ipv4/ip_forward  # Should be 1
```

### Slow Performance

```bash
# Check MTU size (often 1420 for WireGuard)
ip link show wg0 | grep mtu

# Set MTU if too high
sudo ip link set dev wg0 mtu 1420

# Monitor CPU usage on gateway
top -p $(pgrep wg)  # Should be <5% on modern CPU

# Check for packet loss
ping -c 100 10.0.99.1 | grep -i loss
```

### Key Management

```bash
# Rotate keys periodically
wg set wg0 private-key <(wg genkey)

# Update public key on peers
sudo wg show wg0 public-key

# For zero-downtime key rotation:
# 1. Add both old and new keys temporarily
# 2. Rotate peer keys
# 3. Remove old key
```

---

## Best Practices

1. **Key Management**: Rotate keys periodically; secure storage for private keys
2. **Pre-Shared Keys**: Use PSK for additional security on sensitive links
3. **PersistentKeepalive**: Set for NAT traversal (especially mobile clients)
4. **Monitoring**: Track handshakes and transfer rates; alert on disconnects
5. **MTU**: Set appropriately (1420 typical) to avoid fragmentation
6. **Kernel Version**: Use Linux 5.6+ for native kernel support
7. **Firewall**: Allow VPN subnet traffic in iptables rules
8. **IP Forwarding**: Enable on gateways for site-to-site connectivity
9. **DNS**: Configure internal DNS for VPN subnet resolution
10. **Mesh Topology**: Use consistent naming/IP scheme across all nodes

---

*WireGuard modern VPN protocol guide for fast, secure site-to-site and remote access.*
