# Kafka vs Traditional Queue

## Learning Objectives
- Understand Kafka architecture (topics, partitions, offsets)
- Learn when to use Kafka vs RabbitMQ/SQS
- Master consumer groups and replayability

## Notes

### What is Kafka?

**Apache Kafka** is an **event streaming platform**, NOT a traditional message queue.

**Key Philosophy:**
- Events are **facts** that happened (immutable)
- Events are **stored** for replay (like a database)
- Multiple consumers can read the **same events**

### Kafka vs Traditional Queue

| Feature | Traditional Queue (SQS) | Kafka |
|---------|-------------------------|-------|
| **Message lifetime** | Deleted after consumed | Retained (days/years) |
| **Consumption** | Destructive (message gone) | Non-destructive (offset-based) |
| **Replay** | ❌ No | ✅ Yes - rewind to any offset |
| **Ordering** | Within queue | Within partition |
| **Speed** | Good (~100 msg/sec) | Excellent (~millions/sec) |
| **Use case** | Task distribution | Event streaming, audit logs |
| **Retention** | 1 min - 14 days | 1 day - forever |
| **Consumer independence** | Shared queue | Independent offsets per consumer |

### Kafka Architecture

```
Topic: "user-events"
├── Partition 0: [msg0, msg1, msg2, msg3, msg4] ← offset 4
├── Partition 1: [msg0, msg1, msg2, msg3] ← offset 3
└── Partition 2: [msg0, msg1] ← offset 1
```

**Key Components:**

#### 1. Topics
- Named category for events (e.g., "orders", "user-signups")
- Divided into partitions for parallelism

#### 2. Partitions
- Ordered, immutable sequence of events
- Each message has an **offset** (position)
- Enables parallel processing
- **Ordering guaranteed ONLY within a partition**

#### 3. Offsets
- **Your bookmark** in the event stream
- Each consumer tracks their own offset
- Can rewind and replay from any offset

#### 4. Partition Keys
- Determines which partition a message goes to
- **Same key → same partition → guaranteed order**
- Example: `user_id` as key ensures all events for one user are ordered

```python
# Producer example
kafka_producer.send(
    topic='messages',
    key=conversation_id.encode('utf-8'),  # Same conversation → same partition
    value=message_data
)
```

### Consumer Groups

**Kafka's killer feature for scalability:**

```
Topic: "orders" (4 partitions)

Consumer Group "order-processors":
  Consumer 1 → reads Partition 0, 1
  Consumer 2 → reads Partition 2, 3

Consumer Group "analytics":
  Consumer 1 → reads Partition 0, 1, 2, 3

Consumer Group "fraud-detection":
  Consumer 1 → reads Partition 0, 1, 2, 3
```

**Rules:**
1. **Within a consumer group**: Each partition assigned to ONE consumer (work distribution)
2. **Across consumer groups**: All groups read same events independently (broadcast)
3. **Rebalancing**: If consumer dies, partitions reassigned

**Your Insight (97.5% score):**
> "Kafka for order matching queue with offset-based replay for audit compliance"

Perfect! You understood:
- Offset tracking for regulatory compliance
- Replay capability for audits
- Partition key for ordering

### When to Use Kafka

**Use Cases:**
- ✅ Event sourcing (store every state change)
- ✅ Audit logs (need replay capability)
- ✅ Real-time analytics (multiple consumers process same data)
- ✅ Change Data Capture (CDC) from databases
- ✅ Activity tracking (user clicks, page views)
- ✅ Log aggregation from microservices

**Don't Use When:**
- ❌ Simple task queue (SQS is simpler)
- ❌ Request-response pattern (use HTTP/gRPC)
- ❌ Low volume (<100 msg/sec) (overkill, use SQS)
- ❌ Don't need replay (SQS cheaper)

### Kafka Guarantees

#### Ordering
- ✅ **Guaranteed within a partition**
- ❌ **NOT guaranteed across partitions**

**Solution:** Use partition key
```python
# All messages for user_123 → same partition → ordered
kafka.send(topic='events', key='user_123', value=data)
```

#### Delivery Semantics
1. **At-Most-Once**: Send and forget (may lose)
2. **At-Least-Once**: Retry until ack (may duplicate)
3. **Exactly-Once**: Idempotent producer + transactional consumer (expensive)

### Real-Time Chat Example (Your 100% Answer!)

**Your Architecture:**
```
Producer (Sender)
    ↓
Kafka Topic "messages" (source of truth)
    ↓
[Parallel writes]
    ├→ WebSocket Service (real-time delivery)
    └→ History Service (stores in Cassandra)
```

**Why Kafka here?**
1. **Durability**: All messages persisted
2. **Replay**: Can rebuild history if Cassandra fails
3. **Ordering**: Partition by `conversation_id`
4. **Multiple consumers**: WebSocket service + History service read same events
5. **Audit trail**: Compliance requirement

**Brilliant insight:** Parallel write to WebSocket + Kafka for speed + durability

### Message Retention

**Kafka Flexibility:**
```python
# Retention policies
retention.ms = 604800000  # 7 days
retention.bytes = 1073741824  # 1 GB

# Or infinite retention
retention.ms = -1  # Keep forever
```

**Use cases:**
- **7 days**: User activity logs
- **30 days**: Order events
- **1 year**: Financial transactions
- **Forever**: Audit logs

### Consumer Offset Management

```python
# Consumer tracks offset
while True:
    messages = consumer.poll(timeout=1.0)
    for message in messages:
        process(message)
        consumer.commit()  # Save offset checkpoint

# Can manually control offset
consumer.seek(partition, offset=100)  # Rewind to offset 100
```

### Kafka vs SQS vs SNS Decision Tree

**Choose SQS when:**
- Simple task queue
- Work distribution
- Low volume (<1000 msg/sec)
- Don't need replay

**Choose SNS when:**
- Broadcast to multiple services
- Fan-out pattern
- Don't need persistence

**Choose Kafka when:**
- Need event replay
- High throughput (>10K msg/sec)
- Multiple independent consumers
- Audit requirements
- Event sourcing architecture

## Practice Questions

### Question 1: Trading Platform (Score: 39/40 - 97.5%) 🏆

**Your Answer Highlights:**
- Kafka for order matching with offset-based replay
- Partition by `user_id` for ordering
- Consumer groups for parallel processing
- Redis cache for portfolio reads
- Excellent reasoning: "Replayability for regulatory compliance"

**What made this answer exceptional:**
- Understood Kafka's audit trail capability
- Correct partition key choice
- Recognized when NOT to use Kafka (used Redis for reads)

## Real-World Examples

### Uber Trip Events
```
Trip Service → Kafka Topic "trip-events"
             ↓
   Consumer Group "pricing" (calculate surge)
   Consumer Group "analytics" (track metrics)
   Consumer Group "ml-training" (model updates)
   Consumer Group "accounting" (driver payouts)

All groups read same events independently!
```

### Netflix Video Analytics
```
Video Player → Kafka Topic "playback-events"
             ↓
   Consumer Group "recommendations" (what to suggest next)
   Consumer Group "quality" (detect buffering issues)
   Consumer Group "billing" (track watch time)
   Consumer Group "content-team" (what shows are popular)

Retention: 90 days for replay and analysis
```

### LinkedIn Activity Feed
```
User Action → Kafka Topic "user-activities"
   Partition key: user_id (maintains order per user)
             ↓
   Consumer Group "feed-builder" (create personalized feed)
   Consumer Group "notifications" (send alerts)
   Consumer Group "analytics" (track engagement)
   Consumer Group "search-indexer" (update search)
```

### Financial Trading (Your Example!)
```
Trade Service → Kafka Topic "orders"
   Partition key: user_id (FIFO per user)
   Retention: 7 years (regulatory requirement)
             ↓
   Consumer Group "matching-engine" (execute trades)
   Consumer Group "audit" (compliance checking)
   Consumer Group "risk" (monitor exposure)
   Consumer Group "reporting" (generate statements)

Why Kafka?
- Offset-based replay for audits
- Immutable event log
- Can rebuild state from events
```

## Resources & References

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Kafka: The Definitive Guide](https://www.confluent.io/resources/kafka-the-definitive-guide/)
- [Event Sourcing with Kafka](https://www.confluent.io/blog/event-sourcing-cqrs-stream-processing-apache-kafka-whats-connection/)
- [Consumer Groups Explained](https://kafka.apache.org/documentation/#consumergroups)
