---
title: "SQL for Data Engineering"
weight: 3
---

## SQL as the Data Engineer's Core Language

SQL is the lingua franca of data engineering. Every warehouse, most processing engines (Spark SQL, Presto, Trino), and transformation tools (dbt) speak SQL. Mastering advanced SQL patterns is non-negotiable.

---

## Common Table Expressions (CTEs)

CTEs improve readability by breaking complex queries into named, logical steps. They are defined with the `WITH` keyword and referenced like tables.

```sql
WITH daily_revenue AS (
    SELECT
        date_key,
        SUM(total_amount) AS revenue,
        COUNT(DISTINCT customer_key) AS unique_customers
    FROM fact_sales
    GROUP BY date_key
),
revenue_with_avg AS (
    SELECT
        dr.*,
        AVG(revenue) OVER (ORDER BY date_key ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
            AS rolling_7d_avg
    FROM daily_revenue dr
)
SELECT
    d.full_date,
    r.revenue,
    r.unique_customers,
    ROUND(r.rolling_7d_avg, 2) AS rolling_7d_avg
FROM revenue_with_avg r
JOIN dim_date d ON r.date_key = d.date_key
WHERE d.calendar_year = 2025
ORDER BY d.full_date;
```

### Recursive CTEs

Useful for traversing hierarchical data (org charts, category trees).

```sql
WITH RECURSIVE category_tree AS (
    -- Base case: root categories
    SELECT id, name, parent_id, 1 AS depth, name AS path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- Recursive step
    SELECT c.id, c.name, c.parent_id, ct.depth + 1,
           ct.path || ' > ' || c.name
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY path;
```

---

## Window Functions

Window functions compute values across a set of rows related to the current row, without collapsing the result set. They are essential for rankings, running totals, and comparisons.

### Syntax

```sql
function_name(args) OVER (
    [PARTITION BY column_list]
    [ORDER BY column_list]
    [frame_clause]
)
```

### Ranking Functions

```sql
SELECT
    product_key,
    category,
    revenue,
    -- Consecutive rank (no gaps)
    DENSE_RANK() OVER (PARTITION BY category ORDER BY revenue DESC) AS dense_rnk,
    -- Rank with gaps for ties
    RANK() OVER (PARTITION BY category ORDER BY revenue DESC) AS rnk,
    -- Unique row number
    ROW_NUMBER() OVER (PARTITION BY category ORDER BY revenue DESC) AS rn
FROM product_revenue;
```

### Aggregate Windows

```sql
SELECT
    date_key,
    revenue,
    -- Running total
    SUM(revenue) OVER (ORDER BY date_key) AS cumulative_revenue,
    -- Moving average (7-day)
    AVG(revenue) OVER (
        ORDER BY date_key
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7d,
    -- Percentage of total
    ROUND(100.0 * revenue / SUM(revenue) OVER (), 2) AS pct_of_total
FROM daily_revenue;
```

### LAG and LEAD

Compare a row with previous or subsequent rows.

```sql
SELECT
    date_key,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY date_key) AS prev_day_revenue,
    LEAD(revenue, 1) OVER (ORDER BY date_key) AS next_day_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY date_key) AS day_over_day_change
FROM daily_revenue;
```

### FIRST_VALUE and LAST_VALUE

```sql
SELECT
    employee_id,
    department,
    salary,
    FIRST_VALUE(salary) OVER (
        PARTITION BY department ORDER BY salary DESC
    ) AS highest_salary_in_dept,
    salary - FIRST_VALUE(salary) OVER (
        PARTITION BY department ORDER BY salary DESC
    ) AS gap_from_top
FROM employees;
```

---

## MERGE / UPSERT

The `MERGE` statement (SQL:2003 standard) performs insert, update, or delete in a single atomic operation. It is critical for loading dimensions and implementing SCD patterns.

```sql
-- Standard MERGE (supported by Snowflake, BigQuery, SQL Server, Delta Lake)
MERGE INTO dim_product AS target
USING staging_product AS source
ON target.product_id = source.product_id
WHEN MATCHED AND (
    target.product_name <> source.product_name
    OR target.category <> source.category
) THEN UPDATE SET
    product_name = source.product_name,
    category = source.category,
    updated_at = CURRENT_TIMESTAMP
WHEN NOT MATCHED THEN INSERT (
    product_id, product_name, category, created_at, updated_at
) VALUES (
    source.product_id, source.product_name, source.category,
    CURRENT_TIMESTAMP, CURRENT_TIMESTAMP
);
```

### PostgreSQL: INSERT ... ON CONFLICT

PostgreSQL does not support `MERGE` before version 15. Use `INSERT ... ON CONFLICT` instead:

```sql
INSERT INTO dim_product (product_id, product_name, category, updated_at)
VALUES ('P-100', 'Widget Pro', 'Gadgets', NOW())
ON CONFLICT (product_id)
DO UPDATE SET
    product_name = EXCLUDED.product_name,
    category = EXCLUDED.category,
    updated_at = NOW();
```

---

## Pivoting and Unpivoting

### Pivoting (Rows to Columns)

Convert row values into columns — useful for creating summary reports.

```sql
-- Using CASE expressions (works everywhere)
SELECT
    calendar_year,
    SUM(CASE WHEN calendar_quarter = 1 THEN revenue END) AS q1,
    SUM(CASE WHEN calendar_quarter = 2 THEN revenue END) AS q2,
    SUM(CASE WHEN calendar_quarter = 3 THEN revenue END) AS q3,
    SUM(CASE WHEN calendar_quarter = 4 THEN revenue END) AS q4,
    SUM(revenue) AS full_year
FROM quarterly_revenue
GROUP BY calendar_year
ORDER BY calendar_year;
```

```sql
-- Using PIVOT (Snowflake, SQL Server, BigQuery)
SELECT *
FROM quarterly_revenue
PIVOT (SUM(revenue) FOR calendar_quarter IN (1 AS q1, 2 AS q2, 3 AS q3, 4 AS q4));
```

### Unpivoting (Columns to Rows)

```sql
-- Using UNPIVOT (Snowflake, SQL Server)
SELECT calendar_year, quarter, revenue
FROM yearly_summary
UNPIVOT (revenue FOR quarter IN (q1, q2, q3, q4));

-- Using UNION ALL (works everywhere)
SELECT calendar_year, 'Q1' AS quarter, q1 AS revenue FROM yearly_summary
UNION ALL
SELECT calendar_year, 'Q2', q2 FROM yearly_summary
UNION ALL
SELECT calendar_year, 'Q3', q3 FROM yearly_summary
UNION ALL
SELECT calendar_year, 'Q4', q4 FROM yearly_summary;
```

---

## Advanced Aggregations

### GROUPING SETS, ROLLUP, and CUBE

Generate multiple levels of aggregation in a single query.

```sql
-- GROUPING SETS: explicit combinations
SELECT
    COALESCE(country, '(All)') AS country,
    COALESCE(category, '(All)') AS category,
    SUM(revenue) AS total_revenue
FROM sales_summary
GROUP BY GROUPING SETS (
    (country, category),  -- country + category detail
    (country),            -- country subtotal
    (category),           -- category subtotal
    ()                    -- grand total
)
ORDER BY country, category;

-- ROLLUP: hierarchical aggregation
SELECT country, region, city, SUM(revenue)
FROM sales
GROUP BY ROLLUP (country, region, city);
-- Produces: (country, region, city), (country, region), (country), ()

-- CUBE: all possible combinations
SELECT country, category, SUM(revenue)
FROM sales
GROUP BY CUBE (country, category);
-- Produces: (country, category), (country), (category), ()
```

---

## Performance Tuning for Analytical Queries

### Reading Execution Plans

Every major database provides an `EXPLAIN` command that shows how a query will be executed.

```sql
-- PostgreSQL
EXPLAIN ANALYZE
SELECT d.calendar_year, SUM(f.total_amount)
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
GROUP BY d.calendar_year;
```

Key things to look for in execution plans:

| Indicator | Good Sign | Bad Sign |
|-----------|-----------|----------|
| Scan type | Index Scan, Bitmap Scan | Sequential Scan on large tables |
| Join type | Hash Join, Merge Join | Nested Loop on large tables |
| Row estimates | Close to actual | Orders of magnitude off |
| Sort | In-memory | Disk-based (external sort) |
| Partitions | Partition pruning active | All partitions scanned |

### Common Optimisation Techniques

**Partition pruning** — ensure queries filter on the partition key so the engine skips irrelevant partitions:

```sql
-- Good: partition key in WHERE clause
SELECT * FROM fact_sales
WHERE date_key BETWEEN 20250101 AND 20250131;

-- Bad: function on partition key prevents pruning
SELECT * FROM fact_sales
WHERE EXTRACT(YEAR FROM sale_date) = 2025;
```

**Predicate pushdown** — push filters as close to the data source as possible. Modern query engines do this automatically when the query is written correctly.

**Materialised views** — pre-compute expensive aggregations:

```sql
CREATE MATERIALIZED VIEW mv_monthly_revenue AS
SELECT
    date_trunc('month', sale_date) AS sale_month,
    product_category,
    SUM(total_amount) AS revenue,
    COUNT(*) AS num_sales
FROM fact_sales
JOIN dim_product USING (product_key)
GROUP BY 1, 2;

-- Refresh periodically
REFRESH MATERIALIZED VIEW mv_monthly_revenue;
```

**Columnar storage** — analytical queries typically read a few columns from many rows. Columnar formats (Parquet, ORC) and columnar databases (Redshift, BigQuery) dramatically reduce I/O.

---

## Key Takeaways

1. **CTEs** make complex queries readable and maintainable — use them liberally to break queries into logical steps
2. **Window functions** (RANK, LAG, SUM OVER, etc.) are essential for analytics — they compute across related rows without collapsing results
3. **MERGE / UPSERT** enables atomic insert-or-update operations critical for dimension loading and SCD patterns
4. **Pivoting** with CASE expressions works in every SQL dialect — use native PIVOT/UNPIVOT syntax where available for cleaner code
5. **GROUPING SETS, ROLLUP, and CUBE** generate multiple aggregation levels in a single pass, replacing multiple queries with UNION ALL
6. **Performance tuning** starts with execution plans — focus on partition pruning, predicate pushdown, and appropriate join strategies for large analytical tables
