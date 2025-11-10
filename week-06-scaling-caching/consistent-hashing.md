# Consistent Hashing

## Learning Objectives
- Understand the consistent hashing algorithm
- Learn virtual nodes and hash rings
- Master use cases (CDN, distributed cache)

## Notes

### The Rehashing Problem

**Traditional Hashing:**
```python
server_id = hash(key) % num_servers

# With 3 servers:
hash("user_123") % 3 = 2 → Server 2
hash("user_456") % 3 = 0 → Server 0

# Add 1 server (now 4 total):
hash("user_123") % 4 = 3 → Server 3 ← MOVED!
hash("user_456") % 4 = 0 → Server 0 ← Same

# 75% of keys move to different servers! 😱
```

**Problem:** Adding/removing servers causes massive data reshuffling.

### Consistent Hashing Solution

**Key Idea:** Only ~1/N keys move when adding/removing a server (N = number of servers).

**How It Works: Hash Ring**

```
         0°
         ┌─┐
    270° │ │ 90°
         └─┘
        180°

1. Hash servers onto ring:
   Server A: hash("A") = 45°
   Server B: hash("B") = 135°
   Server C: hash("C") = 270°

2. Hash keys onto ring:
   Key "user_123": hash("user_123") = 60°

3. Walk clockwise to find server:
   60° → next server is B at 135°
   So "user_123" goes to Server B
```

### Your Redis Consistent Hashing (Score: 37/40 - 92%) 🏆

**Question:** Analyze consistent hashing for Redis cluster adding a new node.

**Your Answer:**
- Hash ring with 3 nodes
- Adding 4th node → only keys between new node and next node move
- Calculated ~25% data movement → Actually 1/N = 1/4 = 25%
- **Actually closer to 1/(N+1) = 1/4 = 25% for new node, but only ~1/N of total moves**

**Correct Math:**
```
Original: 3 nodes
Each node owns: 360° / 3 = 120° of ring

Add Node D at 200°:
- Node D claims space from 135° (Node B) to 200°
- That's 65° out of 360° = 18% of keys move

More precisely:
- Only keys in range [135°, 200°] move
- That's 1/3.3 ≈ 30% of Node B's keys
- Overall: ~20% of total keys move (not 80%!)
```

**Your Score: 92%** - Excellent understanding! Minor math detail but concept was perfect.

### Virtual Nodes

**Problem:** Real nodes might hash unevenly on ring, causing imbalance.

**Solution:** Each physical node gets multiple virtual positions.

```
Physical Node A → Virtual nodes: A1, A2, A3, ..., A128
Physical Node B → Virtual nodes: B1, B2, B3, ..., B128
Physical Node C → Virtual nodes: C1, C2, C3, ..., C128

Now 384 points on ring (128 × 3) → much more even distribution
```

**Benefits:**
- ✅ Even load distribution
- ✅ When node fails, its load spreads evenly to other nodes
- ✅ Can adjust weights (powerful nodes get more virtual nodes)

### Consistent Hashing Math

**Key Movement When Adding Server:**

Traditional hashing: `80%` of keys move (3→4 servers)
Consistent hashing: `~25%` of keys move (only keys in new server's range)

**Formula:**
```
Keys that move = 1 / (N + 1)

where N = original number of servers

Example:
3 servers → 4 servers
Keys moved = 1 / 4 = 25%
```

**Your calculation was spot on!** 🎯

## Practice Questions

### Question 1: Redis Cluster Consistent Hashing (Score: 37/40 - 92%)

**Your excellent analysis:**
- ✅ Understood hash ring concept
- ✅ Correct that only nearby keys move
- ✅ Calculated ~25% movement (close to actual ~20%)
- ✅ Compared to 80% with traditional hashing

**Minor improvement:** Exact percentage depends on where new node lands on ring, but your reasoning was perfect!

## Real-World Examples

### Amazon DynamoDB

```python
# Consistent hashing for partitioning
# Each partition owns range of hash ring
# Adding partition only affects adjacent ranges
```

### CDN (Content Delivery Network)

```python
# Akamai, Cloudflare use consistent hashing
# When CDN node fails, content redistributed evenly
# Users automatically routed to next closest node
```

### Cassandra

```python
# Consistent hashing with virtual nodes
# Each physical node = 256 virtual nodes (default)
# Even distribution even with different hardware
```

## Resources & References

- [Consistent Hashing Paper](https://www.akamai.com/us/en/multimedia/documents/technical-publication/consistent-hashing-and-random-trees-distributed-caching-protocols-for-relieving-hot-spots-on-the-world-wide-web-technical-publication.pdf)
- [DynamoDB Consistent Hashing](https://aws.amazon.com/blogs/database/amazon-dynamodb-deep-dive-advanced-design-patterns/)
- [Cassandra Virtual Nodes](https://cassandra.apache.org/doc/latest/architecture/vnodes.html)
