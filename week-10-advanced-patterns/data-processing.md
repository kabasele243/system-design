# Data Processing Patterns: ETL, Batch vs Stream, Lambda & Kappa

## Overview

Modern systems process massive amounts of data in different ways depending on requirements. Understanding when to use batch processing vs stream processing, and how to architect hybrid systems, is essential for senior engineers.

## ETL vs ELT

### ETL (Extract, Transform, Load) - Traditional

```
Source DB → Extract → Transform (business logic) → Load → Data Warehouse
```

**Use case:** Clean data before storing (pre-2010s standard)

```python
# ETL Example
def etl_pipeline():
    # 1. Extract
    raw_data = extract_from_postgres()

    # 2. Transform (in application/ETL tool)
    cleaned_data = []
    for row in raw_data:
        cleaned_row = {
            'user_id': row['id'],
            'full_name': f"{row['first_name']} {row['last_name']}",
            'revenue': row['total_purchases'] * 1.1,  # Add 10% markup
            'tier': 'premium' if row['total_purchases'] > 1000 else 'basic'
        }
        cleaned_data.append(cleaned_row)

    # 3. Load
    load_to_warehouse(cleaned_data)
```

**Pros:**
- Data warehouse only stores clean data
- Transformation logic centralized
- Sensitive data can be filtered before storage

**Cons:**
- Slow (transform before loading)
- Raw data is lost (can't re-transform)
- Transformations are bottleneck

### ELT (Extract, Load, Transform) - Modern

```
Source DB → Extract → Load → Data Warehouse → Transform (SQL in warehouse)
```

**Use case:** Load raw data fast, transform with warehouse's power

```sql
-- ELT Example
-- 1. Extract & Load (raw data loaded directly)
COPY raw_users FROM 's3://bucket/users.csv';

-- 2. Transform (using SQL in warehouse)
CREATE TABLE clean_users AS
SELECT
    id AS user_id,
    first_name || ' ' || last_name AS full_name,
    total_purchases * 1.1 AS revenue,
    CASE
        WHEN total_purchases > 1000 THEN 'premium'
        ELSE 'basic'
    END AS tier
FROM raw_users;
```

**Pros:**
- Fast loading (no pre-processing)
- Raw data preserved (can re-transform)
- Leverage warehouse computing power (BigQuery, Snowflake, Redshift)

**Cons:**
- Stores raw + transformed data (more storage)
- Warehouse sees sensitive data (privacy concern)

### When to Use Which?

| **Criteria**              | **ETL**                          | **ELT**                          |
|---------------------------|----------------------------------|----------------------------------|
| Data volume               | Small to medium (<100GB)         | Large (>1TB)                     |
| Transformation complexity | Complex (Python, Spark)          | Simple (SQL)                     |
| Data sensitivity          | Need to filter before storage    | Can store raw                    |
| Speed priority            | Lower                            | Higher                           |
| Warehouse capability      | Limited (older systems)          | Modern (BigQuery, Snowflake)     |

## Batch Processing

### What is Batch Processing?

**Process large volumes of data at scheduled intervals (hourly, daily, weekly)**

```
Collect data for 24 hours → Process all at once → Output results
```

**Examples:**
- Daily sales reports
- Monthly billing
- Nightly ETL jobs
- End-of-day bank reconciliation

### Batch Processing Architecture

```
Data Sources (DBs, Logs, APIs)
    ↓
Storage (S3, HDFS)
    ↓
Batch Processor (Spark, Hadoop, Airflow)
    ↓ (runs every 24 hours)
Aggregated Results
    ↓
Data Warehouse (Redshift, BigQuery)
```

### Batch Processing Example (Apache Spark)

```python
from pyspark.sql import SparkSession

# Initialize Spark
spark = SparkSession.builder.appName("DailyRevenue").getOrCreate()

# Read data for past 24 hours
orders = spark.read.parquet("s3://data/orders/2023-11-13/")

# Aggregate
daily_revenue = orders.groupBy("product_id").agg(
    {"price": "sum", "order_id": "count"}
).withColumnRenamed("sum(price)", "total_revenue") \
 .withColumnRenamed("count(order_id)", "total_orders")

# Write results
daily_revenue.write.parquet("s3://results/daily_revenue/2023-11-13/")
```

**Run via cron:**
```bash
0 1 * * * /usr/bin/spark-submit daily_revenue_job.py  # 1 AM daily
```

### Batch Processing Pros/Cons

**Pros:**
- High throughput (process millions of records efficiently)
- Simple (no state management)
- Cost-effective (run once, process bulk)
- Easy to debug (rerun failed batches)

**Cons:**
- High latency (results delayed by batch interval)
- All-or-nothing (entire batch fails if one record fails)
- Resource spikes (idle most of the time, peak during run)

## Stream Processing

### What is Stream Processing?

**Process data continuously as it arrives, in real-time or near-real-time**

```
Event arrives → Process immediately → Output result (within milliseconds)
```

**Examples:**
- Fraud detection (flag transaction instantly)
- Real-time dashboards (live metrics)
- Stock trading (price alerts)
- Log monitoring (alert on errors)

### Stream Processing Architecture

```
Data Sources (Kafka, Kinesis, Events)
    ↓ (continuous stream)
Stream Processor (Flink, Kafka Streams, Spark Streaming)
    ↓ (processes events as they arrive)
Results (DB, Dashboard, Alerts)
```

### Stream Processing Example (Kafka Streams)

```java
// Real-time revenue calculation
StreamsBuilder builder = new StreamsBuilder();

KStream<String, Order> orders = builder.stream("orders-topic");

// Aggregate revenue per product in 1-minute windows
KTable<Windowed<String>, Long> revenuePerProduct = orders
    .groupBy((key, order) -> order.getProductId())
    .windowedBy(TimeWindows.of(Duration.ofMinutes(1)))
    .aggregate(
        () -> 0L,  // Initial value
        (key, order, total) -> total + order.getPrice()  // Aggregator
    );

// Output to result topic
revenuePerProduct.toStream()
    .to("revenue-per-product-topic");
```

### Stream Processing Pros/Cons

**Pros:**
- Low latency (milliseconds to seconds)
- Continuous (always on, handles data as it comes)
- Scalable (process millions of events/sec)

**Cons:**
- Complex (state management, exactly-once semantics)
- Resource-intensive (always running)
- Harder to debug (can't easily rerun)
- Ordering challenges (out-of-order events)

## Batch vs Stream: The Trade-offs

| **Aspect**        | **Batch**                        | **Stream**                      |
|-------------------|----------------------------------|---------------------------------|
| Latency           | Minutes to hours                 | Milliseconds to seconds         |
| Throughput        | Very high (millions/min)         | High (thousands/sec)            |
| Complexity        | Low                              | High                            |
| Cost              | Lower (runs periodically)        | Higher (always running)         |
| Use case          | Analytics, reports               | Real-time alerts, dashboards    |
| Fault tolerance   | Retry entire batch               | Checkpoint state, replay        |

### When to Use Batch

- **Historical analysis:** "What were last month's sales?"
- **Reporting:** Daily/weekly/monthly reports
- **Data warehouse loading:** Nightly ETL to Redshift
- **ML model training:** Train on past week's data
- **Billing:** Monthly invoices

### When to Use Stream

- **Real-time monitoring:** Alert when server errors spike
- **Fraud detection:** Flag suspicious transaction immediately
- **Live dashboards:** Real-time metrics
- **Personalization:** Update recommendations as user browses
- **Stock trading:** React to price changes instantly

## Lambda Architecture

### The Problem Lambda Solves

**"I need real-time data (stream) AND accurate historical data (batch)"**

Example: E-commerce dashboard
- **Real-time:** Show orders in last 5 minutes (stream)
- **Historical:** Show total orders this month (batch, more accurate)

### Lambda Architecture Structure

```
Data Sources
    ↓
    ├─→ Batch Layer (Hadoop, Spark)
    │       ↓
    │   Batch Views (accurate, slow, complete)
    │
    └─→ Speed Layer (Storm, Flink)
            ↓
        Real-time Views (fast, approximate, recent)

Query merges both:
    Batch View (0-5min old) + Real-time View (last 5min) = Complete View
```

### Lambda Example: Page Views

**Batch Layer (runs every hour):**
```python
# Process page views from last hour
hourly_views = spark.read.parquet("s3://logs/page_views/2023-11-13-14-00/")
total_views = hourly_views.groupBy("page_id").count()
# Store: page_id -> total_views (0-60 min old data)
```

**Speed Layer (real-time):**
```python
# Process page views as they arrive
stream = kafka.stream("page-views-topic")
recent_views = stream.groupBy("page_id").count()
# Store: page_id -> recent_views (last 60 min)
```

**Query (merge both):**
```python
def get_page_views(page_id):
    batch_count = get_from_batch_view(page_id)     # 0-60 min old
    realtime_count = get_from_speed_view(page_id)  # last 60 min
    return batch_count + realtime_count            # Total
```

### Lambda Pros/Cons

**Pros:**
- Best of both worlds (real-time + accurate)
- Fault-tolerant (batch layer can recompute if speed layer fails)
- Flexible (different tools for different needs)

**Cons:**
- **Complex:** Maintain two pipelines (batch + stream)
- **Duplicate logic:** Same computation in both layers
- **Operational overhead:** Two systems to manage
- **Consistency:** Merging batch + stream views is tricky

### Real-World Lambda: LinkedIn

```
Batch Layer (Hadoop):
    - Process 24 hours of user activity logs
    - Generate "who viewed your profile" counts (accurate)

Speed Layer (Kafka + Samza):
    - Process real-time profile views
    - Update counts for last 24 hours (fast, approximate)

Serving Layer:
    - Merge both: "123 views (accurate) + 5 views (last hour)"
```

## Kappa Architecture

### The Simplification

**"What if we only use stream processing for everything?"**

Kappa removes the batch layer. Everything is a stream (even historical data).

```
Data Sources
    ↓
Kafka (immutable log, stores all events forever)
    ↓
Stream Processor (Flink, Kafka Streams)
    ↓
Views (real-time, recomputable from Kafka)
```

### Kappa Example: Same Page Views

**Single stream processor (Kafka Streams):**
```java
// Process ALL page views (historical + real-time) from Kafka
KStream<String, PageView> allViews = builder.stream("page-views-topic");

KTable<String, Long> totalViews = allViews
    .groupBy((key, view) -> view.getPageId())
    .count();  // Aggregates ALL events in Kafka topic (not just recent)

// To recompute from scratch: Reset consumer, replay topic from beginning
```

**Reprocessing (if logic changes):**
```bash
# Reset consumer offset to beginning
kafka-streams-application-reset --application-id page-views-counter

# Restart processor -> Recomputes from all historical data in Kafka
```

### Kappa Pros/Cons

**Pros:**
- **Simpler:** One pipeline (only stream)
- **No duplicate logic:** Write computation once
- **Recomputable:** Replay Kafka topic to recompute
- **Operational simplicity:** One system to manage

**Cons:**
- **Kafka storage costs:** Must store all history
- **Stream processing complexity:** Everything must be streamable
- **Reprocessing time:** Re-reading entire Kafka topic can be slow
- **Not always feasible:** Some batch algorithms don't stream well

### When to Use Kappa

- You can store all data in Kafka (retention = forever)
- Your computations are streamable (aggregations, filters, joins)
- You need to reprocess data occasionally
- You want operational simplicity

**Example:** Uber's real-time trip analytics (all trips in Kafka, recompute if needed)

## Lambda vs Kappa Decision

| **Factor**                | **Lambda**                       | **Kappa**                        |
|---------------------------|----------------------------------|----------------------------------|
| Latency requirement       | Mixed (real-time + batch)        | All real-time                    |
| Data volume               | Massive (can't fit in Kafka)     | Moderate (fits in Kafka)         |
| Reprocessing frequency    | Rare                             | Frequent (logic changes often)   |
| Team expertise            | Have batch + stream experts      | Strong stream processing skills  |
| Operational complexity    | Can handle two systems           | Prefer simplicity                |

**Senior advice:** Start with **Kappa** if possible. Add batch layer (Lambda) only if:
- Kafka storage is too expensive
- Batch algorithms outperform streaming (e.g., complex ML training)

## Micro-Batch (Spark Streaming)

**Hybrid approach:** Process streams in small batches (e.g., every 1 second)

```python
from pyspark.streaming import StreamingContext

# Create streaming context (1-second batches)
ssc = StreamingContext(sparkContext, 1)

# Read stream from Kafka
stream = KafkaUtils.createStream(ssc, "broker:9092", "group", {"topic": 1})

# Process each micro-batch
stream.map(lambda x: parse_json(x)) \
      .filter(lambda x: x['amount'] > 100) \
      .pprint()

ssc.start()
```

**Pros:**
- Leverage Spark's batch processing APIs
- Easier than true streaming
- Good for latency ~1-5 seconds

**Cons:**
- Higher latency than true streaming (1-5 sec vs milliseconds)
- Less efficient than native streaming (Flink, Kafka Streams)

## Data Pipeline Orchestration

### Apache Airflow (Batch Orchestration)

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

# Define DAG (runs daily)
dag = DAG('daily_etl', schedule_interval='@daily', start_date=datetime(2023, 1, 1))

def extract():
    # Extract data from PostgreSQL
    pass

def transform():
    # Transform with Spark
    pass

def load():
    # Load to Redshift
    pass

# Define task dependencies
extract_task = PythonOperator(task_id='extract', python_callable=extract, dag=dag)
transform_task = PythonOperator(task_id='transform', python_callable=transform, dag=dag)
load_task = PythonOperator(task_id='load', python_callable=load, dag=dag)

extract_task >> transform_task >> load_task  # Pipeline: E → T → L
```

### Real-Time Pipeline (Kafka + Flink)

```
Logs/Events → Kafka → Flink (filter/transform) → Kafka → Elasticsearch
                                                       → S3 (archive)
```

## Change Data Capture (CDC)

**Stream database changes in real-time**

```
PostgreSQL (users table)
    ↓ (Debezium CDC)
Kafka (stream of INSERT/UPDATE/DELETE events)
    ↓
Stream Processor (Flink)
    ↓
- Elasticsearch (for search)
- Redis (for caching)
- Data warehouse (for analytics)
```

**Example event:**
```json
{
  "op": "u",  // UPDATE operation
  "before": { "id": 123, "email": "old@example.com" },
  "after": { "id": 123, "email": "new@example.com" },
  "ts_ms": 1699876543000
}
```

## Common Patterns

### Pattern 1: Batch for Training, Stream for Inference

```
ML Model Training (Batch):
    - Run nightly on 30 days of data (Spark)
    - Train model, save to S3

ML Model Inference (Stream):
    - Real-time predictions on Kafka events (Flink)
    - Load model from S3, apply to incoming data
```

### Pattern 2: Stream to Capture, Batch to Analyze

```
Stream (Kafka):
    - Capture all user clicks (raw, fast)
    - Store in S3

Batch (Spark):
    - Nightly analysis of clicks
    - Generate insights, load to warehouse
```

### Pattern 3: Hot/Warm/Cold Data

```
Hot (Stream):
    - Last 24 hours in Redis (real-time queries)

Warm (Micro-batch):
    - Last 30 days in Elasticsearch (fast queries)

Cold (Batch):
    - 30+ days in S3 (archival, slow queries via Spark)
```

## Key Takeaways

1. **ETL vs ELT:** Modern = ELT (load raw fast, transform in warehouse)
2. **Batch vs Stream:** Batch = high throughput, low cost; Stream = low latency, complex
3. **Lambda:** Two pipelines (batch + stream), complex but comprehensive
4. **Kappa:** One stream pipeline, simpler but requires Kafka storage
5. **Start simple:** Use batch. Add stream only if latency matters.
6. **CDC:** Replicate databases in real-time via change streams
7. **Micro-batch:** Middle ground (Spark Streaming, 1-5 sec latency)

**Senior engineer rule:** Choose the simplest architecture that meets latency requirements. Complexity is expensive.
