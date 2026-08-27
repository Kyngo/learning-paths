---
title: "dbt (Data Build Tool)"
weight: 10
---

## What Is dbt?

dbt (data build tool) is a SQL-first transformation framework for the **T in ELT**. It lets data engineers and analytics engineers write SELECT statements that dbt compiles into DDL/DML and executes against the warehouse. dbt manages dependencies, runs tests, generates documentation, and integrates with CI/CD.

### Why dbt Exists

Before dbt, transformations were scattered across stored procedures, Python scripts, and scheduling tools. dbt brings **software engineering practices** to SQL transformations: version control, testing, documentation, modularity, and code review.

### dbt Core vs dbt Cloud

| Feature | dbt Core | dbt Cloud |
|---------|----------|-----------|
| Execution | CLI (local or CI/CD) | Managed (web IDE, scheduler) |
| Scheduling | External (Airflow, cron) | Built-in scheduler |
| Cost | Free (open source) | Paid (SaaS) |
| IDE | Your editor + terminal | Web-based IDE |
| Hosting | Self-managed | Managed |

---

## Project Structure

```text
my_dbt_project/
├── dbt_project.yml           # Project configuration
├── profiles.yml              # Connection configuration (local only)
├── models/
│   ├── staging/              # 1:1 with source tables
│   │   ├── stg_orders.sql
│   │   ├── stg_customers.sql
│   │   └── _staging.yml      # Source + model docs
│   ├── intermediate/         # Reusable building blocks
│   │   └── int_order_items_enriched.sql
│   └── marts/                # Business-ready models
│       ├── fct_orders.sql
│       ├── dim_customers.sql
│       └── _marts.yml        # Model docs + tests
├── seeds/                    # CSV files loaded as tables
│   └── country_codes.csv
├── macros/                   # Reusable SQL (Jinja)
│   └── generate_surrogate_key.sql
├── tests/                    # Custom data tests
│   └── assert_positive_amounts.sql
├── snapshots/                # SCD Type 2 tracking
│   └── snap_customers.sql
└── packages.yml              # External dbt packages
```

---

## Models

A dbt model is a SQL SELECT statement saved as a `.sql` file. dbt handles the CREATE TABLE / CREATE VIEW DDL.

### Staging Model

Staging models are 1:1 with source tables. They rename columns, cast types, and apply light cleaning.

```sql
-- models/staging/stg_orders.sql
WITH source AS (
    SELECT * FROM {{ source('ecommerce', 'raw_orders') }}
)

SELECT
    id AS order_id,
    user_id AS customer_id,
    CAST(order_date AS DATE) AS order_date,
    CAST(amount AS DECIMAL(10, 2)) AS amount,
    LOWER(status) AS status,
    CAST(created_at AS TIMESTAMP) AS created_at
FROM source
WHERE id IS NOT NULL
```

### Mart Model (Fact Table)

```sql
-- models/marts/fct_orders.sql
WITH orders AS (
    SELECT * FROM {{ ref('stg_orders') }}
),

customers AS (
    SELECT * FROM {{ ref('stg_customers') }}
),

enriched AS (
    SELECT
        o.order_id,
        o.customer_id,
        c.customer_name,
        c.country,
        o.order_date,
        o.amount,
        o.status,
        o.created_at
    FROM orders o
    LEFT JOIN customers c ON o.customer_id = c.customer_id
)

SELECT * FROM enriched
```

### Model Configuration

```sql
-- Inline config at the top of a model
{{
    config(
        materialized='table',
        schema='marts',
        tags=['daily', 'finance'],
        cluster_by=['order_date', 'customer_id']
    )
}}
```

Or in `dbt_project.yml`:

```yaml
models:
  my_project:
    staging:
      +materialized: view
      +schema: staging
    marts:
      +materialized: table
      +schema: marts
```

### Materialisation Types

| Type | SQL Generated | When to Use |
|------|--------------|-------------|
| `view` | CREATE VIEW | Lightweight, source-adjacent models |
| `table` | CREATE TABLE AS | Models queried frequently |
| `incremental` | INSERT / MERGE | Large tables; append or upsert new rows |
| `ephemeral` | CTE (inlined) | Reusable logic not needed as a table |

---

## Sources and Seeds

### Sources

Declare raw tables so dbt can track lineage and test freshness.

```yaml
# models/staging/_staging.yml
version: 2

sources:
  - name: ecommerce
    database: raw_db
    schema: public
    tables:
      - name: raw_orders
        loaded_at_field: _loaded_at
        freshness:
          warn_after: {count: 12, period: hour}
          error_after: {count: 24, period: hour}
      - name: raw_customers
```

```bash
# Check source freshness
dbt source freshness
```

### Seeds

Small CSV files that dbt loads as tables — useful for reference data.

```csv
# seeds/country_codes.csv
code,name,region
GB,United Kingdom,Europe
US,United States,North America
DE,Germany,Europe
```

```bash
dbt seed  # loads CSV files as tables in the warehouse
```

---

## Tests

### Built-in Generic Tests

```yaml
# models/marts/_marts.yml
version: 2

models:
  - name: fct_orders
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: customer_id
        tests:
          - not_null
          - relationships:
              to: ref('dim_customers')
              field: customer_id
      - name: amount
        tests:
          - not_null
```

### Custom Singular Tests

Write a SQL query that returns rows that **should not exist** (failures):

```sql
-- tests/assert_positive_amounts.sql
SELECT order_id, amount
FROM {{ ref('fct_orders') }}
WHERE amount < 0
```

If this query returns any rows, the test fails.

### Custom Generic Tests (Macros)

```sql
-- macros/test_is_positive.sql
{% test is_positive(model, column_name) %}
SELECT {{ column_name }}
FROM {{ model }}
WHERE {{ column_name }} < 0
{% endtest %}
```

```yaml
# Usage in YAML
columns:
  - name: amount
    tests:
      - is_positive
```

---

## Jinja Templating

dbt uses Jinja2 for dynamic SQL generation.

```sql
-- Conditional logic
SELECT
    order_id,
    amount,
    {% if target.name == 'prod' %}
        customer_email  -- include PII only in prod
    {% else %}
        'REDACTED' AS customer_email
    {% endif %}
FROM {{ ref('stg_orders') }}
```

```sql
-- Looping
SELECT
    order_id,
    {% for status in ['pending', 'shipped', 'delivered', 'cancelled'] %}
        SUM(CASE WHEN status = '{{ status }}' THEN 1 ELSE 0 END) AS {{ status }}_count
        {% if not loop.last %},{% endif %}
    {% endfor %}
FROM {{ ref('stg_orders') }}
GROUP BY order_id
```

---

## Incremental Models

For large tables, rebuild only the new or changed rows.

```sql
-- models/marts/fct_page_views.sql
{{
    config(
        materialized='incremental',
        unique_key='page_view_id',
        incremental_strategy='merge',
        on_schema_change='append_new_columns'
    )
}}

SELECT
    page_view_id,
    user_id,
    page_url,
    view_timestamp,
    duration_seconds
FROM {{ ref('stg_page_views') }}

{% if is_incremental() %}
    -- Only process new rows since last run
    WHERE view_timestamp > (SELECT MAX(view_timestamp) FROM {{ this }})
{% endif %}
```

| Strategy | Warehouse Support | Behaviour |
|----------|-------------------|-----------|
| `append` | All | INSERT only (no dedup) |
| `merge` | Snowflake, BigQuery, Databricks | MERGE on `unique_key` |
| `delete+insert` | Redshift, Postgres | Delete matching rows, then insert |
| `insert_overwrite` | Spark, BigQuery | Overwrite affected partitions |

---

## dbt Packages

Reusable macros and models published by the community.

```yaml
# packages.yml
packages:
  - package: dbt-labs/dbt_utils
    version: "1.3.0"
  - package: dbt-labs/codegen
    version: "0.12.0"
```

```bash
dbt deps  # install packages
```

```sql
-- Using dbt_utils: generate a surrogate key
SELECT
    {{ dbt_utils.generate_surrogate_key(['order_id', 'line_item_id']) }} AS order_line_key,
    *
FROM {{ ref('stg_order_lines') }}
```

---

## CI/CD for dbt

### CI Pipeline (on merge request)

```yaml
# .gitlab-ci.yml (or GitHub Actions equivalent)
dbt_ci:
  stage: test
  script:
    - dbt deps
    - dbt build --select state:modified+ --defer --state ./prod-manifest
    - dbt test --select state:modified+
  artifacts:
    paths:
      - target/manifest.json
```

Key CI commands:
- `dbt build --select state:modified+` — build only changed models and their downstream dependants
- `--defer --state ./prod-manifest` — use production tables for unchanged models (avoid rebuilding everything)
- `dbt test` — run tests on modified models

### Documentation

```bash
dbt docs generate  # generates a documentation site
dbt docs serve     # serve locally at http://localhost:8080
```

dbt auto-generates a lineage graph (DAG), column descriptions, and test results into a browsable HTML site.

---

## Key Takeaways

1. **dbt** brings software engineering practices (version control, testing, documentation, CI/CD) to SQL-based data transformations — it is the standard tool for the T in ELT
2. **Models** are SELECT statements organised into layers: staging (1:1 with sources), intermediate (reusable logic), and marts (business-ready fact and dimension tables)
3. **`ref()` and `source()`** are the foundation of dbt's dependency graph — never hardcode table names; dbt uses these to build, test, and document in the correct order
4. **Incremental models** avoid full-table rebuilds by processing only new or changed rows, using strategies like merge, delete+insert, or insert_overwrite
5. **Tests** (unique, not_null, relationships, custom SQL) run after every build to catch data quality issues before they reach consumers
6. **CI/CD integration** with `state:modified+` and `--defer` enables fast, safe deployments — only rebuild what changed, test it, and promote to production
