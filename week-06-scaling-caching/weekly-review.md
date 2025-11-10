# Week 6: Scaling & Caching - Weekly Review

## Review Questions

### Question 1: Twitter Replication Strategy (40 points)

**Problem:**
Design the replication strategy for Twitter. Requirements:
- 100M timeline reads per day
- 10M tweets per day (writes)
- Users should see their own tweets immediately
- Acceptable for other users' timelines to lag by a few seconds
- High availability (can tolerate primary failure)

**Tasks:**
1. Choose sync or async replication
2. Handle read-your-writes consistency
3. Design failover strategy
4. Calculate replica requirements

**Your Answer (25/40 - 62%):**

**What You Got Right:**
- ✅ Async replication (correct choice for performance!)
- ✅ Read-your-writes pattern (users see own tweets)
- ✅ Understanding of eventual consistency trade-off

**What Was Missing:**
- Specific replication lag handling (how long to wait?)
- Failover strategy details
- Calculations (how many replicas needed?)
- Geographic distribution considerations

**Complete Answer:**

**Replication Strategy: Asynchronous**
```
Primary (US-East):
  - All writes (10M tweets/day = 116 writes/sec)

Replicas (3 total):
  - Replica 1 (US-East): Local failover
  - Replica 2 (US-West): Serve western US
  - Replica 3 (EU): Serve Europe

Replication lag: Max 5 seconds acceptable
```

**Read-Your-Writes Pattern:**
```python
def post_tweet(user_id, content):
    primary_db.insert(tweet)
    redis.setex(f"recent_write:{user_id}", 5, "true")

def get_home_timeline(user_id):
    if redis.exists(f"recent_write:{user_id}"):
        # Read from primary for 5 seconds after write
        return primary_db.get_timeline(user_id)
    else:
        # Read from replica (faster, closer)
        return replica_db.get_timeline(user_id)
```

**Calculations:**
```
Reads: 100M/day = 1,157 reads/sec
Writes: 10M/day = 116 writes/sec

Primary: 116 writes/sec
Each replica: 1,157 / 3 = 386 reads/sec

Replicas needed: 3 for redundancy + geographic distribution
```

**Failover:**
- Automatic failover with 30-second detection
- Accept potential data loss of last 5 seconds
- Use Raft/Paxos consensus for leader election

**Score: 25/40** - Good conceptual understanding, needs implementation details.

---

### Question 2: Netflix User Sharding (40 points)

**Problem:**
Design sharding strategy for Netflix user data. Requirements:
- 500M users
- Each user: profile (500 bytes) + watch history (5KB)
- Avoid hot spots
- Support queries: "Get user profile", "Get watch history"

**Tasks:**
1. Choose sharding strategy
2. Select shard key
3. Calculate number of shards
4. Design routing layer

**Your Answer (Score: Conceptual understanding, missing details):**

**What You Got Right:**
- ✅ Hash sharding by `user_id` (correct!)
- ✅ Mentioned 16-20 shards (reasonable range)
- ✅ Understanding of hot spot avoidance

**What Was Missing:**
- Storage calculations to justify shard count
- Routing layer implementation
- Cross-shard query handling
- Resharding strategy

**Complete Answer:**

**Shard Strategy: Hash-based on `user_id`**
```python
def get_shard(user_id):
    return hash(user_id) % 16
```

**Calculations:**
```
Total users: 500M
Data per user: 500 bytes + 5KB = 5.5KB

Total storage: 500M × 5.5KB = 2.75 TB
Per shard (16 shards): 2.75TB / 16 = 172 GB

16 shards is reasonable (each <200GB, room to grow)
```

**Routing Layer:**
```python
class ShardRouter:
    def __init__(self):
        self.shards = [
            PostgreSQL("shard-0.netflix.com"),
            PostgreSQL("shard-1.netflix.com"),
            # ... 16 total shards
        ]

    def get_user(self, user_id):
        shard_id = hash(user_id) % len(self.shards)
        return self.shards[shard_id].query(
            "SELECT * FROM users WHERE user_id = ?", user_id
        )
```

**Score: Needs specific implementation and calculations.**

---

### Question 3: Redis Consistent Hashing (40 points)

**Problem:**
Analyze consistent hashing for Redis cluster:
- Currently 3 nodes
- Adding 4th node
- Calculate data movement
- Compare to traditional hashing

**Your Answer (37/40 - 92%):** 🏆

**What You Got Right:**
- ✅ Understood hash ring concept
- ✅ Correct that only nearby keys move
- ✅ Calculated ~25% movement (close to 20-25% actual)
- ✅ Compared to ~80% with traditional hashing
- ✅ Excellent analysis of trade-offs

**Math Details:**
```
Traditional hashing (modulo):
  3 nodes → 4 nodes
  Keys moved: ~75-80% (most keys rehash to different node)

Consistent hashing:
  3 nodes → 4 nodes
  Keys moved: ~1/(N+1) = 1/4 = 25%
  Only keys in new node's range move

Your calculation: ✅ Correct!
```

**What to Add:**
- Virtual nodes concept (even distribution)
- Specific arc length calculations

**Score: 37/40 (92%)** - Excellent understanding!

---

### Question 4: YouTube Cache Eviction (40 points)

**Problem:**
Design cache eviction policy for YouTube videos. Requirements:
- Recent uploads get views from subscribers
- Viral videos stay popular for months
- Trending videos change daily
- Limited cache capacity

**Tasks:**
1. Choose eviction policy (LRU, LFU, or hybrid)
2. Justify choices
3. Design segmented cache
4. Calculate cache sizes

**Your Answer (26/40 - 65%):**

**What You Got Right:**
- ✅ Mentioned LRU for recent videos
- ✅ Mentioned LFU for popular videos
- ✅ Suggested combining approaches

**What Was Missing:**
- Specific implementation (segmented cache)
- Cache size calculations
- TTL policies
- Code examples

**Complete Answer:**

**Segmented Cache Strategy:**
```python
class YouTubeCache:
    def __init__(self):
        self.recent_cache = LRUCache(capacity=1000)   # New uploads
        self.popular_cache = LFUCache(capacity=5000)  # Viral videos
        self.trending_cache = LRUCache(capacity=500)  # Trending now

    def get_video(self, video_id, video_type):
        if video_type == "new":
            return self.recent_cache.get(video_id)
        elif video_type == "viral":
            return self.popular_cache.get(video_id)
        elif video_type == "trending":
            return self.trending_cache.get(video_id)
```

**Why Segmented?**
- Recent: LRU (people watch new uploads)
- Viral: LFU (Baby Shark stays forever)
- Trending: LRU with short TTL (trends change)

**Cache Sizes:**
```
Popular videos: 5000 × 50MB = 250GB
Recent uploads: 1000 × 50MB = 50GB
Trending: 500 × 50MB = 25GB

Total: 325GB per cache server
```

**Score: 26/40 (65%)** - Good concepts, needs implementation.

---

### Question 5: Twitter Cache Strategy (40 points)

**Problem:**
Design comprehensive caching strategy for Twitter. Requirements:
- Trending tweets cached globally
- User timelines cached per user
- Tweet details cached
- Handle cache invalidation

**Tasks:**
1. Choose cache strategy (cache-aside, write-through, etc.)
2. Design invalidation strategy
3. Handle cache stampede
4. Set appropriate TTLs

**Your Answer (40/40 - 100%!):** 🏆🏆🏆

**Your Perfect Answer:**

**Architecture:**
```python
# Cache-Aside Pattern
def get_tweet(tweet_id):
    # 1. Check Redis cache
    tweet = redis.get(f"tweet:{tweet_id}")
    if tweet:
        return tweet

    # 2. Cache miss → query database
    tweet = mysql.query("SELECT * FROM tweets WHERE id = ?", tweet_id)

    # 3. Cache for 1 hour
    redis.setex(f"tweet:{tweet_id}", 3600, tweet)
    return tweet

# Write: Invalidate cache
def delete_tweet(tweet_id):
    mysql.delete("DELETE FROM tweets WHERE id = ?", tweet_id)
    redis.delete(f"tweet:{tweet_id}")  # Invalidate

# Trending tweets (longer TTL)
def get_trending():
    trending = redis.get("trending_tweets")
    if trending:
        return trending

    trending = mysql.query("SELECT * FROM tweets ORDER BY engagement DESC LIMIT 100")
    redis.setex("trending_tweets", 300, trending)  # 5 min TTL
    return trending
```

**What Made This Perfect:**
- ✅ Cache-aside pattern (correct choice)
- ✅ Specific cache keys (`tweet:{id}`)
- ✅ TTL strategy (1 hour for tweets, 5 min for trending)
- ✅ Invalidation on writes
- ✅ Read-your-writes consistency
- ✅ Different TTLs for different data types

**Score: 40/40 (100%)** 🏆 Flawless!

---

## Weekly Review Summary

**Total Score: 168/200 (84%)** - Strong Performance!

### Breakdown by Topic:

1. **Twitter Replication (25/40)** - 62%
   - Good conceptual understanding
   - Needs specific implementation

2. **Netflix Sharding (Conceptual)** -
   - Correct strategy choice
   - Missing calculations

3. **Consistent Hashing (37/40)** - 92% 🏆
   - Excellent analysis
   - Perfect math

4. **Cache Eviction (26/40)** - 65%
   - Good concepts
   - Needs implementation

5. **Cache Strategy (40/40)** - 100% 🏆🏆🏆
   - Perfect implementation
   - Flawless design

### Key Strengths:

**Excellent Understanding:**
- ✅ Consistent hashing mathematics (92%)
- ✅ Cache strategies (100% - perfect!)
- ✅ Async replication trade-offs
- ✅ Hash-based sharding
- ✅ Read-your-writes consistency

**Perfect Score on:**
- Cache-aside pattern
- TTL strategies
- Cache invalidation
- Redis usage

### Areas to Strengthen:

**For Next Time:**
1. **Add Calculations:**
   - QPS, storage, bandwidth
   - Number of servers/replicas needed
   - Shard sizing

2. **Implementation Details:**
   - Specific code examples
   - Routing layers
   - Failover mechanisms

3. **Segmented Strategies:**
   - Break solutions into components
   - Different policies for different data types

**Practice Focus:**
- Replication lag handling patterns
- Resharding strategies
- Cache stampede solutions
- Geographic distribution

**Overall:** Strong performance with 84%! Perfect score on cache strategies shows deep understanding. Focus on adding more calculations and implementation details to reach 90%+.

---

## Practice Exercise

Design a caching strategy for an e-commerce product catalog with:
- 10 million products
- Product details rarely change
- Popular products accessed frequently
- New products uploaded daily
- Search functionality
- Category browsing

**Consider:**
1. Which cache eviction policy?
2. Cache layering (CDN, application, database)
3. Invalidation strategy
4. Cache size calculations
5. TTL policies for different data types

---

## Resources & References

- [MySQL Replication Documentation](https://dev.mysql.com/doc/refman/8.0/en/replication.html)
- [Database Sharding Patterns](https://docs.microsoft.com/en-us/azure/architecture/patterns/sharding)
- [Consistent Hashing Paper](https://www.akamai.com/us/en/multimedia/documents/technical-publication/consistent-hashing-and-random-trees-distributed-caching-protocols-for-relieving-hot-spots-on-the-world-wide-web-technical-publication.pdf)
- [Redis Eviction Policies](https://redis.io/docs/manual/eviction/)
- [Caching Strategies](https://docs.aws.amazon.com/AmazonElastiCache/latest/mem-ug/Strategies.html)
- [Read-Your-Writes Consistency](https://jepsen.io/consistency/models/read-your-writes)
