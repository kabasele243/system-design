# Week 2: Networking & APIs - Interview Questions

## Instructions
These are realistic senior-level system design interview questions. Approach each scenario as you would in an actual interview:
- State your assumptions clearly
- Show your reasoning process
- Discuss trade-offs
- Consider real-world constraints

---

## Question 1: DNS Failure Impact Analysis

**Scenario:**
You're the infrastructure lead at a major e-commerce platform. During Black Friday preparation, the team is evaluating DNS redundancy strategies.

**Current Setup:**
- Single DNS provider (Route 53)
- 4 nameservers all from Route 53
- DNS TTL: 5 minutes for main domain
- 100 million daily active users
- Average order value: $75
- Peak traffic: 50,000 orders/minute during Black Friday

**Incident Report from Competitor:**
A competitor experienced a DNS outage during their biggest sales event:
- Duration: 3 hours
- Cause: DDoS attack on their DNS provider
- Their website was fully operational, but completely unreachable
- Lost sales estimated at $25 million

**Questions:**

1. **Impact Calculation:**
   - If your DNS provider fails for 3 hours during Black Friday peak, calculate the revenue loss
   - Show: Orders/minute × Duration × Average order value
   - Consider: What percentage of users with cached DNS could still access the site?

2. **Multi-Provider DNS Strategy:**
   - Design a multi-provider DNS setup using 2 different providers
   - How would you configure nameservers across providers?
   - What are the operational challenges of keeping DNS records in sync?

3. **TTL Trade-offs:**
   - Current TTL: 5 minutes
   - Proposal A: Increase to 24 hours (better cache hit rate, less DNS load)
   - Proposal B: Decrease to 30 seconds (faster updates, more flexibility)
   - Analyze: What happens if you need to fail over to a backup data center with each TTL setting?

4. **DNS Caching Layers:**
   - During the 3-hour outage, different users have different experiences
   - Calculate what % of users can still access the site based on:
     - Users who visited in the last 5 minutes (cached)
     - Users who visited 6-30 minutes ago (expired)
     - New users who never visited before
   - What does this tell you about DNS TTL strategy during high-traffic events?

---

## Question 2: Load Balancer Architecture for Microservices

**Scenario:**
You're redesigning the load balancing architecture for a ride-sharing app (similar to Uber) that's migrating from a monolith to microservices.

**Current Monolith:**
- Single load balancer (L4)
- 50 identical application servers
- Round-robin distribution
- 1 million requests/minute peak

**New Microservices Architecture:**
```
- User Service (authentication, profiles)
- Ride Service (matching, ride state)
- Payment Service (billing, transactions)
- Notification Service (push notifications, SMS)
- Location Service (driver/rider location tracking)
```

**Questions:**

1. **L4 vs L7 Decision:**
   - Should you use L4 or L7 load balancers for this microservices architecture?
   - Justify your choice considering:
     - Need to route `/users/*` to User Service
     - Need to route `/rides/*` to Ride Service
     - Need to route `/payments/*` to Payment Service
   - What are the performance implications?

2. **Routing Strategy Design:**
   - Design the URL routing rules for an L7 load balancer
   - Include: Path patterns, service destinations, header-based routing
   - How would you handle versioning? (`/v1/rides` vs `/v2/rides`)

3. **Health Check Strategy:**
   - Each microservice needs different health checks:
     - User Service: Database connectivity
     - Payment Service: Payment gateway connectivity
     - Location Service: Redis connectivity
   - Design health check endpoints for each service
   - What's the right health check interval? (Tradeoff: frequent checks vs load on services)

4. **Location Service Special Case:**
   - Location updates: 500,000/minute from drivers
   - Location reads: 2 million/minute from riders
   - Each update is only 100 bytes
   - Would you use the same load balancing algorithm as other services?
   - Should location service use L4 or L7 load balancing? Why?

5. **Load Balancer Redundancy:**
   - The load balancer is a single point of failure
   - Design an Active-Active vs Active-Passive setup
   - Calculate: If each load balancer can handle 100K requests/second:
     - Active-Active: How much capacity do you need if one fails?
     - Active-Passive: Is the passive LB wasted resources?

---

## Question 3: API Gateway vs Reverse Proxy Trade-offs

**Scenario:**
You're the principal engineer at a fintech startup building a banking app. The platform has grown from 3 services to 25 microservices. You're deciding between:
- **Option A:** Stick with NGINX (reverse proxy)
- **Option B:** Migrate to Kong/AWS API Gateway (API gateway)

**Current NGINX Setup:**
- Routes requests to 25 microservices
- 100K requests/second peak
- Basic load balancing and SSL termination
- Cost: $5,000/month

**Business Requirements:**
- Need to expose APIs to partner banks
- Need different rate limits per partner (Partner A: 10K/hour, Partner B: 100K/hour)
- Need detailed API analytics (who's calling what, how often)
- Need to deprecate v1 APIs and migrate partners to v2
- Compliance requirement: Log all API calls for 7 years

**Questions:**

1. **Feature Gap Analysis:**
   - List features you need that NGINX alone can't provide
   - For each feature, could you build it yourself? What's the engineering cost?
   - Would building it yourself be faster/cheaper than using an API Gateway?

2. **Cost-Benefit Analysis:**
   - API Gateway cost: ~$3.50 per million requests
   - Calculate monthly cost at 100K requests/second
   - Calculate engineering cost to build missing features:
     - Rate limiting per API key: 2 engineer-months
     - API analytics: 3 engineer-months
     - Partner management: 2 engineer-months
   - Which is more cost-effective? Show your calculations.

3. **Rate Limiting Design:**
   - You have 50 partners with different rate limits
   - Design rate limiting strategy:
     - Per API key? Per IP? Per endpoint?
     - Sliding window vs fixed window?
     - How do you handle burst traffic?
   - If you implement this in NGINX vs API Gateway, what's the difference?

4. **Migration Strategy:**
   - You can't migrate all 25 services at once (too risky)
   - Design a phased migration plan
   - How would you run NGINX and API Gateway simultaneously?
   - What metrics would determine if migration is successful?

5. **Security Consideration:**
   - API Gateway sits at the edge (public internet → API Gateway → services)
   - What security features do you need?
   - How would you handle:
     - JWT validation
     - OAuth 2.0 flows
     - API key rotation
     - DDoS protection

---

## Question 4: REST API Design for Social Media Platform

**Scenario:**
You're designing the REST API for a new social media platform (competitor to Twitter/Instagram). The interviewer wants you to design the core endpoints and handle edge cases.

**Requirements:**
- Users can post content (text, images, videos)
- Users can follow/unfollow other users
- Users can like, comment, and share posts
- Users have a personalized feed
- Users can search posts and users
- Support pagination (millions of posts)

**Questions:**

1. **Core API Design:**
   Design RESTful endpoints for:
   - Creating a post
   - Getting user's posts
   - Following a user
   - Liking a post
   - Getting a user's feed
   - For each endpoint, specify:
     - HTTP method
     - URL path
     - Request body (if applicable)
     - Response structure
     - Status codes

2. **Nested Resources:**
   - How would you design endpoints for post comments?
   - Option A: `/posts/{post_id}/comments`
   - Option B: `/comments?post_id={post_id}`
   - Which is more RESTful? What are the trade-offs?

3. **Pagination Strategy:**
   - A user's feed might have millions of posts
   - Design pagination for `GET /feed`
   - Compare:
     - Offset-based: `GET /feed?page=2&limit=20`
     - Cursor-based: `GET /feed?cursor=xyz&limit=20`
   - Which would you choose for a social media feed? Why?

4. **Idempotency for Critical Operations:**
   - A user tries to "like" a post 3 times (network issues causing retries)
   - How do you ensure the like is only counted once?
   - Design the endpoint: `POST /posts/{post_id}/like`
   - Should this be idempotent? How would you implement it?

5. **Rate Limiting Headers:**
   - Free users: 100 requests/hour
   - Premium users: 10,000 requests/hour
   - Design the rate limiting headers returned in API responses
   - What happens when a user exceeds their limit?
   - How do you communicate when they can retry?

6. **API Versioning Strategy:**
   - You need to make breaking changes (changing response format)
   - Design a versioning strategy:
     - URL versioning: `/v1/posts` vs `/v2/posts`
     - Header versioning: `Accept: application/vnd.myapp.v2+json`
   - Which would you choose? Why?
   - How do you handle clients still on v1?

---

## Question 5: HTTP Status Codes and Error Handling

**Scenario:**
You're reviewing an API that a junior engineer built. The API returns `200 OK` for everything, including errors:

```json
POST /payments
Response: 200 OK
{
  "success": false,
  "error": "Insufficient funds"
}
```

**Questions:**

1. **What's Wrong Here?**
   - Why is returning `200 OK` with `success: false` problematic?
   - How should this error be handled instead?
   - What status code should be used? Why?

2. **Design Proper Error Responses:**
   Design proper HTTP responses for these scenarios:
   - User tries to create a post without authentication
   - User tries to delete another user's post
   - User tries to like a post that doesn't exist
   - Payment processing fails due to insufficient funds
   - Database is down
   - Rate limit exceeded
   - For each: Specify status code, response body, any relevant headers

3. **Client Retry Strategy:**
   - Client receives `500 Internal Server Error`
   - Client receives `503 Service Unavailable`
   - Client receives `429 Too Many Requests`
   - For each status code:
     - Should the client retry?
     - How long should they wait?
     - What headers would help the client decide?

4. **Idempotent Payment Processing:**
   - A user tries to make a $100 payment
   - Network glitch causes the request to be sent 3 times
   - Design an idempotent payment endpoint:
     - What HTTP method?
     - What request headers?
     - What response if duplicate detected?
     - How long do you store idempotency keys?

---

## Question 6: Global CDN and DNS Strategy

**Scenario:**
You're the infrastructure architect at Netflix. You need to design the DNS and content delivery strategy for 250 million subscribers worldwide.

**Distribution:**
- North America: 40% (100M)
- Europe: 35% (87.5M)
- Asia: 20% (50M)
- South America: 5% (12.5M)

**Content:**
- Video catalog: 10,000 movies/shows
- Average movie size: 5 GB (4K quality)
- Each user streams 2 hours/day average
- Peak: 8-11 PM local time in each region

**Questions:**

1. **DNS Geolocation Routing:**
   - Users type `netflix.com`
   - Design a DNS strategy that routes users to the nearest data center
   - How does DNS know where the user is located?
   - What happens if the nearest data center is down?

2. **DNS TTL Strategy:**
   - What TTL would you set for `netflix.com`?
   - Considerations:
     - Need to failover quickly if a data center goes down
     - Don't want excessive DNS queries (billions of users)
     - Users binge-watch for hours (same session)
   - Justify your choice (5 minutes? 1 hour? 24 hours?)

3. **Multi-CDN Strategy:**
   - Instead of building your own CDN, you use multiple CDN providers:
     - Akamai (largest, most expensive)
     - Cloudflare (good price/performance)
     - Fastly (excellent performance, smaller)
   - Design a strategy to split traffic across 3 CDNs
   - How does DNS help route users to different CDNs?
   - What happens if one CDN fails?

4. **Caching Strategy:**
   - Popular movie (Stranger Things): 50M viewers in first week
   - Should every CDN edge server cache it?
   - Calculate storage: 10,000 movies × 5 GB = 50 TB
   - If you have 1,000 edge servers, can each store 50 TB?
   - Design a hierarchical caching strategy (edge, regional, origin)

5. **DNS Failover Scenario:**
   - US East data center goes down (40% of North American traffic)
   - You have 60 seconds to redirect traffic to US West
   - Current DNS TTL: 5 minutes
   - Problem: Users who cached DNS 4 minutes ago won't get the update for another minute
   - How do you minimize user impact?
   - Would you accept this 60-second failover time, or reduce TTL?

---

## Question 7: Proxy Architecture for Security and Performance

**Scenario:**
You're designing the proxy architecture for a banking application that must comply with strict security requirements.

**Requirements:**
- All traffic must be logged for compliance (7-year retention)
- Must filter malicious traffic (SQL injection, XSS attacks)
- Must terminate SSL/TLS at the edge
- Must support corporate VPN users and public internet users
- Backend servers must never be directly accessible from internet
- Must support 50K concurrent users

**Questions:**

1. **Forward Proxy for Corporate Users:**
   - Corporate employees access the banking app through a VPN
   - Design a forward proxy setup for corporate network
   - What benefits does this provide?
   - How does it help with compliance/security?
   - Can you enforce "only access banking app through forward proxy" policy?

2. **Reverse Proxy Architecture:**
   - Design a reverse proxy setup (NGINX or similar)
   - Features needed:
     - SSL termination (HTTPS → HTTP to backend)
     - Request logging
     - DDoS protection
     - Security filtering
   - Draw the request flow from user to backend server

3. **SSL Termination Trade-offs:**
   - Terminating SSL at reverse proxy vs at application servers
   - Pros of termination at reverse proxy:
     - Less CPU load on app servers
     - Centralized certificate management
   - Cons:
     - Traffic between proxy and app servers is unencrypted (if on same network)
   - For a banking app, is this acceptable? How would you secure proxy → server traffic?

4. **Logging and Compliance:**
   - Must log every API request for 7 years
   - At 50K concurrent users making 10 requests/minute each:
     - Requests/second = ?
     - Log entries/day = ?
     - Storage needed for 7 years = ? (assume 1 KB per log entry)
   - Where should logging happen? Reverse proxy? App servers? Both?

5. **Defense in Depth:**
   - Reverse proxy blocks 99% of malicious traffic
   - But 1% of sophisticated attacks get through
   - Design a multi-layer security architecture:
     - Layer 1: Reverse proxy (NGINX)
     - Layer 2: Application firewall (WAF)
     - Layer 3: Application-level validation
   - For each layer, what attacks does it stop?

---

## Question 8: Load Balancing Algorithm Selection

**Scenario:**
You're optimizing load balancing for different services in a microservices architecture. Each service has different characteristics.

**Service A - User Authentication:**
- Short requests (50ms average)
- Uniform load per request
- Stateless (session in Redis)
- 10,000 requests/second

**Service B - Video Encoding:**
- Long requests (5 minutes average)
- Heavy CPU usage
- Each request pins a worker thread
- 100 requests/second

**Service C - WebSocket Chat:**
- Long-lived connections (hours)
- Low CPU but high memory (maintain connection state)
- 500,000 concurrent connections

**Questions:**

1. **Algorithm Selection:**
   - For each service, choose the best load balancing algorithm:
     - Round Robin
     - Least Connections
     - IP Hash
     - Weighted Round Robin
   - Justify your choice for each service

2. **Service A (Authentication) Deep Dive:**
   - Why might Round Robin work well here?
   - What assumption are we making about server capacity?
   - What if one server has older hardware (half the CPU cores)?

3. **Service B (Video Encoding) Deep Dive:**
   - Why is Least Connections better than Round Robin?
   - Scenario: Server 1 has 2 encoding jobs (10 min remaining), Server 2 has 5 encoding jobs (1 min each remaining)
   - Least Connections would send to Server 1 (2 < 5)
   - Is this optimal? What's the limitation of Least Connections?

4. **Service C (WebSocket Chat) Deep Dive:**
   - Why might you need IP Hash or Cookie-Based stickiness?
   - What happens if a user's request is routed to a different server mid-conversation?
   - Alternative: Make WebSocket servers stateless (state in Redis). Compare approaches.

5. **Weighted Load Balancing:**
   - You're migrating from old servers to new servers
   - Old servers: 4 cores, handle 1,000 req/sec each
   - New servers: 16 cores, handle 4,000 req/sec each
   - You have 6 old servers and 4 new servers
   - Design weighted round-robin: What weights would you assign?

---

## Question 9: API Design for Payment Processing

**Scenario:**
You're designing the payment API for an e-commerce platform (similar to Shopify). The API must handle payment processing for thousands of merchants.

**Requirements:**
- Process credit card payments
- Support refunds
- Handle multiple currencies
- Prevent duplicate charges (idempotency)
- Provide webhook notifications for payment status
- Comply with PCI-DSS (never store credit card numbers)

**Questions:**

1. **Core Payment Endpoints:**
   Design RESTful endpoints for:
   - Creating a payment
   - Getting payment status
   - Refunding a payment
   - Listing all payments for a merchant

   For each endpoint:
   - HTTP method
   - URL path
   - Request/response structure
   - Status codes
   - Critical headers

2. **Idempotency Implementation:**
   - User clicks "Pay" button twice due to slow network
   - Design idempotency mechanism:
     - How do you identify duplicate requests?
     - How long do you store idempotency keys?
     - What response do you return for a duplicate request?

   Example:
   ```
   POST /payments
   Headers: {Idempotency-Key: "order-123-payment-xyz"}
   Body: {amount: 100, currency: "USD"}
   ```

   - First request: Process payment
   - Second request (duplicate): Return what?

3. **Handling Async Payment Processing:**
   - Payment authorization is fast (200ms)
   - But settlement can take minutes/hours
   - Design API flow:
     - Initial request returns what status?
     - How does merchant know when payment is complete?
     - Webhooks vs polling?

4. **Error Handling and Retry Logic:**
   Design error responses for:
   - Insufficient funds
   - Invalid credit card
   - Payment gateway timeout
   - Payment gateway down

   For each error:
   - Status code?
   - Should client retry?
   - What should the error response include?

5. **Webhook Design:**
   - Payment status changes: pending → processing → completed
   - Design webhook payload structure
   - How do you ensure webhooks are delivered?
   - What if merchant's server is down?
   - How do you prevent replay attacks (someone resends webhook)?

---

## Question 10: DNS and Load Balancer Disaster Recovery

**Scenario:**
You're the infrastructure lead at a financial services company. Your primary data center (US East) goes down completely - power outage, expected 6-hour restoration time.

**Infrastructure:**
- **Primary:** US East (handles 70% of traffic normally)
- **Secondary:** US West (handles 30% of traffic normally)
- **Secondary:** Europe (read-only replica, disaster recovery only)

**Current DNS Setup:**
- DNS TTL: 5 minutes
- Weighted routing: 70% to US East, 30% to US West
- Health checks: Every 30 seconds

**Questions:**

1. **Immediate Impact (First 5 Minutes):**
   - DNS health checks detect US East failure
   - DNS updates: Remove US East, 100% traffic to US West
   - But users who cached DNS 4 minutes ago still try US East
   - Calculate:
     - What % of users are affected in minutes 0-5?
     - What's the user experience? (Immediate error? Timeout?)
     - How long until all users are redirected to US West?

2. **Capacity Planning:**
   - US West normally handles 30% of traffic (30K req/sec)
   - Now needs to handle 100% (100K req/sec)
   - You have 10 servers in US West, each handling 3K req/sec
   - Can you handle the load? What happens?
   - What's your immediate action plan?

3. **Database Failover:**
   - Primary database in US East (read/write)
   - Replica database in US West (read-only, 5-second replication lag)
   - When US East fails:
     - Promote US West replica to primary
     - What happens to the 5 seconds of un-replicated data?
     - How do you handle potential data loss?

4. **TTL Trade-off Analysis:**
   - Current TTL: 5 minutes
   - Proposal: Reduce to 30 seconds for faster failover
   - Calculate impact:
     - Current: Up to 5 minutes for all users to failover
     - Proposed: Up to 30 seconds for all users to failover
   - BUT:
     - DNS queries increase from (100M users / 300 seconds) to (100M users / 30 seconds)
     - Calculate QPS increase on DNS servers
     - Is this worth it?

5. **Recovery Plan:**
   - US East comes back online after 6 hours
   - Current DNS: 100% traffic to US West
   - You want to shift back to 70% US East, 30% US West
   - Should you:
     - A) Immediately update DNS weights (instant shift)
     - B) Gradually shift over 1 hour (10% every 6 minutes)
     - What are the risks of each approach?

---

## Evaluation Rubric

When reviewing your answers, assess yourself on:

**API Design (30%)**
- RESTful principles
- Proper HTTP methods and status codes
- Resource naming and nesting
- Idempotency and error handling

**Architecture Decisions (30%)**
- Appropriate use of L4 vs L7 load balancers
- Proxy architecture understanding
- DNS strategy and trade-offs
- Multi-layer security

**Scalability & Performance (20%)**
- Load balancing algorithm selection
- Caching and DNS TTL strategy
- Handling high traffic scenarios
- Geographic distribution

**Operational Excellence (20%)**
- Monitoring and health checks
- Disaster recovery planning
- Capacity planning
- Cost-benefit analysis

---

## Answer Key - Available Separately
Detailed solutions with step-by-step reasoning, architecture diagrams, and best practices are provided in a separate file. Attempt all questions before reviewing answers.
