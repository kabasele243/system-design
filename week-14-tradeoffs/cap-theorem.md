# CAP Theorem

## Definition
The CAP Theorem (Brewer's Theorem) states that distributed systems can only deliver two of the following three guarantees:

1.  **Consistency (C)**: Every read receives the most recent write or an error. (All nodes see the same data at the same time).
2.  **Availability (A)**: Every request receives a (non-error) response, without the guarantee that it contains the most recent write.
3.  **Partition Tolerance (P)**: The system continues to operate despite an arbitrary number of messages being dropped/delayed by the network between nodes.

## The Reality
In a distributed system, **Partition Tolerance (P) is mandatory** because networks are unreliable. Therefore, you must choose between **Consistency (CP)** and **Availability (AP)** during a network partition.

## Trade-offs

| System Type | Choice | Trade-off Description | Examples |
| :--- | :--- | :--- | :--- |
| **CP (Consistency + Partition Tolerance)** | Waiting for consistency | If a partition occurs, the system will return an error or time out rather than return stale data. This is crucial for systems involving money or strict state. | Banking, MongoDB (default), HBase, Redis (Sentinel) |
| **AP (Availability + Partition Tolerance)** | Accepting staleness | If a partition occurs, the system will return the most recent version of the data it has, which might be stale. This provides a better user experience for things like social feeds. | Cassandra, DynamoDB (default), CouchDB, DNS |
| **CA (Consistency + Availability)** | No partitions allowed | Only possible in a non-distributed system (single node). Once you shard/replicate across a network, you must choose P. | Single Node SQL DB |

## Summary
- **Choose CP** when data correctness is more important than uptime (e.g., Bank balance).
- **Choose AP** when uptime/latency is more important than absolute correctness (e.g., Like count on a post).
