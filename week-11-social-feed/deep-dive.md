# Social Media Feed - Deep Dive

## Step 4: Deep Dive - The Fan-out Problem

### The Core Challenge
When a user posts, how do you deliver that post to all their followers?

## Approach 1: Fan-out on Write (Push Model)

### How it works:
1. User creates a post
2. System immediately writes post to feed cache of ALL followers
3. When user requests feed, read directly from their pre-generated cache

### Diagram:
```
User posts → Post Service → Message Queue → Feed Workers
                                              ↓
                                         Update all
                                    followers' feed caches
```

### Pros:
-
-

### Cons:
-
-

### Best for:


## Approach 2: Fan-out on Read (Pull Model)

### How it works:
1. User creates a post
2. Post is stored in database
3. When user requests feed, query database for posts from people they follow
4. Aggregate and sort posts in real-time

### Diagram:
```
User requests feed → Feed Service → Query posts from followees
                                   → Sort by timestamp
                                   → Return feed
```

### Pros:
-
-

### Cons:
-
-

### Best for:


## Approach 3: Hybrid Model (RECOMMENDED)

### How it works:
- **For normal users:** Use fan-out on write
- **For celebrities (>1M followers):** Use fan-out on read
- Detect celebrity status based on follower count

### Why hybrid?
-
-

### The Celebrity Problem:
If a celebrity with 50M followers posts, fan-out on write would:
- Write to 50M feed caches
- Take a long time
- Waste resources (many inactive users)

Solution:
- Don't pre-generate feeds for celebrity posts
- Fetch celebrity posts on-demand when user requests feed
- Mix pre-generated feed + celebrity posts

## Feed Ranking & Filtering

### Chronological vs Algorithmic Feed

**Chronological (Twitter style):**
-

**Algorithmic (Instagram/Facebook style):**
-

## Caching Strategy

### What to cache:
1. User's feed (last 1000 posts)
2. Social graph (followers/following)
3. Post content and metadata

### Cache Structure (Redis):
```
Key: feed:user:{user_id}
Value: List of post_ids sorted by timestamp
[post_id_1, post_id_2, post_id_3, ...]
```

### Cache size calculation:


## Time-series Data Structure

### Storing feeds efficiently:


## Trade-offs Discussion

| Aspect | Fan-out on Write | Fan-out on Read | Hybrid |
|--------|-----------------|-----------------|--------|
| Read latency | | | |
| Write latency | | | |
| Storage cost | | | |
| Handles celebrities | | | |
| Your choice | | | ✓ |

## Next Steps
Continue to [failure-scenarios.md](failure-scenarios.md)
