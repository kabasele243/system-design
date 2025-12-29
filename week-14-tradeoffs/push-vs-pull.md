# Push vs Pull (Fan-out Models)

## Overview
How data moves from producer to consumer significantly affects system latency and complexity, especially in Social Feeds, Notifications, and Chat apps.

## 1. Pull Model (Fan-out on Read)
The consumer proactively asks for data.
*   **Workflow**:
    1.  User posts message -> Stored in DB (Write is Cheap).
    2.  Follower views feed -> System queries DB for all recent posts from all followees (Read is Expensive).
*   **Pros**:
    *   **Simple Writes**: No complex fan-out logic during posting.
    *   **Real-time** (mostly): Data is available as soon as it's written.
    *   **Efficient for Inactive Users**: No resources wasted computing feeds for users who don't login.
*   **Cons**:
    *   **Slow Reads (N+1 Problem)**: Aggregating feeds from 1000 followees at runtime is slow.
    *   **Resouce Intensive**: Heavy load on DB during read time.

## 2. Push Model (Fan-out on Write)
The producer proactively sends data to the consumer.
*   **Workflow**:
    1.  User posts message -> System finds all followers -> Writes post to each follower's "Pre-computed Feed" (Write is Expensive).
    2.  Follower views feed -> System reads single key from Cache (Read is Cheap).
*   **Pros**:
    *   **Fast Reads**: O(1) complexity. Feed is pre-baked.
    *   **Decoupled**: Readers don't hit the main DB.
*   **Cons**:
    *   **Write Amplification**: 1 post -> 1M writes (if 1M followers).
    *   **Lag**: "Thundering Herd" can cause delay in feed updates.
    *   **Wasted Storage**: Storing feeds for users who never login.

## Hybrid Approach
*   Use **Push** for normal users (fast reads).
*   Use **Pull** for celebrities/hot-topics (avoid write amplification).
