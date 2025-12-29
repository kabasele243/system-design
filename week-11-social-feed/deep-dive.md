# Social Media Feed - Deep Dive

## Step 4: Deep Dive - The Fan-out Problem

### The Core Challenge
When a user posts, how do you deliver that post to all their followers?

## Approach 1: Fan-out on Write (Push Model)

### How it works:
1. User creates a post
2. System immediately writes post to feed cache of ALL followers
3. When user requests feed, read directly from their pre-generated cache

### Diagram:
```
User posts → Post Service → Message Queue → Feed Workers
                                              ↓
                                         Update all
                                    followers' feed caches
```

### Pros:
- **Fast Reads**: Feed is pre-computed (O(1) read latency).
- **Scalable Reads**: Very high read throughput (Redis is fast).

### Cons:
- **Write Amplification**: 1 post -> 200 writes. High load on queue/workers.
- **Wasted Storage**: If user never checks feed, we still store it.
- **Latency Delay**: "Lag" between posting and followers seeing it if queue backs up.

### Best for:
- Most users (Small/Medium follower counts).


## Approach 2: Fan-out on Read (Pull Model)

### How it works:
1. User creates a post -> Stored in `Posts` DB.
2. When follower requests feed:
3. System fetches follow list (e.g. 500 people).
4. System queries `Posts` DB for "latest posts from these 500 IDs" (SELECT * FROM posts WHERE user_id IN (...)).
5. Sort/Merge in memory and return.

### Diagram:
```
User requests feed → Feed Service → Query Follows
                                   → Query Posts (for all followees)
                                   → Merge & Sort
                                   → Return feed
```

### Pros:
- **No Write Amplification**: Posting is O(1) write.
- **Real-time**: No lag, post is visible immediately.
- **Storage Efficient**: No duplicated feed storage.

### Cons:
- **Slow Reads**: Large aggregation query every time.
- **Hard to Scale**: Database read heavy (scatter-gather is expensive).

### Best for:
- Celebrities (High follower counts) where push is too expensive.


## Approach 3: Hybrid Model (RECOMMENDED)

### How it works:
- **For normal users:** Use fan-out on write (posts pushed to followers' caches).
- **For celebrities (>1M followers):** Use fan-out on read (posts stored only in DB/Global Cache).
- **Client/Feed Service:** When fetching feed, it:
  1. Pulls pre-computed feed from Redis.
  2. Pulls posts from followed celebrities (asynchronously).
  3. Merges them by timestamp.

### Why hybrid?
- Balances the cost of write amplification (for normal users) with the cost of read aggregation (for celebrities).
- Optimal for both performance and resource usage.

### The Celebrity Problem:
If a celebrity with 50M followers posts, fan-out on write would:
- Write to 50M feed caches.
- Take minutes/hours to complete (lag).
- Overwhelm the cache cluster.

Solution:
- Stop pushing.
- Let followers "pull" these specific posts.


## Feed Ranking & Filtering

### Chronological vs Algorithmic Feed

**Chronological (Twitter style):**
- Simple sort by `created_at`.
- Best for real-time news.

**Algorithmic (Instagram/Facebook style):**
- Weighted score based on affinity, interactions, and freshness.
- Requires an "EdgeRank" service to re-process feeds.
- Complex ML pipeline (out of scope for basic design, but worth mentioning).


## Caching Strategy

### What to cache:
1. User's feed (last 1000 posts IDs).
2. Social graph (Followers/Following lists).
3. Post content (User/Content text) -> Content is read heavy!

### Cache Structure (Redis):
```
# For Pre-generated Feed
Key: feed:user:{user_id}
Value: List (or ZSet if using scores) of Post IDs
[1005, 1004, 1002, 998...]

# For Post Content (Content Delivery)
Key: post:{post_id}
Value: { "content": "...", "author_id": 123, "media": [...] }
```

### Cache size calculation:
- 500M users.
- Keep top 400 posts in cache per user.
- 400 * 8 bytes (ID) = 3.2KB per user feed.
- 500M * 3.2KB = 1.6TB for Feed IDs (Keep entirely in RAM).
- Content cache: LRU for hot posts.


## Time-series Data Structure

### Storing feeds efficiently:
- Redis Lists are great (LPUSH/LTRIM).
- For permanent storage (Timeline history), use Cassandra or HBase (Wide-column stores optimized for writes).


## Trade-offs Discussion

| Aspect | Fan-out on Write | Fan-out on Read | Hybrid |
|--------|-----------------|-----------------|--------|
| Read latency | Low (Fast) | High (Slow) | Low |
| Write latency | High (Async) | Low | Low |
| Storage cost | High (Duplication) | Low | Medium |
| Handles celebrities | No (Lag) | Yes | Yes (Best) |
| Your choice | | | **✓** |

## Next Steps
Continue to [failure-scenarios.md](failure-scenarios.md)
