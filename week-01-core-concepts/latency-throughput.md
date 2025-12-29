# Latency vs Throughput

## Learning Objectives
- Understand the difference between latency and throughput
- Learn how they relate to system performance
- Understand trade-offs between the two

## Notes

### Latency (Response Time)

**Analogy:** Private jet moving 10 people to LA in 5 hours - FAST for each person, but low volume.

**Definition:** How long does ONE request take from start to finish?
- Time for a single operation
- Measured in: milliseconds (ms), seconds
- **Lower is better**

**Formula:** `Latency = Time to complete ONE request`

**Examples:**
- Website loads in 100ms → Low latency ✅ (feels instant)
- Database query takes 5 seconds → High latency ❌ (feels slow)
- Gaming: Button press to action in 10ms → Low latency ✅
- Gaming: Button press to action in 200ms → High latency ❌ (unplayable lag)

**When Latency Matters Most:**
- **Real-time interaction:** Gaming, video calls, live streaming
- **Stock trading:** Microseconds = millions in profit
- **Autonomous vehicles:** Instant decisions to avoid crashes
- **User experience:** Users abandon slow websites

**How to Reduce Latency:**
- Faster hardware (SSDs, high-speed networks)
- Caching (store data closer)
- CDN (servers near users geographically)
- Optimized code and algorithms
- Reduce network hops

**AWS Services for Low Latency:**
- **Amazon CloudFront:** CDN that caches content at edge locations worldwide (reduces latency by serving from nearest location)
- **Amazon ElastiCache:** In-memory caching (Redis/Memcached) for sub-millisecond data access
- **AWS Global Accelerator:** Routes traffic through AWS backbone network to reduce latency
- **Amazon DynamoDB DAX:** In-memory cache for DynamoDB with microsecond response times
- **EC2 Instance Types:** Compute-optimized (C7g) or memory-optimized (R7g) for faster processing
- **Amazon Aurora Global Database:** Read replicas in multiple regions for < 1 second cross-region replication
- **AWS Local Zones:** Deploy compute closer to end users in major cities

### Throughput (Volume Over Time)

**Analogy:** One big airplane moving 1,000 people to LA in 6 hours - slower per person, but HUGE volume.

**Definition:** How many requests can the system handle per unit of time?
- Total work completed in a period
- Measured in: requests/second, GB/second, transactions/minute
- **Higher is better**

**Formula:** `Throughput = Number of requests / Time period`

**Examples:**
- Server handles 10,000 requests/second → High throughput ✅
- Network transfers 1 GB/second → High throughput ✅
- Netflix streams to 100M users simultaneously → High throughput ✅
- Video encoding: Process 1,000 videos/hour → High throughput ✅

**When Throughput Matters Most:**
- **Batch processing:** Data analytics, video encoding, backups
- **High user volume:** Social media, streaming services
- **Data pipelines:** Moving/processing large datasets
- **Mass operations:** Sending millions of emails

**How to Increase Throughput:**
- Horizontal scaling (add more servers)
- Parallel processing
- Efficient algorithms
- Load balancing
- Async/non-blocking operations

**AWS Services for High Throughput:**
- **Auto Scaling Groups:** Automatically add/remove EC2 instances based on demand
- **Application Load Balancer (ALB):** Distributes traffic across multiple targets, handles millions of requests/sec
- **Amazon Kinesis:** Process hundreds of terabytes of streaming data per hour
- **AWS Lambda:** Concurrent execution of up to 1,000 functions simultaneously (can be increased)
- **Amazon SQS:** Handle unlimited messages, process millions of messages per second
- **AWS Batch:** Run hundreds of thousands of batch computing jobs
- **Amazon EMR:** Process petabytes of data with Spark/Hadoop clusters
- **Amazon S3:** 5,500 PUT/POST/DELETE and 5,500 GET requests per second per prefix

### The Trade-off

**Key Insight:** You can have HIGH throughput with HIGH latency!

**Example: Batch Video Processing**
- Each video takes 10 minutes to encode (high latency per video)
- But process 1,000 videos simultaneously (high throughput)
- Individual video is slow, but total volume is huge

**The Inverse:** You can have LOW latency with LOW throughput!

**Example: Premium Gaming Server**
- 10ms response time (low latency - feels instant!)
- But only handles 100 players (low throughput)
- Each player gets fast response, but can't handle massive scale

### Can You Have Both?

**Yes, but it's VERY EXPENSIVE!**

**Example: Google Search**
- Low latency: < 200ms search results
- High throughput: 8.5 billion searches/day
- Cost: Billions in infrastructure, millions of servers, hundreds of data centers

**Why It's Expensive:**
- Fast hardware everywhere (SSDs, high-speed networks)
- Many servers (horizontal scaling for throughput)
- Global data centers (reduce latency for all users)
- Thousands of engineers optimizing code
- Advanced load balancing and caching

**Only companies like Google, Amazon, Netflix can afford both.**

### Prioritization Matrix

| System Type | Priority | Reasoning | Examples |
|-------------|----------|-----------|----------|
| **Real-time interaction** | **Low Latency** | Users need instant feedback | Gaming, video calls, stock trading |
| **Batch processing** | **High Throughput** | Volume matters more than speed | Analytics, backups, email blasts |
| **User-facing + scale** | **Both (expensive!)** | Need fast response AND handle millions | Google, Amazon, banking |


## Practice Questions

1. **Can you have high throughput but high latency? Give an example.**
   - **Yes!** Batch video encoding: Each video takes 10 minutes (high latency), but process 1,000 videos simultaneously (high throughput).
   - Big airplane analogy: 6 hours per trip (higher latency), but moves 1,000 people at once (high throughput).

2. **Can you have low latency but low throughput? Give an example.**
   - **Yes!** Premium gaming server: 10ms response (low latency), but only handles 100 players (low throughput).
   - Private jet analogy: 5 hours per trip (low latency), but only moves 10 people (low throughput).

3. **Which matters more for a video streaming service?**
   - **Both, but differently:**
   - **Live streaming (sports):** Low latency matters (< 2 seconds delay)
   - **On-demand (Netflix):** High throughput matters (millions of concurrent streams), some buffering acceptable

4. **Online gaming - latency or throughput priority?**
   - **Low latency is critical!** 10-50ms = playable, 200ms+ = unplayable
   - Throughput matters less (would rather have 10-player low-latency game than 100-player laggy game)

5. **Why is having both low latency AND high throughput expensive?**
   - Need fast hardware everywhere (expensive)
   - Need many servers for scale (horizontal scaling)
   - Need global data centers (reduce latency worldwide)
   - Need optimized code (senior engineers, time)
   - Only Google/Amazon-level companies can afford both

## Real-World Examples

### Systems Optimizing for LOW LATENCY
- **Online gaming:** 10-50ms response time (microseconds matter in competitive play)
- **Stock trading:** High-frequency trading in microseconds (millions in profit)
- **Video calls:** < 150ms round-trip for natural conversation
- **Autonomous vehicles:** Instant braking decisions
- **Interactive websites:** < 200ms load time (users abandon if slower)

### Systems Optimizing for HIGH THROUGHPUT
- **YouTube video encoding:** Process millions of videos per day
- **Data warehouses:** Analyze terabytes of log data
- **Backup systems:** Transfer petabytes of data
- **Email services:** Send millions of emails per hour
- **Batch analytics:** Process billions of events overnight

### Systems Needing BOTH (Expensive!)
- **Google Search:** < 200ms results + 8.5B searches/day
- **Amazon checkout:** Fast checkout + Black Friday traffic
- **Banking systems:** Quick transactions + millions of users
- **Netflix:** Instant playback + 200M+ concurrent users
- **Uber:** Real-time matching + millions of rides/day

## AWS Architecture Examples

### Example 1: Low Latency Gaming Backend
```
Architecture:
- CloudFront → API Gateway → Lambda (us-east-1)
- DynamoDB with DAX for player state
- ElastiCache Redis for real-time matchmaking

Why low latency?
- CloudFront edge locations reduce geographic latency
- DAX provides microsecond access to player data
- Lambda cold starts mitigated with provisioned concurrency
- Single region to avoid cross-region latency

Trade-off: Limited to regional capacity (lower max throughput)
```

### Example 2: High Throughput Data Pipeline
```
Architecture:
- S3 → Kinesis Data Streams → Lambda → S3/DynamoDB
- Auto Scaling EMR cluster for batch processing
- SQS queues for buffering

Why high throughput?
- Kinesis handles millions of events per second
- Lambda scales to 1,000+ concurrent executions
- EMR processes terabytes in parallel
- SQS buffers unlimited messages

Trade-off: Individual record processing may take seconds (higher latency)
```

### Example 3: Both Low Latency + High Throughput (Expensive!)
```
Architecture:
- Route 53 → CloudFront (global) → ALB (multi-region) → Auto Scaling EC2
- Aurora Global Database (multi-region)
- ElastiCache clusters in each region
- DynamoDB Global Tables

Why both?
- Multi-region for low latency worldwide
- Auto Scaling for high throughput
- Global databases replicate in < 1 second
- Caching at multiple layers

Cost: $10,000+ per month easily
```

## AWS Cost vs Performance Matrix

| Configuration | Monthly Cost | Latency | Throughput | Use Case |
|--------------|--------------|---------|------------|----------|
| Single t3.micro + DynamoDB | $50 | 200-500ms | 100 req/sec | MVP/prototype |
| CloudFront + Lambda + DAX | $500 | 50-100ms | 10,000 req/sec | Production app |
| Multi-region + Auto Scaling | $5,000 | 20-50ms | 100,000 req/sec | High-scale app |
| Global infrastructure + all optimizations | $50,000+ | < 20ms | 1M+ req/sec | Fortune 500 |

## Resources & References
- Airplane analogy: Private jets (low latency, low throughput) vs. Big airplane (higher latency, high throughput)
- Key principle: Most systems must choose based on their primary use case
- Remember: Having both requires Google-level infrastructure investment
- AWS Well-Architected Framework: Performance Efficiency Pillar
