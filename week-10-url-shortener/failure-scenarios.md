# URL Shortener - Failure Scenarios & Monitoring

## Step 5: Discuss Failure Scenarios & Monitoring

### Failure Scenario 1: Database Failure

**What happens?**


**How to detect:**


**How to recover:**


**Mitigation:**


### Failure Scenario 2: Cache Failure

**What happens?**


**How to detect:**


**How to recover:**


**Mitigation:**


### Failure Scenario 3: Server Failure

**What happens?**


**How to detect:**


**How to recover:**


**Mitigation:**


## Monitoring & Metrics

### Key Metrics to Track:
1. **Request Rate:**
   - Shortening requests per second
   - Redirect requests per second

2. **Latency:**
   - P50, P99 latency for redirects
   - Target: < 100ms

3. **Error Rate:**
   - 4xx errors (bad requests)
   - 5xx errors (server errors)

4. **Database:**
   - Query latency
   - Connection pool utilization

5. **Cache:**
   - Hit rate (target: > 80%)
   - Miss rate

### Alerting Rules:
- Error rate > 1% for 5 minutes
- P99 latency > 200ms for 5 minutes
- Cache hit rate < 70% for 10 minutes

## Summary & Final Diagram

**Your complete architecture diagram:**


**Key design decisions:**
1.
2.
3.

**Trade-offs made:**
1.
2.
3.

## Resources
- [Bitly Architecture Blog]()
- [TinyURL System Design]()
