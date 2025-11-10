# Back-of-Envelope Numbers Cheat Sheet

## Powers of 2 (Data Sizes)

| Power | Exact Value | Approximate | Name |
|-------|-------------|-------------|------|
| 10 | 1,024 | ~1 thousand | 1 KB |
| 20 | 1,048,576 | ~1 million | 1 MB |
| 30 | 1,073,741,824 | ~1 billion | 1 GB |
| 40 | 1,099,511,627,776 | ~1 trillion | 1 TB |

## Latency Numbers Every Programmer Should Know

| Operation | Time |
|-----------|------|
| L1 cache reference | 0.5 ns |
| Branch mispredict | 5 ns |
| L2 cache reference | 7 ns |
| Mutex lock/unlock | 100 ns |
| Main memory reference | 100 ns |
| Compress 1KB with Snappy | 10 μs |
| Send 2KB over 1 Gbps network | 20 μs |
| Read 1 MB sequentially from memory | 250 μs |
| Round trip within same datacenter | 500 μs |
| Disk seek | 10 ms |
| Read 1 MB sequentially from network | 10 ms |
| Read 1 MB sequentially from disk | 30 ms |
| Send packet CA → Netherlands → CA | 150 ms |

## Time Conversions

| Unit | Value |
|------|-------|
| 1 million seconds | ≈ 11.5 days |
| 1 billion seconds | ≈ 31.5 years |
| 1 second | 1,000 ms |
| 1 ms | 1,000 μs |
| 1 μs | 1,000 ns |

## Availability Numbers (Nines)

| Availability | Downtime per Year | Downtime per Month | Downtime per Week |
|--------------|-------------------|-------------------|-------------------|
| 99% (Two nines) | 3.65 days | 7.31 hours | 1.68 hours |
| 99.9% (Three nines) | 8.77 hours | 43.83 minutes | 10.08 minutes |
| 99.99% (Four nines) | 52.6 minutes | 4.38 minutes | 1.01 minutes |
| 99.999% (Five nines) | 5.26 minutes | 26.30 seconds | 6.05 seconds |

## QPS Calculation Template

```
Daily Active Users (DAU) × Actions per user per day
÷ 86,400 (seconds in a day)
= Average QPS

Peak QPS = Average QPS × 2 (or 3)
```

### Example:
```
500M DAU × 10 feed reads/day ÷ 86,400
= 57,870 QPS (average)

Peak QPS = 57,870 × 2 = 115,740 QPS
```

## Storage Calculation Template

```
Total Items × Size per Item × Time Period
```

### Example 1: Photo Storage
```
1B users × 2 photos/week × 52 weeks/year × 500 KB/photo
= 1B × 104 photos/year × 500 KB
= 52 TB/year
```

### Example 2: Tweet Storage
```
500M DAU × 2 tweets/day × 365 days/year × 300 bytes/tweet
= 109.5 GB/year (just text)
```

## Bandwidth Calculation

```
QPS × Average Object Size
```

### Example:
```
100,000 QPS × 100 KB = 10 GB/s = 80 Gbps
```

## Common Ratios to Remember

- **Read:Write Ratio:** Usually 100:1 or 10:1 (most systems are read-heavy)
- **Cache Hit Rate:** Target 80%+
- **Peak Traffic:** 2-3x average
- **Database Connections:** ~100-200 per server
- **Memory:** 8-64 GB per server (typical)

## Quick Estimation Rules

1. **Twitter-scale:** 500M DAU, 300M tweets/day
2. **Instagram-scale:** 1B users, 100M photos/day
3. **YouTube-scale:** 2B users, 500 hours video uploaded/minute
4. **Typical web request:** 100 KB
5. **Typical image:** 500 KB - 2 MB
6. **Typical video (1 min):** 10-50 MB

## Practice Problem

**Design a photo-sharing app:**
- 1 billion users
- 10% DAU (100M daily active)
- Each user uploads 2 photos/day
- Each photo is 1 MB

**Calculate:**
1. Write QPS
2. Read QPS (assume 100:1 read:write)
3. Daily storage
4. Storage for 5 years
5. Bandwidth needed

**Your answers:**

