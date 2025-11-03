# Reliability & Availability

## Learning Objectives
- Understand the difference between reliability and availability
- Learn about SLA levels (99.9% vs 99.999%)
- Calculate downtime for different availability levels

## Notes

### Reliability (No Data Corruption)

**Analogy:** "Reliable Rita" - The ATM that works perfectly when it's on, never gives wrong amounts, but goes down for maintenance 1 day/month.

**Definition:** The system does what it's supposed to do correctly.
- No data corruption
- No wrong results
- When it works, it works RIGHT

**Key Principle:** Better to be down than to give wrong data.

**Examples Where Reliability is Critical:**
- **Banking systems:** Wrong amounts could bankrupt customers
- **Medical records:** Wrong diagnosis data could kill someone
- **Airplane systems:** Wrong altitude data = crash
- **Uber payments:** Can't double-charge or lose driver earnings

**Trade-off:** May need to go offline to maintain correctness.

### Availability (System is Responsive)

**Analogy:** "Available Alex" - The ATM that's always running 24/7, but sometimes gives wrong amounts.

**Definition:** The system is always accessible and responsive.
- Always up and running
- Gives you *a* response (might be stale/incorrect)
- Users can always access it

**Key Principle:** Better to show something than nothing.

**Examples Where Availability is Critical:**
- **Social media (Instagram, Twitter):** Stale like counts are fine
- **Netflix recommendations:** Old suggestions better than none
- **Uber driver location:** 30-second stale location is acceptable
- **Online gaming:** Must be responsive (but also needs reliability!)

**Trade-off:** Might show stale or eventually consistent data.

### The Balance: Systems Need BOTH

Most real systems need BOTH, but prioritize based on the feature:

**Uber Example:**
```
Driver Location → HIGH AVAILABILITY (fast updates, stale okay)
Payment Processing → HIGH RELIABILITY (must be exact, can be slower)
```

**E-commerce Example:**
```
Product Browsing → HIGH AVAILABILITY (show products fast)
Checkout/Payment → HIGH RELIABILITY (correct charges)
```

### Availability Levels (The "Nines")

| Availability | Name | Downtime/Year | Downtime/Month | Downtime/Day | Use Case |
|--------------|------|---------------|----------------|--------------|----------|
| 90% | One nine | 36.5 days | 3 days | 2.4 hours | Personal projects |
| 99% | Two nines | 3.65 days | 7.3 hours | 14.4 minutes | Small businesses |
| 99.9% | Three nines | **8.77 hours** | 43.8 minutes | 1.44 minutes | **Most web apps** (Instagram, Twitter) |
| 99.99% | Four nines | **52.6 minutes** | 4.38 minutes | 8.6 seconds | **E-commerce, Gaming** (Uber, LoL) |
| 99.999% | Five nines | **5.26 minutes** | 26 seconds | 0.86 seconds | **Mission critical** (Banking, 911) |

**Key Insight:** Each additional "nine" increases cost and complexity exponentially.

**99.9% → 99.99%:**
- Downtime: 8.77 hours → 52 minutes (Much better!)
- Cost: 2-3x more expensive
- Complexity: Need redundancy, monitoring, faster response teams

**Why Companies Care:**
- 8.77 hours of Instagram downtime = Millions lost in ad revenue
- 52 minutes of bank downtime = Acceptable for scheduled maintenance
- 5.26 minutes for 911 = Lives saved

## Practice Questions

1. **What does 99.9% availability mean in real time?**
   - 8.77 hours of downtime per year
   - 43.8 minutes per month
   - Good enough for most web apps (Instagram, Twitter)

2. **How much downtime per year is acceptable for 99.99% availability?**
   - 52.6 minutes per year
   - 4.38 minutes per month
   - Required for e-commerce, gaming, critical services

3. **What's the difference between a highly reliable vs highly available system?**
   - **Reliable:** Gives correct results, may go offline to stay correct (banking)
   - **Available:** Always responsive, may show stale data (social media)
   - **Best systems:** Balance both based on feature requirements

4. **For Uber driver location tracking, would you choose reliability or availability? Why?**
   - **Availability** - Fast updates with 30-second stale data is acceptable
   - Users care more about responsiveness than perfect accuracy
   - Cars don't teleport, so stale location is "good enough"

5. **For Uber payment processing, would you choose reliability or availability? Why?**
   - **Reliability** - Must charge correct amounts
   - Better to fail transaction than double-charge
   - Can be slower (few seconds is fine)

## Real-World Examples

### High Reliability Systems (Correctness > Uptime)
- **Banking ATMs:** Never give wrong amounts, scheduled downtime acceptable
- **Medical devices:** Wrong data = patient death
- **Air traffic control:** Wrong altitude = crash
- **Stock trading settlement:** Must be exact

### High Availability Systems (Uptime > Perfect Accuracy)
- **Social media feeds:** Stale like counts are fine
- **Netflix:** Stale recommendations better than none
- **Search engines:** Slightly outdated results acceptable
- **Weather apps:** 5-minute-old data is good enough

### Systems Requiring BOTH (Choose per Feature)
- **Uber:** Location (availability), Payments (reliability)
- **E-commerce:** Browsing (availability), Checkout (reliability)
- **Online gaming:** Matchmaking (availability), Game state (both!)
- **Banking apps:** Account balance (reliability), Branch locator (availability)

## Resources & References
- "Reliable Rita" vs "Available Alex" - the ATM analogy
- The "Nines" table - memorize 99.9% = 8.77 hours, 99.99% = 52 minutes
- Key principle: Different parts of a system need different guarantees
