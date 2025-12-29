# Sharding Strategies

## Overview
When a database grows too large for a single server, you must split (shard) it across multiple servers.

## 1. Range Based Sharding
Partition data based on key ranges.
*   **Example**: `User IDs 1-1000` -> Server A, `User IDs 1001-2000` -> Server B.
*   **Pros**:
    *   **Simple**: Easy to implement.
    *   **Efficient Queries**: Range queries (`limit 10 offset 100`) are fast if they fall in the same shard.
*   **Cons**:
    *   **Uneven Data**: "User 1" might store 1TB, "User 1001" might store 1KB.
    *   **Hotspots**: New users usually have incremental IDs. All writes hit the last shard. Requires constant splitting.

## 2. Directory Based Sharding
A lookup table knows where each key lives.
*   **Example**: `Lookup Table`: { "UserA": "Shard1", "UserB": "Shard3" }.
*   **Pros**: Highly flexible.
*   **Cons**: Single point of failure (Lookup DB), Query overhead.

## 3. Consistent Hashing (Hash Based)
Partition based on the Hash of the key.
*   **Mechanism**: `hash(key) % total_shards` -> determines server.
*   **Consistent Hashing**: A ring topology where adding/removing a node only affects `K/N` keys (neighbors), not all keys.
*   **Pros**:
    *   **Uniform Distribution**: Data is spread evenly.
    *   **Balanced Load**: No write hotspots.
*   **Cons**:
    *   **Resharding**: Adding nodes causes data movement (rebalancing).
    *   **Bad for Range Queries**: Data is scattered everywhere. You must query ALL shards to find "Users created yesterday".

## Summary
*   **Most Common**: Consistent Hashing (Cassandra, DynamoDB).
*   **For Analytics/Time-Series**: Range Based (Partition by Day).
