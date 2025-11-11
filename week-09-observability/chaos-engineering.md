# Chaos Engineering

## Learning Objectives
- Understand chaos engineering principles
- Learn failure injection and testing
- Master Netflix Chaos Monkey concepts

## Notes

### What is Chaos Engineering?

**Definition:** Intentionally breaking things in production to find weaknesses before they cause real outages.

**The philosophy:**
> "If we're going to fail anyway, let's fail on purpose during business hours when engineers are awake, not at 3 AM on Friday night."

**Key insight:**
- Systems WILL fail eventually
- Better to discover how they fail under controlled conditions
- Find weaknesses before customers do

### The Origin: Netflix Chaos Monkey

**The problem Netflix faced:**
```
Netflix runs on AWS (thousands of EC2 instances)
AWS instances fail randomly (hardware issues, network problems)
Engineers don't handle failures → System crashes during real failures
```

**Netflix's solution (2011):**
```
"Let's randomly kill production servers during business hours."
→ Forces engineers to build resilient systems
→ Failures become normal, not surprising
```

**Chaos Monkey:**
- Randomly terminates EC2 instances in production
- Runs during business hours (9 AM - 5 PM, Monday-Friday)
- No warning to engineers
- If system breaks → Engineers must fix it immediately

**Result:**
- Netflix services survive instance failures gracefully
- Auto-scaling kicks in automatically
- No customer impact when real failures happen

### Why Chaos Engineering Works

**Traditional testing:**
```
❌ Unit tests: Test individual functions (doesn't catch integration issues)
❌ Integration tests: Test service-to-service (doesn't catch real-world conditions)
❌ Staging environment: Not identical to production (different load, different data)
```

**Chaos engineering:**
```
✅ Tests in production: Real environment, real load, real data
✅ Tests unknown failures: Things you didn't think to test
✅ Validates resilience: Proves system handles failures
```

**Example: Netflix during AWS outage**
- 2012: AWS us-east-1 had major outage
- Most companies: Completely down for hours
- Netflix: Automatic failover to us-west, users didn't notice
- Why? They had been testing failures for years with Chaos Monkey

### The Chaos Engineering Lifecycle

**1. Define Steady State**
```
What does "normal" look like?
- 99.9% request success rate
- p95 latency < 500ms
- 1,000 orders/sec processed
```

**2. Form Hypothesis**
```
"If Payment Service fails, circuit breaker will open and users get 'payment pending' message. Order success rate stays > 99%."
```

**3. Introduce Chaos (The Experiment)**
```
Kill Payment Service instances
OR
Add 10 second latency to Payment Service
OR
Return 500 errors from Payment Service
```

**4. Measure Impact**
```
Did order success rate stay > 99%? ✅
Did circuit breaker open? ✅
Did users see graceful error message? ❌ (Found bug!)
```

**5. Learn and Improve**
```
Fix: Add fallback message when circuit breaker open
Re-run experiment: Now users see graceful message ✅
```

### Types of Chaos Experiments

#### 1. Instance Failure
**Kill random servers**
```bash
# Chaos Monkey
chaos-monkey terminate-instance --cluster order-service --random
```

**What you learn:**
- Does auto-scaling replace failed instances?
- Do load balancers detect failure quickly?
- Is there a single point of failure?

#### 2. Latency Injection
**Make service slow (simulate network issues)**
```javascript
// Add artificial delay
app.use((req, res, next) => {
  if (chaosMode === 'latency') {
    setTimeout(next, 5000); // 5 second delay
  } else {
    next();
  }
});
```

**What you learn:**
- Do timeouts work correctly?
- Does circuit breaker trigger?
- Does slow service cascade to others?

#### 3. Error Injection
**Return fake errors**
```javascript
app.get('/charge', (req, res) => {
  if (chaosMode === 'errors' && Math.random() < 0.5) {
    return res.status(500).json({ error: 'Internal Server Error' });
  }
  // Normal logic...
});
```

**What you learn:**
- Do retries work?
- Are errors logged correctly?
- Is fallback logic triggered?

#### 4. Resource Exhaustion
**Fill disk, exhaust memory, max out CPU**
```bash
# Fill disk space
dd if=/dev/zero of=/tmp/fill bs=1M count=10000

# Consume memory
stress-ng --vm 4 --vm-bytes 90%

# Max out CPU
stress-ng --cpu 8 --cpu-load 100
```

**What you learn:**
- Does monitoring alert on high resource usage?
- Does application handle out-of-memory gracefully?
- Is there proper disk space monitoring?

#### 5. Network Partition (Split Brain)
**Simulate network failure between services**
```bash
# Block traffic to database
iptables -A OUTPUT -d 10.0.1.50 -j DROP
```

**What you learn:**
- How long before failure detected?
- Does application retry correctly?
- Is there cache to fall back on?

#### 6. Dependency Failure
**Kill entire dependent service**
```
Kill all Payment Service instances
→ See if Order Service handles gracefully
```

**What you learn:**
- Circuit breakers configured correctly?
- Fallback strategies work?
- Alerts triggered appropriately?

### Game Days (Scheduled Chaos)

**Definition:** Scheduled chaos engineering event where team runs failure scenarios

**Typical Game Day:**
```
9:00 AM - Planning meeting
  - Define scenarios to test
  - Assign roles (chaos engineer, observers, on-call)
  - Review rollback plan

10:00 AM - Scenario 1: Database failure
  - Kill primary database
  - Observe: Does failover to replica work?
  - Duration: 30 minutes

11:00 AM - Debrief
  - What broke? What worked?
  - Action items to improve

12:00 PM - Scenario 2: Payment service latency
  - Inject 10s latency
  - Observe: Circuit breakers, timeouts
  - Duration: 30 minutes

1:00 PM - Final debrief
  - Document findings
  - Create tickets for improvements
```

**Benefits:**
- Controlled environment (team is ready)
- Learning opportunity for entire team
- Identifies gaps in monitoring and alerts
- Builds confidence in system resilience

### Blast Radius (Safety First)

**Critical rule:** Start small, expand gradually

**Levels:**

**1. Development environment**
```
Safe to break anything
No customer impact
Learn the basics
```

**2. Staging/Pre-production**
```
Real architecture
Synthetic load testing
Still no real customers
```

**3. Production - Single availability zone**
```
Real customers, but limited
Affect 10% of traffic
Can fail over to other zones
```

**4. Production - Multi-zone**
```
Broader impact
30-50% of traffic
More confidence needed
```

**5. Production - Global**
```
Full production chaos
Only after many successful smaller tests
Requires extensive safeguards
```

**Example progression:**
```
Week 1: Kill one server in dev → Success
Week 2: Kill one server in staging → Success
Week 3: Kill one server in prod (single AZ) → Found bug in monitoring
Week 4: Fix bug, retest → Success
Week 5: Kill multiple servers (multi-AZ) → Success
```

### Guardrails and Safety Mechanisms

**1. Circuit breaker for chaos**
```javascript
if (errorRate > 5% || latency_p95 > 2000) {
  console.log('ABORT: Chaos experiment caused user impact');
  stopChaos();
  rollback();
  alertTeam();
}
```

**2. Business hours only**
```
Run chaos 9 AM - 5 PM, Monday-Thursday
Avoid Friday (weekend on-call harder)
Avoid holidays and Black Friday
```

**3. Gradual rollout**
```
Start with 1% of servers
If successful, increase to 10%
Then 30%, then 100%
```

**4. Manual approval**
```
Require engineer to approve each experiment
Review blast radius and rollback plan
Confirm monitoring is working
```

**5. Automatic rollback**
```
If experiment detects issues:
- Stop chaos immediately
- Restore healthy state
- Alert team
```

### Tools for Chaos Engineering

**1. Chaos Monkey (Netflix)**
- Kills random EC2 instances
- Simple, effective for compute failures

**2. Chaos Kong (Netflix)**
- Kills entire AWS regions
- Tests multi-region failover

**3. Gremlin (Commercial)**
- Full chaos engineering platform
- CPU, memory, network, disk attacks
- Scheduled Game Days
- Safety guardrails built-in

**4. Litmus Chaos (Open Source)**
- Kubernetes-native
- YAML-based chaos experiments
- Good for containerized apps

**5. AWS Fault Injection Simulator**
- Managed AWS service
- Chaos experiments on AWS resources
- Built-in safety mechanisms

### Chaos Experiment Example

**Scenario: Test Payment Service failure handling**

**1. Hypothesis:**
```
"When Payment Service is unavailable, Order Service will:
- Detect failure within 5 seconds
- Open circuit breaker
- Return 'Payment Pending' to user
- Queue order for retry
- Maintain 99% order creation success rate"
```

**2. Experiment setup:**
```yaml
experiment:
  name: payment-service-failure
  target: payment-service
  action: terminate-instances
  scope:
    percentage: 100%  # Kill all instances
    duration: 5 minutes
  abort-conditions:
    order-success-rate: < 99%
    latency-p95: > 2000ms
```

**3. Run experiment:**
```bash
chaos-toolkit run payment-failure-experiment.yaml
```

**4. Observations:**
```
✅ Circuit breaker opened after 3 failures (6 seconds)
✅ Order Service returned 'Payment Pending' message
❌ Queue failed to start (found bug!)
❌ 95% success rate (below 99% threshold)
→ Experiment aborted after 2 minutes
```

**5. Improvements:**
```
Fix 1: Auto-start queue when circuit opens
Fix 2: Better error handling in Order Service
Re-run experiment → Success! ✅
```

### Cultural Shift

**Before Chaos Engineering:**
```
Engineer: "I hope my service doesn't fail"
Outage at 2 AM → Panic, scramble to fix
Post-mortem: "We should have tested this"
```

**After Chaos Engineering:**
```
Engineer: "Let's break it and see what happens"
Failure during Game Day → Calm, follow runbook
Post-experiment: "Great, we found 3 things to improve"
```

**Building resilience culture:**
- Failures are learning opportunities
- Breaking things is encouraged (safely)
- Resilience is everyone's responsibility
- Proactive > Reactive

### Metrics to Track

**Before experiment:**
```
Success rate: 99.95%
p95 latency: 320ms
Error rate: 0.05%
```

**During experiment:**
```
Success rate: 99.1% (slight drop, acceptable)
p95 latency: 450ms (increased, but < threshold)
Error rate: 0.9% (increased, monitoring for threshold)
Circuit breaker state: OPEN ✅
```

**After experiment:**
```
Success rate: 99.95% (recovered)
p95 latency: 310ms (back to normal)
Error rate: 0.04% (normal)
Recovery time: 2 minutes ✅
```

## Practice Questions

1. **What is chaos engineering and why is it valuable?**
   - Intentionally breaking production to find weaknesses
   - Find failures under controlled conditions (business hours, monitoring ready)
   - Better than discovering failures at 3 AM during real incident
   - Builds resilient systems and confident teams

2. **What is Netflix Chaos Monkey and what problem does it solve?**
   - Randomly terminates EC2 instances in production during business hours
   - Forces engineers to build systems that survive instance failures
   - Made Netflix resilient to AWS outages that took down other companies
   - Failure becomes expected, not surprising

3. **What is a blast radius and why is it important?**
   - Scope of impact from chaos experiment
   - Start small (dev → staging → 10% prod → 100% prod)
   - Prevents chaos experiment from causing real outage
   - Build confidence through gradual expansion

4. **What is a Game Day?**
   - Scheduled chaos engineering event
   - Team runs failure scenarios in controlled environment
   - Observe how system responds, document findings
   - Learning opportunity + confidence building
   - Typically 2-3 hours with multiple scenarios

## Real-World Examples

- **Netflix**: Created Chaos Monkey (2011), survived AWS us-east-1 outage (2012)
- **Amazon**: Runs Game Days quarterly, tests region failures
- **Google**: DiRT (Disaster Recovery Testing) - annual chaos event
- **Facebook**: Regularly tests datacenter failures
- **Target**: Avoided Black Friday outage through pre-holiday chaos testing

## Resources & References

- Netflix Chaos Engineering Blog
- Principles of Chaos Engineering (chaoseng.io)
- Chaos Monkey GitHub Repository
- Google DiRT (Disaster Recovery Testing)
- "Chaos Engineering" book by Casey Rosenthal

## Practice Questions

## Real-World Examples

## Resources & References
