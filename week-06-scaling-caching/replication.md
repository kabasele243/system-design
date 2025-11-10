# Primary-Secondary Replication

## Learning Objectives
- Understand primary-secondary replication patterns
- Learn synchronous vs asynchronous replication
- Master failover strategies

## Notes

### What is Replication?

**Replication** = Keeping copies of the same data on multiple machines.

**Why Replicate?**
- ✅ High availability (if primary fails, replica takes over)
- ✅ Read scalability (distribute reads across replicas)
- ✅ Reduced latency (place replicas geographically closer to users)
- ✅ Disaster recovery (backup data)

### Primary-Secondary Architecture

```
        ┌──────────┐
        │ Primary  │ ← All WRITES go here
        │ Database │
        └────┬─────┘
             │
    ┌────────┼────────┐
    │        │        │
    ▼        ▼        ▼
┌────────┐ ┌────────┐ ┌────────┐
│Replica │ │Replica │ │Replica │ ← READS distributed here
│   1    │ │   2    │ │   3    │
└────────┘ └────────┘ └────────┘
```

**Key Rules:**
- ONE primary (accepts writes)
- MULTIPLE replicas (read-only)
- Primary replicates changes to replicas

### Synchronous vs Asynchronous Replication

#### Synchronous Replication

**How it works:**
```
Client → Primary: Write data
Primary → Replica: Replicate data
Primary ← Replica: ACK (data written)
Client ← Primary: Success (after replica confirms)
```

**Pros:**
- ✅ Strong consistency (replica always up-to-date)
- ✅ No data loss if primary fails

**Cons:**
- ❌ Slow writes (wait for replica)
- ❌ Availability issues (if replica is down, writes fail)

**Use Cases:**
- Banking systems (can't lose transactions)
- Financial records
- Critical data where consistency > availability

#### Asynchronous Replication (Most Common)

**How it works:**
```
Client → Primary: Write data
Client ← Primary: Success (immediately)
Primary → Replica: Replicate data (in background)
```

**Pros:**
- ✅ Fast writes (don't wait for replicas)
- ✅ High availability (writes succeed even if replicas down)

**Cons:**
- ❌ Replication lag (replicas might be behind)
- ❌ Potential data loss (if primary fails before replication)

**Use Cases:**
- Social media (Twitter, Instagram)
- Content platforms (Medium, YouTube)
- Most web applications

### Replication Lag

**Problem:** Replicas might be seconds/minutes behind primary.

**Example:**
```
Time 0: User posts tweet on Primary
Time 1: User refreshes page, reads from Replica
        → Replica hasn't received tweet yet
        → User doesn't see their own tweet! 😱
```

### Read-Your-Writes Consistency

**Solution:** Ensure user sees their own writes immediately.

**Pattern 1: Read from Primary for Own Data**
```python
def get_user_profile(user_id, requesting_user_id):
    if user_id == requesting_user_id:
        # User viewing their own profile → read from PRIMARY
        return primary_db.get_user(user_id)
    else:
        # Viewing someone else's profile → read from REPLICA
        return replica_db.get_user(user_id)
```

**Pattern 2: Track Last Write Timestamp**
```python
def write_post(user_id, content):
    primary_db.insert(content)
    user_session['last_write_timestamp'] = time.now()

def get_feed(user_id):
    last_write = user_session.get('last_write_timestamp')

    if last_write and (time.now() - last_write < 5):
        # Recent write → read from PRIMARY for 5 seconds
        return primary_db.get_feed(user_id)
    else:
        # Old write or no writes → read from REPLICA
        return replica_db.get_feed(user_id)
```

**Pattern 3: Monotonic Reads**
```python
# Ensure user doesn't see data go "backwards in time"
# Stick same user to same replica

def get_replica_for_user(user_id):
    replica_index = hash(user_id) % num_replicas
    return replicas[replica_index]

# User always reads from same replica → no time travel!
```

### Your Twitter Replication Design (Score: 25/40 - 62%)

**Your Answer:**
- Asynchronous replication (correct choice!)
- Read-your-writes pattern (good!)
- Mentioned eventual consistency

**What Was Missing:**
- Specific replication lag handling (how long to wait?)
- Failover strategy
- Calculations (how many replicas needed?)
- Geographic distribution of replicas

**What You Got Right:**
- ✅ Async replication for performance
- ✅ Read-your-writes consistency pattern
- ✅ Understanding of eventual consistency trade-off

### Failover Strategies

**Failover** = When primary fails, promote a replica to be new primary.

#### Automatic Failover

```
1. Primary database crashes
2. Monitoring detects failure (health checks timeout)
3. Consensus algorithm elects new primary (Raft, Paxos)
4. Replica promoted to primary
5. Application redirected to new primary
```

**Time:** 30 seconds - 2 minutes downtime

**Challenges:**
- Split-brain problem (two primaries)
- Data loss if async replication (un-replicated writes lost)

#### Manual Failover

```
1. Primary fails
2. Operations team notified
3. Human reviews situation
4. Manually promotes replica
5. Updates DNS/load balancer
```

**Time:** Minutes to hours

**Pros:**
- No split-brain risk
- Human judgment

**Cons:**
- Slow
- Requires on-call team

### Multi-Region Replication

**Netflix Example:**
```
US-East (Primary)
  ├─ Replica 1 (US-East)
  ├─ Replica 2 (US-West) ← 80ms replication lag
  └─ Replica 3 (EU) ← 150ms replication lag

Users read from nearest replica for low latency
```

**Trade-offs:**
- ✅ Low read latency (geographic proximity)
- ❌ Higher replication lag (cross-region network)
- ❌ More complex failover

## Practice Questions

### Question 1: Twitter Replication (Your Score: 25/40 - 62%)

**What You Could Improve:**

**Complete Answer Should Include:**

1. **Replication Strategy:**
   - Asynchronous replication (you got this! ✅)
   - 3 replicas for redundancy
   - Max replication lag: 5 seconds acceptable

2. **Read-Your-Writes:**
```python
def post_tweet(user_id, content):
    primary.insert(tweet)
    redis.setex(f"recent_write:{user_id}", 5, "true")

def get_home_timeline(user_id):
    if redis.exists(f"recent_write:{user_id}"):
        # Read from primary for 5 seconds after write
        return primary.get_timeline(user_id)
    else:
        # Read from replica
        return replica.get_timeline(user_id)
```

3. **Calculations:**
```
Reads: 100M/day = 1,157 reads/sec
Writes: 10M tweets/day = 116 writes/sec

Primary handles: 116 writes/sec
Each replica: 1,157 / 3 = 386 reads/sec (easily handled)

Need: 1 primary + 3 replicas
```

4. **Failover:**
- Automatic failover with 30-second detection
- Accept data loss of last 5 seconds
- Use consensus protocol (Raft)

## Real-World Examples

### Instagram Photo Replication

```
Primary (US-East):
  - All photo uploads
  - All likes, comments writes

Replicas:
  - US-West: Serves western US users
  - EU: Serves European users
  - Asia: Serves Asian users

Async replication with 1-2 second lag
Users see their own photos immediately (read from primary)
```

### YouTube Video Metadata

```
Primary Database (MySQL):
  - Video uploads metadata
  - User accounts
  - Comments

Read Replicas (5 replicas):
  - Video search queries
  - Homepage recommendations
  - User profile views

Replication lag: 0.5-2 seconds acceptable
```

### E-commerce Orders

```
Primary:
  - Order placement (ACID critical)
  - Payment processing
  - Inventory updates

Replicas:
  - Order history viewing
  - Analytics queries
  - Customer service lookups

Synchronous replication to 1 replica (no data loss)
Asynchronous to others (for analytics)
```

## Resources & References

- [MySQL Replication](https://dev.mysql.com/doc/refman/8.0/en/replication.html)
- [PostgreSQL Streaming Replication](https://www.postgresql.org/docs/current/warm-standby.html)
- [Designing Data-Intensive Applications](https://dataintensive.net/) - Chapter 5
- [Read-Your-Writes Consistency](https://jepsen.io/consistency/models/read-your-writes)
