Part 1: The Foundation (Weeks 1-4)

Your goal here is to learn the fundamental vocabulary and core concepts. You can't build a house without knowing what bricks, mortar, and beams are.

Week 1: Core Concepts & Scalability

Topics:

Scalability: What is the difference between Vertical Scalability (scaling up) and Horizontal Scalability (scaling out)? Why is horizontal scaling preferred for web-scale systems?

Reliability & Availability: What do "99.9%" (three nines) vs. "99.999%" (five nines) availability mean in real time? What is the difference between reliability (no data corruption) and availability (system is responsive)?

Performance: Understand Latency vs. Throughput.

Back-of-envelope Calculations: Learn to estimate QPS (Queries Per Second), storage needs, and bandwidth.

Reading: Start with "Grokking the System Design Interview" to get a high-level overview.

Practice: Calculate how much storage is needed for 1 billion users posting 2 photos per week for a year.

Weekly Review: Test yourself on the differences between latency, throughput, availability, and reliability.

Week 2: Networking, Load Balancing & API Fundamentals

Topics:

DNS (Domain Name System): How does a URL like google.com get translated to an IP address?

Load Balancers: What are they for? Understand the difference between L4 (Transport Layer, e.g., TCP) and L7 (Application Layer, e.g., HTTP) load balancing.

Proxies: What is a Forward Proxy vs. a Reverse Proxy? (An API Gateway is a type of reverse proxy).

REST API Design: Understand HTTP methods (GET, POST, PUT, DELETE), status codes, and RESTful principles.

Reading: "Designing Data-Intensive Applications" (DDIA), Chapter 1.

Practice: Design a simple REST API for a to-do list app (CRUD operations). Implement a basic round-robin load balancer in your preferred language.

Weekly Review: Explain the request flow from typing a URL to receiving a response.

Week 3: The CAP Theorem & Consistency

Topics:

CAP Theorem: Understand the three components: Consistency, Availability, and Partition Tolerance. Why must a distributed system choose between C and A during a partition?

Consistency Models: What is the difference between Strong Consistency and Eventual Consistency?

ACID vs. BASE: Understand transactional guarantees in databases.

Reading: DDIA, Part II Introduction. This is a dense but critical topic.

Practice: For 3 real-world systems (bank transactions, social media likes, shopping cart), decide if they need CP or AP and explain why.

Weekly Review: Draw diagrams showing how a partition affects a CP vs. AP system.

Week 4: Security Fundamentals

Topics:

Authentication vs. Authorization: What's the difference? Understand OAuth 2.0, JWT tokens.

API Security: Rate limiting, API keys, CORS, SQL injection prevention, XSS protection.

Encryption: Understand encryption at rest vs. in transit (TLS/SSL).

DDoS Protection: Basic mitigation strategies.

Reading: OWASP Top 10 (skim for awareness), focus on A01, A02, A03, A07.

Practice: Design an authentication flow for a web application. Implement a simple rate limiter (e.g., token bucket algorithm).

Weekly Review: List 5 security vulnerabilities and their mitigations.

Part 2: The Building Blocks (Weeks 5-9)

Now you'll learn the "Lego bricks" you'll use to build your systems.

Week 5: Databases (The Heart of the System)

Topics:

SQL vs. NoSQL: This is the classic trade-off. When would you use a relational database (like PostgreSQL) vs. a NoSQL database (like MongoDB, Cassandra, or DynamoDB)? Think about data structure, transaction needs (ACID), and scaling patterns.

Database Indexing: How do B-tree indexes work? Why do they speed up reads but slow down writes?

Bloom Filters: Space-efficient probabilistic data structures for membership testing.

Reading: DDIA, Chapters 2 & 3.

Practice: Implement an LRU cache in your preferred language. Design a database schema for an e-commerce platform.

Weekly Review: Compare when to use SQL vs. NoSQL for 5 different use cases.

Week 6: Database Scaling, Caching & Data Structures

Topics:

Database Replication: Understand Primary-Secondary (Master-Slave) replication. How does it improve read performance and availability?

Database Sharding (Partitioning): How do you split data across multiple databases when it's too big for one?

Consistent Hashing: How to distribute data across nodes and handle node additions/removals gracefully.

Caching: What is a cache? Understand a basic key-value cache (like Redis or Memcached). What are common cache eviction policies (LRU, LFU)?

Cache Strategies: Cache-aside, write-through, write-back, read-through.

Reading: DDIA, Chapters 5 & 6.

Practice: Implement consistent hashing. Design a caching strategy for an e-commerce product catalog.

Weekly Review: Explain the trade-offs between different replication strategies and cache eviction policies.

Week 7: Asynchronous Systems (Decoupling)

Topics:

Message Queues: Why use a queue (like RabbitMQ or SQS)? How do they enable asynchronous processing and make your system more resilient to failure?

Pub/Sub Model: What is the Publish-Subscribe pattern? How does it differ from a simple queue?

Kafka: Understand what makes Kafka (a distributed log) different from a traditional message queue.

Event-Driven Architecture: Understanding event sourcing and CQRS patterns.

Reading: DDIA, Chapter 11.

Practice: Design an order processing system using message queues. Explain when to use queues vs. Kafka.

Weekly Review: Draw the architecture of an event-driven system with multiple consumers.

Week 8: Advanced Communication & Real-Time Systems

Topics:

REST vs. gRPC: What are the pros and cons of each?

WebSockets vs. Long Polling vs. Server-Sent Events: How do you handle real-time communication (e.g., for a chat app)?

GraphQL: When is it better than REST?

API Gateways: Why use one? (Handles rate limiting, auth, routing, API composition).

Reading: Engineering blogs on WebSocket implementations from Slack or Discord.

Practice: Design a real-time notification system. Implement a simple WebSocket server.

Weekly Review: Compare REST, gRPC, and GraphQL for 3 different scenarios.

Week 9: Monitoring, Observability & Resilience

Topics:

Logging: Structured logging, centralized log aggregation (ELK stack, Splunk).

Metrics & Monitoring: Prometheus, Grafana, key metrics (RED: Rate, Errors, Duration; USE: Utilization, Saturation, Errors).

Distributed Tracing: Understanding OpenTelemetry, Jaeger, Zipkin.

Alerting: SLIs, SLOs, SLAs, and error budgets.

Resilience Patterns: Circuit breakers, retries with exponential backoff, bulkheads, timeouts.

Chaos Engineering: Introduction to deliberately breaking systems to test resilience.

Reading: Google SRE Book (Chapter 4 on SLIs/SLOs), "Release It!" by Michael Nygard (patterns section).

Practice: Design an alerting strategy for an e-commerce site. Implement a circuit breaker pattern.

Weekly Review: Define SLIs and SLOs for a web application with 99.9% availability target.

Part 3: The Practice (Weeks 10-14)

This is where you put it all together. The key is to practice out loud or on a whiteboard.

The 5-Step Framework for Any Problem:

Clarify Requirements: Ask questions. What are the functional (e.g., "post a message") and non-functional (e.g., "must be highly available," "low latency reads") requirements?

Estimate Scale: How many users? How many requests per second? How much storage? How much bandwidth? (This justifies your design choices). Practice back-of-envelope calculations.

High-Level Design: Draw the main components (Load Balancer, Web Servers, App Servers, Databases, Caches, Message Queues).

Deep Dive & Bottlenecks: Pick 1-2 key areas to explore. (e.g., "How does the news feed get generated?" or "How do you handle a 'hotspot' in the database?"). Discuss trade-offs.

Discuss Failure Scenarios & Monitoring: What happens when a server fails? How do you detect and recover? What metrics would you track?

Week 10: The Classics (Read-Heavy Systems)

Problem: Design a URL Shortener (like TinyURL or bit.ly).

Focus: Hashing algorithms (MD5, Base62), database design (SQL vs. NoSQL), handling redirects, custom short URLs, analytics.

Scale Estimation: 100M URLs created per month, 10:1 read/write ratio.

Trade-offs to discuss: Short URL collision handling, SQL vs. NoSQL choice.

Engineering Blog: Read about Bitly's architecture.

Practice: Design on whiteboard/paper, present to yourself out loud.

Week 11: The Classics (Write-Heavy & Feeds)

Problem: Design Twitter / Instagram / Facebook News Feed.

Focus: This is the "fan-out" problem. How do you get a post from one user to all their followers? Compare fan-out on write vs. fan-out on read. Celebrity problem (users with millions of followers).

Scale Estimation: 500M users, average 200 followers, 2 posts per day per user.

Trade-offs to discuss: Push vs. Pull model, hybrid approaches, caching strategies.

Data Structures: Time-series data for feed storage.

Engineering Blogs: Read Instagram feed architecture, Twitter timeline architecture.

Practice: Present both push and pull models with pros/cons.

Week 12: Media & Storage Systems

Problem: Design Netflix / YouTube / Google Drive.

Focus: How do you handle massive file (blob) storage? Object storage (S3). How do CDNs (Content Delivery Networks) work? Video processing pipelines (transcoding, adaptive bitrate streaming). How do you handle upload resume?

Scale Estimation: 1B videos, 500MB average size, 100M daily active users.

Trade-offs to discuss: Storage costs, CDN vs. origin servers, different video qualities.

Engineering Blogs: Netflix CDN architecture, YouTube video processing pipeline.

Practice: Draw end-to-end flow from video upload to playback.

Week 13: Real-Time, Geospatial & Advanced Topics

Problem 1: Design a Chat App (WhatsApp / Messenger).

Focus: WebSockets for real-time bidirectional connection, message delivery guarantees, read receipts, group chats, offline message sync, end-to-end encryption.

Problem 2: Design Uber / Lyft.

Focus: How do you find nearby drivers? Geospatial indexing (Quadtrees, Geohashing, S2 geometry). Real-time location updates. Matching algorithm. Surge pricing.

Scale Estimation: 10M drivers, 50M riders, 1M concurrent rides.

Data Structures: Quadtrees for spatial indexing, priority queues for matching.

Engineering Blogs: Uber geospatial indexing, WhatsApp scaling to 1B users.

Practice: Explain how you'd find the 10 nearest drivers in under 100ms.

Week 14: Deep Dive & Trade-off Analysis

Topics:

Review all previous designs. For each system, create a one-page summary with: Requirements, Scale, Core components, Key bottlenecks, Trade-offs made.

Compare: CP vs. AP systems you've designed. When did you choose SQL vs. NoSQL? When did you use caching vs. message queues?

Cost Analysis: For 2-3 systems, estimate cloud infrastructure costs (compute, storage, bandwidth).

Failure Scenarios: For each design, list 3 failure modes and how you'd handle them.

Engineering Blogs (read 2 per week from now on):
- Netflix Tech Blog: CDN, A/B testing, chaos engineering
- Uber Engineering: Geospatial systems, microservices
- Discord Engineering: Real-time messaging at scale
- Slack Engineering: WebSocket scaling
- Dropbox Tech Blog: Storage optimization
- Pinterest Engineering: Sharding, caching
- Airbnb Engineering: Search, payments

Practice: Do 3 mock interviews this week with peers or on platforms like Pramp.

Part 4: Mastery & Interview Prep (Weeks 15-16)

Week 15: Mock Interviews & Refinement

Schedule: Do 5-7 mock interviews this week (aim for 1 per day).

Platforms: Pramp, Interviewing.io, or with peers.

Self-Review: Record yourself or take notes. What did you struggle with? Did you forget to estimate scale? Did you jump to solutions too quickly?

Refinement: Go back to weak areas. If you struggled with caching strategies, re-read DDIA Chapter 5 and design 2 more systems with caching.

Additional Practice Problems:
- Design Amazon (e-commerce at scale)
- Design Ticketmaster (handling ticket sales spikes)
- Design Spotify (music streaming and recommendations)
- Design Rate Limiter (distributed rate limiting)
- Design Web Crawler (at Google/Bing scale)

Week 16: Final Polish & Confidence Building

Review: Create a one-page cheat sheet with:
- CAP theorem trade-offs
- SQL vs. NoSQL decision tree
- Common design patterns (fan-out, sharding strategies, cache patterns)
- Back-of-envelope numbers (QPS calculations, storage, bandwidth)

Mental Models: For each building block (cache, load balancer, database, queue), write down: "When to use it," "How it scales," "Common failure modes."

Real Interview Simulation: Do 2-3 full mock interviews under time pressure (45 minutes).

Read: "Cracking the System Design Interview" sections on behavioral questions and communication strategies.

Confidence: Review your progress. You've gone from zero to understanding production systems in 16 weeks.

Final Practice: Take 3 random system design problems. Set a timer for 45 minutes. Design end-to-end.

Key Resources

Books:
- "Designing Data-Intensive Applications" by Martin Kleppmann (Primary - read Chapters 1-6, 11)
- "System Design Interview" Volume 1 & 2 by Alex Xu (Best for interview patterns)
- "Google SRE Book" (Free online - focus on Chapters 4, 6, 22)
- "Release It!" by Michael Nygard (Resilience patterns)

Online Courses:
- ByteByteGo (Alex Xu's site) - Best for visual learners
- Grokking the System Design Interview (Educative.io)
- System Design Primer on GitHub (Free, comprehensive)

Engineering Blogs (Read 2 articles per week):
- High Scalability: http://highscalability.com
- Netflix Tech Blog: https://netflixtechblog.com
- Uber Engineering: https://eng.uber.com
- Meta Engineering: https://engineering.fb.com
- AWS Architecture Blog: https://aws.amazon.com/blogs/architecture
- Discord Engineering: https://discord.com/category/engineering
- Slack Engineering: https://slack.engineering
- Dropbox Tech: https://dropbox.tech
- Airbnb Engineering: https://medium.com/airbnb-engineering

Practice Platforms:
- Pramp (free peer mock interviews)
- Interviewing.io (mock interviews with engineers)
- SystemsExpert (AlgoExpert's system design course)

YouTube Channels:
- ByteByteGo
- Gaurav Sen (System Design)
- Tech Dummies Narendra L

Study Groups:
- Join system design Discord/Slack communities
- Find an accountability partner
- Schedule weekly mock interviews with peers

Success Metrics:

After 16 weeks, you should be able to:
1. Design any of the classic systems (Twitter, Netflix, Uber, Chat) in 45 minutes
2. Estimate scale (QPS, storage, bandwidth) within 2 minutes
3. Justify every design choice with trade-offs
4. Identify and explain 3+ failure scenarios for any design
5. Switch between SQL/NoSQL, push/pull, cache strategies based on requirements
6. Communicate clearly and drive the conversation in an interview

Remember: System design is about trade-offs, not perfect solutions. There's no single "right" answer. Your goal is to show structured thinking, knowledge of building blocks, and ability to reason about scale and failure.