# CAP Theorem

## Learning Objectives
- Understand Consistency, Availability, and Partition Tolerance
- Learn why you must choose between C and A during a partition
- Identify CP vs AP systems in the real world

## Notes

### What is CAP Theorem?

In a **distributed system** with network partitions, you can only guarantee **2 out of 3**:

**C + A + P → Pick 2 (but P is required in distributed systems!)**

Real choice: **CP or AP**

---

### The Three Components

**Consistency (C):**
- All nodes see the same data at the same time
- After a write completes, ALL reads return the new value immediately
- Example: Write "balance = $500" → Everyone reads "$500" everywhere

**Availability (A):**
- Every request gets a response (success or failure)
- System never hangs or returns timeout
- Always operational, even during failures
- Example: System always responds, even if data might be stale

**Partition Tolerance (P):**
- System continues working even if network fails between nodes
- **NON-NEGOTIABLE** in distributed systems (networks WILL fail!)
- Must handle communication breakdowns between datacenters
- Example: US datacenter can't reach EU datacenter, system still works

---

### Real-World Analogy: The Bank Branch Problem

**Scenario:** 2 bank branches (US and Europe) with $1000 in your account. Network cable between them gets cut.

**Option 1: CP System (Consistency + Partition Tolerance)**
```
You: "Withdraw $500 in US branch"
Branch: "Sorry, we can't connect to Europe branch.
         We can't guarantee you have enough money.
         Come back later." ❌

Result:
- Consistent: Won't give wrong balance ✅
- NOT Available: Rejects your request ❌
```

**Option 2: AP System (Availability + Partition Tolerance)**
```
You: "Withdraw $500 in US branch"
US Branch: "Here's $500!" ✅

Your friend: "Withdraw $700 in Europe"
Europe Branch: "Here's $700!" ✅

(Both branches think balance is $1000)
(Network reconnects: OH NO! Balance is -$200!) 😱

Result:
- Available: Both requests succeeded ✅
- NOT Consistent: Branches had different data ❌
```

---

### Why Choose Between C and A?

**During a network partition:**
- Can't guarantee both consistency AND availability
- Must choose which to sacrifice

**The trade-off:**
- **Choose C (CP)**: Reject requests to maintain correctness
- **Choose A (AP)**: Accept requests but risk temporary inconsistency

**In practice:**
- Banking → Choose C (correctness matters)
- Social media → Choose A (availability matters)

---

### The Reality: CA is Impossible in Distributed Systems

**CA (Consistency + Availability) without Partition Tolerance:**
- Only works with a **single server** (no network = no partitions)
- Not distributed, can't scale
- Single point of failure

**Example:** SQLite on your laptop = CA
- Consistent ✅
- Available ✅
- But can't scale to multiple datacenters

---

### CP Systems (Consistency + Partition Tolerance)

**Philosophy:** "If I can't give you the right answer, I'll give you no answer"

**Examples:**
- **Banking systems** (PostgreSQL, MySQL with replication)
- **ATM withdrawals** (must have correct balance)
- **Zookeeper** (distributed coordination, leader election)
- **MongoDB** (CP by default with majority writes)
- **HBase** (strongly consistent reads)
- **Redis Cluster** (with wait command)

**Use cases:**
- Financial transactions (transfers, payments)
- Inventory management (last item in stock)
- Distributed locks (prevent duplicate job processing)
- Promo code validation (use once constraints)
- Booking systems (can't double-book)

**Behavior during partition:**
```
Write request arrives
↓
Can't reach majority of nodes ❌
↓
Return error: "Service temporarily unavailable"
↓
Data remains consistent (no partial writes)
```

**Real example - Bank transfer:**
```sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 'Alice';
UPDATE accounts SET balance = balance + 100 WHERE id = 'Bob';
COMMIT;

During partition:
→ Transaction blocks or fails ❌
→ Better than: Wrong balances! ✅
```

---

### AP Systems (Availability + Partition Tolerance)

**Philosophy:** "I'll always give you an answer, might not be perfect"

**Examples:**
- **Instagram/Facebook** (Cassandra for feeds, likes)
- **DynamoDB** (AP by default)
- **Cassandra** (tunable consistency, AP by default)
- **DNS** (cached responses, eventual propagation)
- **Riak** (high availability key-value store)
- **CouchDB** (multi-master replication)

**Use cases:**
- Social media feeds (stale feed OK)
- Like/view counters (approximate counts fine)
- Shopping carts (temporary inconsistency acceptable)
- User profiles (eventual consistency sufficient)
- Analytics dashboards (real-time not critical)
- Driver location tracking (slightly stale OK)

**Behavior during partition:**
```
Write request arrives
↓
Write to local datacenter ✅
↓
Return success immediately
↓
Sync to other datacenters in background (eventual consistency)
```

**Real example - Instagram Like:**
```
You like a post in US datacenter
→ US users see it: 0ms ✅
→ Europe users see it: 50ms ✅
→ Asia users see it: 150ms ✅
→ All consistent within 200ms (eventual!)
```

---

### When to Choose CP vs AP

| System | Choice | Why? |
|--------|--------|------|
| **Banking** | CP | Correctness > Availability |
| **ATM withdrawals** | CP | Can't give money twice |
| **Facebook Feed** | AP | Stale feed OK, downtime NOT OK |
| **Instagram Likes** | AP | Eventual consistency fine |
| **E-commerce checkout** | CP | Can't oversell |
| **Shopping cart** | AP | Temporary inconsistency OK |
| **Uber payments** | CP | Money must be exact |
| **Uber driver location** | AP | Stale location acceptable |
| **Promo codes** | CP | One-time use enforcement |
| **Zookeeper** | CP | Distributed locks need consistency |

---

### Premature Optimization Warning

**With 100 users:** Use single database (CA)
- No network partitions (single server)
- Simple PostgreSQL or MySQL
- Cost: $50/month

**With 1M users:** Consider CP vs AP
- Multiple datacenters needed
- Network partitions common
- Cost: $5,000+/month

**Key lesson:** Build for today's problems, not tomorrow's imaginary problems

**Real examples:**
- Twitter started with single MySQL database
- Instagram was on single PostgreSQL server for months
- They evolved architecture as they grew

---

### Configuration in Real Databases

Most databases let you **configure** CP vs AP per operation:

**MongoDB:**
```javascript
// CP - Strong Consistency
db.users.find().readConcern("majority")

// AP - Eventual Consistency
db.users.find().readConcern("local")
```

**Cassandra:**
```sql
-- AP
SELECT * FROM users CONSISTENCY LEVEL ONE;

-- CP
SELECT * FROM users CONSISTENCY LEVEL QUORUM;
```

**DynamoDB:**
```python
# AP
dynamodb.get_item(ConsistentRead=False)

# CP
dynamodb.get_item(ConsistentRead=True)
```

**Pattern:** Choose CP or AP **per feature**, not per system!


## Practice Questions

### Classify These Systems
For each system, decide if it should be CP or AP and explain why:

1. **Bank transactions (transferring money between accounts)**
   - CP or AP?
   - Why?

2. **Social media likes (liking a post on Instagram)**
   - CP or AP?
   - Why?

3. **Shopping cart (adding items to cart)**
   - CP or AP?
   - Why?

## Visual Diagram
Draw diagrams showing how a network partition affects a CP vs AP system.

**Your Diagrams:**


## Resources & References
