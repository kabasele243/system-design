# Database Indexes & B-trees

## Learning Objectives
- Understand how indexes make queries fast
- Learn B-tree structure and operations
- Know when to add indexes (and when not to)

## Notes

### What is an Index?

An **index** is like a book's index - it helps you find information quickly without reading every page.

**Without Index:**
```sql
SELECT * FROM users WHERE email = 'john@example.com';
-- Full table scan: O(n) - checks every row!
-- 1 million users = 1 million rows scanned
```

**With Index:**
```sql
CREATE INDEX idx_users_email ON users(email);
SELECT * FROM users WHERE email = 'john@example.com';
-- Index lookup: O(log n) - binary search!
-- 1 million users = ~20 comparisons (log₂ 1,000,000 ≈ 20)
```

### B-Tree Structure

**B-tree** = Balanced tree optimized for disk reads.

```
                    [M]
                   /   \
                 /       \
            [D, H]       [P, T]
           /  |   \     /  |   \
        [A,C][E,G][I,L][N,O][Q,S][U,Z]
         ↓    ↓    ↓    ↓    ↓    ↓
       data data data data data data
```

**Key Properties:**
- **Balanced**: All leaf nodes at same depth
- **Ordered**: Keys sorted left to right
- **Multi-way**: Each node has multiple children (not just 2)
- **Disk-optimized**: One node = one disk page (4KB-16KB)

**Search Example:**
```
Find "G":
1. Start at root [M]
2. G < M → go left to [D, H]
3. D < G < H → go middle to [E, G]
4. Found G! (3 disk reads total)
```

### Why B-trees are Fast

**Comparison to Binary Tree:**

| Structure | Height for 1M records | Disk Reads |
|-----------|----------------------|------------|
| Binary Tree | ~20 levels | 20 reads |
| B-tree (order 1000) | ~3 levels | 3 reads |

**Why?** B-tree nodes hold hundreds of keys per node!

### Composite Indexes

**Single-Column Index:**
```sql
CREATE INDEX idx_user_id ON orders(user_id);
-- Fast: WHERE user_id = 123
```

**Composite Index:**
```sql
CREATE INDEX idx_user_created ON orders(user_id, created_at);
-- Fast for both:
--   WHERE user_id = 123
--   WHERE user_id = 123 AND created_at > '2024-01-01'
```

**Column Order Matters!**

```sql
-- Index: (user_id, created_at)

-- ✅ CAN use index:
WHERE user_id = 123
WHERE user_id = 123 AND created_at > '2024-01-01'

-- ❌ CANNOT use index:
WHERE created_at > '2024-01-01'  -- Missing leading column!
```

**Rule:** Index can be used from left to right, but not from middle.

### Your YouTube Index Design (40/40 - Perfect!)

**Question:** Design indexes for YouTube-like queries

**Your Answer:**
```sql
-- Query 1: Get videos by user
CREATE INDEX idx_videos_user ON videos(user_id);

-- Query 2: Get videos by user, sorted by views
CREATE INDEX idx_videos_user_views ON videos(user_id, views DESC);

-- Query 3: Search videos by title
CREATE INDEX idx_videos_title ON videos(title);

-- Query 4: Get recent popular videos
CREATE INDEX idx_videos_created_views ON videos(created_at DESC, views DESC);
```

**What Made This Perfect:**
- ✅ Correct composite indexes
- ✅ DESC for sorting columns
- ✅ Leading column matches WHERE clause
- ✅ Understood query patterns

### Covering Index

**Covering Index** = Index contains ALL columns needed by query (no table lookup needed).

```sql
-- Query needs: user_id, email, name
CREATE INDEX idx_covering ON users(user_id, email, name);

SELECT email, name FROM users WHERE user_id = 123;
-- All data in index! No table access needed.
-- Faster because fewer disk reads.
```

**Your Score:** 40/40 - You included covering indexes in YouTube design!

### When to Add Indexes

**Add Index When:**
- ✅ Column used in WHERE clause frequently
- ✅ Column used in JOIN conditions
- ✅ Column used in ORDER BY
- ✅ Query is slow (check EXPLAIN plan)
- ✅ Table has many rows (>10K)

**Example:**
```sql
-- Slow query (full table scan):
SELECT * FROM orders WHERE user_id = 123;

-- Add index:
CREATE INDEX idx_orders_user ON orders(user_id);

-- Now fast! (index seek)
```

### When NOT to Add Indexes

**Don't Add Index When:**
- ❌ Table is small (<1000 rows) - full scan is fast enough
- ❌ Column has low cardinality (few unique values)
- ❌ Write-heavy table (indexes slow down INSERTs)
- ❌ Column rarely queried

**Example - Bad Index:**
```sql
-- Bad: Only 2 values (true/false)
CREATE INDEX idx_users_active ON users(is_active);
-- Full table scan is better!

-- Good: High cardinality
CREATE INDEX idx_users_email ON users(email);
-- Each email is unique
```

### Index Cost Trade-offs

**Indexes Help:**
- ✅ SELECT queries (faster reads)
- ✅ WHERE filtering
- ✅ JOIN operations
- ✅ ORDER BY sorting

**Indexes Hurt:**
- ❌ INSERT operations (must update index)
- ❌ UPDATE operations (must update index if indexed column changes)
- ❌ DELETE operations (must update index)
- ❌ Storage space (index takes disk space)

**Rule of Thumb:**
- Read-heavy workload → More indexes OK
- Write-heavy workload → Fewer indexes better

### EXPLAIN Plan

**Use EXPLAIN to see if index is used:**

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 123;
```

**Good Output (using index):**
```
Seq Scan on orders  (cost=0.00..1.25 rows=1)
  Index Cond: (user_id = 123)
  → Using index: idx_orders_user
```

**Bad Output (full table scan):**
```
Seq Scan on orders  (cost=0.00..10000.00 rows=1)
  Filter: (user_id = 123)
  → No index used! Add index!
```

## Practice Questions

### Question 1: YouTube Composite Indexes (Score: 40/40) 🏆

**Query Patterns:**
1. Get videos by uploader
2. Get videos by uploader sorted by views
3. Search videos by title
4. Get recent popular videos

**Your Answer:**
```sql
CREATE INDEX idx_videos_user ON videos(user_id);
CREATE INDEX idx_videos_user_views ON videos(user_id, views DESC);
CREATE INDEX idx_videos_title ON videos(title);
CREATE INDEX idx_videos_created_views ON videos(created_at DESC, views DESC);
```

**Perfect!** You understood:
- Composite index column order
- DESC for sorting
- Covering indexes
- Query pattern analysis

## Real-World Examples

### Instagram Photo Queries

```sql
-- Index design for Instagram

-- Get photos by user (profile page)
CREATE INDEX idx_photos_user ON photos(user_id, created_at DESC);

-- Get photos by user in date range
-- Same index works! (user_id, created_at)

-- Get liked photos
CREATE INDEX idx_likes_user ON likes(user_id, photo_id);

-- Get photo likes count
CREATE INDEX idx_likes_photo ON likes(photo_id);
-- Use COUNT(*) with index
```

### Twitter Timeline Queries

```sql
-- Get tweets by user
CREATE INDEX idx_tweets_user_created ON tweets(user_id, created_at DESC);

-- Get tweets mentioning user
CREATE INDEX idx_mentions_user ON mentions(mentioned_user_id, tweet_id);

-- Get retweets of tweet
CREATE INDEX idx_retweets_original ON retweets(original_tweet_id, created_at DESC);

-- Search tweets by hashtag
CREATE INDEX idx_hashtags_tag ON hashtags(tag, tweet_id);
```

### E-commerce Order Queries

```sql
-- Get orders by user
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at DESC);

-- Get orders by status (for admin)
CREATE INDEX idx_orders_status ON orders(status, created_at DESC);

-- Get order items
CREATE INDEX idx_items_order ON order_items(order_id);

-- Search products
CREATE INDEX idx_products_name ON products(name);
-- Or use full-text index:
CREATE FULLTEXT INDEX idx_products_fulltext ON products(name, description);
```

## Resources & References

- [PostgreSQL B-tree Internals](https://www.postgresql.org/docs/current/btree.html)
- [Use The Index, Luke!](https://use-the-index-luke.com/) - Comprehensive index guide
- [MySQL Index Optimization](https://dev.mysql.com/doc/refman/8.0/en/optimization-indexes.html)
- Designing Data-Intensive Applications (Chapter 3) - Martin Kleppmann
