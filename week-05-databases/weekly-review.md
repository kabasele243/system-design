# Week 5: Databases - Weekly Review

## Review Questions

### Question 1: Twitter Database Architecture (40 points)

**Problem:**
Design the database architecture for Twitter. Requirements:
- 500M users
- 10M tweets per day
- 100M timeline reads per day
- Users follow 500 people on average
- Need to support: user profiles, tweets, follows, likes

**Tasks:**
1. Choose database types for different data
2. Justify SQL vs NoSQL choices
3. Design schema for each database
4. Explain polyglot persistence strategy

**Your Answer (36/40 - 9/10):**

**Architecture:**
```
1. MySQL (Primary Database):
   - User accounts (ACID critical)
   - Tweet metadata
   - Follower relationships

2. Cassandra (Timeline Storage):
   - Home timeline (denormalized)
   - User timeline
   - Write-heavy workload

3. Redis (Cache):
   - Hot tweets
   - User sessions
   - Rate limiting counters
```

**What You Got Right:**
- ✅ Polyglot persistence (multiple databases for different needs)
- ✅ MySQL for ACID transactions (users, relationships)
- ✅ Cassandra for timelines (write-heavy, denormalized)
- ✅ Redis for caching hot data
- ✅ Understood trade-offs between databases

**What Was Missing:**
- Specific Cassandra schema (partition keys)
- Scale calculations (QPS, storage)
- Replication strategy

**Score: 9/10** - Excellent architecture thinking!

---

### Question 2: YouTube Composite Indexes (40 points)

**Problem:**
Design indexes for YouTube video queries:

```sql
-- Query 1: Get videos by uploader
SELECT * FROM videos WHERE uploader_id = ?;

-- Query 2: Get videos by uploader, sorted by views
SELECT * FROM videos WHERE uploader_id = ? ORDER BY views DESC;

-- Query 3: Search videos by title
SELECT * FROM videos WHERE title LIKE '%cats%';

-- Query 4: Get recent popular videos
SELECT * FROM videos ORDER BY created_at DESC, views DESC LIMIT 50;
```

**Your Answer (40/40 - Perfect!):** 🏆

```sql
-- Query 1: Single column index
CREATE INDEX idx_videos_uploader ON videos(uploader_id);

-- Query 2: Composite index (order matters!)
CREATE INDEX idx_videos_uploader_views ON videos(uploader_id, views DESC);

-- Query 3: Full-text search index
CREATE FULLTEXT INDEX idx_videos_title ON videos(title);
-- Or use Elasticsearch for better search

-- Query 4: Composite index for sorting
CREATE INDEX idx_videos_created_views ON videos(created_at DESC, views DESC);
```

**What Made This Perfect:**
- ✅ Correct composite index design
- ✅ Column order matches query patterns
- ✅ DESC for sorting columns
- ✅ Understood covering indexes
- ✅ Mentioned Elasticsearch alternative

**Score: 40/40** 🏆

---

### Question 3: LinkedIn Bloom Filter (30 points)

**Problem:**
Design a Bloom Filter for LinkedIn username availability checks.
- 500M usernames
- Target false positive rate: 1%
- Calculate memory needed
- Compare to storing all usernames

**Your Answer (30/30 - Perfect!):** 🏆

**Calculations:**
```python
# Bloom Filter formula: m = -(n * ln(p)) / (ln(2)²)
n = 500,000,000  # number of usernames
p = 0.01         # false positive rate (1%)

m = -(500,000,000 * ln(0.01)) / (ln(2)²)
m = -(500,000,000 * -4.605) / 0.4805
m = 4,794,463,601 bits
m = 599 MB

# Number of hash functions: k = (m/n) * ln(2)
k = (4,794,463,601 / 500,000,000) * 0.693
k ≈ 7 hash functions

# Storage comparison:
# - Storing all usernames: 20 bytes × 500M = 10 GB
# - Bloom Filter: 599 MB
# - Savings: 94% memory reduction!
```

**Implementation:**
```python
class LinkedInUsernameChecker:
    def __init__(self):
        self.bloom = BloomFilter(500_000_000, fp_rate=0.01)

        # Load existing usernames on startup
        for username in database.query("SELECT username FROM users"):
            self.bloom.add(username)

    def is_available(self, username):
        if not self.bloom.possibly_contains(username):
            return True  # Definitely available (99% of cases)

        # 1% false positives → check database
        exists = database.query(
            "SELECT 1 FROM users WHERE username = ?", username
        )
        return not exists
```

**Why This Works:**
- 99% of queries filtered by Bloom Filter (sub-millisecond)
- Only 1% hit database (false positives)
- 94% memory savings vs storing all usernames

**Score: 30/30** 🏆 Perfect mathematics and implementation!

---

### Question 4: E-Commerce Schema Normalization (30 points)

**Problem:**
Normalize this e-commerce schema to 3NF:

```sql
-- Denormalized (bad) schema
CREATE TABLE orders (
    order_id INT,
    customer_id INT,
    customer_name VARCHAR(100),      -- Redundant!
    customer_email VARCHAR(100),     -- Redundant!
    product_id INT,
    product_name VARCHAR(100),       -- Redundant!
    product_price DECIMAL(10,2),     -- Redundant!
    quantity INT,
    order_date DATE
);
```

**Your Answer (30/30 - Perfect!):** 🏆

**3NF Schema:**
```sql
-- 1NF: Atomic values, separate order items
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    customer_email VARCHAR(255),
    created_at TIMESTAMP
);

-- 2NF: No partial dependencies
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    price DECIMAL(10,2),
    description TEXT
);

-- 3NF: No transitive dependencies
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date TIMESTAMP,
    status VARCHAR(50),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE order_items (
    order_item_id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    quantity INT,
    price_at_purchase DECIMAL(10,2),  -- Historical price!
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

**Key Insights:**
- ✅ Eliminated all redundancy
- ✅ Proper foreign key relationships
- ✅ `price_at_purchase` preserves historical pricing
- ✅ Follows 1NF, 2NF, 3NF rules

**Score: 30/30** 🏆

---

### Question 5: Uber Database Architecture (50 points)

**Problem:**
Design a complete database architecture for Uber. Requirements:
- Real-time driver locations
- Trip history (billions of records)
- User accounts
- Payment processing
- Trip matching
- Analytics

**Your Answer (50/50 - Masterpiece!):** 🏆🏆🏆

**Architecture:**

**1. PostgreSQL (ACID Transactions):**
```sql
-- User accounts
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255),
    phone VARCHAR(20),
    created_at TIMESTAMP
);

-- Trips (source of truth)
CREATE TABLE trips (
    trip_id UUID PRIMARY KEY,
    rider_id UUID,
    driver_id UUID,
    start_lat DECIMAL(10,8),
    start_lng DECIMAL(11,8),
    end_lat DECIMAL(10,8),
    end_lng DECIMAL(11,8),
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    fare DECIMAL(10,2),
    status VARCHAR(50),
    FOREIGN KEY (rider_id) REFERENCES users(user_id),
    FOREIGN KEY (driver_id) REFERENCES users(user_id)
);

-- Payments
CREATE TABLE payments (
    payment_id UUID PRIMARY KEY,
    trip_id UUID,
    amount DECIMAL(10,2),
    payment_method VARCHAR(50),
    status VARCHAR(50),
    created_at TIMESTAMP,
    FOREIGN KEY (trip_id) REFERENCES trips(trip_id)
);
```

**2. Redis GEO (Real-Time Driver Locations):**
```python
# Store driver locations with geospatial indexing
redis.geoadd(
    "drivers:sf",  # Key per city
    longitude, latitude, driver_id
)

# Find nearby drivers (within 5km)
nearby_drivers = redis.georadius(
    "drivers:sf",
    rider_longitude, rider_latitude,
    5, "km"
)

# Update location every 5 seconds
# TTL = 30 seconds (auto-remove inactive drivers)
```

**3. Cassandra (Trip History - Write-Heavy):**
```sql
-- Denormalized for fast queries
CREATE TABLE trips_by_rider (
    rider_id UUID,
    trip_id UUID,
    driver_name VARCHAR(100),      -- Denormalized!
    start_location TEXT,           -- Denormalized!
    end_location TEXT,             -- Denormalized!
    fare DECIMAL(10,2),
    start_time TIMESTAMP,
    PRIMARY KEY (rider_id, start_time)
) WITH CLUSTERING ORDER BY (start_time DESC);

CREATE TABLE trips_by_driver (
    driver_id UUID,
    trip_id UUID,
    rider_name VARCHAR(100),       -- Denormalized!
    fare DECIMAL(10,2),
    start_time TIMESTAMP,
    PRIMARY KEY (driver_id, start_time)
) WITH CLUSTERING ORDER BY (start_time DESC);
```

**4. Kafka CDC (Change Data Capture):**
```
PostgreSQL (write) → Kafka → Cassandra (read)

When trip completed in PostgreSQL:
1. Write to trips table (ACID)
2. CDC captures change
3. Kafka streams event
4. Cassandra consumer denormalizes and writes
```

**5. S2 Cells (Geospatial Indexing):**
```python
# Divide world into cells for efficient matching
def get_s2_cell(lat, lng):
    return s2.S2CellId.from_lat_lng(lat, lng)

# Group drivers by S2 cell
s2_cell = get_s2_cell(rider_lat, rider_lng)
nearby_cells = get_neighboring_cells(s2_cell)

# Query only relevant cells
for cell in nearby_cells:
    drivers = redis.georadius(f"drivers:cell:{cell}", ...)
```

**What Made This Exceptional:**
- ✅ Polyglot persistence (5 different technologies!)
- ✅ PostgreSQL for ACID transactions
- ✅ Redis GEO for real-time locations
- ✅ Cassandra for massive trip history
- ✅ Kafka CDC for synchronization
- ✅ S2 cells for efficient geospatial matching
- ✅ Denormalization strategy explained
- ✅ Complete schemas with proper data types
- ✅ Understanding of write/read patterns

**Score: 50/50** 🏆🏆🏆 **Staff/Principal Engineer Level!**

---

## Weekly Review Summary

**Total Score: 186/200 (93%)** 🏆

### Breakdown by Topic:

1. **Twitter Architecture (36/40)** - 9/10
   - Excellent polyglot persistence
   - Could add more schema details

2. **YouTube Indexes (40/40)** - 10/10 🏆
   - Perfect composite index design
   - Understood column ordering

3. **Bloom Filter (30/30)** - 10/10 🏆
   - Perfect mathematics
   - Great implementation

4. **Schema Normalization (30/30)** - 10/10 🏆
   - Flawless 3NF design
   - Historical pricing insight

5. **Uber Architecture (50/50)** - 10/10 🏆🏆🏆
   - Masterpiece design
   - Production-ready architecture

### Key Strengths:

**Exceptional Understanding:**
- ✅ Polyglot persistence (using right database for right job)
- ✅ Composite index design
- ✅ Mathematical precision (Bloom Filter)
- ✅ Normalization principles
- ✅ Real-time geospatial systems
- ✅ CDC and data synchronization
- ✅ Trade-off analysis

**Staff/Principal Engineer Skills Demonstrated:**
- Multi-database architecture design
- Geospatial indexing (S2 cells, Redis GEO)
- Denormalization for performance
- Change Data Capture patterns
- Production-scale thinking

### Areas to Strengthen:

**Minor Improvements:**
- Add more specific schema details (partition keys, data types)
- Include scale calculations (QPS, storage, bandwidth)
- Replication strategies for high availability

**Overall:** Outstanding performance! 93% with perfect scores on 4 out of 5 questions. Your Uber architecture was truly exceptional - production-grade system design.

---

## Practice Exercise

Design the database architecture for an e-commerce platform with:
- Product catalog (1M products)
- User accounts (100M users)
- Orders (10M per day)
- Shopping cart
- Reviews and ratings
- Search functionality

**Consider:**
1. Which databases for which data?
2. Schema design
3. Caching strategy
4. Search implementation
5. Analytics queries

---

## Resources & References

- [Designing Data-Intensive Applications](https://dataintensive.net/) - Martin Kleppmann
- [Database Design Tutorial](https://www.guru99.com/database-design.html)
- [NoSQL Databases Explained](https://aws.amazon.com/nosql/)
- [Bloom Filter Calculator](https://hur.st/bloomfilter/)
- [PostgreSQL Index Optimization](https://www.postgresql.org/docs/current/indexes.html)
- [Cassandra Data Modeling](https://cassandra.apache.org/doc/latest/data_modeling/)
- [Redis Geospatial](https://redis.io/commands/geoadd)
