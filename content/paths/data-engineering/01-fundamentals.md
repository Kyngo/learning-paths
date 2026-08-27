---
title: "Data Engineering Fundamentals"
weight: 1
---

## What Is Data Engineering?

Data engineering is the discipline of designing, building, and maintaining the systems that collect, store, transform, and serve data. While data scientists build models and analysts extract insights, data engineers build the **infrastructure and pipelines** that make those activities possible.

### The Data Engineer's Responsibilities

| Responsibility | Description |
|---------------|-------------|
| Ingestion | Collecting data from source systems (APIs, databases, files, streams) |
| Storage | Choosing and managing data stores (lakes, warehouses, object storage) |
| Transformation | Cleaning, enriching, and reshaping data for consumption |
| Orchestration | Scheduling and monitoring pipeline execution |
| Data quality | Validating, testing, and monitoring data correctness |
| Infrastructure | Provisioning compute, networking, and storage for data workloads |
| Security | Managing access control, encryption, and compliance |

### Data Engineering vs Adjacent Roles

| Role | Focus | Typical Tools |
|------|-------|---------------|
| Data Engineer | Pipelines, infrastructure, data platforms | Spark, Airflow, Kafka, SQL, dbt |
| Data Scientist | Models, experiments, statistical analysis | Python, R, Jupyter, scikit-learn |
| Data Analyst | Reporting, dashboards, ad-hoc queries | SQL, Tableau, Looker, Excel |
| Analytics Engineer | Transform + model data for analysts | dbt, SQL, version control |
| ML Engineer | Productionise ML models | MLflow, Kubeflow, TensorFlow Serving |
| Platform Engineer | Infrastructure for data teams | Terraform, Kubernetes, CI/CD |

---

## The Data Lifecycle

Data moves through a series of stages from creation to consumption. Understanding this lifecycle is essential for building reliable systems.

```text
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Generate  │───▶│  Ingest   │───▶│  Store    │───▶│ Transform │───▶│  Serve   │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
                                                                       │
                                                                       ▼
                                                                ┌──────────┐
                                                                │  Consume  │
                                                                └──────────┘
```

### Stage Details

**Generation** — data originates in source systems: application databases, user events, IoT sensors, third-party APIs, logs, and files.

**Ingestion** — data is pulled or pushed into the data platform. This can be batch (scheduled extracts), micro-batch (frequent small loads), or real-time (streaming).

**Storage** — data lands in a storage layer appropriate to its stage: raw files in object storage (S3, GCS), structured tables in a warehouse, or topics in a message broker.

**Transformation** — raw data is cleaned, validated, joined, aggregated, and enriched into models suitable for analysis. This is where most data engineering logic lives.

**Serving** — transformed data is made available to consumers through query engines, APIs, dashboards, or ML feature stores.

**Consumption** — analysts, scientists, applications, and ML models read the served data.

---

## Batch vs Streaming

The two fundamental processing paradigms in data engineering are **batch** and **streaming**. Most production systems use both.

### Batch Processing

Batch processing operates on **bounded datasets** — a finite collection of records processed as a unit. Data accumulates over a period (hourly, daily) and is processed all at once.

```text
Source DB ──▶ Extract (daily) ──▶ Transform ──▶ Load into warehouse
```

**Characteristics:**
- High throughput, efficient resource use
- Higher latency (minutes to hours)
- Simpler error handling and reprocessing
- Easier to reason about correctness

**Common tools:** Apache Spark, dbt, AWS Glue, SQL-based ETL

### Streaming Processing

Streaming processes **unbounded datasets** — data arrives continuously and is processed record-by-record or in small windows.

```text
Event source ──▶ Message broker ──▶ Stream processor ──▶ Sink
```

**Characteristics:**
- Low latency (seconds to milliseconds)
- More complex state management
- Harder to debug and reprocess
- Requires careful handling of late-arriving data

**Common tools:** Apache Kafka, Apache Flink, Kafka Streams, Amazon Kinesis

### Comparison

| Aspect | Batch | Streaming |
|--------|-------|-----------|
| Latency | Minutes to hours | Milliseconds to seconds |
| Data model | Bounded (finite) | Unbounded (infinite) |
| Throughput | Very high | Moderate to high |
| Complexity | Lower | Higher |
| Error recovery | Re-run the batch | Offset management, checkpoints |
| State management | Not required (stateless transforms) | Often required (windows, aggregations) |
| Cost model | Pay per job (compute hours) | Always-on infrastructure |
| Use cases | Reporting, warehousing, ML training | Fraud detection, alerting, real-time dashboards |

### Micro-batch: The Middle Ground

Some systems (Spark Structured Streaming, for example) process data in very small batches (seconds), combining the programming model of batch with near-real-time latency. This is often called **micro-batch** processing.

---

## Data Quality Dimensions

Data is only useful if it is trustworthy. Data quality is measured across several dimensions:

| Dimension | Definition | Example Violation |
|-----------|-----------|-------------------|
| **Accuracy** | Data correctly represents the real-world entity | Customer age is 250 |
| **Completeness** | No missing values where data is expected | Email field is NULL for 30% of users |
| **Consistency** | Same fact has the same value across systems | Order total differs between CRM and warehouse |
| **Timeliness** | Data is available when needed | Dashboard shows yesterday's data at 3 PM |
| **Uniqueness** | No unintended duplicates | Same order appears twice in the fact table |
| **Validity** | Data conforms to defined formats and ranges | Phone number contains letters |

### Measuring Data Quality

```sql
-- Completeness check: percentage of non-null emails
SELECT
    COUNT(*) AS total_rows,
    COUNT(email) AS non_null_emails,
    ROUND(100.0 * COUNT(email) / COUNT(*), 2) AS completeness_pct
FROM customers;

-- Uniqueness check: find duplicate order IDs
SELECT order_id, COUNT(*) AS occurrences
FROM orders
GROUP BY order_id
HAVING COUNT(*) > 1;

-- Validity check: dates in the future
SELECT *
FROM orders
WHERE order_date > CURRENT_DATE;
```

### The Cost of Poor Data Quality

Poor data quality compounds through the pipeline. A single corrupt source can cascade into wrong reports, flawed ML models, and incorrect business decisions. The earlier you catch quality issues, the cheaper they are to fix — this is known as the **shift-left** principle for data quality.

---

## Data Architecture Patterns

### Data Warehouse (Traditional)

A centralised repository of integrated, structured data optimised for analytical queries. Sources are transformed before loading (ETL) or loaded then transformed (ELT).

### Data Lake

A storage layer (typically object storage like S3) that holds raw data in its native format — structured, semi-structured, and unstructured. Schema is applied at read time, not write time.

### Data Lakehouse

Combines the flexibility of a data lake with the performance and ACID guarantees of a warehouse. Technologies like Delta Lake, Apache Iceberg, and Apache Hudi add table-format capabilities on top of object storage.

### Data Mesh

A decentralised architecture where domain teams own and publish their data as products, with a self-serve data platform underneath. Each domain manages its own pipelines and quality.

### Comparison

| Pattern | Schema enforcement | Storage cost | Query performance | Governance | Best for |
|---------|-------------------|-------------|-------------------|------------|----------|
| Warehouse | Schema-on-write | High | Excellent | Centralised | BI, reporting |
| Lake | Schema-on-read | Low | Variable | Harder | ML, exploration |
| Lakehouse | Both | Low–Medium | Good–Excellent | Centralised | Unified analytics |
| Mesh | Per domain | Variable | Variable | Federated | Large organisations |

---

## Underpinning Technologies

A data engineer's toolkit spans several technology categories:

| Category | Examples | Purpose |
|----------|----------|---------|
| Languages | Python, SQL, Scala, Java | Pipeline logic, transformations |
| Processing | Spark, Flink, dbt | Batch and stream computation |
| Orchestration | Airflow, Prefect, Dagster | Workflow scheduling and monitoring |
| Storage | S3, GCS, ADLS | Raw and processed data |
| Warehouses | BigQuery, Redshift, Snowflake | Analytical query engines |
| Messaging | Kafka, Kinesis, Pub/Sub | Event streaming |
| Formats | Parquet, Avro, ORC, JSON | Data serialisation |
| Table formats | Delta Lake, Iceberg, Hudi | ACID on object storage |
| Catalogues | Glue Catalog, Hive Metastore, Unity Catalog | Schema registry and discovery |
| Quality | Great Expectations, Soda, dbt tests | Data validation |

---

## Key Takeaways

1. **Data engineers** build and maintain the infrastructure and pipelines that move data from sources to consumers — they are the backbone of any data-driven organisation
2. **The data lifecycle** (generate → ingest → store → transform → serve → consume) provides a framework for reasoning about where each tool and process fits
3. **Batch processing** trades latency for simplicity and throughput; **streaming** delivers low latency at the cost of complexity — most production systems use both
4. **Data quality** is measured across six dimensions (accuracy, completeness, consistency, timeliness, uniqueness, validity) and should be checked as early as possible in the pipeline
5. **Architecture patterns** (warehouse, lake, lakehouse, mesh) each have distinct tradeoffs — the right choice depends on organisational size, data maturity, and use cases
6. **The modern data stack** combines SQL-first transformations, cloud-native storage, columnar formats, and workflow orchestration into a cohesive platform
