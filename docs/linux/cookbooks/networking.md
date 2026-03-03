# Cookbook: Networking

> [!NOTE]
> Optimized for Ubuntu 22.04 LTS and 24.04 LTS.
> Focuses on **Netplan**, **WireGuard**, and modern **iproute2** tooling.

## 1. Network Interface Configuration (Netplan)

Ubuntu use **Netplan** to manage network configuration via YAML files in `/etc/netplan/`.

### Static IP Configuration
1. **Locate your config file:** Typically `/etc/netplan/01-netcfg.yaml` or `/etc/netplan/50-cloud-init.yaml`.
2. **Apply settings:**
```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```
3. **Validate and Apply:**
```bash
sudo netplan try    # Safe test (reverts if connection lost)
sudo netplan apply  # Permanent change
```

### DHCP Configuration
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: yes
```

---

## 2. Modern Networking Tools (`iproute2`)

Legacy tools like `ifconfig` and `netstat` are deprecated. Use the modern alternatives:

| Legacy Tool | Modern Tool | Common Command |
|---|---|---|
| `ifconfig` | `ip addr` | `ip a` (Show all interfaces) |
| `route` | `ip route` | `ip r` (Show routing table) |
| `netstat` | `ss` | `ss -tulpn` (Listen ports) |
| `arp` | `ip neighbor` | `ip neigh` (ARP table) |
| `nslookup` | `resolvectl` | `resolvectl status` (DNS status) |

---

## 3. VPN: WireGuard (Modern & Fast)

WireGuard is built into the Linux kernel and is significantly faster than OpenVPN.

### Server Setup
1. **Install:** `sudo apt update && sudo apt install wireguard`
2. **Generate Keys:**
```bash
wg genkey | tee privatekey | wg pubkey > publickey
```
3. **Configure (`/etc/wireguard/wg0.conf`):**
```ini
[Interface]
PrivateKey = <Server_Private_Key>
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <Client_Public_Key>
AllowedIPs = 10.0.0.2/32
```
4. **Enable:**
```bash
sudo systemctl enable --now wg-quick@wg0
```

---

## 4. Advanced Firewall Hardening

### UFW Application Profiles
```bash
# List available profiles
sudo ufw app list

# Allow SSH and Nginx
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

### Rate Limiting Brute Force (iptables/nftables)
Ubuntu 24.04 uses **nftables** as the backend. To prevent SSH brute force at the kernel level:

```bash
# Modern Simple Rate Limit for SSH
sudo ufw limit ssh
```

**Advanced nftables Hardening (`/etc/nftables.conf`):**
```text
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;
        
        # Allow established/related
        ct state established,related accept
        
        # Rate limit SSH (e.g., 5 attempts per minute)
        tcp dport 22 ct state new limit rate over 5/minute drop accept
    }
}
```

---

## 5. Load Balancing with HAProxy

HAProxy is ideal for distributing TCP or HTTP traffic.

### Basic TCP Balance (`/etc/haproxy/haproxy.cfg`)
```text
frontend main
    bind *:80
    default_backend app_servers

backend app_servers
    balance leastconn
    server srv1 10.0.1.10:80 check
    server srv2 10.0.1.11:80 check
```
**Apply:** `sudo systemctl restart haproxy`

---

## 6. Troubleshooting Connectivity

### 1. Check Link & IP
```bash
ip a         # Verify IP and UP status
ip link show # Check layer 2 status
```

### 2. Verify Routing
```bash
ip route     # Check default gateway
mtr 8.8.8.8  # Real-time path tracing
```

### 3. Check Service & DNS
```bash
ss -tulpn              # Are services listening?
resolvectl query google.com # Test DNS resolution
```

### 4. Pocket Sniffer (tcpdump)
```bash
# Capture traffic on specific interface and port
sudo tcpdump -i enp0s3 port 80 -n
```
