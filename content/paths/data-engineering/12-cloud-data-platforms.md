---
title: "Cloud Data Platforms"
weight: 12
---

## Cloud Data Platforms Overview

Every major cloud provider offers a suite of managed services that together form a complete data platform — ingestion, storage, transformation, orchestration, serving, and governance. Choosing the right combination depends on existing cloud investments, team skills, and specific workload requirements.

---

## AWS Data Stack

AWS has the broadest set of data services, though the landscape can feel fragmented.

### Core Services

| Service | Role | Description |
|---------|------|-------------|
| S3 | Storage | Object storage — the foundation of every AWS data lake |
| Glue | ETL + Catalogue | Serverless Spark ETL, schema crawler, Data Catalog |
| Athena | Query | Serverless SQL over S3 (Presto/Trino engine) |
| Redshift | Warehouse | MPP columnar warehouse (provisioned or serverless) |
| Lake Formation | Governance | Fine-grained access control, data sharing, lineage |
| Kinesis | Streaming | Data Streams (Kafka-like), Firehose (delivery), Analytics (Flink) |
| EMR | Processing | Managed Spark, Hive, Presto, Flink clusters |
| Step Functions | Orchestration | Serverless workflow orchestration |
| MWAA | Orchestration | Managed Apache Airflow |
| MSK | Streaming | Managed Apache Kafka |

### AWS Data Lake Architecture

```text
Sources                     Ingestion              Storage           Transform          Serve
─────────                   ─────────              ───────           ─────────          ─────
RDS (PostgreSQL) ──▶ Glue ETL Job ──▶ S3 (bronze) ──▶ Glue ETL / dbt ──▶ S3 (gold) ──▶ Athena
Kinesis Streams ──▶ Firehose ──────▶ S3 (bronze)                               └──▶ Redshift
Third-party APIs ──▶ Lambda ───────▶ S3 (bronze)                               └──▶ QuickSight

                    Glue Data Catalog (schema registry, table metadata)
                    Lake Formation (access control, permissions)
```

### AWS Glue ETL Job (PySpark)

```python
import sys
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.transforms import *
from pyspark.context import SparkContext

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(sys.argv[0], {"JOB_NAME": "orders_bronze_to_silver"})

# Read from Glue Data Catalog
orders_dyf = glueContext.create_dynamic_frame.from_catalog(
    database="raw_db",
    table_name="raw_orders"
)

# Transform
orders_df = orders_dyf.toDF()
silver_df = (
    orders_df
    .filter(orders_df.order_id.isNotNull())
    .withColumn("amount", orders_df.amount.cast("decimal(10,2)"))
    .withColumn("order_date", orders_df.order_date.cast("date"))
    .dropDuplicates(["order_id"])
)

# Write to S3 as Parquet, partitioned
silver_df.write \
    .mode("overwrite") \
    .partitionBy("year", "month") \
    .parquet("s3://data-lake/silver/orders/")

job.commit()
```

### Athena Query

```sql
-- Query S3 data via Glue Data Catalog — no infrastructure to manage
SELECT
    order_date,
    COUNT(*) AS order_count,
    SUM(amount) AS total_revenue
FROM silver_db.orders
WHERE year = '2025' AND month = '06'
GROUP BY order_date
ORDER BY order_date;
```

Athena pricing: **$5 per TB scanned**. Use Parquet (not CSV) and partition tables to minimise cost.

### Lake Formation Permissions

```text
-- Grant fine-grained access (column-level and row-level)
Grant SELECT on database silver_db
    table orders
    columns (order_id, order_date, amount, status)
    to IAM role data-analyst-role
    with data filter (country = 'GB')
```

---

## GCP Data Stack

GCP centres around BigQuery as the universal analytics engine, with a tightly integrated ecosystem.

### Core Services

| Service | Role | Description |
|---------|------|-------------|
| GCS | Storage | Object storage (Google Cloud Storage) |
| BigQuery | Warehouse + Lake | Serverless analytics, also supports external tables |
| Dataflow | Processing | Managed Apache Beam (batch + streaming) |
| Pub/Sub | Messaging | Fully managed pub/sub messaging |
| Dataproc | Processing | Managed Spark/Hadoop clusters |
| Cloud Composer | Orchestration | Managed Apache Airflow |
| Data Catalog | Governance | Metadata management, tagging, search |
| Dataform | Transformation | dbt-like SQL transformations (acquired by Google) |

### GCP Data Pipeline

```text
Cloud SQL / APIs ──▶ Dataflow ──▶ GCS (bronze) ──▶ Dataform / dbt ──▶ BigQuery (gold)
Pub/Sub ──────────▶ Dataflow ──▶ BigQuery (streaming insert)
                                                                └──▶ Looker (BI)
```

### Dataflow (Apache Beam) Example

```python
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions

options = PipelineOptions([
    "--runner=DataflowRunner",
    "--project=my-gcp-project",
    "--region=europe-west1",
    "--temp_location=gs://my-bucket/temp/",
])

with beam.Pipeline(options=options) as p:
    (
        p
        | "Read from Pub/Sub" >> beam.io.ReadFromPubSub(topic="projects/my-project/topics/orders")
        | "Parse JSON" >> beam.Map(lambda msg: json.loads(msg))
        | "Filter valid" >> beam.Filter(lambda order: order.get("amount", 0) > 0)
        | "Window" >> beam.WindowInto(beam.window.FixedWindows(300))  # 5-minute windows
        | "Count per status" >> beam.combiners.Count.PerKey(lambda order: order["status"])
        | "Write to BigQuery" >> beam.io.WriteToBigQuery(
            table="my_project:analytics.order_counts",
            schema="status:STRING,count:INTEGER",
            write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND,
        )
    )
```

---

## Azure Data Stack

Azure organises data services around Synapse Analytics as the unified experience.

### Core Services

| Service | Role | Description |
|---------|------|-------------|
| ADLS Gen2 | Storage | Azure Data Lake Storage (built on Blob Storage) |
| Synapse Analytics | Warehouse + Spark | Unified: SQL pools, Spark pools, pipelines |
| Data Factory | ETL/Orchestration | Visual ETL and orchestration (similar to Glue) |
| Event Hubs | Streaming | Kafka-compatible event ingestion |
| Stream Analytics | Streaming | SQL over streaming data |
| Databricks (Azure) | Processing | Managed Databricks on Azure |
| Purview | Governance | Data catalog, lineage, classification |

### Azure Data Pipeline

```text
SQL Server / APIs ──▶ Data Factory ──▶ ADLS Gen2 (bronze) ──▶ Synapse Spark / dbt ──▶ Synapse SQL pool (gold)
Event Hubs ─────────▶ Stream Analytics ──▶ ADLS Gen2 / Synapse                       └──▶ Power BI
                                                                                       
                     Purview (catalog, lineage, governance)
```

---

## Platform Comparison

| Aspect | AWS | GCP | Azure |
|--------|-----|-----|-------|
| Warehouse | Redshift | BigQuery | Synapse SQL |
| Serverless query | Athena | BigQuery | Synapse Serverless SQL |
| ETL engine | Glue (Spark) | Dataflow (Beam) | Data Factory + Synapse Spark |
| Orchestration | MWAA (Airflow), Step Functions | Cloud Composer (Airflow) | Data Factory, Synapse Pipelines |
| Streaming | Kinesis, MSK (Kafka) | Pub/Sub, Dataflow | Event Hubs, Stream Analytics |
| Governance | Lake Formation | Data Catalog | Purview |
| BI integration | QuickSight | Looker | Power BI |
| Spark | EMR, Glue | Dataproc | Synapse Spark, Databricks |
| Strengths | Breadth, maturity, flexibility | BigQuery simplicity, AI/ML | Enterprise integration, Power BI |
| Weakness | Service sprawl, complex IAM | Fewer services, GCP lock-in | Synapse complexity |

---

## Choosing a Platform

### Decision Factors

| Factor | Guidance |
|--------|----------|
| Existing cloud provider | Use the data services of your primary cloud — avoid multi-cloud complexity unless necessary |
| Team skills | SQL-first teams → BigQuery or Snowflake; Spark teams → EMR or Databricks |
| Scale | All three scale to petabytes; choose based on cost model and operational preference |
| Streaming needs | Kafka (MSK) for high throughput; Pub/Sub for simplicity; Event Hubs for Azure |
| BI tool | Power BI → Azure; Looker → GCP; QuickSight or Tableau → AWS |
| Governance maturity | Lake Formation (AWS), Purview (Azure), or Unity Catalog (Databricks) |
| Multi-cloud | Snowflake or Databricks — both run on all three clouds |

### Multi-Cloud Options

| Tool | Runs On | Best For |
|------|---------|----------|
| Snowflake | AWS, GCP, Azure | Multi-cloud warehouse, data sharing |
| Databricks | AWS, GCP, Azure | Multi-cloud lakehouse, ML |
| dbt | Any warehouse | Portable transformations |
| Airflow | Any cloud (self-hosted or managed) | Portable orchestration |
| Apache Kafka | Any cloud (self-hosted or Confluent Cloud) | Portable streaming |

### Common Architecture Patterns

**Lakehouse on AWS:**
S3 + Glue Catalog + Delta Lake/Iceberg + Athena/Redshift Spectrum + dbt + Airflow (MWAA)

**Lakehouse on GCP:**
GCS + BigQuery (with external tables) + Dataform/dbt + Cloud Composer

**Lakehouse on Databricks (any cloud):**
Object storage + Delta Lake + Unity Catalog + Databricks SQL + Databricks Workflows

---

## Key Takeaways

1. **AWS** offers the broadest set of data services (Glue, Athena, Redshift, Kinesis, EMR, Lake Formation) but requires more architecture decisions due to service overlap
2. **GCP** centres around BigQuery as a unified serverless warehouse — simple, powerful, and tightly integrated with Dataflow and Looker
3. **Azure** provides Synapse Analytics as a unified experience combining SQL pools, Spark, and pipelines, with strong Power BI and enterprise integration
4. **Snowflake and Databricks** run on all three clouds, offering portability and avoiding lock-in — consider them when multi-cloud is a requirement
5. **Choose based on existing investments** — the best platform is the one your team already knows, integrated with the cloud you already use
6. **The core pattern is consistent** across all clouds: object storage (lake) → catalogue (metadata) → compute engine (warehouse/Spark) → orchestrator (Airflow/native) → BI tool — the services differ but the architecture is the same
