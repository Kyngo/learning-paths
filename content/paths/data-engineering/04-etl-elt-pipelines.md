---
title: "ETL & ELT Pipelines"
weight: 4
---

## ETL vs ELT

The two dominant pipeline paradigms differ in **where** transformation happens relative to loading.

### ETL (Extract, Transform, Load)

Data is extracted from sources, transformed in a processing engine (Spark, Python, custom code), and then loaded into the target system.

```text
Source ──▶ Extract ──▶ Transform (Spark/Python) ──▶ Load ──▶ Warehouse
```

**When to use:** transformations require compute-heavy logic, data cleansing before landing, legacy systems that can't handle raw data, or when the warehouse lacks transformation power.

### ELT (Extract, Load, Transform)

Data is extracted and loaded raw into the target (usually a cloud warehouse or data lake), then transformed using the target's own compute (SQL, dbt).

```text
Source ──▶ Extract ──▶ Load (raw) ──▶ Transform (SQL/dbt) ──▶ Ready tables
```

**When to use:** cloud warehouses with elastic compute (BigQuery, Snowflake, Redshift), SQL-first transformations, when analysts need access to raw data.

### Comparison

| Aspect | ETL | ELT |
|--------|-----|-----|
| Transform location | External engine | Inside the warehouse |
| Transform language | Python, Spark, Java | SQL, dbt |
| Raw data access | Not in warehouse | Available in warehouse |
| Compute cost model | Separate compute cluster | Warehouse compute (elastic) |
| Complexity | More infrastructure | Less infrastructure |
| Latency | Higher (extra hop) | Lower (fewer stages) |
| Best era | On-premises, Hadoop | Cloud-native, modern data stack |

**Modern practice** strongly favours ELT. Cloud warehouses have made compute cheap and elastic, and dbt has standardised SQL-based transformation.

---

## Pipeline Design Patterns

### Full Load (Snapshot)

Replace the entire target table with a fresh copy from the source on each run.

```python
# Full load pattern
def full_load(source_query: str, target_table: str):
    df = extract(source_query)
    df = transform(df)
    load(df, target_table, mode="overwrite")
```

**Pros:** simple, always consistent, no drift.
**Cons:** inefficient for large tables, wasteful when only a small percentage changes.

### Incremental Load

Load only new or changed records since the last successful run.

```sql
-- Extract only rows modified since last run
SELECT *
FROM source_orders
WHERE updated_at > '{{ last_successful_run }}'
  AND updated_at <= '{{ current_run_timestamp }}';
```

**Pros:** efficient, fast, lower cost.
**Cons:** requires a reliable change indicator (timestamp, CDC), more complex logic.

### Change Data Capture (CDC)

Capture row-level changes (INSERT, UPDATE, DELETE) directly from the source database's transaction log.

| CDC Method | How It Works | Latency | Source Impact |
|------------|-------------|---------|---------------|
| Log-based (Debezium) | Reads DB transaction log | Seconds | Minimal |
| Trigger-based | DB triggers write to change table | Immediate | Moderate |
| Timestamp-based | Query by `updated_at` column | Minutes | Query load |
| Diff-based | Compare snapshots | Hours | Full scan |

```json
// Debezium CDC event (Kafka message)
{
  "before": {"id": 1, "status": "pending", "amount": 100.00},
  "after":  {"id": 1, "status": "shipped", "amount": 100.00},
  "op": "u",
  "ts_ms": 1719849600000,
  "source": {
    "table": "orders",
    "db": "production",
    "connector": "postgresql"
  }
}
```

---

## Idempotency

An idempotent pipeline produces the **same result** whether it runs once or multiple times for the same input. This is the single most important property of a reliable pipeline.

### Why Idempotency Matters

- Retries after failure must not create duplicates
- Backfills must not corrupt existing data
- Orchestrators may re-execute tasks on timeout

### Techniques for Idempotency

**1. Delete-and-replace by partition:**

```sql
-- Delete the partition, then insert fresh data
DELETE FROM fact_orders WHERE order_date = '{{ ds }}';
INSERT INTO fact_orders
SELECT * FROM staging_orders WHERE order_date = '{{ ds }}';
```

**2. MERGE/UPSERT:**

```sql
MERGE INTO dim_product AS t
USING staging_product AS s ON t.product_id = s.product_id
WHEN MATCHED THEN UPDATE SET t.name = s.name, t.updated_at = CURRENT_TIMESTAMP
WHEN NOT MATCHED THEN INSERT (product_id, name, created_at, updated_at)
    VALUES (s.product_id, s.name, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);
```

**3. Write to a temporary table, then swap:**

```python
def idempotent_load(df, target_table: str, partition_date: str):
    tmp_table = f"{target_table}__tmp_{partition_date}"
    write(df, tmp_table, mode="overwrite")
    execute(f"ALTER TABLE {target_table} EXCHANGE PARTITION ('{partition_date}') WITH {tmp_table}")
```

---

## Data Validation

Validate data at each pipeline stage to catch issues before they reach consumers.

### Validation Layers

| Layer | Where | What to Check |
|-------|-------|---------------|
| Schema | On ingestion | Column names, types, nullability |
| Row-level | During transform | Value ranges, formats, referential integrity |
| Aggregate | After load | Row counts, sum totals, null percentages |
| Cross-system | End-to-end | Source row count vs target row count |

### Example: Python Validation

```python
from dataclasses import dataclass

@dataclass
class ValidationResult:
    check_name: str
    passed: bool
    details: str

def validate_orders(df) -> list[ValidationResult]:
    results = []

    # Row count check
    count = len(df)
    results.append(ValidationResult(
        check_name="row_count",
        passed=count > 0,
        details=f"Row count: {count}"
    ))

    # Null check on critical column
    null_pct = df["order_id"].isnull().mean() * 100
    results.append(ValidationResult(
        check_name="order_id_completeness",
        passed=null_pct == 0,
        details=f"Null percentage: {null_pct:.2f}%"
    ))

    # Range check
    future_orders = (df["order_date"] > pd.Timestamp.now()).sum()
    results.append(ValidationResult(
        check_name="no_future_dates",
        passed=future_orders == 0,
        details=f"Future-dated orders: {future_orders}"
    ))

    return results
```

### Example: SQL Validation

```sql
-- Post-load validation query
WITH validation AS (
    SELECT
        COUNT(*) AS total_rows,
        COUNT(DISTINCT order_id) AS unique_orders,
        SUM(CASE WHEN amount < 0 THEN 1 ELSE 0 END) AS negative_amounts,
        SUM(CASE WHEN customer_id IS NULL THEN 1 ELSE 0 END) AS null_customers
    FROM fact_orders
    WHERE order_date = CURRENT_DATE - INTERVAL '1 day'
)
SELECT
    *,
    CASE
        WHEN total_rows = 0 THEN 'FAIL: no rows loaded'
        WHEN total_rows <> unique_orders THEN 'FAIL: duplicate order_ids'
        WHEN negative_amounts > 0 THEN 'FAIL: negative amounts found'
        WHEN null_customers > 0 THEN 'WARN: null customer_ids'
        ELSE 'PASS'
    END AS validation_status
FROM validation;
```

---

## Scheduling and Orchestration Concepts

### Scheduling Patterns

| Pattern | Description | Use Case |
|---------|-------------|----------|
| Time-based | Run at fixed intervals (hourly, daily) | Regular batch loads |
| Event-driven | Trigger on file arrival or message | Data lands in S3, Kafka event |
| Dependency-based | Run when upstream completes | Transform after ingestion |
| Sensor-based | Poll until a condition is met | Wait for file, wait for partition |

### DAG (Directed Acyclic Graph)

Pipelines are modelled as DAGs — tasks are nodes, dependencies are edges. The graph is directed (A must run before B) and acyclic (no circular dependencies).

```text
extract_orders ──▶ transform_orders ──▶ load_fact_orders
                                              │
extract_products ──▶ transform_products ──▶ load_dim_products
                                              │
                                        run_quality_checks ──▶ notify_team
```

### Key Orchestration Properties

| Property | Description |
|----------|-------------|
| Retries | Automatically re-run failed tasks (with backoff) |
| Backfill | Run pipeline for historical dates |
| Catchup | Automatically execute missed runs |
| Alerting | Notify on failure via Slack, email, PagerDuty |
| Logging | Centralised logs for debugging |
| Parameterisation | Pass runtime parameters (dates, environments) |
| Concurrency | Control parallel task execution |

---

## Common File Formats

| Format | Type | Schema | Compression | Splittable | Best For |
|--------|------|--------|-------------|------------|----------|
| CSV | Row-based | None | Gzip | No (when compressed) | Simple interchange |
| JSON | Row-based | None | Gzip | No (when compressed) | Semi-structured, APIs |
| Parquet | Columnar | Embedded | Snappy, Zstd | Yes | Analytics, Spark, warehouses |
| Avro | Row-based | Embedded | Snappy, Deflate | Yes | Kafka, streaming, schema evolution |
| ORC | Columnar | Embedded | Zlib, Snappy | Yes | Hive ecosystem |

**Default choice:** Parquet with Snappy compression for analytical workloads. Avro for streaming and schema evolution.

---

## Key Takeaways

1. **ELT** has replaced ETL as the dominant paradigm in cloud-native data platforms — load raw data, then transform with SQL/dbt inside the warehouse
2. **Idempotency** is the most important pipeline property — every pipeline should produce the same result whether it runs once or ten times for the same input
3. **Incremental loads** with reliable change indicators (timestamps, CDC) are far more efficient than full loads for large tables
4. **Data validation** at every stage (schema, row-level, aggregate, cross-system) catches issues before they reach consumers
5. **Pipelines are DAGs** — tasks with explicit dependencies, retries, and monitoring — not ad hoc scripts run by cron
6. **Parquet** is the default file format for analytical workloads; Avro is preferred for streaming and schema-evolution scenarios
