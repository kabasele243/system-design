# Structured Logging

## Learning Objectives
- Understand structured vs unstructured logs
- Learn log levels and best practices
- Master centralized logging (ELK stack)

## Notes

### The Problem with Unstructured Logs

**Unstructured logging (bad!):**
```
Order failed
Error processing payment
Database timeout
```

**Problems:**
- ❌ No order ID, user ID, timestamp
- ❌ Can't search or filter
- ❌ Can't aggregate failures
- ❌ Useless for debugging at 2 AM

### Structured Logging (Solution)

**JSON format with context:**
```json
{
  "level": "ERROR",
  "message": "Order failed",
  "orderId": "order_12345",
  "userId": "user_789",
  "errorCode": "PAYMENT_DECLINED",
  "timestamp": "2025-11-10T14:30:45Z",
  "service": "order-service",
  "environment": "production"
}
```

**Benefits:**
- ✅ Search: "Show all failures for user_789"
- ✅ Filter: "Show PAYMENT_DECLINED errors"
- ✅ Aggregate: "Count failures by restaurant"
- ✅ Dashboard: Graph error trends
- ✅ Alert: Notify when rate > threshold

### Log Levels (From Least to Most Severe)

**1. TRACE** - Extremely detailed (rarely used)
- Use for: Following code execution step-by-step
- Production: Usually OFF (too noisy)

**2. DEBUG** - Detailed information for developers
- Use for: Development, troubleshooting
- Production: Usually OFF (too much data, costs money)

**3. INFO** - Important business events
- Use for: Key milestones (order placed, payment processed)
- Production: ON for critical events, sample for volume

**4. WARN** - Unexpected but system still works
- Use for: Degraded performance, retry logic triggered
- Production: Always ON

**5. ERROR** - Something failed, user affected
- Use for: User-facing failures, exceptions caught
- Production: Always ON + Alert if rate high

**6. FATAL/CRITICAL** - System dying/crashed
- Use for: System crash imminent, can't recover
- Production: Always ON + Page on-call immediately

### Production Log Strategy

**Enable in production:**
- ✅ CRITICAL/FATAL - Always
- ✅ ERROR - Always
- ✅ WARN - Always
- ⚠️ INFO - Sample 1% or only critical events
- ❌ DEBUG - Off (use metrics instead)
- ❌ TRACE - Off

**Why sample INFO?**
- 1,000 orders/sec × 5 logs each = 5,000 logs/sec
- = 13 billion logs/month
- = 6.5 TB storage = $3,250/month just for INFO logs!

**Better: Use metrics for volume, logs for failures**

### Centralized Logging

**Problem:** Logs scattered across 100 servers
- Manual SSH to each server
- Takes 20+ minutes to find issue

**Solution:** Ship all logs to central store
```
All servers → Logstash/Filebeat → Elasticsearch → Kibana (UI)
```

**Popular solutions:**
- **ELK Stack** (Elasticsearch, Logstash, Kibana) - Open-source
- **Datadog** - SaaS, $1.27/GB ingested
- **Splunk** - Enterprise, expensive
- **AWS CloudWatch Logs** - AWS-native
- **Grafana Loki** - Lightweight, cheaper than ELK

### How Centralized Logging Works

1. **Application writes log**
   ```javascript
   logger.error("Order failed", { orderId: "123", errorCode: "TIMEOUT" });
   ```

2. **Log shipped to collector**
   - Filebeat reads log file
   - Sends to Logstash/Elasticsearch

3. **Elasticsearch indexes**
   - Stores and indexes all fields
   - Fast search (milliseconds)

4. **Search in Kibana**
   ```
   orderId:"order_123" → Shows all logs for this order
   errorCode:"PAYMENT_DECLINED" → Shows all payment failures
   ```

### Alerting on Logs

**Set up automated alerts:**

```javascript
Alert: High Error Rate
Condition: Count of ERROR logs > 100 in 5 minutes
Action: Page on-call engineer

Alert: Specific Error
Condition: errorCode:"DATABASE_CONNECTION_FAILED" > 10/5min
Action: Page database team

Alert: Missing Logs (Service Down)
Condition: service:"order-service" count < 100/5min
Action: Critical alert (service might be crashed)
```

### Alert Best Practices

**✅ DO:**
- Alert on symptoms, not causes ("API error rate high" not "CPU high")
- Include actionable context (error types, affected users, runbook link)
- Different channels by severity (P0=call, P1=SMS, P2=Slack, P3=email)

**❌ DON'T:**
- Alert fatigue (too many alerts = ignored)
- Duplicate alerts (alert once per issue)
- Alert with no action (include runbook)

### What to Log

**Always log:**
- Request/response IDs (for tracing)
- User IDs (for debugging specific users)
- Error codes and messages
- Timestamps (ISO 8601 format)
- Service name and environment
- Important state changes

**Add for debugging:**
- Function parameters
- External API responses
- Retry attempts
- Fallback usage

**Never log:**
- Passwords or secrets
- Credit card numbers
- PII without masking
- Entire request bodies (might have sensitive data)

## Practice Questions

1. **Why use structured logging instead of plain text?**
   - Searchable by any field
   - Can aggregate and create dashboards
   - Easy to parse programmatically
   - Enables powerful queries across millions of logs

2. **Which log levels should be enabled in production?**
   - CRITICAL/ERROR/WARN: Always ON
   - INFO: Sample 1% or only critical events
   - DEBUG/TRACE: OFF (use metrics instead)
   - Reason: Cost and noise vs value

3. **What's the cost of logging every successful order?**
   - 1,000 orders/sec × 500 bytes = 500KB/sec
   - = 13 billion logs/month = 6.5 TB
   - At $0.50/GB = $3,250/month
   - Better: Use metrics for success count, logs for failures

4. **How do centralized logging systems work?**
   - Agents ship logs from all servers
   - Central store (Elasticsearch) indexes them
   - UI (Kibana) provides search interface
   - Can query across all services in seconds

## Real-World Examples

- **Uber**: ELK stack, 10TB logs/day, 30-day retention, sample INFO logs
- **Netflix**: Custom logging, logs errors only, uses metrics for success
- **Stripe**: Logs ALL financial transactions (compliance), samples other logs
- **Amazon**: CloudWatch Logs, separate retention by criticality

## Resources & References

- ELK Stack Documentation
- Datadog Logging Best Practices
- Structured Logging with JSON
- Log Levels Standard (RFC 5424)
