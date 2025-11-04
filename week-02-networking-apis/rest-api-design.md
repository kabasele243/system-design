# REST API Design

## Learning Objectives
- Understand RESTful principles
- Learn HTTP methods and status codes
- Design clean, intuitive APIs

## Notes

### What is REST?

**REST** = **RE**presentational **S**tate **T**ransfer

A standard way to design APIs that allows clients (mobile, web, other services) to interact with server resources over HTTP.

**Key Concept: Resources**
- Resource = Any "thing" your system manages (users, posts, comments, photos, orders)
- Each resource gets a URL: `/users`, `/posts`, `/comments`

---

### RESTful Principles

1. **Use nouns for resources, not verbs**
   - ✅ Good: `GET /users/123`, `POST /posts`
   - ❌ Bad: `GET /getUser?id=123`, `POST /createPost`

2. **Use HTTP methods to represent actions**
   - GET = Read
   - POST = Create
   - PUT = Replace
   - PATCH = Update
   - DELETE = Delete

3. **Use plural names for collections**
   - ✅ Good: `/users`, `/posts`
   - ❌ Bad: `/user`, `/post`

4. **Nest resources to show relationships**
   - `GET /users/123/posts` → All posts by user 123
   - `GET /posts/456/comments` → All comments on post 456

5. **Use query parameters for filtering, sorting, pagination**
   - `GET /posts?user_id=789` → Filter
   - `GET /posts?sort=likes&order=desc` → Sort
   - `GET /posts?page=2&limit=20` → Pagination

---

### HTTP Methods (CRUD Operations)

| HTTP Method | Purpose | Example | Idempotent? |
|-------------|---------|---------|-------------|
| **GET** | Read/Retrieve data | `GET /users/123` → Get user 123 | ✅ Yes |
| **POST** | Create new resource | `POST /users` → Create new user | ❌ No |
| **PUT** | Replace entire resource | `PUT /users/123` → Update user 123 | ✅ Yes |
| **PATCH** | Partial update | `PATCH /users/123` → Update just email | ⚠️ Maybe |
| **DELETE** | Delete resource | `DELETE /users/123` → Delete user 123 | ✅ Yes |

**Idempotency** = Calling multiple times has same effect as calling once
- **POST is NOT idempotent**: Call 3 times → creates 3 resources
- **PUT is idempotent**: Call 3 times → same resource state
- **DELETE is idempotent**: Call 3 times → resource still deleted (404 after first)

**Real-world use:** Idempotency keys prevent duplicate payments/orders from network issues
```
POST /payments
Headers: {Idempotency-Key: "order-xyz-123"}
Body: {amount: 100}
```

---

### Instagram API Example

**1. Get your feed:**
```
GET /feed
Response: [{post1}, {post2}, {post3}...]
```

**2. Create a new post:**
```
POST /posts
Body: {image_url: "...", caption: "Hello!"}
Response: {post_id: 456, created_at: "..."}
```

**3. Like a post:**
```
POST /posts/456/likes
Response: {likes: 101}
```

**4. Get post details:**
```
GET /posts/456
Response: {post_id: 456, caption: "Hello!", likes: 101, ...}
```

**5. Update caption:**
```
PATCH /posts/456
Body: {caption: "Updated caption"}
```

**6. Delete a post:**
```
DELETE /posts/456
Response: 204 No Content
```

---

### Path Parameters vs Query Parameters

**Path Parameters** → Identify specific resource:
```
GET /users/123        ← 123 identifies the user
GET /posts/456        ← 456 identifies the post
```

**Query Parameters** → Filter, sort, paginate:
```
GET /posts?user_id=789                    ← Filter by user
GET /posts?sort=likes&order=desc          ← Sort by likes
GET /posts?page=2&limit=20                ← Pagination
GET /posts?user_id=789&sort=date&limit=10 ← Combine filters
```

---

### RESTful Design Patterns

**1. Nested Resources (show relationship):**
```
GET /users/123/posts         ← All posts by user 123
GET /posts/456/comments      ← All comments on post 456
POST /posts/456/comments     ← Add comment to post 456
```

**2. Actions on Resources (when CRUD isn't enough):**
```
POST /posts/456/like         ← Like a post
POST /posts/456/unlike       ← Unlike a post
POST /rides/789/accept       ← Driver accepts ride
POST /users/123/follow       ← Follow a user
```

**3. Versioning (backwards compatibility):**
```
GET /v1/users/123            ← Version 1 API
GET /v2/users/123            ← Version 2 API (breaking changes)
```

---

### HTTP Status Codes

**2xx Success:**
- **200 OK:** Request succeeded, returning data
  - `GET /users/123` → Returns user data
- **201 Created:** New resource created
  - `POST /users` → User created, returns new user
- **204 No Content:** Success, but no data to return
  - `DELETE /users/123` → User deleted

**4xx Client Errors:**
- **400 Bad Request:** Invalid data from client
  - Missing required fields, invalid JSON
- **401 Unauthorized:** Authentication required
  - Missing or invalid API token
- **403 Forbidden:** Authenticated but not allowed
  - User trying to delete someone else's post
- **404 Not Found:** Resource doesn't exist
  - `GET /users/999` where user 999 doesn't exist

**5xx Server Errors:**
- **500 Internal Server Error:** Server crashed
  - Unhandled exception, database down
- **503 Service Unavailable:** Server overloaded
  - Too many requests, maintenance mode

---

### REST API Best Practices

✅ **Do:**
1. Use nouns for resources (`/users`, `/posts`)
2. Use HTTP methods for actions (not verbs in URL)
3. Use plural names (`/users`, not `/user`)
4. Nest resources for relationships (`/users/123/posts`)
5. Use query parameters for filters/sorting/pagination
6. Version your API (`/v1/users`)
7. Return proper status codes (200, 201, 404, 500)
8. Use idempotency keys for critical operations (payments, orders)

❌ **Don't:**
1. Use verbs in URLs (`/getUser`, `/deletePost`)
2. Mix naming conventions (stick to one style)
3. Return HTML from API (use JSON)
4. Nest too deeply (`/users/123/posts/456/comments/789/likes`)
5. Use different URL patterns for same type of operation

---

### Twitter API Example

**Post a new tweet:**
```
POST /tweets
Body: {text: "Hello world", user_id: 123}
Response: {tweet_id: 456, created_at: "2025-11-03T10:00:00Z"}
```

**Get all tweets by user 789:**
```
GET /users/789/tweets
Response: [{tweet1}, {tweet2}, ...]
```

**Delete tweet 456:**
```
DELETE /tweets/456
Response: 204 No Content
```

**Soft delete (keep in DB, hide from users):**
```
PATCH /tweets/456
Body: {deleted: true}
Response: 200 OK
```

---

### Uber API Example

**Driver accepts ride:**
```
POST /rides/789/accept
Body: {driver_id: 456}
Response: {ride_id: 789, status: "accepted", driver_id: 456}
```

**Get passenger's rides (last 30 days):**
```
GET /users/123/rides?start_date=2025-10-04&end_date=2025-11-03
Response: {rides: [{ride1}, {ride2}...], total: 15}
```

**Passenger rates driver:**
```
POST /rides/789/rating
Body: {rating: 5, passenger_id: 123}
Response: {ride_id: 789, rating: 5}
```

---

### Pagination Best Practice

**Offset-based (simpler, but slower at high offsets):**
```
GET /posts?page=2&limit=20
Response: {
  posts: [{post1}, {post2}, ...],
  total: 450,
  page: 2,
  has_more: true
}
```

**Cursor-based (faster, better for large datasets):**
```
GET /posts?cursor=abc123&limit=20
Response: {
  posts: [{post1}, {post2}, ...],
  next_cursor: "xyz789",
  has_more: true
}
```

---

### Key Takeaways

1. **REST uses URLs to represent resources and HTTP methods to represent actions**
2. **Idempotency matters**: POST creates duplicates, PUT/DELETE are safe to retry
3. **Nested resources show relationships**: `/users/123/posts`
4. **Query parameters for filtering**: `?user_id=789&sort=date`
5. **Use proper status codes**: 200 (success), 404 (not found), 500 (server error)
6. **Idempotency keys prevent duplicate critical operations** (payments, orders)

---

## HTTP Status Codes Deep Dive

### Status Code Categories

Every HTTP response includes a 3-digit status code telling the client what happened.

---

### 1xx: Informational (Rare)
- **100 Continue**: Server received headers, client can send body
- **101 Switching Protocols**: Upgrading to WebSocket

(Almost never used in REST APIs)

---

### 2xx: Success ✅

| Code | Name | When to Use | Example |
|------|------|-------------|---------|
| **200** | OK | Request succeeded, returning data | `GET /users/123` → Returns user |
| **201** | Created | New resource created | `POST /users` → New user created |
| **202** | Accepted | Request accepted, processing async | `POST /videos/encode` → Will process later |
| **204** | No Content | Success, no data to return | `DELETE /users/123` → User deleted |

**Guidelines:**
- **200**: Default success for GET, PATCH, PUT
- **201**: Always use for POST when creating resources (include `Location` header)
- **202**: Long-running async operations (video encoding, batch jobs)
- **204**: DELETE operations or updates with no response body

---

### 3xx: Redirection

| Code | Name | When to Use |
|------|------|-------------|
| **301** | Moved Permanently | Resource permanently moved (SEO keeps link juice) |
| **302** | Found (Temporary) | Temporary redirect |
| **304** | Not Modified | Cached version is still valid (saves bandwidth) |

**ETag Caching Example:**
```
GET /posts/123
Headers: {If-None-Match: "abc123"}

If unchanged:
Response: 304 Not Modified
(Browser uses cached version, saves bandwidth!)

If changed:
Response: 200 OK
Headers: {ETag: "xyz789"}
Body: {updated post data}
```

---

### 4xx: Client Errors (Client's Fault)

| Code | Name | When to Use | Example |
|------|------|-------------|---------|
| **400** | Bad Request | Invalid data from client | Missing required field, invalid JSON |
| **401** | Unauthorized | Authentication required | No API token provided |
| **403** | Forbidden | Authenticated but not allowed | User trying to delete someone else's post |
| **404** | Not Found | Resource doesn't exist | `GET /users/999` (user doesn't exist) |
| **409** | Conflict | Request conflicts with server state | Email already registered |
| **422** | Unprocessable Entity | Valid JSON but business logic fails | Age is -5 (invalid value) |
| **429** | Too Many Requests | Rate limit exceeded | 1000 requests in 1 minute |

**Common Confusion: 401 vs 403**
- **401 Unauthorized**: "Who are you?" (no token or invalid token)
- **403 Forbidden**: "I know who you are, but you can't do this" (valid token, wrong permissions)

**Instagram Examples:**

**400 Bad Request:**
```
POST /posts
Body: {caption: "Hello!"}
Response: 400 Bad Request
{
  error: "image_url is required"
}
```

**401 Unauthorized:**
```
POST /posts
Headers: {Authorization: "Bearer invalid_token"}
Response: 401 Unauthorized
{
  error: "Invalid or expired token"
}
```

**403 Forbidden:**
```
DELETE /posts/456
(User 123 trying to delete post owned by user 789)
Response: 403 Forbidden
{
  error: "You can only delete your own posts"
}
```

**409 Conflict:**
```
POST /users
Body: {email: "john@example.com"}
Response: 409 Conflict
{
  error: "Email already registered"
}
```

**429 Too Many Requests:**
```
GET /feed
(101st request in 1 minute)
Response: 429 Too Many Requests
Headers: {Retry-After: 60}
Body: {
  error: "Rate limit exceeded. Try again in 60 seconds"
}
```

---

### 5xx: Server Errors (Server's Fault)

| Code | Name | When to Use |
|------|------|-------------|
| **500** | Internal Server Error | Unhandled exception, bug in code |
| **502** | Bad Gateway | Upstream server failed (load balancer → app server down) |
| **503** | Service Unavailable | Server overloaded or in maintenance |
| **504** | Gateway Timeout | Upstream server too slow (load balancer waited too long) |

**When You See These:**
- **500**: Bug in your code, check logs immediately
- **502**: One of your backend servers is down
- **503**: Too much traffic, need to scale or database is down
- **504**: Database query too slow, need to optimize

**Example:**
```
GET /users/123
(Database is down)
Response: 503 Service Unavailable
Headers: {Retry-After: 300}
Body: {
  error: "Service temporarily unavailable. Please try again in 5 minutes."
}
```

---

### Status Code Decision Tree

```
Request arrives
    ↓
Is the request valid? (syntax, auth, format)
    ↓ NO → 4xx Client Error
        - No auth token? → 401
        - Invalid token? → 401
        - Valid token but no permission? → 403
        - Resource not found? → 404
        - Invalid data format? → 400
        - Rate limited? → 429
        - Already exists? → 409
    ↓ YES
Can the server process it?
    ↓ NO → 5xx Server Error
        - Unhandled bug/exception? → 500
        - Database down? → 503
        - Server overloaded? → 503
        - Upstream timeout? → 504
    ↓ YES → 2xx Success
        - Returning data? → 200
        - Created new resource? → 201
        - Async processing? → 202
        - No response body? → 204
```

---

### Important HTTP Headers

**Request Headers:**

| Header | Purpose | Example |
|--------|---------|---------|
| **Authorization** | Authentication token | `Bearer abc123` |
| **Content-Type** | Body format | `application/json` |
| **Accept** | Desired response format | `application/json` |
| **User-Agent** | Client info | `Mozilla/5.0...` |
| **If-None-Match** | ETag for caching | `"abc123"` |

**Response Headers:**

| Header | Purpose | Example |
|--------|---------|---------|
| **Content-Type** | Response format | `application/json` |
| **Cache-Control** | Caching rules | `max-age=3600` |
| **ETag** | Resource version | `"abc123"` |
| **Location** | New resource URL (201) | `/users/123` |
| **Retry-After** | When to retry (429, 503) | `60` (seconds) |
| **X-Rate-Limit-Limit** | Max requests allowed | `100` |
| **X-Rate-Limit-Remaining** | Requests left | `23` |
| **X-Rate-Limit-Reset** | When limit resets | `1699000000` (Unix timestamp) |

---

### HTTP Methods: Safe vs Idempotent

| Method | Safe? | Idempotent? | Meaning |
|--------|-------|-------------|---------|
| **GET** | ✅ Yes | ✅ Yes | Read only, no side effects |
| **POST** | ❌ No | ❌ No | Creates new resource each time |
| **PUT** | ❌ No | ✅ Yes | Modifies but same result every time |
| **PATCH** | ❌ No | ⚠️ Maybe | Depends on implementation |
| **DELETE** | ❌ No | ✅ Yes | Deletes once, 404 after |
| **HEAD** | ✅ Yes | ✅ Yes | Like GET but no body (check if exists) |
| **OPTIONS** | ✅ Yes | ✅ Yes | Returns allowed methods (CORS) |

**Safe** = No side effects (read-only)
**Idempotent** = Same result when called multiple times

---

### CORS (Cross-Origin Resource Sharing)

**Problem:**
- Frontend at `app.example.com`
- API at `api.example.com`
- Browser blocks requests (security!)

**Solution: CORS Headers**

**Simple request (GET):**
```
GET /users/123
Origin: https://app.example.com

Response: 200 OK
Headers:
  Access-Control-Allow-Origin: https://app.example.com
  Access-Control-Allow-Credentials: true
```

**Preflight request (POST/PUT/DELETE):**
```
OPTIONS /users/123
Origin: https://app.example.com
Access-Control-Request-Method: DELETE

Response: 200 OK
Headers:
  Access-Control-Allow-Origin: https://app.example.com
  Access-Control-Allow-Methods: GET, POST, PUT, DELETE
  Access-Control-Allow-Headers: Authorization, Content-Type
  Access-Control-Max-Age: 86400
```

---

### Content Negotiation

Same URL, different formats based on `Accept` header:

```
GET /users/123
Accept: application/json
Response: 200 OK
Content-Type: application/json
{"name": "John", "email": "john@example.com"}

GET /users/123
Accept: application/xml
Response: 200 OK
Content-Type: application/xml
<user><name>John</name><email>john@example.com</email></user>
```

---

### Rate Limiting Headers (Twitter Example)

```
GET /tweets
Headers: {Authorization: "Bearer abc123"}

Response: 200 OK
Headers:
  X-Rate-Limit-Limit: 100
  X-Rate-Limit-Remaining: 23
  X-Rate-Limit-Reset: 1699000000

(After 100 requests)
Response: 429 Too Many Requests
Headers:
  Retry-After: 60
Body:
  {"error": "Rate limit exceeded. Try again in 60 seconds"}
```

---

### Status Code Summary

**Remember:**
1. **2xx = Success**: 200 (OK), 201 (Created), 204 (No Content)
2. **4xx = Client's fault**: 400 (Bad Request), 401 (No Auth), 403 (No Permission), 404 (Not Found), 429 (Rate Limit), 409 (Conflict)
3. **5xx = Server's fault**: 500 (Bug), 503 (Down/Overloaded), 504 (Timeout)
4. **401 vs 403**: 401 = "Who are you?", 403 = "I know you, but no"
5. **ETag + 304 saves bandwidth**
6. **429 for rate limiting** with `Retry-After` header
7. **503 is temporary**, 500 is a bug

---

## Practice: Design a To-Do List API

Design a REST API for a simple to-do list application (CRUD operations).

### Your API Design:

**Create a todo:**
```
Method:
Endpoint:
Body:
Response:
```

**Get all todos:**
```
Method:
Endpoint:
Response:
```

**Get a specific todo:**
```
Method:
Endpoint:
Response:
```

**Update a todo:**
```
Method:
Endpoint:
Body:
Response:
```

**Delete a todo:**
```
Method:
Endpoint:
Response:
```

## Resources & References
