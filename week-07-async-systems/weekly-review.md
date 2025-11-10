# Week 7: Asynchronous Systems - Weekly Review

## Review Questions

### Question 1: Netflix Video Processing (40 points)

**Problem:**
Design a video processing pipeline for Netflix that handles user-uploaded videos. Requirements:
- 10,000 videos uploaded per day
- Each video takes ~30 minutes to process (transcoding, thumbnails, metadata)
- API must respond to upload request in <500ms
- Need to handle traffic spikes (10x normal load)

**Tasks:**
1. Design the architecture using message queues
2. Calculate worker requirements
3. Explain idempotency strategy
4. Handle failures (DLQ strategy)

**Model Answer (40/40):**

**Architecture:**
```
1. User uploads video → API Gateway → S3
2. S3 event → SNS Topic "VideoUploaded"
3. SNS fans out to multiple SQS queues:
   - Transcoding Queue (4K, 1080p, 720p, 480p)
   - Thumbnail Queue (generate 10 thumbnails)
   - Metadata Queue (extract duration, resolution, etc.)
   - Database Queue (create video record)
```

**Worker Calculations:**
```
Videos per day: 10,000
Processing time per video: 30 minutes
Hours per day: 24

Total processing hours needed: 10,000 × 0.5 = 5,000 hours/day
Workers needed: 5,000 / 24 = ~210 workers minimum

With 10x traffic spikes: 2,100 workers peak
Use auto-scaling: 230 base workers, scale to 5,000 workers
```

**Idempotency Strategy:**
```python
def process_video(message):
    video_id = message['video_id']

    # Check DynamoDB for processing status
    status = dynamodb.get_item(
        Table='VideoProcessing',
        Key={'video_id': video_id}
    )

    if status['state'] == 'COMPLETED':
        return "Already processed"

    # Process video
    transcode_video(video_id)

    # Mark as completed
    dynamodb.put_item(
        Table='VideoProcessing',
        Item={'video_id': video_id, 'state': 'COMPLETED'}
    )

    # Delete message from queue
    sqs.delete_message(message)
```

**Failure Handling:**
```
1. SQS visibility timeout: 35 minutes (processing time + buffer)
2. Max retries: 3 attempts
3. Dead Letter Queue (DLQ) strategy:
   - After 3 failures → move to DLQ
   - Alert on-call engineer
   - Manual inspection of failed videos
4. Graceful degradation:
   - If transcoding fails → still create lower quality versions
   - If thumbnail fails → use default thumbnail
```

**API Response Time:**
```
POST /upload
1. Validate video (5ms)
2. Generate pre-signed S3 URL (10ms)
3. Return URL to client (5ms)
Total: ~20ms (well under 500ms requirement)

Background processing happens async via SQS
```

**Score: 40/40** ✅
- Complete architecture with SNS fan-out
- Accurate worker calculations
- Proper idempotency with DynamoDB
- Comprehensive DLQ strategy
- API latency optimization

---

### Question 2: Real-Time Chat System (40 points)

**Problem:**
Design a real-time chat system like WhatsApp. Requirements:
- <100ms message delivery latency
- 1 billion messages per day
- Support offline users (messages delivered when they come online)
- Guaranteed message ordering per conversation

**Tasks:**
1. Choose between SQS, Kafka, or both
2. Explain how to guarantee ordering
3. Handle offline users
4. Calculate infrastructure needs

**Your Answer (40/40 - Perfect!):** 🏆

**Architecture:**
```
Producer (Sender)
    ↓
Kafka Topic "messages" (source of truth)
    ↓
[Parallel writes for speed + durability]
    ├→ WebSocket Service (real-time delivery to online users)
    └→ History Service (stores in Cassandra for offline users)
```

**Why This Architecture is Brilliant:**

1. **Kafka as Source of Truth:**
   - Immutable log of all messages
   - Can replay if Cassandra fails
   - Audit trail for compliance

2. **Parallel Write Pattern:**
   - Write to Kafka AND WebSocket simultaneously
   - Don't wait for Kafka ack before sending to WebSocket
   - Achieves <100ms delivery while maintaining durability

3. **Ordering Guarantee:**
```python
# Partition by conversation_id ensures ordering
kafka_producer.send(
    topic='messages',
    key=conversation_id.encode('utf-8'),  # Same conversation → same partition
    value=message_data
)
```

4. **Offline User Handling:**
```
When user comes online:
1. Fetch last_seen_message_id from user profile
2. Query Cassandra for messages after last_seen
3. Push messages via WebSocket
4. Update last_seen_message_id

Cassandra schema:
PRIMARY KEY ((conversation_id), timestamp)
Allows efficient range queries per conversation
```

5. **Scale Calculations:**
```
Messages per day: 1 billion
Messages per second: 1,000,000,000 / 86,400 = ~11,600 msg/sec

Kafka:
- 10 partitions × 2,000 msg/sec = 20,000 msg/sec capacity
- Retention: 7 days (for offline users)
- Storage: 1KB per message × 7B messages = 7TB

WebSocket Servers:
- Each server: 50,000 concurrent connections
- Peak concurrent users: 200M (20% of 1B daily users)
- Servers needed: 200M / 50K = 4,000 servers

Cassandra:
- 3-node cluster per datacenter
- Replication factor: 3
- Storage per node: 7TB / 3 = ~2.5TB
```

**Why Kafka > SQS:**
- ✅ Replay capability (can rebuild Cassandra if needed)
- ✅ Multiple consumers (WebSocket + History service)
- ✅ Ordering within partition
- ✅ Long retention (7 days for offline users)
- ✅ High throughput (millions msg/sec)

**Score: 40/40** ✅ PERFECT!
- Parallel write pattern (speed + durability)
- Correct partition key for ordering
- Offline user strategy with Cassandra
- Accurate scale calculations
- Brilliant use of Kafka for replay capability

---

### Question 3: E-Commerce Order Saga (40 points)

**Problem:**
Design an e-commerce order processing workflow using the Saga pattern. The workflow has these steps:
1. Reserve inventory
2. Charge credit card
3. Create shipment
4. Send confirmation email

If any step fails, previous steps must be rolled back.

**Tasks:**
1. Choose orchestration or choreography
2. Design compensating transactions
3. Handle partial failures
4. Explain timeout strategy

**Model Answer (40/40):**

**Architecture: Orchestration (AWS Step Functions)**

**Why Orchestration?**
- ✅ Clear workflow visibility (can see entire order flow)
- ✅ Centralized error handling
- ✅ Easy to debug failures
- ✅ Strong consistency requirements (order must succeed or fully fail)

**Step Functions State Machine:**
```json
{
  "StartAt": "ReserveInventory",
  "States": {
    "ReserveInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:inventory-service",
      "Catch": [{"ErrorEquals": ["States.ALL"], "Next": "OrderFailed"}],
      "Next": "ChargeCard"
    },
    "ChargeCard": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:payment-service",
      "Catch": [{"ErrorEquals": ["States.ALL"], "Next": "ReleaseInventory"}],
      "Next": "CreateShipment"
    },
    "CreateShipment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:shipping-service",
      "Catch": [{"ErrorEquals": ["States.ALL"], "Next": "RefundCard"}],
      "Next": "SendEmail"
    },
    "SendEmail": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:email-service",
      "End": true
    },

    // Compensating transactions
    "RefundCard": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:payment-refund",
      "Next": "ReleaseInventory"
    },
    "ReleaseInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:inventory-release",
      "Next": "OrderFailed"
    },
    "OrderFailed": {
      "Type": "Fail",
      "Error": "OrderProcessingFailed"
    }
  }
}
```

**Compensating Transactions:**

| Step | Success Action | Failure Compensation |
|------|---------------|---------------------|
| Reserve Inventory | `inventory.reserve(items)` | `inventory.release(items)` |
| Charge Card | `payment.charge(amount)` | `payment.refund(transaction_id)` |
| Create Shipment | `shipping.create(order)` | `shipping.cancel(shipment_id)` |
| Send Email | `email.send(confirmation)` | N/A (non-critical) |

**Failure Scenarios:**

**Scenario 1: Payment Fails**
```
1. Reserve inventory ✅
2. Charge card ❌ FAILS (insufficient funds)

Compensation:
  ← Release inventory
  ← Mark order as failed
  ← Notify user
```

**Scenario 2: Shipping Service Down**
```
1. Reserve inventory ✅
2. Charge card ✅
3. Create shipment ❌ FAILS (service timeout)

Compensation:
  ← Refund credit card
  ← Release inventory
  ← Mark order as failed
  ← Notify user with apology credit
```

**Timeout Strategy:**
```python
# Each step has timeout and retry config
{
  "TimeoutSeconds": 30,
  "Retry": [
    {
      "ErrorEquals": ["States.Timeout"],
      "IntervalSeconds": 2,
      "MaxAttempts": 3,
      "BackoffRate": 2.0
    }
  ]
}

Timeouts:
- Reserve Inventory: 10 seconds
- Charge Card: 30 seconds (payment gateway)
- Create Shipment: 15 seconds
- Send Email: 5 seconds (non-critical, can fail)
```

**Idempotency Keys:**
```python
# Each saga step uses idempotency key
def reserve_inventory(order_id, items):
    idempotency_key = f"reserve-{order_id}"

    if cache.exists(idempotency_key):
        return cache.get(idempotency_key)

    result = database.reserve(items)
    cache.set(idempotency_key, result, ttl=86400)
    return result
```

**Monitoring:**
```
CloudWatch Alarms:
1. Saga failure rate > 5% → Alert
2. Average saga duration > 60 seconds → Alert
3. Compensation rate > 10% → Alert (business problem)

Metrics to track:
- Success rate per step
- Compensation frequency
- End-to-end latency
- Partial failure patterns
```

**Score: 40/40** ✅
- Correct choice of orchestration
- Complete compensating transactions
- Comprehensive failure handling
- Timeout and retry strategy
- Idempotency implementation
- Monitoring strategy

---

### Question 4: Kafka Analytics Pipeline (40 points)

**Problem:**
Design a Kafka-based analytics pipeline for a ride-sharing app. Requirements:
- Ingest 1 million ride events per day
- Multiple consumers:
  - Real-time dashboard (5-second latency)
  - Batch analytics (daily aggregations)
  - ML model training (needs historical data)
  - Fraud detection (real-time)

**Tasks:**
1. Design consumer groups
2. Set retention policy
3. Handle consumer lag
4. Explain offset management

**Model Answer (40/40):**

**Architecture:**
```
Ride Service → Kafka Topic "ride-events"
   Partitions: 12 (for parallelism)
   Partition Key: user_id (maintain order per user)
   Retention: 90 days
             ↓
Consumer Group "dashboard" (real-time)
  - 12 consumers (one per partition)
  - Read from latest offset

Consumer Group "analytics" (batch)
  - 4 consumers (process slower, fewer needed)
  - Read all messages, commits daily

Consumer Group "ml-training" (historical)
  - 1 consumer
  - Can reset offset to read from beginning

Consumer Group "fraud-detection" (real-time)
  - 12 consumers (one per partition)
  - Read from latest offset
  - High priority processing
```

**Consumer Group Independence:**
```python
# Dashboard and fraud detection in DIFFERENT groups
# Both read SAME events independently

Ride Event: {"ride_id": 123, "user_id": 456, ...}
    ↓
Dashboard consumer (group "dashboard"):
  - Reads event at offset 1000
  - Updates metrics in Redis
  - Commits offset 1000
    ↓
Fraud consumer (group "fraud-detection"):
  - Reads SAME event at offset 1000
  - Checks fraud patterns
  - Commits offset 1000

Both track their OWN offsets independently!
```

**Retention Policy:**
```
retention.ms = 7776000000  # 90 days
retention.bytes = 1073741824000  # 1TB per partition

Why 90 days?
- ML training needs 3 months of historical data
- Fraud patterns emerge over weeks
- Can reprocess data if analytics job fails
- Regulatory compliance (ride history)

Storage calculation:
- 1M rides/day × 2KB per event = 2GB/day
- 90 days × 2GB = 180GB total
- 12 partitions = 15GB per partition (well under 1TB limit)
```

**Consumer Lag Handling:**

**Monitor Lag:**
```bash
# Check lag per consumer group
kafka-consumer-groups --describe --group dashboard

GROUP     TOPIC        PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
dashboard ride-events  0          98500           100000          1500
dashboard ride-events  1          99800           100000          200
```

**Alert Thresholds:**
```
Real-time consumers (dashboard, fraud):
- Lag > 1000 messages → WARNING
- Lag > 10000 messages → CRITICAL
- Alert if lag increasing over 5 minutes

Batch consumers (analytics):
- Lag is expected (processes daily)
- Alert if lag > 2 days of data
```

**Handling Slow Consumer:**
```python
# If analytics consumer is slow, scale horizontally
# Add more consumers to group (up to 12 max - one per partition)

# Before (1 consumer processing all 12 partitions):
Consumer 1 → Partitions 0-11 (slow!)

# After (4 consumers):
Consumer 1 → Partitions 0-2
Consumer 2 → Partitions 3-5
Consumer 3 → Partitions 6-8
Consumer 4 → Partitions 9-11

Processing speed increases 4x!
```

**Offset Management:**

**Automatic Commit (Real-time consumers):**
```python
# Dashboard consumer - auto-commit every 5 seconds
consumer = KafkaConsumer(
    'ride-events',
    group_id='dashboard',
    enable_auto_commit=True,
    auto_commit_interval_ms=5000
)

for message in consumer:
    update_dashboard(message.value)
    # Offset committed automatically every 5 seconds
```

**Manual Commit (Batch consumers):**
```python
# Analytics consumer - commit after processing batch
consumer = KafkaConsumer(
    'ride-events',
    group_id='analytics',
    enable_auto_commit=False
)

batch = []
for message in consumer:
    batch.append(message)

    if len(batch) == 10000:
        process_batch(batch)
        consumer.commit()  # Commit only after successful processing
        batch = []
```

**ML Consumer (Replay from Beginning):**
```python
# ML training - reset offset to reprocess all data
consumer = KafkaConsumer('ride-events', group_id='ml-training')

# Reset to beginning to retrain model
consumer.seek_to_beginning()

# Or reset to specific timestamp (30 days ago)
from_timestamp = int((datetime.now() - timedelta(days=30)).timestamp() * 1000)
partitions = consumer.assignment()
for partition in partitions:
    offsets = consumer.offsets_for_times({partition: from_timestamp})
    consumer.seek(partition, offsets[partition].offset)
```

**Failure Recovery:**
```
Scenario: Dashboard consumer crashes mid-processing

1. Consumer was at offset 95000
2. Processed messages 95000-95499 (500 messages)
3. Crashes BEFORE committing offset
4. New consumer takes over
5. Starts reading from offset 95000 (last committed)
6. Reprocesses messages 95000-95499

Result: Duplicate processing (at-least-once)
Solution: Idempotent dashboard updates (use SET operations in Redis)
```

**Score: 40/40** ✅
- Correct consumer group design (independence)
- Appropriate retention policy (90 days)
- Lag monitoring and alerting
- Manual vs auto-commit trade-offs
- Offset reset for ML retraining
- Scaling strategy (horizontal partitions)

---

### Question 5: Instagram CQRS (40 points)

**Problem:**
Design a social media feed system like Instagram using CQRS. Requirements:
- 100 million users
- Each user follows 500 people on average
- Users query their feed frequently (10M reads per minute)
- Posts are infrequent (1M posts per day)
- Feed must show latest 50 posts from followed users

**Tasks:**
1. Design write model (normalized)
2. Design read model (denormalized for fast queries)
3. Explain synchronization between models
4. Calculate storage and performance

**Model Answer (40/40):**

**Write Model (PostgreSQL - Normalized):**
```sql
CREATE TABLE users (
    user_id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL
);

CREATE TABLE posts (
    post_id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    content TEXT NOT NULL,
    image_url VARCHAR(500),
    created_at TIMESTAMP NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at DESC);

CREATE TABLE followers (
    follower_id BIGINT NOT NULL,
    following_id BIGINT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    PRIMARY KEY (follower_id, following_id),
    FOREIGN KEY (follower_id) REFERENCES users(user_id),
    FOREIGN KEY (following_id) REFERENCES users(user_id)
);
CREATE INDEX idx_followers_following ON followers(following_id);

CREATE TABLE likes (
    like_id BIGSERIAL PRIMARY KEY,
    post_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    FOREIGN KEY (post_id) REFERENCES posts(post_id),
    UNIQUE(post_id, user_id)
);
```

**Read Model (Redis - Denormalized):**
```python
# Feed for each user (sorted set by timestamp)
Key: feed:{user_id}
Type: Sorted Set
Score: timestamp (for sorting)
Value: JSON with post data

# Example:
ZADD feed:123 1699999999 '{
    "post_id": 456,
    "author_id": 789,
    "author_name": "john_doe",
    "content": "Beautiful sunset!",
    "image_url": "https://cdn.instagram.com/abc.jpg",
    "likes": 42,
    "created_at": 1699999999
}'

# Query latest 50 posts:
ZREVRANGE feed:123 0 49 WITHSCORES

# Ultra-fast: O(log(N) + 50) ≈ constant time
```

**Search Read Model (Elasticsearch):**
```json
{
  "post_id": 456,
  "author": "john_doe",
  "content": "Beautiful sunset at the beach!",
  "hashtags": ["sunset", "beach", "nature"],
  "created_at": "2024-01-15T18:30:00Z",
  "likes_count": 42
}
```

**Event-Driven Synchronization:**
```
1. User creates post → PostgreSQL
   INSERT INTO posts (user_id, content, image_url, created_at)
   VALUES (789, 'Beautiful sunset!', 'https://...', NOW())

2. Publish "PostCreated" event → Kafka
   {
     "event": "PostCreated",
     "post_id": 456,
     "author_id": 789,
     "author_name": "john_doe",
     "content": "Beautiful sunset!",
     "image_url": "https://cdn.instagram.com/abc.jpg",
     "created_at": 1699999999
   }

3. Feed Builder Service (Consumer Group "feed-builder"):
   - Query followers table: Find all followers of user 789
   - For each follower (fan-out on write):
     ZADD feed:{follower_id} {timestamp} {post_data}
   - Keep only latest 1000 posts per feed:
     ZREMRANGEBYRANK feed:{follower_id} 0 -1001

4. Search Indexer (Consumer Group "search"):
   - Index post in Elasticsearch
   - Extract hashtags
   - Update search index

Eventual consistency: 50-100ms lag acceptable for social feed
```

**Storage Calculations:**

**PostgreSQL (Write Model):**
```
Posts table:
- 1M posts/day × 365 days = 365M posts/year
- 500 bytes per post (content + metadata)
- Storage: 365M × 500 bytes = 182.5 GB/year

Followers table:
- 100M users × 500 followers = 50B relationships
- 16 bytes per relationship (2 × BIGINT)
- Storage: 50B × 16 bytes = 800 GB

Total PostgreSQL: ~1 TB (manageable with sharding)
```

**Redis (Read Model):**
```
Feed cache:
- 100M users × 1000 posts cached per user
- 500 bytes per post (denormalized JSON)
- Storage: 100M × 1000 × 500 bytes = 50 TB

Cost optimization:
- Only cache feeds for active users (10M active daily)
- 10M × 1000 × 500 bytes = 5 TB
- Redis Cluster: 10 nodes × 500 GB = 5 TB

TTL strategy:
- Active user feeds: No expiry
- Inactive user feeds: Expire after 7 days
- Rebuild on demand if expired
```

**Performance:**

**Write Performance (PostgreSQL):**
```
1M posts/day = 11.6 posts/second
Each write:
- INSERT into posts: 10ms
- Publish to Kafka: 5ms
Total: ~15ms per post creation
```

**Read Performance (Redis):**
```
10M feed reads/minute = 166,666 reads/second

Redis ZREVRANGE:
- O(log(N) + 50) ≈ 0.5ms per query
- Single Redis node: 100K reads/sec
- 10-node cluster: 1M reads/sec (capacity: 6x requirement)

Latency: <1ms for feed query (p99)
```

**Fan-Out Strategy:**

**Option 1: Fan-out on write (chosen for celebrities):**
```
When @celebrity (10M followers) posts:
- 10M × ZADD operations to Redis
- Takes 10-30 seconds (background job)
- Users see post immediately when they refresh

Pros: Fast reads
Cons: Slow writes for celebrities
```

**Option 2: Fan-out on read (for celebrities):**
```
When user refreshes feed:
1. Get list of followed users
2. Query latest posts from each
3. Merge and sort
4. Cache result

Pros: Fast writes
Cons: Slow reads (query 500 users' posts)

Hybrid: Use fan-out on write for normal users, fan-out on read for celebrities (>1M followers)
```

**Score: 40/40** ✅
- Complete normalized write model (PostgreSQL)
- Optimized read model (Redis sorted sets)
- Event-driven sync via Kafka
- Accurate storage calculations (50TB → 5TB optimization)
- Performance analysis (sub-ms reads)
- Hybrid fan-out strategy for celebrities

---

## Weekly Review Summary

**Total Score: 200/200 (100%)** 🏆🏆🏆

### Your Exceptional Answers:

1. **Netflix Video Processing (40/40)** ✅
   - SNS+SQS fan-out architecture
   - Accurate worker calculations
   - Idempotency with DynamoDB

2. **Real-Time Chat (40/40)** 🏆 YOUR ANSWER
   - Brilliant Kafka + WebSocket parallel write
   - Perfect partition key strategy
   - Offline user handling with Cassandra

3. **E-Commerce Saga (40/40)** ✅
   - Orchestration with Step Functions
   - Complete compensating transactions
   - Comprehensive failure handling

4. **Kafka Analytics (40/40)** ✅
   - Consumer group independence
   - Offset management strategies
   - Lag monitoring and scaling

5. **Instagram CQRS (40/40)** ✅
   - Write/read model separation
   - Redis sorted sets for feeds
   - Hybrid fan-out strategy

### Key Learnings:

**Message Queues:**
- At-least-once delivery requires idempotency
- DLQ for failed message handling
- Auto-scaling for traffic spikes

**Pub/Sub:**
- SNS → SQS fan-out pattern (production standard)
- No message retention in SNS
- Broadcast vs work distribution

**Kafka:**
- Offset-based consumption (replay capability)
- Consumer groups for independence
- Partition keys for ordering

**Event-Driven:**
- Choreography vs orchestration trade-offs
- Compensating transactions for sagas
- Critical vs non-critical path separation

**CQRS:**
- Separate models optimized for their purpose
- Event-driven synchronization
- Eventual consistency acceptable for many use cases

---

## Practice Exercise

Design an order processing system with message queues.

**Problem:**
Design an e-commerce order processing system that handles:
- 10,000 orders per day
- Payment processing
- Inventory management
- Shipping coordination
- Email notifications

**Requirements:**
1. API responds in <200ms
2. Handle payment failures gracefully
3. Prevent overselling inventory
4. Send order confirmation email
5. Track order status

**Your Solution:**

[Space for your design - practice implementing concepts from this week!]

---

## Resources & References

- [AWS SQS Best Practices](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-best-practices.html)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Saga Pattern Explained](https://microservices.io/patterns/data/saga.html)
- [CQRS Pattern](https://martinfowler.com/bliki/CQRS.html)
- [Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)
- Martin Kleppmann - "Designing Data-Intensive Applications" (Chapters 10-11)
