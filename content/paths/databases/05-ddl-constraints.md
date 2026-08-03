---
title: "Data Definition and Constraints"
weight: 5
---

## Creating Tables

```sql
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,                          -- Auto-increment PK (PostgreSQL)
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    department_id INTEGER REFERENCES departments(id),
    salary NUMERIC(10, 2) DEFAULT 0.00,
    level VARCHAR(20) CHECK (level IN ('junior', 'mid', 'senior', 'lead')),
    is_active BOOLEAN DEFAULT TRUE,
    hire_date DATE NOT NULL DEFAULT CURRENT_DATE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- MySQL equivalent
CREATE TABLE employees (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    department_id INT,
    salary DECIMAL(10, 2) DEFAULT 0.00,
    level ENUM('junior', 'mid', 'senior', 'lead'),
    is_active BOOLEAN DEFAULT TRUE,
    hire_date DATE NOT NULL DEFAULT (CURRENT_DATE),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (department_id) REFERENCES departments(id)
);
```

---

## Constraints

Constraints enforce data integrity rules at the database level — data that violates them is rejected.

| Constraint | Purpose | Example |
|-----------|---------|---------|
| `PRIMARY KEY` | Unique + NOT NULL identifier | `id SERIAL PRIMARY KEY` |
| `FOREIGN KEY` | References PK in another table | `REFERENCES departments(id)` |
| `NOT NULL` | Disallow NULL values | `name VARCHAR(100) NOT NULL` |
| `UNIQUE` | No duplicate values | `email VARCHAR(255) UNIQUE` |
| `CHECK` | Custom validation rule | `CHECK (salary >= 0)` |
| `DEFAULT` | Value when none provided | `DEFAULT CURRENT_DATE` |
| `EXCLUDE` | Prevent overlapping ranges (PostgreSQL) | Scheduling, reservations |

### Named Constraints

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    amount NUMERIC(10, 2) NOT NULL,
    status VARCHAR(20) NOT NULL,
    
    CONSTRAINT fk_customer 
        FOREIGN KEY (customer_id) REFERENCES customers(id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE,
    
    CONSTRAINT chk_amount_positive 
        CHECK (amount > 0),
    
    CONSTRAINT chk_valid_status 
        CHECK (status IN ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled'))
);
```

### Foreign Key Actions

| Action | ON DELETE | ON UPDATE | Behavior |
|--------|-----------|-----------|----------|
| `RESTRICT` | Block delete if children exist | Block update | Safest default |
| `CASCADE` | Delete children too | Update children's FK | Dangerous but convenient |
| `SET NULL` | Set FK to NULL | Set FK to NULL | Requires nullable FK |
| `SET DEFAULT` | Set FK to default value | Set FK to default | Rarely used |
| `NO ACTION` | Same as RESTRICT (checked at end of transaction) | Same | Default in some DBs |

```sql
-- If a department is deleted, set employees' department_id to NULL
FOREIGN KEY (department_id) REFERENCES departments(id)
    ON DELETE SET NULL
    ON UPDATE CASCADE
```

---

## Altering Tables

```sql
-- Add column
ALTER TABLE employees ADD COLUMN phone VARCHAR(20);
ALTER TABLE employees ADD COLUMN bonus NUMERIC(10,2) DEFAULT 0;

-- Remove column
ALTER TABLE employees DROP COLUMN phone;

-- Rename column
ALTER TABLE employees RENAME COLUMN name TO full_name;

-- Change data type
ALTER TABLE employees ALTER COLUMN salary TYPE NUMERIC(12, 2);

-- Add/drop constraint
ALTER TABLE employees ADD CONSTRAINT chk_salary CHECK (salary >= 0);
ALTER TABLE employees DROP CONSTRAINT chk_salary;

-- Add NOT NULL
ALTER TABLE employees ALTER COLUMN email SET NOT NULL;

-- Drop NOT NULL
ALTER TABLE employees ALTER COLUMN phone DROP NOT NULL;

-- Add unique constraint
ALTER TABLE employees ADD CONSTRAINT uq_email UNIQUE (email);

-- Add index
CREATE INDEX idx_employees_department ON employees(department_id);
CREATE UNIQUE INDEX idx_employees_email ON employees(email);

-- Rename table
ALTER TABLE employees RENAME TO team_members;
```

---

## Dropping and Truncating

```sql
-- Drop table (permanently removes table + data)
DROP TABLE employees;
DROP TABLE IF EXISTS employees;              -- No error if doesn't exist
DROP TABLE employees CASCADE;                -- Also drop dependent objects (dangerous!)

-- Truncate (delete all data, keep structure)
TRUNCATE TABLE employees;
TRUNCATE TABLE employees RESTART IDENTITY;   -- Reset auto-increment
TRUNCATE TABLE orders CASCADE;               -- Also truncate referencing tables
```

---

## Views

A view is a saved query that acts like a virtual table:

```sql
-- Create view
CREATE VIEW employee_directory AS
SELECT
    e.id,
    e.name,
    e.email,
    d.name AS department,
    e.hire_date
FROM employees e
JOIN departments d ON e.department_id = d.id
WHERE e.is_active = TRUE;

-- Use like a table
SELECT * FROM employee_directory WHERE department = 'Engineering';

-- Drop
DROP VIEW employee_directory;
```

### Materialized Views (PostgreSQL)

Stores the result physically — faster reads, must be refreshed:

```sql
CREATE MATERIALIZED VIEW monthly_stats AS
SELECT
    DATE_TRUNC('month', order_date) AS month,
    COUNT(*) AS order_count,
    SUM(amount) AS revenue
FROM orders
GROUP BY DATE_TRUNC('month', order_date);

-- Refresh when data changes
REFRESH MATERIALIZED VIEW monthly_stats;
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_stats;  -- No lock
```

---

## Sequences and Identity

```sql
-- PostgreSQL: SERIAL (legacy) or IDENTITY (modern)
CREATE TABLE products (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(200) NOT NULL
);

-- Manual sequence control
CREATE SEQUENCE order_number_seq START WITH 1000 INCREMENT BY 1;
SELECT nextval('order_number_seq');

-- UUID as primary key
CREATE TABLE events (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    event_type VARCHAR(50) NOT NULL,
    payload JSONB
);
```

---

## Temporary Tables

Exist only for the duration of a session or transaction:

```sql
-- Session-scoped (dropped when connection closes)
CREATE TEMPORARY TABLE temp_results (
    id INTEGER,
    score NUMERIC
);

-- Use for complex multi-step queries
CREATE TEMP TABLE active_users AS
SELECT id, name FROM users WHERE last_login > NOW() - INTERVAL '30 days';

SELECT * FROM active_users;
-- Automatically gone when session ends
```

---

## Key Takeaways

1. **Constraints are your safety net** — enforce rules in the database, not just application code
2. **Name your constraints** — `CONSTRAINT chk_salary_positive` is debuggable; anonymous constraints aren't
3. **Use ON DELETE RESTRICT** as the default FK action — CASCADE deletes can wipe out more data than you expect
4. **Views simplify access patterns** — create views for common joins so application code stays clean
5. **Materialized views** trade freshness for speed — perfect for dashboards and reports
6. **Always use IF EXISTS/IF NOT EXISTS** in DDL scripts — makes them safely re-runnable
7. **Migrations should be reversible** — write ALTER TABLE with the matching undo step
