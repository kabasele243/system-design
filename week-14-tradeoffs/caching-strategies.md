# Caching Strategies

## Overview
How the cache interacts with the database determines data consistency and write performance.

## 1. Cache-Aside (Lazy Loading)
Application talks to Cache and DB separately.
1.  App checks Cache.
    *   Hit: Return data.
    *   Miss: App reads DB -> App writes to Cache -> Return data.
*   **Pros**: Resilient (If cache fails, system works), Data model flexible.
*   **Cons**: Initial request is slow (Miss), Stale data (DB update doesn't update cache automatically).
*   **Best For**: General purpose (Redis + SQL).

## 2. Write-Through
App writes to Cache, and Cache writes to DB synchronously.
*   **Pros**: Data consistency (Cache == DB), Read performance (Data always in cache).
*   **Cons**: Write latency (2 writes), Cache pollution (Storing data that might never be read).
*   **Best For**: Critical data where consistency is key.

## 3. Write-Back (Write-Behind)
App writes to Cache, and Cache writes to DB *asynchronously* (later).
*   **Pros**: Massive Write Throughput (DB isn't a bottleneck).
*   **Cons**: **Data Loss Risk** (If cache crashes before flushing to DB).
*   **Best For**: Analytics counters, Non-critical high-volume writes.

## 4. Write-Around
App writes directly to DB (skipping cache). Cache is only populated on Read Miss.
*   **Pros**: Reduces cache flood for "write-once-read-never" data.
*   **Best For**: Archival data, Large logs.

## Summary
*   **Most Common**: Cache-Aside + TTL (Time To Live).
*   **High Performance**: Write-Back (with risk).
*   **High Consistency**: Write-Through.
