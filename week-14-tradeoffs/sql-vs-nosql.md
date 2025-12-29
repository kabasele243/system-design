# SQL vs NoSQL

## Overview
Choosing the right database is one of the most critical decisions in system design.

## Comparison

| Feature | SQL (Relational) | NoSQL (Non-Relational) |
| :--- | :--- | :--- |
| **Structure** | Structured tables with fixed schema. | Flexible schema (Document, Key-Value, Graph, Wide-Column). |
| **Transactions** | ACID (Atomicity, Consistency, Isolation, Durability). | Mostly BASE (Basically Available, Soft state, Eventual consistency). Some offer ACID. |
| **Scaling** | Vertical (Scale Up) - increasing CPU/RAM. Horizontal possible but complex (Sharding). | Horizontal (Scale Out) - adding more servers. Built-in sharding. |
| **Querying** | SQL (Standardized, powerful JOINs). | Proprietary APIs, Limited JOIN support. |

## Approaches

### 1. SQL (RDBMS)
*   **Examples**: PostgreSQL, MySQL, Oracle.
*   **Best for**:
    *   Financial systems (strict ACID).
    *   Complex relationships (many-to-many).
    *   Structured data that doesn't change format often.
*   **Trade-off**: Harder to scale horizontally; Schema changes can be painful (downtime/lock).

### 2. NoSQL
*   **Types**:
    *   **Document**: MongoDB, Couchbase (JSON-like, flexible).
    *   **Key-Value**: Redis, DynamoDB (Fast, simple lookups).
    *   **Wide-Column**: Cassandra, HBase (Write-heavy, time-series, huge scale).
    *   **Graph**: Neo4j (Complex relationships/traverse).
*   **Best for**:
    *   Rapid prototyping (Flexible schema).
    *   Massive scale (TB/PB of data).
    *   High write throughput.
    *   Unstructured data (Product catalogs, User profiles).
*   **Trade-off**: Weaker consistency (usually), Limited query capability (No complex JOINS).

## Decision Logic
*   Need ACID/Transactions? -> **SQL**
*   Need Infinite Scale/High Throughput? -> **NoSQL (Cassandra/Dynamo)**
*   Need Flexible Schema? -> **NoSQL (Mongo)**
*   Need Complex Relationships? -> **Graph or SQL**
