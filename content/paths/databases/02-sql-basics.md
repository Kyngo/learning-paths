---
title: "SQL Basics"
weight: 2
---

## What Is SQL?

**SQL** (Structured Query Language) is the standard language for interacting with relational databases. Despite variations between vendors, core SQL is remarkably portable. It's declarative — you describe *what* you want, not *how* to get it.

### SQL Statement Categories

| Category | Acronym | Statements | Purpose |
|----------|---------|-----------|---------|
| Data Query Language | DQL | `SELECT` | Read data |
| Data Manipulation Language | DML | `INSERT`, `UPDATE`, `DELETE` | Modify data |
| Data Definition Language | DDL | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` | Define structure |
| Data Control Language | DCL | `GRANT`, `REVOKE` | Manage permissions |
| Transaction Control | TCL | `BEGIN`, `COMMIT`, `ROLLBACK` | Manage transactions |

---

## SELECT — Reading Data

### Basic Syntax

```sql
SELECT columns
FROM table
WHERE condition
ORDER BY column [ASC|DESC]
LIMIT count;
```

### Selecting Columns

```sql
-- All columns
SELECT * FROM employees;

-- Specific columns
SELECT name, salary, department FROM employees;

-- With aliases
SELECT name AS employee_name, salary AS annual_salary FROM employees;

-- Computed columns
SELECT name, salary, salary * 12 AS annual_total FROM employees;

-- Distinct values (remove duplicates)
SELECT DISTINCT department FROM employees;
```

### Filtering with WHERE

```sql
-- Comparison operators
SELECT * FROM employees WHERE salary > 80000;
SELECT * FROM employees WHERE department = 'Engineering';
SELECT * FROM employees WHERE hire_date >= '2021-01-01';

-- Multiple conditions
SELECT * FROM employees WHERE department = 'Engineering' AND salary > 90000;
SELECT * FROM employees WHERE department = 'Sales' OR department = 'Marketing';
SELECT * FROM employees WHERE NOT department = 'Sales';

-- IN (shorthand for multiple OR)
SELECT * FROM employees WHERE department IN ('Engineering', 'Marketing', 'Sales');

-- BETWEEN (inclusive range)
SELECT * FROM employees WHERE salary BETWEEN 70000 AND 100000;

-- LIKE (pattern matching)
SELECT * FROM employees WHERE name LIKE 'A%';       -- Starts with A
SELECT * FROM employees WHERE name LIKE '%son';     -- Ends with son
SELECT * FROM employees WHERE name LIKE '%ar%';     -- Contains ar
SELECT * FROM employees WHERE name LIKE '_ob%';     -- Second and third chars are 'ob'

-- NULL checks (never use = NULL)
SELECT * FROM employees WHERE manager_id IS NULL;
SELECT * FROM employees WHERE manager_id IS NOT NULL;
```

### Sorting with ORDER BY

```sql
-- Ascending (default)
SELECT * FROM employees ORDER BY salary;

-- Descending
SELECT * FROM employees ORDER BY salary DESC;

-- Multiple columns
SELECT * FROM employees ORDER BY department ASC, salary DESC;

-- By column position (less readable)
SELECT name, salary FROM employees ORDER BY 2 DESC;
```

### Limiting Results

```sql
-- First N rows
SELECT * FROM employees ORDER BY salary DESC LIMIT 10;

-- Pagination (skip 20, take 10)
SELECT * FROM employees ORDER BY id LIMIT 10 OFFSET 20;

-- SQL Server uses TOP
SELECT TOP 10 * FROM employees ORDER BY salary DESC;
```

---

## INSERT — Adding Data

```sql
-- Single row (specify columns)
INSERT INTO employees (name, department, salary, hire_date)
VALUES ('Eve Brown', 'Engineering', 92000, '2023-06-01');

-- Single row (all columns in order — fragile, avoid)
INSERT INTO employees
VALUES (5, 'Eve Brown', 'Engineering', 92000, '2023-06-01');

-- Multiple rows
INSERT INTO employees (name, department, salary, hire_date)
VALUES
    ('Frank Lee', 'Marketing', 75000, '2023-07-15'),
    ('Grace Kim', 'Sales', 71000, '2023-08-01'),
    ('Henry Park', 'Engineering', 98000, '2023-09-10');

-- Insert from SELECT
INSERT INTO archive_employees (name, department, salary)
SELECT name, department, salary
FROM employees
WHERE hire_date < '2020-01-01';
```

---

## UPDATE — Modifying Data

```sql
-- Update specific rows (always include WHERE!)
UPDATE employees
SET salary = 100000
WHERE id = 3;

-- Update multiple columns
UPDATE employees
SET salary = 95000, department = 'Engineering'
WHERE name = 'Bob Smith';

-- Computed update
UPDATE employees
SET salary = salary * 1.10
WHERE department = 'Engineering';

-- Update from another table
UPDATE employees e
SET department = d.new_department
FROM department_changes d
WHERE e.id = d.employee_id;
```

> ⚠️ **WARNING:** An `UPDATE` without `WHERE` modifies ALL rows in the table. Always double-check before executing.

---

## DELETE — Removing Data

```sql
-- Delete specific rows
DELETE FROM employees WHERE id = 4;

-- Delete matching a condition
DELETE FROM employees WHERE hire_date < '2019-01-01';

-- Delete all rows (keeps table structure, logs each row)
DELETE FROM employees;

-- TRUNCATE (faster, resets auto-increment, minimal logging)
TRUNCATE TABLE employees;
```

> ⚠️ **WARNING:** A `DELETE` without `WHERE` removes ALL rows. Use `TRUNCATE` for intentional full-table clearing.

---

## Data Types

### Common Types Across Databases

| Category | PostgreSQL | MySQL | Use For |
|----------|-----------|-------|---------|
| Integer | `INTEGER`, `BIGINT` | `INT`, `BIGINT` | IDs, counts, quantities |
| Decimal | `NUMERIC(10,2)`, `DECIMAL` | `DECIMAL(10,2)` | Money, precise calculations |
| Float | `REAL`, `DOUBLE PRECISION` | `FLOAT`, `DOUBLE` | Scientific data (imprecise!) |
| String (fixed) | `CHAR(10)` | `CHAR(10)` | Fixed-length codes |
| String (variable) | `VARCHAR(255)`, `TEXT` | `VARCHAR(255)`, `TEXT` | Names, descriptions |
| Boolean | `BOOLEAN` | `TINYINT(1)`, `BOOLEAN` | True/false flags |
| Date | `DATE` | `DATE` | Calendar dates |
| Timestamp | `TIMESTAMP`, `TIMESTAMPTZ` | `DATETIME`, `TIMESTAMP` | Points in time |
| UUID | `UUID` | `CHAR(36)` / `BINARY(16)` | Distributed IDs |
| JSON | `JSON`, `JSONB` | `JSON` | Semi-structured data |

### Choosing Types

- **Money:** Use `NUMERIC(10,2)` or `DECIMAL` — never `FLOAT` (rounding errors)
- **IDs:** `SERIAL` (PostgreSQL) / `AUTO_INCREMENT` (MySQL) for sequential, `UUID` for distributed
- **Strings:** `VARCHAR(n)` with a reasonable max; `TEXT` if unbounded
- **Timestamps:** Always store in UTC; use `TIMESTAMPTZ` (PostgreSQL) when timezone matters

---

## Operators and Expressions

### Arithmetic

```sql
SELECT price * quantity AS total FROM order_items;
SELECT salary / 12 AS monthly_salary FROM employees;
SELECT 17 % 5 AS remainder;  -- Modulo = 2
```

### String Functions

```sql
SELECT UPPER(name) FROM employees;
SELECT LOWER(email) FROM users;
SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM users;
SELECT LENGTH(name) FROM employees;
SELECT SUBSTRING(name, 1, 3) FROM employees;       -- First 3 chars
SELECT TRIM('  hello  ');                           -- Remove whitespace
SELECT REPLACE(phone, '-', '') FROM contacts;      -- Remove dashes
```

### Date Functions

```sql
-- Current date/time
SELECT NOW();                -- Timestamp
SELECT CURRENT_DATE;         -- Date only
SELECT CURRENT_TIMESTAMP;    -- Same as NOW()

-- Extract parts
SELECT EXTRACT(YEAR FROM hire_date) FROM employees;
SELECT EXTRACT(MONTH FROM hire_date) FROM employees;

-- Date arithmetic
SELECT hire_date + INTERVAL '1 year' FROM employees;       -- PostgreSQL
SELECT DATE_ADD(hire_date, INTERVAL 1 YEAR) FROM employees; -- MySQL

-- Difference
SELECT AGE(NOW(), hire_date) FROM employees;                -- PostgreSQL
SELECT DATEDIFF(NOW(), hire_date) FROM employees;           -- MySQL (days)
```

### Conditional Expressions

```sql
-- CASE (if/else in SQL)
SELECT name, salary,
    CASE
        WHEN salary >= 100000 THEN 'Senior'
        WHEN salary >= 70000 THEN 'Mid'
        ELSE 'Junior'
    END AS level
FROM employees;

-- COALESCE (first non-NULL value)
SELECT name, COALESCE(phone, email, 'No contact') AS contact
FROM employees;

-- NULLIF (returns NULL if values are equal)
SELECT NULLIF(discount, 0) FROM products;  -- Avoid division by zero
```

---

## Key Takeaways

1. **SQL is declarative** — describe what you want, the database figures out how
2. **Always use WHERE** with UPDATE and DELETE — forgetting it affects every row
3. **Prefer specific columns** over `SELECT *` — better performance, clearer intent
4. **NULL is not a value** — use `IS NULL` / `IS NOT NULL`, never `= NULL`
5. **Use parameterized queries** in application code — never concatenate user input into SQL (prevents SQL injection)
6. **Choose data types carefully** — `NUMERIC` for money, `TIMESTAMPTZ` for time, `VARCHAR` for variable strings
