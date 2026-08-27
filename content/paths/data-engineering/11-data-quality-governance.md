---
title: "Data Quality & Governance"
weight: 11
---

## Why Data Quality and Governance?

A pipeline that delivers wrong data on time is worse than one that delivers nothing — consumers make decisions based on the output. Data governance ensures data is **correct, discoverable, secure, and compliant**. Without it, trust erodes and data becomes a liability instead of an asset.

---

## Data Quality Frameworks

### Great Expectations

Great Expectations (GX) is a Python framework for defining, running, and documenting data quality checks.

```python
import great_expectations as gx

context = gx.get_context()

# Connect to a data source
datasource = context.data_sources.add_pandas("orders_source")
data_asset = datasource.add_dataframe_asset("orders")
batch = data_asset.add_batch_definition_whole_dataframe("full").get_batch(
    batch_parameters={"dataframe": orders_df}
)

# Define expectations
expectations = context.suites.add(gx.ExpectationSuite(name="orders_suite"))
expectations.add_expectation(gx.expectations.ExpectColumnValuesToNotBeNull(column="order_id"))
expectations.add_expectation(gx.expectations.ExpectColumnValuesToBeUnique(column="order_id"))
expectations.add_expectation(gx.expectations.ExpectColumnValuesToBeBetween(
    column="amount", min_value=0, max_value=1_000_000
))
expectations.add_expectation(gx.expectations.ExpectColumnValuesToBeInSet(
    column="status", value_set=["pending", "shipped", "delivered", "cancelled"]
))

# Validate
validation_result = batch.validate(expectations)
print(f"Success: {validation_result.success}")
print(f"Results: {len(validation_result.results)} expectations checked")
```

### Soda

Soda uses a YAML-based DSL called **SodaCL** for defining data quality checks.

```yaml
# checks/orders.yml
checks for orders:
  # Row count
  - row_count > 0

  # Freshness
  - freshness(created_at) < 24h

  # Completeness
  - missing_count(order_id) = 0
  - missing_percent(email) < 5%

  # Validity
  - invalid_count(amount) = 0:
      valid min: 0
  - invalid_percent(status) = 0:
      valid values: ["pending", "shipped", "delivered", "cancelled"]

  # Uniqueness
  - duplicate_count(order_id) = 0

  # Cross-check
  - row_count same as orders in source_db

  # Schema
  - schema:
      fail:
        when required column missing: [order_id, customer_id, amount]
        when wrong type:
          order_id: varchar
          amount: decimal
```

```bash
# Run Soda checks
soda scan -d warehouse -c configuration.yml checks/orders.yml
```

### Comparison

| Feature | Great Expectations | Soda | dbt Tests |
|---------|-------------------|------|-----------|
| Language | Python | YAML (SodaCL) | SQL + YAML |
| Integration | Python scripts, Airflow | CLI, Airflow, CI/CD | Built into dbt |
| Data profiling | Yes | Yes | No |
| Documentation | Auto-generated HTML | Soda Cloud dashboards | dbt docs site |
| Alerting | Custom + integrations | Soda Cloud | CI/CD failures |
| Best for | Python-heavy pipelines | Ops-friendly, YAML-first | dbt-centric workflows |

---

## Data Contracts

A data contract is a **formal agreement** between a data producer and consumer specifying the schema, quality guarantees, and SLAs of a dataset.

### Contract Structure

```yaml
# contracts/orders.contract.yml
apiVersion: v1
kind: DataContract
metadata:
  name: orders
  owner: order-service-team
  domain: commerce
  version: "2.1.0"

schema:
  type: table
  columns:
    - name: order_id
      type: STRING
      required: true
      unique: true
      description: Unique order identifier
    - name: customer_id
      type: STRING
      required: true
      description: FK to customers
    - name: amount
      type: DECIMAL(10,2)
      required: true
      constraints:
        - min: 0
    - name: status
      type: STRING
      required: true
      allowed_values: [pending, shipped, delivered, cancelled]
    - name: order_date
      type: DATE
      required: true

quality:
  freshness:
    max_delay: 2h
  completeness:
    order_id: 100%
    customer_id: 100%
    amount: 100%
  uniqueness:
    order_id: 100%

sla:
  availability: 99.5%
  update_frequency: hourly
```

### Why Contracts Matter

| Without Contracts | With Contracts |
|-------------------|----------------|
| Schema changes break downstream silently | Breaking changes require version bump and consumer notification |
| No ownership — "who owns this table?" | Clear owner per dataset |
| Quality is discovered after breakage | Quality is guaranteed and monitored |
| Producers don't know who consumes their data | Consumers are registered, impact is assessable |

---

## Schema Registries

A schema registry stores and manages schemas for data in transit (Kafka topics, event streams). It enforces compatibility rules to prevent breaking changes.

### Confluent Schema Registry (Kafka)

```bash
# Register an Avro schema
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{
    "schema": "{\"type\":\"record\",\"name\":\"Order\",\"fields\":[{\"name\":\"order_id\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"double\"},{\"name\":\"status\",\"type\":\"string\"}]}"
  }' \
  http://schema-registry:8081/subjects/orders-value/versions

# Get the latest schema
curl http://schema-registry:8081/subjects/orders-value/versions/latest
```

### Compatibility Modes

| Mode | Rule | Safe Changes |
|------|------|-------------|
| BACKWARD | New schema can read old data | Add optional fields, remove fields |
| FORWARD | Old schema can read new data | Add fields, remove optional fields |
| FULL | Both backward and forward | Add/remove optional fields only |
| NONE | No compatibility check | Any change (dangerous) |

**Default recommendation:** BACKWARD compatibility. Consumers upgrade before producers.

---

## Data Lineage

Lineage tracks where data comes from, how it is transformed, and where it goes. It answers: "if this source changes, what is affected?"

### Lineage Sources

| Tool | How It Captures Lineage |
|------|------------------------|
| dbt | Parses SQL `ref()` and `source()` calls |
| Apache Atlas | Hooks into Hive, Spark, Kafka |
| OpenLineage | Open standard — integrates with Airflow, Spark, dbt |
| Datahub | Ingests metadata from multiple sources |
| Unity Catalog | Native lineage for Databricks |

### OpenLineage Example (Airflow)

```python
# OpenLineage integration captures lineage automatically
# when using supported operators (SQLOperator, SparkOperator, etc.)

# In airflow.cfg or environment:
# OPENLINEAGE_URL=http://marquez:5000
# OPENLINEAGE_NAMESPACE=production

# Lineage is emitted as events:
{
    "eventType": "COMPLETE",
    "job": {"namespace": "production", "name": "daily_orders_pipeline.transform_orders"},
    "inputs": [{"namespace": "warehouse", "name": "staging.stg_orders"}],
    "outputs": [{"namespace": "warehouse", "name": "marts.fct_orders"}]
}
```

---

## Data Catalogues

A data catalogue is a **searchable inventory** of all data assets, their schemas, owners, descriptions, and lineage. It enables data discovery — finding the right dataset without asking someone.

### Popular Catalogues

| Catalogue | Type | Key Feature |
|-----------|------|-------------|
| AWS Glue Data Catalog | Managed (AWS) | Integrated with Athena, Redshift, EMR |
| Datahub | Open source | Rich lineage, metadata search |
| Apache Atlas | Open source | Hive/Hadoop ecosystem |
| Unity Catalog | Databricks | Unified governance for Databricks |
| Alation | Commercial | AI-driven discovery |
| Google Data Catalog | Managed (GCP) | Integrated with BigQuery |

### What a Good Catalogue Provides

| Capability | Benefit |
|-----------|---------|
| Search | Find datasets by name, tag, or description |
| Schema browsing | View columns, types, and sample data |
| Ownership | Know who to contact for questions |
| Lineage | Understand upstream dependencies and downstream impact |
| Documentation | Rich descriptions, business glossary |
| Quality metrics | Freshness, completeness, test results |
| Access control | Request and manage permissions |

---

## Compliance (GDPR)

Data engineers must build systems that comply with privacy regulations. GDPR is the most impactful.

### Key GDPR Requirements for Data Engineers

| Requirement | Data Engineering Implication |
|------------|------------------------------|
| Right to erasure | Ability to delete all data for a given user |
| Right to access | Ability to export all data for a given user |
| Data minimisation | Don't collect more than needed |
| Purpose limitation | Data used only for declared purposes |
| Pseudonymisation | Replace PII with pseudonyms where possible |
| Retention limits | Automated deletion after retention period |
| Breach notification | Logging and monitoring for unauthorised access |

### Practical Patterns

```sql
-- Pseudonymisation: hash PII in silver/gold layers
SELECT
    order_id,
    SHA2(customer_email, 256) AS customer_email_hash,
    country,  -- non-PII, kept as-is
    amount
FROM bronze.orders;

-- Right to erasure: delete by customer_id across all tables
DELETE FROM silver.orders WHERE customer_id = 'CUST-123';
DELETE FROM gold.fct_orders WHERE customer_id = 'CUST-123';
DELETE FROM gold.dim_customers WHERE customer_id = 'CUST-123';
```

**Tip:** maintain a PII registry — a document listing every table, column, and pipeline that contains personal data. This makes compliance audits and erasure requests manageable.

---

## Key Takeaways

1. **Data quality tools** (Great Expectations, Soda, dbt tests) automate validation at every pipeline stage — catching issues before they reach consumers is far cheaper than fixing them after
2. **Data contracts** formalise the agreement between producers and consumers — schema, quality guarantees, and SLAs — preventing silent breakage
3. **Schema registries** enforce compatibility rules on streaming data, preventing producers from publishing breaking schema changes
4. **Data lineage** (OpenLineage, dbt lineage, Datahub) tracks the flow of data from source to consumption, enabling impact analysis and debugging
5. **Data catalogues** make data discoverable — without one, engineers waste time asking "who owns this table?" and "where does this data come from?"
6. **GDPR compliance** requires data engineers to build systems that support erasure, access requests, pseudonymisation, and retention policies from day one — retrofitting is expensive
