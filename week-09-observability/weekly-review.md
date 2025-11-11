# Week 9: Observability & Resilience - Weekly Review

## Review Questions

### Question 1: Observability Strategy

**Scenario:** You're building a food delivery app with these services: Order Service, Payment Service, Restaurant Service, Delivery Tracking Service. Design a complete observability strategy.

**What you need to cover:**
1. What metrics would you track for each service?
2. What would you log and at what levels?
3. Which traces would be most critical?
4. How would you correlate logs, metrics, and traces?

**Example answer structure:**
```
Metrics (RED method for each service):
- Order Service: orders/sec, error rate, p95 latency
- Payment Service: payment_success_rate, p99 latency
- ...

Logging strategy:
- CRITICAL/ERROR: Always on, centralized (ELK)
- INFO: Sample 10% in production
- DEBUG: Off in production, on-demand for troubleshooting

Distributed tracing:
- Trace critical paths: Order placement (API → Order → Payment → Restaurant)
- Sample: 100% errors, 100% slow (>2s), 5% normal traffic
- Use trace_id in all logs for correlation

Correlation:
- Include trace_id, order_id, user_id in all logs
- Click log → view trace
- Click trace → view related logs
- Grafana dashboards link to Jaeger traces
```

---

### Question 2: Metrics Design (RED/USE)

**Scenario:** You notice your Order Service p95 latency spiked from 300ms to 2,500ms at 6 PM. Walk through your investigation process.

**Questions to answer:**
1. What RED metrics would you check first?
2. What USE metrics would you check on your infrastructure?
3. How would you use distributed tracing to find the bottleneck?
4. What query would you run in Jaeger?

**Example investigation:**
```
Step 1: Check RED metrics
- Rate: Still 1,000 req/sec (normal)
- Errors: 0.2% (normal)
- Duration: p95 = 2,500ms (8x normal!) ❌

Step 2: Check USE metrics
- CPU: 45% (normal)
- Memory: 67% (normal)
- Database connections: 98/100 (near limit!) ⚠️

Step 3: Query Jaeger
Search: service="order-service" AND duration > 2s AND startTime > 6pm
Find: 45 slow traces, all have 2s spent in database query

Step 4: Root cause
Database connection pool exhausted (98/100 used)
Queries queuing, waiting for available connection

Step 5: Fix
Increase connection pool from 100 → 200
Monitor: p95 drops back to 320ms ✅
```

---

### Question 3: Distributed Tracing

**Scenario:** A user reports their order took 8 seconds to process. You have the trace_id. Walk through how you'd use distributed tracing to debug this.

**Questions to answer:**
1. What would you look for in the trace waterfall?
2. How do you identify the bottleneck?
3. What if a span is missing from the trace?
4. How would you correlate the trace with logs?

**Example debugging process:**
```
Step 1: Open Jaeger, search by trace_id
View waterfall visualization

Step 2: Analyze spans
Total: 8,234ms
├─ API Gateway: 23ms ✅
├─ Order Service: 45ms ✅
├─ Payment Service: 134ms ✅
└─ Fraud Detection: 8,012ms ❌
   └─ Database query: 7,987ms ❌ (BOTTLENECK FOUND!)

Step 3: Click span "Database query"
View span attributes:
- db.statement: "SELECT * FROM user_risk_scores WHERE ..."
- db.rows_examined: 45,000,000 ⚠️

Step 4: Correlate with logs
Click "View logs for this span"
Find: "SLOW QUERY: 7.9s, examined 45M rows, missing index"

Step 5: Fix
Add index on user_risk_scores(user_id)
Verify: Same query now 50ms ✅
```

---

### Question 4: SLO Definition

**Scenario:** Define SLIs, SLOs, and SLAs for your food delivery Order Service.

**Requirements:**
1. Choose appropriate SLIs to measure
2. Set realistic SLOs (with reasoning)
3. Define customer-facing SLA (with buffer from SLO)
4. Calculate error budget
5. Explain what happens when error budget is 90% consumed

**Example answer:**
```
SLIs (What we measure):
1. Availability = successful_orders / total_orders
2. Latency = p95_order_placement_duration
3. Error rate = failed_orders / total_orders

SLOs (Internal targets):
1. Availability: 99.9% over 30 days
   - Reasoning: Standard for consumer apps, achievable with multi-AZ
2. Latency: p95 < 500ms over 7 days
   - Reasoning: Users expect fast checkout
3. Error rate: < 0.5% over 24 hours
   - Reasoning: Minimizes customer complaints

SLA (Customer promise):
1. Availability: 99.5% over 30 days
   - Penalty: 10% credit if violated
   - Buffer: 0.4% safety margin between SLO (99.9%) and SLA (99.5%)

Error Budget Calculation:
30 days = 43,200 minutes
99.9% SLO → 0.1% allowed downtime = 43.2 minutes/month

When 90% consumed (38.9 minutes used):
Action: Deploy freeze until next month
Only critical bug fixes allowed
Focus on stability, not features
Preserve remaining 4.3 minutes for emergencies
```

---

### Question 5: Circuit Breaker Implementation

**Scenario:** Implement a circuit breaker for your Order Service → Payment Service calls. The Payment Service occasionally has 30-second timeouts.

**Requirements:**
1. Define the three states and transitions
2. Set appropriate thresholds
3. Implement fallback strategy
4. Explain how to avoid flapping
5. What metrics would you track?

**Example implementation:**
```javascript
const paymentCircuitBreaker = new CircuitBreaker({
  failureThreshold: 0.5,     // Trip at 50% error rate
  timeout: 60000,             // Wait 60s before retry (long enough for recovery)
  monitoringWindow: 20,       // Track last 20 requests
  successThreshold: 3         // Need 3 successes in HALF-OPEN before CLOSED
});

// States and transitions:
// CLOSED → (10+ failures in 20 requests) → OPEN
// OPEN → (wait 60s) → HALF-OPEN
// HALF-OPEN → (3 consecutive successes) → CLOSED
// HALF-OPEN → (1 failure) → OPEN

// Fallback strategy
async function processPayment(order) {
  try {
    return await paymentCircuitBreaker.execute(async () => {
      return await paymentService.charge(order);
    });
  } catch (error) {
    if (error.message === 'Circuit breaker is OPEN') {
      // Fallback: Queue for later processing
      await paymentQueue.add(order);
      return {
        status: 'PENDING',
        message: 'Payment processing, you'll receive confirmation shortly'
      };
    }
    throw error;
  }
}

// Avoid flapping:
// 1. Long timeout (60s) - Give service time to fully recover
// 2. Require 3 successes before closing - Don't close on first success
// 3. Singleton instance - Not per-request
// 4. Exponential backoff - Double timeout on repeated failures

// Metrics to track:
const circuitBreakerState = new prometheus.Gauge({
  name: 'circuit_breaker_state',
  help: '0=CLOSED, 1=OPEN, 2=HALF-OPEN',
  labelNames: ['service']
});

const circuitBreakerTrips = new prometheus.Counter({
  name: 'circuit_breaker_trips_total',
  help: 'Times circuit opened',
  labelNames: ['service']
});

// Alert when circuit opens
Alert: circuit_breaker_state{service="payment"} == 1
Action: Page on-call immediately
```

---

## Practice Exercise

**Design a complete observability and resilience strategy for this e-commerce system:**

### System Architecture:
```
API Gateway
├─ Product Service (catalog, search)
├─ Order Service (order placement)
│  ├─ Payment Service (charges credit cards)
│  ├─ Inventory Service (stock management)
│  └─ Notification Service (emails, SMS)
└─ User Service (authentication, profiles)
```

### Requirements:

**1. Metrics Strategy (30 points)**
- Define RED metrics for each service
- Define USE metrics for databases
- Design Grafana dashboard layout
- Set up alerting thresholds

**2. Logging Strategy (20 points)**
- What log levels for production?
- Structured logging format
- Cost estimation (1M requests/day)
- Centralized logging architecture

**3. Distributed Tracing (20 points)**
- Which flows to trace?
- Sampling strategy
- How to handle trace context propagation?
- Integration with logs

**4. SLOs & Error Budgets (15 points)**
- Define SLIs for Order Service
- Set SLO targets with reasoning
- Calculate error budget
- Error budget policy (when to freeze deploys)

**5. Circuit Breakers (15 points)**
- Where to place circuit breakers?
- Configuration (thresholds, timeouts)
- Fallback strategies
- How to monitor circuit breaker health?

### Example Solution Structure:

```markdown
## 1. Metrics Strategy

### RED Metrics Per Service:

**Order Service:**
- Rate: orders_per_second (track spikes during peak hours)
- Errors: order_failure_rate (alert if > 1%)
- Duration: order_placement_p95_latency (alert if > 500ms)

**Payment Service:**
- Rate: payment_attempts_per_second
- Errors: payment_failure_rate (alert if > 2% - includes legitimate declines)
- Duration: payment_processing_p99_latency (alert if > 1s)

[Continue for all services...]

### USE Metrics:

**Order Service Database:**
- Utilization:
  - CPU: 45% (alert > 80%)
  - Memory: 67% (alert > 85%)
  - Connections: 47/100 (alert > 90)
- Saturation:
  - Query queue depth (alert if > 10)
  - Lock wait time (alert if > 100ms avg)
- Errors:
  - Connection failures (alert > 5/min)
  - Query timeouts (alert > 10/min)

[Continue for all databases...]

### Grafana Dashboard:

**Top Row (Critical Health):**
```
[Overall Success Rate] [Total Orders/sec] [p95 Latency] [Active Alerts]
     99.94% ✅             1,247            320ms          2 ⚠️
```

**Service Health Row:**
```
[Product]  [Order]   [Payment]  [Inventory]  [Notification]
  99.9% ✅  99.8% ✅  99.7% ⚠️   99.95% ✅    99.1% ⚠️
```

**Detailed Metrics (6 panels):**
- Request rate graph (24 hours)
- Error rate graph (with breakdown by type)
- Latency percentiles (p50, p95, p99)
- Database connection pool usage
- Circuit breaker states
- Error budget burn rate

### Alerting Thresholds:

**Critical (Page on-call):**
- Error rate > 5% for 2 minutes
- p95 latency > 2s for 5 minutes
- Any service down (success rate < 50%)
- Circuit breaker opens for payment service

**Warning (Slack notification):**
- Error rate > 1% for 10 minutes
- p95 latency > 1s for 10 minutes
- Database connections > 90%
- Error budget burn rate > 2x

**Info (Email):**
- Traffic spike > 2x normal
- New deployment started
- Circuit breaker enters HALF-OPEN

## 2. Logging Strategy

### Production Log Levels:

**Always ON:**
- ERROR: All errors (500s, exceptions, failures)
- WARN: Degraded functionality, retries, circuit breaker opens

**Sampled (10%):**
- INFO: Request/response summary, business events

**OFF:**
- DEBUG: Detailed debugging, variable values
- TRACE: Function entry/exit

**Dynamic (On-demand):**
- Enable DEBUG for specific user_id when troubleshooting
- Enable TRACE for specific trace_id

### Structured Logging Format:

```json
{
  "timestamp": "2025-11-10T18:45:32.123Z",
  "level": "ERROR",
  "service": "order-service",
  "trace_id": "abc123",
  "span_id": "span456",
  "user_id": "user_789",
  "order_id": "order_12345",
  "message": "Payment processing failed",
  "error": {
    "type": "PaymentDeclined",
    "code": "INSUFFICIENT_FUNDS",
    "message": "Card declined by bank"
  },
  "context": {
    "amount": 47.99,
    "currency": "USD",
    "attempt": 1
  }
}
```

### Cost Estimation (1M requests/day):

**Assumptions:**
- 1M requests/day
- 5 logs per request average
- 500 bytes per log

**Calculation:**
- 1M × 5 = 5M logs/day
- 5M × 500 bytes = 2.5 GB/day
- 2.5 GB × 30 days = 75 GB/month
- Cost: 75 GB × $0.10/GB = $7.50/month (very affordable!)

**With 10% INFO sampling:**
- ERROR/WARN: 1M logs/day (always on)
- INFO: 4M × 10% = 400K logs/day
- Total: 1.4M logs/day = 21 GB/month = $2.10/month

### Centralized Logging Architecture:

```
Services → Fluentd Aggregator → Elasticsearch → Kibana
                                      ↓
                                  S3 Archive
                                  (7 day retention in ES,
                                   90 day retention in S3)
```

## 3. Distributed Tracing

### Critical Flows to Trace:

**1. Order Placement (100% of errors + slow):**
```
API Gateway → Order Service → Payment Service → Inventory Service
Sample: 100% errors, 100% if >2s, 5% otherwise
```

**2. Product Search (5% sampling):**
```
API Gateway → Product Service → Elasticsearch
Sample: 5% (high volume, less critical)
```

**3. User Authentication (10% sampling):**
```
API Gateway → User Service → Database
Sample: 10% + 100% failures
```

### Sampling Strategy:

```javascript
function shouldTrace(request, response, duration) {
  // Always trace errors
  if (response.statusCode >= 500) return true;

  // Always trace slow requests
  if (duration > 2000) return true;

  // Sample by endpoint criticality
  if (request.path.startsWith('/orders')) {
    return Math.random() < 0.05; // 5% of orders
  } else if (request.path.startsWith('/search')) {
    return Math.random() < 0.01; // 1% of searches
  }

  return Math.random() < 0.05; // 5% default
}
```

**Cost impact:**
- Without sampling: 1M req/day × 10 spans = 10M spans/day = $300/month
- With sampling: ~500K spans/day = $15/month

### Trace Context Propagation:

**HTTP Headers (W3C Trace Context standard):**
```javascript
// Incoming request
const traceParent = req.headers['traceparent'];
// Format: 00-{trace-id}-{parent-span-id}-{flags}

// Extract
const [version, traceId, parentSpanId, flags] = traceParent.split('-');

// Create new span for this service
const currentSpanId = generateSpanId();

// Outgoing request to downstream service
await fetch('https://payment-service/charge', {
  headers: {
    'traceparent': `00-${traceId}-${currentSpanId}-01`
  }
});
```

### Integration with Logs:

**Include trace_id in every log:**
```javascript
logger.info('Order created', {
  trace_id: getCurrentTraceId(),
  span_id: getCurrentSpanId(),
  order_id: order.id
});
```

**Bi-directional linking:**
- Jaeger span → "View logs" button → Kibana filtered by trace_id
- Kibana log → Click trace_id → Opens Jaeger trace
- Grafana alert → Link to Jaeger showing slow traces

## 4. SLOs & Error Budgets

### SLIs for Order Service:

**1. Availability:**
```
availability = successful_orders / total_order_attempts
Measurement: Count 200/201 responses vs 500/503 errors
```

**2. Latency:**
```
latency_p95 = 95th percentile of order_placement_duration
Measurement: Histogram from order start to completion
```

**3. Correctness:**
```
order_correctness = orders_with_correct_inventory / total_orders
Measurement: Compare order items vs inventory deductions
```

### SLO Targets:

**Availability SLO: 99.9% over 30 days**
- Reasoning: Industry standard for consumer apps
- Cost: ~$100K/year infrastructure
- User perception: "Always available"
- Downtime allowed: 43.2 minutes/month

**Latency SLO: p95 < 500ms over 7 days**
- Reasoning: Users expect fast checkout
- Shorter window (7 days) for faster feedback
- 500ms feels instant to users

**Correctness SLO: 99.99% over 30 days**
- Reasoning: Wrong inventory = customer complaints
- Stricter than availability (correctness matters more)

### Error Budget Calculation:

**Availability budget:**
```
30 days = 43,200 minutes
SLO: 99.9% → 0.1% downtime allowed
Error budget: 43.2 minutes/month

Current usage:
Week 1: 10 min (23%)
Week 2: 5 min (12%)
Week 3: 15 min (35%)
Total: 30 min used (69% consumed)
Remaining: 13.2 min (31% left) ✅ Healthy
```

**Latency budget:**
```
7 days of requests: ~10M requests
SLO: p95 < 500ms
Error budget: 5% of requests can be slow (500K requests)

Current: 3% slow (300K requests)
Budget used: 60%
Remaining: 40% ✅ Healthy
```

### Error Budget Policy:

**When budget 50% consumed:** Normal operations
- Deploy 10x/day
- Experiment with new features
- Take calculated risks

**When budget 75% consumed:** Cautious
- Slow down deploys to 3x/day
- Extra testing before production
- Focus on stability improvements

**When budget 90% consumed:** Deploy freeze
- **FREEZE** all feature deployments
- Only critical bug fixes
- Blameless post-mortem to identify root causes
- Focus entirely on reliability

**When budget 100% consumed:** SLO violated
- Emergency response
- Executive visibility
- Mandatory post-mortem
- Freeze continues until next period

## 5. Circuit Breakers

### Where to Place Circuit Breakers:

**Critical dependencies (must have):**
```
Order Service → Payment Service ✅
Order Service → Inventory Service ✅
Order Service → Notification Service ✅
```

**Less critical (optional):**
```
Order Service → User Service ⚠️
  Fallback: Use cached user data
Product Service → Recommendation Service ⚠️
  Fallback: Show popular products instead
```

**Don't use circuit breakers:**
```
Order Service → Database ❌
  Can't fallback on orders database
  Use connection pooling + timeouts instead
```

### Configuration:

**Payment Service Circuit Breaker:**
```javascript
const paymentCircuitBreaker = new CircuitBreaker({
  name: 'payment-service',

  // When to open
  failureThreshold: 0.5,      // 50% error rate
  monitoringWindow: 20,        // Last 20 requests
  minRequests: 5,              // Need at least 5 requests before tripping

  // When to retry
  timeout: 60000,              // Wait 60s before testing recovery

  // When to close
  successThreshold: 3,         // Need 3 consecutive successes

  // Fallback
  fallback: async (order) => {
    await paymentQueue.add(order);
    return {
      status: 'PENDING',
      message: 'Payment processing, confirmation coming shortly'
    };
  }
});
```

**Inventory Service Circuit Breaker:**
```javascript
const inventoryCircuitBreaker = new CircuitBreaker({
  name: 'inventory-service',
  failureThreshold: 0.3,       // More strict (30% error rate)
  timeout: 30000,              // Faster recovery (30s)
  fallback: async (order) => {
    // Optimistic: Assume inventory available
    // Background job reconciles later
    return { status: 'RESERVED', message: 'Item reserved' };
  }
});
```

### Fallback Strategies:

**Payment Service (Critical):**
```javascript
if (circuitBreakerOpen) {
  // Strategy: Queue for async processing
  await paymentQueue.add(order);

  // User sees graceful message
  return {
    status: 'PENDING',
    orderId: order.id,
    message: 'Your order is confirmed! Payment processing, receipt coming soon.'
  };
}
```

**Inventory Service (Important):**
```javascript
if (circuitBreakerOpen) {
  // Strategy: Optimistic with reconciliation
  // Most orders succeed, overselling is rare

  logWarning('Inventory service down, proceeding optimistically');

  // Background job later:
  // - Check actual inventory
  // - Refund if oversold
  // - Alert ops team

  return { status: 'RESERVED' };
}
```

**Notification Service (Nice-to-have):**
```javascript
if (circuitBreakerOpen) {
  // Strategy: Retry queue
  await notificationQueue.add(order);

  // User doesn't need to know
  // Email will arrive when service recovers

  return { status: 'QUEUED' };
}
```

### Monitoring Circuit Breakers:

**Prometheus Metrics:**
```javascript
// State gauge
const circuitBreakerState = new prometheus.Gauge({
  name: 'circuit_breaker_state',
  help: '0=CLOSED, 1=OPEN, 2=HALF_OPEN',
  labelNames: ['service']
});

// Trip counter
const circuitBreakerTrips = new prometheus.Counter({
  name: 'circuit_breaker_trips_total',
  help: 'Times circuit has opened',
  labelNames: ['service']
});

// Request counter by result
const circuitBreakerRequests = new prometheus.Counter({
  name: 'circuit_breaker_requests_total',
  help: 'Requests through circuit breaker',
  labelNames: ['service', 'result'] // success, failure, rejected
});
```

**Grafana Dashboard:**
```
┌───────────────────────────────────────┐
│ Circuit Breaker Status                │
├───────────────────────────────────────┤
│ Payment Service:    🟢 CLOSED         │
│   Success rate: 99.8%                 │
│   Last trip: 2 days ago               │
│                                        │
│ Inventory Service:  🔴 OPEN           │
│   Failed: 15/20 requests (75%)        │
│   Opens again in: 45 seconds          │
│   Trips today: 3 ⚠️                   │
│                                        │
│ Notification:       🟡 HALF-OPEN      │
│   Testing recovery...                 │
│   Last success: 10 seconds ago        │
└───────────────────────────────────────┘
```

**Alerts:**
```yaml
# Critical: Circuit opened for payment
- alert: PaymentCircuitBreakerOpen
  expr: circuit_breaker_state{service="payment"} == 1
  for: 1m
  annotations:
    summary: "Payment circuit breaker OPEN"
    description: "Orders are being queued, payment service degraded"
  labels:
    severity: critical

# Warning: Circuit flapping (opening/closing repeatedly)
- alert: CircuitBreakerFlapping
  expr: rate(circuit_breaker_trips_total[10m]) > 3
  for: 5m
  annotations:
    summary: "Circuit breaker flapping for {{ $labels.service }}"
    description: "Tripped {{ $value }} times in 10 minutes"
  labels:
    severity: warning
```
```

---

## Resources & References

### Week 9 Core Materials:
- [Logging](logging.md) - Structured logging, log levels, centralized logging
- [Metrics](metrics.md) - RED/USE methods, Prometheus, Grafana
- [Tracing](tracing.md) - Distributed tracing, OpenTelemetry, Jaeger
- [SLOs/SLAs](slos-slas.md) - Service level objectives, error budgets
- [Circuit Breaker](circuit-breaker.md) - Resilience patterns, preventing cascading failures
- [Chaos Engineering](chaos-engineering.md) - Netflix Chaos Monkey, Game Days

### Industry Resources:
- **Google SRE Books:**
  - [Site Reliability Engineering](https://sre.google/sre-book/table-of-contents/)
  - [The Site Reliability Workbook](https://sre.google/workbook/table-of-contents/)

- **Observability:**
  - [Prometheus Documentation](https://prometheus.io/docs/)
  - [OpenTelemetry Specification](https://opentelemetry.io/docs/)
  - [Jaeger Documentation](https://www.jaegertracing.io/docs/)

- **Chaos Engineering:**
  - [Principles of Chaos Engineering](https://principlesofchaos.org/)
  - [Netflix Chaos Monkey](https://netflix.github.io/chaosmonkey/)

- **Circuit Breakers:**
  - [Martin Fowler: Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html)
  - [Release It! by Michael Nygard](https://pragprog.com/titles/mnee2/release-it-second-edition/)

### Tools:
- **Metrics**: Prometheus, Grafana, Datadog, New Relic
- **Logging**: ELK Stack, Splunk, Datadog
- **Tracing**: Jaeger, Zipkin, AWS X-Ray, Datadog APM
- **Chaos**: Chaos Monkey, Gremlin, Litmus Chaos, AWS FIS
- **Circuit Breakers**: Hystrix, Resilience4j, Polly (.NET)
