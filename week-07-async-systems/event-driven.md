# Event-Driven Architecture

## Learning Objectives
- Understand event-driven design patterns
- Learn event sourcing and CQRS
- Master saga pattern for distributed transactions

## Notes

### What is Event-Driven Architecture?

**Event-Driven Architecture (EDA)** = Systems communicate through **events** (notifications that something happened).

**Key Principle:**
- Services react to events instead of calling each other directly
- Decoupled: Services don't know about each other
- Asynchronous: Non-blocking communication

### The Coffee Shop Analogy

**Traditional (Synchronous):**
```
You → (wait) → Barista → (wait) → Cashier → (wait) → Kitchen
(Blocking at every step)
```

**Event-Driven (Asynchronous):**
```
You place order → Order Event published
  ↓
[Multiple workers react simultaneously]
  - Barista starts making drink
  - Kitchen starts making food
  - Cashier processes payment
  - Display updates order status
```

### Event Naming Convention

**Events = Past Tense (Facts)**

Good examples:
- ✅ `OrderPlaced`
- ✅ `PaymentProcessed`
- ✅ `UserRegistered`
- ✅ `DriverAssigned`

Bad examples:
- ❌ `PlaceOrder` (command, not event)
- ❌ `ProcessingPayment` (present tense)
- ❌ `RegisterUser` (command)

**Why past tense?** Events are **immutable facts** that already happened.

### Choreography vs Orchestration

#### Choreography (Decentralized)
**Each service knows what to do when an event occurs**

```
Order Service → publishes "OrderPlaced"
    ↓
[Services react independently]
    - Payment Service: "I'll charge the card"
    - Inventory Service: "I'll reserve items"
    - Shipping Service: "I'll create shipment"
    - Email Service: "I'll send confirmation"
```

**Pros:**
- No central coordinator
- Easy to add new services
- Each service autonomous

**Cons:**
- Hard to see full workflow
- Difficult to debug
- No single point of control

#### Orchestration (Centralized)
**A coordinator tells services what to do**

```
Order Saga Orchestrator:
  1. Call Payment Service → charge card
  2. If success → Call Inventory → reserve items
  3. If success → Call Shipping → create shipment
  4. If success → Call Email → send confirmation
  5. If any fail → Rollback previous steps
```

**Pros:**
- Clear workflow visibility
- Easy to debug
- Centralized error handling

**Cons:**
- Single point of failure
- Orchestrator can become complex
- Coupling to orchestrator

### Your Food Delivery Answer (40/40 - Perfect!) 🏆

**Your Event Flow:**
```
Events:
1. OrderPlaced
2. PaymentAuthorized
3. OrderAcceptedByRestaurant
4. DriverAssigned
5. OrderPickedUp
6. OrderDelivered
7. OrderRated
```

**Why Choreography?**
> "Services react independently. If driver service goes down, payment still works."

**Brilliant reasoning!** You understood:
- Event names (past tense, immutable)
- Service autonomy
- Fault isolation
- Non-critical path separation

### Compensating Transactions (Saga Pattern)

**Problem:** What if a step in a distributed transaction fails?

**Solution:** Compensating transactions = Undo previous steps

#### Example: E-Commerce Order

**Happy Path (Orchestration):**
```
1. Reserve inventory ✅
2. Charge credit card ✅
3. Create shipment ✅
4. Send email ✅
```

**Failure Path:**
```
1. Reserve inventory ✅
2. Charge credit card ✅
3. Create shipment ❌ FAILS

Compensating transactions:
  ← Refund credit card
  ← Release inventory
```

**Your Understanding:**
> "If payment fails, trigger compensating event to release inventory"

Perfect! Compensation = rollback in distributed systems.

### Event Sourcing

**Traditional Approach:**
```
Database stores CURRENT state:
  User: {id: 123, balance: $500}
```

**Event Sourcing:**
```
Store ALL events that changed the state:
  1. AccountCreated (balance: $0)
  2. MoneyDeposited ($1000)
  3. MoneyWithdrawn ($300)
  4. MoneyWithdrawn ($200)

Current balance = replay all events = $500
```

**Benefits:**
- Complete audit trail
- Can replay to any point in time
- Never lose data

**Use cases:**
- Banking (regulatory compliance)
- Trading platforms (audit logs)
- Version control systems (Git!)

### CQRS (Command Query Responsibility Segregation)

**Separate read and write models**

#### Your Social Media Answer (24/40 - Good Concepts!)

**What you got right:**
- ✅ Separate write model (posts table) and read model (Redis feed)
- ✅ Event-driven sync between models
- ✅ Eventual consistency acceptable for social feed

**What needed more detail:**
- Implementation specifics (schemas, Redis data structures)
- How to handle different query patterns
- Scale calculations

#### Complete CQRS Example

**Write Model (PostgreSQL - Normalized):**
```sql
CREATE TABLE posts (
    post_id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL
);

CREATE TABLE followers (
    follower_id BIGINT,
    following_id BIGINT
);
```

**Read Model (Redis - Denormalized):**
```python
# Feed for user (sorted set by timestamp)
ZADD feed:{user_id} {timestamp} {post_data}

# User can query feed:
ZREVRANGE feed:{user_id} 0 49  # Get latest 50 posts
```

**Event Flow:**
```
1. User creates post → Save to PostgreSQL
2. Publish "PostCreated" event
3. Feed Builder Service:
   - Find all followers
   - Add post to each follower's Redis feed
4. User queries feed → Read from Redis (fast!)
```

**Why CQRS?**
- Write: Complex validations, ACID transactions (PostgreSQL)
- Read: Simple, fast queries (Redis)
- Optimize each for its purpose

### Critical Path vs Non-Critical Path

**Your Food Delivery Insight:**

**Critical Path (Must complete for order to succeed):**
1. OrderPlaced
2. PaymentAuthorized
3. OrderAcceptedByRestaurant

**Non-Critical Path (Can happen async):**
1. DriverAssigned
2. Email notifications
3. Analytics logging
4. Push notifications

**Why separate?**
- Critical path = synchronous, must succeed
- Non-critical = async events, can fail/retry later
- User doesn't wait for email to be sent

### When to Use Event-Driven Architecture

**Use Cases:**
- ✅ Microservices communication
- ✅ Order processing workflows
- ✅ User activity tracking
- ✅ Notification systems
- ✅ Decoupling services

**Don't Use When:**
- ❌ Need immediate response (use HTTP/gRPC)
- ❌ Simple CRUD app (overkill)
- ❌ Strong consistency required (use transactions)
- ❌ Complex queries across services (use API gateway)

## Practice Questions

### Question 1: Food Delivery Event Flow (Score: 40/40 - 100%) 🏆

**Your Answer Highlights:**
```
Events:
- OrderPlaced
- PaymentAuthorized
- OrderAcceptedByRestaurant
- DriverAssigned
- OrderPickedUp
- OrderDelivered
- OrderRated

Pattern: Choreography
Reasoning: Service autonomy, fault isolation
Compensation: Release inventory if payment fails
```

**What made this answer perfect:**
- Event naming (past tense)
- Choreography reasoning (decoupled services)
- Understood compensating transactions
- Separated critical vs non-critical path

### Question 2: Social Media CQRS (Score: 24/40 - 60%)

**Your Answer (Conceptual):**
- Write model: Posts table
- Read model: Redis feed
- Event-driven sync

**What was missing:**
- Specific Redis data structures (sorted sets)
- Schema details
- How to handle search (Elasticsearch)
- Scale calculations

**Improvement areas:**
- More implementation details
- Specific technology choices
- Concrete examples with code

## Real-World Examples

### Uber Ride Request (Event-Driven + CQRS)

**Events (Choreography):**
```
1. RideRequested
2. DriverMatched
3. DriverArrived
4. RideStarted
5. RideCompleted
6. PaymentProcessed
7. RideRated
```

**Services react independently:**
- Matching Service: Find nearby drivers
- Notification Service: Send push notifications
- Pricing Service: Calculate fare
- Payment Service: Charge rider
- Analytics Service: Log ride data

**CQRS:**
- Write: Ride data to PostgreSQL (ACID)
- Read: Driver locations in Redis GEO (fast queries)

### Netflix Video Processing (Orchestration)

**AWS Step Functions Saga:**
```
1. Video uploaded → S3
2. Transcode to 4K → Success
3. Transcode to 1080p → Success
4. Transcode to 720p → FAILS

Compensating transactions:
  ← Delete 1080p version
  ← Delete 4K version
  ← Mark video as failed
```

### Banking Transfer (Event Sourcing)

**Events stored forever:**
```
AccountId: 123
Events:
  1. AccountOpened (balance: $0)
  2. DepositMade ($1000) - timestamp: 2024-01-01
  3. TransferOut ($200) - timestamp: 2024-01-05
  4. InterestAdded ($10) - timestamp: 2024-01-31
  5. WithdrawalMade ($300) - timestamp: 2024-02-01

Current balance = $510

Can query: "What was balance on 2024-01-31?"
Replay events up to that date = $810
```

**Regulatory compliance:** Complete audit trail for 7 years.

### Instagram Feed (Your Example - Enhanced)

**Write Model (PostgreSQL):**
```sql
-- Normalized schema
posts (post_id, user_id, content, media_url, created_at)
likes (like_id, post_id, user_id, created_at)
comments (comment_id, post_id, user_id, text, created_at)
```

**Read Models:**

1. **Feed (Redis Sorted Set):**
```python
# Key: feed:{user_id}
# Score: timestamp
ZADD feed:123 1699999999 '{"post_id": 456, "content": "..."}'
```

2. **Search (Elasticsearch):**
```json
{
  "post_id": 456,
  "content": "My vacation in Paris",
  "hashtags": ["paris", "travel"],
  "created_at": "2024-01-15"
}
```

3. **Analytics (ClickHouse):**
```sql
-- Time-series data for metrics
post_views (post_id, timestamp, view_count)
```

**Event Flow:**
```
1. User creates post → PostgreSQL
2. Publish "PostCreated" event → Kafka
3. Multiple consumers:
   - Feed Builder → Add to followers' Redis feeds
   - Search Indexer → Index in Elasticsearch
   - Analytics → Log to ClickHouse
   - CDN → Cache media
```

## Resources & References

- [Event-Driven Architecture Patterns](https://martinfowler.com/articles/201701-event-driven.html)
- [Saga Pattern Explained](https://microservices.io/patterns/data/saga.html)
- [Event Sourcing by Martin Fowler](https://martinfowler.com/eaaDev/EventSourcing.html)
- [CQRS Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [AWS Step Functions (Orchestration)](https://aws.amazon.com/step-functions/)
