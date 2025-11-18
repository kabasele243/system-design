# API Versioning & Backward Compatibility

## Why API Versioning Matters

As a senior engineer, you'll face this reality: **your API is a contract with clients you don't control**. Mobile apps can't be force-updated. Third-party integrations take months to change. Breaking that contract breaks revenue.

## Versioning Strategies

### 1. URL Path Versioning

**Most common and explicit**

```
GET /v1/users/123
GET /v2/users/123
GET /v3/users/123
```

**Pros:**
- Extremely clear and visible
- Easy to route (load balancer level)
- Easy to cache (different URLs)
- Works with all HTTP clients

**Cons:**
- URLs change
- Resource links become versioned
- HATEOAS links need version awareness

**When to use:** Public APIs, third-party integrations, mobile apps

**Example: Stripe API**
```bash
curl https://api.stripe.com/v1/charges
```

### 2. Header Versioning

```http
GET /users/123
Accept: application/vnd.myapi.v2+json

or

GET /users/123
API-Version: 2023-11-01
```

**Pros:**
- Clean URLs
- Same endpoint, different versions
- Follows REST principles better

**Cons:**
- Less visible (harder to debug)
- Caching is tricky (Vary: Accept header)
- Not all HTTP clients make headers easy

**When to use:** Internal APIs, versioning by date (Stripe does this too)

**Example: GitHub API**
```bash
curl https://api.github.com/user \
  -H "Accept: application/vnd.github.v3+json"
```

### 3. Query Parameter Versioning

```
GET /users/123?version=2
GET /users/123?api-version=2023-11-01
```

**Pros:**
- Easy to test in browser
- Doesn't change URL structure
- Simple to implement

**Cons:**
- Pollutes query namespace
- Caching complexity
- Feels hacky

**When to use:** Internal tools, admin panels, quick prototypes

### 4. Content Negotiation

```http
GET /users/123
Accept: application/json; version=2
```

**Pros:**
- HTTP-spec compliant
- Semantic versioning

**Cons:**
- Complex to implement
- Not widely understood
- Hard to debug

**When to use:** Rare, academic APIs

## Choosing a Strategy

| **Scenario**                | **Best Strategy**           |
|-----------------------------|-----------------------------|
| Public API                  | URL Path (`/v1/`)           |
| Internal microservices      | Header (date-based)         |
| Mobile app backend          | URL Path                    |
| Third-party integrations    | URL Path                    |
| Rapid prototyping           | Query parameter             |
| Date-based releases         | Header (`API-Version: date`)|

**Senior advice:** For production systems serving external clients, **URL path versioning** wins. Clarity > elegance.

## Semantic Versioning for APIs

Follow **MAJOR.MINOR.PATCH** (but simplified for APIs):

- **MAJOR (v1, v2, v3):** Breaking changes
  - Removed fields
  - Changed types
  - Changed behavior
  - Renamed endpoints

- **MINOR (implicit or date-based):** Backward-compatible additions
  - New fields
  - New endpoints
  - New optional parameters

- **PATCH (rarely exposed):** Bug fixes
  - Doesn't change contract
  - Internal implementation fixes

**Example:**
```
v1: 2020-01-01 - Launch
v1.1: 2020-06-01 - Added "created_at" field (backward compatible)
v1.2: 2020-09-01 - Added /users/:id/orders endpoint (new feature)
v2: 2021-01-01 - Removed "legacy_id", changed pagination (BREAKING)
```

## Backward Compatibility Patterns

### Adding Fields (Safe)

**v1 Response:**
```json
{
  "id": 123,
  "name": "John Doe"
}
```

**v2 Response (backward compatible):**
```json
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",  // New field
  "created_at": "2023-11-13T10:00:00Z"  // New field
}
```

**Why safe?** Old clients ignore unknown fields.

### Adding Optional Parameters (Safe)

```python
# v1: Only required param
GET /users?limit=10

# v2: Add optional filter (backward compatible)
GET /users?limit=10&status=active
```

**Implementation:**
```python
def get_users(limit: int, status: str = None):  # Default makes it optional
    query = "SELECT * FROM users"
    if status:
        query += f" WHERE status = '{status}'"
    query += f" LIMIT {limit}"
    return db.execute(query)
```

### Changing Field Types (Breaking)

**Bad (breaks v1 clients):**
```json
// v1
{
  "price": 1999  // Integer (cents)
}

// v2 (BREAKING!)
{
  "price": "19.99"  // String (dollars)
}
```

**Good (maintain both):**
```json
// v1
{
  "price": 1999
}

// v2
{
  "price_cents": 1999,  // Keep old
  "price": "19.99"      // Add new
}
```

Or use separate versions:
```python
if api_version == 1:
    return {"price": 1999}
elif api_version == 2:
    return {"price": "19.99", "currency": "USD"}
```

### Removing Fields (Breaking)

**Never remove directly.** Use deprecation:

**Phase 1: Mark as deprecated (v1.5)**
```json
{
  "id": 123,
  "legacy_user_id": 999,  // Deprecated: Use 'id' instead
  "name": "John Doe"
}

// Also include in response headers
HTTP/1.1 200 OK
Warning: 299 - "Field 'legacy_user_id' is deprecated. Use 'id' instead. Will be removed in v2."
```

**Phase 2: Remove in next major version (v2)**
```json
{
  "id": 123,
  "name": "John Doe"
  // legacy_user_id removed
}
```

## Deprecation Strategy

### The 3-Phase Deprecation

**Phase 1: Announce (6+ months before removal)**
- Document in API docs
- Add `Warning` header to responses
- Email API consumers
- Add deprecation notice in developer dashboard

```http
Warning: 299 - "This endpoint is deprecated. Migrate to /v2/users by 2024-06-01."
Sunset: Sat, 01 Jun 2024 23:59:59 GMT
```

**Phase 2: Freeze (no new features)**
- Stop adding features to deprecated version
- Only security/critical bug fixes
- Continue monitoring usage

**Phase 3: Sunset (remove after deadline)**
- Return 410 Gone for deprecated endpoints
- Redirect to migration guide

```http
HTTP/1.1 410 Gone
Content-Type: application/json

{
  "error": "API v1 has been sunset",
  "message": "Please migrate to v2. See https://api.example.com/v1-migration",
  "sunset_date": "2024-06-01"
}
```

### Monitoring Deprecation

```python
from prometheus_client import Counter

deprecated_api_calls = Counter(
    'api_deprecated_calls_total',
    'Calls to deprecated API endpoints',
    ['version', 'endpoint', 'client_id']
)

@app.route('/v1/users')
def get_users_v1():
    deprecated_api_calls.labels(
        version='v1',
        endpoint='/users',
        client_id=request.headers.get('X-Client-ID')
    ).inc()

    # Send warning header
    response.headers['Warning'] = '299 - "v1 is deprecated"'

    return jsonify(users)
```

## Handling Breaking Changes

### Change: Pagination Format

**v1 (offset-based):**
```json
GET /users?page=2&limit=10

{
  "users": [...],
  "page": 2,
  "total_pages": 50
}
```

**v2 (cursor-based, better for scale):**
```json
GET /users?cursor=abc123&limit=10

{
  "users": [...],
  "next_cursor": "xyz789",
  "has_more": true
}
```

**Implementation:**
```python
class UserAPIv1:
    def get_users(self, page=1, limit=10):
        offset = (page - 1) * limit
        users = db.query("SELECT * FROM users LIMIT ? OFFSET ?", limit, offset)
        total = db.query("SELECT COUNT(*) FROM users")[0]
        return {
            "users": users,
            "page": page,
            "total_pages": math.ceil(total / limit)
        }

class UserAPIv2:
    def get_users(self, cursor=None, limit=10):
        if cursor:
            users = db.query("""
                SELECT * FROM users
                WHERE id > ?
                ORDER BY id
                LIMIT ?
            """, decode_cursor(cursor), limit)
        else:
            users = db.query("SELECT * FROM users ORDER BY id LIMIT ?", limit)

        next_cursor = encode_cursor(users[-1]['id']) if users else None
        return {
            "users": users,
            "next_cursor": next_cursor,
            "has_more": len(users) == limit
        }
```

### Change: Error Response Format

**v1:**
```json
{
  "error": "User not found"
}
```

**v2 (RFC 7807 Problem Details):**
```json
{
  "type": "https://api.example.com/errors/user-not-found",
  "title": "User Not Found",
  "status": 404,
  "detail": "User with ID 123 does not exist",
  "instance": "/users/123"
}
```

## Version Routing Implementation

### Nginx Routing

```nginx
# Route based on URL path
location /v1/ {
    proxy_pass http://api-v1-service:8000/;
}

location /v2/ {
    proxy_pass http://api-v2-service:8001/;
}

# Route based on header
location /api/ {
    if ($http_api_version = "1") {
        proxy_pass http://api-v1-service:8000/;
    }
    if ($http_api_version = "2") {
        proxy_pass http://api-v2-service:8001/;
    }
}
```

### Application-Level Routing (Express.js)

```javascript
const express = require('express');
const app = express();

// URL-based versioning
app.use('/v1', require('./routes/v1'));
app.use('/v2', require('./routes/v2'));

// Header-based versioning
app.use('/api', (req, res, next) => {
    const version = req.headers['api-version'] || '1';

    if (version === '1') {
        require('./routes/v1')(req, res, next);
    } else if (version === '2') {
        require('./routes/v2')(req, res, next);
    } else {
        res.status(400).json({
            error: 'Unsupported API version',
            supported_versions: ['1', '2']
        });
    }
});
```

### Flask Example

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

def get_api_version():
    # Check URL path
    if request.path.startswith('/v1/'):
        return 1
    elif request.path.startswith('/v2/'):
        return 2
    # Check header
    return int(request.headers.get('API-Version', 1))

@app.route('/v1/users/<int:user_id>')
def get_user_v1(user_id):
    user = db.get_user(user_id)
    return jsonify({
        'id': user.id,
        'name': user.name
    })

@app.route('/v2/users/<int:user_id>')
def get_user_v2(user_id):
    user = db.get_user(user_id)
    return jsonify({
        'id': user.id,
        'name': user.name,
        'email': user.email,        # New field
        'created_at': user.created  # New field
    })
```

## Date-Based Versioning (Stripe Style)

```python
# Client specifies a date, gets API as it was on that date
GET /users/123
Stripe-Version: 2023-11-13

# Implementation
API_CHANGES = {
    '2023-01-01': {
        'users': {
            'fields': ['id', 'name'],
            'pagination': 'offset'
        }
    },
    '2023-11-13': {
        'users': {
            'fields': ['id', 'name', 'email', 'created_at'],
            'pagination': 'cursor'
        }
    }
}

def get_user(user_id, api_date):
    user = db.get_user(user_id)
    schema = get_schema_for_date('users', api_date)

    response = {}
    for field in schema['fields']:
        response[field] = getattr(user, field)

    return response
```

**Pros of date-based versioning:**
- No breaking changes ever (clients pin a date)
- Smooth evolution
- Clients upgrade on their timeline

**Cons:**
- Complex to maintain (every date is a version)
- Need to track changes over time
- Testing explosion

## Versioning in Microservices

### Service-to-Service Versioning

```python
# User Service calls Order Service
# Uses content negotiation

import requests

def get_user_orders(user_id, api_version=2):
    response = requests.get(
        f'http://order-service/users/{user_id}/orders',
        headers={'Accept': f'application/vnd.orders.v{api_version}+json'}
    )
    return response.json()
```

### Schema Evolution with Protobuf (gRPC)

```protobuf
// v1
message User {
  int64 id = 1;
  string name = 2;
}

// v2 (backward compatible - only additions)
message User {
  int64 id = 1;
  string name = 2;
  string email = 3;       // New field (optional)
  int64 created_at = 4;   // New field (optional)
}

// Clients using v1 definition will ignore fields 3, 4
```

**Protobuf's golden rule:** Never change field numbers, never reuse them.

## API Gateway Version Management

### Kong Example

```yaml
services:
  - name: user-service-v1
    url: http://user-v1:8000
    routes:
      - paths:
          - /v1/users

  - name: user-service-v2
    url: http://user-v2:8001
    routes:
      - paths:
          - /v2/users
```

### AWS API Gateway

```json
{
  "swagger": "2.0",
  "basePath": "/v1",
  "paths": {
    "/users/{id}": {
      "x-amazon-apigateway-integration": {
        "uri": "http://user-service-v1.internal/users/{id}",
        "httpMethod": "GET"
      }
    }
  }
}
```

## Client SDK Versioning

**Package multiple versions:**
```python
# client-sdk/
#   v1/
#     __init__.py
#     users.py
#   v2/
#     __init__.py
#     users.py

from myapi.v1 import UsersClient as UsersClientV1
from myapi.v2 import UsersClient as UsersClientV2

# Client code
client_v1 = UsersClientV1(api_key="...")
client_v2 = UsersClientV2(api_key="...")
```

## Testing Multiple Versions

```python
import pytest

@pytest.mark.parametrize("api_version", [1, 2])
def test_get_user(api_version):
    response = requests.get(
        f'/v{api_version}/users/123',
        headers={'Authorization': 'Bearer token'}
    )

    assert response.status_code == 200

    data = response.json()

    # Common fields across all versions
    assert 'id' in data
    assert 'name' in data

    # Version-specific fields
    if api_version >= 2:
        assert 'email' in data
        assert 'created_at' in data
```

## Real-World Examples

### **Stripe**: Date-based versioning
- Pin API version by date: `2023-11-13`
- Never breaks existing integrations
- Clients upgrade on their timeline

### **GitHub**: Header-based versioning
- `Accept: application/vnd.github.v3+json`
- Supports v3 and v4 (GraphQL) simultaneously

### **Twitter**: URL path versioning
- `/1.1/` and `/2/` endpoints
- Clear separation between old and new

### **AWS**: Service-specific versioning
- Each service has its own version scheme
- S3: No version in URL
- EC2: API version as parameter `?Version=2016-11-15`

## Common Pitfalls

1. **Versioning too late**
   - ❌ Realize need after launch
   - ✅ Plan versioning from day 1

2. **Too many active versions**
   - ❌ Supporting v1, v2, v3, v4 simultaneously
   - ✅ Support max 2 versions, deprecate old

3. **No deprecation timeline**
   - ❌ "v1 is deprecated" (no date)
   - ✅ "v1 sunsets 2024-06-01"

4. **Breaking changes in minor versions**
   - ❌ v1.1 removes fields
   - ✅ Only add in minor, break in major

5. **Inconsistent versioning across services**
   - ❌ User service uses URL, Order service uses headers
   - ✅ Company-wide standard

## Decision Framework

**When to create a new version:**
- Removing fields
- Changing field types
- Changing response structure
- Changing authentication
- Changing rate limits (if tighter)

**When NOT to create new version:**
- Adding optional fields
- Adding new endpoints
- Performance improvements
- Bug fixes
- Looser rate limits

## Key Takeaways

1. **URL path versioning** for external APIs (clarity wins)
2. **Deprecate gracefully** (6+ months notice)
3. **Support max 2 versions** (current + previous)
4. **Add, don't change** (new fields are safe, changed fields break)
5. **Monitor usage** (know when it's safe to sunset)
6. **Version from day 1** (adding versioning later is painful)
