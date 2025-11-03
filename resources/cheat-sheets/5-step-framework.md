# The 5-Step System Design Framework

Use this framework for ANY system design problem.

## Step 1: Clarify Requirements (5-10 minutes)

### Functional Requirements
Ask: "What features does the system need?"
- What can users do?
- What operations are supported?
- Any specific constraints?

### Non-Functional Requirements
- Availability (99.9%? 99.99%?)
- Latency requirements (< 100ms? < 1s?)
- Consistency (strong or eventual?)
- Scalability needs
- Durability requirements

## Step 2: Estimate Scale (5 minutes)

### Key Numbers to Calculate:
1. **QPS (Queries Per Second)**
   ```
   DAU × actions per user per day ÷ 86400 seconds
   Peak QPS = Average QPS × 2
   ```

2. **Storage**
   ```
   Total items × size per item × time period
   ```

3. **Bandwidth**
   ```
   QPS × average object size
   ```

### Important: ALWAYS estimate scale!
- Justifies your design choices
- Shows you think about scalability
- Helps identify bottlenecks

## Step 3: High-Level Design (10-15 minutes)

### Draw the main components:
```
[Client] → [Load Balancer] → [Web Servers] → [App Servers]
                                                    ↓
                                             [Database]
                                                    ↓
                                                [Cache]
                                                    ↓
                                            [Message Queue]
```

### Define API endpoints:
- What are the key operations?
- RESTful or RPC?
- Request/Response format

### Choose databases:
- SQL or NoSQL?
- Why?

## Step 4: Deep Dive & Bottlenecks (15-20 minutes)

### Pick 1-2 key areas to explore:
- Algorithm (e.g., short URL generation)
- Data structure (e.g., social graph)
- Bottleneck (e.g., celebrity fan-out problem)
- Database design
- Caching strategy

### Discuss trade-offs:
- Option A vs Option B
- Pros and cons of each
- Why you chose one

## Step 5: Failure Scenarios & Monitoring (5 minutes)

### Ask yourself:
1. What happens when a component fails?
2. How do we detect failures?
3. How do we recover?

### Key metrics to track:
- Request rate (QPS)
- Error rate (5xx errors)
- Latency (P50, P99)
- Resource utilization (CPU, memory)

### Alerting:
- SLIs and SLOs
- When to alert?

## Time Management (45 minutes total)

| Step | Time | What to do |
|------|------|-----------|
| 1. Requirements | 5-10 min | Ask clarifying questions |
| 2. Scale | 5 min | Calculate QPS, storage, bandwidth |
| 3. High-Level | 10-15 min | Draw architecture, API, database |
| 4. Deep Dive | 15-20 min | Focus on 1-2 interesting problems |
| 5. Failures | 5 min | Discuss monitoring and failure handling |

## Pro Tips

1. **Ask before designing:** Don't assume requirements
2. **Think out loud:** Explain your reasoning
3. **Draw diagrams:** Visual helps interviewer follow
4. **Discuss trade-offs:** No perfect solution, only trade-offs
5. **Be humble:** "I would need to test/measure this"
6. **Drive the conversation:** You lead, don't wait to be asked

## Common Mistakes to Avoid

❌ Jumping to solution without clarifying requirements
❌ Skipping scale estimation
❌ Not discussing trade-offs
❌ Overcomplicating the design
❌ Not talking through your thought process
❌ Ignoring failure scenarios
❌ Not managing time well
