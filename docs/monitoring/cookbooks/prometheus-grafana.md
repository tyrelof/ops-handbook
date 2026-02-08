# Prometheus & Grafana - Monitoring Stack

Complete guide to deploying Prometheus for metrics collection, Grafana for visualization, and Alertmanager for intelligent alerting.

## Table of Contents
1. [Architecture](#architecture)
2. [Prometheus Installation](#prometheus-installation)
3. [Configuration](#configuration)
4. [Grafana Setup](#grafana-setup)
5. [Dashboards](#dashboards)
6. [Alerting](#alerting)
7. [Data Retention](#data-retention)
8. [Troubleshooting](#troubleshooting)

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│           Monitoring Stack (10.0.30.x)          │
├─────────────────────────────────────────────────┤
│  Prometheus              Grafana                 │
│  (10.0.30.10:9090)      (10.0.30.11:3000)       │
│  - Scrapes metrics       - Visualizes data      │
│  - Time-series DB        - Dashboards          │
│  - Alertmanager          - Alerts via UI       │
└─────────────────────────────────────────────────┘
                    ↑
         ┌──────────┼──────────┐
         │          │          │
    Exporters    Exporters  Exporters
   Node (9100)  Postgres   Nginx
   (Servers)    (9187)     (9113)
   (10.0.x.x)   (10.0.x.x) (10.0.x.x)
```

---

## Prometheus Installation

### Download & Install

```bash
# Create prometheus user
sudo useradd -rs /bin/false prometheus
sudo useradd -rs /bin/false alertmanager

# Download Prometheus
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.42.0/prometheus-2.42.0.linux-amd64.tar.gz
tar -xvzf prometheus-2.42.0.linux-amd64.tar.gz

# Install
sudo cp prometheus-2.42.0.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-2.42.0.linux-amd64/promtool /usr/local/bin/
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo cp -r prometheus-2.42.0.linux-amd64/consoles /etc/prometheus/
sudo cp -r prometheus-2.42.0.linux-amd64/console_libraries /etc/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus

# Verify
prometheus --version
```

### Systemd Service

Create `/etc/systemd/system/prometheus.service`:

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --storage.tsdb.retention.time=30d

Restart=on-failure
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

Enable & start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus

# Verify at http://10.0.30.10:9090
```

---

## Configuration

### Basic prometheus.yml

Create `/etc/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s         # Scrape every 15 seconds
  evaluation_interval: 15s      # Evaluate rules every 15s
  external_labels:
    cluster: 'production'
    environment: 'prod'

scrape_configs:
  # Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node Exporter (system metrics)
  - job_name: 'linux-servers'
    static_configs:
      - targets:
          - '10.0.10.10:9100'  # Server 1
          - '10.0.10.11:9100'  # Server 2
          - '10.0.10.12:9100'  # Server 3

  # PostgreSQL
  - job_name: 'postgresql'
    static_configs:
      - targets: ['10.0.30.1:9187']

  # Nginx
  - job_name: 'nginx'
    static_configs:
      - targets: ['10.0.50.10:9113']

  # Docker/Container metrics (via cAdvisor)
  - job_name: 'docker'
    static_configs:
      - targets: ['10.0.10.10:8080']
```

### Service Discovery (Dynamic)

For dynamic environments, use file-based SD:

```yaml
scrape_configs:
  - job_name: 'dynamic-servers'
    file_sd_configs:
      - files:
          - '/etc/prometheus/targets.json'
        refresh_interval: 30s
```

Create `/etc/prometheus/targets.json`:

```json
[
  {
    "targets": ["10.0.10.10:9100", "10.0.10.11:9100"],
    "labels": {
      "job": "web-servers",
      "environment": "production"
    }
  },
  {
    "targets": ["10.0.30.1:9100"],
    "labels": {
      "job": "database-server",
      "environment": "production"
    }
  }
]
```

---

## Grafana Setup

### Install

```bash
# Add repository
sudo apt-get install -y software-properties-common
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo apt-get update

# Install Grafana
sudo apt-get install -y grafana-server

# Start
sudo systemctl start grafana-server
sudo systemctl enable grafana-server

# Access at http://10.0.30.11:3000 (admin/admin)
```

### Add Prometheus Data Source

1. **Configuration → Data Sources → Add**
2. **Name**: `Prometheus`
3. **Type**: Prometheus
4. **URL**: `http://10.0.30.10:9090`
5. **Save & Test**

---

## Dashboards

### Official Dashboards

Import pre-built dashboards from [grafana.com/dashboards](https://grafana.com/dashboards):

- **Node Exporter Full** (ID: 1860): System metrics
- **PostgreSQL Database** (ID: 3662): Database stats
- **Nginx** (ID: 1141): Web server metrics
- **Prometheus Stats** (ID: 2): Prometheus health

**To Import**:
1. **Dashboards → Import**
2. Enter Dashboard ID
3. Select Prometheus data source
4. Import

### Custom Dashboard (Simple)

Create dashboard with panels:

**Panel 1: CPU Usage**
```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Panel 2: Memory Usage**
```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

**Panel 3: Disk Usage**
```promql
(node_filesystem_size_bytes{mountpoint="/"} - node_filesystem_avail_bytes{mountpoint="/"}) 
/ node_filesystem_size_bytes{mountpoint="/"} * 100
```

---

## Alerting

### Alertmanager Installation

```bash
# Download
wget https://github.com/prometheus/alertmanager/releases/download/v0.25.0/alertmanager-0.25.0.linux-amd64.tar.gz
tar -xvzf alertmanager-0.25.0.linux-amd64.tar.gz

# Install
sudo cp alertmanager-0.25.0.linux-amd64/alertmanager /usr/local/bin/
sudo mkdir -p /etc/alertmanager /var/lib/alertmanager
sudo chown -R alertmanager:alertmanager /etc/alertmanager /var/lib/alertmanager
```

### Alertmanager Config

Create `/etc/alertmanager/alertmanager.yml`:

```yaml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'

route:
  receiver: 'default'
  group_by: ['job']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h

  routes:
    - match:
        severity: critical
      receiver: 'critical-team'
      continue: true
    
    - match:
        severity: warning
      receiver: 'ops-team'

receivers:
  - name: 'default'
    slack_configs:
      - channel: '#alerts'
        title: 'Alert: {{ .GroupLabels.job }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'critical-team'
    slack_configs:
      - channel: '#critical-alerts'
    email_configs:
      - to: 'oncall@example.com'
        from: 'alerts@example.com'
        smarthost: 'smtp.example.com:587'
        auth_username: 'alerts@example.com'
        auth_password: 'password'

  - name: 'ops-team'
    slack_configs:
      - channel: '#ops-alerts'
```

### Alert Rules

Create `/etc/prometheus/alert-rules.yml`:

```yaml
groups:
  - name: linux-alerts
    interval: 30s
    rules:
      - alert: HighCPUUsage
        expr: (100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on {{ $labels.instance }}"
          description: "CPU usage is above 80% for 5 minutes"

      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory on {{ $labels.instance }}"
          description: "Memory usage is above 85%"

      - alert: DiskSpaceLow
        expr: (node_filesystem_size_bytes - node_filesystem_avail_bytes) / node_filesystem_size_bytes > 0.85
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Disk {{ $labels.device }} is above 85% full"

  - name: database-alerts
    interval: 30s
    rules:
      - alert: PostgreSQLConnectionsHigh
        expr: sum(pg_stat_activity_count) > 150
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High PostgreSQL connections ({{ $value }})"

  - name: service-alerts
    interval: 30s
    rules:
      - alert: ServiceDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down"
```

Update `prometheus.yml` to include rules:

```yaml
rule_files:
  - '/etc/prometheus/alert-rules.yml'

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']
```

### Systemd for Alertmanager

Create `/etc/systemd/system/alertmanager.service`:

```ini
[Unit]
Description=Alertmanager
After=network-online.target

[Service]
Type=simple
User=alertmanager
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable & start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable alertmanager
sudo systemctl start alertmanager

# Verify at http://10.0.30.10:9093
```

---

## Data Retention

Prometheus stores TSDB data locally. Configure retention:

In `prometheus.service`:
```ini
ExecStart=/usr/local/bin/prometheus \
  --storage.tsdb.retention.time=30d
```

Or by size:
```ini
ExecStart=/usr/local/bin/prometheus \
  --storage.tsdb.retention.size=50GB
```

For long-term storage, use remote backends:
- Cortex
- Thanos
- InfluxDB
- TimescaleDB (PostgreSQL)

---

## Troubleshooting

### Metrics Missing

```bash
# Check targets status
curl http://10.0.30.10:9090/api/v1/targets

# Check scrape errors
# WebUI: Status → Targets

# Test exporter connectivity
curl http://10.0.10.10:9100/metrics
```

### High Disk Usage

```bash
# Check TSDB size
du -sh /var/lib/prometheus/metrics-*/

# Reduce retention
# Edit prometheus.service and restart
```

### Alertmanager Issues

```bash
# Check config syntax
/usr/local/bin/amtool check-config /etc/alertmanager/alertmanager.yml

# View alerts
curl http://10.0.30.10:9093/api/v1/alerts

# Logs
journalctl -u alertmanager -f
```

---

## Advanced PromQL Queries

### Operator Precedence & Examples

```
# Instant vector (current value) vs Range vector (values over time)
node_cpu_seconds_total                    # Instant
node_cpu_seconds_total[5m]                # Range (last 5 minutes)

# Rate (per second increase)
rate(node_cpu_seconds_total[5m])          # CPU usage per second

# Aggregation operators
sum(rate(http_requests_total[5m]))        # Total requests/sec
avg(node_memory_MemFree_bytes) by (instance)  # Avg free mem per server
count(up)                                  # Count online instances
```

### Complex Queries for Dashboards

```
# Disk usage percentage
(1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100

# Query error rate
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) * 100

# P95 latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Database replication lag in seconds
pg_wal_lsn_last_removed_bytes - pg_wal_lsn_last_wal_received_bytes
```

---

## Custom Exporters

### Creating a Custom Metric Exporter

Python example (`/opt/custom_exporter.py`):
```python
#!/usr/bin/env python3
from prometheus_client import CollectorRegistry, Gauge, start_http_server
import psutil
import time

registry = CollectorRegistry()

# Define custom metrics
cpu_percent = Gauge('custom_cpu_percent', 'CPU percentage', registry=registry)
disk_free = Gauge('custom_disk_free_bytes', 'Free disk space', registry=registry)

def collect_metrics():
    while True:
        cpu_percent.set(psutil.cpu_percent(interval=1))
        disk_free.set(psutil.disk_usage('/').free)
        time.sleep(10)

if __name__ == '__main__':
    start_http_server(9999, registry=registry)
    collect_metrics()
```

Run:
```bash
nohup /usr/bin/python3 /opt/custom_exporter.py > /var/log/custom_exporter.log 2>&1 &

# Verify metrics at http://localhost:9999/metrics
curl http://localhost:9999/metrics
```

Add to prometheus.yml:
```yaml
scrape_configs:
  - job_name: 'custom'
    static_configs:
      - targets: ['localhost:9999']
```

---

## Best Practices

1. **Scrape Interval**: 15-30s for normal environments
2. **Retention**: 30 days for local TSDB; use remote storage for long-term
3. **Labels**: Add context (env, region, team) to improve queries
4. **Alerts**: Set thresholds conservatively; avoid alert fatigue
5. **Dashboards**: Use business metrics, not just raw data
6. **Security**: Use authentication (reverse proxy); limit exporter access
7. **Scalability**: Use Prometheus federation for multi-cluster
8. **Custom Metrics**: Create exporters for application-specific monitoring
9. **PromQL**: Master aggregation and rate functions for accurate trending
10. **Testing**: Test alert rules before production deployment

---

*Prometheus & Grafana monitoring stack for infrastructure and application observability.*
