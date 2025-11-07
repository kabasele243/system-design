# Learning Progress Tracker

Track all concepts you've learned to maintain a clear record of your system design knowledge.

---

## Progress Overview

**Started:** 2025-11-03
**Current Week:** Week 6
**Completion:** 21%

---

## Part 1: Foundation (Weeks 1-4)

### Week 1: Core Concepts & Scalability ✅

| Concept | Status | Date Completed | Confidence (1-5) | Notes |
|---------|--------|----------------|------------------|-------|
| Vertical Scalability | ✅ Mastered | 2025-11-03 | ★★★★★ | Coffee shop analogy - one super barista |
| Horizontal Scalability | ✅ Mastered | 2025-11-03 | ★★★★★ | Multiple locations - preferred for web-scale |
| Reliability vs Availability | ✅ Mastered | 2025-11-03 | ★★★★★ | ATM analogy - Rita vs Alex. Different features need different priorities |
| Latency vs Throughput | ✅ Mastered | 2025-11-03 | ★★★★★ | Airplane analogy - private jet vs big plane. Gaming needs low latency |
| Back-of-Envelope Calculations | ✅ Mastered | 2025-11-03 | ★★★★★ | 3 formulas: QPS, Storage, Bandwidth. Use 100k seconds/day |

**Weekly Review Completed:** ✅ Yes (2025-11-03) - Perfect score on all questions!
**Practice Problems Done:** 5/5

---

### Week 2: Networking & APIs ✅

| Concept | Status | Date Completed | Confidence (1-5) | Notes |
|---------|--------|----------------|------------------|-------|
| DNS Resolution | ✅ Mastered | 2025-11-03 | ★★★★★ | 3-level hierarchy, multi-level caching, TTL. Cache = performance! |
| L4 vs L7 Load Balancing | ✅ Mastered | 2025-11-03 | ★★★★★ | L4=fast/IP, L7=smart/HTTP. Health checks. Stateless+RoundRobin |
| Forward vs Reverse Proxy | ✅ Mastered | 2025-11-03 | ★★★★★ | Forward=client-side/VPN, Reverse=server-side/NGINX. API Gateway! |
| REST API Design | ✅ Mastered | 2025-11-03 | ★★★★★ | Resources+HTTP methods. Idempotency! Nested resources. Path vs Query |
| HTTP Methods & Status Codes | ✅ Mastered | 2025-11-03 | ★★★★★ | 401 vs 403! 2xx/4xx/5xx. Safe vs Idempotent. CORS. Rate limiting |

**Hands-on Practice:**
- [ ] Implemented round-robin load balancer
- [ ] Designed REST API for to-do app

**Weekly Review Completed:** ✅ Yes (2025-11-03) - Perfect score! Deep understanding of all concepts.

---

### Week 3: CAP Theorem & Consistency ✅

| Concept | Status | Date Completed | Confidence (1-5) | Notes |
|---------|--------|----------------|------------------|-------|
| CAP Theorem | ✅ Mastered | 2025-11-05 | ★★★★★ | C+A+P pick 2, but P required! Bank branch analogy. Premature optimization! |
| CP vs AP Systems | ✅ Mastered | 2025-11-05 | ★★★★★ | Configurable per-operation! MongoDB/Cassandra/DynamoDB examples |
| Consistency Spectrum | ✅ Mastered | 2025-11-05 | ★★★★★ | Strong→Bounded→Session→Prefix→Eventual. Spotify use cases! |
| ACID vs BASE | ✅ Mastered | 2025-11-05 | ★★★★★ | Atomicity/Isolation! Hybrid approach. Uber architecture example |

**Practice:**
- [x] Classified systems as CP or AP (Banking, Instagram, Uber, E-commerce, Spotify)
- [x] Chose consistency levels (Strong, Bounded, Session, Eventual) for Spotify features
- [x] Applied ACID vs BASE to ride-sharing app features

**Weekly Review Completed:** ✅ Yes (2025-11-05) - Perfect score! Excellent reasoning on all concepts.

---

### Week 4: Security Fundamentals ✅

| Concept | Status | Date Completed | Confidence (1-5) | Notes |
|---------|--------|----------------|------------------|-------|
| Authentication vs Authorization | ✅ Mastered | 2025-11-06 | ★★★★★ | Who vs What. MFA. RBAC vs ABAC. Coffee shop & Uber examples |
| OAuth 2.0 & JWT | ✅ Mastered | 2025-11-06 | ★★★★★ | 4 players! Hotel key card analogy. Signature prevents tampering |
| API Security | ✅ Mastered | 2025-11-06 | ★★★★★ | OWASP Top 10. SQL injection. Input validation. HTTPS enforcement |
| Encryption (at rest/in transit) | ✅ Mastered | 2025-11-06 | ★★★★★ | Symmetric vs Asymmetric. bcrypt for passwords! Salting prevents rainbow tables |
| Rate Limiting | ✅ Mastered | 2025-11-06 | ★★★★★ | 5 algorithms! Token Bucket=bursts. Sliding Window Counter=production |

**Practice:**
- [x] Designed healthcare security architecture (ABAC + OIDC + Multi-layer encryption + Audit logs)
- [x] Debugged OAuth flow (Implicit vs Authorization Code)
- [x] Compared password storage methods (plaintext vs SHA-256 vs bcrypt)
- [x] Designed rate limiting for Spotify API (Token Bucket + Sliding Window Counter)
- [x] Explained JWT tampering prevention (HMAC signature with secret key)

**Weekly Review Completed:** ✅ Yes (2025-11-06) - Outstanding! Production-grade security architecture thinking. 10/10 on most questions.

---

## Part 2: Building Blocks (Weeks 5-9)

### Week 5: Databases ✅

| Concept | Status | Date Completed | Confidence (1-5) | Notes |
|---------|--------|----------------|------------------|-------|
| SQL vs NoSQL Trade-offs | ✅ Mastered | 2025-11-06 | ★★★★★ | Decision tree! 4 NoSQL types. Twitter example. Polyglot persistence |
| B-tree Indexes | ✅ Mastered | 2025-11-06 | ★★★★★ | Composite indexes! Column order matters. Covering indexes. When NOT to index |
| Bloom Filters | ✅ Mastered | 2025-11-06 | ★★★★★ | Space-efficient! No false negatives. Username checks. 859 MB for 500M items |
| Database Schema Design | ✅ Mastered | 2025-11-06 | ★★★★★ | 1NF/2NF/3NF! Denormalization strategies. Hybrid approach. CDC pipeline |

**Practice:**
- [x] Designed Twitter database architecture (MySQL + Cassandra + Redis - scored 9/10)
- [x] Created composite indexes for YouTube queries (40/40 - perfect score!)
- [x] Designed LinkedIn Bloom Filter with cost analysis (30/30 - perfect mathematics!)
- [x] Normalized e-commerce schema to 3NF with CDC hybrid approach (30/30 - perfect!)
- [x] Designed complete Uber database architecture (50/50 - masterpiece with Kafka, Redis GEO, S2 cells!)

**Weekly Review Completed:** ✅ Yes (2025-11-06) - EXCEPTIONAL! 198/200 (99%). Staff/Principal Engineer level thinking on all 5 questions.

---

### Week 6: Scaling & Caching ⬜

| Concept | Status | Date Completed | Confidence (1-5) | Notes |
|---------|--------|----------------|------------------|-------|
| Primary-Secondary Replication | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |
| Database Sharding | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |
| Consistent Hashing | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |
| Cache Eviction Policies | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |
| Cache Strategies | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |

**Hands-on Practice:**
- [ ] Implemented consistent hashing
- [ ] Designed caching strategy for product catalog

**Weekly Review Completed:** ⬜ Yes / ⬜ No

---

### Week 7: Asynchronous Systems ⬜

| Concept | Status | Date Completed | Confidence (1-5) | Notes |
|---------|--------|----------------|------------------|-------|
| Message Queues | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |
| Pub/Sub Model | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |
| Kafka vs Traditional Queue | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |
| Event-Driven Architecture | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |
| CQRS Pattern | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |

**Practice:**
- [ ] Designed order processing system with queues

**Weekly Review Completed:** ⬜ Yes / ⬜ No

---

### Week 8: Communication & Real-Time ⬜

| Concept | Status | Date Completed | Confidence (1-5) | Notes |
|---------|--------|----------------|------------------|-------|
| REST vs gRPC | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |
| WebSockets | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |
| Server-Sent Events | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |
| GraphQL | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |
| API Gateway | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |

**Hands-on Practice:**
- [ ] Designed real-time notification system
- [ ] Implemented WebSocket server

**Weekly Review Completed:** ⬜ Yes / ⬜ No

---

### Week 9: Observability & Resilience ⬜

| Concept | Status | Date Completed | Confidence (1-5) | Notes |
|---------|--------|----------------|------------------|-------|
| Structured Logging | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |
| Metrics (RED/USE) | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |
| Distributed Tracing | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |
| SLIs/SLOs/SLAs | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |
| Circuit Breaker Pattern | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |
| Chaos Engineering | ⬜ / 🟡 / ✅ | | ☆☆☆☆☆ | |

**Hands-on Practice:**
- [ ] Designed alerting strategy
- [ ] Implemented circuit breaker

**Weekly Review Completed:** ⬜ Yes / ⬜ No

---

## Part 3: Practice Problems (Weeks 10-14)

### Week 10: URL Shortener ⬜

**Completed:** ⬜ Yes / ⬜ No
**Date:** ___________

**5-Step Framework:**
- [ ] Clarified requirements & estimated scale
- [ ] Designed high-level architecture
- [ ] Deep dive: URL generation algorithm
- [ ] Deep dive: SQL vs NoSQL choice
- [ ] Discussed failure scenarios & monitoring

**Key Learnings:**
-
-

**What I struggled with:**
-

**Time taken:** ___ minutes

---

### Week 11: Social Media Feed ⬜

**Completed:** ⬜ Yes / ⬜ No
**Date:** ___________

**5-Step Framework:**
- [ ] Clarified requirements & estimated scale
- [ ] Designed high-level architecture
- [ ] Deep dive: Fan-out on write vs read
- [ ] Deep dive: Celebrity problem solution
- [ ] Discussed failure scenarios & monitoring

**Key Learnings:**
-
-

**What I struggled with:**
-

**Time taken:** ___ minutes

---

### Week 12: Media Storage (Netflix/YouTube) ⬜

**Completed:** ⬜ Yes / ⬜ No
**Date:** ___________

**5-Step Framework:**
- [ ] Clarified requirements & estimated scale
- [ ] Designed high-level architecture
- [ ] Deep dive: CDN strategy
- [ ] Deep dive: Video processing pipeline
- [ ] Discussed failure scenarios & monitoring

**Key Learnings:**
-
-

**What I struggled with:**
-

**Time taken:** ___ minutes

---

### Week 13: Chat App & Uber ⬜

**Problem 1: Chat App**
**Completed:** ⬜ Yes / ⬜ No
**Date:** ___________

**5-Step Framework:**
- [ ] Clarified requirements & estimated scale
- [ ] Designed high-level architecture
- [ ] Deep dive: WebSocket implementation
- [ ] Deep dive: Message delivery guarantees
- [ ] Discussed failure scenarios & monitoring

**Problem 2: Uber/Lyft**
**Completed:** ⬜ Yes / ⬜ No
**Date:** ___________

**5-Step Framework:**
- [ ] Clarified requirements & estimated scale
- [ ] Designed high-level architecture
- [ ] Deep dive: Geospatial indexing
- [ ] Deep dive: Matching algorithm
- [ ] Discussed failure scenarios & monitoring

**Time taken (both):** ___ minutes

---

### Week 14: Trade-off Analysis ⬜

**Activities:**
- [ ] Created one-page summaries for all designs
- [ ] Compared CP vs AP systems designed
- [ ] Analyzed SQL vs NoSQL decisions made
- [ ] Completed cost analysis for 2-3 systems
- [ ] Listed failure modes for each design
- [ ] Read 2 engineering blog articles
- [ ] Completed 3 mock interviews

**Key Insights:**
-
-

---

## Part 4: Interview Prep (Weeks 15-16)

### Week 15: Mock Interviews ⬜

**Mock Interviews Completed:** ___/5-7

| Interview # | Date | Platform | Problem | Time | Feedback | Areas to Improve |
|-------------|------|----------|---------|------|----------|------------------|
| 1 | | | | | | |
| 2 | | | | | | |
| 3 | | | | | | |
| 4 | | | | | | |
| 5 | | | | | | |

**Additional Practice Problems:**
- [ ] Design Amazon (e-commerce)
- [ ] Design Ticketmaster
- [ ] Design Spotify
- [ ] Design Rate Limiter
- [ ] Design Web Crawler

---

### Week 16: Final Prep ⬜

**Activities:**
- [ ] Created one-page cheat sheet
- [ ] Reviewed all building blocks
- [ ] Completed 2-3 timed mock interviews (45 min)
- [ ] Practiced 3 random problems under time pressure

**Cheat Sheet Topics:**
- [ ] CAP theorem trade-offs
- [ ] SQL vs NoSQL decision tree
- [ ] Common design patterns
- [ ] Back-of-envelope numbers

**Final Confidence Check:** ☆☆☆☆☆ (1-5)

---

## Skills Mastery Summary

### Core Skills

| Skill | Proficiency (1-5) | Notes |
|-------|-------------------|-------|
| Clarifying requirements | ☆☆☆☆☆ | |
| Scale estimation (QPS, storage, bandwidth) | ☆☆☆☆☆ | |
| High-level architecture design | ☆☆☆☆☆ | |
| API design | ☆☆☆☆☆ | |
| Database selection (SQL vs NoSQL) | ☆☆☆☆☆ | |
| Caching strategies | ☆☆☆☆☆ | |
| Identifying bottlenecks | ☆☆☆☆☆ | |
| Discussing trade-offs | ☆☆☆☆☆ | |
| Failure scenario analysis | ☆☆☆☆☆ | |
| Communication & presentation | ☆☆☆☆☆ | |

---

## Concepts I Can Explain Confidently

List concepts you can teach to someone else:
1.
2.
3.
4.
5.

---

## Concepts That Still Need Work

List concepts you need to review:
1.
2.
3.

---

## Engineering Blogs Read

Track articles you've read and key takeaways:

| Date | Company | Topic | Link | Key Takeaways |
|------|---------|-------|------|---------------|
| | Netflix | | | |
| | Uber | | | |
| | Discord | | | |
| | Slack | | | |

**Total Articles Read:** ___

---

## Project Portfolio

List side projects or implementations:

1. **Project:** ___________
   - **Concepts Applied:**
   - **Link/Notes:**

2. **Project:** ___________
   - **Concepts Applied:**
   - **Link/Notes:**

---

## Success Metrics Checklist

After 16 weeks, can you:

- [ ] Design any classic system (Twitter, Netflix, Uber, Chat) in 45 minutes
- [ ] Estimate scale (QPS, storage, bandwidth) within 2 minutes
- [ ] Justify every design choice with trade-offs
- [ ] Identify and explain 3+ failure scenarios for any design
- [ ] Switch between SQL/NoSQL, push/pull, cache strategies based on requirements
- [ ] Communicate clearly and drive the conversation in an interview

---

## Reflections & Notes

### What's Working Well
-
-

### What Needs Improvement
-
-

### Next Steps
-
-

---

**Last Updated:** [Date]

**Next Review:** [Date]
