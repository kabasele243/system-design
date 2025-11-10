# Pub/Sub Model

## Learning Objectives
- Understand publish-subscribe pattern
- Learn topics and subscriptions
- Master fan-out patterns

## Notes

### What is Pub/Sub?

**Pub/Sub** = Publish/Subscribe pattern where one message is **broadcast** to multiple subscribers.

**Key Difference from Queue:**
- **Queue (SQS)**: One message → ONE consumer (work distribution)
- **Pub/Sub (SNS)**: One message → MANY subscribers (broadcast)

### The Coffee Shop Analogy

**Message Queue (SQS):**
- Order board with tasks
- First available barista takes the order
- One order = one barista

**Pub/Sub (SNS):**
- Manager announces "We're out of milk!"
- ALL staff hear the announcement
- Baristas, cashiers, kitchen staff ALL react

### AWS SNS Architecture

```
Publisher → SNS Topic → Subscriber 1 (Email)
                     → Subscriber 2 (SQS Queue)
                     → Subscriber 3 (Lambda)
                     → Subscriber 4 (HTTP endpoint)
```

**Key Components:**
- **Publisher**: Sends message to topic (one time)
- **Topic**: Named channel for messages
- **Subscribers**: Services that want to receive messages
- **Message**: Delivered to ALL subscribers simultaneously

### Fan-Out Pattern (SNS → SQS)

**Most Common Production Pattern:**

```
SNS Topic → SQS Queue 1 (Payment Service workers)
         → SQS Queue 2 (Inventory Service workers)
         → SQS Queue 3 (Email Service workers)
         → SQS Queue 4 (Analytics Service workers)
```

**Why SNS → SQS (not direct SNS → Lambda)?**
1. **Retry control**: SQS provides retry with DLQ
2. **Rate limiting**: Control processing speed
3. **Buffering**: Handle traffic spikes
4. **Decoupling**: Each service processes at own pace

### SNS vs SQS vs EventBridge

| Feature | AWS SQS | AWS SNS | AWS EventBridge |
|---------|---------|---------|-----------------|
| **Pattern** | Point-to-point queue | Pub/Sub broadcast | Event bus with routing |
| **Consumers** | Multiple instances of SAME worker | Multiple DIFFERENT services | Multiple services with filtering |
| **Message Retention** | 1 min - 14 days | NO retention (deliver or drop) | NO retention |
| **Use Case** | Work distribution | Fan-out notifications | Event-driven architecture |
| **Message Delivery** | Pull (poll) | Push (to subscribers) | Push (to targets) |
| **Filtering** | No | Basic | Advanced (content-based) |
| **Ordering** | FIFO queues available | No ordering | No ordering |
| **Example** | 10 payment workers processing orders | Order placed → notify payment, inventory, email | User signup → 20 different microservices react |

### Key SNS Concepts

#### 1. Topics
- Named channel (e.g., "order-events", "user-signups")
- Publishers send to topic
- Subscribers listen to topic

#### 2. Subscriptions
- Services register to receive messages from topic
- Can filter messages (e.g., only "premium" orders)
- Supports: SQS, Lambda, Email, SMS, HTTP, Kinesis

#### 3. Message Filtering
```json
// Subscription filter policy - only get premium orders
{
  "order_type": ["premium"],
  "amount": [{"numeric": [">", 100]}]
}
```

#### 4. No Message Retention
- **Critical difference**: SNS doesn't store messages
- If subscriber is down → message lost
- Solution: SNS → SQS (SQS stores messages)

### SNS Delivery Guarantees

- **At-Least-Once**: Same as SQS
- May deliver duplicates
- Subscribers must handle idempotency

### Multiple Consumers Clarification

**Your Key Insight:**
> "SQS can have multiple instances of the same worker. But Pub/Sub can have multiple different subscribers."

**Perfect! Let's clarify:**

**SQS (Work Distribution):**
```
Order Queue → Payment Worker Instance 1
           → Payment Worker Instance 2
           → Payment Worker Instance 3
(Each order processed by ONE worker)
```

**SNS (Broadcast):**
```
Order Topic → Payment Service (all instances react)
           → Inventory Service (all instances react)
           → Email Service (all instances react)
           → Analytics Service (all instances react)
(Each service gets EVERY order)
```

### When to Use Pub/Sub

**Use Cases:**
- ✅ Order placed → notify multiple services (payment, inventory, shipping, email)
- ✅ User signup → update CRM, send welcome email, create profile, log analytics
- ✅ Video uploaded → transcode, generate thumbnail, update search index
- ✅ Alert/notification to multiple systems

**Don't Use When:**
- ❌ Need work distribution → Use SQS
- ❌ Need message replay → Use Kafka
- ❌ Need guaranteed delivery → Use SNS → SQS

### Production Pattern: SNS → SQS

**Why this is the gold standard:**

```
Publisher → SNS Topic → SQS Queue 1 → Payment Workers (3 instances)
                     → SQS Queue 2 → Inventory Workers (5 instances)
                     → SQS Queue 3 → Email Workers (2 instances)
```

**Benefits:**
1. **SNS**: Broadcast to all services
2. **SQS**: Each service has own queue with:
   - Message persistence (14 days)
   - Retry logic
   - DLQ for failed messages
   - Multiple worker instances
   - Rate limiting

## Practice Questions

### Your Questions During Learning

**Q: "What the difference between AWS SQS, SNS and Event bridge?"**

**A: Key Differences:**
- **SQS**: Work queue, one consumer gets message, 14-day retention
- **SNS**: Pub/sub, all subscribers get message, no retention
- **EventBridge**: Advanced event bus with content-based routing, schema registry, integrations

**Q: "So an aws queue can have multiple consumer??"**

**A: YES!**
- Multiple consumer **instances** (same worker type)
- Each message goes to ONE consumer
- Used for parallel processing

## Real-World Examples

### Netflix Video Upload
```
1. User uploads video → S3
2. S3 event → SNS Topic "VideoUploaded"
3. SNS fans out to:
   → SQS Transcoding Queue (30 workers)
   → SQS Thumbnail Queue (10 workers)
   → SQS Metadata Queue (5 workers)
   → Lambda (update database)
   → Lambda (send notification to user)
```

### E-commerce Order Placed
```
Order Service → SNS Topic "OrderPlaced"
             → Payment SQS → Charge credit card
             → Inventory SQS → Reserve items
             → Shipping SQS → Create shipment
             → Email SQS → Send confirmation
             → Analytics SQS → Log event
```

### Uber Ride Completed
```
Ride Service → SNS Topic "RideCompleted"
            → Payment SQS → Charge rider
            → Driver SQS → Pay driver
            → Rating SQS → Request rating
            → Analytics SQS → Log ride data
            → ML SQS → Update pricing model
```

## Resources & References

- [AWS SNS Documentation](https://docs.aws.amazon.com/sns/)
- [SNS Message Filtering](https://docs.aws.amazon.com/sns/latest/dg/sns-message-filtering.html)
- [Fan-Out Pattern Best Practices](https://aws.amazon.com/blogs/compute/managing-backend-requests-and-frontend-notifications-in-serverless-web-apps/)
