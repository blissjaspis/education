# Redis: From Beginner to Expert

## Table of Contents

1. [Beginner Level](#beginner-level)
   - [What is Redis?](#what-is-redis)
   - [Installation](#installation)
   - [Basic Commands](#basic-commands)
   - [Data Types](#data-types)

2. [Intermediate Level](#intermediate-level)
   - [Advanced Data Structures](#advanced-data-structures)
   - [Persistence](#persistence)
   - [Pub/Sub](#pubsub)
   - [Transactions](#transactions)
   - [Pipelining](#pipelining)

3. [Advanced Level](#advanced-level)
   - [Replication](#replication)
   - [Clustering](#clustering)
   - [Lua Scripting](#lua-scripting)
   - [Memory Management](#memory-management)
   - [Performance Optimization](#performance-optimization)

4. [Expert Level](#expert-level)
   - [Redis Modules](#redis-modules)
   - [Monitoring and Debugging](#monitoring-and-debugging)
   - [Security](#security)
   - [High Availability](#high-availability)
   - [Production Best Practices](#production-best-practices)

---

## Beginner Level

### What is Redis?

Redis (Remote Dictionary Server) is an open-source, in-memory data structure store that can be used as:
- **Database**: Store and retrieve data
- **Cache**: Fast temporary storage
- **Message Broker**: Pub/Sub messaging
- **Session Store**: Web session management

#### Key Features:
- **In-memory storage** for ultra-fast performance
- **Persistence** options to disk
- **Atomic operations** for data consistency
- **Built-in replication** for high availability
- **Rich data types** beyond simple key-value pairs

### Installation

#### Using Package Managers:

**macOS (Homebrew):**
```bash
brew install redis
brew services start redis
```

**Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install redis-server
sudo systemctl start redis-server
```

**CentOS/RHEL:**
```bash
sudo yum install redis
sudo systemctl start redis
```

#### Using Docker:
```bash
docker run -d --name redis-server -p 6379:6379 redis:latest
```

#### Verify Installation:
```bash
redis-cli ping
# Expected output: PONG
```

### Basic Commands

#### Connection and Server Info:
```bash
# Connect to Redis CLI
redis-cli

# Check if Redis is running
PING

# Get server information
INFO

# Select database (0-15, default is 0)
SELECT 1
```

#### Basic Key Operations:
```bash
# Set a key-value pair
SET mykey "Hello Redis"

# Get value by key
GET mykey

# Check if key exists
EXISTS mykey

# Delete a key
DEL mykey

# Set key with expiration (in seconds)
SETEX mykey 60 "Expires in 60 seconds"

# Set expiration on existing key
EXPIRE mykey 30

# Get time to live
TTL mykey

# Get all keys (use carefully in production)
KEYS *

# Get keys matching pattern
KEYS user:*
```

### Data Types

#### 1. Strings
```bash
# Basic string operations
SET name "John Doe"
GET name

# Append to string
APPEND name " Jr."
GET name  # "John Doe Jr."

# String length
STRLEN name

# Increment/Decrement numbers
SET counter 10
INCR counter     # 11
INCRBY counter 5 # 16
DECR counter     # 15
DECRBY counter 3 # 12
```

#### 2. Lists
```bash
# Add elements to list
LPUSH mylist "item1"  # Add to left
RPUSH mylist "item2"  # Add to right

# Get list elements
LRANGE mylist 0 -1    # Get all elements

# Remove elements
LPOP mylist           # Remove from left
RPOP mylist           # Remove from right

# Get list length
LLEN mylist

# Get element by index
LINDEX mylist 0
```

#### 3. Sets
```bash
# Add members to set
SADD myset "member1" "member2" "member3"

# Get all members
SMEMBERS myset

# Check membership
SISMEMBER myset "member1"

# Remove member
SREM myset "member2"

# Get set size
SCARD myset

# Set operations
SADD set1 "a" "b" "c"
SADD set2 "b" "c" "d"
SINTER set1 set2      # Intersection
SUNION set1 set2      # Union
SDIFF set1 set2       # Difference
```

#### 4. Hashes
```bash
# Set hash fields
HSET user:1 name "John" age 30 email "john@example.com"

# Get hash field
HGET user:1 name

# Get all fields and values
HGETALL user:1

# Get multiple fields
HMGET user:1 name age

# Check if field exists
HEXISTS user:1 name

# Delete field
HDEL user:1 email

# Get all field names
HKEYS user:1

# Get all values
HVALS user:1
```

#### 5. Sorted Sets
```bash
# Add members with scores
ZADD leaderboard 100 "player1" 200 "player2" 150 "player3"

# Get members by rank (0-based)
ZRANGE leaderboard 0 -1

# Get members with scores
ZRANGE leaderboard 0 -1 WITHSCORES

# Get members by score range
ZRANGEBYSCORE leaderboard 100 200

# Get member rank
ZRANK leaderboard "player1"

# Get member score
ZSCORE leaderboard "player2"

# Remove member
ZREM leaderboard "player1"
```

---

## Intermediate Level

### Advanced Data Structures

#### Bitmaps
```bash
# Set bit
SETBIT user:visits:2023 100 1

# Get bit
GETBIT user:visits:2023 100

# Count set bits
BITCOUNT user:visits:2023

# Bitwise operations
BITOP AND result user:visits:2023 user:visits:2024
```

#### HyperLogLog
```bash
# Add elements
PFADD unique_visitors "user1" "user2" "user3"

# Get approximate count
PFCOUNT unique_visitors

# Merge HyperLogLogs
PFMERGE total_visitors unique_visitors_mobile unique_visitors_web
```

#### Streams
```bash
# Add entry to stream
XADD mystream * field1 value1 field2 value2

# Read from stream
XREAD STREAMS mystream 0

# Read latest entries
XREAD STREAMS mystream $

# Create consumer group
XGROUP CREATE mystream mygroup 0

# Read with consumer group
XREADGROUP GROUP mygroup consumer1 STREAMS mystream >
```

### Persistence

#### RDB (Redis Database Backup)
```bash
# Manual snapshot
BGSAVE

# Check last save time
LASTSAVE

# Configuration in redis.conf:
# save 900 1      # Save if at least 1 key changed in 900 seconds
# save 300 10     # Save if at least 10 keys changed in 300 seconds
# save 60 10000   # Save if at least 10000 keys changed in 60 seconds
```

#### AOF (Append Only File)
```bash
# Enable AOF in redis.conf:
# appendonly yes
# appendfsync everysec  # Options: always, everysec, no

# Manual AOF rewrite
BGREWRITEAOF
```

### Pub/Sub

#### Basic Pub/Sub
```bash
# Subscribe to channel
SUBSCRIBE news

# Publish message
PUBLISH news "Breaking news!"

# Subscribe to pattern
PSUBSCRIBE news.*

# Unsubscribe
UNSUBSCRIBE news
```

#### Advanced Pub/Sub with Streams
```bash
# Create consumer group
XGROUP CREATE notifications processing 0

# Consumer reads messages
XREADGROUP GROUP processing worker1 COUNT 1 STREAMS notifications >

# Acknowledge message processing
XACK notifications processing message_id
```

### Transactions

```bash
# Start transaction
MULTI

# Queue commands
SET key1 "value1"
SET key2 "value2"
INCR counter

# Execute transaction
EXEC

# Discard transaction
DISCARD

# Watch keys for changes
WATCH key1
MULTI
SET key1 "new_value"
EXEC  # Will fail if key1 was modified
```

### Pipelining

```bash
# Using redis-cli pipeline
echo -e "SET key1 value1\nSET key2 value2\nGET key1\nGET key2" | redis-cli --pipe
```

---

## Advanced Level

### Replication

#### Master-Slave Setup

**Master Configuration (redis-master.conf):**
```
bind 127.0.0.1
port 6379
```

**Slave Configuration (redis-slave.conf):**
```
bind 127.0.0.1
port 6380
replicaof 127.0.0.1 6379
```

#### Commands:
```bash
# Check replication info
INFO replication

# Make server a replica
REPLICAOF 127.0.0.1 6379

# Stop replication
REPLICAOF NO ONE
```

### Clustering

#### Cluster Setup
```bash
# Create cluster nodes
mkdir cluster-test
cd cluster-test
mkdir 7000 7001 7002 7003 7004 7005

# Configuration for each node (redis.conf):
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

#### Cluster Commands:
```bash
# Create cluster
redis-cli --cluster create 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 --cluster-replicas 1

# Cluster info
CLUSTER INFO
CLUSTER NODES

# Resharding
redis-cli --cluster reshard 127.0.0.1:7000
```

### Lua Scripting

```bash
# Simple Lua script
EVAL "return redis.call('SET', KEYS[1], ARGV[1])" 1 mykey myvalue

# Conditional increment
EVAL "
local current = redis.call('GET', KEYS[1])
if current == false then
    current = 0
else
    current = tonumber(current)
end
return redis.call('SET', KEYS[1], current + tonumber(ARGV[1]))
" 1 counter 5

# Load script and get SHA
SCRIPT LOAD "return redis.call('GET', KEYS[1])"

# Execute by SHA
EVALSHA sha1_hash 1 mykey
```

### Memory Management

```bash
# Check memory usage
INFO memory

# Memory usage of specific key
MEMORY USAGE mykey

# Set max memory and eviction policy
CONFIG SET maxmemory 100mb
CONFIG SET maxmemory-policy allkeys-lru

# Analyze memory
redis-cli --bigkeys
redis-cli --memkeys
```

### Performance Optimization

#### Monitoring Commands:
```bash
# Monitor all commands
MONITOR

# Get slow queries
SLOWLOG GET 10

# Reset slow log
SLOWLOG RESET

# Real-time stats
redis-cli --stat

# Latency monitoring
redis-cli --latency
redis-cli --latency-history
```

#### Configuration Tuning:
```
# redis.conf optimizations
tcp-keepalive 60
timeout 300
tcp-backlog 511
databases 16
rdbcompression yes
rdbchecksum yes
stop-writes-on-bgsave-error yes
```

---

## Expert Level

### Redis Modules

#### Popular Modules:
- **RedisJSON**: JSON data type support
- **RediSearch**: Full-text search
- **RedisGraph**: Graph database
- **RedisTimeSeries**: Time series data
- **RedisBloom**: Probabilistic data structures

#### Loading Modules:
```bash
# Load module at startup
redis-server --loadmodule /path/to/module.so

# Load module at runtime
MODULE LOAD /path/to/module.so

# List loaded modules
MODULE LIST
```

### Monitoring and Debugging

#### Redis Insight Setup:
```bash
# Docker setup
docker run -d --name redisinsight -p 8001:8001 redislabs/redisinsight:latest
```

#### Custom Monitoring Script:
```bash
#!/bin/bash
# redis-monitor.sh

redis-cli INFO | grep -E "(used_memory_human|connected_clients|total_commands_processed|keyspace_hits|keyspace_misses)"
```

#### Debugging Commands:
```bash
# Debug object
DEBUG OBJECT mykey

# Get object encoding
OBJECT ENCODING mykey

# Memory doctor
MEMORY DOCTOR

# Client list
CLIENT LIST
CLIENT KILL ip:port
```

### Security

#### Authentication:
```
# redis.conf
requirepass your_password
```

#### ACL (Access Control Lists):
```bash
# Create user with limited permissions
ACL SETUSER alice on >password ~cached:* +get +set

# List users
ACL LIST

# Current user info
ACL WHOAMI

# Generate ACL configuration
ACL GENPASS
```

#### Network Security:
```
# redis.conf
bind 127.0.0.1
protected-mode yes
port 0                    # Disable TCP port
unixsocket /tmp/redis.sock
unixsocketperm 700
```

### High Availability

#### Redis Sentinel Setup:

**Sentinel Configuration (sentinel.conf):**
```
port 26379
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
sentinel parallel-syncs mymaster 1
```

#### Sentinel Commands:
```bash
# Get master info
SENTINEL masters

# Get slaves info
SENTINEL slaves mymaster

# Manual failover
SENTINEL failover mymaster
```

### Production Best Practices

#### 1. Configuration Best Practices:
```
# redis.conf production settings
maxmemory-policy allkeys-lru
save 900 1
save 300 10
save 60 10000
appendonly yes
appendfsync everysec
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

#### 2. Monitoring Checklist:
- Memory usage and trends
- Hit/miss ratios
- Connection count
- Slow queries
- Replication lag
- Disk I/O for persistence

#### 3. Backup Strategy:
```bash
#!/bin/bash
# backup-redis.sh

BACKUP_DIR="/backup/redis"
DATE=$(date +%Y%m%d_%H%M%S)

# RDB backup
cp /var/lib/redis/dump.rdb "$BACKUP_DIR/dump_$DATE.rdb"

# AOF backup
cp /var/lib/redis/appendonly.aof "$BACKUP_DIR/appendonly_$DATE.aof"

# Compress and remove old backups
gzip "$BACKUP_DIR/dump_$DATE.rdb"
find "$BACKUP_DIR" -name "*.gz" -mtime +7 -delete
```

#### 4. Performance Testing:
```bash
# Benchmark Redis performance
redis-benchmark -h localhost -p 6379 -c 50 -n 10000

# Test specific operations
redis-benchmark -h localhost -p 6379 -t set,get -n 100000 -q

# Test with pipelining
redis-benchmark -h localhost -p 6379 -t set,get -n 100000 -P 16 -q
```

#### 5. Troubleshooting Commands:
```bash
# Check Redis status
redis-cli ping

# Get configuration
CONFIG GET "*"

# Check replication status
INFO replication

# Monitor memory usage
redis-cli --bigkeys --i 0.01

# Check for blocking operations
CLIENT LIST
```

---

## Common Use Cases and Examples

### 1. Caching
```python
import redis

r = redis.Redis(host='localhost', port=6379, db=0)

def get_user(user_id):
    # Check cache first
    cached_user = r.get(f"user:{user_id}")
    if cached_user:
        return json.loads(cached_user)
    
    # Fetch from database
    user = database.get_user(user_id)
    
    # Cache for 1 hour
    r.setex(f"user:{user_id}", 3600, json.dumps(user))
    return user
```

### 2. Session Store
```python
def create_session(user_id):
    session_id = str(uuid.uuid4())
    session_data = {
        'user_id': user_id,
        'created_at': datetime.now().isoformat()
    }
    
    # Store session for 24 hours
    r.setex(f"session:{session_id}", 86400, json.dumps(session_data))
    return session_id
```

### 3. Rate Limiting
```python
def is_rate_limited(user_id, limit=100, window=3600):
    key = f"rate_limit:{user_id}:{int(time.time() / window)}"
    current = r.incr(key)
    
    if current == 1:
        r.expire(key, window)
    
    return current > limit
```

### 4. Real-time Analytics
```python
def track_page_view(page, user_id):
    # Increment page view counter
    r.incr(f"page_views:{page}:{date.today()}")
    
    # Add to unique visitors set
    r.sadd(f"unique_visitors:{page}:{date.today()}", user_id)
    
    # Update real-time dashboard
    r.publish("analytics", json.dumps({
        'event': 'page_view',
        'page': page,
        'timestamp': datetime.now().isoformat()
    }))
```

---

## Conclusion

This comprehensive guide covers Redis from basic concepts to expert-level implementation. Redis is a powerful tool that can significantly improve application performance when used correctly. Remember to:

1. **Start simple** with basic key-value operations
2. **Understand data structures** and their use cases
3. **Implement proper persistence** and backup strategies
4. **Monitor performance** continuously
5. **Secure your installation** for production use
6. **Plan for scaling** with replication and clustering

For continued learning:
- Practice with Redis CLI
- Experiment with different data structures
- Build real projects using Redis
- Study Redis source code
- Join Redis community forums
- Read Redis documentation and release notes

Happy Redis learning! 🚀
