# Distributed Tracing

## Learning Objectives
- Understand distributed tracing concepts
- Learn OpenTelemetry and Jaeger
- Master trace context propagation

## Notes

### The Problem Distributed Tracing Solves

**Scenario:** User reports "Order creation is slow" (taking 8 seconds)

**Without tracing:** You have to manually investigate
```
1. Check API Gateway logs → "Request took 8,234ms" (doesn't say why)
2. SSH to Order Service → Check logs → "Called Payment Service"
3. SSH to Payment Service → Check logs → "Called Fraud Detection"
4. SSH to Fraud Detection → Check logs → "Database query slow"
5. Total time wasted: 45 minutes of detective work
```

**With distributed tracing:** One dashboard shows you instantly
```
Total: 8,234ms
├─ API Gateway: 23ms
├─ Order Service: 45ms
├─ Payment Service: 134ms
└─ Fraud Detection: 8,012ms ❌ (Found the culprit!)
   └─ Database Query: 7,987ms ❌ (Exact problem!)
```

**You found the issue in 30 seconds instead of 45 minutes!**

### Core Concepts

#### 1. Trace
**A trace = One complete request journey across all services**

```
User clicks "Place Order"
  ↓
Trace ID: abc123 (follows request everywhere)
  ↓
API Gateway → Order Service → Payment → Inventory → Database
  ↓
Response back to user
```

**Each trace has unique ID** (trace_id) that follows request through entire system.

#### 2. Span
**A span = One operation within a trace**

```
Trace: "Place Order" (abc123)
├─ Span 1: API Gateway receives request (50ms)
├─ Span 2: Order Service processes (120ms)
│  ├─ Span 2a: Validate order data (20ms)
│  └─ Span 2b: Call Payment Service (80ms)
├─ Span 3: Payment Service charges card (200ms)
│  └─ Span 3a: Database insert payment (150ms)
└─ Span 4: Send confirmation email (100ms)
```

**Each span tracks:**
- Operation name (`POST /orders`)
- Start time and duration
- Parent span (nested structure)
- Metadata (HTTP status, user_id, error info)

#### 3. Context Propagation
**The magic:** How trace_id follows request across services

**HTTP Headers:**
```javascript
// Order Service receives request
const traceId = req.headers['x-trace-id'] || generateTraceId();
const spanId = req.headers['x-span-id'];
const parentSpanId = req.headers['x-parent-span-id'];

// Order Service calls Payment Service
await fetch('https://payment-service/charge', {
  headers: {
    'x-trace-id': traceId,        // Same trace!
    'x-span-id': newSpanId,        // New span
    'x-parent-span-id': spanId     // Current span is parent
  }
});
```

**Why it matters:**
- Without propagation: Each service thinks it's a separate request
- With propagation: All services know they're part of same user request

### Waterfall Visualization

**What you see in Jaeger/Zipkin:**

```
Trace ID: abc123 (Total: 8,234ms)

API Gateway         |████| 23ms
Order Service          |██████| 45ms
  Validate             |██| 12ms
  Payment Service         |████████████| 134ms
    Fraud Detection          |███████████████████████████████| 8,012ms
      Database Query           |██████████████████████████████| 7,987ms
  Inventory Check                                                   |███| 20ms

0ms    1s    2s    3s    4s    5s    6s    7s    8s
```

**Instantly see:**
- Fraud Detection is the bottleneck (8s)
- Specifically: Database query inside Fraud Detection
- Most services are fast (<200ms)

### Implementing Distributed Tracing

#### OpenTelemetry (Industry Standard)

**Instrumentation code:**

```javascript
const { trace } = require('@opentelemetry/api');
const tracer = trace.getTracer('order-service');

async function createOrder(orderData, traceContext) {
  // Start a span for this operation
  const span = tracer.startSpan('create_order', {
    attributes: {
      'order.id': orderData.orderId,
      'user.id': orderData.userId,
      'order.total': orderData.totalAmount
    }
  });

  try {
    // Validate order
    const validateSpan = tracer.startSpan('validate_order', { parent: span });
    await validateOrder(orderData);
    validateSpan.end();

    // Call payment service (propagate context)
    const paymentSpan = tracer.startSpan('call_payment_service', { parent: span });
    const payment = await fetch('https://payment-service/charge', {
      headers: {
        // OpenTelemetry automatically injects trace context
      },
      body: JSON.stringify(orderData)
    });
    paymentSpan.end();

    // Save to database
    const dbSpan = tracer.startSpan('database_insert', { parent: span });
    await db.createOrder(orderData);
    dbSpan.setAttribute('db.statement', 'INSERT INTO orders...');
    dbSpan.end();

    span.setStatus({ code: 'OK' });
    return order;

  } catch (error) {
    span.setStatus({ code: 'ERROR', message: error.message });
    span.recordException(error);
    throw error;
  } finally {
    span.end();
  }
}
```

#### What Gets Sent to Jaeger

```json
{
  "traceId": "abc123",
  "spanId": "span_456",
  "parentSpanId": "span_123",
  "operationName": "create_order",
  "startTime": 1699564800000,
  "duration": 234,
  "tags": {
    "order.id": "order_789",
    "user.id": "user_456",
    "http.status_code": 200,
    "service.name": "order-service"
  },
  "logs": [
    {
      "timestamp": 1699564800100,
      "event": "payment_started"
    }
  ]
}
```

### Sampling Strategy

**Problem:** Tracing every request = expensive
- 1,000 req/sec × 10 spans each = 10,000 spans/sec
- 864 million spans/day
- Storage cost: $10K+/month

**Solution: Sampling**

**1. Probabilistic Sampling (Simple)**
```javascript
// Trace 1% of requests randomly
const shouldTrace = Math.random() < 0.01;
```
- ✅ Easy, predictable cost
- ❌ Might miss rare errors

**2. Rate Limiting Sampling**
```javascript
// Trace max 100 requests/second
if (tracesThisSecond < 100) {
  shouldTrace = true;
}
```
- ✅ Controlled cost
- ❌ During traffic spike, drops traces

**3. Intelligent Sampling (Best)**
```javascript
// Always trace:
// - Errors (100%)
// - Slow requests (>1s) (100%)
// - Fast successful requests (1%)

if (error || duration > 1000) {
  shouldTrace = true;
} else {
  shouldTrace = Math.random() < 0.01;
}
```
- ✅ Captures all problems
- ✅ Samples normal traffic
- ✅ Minimal cost

**Production sampling:**
- Errors: 100%
- Slow requests (>1s): 100%
- Normal requests: 1-5%

### Finding Issues with Traces

#### Use Case 1: High Latency

**Query Jaeger:** "Show traces where duration > 1 second"

**Find:**
```
50 slow traces → All have same pattern:
└─ Database query in Fraud Detection takes 8-12 seconds
```

**Action:**
1. Check Fraud Detection database
2. Find missing index on `user_risk_score` column
3. Add index
4. Latency drops to 50ms ✅

#### Use Case 2: Cascading Failures

**Scenario:** 50% of orders failing

**Trace shows:**
```
API Gateway (200ms)
└─ Order Service (error!)
   └─ Payment Service (timeout after 30s)
      └─ Fraud Detection (no response)
```

**Root cause:** Fraud Detection is down, causing everything upstream to fail.

**Action:** Circuit breaker on Payment → Fraud Detection

#### Use Case 3: Unexpected Dependencies

**Discover:** Order creation calls 15 services (should be 3)

**Trace reveals:**
```
Order Service
├─ Payment ✅
├─ Inventory ✅
├─ Notification ✅ (expected)
└─ Recommendation Service ❌ (adds 500ms, not needed!)
   └─ ML Model Service ❌ (adds 1,200ms)
      └─ Feature Store ❌ (adds 800ms)
```

**Action:** Remove recommendation call from order flow (move to async)

### Architecture

```
┌─────────────────────────────────────────┐
│   Microservices (instrumented)         │
│   - API Gateway                         │
│   - Order Service                       │
│   - Payment Service                     │
└──────────────┬──────────────────────────┘
               │
               │ Send spans
               ↓
┌─────────────────────────────────────────┐
│   OpenTelemetry Collector               │
│   - Receives spans from all services    │
│   - Batches and processes               │
│   - Sampling decisions                  │
└──────────────┬──────────────────────────┘
               │
               ↓
┌─────────────────────────────────────────┐
│   Jaeger Backend                        │
│   - Cassandra/Elasticsearch storage     │
│   - Indexes traces for fast search      │
└──────────────┬──────────────────────────┘
               │
               ↓
┌─────────────────────────────────────────┐
│   Jaeger UI                             │
│   - Waterfall visualization             │
│   - Search traces                       │
│   - Service dependency graph            │
└─────────────────────────────────────────┘
```

**Why OpenTelemetry Collector in middle?**
- Decouples services from backend (can switch Jaeger → Zipkin)
- Batching reduces network overhead
- Central sampling decisions
- Can enrich spans with metadata

### Service Dependency Graph

**Auto-generated from traces:**

```
          API Gateway
               │
        ┌──────┴──────┐
        │             │
   Order Service   Search Service
        │             │
    ┌───┴───┐     ┌───┴───┐
    │       │     │       │
Payment  Inventory  Recommendation
    │       │
Fraud DB  Warehouse DB
```

**Shows:**
- Which services talk to each other
- Request volume (arrow thickness)
- Error rates (red arrows)
- Latency (arrow color gradient)

**Use cases:**
- Understand system architecture (even without docs!)
- Find unexpected dependencies
- Identify critical path services

### Correlation with Logs & Metrics

**The power of trace_id everywhere:**

**1. From trace → logs**
```
Trace shows: Fraud Detection slow (8s)
Click span → "View logs for this span"
→ Shows exact database error log
```

**2. From logs → trace**
```
Log: "ERROR: Payment failed, order_id=123, trace_id=abc123"
Click trace_id → See full request journey
```

**3. From metrics → traces**
```
Grafana: "p99 latency spiked to 5s at 2:15 PM"
→ Query Jaeger: Show traces from 2:15 PM where duration > 5s
→ Find common pattern causing slowness
```

**This correlation = Observability superpowers!**

### Best Practices

**✅ DO:**
- Trace critical user paths (checkout, login, search)
- Add business context to spans (order_id, user_id, amount)
- Always trace errors (100% sampling)
- Use intelligent sampling (errors + slow + 1% normal)
- Set up alerts on trace patterns

**❌ DON'T:**
- Trace 100% of traffic (too expensive)
- Add PII to span attributes (credit cards, passwords)
- Create spans for every function (too granular, high overhead)
- Forget to propagate context (breaks trace)
- Use tracing for debugging individual requests (use logs)

**Span granularity:**
- ✅ Network calls (HTTP, gRPC, database)
- ✅ Major business logic (validate, process payment)
- ❌ Individual functions (too much overhead)
- ❌ Loops (creates thousands of spans)

### Common Issues

**1. Missing spans (broken trace)**
- **Cause:** Service didn't propagate trace context
- **Fix:** Check HTTP headers are passed correctly

**2. Orphaned spans**
- **Cause:** Parent span ended before child
- **Fix:** Ensure proper span lifecycle (await child before ending parent)

**3. High overhead**
- **Cause:** Too many spans or 100% sampling
- **Fix:** Reduce span granularity, lower sampling rate

**4. Storage explosion**
- **Cause:** Tracing high volume without sampling
- **Fix:** Implement intelligent sampling (1-5% normal traffic)

## Practice Questions

1. **What's the difference between a trace and a span?**
   - **Trace**: Entire request journey across all services (trace_id)
   - **Span**: Single operation within a trace (one service call or database query)
   - Trace contains multiple spans in parent-child hierarchy
   - Example: Trace = "Order flow", Spans = API call, DB query, payment processing

2. **How does trace context propagation work?**
   - trace_id passed via HTTP headers (x-trace-id, x-span-id)
   - Each service extracts trace_id from incoming request
   - Each service passes same trace_id to downstream services
   - Creates continuous chain across all services
   - Without propagation, each service thinks it's separate request

3. **Why not trace 100% of requests?**
   - Cost: 1,000 req/sec = 864M spans/day = $10K+/month
   - Storage: Traces much larger than metrics
   - Performance overhead: Creating spans costs CPU
   - Solution: Intelligent sampling (100% errors, 1-5% normal)

4. **How do you find a bottleneck using traces?**
   - Query slow traces (duration > threshold)
   - Look at waterfall visualization
   - Identify longest span (biggest bar)
   - Drill down into child spans if needed
   - Example: 8s trace → 7.9s in one database query = bottleneck found

## Real-World Examples

- **Uber**: Jaeger (they created it), traces rider/driver matching flow
- **Netflix**: Traces video streaming request path across 50+ microservices
- **Google**: Dapper (invented distributed tracing), inspired Jaeger/Zipkin
- **Twitter**: Zipkin, reduced latency by 40% after finding bottlenecks via traces
- **Lyft**: Envoy + OpenTelemetry, automatic trace instrumentation

## Resources & References

- OpenTelemetry Documentation
- Jaeger Architecture
- Google Dapper Paper (2010) - Original distributed tracing
- Zipkin Documentation
- OpenTelemetry Best Practices

## Practice Questions

## Real-World Examples

## Resources & References
