# Database Schema Design

## Learning Objectives
- Learn normalization (1NF, 2NF, 3NF)
- Understand when to denormalize
- Master schema design patterns

## Notes

### Normalization

**Goal:** Eliminate redundancy and ensure data consistency.

#### First Normal Form (1NF)

**Rule:** Each cell contains a single value (atomic). No arrays or lists.

**❌ Violates 1NF:**
```sql
CREATE TABLE orders (
    order_id INT,
    customer_name VARCHAR(100),
    products VARCHAR(500)  -- "laptop,mouse,keyboard" ← Multiple values!
);
```

**✅ Follows 1NF:**
```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_name VARCHAR(100)
);

CREATE TABLE order_items (
    order_id INT,
    product_name VARCHAR(100),
    PRIMARY KEY (order_id, product_name)
);
```

#### Second Normal Form (2NF)

**Rule:** Must be in 1NF + No partial dependencies (non-key columns depend on entire primary key).

**❌ Violates 2NF:**
```sql
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    product_name VARCHAR(100),  -- Depends only on product_id (partial dependency)
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);
```

**✅ Follows 2NF:**
```sql
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100)
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);
```

#### Third Normal Form (3NF)

**Rule:** Must be in 2NF + No transitive dependencies (non-key columns don't depend on other non-key columns).

**❌ Violates 3NF:**
```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(100),  -- Depends on customer_id (transitive dependency)
    customer_email VARCHAR(100)  -- Depends on customer_id
);
```

**✅ Follows 3NF:**
```sql
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    customer_email VARCHAR(100)
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

### Your E-Commerce Normalization (30/30 - Perfect!)

**You normalized the schema to 3NF with proper foreign keys and eliminated all redundancy!**

### When to Denormalize

**Denormalization** = Intentionally adding redundancy for performance.

**Reasons to Denormalize:**
1. Avoid expensive JOINs (read-heavy workloads)
2. Reduce query latency
3. Simplify queries

**Example: User Profile with Post Count**

**Normalized (3NF):**
```sql
SELECT users.*, COUNT(posts.id) as post_count
FROM users
LEFT JOIN posts ON users.id = posts.user_id
GROUP BY users.id;
-- Expensive JOIN + GROUP BY on every query!
```

**Denormalized:**
```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50),
    post_count INT DEFAULT 0  -- Denormalized!
);

-- Update post_count when post is created:
BEGIN TRANSACTION;
  INSERT INTO posts (user_id, content) VALUES (123, 'Hello');
  UPDATE users SET post_count = post_count + 1 WHERE user_id = 123;
COMMIT;

-- Query is now simple and fast:
SELECT * FROM users WHERE user_id = 123;  -- No JOIN needed!
```

**Trade-offs:**
- ✅ Faster reads (no JOIN)
- ❌ Slower writes (must update two tables)
- ❌ Risk of inconsistency (if one update fails)

### Hybrid Approach with CDC

**Your Uber Database Design (50/50 - Masterpiece!):**

**Write Model (Normalized - PostgreSQL):**
```sql
CREATE TABLE trips (
    trip_id UUID PRIMARY KEY,
    driver_id UUID,
    rider_id UUID,
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    fare DECIMAL(10,2)
);

CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255)
);
```

**Read Model (Denormalized - Cassandra):**
```sql
-- Materialized view optimized for queries
CREATE TABLE trips_by_driver (
    driver_id UUID,
    trip_id UUID,
    rider_name VARCHAR(100),  -- Denormalized!
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    fare DECIMAL(10,2),
    PRIMARY KEY (driver_id, start_time)
) WITH CLUSTERING ORDER BY (start_time DESC);
```

**Change Data Capture (CDC):**
```python
# Kafka CDC pipeline
PostgreSQL (normalized) → Kafka → Cassandra (denormalized)

# When trip is created in PostgreSQL:
1. INSERT into trips table
2. CDC captures change
3. Kafka streams event
4. Consumer writes to Cassandra with denormalized data
```

**Why This Works:**
- PostgreSQL: ACID for writes
- Cassandra: Fast reads with denormalized data
- CDC: Keeps them in sync
- Best of both worlds!

## Practice Questions

### Question 1: E-Commerce Normalization (Score: 30/30) 🏆

You successfully normalized an e-commerce schema to 3NF with proper foreign keys!

### Question 2: Uber Hybrid Architecture (Score: 50/50) 🏆

Your design included:
- PostgreSQL for normalized writes
- Cassandra for denormalized reads
- Kafka CDC for synchronization
- Redis GEO for driver locations
- S2 cells for geospatial indexing

**Staff/Principal Engineer level thinking!**

## Real-World Examples

### Twitter Timeline Architecture

**Normalized (MySQL):**
```sql
CREATE TABLE users (user_id, username, email);
CREATE TABLE tweets (tweet_id, user_id, content, created_at);
CREATE TABLE follows (follower_id, following_id);
```

**Denormalized (Cassandra):**
```sql
-- Fan-out on write: Denormalized timeline
CREATE TABLE home_timeline (
    user_id UUID,
    tweet_id UUID,
    author_id UUID,
    author_name VARCHAR(50),  -- Denormalized!
    content TEXT,             -- Denormalized!
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, created_at)
);
```

**When user tweets:**
1. Write to MySQL (normalized)
2. Write to each follower's timeline in Cassandra (denormalized)
3. Fast reads: SELECT * FROM home_timeline WHERE user_id = ?

## Resources & References

- [Database Normalization](https://www.essentialsql.com/get-ready-to-learn-sql-database-normalization-explained-in-simple-english/)
- [Denormalization Patterns](https://docs.microsoft.com/en-us/azure/architecture/patterns/materialized-view)
- [Change Data Capture (CDC)](https://www.confluent.io/blog/how-change-data-capture-works-patterns-solutions-implementation/)
