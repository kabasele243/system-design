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
   - Handles creating, updating, deleting posts.
   - Persists post data to the `Posts` database.
   - Publishes "new_post" events to the Message Queue (Kafka) for async processing.

2. **Feed Service:**
   - Generates and retrieves user's news feed.
   - Reads from **Feed Cache** (Redis) for fast access (< 200ms).
   - If cache miss (rare), queries DB and rebuilds feed (fallback).

3. **Feed Worker:**
   - Consumes "new_post" events from the queue.
   - **Fan-out Service**: Fetches followers of the author.
   - **Privacy Service**: Checks if followers are allowed to see the post (ignored for this scale for simplicity).
   - Updates followers' feed caches (push model).

4. **Cache:**
   - **Global Cache**: Stores metadata and user profiles.
   - **Feed Cache (Redis)**: Stores the pre-computed feed for active users (e.g., list of Post IDs).
   - **Social Graph Cache**: Caches "Following" lists to speed up fan-out.

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
