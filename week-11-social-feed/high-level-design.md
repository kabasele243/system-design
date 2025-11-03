# Social Media Feed - High-Level Design

## Step 3: High-Level Architecture

### Components

```
[Clients] → [Load Balancer] → [API Gateway]
                                    ↓
                    [Feed Service] ← [Cache (Redis)]
                          ↓
            [Post Service] → [Message Queue]
                   ↓              ↓
            [Database]     [Feed Worker]
                                 ↓
                          [Feed Cache]
```

### Core Components:

1. **Post Service:**
   - Handles creating, updating, deleting posts
   - Publishes post events to message queue

2. **Feed Service:**
   - Generates user's feed
   - Reads from feed cache

3. **Feed Worker:**
   - Consumes post events from queue
   - Updates followers' feed caches

4. **Cache:**
   - Stores pre-generated feeds
   - Stores user social graph (followers/following)

## API Design

### Create Post
```
POST /api/v1/posts
Request:
{
  "user_id": 123,
  "content": "Hello world!",
  "media_urls": ["url1", "url2"],
  "type": "text|image|video"
}

Response:
{
  "post_id": 456,
  "created_at": "2024-01-01T00:00:00Z"
}
```

### Get Feed
```
GET /api/v1/feed?user_id=123&page=1&size=20
Response:
{
  "posts": [
    {
      "post_id": 789,
      "author": {...},
      "content": "...",
      "created_at": "...",
      "likes_count": 100,
      "comments_count": 50
    }
  ],
  "next_page": 2
}
```

## Database Schema

### Users Table
```sql
CREATE TABLE users (
  user_id BIGINT PRIMARY KEY,
  username VARCHAR(50),
  created_at TIMESTAMP
);
```

### Posts Table
```sql
CREATE TABLE posts (
  post_id BIGINT PRIMARY KEY,
  user_id BIGINT,
  content TEXT,
  media_urls JSON,
  created_at TIMESTAMP,
  likes_count INT DEFAULT 0,
  INDEX idx_user_created (user_id, created_at)
);
```

### Follow Graph
```sql
CREATE TABLE follows (
  follower_id BIGINT,
  followee_id BIGINT,
  created_at TIMESTAMP,
  PRIMARY KEY (follower_id, followee_id)
);
```

## Next Steps
Continue to [deep-dive.md](deep-dive.md)
