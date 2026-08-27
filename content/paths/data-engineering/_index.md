---
title: "Data Engineering"
weight: 46
bookCollapseSection: true
---

# Data Engineering

A comprehensive path through data engineering — from foundational concepts and data modelling to building production pipelines, orchestrating workflows, and operating cloud-native data platforms at scale.

## Sections

| # | Section | Topics |
|---|---------|--------|
| 1 | [Data Engineering Fundamentals]({{< relref "01-fundamentals" >}}) | Role of a data engineer, data lifecycle, batch vs streaming, data quality dimensions |
| 2 | [Data Modelling for Analytics]({{< relref "02-data-modelling" >}}) | Star schema, snowflake schema, slowly changing dimensions, Kimball vs Inmon |
| 3 | [SQL for Data Engineering]({{< relref "03-sql-for-data-engineering" >}}) | Window functions, CTEs, MERGE/UPSERT, pivoting, advanced aggregations, performance tuning |
| 4 | [ETL & ELT Pipelines]({{< relref "04-etl-elt-pipelines" >}}) | ETL vs ELT, pipeline design patterns, idempotency, data validation, scheduling |
| 5 | [Apache Spark]({{< relref "05-apache-spark" >}}) | RDDs, DataFrames, Spark SQL, PySpark, partitioning, caching, Spark on EMR/Databricks |
| 6 | [Workflow Orchestration]({{< relref "06-workflow-orchestration" >}}) | Apache Airflow, Prefect, Dagster, DAGs, operators, sensors, scheduling, backfill |
| 7 | [Streaming & Real-Time Data]({{< relref "07-streaming-real-time" >}}) | Apache Kafka, Kafka Streams, Flink, Kinesis, event-time vs processing-time |
| 8 | [Data Lakes & Lakehouses]({{< relref "08-data-lakes-lakehouses" >}}) | Bronze/silver/gold, Delta Lake, Apache Iceberg, Hudi, schema evolution, time travel |
| 9 | [Data Warehouses]({{< relref "09-data-warehouses" >}}) | BigQuery, Redshift, Snowflake, ClickHouse, columnar storage, MPP, cost models |
| 10 | [dbt (Data Build Tool)]({{< relref "10-dbt" >}}) | Models, sources, seeds, tests, Jinja templating, incremental models, CI/CD for dbt |
| 11 | [Data Quality & Governance]({{< relref "11-data-quality-governance" >}}) | Great Expectations, Soda, data contracts, schema registries, lineage, catalogues |
| 12 | [Cloud Data Platforms]({{< relref "12-cloud-data-platforms" >}}) | AWS data stack, GCP BigQuery/Dataflow, Azure Synapse, choosing a platform |

## Prerequisites

- Solid SQL knowledge (joins, aggregations, subqueries)
- Basic Python programming
- Familiarity with command-line tools and version control
- Understanding of databases (relational concepts, basic NoSQL awareness)

## What You'll Be Able To Do

- Design and build production-grade data pipelines (batch and streaming)
- Model data for analytical workloads using dimensional modelling techniques
- Orchestrate complex workflows with Airflow, Prefect, or Dagster
- Process large-scale data with Apache Spark and streaming frameworks
- Operate data lakes, lakehouses, and cloud data warehouses
- Implement data quality checks, governance policies, and data contracts
- Choose and integrate cloud data platform services for end-to-end architectures
