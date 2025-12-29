# Normalization vs Denormalization

## Overview
How you structure your database tables impacts write speed, storage size, and read performance.

## 1. Normalization (Optimized for Writes)
Organizing data to minimize redundancy and dependency (3NF).
*   **Example**: Storing `user_id` in the `orders` table, and looking up the name in the `users` table via JOIN.
*   **Pros**:
    *   **Data Integrity**: Update a user's name in one place, it reflects everywhere.
    *   **Write Speed**: Smaller payloads, less data to write.
    *   **Storage**: Efficient (No duplicate strings).
*   **Cons**:
    *   **Read Speed**: Requires complex JOINs (CPU heavy).

## 2. Denormalization (Optimized for Reads)
Adding redundant data to speed up reads.
*   **Example**: Storing `user_name` directly in the `orders` table.
*   **Pros**:
    *   **Read Speed**: No JOINs required. `SELECT * FROM orders` gives you everything.
    *   **Scalability**: Easier to shard (data is self-contained).
*   **Cons**:
    *   **Data Integrity**: If user changes name, you must update 1000s of order rows (Write Amplification).
    *   **Storage**: Higher disk usage.

## Comparison Table

| Feature | Normalized | Denormalized |
| :--- | :--- | :--- |
| **Integrity** | High (Single source of truth) | Low (Data drift possible) |
| **Read Speed** | Slow (Joins) | Fast (Single lookup) |
| **Write Speed** | Fast | Slow (Update multiple places) |
| **Best For** | SQL / Transactions | NoSQL / Analytics / High Read QPS |
