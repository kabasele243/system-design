# REST vs gRPC

## Learning Objectives
- Understand REST API design principles
- Learn gRPC and Protocol Buffers
- Master when to use each protocol

## Notes

### REST API Overview
- Uses HTTP methods (GET, POST, PUT, DELETE)
- Returns JSON (human-readable text)
- URLs represent resources (`/restaurants/123`)
- Stateless requests
- What you're already familiar with!

### Problems with REST at Scale

#### 1. JSON is Text-Based
- Needs parsing (CPU intensive)
- Large payload size (field names repeated every response)
- Example: `{"id": 123, "name": "Pizza Palace"}` = ~80 bytes

#### 2. HTTP/1.1 Limitations
- One request per connection
- Sequential requests (can't multiplex)
- High latency for multiple calls

#### 3. No Type Safety
- No enforced schema
- Can send `"rating": "four"` instead of `4.5`
- Errors discovered at runtime, not compile time

#### 4. Bandwidth Waste at High Scale
- 1000 req/sec × 150 bytes overhead = 150 KB/sec wasted
- At Netflix scale (1M req/sec) = 12.96 TB/day wasted!
- Costs $425K/year just in bandwidth savings

### gRPC - Optimized for Service Communication

**gRPC = Google Remote Procedure Call**

#### Key Advantages

**1. Binary Format (Protocol Buffers)**
- ~70% smaller than JSON
- 5-10x faster to parse
- Not human-readable, but computers process faster
- Example: Same data = 22 bytes vs 80 bytes JSON

**2. Type Safety with Schema**
```protobuf
message Restaurant {
  int32 id = 1;           // MUST be integer
  string name = 2;        // MUST be string
  float rating = 3;       // MUST be float
  bool isOpen = 4;        // MUST be boolean
}
```
- Enforced contract both sides agree on
- Errors caught at compile time, not production
- Auto-generates client/server code

**3. HTTP/2 Multiplexing**
- Multiple requests over one connection simultaneously
- Parallel, not sequential
- Much faster for multiple calls

#### gRPC Communication Patterns

**1. Unary RPC (like REST)**
- Client sends 1 request → Server sends 1 response
- Use for: Simple queries, single operations

**2. Server Streaming**
- Client sends 1 request → Server sends MULTIPLE responses
- Use for: Live order tracking, progress updates
- No polling needed!

**3. Client Streaming**
- Client sends MULTIPLE requests → Server sends 1 response
- Use for: File uploads in chunks, batch operations

**4. Bidirectional Streaming**
- Both sides send/receive multiple messages simultaneously
- Use for: Live chat, real-time collaboration
- Like WebSockets but with type safety!

### When to Use REST vs gRPC

#### Use REST for:
✅ **Client-facing APIs** (browsers, mobile apps)
- Browsers natively support REST
- Human-readable for debugging (curl, Postman)
- JSON is flexible
- HTTP caching works out-of-box

✅ **Public APIs** for third-party developers
- Easier to understand and use
- Better documentation
- Examples: Stripe, Twitter, GitHub APIs

#### Use gRPC for:
✅ **Service-to-service communication** (backend microservices)
- 5-10x faster performance
- 70% smaller payloads
- Type safety prevents bugs
- Built-in streaming

✅ **High-performance requirements**
- Low latency critical
- High throughput needed
- Examples: Google internal services, Netflix microservices

### Real-World Architecture Pattern

**Most companies use BOTH:**

```
Mobile App ───REST───┐
                     ├──→ API Gateway ───gRPC───→ Microservices
Web Browser ──REST───┘                            (Order, Payment, etc.)
```

- External clients use REST (easy, compatible)
- API Gateway translates REST → gRPC
- Internal services use gRPC (fast, efficient)

### Event-Driven vs Request-Response

**They are COMPLEMENTARY, not opposing!**

#### Request-Response (REST/gRPC) - Synchronous
- **Use when**: You need answer RIGHT NOW
- **Example**: "Can I charge this card?" → Must wait for yes/no
- **Characteristics**: Tight coupling, immediate response, fails if service down

#### Event-Driven (Message Queue) - Asynchronous
- **Use when**: Fire-and-forget, side effects
- **Example**: "OrderPlaced" → Email/Analytics/Loyalty all react
- **Characteristics**: Loose coupling, eventual consistency, resilient

#### Use Both Together
**Order placement flow:**
1. **Synchronous validation** (gRPC): Check restaurant open, validate payment
2. **Async side effects** (Events): Send receipt email, update analytics, add loyalty points

**Golden rule**: Synchronous for critical path, async for everything else

## Practice Questions

1. **Why does gRPC use Protocol Buffers instead of JSON?**
   - 70% smaller payload size
   - 5-10x faster parsing (binary vs text)
   - Type safety enforced at compile time
   - Bandwidth savings at scale = significant cost reduction

2. **When would you choose REST over gRPC?**
   - Browser/mobile client-facing APIs
   - Public APIs for external developers
   - Human-readable debugging needed
   - HTTP caching important

3. **What's the difference between server streaming and bidirectional streaming?**
   - Server streaming: Client sends once, server pushes multiple responses
   - Bidirectional: Both sides send/receive multiple messages simultaneously
   - Bidirectional = like WebSockets but with type safety

4. **How do event-driven and request-response patterns work together?**
   - Request-response for critical path (must succeed for operation to complete)
   - Events for side effects (can happen async, not blocking)
   - Example: Charge card (sync) then send receipt (async)

## Real-World Examples

- **Google**: 100% gRPC internally (Gmail, YouTube, Maps all use gRPC)
- **Netflix**: gRPC for service-to-service, REST for client apps
- **Uber**: gRPC between backend services, REST for mobile apps
- **Square**: Public REST API, internal gRPC for microservices

## Resources & References

- gRPC Official Documentation
- Protocol Buffers Guide
- HTTP/2 Specification
- Martin Fowler: Remote Procedure Calls
