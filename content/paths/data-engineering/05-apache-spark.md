---
title: "Apache Spark"
weight: 5
---

## What Is Apache Spark?

Apache Spark is a distributed computing engine for large-scale data processing. It runs batch and streaming workloads across clusters of machines, processing data in memory for speed. Spark is the de facto standard for big data processing.

### Spark Architecture

```text
┌─────────────┐
│   Driver     │  ← Your application code
│  (SparkContext) │
└──────┬──────┘
       │
  ┌────┴────┐
  │ Cluster  │  ← YARN, Kubernetes, Standalone, Mesos
  │ Manager  │
  └────┬────┘
       │
┌──────┴──────┬──────────────┐
│  Executor 1  │  Executor 2  │  Executor N  │
│  (Tasks)     │  (Tasks)     │  (Tasks)     │
│  [Cache]     │  [Cache]     │  [Cache]     │
└─────────────┴──────────────┘
```

**Driver** — the main process that creates the SparkSession, defines transformations, and coordinates execution.
**Executors** — worker processes that execute tasks and cache data in memory.
**Cluster Manager** — allocates resources (YARN, Kubernetes, or Spark Standalone).

---

## RDDs, DataFrames, and Datasets

Spark offers three APIs, each building on the previous:

| API | Type Safety | Optimisation | Language Support | When to Use |
|-----|------------|-------------|-----------------|-------------|
| RDD | Compile-time (Scala/Java) | No Catalyst optimiser | Scala, Java, Python | Low-level control, custom partitioning |
| DataFrame | Runtime only | Catalyst + Tungsten | Scala, Java, Python, R | Most workloads (recommended) |
| Dataset | Compile-time (Scala/Java) | Catalyst + Tungsten | Scala, Java | Type-safe OOP in JVM languages |

**Use DataFrames for almost everything.** RDDs are rarely needed in modern Spark code.

---

## PySpark Fundamentals

### Creating a SparkSession

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("my-etl-job") \
    .config("spark.sql.adaptive.enabled", "true") \
    .getOrCreate()
```

### Reading Data

```python
# Read Parquet
orders_df = spark.read.parquet("s3://data-lake/raw/orders/")

# Read CSV with schema inference
csv_df = spark.read \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .csv("s3://data-lake/raw/uploads/customers.csv")

# Read with explicit schema (preferred for production)
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, DecimalType, DateType

order_schema = StructType([
    StructField("order_id", StringType(), nullable=False),
    StructField("customer_id", StringType(), nullable=False),
    StructField("amount", DecimalType(10, 2), nullable=False),
    StructField("order_date", DateType(), nullable=False),
    StructField("status", StringType(), nullable=True),
])

orders_df = spark.read.schema(order_schema).parquet("s3://data-lake/raw/orders/")
```

### Transformations and Actions

Spark operations fall into two categories:

| Type | Behaviour | Examples |
|------|-----------|----------|
| **Transformation** | Lazy — builds an execution plan, doesn't run yet | `filter`, `select`, `groupBy`, `join`, `withColumn` |
| **Action** | Triggers execution of the plan | `count`, `show`, `collect`, `write` |

```python
from pyspark.sql import functions as F

# Transformations (lazy — nothing executes yet)
result = (
    orders_df
    .filter(F.col("status") == "completed")
    .withColumn("year", F.year("order_date"))
    .groupBy("year", "customer_id")
    .agg(
        F.sum("amount").alias("total_spent"),
        F.count("order_id").alias("num_orders")
    )
    .filter(F.col("total_spent") > 1000)
    .orderBy(F.desc("total_spent"))
)

# Action (triggers execution)
result.show(20)
```

---

## Spark SQL

Spark SQL lets you query DataFrames with SQL syntax. It uses the same Catalyst optimiser as the DataFrame API — performance is identical.

```python
# Register DataFrame as a temp view
orders_df.createOrReplaceTempView("orders")

# Query with SQL
top_customers = spark.sql("""
    SELECT
        customer_id,
        COUNT(*) AS order_count,
        SUM(amount) AS total_spent,
        AVG(amount) AS avg_order_value
    FROM orders
    WHERE status = 'completed'
      AND order_date >= '2025-01-01'
    GROUP BY customer_id
    HAVING SUM(amount) > 5000
    ORDER BY total_spent DESC
    LIMIT 100
""")

top_customers.show()
```

---

## Partitioning

Partitioning controls how data is distributed across the cluster. Getting partitioning right is key to Spark performance.

### Data Partitioning on Disk

Write data partitioned by commonly filtered columns to enable partition pruning:

```python
# Write partitioned by year and month
(
    orders_df
    .withColumn("year", F.year("order_date"))
    .withColumn("month", F.month("order_date"))
    .write
    .partitionBy("year", "month")
    .mode("overwrite")
    .parquet("s3://data-lake/processed/orders/")
)
```

Reading with a filter on the partition column skips irrelevant directories:

```python
# Only reads the year=2025/month=6/ directory
june_orders = spark.read.parquet("s3://data-lake/processed/orders/") \
    .filter((F.col("year") == 2025) & (F.col("month") == 6))
```

### Shuffle Partitioning

The number of partitions after a shuffle (join, groupBy) defaults to 200. Tune this based on data size:

```python
# Set shuffle partitions
spark.conf.set("spark.sql.shuffle.partitions", 100)
```

**Rule of thumb:** target partitions of 100–200 MB each. Too many small partitions waste overhead; too few large partitions cause memory pressure and long task times.

### Repartition vs Coalesce

```python
# Repartition: full shuffle, can increase or decrease partitions
df = df.repartition(50, "customer_id")  # hash-based repartition

# Coalesce: no shuffle, can only decrease partitions
df = df.coalesce(10)  # use when reducing partitions (e.g., before write)
```

---

## Caching and Persistence

Cache DataFrames that are reused across multiple actions to avoid recomputation.

```python
from pyspark import StorageLevel

# Cache in memory (default)
orders_df.cache()

# Persist with a specific storage level
orders_df.persist(StorageLevel.MEMORY_AND_DISK)

# Trigger materialisation
orders_df.count()

# Unpersist when done
orders_df.unpersist()
```

| Storage Level | Memory | Disk | Serialised | When to Use |
|--------------|--------|------|------------|-------------|
| MEMORY_ONLY | Yes | No | No | Small DataFrames, fast access |
| MEMORY_AND_DISK | Yes | Spill to disk | No | Default recommendation |
| DISK_ONLY | No | Yes | Yes | Very large DataFrames |
| MEMORY_ONLY_SER | Yes | No | Yes | Reduce memory at cost of CPU |

---

## Joins

Spark supports several join strategies, chosen automatically by the Catalyst optimiser:

| Strategy | When Used | Performance |
|----------|-----------|-------------|
| Broadcast Hash Join | One side < `spark.sql.autoBroadcastJoinThreshold` (10 MB default) | Fastest — no shuffle |
| Sort-Merge Join | Both sides are large | Good — requires sort + shuffle |
| Shuffle Hash Join | One side significantly smaller (but > broadcast threshold) | Moderate |

```python
from pyspark.sql.functions import broadcast

# Force broadcast join for a small dimension table
result = orders_df.join(
    broadcast(dim_product_df),
    on="product_id",
    how="inner"
)
```

---

## Spark on Cloud Platforms

| Platform | Service | Key Feature |
|----------|---------|-------------|
| AWS | EMR (Elastic MapReduce) | Managed Spark on EC2, spot instances |
| AWS | Glue | Serverless Spark ETL, integrated with Glue Catalog |
| Databricks | Databricks Runtime | Optimised Spark, Delta Lake native, notebooks |
| GCP | Dataproc | Managed Spark on GCE |
| Azure | HDInsight / Synapse Spark | Managed Spark in Azure ecosystem |

### EMR Example (AWS CLI)

```bash
aws emr create-cluster \
  --name "spark-etl-daily" \
  --release-label emr-7.1.0 \
  --applications Name=Spark \
  --instance-groups \
    InstanceGroupType=MASTER,InstanceCount=1,InstanceType=m6g.xlarge \
    InstanceGroupType=CORE,InstanceCount=4,InstanceType=m6g.2xlarge \
  --steps Type=Spark,Name="daily-etl",\
    ActionOnFailure=TERMINATE_CLUSTER,\
    Args=[--deploy-mode,cluster,s3://my-bucket/jobs/daily_etl.py] \
  --auto-terminate \
  --log-uri s3://my-bucket/emr-logs/
```

---

## Key Takeaways

1. **Use DataFrames over RDDs** for virtually all workloads — the Catalyst optimiser makes DataFrame and Spark SQL performance identical and superior to hand-written RDD code
2. **Transformations are lazy, actions trigger execution** — Spark builds an optimised execution plan before running anything, so chain transformations freely
3. **Partitioning is critical** — partition data on disk by commonly filtered columns, and tune `spark.sql.shuffle.partitions` to match your data volume
4. **Broadcast small tables** in joins to avoid expensive shuffles — any dimension table under a few hundred MB is a candidate
5. **Cache selectively** — only persist DataFrames that are reused across multiple actions; uncache when done to free memory
6. **Spark runs everywhere** — EMR, Databricks, Glue, Dataproc, and Synapse all offer managed Spark with different cost and convenience tradeoffs
