---
title: "Aggregations and Grouping"
weight: 4
---

## Aggregate Functions

Aggregate functions compute a single value from a set of rows:

| Function | Purpose | Example |
|----------|---------|---------|
| `COUNT(*)` | Number of rows | Total employees |
| `COUNT(column)` | Non-NULL values in column | Employees with a phone |
| `COUNT(DISTINCT col)` | Unique non-NULL values | Number of departments |
| `SUM(column)` | Total | Total payroll |
| `AVG(column)` | Average | Average salary |
| `MIN(column)` | Smallest | Lowest salary |
| `MAX(column)` | Largest | Highest salary |

```sql
SELECT
    COUNT(*) AS total_employees,
    COUNT(DISTINCT department_id) AS departments,
    SUM(salary) AS total_payroll,
    AVG(salary) AS avg_salary,
    MIN(salary) AS min_salary,
    MAX(salary) AS max_salary
FROM employees;
```

---

## GROUP BY

Groups rows sharing a value, then applies aggregate functions per group:

```sql
-- Count and average salary per department
SELECT
    department_id,
    COUNT(*) AS headcount,
    AVG(salary) AS avg_salary,
    SUM(salary) AS total_cost
FROM employees
GROUP BY department_id;
```

### Rules

1. Every column in SELECT must be either aggregated OR in GROUP BY
2. GROUP BY executes after WHERE but before ORDER BY
3. You can group by multiple columns

```sql
-- Headcount by department AND level
SELECT department_id, level, COUNT(*) AS headcount
FROM employees
GROUP BY department_id, level
ORDER BY department_id, level;
```

### With Joins

```sql
SELECT
    d.name AS department,
    COUNT(e.id) AS employees,
    ROUND(AVG(e.salary), 0) AS avg_salary
FROM departments d
LEFT JOIN employees e ON d.id = e.department_id
GROUP BY d.name
ORDER BY employees DESC;
```

---

## HAVING

Filters **groups** (after aggregation). WHERE filters rows before grouping.

```sql
-- Departments with more than 5 employees
SELECT department_id, COUNT(*) AS headcount
FROM employees
GROUP BY department_id
HAVING COUNT(*) > 5;

-- Departments where average salary exceeds 90k
SELECT d.name, AVG(e.salary) AS avg_salary
FROM employees e
JOIN departments d ON e.department_id = d.id
GROUP BY d.name
HAVING AVG(e.salary) > 90000
ORDER BY avg_salary DESC;
```

### Execution Order

```
FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
```

| Clause | Filters | Stage |
|--------|---------|-------|
| `WHERE` | Individual rows (before grouping) | Before aggregation |
| `HAVING` | Groups (after aggregation) | After aggregation |

---

## Window Functions

Window functions perform calculations **across a set of rows related to the current row** — without collapsing the result into groups. Every row keeps its identity.

### Syntax

```sql
function_name() OVER (
    [PARTITION BY column]
    [ORDER BY column]
    [ROWS/RANGE frame_spec]
)
```

### ROW_NUMBER, RANK, DENSE_RANK

```sql
SELECT
    name,
    department_id,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num,
    RANK() OVER (ORDER BY salary DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank
FROM employees;
```

| Salary | ROW_NUMBER | RANK | DENSE_RANK |
|--------|-----------|------|------------|
| 120000 | 1 | 1 | 1 |
| 110000 | 2 | 2 | 2 |
| 110000 | 3 | 2 | 2 |
| 95000 | 4 | 4 | 3 |

- **ROW_NUMBER:** Always unique, arbitrary for ties
- **RANK:** Same rank for ties, gaps after (1,2,2,4)
- **DENSE_RANK:** Same rank for ties, no gaps (1,2,2,3)

### PARTITION BY

Apply the window function within groups:

```sql
-- Rank employees within their department
SELECT
    name,
    department_id,
    salary,
    RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dept_rank
FROM employees;
```

### Running Totals and Moving Averages

```sql
-- Running total of sales by date
SELECT
    order_date,
    amount,
    SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM orders;

-- 7-day moving average
SELECT
    order_date,
    amount,
    AVG(amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7d
FROM daily_sales;
```

### LAG and LEAD

Access previous/next row values:

```sql
-- Compare each month's revenue to the previous month
SELECT
    month,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY month) AS prev_month,
    revenue - LAG(revenue, 1) OVER (ORDER BY month) AS change,
    ROUND(
        (revenue - LAG(revenue, 1) OVER (ORDER BY month))::numeric /
        LAG(revenue, 1) OVER (ORDER BY month) * 100, 1
    ) AS pct_change
FROM monthly_revenue;
```

### FIRST_VALUE, LAST_VALUE, NTH_VALUE

```sql
-- Each employee compared to the highest earner in their department
SELECT
    name,
    department_id,
    salary,
    FIRST_VALUE(name) OVER (
        PARTITION BY department_id ORDER BY salary DESC
    ) AS top_earner,
    FIRST_VALUE(salary) OVER (
        PARTITION BY department_id ORDER BY salary DESC
    ) AS top_salary
FROM employees;
```

### Practical Patterns

```sql
-- Top 3 earners per department
SELECT * FROM (
    SELECT
        name,
        department_id,
        salary,
        ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rn
    FROM employees
) ranked
WHERE rn <= 3;

-- Percentage of total
SELECT
    name,
    department_id,
    salary,
    ROUND(salary::numeric / SUM(salary) OVER () * 100, 1) AS pct_of_total,
    ROUND(salary::numeric / SUM(salary) OVER (PARTITION BY department_id) * 100, 1) AS pct_of_dept
FROM employees;

-- Cumulative distribution
SELECT
    name,
    salary,
    CUME_DIST() OVER (ORDER BY salary) AS cumulative_pct,
    NTILE(4) OVER (ORDER BY salary) AS quartile
FROM employees;
```

---

## GROUPING SETS, ROLLUP, CUBE

For multi-level summaries in a single query:

```sql
-- ROLLUP: hierarchical subtotals + grand total
SELECT
    department_id,
    level,
    COUNT(*) AS headcount,
    SUM(salary) AS total_salary
FROM employees
GROUP BY ROLLUP (department_id, level);
-- Returns: per department+level, per department, grand total

-- CUBE: every combination of subtotals
SELECT
    department_id,
    level,
    COUNT(*) AS headcount
FROM employees
GROUP BY CUBE (department_id, level);
-- Returns: all combinations including individual dimensions and grand total
```

---

## Key Takeaways

1. **GROUP BY** collapses rows into groups — every selected column must be aggregated or grouped
2. **HAVING** filters groups (post-aggregation); **WHERE** filters rows (pre-aggregation)
3. **Window functions** are the most powerful analytical SQL feature — learn ROW_NUMBER, RANK, LAG, and running SUM
4. **PARTITION BY** is the window function equivalent of GROUP BY — it defines the scope without collapsing rows
5. **Use CTEs** to name your window queries for readability
6. **Frame specs** (`ROWS BETWEEN ...`) control exactly which rows the window sees — essential for moving averages
