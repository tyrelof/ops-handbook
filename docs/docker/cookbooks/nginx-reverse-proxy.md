# Cookbook: Nginx Reverse Proxy & Load Balancing

> [!NOTE]
> Deploy and configure Nginx as a reverse proxy for containerized workloads.
> Includes: Upstream routing, load balancing, SSL termination, and health checks.

## 1. Basic Reverse Proxy Setup

### Dockerfile
```dockerfile
FROM nginx:latest
COPY nginx.conf /etc/nginx/nginx.conf
COPY default.conf /etc/nginx/conf.d/default.conf
EXPOSE 80 443
CMD ["nginx", "-g", "daemon off;"]
```

### nginx.conf
```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 100M;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1000;
    gzip_types text/plain text/css text/xml text/javascript 
               application/x-javascript application/xml+rss 
               application/json application/javascript;

    include /etc/nginx/conf.d/*.conf;
}
```

### default.conf (Routing)
```nginx
upstream backend {
    # For Kubernetes: use service DNS name
    server api-service:3000;
    server api-service-2:3000;
    
    # Load balancing method (default: round-robin)
    # least_conn;  # Send to least busy server
    # ip_hash;     # Same IP always goes to same server
}

server {
    listen 80;
    server_name example.com www.example.com;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffering
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }

    # Health check endpoint (app returns 200)
    location /health {
        access_log off;
        proxy_pass http://backend;
    }
}
```

## 2. Load Balancing Methods

```nginx
upstream backend {
    # 1. Round-robin (default) - equal distribution
    server server1:3000;
    server server2:3000;
    server server3:3000;

    # 2. Least connections - send to least busy
    least_conn;

    # 3. IP Hash - same IP always to same server
    ip_hash;
    server server1:3000;
    server server2:3000;

    # 4. Weighted - distribute by weight
    server server1:3000 weight=5;  # Gets 5x more traffic
    server server2:3000 weight=3;
    server server3:3000 weight=1;

    # 5. Hash (consistent hashing)
    hash $request_uri consistent;

    # 6. Random
    random;

    # Keepalive connections to upstream
    keepalive 32;
}
```

## 3. Health Checks (Active)

```nginx
upstream backend {
    server api-1:3000;
    server api-2:3000;

    zone backend_zone 64k;  # Shared memory zone

    # Active health checks (Nginx Plus feature)
    # For open-source, use passive or external monitoring
}

# Passive health checks (detect failures automatically)
server {
    location / {
        proxy_pass http://backend;
        
        # Mark server down after 3 failures in 30s
        proxy_next_upstream error timeout http_502 http_503;
        proxy_next_upstream_tries 2;
        proxy_next_upstream_timeout 10s;
    }
}
```

## 4. SSL/TLS Termination

```nginx
server {
    listen 80;
    server_name example.com;
    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/nginx/certs/cert.pem;
    ssl_certificate_key /etc/nginx/certs/key.pem;

    # Strong SSL config
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # HSTS (force HTTPS)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        proxy_pass http://backend;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

## 5. Docker Compose Example

```yaml
version: '3.9'

services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./conf.d:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - api-1
      - api-2
    networks:
      - backend

  api-1:
    image: my-api:latest
    environment:
      - NODE_ENV=production
    networks:
      - backend
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  api-2:
    image: my-api:latest
    environment:
      - NODE_ENV=production
    networks:
      - backend
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  backend:
    driver: bridge
```

## 6. Kubernetes Deployment

### Nginx ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: default
data:
  nginx.conf: |
    user nginx;
    worker_processes auto;
    events { worker_connections 1024; }
    http {
      include /etc/nginx/mime.types;
      upstream backend {
        server api-service.default.svc.cluster.local:3000;
      }
      server {
        listen 80;
        location / {
          proxy_pass http://backend;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
      }
    }
```

### Nginx Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: config
          mountPath: /etc/nginx
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
      volumes:
      - name: config
        configMap:
          name: nginx-config
```

### Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: https
    port: 443
    targetPort: 443
```

## 7. Monitoring & Logging

### Accessing Logs
```bash
# In Kubernetes
kubectl logs -l app=nginx -n default -f

# In Docker
docker logs <container-id> -f
```

### Key Metrics to Monitor
```nginx
# Enable metrics collection (use Prometheus module in Nginx Plus or scrape logs)
server {
    location /nginx-status {
        stub_status on;
        access_log off;
        allow 127.0.0.1;
        allow 10.0.0.0/8;  # Allow from cluster
        deny all;
    }
}
```

Get metrics:
```bash
curl http://localhost/nginx-status

# Output:
# Active connections: 42
# server accepts handled requests
#  45680 45680 130356
# Reading: 0 Writing: 6 Waiting: 36
```

### Key Metrics
- **Active connections**: Current open connections
- **Accepts**: Total connections since startup
- **Handled**: Connections successfully handled
- **Requests**: Total requests processed
- **Reading**: Nginx reading request headers
- **Writing**: Nginx writing response to client
- **Waiting**: Idle keepalive connections

## 8. Common Patterns

### Caching
```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m;

server {
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        proxy_cache my_cache;
        proxy_cache_valid 200 1d;
        proxy_cache_key "$scheme$request_method$host$request_uri";
        add_header X-Cache-Status $upstream_cache_status;
    }
}
```

### Rate Limiting
```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

server {
    location /api/ {
        limit_req zone=api_limit burst=20 nodelay;
        proxy_pass http://backend;
    }
}
```

### URL Rewriting
```nginx
server {
    rewrite ^/old/(.*) /new/$1 permanent;
    
    location /api/v1/ {
        rewrite ^/api/v1/(.*)$ /api/v2/$1 break;
        proxy_pass http://backend;
    }
}
```

### Conditional Routing
```nginx
upstream mobile_backend {
    server mobile-api:3000;
}

upstream desktop_backend {
    server desktop-api:3000;
}

server {
    location / {
        if ($http_user_agent ~* "(mobile|iphone|android)" ) {
            proxy_pass http://mobile_backend;
        }
        proxy_pass http://desktop_backend;
    }
}
```

## 9. Troubleshooting

### Check Nginx syntax
```bash
docker exec <nginx-container> nginx -t
```

### Monitor in real-time
```bash
# Watch active connections
watch -n 1 'curl -s http://localhost/nginx-status | tail -5'

# Tail logs with timestamps
docker logs -f <nginx-container> | while IFS= read -r line; do
    echo "[$(date '+%T')] $line"
done
```

### Common issues
- **502 Bad Gateway**: Upstream servers unavailable
  ```bash
  # Check upstream connectivity
  docker exec <nginx> curl -i http://api-service:3000/health
  ```

- **Connection refused**: Nginx can't reach upstream
  ```bash
  # Verify DNS resolution
  docker exec <nginx> nslookup api-service
  ```

- **Slow requests**: Check buffer settings, proxy timeouts, upstream slowness
  ```nginx
  proxy_buffer_size 32k;
  proxy_buffers 8 32k;
  ```

---

**Next Steps**:
- Set up SSL certificates (Let's Encrypt)
- Add rate limiting for public APIs
- Configure monitoring/alerting
- Test failover scenarios
