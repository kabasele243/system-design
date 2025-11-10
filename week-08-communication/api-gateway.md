# API Gateway

## Learning Objectives
- Understand API Gateway patterns
- Learn request routing, rate limiting, auth
- Master Backend for Frontend (BFF) pattern

## Notes

### What is an API Gateway?
- **Single entry point** for all client requests to backend microservices
- Acts like a hotel concierge - coordinates multiple services instead of clients calling each directly
- Solves problems: too many network calls, over-fetching, authentication complexity, protocol translation

### Problems Without API Gateway
1. **Multiple calls** - Mobile app makes 5-10 requests to load one screen (slow)
2. **Over-fetching** - Each service returns all data, client only needs subset
3. **Authentication chaos** - Client authenticates with 5 different services
4. **Different protocols** - Client handles REST, gRPC, etc.
5. **Tight coupling** - Clients know all service URLs and endpoints

### Core API Gateway Features

#### 1. Request Routing
- Routes requests to appropriate backend service based on URL path
- Acts as reverse proxy/traffic cop
- Example: `/restaurants/*` → Restaurant Service, `/orders/*` → Order Service

#### 2. Authentication & Authorization
- **Validates auth once** at gateway before reaching backend services
- Backend services trust the gateway (in private subnet)
- Gateway adds user context headers (`X-User-Id`, `X-User-Role`)
- **Network isolation**: Backend services in private subnet, only gateway can reach them

#### 3. Rate Limiting
- Prevents API abuse and system overload
- Uses Redis for distributed rate limiting across multiple gateway instances
- Strategies:
  - Per-user limits (100 req/min)
  - Per-IP limits (for anonymous users)
  - Tiered limits (Free: 100/hr, Pro: 10K/hr)
- Returns `HTTP 429 Too Many Requests` when exceeded

**Why Redis for rate limiting?**
- Multiple gateway instances need shared counter
- Local memory would allow 100 req/min PER gateway (not total)
- Redis provides atomic increments and automatic expiration

#### 4. Backend for Frontend (BFF) Pattern
- **Problem**: Different clients need different data (mobile vs web vs watch)
- **Solution**: Separate gateway per client type
  - Mobile BFF - Returns 50KB optimized for 4G
  - Web BFF - Returns 500KB with rich data for fast WiFi
  - Watch BFF - Returns 5KB minimal data

**Trade-offs:**
- ✅ Optimized responses per client
- ✅ Better performance
- ❌ Code duplication across BFFs
- ❌ Maintenance overhead (3 gateways to update)

**When to use BFF:**
- Very different client requirements
- Performance critical (mobile bandwidth)
- Large team (can afford multiple teams)

**When to use Single Gateway:**
- Simple API needs
- Small team
- Clients need similar data

### High Availability
- **Single Point of Failure risk** - If gateway down, entire system unreachable
- **Solution**: Multiple gateway instances + Load Balancer
- Load balancer health checks, routes traffic to healthy gateways
- Typically run 3-5 gateway instances

### Implementation Options

#### Option 1: Managed Cloud Services
- **AWS API Gateway, Azure API Management, GCP API Gateway**
- ✅ Zero infrastructure management
- ✅ Auto-scaling, monitoring built-in
- ✅ Fast to set up
- ❌ Vendor lock-in
- ❌ Expensive at scale ($3.50/million requests)
- **Best for**: Startups, small teams, fast iteration

#### Option 2: Open-Source Software
- **NGINX**: Fast, lightweight, but complex config
- **Kong**: Built on NGINX, plugin ecosystem, easier
- **Envoy**: Modern, cloud-native, excellent observability
- ✅ More control, cheaper at scale
- ❌ Must deploy and manage yourself
- **Best for**: Mid-size companies with infrastructure team

#### Option 3: Build Your Own
- ❌ Almost never recommended
- Only for extremely specific needs

## Practice Questions

1. **Why is rate limiting needed at the gateway level?**
   - Prevents buggy clients from overwhelming system
   - Stops malicious actors from abuse
   - Ensures fair resource distribution
   - Protects against cost explosions

2. **How does the gateway prevent becoming a single point of failure?**
   - Run multiple instances (3-5+)
   - Load balancer distributes traffic
   - Health checks detect failures
   - Failed instances automatically removed from rotation

3. **Why use Redis instead of local memory for rate limiting?**
   - Multiple gateway instances need shared state
   - Local memory = per-instance limits (can be bypassed)
   - Redis provides atomic operations and auto-expiration

4. **When would you choose BFF pattern vs single gateway?**
   - BFF: Very different client needs, large team, performance critical
   - Single: Similar data needs, small team, rapid iteration

## Real-World Examples

- **Netflix**: Uses BFF pattern (TV app vs mobile app vs web)
- **Uber**: Separate gateways for rider app vs driver app
- **Stripe**: Single gateway (consistent API for all clients)
- **Amazon**: API Gateway handles millions of requests with auto-scaling

## Resources & References

- AWS API Gateway Documentation
- Kong Gateway Architecture
- NGINX as API Gateway
- Martin Fowler: Pattern: Backends For Frontends
