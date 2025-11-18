# Deployment Strategies

## Overview

As a senior engineer, you need to deploy changes without downtime, minimize risk, and have quick rollback capabilities. Modern deployment strategies enable continuous delivery while protecting users from bugs.

## The Deployment Spectrum

```
Risk ←                                                    → Safety
Speed →                                                   ← Slow

Big Bang → Rolling → Blue-Green → Canary → Feature Flags
```

## Big Bang Deployment (Don't Do This)

**Replace all instances at once**

```
All servers v1 (10:00 PM) → Take down → Deploy v2 → All servers v2 (10:30 PM)
                           ← 30 min downtime →
```

**Why it's bad:**
- Downtime (unacceptable for production)
- All users affected if bugs
- No rollback (must deploy previous version)

**Only acceptable for:**
- Internal tools
- Maintenance windows
- Startups with no traffic

## Rolling Deployment

**Update servers one-by-one or in small groups**

```
Time 0:  [v1] [v1] [v1] [v1] [v1] [v1]
Time 1:  [v2] [v2] [v1] [v1] [v1] [v1]  ← Update 2 servers
Time 2:  [v2] [v2] [v2] [v2] [v1] [v1]  ← Update 2 more
Time 3:  [v2] [v2] [v2] [v2] [v2] [v2]  ← Complete
```

### Implementation (Kubernetes)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # Max 1 pod down at a time
      maxSurge: 1        # Max 1 extra pod during update
  template:
    spec:
      containers:
      - name: app
        image: myapp:v2
```

**How it works:**
1. Create 1 new pod (v2)
2. Wait for health check to pass
3. Terminate 1 old pod (v1)
4. Repeat until all pods are v2

### Rolling Deployment with AWS ECS

```json
{
  "serviceName": "my-service",
  "deploymentConfiguration": {
    "minimumHealthyPercent": 50,  // Keep 50% running
    "maximumPercent": 200         // Can temporarily run 200% capacity
  }
}
```

### Pros/Cons

**Pros:**
- Zero downtime
- Simple to implement
- No extra infrastructure

**Cons:**
- Both versions running simultaneously (compatibility required)
- Slow rollback (must roll forward with previous version)
- Database migrations tricky (both versions must work with schema)
- Gradual rollout of bugs (affects more users over time)

## Blue-Green Deployment

**Maintain two identical environments, switch traffic instantly**

```
Blue (v1) ← 100% traffic
Green (v2) ← 0% traffic (new version deployed, tested)

[Switch traffic]

Blue (v1) ← 0% traffic (keep for quick rollback)
Green (v2) ← 100% traffic
```

### Implementation (AWS with Load Balancer)

```bash
# 1. Deploy v2 to Green environment
aws ecs update-service --service green-service --task-definition app:v2

# 2. Test Green environment
curl https://green.internal.example.com

# 3. Switch traffic (update load balancer target group)
aws elbv2 modify-listener --listener-arn $LISTENER_ARN \
  --default-actions Type=forward,TargetGroupArn=$GREEN_TARGET_GROUP

# 4. If issues, instantly rollback
aws elbv2 modify-listener --listener-arn $LISTENER_ARN \
  --default-actions Type=forward,TargetGroupArn=$BLUE_TARGET_GROUP
```

### Blue-Green with DNS

```bash
# Blue environment
app-blue.example.com -> Load Balancer 1

# Green environment
app-green.example.com -> Load Balancer 2

# Production DNS (switch instantly)
www.example.com CNAME app-blue.example.com  # Before
www.example.com CNAME app-green.example.com # After (instant switch)
```

**Warning:** DNS TTL can cause delays (cached DNS entries)

### Pros/Cons

**Pros:**
- Instant rollback (switch back to Blue)
- Test Green in production before switching
- No version mixing (only one version receives traffic)
- Low-risk

**Cons:**
- **2x infrastructure cost** (both environments running)
- Database migrations still tricky (shared DB between Blue/Green)
- Not true incremental rollout (all users switch at once)

## Canary Deployment

**Release to small percentage of users first, gradually increase**

```
Time 0:  v1: 100% traffic
Time 1:  v1: 95% traffic, v2: 5% traffic  (canary)
Time 2:  v1: 90% traffic, v2: 10% traffic
Time 3:  v1: 50% traffic, v2: 50% traffic
Time 4:  v1: 0% traffic, v2: 100% traffic
```

**If errors detected at any stage → rollback, only 5% of users affected**

### Implementation (Nginx)

```nginx
upstream backend {
    server v1-server:8000 weight=95;  # 95% to v1
    server v2-server:8001 weight=5;   # 5% to v2 (canary)
}

server {
    location / {
        proxy_pass http://backend;
    }
}
```

### Implementation (Kubernetes with Istio)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app
spec:
  hosts:
  - myapp.example.com
  http:
  - match:
    - headers:
        cookie:
          regex: ".*canary=true.*"  # Internal testing
    route:
    - destination:
        host: myapp-v2
        weight: 100
  - route:
    - destination:
        host: myapp-v1
        weight: 95  # 95% to v1
    - destination:
        host: myapp-v2
        weight: 5   # 5% to v2 (canary)
```

### Canary with AWS ALB

```json
{
  "Type": "forward",
  "ForwardConfig": {
    "TargetGroups": [
      {
        "TargetGroupArn": "arn:aws:elasticloadbalancing:...v1",
        "Weight": 95
      },
      {
        "TargetGroupArn": "arn:aws:elasticloadbalancing:...v2",
        "Weight": 5
      }
    ]
  }
}
```

### Automated Canary Analysis

```python
import time

def canary_deployment():
    # Stage 1: 5% traffic
    set_traffic_split(v1=95, v2=5)
    time.sleep(600)  # Wait 10 minutes

    if error_rate("v2") > error_rate("v1") * 1.5:
        rollback()
        return

    # Stage 2: 25% traffic
    set_traffic_split(v1=75, v2=25)
    time.sleep(600)

    if error_rate("v2") > error_rate("v1") * 1.2:
        rollback()
        return

    # Stage 3: 50% traffic
    set_traffic_split(v1=50, v2=50)
    time.sleep(600)

    if error_rate("v2") > error_rate("v1") * 1.1:
        rollback()
        return

    # Stage 4: 100% traffic
    set_traffic_split(v1=0, v2=100)

def error_rate(version):
    # Query metrics from Prometheus/CloudWatch
    return get_metric(f"error_rate__{version}")

def rollback():
    set_traffic_split(v1=100, v2=0)
    alert_team("Canary failed, rolled back to v1")
```

### Pros/Cons

**Pros:**
- Low risk (only 5% of users affected initially)
- Gradual rollout (catch bugs early)
- Data-driven (automated based on metrics)
- Quick rollback

**Cons:**
- Complex infrastructure (traffic splitting)
- Monitoring required (need metrics to decide)
- Slower than blue-green (gradual over hours)

## Feature Flags (Feature Toggles)

**Deploy code with new features disabled, enable selectively**

```python
# Code deployed to production (new feature hidden)
def checkout(user):
    if feature_enabled("new_checkout_flow", user):
        return new_checkout()  # New code
    else:
        return old_checkout()  # Old code

# Control via config/database
feature_flags = {
    "new_checkout_flow": {
        "enabled": True,
        "rollout_percentage": 10,  # 10% of users
        "allowlist": ["user_123", "user_456"]  # Internal testers
    }
}
```

### Feature Flag Service

```python
class FeatureFlagService:
    def __init__(self):
        self.flags = self.load_from_db()

    def is_enabled(self, flag_name, user_id):
        flag = self.flags.get(flag_name)

        if not flag or not flag['enabled']:
            return False

        # Allowlist (specific users)
        if user_id in flag.get('allowlist', []):
            return True

        # Rollout percentage (hash-based deterministic)
        rollout = flag.get('rollout_percentage', 0)
        user_hash = hash(f"{flag_name}:{user_id}") % 100
        return user_hash < rollout

# Usage
flags = FeatureFlagService()

if flags.is_enabled("new_checkout_flow", current_user.id):
    render_new_checkout()
else:
    render_old_checkout()
```

### Feature Flag Providers

**LaunchDarkly:**
```python
from launchdarkly import LDClient

client = LDClient("sdk-key")

user = {
    "key": "user-123",
    "email": "user@example.com"
}

if client.variation("new-checkout", user, False):
    new_checkout()
else:
    old_checkout()
```

**Split.io, Optimizely, ConfigCat** (similar APIs)

### Progressive Rollout with Feature Flags

```
Day 1:  10% of users see new feature
Day 2:  25% of users
Day 3:  50% of users
Day 4:  100% of users (if metrics look good)

If issues at any stage -> set percentage to 0% (instant rollback, no deployment)
```

### Feature Flag Types

1. **Release Flags (short-lived)**
   - Deploy feature disabled
   - Enable gradually
   - Remove flag after full rollout

2. **Ops Flags (long-lived)**
   - Circuit breakers
   - Kill switches (disable feature under load)
   - Example: "Disable image uploads if S3 is down"

3. **Experiment Flags (A/B testing)**
   - Test variations
   - Example: "50% see blue button, 50% see red button"

4. **Permission Flags**
   - Enable for premium users only
   - Example: "AI features only for paid tier"

### Pros/Cons

**Pros:**
- **Decouple deployment from release** (deploy anytime, enable later)
- Instant rollback (no code deployment)
- Targeted rollout (specific users, percentages)
- A/B testing built-in

**Cons:**
- **Code complexity** (if/else everywhere)
- **Technical debt** (old code paths remain)
- **Testing explosion** (must test all flag combinations)
- **Flag sprawl** (hundreds of flags over time)

**Best practice:** Clean up flags after full rollout (remove old code paths)

## Shadow Deployment (Dark Launch)

**Deploy new version, run it in parallel, don't return results to users**

```
User Request
    ↓
Load Balancer
    ├→ v1 (returns response to user)
    └→ v2 (processes request, logs results, doesn't return to user)
```

### Implementation

```python
def handle_request(request):
    # v1 handles request (production)
    response_v1 = process_v1(request)

    # v2 processes in background (shadow)
    async_process_v2(request)  # Non-blocking

    return response_v1  # User only sees v1 response

def async_process_v2(request):
    response_v2 = process_v2(request)

    # Compare results
    if response_v1 != response_v2:
        log_difference(request, response_v1, response_v2)
```

### Use Cases

- **Test performance:** Does v2 handle production load?
- **Compare results:** Does v2 produce same output as v1?
- **Validate refactoring:** Ensure behavior unchanged

**Example:** GitHub used shadow deployment for MySQL → Redis migration

### Pros/Cons

**Pros:**
- Zero user impact (only v1 returns results)
- Test with production traffic
- Validate performance and correctness

**Cons:**
- 2x resource usage (both versions process requests)
- Only works for side-effect-free operations (reads, not writes)

## Deployment Strategy Comparison

| **Strategy**      | **Risk**   | **Rollback Speed** | **Cost**    | **Complexity** | **Use Case**                          |
|-------------------|------------|--------------------|-------------|----------------|---------------------------------------|
| Big Bang          | Very High  | Slow               | Low         | Low            | Avoid in production                   |
| Rolling           | Medium     | Slow               | Low         | Low            | Stateless apps, compatible versions   |
| Blue-Green        | Low        | Instant            | High (2x)   | Medium         | Critical apps, instant rollback needed|
| Canary            | Very Low   | Fast               | Low-Medium  | High           | Large user base, gradual validation   |
| Feature Flags     | Very Low   | Instant            | Low         | High           | Decouple deploy from release          |

## Database Migrations & Deployments

### The Problem

```
v1 code expects: users(id, name)
v2 code expects: users(id, first_name, last_name)

During rolling deploy, both versions run simultaneously!
```

### Solution: Multi-Step Migration

**Step 1: Add new columns (nullable)**
```sql
ALTER TABLE users ADD COLUMN first_name VARCHAR(100);
ALTER TABLE users ADD COLUMN last_name VARCHAR(100);
```

**Step 2: Deploy v1.5 (dual-write)**
```python
def save_user(user_id, name, first_name, last_name):
    db.execute("""
        UPDATE users
        SET name = ?, first_name = ?, last_name = ?
        WHERE id = ?
    """, name, first_name, last_name, user_id)
```

**Step 3: Backfill data**
```python
# Batch job
users = db.query("SELECT id, name FROM users WHERE first_name IS NULL")
for user in users:
    parts = user['name'].split(' ', 1)
    first = parts[0]
    last = parts[1] if len(parts) > 1 else ''
    db.execute("UPDATE users SET first_name = ?, last_name = ? WHERE id = ?",
               first, last, user['id'])
```

**Step 4: Deploy v2 (only use new columns)**

**Step 5: Drop old column (separate deployment)**
```sql
ALTER TABLE users DROP COLUMN name;
```

## Monitoring During Deployments

### Key Metrics to Watch

```python
# Error rate
errors_v2 / requests_v2 vs errors_v1 / requests_v1

# Latency
p95_latency_v2 vs p95_latency_v1

# Success rate
(requests_v2 - errors_v2) / requests_v2

# Resource usage
cpu_v2, memory_v2 vs cpu_v1, memory_v1
```

### Automated Rollback Triggers

```python
def should_rollback(v1_metrics, v2_metrics):
    # Error rate 50% higher
    if v2_metrics['error_rate'] > v1_metrics['error_rate'] * 1.5:
        return True

    # Latency 2x higher
    if v2_metrics['p95_latency'] > v1_metrics['p95_latency'] * 2:
        return True

    # CPU usage excessive
    if v2_metrics['cpu'] > 80:
        return True

    return False
```

### Deployment Health Dashboard

```
┌─────────────────────────────────────────┐
│ Canary Deployment: v2.3.0               │
├─────────────────────────────────────────┤
│ Traffic Split:  v1: 90%   v2: 10%       │
│ Stage: 2/4 (10 minutes remaining)       │
├─────────────────────────────────────────┤
│           v1        v2       Delta      │
│ Requests  900/s     100/s    +0%        │
│ Errors    0.5%      0.6%     +20% ⚠️    │
│ P95 Lat   250ms     245ms    -2% ✓      │
│ CPU       45%       47%      +4% ✓      │
├─────────────────────────────────────────┤
│ Status: HEALTHY ✓                       │
│ Next: Increase to 25% in 10min          │
└─────────────────────────────────────────┘
```

## Real-World Examples

### **Netflix**: Canary + Feature Flags
- Canary to 1% of users
- Monitor error rates, latency
- Gradual increase: 5% → 10% → 25% → 50% → 100%
- Feature flags for A/B testing

### **Facebook**: Feature Flags at Scale
- Every change behind feature flag
- Gradual rollout: employees → 1% → 5% → 50% → 100%
- Instant disable if issues

### **Amazon**: Blue-Green for Critical Services
- Payment services use blue-green
- Instant rollback capability
- Test green environment before switch

### **Google**: Gradual Rollouts
- New features rolled out over weeks
- Data centers updated one-by-one
- Heavy monitoring and automated rollbacks

## Common Pitfalls

1. **No rollback plan**
   - ❌ Deploy and hope
   - ✅ Test rollback before deploying

2. **Incompatible database changes**
   - ❌ Change column type during rolling deploy
   - ✅ Multi-step migration (expand-contract)

3. **No monitoring**
   - ❌ Deploy and assume success
   - ✅ Watch metrics, automate rollback

4. **Feature flag sprawl**
   - ❌ 500 active feature flags
   - ✅ Clean up flags after rollout

5. **Testing only happy path**
   - ❌ Test works when flag is on
   - ✅ Test all flag combinations

## Deployment Checklist

**Before deploying:**
- [ ] Backward-compatible database changes?
- [ ] Rollback plan tested?
- [ ] Monitoring dashboards ready?
- [ ] Rollback triggers configured?
- [ ] Off-hours deployment (if risky)?

**During deployment:**
- [ ] Monitor error rates
- [ ] Monitor latency
- [ ] Monitor resource usage
- [ ] Check logs for errors
- [ ] Verify functionality in production

**After deployment:**
- [ ] All metrics normal?
- [ ] Feature flags cleaned up (if temporary)?
- [ ] Old infrastructure decommissioned (blue-green)?
- [ ] Post-mortem if issues occurred

## Key Takeaways

1. **Never big bang** in production (use rolling at minimum)
2. **Canary = safest** for large user bases (gradual + automated)
3. **Feature flags decouple deploy from release** (deploy anytime, enable selectively)
4. **Blue-green = instant rollback** (but 2x cost)
5. **Monitor everything** during deployments (automate rollback)
6. **Database migrations require multi-step** process (expand-contract pattern)
7. **Start conservative** (5% canary), increase gradually

**Senior engineer principle:** The goal isn't just to deploy. It's to deploy safely, quickly, and with confidence.
