# Message Queues

## Learning Objectives
- Understand message queue fundamentals
- Learn producer-consumer pattern
- Master queue guarantees (at-least-once, exactly-once)

## Notes

### What is a Message Queue?

A **message queue** is like a waiting line at a coffee shop. Customers (producers) place orders, which go into a queue, and baristas (consumers) process orders one by one.

**Key Characteristics:**
- **Decoupling**: Producer and consumer don't need to know about each other
- **Asynchronous**: Producer can continue working without waiting for consumer
- **Load leveling**: Queue acts as buffer during traffic spikes
- **Persistence**: Messages stored until processed

### Producer-Consumer Pattern

```
Producer → [Queue] → Consumer
(Sends)              (Processes)
```

**Real Example - Order Processing:**
```
Web Server → [Order Queue] → Payment Service
             [Order Queue] → Inventory Service
             [Order Queue] → Notification Service
```

### Delivery Guarantees

#### 1. At-Most-Once
- Message delivered **0 or 1 times**
- May lose messages
- Use case: Logging, metrics (OK to lose some data)

#### 2. At-Least-Once (Most Common - AWS SQS)
- Message delivered **1+ times**
- May get duplicates
- Use case: Payment processing (with idempotency)
- **Solution**: Idempotency keys to handle duplicates

#### 3. Exactly-Once
- Message delivered **exactly 1 time**
- Hard to implement, expensive
- Use case: Financial transactions

### AWS SQS Key Concepts

#### Visibility Timeout
```
1. Consumer receives message
2. Message becomes "invisible" for 30 seconds (configurable)
3. Consumer processes and DELETES message
4. If not deleted → message reappears in queue
```

**Important:** Messages DON'T auto-delete! You must explicitly delete them.

#### Multiple Consumers
- **YES**, SQS supports multiple consumers
- Each message goes to **ONE consumer** (work distribution)
- Multiple instances of the **same worker** process messages in parallel
- Example: 10 payment workers pulling from same queue

#### Dead Letter Queue (DLQ)
```
Main Queue → [Failed 3 times] → Dead Letter Queue
```
- Captures messages that fail repeatedly
- Prevents infinite retry loops
- Allows manual inspection and reprocessing

### Message Retention
- **AWS SQS**: 1 minute to 14 days (default 4 days)
- After max retention → message deleted
- Messages stay until:
  1. Explicitly deleted by consumer, OR
  2. Retention period expires

### Idempotency Pattern

**Problem:** At-least-once delivery means duplicates

**Solution:** Idempotency key
```python
def process_payment(message):
    payment_id = message['payment_id']  # Idempotency key

    # Check if already processed
    if database.exists(payment_id):
        return "Already processed - skip"

    # Process payment
    charge_credit_card(message['amount'])
    database.save(payment_id)

    # Delete from queue
    sqs.delete_message(message)
```

### When to Use Message Queues

**Use Cases:**
- Background jobs (email sending, image processing)
- Order processing
- Task distribution across workers
- Decoupling microservices
- Load leveling during traffic spikes

**Don't Use When:**
- Need broadcast to multiple services → Use Pub/Sub
- Need message replay → Use Kafka
- Need request-response pattern → Use HTTP/gRPC

## Practice Questions

### Question 1: Trading Platform with Kafka (Score: 39/40 - 97.5%)
**Your Answer Highlights:**
- Kafka for order matching queue (offset-based replay for audit)
- Partition by `user_id` for ordering
- Consumer groups for parallel processing
- Redis cache for portfolio reads
- Excellent reasoning on replayability

## Real-World Examples

### Netflix Video Processing
- User uploads video → S3
- S3 event → SNS topic
- SNS fans out to multiple SQS queues:
  - Transcoding queue (convert formats)
  - Thumbnail queue (generate thumbnails)
  - Metadata queue (extract info)
- Each queue has auto-scaling workers
- Idempotency: Check if video already processed in DynamoDB

### E-commerce Order Flow
```
1. Order placed → Order Queue
2. Payment Worker:
   - Reserve inventory
   - Charge credit card
   - Save order in DB
   - Delete message from queue
3. If payment fails:
   - Release inventory
   - Don't delete message → goes to DLQ after retries
```

## Resources & References

- [AWS SQS Documentation](https://docs.aws.amazon.com/sqs/)
- [At-Least-Once Delivery Explained](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues.html)
- Martin Kleppmann - "Designing Data-Intensive Applications" (Chapter 11)
