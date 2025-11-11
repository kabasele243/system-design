# SLIs/SLOs/SLAs

## Learning Objectives
- Understand Service Level Indicators (SLIs)
- Learn Service Level Objectives (SLOs)
- Master Service Level Agreements (SLAs)

## Notes

### The Reliability Hierarchy

```
SLA (Legal contract with customers)
  ↓
SLO (Internal target, stricter than SLA)
  ↓
SLI (Actual measurements)
```

### SLI (Service Level Indicator)

**Definition:** A measurement of your service's behavior
- Raw numbers from your system
- What you actually measure

**Common SLIs:**

**1. Availability** (uptime)
```
availability = successful_requests / total_requests
Example: 9,987 success / 10,000 total = 99.87%
```

**2. Latency** (speed)
```
p95_latency = 95th percentile response time
Example: 95% of requests complete in < 500ms
```

**3. Error Rate**
```
error_rate = failed_requests / total_requests
Example: 13 failures / 10,000 requests = 0.13%
```

**4. Throughput**
```
throughput = requests_per_second
Example: Handle 1,000 orders/sec
```

**Real example - Order Service SLIs:**
```
✅ Availability: 99.95% (5 failures in 10,000 requests)
✅ p95 latency: 320ms
✅ Error rate: 0.05%
✅ Throughput: 1,247 req/sec
```

### SLO (Service Level Objective)

**Definition:** Target value for an SLI
- Internal goal (not shown to customers)
- More strict than SLA (buffer for safety)
- What you aim to achieve

**Format:** "X% of requests should complete in < Y ms over Z time period"

**Example SLOs:**

```
1. Availability SLO
   "99.9% of requests succeed over 30 days"

2. Latency SLO
   "95% of requests complete in < 500ms over 7 days"

3. Error Rate SLO
   "< 1% error rate over 24 hours"
```

**Why multiple time windows?**
- 30-day window: Long-term trend, smoother
- 7-day window: Medium-term, catch issues faster
- 24-hour window: Immediate problems, very sensitive

### SLA (Service Level Agreement)

**Definition:** Legal contract with customers
- Public promise (written in contract)
- Less strict than SLO (safety buffer)
- Financial penalty if violated

**Example SLA:**
```
"Uber Eats guarantees 99.5% API availability per month.
If we fall below 99.5%, you get 10% credit on your bill."
```

**Key components:**
1. **Measurement period**: Monthly, quarterly
2. **Target**: 99.5% availability
3. **Penalty**: Refund/credit if missed
4. **Exclusions**: Scheduled maintenance, customer's fault

### The Buffer Between SLO and SLA

**Critical:** SLO must be stricter than SLA!

```
SLA (Customer promise): 99.5%
    ↑ (0.4% buffer)
SLO (Internal target): 99.9%
    ↑ (actual performance)
SLI (What we measure): 99.95%
```

**Why the buffer?**
- Gives room for incidents without breaking SLA
- Time to fix issues before customers notice
- Avoid paying penalties

**Bad scenario (no buffer):**
```
❌ SLA: 99.9%
❌ SLO: 99.9% (same!)
→ One small incident = Violated SLA = Pay penalty
```

**Good scenario (with buffer):**
```
✅ SLA: 99.5%
✅ SLO: 99.9% (0.4% buffer)
→ Small incident = Missed SLO but SLA safe
→ Time to fix before breaking customer promise
```

### The "Nines" (Availability Levels)

```
99%      = 3.65 days downtime/year
99.9%    = 8.76 hours downtime/year  ← Most consumer apps
99.95%   = 4.38 hours downtime/year
99.99%   = 52 minutes downtime/year  ← Financial apps
99.999%  = 5.26 minutes downtime/year ← Emergency services
```

### The Industry Standard: 99.9% (Three Nines)

**Most consumer apps target 99.9% - here's why:**

**Cost vs Benefit Analysis:**

**99.9% (8.76 hours downtime/year):**
- Annual cost: ~$100K
- Infrastructure: Multi-AZ (not multi-region)
- On-call: Business hours + occasional nights
- User perception: "App works most of the time" ✅

**99.99% (52 minutes downtime/year):**
- Annual cost: ~$500K (5x more expensive!)
- Infrastructure: Multi-region, complex failover
- On-call: 24/7/365
- User perception: Still "App works most of the time" ✅

**The difference:** Users don't notice 8 hours vs 52 minutes over a YEAR!

**When to choose 99.99%+:**
- Financial transactions (money lost during downtime)
- Healthcare (life-critical systems)
- Emergency services (911, ambulance dispatch)
- Large scale (Amazon - every minute down = millions lost)

**When 99.9% is enough:**
- Social media apps
- Food delivery
- E-commerce (not Black Friday)
- SaaS tools

### Error Budget

**Definition:** How much failure you're allowed before breaking SLO

**Calculation:**
```
SLO: 99.9% availability over 30 days

Total minutes in 30 days: 43,200 minutes
Allowed downtime (0.1%): 43.2 minutes

Error budget = 43.2 minutes of downtime
```

**Monthly tracking:**
```
Day 1:  Incident (10 min) → Budget remaining: 33.2 min
Day 7:  Incident (5 min)  → Budget remaining: 28.2 min
Day 15: Incident (8 min)  → Budget remaining: 20.2 min
Day 22: Incident (15 min) → Budget remaining: 5.2 min ⚠️
```

**When 93% of error budget used (Day 22):**
```
❌ BAD: "Deploy carefully with extra testing"
→ Testing takes time, might still break
→ Slow down velocity

✅ GOOD: "Freeze all deployments until next month"
→ Preserve remaining 5.2 minutes
→ Only emergency fixes allowed
→ Focus on stability
```

### Error Budget Burn Rate

**How fast are you using your error budget?**

**Example:** 30-day SLO with 43.2 minutes error budget

**Burn rate calculation:**
```
If 10 minutes used in first 3 days:
Burn rate = (10 min / 3 days) × 30 days = 100 minutes

→ At this rate, you'll exceed budget by Day 13! ⚠️
```

**Alert on burn rate:**
```
If burn rate > 2x normal → Page on-call immediately
(Using error budget twice as fast as sustainable)
```

### Multi-Window Multi-Burn-Rate Alerts

**Google's SRE approach:**

```yaml
# Fast burn (2% budget in 1 hour)
Alert: Critical
Condition: 2% error budget burned in last 1 hour
Action: Page on-call immediately
Why: At this rate, budget gone in 2 days!

# Medium burn (5% budget in 6 hours)
Alert: Warning
Condition: 5% error budget burned in last 6 hours
Action: Slack notification
Why: Trending toward trouble

# Slow burn (10% budget in 3 days)
Alert: FYI
Condition: 10% error budget burned in last 3 days
Action: Email team
Why: Keep an eye on it
```

### SLO-Based Alerting

**Traditional alerting (bad):**
```
❌ Alert: CPU > 80%
→ So what? Is user affected?
→ Lots of noise
```

**SLO-based alerting (good):**
```
✅ Alert: Error budget burn rate > 2x
→ Directly tied to user experience
→ Only alerts when users are affected
→ Actionable: "Stop deploys" or "Fix now"
```

**Symptom-based (what user sees) vs Cause-based (why it happened):**
```
✅ "p95 latency > 1s" (symptom - user affected)
❌ "Database CPU > 90%" (cause - might not affect users)
```

**Alert on symptoms, investigate causes.**

### Real-World Example: Uber Eats

**SLAs (Customer promise):**
```
API Availability: 99.5% per month
→ Customer gets 10% credit if missed
```

**SLOs (Internal targets):**
```
API Availability: 99.9% per month
Order Placement: 99.95% success rate
p95 Latency: < 500ms
Error Rate: < 0.5%
```

**SLIs (What they measure):**
```
Availability = successful_api_calls / total_api_calls
Latency = histogram_quantile(0.95, order_placement_duration)
Error Rate = failed_orders / total_orders
```

**Error Budget:**
```
99.9% SLO over 30 days = 43.2 minutes downtime allowed

Budget usage:
Week 1: 10 minutes (23%)
Week 2: 8 minutes (19%)
Week 3: 15 minutes (35%) ← 77% used total
Week 4: ⚠️ Deploy freeze! Only 10 minutes left
```

### Tracking SLOs in Practice

**Dashboard example:**
```
┌────────────────────────────────────────┐
│ Order Service SLO Dashboard            │
├────────────────────────────────────────┤
│ Availability (30-day SLO: 99.9%)       │
│ Current: 99.94% ✅                     │
│ Error Budget: 17.2 min remaining (40%) │
│                                         │
│ Latency (7-day SLO: p95 < 500ms)      │
│ Current: p95 = 420ms ✅                │
│                                         │
│ Error Rate (24-hour SLO: < 1%)         │
│ Current: 0.3% ✅                       │
└────────────────────────────────────────┘
```

**What happens when SLO at risk:**
```
1. Error budget 70% used → Warning to team
2. Error budget 90% used → Deploy freeze
3. SLO missed → Post-mortem required
4. SLA missed → Pay customer penalty + CEO involved
```

### Trade-offs: Reliability vs Velocity

**Too strict SLO (99.99%):**
- ❌ Slow feature development
- ❌ Deploy freeze frequently
- ❌ Team burnout from on-call
- ✅ Extremely reliable

**Too loose SLO (99%):**
- ✅ Fast feature development
- ✅ Deploy anytime
- ❌ Users frustrated by downtime
- ❌ Reputation damage

**Balanced SLO (99.9%):**
- ✅ Deploy regularly
- ✅ Innovation continues
- ✅ Reliability good enough for users
- ✅ Error budget allows experimentation

**Use error budget to balance:**
```
Budget healthy (50%+ remaining):
→ Deploy 10x/day, experiment, take risks

Budget critical (90%+ used):
→ Deploy freeze, only critical fixes, focus on stability
```

## Practice Questions

1. **What's the relationship between SLI, SLO, and SLA?**
   - **SLI**: What you measure (99.95% actual availability)
   - **SLO**: Internal target (99.9% goal)
   - **SLA**: Customer promise (99.5% contractual)
   - SLO must be stricter than SLA to provide safety buffer

2. **Why is 99.9% the common target for consumer apps?**
   - Only 8.76 hours downtime per year
   - Users perceive as "always available"
   - Cost: ~$100K/year (affordable for most companies)
   - 99.99% costs 5x more but users don't notice difference
   - Good balance between reliability and cost

3. **What is an error budget and how is it used?**
   - Amount of failure allowed before breaking SLO
   - 99.9% SLO = 43.2 minutes downtime per month
   - Budget used = actual downtime
   - When 90% used: Deploy freeze until next period
   - Balances innovation (deployments) with reliability

4. **What should you do when error budget is 93% consumed?**
   - **Freeze all deployments** until next month
   - Only critical bug fixes allowed
   - Focus on stability, not features
   - Alternative (worse): Deploy carefully still risks breaking SLA
   - Preserve remaining 7% for unexpected issues

## Real-World Examples

- **Google**: Invented SRE and error budgets, publishes SLO best practices
- **AWS**: 99.99% SLA for EC2, multi-region for higher availability
- **Stripe**: 99.99% API availability SLA for payment processing
- **Netflix**: 99.99% streaming availability, costs justified by scale
- **Uber**: 99.9% for consumer apps, 99.95% for driver-facing critical services

## Resources & References

- Google SRE Book: Implementing SLOs
- Site Reliability Engineering Workbook
- AWS Well-Architected Framework: Reliability Pillar
- The Art of SLOs (Alex Hidalgo)
- SLO Workshop Materials (Google)

## Practice Questions

## Real-World Examples

## Resources & References
The Industry Standard: 99.9% (Three Nines)
Most consumer apps target 99.9% - here's why:
Cost vs Benefit Analysis
99.9% (8.76 hours downtime/year):
Annual cost: ~$100K
Infrastructure: Multi-AZ (not multi-region)
On-call: Business hours + occasional nights
User perception: "App works most of the time" ✅
99.99% (52 minutes downtime/year):
Annual cost: ~$500K (5x more expensive!)
Infrastructure: Multi-region, complex failover
On-call: 24/7/365
User perception: Still "App works most of the time" ✅
The difference: Users don't notice 8 hours vs 52 minutes over a YEAR!
