# Bloom Filters

## Learning Objectives
- Understand probabilistic data structures
- Learn how Bloom Filters work
- Know when to use Bloom Filters

## Notes

### What is a Bloom Filter?

A **Bloom Filter** is a space-efficient probabilistic data structure that tells you:
- "**Definitely NOT in the set**" (100% accurate)
- "**Possibly in the set**" (might be false positive)

**Key Property:** No false negatives, but possible false positives.

### The Problem Bloom Filters Solve

**Scenario:** LinkedIn wants to check if a username exists before querying the database.

**Option 1: Query Database**
```sql
SELECT * FROM users WHERE username = 'john_doe';
-- Slow! 10-50ms per query
-- 1M username checks/day = expensive!
```

**Option 2: Store All Usernames in Memory**
```python
usernames = set()  # 500M usernames
# 20 bytes per username = 10 GB RAM
# Expensive!
```

**Option 3: Bloom Filter**
```python
bloom_filter = BloomFilter(size=1_000_000, false_positive_rate=0.01)
# Only 859 MB for 500M items!
# Sub-millisecond lookups
```

### How Bloom Filters Work

**Structure:** Array of bits + multiple hash functions

**Insert "john":**
```
1. Hash1("john") = 3
2. Hash2("john") = 7
3. Hash3("john") = 12

Bit array: [0,0,0,1,0,0,0,1,0,0,0,0,1,0,0,0]
            ↑       ↑           ↑
           3       7          12
Set these bits to 1
```

**Check if "john" exists:**
```
1. Hash1("john") = 3 → bit[3] = 1 ✅
2. Hash2("john") = 7 → bit[7] = 1 ✅
3. Hash3("john") = 12 → bit[12] = 1 ✅

All bits are 1 → "Possibly exists" (might be false positive)
```

**Check if "jane" exists:**
```
1. Hash1("jane") = 5 → bit[5] = 0 ❌

Any bit is 0 → "Definitely does NOT exist" (guaranteed)
```

### False Positives Explained

**False Positive:** Bloom filter says "possibly exists" but item is NOT actually in set.

**Example:**
```
Inserted: "john", "mary", "bob"

Bits set: 3, 7, 12, 5, 9, 15, 2, 11, 14

Check "alice":
  Hash1("alice") = 3 (already set by "john")
  Hash2("alice") = 9 (already set by "mary")
  Hash3("alice") = 15 (already set by "bob")

Result: "Possibly exists" ← FALSE POSITIVE!
Alice was never inserted, but all her bits are set by other items.
```

**Trade-off:**
- More bits = fewer false positives, more memory
- Fewer bits = more false positives, less memory

### Your LinkedIn Bloom Filter Design (30/30 - Perfect!)

**Question:** Design a Bloom Filter for LinkedIn username checks (500M users).

**Your Answer:**
```python
# Requirements:
# - 500M usernames
# - False positive rate: 1% (acceptable)
# - Minimize memory usage

# Bloom Filter Calculation:
# Formula: m = -(n * ln(p)) / (ln(2)²)
# where:
#   n = 500,000,000 (number of items)
#   p = 0.01 (false positive rate)
#   m = bits needed

m = -(500_000_000 * ln(0.01)) / (ln(2)²)
m = -(500_000_000 * -4.605) / 0.4805
m = 4,794,463,601 bits
m = 599 MB

# Number of hash functions:
# k = (m/n) * ln(2)
k = (4_794_463_601 / 500_000_000) * 0.693
k ≈ 7 hash functions

# Storage: 599 MB vs 10 GB (storing all usernames)
# Savings: 94% memory reduction!
```

**What Made This Perfect:**
- ✅ Correct formula application
- ✅ Accurate calculations
- ✅ Memory comparison
- ✅ Understood false positive trade-off

### Bloom Filter Formulas

**1. Bits needed (m):**
```
m = -(n * ln(p)) / (ln(2)²)

where:
  n = number of items
  p = desired false positive rate
  m = bits needed
```

**2. Optimal number of hash functions (k):**
```
k = (m/n) * ln(2)
```

**3. Actual false positive rate:**
```
p = (1 - e^(-kn/m))^k
```

### When to Use Bloom Filters

**Use Cases:**

1. **Username/Email Existence Checks**
   - LinkedIn, Twitter username availability
   - Prevents database queries for non-existent users
   - 99% of queries filtered out

2. **Cache Key Existence**
   - Check if key exists in cache before querying
   - Avoids expensive cache misses
   - Medium posts: "Is article cached?"

3. **Malicious URL Detection**
   - Chrome Safe Browsing
   - Check if URL is in malicious list
   - Millions of URLs, sub-millisecond checks

4. **Duplicate Detection**
   - Web crawlers: "Have I seen this URL before?"
   - Bitcoin: Check if transaction is duplicate
   - Spam filters: "Is this email spam?"

5. **Database Query Optimization**
   - Cassandra/HBase: Check if SSTable contains key
   - Avoids expensive disk reads
   - Per-SSTable bloom filters

### Real-World Examples

#### LinkedIn Username Availability

**Without Bloom Filter:**
```python
# Every username check hits database
def is_username_available(username):
    user = db.query("SELECT id FROM users WHERE username = ?", username)
    return user is None

# 10M checks/day × 20ms = 200,000 seconds = 55 hours of DB time!
```

**With Bloom Filter:**
```python
bloom = BloomFilter(500_000_000, false_positive_rate=0.01)

# Load all usernames into Bloom Filter (on startup)
for user in db.query("SELECT username FROM users"):
    bloom.add(user.username)

def is_username_available(username):
    if not bloom.possibly_contains(username):
        return True  # Definitely available! (99% of cases)

    # Only 1% of queries reach database (false positives)
    user = db.query("SELECT id FROM users WHERE username = ?", username)
    return user is None

# 10M checks/day:
#   - 9.9M filtered by Bloom Filter (sub-ms)
#   - 100K hit database (false positives)
# Database load reduced by 99%!
```

#### Chrome Safe Browsing

```python
# Google maintains list of malicious URLs (millions)
# Can't store all URLs on client (too big)

# Solution: Bloom Filter
malicious_urls = BloomFilter(10_000_000, fp_rate=0.001)

def check_url(url):
    if not malicious_urls.possibly_contains(url):
        return "Safe"  # 99.9% of cases

    # False positive: Query Google API to confirm
    return google_api.check_url(url)

# 99.9% of checks are instant (no network request)
```

#### Medium Article Caching

```python
# Check if article is cached before querying database

article_cache_keys = BloomFilter(1_000_000, fp_rate=0.05)

def get_article(article_id):
    cache_key = f"article:{article_id}"

    if article_cache_keys.possibly_contains(cache_key):
        # Check actual cache
        cached = redis.get(cache_key)
        if cached:
            return cached

    # Not in cache, query database
    article = db.query("SELECT * FROM articles WHERE id = ?", article_id)
    redis.set(cache_key, article)
    article_cache_keys.add(cache_key)
    return article

# Avoids Redis lookups for articles not in cache
```

### Bloom Filter Limitations

**Cannot Remove Items:**
```python
bloom.add("john")
# Can't delete "john" later!
# Deleting would affect other items
```

**Solution:** Counting Bloom Filter (uses counters instead of bits, but uses more space)

**No Listing Items:**
```python
# Can't do: bloom.list_all_items()
# Bloom Filter doesn't store actual values
```

**False Positives Accumulate:**
```python
# As more items added, false positive rate increases
bloom = BloomFilter(1000, fp_rate=0.01)
for i in range(1000):
    bloom.add(f"item{i}")  # Now at capacity

# Adding more items increases false positive rate above 1%
```

**Solution:** Resize or use multiple Bloom Filters

## Practice Questions

### Question 1: LinkedIn Username Bloom Filter (Score: 30/30) 🏆

**Your calculation:**
- 500M usernames
- 1% false positive rate
- Result: 599 MB, 7 hash functions
- 94% memory savings vs storing all usernames

**Perfect mathematics and reasoning!**

## Real-World Examples

### Cassandra SSTable Bloom Filters

```
Cassandra stores data in SSTables (on-disk files)

Without Bloom Filters:
  Query for key "user_123"
  → Check 100 SSTables (expensive disk reads)

With Bloom Filters:
  Each SSTable has Bloom Filter in memory
  → 99 SSTables: "Definitely not here" (instant)
  → 1 SSTable: "Possibly here" (check disk)
  Result: 99% fewer disk reads!
```

### Bitcoin Transaction Validation

```python
# Bitcoin node needs to check if transaction already exists
# Blockchain has millions of transactions

bloom = BloomFilter(100_000_000, fp_rate=0.001)

def is_transaction_seen(tx_id):
    if not bloom.possibly_contains(tx_id):
        return False  # New transaction

    # Check actual blockchain (rare)
    return blockchain.has_transaction(tx_id)

# 99.9% of checks are instant
```

## Resources & References

- [Bloom Filters by Example](https://llimllib.github.io/bloomfilter-tutorial/)
- [Bloom Filter Calculator](https://hur.st/bloomfilter/)
- [Cassandra Bloom Filters](https://cassandra.apache.org/doc/latest/operating/bloom_filters.html)
- [Space/Time Trade-offs in Hash Coding with Allowable Errors](https://dl.acm.org/doi/10.1145/362686.362692) - Original paper (1970)
