# Social Media Feed Design - Requirements

## Problem Statement
Design a social media news feed like Twitter, Instagram, or Facebook

## Step 1: Clarify Requirements

### Functional Requirements
- [ ] User can post content (text, images, videos)
- [ ] User can follow other users
- [ ] User sees feed of posts from people they follow
- [ ] Posts are ordered by time (most recent first)
- [ ] User can like/comment on posts

### Non-Functional Requirements
- [ ] Highly available
- [ ] Low latency for feed generation (< 200ms)
- [ ] Scalable to billions of users
- [ ] Handle celebrities with millions of followers

## Step 2: Estimate Scale

### Given:
- 500M daily active users
- Average 200 followers per user
- Average 2 posts per user per day

### Calculations:

**Write QPS (Posting):**
```
500M users × 2 posts/day ÷ 86400 seconds =
Peak write QPS (2x) =
```

**Read QPS (Feed Generation):**
```
Assume each user checks feed 10 times/day
500M users × 10 reads/day ÷ 86400 seconds =
Peak read QPS =
```

**Storage:**
```
Post size:
- Text: 1KB
- Images: 500KB average
- Metadata: 1KB
Average post: 100KB

Daily posts: 500M users × 2 posts = 1B posts/day
Daily storage: 1B × 100KB = 100TB/day
```

**Fan-out Calculation:**
```
1B posts/day × 200 followers = 200B feed updates/day
```

## Next Steps
Continue to [high-level-design.md](high-level-design.md)
