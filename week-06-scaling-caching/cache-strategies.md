# Cache Strategies

## Learning Objectives
- Understand cache-aside, write-through, write-back patterns
- Learn cache invalidation strategies
- Master CDN caching

## Notes

### Cache Strategies Overview

**Question:** When and how should we update the cache?

### 1. Cache-Aside (Lazy Loading)

**Pattern:** Application manages cache explicitly.

**Read Flow:**
```python
def get_user(user_id):
    # 1. Try cache first
    user = cache.get(f"user:{user_id}")

    if user:
        return user  # Cache hit!

    # 2. Cache miss → query database
    user = database.query("SELECT * FROM users WHERE id = ?", user_id)

    # 3. Store in cache for next time
    cache.set(f"user:{user_id}", user, ttl=3600)

    return user
```

**Write Flow:**
```python
def update_user(user_id, data):
    # 1. Update database
    database.update("UPDATE users SET ... WHERE id = ?", user_id, data)

    # 2. Invalidate cache
    cache.delete(f"user:{user_id}")

    # Next read will fetch fresh data from database
```

**Pros:**
- ✅ Only cache what's actually requested (efficient)
- ✅ Cache failures don't affect writes
- ✅ Simple to implement

**Cons:**
- ❌ Cache miss penalty (3 steps: check cache, query DB, update cache)
- ❌ Stale data possible if cache not invalidated on writes
- ❌ Cache stampede (many requests hit DB when cache expires)

**Use Cases:**
- Read-heavy workloads
- Expensive database queries
- User profiles, product details

**Your Twitter Cache Design (Score: 40/40 - 100%!) 🏆**

You perfectly described cache-aside pattern with:
- Check Redis first
- Query database on miss
- TTL for automatic expiration
- Invalidation on tweet deletion

### 2. Write-Through Cache

**Pattern:** Write to cache and database simultaneously (synchronously).

**Write Flow:**
```python
def update_user(user_id, data):
    # 1. Update cache
    cache.set(f"user:{user_id}", data)

    # 2. Update database
    database.update("UPDATE users SET ... WHERE id = ?", user_id, data)

    # Both complete before returning to client
```

**Read Flow:**
```python
def get_user(user_id):
    # Always read from cache (guaranteed to be there)
    return cache.get(f"user:{user_id}")
```

**Pros:**
- ✅ Cache always consistent with database
- ✅ No stale data
- ✅ Reads always fast (always in cache)

**Cons:**
- ❌ Slow writes (wait for both cache and DB)
- ❌ Cache pollution (write everything, even if not read)
- ❌ Higher write latency

**Use Cases:**
- Read-heavy with critical consistency
- Small datasets that fit in cache
- Session stores

### 3. Write-Back (Write-Behind) Cache

**Pattern:** Write to cache immediately, update database asynchronously (later).

**Write Flow:**
```python
def update_user(user_id, data):
    # 1. Update cache immediately
    cache.set(f"user:{user_id}", data)

    # 2. Mark as dirty
    dirty_queue.add(user_id)

    # 3. Return immediately (fast!)
    return "Success"

# Background worker:
def sync_worker():
    while True:
        user_id = dirty_queue.pop()
        data = cache.get(f"user:{user_id}")
        database.update("UPDATE users SET ... WHERE id = ?", user_id, data)
```

**Pros:**
- ✅ Very fast writes (only cache latency)
- ✅ Batch database writes (efficient)
- ✅ Good for write-heavy workloads

**Cons:**
- ❌ Risk of data loss (if cache crashes before DB sync)
- ❌ Complex to implement
- ❌ Eventual consistency with database

**Use Cases:**
- Gaming leaderboards (frequent updates)
- Analytics counters
- Non-critical data where speed > durability

### 4. Refresh-Ahead Cache

**Pattern:** Proactively refresh cache before expiration.

**Implementation:**
```python
def get_user(user_id):
    user, expiry_time = cache.get_with_ttl(f"user:{user_id}")

    if user:
        # Check if expiring soon
        time_until_expiry = expiry_time - time.now()

        if time_until_expiry < 60:  # Less than 1 minute
            # Refresh in background
            background_task(refresh_user_cache, user_id)

        return user

    # Cache miss → load from DB
    user = database.query("SELECT * FROM users WHERE id = ?", user_id)
    cache.set(f"user:{user_id}", user, ttl=3600)
    return user

def refresh_user_cache(user_id):
    user = database.query("SELECT * FROM users WHERE id = ?", user_id)
    cache.set(f"user:{user_id}", user, ttl=3600)
```

**Pros:**
- ✅ No cache miss penalty for popular items
- ✅ Always fast reads

**Cons:**
- ❌ Complexity
- ❌ May refresh items that won't be accessed
- ❌ Requires prediction of what to refresh

**Use Cases:**
- Homepage content
- Popular product pages
- Frequently accessed dashboards

### Cache Invalidation Strategies

**"There are only two hard things in Computer Science: cache invalidation and naming things."** - Phil Karlton

#### 1. TTL (Time To Live)

```python
cache.set("user:123", user_data, ttl=3600)  # Expires in 1 hour
```

**Pros:** Simple, automatic cleanup
**Cons:** Data can be stale until expiry

#### 2. Explicit Invalidation

```python
def update_user(user_id, data):
    database.update(user_id, data)
    cache.delete(f"user:{user_id}")  # Explicit invalidation
```

**Pros:** Immediate consistency
**Cons:** Must remember all cache keys to invalidate

#### 3. Tag-Based Invalidation

```python
# Tag related cache entries
cache.set("user:123", data, tags=["user", "profile", "user_123"])

# Invalidate all entries with tag
cache.invalidate_tag("user_123")
# Removes: user profile, user posts, user comments, etc.
```

#### 4. Event-Based Invalidation

```python
# Kafka event
user_service.publish_event("UserUpdated", {"user_id": 123})

# Cache invalidation consumer
def on_user_updated(event):
    cache.delete(f"user:{event['user_id']}")
    cache.delete(f"user_posts:{event['user_id']}")
    cache.delete(f"user_timeline:{event['user_id']}")
```

### Cache Stampede Problem

**Problem:** Cache expires → Many requests hit DB simultaneously.

```
Time 0: Cache expires
Time 1: Request 1 → Cache miss → Query DB
Time 1: Request 2 → Cache miss → Query DB  ← Duplicate!
Time 1: Request 3 → Cache miss → Query DB  ← Duplicate!
Time 1: Request 4 → Cache miss → Query DB  ← Duplicate!

Result: 4 identical DB queries! (Thundering herd)
```

**Solution 1: Lock-Based**
```python
def get_user(user_id):
    user = cache.get(f"user:{user_id}")
    if user:
        return user

    # Try to acquire lock
    lock_key = f"lock:user:{user_id}"
    if cache.set_nx(lock_key, "locked", ttl=10):
        # Got lock → query DB
        user = database.query("SELECT * FROM users WHERE id = ?", user_id)
        cache.set(f"user:{user_id}", user, ttl=3600)
        cache.delete(lock_key)
        return user
    else:
        # Someone else is loading → wait and retry
        time.sleep(0.1)
        return get_user(user_id)  # Retry
```

**Solution 2: Probabilistic Early Expiration**
```python
import random

def get_user(user_id):
    user, expiry_time = cache.get_with_ttl(f"user:{user_id}")

    if user:
        # Probabilistically refresh before expiry
        time_until_expiry = expiry_time - time.now()
        refresh_probability = 1 - (time_until_expiry / 3600)

        if random.random() < refresh_probability:
            background_refresh(user_id)

        return user

    # Cache miss
    user = database.query("SELECT * FROM users WHERE id = ?", user_id)
    cache.set(f"user:{user_id}", user, ttl=3600)
    return user
```

### CDN Caching

**Content Delivery Network** = Geographically distributed cache servers.

**How it Works:**
```
User in Tokyo requests image:
  1. Request goes to nearest CDN edge server (Tokyo)
  2. Tokyo CDN checks local cache
  3. If miss → request from origin server (US)
  4. Tokyo CDN caches response
  5. Future Tokyo users get instant response from local cache
```

**Cache Headers:**
```http
HTTP/1.1 200 OK
Cache-Control: public, max-age=31536000  # Cache for 1 year
ETag: "abc123"  # Version identifier
Last-Modified: Mon, 01 Jan 2024 00:00:00 GMT
```

**Invalidation:**
```python
# Option 1: Versioned URLs
image_url = f"https://cdn.example.com/image.jpg?v={version}"

# Option 2: Purge API
cdn.purge("/image.jpg")
```

## Practice Questions

### Question 1: Twitter Cache Strategy (Score: 40/40 - 100%!) 🏆

**Your Perfect Answer:**
- Cache-aside pattern
- Redis for hot tweets
- TTL for automatic expiration
- Invalidation on deletion
- Read-your-writes consistency

**What Made It Perfect:**
- ✅ Correct pattern choice
- ✅ Specific cache keys
- ✅ TTL strategy
- ✅ Invalidation on writes
- ✅ Consistency considerations

## Real-World Examples

### Facebook News Feed

```python
# Multi-layer caching
def get_news_feed(user_id):
    # Layer 1: Browser cache (5 minutes)
    # Layer 2: CDN cache (1 minute)
    # Layer 3: Application cache (Redis, 30 seconds)
    feed = redis.get(f"feed:{user_id}")

    if not feed:
        # Layer 4: Database
        feed = generate_feed_from_db(user_id)
        redis.setex(f"feed:{user_id}", 30, feed)

    return feed
```

### YouTube Video Metadata

```python
# Write-through for video metadata (critical)
def upload_video(video_id, metadata):
    cache.set(f"video:{video_id}", metadata)
    database.insert("videos", metadata)

# Cache-aside for views (non-critical)
def increment_views(video_id):
    # Write-back pattern
    cache.incr(f"views:{video_id}")

    # Sync to DB every 1000 views
    views = cache.get(f"views:{video_id}")
    if views % 1000 == 0:
        database.update(f"UPDATE videos SET views = {views} WHERE id = ?", video_id)
```

### Amazon Product Catalog

```python
# Refresh-ahead for popular products
def get_product(product_id):
    product = cache.get(f"product:{product_id}")

    if product:
        views = cache.incr(f"product_views:{product_id}")

        if views % 100 == 0:  # Popular product!
            background_refresh(product_id)

        return product

    product = database.query("SELECT * FROM products WHERE id = ?", product_id)
    cache.set(f"product:{product_id}", product, ttl=3600)
    return product
```

## Resources & References

- [Caching Strategies](https://docs.aws.amazon.com/AmazonElastiCache/latest/mem-ug/Strategies.html)
- [CDN Caching Best Practices](https://www.cloudflare.com/learning/cdn/caching-best-practices/)
- [Cache Stampede Solutions](https://instagram-engineering.com/thundering-herds-promises-82191c8af57d)
- [Redis Caching Patterns](https://redis.io/docs/manual/patterns/)
