# System Design Practice Problems

**Total Problems:** 42 (across 10 categories)
**Goal:** Complete all problems after finishing Weeks 1-9 foundational learning

---

## How to Use This Folder

1. **After Week 9:** Start practicing with these problems
2. **Time yourself:** Aim for 45 minutes per problem (interview pace)
3. **Use the 5-step framework:**
   - Step 1: Clarify requirements (5 min)
   - Step 2: Estimate scale (5 min)
   - Step 3: High-level design (10 min)
   - Step 4: Deep dive (20 min)
   - Step 5: Discuss trade-offs (5 min)
4. **Track progress:** Mark problems as completed in each category file

---

## Problem Categories

| Category | File | Problems | Completed |
|----------|------|----------|-----------|
| 1. Read-Heavy Systems | [01-read-heavy-systems.md](01-read-heavy-systems.md) | 4 | ___/4 |
| 2. Write-Heavy Systems | [02-write-heavy-systems.md](02-write-heavy-systems.md) | 4 | ___/4 |
| 3. User-Facing & Social | [03-user-facing-social.md](03-user-facing-social.md) | 5 | ___/5 |
| 4. Search, Messaging, Delivery | [04-search-messaging-delivery.md](04-search-messaging-delivery.md) | 5 | ___/5 |
| 5. Streaming, Sync, Media | [05-streaming-sync-media.md](05-streaming-sync-media.md) | 5 | ___/5 |
| 6. Strong Consistency | [06-strong-consistency.md](06-strong-consistency.md) | 4 | ___/4 |
| 7. Infrastructure & Storage | [07-infrastructure-storage.md](07-infrastructure-storage.md) | 5 | ___/5 |
| 8. Realtime & Event-Driven | [08-realtime-event-driven.md](08-realtime-event-driven.md) | 5 | ___/5 |
| 9. Scheduler & Notification | [09-scheduler-notification.md](09-scheduler-notification.md) | 3 | ___/3 |
| 10. Trie / Proximity / Location | [10-trie-proximity-location.md](10-trie-proximity-location.md) | 2 | ___/2 |

**Total Progress:** ___/42 (___%)

---

## Recommended Learning Path

### Weeks 10-11: Easy Problems (Build Confidence)
- ✅ URL Shortener
- ✅ Rate Limiter
- ✅ Word Dictionary
- ✅ Recent Searches

### Weeks 12-13: Medium Problems (Apply Multiple Concepts)
- ✅ NewsFeed System
- ✅ Video Processing Pipeline
- ✅ Ticket Booking
- ✅ Online/Offline Indicator

### Week 14: Hard Problems (Complex Systems)
- ✅ Twitter/Facebook (full social platform)
- ✅ Uber/Lyft (location-based, real-time)
- ✅ E-Commerce Website (strong consistency + scale)

### Weeks 15-16: Mock Interviews
- Practice 5-7 random problems under timed conditions
- Get feedback from peers or mentors
- Review and refine your approach

---

## Key Patterns to Master

**By Category:**

1. **Read-Heavy:** Caching (Redis, CDN), Read replicas, Denormalization
2. **Write-Heavy:** Message queues (Kafka), Partitioning, Async processing
3. **Social Systems:** Fan-out (write vs read), Graph databases, Timeline generation
4. **Search:** Inverted index, Trie, Elasticsearch
5. **Media/Streaming:** Object storage (S3), Video encoding, CDN
6. **Strong Consistency:** ACID transactions, Locking, Idempotency
7. **Infrastructure:** Sharding, Replication, Consistent hashing
8. **Realtime:** WebSockets, Pub/Sub, Event-driven architecture
9. **Scheduler:** Cron jobs, Distributed locks, Task queues
10. **Location:** Geohashing, Quadtree, Proximity search

---

## Success Metrics

After completing all 42 problems, you should be able to:

- [ ] Design any system in 45 minutes
- [ ] Estimate scale (QPS, storage, bandwidth) in 2 minutes
- [ ] Justify every design choice with trade-offs
- [ ] Identify 3+ failure scenarios for any design
- [ ] Switch between SQL/NoSQL, push/pull, cache strategies based on requirements
- [ ] Communicate clearly and drive the conversation

---

## Additional Resources

- **System Design Primer:** https://github.com/donnemartin/system-design-primer
- **Engineering Blogs:** Netflix, Uber, Discord, Slack
- **Mock Interview Platforms:** Pramp, interviewing.io
- **YouTube:** System Design Interview, Gaurav Sen

---

**Last Updated:** [Date]
