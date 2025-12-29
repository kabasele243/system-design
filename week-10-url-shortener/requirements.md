# URL Shortener Design - Requirements

## Problem Statement
Design a URL shortener service like TinyURL or bit.ly

## Step 1: Clarify Requirements

### Functional Requirements
- [ ] Create short URL from long URL
- [ ] Redirect short URL to original long URL
- [ ] Custom short URLs (optional)
- [ ] URL expiration
- [ ] Analytics (click tracking)

### Non-Functional Requirements
- [ ] High availability
- [ ] Low latency for redirects
- [ ] Scalable to handle millions of URLs
- [ ] Prevent collisions

## Step 2: Estimate Scale

### Given:
- 100M URLs created per month
- Read/Write ratio: 10:1

### Calculations:

**Write QPS:**
```
100M URLs/month ÷ (30 days × 24 hours × 3600 seconds) = ~40 write requests/second
Peak write QPS (2x) = ~80 write requests/second
```

**Read QPS:**
```
Read ratio is 10:1, so:
Average read QPS = ~400 read requests/second
Peak read QPS = ~800 read requests/second
```

**Storage:**
```
Assumptions:
- Original URL length: 500 bytes
- Short URL: 7 characters = 7 bytes
- Metadata: 100 bytes
- Total per URL: 607 bytes ≈ 1KB

For 100M URLs/month over 5 years:
100M × 12 months × 5 years = 6 billion URLs
6B × 1KB = 6TB
```

**Bandwidth:**
```
Write: 40 QPS × 1KB = 40 KB/s
Read: 400 QPS × 1KB = 400 KB/s
```

## Next Steps
Continue to [high-level-design.md](high-level-design.md)
