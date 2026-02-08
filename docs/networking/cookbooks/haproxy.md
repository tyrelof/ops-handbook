# HAProxy - High-Performance Load Balancing

Complete guide to HAProxy for distributing traffic across backend servers, SSL/TLS termination, advanced routing rules, and session persistence.

## Table of Contents
1. [Installation](#installation)
2. [Configuration Basics](#configuration-basics)
3. [Frontend & Backend](#frontend--backend)
4. [Load Balancing Algorithms](#load-balancing-algorithms)
5. [SSL/TLS Termination](#ssltls-termination)
6. [Advanced Routing](#advanced-routing)
7. [Health Checks](#health-checks)
8. [Session Persistence](#session-persistence)
9. [Monitoring & Stats](#monitoring--stats)
10. [High Availability](#high-availability)
11. [Troubleshooting](#troubleshooting)

---

## Installation

### Ubuntu/Debian

```bash
sudo apt update
sudo apt install -y haproxy

# Verify
haproxy -v

# Enable service
sudo systemctl enable haproxy
sudo systemctl start haproxy
```

### CentOS/RHEL

```bash
sudo yum install -y haproxy

# Enable service
sudo systemctl enable haproxy
sudo systemctl start haproxy
```

### Configuration Location

```bash
/etc/haproxy/haproxy.cfg      # Main config
/var/log/haproxy.log          # Log file
```

---

## Configuration Basics

### Minimal haproxy.cfg

```
global
  log /dev/log  local0
  log /dev/log  local1 notice
  chroot /var/lib/haproxy
  stats socket /run/haproxy/admin.sock mode 660 level admin
  stats timeout 30s
  user haproxy
  group haproxy
  daemon

  # Performance
  maxconn 4096
  tune.ssl.default-dh-param 2048

defaults
  log     global
  mode    http
  option  httplog
  option  denyall-close
  timeout connect 5000
  timeout client  50000
  timeout server  50000

# Simple HTTP load balancer
frontend web_in
  bind *:80
  default_backend web_servers

backend web_servers
  balance roundrobin
  server web1 10.0.10.10:80 check
  server web2 10.0.10.11:80 check
  server web3 10.0.10.12:80 check
```

### Reload Configuration

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg  # Check syntax

sudo systemctl reload haproxy                  # Reload (graceful)
sudo systemctl restart haproxy                 # Restart (connections drop)
```

---

## Frontend & Backend

### Multi-Frontend Setup

```
frontend web_in
  bind *:80
  bind *:443 ssl crt /etc/ssl/certs/example.pem
  
  # HTTP to HTTPS redirect
  http-request redirect scheme https code 301 if !{ ssl_fc }
  
  # Host-based routing
  acl is_api_req hdr(host) -i api.example.com
  acl is_app_req hdr(host) -i app.example.com
  
  use_backend api_servers if is_api_req
  use_backend app_servers if is_app_req
  default_backend web_servers

frontend admin_in
  bind *:8080
  default_backend admin_servers

# Web servers backend
backend web_servers
  balance roundrobin
  option httpchk GET / HTTP/1.1\r\nHost:\ www.example.com
  server web1 10.0.10.10:80 check inter 2s fall 3 rise 2
  server web2 10.0.10.11:80 check inter 2s fall 3 rise 2
  server web3 10.0.10.12:80 check inter 2s fall 3 rise 2

# API servers backend (different checks)
backend api_servers
  balance leastconn
  option httpchk GET /health HTTP/1.1\r\nHost:\ api.example.com
  server api1 10.0.10.20:8080 check
  server api2 10.0.10.21:8080 check

# Admin backend
backend admin_servers
  balance roundrobin
  server admin1 10.0.10.50:8080 check
```

### TCP/Non-HTTP Backend

```
frontend mysql_in
  bind *:3306
  mode tcp
  default_backend mysql_servers

backend mysql_servers
  mode tcp
  balance roundrobin
  option tcp-check
  tcp-check connect port 3306
  server db1 10.0.30.10:3306 check
  server db2 10.0.30.11:3306 check
```

---

## Load Balancing Algorithms

### Available Algorithms

```
balance roundrobin        # Default: sequential round-robin
balance leastconn         # Fewest active connections (sticky)
balance static-rr         # Round-robin, weighted (must define weights)
balance source            # Source IP hash (persistent per client)
balance uri               # URI hash (sticky per URL)
balance url_param hash    # Hash on query parameter
balance hdr(X-Custom)     # Hash on custom header
balance random            # Random selection
```

### Weighted Round-Robin

```
backend web_servers
  balance static-rr
  
  # Heavier server gets more traffic
  server web1 10.0.10.10:80 weight 3 check
  server web2 10.0.10.11:80 weight 1 check
  server web3 10.0.10.12:80 weight 2 check
```

### Source IP Persistence (Session Affinity)

```
backend app_servers
  balance source              # Hash based on client IP
  hash-type consistent        # Consistent hashing (adds/removes server without rebalancing all)
  server app1 10.0.10.20:8080 check
  server app2 10.0.10.21:8080 check
  server app3 10.0.10.22:8080 check
```

---

## SSL/TLS Termination

### Generate Self-Signed Certificate

```bash
# Generate private key and certificate (valid 365 days)
openssl req -x509 -newkey rsa:2048 -keyout /tmp/key.pem -out /tmp/cert.pem -days 365 -nodes
# or use Let's Encrypt with Certbot

# Combine cert and key into single PEM
cat /tmp/cert.pem /tmp/key.pem > /tmp/example.pem
sudo mv /tmp/example.pem /etc/ssl/certs/
sudo chmod 600 /etc/ssl/certs/example.pem
```

### HTTPS Frontend with Certificate

```
frontend web_secure
  bind *:443 ssl crt /etc/ssl/certs/example.pem
  
  # Modern TLS only
  ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets
  
  # HSTS header
  http-response set-header Strict-Transport-Security "max-age=31536000; includeSubDomains" if { ssl_fc }
  
  # Remove Server header
  http-response del-header Server
  
  default_backend web_servers

# Decrypt and re-encrypt to backend (if needed)
backend web_servers
  balance roundrobin
  server web1 10.0.10.10:443 ssl verify none check  # Backend HTTPS
```

### Multiple Certificates (SNI)

```
frontend web_secure
  bind *:443 ssl \
    crt /etc/ssl/certs/example.pem \
    crt /etc/ssl/certs/api.pem \
    crt /etc/ssl/certs/app.pem
  
  # SNI-based routing
  acl api_sni req.ssl_sni -i api.example.com
  acl app_sni req.ssl_sni -i app.example.com
  
  use_backend api_servers if api_sni
  use_backend app_servers if app_sni
  default_backend web_servers
```

---

## Advanced Routing

### ACL-Based Routing

```
frontend web_in
  bind *:80
  
  # Define ACLs (Access Control Lists)
  acl is_get method GET
  acl is_post method POST
  acl path_api path_beg /api/
  acl path_static path_beg /static/
  acl path_admin path_beg /admin/
  acl header_json hdr(Content-Type) -i application/json
  acl src_internal src 10.0.0.0/8
  
  # Route based on ACLs
  use_backend api_servers if path_api
  use_backend static_servers if path_static
  use_backend admin_servers if path_admin src_internal
  use_backend json_api if header_json is_post
  
  default_backend web_servers
```

### URL/Domain Rewriting

```
frontend web_in
  bind *:80
  
  # Rewrite request path
  http-request set-path /api/v2%[path] if { path_beg /api/ }
  
  # Remove query parameter
  http-request set-query %[query,regsub(/token=[^&]*&?,/)] if { query_match /token=/i }
  
  # Add/modify header
  http-request add-header X-Forwarded-For %[src]
  http-request add-header X-Real-IP %[src]
  
  default_backend web_servers
```

### Content-Based Routing (A/B Testing)

```
backend web_servers
  balance roundrobin
  
  # Route 10% traffic to canary version
  acl is_canary rand(0..100) lt 10
  
  server web_stable 10.0.10.10:80 check if !is_canary
  server web_stable 10.0.10.11:80 check if !is_canary
  server web_canary 10.0.10.20:80 check if is_canary
```

---

## Health Checks

### HTTP Health Checks

```
backend web_servers
  # HTTP health check
  option httpchk GET /health HTTP/1.1\r\nHost:\ example.com
  
  # Check parameters: interval 2s, 3 failures to down, 2 successes to up
  server web1 10.0.10.10:80 check inter 2000 fall 3 rise 2
  server web2 10.0.10.11:80 check inter 2000 fall 3 rise 2
  
  # Drain a server (stop new connections but keep existing)
  server web3 10.0.10.12:80 check weight 0
```

### TCP Health Checks

```
backend mysql_servers
  mode tcp
  
  # TCP connect check on port 3306
  option tcp-check
  tcp-check connect port 3306
  
  server db1 10.0.30.10:3306 check inter 3000
  server db2 10.0.30.11:3306 check inter 3000
```

### SSL Health Checks

```
backend api_servers
  option httpchk GET /status HTTP/1.1\r\nHost:\ api.example.com
  
  # Check backend via HTTPS
  server api1 10.0.10.20:443 ssl verify none check
  server api2 10.0.10.21:443 ssl verify none check
```

---

## Session Persistence

### Cookie-Based Persistence

```
backend app_servers
  # Insert cookie with server ID
  cookie SERVERID insert indirect nocache
  
  server app1 10.0.10.20:8080 check cookie app1
  server app2 10.0.10.21:8080 check cookie app2
  server app3 10.0.10.22:8080 check cookie app3
```

### Source IP Affinity

```
backend web_servers
  # Hash on client source IP
  balance source
  
  server web1 10.0.10.10:80 check
  server web2 10.0.10.11:80 check
  server web3 10.0.10.12:80 check
```

### Stick Table (Advanced Persistence)

```
backend app_servers
  # Track client IP with 30-minute expiration
  stick-table type ip size 100k expire 30m
  stick on src
  
  server app1 10.0.10.20:8080 check
  server app2 10.0.10.21:8080 check
```

---

## Monitoring & Stats

### Enable Stats Page

```
global
  stats socket /run/haproxy/admin.sock mode 660 level admin
  stats timeout 30s

# Stats UI frontend
frontend stats
  bind *:8404
  stats enable
  stats uri /stats
  stats refresh 30s
  stats admin if TRUE
```

Access at: `http://localhost:8404/stats`

### Export Metrics for Prometheus

```
global
  stats socket /run/haproxy/admin.sock mode 660 level admin

# HAProxy Prometheus exporter (separate service)
# docker run -d -p 8404:8404 prometheuscommunity/haproxy-exporter:latest \
#   --haproxy.scrape-uri=http://10.0.10.100:8404/stats;csv
```

Add to Prometheus `prometheus.yml`:
```yaml
scrape_configs:
  - job_name: 'haproxy'
    static_configs:
      - targets: ['10.0.10.100:8404']
```

### Socket Commands

```bash
# Access admin socket
sudo socat readline /run/haproxy/admin.sock

# Show stats
show stat

# Disable a backend server
disable server web_servers/web1

# Re-enable server
enable server web_servers/web1

# Show current connections
show sess

# Show frontend/backend info
show info
```

---

## High Availability

### HAProxy with Keepalived (CARP Alternative for Linux)

Install:
```bash
sudo apt install -y keepalived
```

Configure `/etc/keepalived/keepalived.conf`:
```
vrrp_script check_haproxy {
  script "/usr/local/bin/check_haproxy.sh"
  interval 2
  weight -20
}

vrrp_instance haproxy_group {
  state MASTER
  interface eth0
  virtual_router_id 100
  priority 150
  
  # Virtual IP shared between HAProxy nodes
  virtual_ipaddress {
    10.0.10.254/24
  }
  
  track_script {
    check_haproxy
  }
}
```

Create `/usr/local/bin/check_haproxy.sh`:
```bash
#!/bin/bash
if systemctl is-active --quiet haproxy; then
  exit 0
else
  exit 1
fi
```

Enable:
```bash
sudo systemctl enable keepalived
sudo systemctl start keepalived
```

### HAProxy Active-Active Clustering

Multiple HAProxy instances behind load balancer:
```
Client Traffic
    |
    v
Keepalived Virtual IP (10.0.10.254)
    |
    +---> HAProxy-1 (10.0.10.100)
    |
    +---> HAProxy-2 (10.0.10.101)
    |
    +---> HAProxy-3 (10.0.10.102)
    
    v
Backend Servers (10.0.10.10-12)
```

---

## Troubleshooting

### Connection Refused

```bash
# Check if HAProxy is listening
sudo netstat -tulpn | grep haproxy

# Test backend connectivity
nc -zv 10.0.10.10 80

# Check backend health via HAProxy socket
echo "show stat" | sudo socat - /run/haproxy/admin.sock | grep web_servers
```

### High Connection Errors

```bash
# Check HAProxy logs
sudo tail -f /var/log/haproxy.log | grep -i error

# Monitor active connections
echo "show info" | sudo socat - /run/haproxy/admin.sock | grep Conn

# Check system limits
ulimit -n  # Open files
ulimit -a  # All limits

# Increase limits if needed
echo "* soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "* hard nofile 65536" | sudo tee -a /etc/security/limits.conf
```

### Slow Backend Responses

```bash
# Check backend response time in stats
# Stats page shows Ttime (total time), Tw (wait time)

# Monitor with:
echo "show stat" | sudo socat - /run/haproxy/admin.sock | \
  awk -F, '{print $1, $2, $40, $41}' | column -t

# Slow query? Check if backend server is overloaded
ssh 10.0.10.10 'top -b -n 1 | head -n 20'
```

### Certificate Errors (HTTPS)

```bash
# Verify certificate
openssl x509 -in /etc/ssl/certs/example.pem -text -noout | grep -E "Subject:|Issuer:|Not"

# Test HTTPS connection
curl -v https://localhost --insecure 2>&1 | grep -E "SSL|certificate"

# Check certificate expiry
echo | openssl s_client -servername example.com -connect localhost:443 2>/dev/null | \
  openssl x509 -noout -dates
```

### Stats Page Not Loading

```bash
# Verify stats frontend is enabled
grep -n "stats enable" /etc/haproxy/haproxy.cfg

# Check socket permissions
sudo ls -la /run/haproxy/admin.sock

# Test socket
echo "show stat" | sudo socat - /run/haproxy/admin.sock | head -n 5
```

---

## Best Practices

1. **Health Checks**: Enable on all backends; adjust intervals based on sensitivity
2. **Timeouts**: Set appropriately (connect 5s, client 50s, server 50s) to avoid hanging connections
3. **Monitoring**: Export metrics to Prometheus; alert on backend down, high error rates
4. **Security**: Always use HTTPS in production; remove Server headers; validate ACLs
5. **Logging**: Send to syslog for central logging; monitor for backend errors
6. **Graceful Reload**: Use `systemctl reload` to avoid dropping connections
7. **Capacity Planning**: Monitor connections per backend; add servers before hitting limits
8. **Session Persistence**: Choose based on application needs (cookie vs source IP)
9. **A/B Testing**: Use consistent hashing when adding/removing backends to minimize disruption
10. **Documentation**: Keep ACLs and routing rules documented; complex rules can be hard to debug

---

*HAProxy load balancing and high-performance routing guide for production environments.*
