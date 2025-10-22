# Redis Code Demonstrations

This repository showcases the power and advantages of Redis through practical examples and comparisons with traditional approaches.

## 📚 Contents

### 🎯 Start Here: [Visual Comparison](VISUAL_COMPARISON.md) 👀

The best place to start! See the dramatic difference between traditional databases and Redis JSON through clear visual diagrams and side-by-side code comparisons.

- 📊 **Flow Diagrams** - Visualize how each approach works
- 📈 **Performance Charts** - See the speed difference
- 💡 **Real-World Impact** - Understand the practical benefits
- 🏆 **Clear Winner** - Why Redis JSON dominates for JSON operations

---

### 1. [Redis JSON Advantages](REDIS_JSON_ADVANTAGES.md) ⭐

A comprehensive guide explaining why Redis JSON is superior for handling JSON documents compared to traditional databases. Includes:

- 🎯 **The Problem** - Why traditional databases struggle with JSON updates
- ⚡ **The Solution** - How Redis JSON handles it efficiently
- 📊 **Side-by-Side Comparisons** - Code snippets showing traditional vs. Redis approaches
- 🔥 **Real-World Examples** - User profiles, shopping carts, IoT sensor data
- 🎓 **Advanced Features** - Atomic operations, partial retrieval, array manipulation
- 📈 **Performance Benchmarks** - 10-100x speedup for partial updates
- 🚀 **Quick Start Guide** - Get up and running in minutes

**Key Takeaway:** Redis JSON allows you to update specific fields in large JSON documents without retrieving, parsing, or storing the entire document - resulting in dramatically better performance!

### 2. [Redis Use Cases & Patterns](REDIS_USE_CASES.md) 🔥

Practical examples and patterns for common scenarios:

- 🎮 **Gaming Leaderboards** - Real-time rankings with sorted sets
- 📱 **Session Management** - Fast, auto-expiring session storage
- 🛒 **Inventory Management** - Atomic operations to prevent overselling
- 📊 **Real-Time Analytics** - High-throughput metrics and dashboards
- 🔔 **Rate Limiting** - Sliding window and token bucket implementations
- 🔍 **Full-Text Search** - Using RediSearch for product catalogs
- 💡 **Best Practices** - Key naming, pipelines, transactions, memory management

**Key Takeaway:** Redis provides specialized data structures (sorted sets, hashes, etc.) that make common patterns trivial to implement and blazing fast!

### 3. [Redis Quick Reference Cheat Sheet](REDIS_CHEATSHEET.md) ⚡

Quick reference guide for developers:

- 📊 **Visual Comparisons** - See the difference at a glance
- 💻 **Common Operations** - Most-used commands with examples
- 🎯 **Data Types Guide** - When to use each type
- 📈 **Performance Table** - Real-world benchmarks
- 🚀 **Quick Start** - Get running in 2 minutes
- 💡 **Pro Tips** - Best practices for production

**Key Takeaway:** Keep this handy as your go-to reference when working with Redis!

## 🎯 Why Redis?

### Traditional Database Approach:
```python
# Update one field in a 100KB JSON document
document = db.get_document()     # Retrieve 100KB
data = json.loads(document)      # Parse 100KB
data["field"] = "new_value"      # Change one field
document = json.dumps(data)      # Serialize 100KB
db.save_document(document)       # Store 100KB
```

### Redis JSON Approach:
```python
# Update one field - ONE command!
r.json().set('doc:1', '$.field', 'new_value')
```

**Result: 10-100x faster!** ⚡

## 🚀 Quick Start

1. Install Redis:
   ```bash
   # macOS
   brew install redis && brew services start redis
   
   # Linux
   sudo apt-get install redis-server
   
   # Docker
   docker run -d -p 6379:6379 redis/redis-stack-server:latest
   ```

2. Install Python client:
   ```bash
   pip install redis
   ```

3. Try it:
   ```python
   import redis
   r = redis.Redis(decode_responses=True)
   
   # Store JSON
   r.json().set('user:1', '$', {"name": "Alice", "score": 100})
   
   # Update one field
   r.json().set('user:1', '$.score', 150)
   
   # Get specific field
   score = r.json().get('user:1', '$.score')
   ```

## 📖 Learn More

- [Redis JSON Documentation](https://redis.io/docs/stack/json/)
- [Redis Commands Reference](https://redis.io/commands/)
- [Redis Python Client](https://redis-py.readthedocs.io/)

## 🤝 Contributing

Feel free to add more examples! Ideas:
- Node.js examples
- Java examples
- Real-time leaderboard
- Caching strategies
- Pub/Sub messaging
- Stream processing

## 📄 License

See [LICENSE](LICENSE) file for details.
