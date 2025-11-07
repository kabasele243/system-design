# Rate Limiting

## Learning Objectives
- Understand why rate limiting is important
- Learn different rate limiting algorithms
- Implement rate limiting strategies

## Notes

### Why Rate Limiting?

**Problems without rate limiting:**
- 🤖 **Bot scraping:** Competitor crawls your entire API
- 💣 **DDoS attacks:** Overwhelm server with requests
- 🔑 **Brute force:** Try millions of passwords
- 💸 **Cost:** Cloud bills skyrocket
- 😡 **Bad UX:** Slow for legitimate users

**Real-world examples:**
```
GitHub API: 5000 req/hour authenticated, 60 req/hour unauthenticated
Twitter: 300 tweets / 3 hours
Stripe: 100 req/sec in test mode
```

---

### Rate Limiting Algorithms

**1. Token Bucket**

**Analogy:** Coffee shop loyalty card
- Bucket holds tokens (like stamps on card)
- Each request costs 1 token
- Tokens refill at constant rate
- Can accumulate tokens (up to max)
- **Allows bursts!**

**How it works:**
```
Bucket capacity: 10 tokens
Refill rate: 1 token/second

Time 0s:  Bucket = 10 tokens
Request:  Bucket = 9 tokens ✅
Request:  Bucket = 8 tokens ✅
...
Request:  Bucket = 1 token ✅
Request:  Bucket = 0 tokens ✅
Request:  Bucket = 0 tokens ❌ RATE LIMITED!

Time 1s:  Bucket = 1 token (refilled)
Request:  Bucket = 0 tokens ✅
```

**Implementation:**
```javascript
class TokenBucket {
    constructor(capacity, refillRate) {
        this.capacity = capacity;
        this.tokens = capacity;
        this.refillRate = refillRate;  // tokens per second
        this.lastRefill = Date.now();
    }

    allowRequest() {
        this.refill();

        if (this.tokens >= 1) {
            this.tokens -= 1;
            return true;  // Allow
        }
        return false;  // Deny
    }

    refill() {
        const now = Date.now();
        const timePassed = (now - this.lastRefill) / 1000;  // seconds
        const tokensToAdd = timePassed * this.refillRate;

        this.tokens = Math.min(this.capacity, this.tokens + tokensToAdd);
        this.lastRefill = now;
    }
}

// Usage
const bucket = new TokenBucket(10, 1);  // 10 tokens, 1/sec refill

if (bucket.allowRequest()) {
    // Process request
} else {
    res.status(429).json({error: 'Too many requests'});
}
```

**Pros:**
- ✅ Allows bursts (post 10 tweets quickly)
- ✅ Smooth long-term rate
- ✅ Memory efficient

**Cons:**
- ❌ Complex implementation
- ❌ Edge case: Attacker can exhaust tokens

**Use cases:**
- Twitter (burst tweeting)
- Spotify (burst song skips)
- API calls (spike traffic OK)

---

**2. Leaky Bucket**

**Analogy:** Water bucket with hole at bottom
- Requests fill bucket
- Bucket "leaks" at constant rate
- If bucket overflows → reject
- **Constant output rate** (like a funnel)

**How it works:**
```
Bucket capacity: 10 requests
Leak rate: 1 request/second

Requests come in fast:
→ → → → → → → → → → → →
[Bucket fills up]

Processed at constant rate:
→   →   →   →   (1/sec, no burst)
```

**Implementation:**
```javascript
class LeakyBucket {
    constructor(capacity, leakRate) {
        this.capacity = capacity;
        this.queue = [];
        this.leakRate = leakRate;  // requests per second

        // Process queue at constant rate
        setInterval(() => {
            if (this.queue.length > 0) {
                const request = this.queue.shift();
                this.processRequest(request);
            }
        }, 1000 / leakRate);
    }

    allowRequest(request) {
        if (this.queue.length < this.capacity) {
            this.queue.push(request);
            return true;  // Queued
        }
        return false;  // Bucket full, reject
    }

    processRequest(request) {
        // Handle request
    }
}
```

**Pros:**
- ✅ Constant output rate (predictable load)
- ✅ Smooths traffic spikes

**Cons:**
- ❌ No bursts allowed
- ❌ Higher latency (requests queued)

**Use cases:**
- Payment processing (steady rate)
- Video streaming (constant bitrate)
- Background jobs

---

**3. Fixed Window Counter**

**Analogy:** Gym membership with daily visit limit
- Window: 1 hour
- Limit: 100 requests
- Reset: Every hour (00:00, 01:00, 02:00...)

**How it works:**
```
Window: 10:00-11:00, Limit: 100

10:00 → counter = 0
10:30 → counter = 50 ✅
10:59 → counter = 99 ✅
11:00 → counter = 0 (reset!)
11:01 → counter = 1 ✅
```

**Implementation:**
```javascript
class FixedWindowCounter {
    constructor(limit, windowMs) {
        this.limit = limit;
        this.windowMs = windowMs;
        this.counters = new Map();  // userId → {count, resetTime}
    }

    allowRequest(userId) {
        const now = Date.now();
        const user = this.counters.get(userId) || {count: 0, resetTime: now + this.windowMs};

        // Reset if window expired
        if (now >= user.resetTime) {
            user.count = 0;
            user.resetTime = now + this.windowMs;
        }

        // Check limit
        if (user.count < this.limit) {
            user.count++;
            this.counters.set(userId, user);
            return true;
        }

        return false;  // Rate limited
    }
}

// Usage
const limiter = new FixedWindowCounter(100, 60 * 60 * 1000);  // 100/hour
```

**Pros:**
- ✅ Simple to implement
- ✅ Memory efficient

**Cons:**
- ❌ **Edge case:** 200 requests in 1 second!

```
Window 1: 10:00-11:00  →  100 requests at 10:59 ✅
Window 2: 11:00-12:00  →  100 requests at 11:00 ✅

Total: 200 requests in 1 second! ❌
```

**Use cases:**
- Simple APIs
- Low-traffic endpoints
- Internal services

---

**4. Sliding Window Log**

**Analogy:** Teacher tracking attendance with timestamps
- Store timestamp of every request
- Remove timestamps older than window
- Count remaining timestamps

**How it works:**
```
Window: 1 hour, Limit: 100

Requests: [10:00, 10:05, 10:10, ..., 10:55]
Current time: 11:00

Remove requests older than 1 hour:
[10:00, 10:05, 10:10] ← removed
[10:15, 10:20, ..., 10:55] ← kept

Count = 85 ✅ (allow request)
```

**Implementation:**
```javascript
class SlidingWindowLog {
    constructor(limit, windowMs) {
        this.limit = limit;
        this.windowMs = windowMs;
        this.logs = new Map();  // userId → [timestamps]
    }

    allowRequest(userId) {
        const now = Date.now();
        const userLog = this.logs.get(userId) || [];

        // Remove old timestamps
        const validLog = userLog.filter(timestamp => now - timestamp < this.windowMs);

        // Check limit
        if (validLog.length < this.limit) {
            validLog.push(now);
            this.logs.set(userId, validLog);
            return true;
        }

        this.logs.set(userId, validLog);
        return false;
    }
}
```

**Pros:**
- ✅ Accurate (no edge case)
- ✅ Smooth rate limiting

**Cons:**
- ❌ **Memory intensive** (stores all timestamps)
- ❌ Expensive at scale

**Use cases:**
- High-security endpoints
- Low-traffic critical APIs
- Short windows (1 minute)

---

**5. Sliding Window Counter (BEST for production!)**

**Hybrid:** Combines Fixed Window + Sliding Window

**How it works:**
```
Current window: 11:00-12:00, count = 50
Previous window: 10:00-11:00, count = 100

Current time: 11:30 (50% into window)

Estimated count = (previous window × 50%) + current window
                = (100 × 0.5) + 50
                = 100

If 100 < limit ✅ → Allow
```

**Implementation:**
```javascript
class SlidingWindowCounter {
    constructor(limit, windowMs) {
        this.limit = limit;
        this.windowMs = windowMs;
        this.windows = new Map();  // userId → {current, previous, resetTime}
    }

    allowRequest(userId) {
        const now = Date.now();
        const window = this.windows.get(userId) || {
            current: 0,
            previous: 0,
            resetTime: now + this.windowMs
        };

        // Reset if window expired
        if (now >= window.resetTime) {
            window.previous = window.current;
            window.current = 0;
            window.resetTime = now + this.windowMs;
        }

        // Calculate weighted count
        const percentageIntoWindow = (now - (window.resetTime - this.windowMs)) / this.windowMs;
        const weightedCount = (window.previous * (1 - percentageIntoWindow)) + window.current;

        // Check limit
        if (weightedCount < this.limit) {
            window.current++;
            this.windows.set(userId, window);
            return true;
        }

        this.windows.set(userId, window);
        return false;
    }
}
```

**Pros:**
- ✅ Accurate (smooth rate limiting)
- ✅ Memory efficient (only 2 counters)
- ✅ No edge case

**Cons:**
- ❌ Slightly complex

**Use cases:**
- **Production APIs** (best choice!)
- High-traffic endpoints
- Distributed systems

---

### Comparison Table

| Algorithm | Memory | Accuracy | Bursts | Complexity | Use Case |
|-----------|--------|----------|--------|------------|----------|
| **Token Bucket** | Low | Good | ✅ Yes | Medium | Twitter, Spotify |
| **Leaky Bucket** | Medium | Good | ❌ No | Medium | Payments, streaming |
| **Fixed Window** | Low | Poor (edge case) | ❌ No | Easy | Simple APIs |
| **Sliding Log** | High | Perfect | ❌ No | Easy | Critical endpoints |
| **Sliding Counter** | Low | Great | Partial | Medium | **Production (best!)** |

---

### Rate Limiting Strategies

**1. Rate Limit by User**

```javascript
app.get('/api/tweets', requireAuth, async (req, res) => {
    const userId = req.user.id;

    if (!limiter.allowRequest(userId)) {
        return res.status(429).json({error: 'Rate limit exceeded'});
    }

    // Process request
});
```

**When:** User is authenticated
**Example:** Twitter (300 tweets/3 hours per user)

---

**2. Rate Limit by IP Address**

```javascript
app.get('/api/search', async (req, res) => {
    const ip = req.ip;

    if (!limiter.allowRequest(ip)) {
        return res.status(429).json({error: 'Too many requests from this IP'});
    }

    // Process request
});
```

**When:** Public endpoints (no login)
**Example:** Google Search

**Problem:** Multiple users behind same IP (NAT, office network)

---

**3. Rate Limit by API Key**

```javascript
app.get('/api/data', async (req, res) => {
    const apiKey = req.headers['x-api-key'];

    if (!limiter.allowRequest(apiKey)) {
        return res.status(429).json({
            error: 'API key rate limit exceeded',
            resetAt: limiter.getResetTime(apiKey)
        });
    }

    // Process request
});
```

**When:** Third-party developers
**Example:** GitHub API, Stripe API

---

### Implementation Considerations

**Where to implement:**

**1. Application level (Express middleware):**
```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
    windowMs: 15 * 60 * 1000,  // 15 minutes
    max: 100,                   // Limit per window
    standardHeaders: true,
    legacyHeaders: false,
    handler: (req, res) => {
        res.status(429).json({
            error: 'Too many requests',
            retryAfter: req.rateLimit.resetTime
        });
    }
});

app.use('/api/', limiter);
```

**2. API Gateway (NGINX, Kong, AWS API Gateway):**
```nginx
# NGINX config
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

location /api/ {
    limit_req zone=api burst=20 nodelay;
}
```

**3. Redis (distributed rate limiting):**
```javascript
const redis = require('redis');
const client = redis.createClient();

async function allowRequest(userId) {
    const key = `rate_limit:${userId}`;
    const count = await client.incr(key);

    if (count === 1) {
        await client.expire(key, 3600);  // 1 hour window
    }

    return count <= 100;  // Limit: 100/hour
}
```

---

**Response headers:**

```javascript
app.use((req, res, next) => {
    res.setHeader('X-RateLimit-Limit', '100');
    res.setHeader('X-RateLimit-Remaining', req.rateLimit.remaining);
    res.setHeader('X-RateLimit-Reset', req.rateLimit.resetTime);
    next();
});
```

**Example response:**
```
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1677686400
Retry-After: 3600

{
    "error": "Rate limit exceeded",
    "message": "Try again in 1 hour"
}
```

---

**Error handling:**

```javascript
app.use((req, res, next) => {
    if (!limiter.allowRequest(req.user.id)) {
        return res.status(429).json({
            error: 'Too Many Requests',
            message: 'You have exceeded the rate limit',
            retryAfter: limiter.getResetTime(req.user.id),
            limit: limiter.limit,
            remaining: 0
        });
    }
    next();
});
```

---

### Real-World Example: Twitter API Rate Limiting

**Tweet Posting:**
```
Algorithm: Token Bucket
Limit: 300 tweets / 3 hours
Rate Limit By: User ID
Reason: Allows burst tweeting (10 quick tweets), then cooldown
```

**Timeline Reading:**
```
Algorithm: Sliding Window Counter
Limit: 15 requests / 15 minutes
Rate Limit By: User ID
Reason: Authenticated users, prevent excessive polling
```

**Search Endpoint:**
```
Algorithm: Sliding Window Counter
Limit: 450 requests / 15 minutes (authenticated)
         180 requests / 15 minutes (unauthenticated)
Rate Limit By: IP Address (if not logged in)
Reason: Public endpoint, higher limit for authenticated
```

**Implementation:**
```javascript
// Express middleware
const tweetLimiter = new TokenBucket(300, 100/3600);  // 300 capacity, 100/hour refill
const timelineLimiter = new SlidingWindowCounter(15, 15 * 60 * 1000);
const searchLimiter = new SlidingWindowCounter(450, 15 * 60 * 1000);

app.post('/api/tweets', requireAuth, (req, res) => {
    if (!tweetLimiter.allowRequest(req.user.id)) {
        return res.status(429).json({error: 'Tweet limit exceeded'});
    }
    // Post tweet
});

app.get('/api/timeline', requireAuth, (req, res) => {
    if (!timelineLimiter.allowRequest(req.user.id)) {
        return res.status(429).json({error: 'Timeline rate limit exceeded'});
    }
    // Get timeline
});

app.get('/api/search', (req, res) => {
    const identifier = req.user?.id || req.ip;
    const limit = req.user ? 450 : 180;

    if (!searchLimiter.allowRequest(identifier, limit)) {
        return res.status(429).json({error: 'Search rate limit exceeded'});
    }
    // Search
});
```

---

## Practice Questions

1. **Explain the token bucket algorithm**

Answer:
- Bucket holds tokens (like a coffee shop loyalty card)
- Each request costs 1 token
- Tokens refill at constant rate (e.g., 1/second)
- Bucket has max capacity (e.g., 10 tokens)
- Allows bursts: Can use all 10 tokens quickly
- After burst, must wait for refill
- Used by: Twitter (burst tweeting), Spotify (burst skips)

2. **What are the pros and cons of IP-based rate limiting?**

**Pros:**
- Works for public endpoints (no login required)
- Prevents bot scraping
- Simple to implement

**Cons:**
- Multiple users behind same IP (NAT, office network) → all share limit
- VPN users can bypass by changing IP
- Mobile users change IPs frequently

**Better approach:**
- IP-based for public endpoints
- User ID for authenticated endpoints
- API key for third-party developers

3. **Design a rate limiting system for Twitter's API**

See "Real-World Example: Twitter API Rate Limiting" section above.

---

## Real-World Examples

**GitHub API:**
- 5000 req/hour authenticated (User ID)
- 60 req/hour unauthenticated (IP Address)
- Algorithm: Sliding Window Counter
- Headers: X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset

**Stripe API:**
- 100 req/sec in test mode
- Algorithm: Token Bucket (allows bursts)
- Rate Limit By: API Key
- Exponential backoff on 429

**Cloudflare:**
- 10,000 req/sec per IP
- Algorithm: Leaky Bucket (constant rate)
- DDoS protection
- Configurable per zone

---

## Resources & References

- [NGINX Rate Limiting](https://www.nginx.com/blog/rate-limiting-nginx/)
- [Redis Rate Limiting](https://redis.io/commands/incr#pattern-rate-limiter)
- [express-rate-limit (Node.js)](https://www.npmjs.com/package/express-rate-limit)
- [Rate Limiting Strategies (Cloudflare)](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/)
- [Token Bucket Algorithm (Wikipedia)](https://en.wikipedia.org/wiki/Token_bucket)
