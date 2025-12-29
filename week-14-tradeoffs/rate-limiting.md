# Rate Limiting

## Overview
Preventing abuse and protecting services from overload.

## Algorithms

### 1. Token Bucket
*   **Concept**: You have a "bucket" of tokens. Tokens refill at rate `r`. Each request consumes a token. If bucket empty, drop request.
*   **Pros**: **Allows Bursts**. If bucket is full, you can handle a sudden spike.
*   **Cons**: Complex to implement distributedly.

### 2. Leaky Bucket
*   **Concept**: Requests enter a queue. Queue drains at a constant rate. Check for overflow.
*   **Pros**: **Smooths traffic**. Output rate is constant.
*   **Cons**: Bursts are lost/delayed.

### 3. Fixed Window Counter
*   **Concept**: "100 requests per minute". Reset counter at `:00`, `:01`, etc.
*   **Pros**: Simple memory footprint (Atomic Int).
*   **Cons**: **Edge Case Spike**. You can send 100 reqs at `00:59` and 100 reqs at `01:01`. Total 200 reqs in 2 seconds allowed.

### 4. Sliding Window Log
*   **Concept**: Store timestamp of every request. Count how many in the last minute.
*   **Pros**: Perfectly accurate. No edge spikes.
*   **Cons**: **Memory Heavy**. Storing timestamps for millions of users is expensive.

## Scope
*   **Global**: Limit across whole system (Requires central Redis).
*   **Local**: Limit per server instance (In-memory, faster, but less accurate).
