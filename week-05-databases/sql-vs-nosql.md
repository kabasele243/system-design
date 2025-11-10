# SQL vs NoSQL

## Learning Objectives
- Understand when to use SQL vs NoSQL
- Learn different NoSQL types (Key-Value, Document, Column-Family, Graph)
- Master the decision tree for database selection

## Notes

### The Decision Framework

**Start with SQL unless you have a specific reason not to.**

SQL databases are:
- ✅ Well-understood (40+ years of best practices)
- ✅ ACID compliant (strong consistency)
- ✅ Great for complex queries with JOINs
- ✅ Excellent tooling and ecosystem

### When to Use NoSQL

**4 Main Reasons to Choose NoSQL:**

1. **Need Horizontal Scalability**
   - SQL: Vertical scaling (bigger machine)
   - NoSQL: Horizontal scaling (more machines)
   - Example: Instagram stores billions of posts → Cassandra

2. **Schema Flexibility**
   - SQL: Fixed schema (must migrate)
   - NoSQL: Schema-less or flexible
   - Example: Content management systems with varying content types

3. **Specific Data Model**
   - Key-Value: Session storage (Redis)
   - Document: Product catalogs (MongoDB)
   - Wide-Column: Time-series data (Cassandra)
   - Graph: Social networks (Neo4j)

4. **Extreme Performance Requirements**
   - NoSQL often faster for simple lookups
   - Redis: Sub-millisecond reads
   - DynamoDB: Single-digit millisecond reads

### The 4 NoSQL Types

#### 1. Key-Value Stores (Redis, DynamoDB)

**Structure:**
```
Key: "user:123" → Value: {"name": "John", "age": 30}
```

**Use Cases:**
- Session storage
- Caching
- Shopping carts
- Real-time leaderboards

**Pros:**
- Extremely fast (O(1) lookups)
- Simple to understand
- Horizontally scalable

**Cons:**
- No complex queries
- No JOINs
- Limited querying by value

**Real Example - Twitter:**
```
Redis Key-Value:
  "timeline:user_123" → [tweet_id_1, tweet_id_2, ...]
  "tweet:456" → {"text": "Hello", "user_id": 123, ...}
```

#### 2. Document Stores (MongoDB, CouchDB)

**Structure:**
```json
{
  "_id": "123",
  "name": "John Doe",
  "email": "john@example.com",
  "addresses": [
    {"street": "123 Main St", "city": "NYC"},
    {"street": "456 Oak Ave", "city": "LA"}
  ]
}
```

**Use Cases:**
- Product catalogs (varying attributes)
- Content management systems
- User profiles
- Event logging

**Pros:**
- Flexible schema
- Nested data (no JOINs needed)
- Good query capabilities

**Cons:**
- Weak consistency (eventual)
- No ACID across documents
- Can lead to data duplication

**Real Example - E-commerce Product:**
```json
// Product can have different attributes
{
  "product_id": "laptop-123",
  "category": "electronics",
  "specs": {
    "ram": "16GB",
    "storage": "512GB SSD",
    "screen": "15.6 inch"
  }
}

{
  "product_id": "shirt-456",
  "category": "clothing",
  "specs": {
    "size": "L",
    "color": "blue",
    "material": "cotton"
  }
}
```

#### 3. Wide-Column Stores (Cassandra, HBase, BigTable)

**Structure:**
```
Row Key: user_123
  ├─ Column Family: profile
  │   ├─ name: "John"
  │   └─ email: "john@example.com"
  └─ Column Family: activity
      ├─ 2024-01-15:10:30:00: "login"
      ├─ 2024-01-15:10:35:00: "view_post"
      └─ 2024-01-15:10:40:00: "logout"
```

**Use Cases:**
- Time-series data
- Event logging
- IoT sensor data
- Write-heavy workloads

**Pros:**
- Handles massive write throughput
- Time-based queries are fast
- Horizontally scalable

**Cons:**
- Query flexibility limited
- No JOINs
- Denormalization required

**Real Example - Instagram Posts:**
```
Row Key: post_456
  ├─ user_id: 123
  ├─ caption: "Beautiful sunset"
  ├─ timestamp: 1699999999
  └─ likes:
      ├─ user_789: 1699999999
      ├─ user_101: 1700000010
      └─ user_202: 1700000025
```

#### 4. Graph Databases (Neo4j, Amazon Neptune)

**Structure:**
```
(User:John)-[:FOLLOWS]->(User:Jane)
(User:John)-[:LIKES]->(Post:123)
(User:Jane)-[:POSTED]->(Post:123)
```

**Use Cases:**
- Social networks
- Recommendation engines
- Fraud detection
- Knowledge graphs

**Pros:**
- Natural for relationship queries
- Fast traversals (friends of friends)
- Flexible relationships

**Cons:**
- Not general-purpose
- Learning curve
- Limited scalability

**Real Example - LinkedIn Connections:**
```cypher
// Find friends of friends (2 degrees of separation)
MATCH (me:User {id: 123})-[:CONNECTED_TO*2]-(friend_of_friend)
WHERE NOT (me)-[:CONNECTED_TO]-(friend_of_friend)
RETURN friend_of_friend
```

### SQL vs NoSQL Decision Tree

```
Start here: Do you need ACID transactions across multiple records?
├─ YES → Use SQL (PostgreSQL, MySQL)
│   Example: Banking, E-commerce orders
└─ NO → Continue...

Do you need complex queries with JOINs and aggregations?
├─ YES → Use SQL
│   Example: Analytics, Reporting
└─ NO → Continue...

What type of access pattern?
├─ Simple key-based lookups (get by ID)
│   → Key-Value Store (Redis, DynamoDB)
│   Example: Session storage, Caching
│
├─ Flexible documents with nested data
│   → Document Store (MongoDB)
│   Example: Product catalogs, CMS
│
├─ Time-series or append-heavy workloads
│   → Wide-Column Store (Cassandra)
│   Example: Logs, IoT data
│
└─ Relationship traversals (friends, recommendations)
    → Graph Database (Neo4j)
    Example: Social networks, Fraud detection
```

### Polyglot Persistence

**Modern systems use MULTIPLE databases!**

**Twitter Architecture (Your Example - 9/10 score!):**
```
1. MySQL (Primary database)
   - User accounts
   - Tweet metadata
   - Relationships (followers)
   - ACID for critical data

2. Cassandra (Timeline storage)
   - Home timeline (denormalized)
   - User timeline
   - Handles billions of tweets
   - Write-heavy workload

3. Redis (Cache)
   - Hot tweets
   - User sessions
   - Rate limiting counters
   - Sub-millisecond reads
```

**Why Multiple Databases?**
- Each database optimized for its purpose
- Don't force one database to do everything
- Trade-offs: Increased complexity

### Your Key Insight

> "Use PostgreSQL for structured data with relationships, Cassandra for timelines (write-heavy), and Redis for caching hot data."

**Perfect!** This shows understanding of:
- SQL for ACID and relationships
- Cassandra for massive scale
- Redis for performance

## Practice Questions

### Question 1: Twitter Database Design (Score: 9/10)

**Your Answer Highlights:**
- MySQL for users, relationships (ACID required)
- Cassandra for timelines (denormalized for performance)
- Redis for caching trending tweets
- Excellent reasoning on trade-offs

**Minor improvement:** Could have mentioned specific Cassandra schema design (partition keys).

## Real-World Examples

### Uber Database Stack

```
PostgreSQL:
- Ride records (ACID transactions)
- Driver accounts
- Payment processing

Cassandra:
- GPS location history (write-heavy)
- Trip logs (billions of records)
- Time-series data

Redis:
- Driver locations (real-time)
- Surge pricing cache
- Session management

Neo4j:
- Driver-rider matching (graph traversal)
- Fraud detection patterns
```

### Netflix

```
MySQL:
- User accounts
- Subscription billing
- ACID transactions

Cassandra:
- Viewing history (billions of events)
- Personalization data
- A/B test results

Elasticsearch:
- Search catalog
- Content discovery

Redis:
- Playback session state
- API rate limiting
```

### Spotify

```
PostgreSQL:
- User accounts
- Playlist metadata
- Subscription data

Cassandra:
- Listening history (100B+ events/year)
- User activity logs
- Recommendation training data

Redis:
- Currently playing cache
- Friend activity feed
- Session tokens
```

## Resources & References

- [CAP Theorem and Database Trade-offs](https://www.mongodb.com/nosql-explained/cap-theorem)
- [Designing Data-Intensive Applications](https://dataintensive.net/) - Martin Kleppmann (Chapter 2)
- [NoSQL Database Types Explained](https://aws.amazon.com/nosql/)
- [When to Use MongoDB vs PostgreSQL](https://www.mongodb.com/compare/mongodb-postgresql)
