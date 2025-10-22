# Redis Use Cases & Patterns

This document provides additional examples and patterns for using Redis effectively in different scenarios.

## 🎮 Use Case 1: Gaming Leaderboard

### The Challenge
Gaming leaderboards need to:
- Update player scores in real-time
- Retrieve top players quickly
- Handle millions of score updates per day
- Support ranked queries (What's my rank?)

### Traditional Database Approach

```sql
-- Update score (must read, calculate, write)
UPDATE leaderboard 
SET score = score + 100 
WHERE player_id = 'player_123';

-- Get top 10 players (expensive query)
SELECT player_id, score 
FROM leaderboard 
ORDER BY score DESC 
LIMIT 10;

-- Get player rank (very expensive!)
SELECT COUNT(*) + 1 as rank
FROM leaderboard
WHERE score > (SELECT score FROM leaderboard WHERE player_id = 'player_123');
```

**Problems:**
- Slow for frequent updates
- Sorting millions of rows is expensive
- Rank calculation requires full table scan
- Can't handle real-time traffic

### Redis Sorted Sets Solution

```python
import redis
r = redis.Redis(decode_responses=True)

# Update player score (O(log N) - super fast!)
r.zincrby('leaderboard:global', 100, 'player_123')

# Get top 10 players with scores
top_10 = r.zrevrange('leaderboard:global', 0, 9, withscores=True)
# Returns: [('player_999', 15000), ('player_567', 14800), ...]

# Get player rank (O(log N) - instant!)
rank = r.zrevrank('leaderboard:global', 'player_123')
# Returns: 42 (player is #43)

# Get player score
score = r.zscore('leaderboard:global', 'player_123')

# Get players in rank range (e.g., rank 40-50)
players = r.zrevrange('leaderboard:global', 40, 50, withscores=True)
```

**Benefits:**
- ⚡ Microsecond response times
- 🔥 Handles millions of updates/sec
- 📊 Built-in ranking and sorting
- 💪 Scales horizontally

---

## 📱 Use Case 2: Session Management

### The Challenge
Modern web applications need:
- Fast session storage and retrieval
- Automatic expiration
- Partial updates (update last activity without loading entire session)
- Scale to millions of concurrent users

### Traditional Approach (Database or File-based)

```python
import json
import time

# Load session (read entire file/row)
with open(f'sessions/{session_id}.json', 'r') as f:
    session = json.load(f)

# Update last activity
session['last_activity'] = time.time()

# Write entire session back
with open(f'sessions/{session_id}.json', 'w') as f:
    json.dump(session, f)

# Manual cleanup of expired sessions (cron job)
# Requires scanning all sessions periodically
```

**Problems:**
- Slow disk I/O
- Must write entire session for small updates
- Manual expiration management
- Doesn't scale well

### Redis Solution

```python
import redis
import json
from datetime import timedelta

r = redis.Redis(decode_responses=True)

# Store session with auto-expiration (30 minutes)
session_data = {
    'user_id': 'user_123',
    'username': 'alice',
    'cart': [],
    'preferences': {},
    'last_activity': time.time()
}
r.json().set(f'session:{session_id}', '$', session_data)
r.expire(f'session:{session_id}', timedelta(minutes=30))

# Update just last activity (no need to load entire session!)
r.json().set(f'session:{session_id}', '$.last_activity', time.time())
r.expire(f'session:{session_id}', timedelta(minutes=30))  # Reset TTL

# Add item to cart
r.json().arrappend(f'session:{session_id}', '$.cart', 
    {'product_id': 'prod_789', 'qty': 1})

# Get specific field
user_id = r.json().get(f'session:{session_id}', '$.user_id')

# Sessions auto-expire - no cleanup needed!
```

**Benefits:**
- ⚡ In-memory speed
- 🔄 Automatic expiration
- 📦 Partial updates
- 🚀 Scales to millions of sessions

---

## 🛒 Use Case 3: Real-Time Inventory Management

### The Challenge
E-commerce platforms need:
- Prevent overselling (race conditions)
- Real-time inventory updates
- Fast stock checks
- Handle concurrent purchases

### Traditional Database Approach

```python
# Check and decrement stock (race condition risk!)
cursor.execute("SELECT stock FROM products WHERE id = ?", (product_id,))
stock = cursor.fetchone()[0]

if stock > 0:
    # Problem: Another request could check stock at the same time!
    cursor.execute("UPDATE products SET stock = stock - 1 WHERE id = ?", (product_id,))
    conn.commit()
    return "Success"
else:
    return "Out of stock"
```

**Problems:**
- ⚠️ Race conditions (two users can buy the last item)
- 🐌 Lock contention with high traffic
- 💥 Database can become bottleneck

### Redis Solution with Atomic Operations

```python
import redis

r = redis.Redis(decode_responses=True)

# Initialize inventory
r.json().set('product:123', '$', {
    'name': 'Laptop',
    'price': 999,
    'stock': 100
})

# Atomic decrement (no race condition!)
new_stock = r.json().numincrby('product:123', '$.stock', -1)

if new_stock[0] >= 0:
    return "Purchase successful"
else:
    # Rollback
    r.json().numincrby('product:123', '$.stock', 1)
    return "Out of stock"

# Even better: Use Redis Transactions
pipeline = r.pipeline()
pipeline.json().get('product:123', '$.stock')
pipeline.json().numincrby('product:123', '$.stock', -1)
result = pipeline.execute()

if result[1][0] >= 0:
    return "Success"
else:
    r.json().numincrby('product:123', '$.stock', 1)
    return "Out of stock"
```

**Benefits:**
- 🔒 Atomic operations prevent overselling
- ⚡ Blazing fast
- 📈 Handles high concurrency
- 💯 No race conditions

---

## 📊 Use Case 4: Real-Time Analytics Dashboard

### The Challenge
Analytics dashboards need:
- Track metrics in real-time (page views, clicks, errors)
- Aggregate data by time windows
- Fast reads for dashboard updates
- Handle high write throughput

### Traditional Approach

```python
# Increment page view counter
cursor.execute("""
    UPDATE analytics 
    SET page_views = page_views + 1 
    WHERE date = ? AND page = ?
""", (today, page_url))

# Get today's stats (multiple queries)
cursor.execute("SELECT SUM(page_views) FROM analytics WHERE date = ?", (today,))
cursor.execute("SELECT SUM(errors) FROM analytics WHERE date = ?", (today,))
cursor.execute("SELECT SUM(api_calls) FROM analytics WHERE date = ?", (today,))
```

**Problems:**
- Database writes are slow
- Aggregation queries are expensive
- Can't handle real-time traffic
- Locks cause contention

### Redis Solution

```python
import redis
from datetime import datetime

r = redis.Redis(decode_responses=True)

# Increment metrics (atomic and fast!)
today = datetime.now().strftime('%Y-%m-%d')

r.hincrby(f'analytics:{today}', 'page_views', 1)
r.hincrby(f'analytics:{today}', 'api_calls', 1)
r.zincrby(f'analytics:{today}:pages', 1, '/home')

# Get all today's metrics in one call
stats = r.hgetall(f'analytics:{today}')
# Returns: {'page_views': '15234', 'api_calls': '8765', 'errors': '12'}

# Get top 10 most viewed pages
top_pages = r.zrevrange(f'analytics:{today}:pages', 0, 9, withscores=True)
# Returns: [('/home', 5000), ('/products', 3500), ...]

# Time-series data with sorted sets
timestamp = int(time.time())
r.zadd(f'metrics:cpu', {timestamp: 75.5})  # 75.5% CPU usage
r.zadd(f'metrics:memory', {timestamp: 82.3})

# Get last hour of data
one_hour_ago = timestamp - 3600
cpu_data = r.zrangebyscore(f'metrics:cpu', one_hour_ago, timestamp)
```

**Benefits:**
- 🚀 Sub-millisecond writes
- 📊 Built-in aggregation (hashes, sorted sets)
- ⚡ Instant reads
- 💪 Handles millions of events/sec

---

## 🔔 Use Case 5: Rate Limiting

### The Challenge
APIs need to:
- Limit requests per user/IP
- Prevent abuse
- Handle distributed systems
- Fast check without database queries

### Redis Solution with Sliding Window

```python
import redis
import time

r = redis.Redis(decode_responses=True)

def is_rate_limited(user_id, max_requests=100, window_seconds=60):
    """
    Sliding window rate limiter
    Allows 100 requests per 60 seconds
    """
    key = f'rate_limit:{user_id}'
    now = time.time()
    window_start = now - window_seconds
    
    # Remove old entries
    r.zremrangebyscore(key, 0, window_start)
    
    # Count requests in current window
    request_count = r.zcard(key)
    
    if request_count < max_requests:
        # Add current request
        r.zadd(key, {now: now})
        r.expire(key, window_seconds)
        return False  # Not limited
    else:
        return True  # Rate limited

# Usage
if is_rate_limited('user_123'):
    return "Too many requests, try again later"
else:
    # Process request
    pass
```

**Alternative: Token Bucket with Redis**

```python
def check_rate_limit_token_bucket(user_id, capacity=100, refill_rate=10):
    """
    Token bucket algorithm
    capacity: max tokens
    refill_rate: tokens added per second
    """
    key = f'tokens:{user_id}'
    
    # Get current tokens and last refill time
    data = r.json().get(key)
    
    if not data:
        # Initialize
        r.json().set(key, '$', {
            'tokens': capacity,
            'last_refill': time.time()
        })
        r.expire(key, 3600)
        return True
    
    now = time.time()
    elapsed = now - data['last_refill']
    
    # Refill tokens
    new_tokens = min(capacity, data['tokens'] + (elapsed * refill_rate))
    
    if new_tokens >= 1:
        # Use one token
        r.json().set(key, '$', {
            'tokens': new_tokens - 1,
            'last_refill': now
        })
        return True
    else:
        return False
```

---

## 🔍 Use Case 6: Full-Text Search with RediSearch

### Redis with RediSearch Module

```python
from redis import Redis
from redis.commands.search.field import TextField, NumericField
from redis.commands.search.indexDefinition import IndexDefinition, IndexType

r = Redis(decode_responses=True)

# Create search index
r.ft('idx:products').create_index(
    [
        TextField('name'),
        TextField('description'),
        NumericField('price'),
        NumericField('stock')
    ],
    definition=IndexDefinition(prefix=['product:'], index_type=IndexType.JSON)
)

# Add products
r.json().set('product:1', '$', {
    'name': 'MacBook Pro',
    'description': 'Powerful laptop for developers',
    'price': 2499,
    'stock': 50
})

# Search
results = r.ft('idx:products').search('laptop @price:[1000 3000]')
# Finds all laptops between $1000-$3000

# Auto-complete
r.ft('idx:products').sugadd('autocomplete', 'MacBook Pro', 1.0)
suggestions = r.ft('idx:products').sugget('autocomplete', 'Mac')
# Returns: ['MacBook Pro']
```

---

## 💡 Best Practices

### 1. Key Naming Convention
```python
# Good: Hierarchical, readable
user:123:profile
user:123:sessions:abc
product:456:reviews
order:789:items

# Bad: Unclear
u123
p456data
```

### 2. Use Expiration for Temporary Data
```python
# Set with expiration
r.setex('temp:token', 3600, token_value)  # Expires in 1 hour

# Or set expiration separately
r.set('key', 'value')
r.expire('key', 3600)
```

### 3. Use Pipelines for Multiple Operations
```python
# Without pipeline (multiple round trips)
r.set('key1', 'val1')
r.set('key2', 'val2')
r.set('key3', 'val3')

# With pipeline (single round trip)
pipe = r.pipeline()
pipe.set('key1', 'val1')
pipe.set('key2', 'val2')
pipe.set('key3', 'val3')
pipe.execute()
```

### 4. Use Redis Transactions for Atomicity
```python
# WATCH for optimistic locking
r.watch('balance:user_123')

balance = int(r.get('balance:user_123'))
if balance >= 100:
    pipe = r.pipeline()
    pipe.decrby('balance:user_123', 100)
    pipe.incrby('balance:user_456', 100)
    pipe.execute()
```

### 5. Monitor Memory Usage
```python
# Get memory usage of a key
memory = r.memory_usage('user:123')

# Set max memory policy in redis.conf
# maxmemory 2gb
# maxmemory-policy allkeys-lru
```

---

## 🎯 Summary

Redis excels at:
- ✅ Real-time data (sessions, carts, live stats)
- ✅ Caching frequently accessed data
- ✅ Rate limiting and throttling
- ✅ Leaderboards and rankings
- ✅ Pub/Sub messaging
- ✅ Time-series data
- ✅ Geospatial data
- ✅ Atomic counters and operations

Use traditional databases for:
- 📊 Complex relational queries
- 📝 Long-term data storage
- 🔗 Multi-table joins
- 📈 Historical reporting

**Best approach:** Use both! Redis for hot data, traditional DB for cold data. 🚀
