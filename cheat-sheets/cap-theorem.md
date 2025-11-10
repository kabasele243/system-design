# CAP Theorem Cheat Sheet

## The Three Properties

```
        Consistency
           /  \
          /    \
         /      \
        /        \
   Partition  Availability
   Tolerance
```

**Pick 2 out of 3 during a network partition**

### Consistency (C)
- All nodes see the same data at the same time
- Every read receives the most recent write
- Strong consistency guarantee

### Availability (A)
- Every request receives a response (success or failure)
- System remains operational
- May not be the most recent data

### Partition Tolerance (P)
- System continues despite network failures
- Required for distributed systems
- You MUST support partitions in real-world systems

## The Reality: CP vs AP

Since partitions will happen in distributed systems, you must choose:

### CP Systems (Consistency + Partition Tolerance)
**Choose Consistency:** Reject requests if cannot guarantee latest data

**Examples:**
- Bank transactions
- Inventory systems
- Booking systems
- MongoDB (default)
- HBase
- Redis Cluster (with wait)

**Use when:**
- Data accuracy is critical
- Incorrect data has serious consequences
- Prefer to show error than stale data

### AP Systems (Availability + Partition Tolerance)
**Choose Availability:** Always respond, even with stale data

**Examples:**
- Social media likes/views
- Shopping cart (pre-checkout)
- DNS
- Cassandra
- DynamoDB (eventual consistency)
- Riak

**Use when:**
- Uptime is critical
- Stale data is acceptable temporarily
- Eventual consistency is fine

## Quick Decision Guide

| Scenario | CP or AP? | Why? |
|----------|-----------|------|
| Bank withdrawal | CP | Cannot have duplicate/wrong amounts |
| Social media like | AP | Okay if like count is delayed |
| Flight booking | CP | Cannot double-book seats |
| Shopping cart | AP | Okay if cart syncs eventually |
| Password change | CP | Must be consistent immediately |
| News feed | AP | Okay if post appears with delay |
| Payment processing | CP | Critical to be accurate |
| Recommendation system | AP | Stale recommendations are fine |

## Common Patterns

### CP System Pattern (Consistent but may be unavailable)
```
During network partition:
- Return error to client
- Wait until partition heals
- Guarantee consistency
```

### AP System Pattern (Available but may be inconsistent)
```
During network partition:
- Accept writes on both sides
- Return success to client
- Resolve conflicts later (eventual consistency)
```

## Eventual Consistency Patterns

### Conflict Resolution Strategies:
1. **Last Write Wins (LWW):** Use timestamp
2. **Version Vector:** Track causality
3. **Application-level:** Let app decide
4. **Merge:** Combine both versions

## Trade-offs

| Aspect | CP Systems | AP Systems |
|--------|-----------|------------|
| Consistency | Strong | Eventual |
| Availability | Lower during partition | Higher |
| Latency | May be higher | Usually lower |
| Complexity | Simpler conflict handling | Complex conflict resolution |
| Use cases | Financial, booking | Social, caching |

## Remember

1. **Partition tolerance is mandatory** for distributed systems
2. **You must choose between C and A** during network partition
3. **When no partition:** You can have all three (CA)
4. **It's a spectrum:** Not binary, degrees of consistency/availability
5. **Per-operation choice:** Can mix CP and AP in same system

## Interview Tips

- Always ask: "Does this data need strong consistency?"
- Consider consequences of stale data
- Discuss trade-offs explicitly
- Real systems often use hybrid approaches
