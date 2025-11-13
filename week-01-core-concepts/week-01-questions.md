# Week 1: Core Concepts - Interview Questions

## Instructions
These are realistic senior-level system design interview questions. Approach each scenario as you would in an actual interview:
- State your assumptions clearly
- Show your calculations and reasoning
- Discuss trade-offs
- Consider real-world constraints

---

## Question 1: E-Commerce Platform Scaling Decision

**Scenario:**
You're the lead architect at an e-commerce company that currently runs on a single powerful database server (64 cores, 512 GB RAM, enterprise SSD). The business is growing rapidly:
- Current: 50,000 daily active users
- Projected: 2 million daily active users within 6 months
- Peak traffic during sales events can be 5x normal load
- Current database response time: 50ms average, 200ms at peak
- Budget constraint: Need to optimize costs while maintaining performance

**Questions:**
1. Would you recommend vertical or horizontal scaling for the database layer? Justify your choice with specific reasoning about the trade-offs.

2. The CTO suggests "let's just upgrade to a bigger server with 1 TB RAM." What are the risks and limitations of this approach for the next 2 years?

3. If you choose horizontal scaling, what are the top 3 technical challenges you'll need to solve, and how would you address each one?

4. Design a hybrid approach: Which components should scale vertically and which should scale horizontally? Why?

---

## Question 2: Payment Processing Reliability vs Availability

**Scenario:**
You're designing a payment processing system for a ride-sharing app similar to Uber. The system handles:
- 10 million rides per day
- Average transaction value: $25
- Total daily transaction volume: $250 million
- Payment processing happens in multiple data centers (US East, US West, Europe, Asia)

During a recent incident, the network connection between US East and US West data centers was severed for 45 minutes. The engineering team is debating two approaches:

**Approach A (High Reliability):**
- Reject all payment requests if we can't verify with the primary database
- Ensure no double charges or incorrect amounts
- May result in failed rides during network partitions

**Approach B (High Availability):**
- Process payments locally and sync later
- Riders can always complete trips
- Risk of temporary inconsistencies (double charges that need to be refunded)

**Questions:**
1. Which approach would you recommend and why? Consider both user experience and business risk.

2. Calculate the business impact: If you choose Approach A and reject payments during network partitions, what's the revenue loss during a 45-minute outage? Show your calculations.

3. If you choose Approach B and 0.1% of transactions during the partition result in double charges that require manual refunds, what's the operational cost? (Assume $5 cost per manual refund processing)

4. Can you design a hybrid solution that provides high reliability for payments but high availability for other features (like viewing ride history)? Describe the architecture.

---

## Question 3: Real-Time Gaming Latency Optimization

**Scenario:**
You're the infrastructure lead for a competitive online multiplayer game. Current metrics:
- 500,000 concurrent players globally
- Players distributed: 40% North America, 35% Europe, 20% Asia, 5% South America
- Current average latency: 120ms
- Player complaints: "The game feels laggy"
- Competitive players expect: <50ms latency
- Current setup: Single data center in US East (Virginia)

**Questions:**
1. What is the primary cause of high latency for European and Asian players? Explain the physics/network fundamentals.

2. Design a solution to achieve <50ms latency for 90% of your players. What infrastructure changes would you make?

3. Calculate the trade-offs:
   - Current: 1 data center, $50,000/month operational cost
   - Proposed: 4 data centers (US, EU, Asia, SA), estimated $180,000/month
   - Is this investment justified? What metrics would prove ROI?

4. A stakeholder suggests: "Can't we just optimize our code to reduce latency?" Explain why code optimization alone won't solve this problem and what the realistic limits are.

---

## Question 4: Video Streaming Throughput vs Latency

**Scenario:**
You're architecting a video streaming platform that will offer two distinct services:

**Service A - Live Sports Streaming:**
- 5 million concurrent viewers during major events
- Users expect <3 second delay from live action
- Video quality: 1080p, ~5 Mbps bitrate
- Critical: Everyone sees the same moment at nearly the same time

**Service B - On-Demand Movie Streaming:**
- 20 million subscribers
- 2 million concurrent viewers average
- Video quality: 4K, ~25 Mbps bitrate
- Users accept 10-15 second buffering before playback

**Questions:**
1. For each service, identify whether latency or throughput is the primary concern. Justify your answer.

2. Calculate the bandwidth requirements:
   - Live Sports: 5M viewers × 5 Mbps = ? TB/second
   - On-Demand: 2M viewers × 25 Mbps = ? TB/second
   - Show your work and convert to appropriate units.

3. Why is achieving low latency AND high throughput for live sports streaming significantly harder (and more expensive) than on-demand streaming? Explain the technical reasons.

4. Design different CDN strategies for each service. How would your caching, edge server placement, and network architecture differ?

---

## Question 5: Social Media Platform Availability SLA

**Scenario:**
You're the VP of Engineering at a social media platform with 100 million daily active users. You need to present availability SLA recommendations to the board.

**Current Performance:**
- Revenue: $500 million/year (mostly from ads)
- Ads shown only when users are active on platform
- Average revenue per user per hour: $0.57
- Current availability: 99.5% (43.8 hours downtime/year)
- Infrastructure cost: $20 million/year

**Three Proposals:**

| Option | Availability | Downtime/Year | Infrastructure Cost | Additional Ops Cost |
|--------|--------------|---------------|---------------------|---------------------|
| Current | 99.5% | 43.8 hours | $20M | $2M |
| Option A | 99.9% | 8.77 hours | $35M | $4M |
| Option B | 99.99% | 52.6 minutes | $60M | $8M |

**Questions:**
1. Calculate the revenue loss from downtime for each option:
   - Current: 43.8 hours × 100M users × $0.57/hour = ?
   - Option A: 8.77 hours × 100M users × $0.57/hour = ?
   - Option B: 52.6 minutes × 100M users × $0.57/hour = ?

2. Calculate the net financial impact (revenue saved from reduced downtime minus additional costs) for each upgrade option.

3. Based purely on ROI, which option makes business sense? Show your calculations.

4. Beyond pure ROI, what other factors should influence this decision? (Consider: user trust, competitive pressure, brand reputation, investor expectations)

5. Would your recommendation change if this were a banking app instead of social media? Why or why not?

---

## Question 6: Instagram-Scale Storage Estimation

**Scenario:**
You're interviewing at Instagram as a senior infrastructure engineer. The interviewer asks you to estimate storage requirements for the next 3 years.

**Given Information:**
- Current: 1.5 billion monthly active users
- Growth: 15% per year
- User behavior:
  - Average user posts 2 photos per week
  - Average user posts 1 story per day (24-hour expiration)
  - Average user shares 3 videos per month
- File sizes:
  - Photo: 500 KB (original + 4 optimized versions = 2 MB total)
  - Story: 300 KB (deleted after 24 hours)
  - Video: 50 MB average

**Questions:**
1. Calculate photo storage required for Year 1:
   - Show your work step by step
   - Use 100,000 seconds/day for time conversions
   - Round appropriately for mental math

2. Calculate video storage required for Year 1:
   - Include monthly active users calculation
   - Show unit conversions clearly

3. Should you count story storage in your long-term capacity planning? Why or why not? If yes, how much storage is needed for 24 hours of stories?

4. Calculate total storage needed for 3 years, accounting for 15% annual user growth. Show the year-by-year breakdown.

5. If storage costs $0.02 per GB per month, what's the monthly and annual storage cost for Year 3? Is this realistic? What optimizations would you recommend?

---

## Question 7: Twitter QPS and Database Sharding

**Scenario:**
You're designing the tweet posting system for Twitter. Current scale:
- 500 million daily active users
- Average user posts 2 tweets per day
- Average user reads timeline 20 times per day
- Peak traffic (during major events): 3x average

**Questions:**
1. Calculate write QPS (tweets being posted):
   - Average write QPS = ?
   - Peak write QPS = ?
   - Show your calculations using 100,000 seconds/day

2. Calculate read QPS (timeline loads):
   - Average read QPS = ?
   - Peak read QPS = ?

3. If a single database server can handle 10,000 QPS, how many database servers do you need?
   - For writes?
   - For reads?
   - What does this tell you about read vs write patterns?

4. Design a database sharding strategy for tweets:
   - How would you partition the data? (By user? By time? By tweet ID?)
   - What are the trade-offs of each approach?
   - How would this affect timeline loading performance?

---

## Question 8: Netflix Bandwidth and CDN Strategy

**Scenario:**
You're the principal engineer at Netflix, planning bandwidth requirements for global expansion.

**Current Metrics:**
- 250 million subscribers worldwide
- Average: 2 hours of streaming per subscriber per day
- Video bitrates:
  - SD (480p): 3 Mbps - 30% of streams
  - HD (1080p): 5 Mbps - 50% of streams
  - 4K: 25 Mbps - 20% of streams
- Geographic distribution:
  - North America: 40%
  - Europe: 35%
  - Asia: 15%
  - Others: 10%

**Questions:**
1. Calculate weighted average bitrate across all streams:
   - (0.30 × 3) + (0.50 × 5) + (0.20 × 25) = ? Mbps

2. Calculate total bandwidth required:
   - Concurrent viewers during peak (assume 30% of subscribers): ?
   - Peak bandwidth = Concurrent viewers × Average bitrate = ? TB/second
   - Show unit conversions

3. CDN vs Origin servers: If 95% of requests are served from CDN edge servers and only 5% hit origin servers, what's the load on origin servers? Why is this CDN hit rate so critical?

4. Cost analysis:
   - CDN cost: $0.02 per GB transferred
   - Calculate daily data transfer in PB
   - Calculate monthly CDN cost
   - What optimizations would you implement to reduce costs while maintaining quality?

---

## Question 9: Uber Driver Location Updates - Latency vs Throughput Trade-offs

**Scenario:**
You're designing the real-time location tracking system for Uber drivers.

**Requirements:**
- 5 million active drivers during peak hours
- Each driver sends location update every 3 seconds
- Each location update: 100 bytes (lat, long, timestamp, driver_id, status)
- Riders should see driver location with <5 second staleness
- System must handle 2x surge during busy hours

**Questions:**
1. Calculate location update write QPS:
   - 5M drivers ÷ 3 seconds = ? updates/second
   - Peak (2x surge) = ?
   - Show your calculation

2. Calculate bandwidth for location updates:
   - Updates/second × 100 bytes = ? MB/second
   - Is this significant compared to other system components?

3. Trade-off analysis: What if you reduced update frequency to every 10 seconds instead of 3 seconds?
   - How much would this reduce write QPS and bandwidth?
   - What's the impact on user experience?
   - Would you recommend this change? Why or why not?

4. A stakeholder asks: "Why can't we just show the exact real-time location like a GPS app?" Explain the difference between:
   - Local GPS tracking (phone showing your own location)
   - Distributed tracking (server showing driver location to rider)
   - What are the fundamental latency limits?

---

## Question 10: Multi-Region System Design - Reliability and Availability Balance

**Scenario:**
You're designing a global SaaS platform for project management (similar to Asana/Jira). The platform needs to serve customers in:
- North America: 1M users
- Europe: 800K users
- Asia: 500K users

**Current Setup:**
- Single region (US East)
- Availability: 99.9%
- European users complain: "The app is slow" (300ms latency)
- Asian users: "Even slower" (450ms latency)

**Two Proposed Solutions:**

**Solution A: Multi-region Active-Active**
- Deploy to 3 regions (US, EU, Asia)
- Each region has full copy of data
- Users routed to nearest region
- Data syncs across regions
- Cost: 3x current infrastructure

**Solution B: Multi-region Read Replicas**
- Primary database in US (all writes go here)
- Read replicas in EU and Asia
- Users read from nearest replica
- Writes still go to US primary
- Cost: 1.8x current infrastructure

**Questions:**
1. Compare latency for Solution A vs Solution B:
   - European user updates a task
   - European user reads task list
   - Which solution provides better latency for each operation?

2. Compare reliability/availability:
   - What happens if the US region goes down in Solution A?
   - What happens if the US region goes down in Solution B?
   - Calculate availability if each region has 99.9% availability

3. Consistency considerations:
   - In Solution A, a US user and EU user update the same task simultaneously. What problem occurs?
   - How would you resolve this conflict?
   - What's the trade-off?

4. Cost-benefit analysis:
   - Solution A: 3x cost, what benefits justify this?
   - Solution B: 1.8x cost, what are the limitations?
   - For a project management tool, which solution would you recommend and why?

---

## Evaluation Rubric

When reviewing your answers, assess yourself on:

**Technical Accuracy (40%)**
- Correct calculations and units
- Proper understanding of concepts
- Realistic numbers and estimates

**Trade-off Analysis (30%)**
- Identifying key trade-offs
- Comparing alternatives
- Justifying recommendations

**Business Acumen (20%)**
- Considering costs and ROI
- Understanding user impact
- Balancing technical and business needs

**Communication (10%)**
- Clear explanations
- Showing your work
- Structured reasoning

---

## Answer Key - Available Separately
Detailed solutions with step-by-step calculations, trade-off analysis, and architectural diagrams are provided in a separate file. Attempt all questions before reviewing answers.
