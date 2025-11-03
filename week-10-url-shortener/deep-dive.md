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


### 2. Database Choice: SQL vs NoSQL

**SQL (PostgreSQL):**
- Pros:
- Cons:

**NoSQL (DynamoDB, Cassandra):**
- Pros:
- Cons:

**Your choice and why:**


### 3. Handling Collisions

**How to prevent/handle short code collisions:**


### 4. Caching Strategy

**What to cache:**
- Popular URLs (80/20 rule)

**Cache eviction policy:**
- LRU (Least Recently Used)

**Cache size calculation:**


### 5. Custom Short URLs

**How to handle user-provided custom aliases:**


### 6. Analytics

**How to track clicks efficiently:**


## Trade-offs Discussion

| Decision | Option A | Option B | Your Choice | Why? |
|----------|----------|----------|-------------|------|
| URL Generation | Hash-based | Base62 | | |
| Database | SQL | NoSQL | | |
| Redirect Type | 301 (permanent) | 302 (temporary) | | |

## Next Steps
Continue to [failure-scenarios.md](failure-scenarios.md)
