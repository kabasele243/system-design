# Back-of-Envelope Calculations

## Learning Objectives
- Learn to estimate QPS (Queries Per Second)
- Calculate storage needs
- Estimate bandwidth requirements

## Common Numbers to Memorize
- 1 million seconds ≈ 11.5 days
- 1 billion seconds ≈ 31.5 years
- 1 MB = 10^6 bytes
- 1 GB = 10^9 bytes
- 1 TB = 10^12 bytes

## Calculation Templates

### QPS Calculation
```
Daily Active Users (DAU) × Actions per user per day / 86400 seconds
Peak QPS = Average QPS × 2 (or 3 for safety)
```

### Storage Calculation
```
Total Users × Data per User × Time Period
```

### Bandwidth Calculation
```
Average object size × QPS
```

## Practice Problems

### Problem 1: Photo Storage
Calculate how much storage is needed for 1 billion users posting 2 photos per week for a year.

**Assumptions:**
- 1 billion users
- 2 photos per week
- Average photo size:
- Retention period: 1 year

**Solution:**


### Problem 2: Video Streaming
Calculate bandwidth needed for a video streaming service.

**Your Solution:**


## Resources & References
