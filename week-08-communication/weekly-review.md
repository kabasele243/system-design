# Week 8: Communication & Real-Time - Weekly Review

## Summary of Key Concepts

### Communication Protocols
1. **REST** - Client-facing APIs, simple CRUD, human-readable
2. **gRPC** - Service-to-service, 70% smaller, 5-10x faster, type-safe
3. **GraphQL** - Flexible querying, solve over/under-fetching, N+1 problem
4. **WebSockets** - Bidirectional real-time for browsers
5. **SSE** - One-way real-time, auto-reconnect, simpler than WebSockets

### API Gateway
- Single entry point for microservices
- Core features: Routing, authentication, rate limiting, protocol translation
- BFF pattern for client-specific gateways
- Implementation: Managed (AWS) vs Open-source (Kong, Envoy)

### Decision Tree
```
External clients? → REST
Service-to-service? → gRPC
Complex queries? → GraphQL
Real-time bidirectional? → WebSockets
Real-time one-way? → SSE
```

---

## Review Questions

### Question 1: Protocol Selection (REST vs gRPC vs WebSocket)

**Scenario:** You're building a food delivery app with these requirements:
- Mobile app needs restaurant data
- Backend microservices communicate frequently
- Users want real-time order tracking
- Admin dashboard shows live metrics

**Which protocol would you use for each and why?**

<details>
<summary>Answer</summary>

1. **Mobile app → API Gateway: REST**
   - Browser/mobile native support
   - Human-readable, easy debugging
   - Flexible JSON format

2. **API Gateway → Backend microservices: gRPC**
   - 5-10x faster than REST
   - 70% smaller payloads (bandwidth savings)
   - Type safety prevents bugs
   - High throughput internal communication

3. **Real-time order tracking: WebSockets**
   - Bidirectional (driver sends location, user sends messages)
   - Sub-second latency
   - Persistent connection

4. **Admin live metrics: SSE**
   - One-way (server pushes metrics)
   - Simpler than WebSockets
   - Auto-reconnect built-in
   - Perfect for dashboard updates

</details>

---

### Question 2: WebSocket Architecture

**Scenario:** Your delivery app has 1 million concurrent users tracking orders. Design the WebSocket architecture.

**Address:**
- How do you prevent one server from handling all connections?
- How do you broadcast driver location to users on different servers?
- How do you handle connection failures?

<details>
<summary>Answer</summary>

**Architecture:**

```
         Load Balancer (sticky sessions)
         /       |       \
   Server1   Server2   Server3
   (333K)    (333K)    (333K)
      \        |        /
       \       |       /
         Redis Pub/Sub
            ↑
      Backend Services
```

**Solutions:**

1. **Horizontal Scaling:**
   - 3+ servers, each handles 333K connections
   - Sticky sessions: User always routes to same server
   - Each connection: ~10KB memory

2. **Broadcasting Across Servers:**
   - Driver updates location → Backend publishes to Redis
   - All WebSocket servers subscribe to Redis
   - Each server forwards to connected clients
   - Room pattern: Only users tracking that order get update

3. **Connection Failures:**
   - **Heartbeat/Ping-Pong**: Server pings every 30s, detect dead connections
   - **Client reconnection**: Exponential backoff (3s, 6s, 12s...)
   - **Message acknowledgment**: Client confirms receipt, retry if missing

4. **Optimization:**
   - WebSocket compression: 60-80% size reduction
   - Group connections by order ID (rooms)
   - Dedicated WebSocket gateway separate from business logic

</details>

---

### Question 3: GraphQL Design

**Scenario:** Your REST API has performance issues:
- Mobile app makes 5 API calls to load restaurant page
- Returns 500KB but mobile only needs 50KB
- Adding new fields requires backend deployment

**Design a GraphQL solution addressing:**
- Schema design
- N+1 problem prevention
- Query complexity protection
- Caching strategy

<details>
<summary>Answer</summary>

**1. Schema Design:**
```graphql
type Restaurant {
  id: ID!
  name: String!
  rating: Float!
  menu(limit: Int): [MenuItem!]!
  reviews(limit: Int): [Review!]!
}

type Query {
  restaurant(id: ID!): Restaurant
  searchRestaurants(cuisine: String, limit: Int): [Restaurant!]!
}
```

**2. N+1 Prevention with DataLoader:**
```javascript
const restaurantLoader = new DataLoader(async (ids) => {
  // Batches all requests into ONE query
  return await db.query('SELECT * FROM restaurants WHERE id IN (?)', ids);
});

const resolvers = {
  Order: {
    restaurant: (order) => restaurantLoader.load(order.restaurant_id)
  }
};
```
- 10 orders → 2 queries instead of 11
- DataLoader batches and caches within request

**3. Query Complexity Protection:**
```javascript
const server = new ApolloServer({
  validationRules: [
    depthLimit(5),                    // Max nesting depth
    createComplexityLimitRule(1000)   // Max complexity score
  ],
  plugins: [
    queryTimeout({ timeout: 5000 })   // 5 second timeout
  ]
});
```

**4. Normalized Caching (Apollo Client):**
```javascript
// Cache by TYPE + ID, not query
{
  "Restaurant:123": { name: "Pizza Palace", rating: 4.5 },
  "MenuItem:1": { name: "Margherita", price: 12 }
}
```
- Any query for Restaurant:123 gets cache hit
- Missing fields trigger partial fetch
- 80% cache hit rate vs 20% with query-based

</details>

---

### Question 4: API Gateway Patterns

**Scenario:** Your API Gateway is becoming a bottleneck and single point of failure.

**Design solutions for:**
- High availability
- Rate limiting across multiple gateways
- BFF pattern for mobile vs web
- Security (preventing backend bypass)

<details>
<summary>Answer</summary>

**1. High Availability:**
```
         Load Balancer (health checks)
         /      |      \
    Gateway1 Gateway2 Gateway3
        \      |      /
         Backend Services (private subnet)
```
- 3-5 gateway instances
- Load balancer distributes traffic
- Health checks every 5 seconds
- Remove failed gateways automatically

**2. Distributed Rate Limiting (Redis):**
```javascript
// All gateways share Redis counter
const count = await redis.incr(`rate_limit:user_${userId}:${minute}`);
if (count === 1) {
  await redis.expire(key, 60);
}
if (count > 100) {
  return res.status(429).send('Too Many Requests');
}
```
- Atomic increment in Redis
- All gateways see same count
- Auto-expire after 60 seconds

**3. BFF Pattern:**
```
Mobile App → Mobile BFF (50KB, top 5 items)
                ↓
Web App → Web BFF (500KB, all data, reviews)
                ↓
            Backend Services
```
**Trade-offs:**
- ✅ Optimized per client (mobile saves bandwidth)
- ❌ Code duplication (use shared libraries)
- **When to use:** Very different client needs, large team

**4. Network Security:**
- **Gateway**: Public subnet (accessible from internet)
- **Backend**: Private subnet (only gateway can reach)
- **Firewall rules**: Block all external traffic to backend
- **Service trust**: Backend trusts gateway-added headers (`X-User-Id`)

</details>

---

### Question 5: Real-Time System Design

**Scenario:** Design a real-time notification system for a social media app supporting:
- New post notifications (100K users)
- Live comments on posts
- Typing indicators
- Offline message queue

**Address: Protocol choice, scaling, reliability, delivery guarantees**

<details>
<summary>Answer</summary>

**Architecture:**

```
Mobile/Web Clients
      ↓
   API Gateway (REST for posts, auth)
      ↓
┌─────────────────────────────────┐
│   WebSocket Gateway Layer       │  (handles 100K connections)
│   (Server 1-5, sticky sessions) │
└─────────────────────────────────┘
      ↓
   Redis Pub/Sub (message routing)
      ↓
┌─────────────────────────────────┐
│   Backend Services              │
│   - Post Service                │
│   - Comment Service             │
│   - Notification Service        │
└─────────────────────────────────┘
      ↓
   PostgreSQL + Message Queue
```

**1. Protocol Choices:**
- **New post notifications**: SSE
  - One-way (server pushes)
  - Auto-reconnect
  - Simple implementation

- **Live comments + typing**: WebSockets
  - Bidirectional (users post comments)
  - Real-time typing indicators
  - Sub-second latency

**2. Scaling:**
- **Connection layer**: 5 WebSocket servers × 20K connections
- **Sticky sessions**: User always routes to same server
- **Redis Pub/Sub**: Broadcast across servers
  ```javascript
  // User posts comment
  redis.publish(`post_${postId}_comments`, commentData);

  // All servers subscribe
  redis.subscribe(`post_*_comments`, (channel, data) => {
    // Forward to WebSocket clients watching this post
    rooms[postId].forEach(ws => ws.send(data));
  });
  ```

**3. Reliability:**
- **Message acknowledgment**:
  ```javascript
  ws.send({ id: 'msg_123', type: 'comment', data: {...} });
  // Wait for ack, retry after 5s if missing
  ```
- **Heartbeat**: Ping every 30s, detect dead connections
- **Reconnection**: Exponential backoff (3s, 6s, 12s...)

**4. Offline Message Queue:**
- **Persistent storage**: PostgreSQL for offline users
  ```sql
  CREATE TABLE pending_notifications (
    user_id INT,
    notification JSON,
    created_at TIMESTAMP
  );
  ```
- **On reconnect**: Send missed notifications
  ```javascript
  ws.on('open', async () => {
    const pending = await db.query(
      'SELECT * FROM pending_notifications WHERE user_id = ?',
      userId
    );
    pending.forEach(notif => ws.send(notif));
    await db.query('DELETE FROM pending_notifications WHERE user_id = ?', userId);
  });
  ```

**5. Delivery Guarantees:**
- **At-least-once delivery**: Retry on failure, use idempotency keys
- **Event sourcing**: Store all events, rebuild state if needed
- **Dead letter queue**: Failed messages after 3 retries → investigate

**6. Optimizations:**
- **Room pattern**: Group users by post ID (don't broadcast to all 100K)
- **Compression**: Enable WebSocket compression (60% reduction)
- **Rate limiting per user**: 100 messages/minute

</details>

---

## Practice Exercise

**Design a real-time notification system for a social media app.**

### Requirements:
- 1 million active users
- New post notifications
- Live comments and likes
- Typing indicators
- Must work offline (queue messages)
- Sub-second latency for online users

### Components to Design:
1. **Protocol selection** (REST/gRPC/WebSockets/SSE)
2. **Architecture diagram** (clients, gateways, services, databases)
3. **Scaling strategy** (how to handle 1M connections)
4. **Reliability** (message delivery guarantees, reconnection)
5. **Offline handling** (queue and replay)

### Bonus Challenges:
- How do you prevent notification spam?
- How do you prioritize urgent notifications?
- How do you support push notifications on mobile?
- What metrics would you monitor?

---

## Key Takeaways

### Protocol Decision Matrix
| Need | Protocol |
|------|----------|
| Client-facing API | REST |
| Service-to-service | gRPC |
| Complex flexible queries | GraphQL |
| Real-time bidirectional | WebSockets |
| Real-time one-way push | SSE |

### Scaling Patterns
- **Horizontal scaling**: Multiple instances + load balancer
- **Sticky sessions**: Required for stateful connections (WebSockets/SSE)
- **Redis Pub/Sub**: Broadcast across servers
- **Room pattern**: Group connections, don't broadcast to all

### Common Pitfalls
1. **N+1 queries in GraphQL** → Use DataLoader
2. **Expensive GraphQL queries** → Depth/complexity limits
3. **WebSocket connection exhaustion** → Horizontal scaling + Redis
4. **API Gateway SPOF** → Multiple instances + health checks
5. **Rate limiting per gateway** → Shared Redis counter

---

## Resources & References

### Official Documentation
- gRPC Documentation
- GraphQL Specification
- WebSocket RFC 6455
- Server-Sent Events W3C Spec

### Architecture Patterns
- Martin Fowler: Backend for Frontend (BFF)
- API Gateway Pattern
- Event-Driven Architecture

### Real-World Examples
- Netflix: gRPC + BFF architecture
- Facebook: GraphQL creator
- Slack: WebSockets scaling
- Uber: Real-time location tracking

### Tools & Libraries
- Kong/Envoy (API Gateways)
- DataLoader (GraphQL N+1 prevention)
- Socket.io (WebSocket library)
- Redis Pub/Sub (Message broadcasting)
