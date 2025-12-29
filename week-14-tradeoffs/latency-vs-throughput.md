# Latency vs Throughput

## Definitions

### Latency
**Time to complete a single task.**
*   "How fast can I get a response?"
*   Measured in milliseconds (ms).
*   Crucial for: API responses, Gaming, Trading.

### Throughput
**Number of tasks completed per unit of time.**
*   "How much work can the system handle?"
*   Measured in Requests Per Second (RPS), Transactions Per Second (TPS), or GB/s.
*   Crucial for: Data ingestion, Batch processing, Logging.

## The Trade-off
Often, improving one degrades the other.

*   **Optimizing for Throughput (Batching)**:
    *   Grouping 100 DB writes into 1 transaction improves Throughput.
    *   *But*, the first request has to wait for the 100th request to arrive -> Worst Latency.

*   **Optimizing for Latency (Real-time)**:
    *   Processing every request immediately improves Latency.
    *   *But*, the overhead of context switching/IO for each request lowers overall Throughput.

## Tuning Strategy

### For Latency:
1.  **Concurrency**: Do things in parallel.
2.  **Caching**: Avoid computing/fetching.
3.  **CDN**: Move data closer to user.
4.  **No-Batching**: Process immediately.

### For Throughput:
1.  **Batching**: Group operations (IO is expensive).
2.  **Asynchronous processing**: Don't block threads.
3.  **Compression**: Trade CPU for Bandwidth.
4.  **Buffer**: Use queues (Kafka) to smooth out spikes.

## Little's Law
`Concurrency = Throughput × Latency`
*   If your system takes 100ms (0.1s) to respond, and you want to handle 1000 RPS.
*   You need `1000 * 0.1 = 100` concurrent worker threads/connections.
