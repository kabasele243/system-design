# URL Shortener - High-Level Design

## Step 3: High-Level Architecture

### Components

```
[Client] → [Load Balancer] → [Web Servers] → [App Servers] → [Database]
                                                    ↓
                                                [Cache (Redis)]
```

### Core Components:

1. **Load Balancer:**
   - Distributes incoming HTTP traffic across multiple web servers.
   - Handles SSL termination and enforces security policies.
   - Strategies: Round Robin, Least Connections.

2. **Web/App Servers:**
   - Stateless servers responsible for handling API requests (shorten, redirect).
   - Validates input and interacts with the database and cache.
   - Can be easily scaled horizontally by adding more instances.

3. **Database:**
   - Stores the persistent mapping between Short URLs and Long URLs.
   - Stores user data and analytics logs.
   - Requirements: High availability, eventual consistency (for analytics), strong consistency (for URL creation to avoid collisions).

4. **Cache (Redis/Memcached):**
   - Stores frequently accessed short-to-long URL mappings.
   - Reduces load on the database and improves redirect latency.
   - Eviction Policy: LRU (Least Recently Used) is appropriate as recent links are more likely to be clicked.

## API Design

### Create Short URL
```
POST /api/v1/shorten
Request Body:
{
  "long_url": "https://example.com/very/long/url",
  "custom_alias": "mylink" (optional),
  "expiration": "2024-12-31" (optional)
}

Response:
{
  "short_url": "https://short.ly/abc123",
  "long_url": "https://example.com/very/long/url",
  "created_at": "2024-01-01T00:00:00Z"
}
```

### Redirect Short URL
```
GET /{shortCode}
Response: 301/302 Redirect to long_url
```

## Database Schema

### URLs Table
```sql
CREATE TABLE urls (
  id BIGINT PRIMARY KEY,
  short_code VARCHAR(10) UNIQUE,
  long_url TEXT NOT NULL,
  user_id BIGINT,
  created_at TIMESTAMP,
  expires_at TIMESTAMP,
  clicks INT DEFAULT 0
);
```

## Next Steps
Continue to [deep-dive.md](deep-dive.md)
