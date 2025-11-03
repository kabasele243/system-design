# System Design Study Repository

A comprehensive 16-week curriculum to master system design interviews and build scalable systems.

## 📚 Study Plan

See [study-plan.md](study-plan.md) for the complete 16-week roadmap.

## 🗂️ Folder Structure

### Part 1: Foundation (Weeks 1-4)
- [`week-01-core-concepts/`](week-01-core-concepts/) - Scalability, Reliability, Availability, Latency vs Throughput
- [`week-02-networking-apis/`](week-02-networking-apis/) - DNS, Load Balancers, Proxies, REST APIs
- [`week-03-cap-consistency/`](week-03-cap-consistency/) - CAP Theorem, Consistency Models, ACID vs BASE
- [`week-04-security/`](week-04-security/) - Authentication, Authorization, API Security, Encryption

### Part 2: Building Blocks (Weeks 5-9)
- [`week-05-databases/`](week-05-databases/) - SQL vs NoSQL, Indexing, Bloom Filters
- [`week-06-scaling-caching/`](week-06-scaling-caching/) - Replication, Sharding, Consistent Hashing, Cache Strategies
- [`week-07-async-systems/`](week-07-async-systems/) - Message Queues, Pub/Sub, Kafka, Event-Driven Architecture
- [`week-08-communication/`](week-08-communication/) - REST vs gRPC, WebSockets, GraphQL, API Gateways
- [`week-09-observability/`](week-09-observability/) - Logging, Metrics, Tracing, Circuit Breakers, Chaos Engineering

### Part 3: Practice Problems (Weeks 10-14)
- [`week-10-url-shortener/`](week-10-url-shortener/) - Design TinyURL/bit.ly (Read-heavy system)
- [`week-11-social-feed/`](week-11-social-feed/) - Design Twitter/Instagram Feed (Fan-out problem)
- [`week-12-media-storage/`](week-12-media-storage/) - Design Netflix/YouTube (CDN, Video streaming)
- [`week-13-realtime-geospatial/`](week-13-realtime-geospatial/) - Design Chat App & Uber/Lyft
- [`week-14-tradeoffs/`](week-14-tradeoffs/) - Deep dive into trade-offs and cost analysis

### Part 4: Interview Prep (Weeks 15-16)
- [`week-15-mock-interviews/`](week-15-mock-interviews/) - Mock interview practice and refinement
- [`week-16-final-prep/`](week-16-final-prep/) - Final polish, cheat sheets, confidence building

### Resources
- [`resources/cheat-sheets/`](resources/cheat-sheets/) - Quick reference guides
  - [5-Step Framework](resources/cheat-sheets/5-step-framework.md)
  - [Back-of-Envelope Numbers](resources/cheat-sheets/back-of-envelope-numbers.md)
  - [CAP Theorem](resources/cheat-sheets/cap-theorem.md)
  - [SQL vs NoSQL](resources/cheat-sheets/sql-vs-nosql.md)
- [`resources/books/`](resources/books/) - Reading notes
- [`resources/blogs/`](resources/blogs/) - Engineering blog summaries
- [`resources/practice-platforms/`](resources/practice-platforms/) - Mock interview platforms

## 🎯 How to Use This Repository

### Week-by-Week Approach:
1. Read the topic markdown files in each week's folder
2. Take notes in the provided sections
3. Complete practice problems
4. Do the weekly review self-assessment
5. Move to next week only after completing checklist

### System Design Problems Approach:
Each problem (weeks 10-13) has 4 files:
1. `requirements.md` - Clarify requirements & estimate scale
2. `high-level-design.md` - Draw architecture & APIs
3. `deep-dive.md` - Solve key challenges & trade-offs
4. `failure-scenarios.md` - Discuss failures & monitoring

Follow the **5-Step Framework** for every problem.

## 📖 Key Resources

### Books (in order of priority):
1. **"Designing Data-Intensive Applications"** by Martin Kleppmann (chapters 1-6, 11)
2. **"System Design Interview"** Vol 1 & 2 by Alex Xu
3. **"Google SRE Book"** (free online, chapters 4, 6, 22)
4. **"Release It!"** by Michael Nygard

### Online Courses:
- ByteByteGo (Alex Xu)
- Grokking the System Design Interview (Educative)
- System Design Primer (GitHub - free)

### Engineering Blogs:
Read 2 articles per week from:
- High Scalability
- Netflix Tech Blog
- Uber Engineering
- Meta Engineering
- Discord Engineering

## ✅ Success Metrics

After 16 weeks, you should be able to:
1. ✅ Design any classic system (Twitter, Netflix, Uber, Chat) in 45 minutes
2. ✅ Estimate scale (QPS, storage, bandwidth) within 2 minutes
3. ✅ Justify every design choice with trade-offs
4. ✅ Identify and explain 3+ failure scenarios for any design
5. ✅ Switch between SQL/NoSQL, push/pull, cache strategies based on requirements
6. ✅ Communicate clearly and drive the conversation in an interview

## 🎓 Study Tips

1. **Be consistent:** Study 1-2 hours daily rather than cramming
2. **Practice out loud:** Explain designs as if in an interview
3. **Draw diagrams:** Visual thinking is crucial
4. **Find a study partner:** Practice mock interviews together
5. **Read engineering blogs:** Learn how real companies solve problems
6. **Build projects:** Implement concepts hands-on when possible

## 🤝 Teaching Style

This repository follows an incremental teaching approach (see [prompt.md](prompt.md)):
- Start with requirements and real-world analogies
- Build one component at a time
- Interactive learning with questions
- Patient, mentor-style explanations
- Focus on understanding "why" not just "what"

## 📝 Progress Tracking

Use the weekly review checklists to track your progress. Mark items complete as you master them.

## 🚀 Getting Started

1. Start with Week 1: [week-01-core-concepts/](week-01-core-concepts/)
2. Read the [5-Step Framework](resources/cheat-sheets/5-step-framework.md)
3. Memorize the [Back-of-Envelope Numbers](resources/cheat-sheets/back-of-envelope-numbers.md)
4. Begin your journey!

---

**Remember:** System design is about trade-offs, not perfect solutions. There's no single "right" answer. Your goal is to show structured thinking, knowledge of building blocks, and ability to reason about scale and failure.

Good luck! 🎯
