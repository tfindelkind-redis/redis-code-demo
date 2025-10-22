# Redis Quick Reference Cheat Sheet

## 🚀 Why Redis?

```
Traditional DB:           Redis:
┌──────────────┐         ┌──────────────┐
│ Get 100KB    │   VS    │ Update 1     │
│ Parse 100KB  │         │ field        │
│ Change 1     │         │ (microsecs)  │
│ field        │         └──────────────┘
│ Serialize    │         
│ 100KB        │         Result: 10-100x
│ Store 100KB  │         faster! ⚡
└──────────────┘
```

---

## 📊 Common Operations Comparison

### Update Single Field in JSON

```python
# ❌ Traditional Database
doc = db.get("user_123")                # Get entire 50KB
data = json.loads(doc)                  # Parse 50KB
data["settings"]["theme"] = "dark"      # Change field
doc = json.dumps(data)                  # Serialize 50KB
db.save("user_123", doc)                # Store 50KB
# Time: ~50ms

# ✅ Redis JSON
r.json().set('user:123', '$.settings.theme', 'dark')
# Time: ~0.5ms (100x faster!)
```

### Increment Counter

```python
# ❌ Traditional (Race Condition Risk!)
count = db.get("views")
count += 1
db.set("views", count)

# ✅ Redis (Atomic!)
r.incr('views')  # or
r.json().numincrby('stats', '$.views', 1)
```

### Add to Array

```python
# ❌ Traditional
cart = json.loads(db.get("cart"))
cart["items"].append(new_item)
db.set("cart", json.dumps(cart))

# ✅ Redis
r.json().arrappend('cart', '$.items', new_item)
```

---

## 🎯 Redis Data Types Quick Guide

| Type | Use Case | Example |
|------|----------|---------|
| **String** | Simple values, counters | `r.set('key', 'value')` |
| **Hash** | Objects with fields | `r.hset('user:1', mapping={'name': 'Alice'})` |
| **List** | Queues, recent items | `r.lpush('queue', 'task1')` |
| **Set** | Unique items, tags | `r.sadd('tags', 'python', 'redis')` |
| **Sorted Set** | Leaderboards, rankings | `r.zadd('scores', {'player1': 100})` |
| **JSON** | Complex documents | `r.json().set('doc', '$', {...})` |

---

## 💡 Most Useful Commands

### JSON Operations
```python
# Store document
r.json().set('user:1', '$', {'name': 'Alice', 'age': 30})

# Get entire document
r.json().get('user:1')

# Get specific field
r.json().get('user:1', '$.name')

# Update field
r.json().set('user:1', '$.age', 31)

# Increment number
r.json().numincrby('user:1', '$.age', 1)

# Append to array
r.json().arrappend('user:1', '$.tags', 'premium')

# Array length
r.json().arrlen('user:1', '$.tags')
```

### Sorted Set (Leaderboard)
```python
# Add/update score
r.zadd('leaderboard', {'player1': 1000})

# Increment score
r.zincrby('leaderboard', 100, 'player1')

# Get top 10
r.zrevrange('leaderboard', 0, 9, withscores=True)

# Get rank (0-based)
r.zrevrank('leaderboard', 'player1')

# Get score
r.zscore('leaderboard', 'player1')
```

### Hash (Simple Objects)
```python
# Set fields
r.hset('user:1', mapping={'name': 'Alice', 'email': 'alice@ex.com'})

# Get single field
r.hget('user:1', 'name')

# Get all fields
r.hgetall('user:1')

# Increment number field
r.hincrby('user:1', 'login_count', 1)
```

### Expiration
```python
# Set with expiration (seconds)
r.setex('session:abc', 3600, session_data)

# Set expiration on existing key
r.expire('session:abc', 3600)

# Get TTL
r.ttl('session:abc')
```

### Transactions & Pipelines
```python
# Pipeline (batch commands)
pipe = r.pipeline()
pipe.set('key1', 'val1')
pipe.set('key2', 'val2')
pipe.incr('counter')
pipe.execute()

# Transaction (atomic)
r.watch('balance')
pipe = r.pipeline()
pipe.decrby('balance:user1', 100)
pipe.incrby('balance:user2', 100)
pipe.execute()
```

---

## 📈 Performance Comparison Table

| Operation | Traditional DB | Redis | Speedup |
|-----------|---------------|-------|---------|
| Get full document (5KB) | 12ms | 0.8ms | 15x |
| Update 1 field (5KB doc) | 18ms | 0.9ms | 20x |
| Update 1 field (50KB doc) | 45ms | 0.9ms | 50x |
| Update 1 field (500KB doc) | 180ms | 1.0ms | 180x |
| Increment counter | 8ms | 0.1ms | 80x |
| Get leaderboard top 10 | 250ms | 1ms | 250x |
| Get player rank | 500ms | 1ms | 500x |

---

## 🎯 When to Use What

### Use Redis When:
- ✅ Need speed (sub-millisecond)
- ✅ Frequent updates to specific fields
- ✅ Real-time operations
- ✅ Caching
- ✅ Session storage
- ✅ Rate limiting
- ✅ Leaderboards
- ✅ Counters
- ✅ Pub/Sub messaging

### Use Traditional DB When:
- 📊 Complex queries with joins
- 📝 Long-term persistence
- 🔍 Ad-hoc reporting
- 📈 Historical analysis
- 🔗 Referential integrity

### Best Practice:
```
┌─────────────┐      ┌──────────────┐
│   Redis     │      │ Traditional  │
│   (Hot)     │<---->│      DB      │
│             │ sync │   (Cold)     │
│ - Sessions  │      │ - History    │
│ - Carts     │      │ - Orders     │
│ - Realtime  │      │ - Reports    │
└─────────────┘      └──────────────┘
```

---

## 🚀 Quick Start

```bash
# Install
brew install redis  # macOS
# or
docker run -d -p 6379:6379 redis/redis-stack-server

# Start
redis-server

# Python
pip install redis
```

```python
import redis

# Connect
r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Test
r.set('hello', 'world')
print(r.get('hello'))  # 'world'

# JSON
r.json().set('user:1', '$', {'name': 'Alice', 'score': 100})
r.json().numincrby('user:1', '$.score', 50)
print(r.json().get('user:1'))  # {'name': 'Alice', 'score': 150}
```

---

## 🔗 Resources

- [Official Docs](https://redis.io/docs/)
- [Commands Reference](https://redis.io/commands/)
- [Redis JSON](https://redis.io/docs/stack/json/)
- [Python Client](https://redis-py.readthedocs.io/)

---

## 💡 Pro Tips

1. **Key Naming:** Use `:` for hierarchy: `user:123:profile`
2. **Always Set Expiration** for temporary data
3. **Use Pipelines** for multiple operations
4. **Monitor Memory:** `r.memory_usage('key')`
5. **Use Appropriate Data Types:** Don't store JSON if a Hash works
6. **Connection Pooling:** Reuse connections in production

---

Made with ❤️ for Redis developers
