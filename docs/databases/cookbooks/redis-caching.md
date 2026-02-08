# Redis - In-Memory Caching & Data Store

Complete guide to Redis for caching, sessions, real-time analytics, and high-performance data operations.

## Table of Contents
1. [Installation](#installation)
2. [Data Types & Operations](#data-types--operations)
3. [Persistence](#persistence)
4. [Replication](#replication)
5. [Cluster Setup](#cluster-setup)
6. [Pub/Sub & Streams](#pubsub--streams)
7. [Lua Scripting](#lua-scripting)
8. [Performance Tuning](#performance-tuning)
9. [Monitoring](#monitoring)
10. [Troubleshooting](#troubleshooting)

---

## Installation

### Ubuntu/Debian

```bash
sudo apt update
sudo apt install -y redis-server redis-tools

# Verify
redis-cli --version
redis-server --version

# Enable and start
sudo systemctl enable redis-server
sudo systemctl start redis-server

# Test connection
redis-cli ping
# Output: PONG
```

### Configuration

Main config: `/etc/redis/redis.conf`

```bash
# Bind to network interface
sudo sed -i 's/^bind 127.0.0.1/bind 10.0.30.10/' /etc/redis/redis.conf

# Require password
echo "requirepass redispass123" | sudo tee -a /etc/redis/redis.conf

# Set max memory (4GB)
echo "maxmemory 4gb" | sudo tee -a /etc/redis/redis.conf

# Eviction policy when max memory reached
echo "maxmemory-policy allkeys-lru" | sudo tee -a /etc/redis/redis.conf

# Reload config
sudo systemctl restart redis-server
```

---

## Data Types & Operations

### Strings (Simple Cache)

```bash
redis-cli

# Set key-value with expiration
SET cache:user:123 '{"id":123,"name":"alice","email":"alice@example.com"}' EX 3600

# Get value
GET cache:user:123

# Increment counter
INCR stats:pageviews:today
INCRBY stats:pageviews:today 5

# Decrement
DECR stats:pageviews:today

# Set if not exists
SETNX cache:session:abc123 "session_data"

# Get and set (returns old value)
GETSET cache:current_version "v2.1.0"
```

### Hashes (Structured Data)

```bash
# Set multiple fields in hash
HSET user:123 id 123 name alice email alice@example.com role admin

# Get single field
HGET user:123 name
# Output: "alice"

# Get all fields
HGETALL user:123
# Output:
# 1) "id"
# 2) "123"
# 3) "name"
# 4) "alice"
# 5) "email"
# 6) "alice@example.com"
# 7) "role"
# 8) "admin"

# Check if field exists
HEXISTS user:123 email

# Increment field value
HINCRBY stats:server:web1 connections 5
```

### Lists (Queues)

```bash
# Push to list (right = tail, left = head)
RPUSH queue:jobs job1 job2 job3
LPUSH queue:jobs job0

# Get list length
LLEN queue:jobs

# Get range
LRANGE queue:jobs 0 -1

# Pop from list (FIFO with LPOP, LIFO with RPOP)
LPOP queue:jobs

# Blocking pop (wait up to 5 seconds for item)
BLPOP queue:jobs 5

# Push to one list, pop from another (atomically)
RPOPLPUSH queue:incoming queue:processing
```

### Sets (Unique Values)

```bash
# Add members to set
SADD users:online alice bob charlie

# Check membership
SISMEMBER users:online alice

# Get all members
SMEMBERS users:online

# Set operations
SADD users:group:admin alice bob
SADD users:group:mods charlie david

# Intersection (who is both admin AND mod?)
SINTER users:group:admin users:group:mods

# Union (who is admin OR mod?)
SUNION users:group:admin users:group:mods

# Difference (admin but NOT mod?)
SDIFF users:group:admin users:group:mods
```

### Sorted Sets (Leaderboards/Rankings)

```bash
# Add members with scores
ZADD leaderboard 100 player1 250 player2 150 player3 300 player4

# Get by rank (highest score)
ZREVRANGE leaderboard 0 -1 WITHSCORES

# Get players between scores 100-200
ZRANGEBYSCORE leaderboard 100 200 WITHSCORES

# Increment player score
ZINCRBY leaderboard 50 player1

# Get rank of player
ZREVRANK leaderboard player2
# Output: 0 (first place)
```

---

## Persistence

### RDB (Snapshots)

Periodic snapshots to disk. Fast but loses data since last snapshot.

```bash
# Edit /etc/redis/redis.conf
save 900 1        # Save if 1 key changed in 900 seconds (15 min)
save 300 10       # Save if 10 keys changed in 300 seconds (5 min)
save 60 10000     # Save if 10000 keys changed in 60 seconds

# Disable auto-save
save ""

# Manual snapshot
redis-cli BGSAVE
```

### AOF (Append-Only File)

Log every write operation. Slower but safer. Can replay all operations on startup.

```bash
# Edit /etc/redis/redis.conf
appendonly yes
appendfilename "appendonly.aof"

# Fsync strategy
appendfsync everysec    # Safe + fast (default)
appendfsync always      # Slowest but safest (every command)
appendfsync no          # Fastest but unsafe (let OS handle)

# Rewrite AOF when it grows
auto-aof-rewrite-percentage 100    # Rewrite when 100% larger
auto-aof-rewrite-min-size 64mb     # Don't rewrite if smaller than 64MB
```

### Hybrid Persistence

Combine RDB + AOF:
```bash
aof-use-rdb-preamble yes    # Use RDB for snapshot, AOF for changes
```

---

## Replication

### Master-Slave Setup

**Master (10.0.30.10)**:
```bash
# Edit /etc/redis/redis.conf
bind 10.0.30.10
port 6379
requirepass master_password

sudo systemctl restart redis-server
```

**Slave (10.0.30.11)**:
```bash
# Edit /etc/redis/redis.conf
bind 10.0.30.11
port 6379
requirepass slave_password

# Configure replication from master
replicaof 10.0.30.10 6379
masterauth master_password

# Slave is read-only
replica-read-only yes

sudo systemctl restart redis-server
```

Verify:
```bash
# On master
redis-cli -a master_password INFO replication
# Output shows: role:master, connected_slaves:1

# On slave
redis-cli -a slave_password INFO replication
# Output shows: role:slave, master_host:10.0.30.10, master_port:6379
```

### Replication Lag Monitoring

```bash
# Monitor on slave
redis-cli -a slave_password INFO replication | grep offset
# master_repl_offset: (master writes)
# slave_repl_offset:  (slave reads)
# Difference = lag in bytes (lower is better)
```

---

## Cluster Setup

### 6-Node Cluster (3 Masters + 3 Slaves)

```bash
# Install redis-tools for cluster creation
sudo apt install -y redis-tools

# Create config for each node (6379-6384)
for i in {0..5}; do
  port=$((6379 + i))
  sudo mkdir -p /etc/redis/cluster
  
  cat > /tmp/redis-$port.conf <<EOF
port $port
bind 10.0.30.10
cluster-enabled yes
cluster-config-file /var/lib/redis/nodes-$port.conf
cluster-node-timeout 5000
appendonly yes
EOF

  sudo cp /tmp/redis-$port.conf /etc/redis/cluster/
done

# Start Redis cluster nodes (6379-6384)
for i in {0..5}; do
  port=$((6379 + i))
  redis-server /etc/redis/cluster/redis-$port.conf --daemonize yes
done

# Create cluster (option 1: automatic, 3 masters and 3 replicas)
redis-cli --cluster create \
  10.0.30.10:6379 \
  10.0.30.10:6380 \
  10.0.30.10:6381 \
  10.0.30.10:6382 \
  10.0.30.10:6383 \
  10.0.30.10:6384 \
  --cluster-replicas 1

# Verify cluster
redis-cli -p 6379 CLUSTER INFO
redis-cli -p 6379 CLUSTER NODES
```

### Cluster Slots Distribution

```
Cluster splits 16384 slots across 3 masters:
Master 1 (node-1): slots 0-5460
Master 2 (node-2): slots 5461-10922
Master 3 (node-3): slots 10923-16383

When client writes:
1. Key is hashed: CRC16(key) % 16384 = slot
2. Find which master owns that slot
3. Write to that master
4. Master replicates to its slave
```

### Client Connection to Cluster

```bash
# Connect to any cluster node (redirects automatically)
redis-cli -h 10.0.30.10 -p 6379 -c

# Write a key
SET foo bar
# Internally hashes to slot X, routed to master Y

# Get the key
GET foo
```

---

## Pub/Sub & Streams

### Publish/Subscribe (Real-Time Messaging)

```bash
# Terminal 1 (Subscriber)
redis-cli SUBSCRIBE notifications alerts
# Output: subscription messages as they arrive

# Terminal 2 (Publisher)
redis-cli PUBLISH notifications "Server disk full"
redis-cli PUBLISH alerts "CPU > 80%"
```

### Redis Streams (Persistent Pub/Sub)

Messages are stored, can be replayed (like Kafka):

```bash
# Add message to stream
XADD events:user-actions * user:123 action:login timestamp:2025-02-07T10:00:00Z

# Read all messages in stream
XRANGE events:user-actions - +

# Consumer group (for processing)
XGROUP CREATE events:user-actions consumer_group $

# Read as consumer (blocking)
XREADGROUP GROUP consumer_group consumer1 COUNT 1 STREAMS events:user-actions >

# Acknowledge message (won't be re-delivered)
XACK events:user-actions consumer_group message_id
```

---

## Lua Scripting

Execute atomically on server (no race conditions).

### Simple Script

```bash
redis-cli EVAL "
  local val = redis.call('GET', KEYS[1])
  if val == ARGV[1] then
    redis.call('SET', KEYS[1], ARGV[2])
    return 'updated'
  else
    return 'value mismatch'
  end
" 1 mykey oldvalue newvalue
```

### Rate Limiting Script

```bash
redis-cli EVAL "
  local key = KEYS[1]
  local limit = tonumber(ARGV[1])
  local ttl = tonumber(ARGV[2])
  
  local count = redis.call('GET', key)
  if not count then
    redis.call('SET', key, 1, 'EX', ttl)
    return 1
  end
  
  if tonumber(count) < limit then
    redis.call('INCR', key)
    return tonumber(count) + 1
  else
    return -1  -- rate limit exceeded
  end
" 1 ratelimit:user:123 10 60
# Returns: 1, 2, 3, ... 10 (allowed)
# Returns: -1 (rejected, retry after 60 seconds)
```

### Distributed Lock Script

```bash
redis-cli EVAL "
  local key = KEYS[1]
  local token = ARGV[1]
  local ttl = tonumber(ARGV[2])
  
  if redis.call('GET', key) == token then
    redis.call('DEL', key)
    return 1
  else
    return 0
  end
" 1 lock:resource:123 my_unique_token 30
```

---

## Performance Tuning

### Memory Optimization

```bash
# Check memory usage
redis-cli INFO memory

# Configure eviction policy (what to delete when max_memory reached)
maxmemory-policy allkeys-lru      # Remove least recently used key
maxmemory-policy volatile-lru     # Remove LRU key with TTL
maxmemory-policy allkeys-random   # Random removal
maxmemory-policy volatile-ttl     # Remove key with shortest TTL
maxmemory-policy noeviction       # Reject writes (default, safer)

# Estimate key memory usage
redis-cli DEBUG OBJECT user:123
# Output: encoding:raw, serializedlength:..., lru_seconds_idle:...
```

### Pipeline (Batch Commands)

Reduce network overhead:

```bash
# Inefficient: 1000 round trips
redis-cli SET key1 val1
redis-cli SET key2 val2
... (repeat 1000x)

# Efficient: single round trip
cat <<EOF | redis-cli --pipe
SET key1 val1
SET key2 val2
...
EOF

# Or in application code:
pipeline = redis.pipeline()
for i in range(1000):
  pipeline.set(f'key{i}', f'val{i}')
pipeline.execute()
```

### Compression

```bash
# For large values, compress before storing
# In Python:
import zlib
compressed = zlib.compress(json.dumps(large_object).encode())
redis.set('key', compressed)

# On retrieval:
data = redis.get('key')
decompressed = json.loads(zlib.decompress(data).decode())
```

---

## Monitoring

### Redis INFO Command

```bash
redis-cli INFO

# Key sections:
# - Server: version, config, uptime
# - Clients: connected clients, blocked clients
# - Memory: used_memory, peak_memory, eviction_policy
# - Stats: total_connections_received, total_commands_processed
# - Replication: role, connected_slaves, repl_offset
# - CPU: used_cpu_sys, used_cpu_user
# - Cluster: cluster_state, cluster_slots_assigned
```

### Prometheus Exporter

```bash
# Install redis_exporter
docker run -d -p 9121:9121 oliver006/redis_exporter:latest \
  --redis.addr=redis://10.0.30.10:6379

# Add to prometheus.yml
scrape_configs:
  - job_name: 'redis'
    static_configs:
      - targets: ['10.0.30.10:9121']
```

### Monitor Real-Time Commands

```bash
redis-cli MONITOR

# Output shows every command executed (watch for slow operations)
1000.01 [0 10.0.30.100:12345] "GET" "cache:user:123"
1000.02 [0 10.0.30.101:12346] "SET" "cache:session:abc" "session_data"
```

### Slow Query Log

```bash
# Set threshold (microseconds): queries > 10ms logged
CONFIG SET slowlog-log-slower-than 10000

# Get slow queries
SLOWLOG GET 10

# Output:
# ID, timestamp, duration_us, command, client_ip

# Clear log
SLOWLOG RESET
```

---

## Troubleshooting

### High Memory Usage

```bash
# Check memory breakdown
redis-cli INFO memory | grep -E "used|peak|max"

# Identify large keys
redis-cli --bigkeys

# Check eviction stats
redis-cli INFO stats | grep evicted

# Solutions:
# 1. Increase maxmemory
# 2. Change eviction policy (LRU better than FIFO)
# 3. Add replicas for read scaling
# 4. Compress large values
# 5. Use shorter key names
```

### Replication Lag

```bash
# Check on slave
redis-cli INFO replication | grep -E "offset|backlog"

# If master_repl_offset >> slave_repl_offset:
# - Network is slow
# - Master is writing too fast
# - Slave CPU is maxed

# Improve:
# 1. Increase slave buffer (repl-backlog-size 16mb default)
# 2. Reduce master write load
# 3. Upgrade slave hardware
```

### Connection Refused

```bash
# Check if Redis is listening
sudo netstat -tulpn | grep 6379

# Check ACL/auth
redis-cli -p 6379 PING
# If error: AUTH required

redis-cli -a password PING

# Check firewall
sudo ufw allow 6379/tcp
```

### Cluster Slot Distribution Issues

```bash
# Check cluster health
redis-cli -p 6379 CLUSTER INFO

# Rebalance slots if uneven
redis-cli --cluster rebalance 10.0.30.10:6379

# Fix unhealthy nodes
redis-cli --cluster fix 10.0.30.10:6379
```

---

## Best Practices

1. **Persistence**: Use AOF for safety; RDB + AOF hybrid for balance
2. **Replication**: Always replicate; enables high availability
3. **Expiration**: Set TTL on all cache keys to avoid unbounded growth
4. **Monitoring**: Track memory, hit/miss ratio, latency, slow queries
5. **Security**: Require authentication; restrict network access; disable FLUSHDB/FLUSHALL
6. **Cluster**: Use cluster for horizontal scaling; understand slot distribution
7. **Pipeline**: Batch commands to reduce latency for bulk operations
8. **Connection Pooling**: Reuse connections; don't create new for each request
9. **Key Naming**: Use namespacing (e.g., `cache:`, `session:`, `queue:`) for organization
10. **Backup**: Regular snapshots for disaster recovery

---

*Redis in-memory caching and data store guide for high-performance applications.*
