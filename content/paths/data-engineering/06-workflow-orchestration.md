---
title: "Workflow Orchestration"
weight: 6
---

## Why Orchestration?

Data pipelines are not single scripts — they are collections of interdependent tasks that must run in a specific order, on a schedule, with retries, monitoring, and alerting. An **orchestrator** manages this complexity.

Without orchestration, teams resort to cron jobs, manual triggers, and tribal knowledge. This breaks down quickly: no dependency tracking, no retry logic, no visibility into failures, and no way to backfill historical data.

---

## Apache Airflow

Airflow is the most widely adopted workflow orchestrator. Originally created at Airbnb, it defines pipelines as Python code (DAGs) and provides a web UI, scheduler, and extensible operator framework.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **DAG** | Directed Acyclic Graph — a collection of tasks with defined dependencies |
| **Task** | A single unit of work (an operator instance) |
| **Operator** | A template for a task (PythonOperator, BashOperator, SQLOperator, etc.) |
| **Sensor** | A special operator that waits until a condition is met |
| **Connection** | Stored credentials for external systems (databases, APIs, cloud services) |
| **Variable** | Key-value configuration stored in Airflow's metadata DB |
| **XCom** | Cross-communication — pass small data between tasks |
| **Pool** | Limits concurrent task execution for resource-constrained systems |

### A Complete Airflow DAG

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator
from airflow.sensors.s3_key_sensor import S3KeySensor

default_args = {
    "owner": "data-engineering",
    "depends_on_past": False,
    "email_on_failure": True,
    "email": ["data-team@example.com"],
    "retries": 2,
    "retry_delay": timedelta(minutes=5),
}

with DAG(
    dag_id="daily_orders_pipeline",
    default_args=default_args,
    description="Ingest, transform, and validate daily orders",
    schedule="0 6 * * *",  # 06:00 UTC daily
    start_date=datetime(2025, 1, 1),
    catchup=False,
    tags=["orders", "daily"],
) as dag:

    # 1. Wait for source file
    wait_for_file = S3KeySensor(
        task_id="wait_for_orders_file",
        bucket_name="raw-data",
        bucket_key="orders/{{ ds }}/orders.parquet",
        poke_interval=300,  # check every 5 minutes
        timeout=3600,       # fail after 1 hour
        mode="poke",
    )

    # 2. Extract and load raw data
    def extract_orders(**context):
        execution_date = context["ds"]
        # Read from S3, apply schema, write to staging
        # ... (Spark submit or direct S3 read)
        return f"Extracted orders for {execution_date}"

    extract = PythonOperator(
        task_id="extract_orders",
        python_callable=extract_orders,
    )

    # 3. Transform with SQL
    transform = SQLExecuteQueryOperator(
        task_id="transform_orders",
        conn_id="warehouse_conn",
        sql="sql/transform_orders.sql",
        params={"run_date": "{{ ds }}"},
    )

    # 4. Data quality check
    def validate_orders(**context):
        execution_date = context["ds"]
        # Run validation queries, raise on failure
        row_count = run_query(f"SELECT COUNT(*) FROM fact_orders WHERE order_date = '{execution_date}'")
        if row_count == 0:
            raise ValueError(f"No rows loaded for {execution_date}")

    validate = PythonOperator(
        task_id="validate_orders",
        python_callable=validate_orders,
    )

    # Dependencies
    wait_for_file >> extract >> transform >> validate
```

### Templating with Jinja

Airflow templates use Jinja2 to inject runtime values:

| Template Variable | Value | Example Output |
|-------------------|-------|----------------|
| `{{ ds }}` | Execution date (YYYY-MM-DD) | `2025-06-15` |
| `{{ ds_nodash }}` | Execution date without dashes | `20250615` |
| `{{ data_interval_start }}` | Start of the data interval | `2025-06-15T00:00:00+00:00` |
| `{{ data_interval_end }}` | End of the data interval | `2025-06-16T00:00:00+00:00` |
| `{{ params.key }}` | User-defined parameters | (varies) |

### Backfill

Run a DAG for historical dates to fill in missing data:

```bash
# Backfill orders pipeline for the first two weeks of June
airflow dags backfill daily_orders_pipeline \
    --start-date 2025-06-01 \
    --end-date 2025-06-15
```

Backfill works because Airflow treats each execution date as an independent, idempotent run.

### Task Dependencies and Trigger Rules

```python
# Default: all upstream tasks must succeed
task_a >> task_b  # task_b runs only if task_a succeeds

# Trigger rules for non-default behaviour
from airflow.utils.trigger_rule import TriggerRule

cleanup = PythonOperator(
    task_id="cleanup",
    python_callable=cleanup_fn,
    trigger_rule=TriggerRule.ALL_DONE,  # runs regardless of upstream success/failure
)
```

| Trigger Rule | Behaviour |
|-------------|-----------|
| `all_success` | Default — all parents must succeed |
| `all_failed` | All parents must fail |
| `all_done` | All parents complete (success or failure) |
| `one_success` | At least one parent succeeds |
| `none_failed` | No parent has failed (skipped is OK) |

---

## Prefect

Prefect is a modern orchestrator that uses Python decorators instead of explicit DAG definitions.

```python
from prefect import flow, task
from prefect.tasks import task_input_hash
from datetime import timedelta

@task(retries=3, retry_delay_seconds=60, cache_key_fn=task_input_hash, cache_expiration=timedelta(hours=1))
def extract_orders(date: str) -> dict:
    # Extract logic here
    return {"rows": 1500, "date": date}

@task
def transform_orders(raw_data: dict) -> dict:
    # Transform logic here
    return {"rows": raw_data["rows"], "transformed": True}

@task
def validate_orders(data: dict):
    assert data["rows"] > 0, "No rows in output"

@flow(name="daily-orders-pipeline", log_prints=True)
def daily_orders(date: str):
    raw = extract_orders(date)
    transformed = transform_orders(raw)
    validate_orders(transformed)
    print(f"Pipeline complete for {date}: {transformed['rows']} rows")

# Run
daily_orders("2025-06-15")
```

### Airflow vs Prefect

| Aspect | Airflow | Prefect |
|--------|---------|---------|
| DAG definition | Explicit DAG object | Implicit from function calls |
| UI | Self-hosted web server | Cloud UI (Prefect Cloud) or self-hosted |
| Scheduling | Built-in scheduler | Prefect Cloud or deployments |
| Dynamic workflows | TaskFlow API (2.x+) | Native Python control flow |
| Deployment | Heavyweight (scheduler, webserver, workers, DB) | Lightweight (agent + API server) |
| Ecosystem | Massive provider library | Growing, Python-native |
| Maturity | 2015, battle-tested | 2018, rapidly evolving |

---

## Dagster

Dagster is an orchestrator built around **software-defined assets** — you define what data should exist, and Dagster figures out how to materialise it.

```python
from dagster import asset, Definitions, ScheduleDefinition

@asset
def raw_orders():
    """Extract orders from source system."""
    return extract_from_source("orders")

@asset
def clean_orders(raw_orders):
    """Clean and validate order data."""
    df = raw_orders
    df = df.dropna(subset=["order_id"])
    df = df[df["amount"] > 0]
    return df

@asset
def order_summary(clean_orders):
    """Aggregate orders by customer."""
    return clean_orders.groupby("customer_id").agg(
        total_spent=("amount", "sum"),
        order_count=("order_id", "count")
    )

daily_schedule = ScheduleDefinition(
    job=define_asset_job("daily_orders_job", selection=[raw_orders, clean_orders, order_summary]),
    cron_schedule="0 6 * * *",
)

defs = Definitions(
    assets=[raw_orders, clean_orders, order_summary],
    schedules=[daily_schedule],
)
```

### Key Dagster Concepts

| Concept | Description |
|---------|-------------|
| **Asset** | A persistent data object (table, file, ML model) |
| **Op** | A single computation (like an Airflow operator) |
| **Job** | A collection of ops or assets to execute together |
| **Resource** | External system configuration (DB connections, APIs) |
| **IO Manager** | Controls how assets are stored and loaded |
| **Partition** | Data segmented by a key (date, region) |
| **Sensor** | Triggers jobs based on external events |

---

## Choosing an Orchestrator

| Factor | Airflow | Prefect | Dagster |
|--------|---------|---------|---------|
| Best for | Complex, heterogeneous pipelines | Python-native data workflows | Asset-centric data platforms |
| Learning curve | Medium | Low | Medium |
| Operational overhead | High (many components) | Low–Medium | Medium |
| Community size | Very large | Growing | Growing |
| Cloud offering | MWAA, Astronomer, Cloud Composer | Prefect Cloud | Dagster Cloud |
| When to pick | Established teams, diverse task types | Small teams, Python-first | Data asset focus, lineage-first |

---

## Key Takeaways

1. **Orchestrators** manage dependencies, retries, scheduling, and monitoring — they replace fragile cron-based workflows with observable, maintainable pipelines
2. **Apache Airflow** is the industry standard, with DAGs defined in Python, a rich operator ecosystem, and built-in support for backfill and templating
3. **Backfill** is a critical capability — the ability to re-run pipelines for historical dates requires idempotent task design
4. **Prefect** offers a simpler, more Pythonic alternative with decorator-based flows and less operational overhead
5. **Dagster** shifts the mental model from "tasks to execute" to "assets to materialise", providing built-in lineage and partition-aware scheduling
6. **Pick based on team maturity**: Airflow for established teams with diverse workloads, Prefect for Python-first simplicity, Dagster for asset-centric data platforms
