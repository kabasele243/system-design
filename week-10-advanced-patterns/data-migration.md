# Data Migration & Schema Evolution

## Overview

Data migration and schema evolution are critical challenges in production systems. As a senior engineer, you need to modify databases without downtime, maintain backward compatibility, and ensure data integrity throughout the process.

## Zero-Downtime Migrations

### The Challenge
- Production systems can't afford downtime
- Millions of requests happening during migration
- Old and new code versions running simultaneously
- Need to rollback if something goes wrong

### The Expand-Contract Pattern

This is the gold standard for zero-downtime migrations.

**Phase 1: Expand (Add new structure)**
```sql
-- Add new column (nullable or with default)
ALTER TABLE users ADD COLUMN email_verified BOOLEAN DEFAULT false;

-- Add new index in background
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
```

**Phase 2: Dual-Write (Write to both old and new)**
```python
# Application code writes to both
def update_user(user_id, email):
    # Write to old column
    db.execute("UPDATE users SET email = ? WHERE id = ?", email, user_id)
    # Write to new column
    db.execute("UPDATE users SET email_verified = ? WHERE id = ?", False, user_id)
```

**Phase 3: Backfill (Migrate existing data)**
```sql
-- Backfill in batches to avoid lock contention
UPDATE users
SET email_verified = (email IS NOT NULL AND email != '')
WHERE id >= ? AND id < ?
LIMIT 10000;
```

**Phase 4: Contract (Remove old structure)**
```sql
-- After all code uses new column
ALTER TABLE users DROP COLUMN old_email_field;
```

### Multi-Step Deployments

**Step 1: Deploy code that reads from both (old preferred)**
```python
def get_user_email(user):
    return user.email or user.legacy_email  # Fallback to old
```

**Step 2: Deploy code that writes to both**
```python
def save_email(user_id, email):
    db.execute("UPDATE users SET email = ?, legacy_email = ? WHERE id = ?",
               email, email, user_id)
```

**Step 3: Backfill data**

**Step 4: Deploy code that reads from new only**

**Step 5: Deploy code that stops writing to old**

**Step 6: Remove old column**

## Database Schema Versioning

### Migration Tools Approach

**Flyway/Liquibase Pattern:**
```
migrations/
  V1__initial_schema.sql
  V2__add_user_email.sql
  V3__add_email_verification.sql
  V4__remove_legacy_email.sql
```

**Rails/Django/Alembic Pattern:**
```python
# migrations/0042_add_email_verified.py
def upgrade():
    op.add_column('users',
        sa.Column('email_verified', sa.Boolean(),
                  nullable=True, server_default='false'))

def downgrade():
    op.drop_column('users', 'email_verified')
```

### Migration Best Practices

1. **Always make migrations reversible**
   - Include `DOWN` migration
   - Test rollback before deploying

2. **Never alter data in schema migrations**
   - Schema changes in one migration
   - Data backfills in separate task

3. **Use transactions carefully**
   ```sql
   -- Good: Fast operations in transaction
   BEGIN;
   ALTER TABLE users ADD COLUMN status VARCHAR(20) DEFAULT 'active';
   COMMIT;

   -- Bad: Slow backfill in transaction (locks table)
   BEGIN;
   UPDATE users SET status = 'active' WHERE status IS NULL; -- Millions of rows!
   COMMIT;
   ```

4. **Add indexes concurrently (PostgreSQL)**
   ```sql
   -- Blocks writes
   CREATE INDEX idx_users_email ON users(email);

   -- Doesn't block (preferred)
   CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
   ```

## Backward & Forward Compatibility

### Column Addition

**Bad (breaks old code):**
```sql
ALTER TABLE orders ADD COLUMN shipping_address TEXT NOT NULL;
```

**Good (compatible):**
```sql
-- Step 1: Add nullable
ALTER TABLE orders ADD COLUMN shipping_address TEXT;

-- Step 2: Deploy code that populates it
-- Step 3: Make NOT NULL after backfill
ALTER TABLE orders ALTER COLUMN shipping_address SET NOT NULL;
```

### Column Removal

**The 3-Deploy Process:**

**Deploy 1: Stop writing to column**
```python
# Remove code that sets old_field
def create_order(data):
    db.execute("INSERT INTO orders (customer_id, total) VALUES (?, ?)",
               data['customer_id'], data['total'])
    # Removed: , old_field
```

**Deploy 2: Stop reading from column**
```python
# Remove code that accesses old_field
def get_order(order_id):
    row = db.query("SELECT customer_id, total FROM orders WHERE id = ?", order_id)
    # Removed: row['old_field']
```

**Deploy 3: Drop column**
```sql
ALTER TABLE orders DROP COLUMN old_field;
```

### Column Rename

**Never rename directly in production!**

Instead, use the expand-contract pattern:

```sql
-- 1. Add new column
ALTER TABLE users ADD COLUMN full_name TEXT;

-- 2. Dual-write to both
UPDATE users SET full_name = name;

-- 3. Backfill
UPDATE users SET full_name = name WHERE full_name IS NULL;

-- 4. Switch reads to new column
-- 5. Stop writing to old column
-- 6. Drop old column
ALTER TABLE users DROP COLUMN name;
```

## Data Type Changes

### Example: Change `price` from INT to DECIMAL

**Naive approach (dangerous):**
```sql
ALTER TABLE products ALTER COLUMN price TYPE DECIMAL(10,2);
-- Locks table, rewrites entire table!
```

**Production approach:**
```sql
-- 1. Add new column
ALTER TABLE products ADD COLUMN price_decimal DECIMAL(10,2);

-- 2. Backfill in batches
UPDATE products
SET price_decimal = price / 100.0
WHERE id >= ? AND id < ?;

-- 3. Deploy code to read price_decimal, fallback to price
-- 4. Deploy code to write to both
-- 5. Drop old column
ALTER TABLE products DROP COLUMN price;

-- 6. Rename new column
ALTER TABLE products RENAME COLUMN price_decimal TO price;
```

## Large-Scale Data Migrations

### Batch Processing Pattern

```python
def migrate_user_data(batch_size=1000):
    last_id = 0

    while True:
        # Process in batches to avoid long locks
        rows = db.execute("""
            SELECT id, old_data
            FROM users
            WHERE id > ?
            ORDER BY id
            LIMIT ?
        """, last_id, batch_size)

        if not rows:
            break

        for row in rows:
            new_data = transform(row['old_data'])
            db.execute("""
                UPDATE users
                SET new_data = ?
                WHERE id = ?
            """, new_data, row['id'])

        last_id = rows[-1]['id']

        # Throttle to avoid overwhelming DB
        time.sleep(0.1)
```

### Online vs Offline Migrations

**Online (zero-downtime):**
- Use expand-contract pattern
- Batch processing with throttling
- Monitor replication lag
- Takes days/weeks for large datasets

**Offline (maintenance window):**
- Simpler, faster
- Direct ALTER TABLE commands
- Acceptable for small tables or off-peak
- Use for startups or internal tools

## Handling Breaking Changes

### API Contract Evolution

```python
# Version 1: Returns user_name
{
    "user_name": "John Doe",
    "email": "john@example.com"
}

# Version 2: Returns first_name, last_name
# Bad: Breaking change!
{
    "first_name": "John",
    "last_name": "Doe",
    "email": "john@example.com"
}

# Good: Include both during transition
{
    "user_name": "John Doe",  # Deprecated but still present
    "first_name": "John",
    "last_name": "Doe",
    "email": "john@example.com"
}
```

### Deprecation Warnings

```python
class UserAPI:
    def get_user_name(self):
        warnings.warn(
            "get_user_name() is deprecated. Use get_full_name() instead.",
            DeprecationWarning,
            stacklevel=2
        )
        return f"{self.first_name} {self.last_name}"
```

## Blue-Green Deployments for Data

### Pattern: Dual-Database Migration

**Scenario: Migrating from MongoDB to PostgreSQL**

1. **Both databases running (GREEN = MongoDB, BLUE = PostgreSQL)**
2. **Dual-write to both**
   ```python
   def create_user(data):
       mongo_db.users.insert_one(data)  # Green (current)
       postgres_db.execute("INSERT INTO users (...) VALUES (...)")  # Blue (new)
   ```
3. **Backfill historical data to PostgreSQL**
4. **Switch reads to PostgreSQL (with fallback)**
   ```python
   def get_user(user_id):
       user = postgres_db.query("SELECT * FROM users WHERE id = ?", user_id)
       if not user:
           user = mongo_db.users.find_one({"id": user_id})  # Fallback
       return user
   ```
5. **Monitor for discrepancies**
6. **Stop writing to MongoDB**
7. **Decommission MongoDB**

## Testing Migrations

### Pre-Production Checklist

```bash
# 1. Test on production-sized dataset (staging)
# 2. Measure migration time
time ./run_migration.sh

# 3. Test rollback
./rollback_migration.sh

# 4. Check replication lag
# PostgreSQL
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;

# MySQL
SHOW SLAVE STATUS\G

# 5. Validate data integrity
SELECT COUNT(*) FROM users WHERE email_verified IS NULL AND email IS NOT NULL;
```

### Shadow Testing

Run new code path alongside old, compare results:

```python
def get_user_orders(user_id):
    # Old path
    old_result = legacy_query(user_id)

    # New path
    new_result = optimized_query(user_id)

    # Compare and log differences
    if old_result != new_result:
        log_difference(old_result, new_result)

    # Return old result (safe)
    return old_result
```

## Common Pitfalls

1. **Adding NOT NULL column without default**
   - ❌ Breaks old code instantly
   - ✅ Add nullable first, then backfill, then NOT NULL

2. **Renaming in one step**
   - ❌ All code breaks at once
   - ✅ Add new column, dual-write, switch reads, drop old

3. **Long-running transactions**
   - ❌ `BEGIN; UPDATE 10M rows; COMMIT;`
   - ✅ Batch updates outside transaction

4. **Forgetting replication lag**
   - ❌ Primary updated, replica serves stale data
   - ✅ Monitor lag, throttle migrations

5. **No rollback plan**
   - ❌ Migration fails, data corrupted
   - ✅ Test rollback, keep old columns until confident

## Senior Engineer Checklist

When planning a migration:

- [ ] Can I do this without downtime?
- [ ] What's my rollback strategy?
- [ ] How long will backfill take?
- [ ] Will this lock tables?
- [ ] How will I validate success?
- [ ] What's the blast radius if it fails?
- [ ] Have I tested on production-sized data?
- [ ] Is my migration idempotent (safe to re-run)?
- [ ] How will I monitor progress?
- [ ] What's the performance impact on live traffic?

## Real-World Examples

### GitHub: Migrating to UUID Primary Keys
- Took **years** to migrate all tables
- Added UUID columns alongside integer IDs
- Dual-indexed both
- Gradually migrated foreign keys
- Blog: https://github.blog/

### Stripe: Migrating Payment Data
- Zero-downtime migration of billions of records
- Used shadow writes and reads
- Validated data consistency before cutover
- Maintained rollback capability for months

### Slack: Database Sharding
- Moved from single DB to sharded architecture
- Used proxy layer to route queries
- Migrated teams one-by-one
- Took over a year for complete migration

## Key Takeaways

1. **Never rush migrations** - Plan for weeks/months, not days
2. **Always be able to rollback** - Keep old structures until confident
3. **Test on production-scale data** - Staging must match production size
4. **Monitor everything** - Replication lag, query performance, error rates
5. **Communicate broadly** - Migrations affect entire engineering org
