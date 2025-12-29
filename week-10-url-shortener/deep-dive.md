# URL Shortener - Deep Dive

## Step 4: Deep Dive & Bottlenecks

### 1. Short URL Generation

**Approach 1: Hash-based (MD5, SHA-256)**
- Pros:
- Cons:
- Collision handling:

**Approach 2: Base62 Encoding**
- Characters: [a-z, A-Z, 0-9] = 62 characters
- 7 characters = 62^7 = 3.5 trillion URLs
- Algorithm:
  1. Generate unique ID (auto-increment or distributed ID generator)
  2. Convert ID to Base62
  3. Store mapping in database

**Your chosen approach and why:**
**Base62 Encoding**.
- It results in the shortest possible URL for the given keyspace (base 62 vs base 16 for hex).
- 7 characters gives 3.5 trillion combinations, which is sufficient for decades at strict usage.
- Does not require complex collision resolution if built on a unique ID generator (e.g. Snowflake).


### 2. Database Choice: SQL vs NoSQL

**SQL (PostgreSQL):**
- Pros: Structured data, ACID transactions (crucial for uniqueness), indexing.
- Cons: Scaling writes requires sharding (though 40 write QPS is low).

**NoSQL (DynamoDB, Cassandra):**
- Pros: High scalability, flexible schema.
- Cons: Eventual consistency can lead to duplicate short codes if not careful.

**Your choice and why:**
**PostgreSQL**.
- The specific requirement of unique short codes benefits from relational uniqueness constraints.
- 6TB of data is manageable with partitioning or strict sharding.
- The read/write throughput (~400/40) is easily handled by a single primary with read replicas, no immediate need for NoSQL scale.


### 3. Handling Collisions

**How to prevent/handle short code collisions:**
- **With Base62 + Unique ID**: Collisions are theoretically impossible because the inputs (IDs) are unique. The ID generator (e.g., Snowflake, or a dedicated ticket server) guarantees uniqueness.
- **With Custom URLs**: Database unique constraint on the `short_code` column will fail the insert. The application catches the exception and prompts the user to choose another index.


### 4. Caching Strategy

**What to cache:**
- Popular URLs (80/20 rule): 20% of the URLs will generate 80% of the traffic.

**Cache eviction policy:**
- LRU (Least Recently Used): Discard the least recently requested items first.

**Cache size calculation:**
- Daily Read Requests: 100M * 10 / 30 = ~33M reads/day.
- 20% of traffic = 6.6M requests.
- If we cache 20% of unique URLs accessed daily:
- Assumption: 1M unique hot URLs/day.
- 1M * 1KB = 1GB of memory. This is very small, so we can afford to cache much more (e.g., 20% of all active URLs).


### 5. Custom Short URLs

**How to handle user-provided custom aliases:**
- Input validation: max length (e.g. 16), allowed characters.
- Check existence: DB query to see if taken.
- Store: Insert into DB. If unique constraint violation, return "Alias already taken".


### 6. Analytics

**How to track clicks efficiently:**
- **Async Processing**: Do not write to DB on the critical path of the redirect.
- **Flow**:
  1. Redirect service sends event to Message Queue (Kafka).
  2. Analytics consumers read from Kafka.
  3. Aggregate data (clicks per hour/day, country, referrer) and store in OLAP DB (e.g., ClickHouse) or time-series DB.


## Trade-offs Discussion

| Decision | Option A | Option B | Your Choice | Why? |
|----------|----------|----------|-------------|------|
| URL Generation | Hash-based | Base62 | **Base62** | Shorter, predictable, no collisions with unique ID. |
| Database | SQL | NoSQL | **SQL** | Strong consistency for unique constraints; scale is manageable. |
| Redirect Type | 301 (permanent) | 302 (temporary) | **302** | Allows us to track analytics (301 is cached by browser permanently). |

## Next Steps
Continue to [failure-scenarios.md](failure-scenarios.md)
