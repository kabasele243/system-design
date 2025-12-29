# Consistency Models

## Overview
How synchronized is your data across distributed nodes?

## 1. Strong Consistency
After a write completes, **all** subsequent reads will return that value.
*   **Technologies**: SQL (ACID), Redis (Single Node), HBase, CP systems.
*   **Trade-off**: Higher Latency (must wait for replication) & Lower Availability (CAP theorem).
*   **Use Case**: Financial ledgers, Inventory counters, Password changes.

## 2. Eventual Consistency
After a write completes, reads **may** return stale data for a short time. Eventually, all nodes will converge.
*   **Technologies**: Cassandra, DynamoDB, DNS, CDN.
*   **Trade-off**: Lower Latency (write returns immediately) & High Availability.
*   **Use Case**: Social feeds, Comments, View counts.
    *   *If I like a post, it's okay if my friend doesn't see it for 2 seconds.*

## 3. Causal Consistency
A middle ground. Writes that are potentially related (causal) must be seen in order.
*   **Example**: If I reply to a comment, my reply must appear *after* the comment. Unrelated comments can appear in any order.

## 4. Weak Consistency
No guarantee when data will be seen.
*   **Use Case**: VoIP, Sensor data (if you miss a packet, it doesn't matter).

## Summary
*   **Money** = Strong Consistency.
*   **Social/Content** = Eventual Consistency.
