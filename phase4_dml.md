<div align="center">

# <span style="color:#0A2FA8">PostgreSQL : Phase 4</span>

<sub>DML - Data Manipulation Language · Complete Notes · INSERT, SELECT, UPDATE, DELETE, UPSERT, COPY</sub>

</div>

---

## <span style="color:#1565C0">Overview - DML vs DDL</span>

| Category | Stands For | What it does | Commands |
|:---|:---|:---|:---|
| **DDL** | Data Definition Language | Defines and changes structure | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` |
| **DML** | Data Manipulation Language | Works with the actual data inside tables | `INSERT`, `SELECT`, `UPDATE`, `DELETE` |

> **Definition:** DML (Data Manipulation Language) is the set of SQL commands used to read, insert, modify, and remove data from existing tables. DDL changes the shape of tables; DML works with what's inside them.

---

## <span style="color:#1565C0">4.1 INSERT</span>

> **Definition:** `INSERT` adds one or more new rows of data into a table.

---

### <span style="color:#2E86AB">Basic Syntax</span>

```sql
INSERT INTO table_name (column1, column2, ...)
VALUES (value1, value2, ...);
```

---

### <span style="color:#2E86AB">Single Row Insert</span>

```sql
-- Setup: reference table used throughout this phase
CREATE TABLE employees (
  id          SERIAL PRIMARY KEY,
  first_name  TEXT NOT NULL,
  last_name   TEXT NOT NULL,
  email       TEXT UNIQUE NOT NULL,
  department  TEXT,
  salary      NUMERIC(10, 2),
  is_active   BOOLEAN DEFAULT TRUE,
  hired_on    DATE DEFAULT CURRENT_DATE,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Insert specifying all non-default columns
INSERT INTO employees (first_name, last_name, email, department, salary)
VALUES ('Alice', 'Kumar', 'alice@company.com', 'Engineering', 85000.00);

-- Insert ALL columns (must match exact order of table definition)
-- Not recommended - fragile if table structure changes
INSERT INTO employees
VALUES (DEFAULT, 'Bob', 'Singh', 'bob@company.com', 'Marketing', 70000.00, TRUE, '2023-03-01', NOW());

-- Insert with explicit NULLs
INSERT INTO employees (first_name, last_name, email, department, salary)
VALUES ('Carol', 'Sharma', 'carol@company.com', NULL, NULL);
-- department and salary will be NULL
-- is_active gets DEFAULT = TRUE
-- hired_on gets DEFAULT = CURRENT_DATE
-- created_at gets DEFAULT = NOW()
```

---

### <span style="color:#2E86AB">Multi-Row Insert</span>

> **Definition:** Multiple rows can be inserted in a single `INSERT` statement by providing a comma-separated list of `VALUES` tuples. This is significantly faster than running individual `INSERT` statements in a loop.

```sql
-- Insert multiple rows in one statement
INSERT INTO employees (first_name, last_name, email, department, salary)
VALUES
  ('Dave',  'Patel',   'dave@company.com',  'Engineering', 90000.00),
  ('Eva',   'Mehta',   'eva@company.com',   'HR',          65000.00),
  ('Frank', 'Joshi',   'frank@company.com', 'Finance',     75000.00),
  ('Grace', 'Nair',    'grace@company.com', 'Engineering', 92000.00),
  ('Hank',  'Reddy',   'hank@company.com',  'Marketing',   68000.00);

-- All 5 rows inserted in a single round-trip to the database
-- vs. 5 separate INSERTs which would be much slower
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 580px;">

### <span style="color:#D4A017">Performance: Multi-row INSERT is always preferred over looping individual INSERTs. One statement = one transaction, one parse, one plan, one network round-trip.</span>

</div>

---

### <span style="color:#2E86AB">INSERT with DEFAULT Values</span>

```sql
-- DEFAULT keyword - explicitly use column's default value
INSERT INTO employees (first_name, last_name, email, is_active, hired_on)
VALUES ('Ivan', 'Das', 'ivan@company.com', DEFAULT, DEFAULT);
-- is_active = TRUE (default)
-- hired_on  = CURRENT_DATE (default)

-- INSERT DEFAULT VALUES - all columns get their defaults
-- (only works when every column has a DEFAULT or is nullable)
CREATE TABLE audit_log (
  id         SERIAL PRIMARY KEY,
  logged_at  TIMESTAMPTZ DEFAULT NOW(),
  server     TEXT DEFAULT inet_server_addr()::TEXT
);

INSERT INTO audit_log DEFAULT VALUES;
-- id auto-incremented, logged_at = NOW(), server = server address

-- Omitting a column is the same as DEFAULT
INSERT INTO employees (first_name, last_name, email)
VALUES ('Julia', 'Bose', 'julia@company.com');
-- is_active, hired_on, created_at all use their defaults
-- department, salary are NULL (no default defined)
```

---

### <span style="color:#2E86AB">INSERT INTO ... SELECT</span>

> **Definition:** `INSERT INTO ... SELECT` inserts rows that are the result of a `SELECT` query. No `VALUES` clause is used. This is the most powerful form of INSERT - used for copying, transforming, and migrating data.

```sql
-- Basic: copy all rows from one table to another
CREATE TABLE employees_backup (LIKE employees INCLUDING ALL);

INSERT INTO employees_backup
SELECT * FROM employees;

-- Copy with filtering
INSERT INTO employees_backup
SELECT * FROM employees WHERE department = 'Engineering';

-- Copy with transformation
CREATE TABLE employee_summary (
  full_name   TEXT,
  department  TEXT,
  salary      NUMERIC(10,2),
  copied_at   TIMESTAMPTZ
);

INSERT INTO employee_summary (full_name, department, salary, copied_at)
SELECT
  first_name || ' ' || last_name,  -- concatenate
  department,
  salary,
  NOW()                             -- add copy timestamp
FROM employees
WHERE is_active = TRUE;

-- Cross-table aggregation insert
CREATE TABLE dept_stats (
  department    TEXT,
  headcount     INTEGER,
  avg_salary    NUMERIC(10,2),
  total_payroll NUMERIC(12,2)
);

INSERT INTO dept_stats (department, headcount, avg_salary, total_payroll)
SELECT
  department,
  COUNT(*),
  ROUND(AVG(salary), 2),
  SUM(salary)
FROM employees
WHERE is_active = TRUE
GROUP BY department;
```

---

### <span style="color:#2E86AB">INSERT and Constraint Violations</span>

```sql
-- What happens when a constraint is violated

-- NOT NULL violation
INSERT INTO employees (first_name, last_name, email)
VALUES (NULL, 'Doe', 'doe@mail.com');
-- ERROR: null value in column "first_name" violates not-null constraint

-- UNIQUE violation
INSERT INTO employees (first_name, last_name, email)
VALUES ('Test', 'User', 'alice@company.com');
-- ERROR: duplicate key value violates unique constraint "employees_email_key"
-- DETAIL: Key (email)=(alice@company.com) already exists.

-- CHECK violation
CREATE TABLE products (
  id    SERIAL PRIMARY KEY,
  price NUMERIC(10,2) NOT NULL CHECK (price > 0)
);

INSERT INTO products (price) VALUES (-5);
-- ERROR: new row for relation "products" violates check constraint "products_price_check"

-- FOREIGN KEY violation
CREATE TABLE orders (
  id          SERIAL PRIMARY KEY,
  employee_id INTEGER REFERENCES employees(id)
);

INSERT INTO orders (employee_id) VALUES (9999);
-- ERROR: insert or update on table "orders" violates foreign key constraint
-- DETAIL: Key (employee_id)=(9999) is not present in table "employees".
```

---

## <span style="color:#1565C0">4.2 SELECT - Basic Syntax</span>

> **Definition:** `SELECT` retrieves data from one or more tables. It is the most frequently used SQL command. A full SELECT query moves through several logical clauses, each narrowing or shaping the result.

> **Note:** Core SELECT (filtering, sorting, grouping, joins, window functions, CTEs) is covered in full depth in Phases 5 through 10. This section covers the foundational syntax as part of DML.

---

### <span style="color:#2E86AB">Basic SELECT Syntax</span>

```sql
SELECT column1, column2, ...
FROM   table_name
WHERE  condition
ORDER BY column ASC|DESC
LIMIT  n;
```

---

### <span style="color:#2E86AB">Column Selection</span>

```sql
-- Select all columns (wildcard *)
SELECT * FROM employees;

-- Select specific columns
SELECT first_name, last_name, email FROM employees;

-- Column aliases with AS
SELECT
  first_name AS "First Name",
  last_name  AS "Last Name",
  salary     AS annual_salary
FROM employees;

-- Expressions in SELECT
SELECT
  first_name || ' ' || last_name AS full_name,
  salary / 12                    AS monthly_salary,
  salary * 1.10                  AS salary_with_raise,
  UPPER(email)                   AS email_upper
FROM employees;

-- Literal values in SELECT
SELECT
  id,
  first_name,
  'Employee'   AS record_type,   -- string literal
  0            AS bonus,         -- numeric literal
  TRUE         AS is_current     -- boolean literal
FROM employees;
```

---

### <span style="color:#2E86AB">SELECT Without FROM</span>

```sql
-- PostgreSQL allows SELECT without a FROM clause
-- Useful for testing expressions, functions, type casts

SELECT 1 + 1;                     -- 2
SELECT NOW();                     -- current timestamp
SELECT UPPER('hello world');      -- HELLO WORLD
SELECT '2024-01-01'::DATE + 30;   -- 2024-01-31
SELECT gen_random_uuid();         -- a fresh UUID
SELECT version();                 -- PostgreSQL version info
SELECT current_user;              -- current DB user
SELECT pg_size_pretty(1024*1024); -- '1024 kB'
```

---

### <span style="color:#2E86AB">SELECT Clause Execution Order</span>

While SQL is written in this order:

```
SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY → LIMIT
```

PostgreSQL actually **executes** clauses in a different logical order:

```
FROM        (1) - identify the source tables
WHERE       (2) - filter rows
GROUP BY    (3) - group rows
HAVING      (4) - filter groups
SELECT      (5) - compute output columns
ORDER BY    (6) - sort results
LIMIT       (7) - restrict row count
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 580px;">

### <span style="color:#D4A017">Why this matters: You cannot use a SELECT alias in a WHERE clause (WHERE runs before SELECT). But you CAN use SELECT aliases in ORDER BY (ORDER BY runs after SELECT).</span>

</div>

```sql
-- This FAILS - WHERE runs before SELECT, so alias doesn't exist yet
SELECT salary * 12 AS annual FROM employees WHERE annual > 100000;
-- ERROR: column "annual" does not exist

-- This WORKS - repeat the expression in WHERE
SELECT salary * 12 AS annual FROM employees WHERE salary * 12 > 100000;

-- This WORKS - ORDER BY runs after SELECT, alias is available
SELECT salary * 12 AS annual FROM employees ORDER BY annual DESC;
```

---

## <span style="color:#1565C0">4.3 UPDATE</span>

> **Definition:** `UPDATE` modifies existing rows in a table. It changes column values for rows that match a condition. Without a `WHERE` clause, it updates every row in the table.

---

### <span style="color:#2E86AB">Basic Syntax</span>

```sql
UPDATE table_name
SET column1 = value1,
    column2 = value2,
    ...
WHERE condition;
```

---

### <span style="color:#2E86AB">Updating a Single Column</span>

```sql
-- Update one row by primary key
UPDATE employees
SET salary = 90000.00
WHERE id = 1;

-- Update using a condition other than PK
UPDATE employees
SET department = 'Product'
WHERE email = 'alice@company.com';

-- Update all rows in a department
UPDATE employees
SET is_active = FALSE
WHERE department = 'Marketing';
```

---

### <span style="color:#2E86AB">Updating Multiple Columns</span>

```sql
-- Update several columns at once
UPDATE employees
SET
  salary      = 95000.00,
  department  = 'Senior Engineering',
  is_active   = TRUE
WHERE id = 3;

-- Update using expressions
UPDATE employees
SET
  salary    = salary * 1.10,          -- 10% raise
  hired_on  = hired_on + INTERVAL '0' -- no change (just showing expressions work)
WHERE department = 'Engineering';

-- Update using CASE expression
UPDATE employees
SET salary = CASE
  WHEN department = 'Engineering' THEN salary * 1.12  -- 12% raise
  WHEN department = 'HR'          THEN salary * 1.08  -- 8% raise
  WHEN department = 'Finance'     THEN salary * 1.10  -- 10% raise
  ELSE salary * 1.05                                  -- 5% for all others
END;

-- Update with subquery
UPDATE employees
SET salary = (
  SELECT AVG(salary) * 1.05
  FROM employees
  WHERE department = 'Engineering'
)
WHERE department = 'Engineering' AND salary < 80000;
```

---

### <span style="color:#2E86AB">UPDATE with WHERE</span>

```sql
-- Multiple conditions
UPDATE employees
SET is_active = FALSE
WHERE department = 'Marketing'
  AND salary < 60000
  AND hired_on < '2020-01-01';

-- Using IN
UPDATE employees
SET department = 'Tech'
WHERE department IN ('Engineering', 'Product', 'DevOps');

-- Using LIKE
UPDATE employees
SET email = LOWER(email)
WHERE email <> LOWER(email);   -- normalise any uppercase emails

-- Null-check in WHERE
UPDATE employees
SET department = 'Unassigned'
WHERE department IS NULL;

-- UPDATE without WHERE - updates ALL rows (be very careful!)
UPDATE employees SET is_active = TRUE;   -- activates every employee
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 580px;">

### <span style="color:#D4A017">Always verify your WHERE clause first with a SELECT before running UPDATE. Run: SELECT * FROM employees WHERE &lt;your condition&gt; to confirm which rows will be affected.</span>

</div>

---

### <span style="color:#2E86AB">UPDATE with FROM - PostgreSQL Join Update</span>

> **Definition:** PostgreSQL's `UPDATE ... FROM` extension allows you to join another table (or subquery) inside an `UPDATE` statement. This lets you update column values in one table based on data from another table.

```sql
-- Syntax
UPDATE target_table
SET    column = value
FROM   other_table
WHERE  target_table.join_col = other_table.join_col
  AND  other_condition;

-- Example 1: Update employee salaries based on a raise table
CREATE TABLE salary_adjustments (
  department  TEXT,
  raise_pct   NUMERIC(5,2)
);

INSERT INTO salary_adjustments VALUES
  ('Engineering', 12.00),
  ('HR',           8.00),
  ('Finance',     10.00),
  ('Marketing',    5.00);

-- Apply raises from adjustment table to employees
UPDATE employees e
SET salary = ROUND(e.salary * (1 + sa.raise_pct / 100), 2)
FROM salary_adjustments sa
WHERE e.department = sa.department;

-- Example 2: Update orders with customer info from another table
CREATE TABLE orders (
  id            SERIAL PRIMARY KEY,
  employee_id   INTEGER REFERENCES employees(id),
  amount        NUMERIC(10,2),
  region        TEXT
);

CREATE TABLE employee_regions (
  employee_id  INTEGER,
  region       TEXT
);

-- Set the region on each order from the employee_regions table
UPDATE orders o
SET region = er.region
FROM employee_regions er
WHERE o.employee_id = er.employee_id
  AND o.region IS NULL;

-- Example 3: UPDATE with a subquery in FROM
UPDATE employees e
SET salary = avg_table.avg_dept_salary
FROM (
  SELECT department, AVG(salary) AS avg_dept_salary
  FROM employees
  GROUP BY department
) avg_table
WHERE e.department = avg_table.department
  AND e.salary < avg_table.avg_dept_salary * 0.80;
-- Bumps up underpaid employees (below 80% of dept avg) to the dept average
```

---

### <span style="color:#2E86AB">How Many Rows Were Updated?</span>

```sql
-- PostgreSQL always reports the number of affected rows
UPDATE employees SET salary = salary * 1.05 WHERE department = 'HR';
-- UPDATE 2  ← means 2 rows were updated

-- Check in application code (psycopg2 / node-postgres)
-- cursor.rowcount  →  number of rows updated
```

---

## <span style="color:#1565C0">4.4 DELETE</span>

> **Definition:** `DELETE` removes rows from a table that match a condition. Without a `WHERE` clause it removes all rows (though `TRUNCATE` is preferred for that case). Unlike `TRUNCATE`, `DELETE` fires row-level triggers and is slower on large tables.

---

### <span style="color:#2E86AB">Basic Syntax</span>

```sql
DELETE FROM table_name
WHERE condition;
```

---

### <span style="color:#2E86AB">DELETE with WHERE</span>

```sql
-- Delete a specific row by primary key
DELETE FROM employees WHERE id = 5;

-- Delete by condition
DELETE FROM employees WHERE is_active = FALSE;

-- Delete by date range
DELETE FROM employees WHERE hired_on < '2015-01-01';

-- Delete with multiple conditions
DELETE FROM employees
WHERE department = 'Marketing'
  AND salary < 50000
  AND is_active = FALSE;

-- Delete using IN
DELETE FROM employees WHERE department IN ('Marketing', 'Sales');

-- Delete using LIKE
DELETE FROM employees WHERE email LIKE '%@oldcompany.com';

-- Delete using subquery
DELETE FROM orders
WHERE employee_id IN (
  SELECT id FROM employees WHERE is_active = FALSE
);

-- Delete ALL rows (but keep table structure - prefer TRUNCATE for large tables)
DELETE FROM employees;
-- DELETE n  ← slow row-by-row; use TRUNCATE for bulk removal
```

---

### <span style="color:#2E86AB">DELETE with USING - PostgreSQL Join Delete</span>

> **Definition:** PostgreSQL's `DELETE ... USING` extension allows joining another table inside a `DELETE` statement, so you can delete rows based on conditions involving another table's data.

```sql
-- Syntax
DELETE FROM target_table
USING  other_table
WHERE  target_table.col = other_table.col
  AND  other_condition;

-- Example 1: Delete orders belonging to inactive employees
DELETE FROM orders o
USING employees e
WHERE o.employee_id = e.id
  AND e.is_active = FALSE;

-- Example 2: Delete rows using a staging table
CREATE TEMP TABLE ids_to_delete (employee_id INTEGER);
INSERT INTO ids_to_delete VALUES (3), (7), (12), (15);

DELETE FROM employees e
USING ids_to_delete d
WHERE e.id = d.employee_id;

-- Example 3: Delete with multiple joined tables
DELETE FROM order_items oi
USING orders o, employees e
WHERE oi.order_id = o.id
  AND o.employee_id = e.id
  AND e.department = 'Marketing'
  AND o.created_at < '2022-01-01';

-- Equivalent using a subquery (standard SQL, works across databases)
DELETE FROM order_items
WHERE order_id IN (
  SELECT o.id
  FROM orders o
  JOIN employees e ON o.employee_id = e.id
  WHERE e.department = 'Marketing'
    AND o.created_at < '2022-01-01'
);
```

---

### <span style="color:#2E86AB">DELETE and Foreign Keys</span>

```sql
-- If a FK exists without CASCADE, delete on parent fails
DELETE FROM employees WHERE id = 1;
-- ERROR: update or delete on table "employees" violates foreign key constraint
-- DETAIL: Key (id)=(1) is still referenced from table "orders"

-- Solutions:
-- Option 1: Delete children first, then parent
DELETE FROM orders WHERE employee_id = 1;
DELETE FROM employees WHERE id = 1;

-- Option 2: Use FK with ON DELETE CASCADE (set up at table creation)
-- Then parent delete automatically cascades

-- Option 3: Temporarily disable FK check (not recommended in production)
SET session_replication_role = 'replica';
DELETE FROM employees WHERE id = 1;
SET session_replication_role = 'origin';
```

---

## <span style="color:#1565C0">4.5 RETURNING Clause</span>

> **Definition:** The `RETURNING` clause is a PostgreSQL extension to `INSERT`, `UPDATE`, and `DELETE`. It returns column values of the rows that were affected by the statement - without requiring a separate `SELECT` query afterwards.

---

### <span style="color:#2E86AB">RETURNING with INSERT</span>

```sql
-- Get the auto-generated id after insert
INSERT INTO employees (first_name, last_name, email, department, salary)
VALUES ('Kim', 'Iyer', 'kim@company.com', 'Engineering', 88000)
RETURNING id;
-- Returns: id = 9 (whatever was auto-generated)

-- Return multiple columns
INSERT INTO employees (first_name, last_name, email, salary)
VALUES ('Leo', 'Ghosh', 'leo@company.com', 72000)
RETURNING id, first_name, hired_on, created_at;

-- Return everything with *
INSERT INTO employees (first_name, last_name, email)
VALUES ('Mia', 'Roy', 'mia@company.com')
RETURNING *;
-- Returns the full row as inserted (including all defaults)

-- Multi-row insert with RETURNING
INSERT INTO employees (first_name, last_name, email, department)
VALUES
  ('Nina', 'Sen',  'nina@company.com',  'HR'),
  ('Omar', 'Khan', 'omar@company.com',  'Finance')
RETURNING id, first_name, department;
-- Returns 2 rows - one for each inserted employee

-- Use RETURNING in application code to get generated ID
-- Without RETURNING: INSERT then SELECT currval('employees_id_seq')
-- With RETURNING:    INSERT ... RETURNING id  ← single round-trip, safer
```

---

### <span style="color:#2E86AB">RETURNING with UPDATE</span>

```sql
-- See the updated values after UPDATE
UPDATE employees
SET salary = salary * 1.10
WHERE department = 'Engineering'
RETURNING id, first_name, salary;
-- Returns the NEW (post-update) salary for each affected row

-- Return old and new values using RETURNING with a subquery
-- (For old values, you need a CTE - covered in Phase 8)

-- Track which rows had salary updated
UPDATE employees
SET salary = 95000
WHERE id = 1
RETURNING id, first_name, salary AS new_salary;

-- Update and immediately log the change
WITH updated AS (
  UPDATE employees
  SET salary = salary * 1.05
  WHERE department = 'Finance'
  RETURNING id, first_name, salary
)
INSERT INTO audit_log (employee_id, employee_name, new_salary, changed_at)
SELECT id, first_name, salary, NOW()
FROM updated;
```

---

### <span style="color:#2E86AB">RETURNING with DELETE</span>

```sql
-- See which rows were deleted
DELETE FROM employees
WHERE is_active = FALSE
RETURNING id, first_name, last_name, department;
-- Returns the full details of every deleted row (for logging)

-- Archive before delete
WITH deleted AS (
  DELETE FROM employees
  WHERE hired_on < '2010-01-01'
  RETURNING *
)
INSERT INTO employees_archive
SELECT * FROM deleted;
-- Atomically moves old employees to archive - no separate SELECT needed

-- Delete and count
WITH deleted AS (
  DELETE FROM sessions WHERE expires_at < NOW()
  RETURNING id
)
SELECT COUNT(*) AS expired_sessions_removed FROM deleted;
```

---

### <span style="color:#2E86AB">Why RETURNING Matters</span>

| Without RETURNING | With RETURNING |
|:---|:---|
| INSERT → separate SELECT to get generated ID | INSERT RETURNING id → one round-trip |
| UPDATE → separate SELECT to verify new values | UPDATE RETURNING new values → immediate |
| DELETE → unknown which rows were removed | DELETE RETURNING → see exactly what was removed |
| Archive pattern requires SELECT then DELETE | CTE + DELETE RETURNING → atomic, safer |

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 580px;">

### <span style="color:#D4A017">RETURNING is PostgreSQL-specific (not standard SQL). It reduces round-trips, enables atomic read-modify-write patterns, and is essential for getting auto-generated values after INSERT.</span>

</div>

---

## <span style="color:#1565C0">4.6 UPSERT - INSERT ... ON CONFLICT</span>

> **Definition:** An UPSERT (Update + Insert) either inserts a new row or updates an existing one if a conflict (duplicate key violation) occurs. PostgreSQL implements UPSERT via the `INSERT ... ON CONFLICT` syntax.

---

### <span style="color:#2E86AB">The Problem UPSERT Solves</span>

```sql
-- Without UPSERT - you'd need to check existence first
-- This is a race condition (not atomic):
IF NOT EXISTS (SELECT 1 FROM users WHERE email = 'a@b.com') THEN
  INSERT INTO users (email, name) VALUES ('a@b.com', 'Alice');
ELSE
  UPDATE users SET name = 'Alice' WHERE email = 'a@b.com';
END IF;

-- With UPSERT - atomic, single statement, no race condition
INSERT INTO users (email, name) VALUES ('a@b.com', 'Alice')
ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name;
```

---

### <span style="color:#2E86AB">ON CONFLICT DO NOTHING</span>

> **Definition:** `ON CONFLICT DO NOTHING` silently ignores the INSERT if it would cause a conflict. The row is not inserted, no error is raised, and the existing row is left unchanged.

```sql
-- Setup
CREATE TABLE tags (
  id    SERIAL PRIMARY KEY,
  name  TEXT UNIQUE NOT NULL
);

INSERT INTO tags (name) VALUES ('postgresql'), ('database'), ('sql');

-- Try to insert duplicates - no error, just skipped
INSERT INTO tags (name) VALUES ('postgresql')
ON CONFLICT DO NOTHING;
-- INSERT 0 0  ← 0 rows inserted (conflict was ignored)

-- Specify the conflict target explicitly (clearer and safer)
INSERT INTO tags (name) VALUES ('postgresql')
ON CONFLICT (name) DO NOTHING;

-- Multi-row insert - only non-conflicting rows get inserted
INSERT INTO tags (name)
VALUES ('postgresql'), ('mongodb'), ('redis'), ('database')
ON CONFLICT (name) DO NOTHING;
-- 'postgresql' and 'database' already exist → skipped
-- 'mongodb' and 'redis' are new → inserted
-- INSERT 0 2  ← 2 rows inserted, 2 skipped

-- Conflict on constraint name (instead of column name)
INSERT INTO tags (name) VALUES ('mysql')
ON CONFLICT ON CONSTRAINT tags_name_key DO NOTHING;
```

---

### <span style="color:#2E86AB">ON CONFLICT DO UPDATE SET - True Upsert</span>

> **Definition:** `ON CONFLICT DO UPDATE SET` updates the existing row when a conflict occurs. The special `EXCLUDED` table refers to the row that was attempted to be inserted (the new values).

```sql
-- Syntax
INSERT INTO table (col1, col2, ...)
VALUES (val1, val2, ...)
ON CONFLICT (conflict_column)
DO UPDATE SET
  col1 = EXCLUDED.col1,
  col2 = EXCLUDED.col2;

-- EXCLUDED = the row that was blocked (the "would-be" insert)
```

---

#### <span style="color:#5B8DB8">Example 1 - Simple Upsert</span>

```sql
CREATE TABLE user_settings (
  user_id   INTEGER PRIMARY KEY,
  theme     TEXT DEFAULT 'light',
  language  TEXT DEFAULT 'en',
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- First call: inserts new row
INSERT INTO user_settings (user_id, theme, language)
VALUES (1, 'dark', 'en')
ON CONFLICT (user_id)
DO UPDATE SET
  theme      = EXCLUDED.theme,
  language   = EXCLUDED.language,
  updated_at = NOW();

-- Second call with same user_id: updates existing row
INSERT INTO user_settings (user_id, theme, language)
VALUES (1, 'light', 'fr')
ON CONFLICT (user_id)
DO UPDATE SET
  theme      = EXCLUDED.theme,
  language   = EXCLUDED.language,
  updated_at = NOW();
-- Now user 1 has theme='light', language='fr'
```

---

#### <span style="color:#5B8DB8">Example 2 - Update Only Specific Columns</span>

```sql
CREATE TABLE products (
  sku         TEXT PRIMARY KEY,
  name        TEXT NOT NULL,
  price       NUMERIC(10,2),
  stock       INTEGER DEFAULT 0,
  last_synced TIMESTAMPTZ DEFAULT NOW()
);

-- Upsert product - update price and stock only, never update name
INSERT INTO products (sku, name, price, stock)
VALUES ('ABC-001', 'Widget Pro', 29.99, 150)
ON CONFLICT (sku)
DO UPDATE SET
  price       = EXCLUDED.price,     -- update price
  stock       = EXCLUDED.stock,     -- update stock
  last_synced = NOW();
  -- name is NOT updated - keeps whatever was there before
```

---

#### <span style="color:#5B8DB8">Example 3 - Referencing Both EXCLUDED and Existing Row</span>

```sql
CREATE TABLE page_views (
  page_url   TEXT PRIMARY KEY,
  view_count INTEGER DEFAULT 0,
  first_seen TIMESTAMPTZ DEFAULT NOW(),
  last_seen  TIMESTAMPTZ DEFAULT NOW()
);

-- Increment counter on conflict (don't reset it!)
INSERT INTO page_views (page_url, view_count)
VALUES ('/home', 1)
ON CONFLICT (page_url)
DO UPDATE SET
  view_count = page_views.view_count + 1,  -- page_views = current row in table
  last_seen  = NOW();
  -- first_seen is NOT updated - preserves original value

-- EXCLUDED.view_count = 1 (the value we tried to insert)
-- page_views.view_count = existing count in the table
-- page_views.view_count + 1 = increment existing value
```

---

#### <span style="color:#5B8DB8">Example 4 - UPSERT with WHERE (Conditional Update)</span>

```sql
-- Only update if the new price is lower than the current price
INSERT INTO products (sku, name, price, stock)
VALUES ('ABC-001', 'Widget Pro', 19.99, 200)
ON CONFLICT (sku)
DO UPDATE SET
  price = EXCLUDED.price,
  stock = EXCLUDED.stock
WHERE EXCLUDED.price < products.price;  -- only update if new price is cheaper
```

---

#### <span style="color:#5B8DB8">Example 5 - Conflict on Composite Key</span>

```sql
CREATE TABLE student_courses (
  student_id  INTEGER,
  course_id   INTEGER,
  grade       TEXT,
  enrolled_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (student_id, course_id)  -- composite PK
);

-- Upsert on composite key
INSERT INTO student_courses (student_id, course_id, grade)
VALUES (42, 101, 'B+')
ON CONFLICT (student_id, course_id)
DO UPDATE SET grade = EXCLUDED.grade;
```

---

#### <span style="color:#5B8DB8">Example 6 - Conflict on Named Constraint</span>

```sql
-- Reference a constraint by name instead of column list
ALTER TABLE products ADD CONSTRAINT products_sku_unique UNIQUE (sku);

INSERT INTO products (sku, name, price)
VALUES ('ABC-001', 'Widget', 25.00)
ON CONFLICT ON CONSTRAINT products_sku_unique
DO UPDATE SET price = EXCLUDED.price;
```

---

### <span style="color:#2E86AB">EXCLUDED - Understanding the Special Table</span>

```
EXCLUDED = the row that was blocked from being inserted
         = the values you provided in the VALUES clause

table_name = the row that already exists in the table

SET col = EXCLUDED.col   → use the NEW value (from your INSERT)
SET col = table_name.col → keep the OLD value (already in table)
SET col = table_name.col + EXCLUDED.col → combine both
```

```sql
-- Full demonstration
INSERT INTO inventory (item_id, quantity, price)
VALUES (10, 50, 9.99)
ON CONFLICT (item_id)
DO UPDATE SET
  quantity = inventory.quantity + EXCLUDED.quantity,  -- ADD new qty to existing
  price    = EXCLUDED.price,                          -- REPLACE with new price
  updated  = NOW();
```

---

### <span style="color:#2E86AB">DO NOTHING vs DO UPDATE - When to Use Which</span>

| Use Case | Strategy |
|:---|:---|
| Insert if not exists, skip if exists | `ON CONFLICT DO NOTHING` |
| Insert or fully replace on conflict | `ON CONFLICT DO UPDATE SET col = EXCLUDED.col` |
| Insert or increment a counter | `ON CONFLICT DO UPDATE SET count = table.count + 1` |
| Insert or update only if new data is "better" | `ON CONFLICT DO UPDATE SET ... WHERE condition` |
| Sync / bulk import without errors | `ON CONFLICT DO NOTHING` |

---

## <span style="color:#1565C0">4.7 COPY Command</span>

> **Definition:** `COPY` is PostgreSQL's high-performance bulk data transfer command. It moves data between a table and a file (or stdin/stdout) directly in the server process - far faster than individual `INSERT` statements for large datasets.

---

### <span style="color:#2E86AB">COPY vs INSERT Performance</span>

| Method | 1 million rows |
|:---|:---|
| Individual `INSERT` statements | Minutes |
| Multi-row `INSERT` (1000 rows/stmt) | Seconds |
| `COPY` | Sub-second |

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 580px;">

### <span style="color:#D4A017">COPY is the fastest way to bulk-load data into PostgreSQL. It bypasses most overhead, uses minimal WAL logging, and streams data directly to the storage engine.</span>

</div>

---

### <span style="color:#2E86AB">COPY FROM - Import Data into a Table</span>

```sql
-- Import from a CSV file (server-side path - must have superuser or pg_read_server_files)
COPY employees (first_name, last_name, email, department, salary)
FROM '/tmp/employees.csv'
WITH (FORMAT CSV, HEADER TRUE);

-- Full syntax with all options
COPY employees
FROM '/tmp/employees.csv'
WITH (
  FORMAT     CSV,          -- CSV, TEXT, or BINARY
  HEADER     TRUE,         -- skip the first header row
  DELIMITER  ',',          -- column separator (default for CSV)
  QUOTE      '"',          -- quote character for string fields
  ESCAPE     '"',          -- escape character inside quoted fields
  NULL       '',           -- what string represents NULL (empty string here)
  ENCODING   'UTF8'        -- file encoding
);

-- Import TSV (tab-separated values)
COPY products (sku, name, price)
FROM '/tmp/products.tsv'
WITH (FORMAT TEXT, DELIMITER E'\t', HEADER TRUE);

-- Import specific columns (others get DEFAULT or NULL)
COPY employees (first_name, last_name, email)
FROM '/tmp/names_only.csv'
WITH (FORMAT CSV, HEADER TRUE);

-- Import from stdin (pipe data directly)
COPY employees FROM STDIN WITH (FORMAT CSV);
-- Then type/paste rows, end with \.
```

---

### <span style="color:#2E86AB">COPY TO - Export Data from a Table</span>

```sql
-- Export entire table to a CSV file
COPY employees
TO '/tmp/employees_export.csv'
WITH (FORMAT CSV, HEADER TRUE);

-- Export specific columns
COPY employees (id, first_name, last_name, email)
TO '/tmp/employees_names.csv'
WITH (FORMAT CSV, HEADER TRUE);

-- Export a query result (not just a full table)
COPY (
  SELECT first_name, last_name, email, department, salary
  FROM employees
  WHERE is_active = TRUE
  ORDER BY department, last_name
)
TO '/tmp/active_employees.csv'
WITH (FORMAT CSV, HEADER TRUE);

-- Export to TSV
COPY employees TO '/tmp/employees.tsv'
WITH (FORMAT TEXT, HEADER TRUE);

-- Export to stdout (useful in scripts)
COPY employees TO STDOUT WITH (FORMAT CSV, HEADER TRUE);

-- Custom NULL representation
COPY employees TO '/tmp/out.csv'
WITH (FORMAT CSV, HEADER TRUE, NULL 'NULL');
-- NULL values written as the string "NULL" instead of empty
```

---

### <span style="color:#2E86AB">\copy - Client-Side COPY (psql)</span>

> **Definition:** `\copy` is psql's client-side equivalent of `COPY`. The difference is critical: `COPY` reads/writes files on the **server** filesystem, while `\copy` reads/writes files on the **client** (your local machine) filesystem. `\copy` does not require superuser privileges.

```
          COPY                    \copy
            ↓                       ↓
   Files on DB server        Files on your laptop
   Needs superuser           Needs no special privilege
   Faster (no network)       Slightly slower (streams over connection)
```

```sql
-- \copy is used inside psql (not a SQL statement - it's a meta-command)
-- Note: \copy is ONE line - no line breaks allowed

-- Import from local file
\copy employees (first_name, last_name, email, department, salary) FROM '/home/sagar/employees.csv' WITH (FORMAT CSV, HEADER TRUE)

-- Export to local file
\copy employees TO '/home/sagar/employees_backup.csv' WITH (FORMAT CSV, HEADER TRUE)

-- Export a query to local file
\copy (SELECT * FROM employees WHERE department = 'Engineering') TO '/home/sagar/eng_team.csv' WITH (FORMAT CSV, HEADER TRUE)

-- Import TSV from local file
\copy products FROM '/home/sagar/products.tsv' WITH (FORMAT TEXT, DELIMITER E'\t', HEADER TRUE)
```

---

### <span style="color:#2E86AB">COPY Format Options</span>

| Option | Values | Notes |
|:---|:---|:---|
| `FORMAT` | `CSV`, `TEXT`, `BINARY` | CSV is most common; BINARY is fastest but not human-readable |
| `HEADER` | `TRUE`, `FALSE` | Skip/add header row in CSV/TEXT |
| `DELIMITER` | any single char | Default: `,` for CSV, tab for TEXT |
| `QUOTE` | any single char | Default: `"` for CSV |
| `ESCAPE` | any single char | Default: same as QUOTE for CSV |
| `NULL` | string | How NULLs are represented (default: empty string for CSV) |
| `ENCODING` | encoding name | `UTF8`, `LATIN1`, etc. |
| `FORCE_QUOTE` | column list | Always quote these columns |
| `FORCE_NOT_NULL` | column list | Never treat as NULL |
| `FREEZE` | boolean | Mark rows as frozen for VACUUM (import optimisation) |

---

### <span style="color:#2E86AB">COPY Error Handling</span>

```sql
-- If any row fails, the entire COPY fails (all-or-nothing)
-- To handle bad rows, use a staging table approach:

-- Step 1: Create a staging table with all TEXT columns
CREATE TEMP TABLE employees_staging (
  first_name  TEXT,
  last_name   TEXT,
  email       TEXT,
  department  TEXT,
  salary      TEXT   -- TEXT to accept anything
);

-- Step 2: COPY raw data into staging (won't fail on type mismatches)
COPY employees_staging FROM '/tmp/messy_data.csv'
WITH (FORMAT CSV, HEADER TRUE);

-- Step 3: Validate and insert clean rows into the real table
INSERT INTO employees (first_name, last_name, email, department, salary)
SELECT
  first_name,
  last_name,
  email,
  department,
  salary::NUMERIC(10,2)     -- cast TEXT to NUMERIC
FROM employees_staging
WHERE email    LIKE '%@%.%'  -- basic email format check
  AND salary   ~ '^\d+(\.\d+)?$';  -- valid number format

-- Step 4: Log bad rows
SELECT * FROM employees_staging
WHERE salary !~ '^\d+(\.\d+)?$';
```

---

### <span style="color:#2E86AB">COPY vs INSERT - When to Use Which</span>

| Scenario | Use |
|:---|:---|
| Load millions of rows from a CSV/file | `COPY` or `\copy` |
| Insert a few rows from application code | `INSERT` |
| Insert rows with complex logic or FK lookups | `INSERT` with subqueries |
| Sync/update existing rows | `INSERT ON CONFLICT` |
| Export data for backup or sharing | `COPY TO` |
| ETL pipeline - bulk load into staging | `COPY` into staging → `INSERT INTO ... SELECT` |

---

## <span style="color:#1565C0">Phase 4 - Quick Reference Card</span>

### <span style="color:#2E86AB">INSERT Patterns</span>

```sql
-- Single row
INSERT INTO t (c1, c2) VALUES (v1, v2);

-- Multi-row
INSERT INTO t (c1, c2) VALUES (v1, v2), (v3, v4), (v5, v6);

-- From a query
INSERT INTO t (c1, c2) SELECT c1, c2 FROM source WHERE condition;

-- With return of generated values
INSERT INTO t (c1, c2) VALUES (v1, v2) RETURNING id, created_at;

-- Upsert - skip on conflict
INSERT INTO t (c1) VALUES (v1) ON CONFLICT (c1) DO NOTHING;

-- Upsert - update on conflict
INSERT INTO t (c1, c2) VALUES (v1, v2)
ON CONFLICT (c1) DO UPDATE SET c2 = EXCLUDED.c2;
```

### <span style="color:#2E86AB">UPDATE Patterns</span>

```sql
-- Simple update
UPDATE t SET c1 = v1 WHERE condition;

-- Multi-column update
UPDATE t SET c1 = v1, c2 = v2, c3 = v3 WHERE condition;

-- Update from another table (PostgreSQL join update)
UPDATE t SET c1 = other.c1 FROM other WHERE t.id = other.id;

-- Update with RETURNING
UPDATE t SET c1 = v1 WHERE condition RETURNING id, c1;
```

### <span style="color:#2E86AB">DELETE Patterns</span>

```sql
-- Simple delete
DELETE FROM t WHERE condition;

-- Delete from another table's condition (PostgreSQL join delete)
DELETE FROM t USING other WHERE t.id = other.t_id AND other.condition;

-- Delete with RETURNING
DELETE FROM t WHERE condition RETURNING *;

-- Archive then delete (CTE)
WITH deleted AS (DELETE FROM t WHERE condition RETURNING *)
INSERT INTO archive SELECT * FROM deleted;
```

### <span style="color:#2E86AB">COPY Patterns</span>

```sql
-- Server-side import
COPY t FROM '/path/file.csv' WITH (FORMAT CSV, HEADER TRUE);

-- Server-side export
COPY t TO '/path/file.csv' WITH (FORMAT CSV, HEADER TRUE);

-- Client-side import (psql)
\copy t FROM '/local/path/file.csv' WITH (FORMAT CSV, HEADER TRUE)

-- Client-side export (psql)
\copy t TO '/local/path/file.csv' WITH (FORMAT CSV, HEADER TRUE)

-- Export query result
COPY (SELECT * FROM t WHERE cond) TO '/path/out.csv' WITH (FORMAT CSV, HEADER TRUE);
```

### <span style="color:#2E86AB">RETURNING - Key Use Cases</span>

| Statement | Use RETURNING for |
|:---|:---|
| `INSERT` | Get auto-generated `id`, `created_at`, `uuid` |
| `UPDATE` | Verify new values after update |
| `DELETE` | Log deleted rows, archive data |
| Any | Avoid a separate SELECT round-trip |