# Cookbook: Performance Monitoring

> [!IMPORTANT]
> This guide covers modern monitoring and benchmarking tools for Ubuntu 22.04/24.04 LTS. 
> Key modernizations include **btop** (visual monitoring), **ss** (network stats), and **fio** (precise I/O benchmarking).

## 1. Modern System Overviews

While `top` and `htop` are classics, modern tools provide much better visual context and real-time graphing.

### btop: The Modern Successor to htop
`btop` is a high-performance, visually stunning system monitor.
```bash
sudo apt update
sudo apt install btop -y
# To run
btop
```
*Features: Real-time graphs for CPU/RAM/Net/Disk, mouse support, and process management.*

### Glances: Remote & Web Monitoring
`glances` is an all-in-one monitoring tool that can be used via CLI or a web browser.
```bash
sudo apt install glances -y
# Web mode (Access via http://your-ip:61208)
glances -w
```

---

## 2. Resource-Specific Monitoring

### CPU Analysis
- **pidstat**: Monitor individual processes over time.
  ```bash
  pidstat -u 2 5  # 5 reports every 2 seconds
  ```
- **mpstat**: Multi-processor statistics (from `sysstat` package).
  ```bash
  mpstat -P ALL 2
  ```

### Memory & Swap
- **free**: Quick summary in human-readable labels.
  ```bash
  free -h
  ```
- **vmstat**: Virtual memory and system-wide activity.
  ```bash
  vmstat 1 5
  ```
*Note: Check the `si` (swap-in) and `so` (swap-out) columns for active disk swapping.*

### Network Monitoring
- **ss**: Replacing legacy `netstat`. Lists listening ports and active connections.
  ```bash
  ss -ptl  # All listening TCP ports with process names
  ss -s    # Overall socket statistics
  ```
- **nethogs**: Monitor bandwidth per-process.
  ```bash
  sudo nethogs eth0
  ```

### Storage Performance
- **iostat**: Disk I/O utilization.
  ```bash
  iostat -xz 1  # Extended stats excluding idle disks
  ```
- **iotop**: Identify which processes are using the most disk I/O.
  ```bash
  sudo iotop -o  # Only show processes doing actual I/O
  ```

---

## 3. Performance Benchmarking

### CPU & Memory with sysbench
```bash
sudo apt install sysbench -y
# CPU Benchmark (Prime number calculation)
sysbench cpu --threads=4 --cpu-max-prime=20000 run
# Memory Benchmark
sysbench memory --threads=4 --memory-total-size=10G run
```

### Disk I/O with fio (The Industry Standard)
`fio` (Flexible I/O Tester) is significantly more accurate than `sysbench` for storage.
```bash
sudo apt install fio -y
# Random Read/Write Test
fio --name=random-rw --ioengine=libaio --rw=randrw --bs=4k --size=1G --numjobs=4 --runtime=60 --time_based --end_fsync=1
```

---

## 4. Historical Analysis (sar)

The **System Activity Reporter (sar)** logs system data over time. In Ubuntu 24.04, ensure it is enabled:
```bash
# Enable collection in /etc/default/sysstat
sudo sed -i 's/ENABLED="false"/ENABLED="true"/' /etc/default/sysstat
sudo systemctl restart sysstat
# Query past CPU data
sar -u
# Query today's network data
sar -n DEV
```

---

## 5. Automated Dashboards: Netdata

For a "set it and forget it" high-resolution dashboard:
```bash
curl https://get.netdata.cloud/kickstart.sh | sh
```
*Access via port 19999. It automatically monitors thousands of metrics without configuration.*
