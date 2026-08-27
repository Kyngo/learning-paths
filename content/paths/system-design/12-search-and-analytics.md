---
title: "Search & Analytics"
weight: 12
---

# Search & Analytics

Most applications eventually need full-text search, analytics dashboards, or both. This section covers the systems and patterns for indexing, querying, and analysing large datasets — from Elasticsearch to data lake architectures.

---

## Full-Text Search

### Why Not Just SQL LIKE?

```sql
SELECT * FROM products WHERE name LIKE '%wireless headphones%';
```

| Problem | Explanation |
|---------|------------|
| No relevance ranking | Results are unordered — no way to rank "best match" |
| No fuzzy matching | "headphnes" won't match "headphones" |
| No stemming | "running" won't match "run" |
| Full table scan | `LIKE '%term%'` cannot use indexes — O(n) |
| No synonym support | "laptop" won't match "notebook" |

### Search Engine Architecture

```
Documents → Analyser (tokenise, stem, normalise) → Inverted Index → Query → Ranked Results
```

**Inverted index:** maps each term to the list of documents containing it:

```
"wireless"    → [doc1, doc3, doc7]
"headphone"   → [doc1, doc5, doc9]
"bluetooth"   → [doc1, doc3, doc5]
```

### Elasticsearch / OpenSearch

| Concept | Description |
|---------|------------|
| Index | Collection of documents (like a database table) |
| Document | JSON object (like a row) |
| Mapping | Schema definition (field types, analysers) |
| Shard | Horizontal partition of an index |
| Replica | Copy of a shard for redundancy and read scaling |
| Analyser | Pipeline: character filter → tokeniser → token filters |

### When to Use Search vs Database

| Use Search Engine | Use Database |
|------------------|-------------|
| Full-text search with relevance | Exact lookups by ID or key |
| Fuzzy matching, typo tolerance | Structured queries (joins, aggregations) |
| Faceted navigation (filters + counts) | Transactions, ACID |
| Log aggregation and analysis | Primary data store |
| Autocomplete / suggestions | Source of truth |

**Rule:** The database is the source of truth. The search index is a derived, eventually consistent view.

---

## OLTP vs OLAP

| | OLTP | OLAP |
|-|------|------|
| Purpose | Serve transactions | Analyse data |
| Queries | Simple, by primary key | Complex, aggregating millions of rows |
| Schema | Normalised (3NF) | Denormalised (star/snowflake schema) |
| Latency | Milliseconds | Seconds to minutes |
| Examples | PostgreSQL, MySQL, DynamoDB | BigQuery, Redshift, Snowflake, ClickHouse |
| Optimised for | Writes + point reads | Columnar scans + aggregations |

### Column-Oriented Storage

OLAP databases store data by column, not by row:

```
Row-oriented (OLTP):
[id:1, name:"Alice", amount:100] [id:2, name:"Bob", amount:200] ...

Column-oriented (OLAP):
id:     [1, 2, 3, 4, ...]
name:   ["Alice", "Bob", "Carol", ...]
amount: [100, 200, 150, 300, ...]
```

Benefits: better compression (similar values together), only read columns needed for the query, vectorised processing.

---

## Analytics Pipeline

### The Modern Data Stack

```
Sources → Ingestion → Storage → Transformation → Serving → Visualisation
```

| Stage | Tools |
|-------|-------|
| Ingestion | Kafka, Kinesis, Firehose, Debezium (CDC), Airbyte |
| Storage | S3 (data lake), Snowflake, BigQuery, Redshift |
| Transformation | dbt, Spark, Airflow |
| Serving | Materialised views, OLAP cubes, pre-aggregated tables |
| Visualisation | Grafana, Metabase, Looker, Tableau |

### Batch vs Stream Processing

| | Batch | Stream |
|-|-------|--------|
| Latency | Minutes to hours | Seconds to milliseconds |
| Completeness | Full dataset | Partial (windows, watermarks) |
| Tools | Spark, dbt, Airflow | Kafka Streams, Flink, Kinesis |
| Use case | Daily reports, ML training | Real-time dashboards, alerting |
| Complexity | Lower | Higher (ordering, exactly-once) |

---

## Time-Series Databases

Optimised for timestamped data (metrics, IoT, financial data):

| Database | Use Case |
|----------|----------|
| InfluxDB | Metrics, IoT |
| TimescaleDB | PostgreSQL extension for time-series |
| Prometheus | Infrastructure metrics (pull-based) |
| ClickHouse | Analytics + time-series at scale |
| Amazon Timestream | Managed AWS time-series |

### Design Patterns

- **Downsampling:** Aggregate old data (1s → 1min → 1h) to reduce storage
- **Retention policies:** Auto-delete data older than N days
- **Tag-based queries:** Filter by dimensions (host, service, region) before aggregating

---

## Data Lake Architecture

| Layer | Contents | Format |
|-------|----------|--------|
| Raw (Bronze) | Unprocessed source data | JSON, CSV, Parquet |
| Cleaned (Silver) | Validated, deduplicated, typed | Parquet, Delta Lake |
| Curated (Gold) | Business-ready aggregates | Parquet, materialised views |

### Lake vs Warehouse

| | Data Lake | Data Warehouse |
|-|-----------|---------------|
| Schema | Schema-on-read (flexible) | Schema-on-write (enforced) |
| Data types | Structured + unstructured | Structured only |
| Cost | Low (object storage) | Higher (compute + storage) |
| Query speed | Slower (unless indexed) | Fast (optimised engine) |
| Use case | ML training, raw archives | BI dashboards, reports |

**Lakehouse** (Delta Lake, Apache Iceberg): combines lake flexibility with warehouse features (ACID, time travel, schema evolution).

---

## Key Takeaways

- Full-text search requires an inverted index (Elasticsearch/OpenSearch). SQL `LIKE` does not scale.
- Search engines are derived views, not sources of truth. Sync from the database.
- OLTP (PostgreSQL) for transactions, OLAP (BigQuery/ClickHouse) for analytics. Different tools for different jobs.
- Column-oriented storage makes analytical queries orders of magnitude faster than row-oriented.
- Batch processing for completeness, stream processing for freshness. Most systems need both.
- Data lake (cheap, flexible) → data warehouse (fast, structured) → lakehouse (both).
