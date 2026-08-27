---
title: "Data Modelling for Analytics"
weight: 2
---

## Why Analytical Data Modelling?

Transactional databases (OLTP) are designed for fast reads and writes of individual records. Analytical databases (OLAP) are designed for aggregating large volumes of data across many dimensions. The schemas that serve OLTP well — highly normalised, with many joins — are inefficient for analytical queries. Analytical data modelling restructures data to optimise for **read-heavy, aggregation-heavy workloads**.

### OLTP vs OLAP

| Aspect | OLTP | OLAP |
|--------|------|------|
| Purpose | Transactional operations | Analytical queries |
| Query pattern | Point lookups, small updates | Full-table scans, aggregations |
| Schema | Normalised (3NF+) | Denormalised (star/snowflake) |
| Users | Applications, microservices | Analysts, BI tools, dashboards |
| Optimised for | Write throughput, consistency | Read throughput, query speed |
| Example | Insert an order | Total revenue by region per quarter |

---

## Dimensional Modelling

Dimensional modelling, pioneered by Ralph Kimball, organises data into **fact tables** (measurements) and **dimension tables** (context). It is the most widely used approach for analytical data warehouses.

### Fact Tables

Fact tables store **quantitative, measurable events** — the "what happened" of your business. Each row represents a single event or transaction.

| Column Type | Description | Examples |
|------------|-------------|----------|
| Foreign keys | Links to dimension tables | `customer_id`, `product_id`, `date_id` |
| Measures | Numeric values to aggregate | `revenue`, `quantity`, `discount_amount` |
| Degenerate dimensions | Identifiers without a dimension table | `order_number`, `invoice_id` |

**Fact table grain** — the grain defines what a single row represents. Getting the grain right is the most critical modelling decision. Example: "one row per order line item per day".

### Dimension Tables

Dimension tables store **descriptive attributes** — the "who, what, where, when, why" that gives context to facts.

```sql
CREATE TABLE dim_customer (
    customer_key    BIGINT PRIMARY KEY,  -- surrogate key
    customer_id     VARCHAR(50),          -- natural key from source
    name            VARCHAR(200),
    email           VARCHAR(200),
    segment         VARCHAR(50),
    city            VARCHAR(100),
    country         VARCHAR(100),
    created_at      DATE,
    is_current      BOOLEAN,
    valid_from      DATE,
    valid_to        DATE
);
```

Key properties of dimension tables:
- **Wide** — many columns (50+ is common)
- **Shallow** — relatively few rows compared to fact tables
- **Descriptive** — text attributes used for filtering and grouping
- **Surrogate keys** — system-generated integer keys, not natural keys

---

## Star Schema

The star schema is the simplest and most common dimensional model. A central fact table connects directly to dimension tables via foreign keys — the diagram resembles a star.

```text
              dim_date
                 │
dim_customer ── fact_sales ── dim_product
                 │
              dim_store
```

```sql
-- Star schema example
CREATE TABLE fact_sales (
    sale_key        BIGINT PRIMARY KEY,
    date_key        INT REFERENCES dim_date(date_key),
    customer_key    BIGINT REFERENCES dim_customer(customer_key),
    product_key     BIGINT REFERENCES dim_product(product_key),
    store_key       INT REFERENCES dim_store(store_key),
    quantity        INT,
    unit_price      DECIMAL(10, 2),
    discount_amount DECIMAL(10, 2),
    total_amount    DECIMAL(10, 2)
);

-- Typical analytical query against a star schema
SELECT
    d.calendar_year,
    d.calendar_quarter,
    p.category,
    s.country,
    SUM(f.total_amount) AS total_revenue,
    SUM(f.quantity) AS total_units
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
JOIN dim_store s ON f.store_key = s.store_key
WHERE d.calendar_year = 2025
GROUP BY d.calendar_year, d.calendar_quarter, p.category, s.country
ORDER BY total_revenue DESC;
```

**Advantages:** simple to understand, fast queries (fewer joins), works well with BI tools.
**Disadvantage:** some data redundancy in dimension tables.

---

## Snowflake Schema

The snowflake schema normalises dimension tables into sub-dimensions. Instead of a single `dim_product` table with a `category_name` column, you have a separate `dim_category` table.

```text
dim_category ── dim_product ── fact_sales ── dim_customer
                                    │
                                dim_date
```

```sql
-- Normalised product dimension
CREATE TABLE dim_category (
    category_key  INT PRIMARY KEY,
    category_name VARCHAR(100),
    department    VARCHAR(100)
);

CREATE TABLE dim_product (
    product_key   BIGINT PRIMARY KEY,
    product_name  VARCHAR(200),
    brand         VARCHAR(100),
    category_key  INT REFERENCES dim_category(category_key),
    unit_cost     DECIMAL(10, 2)
);
```

| Aspect | Star Schema | Snowflake Schema |
|--------|-------------|-----------------|
| Dimension normalisation | Denormalised | Normalised |
| Number of joins | Fewer | More |
| Query simplicity | Simpler | More complex |
| Storage efficiency | More redundancy | Less redundancy |
| ETL complexity | Simpler loads | More complex loads |
| BI tool compatibility | Excellent | Good (some tools prefer star) |

**In practice**, star schemas are preferred for most analytical workloads. The additional joins in snowflake schemas rarely save enough storage to justify the query complexity.

---

## Slowly Changing Dimensions (SCD)

Dimensions change over time — a customer moves city, a product changes category. **Slowly Changing Dimensions (SCD)** are strategies for handling these changes while preserving historical accuracy.

### SCD Type 0: Retain Original

Never update the dimension. The original value is always kept. Used for attributes that should never change (e.g., date of birth, original signup date).

### SCD Type 1: Overwrite

Simply overwrite the old value with the new one. No history is preserved.

```sql
-- SCD Type 1: customer moves from London to Manchester
UPDATE dim_customer
SET city = 'Manchester'
WHERE customer_id = 'C-1001';
```

**Use when:** history doesn't matter for that attribute, or the old value was incorrect.

### SCD Type 2: Add New Row

Create a new row for each change, with validity dates. This is the most common approach for preserving history.

```sql
-- Before: single active row
-- customer_key | customer_id | city    | is_current | valid_from | valid_to
-- 1001         | C-1001      | London  | true       | 2023-01-15 | 9999-12-31

-- After SCD Type 2 update:
UPDATE dim_customer
SET is_current = FALSE, valid_to = '2025-06-01'
WHERE customer_key = 1001;

INSERT INTO dim_customer (customer_key, customer_id, city, is_current, valid_from, valid_to)
VALUES (1002, 'C-1001', 'Manchester', TRUE, '2025-06-01', '9999-12-31');
```

**Result:** fact rows linked to `customer_key = 1001` reflect the London era; new facts use `customer_key = 1002` (Manchester).

### SCD Type 3: Add New Column

Add a column for the previous value. Tracks only one level of history.

```sql
ALTER TABLE dim_customer ADD COLUMN previous_city VARCHAR(100);

UPDATE dim_customer
SET previous_city = city, city = 'Manchester'
WHERE customer_id = 'C-1001';
```

### SCD Type Comparison

| Type | History preserved | Storage impact | Query complexity | Use case |
|------|------------------|----------------|-----------------|----------|
| 0 | None (immutable) | None | Simple | Fixed attributes |
| 1 | None (overwrite) | None | Simple | Corrections, non-critical attributes |
| 2 | Full history | Rows grow over time | Medium (filter on `is_current`) | Most dimensions needing history |
| 3 | One previous value | One extra column | Simple | Limited change tracking |

---

## Kimball vs Inmon

The two dominant schools of thought for data warehouse architecture:

### Kimball (Bottom-Up)

- Build **dimensional models** (star schemas) organised by business process
- Each business process gets its own fact table (sales, inventory, shipping)
- **Conformed dimensions** (shared across fact tables) provide integration
- Data marts are the warehouse — no separate integrated layer
- Faster time to value; iterative delivery

### Inmon (Top-Down)

- Build a **centralised, normalised enterprise data warehouse** (3NF)
- Data marts are created from the warehouse for departmental use
- Single source of truth with strict enterprise-wide modelling
- Longer initial build time; comprehensive from day one

### Comparison

| Aspect | Kimball | Inmon |
|--------|---------|-------|
| Approach | Bottom-up | Top-down |
| Central store | Dimensional (star schemas) | Normalised (3NF) |
| Data marts | Are the warehouse | Built from the warehouse |
| Integration | Conformed dimensions | Enterprise data model |
| Time to first value | Weeks | Months |
| Maintenance | Per business process | Enterprise-wide |
| Best for | Agile teams, iterative | Large enterprises, strict governance |

**Modern practice** typically follows a Kimball-influenced approach, with dbt and ELT patterns making iterative dimensional modelling the standard. The Inmon approach is still relevant in heavily regulated industries where a single normalised source is required.

---

## The Date Dimension

Every analytical model needs a date dimension. It is pre-populated with one row per day and contains attributes that make time-based analysis simple.

```sql
CREATE TABLE dim_date (
    date_key          INT PRIMARY KEY,       -- YYYYMMDD format
    full_date         DATE NOT NULL,
    day_of_week       SMALLINT,              -- 1=Monday, 7=Sunday
    day_name          VARCHAR(10),           -- 'Monday', 'Tuesday', ...
    day_of_month      SMALLINT,
    day_of_year       SMALLINT,
    week_of_year      SMALLINT,
    calendar_month    SMALLINT,
    month_name        VARCHAR(10),
    calendar_quarter  SMALLINT,
    calendar_year     SMALLINT,
    is_weekend        BOOLEAN,
    is_holiday        BOOLEAN,
    fiscal_quarter    SMALLINT,
    fiscal_year       SMALLINT
);

-- Query: revenue by fiscal quarter
SELECT
    d.fiscal_year,
    d.fiscal_quarter,
    SUM(f.total_amount) AS revenue
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
GROUP BY d.fiscal_year, d.fiscal_quarter
ORDER BY d.fiscal_year, d.fiscal_quarter;
```

---

## Key Takeaways

1. **Dimensional modelling** (fact tables + dimension tables) is the standard approach for analytical data warehouses — it optimises for read-heavy, aggregation-focused queries
2. **Star schemas** are preferred over snowflake schemas for most use cases — fewer joins, simpler queries, better BI tool compatibility
3. **SCD Type 2** (new row with validity dates) is the most common strategy for tracking dimension changes while preserving full history
4. **The grain** of a fact table — what a single row represents — is the most important modelling decision; get it wrong and the model fails
5. **Kimball's bottom-up approach** dominates modern practice, especially with dbt and ELT patterns enabling iterative dimensional modelling
6. **Every warehouse needs a date dimension** — pre-populated, rich in attributes, and shared across all fact tables as a conformed dimension
