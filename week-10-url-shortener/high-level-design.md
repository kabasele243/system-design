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

2. **Web/App Servers:**

3. **Database:**

4. **Cache:**

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
