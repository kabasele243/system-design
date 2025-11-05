# Consistency Models

## Learning Objectives
- Understand Strong Consistency
- Understand Eventual Consistency
- Learn the trade-offs between them

## Notes

### The Consistency Spectrum

Consistency isn't binary - there's a **spectrum** from strong to eventual:

```
Strong Consistency (Slow, Always Correct)
    ↓
Bounded Staleness (Stale by max X seconds)
    ↓
Session Consistency (Your writes visible to you)
    ↓
Consistent Prefix (See writes in order)
    ↓
Eventual Consistency (Fast, Eventually Correct)
```

---

### 1. Strong Consistency (Linearizability)

**Definition:** After a write completes, ALL reads return the new value immediately, everywhere.

**Guarantee:** Every read sees the most recent write, always.

**Timeline:**
```
Time: 0ms  - Write: "likes = 100"
Time: 1ms  - Read in US: "likes = 100" ✅
Time: 1ms  - Read in EU: "likes = 100" ✅
Time: 1ms  - Read in Asia: "likes = 100" ✅

ALL reads see the same value immediately!
```

**How it works:**
1. Write goes to primary
2. Primary syncs to ALL replicas
3. Wait for majority confirmation ⏳
4. Only then return success ✅
5. Future reads guaranteed to see new value

**Real-world analogy:** Bank ATM
```
You deposit $100 at ATM A
Friend checks balance at ATM B (1 second later)
Friend MUST see the $100 deposit ✅
```

**Use cases:**
- Banking transactions
- Inventory management (last item in stock)
- Distributed locks
- Leader election
- Promo code validation

**Cost:** High latency (300-500ms cross-continent)

**Configuration example:**
```javascript
// MongoDB
db.users.find().readConcern("majority")

// DynamoDB
dynamodb.get_item(ConsistentRead=True)

// Cassandra
SELECT * FROM users CONSISTENCY LEVEL QUORUM;
```

---

### 2. Bounded Staleness

**Definition:** Reads might be stale, but never more than X seconds/versions old.

**Guarantee:** Data is at most X seconds or N versions behind.

**Timeline:**
```
Write: stock_price = $150
↓
User reads: "Price as of 2 seconds ago: $148"
Guaranteed: Never shows price older than 5 seconds
```

**Real-world analogy:** Stock prices
```
Apple stock price updates
You see price that's max 5 seconds old
Never older than 5 seconds ✅
```

**Configuration (Cosmos DB):**
```javascript
db.stocks.read({
    consistencyLevel: 'BoundedStaleness',
    maxStalenessSeconds: 5  // Max 5 seconds old
})
```

**Use cases:**
- Stock quotes
- Sports scores
- Real-time dashboards
- Monitoring systems
- Uber driver location (max 2 seconds stale)

**Cost:** Medium latency (50-100ms)

---

### 3. Session Consistency

**Definition:** YOUR writes are visible to YOU immediately. Others might see them later.

**Guarantee:** Read-your-writes + monotonic reads within a session.

**Timeline:**
```
User A writes: "Hello"
↓
User A reads: "Hello" ✅ (sees own write immediately)
User B reads: "" ⚠️ (hasn't synced yet)
User B reads (100ms later): "Hello" ✅ (now synced)
```

**Real-world analogy:** Editing Google Docs
```
You type "Hello"
YOU see it immediately ✅
Your friend sees it 200ms later ✅
But YOU always see your own changes!
```

**How it works:**
- Client gets session token
- Reads include session token
- Server ensures you see your own writes

**Use cases:**
- Shopping carts (you see your items immediately)
- Social media posts (you see your post right away)
- Comment sections
- User profiles
- Spotify "Recently Played"

**Cost:** Low latency (10-50ms)

**Configuration:**
```javascript
// Azure Cosmos DB
const session = await client.createSession();
db.cart.insert({...}, {sessionToken: session.token});
db.cart.read({sessionToken: session.token});  // See your write!
```

---

### 4. Consistent Prefix

**Definition:** You see writes in order, but might not see all writes yet.

**Guarantee:** If you see write B, you also see write A (if A happened before B).

**Timeline:**
```
Writes: A → B → C → D
↓
User 1 sees: A, B, C, D ✅ (all)
User 2 sees: A, B ✅ (partial but in order)
User 3 sees: A ✅ (partial but in order)

Never sees: B before A ❌
```

**Real-world analogy:** Group chat
```
Alice: "Want to get lunch?"
Bob: "Sure, where?"
Alice: "How about pizza?"

You might see:
- Just Alice's first message ✅
- Alice's first + Bob's reply ✅
- All three messages ✅

But NEVER:
- Bob's reply before Alice's question ❌
```

**Use cases:**
- Chat messages
- Social media timelines
- Event logs
- Audit trails
- Tweet threads

**Cost:** Low latency (5-20ms)

---

### 5. Eventual Consistency

**Definition:** Eventually all replicas converge to same value. No guarantees about timing or order.

**Guarantee:** Given enough time with no new writes, all replicas will converge.

**Timeline:**
```
Time: 0ms   - Write: "likes = 101"
Time: 1ms   - Read in US: "likes = 101" ✅
Time: 50ms  - Read in EU: "likes = 99" ⚠️ (still old!)
Time: 100ms - Read in EU: "likes = 101" ✅ (now updated!)
Time: 150ms - Read in Asia: "likes = 101" ✅

Eventually consistent! Just takes time to propagate.
```

**Real-world analogy:** Instagram likes
```
Post has 100 likes
You like it → 101
↓
Your phone: 101 ✅
Friend in NYC: 101 ✅ (50ms later)
Friend in Tokyo: 100 ⚠️ (still syncing)
Friend in Tokyo: 101 ✅ (200ms later)

Eventually everyone sees 101!
```

**How it works:**
```
Write: likes = 101
↓
Sync to replicas in background
↓
Different users see different values temporarily
↓
After X milliseconds: Everyone sees 101 ✅
```

**Use cases:**
- Social media likes/views
- DNS
- CDN caching
- Analytics dashboards
- Read-heavy systems
- Play counters

**Cost:** Minimal latency (1-5ms)

**Configuration:**
```javascript
// DynamoDB
dynamodb.get_item(ConsistentRead=False)  // Eventual

// Cassandra
SELECT * FROM users CONSISTENCY LEVEL ONE;  // Eventual

// MongoDB
db.users.find().readConcern("local")  // Eventual
```

---

## Trade-offs

| Aspect | Strong Consistency | Eventual Consistency |
|--------|-------------------|---------------------|
| **Latency** | High (300-500ms) ❌ | Low (1-5ms) ✅ |
| **Availability** | Lower (blocks during partition) ❌ | Higher (always responds) ✅ |
| **Complexity** | Simpler (always correct) ✅ | Complex (handle conflicts) ❌ |
| **Scalability** | Harder (coordination overhead) ❌ | Easier (independent writes) ✅ |
| **Cost** | Expensive (cross-region sync) ❌ | Cheap (local writes) ✅ |
| **Use Cases** | Banking, payments, inventory | Social media, analytics, DNS |
| **User Experience** | Slower but always correct | Faster but occasionally stale |

---

## Visual Comparison

```
STRONG CONSISTENCY
Timeline: 0ms -------- 300ms -------- 600ms
Write:    ●
Read US:               ✅ (101)      ✅ (101)
Read EU:               ✅ (101)      ✅ (101)
Read Asia:             ✅ (101)      ✅ (101)
All see same value immediately!
Latency: 300ms ❌


SESSION CONSISTENCY
Timeline: 0ms -- 50ms -- 100ms -- 150ms
Write:    ● (User A)
User A:   ✅       ✅        ✅        ✅
          (101)   (101)     (101)     (101)
User B:   ⚠️       ⚠️        ✅        ✅
          (100)   (100)     (101)     (101)
Your writes visible to YOU immediately!
Latency: 5ms ✅


EVENTUAL CONSISTENCY
Timeline: 0ms -- 10ms -- 50ms -- 150ms -- 300ms
Write:    ●
Read US:  ✅      ✅       ✅       ✅        ✅
          (101)  (101)    (101)    (101)     (101)
Read EU:  ⚠️      ⚠️       ✅       ✅        ✅
          (100)  (100)    (101)    (101)     (101)
Read Asia:⚠️      ⚠️       ⚠️       ✅        ✅
          (100)  (100)    (100)    (101)     (101)
Eventually consistent!
Latency: 1ms ✅✅
```

---

## Real Systems and Their Choices

| System | Consistency Level | Why? |
|--------|------------------|------|
| **Bank transfers** | Strong | Money must be exact |
| **Stock trading** | Bounded (5 sec) | Recent price OK, not stale |
| **Shopping cart** | Session | You see your items |
| **Twitter timeline** | Consistent Prefix | See tweets in order |
| **Instagram likes** | Eventual | Who cares if count is off? |
| **Google Docs** | Session + Eventual | Your edits immediate, others eventual |
| **DNS** | Eventual | Caching everywhere |
| **Uber driver location** | Bounded (2 sec) | Recent location OK |
| **Payment processing** | Strong | Must be correct |
| **Spotify play counter** | Eventual | Approximate count fine |
| **Spotify subscription** | Strong | Money involved |
| **Spotify Recently Played** | Session | See your plays immediately |
| **Spotify Monthly Listeners** | Bounded (5 min) | Reasonably fresh |

## Practice Questions
1. When would you choose eventual consistency over strong consistency?
2. What problems can eventual consistency cause?
3. How does social media handle consistency for likes/views?

## Real-World Examples


## Resources & References
