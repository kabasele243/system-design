# Database Sharding

## Learning Objectives
- Understand horizontal partitioning (sharding)
- Learn sharding strategies (range, hash, geographic)
- Master shard key selection

## Notes

### What is Sharding?

**Sharding** = Splitting data across multiple databases (horizontal partitioning).

**Replication vs Sharding:**

| Aspect | Replication | Sharding |
|--------|-------------|----------|
| **Data** | Same data on all nodes | Different data on each node |
| **Purpose** | Availability & read scale | Write scale & storage capacity |
| **Reads** | Distributed across replicas | Go to specific shard |
| **Writes** | All go to primary | Distributed across shards |

**Your Key Insight:**
> "Replication = copies of same data. Sharding = split different data across nodes."

**Perfect!** You understood the fundamental difference!

### Why Shard?

**Problems Sharding Solves:**

1. **Storage Capacity:**
   - Single database: 10TB limit
   - Sharded: 10 shards × 10TB = 100TB total

2. **Write Throughput:**
   - Single database: 10K writes/sec max
   - Sharded: 10 shards × 10K = 100K writes/sec

3. **Query Performance:**
   - Single database: Queries scan billions of rows
   - Sharded: Each shard has millions of rows (faster queries)

### Sharding Strategies

#### 1. Hash-Based Sharding (Most Common)

**How it works:**
```python
shard_id = hash(user_id) % num_shards

# Example:
user_id = 12345
shard_id = hash(12345) % 16 = 7

# User 12345's data goes to Shard 7
```

**Pros:**
- ✅ Even distribution (no hot spots)
- ✅ Simple to implement
- ✅ Predictable shard location

**Cons:**
- ❌ Can't do range queries across users
- ❌ Adding shards requires resharding (rehash everything)
- ❌ Related data might be on different shards

**Use Cases:**
- User data (Instagram, Facebook)
- Session storage
- Any data with good hash key (user_id, order_id)

#### 2. Range-Based Sharding

**How it works:**
```python
# Shard by user_id ranges
Shard 1: user_id 1 - 1,000,000
Shard 2: user_id 1,000,001 - 2,000,000
Shard 3: user_id 2,000,001 - 3,000,000
...
```

**Pros:**
- ✅ Range queries are efficient
- ✅ Easy to add shards (just split ranges)
- ✅ Related data often together (sequential IDs)

**Cons:**
- ❌ Hot spots (recent users get more traffic)
- ❌ Uneven distribution
- ❌ Requires monitoring and rebalancing

**Use Cases:**
- Time-series data (logs, events)
- Sequential data (order history)
- Data with natural ranges

#### 3. Geographic Sharding

**How it works:**
```python
if user.country == "US":
    shard = us_shard
elif user.country == "EU":
    shard = eu_shard
elif user.country == "ASIA":
    shard = asia_shard
```

**Pros:**
- ✅ Low latency (data close to users)
- ✅ Data sovereignty compliance (GDPR)
- ✅ Easy to understand

**Cons:**
- ❌ Uneven distribution (more US users than others)
- ❌ Cross-region queries are slow
- ❌ Complex for global features

**Use Cases:**
- Uber (rides in each city/region)
- E-commerce (orders by country)
- Compliance-heavy apps (GDPR, data residency)

### Your Netflix Sharding Design

**Question:** Design sharding for Netflix user data (500M users).

**Your Answer:**
- Hash sharding by `user_id`
- 16-20 shards
- Mentioned need to avoid hot spots

**Score:** Good conceptual understanding, but missing calculations and implementation details.

**Complete Answer:**

**Shard Strategy: Hash-based on `user_id`**

```python
def get_shard(user_id):
    return hash(user_id) % 16

# Example:
# user_id = 123 → Shard 11
# user_id = 456 → Shard 3
# user_id = 789 → Shard 5
```

**Number of Shards:**
```
Total users: 500M
Users per shard: 500M / 16 = 31.25M users per shard

Storage per shard:
- User profile: 500 bytes/user
- Viewing history: 5KB/user (last 1000 watches)
- Total: 5.5KB/user

Per shard storage: 31.25M × 5.5KB = 172 GB
Total storage: 172GB × 16 = 2.75 TB

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

    def get_watch_history(self, user_id):
        shard_id = hash(user_id) % len(self.shards)
        return self.shards[shard_id].query(
            "SELECT * FROM watch_history WHERE user_id = ?", user_id
        )
```

### Shard Key Selection

**Critical Decision:** Choose the right shard key!

**Good Shard Keys:**
- ✅ High cardinality (many unique values)
- ✅ Even distribution (no hot spots)
- ✅ Frequently used in queries
- ✅ Immutable (doesn't change)

**Examples:**
- ✅ `user_id` (Instagram, Facebook, Netflix)
- ✅ `order_id` (Amazon, Shopify)
- ✅ `conversation_id` (WhatsApp, Slack)
- ✅ `tenant_id` (B2B SaaS apps)

**Bad Shard Keys:**
- ❌ `created_at` (all new records go to same shard)
- ❌ `status` (only few values: "active", "inactive")
- ❌ `country` (US has 10x more users than others)
- ❌ Frequently changing values

### Cross-Shard Queries

**Problem:** User wants to see all orders (data spans multiple shards).

**Solution 1: Fan-out Query (Scatter-Gather)**
```python
def get_all_orders():
    results = []
    for shard in shards:
        shard_results = shard.query("SELECT * FROM orders")
        results.extend(shard_results)
    return sorted(results, key=lambda x: x.created_at)

# Slow! Query all 16 shards
```

**Solution 2: Denormalize with Secondary Index**
```python
# Maintain order index in separate database
order_index_db.store({
    "order_id": 12345,
    "user_id": 789,
    "shard_id": 3  # Which shard has this order
})

def get_user_orders(user_id):
    # Query index to find which shards have user's orders
    shard_ids = order_index_db.get_shards_for_user(user_id)

    # Query only relevant shards
    results = []
    for shard_id in shard_ids:
        results.extend(shards[shard_id].query(
            "SELECT * FROM orders WHERE user_id = ?", user_id
        ))
    return results

# Much faster! Only query relevant shards
```

**Solution 3: Dual Writes**
```python
# Write to both shard and aggregated view
def create_order(order_id, user_id, items):
    shard_id = hash(order_id) % num_shards

    # Write to shard (source of truth)
    shards[shard_id].insert({
        "order_id": order_id,
        "user_id": user_id,
        "items": items
    })

    # Also write to user's order list (denormalized)
    user_shard_id = hash(user_id) % num_shards
    shards[user_shard_id].insert_into_user_orders({
        "user_id": user_id,
        "order_id": order_id
    })
```

### Resharding (Adding More Shards)

**Problem:** Data outgrows current shards, need to add more.

**Challenge:** Changing shard count requires moving data!

**Example:**
```
Original: 4 shards
hash(user_123) % 4 = 3 → Shard 3

New: 8 shards
hash(user_123) % 8 = 7 → Shard 7 (different!)

Must move user_123's data from Shard 3 → Shard 7
```

**Resharding Strategies:**

**1. Stop-the-World Resharding (Simple but Downtime)**
```
1. Put app in maintenance mode
2. Migrate data to new shards
3. Update shard count in code
4. Bring app back online

Downtime: Hours to days
```

**2. Live Resharding (No Downtime)**
```
1. Add new shards (don't remove old ones yet)
2. Dual-write: Write to both old and new shards
3. Backfill: Migrate data from old to new shards
4. Switch reads to new shards
5. Remove old shards

Downtime: None
```

**3. Consistent Hashing (Minimal Data Movement)**
```
Use consistent hashing instead of modulo
Only ~1/N data needs to move when adding shard
(covered in consistent-hashing.md)
```

## Practice Questions

### Question 1: Netflix User Sharding

**Your Answer:** Hash sharding by user_id, 16-20 shards

**What You Got Right:**
- ✅ Hash sharding (correct choice)
- ✅ user_id as shard key (good!)
- ✅ Understanding of hot spot avoidance

**What to Add Next Time:**
- Storage calculations
- Routing layer implementation
- Cross-shard query handling

## Real-World Examples

### Instagram Photo Sharding

```python
# 2 billion users, sharded by user_id

def get_shard(user_id):
    return hash(user_id) % 1000  # 1000 shards

# Each shard:
# - 2M users
# - ~200GB storage
# - PostgreSQL database

# Queries:
# "Get user_123's photos" → Go to specific shard
# "Get all photos with #sunset" → Fan-out to all shards (slow, use Elasticsearch instead)
```

### Uber Trips Sharding

```python
# Shard by city (geographic sharding)

SHARDS = {
    "SF": shard_sf,
    "NYC": shard_nyc,
    "London": shard_london,
    # ... one shard per major city
}

def get_shard_for_trip(trip):
    return SHARDS[trip.city]

# Benefits:
# - Trips in same city are together (good for driver-rider matching)
# - Comply with local data laws
# - Scale per-city (add capacity for busy cities)

# Challenge:
# - Users traveling to different cities (cross-shard)
# - Uneven distribution (NYC >> rural cities)
```

### Discord Message Sharding

```python
# Shard by server_id (tenant-based sharding)

def get_shard(server_id):
    return hash(server_id) % 128

# Benefits:
# - All messages for one server on same shard
# - No cross-shard queries for channels
# - Each Discord server = isolated tenant

# 128 shards for millions of servers
```

## Resources & References

- [Sharding Patterns](https://docs.microsoft.com/en-us/azure/architecture/patterns/sharding)
- [Instagram Sharding Architecture](https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c)
- [Uber's Schemaless (Sharding)](https://eng.uber.com/schemaless-part-one/)
- [Vitess (MySQL Sharding)](https://vitess.io/)
