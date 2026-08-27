---
title: "Data Lakes & Lakehouses"
weight: 8
---

## Data Lake Architecture

A data lake stores raw data in object storage (S3, GCS, ADLS) at low cost, in any format. The **medallion architecture** (bronze / silver / gold) organises data by quality level.

### Medallion Architecture

```text
Sources ──▶ Bronze (Raw) ──▶ Silver (Cleaned) ──▶ Gold (Business-ready)
```

| Layer | Purpose | Data Quality | Format | Consumers |
|-------|---------|-------------|--------|-----------|
| **Bronze** | Raw ingestion, exact copy of source | As-is (may contain errors) | JSON, CSV, Avro, Parquet | Data engineers |
| **Silver** | Cleaned, deduplicated, conformed | Validated, typed, deduplicated | Parquet / Delta / Iceberg | Data engineers, analysts |
| **Gold** | Aggregated, business-ready models | Trusted, documented, SLA-backed | Parquet / Delta / Iceberg | Analysts, BI tools, ML |

### Example Directory Layout

```text
s3://data-lake/
├── bronze/
│   ├── orders/
│   │   ├── year=2025/month=06/day=15/
│   │   │   └── orders_20250615_143022.parquet
│   │   └── _schema/orders_v1.avsc
│   └── customers/
│       └── full_export_20250615.json
├── silver/
│   ├── orders/
│   │   └── year=2025/month=06/
│   │       └── part-00000.parquet
│   └── customers/
│       └── part-00000.parquet
└── gold/
    ├── fact_orders/
    │   └── year=2025/month=06/
    └── dim_customers/
        └── part-00000.parquet
```

---

## The Problem with Plain Data Lakes

Raw files on object storage lack features that databases take for granted:

| Missing Feature | Consequence |
|-----------------|-------------|
| ACID transactions | Concurrent writes corrupt data |
| Schema enforcement | Bad data enters undetected |
| Efficient updates/deletes | Must rewrite entire partitions |
| Time travel | No rollback to previous state |
| Statistics/indexing | Full partition scans for every query |

**Table formats** (Delta Lake, Apache Iceberg, Apache Hudi) add these capabilities on top of object storage, creating the **lakehouse**.

---

## Delta Lake

Delta Lake, developed by Databricks, adds ACID transactions, schema enforcement, and time travel to Parquet files on object storage.

### How It Works

Delta Lake stores data as Parquet files plus a **transaction log** (`_delta_log/`) that records every change as a JSON file.

```text
s3://data-lake/silver/orders/
├── _delta_log/
│   ├── 00000000000000000000.json  ← initial commit
│   ├── 00000000000000000001.json  ← second commit
│   └── 00000000000000000010.checkpoint.parquet  ← checkpoint
├── part-00000-abc123.parquet
├── part-00001-def456.parquet
└── part-00002-ghi789.parquet
```

### Delta Lake Operations (PySpark)

```python
from delta.tables import DeltaTable
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .config("spark.jars.packages", "io.delta:delta-spark_2.12:3.2.0") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
    .getOrCreate()

# Write as Delta table
orders_df.write.format("delta").mode("overwrite").save("s3://lake/silver/orders")

# Read Delta table
df = spark.read.format("delta").load("s3://lake/silver/orders")

# MERGE (upsert)
delta_table = DeltaTable.forPath(spark, "s3://lake/silver/orders")

delta_table.alias("target").merge(
    new_orders_df.alias("source"),
    "target.order_id = source.order_id"
).whenMatchedUpdateAll() \
 .whenNotMatchedInsertAll() \
 .execute()

# Time travel — read a previous version
df_v5 = spark.read.format("delta").option("versionAsOf", 5).load("s3://lake/silver/orders")

# Time travel — read as of a timestamp
df_yesterday = spark.read.format("delta") \
    .option("timestampAsOf", "2025-06-14T00:00:00Z") \
    .load("s3://lake/silver/orders")

# View history
delta_table.history().show()
```

### Schema Evolution

```python
# Enable schema evolution — new columns are added automatically
new_df.write.format("delta") \
    .mode("append") \
    .option("mergeSchema", "true") \
    .save("s3://lake/silver/orders")
```

---

## Apache Iceberg

Apache Iceberg is an open table format designed for large analytical datasets. It has broad engine support (Spark, Trino, Flink, Presto, Dremio, Snowflake) and is not tied to any single vendor.

### Iceberg Architecture

```text
┌────────────────┐
│   Catalog       │  ← Tracks current metadata pointer per table
├────────────────┤
│ Metadata Files  │  ← Table schema, partitioning, snapshots
├────────────────┤
│ Manifest Lists  │  ← List of manifest files per snapshot
├────────────────┤
│ Manifest Files  │  ← List of data files with stats (min/max, null counts)
├────────────────┤
│ Data Files      │  ← Parquet/ORC/Avro files
└────────────────┘
```

### Iceberg Operations (Spark SQL)

```sql
-- Create an Iceberg table
CREATE TABLE catalog.silver.orders (
    order_id    STRING,
    customer_id STRING,
    amount      DECIMAL(10, 2),
    order_date  DATE,
    status      STRING
)
USING iceberg
PARTITIONED BY (months(order_date));

-- Insert data
INSERT INTO catalog.silver.orders
SELECT * FROM staging_orders;

-- Upsert with MERGE
MERGE INTO catalog.silver.orders AS target
USING staging_orders AS source
ON target.order_id = source.order_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;

-- Time travel
SELECT * FROM catalog.silver.orders VERSION AS OF 12345678;
SELECT * FROM catalog.silver.orders TIMESTAMP AS OF '2025-06-14 00:00:00';

-- View snapshots
SELECT * FROM catalog.silver.orders.snapshots;

-- Rollback to a previous snapshot
CALL catalog.system.rollback_to_snapshot('silver.orders', 12345678);
```

### Partition Evolution

Iceberg supports changing the partition scheme **without rewriting data** — a unique feature.

```sql
-- Original partitioning: by month
CREATE TABLE orders (...) PARTITIONED BY (months(order_date));

-- Later: add partitioning by status (no rewrite needed)
ALTER TABLE orders ADD PARTITION FIELD status;
```

Iceberg tracks which partition spec was active for each data file and applies the correct pruning logic regardless of when the data was written.

---

## Apache Hudi

Hudi (Hadoop Upserts Deletes Incrementals) specialises in **upsert-heavy workloads** and near-real-time ingestion.

### Hudi Table Types

| Type | Write | Read | Use Case |
|------|-------|------|----------|
| Copy-on-Write (CoW) | Rewrites files on write | Fast reads (Parquet) | Read-heavy workloads |
| Merge-on-Read (MoR) | Appends to log files | Merges on read | Write-heavy, near-real-time |

---

## Table Format Comparison

| Feature | Delta Lake | Apache Iceberg | Apache Hudi |
|---------|-----------|---------------|-------------|
| ACID transactions | Yes | Yes | Yes |
| Time travel | Yes | Yes | Yes |
| Schema evolution | Yes | Yes | Yes |
| Partition evolution | No (requires rewrite) | Yes (without rewrite) | Limited |
| Engine support | Spark, Trino, Flink, Presto | Spark, Trino, Flink, Presto, Snowflake, BigQuery | Spark, Trino, Flink, Presto |
| Vendor alignment | Databricks | Open (Apache Foundation) | Apache Foundation |
| Merge/Upsert | Native | Native | Core feature |
| Hidden partitioning | No | Yes | No |
| File statistics | Yes | Yes (per-column min/max) | Yes |
| Adoption trend | Strong (Databricks ecosystem) | Fastest growing | Niche (upsert-focused) |

---

## Partition Pruning and Performance

All table formats leverage **partition pruning** and **file-level statistics** to skip irrelevant data.

### Partition Pruning

If a table is partitioned by `order_date` and you query `WHERE order_date = '2025-06-15'`, the engine reads only the files in that partition directory.

### Data Skipping (Min/Max Statistics)

Table formats store per-file (and per-column) min/max statistics. If a file's `amount` column has min=10 and max=500, a query for `WHERE amount > 1000` skips that file entirely.

### Z-Ordering / Clustering

Sort data within partitions by frequently filtered columns to improve data skipping:

```sql
-- Delta Lake: Z-order by customer_id and order_date
OPTIMIZE silver.orders ZORDER BY (customer_id, order_date);

-- Iceberg: sort order
ALTER TABLE silver.orders WRITE ORDERED BY customer_id, order_date;
```

---

## Key Takeaways

1. **The medallion architecture** (bronze → silver → gold) organises data lakes by quality level — raw ingestion, cleaned/validated, and business-ready models
2. **Table formats** (Delta Lake, Iceberg, Hudi) solve the fundamental problems of plain data lakes by adding ACID transactions, schema enforcement, time travel, and efficient updates
3. **Apache Iceberg** offers the broadest engine compatibility and unique features like hidden partitioning and partition evolution without data rewrites
4. **Delta Lake** is the strongest choice within the Databricks ecosystem and has growing support in other engines
5. **Time travel** enables rollbacks, auditing, and reproducible queries — specify a version number or timestamp to read historical data
6. **Performance tuning** in lakehouses relies on partition pruning, file-level statistics (min/max), and clustering/Z-ordering to minimise data scanned per query
