# Cookbook: Linux Server Monitoring & Performance Debugging

> [!NOTE]
> Practical guide to monitoring system health and debugging performance bottlenecks.
> Covers: CPU, memory, disk, network, and application-level diagnostics.

## 1. The Monitoring Stack

```bash
# Essential tools
apt-get install -y \
    htop           # Interactive process monitor
    iotop          # I/O by process
    netstat        # Network connections
    tcpdump        # Packet capture
    strace         # System call tracer
    perf           # Performance profiler
    sysstat        # sar, iostat (historical stats)
    curl           # HTTP client
    nc/ncat        # Network debugging
```

## 2. CPU Monitoring

### Real-Time CPU Usage
```bash
# Top processes by CPU
top -o %CPU

# Or use htop (interactive, sortable)
htop

# Show CPU cores and usage
nproc                    # Number of cores
lscpu                    # CPU info
cat /proc/cpuinfo        # Detailed CPU info
```

### Historical CPU Stats
```bash
# Average CPU over time (every 1 second, 10 samples)
sar -u 1 10

# Load average (1min, 5min, 15min)
uptime
cat /proc/loadavg
```

### Drill Down: Which Process is Using CPU?
```bash
# Show CPU usage by process, sorted
ps aux --sort=-%cpu | head -20

# For a specific process (PID 1234)
top -p 1234

# Get all threads (if multi-threaded app)
ps -eLf | grep <process-name>

# Trace system calls (see what it's doing)
strace -p 1234 -c       # Count syscalls
strace -p 1234 -e openat,read,write  # Filter specific calls
```

### Fix CPU Overload
```bash
# 1. Identify the process
ps aux --sort=-%cpu | head -5

# 2. Check if it's legitimate
# - Web server handling traffic? OK
# - Background worker? Check if it's stuck
# - Unknown process? Kill it

# 3. Limit CPU usage (nice/renice)
nice -n 19 ./heavy_process    # Start with lower priority
renice 19 -p <PID>             # Change running process

# 4. If still high, add CPU resources or optimize app
```

## 3. Memory Monitoring

### Real-Time Memory Usage
```bash
# Free memory overview
free -h

# Show memory by process
ps aux --sort=-%mem | head -20

# Detailed memory info
cat /proc/meminfo

# Memory usage in htop
htop  # Press 'M' to sort by memory
```

### Historical Memory Stats
```bash
# Memory usage over time
sar -r 1 10

# Show memory trend
sar -r | tail -20
```

### Find Memory Leaks
```bash
# Monitor specific process memory
watch -n 1 'ps aux | grep <process> | grep -v grep'

# Or more detailed
while true; do
  BYTES=$(ps aux | grep <process> | grep -v grep | awk '{print $6}')
  echo "$(date +%H:%M:%S) - Memory: ${BYTES}KB"
  sleep 5
done

# If growing continuously = memory leak
```

### Reduce Memory Usage
```bash
# 1. Find the culprit
ps aux --sort=-%mem | head -5

# 2. Check if it's a legitimate application
# Example: Docker daemon consuming too much?
systemctl restart docker

# 3. For applications: check for leaks or increase memory
# Restart the service
systemctl restart <service>

# 4. Or limit process memory (cgroup)
systemd-run --scope -p MemoryLimit=512M ./app
```

## 4. Disk Monitoring

### Disk Space
```bash
# Human-readable disk usage
df -h

# Detailed filesystem info
df -Th

# Show inode usage (filename limit)
df -i
```

### Disk I/O (Latency & Throughput)
```bash
# I/O stats by device
iostat -x 1 10   # Every 1 sec, 10 samples

# Key metrics:
# r/s = reads per second
# w/s = writes per second
# rkB/s = read throughput KB/sec
# wkB/s = write throughput KB/sec
# await = average wait time (ms)
# %util = disk utilization

# High await = slow disk or queue buildup
# High %util = disk is bottleneck
```

### Which Process is Using Disk?
```bash
# Top I/O consumers (requires iotop)
iotop -o -b -n 5

# Or trace open/read/write syscalls
strace -p <PID> -e openat,read,write -c

# Find large files
find / -type f -size +100M -exec ls -lh {} \;

# Find directories using most space
du -sh /*  # Show top-level dirs
du -sh /var/log/*  # Check logs
```

### Fix Disk Issues
```bash
# 1. Check what's filling up
du -sh /* | sort -h | tail -10

# 2. Common culprits: logs, caches, old Docker images
ls -lh /var/log/
docker system df
docker system prune -a  # Remove unused images/containers

# 3. Clean up old files
find /var/log -name "*.log" -mtime +30 -delete  # Delete logs older than 30 days

# 4. Or rotate logs
logrotate -f /etc/logrotate.conf
```

### Disk Space Alerts
```bash
# Check if disk is critical
df -h / | awk '{print $5}' | sed 's/%//' | tail -1

# Alert if >90%
USAGE=$(df -h / | awk '{print $5}' | sed 's/%//' | tail -1)
if [ "$USAGE" -gt 90 ]; then
    echo "WARNING: Disk usage at ${USAGE}%"
fi
```

## 5. Network Monitoring

### Network Interface Stats
```bash
# Summary
ip -s link show

# Real-time
watch -n 1 'cat /proc/net/dev'

# Or use iftop
iftop -n  # Top connections by bandwidth
```

### Active Connections
```bash
# Show all connections
netstat -tupan | head -20

# Or modern: ss (socket statistics)
ss -tupan | head -20

# Established connections only
ss -tpan | grep ESTAB

# Count connections by state
ss -tpan | grep -o 'State' | wc -l
```

### DNS Resolution
```bash
# Test DNS
nslookup example.com
dig example.com

# Check local DNS resolver
cat /etc/resolv.conf
```

### Network Latency & Packet Loss
```bash
# Ping (ICMP)
ping -c 10 8.8.8.8

# Trace route
traceroute example.com

# Check MTU
ip link show | grep mtu
```

### Packet Capture
```bash
# Capture traffic on interface
tcpdump -i eth0

# Capture to file
tcpdump -i eth0 -w capture.pcap

# Filter traffic
tcpdump -i eth0 'port 3000'
tcpdump -i eth0 'src 10.0.0.5'
tcpdump -i eth0 'tcp and (dst port 80 or dst port 443)'

# Analyze file
tcpdump -r capture.pcap
```

### Fix Network Issues
```bash
# 1. Check connectivity
ping <target>

# 2. Check routing
ip route show
traceroute <target>

# 3. Check DNS
nslookup <domain>

# 4. Check if service is listening
ss -tpan | grep <port>
netstat -tupan | grep <port>

# 5. Check firewall rules
iptables -L -n
ufw status

# 6. Restart networking
systemctl restart networking
# or
ip link set <interface> down && ip link set <interface> up
```

## 6. Application-Level Debugging

### Check if Service is Running
```bash
systemctl status <service>

# Or check if port is listening
netstat -tupan | grep <port>
ss -tpan | grep <port>

# Or curl health endpoint
curl http://localhost:3000/health
curl -i http://localhost:3000/health
```

### View Service Logs
```bash
# Systemd services
journalctl -u <service> -f

# Docker containers
docker logs <container> -f

# Kubernetes pods
kubectl logs <pod> -f
kubectl logs <pod> -f --previous  # Previous run if crashed
```

### Profile Application
```bash
# CPU flame graph (using perf)
perf record -F 99 -p <PID> -- sleep 30
perf report

# Memory profiling (app-specific)
# For Node.js
node --inspect app.js
# Then visit chrome://inspect

# For Python
python -m cProfile -s cumtime app.py

# For Go
import _ "net/http/pprof"
# Visit http://localhost:6060/debug/pprof/
```

### Trace System Calls
```bash
# Trace a running process
strace -p <PID> -e trace=file,network -f

# Trace a new process and count syscalls
strace -c ./my_app

# Follow threads (useful for multi-threaded apps)
strace -p <PID> -f

# Filter by syscall
strace -p <PID> -e openat,write
```

## 7. Monitoring Dashboard (Simple Shell Script)

```bash
#!/bin/bash
# monitor.sh - Quick system health check

clear

echo "=== SYSTEM HEALTH CHECK ==="
echo ""

echo "[CPU]"
uptime
echo ""

echo "[MEMORY]"
free -h | head -2
echo ""

echo "[DISK]"
df -h / | tail -1
echo ""

echo "[TOP PROCESSES (CPU)]"
ps aux --sort=-%cpu | head -6
echo ""

echo "[TOP PROCESSES (MEMORY)]"
ps aux --sort=-%mem | head -6
echo ""

echo "[NETWORK CONNECTIONS]"
ss -tpan | grep ESTAB | wc -l
echo "Established connections: $(ss -tpan | grep ESTAB | wc -l)"
echo ""

echo "[SYSTEMD FAILED UNITS]"
systemctl status | grep failed
```

Run: `./monitor.sh`

## 8. Automated Monitoring with `watch`

```bash
# CPU every 2 seconds
watch -n 2 'top -b -n 1 | head -15'

# Memory trend
watch -n 5 'free -h'

# Disk space
watch -n 10 'df -h'

# Network
watch -n 1 'ifstat'

# Processes handling connections
watch -n 5 'ss -tpan | grep ESTAB | wc -l'
```

## 9. Common Issues & Solutions

| Issue | Diagnosis | Fix |
|-------|-----------|-----|
| High CPU | `top`, `ps aux --sort=-%cpu` | Kill/restart process, optimize app, add resources |
| Memory leak | `watch 'ps aux \| grep <proc>'` | Restart service, fix leak in code |
| Slow disk | `iostat -x`, `iotop` | Check queue, reduce I/O, upgrade disk |
| Full disk | `du -sh /` | Delete files, compress logs, clean cache |
| Network lag | `ping`, `traceroute` | Check routes, firewall, MTU, TCP retransmits |
| Service down | `systemctl status` | Check logs, restart, debug startup errors |

---

**Best Practices**:
- Monitor regularly, not just during crises
- Set up alerts for thresholds (CPU >80%, Memory >90%, Disk >85%)
- Keep historical data (sar, prometheus, etc.)
- Always check logs first
- Use proper profiling tools, not guesswork
