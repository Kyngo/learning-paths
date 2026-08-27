---
title: "Data Warehouses"
weight: 9
---

## What Is a Data Warehouse?

A data warehouse is a system optimised for **analytical queries** — aggregations, joins, and scans over large volumes of data. Unlike transactional databases (OLTP), warehouses are designed for read-heavy, column-oriented workloads where queries touch many rows but few columns.

### Key Properties

| Property | Description |
|----------|-------------|
| Columnar storage | Data stored by column, not by row — reads only needed columns |
| MPP architecture | Massively Parallel Processing — queries split across many nodes |
| Compression | Columnar data compresses well (similar values together) |
| Schema-on-write | Data is structured and typed at load time |
| Separation of storage and compute | Most cloud warehouses decouple these for elastic scaling |

---

## Columnar Storage

Row-based storage (PostgreSQL, MySQL) stores each row contiguously on disk. Columnar storage stores each column contiguously.

```text
Row-based (OLTP):
  Row 1: [id=1, name="Alice", amount=100, date="2025-06-15"]
  Row 2: [id=2, name="Bob",   amount=200, date="2025-06-15"]
  Row 3: [id=3, name="Carol", amount=150, date="2025-06-16"]

Columnar (OLAP):
  id:     [1, 2, 3]
  name:   ["Alice", "Bob", "Carol"]
  amount: [100, 200, 150]
  date:   ["2025-06-15", "2025-06-15", "2025-06-16"]
```

**Why columnar wins for analytics:**
- `SELECT SUM(amount) FROM orders` reads only the `amount` column — not every column in the table
- Similar values in a column compress far better than mixed-type row data
- SIMD (vectorised) processing works on contiguous arrays of the same type

---

## MPP Architecture

Massively Parallel Processing distributes data and computation across multiple nodes.

```text
                 ┌─────────────┐
Client query ──▶ │   Leader     │  ← Parses, optimises, distributes
                 │   Node       │
                 └──────┬──────┘
           ┌────────────┼────────────┐
           ▼            ▼            ▼
     ┌──────────┐ ┌──────────┐ ┌──────────┐
     │ Compute  │ │ Compute  │ │ Compute  │
     │ Node 1   │ │ Node 2   │ │ Node 3   │
     │ (slice)  │ │ (slice)  │ │ (slice)  │
     └──────────┘ └──────────┘ └──────────┘
```

Each compute node processes a slice of the data in parallel. The leader node coordinates the work and returns the combined result. This is how Redshift, BigQuery, Snowflake, and ClickHouse achieve sub-second queries over terabytes.

---

## Cloud Data Warehouses

### BigQuery (Google Cloud)

**Architecture:** serverless — no clusters to manage. Compute and storage are fully separated.

```sql
-- BigQuery SQL (standard SQL with extensions)
SELECT
    DATE_TRUNC(order_date, MONTH) AS order_month,
    country,
    COUNT(*) AS order_count,
    SUM(amount) AS total_revenue,
    APPROX_QUANTILES(amount, 100)[OFFSET(50)] AS median_amount
FROM `project.dataset.orders`
WHERE order_date >= '2025-01-01'
GROUP BY 1, 2
ORDER BY total_revenue DESC;
```

| Feature | Detail |
|---------|--------|
| Pricing model | On-demand ($6.25/TB scanned) or flat-rate slots |
| Storage | Columnar (Capacitor format), auto-compressed |
| Partitioning | Time-based, integer-range, ingestion-time |
| Clustering | Up to 4 columns, auto-sorted within partitions |
| Max columns | 10,000 |
| Streaming inserts | Yes (via Streaming API) |
| ML | Built-in (BigQuery ML) |

### Amazon Redshift

**Architecture:** managed MPP cluster with leader + compute nodes. Redshift Serverless available for auto-scaling.

```sql
-- Redshift: create a table with distribution and sort keys
CREATE TABLE fact_orders (
    order_id    BIGINT ENCODE az64,
    customer_id BIGINT ENCODE az64,
    order_date  DATE   ENCODE delta,
    amount      DECIMAL(10,2) ENCODE az64,
    status      VARCHAR(20) ENCODE lzo
)
DISTKEY(customer_id)      -- distribute rows by customer_id
SORTKEY(order_date);       -- sort by order_date for range queries
```

| Feature | Detail |
|---------|--------|
| Pricing model | Per-node-hour (provisioned) or RPU-hours (serverless) |
| Distribution | KEY, ALL, EVEN, AUTO |
| Sort keys | Compound or interleaved |
| Compression | Automatic (AZ64, LZO, Zstd, etc.) |
| Spectrum | Query S3 data directly via external tables |
| Concurrency | Concurrency Scaling (auto-adds clusters) |

### Snowflake

**Architecture:** fully separates storage, compute (virtual warehouses), and cloud services. Multiple compute clusters can query the same data independently.

```sql
-- Snowflake: virtual warehouse management
ALTER WAREHOUSE analytics_wh SET
    WAREHOUSE_SIZE = 'MEDIUM'
    AUTO_SUSPEND = 300     -- suspend after 5 minutes idle
    AUTO_RESUME = TRUE
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 3; -- multi-cluster auto-scaling

-- Clustering key for large tables
ALTER TABLE fact_orders CLUSTER BY (order_date, customer_id);

-- Query with result caching
SELECT country, SUM(amount) AS revenue
FROM fact_orders
WHERE order_date BETWEEN '2025-01-01' AND '2025-06-30'
GROUP BY country;
-- Second execution returns cached result instantly
```

| Feature | Detail |
|---------|--------|
| Pricing model | Credits per compute-second + storage per TB/month |
| Compute isolation | Each warehouse is independent |
| Data sharing | Zero-copy sharing across accounts |
| Time travel | Up to 90 days |
| Semi-structured | Native VARIANT type for JSON |
| Multi-cloud | Available on AWS, GCP, Azure |

### ClickHouse

**Architecture:** open-source, columnar OLAP database designed for sub-second analytical queries. Often used for real-time analytics and observability.

```sql
-- ClickHouse: create a table with MergeTree engine
CREATE TABLE events (
    event_date Date,
    user_id    UInt64,
    event_type LowCardinality(String),
    duration   Float32,
    properties String
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_type, user_id, event_date);

-- Fast aggregation query
SELECT
    event_type,
    count() AS cnt,
    avg(duration) AS avg_duration,
    quantile(0.95)(duration) AS p95_duration
FROM events
WHERE event_date >= '2025-06-01'
GROUP BY event_type
ORDER BY cnt DESC;
```

| Feature | Detail |
|---------|--------|
| Pricing model | Self-hosted (free) or ClickHouse Cloud |
| Engine | MergeTree (primary), plus specialised engines |
| Ingestion speed | Millions of rows per second |
| Compression | LZ4, ZSTD — typically 10–15× compression |
| SQL | Mostly standard with ClickHouse extensions |
| Best for | Real-time analytics, observability, logs |

---

## Warehouse Comparison

| Aspect | BigQuery | Redshift | Snowflake | ClickHouse |
|--------|----------|----------|-----------|------------|
| Architecture | Serverless | MPP cluster / serverless | Separated compute + storage | Self-hosted / Cloud |
| Scaling | Automatic | Manual (resize) or serverless | Instant (resize warehouse) | Add nodes |
| Pricing | Per-TB scanned or flat-rate | Per-node-hour or RPU | Per-credit + storage | Self-hosted or usage-based |
| Semi-structured | Yes (JSON) | Yes (SUPER type) | Yes (VARIANT) | Yes (JSON functions) |
| Streaming support | Streaming inserts | Kinesis integration | Snowpipe | Kafka engine, HTTP |
| Open source | No | No | No | Yes |
| Multi-cloud | GCP only | AWS only | AWS, GCP, Azure | Any cloud / on-prem |
| Best for | GCP-native, ad-hoc | AWS-native, heavy ETL | Multi-cloud, data sharing | Real-time, self-hosted |

---

## External Tables

Query data stored in object storage (S3, GCS) without loading it into the warehouse.

```sql
-- BigQuery external table
CREATE EXTERNAL TABLE dataset.external_logs
OPTIONS (
    format = 'PARQUET',
    uris = ['gs://my-bucket/logs/year=2025/*']
);

-- Redshift Spectrum
CREATE EXTERNAL SCHEMA spectrum_schema
FROM DATA CATALOG
DATABASE 'my_glue_db'
IAM_ROLE 'arn:aws:iam::123456789:role/redshift-spectrum-role';

SELECT * FROM spectrum_schema.external_logs WHERE log_date = '2025-06-15';

-- Snowflake external table
CREATE EXTERNAL TABLE ext_logs (
    log_date DATE AS (VALUE:log_date::DATE),
    message  STRING AS (VALUE:message::STRING)
)
WITH LOCATION = @my_s3_stage/logs/
FILE_FORMAT = (TYPE = PARQUET);
```

**Use external tables for:** cold data, infrequent queries, cost savings on rarely accessed data. For hot data queried frequently, load it into native warehouse tables for better performance.

---

## Cost Optimisation

| Strategy | Applies To | Impact |
|----------|-----------|--------|
| Partition/cluster tables | All warehouses | Reduce data scanned per query |
| Use columnar formats | External tables | Avoid scanning CSV/JSON |
| Set auto-suspend | Snowflake, Redshift Serverless | Don't pay for idle compute |
| Use on-demand/serverless | Infrequent workloads | Pay only for what you use |
| Limit SELECT * | All warehouses | Read only needed columns |
| Materialise hot queries | All warehouses | Avoid repeated computation |
| Use result caching | Snowflake, BigQuery | Identical repeated queries are free |

---

## Key Takeaways

1. **Columnar storage** is what makes analytical warehouses fast — they read only the columns needed and compress similar values efficiently
2. **MPP architecture** distributes queries across multiple compute nodes in parallel, enabling sub-second responses over terabytes of data
3. **BigQuery** is serverless and pay-per-query, ideal for GCP-native teams; **Redshift** is deeply integrated with AWS; **Snowflake** offers multi-cloud flexibility and compute isolation
4. **ClickHouse** fills the real-time analytics niche with sub-second queries, open-source licensing, and exceptionally fast ingestion
5. **External tables** let you query object storage data without loading it, useful for cold data and cost savings — but performance is lower than native tables
6. **Cost optimisation** in warehouses centres on reducing data scanned (partitioning, clustering, columnar formats) and minimising idle compute (auto-suspend, serverless)
