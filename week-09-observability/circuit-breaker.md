# Circuit Breaker Pattern

## Learning Objectives
- Understand circuit breaker states (closed, open, half-open)
- Learn failure thresholds and timeout strategies
- Master resilience patterns (retry, bulkhead)

## Notes

### The Problem: Cascading Failures

**Scenario:** Payment Service is down (database crashed)

**Without Circuit Breaker:**
```
1. Order Service calls Payment Service
2. Payment Service times out after 30 seconds
3. Order Service retries (another 30 seconds)
4. 60 seconds wasted per order attempt
5. 1,000 req/sec × 60 seconds = 60,000 threads blocked
6. Order Service runs out of threads → crashes too!
7. API Gateway calls Order Service → timeout → API Gateway crashes!
8. Entire system down because ONE service failed
```

**This is cascading failure** - one failure causes domino effect.

### Circuit Breaker Pattern (The Solution)

**Like an electrical circuit breaker in your house:**
- When electricity overloads → breaker trips → stops flow
- Prevents fire from electrical overload
- Must manually reset after fixing problem

**Same concept for services:**
- When service fails too much → circuit breaker trips → stops calls
- Prevents cascading failure
- Automatically retries to check if service recovered

### Three States

```
CLOSED (Normal operation)
   ↓ (Too many failures)
OPEN (Stop calling, fail fast)
   ↓ (After timeout, try one request)
HALF-OPEN (Testing if service recovered)
   ↓ (Success: reset) or (Failure: back to OPEN)
CLOSED
```

#### 1. CLOSED State (Normal)

**All requests pass through:**
```javascript
try {
  const response = await paymentService.charge(order);
  circuitBreaker.recordSuccess();
  return response;
} catch (error) {
  circuitBreaker.recordFailure();
  throw error;
}
```

**Tracks failures:**
- Failure threshold: 50% error rate in last 10 requests
- If threshold exceeded → Trip to OPEN

#### 2. OPEN State (Service Down)

**Immediately fail without calling service:**
```javascript
if (circuitBreaker.isOpen()) {
  throw new Error('Payment Service unavailable (circuit breaker OPEN)');
  // No call to Payment Service!
  // Fail in 1ms instead of 30 second timeout
}
```

**Benefits:**
- ✅ Fail fast (1ms vs 30s timeout)
- ✅ No wasted threads waiting
- ✅ Order Service stays healthy
- ✅ Prevents cascading failure

**Timeout before retrying:**
- Wait 30 seconds before trying again
- After 30 seconds → Move to HALF-OPEN

#### 3. HALF-OPEN State (Testing Recovery)

**Allow ONE test request:**
```javascript
if (circuitBreaker.isHalfOpen()) {
  try {
    const response = await paymentService.charge(order);
    circuitBreaker.reset(); // Success! → CLOSED
    return response;
  } catch (error) {
    circuitBreaker.tripAgain(); // Still failing → OPEN
    throw error;
  }
}
```

**Success → CLOSED:** Service recovered, resume normal operation
**Failure → OPEN:** Still broken, wait another 30 seconds

### Implementation Example

```javascript
class CircuitBreaker {
  constructor(options) {
    this.failureThreshold = options.failureThreshold || 0.5; // 50%
    this.timeout = options.timeout || 30000; // 30 seconds
    this.monitoringWindow = options.monitoringWindow || 10; // last 10 requests

    this.state = 'CLOSED';
    this.failures = 0;
    this.successes = 0;
    this.nextAttemptTime = null;
  }

  async execute(fn) {
    // OPEN state: Fail fast
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttemptTime) {
        throw new Error('Circuit breaker is OPEN');
      }
      // Time to try again
      this.state = 'HALF-OPEN';
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  onSuccess() {
    this.failures = 0;
    if (this.state === 'HALF-OPEN') {
      this.state = 'CLOSED'; // Service recovered!
    }
  }

  onFailure() {
    this.failures++;
    const totalRequests = this.failures + this.successes;

    if (totalRequests >= this.monitoringWindow) {
      const errorRate = this.failures / totalRequests;

      if (errorRate >= this.failureThreshold) {
        this.state = 'OPEN';
        this.nextAttemptTime = Date.now() + this.timeout;
      }
    }

    if (this.state === 'HALF-OPEN') {
      this.state = 'OPEN'; // Still failing
      this.nextAttemptTime = Date.now() + this.timeout;
    }
  }
}

// Usage
const paymentCircuitBreaker = new CircuitBreaker({
  failureThreshold: 0.5,  // Trip at 50% error rate
  timeout: 30000,          // Wait 30s before retry
  monitoringWindow: 10     // Track last 10 requests
});

async function processPayment(order) {
  try {
    return await paymentCircuitBreaker.execute(async () => {
      return await paymentService.charge(order);
    });
  } catch (error) {
    if (error.message === 'Circuit breaker is OPEN') {
      // Return fallback response
      return { status: 'PENDING', message: 'Payment processing delayed' };
    }
    throw error;
  }
}
```

### Fallback Strategies

**When circuit breaker is OPEN, what do you return?**

**1. Cached response (for reads)**
```javascript
if (circuitBreakerOpen) {
  return cachedUserProfile; // Stale data better than no data
}
```

**2. Default value**
```javascript
if (circuitBreakerOpen) {
  return { recommendations: [] }; // Empty list instead of error
}
```

**3. Degraded functionality**
```javascript
if (circuitBreakerOpen) {
  return {
    status: 'PENDING',
    message: 'Order placed, payment will process when service recovers'
  };
}
```

**4. Queue for later**
```javascript
if (circuitBreakerOpen) {
  await queue.add(order); // Process when service recovers
  return { status: 'QUEUED' };
}
```

### Configuration Best Practices

**Failure threshold:**
```
Too low (10%): Circuit trips too often (flaky)
Too high (90%): Doesn't protect against failures
Sweet spot: 50% error rate
```

**Timeout:**
```
Too short (5s): Doesn't give service time to recover
Too long (5 min): Users wait too long
Sweet spot: 30-60 seconds
```

**Monitoring window:**
```
Too small (3 requests): One failure trips circuit (too sensitive)
Too large (1000 requests): Takes too long to detect failure
Sweet spot: 10-50 requests
```

### Common Mistake: Per-Request Circuit Breaker

**❌ WRONG: Circuit breaker per request**
```javascript
// BAD: Creates new circuit breaker for each request!
app.post('/order', async (req, res) => {
  const cb = new CircuitBreaker(); // ❌ New instance!
  await cb.execute(() => paymentService.charge(req.body));
});
```

**Problem:**
- Each request has its own circuit breaker
- No shared state = can't track failures across requests
- Circuit never opens!

**✅ CORRECT: Singleton circuit breaker**
```javascript
// GOOD: One circuit breaker instance shared across all requests
const paymentCircuitBreaker = new CircuitBreaker();

app.post('/order', async (req, res) => {
  await paymentCircuitBreaker.execute(() => paymentService.charge(req.body));
});
```

**This is why your circuit breaker was "flapping" (opening and closing rapidly)!**

### Circuit Breaker Flapping

**Problem:** Circuit keeps opening and closing rapidly

**Scenario:**
```
Request 1: Fail → Circuit OPEN
[30 seconds pass]
Request 2: Test (HALF-OPEN) → Success → Circuit CLOSED
Request 3: Fail → Circuit OPEN again
[30 seconds pass]
Request 4: Test → Success → CLOSED
...endless cycle
```

**Causes:**
1. **Timeout too short:** Service needs 60s to recover, you're testing every 30s
2. **Intermittent failures:** Service has 50% error rate (not fully down)
3. **HALF-OPEN allows too few tests:** One success isn't enough to confirm recovery

**Solutions:**

**1. Longer timeout:**
```javascript
timeout: 60000 // Give service more time to fully recover
```

**2. Multiple successful requests before closing:**
```javascript
// Require 3 consecutive successes in HALF-OPEN before → CLOSED
if (this.state === 'HALF-OPEN') {
  this.halfOpenSuccesses++;
  if (this.halfOpenSuccesses >= 3) {
    this.state = 'CLOSED';
    this.halfOpenSuccesses = 0;
  }
}
```

**3. Exponential backoff:**
```javascript
// Double timeout after each failure
this.nextAttemptTime = Date.now() + (this.timeout * Math.pow(2, this.consecutiveTrips));
// First trip: 30s, Second: 60s, Third: 120s, ...
```

### Monitoring Circuit Breakers

**Key metrics to track:**

```javascript
// Prometheus metrics
const circuitBreakerState = new prometheus.Gauge({
  name: 'circuit_breaker_state',
  help: 'Circuit breaker state (0=CLOSED, 1=OPEN, 2=HALF-OPEN)',
  labelNames: ['service']
});

const circuitBreakerTrips = new prometheus.Counter({
  name: 'circuit_breaker_trips_total',
  help: 'Total times circuit breaker tripped',
  labelNames: ['service']
});

// Alert when circuit opens
if (this.state === 'OPEN') {
  circuitBreakerTrips.inc({ service: 'payment-service' });
  logger.error('Circuit breaker OPEN for payment-service');
  alerting.page('on-call', 'Payment service circuit breaker tripped');
}
```

**Dashboard:**
```
┌────────────────────────────────────┐
│ Circuit Breaker Status             │
├────────────────────────────────────┤
│ Payment Service:    🟢 CLOSED      │
│ Inventory Service:  🔴 OPEN        │
│ Email Service:      🟡 HALF-OPEN   │
│                                     │
│ Trips (last 24h):                  │
│ Payment: 2                          │
│ Inventory: 15 ⚠️                   │
│ Email: 1                            │
└────────────────────────────────────┘
```

### Circuit Breaker + Retry Pattern

**Often used together:**

```javascript
async function callServiceWithRetry(service, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await circuitBreaker.execute(() => service.call());
    } catch (error) {
      if (error.message === 'Circuit breaker is OPEN') {
        throw error; // Don't retry if circuit is open
      }

      if (attempt === maxRetries) {
        throw error; // Last attempt failed
      }

      // Exponential backoff: 1s, 2s, 4s
      await sleep(1000 * Math.pow(2, attempt - 1));
    }
  }
}
```

**Strategy:**
1. Try request
2. If transient error (timeout, 503) → Retry with backoff
3. If circuit breaker OPEN → Don't retry, fail fast
4. If persistent error (400, 404) → Don't retry

### Other Resilience Patterns

**1. Bulkhead Pattern**
- Isolate resources (separate thread pools per service)
- If Payment Service slow, doesn't affect Inventory Service threads

**2. Timeout Pattern**
- Set max wait time (don't wait forever)
- `fetch(url, { timeout: 5000 })`

**3. Retry Pattern**
- Retry transient failures
- Exponential backoff

**4. Rate Limiting**
- Don't overwhelm recovering service
- Only send 10 req/sec even if circuit closed

## Practice Questions

1. **Why does a circuit breaker prevent cascading failures?**
   - Fails fast (1ms) instead of waiting for timeout (30s)
   - Doesn't waste threads waiting for failing service
   - Keeps calling service healthy
   - Prevents domino effect (Service A down → B down → C down)

2. **What are the three states and when does each transition?**
   - **CLOSED**: Normal operation, tracks failures
   - **OPEN**: Too many failures (>50% error rate), fail fast for 30s
   - **HALF-OPEN**: After timeout, test one request to check recovery
   - Success in HALF-OPEN → CLOSED, Failure → back to OPEN

3. **What causes circuit breaker flapping?**
   - Timeout too short (service needs more recovery time)
   - Per-request circuit breaker instead of singleton
   - Testing recovery with only 1 request (need 3+ successes)
   - Intermittent failures (50% error rate)

4. **What should you return when circuit breaker is OPEN?**
   - Cached/stale data (for reads)
   - Default values (empty list, fallback recommendations)
   - Degraded functionality (queue for later, "pending" status)
   - Never return error to user if possible
   - Graceful degradation > complete failure

## Real-World Examples

- **Netflix**: Hystrix library (circuit breakers for every dependency)
- **AWS**: Application Load Balancer has built-in circuit breaking
- **Uber**: Circuit breakers on all service-to-service calls
- **Amazon**: Prevents Prime Day crashes by isolating failing services
- **Twitter**: Used to prevent cascading failures during traffic spikes

## Resources & References

- Netflix Hystrix Documentation
- Release It! (Michael Nygard) - Circuit Breaker Pattern
- Martin Fowler: Circuit Breaker Pattern
- AWS Well-Architected: Reliability Patterns
- Resilience4j (Java circuit breaker library)

## Practice Questions

## Real-World Examples

## Resources & References
