# Metrics (RED/USE)

## Learning Objectives
- Understand RED (Rate, Errors, Duration) metrics
- Learn USE (Utilization, Saturation, Errors) metrics
- Master Prometheus and Grafana

## Notes

### What Are Metrics?

**Metrics = Numbers that tell you system health**
- Unlike logs (individual events), metrics are **aggregated numbers**
- Cheap to store (much smaller than logs)
- Easy to graph and alert on

**Example:**
- **Log**: "Order order_123 took 450ms" (one event)
- **Metric**: "Average order latency: 320ms" (aggregated from thousands)

### Why Metrics Over Logs for Volume?

**Cost comparison:**
- **Logs**: 1,000 requests/sec × 500 bytes = 13 billion logs/month = $3,250/month
- **Metrics**: Same data = 1 data point every 10 seconds = 260K points/month = $5/month

**That's 650x cheaper!**

### The Two Key Frameworks

#### RED Method (For Request-Based Services)

**Use for:** APIs, microservices, web servers

**R = Rate** (requests per second)
```
orders_per_second = 1,247
search_requests_per_second = 8,934
```

**E = Errors** (error rate %)
```
error_rate = errors / total_requests
Example: 50 errors / 10,000 requests = 0.5% error rate
```

**D = Duration** (latency)
```
p50 = 120ms (median - half of requests faster)
p95 = 450ms (95% faster than this)
p99 = 1,200ms (99% faster than this)
```

**Why track all three?**
- **Rate drops** = Users can't reach service (might be crashed)
- **Errors spike** = Something breaking (database timeout, bad deployment)
- **Duration increases** = System slowing down (overload, database issues)

#### USE Method (For Infrastructure/Resources)

**Use for:** Databases, servers, queues, caches

**U = Utilization** (% of capacity used)
```
cpu_utilization = 45%
memory_utilization = 78%
disk_utilization = 92% ⚠️ (dangerous!)
```

**S = Saturation** (queue depth / work waiting)
```
database_connection_pool_waiting = 23 connections
redis_queue_depth = 1,247 jobs
```

**E = Errors**
```
disk_read_errors = 5
database_connection_failures = 12
```

**Key insight:** High utilization + high saturation = about to fail!

### Metric Types

**1. Counter** - Only goes up (resets on restart)
```javascript
// Total orders processed since startup
orders_total = 1,247,923

// Use case: Track totals, calculate rate of change
rate(orders_total[5m]) // Orders per second over last 5 minutes
```

**2. Gauge** - Goes up and down (current value)
```javascript
// Current active users online right now
active_users = 8,472

// Current CPU usage
cpu_percent = 67%

// Use case: Snapshot of current state
```

**3. Histogram** - Buckets of values (for percentiles)
```javascript
// Response time buckets
response_time_ms {
  le="100ms": 5,432 requests  // 5,432 requests took ≤100ms
  le="500ms": 9,234 requests  // 9,234 requests took ≤500ms
  le="1000ms": 9,987 requests // 9,987 requests took ≤1000ms
}

// Automatically calculates: p50, p95, p99
```

**4. Summary** - Similar to histogram, pre-calculates percentiles
```
// Already calculated percentiles
response_time_ms {
  quantile="0.5": 120ms   // p50
  quantile="0.95": 450ms  // p95
  quantile="0.99": 1200ms // p99
}
```

**Histogram vs Summary:**
- **Histogram**: Calculate percentiles server-side (more flexible, can aggregate)
- **Summary**: Calculate percentiles client-side (less storage, can't aggregate)
- **Use histogram** unless you have a specific reason not to

### Why p99 Matters More Than Average

**Scenario:** API latency
- **Average**: 200ms ✅ Looks great!
- **p50 (median)**: 150ms ✅ Most users happy
- **p95**: 800ms ⚠️ 5% of users frustrated
- **p99**: 3,000ms ❌ 1% of users experiencing 3 second delays

**Real-world impact:**
- 1% of 1 million users = **10,000 angry users**
- They complain on Twitter, leave bad reviews
- Average doesn't show this pain!

**Production rule:**
- ✅ Monitor p95 and p99
- ❌ Don't just monitor average

### Prometheus Architecture

**How it works:**

```
┌─────────────────────────┐
│   Your Application      │
│   (exposes /metrics)    │
└──────────┬──────────────┘
           │
           │ Prometheus scrapes every 15s
           ↓
┌─────────────────────────┐
│   Prometheus Server     │
│   - Stores time-series  │
│   - Evaluates alerts    │
└──────────┬──────────────┘
           │
           │ Grafana queries
           ↓
┌─────────────────────────┐
│   Grafana Dashboard     │
│   - Pretty graphs       │
│   - Team visibility     │
└─────────────────────────┘
```

**Pull vs Push model:**
- **Prometheus pulls** (scrapes) metrics from your app
- Your app exposes `/metrics` endpoint
- Prometheus decides when to collect (every 15s)

**Why pull?**
- ✅ Service discovery (Prometheus finds new instances automatically)
- ✅ Monitoring system controls load
- ✅ Can tell if service is down (failed scrape)

### Instrumenting Your Code

**Example: Tracking order creation**

```javascript
const prometheus = require('prom-client');

// Counter for total orders
const ordersTotal = new prometheus.Counter({
  name: 'orders_total',
  help: 'Total orders created',
  labelNames: ['status'] // success or failed
});

// Histogram for order latency
const orderDuration = new prometheus.Histogram({
  name: 'order_duration_ms',
  help: 'Order creation duration',
  buckets: [100, 500, 1000, 2000, 5000] // milliseconds
});

async function createOrder(orderData) {
  const start = Date.now();

  try {
    const order = await db.createOrder(orderData);

    ordersTotal.inc({ status: 'success' }); // Increment counter
    orderDuration.observe(Date.now() - start); // Record duration

    return order;
  } catch (error) {
    ordersTotal.inc({ status: 'failed' });
    orderDuration.observe(Date.now() - start);

    throw error;
  }
}

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', prometheus.register.contentType);
  res.end(await prometheus.register.metrics());
});
```

**What Prometheus scrapes:**
```
# HELP orders_total Total orders created
# TYPE orders_total counter
orders_total{status="success"} 1247923
orders_total{status="failed"} 1832

# HELP order_duration_ms Order creation duration
# TYPE order_duration_ms histogram
order_duration_ms_bucket{le="100"} 5432
order_duration_ms_bucket{le="500"} 9234
order_duration_ms_bucket{le="1000"} 9987
order_duration_ms_sum 2847293
order_duration_ms_count 10000
```

### Alerting on Metrics

**Alert Rule Examples:**

```yaml
# High error rate
- alert: HighErrorRate
  expr: |
    rate(orders_total{status="failed"}[5m])
    /
    rate(orders_total[5m])
    > 0.05
  for: 2m
  annotations:
    summary: "Order error rate above 5% for 2 minutes"
    description: "Current rate: {{ $value | humanizePercentage }}"

# Slow API (p95 latency)
- alert: SlowAPILatency
  expr: |
    histogram_quantile(0.95,
      rate(order_duration_ms_bucket[5m])
    ) > 1000
  for: 5m
  annotations:
    summary: "p95 latency above 1 second"

# Low request rate (service might be down)
- alert: LowTraffic
  expr: rate(orders_total[5m]) < 10
  for: 5m
  annotations:
    summary: "Order rate below 10/sec - service might be down"
```

**Alert best practices:**
- Use `for: 5m` to avoid alerting on brief spikes (flapping)
- Alert on symptoms (high latency) not causes (high CPU)
- Include context in annotations (current value, runbook link)

### Cardinality Explosion (The Danger)

**Problem:** Too many unique label combinations = too much data

**❌ BAD: Using user_id as label**
```javascript
const requests = new prometheus.Counter({
  name: 'api_requests',
  labelNames: ['user_id'] // 1 million users = 1 million time series!
});

requests.inc({ user_id: 'user_12345' });
```

**Why bad:**
- 1 million users = 1 million separate counters
- Prometheus stores each as separate time series
- Query performance dies, storage explodes

**✅ GOOD: Use low-cardinality labels**
```javascript
const requests = new prometheus.Counter({
  name: 'api_requests',
  labelNames: ['method', 'endpoint', 'status_code']
  // 5 methods × 20 endpoints × 5 status codes = 500 time series ✅
});

requests.inc({ method: 'POST', endpoint: '/orders', status_code: '200' });
```

**Safe labels:**
- HTTP method (GET, POST, etc.) - 10 values max
- HTTP status code (200, 404, 500, etc.) - 20 values max
- Service name - 50 services max
- Environment (prod, staging) - 3 values max

**Dangerous labels:**
- User ID (millions of users)
- Order ID (billions of orders)
- Email address (millions)
- Session ID (millions)

**Rule:** Total unique combinations < 10,000

### Dashboards That Actually Help

**Essential dashboard sections:**

**1. RED Metrics (Top)**
```
[Orders/sec]  [Error Rate %]  [p95 Latency]
   1,247         0.3%            320ms
```

**2. Traffic Patterns (Next)**
```
Graph: Orders per second over last 24 hours
- See traffic spikes
- Identify peak hours
- Spot anomalies
```

**3. Error Breakdown**
```
Table: Top error types
- PAYMENT_DECLINED: 45%
- DATABASE_TIMEOUT: 30%
- INVALID_ADDRESS: 25%
```

**4. Latency Distribution**
```
Graph: p50, p95, p99 over time
- Did deployment slow things down?
- Is p99 getting worse?
```

**5. Infrastructure (Bottom)**
```
CPU: 45%  |  Memory: 78%  |  DB Connections: 47/100
```

**Dashboard anti-patterns:**
- ❌ 50+ graphs on one page (overwhelming)
- ❌ Graphs with no context (what's good vs bad?)
- ❌ Averages without percentiles
- ❌ No time range selector

**✅ Good dashboard:**
- 5-10 key metrics at top
- Clear "red/yellow/green" zones
- Answers: "Is production healthy?"

## Practice Questions

1. **What's the difference between RED and USE methods?**
   - **RED**: For request-based services (APIs) - Rate, Errors, Duration
   - **USE**: For resources (servers, databases) - Utilization, Saturation, Errors
   - Use RED for application metrics, USE for infrastructure

2. **Why monitor p99 instead of just average latency?**
   - Average hides outliers (could be 200ms average but 5s for 1% of users)
   - 1% of 1 million users = 10,000 frustrated users
   - p99 shows worst-case experience for most users
   - Production systems should optimize for worst case, not average

3. **What causes cardinality explosion?**
   - Using high-cardinality labels (user_id, order_id, session_id)
   - Each unique label combination = separate time series
   - 1 million users as labels = 1 million time series
   - Solution: Use only low-cardinality labels (<10,000 combinations)

4. **When would you use a histogram vs a counter?**
   - **Counter**: Track totals (total orders, total errors)
   - **Histogram**: Track distributions (latency percentiles, request sizes)
   - Counter only goes up, histogram tracks buckets for percentiles
   - Use counter for "how many", histogram for "how long/large"

## Real-World Examples

- **Google**: Invented monitoring with Borgmon (precursor to Prometheus)
- **Uber**: Tracks p99 latency, alerts if >500ms for 5 minutes
- **Netflix**: RED metrics for all microservices, 99.99% availability target
- **Stripe**: Monitors payment processing latency at p99.9 (worst 0.1%)
- **Amazon**: "Every 100ms of latency costs 1% of sales" - obsessed with p99

## Resources & References

- Prometheus Documentation
- Grafana Dashboard Best Practices
- Google SRE Book: Monitoring Distributed Systems
- RED Method (Tom Wilkie)
- USE Method (Brendan Gregg)

## Practice Questions

## Real-World Examples

## Resources & References
