# Multi-Tenancy Architecture

## Overview

Multi-tenancy is the architectural pattern where a single application instance serves multiple customers (tenants). As a senior engineer, you need to balance isolation, resource sharing, and cost-efficiency.

## What is a Tenant?

**Tenant = Customer/Organization using your SaaS product**

Examples:
- **Salesforce:** Each company is a tenant
- **Slack:** Each workspace is a tenant
- **Shopify:** Each store is a tenant
- **AWS:** Each account is a tenant

## Single-Tenant vs Multi-Tenant

### Single-Tenant (Dedicated)

```
Tenant A → App Instance A → Database A
Tenant B → App Instance B → Database B
Tenant C → App Instance C → Database C
```

**Pros:**
- Complete isolation (security, performance)
- Custom configurations per tenant
- Easy compliance (data residency)

**Cons:**
- High cost (N tenants = N infrastructures)
- Operational overhead (manage N deployments)
- Slow to onboard (provision new infrastructure)

### Multi-Tenant (Shared)

```
Tenant A ┐
Tenant B ├→ App Instance → Shared Database
Tenant C ┘
```

**Pros:**
- Cost-efficient (shared resources)
- Easy to scale (add tenants without new infra)
- Centralized updates (deploy once)

**Cons:**
- Security risks (tenant A could access tenant B's data)
- Noisy neighbor (tenant A's load affects tenant B)
- Complex isolation logic

## Multi-Tenancy Data Models

### 1. Shared Database, Shared Schema

**All tenants in same tables, distinguished by `tenant_id` column**

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    tenant_id INT NOT NULL,  -- Key field
    email VARCHAR(255),
    name VARCHAR(255)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);

-- Query for tenant A
SELECT * FROM users WHERE tenant_id = 123;
```

**Pros:**
- Simplest architecture
- Lowest cost (maximum resource sharing)
- Easy to add tenants (just insert data)

**Cons:**
- **Security risk:** Forgot `WHERE tenant_id = ?` → leak all data!
- **Performance:** One tenant's large dataset slows everyone
- **No customization:** All tenants share same schema
- **Compliance:** Can't isolate data by region

**Best for:** Small tenants, similar usage patterns, low security requirements

### 2. Shared Database, Separate Schemas

**Each tenant has own schema in same database**

```sql
-- Tenant A
CREATE SCHEMA tenant_123;
CREATE TABLE tenant_123.users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255),
    name VARCHAR(255)
);

-- Tenant B
CREATE SCHEMA tenant_456;
CREATE TABLE tenant_456.users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255),
    name VARCHAR(255)
);

-- Query for tenant A
SELECT * FROM tenant_123.users;
```

**Pros:**
- Better isolation than shared schema
- Can customize schema per tenant
- Easier to backup/restore per tenant

**Cons:**
- Database size limits (PostgreSQL: 1000s of schemas OK)
- Cross-tenant queries harder
- Migrations complex (run for each schema)

**Best for:** Medium tenants, need schema customization

### 3. Separate Databases (Database per Tenant)

**Each tenant has dedicated database**

```
Tenant A → Database A
Tenant B → Database B
Tenant C → Database C
```

**Connection routing:**
```python
def get_database_connection(tenant_id):
    db_config = TENANT_DB_MAP[tenant_id]  # { 123: "db-tenant-a", 456: "db-tenant-b" }
    return connect(db_config)

# Query
db = get_database_connection(tenant_id)
users = db.query("SELECT * FROM users")  # No tenant_id needed!
```

**Pros:**
- Strong isolation (security, performance)
- Easy backup/restore per tenant
- Can host in different regions (compliance)
- Can scale databases independently

**Cons:**
- High cost (N databases)
- Complex connection pooling
- Cross-tenant analytics hard
- Schema migrations for all databases

**Best for:** Large tenants, strict isolation, compliance needs

## Hybrid Approach (Tiered Multi-Tenancy)

**Small tenants share, large tenants get dedicated**

```
Free/Small Tenants → Shared DB (Schema 1)
Medium Tenants     → Shared DB (Separate schemas)
Enterprise Tenants → Dedicated DB per tenant
```

**Example: Slack**
- Small workspaces: Shared database
- Enterprise customers: Dedicated database

## Tenant Isolation Strategies

### Application-Level Isolation

**Enforce tenant_id in code**

```python
class TenantAwareQuery:
    def __init__(self, tenant_id):
        self.tenant_id = tenant_id

    def get_users(self):
        # Automatically adds WHERE tenant_id = ?
        return db.query(
            "SELECT * FROM users WHERE tenant_id = ?",
            self.tenant_id
        )

    def get_user(self, user_id):
        # Even by ID, ensure tenant_id matches
        return db.query(
            "SELECT * FROM users WHERE id = ? AND tenant_id = ?",
            user_id, self.tenant_id
        )

# Usage (tenant from auth context)
tenant_query = TenantAwareQuery(current_tenant_id)
users = tenant_query.get_users()
```

**Row-Level Security (PostgreSQL):**
```sql
-- Enable RLS on table
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only see rows matching their tenant
CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.current_tenant')::int);

-- Set tenant in session
SET app.current_tenant = 123;

-- Now all queries automatically filtered
SELECT * FROM users;  -- Only returns tenant 123's users
```

### Network-Level Isolation (VPC per Tenant)

**Enterprise tenants get dedicated VPC**

```
Tenant A → VPC A → App A → Database A
Tenant B → VPC B → App B → Database B
```

**Use case:** Financial services, healthcare (strict compliance)

## Tenant Identification

### URL-Based (Subdomain)

```
https://tenant-a.myapp.com → Tenant A
https://tenant-b.myapp.com → Tenant B
```

**Routing:**
```python
@app.before_request
def identify_tenant():
    subdomain = request.host.split('.')[0]  # "tenant-a"
    tenant = Tenant.query.filter_by(subdomain=subdomain).first()
    g.tenant_id = tenant.id
```

**Pros:**
- Easy to identify tenant
- Can route to different backends per tenant

**Cons:**
- DNS/SSL cert management (wildcard cert needed)
- Harder for mobile apps (separate domains)

### Header-Based

```http
GET /users
Host: api.myapp.com
X-Tenant-ID: 123
```

```python
@app.before_request
def identify_tenant():
    tenant_id = request.headers.get('X-Tenant-ID')
    g.tenant_id = int(tenant_id)
```

**Pros:**
- Single domain
- Clean URLs

**Cons:**
- Clients must send header
- Risk of forgetting/manipulating header

### JWT-Based (Most Secure)

```python
# JWT contains tenant claim
{
    "user_id": 789,
    "tenant_id": 123,  # Embedded in token
    "exp": 1699999999
}

@app.before_request
def identify_tenant():
    token = request.headers.get('Authorization').split(' ')[1]
    payload = jwt.decode(token, SECRET_KEY)
    g.tenant_id = payload['tenant_id']
    g.user_id = payload['user_id']
```

**Pros:**
- Secure (signed token)
- Can't be manipulated

**Cons:**
- Requires JWT implementation

## Resource Allocation & Quotas

### Per-Tenant Rate Limiting

```python
from redis import Redis

redis = Redis()

def rate_limit(tenant_id, limit=1000):
    key = f"rate_limit:tenant:{tenant_id}:{datetime.now().strftime('%Y%m%d%H')}"
    count = redis.incr(key)
    redis.expire(key, 3600)  # 1 hour TTL

    if count > limit:
        raise Exception("Rate limit exceeded")

# Usage
@app.route('/api/data')
def get_data():
    rate_limit(g.tenant_id, limit=TENANT_LIMITS[g.tenant_id])
    # ... return data
```

### Storage Quotas

```python
def check_storage_quota(tenant_id, upload_size_mb):
    current_usage = db.query(
        "SELECT SUM(file_size_mb) FROM files WHERE tenant_id = ?",
        tenant_id
    )[0]

    tenant = Tenant.query.get(tenant_id)
    quota = tenant.storage_quota_mb  # e.g., 10000 MB for premium

    if current_usage + upload_size_mb > quota:
        raise Exception("Storage quota exceeded")
```

### Compute Isolation (Kubernetes Namespaces)

```yaml
# Namespace per tenant
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-123

---
# Resource limits for namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-123-quota
  namespace: tenant-123
spec:
  hard:
    requests.cpu: "4"      # Max 4 CPUs
    requests.memory: 8Gi   # Max 8GB RAM
    pods: "10"             # Max 10 pods
```

## Noisy Neighbor Problem

**Scenario:** Tenant A runs expensive query, slows down Tenant B (sharing same database)

### Solutions

**1. Query Timeouts**
```sql
SET statement_timeout = '5s';
SELECT * FROM huge_table WHERE tenant_id = 123;
```

**2. Dedicated Read Replicas for Large Tenants**
```
Tenant A (small) ┐
Tenant B (small) ├→ Primary DB → Read Replica 1 (for small tenants)
Tenant C (large) ─→ Primary DB → Read Replica 2 (dedicated for Tenant C)
```

**3. Rate Limiting (as shown above)**

**4. Async Processing for Heavy Tasks**
```python
# Don't run expensive reports synchronously
@app.route('/generate-report')
def generate_report():
    task_id = queue.enqueue(
        'generate_report_task',
        tenant_id=g.tenant_id,
        priority='low' if tenant.tier == 'free' else 'high'
    )
    return {"task_id": task_id, "status": "queued"}
```

## Cross-Tenant Features

### Analytics Across Tenants

**Challenge:** Need aggregate stats without tenant_id filter

```sql
-- Example: Total users across all tenants
SELECT tenant_id, COUNT(*) as user_count
FROM users
GROUP BY tenant_id;

-- But shared schema makes this slow (full table scan)
```

**Solution:** Separate analytics database
```
Operational DB (tenant-isolated) → CDC → Analytics DB (aggregated)
```

### Multi-Tenant Search

```json
// Elasticsearch: Route to tenant-specific index
PUT /tenant-123-products/_doc/1
{
  "name": "Widget",
  "price": 19.99
}

// Search (automatically scoped to tenant index)
GET /tenant-123-products/_search
```

## Tenant Onboarding

### Fast Onboarding (Shared Schema)

```python
def create_tenant(name, subdomain):
    tenant = Tenant.create(name=name, subdomain=subdomain)
    # No infrastructure needed, just insert row
    return tenant  # Ready instantly
```

### Slow Onboarding (Dedicated Database)

```python
def create_tenant(name, subdomain):
    # 1. Create tenant record
    tenant = Tenant.create(name=name, subdomain=subdomain)

    # 2. Provision database
    db_name = f"tenant_{tenant.id}"
    create_database(db_name)

    # 3. Run schema migrations
    run_migrations(db_name)

    # 4. Create S3 bucket
    create_s3_bucket(f"tenant-{tenant.id}-files")

    # 5. Update routing config
    update_load_balancer_routing(subdomain, db_name)

    return tenant  # Takes ~5 minutes
```

## Data Compliance & Residency

### Problem: GDPR Requires EU Data in EU

**Solution 1: Database per Region**
```
EU Tenants     → Database (Frankfurt)
US Tenants     → Database (Virginia)
APAC Tenants   → Database (Singapore)
```

**Solution 2: Tenant-Specific Database Location**
```python
TENANT_DB_LOCATIONS = {
    123: "eu-west-1",     # EU tenant
    456: "us-east-1",     # US tenant
}

def get_db_connection(tenant_id):
    region = TENANT_DB_LOCATIONS[tenant_id]
    return get_connection(region)
```

## Tenant Upgrades & Migrations

### Migrating Tenant to Dedicated Database

```python
def migrate_tenant_to_dedicated_db(tenant_id):
    # 1. Create new dedicated database
    new_db = create_database(f"tenant_{tenant_id}")

    # 2. Copy data from shared DB
    copy_data(
        from_db="shared_db",
        to_db=new_db,
        where=f"tenant_id = {tenant_id}"
    )

    # 3. Enable dual-write (new writes go to both)
    enable_dual_write(tenant_id, new_db)

    # 4. Switch reads to new DB
    switch_reads(tenant_id, new_db)

    # 5. Disable dual-write
    disable_dual_write(tenant_id)

    # 6. Delete from shared DB
    delete_from_shared_db(tenant_id)
```

## Real-World Examples

### **Salesforce**
- Shared schema for small orgs
- Dedicated instances for large enterprises
- Metadata-driven customization

### **Shopify**
- Database sharding by shop_id
- Shared infrastructure for small stores
- Dedicated "Shopify Plus" for high-volume

### **Slack**
- Small workspaces: Shared databases
- Enterprise Grid: Dedicated infrastructure
- Message sharding by channel

### **AWS**
- Each account is a tenant
- Strong isolation (VPC, IAM)
- Shared control plane, isolated data plane

## Common Pitfalls

1. **Forgetting tenant_id in queries**
   - ❌ `SELECT * FROM users WHERE id = 123`
   - ✅ `SELECT * FROM users WHERE id = 123 AND tenant_id = ?`

2. **No backup strategy per tenant**
   - ❌ Backup entire shared database
   - ✅ Backup per tenant or per schema

3. **Hard-coded limits**
   - ❌ All tenants get 1000 API calls/hour
   - ✅ Limits based on tenant tier (free/pro/enterprise)

4. **No tenant metrics**
   - ❌ Overall system metrics
   - ✅ Per-tenant metrics (usage, errors, latency)

5. **Cross-tenant data leaks**
   - ❌ Trust client-provided tenant_id
   - ✅ Get tenant from authenticated session

## Multi-Tenancy Decision Matrix

| **Factor**                | **Shared Schema** | **Shared DB, Separate Schemas** | **Database per Tenant** |
|---------------------------|-------------------|---------------------------------|-------------------------|
| Cost                      | Lowest            | Medium                          | Highest                 |
| Isolation                 | Weak              | Medium                          | Strong                  |
| Onboarding speed          | Instant           | Fast (minutes)                  | Slow (hours)            |
| Compliance (GDPR)         | Hard              | Medium                          | Easy                    |
| Tenant customization      | None              | Schema-level                    | Full                    |
| Number of tenants         | 10,000+           | 1,000+                          | 100+                    |

## Key Takeaways

1. **Start with shared schema** (simplest, cheapest)
2. **Add isolation as you grow** (migrate large tenants to dedicated)
3. **Always filter by tenant_id** (enforce in code, use RLS)
4. **Monitor per-tenant metrics** (catch noisy neighbors)
5. **Plan for compliance early** (data residency requirements)
6. **Hybrid approach wins** (shared for small, dedicated for large)
7. **Use JWT for tenant identification** (secure, can't be manipulated)

**Senior engineer principle:** Multi-tenancy is about balancing cost efficiency with isolation. Start simple, add complexity only when justified by revenue or compliance.
