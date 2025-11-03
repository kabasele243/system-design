# Social Media Feed - Failure Scenarios

## Step 5: Failure Scenarios & Monitoring

### Failure Scenario 1: Message Queue Failure

**What happens?**


**How to detect:**


**How to recover:**


**Mitigation:**


### Failure Scenario 2: Feed Cache Failure

**What happens?**


**How to detect:**


**How to recover:**


**Mitigation:**


### Failure Scenario 3: Database Hotspot (Celebrity Reads)

**What happens?**


**How to detect:**


**How to recover:**


**Mitigation:**


## Monitoring & Metrics

### Key Metrics:
1. **Feed Generation Latency:**
   - P50, P99 latency
   - Target: < 200ms

2. **Post Creation Latency:**

3. **Cache Hit Rate:**

4. **Message Queue Lag:**

5. **Feed Freshness:**
   - Time between post creation and appearance in follower feeds

## Summary

**Your complete architecture diagram:**


**Key design decisions:**
1.
2.
3.

**Fan-out strategy chosen:**


**How you handle celebrities:**


## Resources
- [Instagram Feed Architecture](https://instagram-engineering.com)
- [Twitter Timeline Architecture](https://blog.twitter.com)
