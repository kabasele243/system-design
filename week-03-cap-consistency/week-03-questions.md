# Week 3: CAP Theorem & Consistency - Interview Questions

## Instructions
These are realistic senior-level system design interview questions. Approach each scenario as you would in an actual interview:
- State your assumptions clearly
- Show your reasoning process
- Discuss trade-offs
- Consider real-world constraints

---

## Question 1: E-Commerce Inventory - CP vs AP Decision

**Scenario:**
You're the principal engineer at a major e-commerce platform (similar to Amazon) designing the inventory management system. The Black Friday sale is approaching.

**Scale:**
- 500 million products
- 10,000 orders/second peak
- Multi-region: US East, US West, Europe, Asia
- Products with limited inventory (e.g., PS5 - only 1,000 units)

**Two Architecture Proposals:**

**Proposal A: CP System (Consistency + Partition Tolerance)**
- Strong consistency for inventory
- During network partition: Reject orders from partitioned region
- Never oversell inventory
- May lose orders during partition

**Proposal B: AP System (Availability + Partition Tolerance)**
- Eventual consistency for inventory
- During network partition: Accept all orders
- May oversell by ~1-2%
- Process refunds for oversold items

**Questions:**

1. **Business Impact Analysis:**
   - Black Friday: 50,000 orders/second peak, average order value $150
   - Expected network partition: 2 minutes during peak
   - Calculate revenue loss for Proposal A (CP - reject orders during partition)
   - Calculate refund cost for Proposal B (AP - 1% oversell rate)
   - Which is more cost-effective?

2. **User Experience:**
   - Proposal A: User sees "Service temporarily unavailable, try again"
   - Proposal B: User completes order, might receive refund email 2 hours later
   - Which creates better user experience? Why?
   - What's the reputational damage of each approach?

3. **Hybrid Solution:**
   - Can you design a system that uses CP for some products and AP for others?
   - Which products should be CP? Which should be AP?
   - Example: PS5 (limited stock) vs. USB cables (unlimited stock)

4. **Implementation Details:**
   - For CP approach: What happens when US East can't reach US West?
   - For AP approach: How do you detect and handle oversells?
   - Design the reconciliation process for oversold items

---

## Question 2: Social Media Platform - Consistency Model Selection

**Scenario:**
You're architecting a social media platform (competitor to Instagram). The platform has different features with different consistency requirements.

**Features:**
1. **Post Likes** - Users like posts
2. **Follower Counts** - Display follower numbers
3. **Direct Messages** - Private messaging
4. **Payment for Promoted Posts** - Users pay to promote content
5. **User Profile Edits** - Users update bio, profile picture

**Scale:**
- 1 billion users
- 50 million concurrent users
- 100,000 likes/second
- 10,000 messages/second
- 1,000 payments/second

**Questions:**

1. **Consistency Model Assignment:**
   For each feature, choose the appropriate consistency model:
   - Strong Consistency
   - Bounded Staleness (specify max staleness)
   - Session Consistency
   - Consistent Prefix
   - Eventual Consistency

   Justify each choice with specific reasoning.

2. **Post Likes Deep Dive:**
   - Current: Post shows "1,234 likes"
   - User A (US) likes it → 1,235
   - User B (Europe) sees what? When?
   - User C (Asia) sees what? When?

   Design the system:
   - What consistency model?
   - How do you aggregate counts across regions?
   - Is it acceptable to show different counts to different users?

3. **Direct Messages Complexity:**
   - User A sends message to User B
   - User A and User B are in different regions
   - User A must see their sent message immediately
   - User B should receive message quickly

   What consistency model?
   - Strong consistency (wait for cross-region sync)?
   - Session consistency (sender sees it, receiver gets it eventually)?
   - Design the trade-offs

4. **Payment System:**
   - User pays $100 to promote a post
   - Must guarantee: Money deducted exactly once
   - Must guarantee: Post promoted exactly once
   - What consistency model? Why?
   - How do you handle network partitions?

5. **Configuration in Code:**
   Show how you'd configure different consistency levels in a real database:
   ```javascript
   // Post Likes
   db.likes.insert({...}, {consistencyLevel: ???});

   // Direct Messages
   db.messages.insert({...}, {consistencyLevel: ???});

   // Payments
   db.payments.insert({...}, {consistencyLevel: ???});
   ```

---

## Question 3: Banking System - ACID vs BASE Trade-offs

**Scenario:**
You're the architect at a modern digital bank (similar to Chime) with two competing proposals for the backend architecture.

**Proposal A: Traditional ACID (SQL Database)**
- PostgreSQL with multi-master replication
- ACID transactions for everything
- Strong consistency everywhere
- Estimated: 10,000 transactions/second capacity

**Proposal B: Modern BASE (NoSQL Database)**
- Cassandra for high throughput
- Eventual consistency with background reconciliation
- Estimated: 100,000 transactions/second capacity

**Current Scale:**
- 10 million customers
- 5,000 transactions/second average
- 20,000 transactions/second peak (payday Fridays)
- Expected growth: 3x in next year

**Questions:**

1. **Critical Analysis:**
   - Can a bank use eventual consistency for account balances?
   - What could go wrong?
   - Provide specific failure scenarios

2. **Hybrid Architecture:**
   Design a hybrid system that uses both ACID and BASE:
   ```
   Which operations use ACID?
   Which operations use BASE?
   ```

   Consider these features:
   - Account balance checks
   - Money transfers
   - Transaction history
   - Spending analytics
   - Fraud detection logs
   - Customer profile updates

3. **The Oversell Problem:**
   Scenario using BASE approach:
   - User has $1,000 balance
   - Simultaneously tries to:
     - Pay $800 bill (US datacenter)
     - Withdraw $500 cash (EU datacenter)
   - Both datacenters think balance is $1,000
   - Both approve → Account goes to -$300

   Design solutions:
   - Can you prevent this with BASE?
   - What compensating mechanisms?
   - How long until the system detects the problem?

4. **BASE Reconciliation:**
   If you use BASE for non-critical features:
   - Design a background reconciliation job
   - How often does it run?
   - What does it fix?
   - How do you handle conflicts?

5. **Regulatory Compliance:**
   - Banking regulations require: "All transactions must be accurate within 1 second"
   - Does this rule out eventual consistency?
   - Can you use bounded staleness? What max staleness?

---

## Question 4: Multi-Region Gaming Platform - Network Partition Handling

**Scenario:**
You're designing a competitive online game (similar to League of Legends) with global player base.

**Infrastructure:**
- 3 regions: North America, Europe, Asia
- Each region has game servers and database
- Players matched within region for low latency
- Player profiles/inventory shared globally

**Current Architecture:**
- Player inventory (skins, items): Single master database in US
- During network partition between regions:
  - Asian players can't access inventory
  - Can't start games (need to load player skins)

**Problem:**
Network partition between US and Asia (underwater cable cut):
- Duration: 4 hours
- Affected: 10 million Asian players
- Lost revenue: Players can't buy new skins

**Questions:**

1. **CP vs AP Decision:**
   - Current system is CP (reject requests during partition)
   - Proposal: Make it AP (allow stale inventory reads)

   Trade-offs:
   - What if player buys skin in Asia, but purchase doesn't sync to US for 4 hours?
   - What if player buys same skin in US during partition?
   - How do you resolve conflicts?

2. **Split-Brain Scenario:**
   During 4-hour partition:
   - Player buys "Dragon Skin" for $20 in Asia datacenter
   - Player also buys "Phoenix Skin" for $15 in US datacenter (traveling)
   - Both purchases succeed (AP system)
   - Network reconnects

   What happens?
   - Do both purchases count? (User charged $35)
   - Or one gets rolled back? (User gets refund, loses skin)
   - How do you decide?

3. **Quorum-Based Approach:**
   Use 3 datacenters with quorum writes:
   - Write succeeds if 2 out of 3 datacenters confirm
   - During partition (US-Asia broken):
     - US + Europe = 2 datacenters (quorum!) ✅
     - Asia alone = 1 datacenter (no quorum) ❌

   Analysis:
   - Does this solve the problem?
   - What happens to Asian players?
   - Is this fair? (US/EU players can play, Asian players can't)

4. **Eventual Consistency with Compensation:**
   Design an AP system with compensating transactions:
   - Allow purchases during partition
   - Detect conflicts after partition heals
   - How do you handle:
     - Duplicate purchases
     - Insufficient funds (account balance)
     - Inventory oversell (limited edition items)

---

## Question 5: Uber - Different Consistency for Different Features

**Scenario:**
You're a staff engineer at Uber. Different features have different consistency requirements. Design the appropriate consistency model for each.

**Features:**
1. **Driver Location Updates** (500K updates/second)
2. **Ride Matching** (Driver accepts ride request)
3. **Payment Processing** ($25 average ride)
4. **Surge Pricing** (Price multiplier updates)
5. **Trip History** (Past rides)
6. **Driver Earnings** (Daily/weekly totals)

**Questions:**

1. **Feature-by-Feature Analysis:**
   For each feature, specify:
   - Consistency model (Strong, Bounded, Session, Eventual)
   - CP or AP?
   - Justification
   - What could go wrong with weaker consistency?
   - What's the cost of stronger consistency?

2. **Driver Location (Eventual Consistency):**
   - Driver at Location A (0.0000, 0.0000)
   - Moves to Location B (0.0001, 0.0001)
   - Rider sees driver location with 2-second delay

   Acceptable?
   - What's the max staleness you'd allow?
   - What if delay is 30 seconds? Still OK?
   - How do you measure and monitor staleness?

3. **Ride Matching (Strong Consistency):**
   Scenario:
   - 1 driver, 2 riders requesting simultaneously
   - Both US East and US West datacenters receive requests
   - Only 1 driver can accept

   Design:
   - How do you prevent double-booking?
   - Use distributed lock?
   - Use quorum writes?
   - What happens during network partition?

4. **Payment Processing (ACID Required):**
   Transaction:
   ```
   1. Charge rider $25
   2. Credit driver $20
   3. Uber keeps $5 commission
   ```

   Requirements:
   - All three must succeed or all fail
   - No partial transactions
   - Money never lost or duplicated

   Design:
   - ACID transaction across 3 accounts?
   - What if payment gateway times out?
   - Retry logic?
   - Idempotency?

5. **Hybrid Database Strategy:**
   Design a multi-database architecture:
   ```
   PostgreSQL (ACID):
   - Which features?

   Cassandra (BASE):
   - Which features?

   Redis (Cache):
   - Which features?
   ```

---

## Question 6: Session Consistency Implementation

**Scenario:**
You're implementing session consistency for a collaborative document editing platform (similar to Google Docs).

**Requirement:**
- User A types "Hello"
- User A must see "Hello" immediately
- User B might see it 100ms later (eventual consistency)
- But User A must ALWAYS see their own edits

**Questions:**

1. **Implementation Design:**
   Design session consistency mechanism:
   ```javascript
   // User A writes
   await doc.write("Hello", {userId: "A"});

   // User A reads (must see "Hello")
   const content = await doc.read({userId: "A"});

   // User B reads (might see "" or "Hello")
   const content = await doc.read({userId: "B"});
   ```

   How does the system track session?
   - Session tokens?
   - User ID tracking?
   - Timestamp comparison?

2. **Read-After-Write Consistency:**
   - User writes at timestamp T1
   - User reads at timestamp T2 (T2 > T1)
   - System must ensure: Read includes all writes before T2

   Implementation:
   - How do you track write timestamps?
   - How do you ensure read sees write?
   - What if write hasn't replicated yet?

3. **Monotonic Reads:**
   - User reads version V1: "Hello"
   - User reads again: Must see V1 or later, never older
   - Prevents: User seeing "Hello" then "" (going backward)

   How do you implement this?
   - Version vectors?
   - Causal consistency?

4. **Session Token Expiry:**
   - Session tokens can't live forever (memory/storage)
   - After expiry: Fall back to eventual consistency

   Design:
   - How long should sessions live?
   - What happens when session expires mid-edit?
   - Can you extend session timeout?

---

## Question 7: CAP Theorem - Real Network Partition Scenario

**Scenario:**
You're on-call at Netflix when a major network partition occurs. The underwater cable connecting US East to US West is severed (estimated 6-hour repair).

**Infrastructure:**
- **US East:** 60% of user traffic, master database
- **US West:** 40% of user traffic, replica database
- **Current System:** CP (strong consistency)

**Problem:**
- US West can't reach US East
- If you maintain CP: US West users can't stream (40% of users affected)
- If you switch to AP: US West might serve stale data

**Questions:**

1. **Immediate Decision (First 5 Minutes):**
   You have 5 minutes to decide:
   - **Option A:** Maintain CP - Reject 40% of traffic (US West)
   - **Option B:** Switch to AP - Serve potentially stale data

   Calculate:
   - Revenue impact of Option A: 40% of users × 6 hours × ?
   - User impact of Option B: Stale recommendations vs no service
   - What do you choose?

2. **Stale Data Acceptable?**
   In AP mode, US West serves data from last sync (5 minutes old):
   - User's watch history: 5 minutes stale - OK?
   - Continue watching position: 5 minutes stale - OK?
   - Account subscription status: 5 minutes stale - OK?
   - Recently added movies: 5 minutes stale - OK?

   Which features are safe with stale data?
   Which features need strong consistency?

3. **Write Operations During Partition:**
   - User in US West adds movie to "My List"
   - Can't sync to US East (partition)
   - Store locally and sync later? (AP)
   - Reject the write? (CP)

   Design the write handling strategy

4. **Partition Healing:**
   After 6 hours, cable is repaired:
   - US West has 6 hours of writes
   - US East has 6 hours of writes
   - How do you merge?
   - What if same user made conflicting changes?

5. **Lessons Learned:**
   - Should Netflix be CP or AP?
   - What's the right balance?
   - How do you prepare for future partitions?

---

## Question 8: DynamoDB vs PostgreSQL - Consistency Configuration

**Scenario:**
You're migrating a SaaS application from PostgreSQL (ACID) to DynamoDB (eventual consistency by default). The application has different features with different requirements.

**Features:**
1. User login (check credentials)
2. Credit card processing
3. Analytics dashboard (pageviews, clicks)
4. User activity logs
5. Real-time notifications

**Questions:**

1. **DynamoDB Consistency Options:**
   DynamoDB offers:
   - `ConsistentRead=False` (eventual consistency, faster, cheaper)
   - `ConsistentRead=True` (strong consistency, slower, 2x cost)

   For each feature, choose the right option and justify:
   ```javascript
   // User login
   dynamodb.getItem('users', {userId}, {ConsistentRead: ???});

   // Credit card processing
   dynamodb.putItem('payments', {...}, {ConsistentRead: ???});

   // Analytics dashboard
   dynamodb.query('analytics', {...}, {ConsistentRead: ???});
   ```

2. **Cost-Benefit Analysis:**
   - Eventual consistency: $1 per million reads
   - Strong consistency: $2 per million reads
   - Expected: 100 million reads/day

   If you use strong consistency for everything:
   - Daily cost: ?
   - Yearly cost: ?

   If you use eventual for 80% of reads:
   - Daily cost: ?
   - Yearly cost: ?
   - Savings: ?

3. **Eventual Consistency Gotcha:**
   Scenario:
   ```javascript
   // User updates email
   await db.putItem('users', {userId: 123, email: 'new@example.com'});

   // Immediately read back
   const user = await db.getItem('users', {userId: 123}, {ConsistentRead: false});
   console.log(user.email); // What prints?
   ```

   - Could print old email!
   - How do you handle this?
   - When is this acceptable?

4. **Migration Strategy:**
   Can't migrate all at once (too risky):
   - Phase 1: Migrate which features first?
   - Phase 2: Migrate which features?
   - Phase 3: Migrate which features last?

   Justify the phased approach

---

## Question 9: Bounded Staleness for Financial Services

**Scenario:**
You're designing a stock trading platform. Different features have different latency requirements.

**Features:**
1. **Live Stock Prices** - Current price of AAPL, GOOGL, etc.
2. **Order Execution** - Buy/sell stocks
3. **Portfolio Value** - Total account value
4. **Historical Charts** - Stock price history
5. **Trading Recommendations** - AI-generated suggestions

**Questions:**

1. **Bounded Staleness Configuration:**
   For "Live Stock Prices," choose max staleness:
   - 100ms?
   - 1 second?
   - 5 seconds?
   - 1 minute?

   Justify your choice considering:
   - User expectations
   - Technical feasibility
   - Cost
   - Regulatory requirements

2. **Order Execution Scenario:**
   User sees: "AAPL = $150" (5 seconds stale)
   User clicks: "Buy 10 shares at $150"
   Actual price: $151 (increased in last 5 seconds)

   What happens?
   - Execute at $150? (User expects this)
   - Execute at $151? (Actual current price)
   - Reject order? (Price changed)

   Design the order execution logic

3. **Portfolio Value Accuracy:**
   User has:
   - 100 shares AAPL ($150 each)
   - 50 shares GOOGL ($2,800 each)
   - Total: $155,000

   If stock prices are 5 seconds stale:
   - Portfolio value is also stale
   - Acceptable?
   - What's the max error you'd allow?

4. **Consistency Model Selection:**
   For each feature, choose:
   - Strong consistency
   - Bounded staleness (specify max)
   - Eventual consistency

   Justify each choice

---

## Question 10: Instagram - Distributed Counter Problem

**Scenario:**
You're solving the "distributed counter" problem at Instagram. A viral post gets 10 million likes in 1 hour.

**Challenge:**
- Likes come from users worldwide
- Need to display like count
- Update count in real-time
- Can't do strongly consistent counter (too slow)

**Questions:**

1. **Naive Approach (Strong Consistency):**
   ```sql
   UPDATE posts SET likes = likes + 1 WHERE post_id = 123;
   SELECT likes FROM posts WHERE post_id = 123;
   ```

   Problems:
   - Every like requires distributed transaction
   - 10M likes = 10M transactions
   - Each transaction: 100-500ms latency
   - Calculate: Throughput limit (transactions/second)
   - Can this handle 10M likes/hour?

2. **Eventual Consistency Approach:**
   Design:
   - Each datacenter maintains local counter
   - Periodically sync counters
   - Display: Sum of all counters

   Implementation:
   ```javascript
   // US datacenter: 5,000,000 likes
   // EU datacenter: 3,000,000 likes
   // Asia datacenter: 2,000,000 likes
   // Display: 10,000,000 likes
   ```

   Problems:
   - What if user refreshes quickly?
   - Count might go: 10,000,000 → 9,999,950 → 10,000,100
   - Acceptable?

3. **Hybrid Approach:**
   - Strongly consistent counter in database
   - Eventually consistent cache in Redis
   - Background job syncs every 10 seconds

   Trade-offs:
   - Pro: Fast reads (from cache)
   - Con: Stale by up to 10 seconds
   - Question: How do you handle cache stampede (10M users hitting cache simultaneously)?

4. **Approximate Counting:**
   - Instead of exact count, show approximate:
     - Exact: "10,234,567 likes"
     - Approximate: "10.2M likes"

   Design:
   - At what threshold switch from exact to approximate?
   - How do you compute approximate count?
   - Is this acceptable to users?

5. **Monitoring Staleness:**
   How do you measure:
   - Current staleness of like counts?
   - Max staleness in last 24 hours?
   - Alert when staleness exceeds threshold?

   Design monitoring dashboard

---

## Evaluation Rubric

When reviewing your answers, assess yourself on:

**CAP Theorem Understanding (30%)**
- Correct application of CP vs AP
- Understanding of partition scenarios
- Real-world trade-off analysis
- Business impact assessment

**Consistency Models (30%)**
- Appropriate model selection
- Understanding of consistency spectrum
- Implementation feasibility
- Performance implications

**ACID vs BASE (20%)**
- When to use each paradigm
- Hybrid architecture design
- Transaction handling
- Reconciliation strategies

**System Design (20%)**
- Practical solutions
- Handling edge cases
- Monitoring and observability
- Cost-benefit analysis

---

## Answer Key - Available Separately
Detailed solutions with architecture diagrams, implementation examples, and production best practices are provided in a separate file. Attempt all questions before reviewing answers.
