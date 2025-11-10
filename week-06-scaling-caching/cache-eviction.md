# Cache Eviction Policies

## Learning Objectives
- Understand cache eviction algorithms (LRU, LFU, FIFO)
- Learn when to use each policy
- Implement LRU cache

## Notes

### Why Eviction?

**Problem:** Cache has limited memory. When full, must evict (remove) items to make room.

**Question:** Which item should we evict?

### Common Eviction Policies

#### 1. LRU (Least Recently Used)

**Rule:** Evict the item that hasn't been accessed for the longest time.

**How it works:**
```
Cache: [A, B, C, D] (capacity: 4)

Access C → [A, B, D, C] (C moves to end)
Access B → [A, D, C, B] (B moves to end)
Add E → [D, C, B, E] (A evicted - least recently used)
```

**Implementation:**
```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.cache = OrderedDict()
        self.capacity = capacity

    def get(self, key):
        if key not in self.cache:
            return None
        # Move to end (most recently used)
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key, value):
        if key in self.cache:
            # Update and move to end
            self.cache.move_to_end(key)
        self.cache[key] = value

        if len(self.cache) > self.capacity:
            # Remove first item (least recently used)
            self.cache.popitem(last=False)
```

**Pros:**
- ✅ Simple to understand and implement
- ✅ Works well for most workloads
- ✅ Recently accessed items likely to be accessed again

**Cons:**
- ❌ Doesn't consider access frequency
- ❌ One-time accesses can evict frequently used items
- ❌ Cache pollution from scans

**Use Cases:**
- Web page caching (recently viewed pages)
- Database query results
- Session data
- Most general-purpose caches

#### 2. LFU (Least Frequently Used)

**Rule:** Evict the item accessed the fewest times.

**How it works:**
```
Cache with frequency counts:
A: 5 accesses
B: 3 accesses
C: 8 accesses
D: 2 accesses

Cache full, add E:
- Evict D (only 2 accesses)
- E starts with 1 access
```

**Implementation:**
```python
from collections import defaultdict
import heapq

class LFUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}  # key -> (value, frequency)
        self.freq_map = defaultdict(list)  # frequency -> [keys]
        self.min_freq = 0

    def get(self, key):
        if key not in self.cache:
            return None

        value, freq = self.cache[key]
        # Increase frequency
        self.cache[key] = (value, freq + 1)
        self.freq_map[freq + 1].append(key)

        return value

    def put(self, key, value):
        if self.capacity == 0:
            return

        if key in self.cache:
            _, freq = self.cache[key]
            self.cache[key] = (value, freq + 1)
            return

        if len(self.cache) >= self.capacity:
            # Evict least frequently used
            evict_key = self.freq_map[self.min_freq].pop(0)
            del self.cache[evict_key]

        self.cache[key] = (value, 1)
        self.freq_map[1].append(key)
        self.min_freq = 1
```

**Pros:**
- ✅ Keeps frequently accessed items
- ✅ Better than LRU for access pattern with hot items
- ✅ Protects against one-time scans

**Cons:**
- ❌ More complex to implement
- ❌ Early items accumulate high frequency (hard to evict)
- ❌ New items evicted quickly even if will be popular

**Use Cases:**
- CDN caching (popular content stays)
- Video streaming (popular videos)
- Product catalog (bestsellers)

#### 3. FIFO (First In, First Out)

**Rule:** Evict oldest item (by insertion time).

**How it works:**
```
Cache: [A, B, C, D] (inserted in order)

Add E → [B, C, D, E] (A evicted - first in)
Add F → [C, D, E, F] (B evicted - first in)
```

**Pros:**
- ✅ Simplest to implement
- ✅ Fair (all items get equal time)

**Cons:**
- ❌ Ignores access patterns completely
- ❌ May evict frequently accessed items
- ❌ Poor cache hit rate

**Use Cases:**
- Simple scenarios where access pattern is uniform
- Message buffers
- Log rotation

#### 4. Random Replacement

**Rule:** Evict a random item.

**Pros:**
- ✅ Extremely simple
- ✅ No overhead

**Cons:**
- ❌ Unpredictable performance
- ❌ May evict hot items

**Use Cases:**
- Simple caches where performance isn't critical
- Academic/testing purposes

### Your YouTube Cache Design (Score: 26/40 - 65%)

**Question:** Design cache eviction for YouTube videos.

**Your Answer:**
- Mentioned LRU for recent videos
- Mentioned LFU for popular videos
- Suggested combining approaches

**What Was Missing:**
- Specific implementation details
- Segment-based caching strategy
- Calculations for cache size
- TTL policies

**Complete Answer:**

**Segmented Cache Strategy:**

```python
class YouTubeCache:
    def __init__(self):
        # Different policies for different video types
        self.recent_cache = LRUCache(capacity=1000)  # Recently uploaded
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
- Recent videos: LRU (people watch new uploads)
- Viral videos: LFU (Baby Shark stays forever)
- Trending: LRU with TTL (trends change quickly)

**Cache Size Calculation:**
```
Popular videos: Top 5000 videos × 50MB/video = 250GB
Recent uploads: 1000 videos × 50MB = 50GB
Trending: 500 videos × 50MB = 25GB

Total: 325GB per cache server
```

### LRU vs LFU Trade-offs

| Scenario | LRU | LFU |
|----------|-----|-----|
| **News site** | ✅ Good (recent news hot) | ❌ Bad (yesterday's news useless) |
| **Video streaming** | ❌ Bad (old popular videos evicted) | ✅ Good (Baby Shark always hot) |
| **E-commerce** | ✅ Good (seasonal items) | ⚠️ Mixed (bestsellers + seasonal) |
| **Social media** | ✅ Good (recent posts hot) | ❌ Bad (old posts not accessed) |

### Advanced: Adaptive Replacement Cache (ARC)

**Combines LRU and LFU:**
```
Maintains 4 lists:
1. T1: Recently used once (LRU)
2. T2: Recently used multiple times (LFU)
3. B1: Ghost entries recently evicted from T1
4. B2: Ghost entries recently evicted from T2

Dynamically balances between LRU and LFU based on workload
```

**Used by:** ZFS filesystem, some database systems

## Practice Questions

### Question 1: YouTube Cache Eviction (Score: 26/40 - 65%)

**What You Got Right:**
- ✅ Mentioned both LRU and LFU
- ✅ Understood different policies for different content

**What to Improve:**
- Add segmented cache strategy
- Include cache size calculations
- Specify TTL policies
- Implementation details

## Real-World Examples

### Redis Eviction Policies

```python
# Redis configuration
maxmemory 2gb
maxmemory-policy allkeys-lru  # Options:
# - noeviction: Return errors when full
# - allkeys-lru: LRU across all keys
# - volatile-lru: LRU only on keys with TTL
# - allkeys-lfu: LFU across all keys
# - volatile-lfu: LFU only on keys with TTL
# - allkeys-random: Random eviction
# - volatile-random: Random among keys with TTL
# - volatile-ttl: Evict soonest expiring keys
```

### Nginx Cache

```nginx
# LRU with size limit
proxy_cache_path /var/cache/nginx
    levels=1:2
    keys_zone=my_cache:10m
    max_size=10g
    inactive=60m;  # Evict if not accessed for 60 min
```

### CDN Caching (Cloudflare)

```
Popular content (>1000 requests/day): LFU
Recent content (<7 days old): LRU
Everything else: LRU with 30-day TTL
```

## Resources & References

- [LRU Cache Implementation](https://leetcode.com/problems/lru-cache/)
- [Redis Eviction Policies](https://redis.io/docs/manual/eviction/)
- [ARC Paper](https://www.usenix.org/legacy/events/fast03/tech/full_papers/megiddo/megiddo.pdf)
- [Cache Replacement Policies](https://en.wikipedia.org/wiki/Cache_replacement_policies)
