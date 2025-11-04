# Back-of-Envelope Calculations

## Learning Objectives
- Learn to estimate QPS (Queries Per Second)
- Calculate storage needs
- Estimate bandwidth requirements

## Common Numbers to Memorize (CRITICAL!)

### Powers of 10
| Name | Number | Scientific | Usage |
|------|--------|------------|-------|
| Thousand | 1,000 | 10³ | 1K users |
| Million | 1,000,000 | 10⁶ | 1M users |
| Billion | 1,000,000,000 | 10⁹ | 1B users (Instagram scale) |
| Trillion | 1,000,000,000,000 | 10¹² | 1T requests |

### Time Conversions
| Time Period | Exact Seconds | **Use in Interviews** | Why |
|-------------|---------------|----------------------|-----|
| 1 day | 86,400 | **100,000** | Easier mental math! |
| 1 month | ~2.5 million | 2.5 × 10⁶ | 30 × 86,400 |
| 1 year | ~31.5 million | 30 × 10⁶ | 365 × 86,400 |

**Key Insight:** Use **100,000 seconds/day** in interviews - interviewers expect rounding!

### Data Sizes
| Unit | Size | Scientific | Example |
|------|------|------------|---------|
| 1 KB | ~1,000 bytes | 10³ | Text message (100 bytes) |
| 1 MB | ~1,000 KB | 10⁶ | Small photo (500 KB - 2 MB) |
| 1 GB | ~1,000 MB | 10⁹ | HD movie clip |
| 1 TB | ~1,000 GB | 10¹² | 1,000 HD movies |
| 1 PB | ~1,000 TB | 10¹⁵ | Instagram yearly storage |

### Common Object Sizes (When Not Given)
- **Text message:** 100 bytes
- **Tweet:** 300 bytes (280 chars)
- **Photo:** 500 KB - 2 MB (use 500 KB as default)
- **Video (1 min):** 10-50 MB
- **User profile:** 1 KB

---

## The Three Essential Calculations

### 1. QPS (Queries Per Second)

**Formula:**
```
QPS = (Daily Active Users × Actions per day) ÷ 100,000 seconds
Peak QPS = Average QPS × 2 (or × 3 for safety)
```

**Why it matters:** Tells you how many servers you need!

**Example: Instagram Feed Reads**
```
500 million DAU × 10 reads/day ÷ 100,000
= 5,000 million ÷ 100,000
= 50,000 QPS (average)
Peak = 50,000 × 2 = 100,000 QPS
```

### 2. Storage Calculation

**Formula:**
```
Total Storage = Number of items × Size per item × Time period
```

**Example: Instagram Photos**
```
1 billion users × 2 photos/week × 52 weeks × 500 KB
= 104 billion photos/year × 500 KB
≈ 100 billion × 500 KB (round for simplicity)
= 50,000 TB = 50 PB per year
```

### 3. Bandwidth Calculation

**Formula:**
```
Bandwidth = Total data transferred ÷ Time (per second)
```

**Example: YouTube Daily**
```
2.5 billion videos/day × 50 MB/video = 125 PB/day
125 PB ÷ 100,000 seconds = 1.25 TB/second
```

**Storage vs Bandwidth:**
- **Storage:** Data stored on disk (cumulative)
- **Bandwidth:** Data moving through network per second (continuous)

## Practice Problems (Solved)

### Problem 1: Instagram Photo Storage
**Question:** Calculate storage for 1 billion users posting 2 photos per week for a year (500 KB per photo).

**Solution:**
```
Step 1: Photos per year
1B users × 2 photos/week × 52 weeks = 104B photos/year
Round to: 100B photos

Step 2: Total storage
100B photos × 500 KB = 50,000 TB = 50 PB per year
```
**Answer: ~50 PB/year**

---

### Problem 2: WhatsApp Messages
**Question:** 2 billion users send 50 messages/day at 100 bytes each. Storage per day?

**Solution:**
```
Step 1: Messages per day
2B users × 50 messages = 100B messages/day

Step 2: Storage
100B × 100 bytes = 10 trillion bytes = 10 TB/day
```
**Answer: 10 TB/day**

---

### Problem 3: YouTube Bandwidth
**Question:** 500M DAU watch 5 videos/day at 50 MB each. Bandwidth needed?

**Solution:**
```
Step 1: Videos per day
500M × 5 = 2.5B videos/day

Step 2: Total data
2.5B × 50 MB = 125 PB/day

Step 3: Bandwidth (per second)
125 PB ÷ 100,000 seconds = 1.25 TB/second
```
**Answer: 1.25 TB/second**

---

### Problem 4: Twitter Write QPS
**Question:** 200M DAU tweet 2 times/day. Write QPS?

**Solution:**
```
200M × 2 tweets ÷ 100,000 seconds = 4,000 QPS
Peak QPS = 4,000 × 2 = 8,000 QPS
```
**Answer: 4,000 QPS (average), 8,000 QPS (peak)**

---

### Problem 5: Facebook Feed Read QPS
**Question:** 2B DAU load feed 20 times/day. Read QPS?

**Solution:**
```
2B × 20 reads ÷ 100,000 = 400,000 QPS
Peak QPS = 400,000 × 2 = 800,000 QPS
```
**Answer: 400,000 QPS (average), 800,000 QPS (peak)**

---

## Interview Cheat Sheet (Print & Memorize!)

### The 3 Formulas
1. **QPS:** `(Users × Actions) ÷ 100,000`
2. **Storage:** `Items × Size × Time`
3. **Bandwidth:** `Data ÷ Time (per second)`

### Quick Numbers
- 1 day = **100,000 seconds** (use this!)
- Peak = Average × **2**
- Photo = **500 KB**
- Video (1 min) = **50 MB**
- Text message = **100 bytes**

### Process in Interviews
1. **State assumptions** ("Assuming 500 KB per photo...")
2. **Show your work** (don't just give answer)
3. **Round for simplicity** (104B → 100B)
4. **Get to ballpark** (exact number doesn't matter)
5. **Convert to meaningful units** (50,000 TB → 50 PB)

## Resources & References
- **Key insight:** Use 100,000 seconds/day for mental math
- **Always calculate:** QPS, Storage, Bandwidth for every system design
- **Remember:** Peak traffic = 2-3x average
- **Pro tip:** Interviewer cares about process, not exact numbers
