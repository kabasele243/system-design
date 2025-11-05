# ACID vs BASE

## Learning Objectives
- Understand ACID properties for databases
- Understand BASE properties
- Learn when to use each

## Notes

### ACID vs BASE: Two Philosophies

**ACID** = Traditional SQL databases (CP systems)
- Philosophy: "Do it right or don't do it at all"
- Focus: Correctness first

**BASE** = Modern NoSQL databases (AP systems)
- Philosophy: "Availability first, consistency eventually"
- Focus: Availability first

---

### ACID (Relational Databases)

ACID ensures transactions are processed reliably.

**A - Atomicity:**

**Definition:** All or nothing - Transaction either completes fully or rolls back completely.

**Bank transfer example:**
```sql
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 'Alice';
  UPDATE accounts SET balance = balance + 100 WHERE id = 'Bob';
COMMIT;

If power goes out after first UPDATE:
→ Both changes are rolled back ✅
→ Money doesn't disappear! ✅
```

**Real-world:** If any part of transaction fails, entire transaction is undone.
- Transfer $100: Either both accounts update OR neither updates
- Never: Alice loses $100 but Bob doesn't gain $100 ❌

---

**C - Consistency:**

**Definition:** Data must be valid - All rules (constraints, triggers, cascades) are enforced.

**E-commerce example:**
```sql
-- Rule: Can't have negative inventory
UPDATE products
SET inventory = inventory - 5
WHERE id = 'iPhone';

If inventory = 3:
→ Transaction fails ❌
→ Can't go to -2 inventory ✅
```

**Real-world:** Database enforces all constraints
- Foreign keys must exist
- Unique constraints maintained
- Check constraints validated
- Triggers executed

---

**I - Isolation:**

**Definition:** Transactions don't interfere - Concurrent transactions act like they're running alone.

**Concert tickets example:**
```
Last ticket for Taylor Swift concert

User A: "Buy ticket" (starts transaction)
User B: "Buy ticket" (starts transaction at same time)

With Isolation:
→ User A sees 1 ticket, buys it ✅
→ User B sees 0 tickets (blocked until A commits) ✅
→ Only 1 ticket sold ✅

Without Isolation:
→ Both see 1 ticket
→ Both buy it ❌
→ 2 tickets sold from 1 available! ❌
```

**Isolation Levels:**
1. **Read Uncommitted** (lowest) - Can see uncommitted changes
2. **Read Committed** - Only see committed changes
3. **Repeatable Read** - Same query returns same result
4. **Serializable** (highest) - Fully isolated, like running one at a time

**Real-world:** Promo code "use once"
```sql
SELECT * FROM promo_codes WHERE code = 'SUMMER' FOR UPDATE;  -- LOCK IT!
-- No one else can use it until we're done
UPDATE promo_codes SET used = true WHERE code = 'SUMMER';
COMMIT;  -- Release lock
```

---

**D - Durability:**

**Definition:** Committed = permanent - Once transaction commits, data survives crashes, power failures.

**Example:**
```
You: "Transfer $1000 to savings"
Bank: "Transaction committed!" ✅
(Server crashes 1 second later)
(Server reboots)
→ Transfer is still there ✅
→ Saved to disk, not just RAM
```

**How it works:**
- Write-Ahead Logging (WAL): Log changes before applying them
- Fsync: Force write to physical disk
- Replication: Copy to multiple disks/servers

**Real-world:** Your payment goes through, server crashes
- Payment still recorded ✅
- Never lost ✅
- Recovered from logs

---

### BASE (NoSQL Databases)

BASE prioritizes availability over consistency.

**BA - Basically Available:**

**Definition:** System always responds - Might return stale data or error, but doesn't hang.

**Instagram example:**
```
Load your feed during network issues
→ Shows cached feed (5 minutes old) ✅
→ Better than: Loading spinner forever ❌
```

**Real-world:** Rather than blocking, system returns:
- Stale data (from cache)
- Partial results
- Graceful degradation
- Error message (but fast!)

**Not available:**
```
User: "Load feed"
System: *spinning... spinning... timeout* ❌
```

**Basically available:**
```
User: "Load feed"
System: "Here's cached feed from 2 min ago" ✅
```

---

**S - Soft state:**

**Definition:** State may change over time - Even without new writes (due to eventual consistency).

**Example:**
```
Time 0:   Post has 100 likes (your view)
Time 50ms: Post has 102 likes (system syncing)
Time 100ms: Post has 102 likes (your view updates)

Your view changed without YOU doing anything!
This is "soft state"
```

**Real-world:** Your data can "drift" as replicas sync
- Not a bug, it's by design!
- Eventually converges to correct state
- Temporary inconsistency is expected

**Contrast with ACID:**
- ACID: State only changes with explicit writes
- BASE: State changes as system syncs

---

**E - Eventual consistency:**

**Definition:** Eventually all replicas converge - But temporarily might differ.

**Twitter example:**
```
You tweet "Hello World!"
→ Your followers in US see it: 10ms ✅
→ Your followers in EU see it: 50ms ✅
→ Your followers in Asia see it: 150ms ✅

Eventually consistent!
```

**Real-world:** After you write:
- Local datacenter sees it immediately
- Other datacenters see it eventually (50-200ms)
- Given enough time, everyone sees same value

---

## Comparison

| Property | ACID | BASE |
|----------|------|------|
| **Philosophy** | Correctness first | Availability first |
| **Consistency** | Strong (immediate) | Eventual (delayed) |
| **Transactions** | Multi-row, all-or-nothing | Single-row, best-effort |
| **Failures** | Abort and rollback | Return partial results |
| **Performance** | Slower (locking, waiting) | Faster (no waiting) |
| **Scalability** | Harder to scale | Easy to scale |
| **Database Type** | SQL (PostgreSQL, MySQL) | NoSQL (Cassandra, DynamoDB) |
| **Use Cases** | Banking, payments, inventory | Social media, analytics, logs |
| **Latency** | High (100-500ms) | Low (1-10ms) |
| **Cost** | Expensive (coordination) | Cheap (independent writes) |

---

## Side-by-Side Example: E-commerce Checkout

### ACID Approach (Traditional SQL)

```sql
BEGIN TRANSACTION;

-- Check inventory (LOCK THE ROW!)
SELECT inventory FROM products WHERE id = 'iPhone' FOR UPDATE;

-- Decrement inventory
UPDATE products SET inventory = inventory - 1 WHERE id = 'iPhone';

-- Create order
INSERT INTO orders (user_id, product_id, amount) VALUES (...);

-- Charge credit card
INSERT INTO payments (order_id, amount, status) VALUES (...);

COMMIT;
-- If ANYTHING fails → Everything rolls back
```

**Pros:**
- ✅ Inventory never oversold
- ✅ Order and payment always match
- ✅ Data always consistent

**Cons:**
- ❌ Slow (locking, waiting)
- ❌ Doesn't scale to millions of users
- ❌ Single point of failure

---

### BASE Approach (NoSQL)

```javascript
// Step 1: Optimistically reserve inventory (no locks!)
await db.inventory.decrement({productId: 'iPhone'});

// Step 2: Create order (might succeed even if inventory wrong!)
await db.orders.create({userId, productId, amount});

// Step 3: Charge card (separate operation)
await payments.charge({orderId, amount});

// Step 4: Background job checks consistency
async function reconcileOrders() {
    const oversold = await findOversoldProducts();
    for (const product of oversold) {
        // Cancel recent orders, refund customers
        await cancelAndRefund(product);
    }
}
```

**Pros:**
- ✅ Fast (no locking)
- ✅ Scales to millions of users
- ✅ High availability

**Cons:**
- ❌ Might oversell temporarily
- ❌ Need background jobs to fix inconsistencies
- ❌ More complex error handling

---

## Real Companies' Choices

### ACID Systems

| Company | Use Case | Database | Why? |
|---------|----------|----------|------|
| **Stripe** | Payment processing | PostgreSQL | Money must be exact |
| **Shopify** | E-commerce checkout | MySQL | Can't oversell inventory |
| **Airbnb** | Booking system | MySQL | Can't double-book |
| **Banks** | Transactions | Oracle/PostgreSQL | Correctness critical |

### BASE Systems

| Company | Use Case | Database | Why? |
|---------|----------|----------|------|
| **Instagram** | Feed, likes | Cassandra | High availability needed |
| **Netflix** | Viewing analytics | Cassandra | Approximate counts fine |
| **WhatsApp** | Messages | Cassandra | Eventual delivery OK |
| **Uber** | Driver locations | Cassandra | Stale location acceptable |

### Hybrid Approach (Most Common!)

**Amazon:**
```
Shopping Cart → BASE (DynamoDB)
- Fast, available
- Temporary inconsistencies OK

Checkout → ACID (RDS/PostgreSQL)
- Slow, consistent
- Must be correct!

Product Catalog → BASE (DynamoDB)
- Fast reads
- Eventual consistency fine

Order History → BASE (DynamoDB)
- Read-heavy
- Stale data acceptable
```

**Uber:**
```
Driver Location → BASE (Cassandra)
- High write throughput
- Eventual consistency OK

Trip Payments → ACID (PostgreSQL)
- Money must be exact

Trip History → BASE (Cassandra)
- Massive read volume
- Stale OK

Promo Codes → ACID (PostgreSQL)
- One-time use enforcement
```

---

## When to Choose ACID vs BASE

### Choose ACID when:
- ✅ Money involved (payments, transfers)
- ✅ Inventory management (can't oversell)
- ✅ One-time constraints (promo codes, bookings)
- ✅ Correctness > Speed
- ✅ Small to medium scale (< 10M users)

### Choose BASE when:
- ✅ Social media (likes, views, feeds)
- ✅ Analytics (approximate counts fine)
- ✅ High write volume (logs, events)
- ✅ Speed > Perfect consistency
- ✅ Massive scale (100M+ users)

### Use Both (Hybrid):
- ✅ Critical data → ACID
- ✅ Everything else → BASE
- ✅ Most production systems!

---

## Key Takeaway

**Modern systems use BOTH:**

```javascript
class App {
    // BASE - Driver location (NoSQL)
    async updateDriverLocation(driverId, lat, lng) {
        await cassandra.execute(
            'INSERT INTO driver_locations ...'
        );
        // No transaction, fire and forget!
    }

    // ACID - Payment processing (SQL)
    async processPayment(tripId, amount) {
        const transaction = await postgres.transaction();
        try {
            await transaction.query('SELECT * FROM trips ... FOR UPDATE');
            await transaction.query('INSERT INTO payments ...');
            await transaction.query('UPDATE driver_earnings ...');
            await transaction.commit();  // All or nothing!
        } catch (error) {
            await transaction.rollback();  // Undo everything!
            throw error;
        }
    }
}
```

**Pattern:** Choose ACID or BASE **per feature**, based on requirements!

## Practice Questions
1. Why do banks use ACID databases?
2. When would you choose BASE over ACID?
3. Can you have ACID properties in a distributed system?

## Real-World Examples


## Resources & References
