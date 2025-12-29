# Data Processing: Batch vs Stream

## Overview
How and when we process data determines the freshness of insights and system architecture.

## 1. Batch Processing
Processing a large volume of data all at once, typically on a scheduled basis (e.g., Daily, Hourly).
*   **Bounded Data**: Finite set of data (e.g., "Yesterday's logs").
*   **Technologies**: Hadoop MapReduce, Spark (Batch mode), Hive, Snowflake, BigQuery.
*   **Pros**:
    *   High Throughput (optimized for large volumes).
    *   Accurate (Can re-process if failed).
    *   Simpler (stateless logic per item, mostly).
*   **Cons**:
    *   High Latency (Insights are hours/days old).
    *   "Spiky" resource usage.

## 2. Stream Processing
Processing data in real-time as it arrives.
*   **Unbounded Data**: Infinite flow of data (e.g., Clickstream, Sensor data).
*   **Technologies**: Apache Kafka, Apache Flink, Spark Streaming, Kinesis.
*   **Pros**:
    *   Low Latency (Insights in milliseconds/seconds).
    *   Smooth resource usage.
*   **Cons**:
    *   Complex (Handling late data, out-of-order data, windowing).
    *   Harder to re-process/backfill.

## Lambda Architecture
A hybrid approach to get the best of both.
*   **Speed Layer (Stream)**: Provides low-latency, approximate results.
*   **Batch Layer (Batch)**: Provides high-latency, accurate/comprehensive results.
*   *Note: Modern frameworks like Flink/Spark (Kappa Architecture) are deprecating this by unifying batch/stream.*

## Trade-off Summary
| Feature | Batch | Stream |
| :--- | :--- | :--- |
| **Latency** | Hours/Days | Milliseconds/Seconds |
| **Data Size** | Huge (PB) | Small per item, but infinite total |
| **Complexity** | Low | High |
| **Cost** | Cheaper (Transient clusters) | Expensive (Always-on clusters) |
