# Linux Networking

## Connectivity Checks
```bash
# Is the port open?
nc -vz <host> <port>

# DNS Lookup (trace)
dig +trace example.com

# HTTP Headers only
curl -I https://example.com
```

## Socket Statistics (ss)
Replace obsolete `netstat`.

```bash
# List listening TCP ports (numeric)
ss -ltn

# Show process using the port
ss -ltnp
```

## Interface & IP
```bash
ip addr show
ip route show
```

## Firewall (Quick Check)
```bash
# UFW (Ubuntu)
sudo ufw status

# Iptables (Raw)
sudo iptables -L -n -v
```
