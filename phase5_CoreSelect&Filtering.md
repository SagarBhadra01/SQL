<div align="center">

# <span style="color:#0A2FA8">PostgreSQL : Phase 5</span>

<sub>Core SELECT & Filtering · Complete Notes · WHERE, ORDER BY, LIMIT, SET Operations, Expressions</sub>

</div>

---

## <span style="color:#1565C0">Reference Table - Used Throughout Phase 5</span>

```sql
-- All examples in this phase use these tables

CREATE TABLE employees (
  id          SERIAL PRIMARY KEY,
  first_name  TEXT NOT NULL,
  last_name   TEXT NOT NULL,
  email       TEXT UNIQUE NOT NULL,
  department  TEXT,
  salary      NUMERIC(10,2),
  is_active   BOOLEAN DEFAULT TRUE,
  hired_on    DATE DEFAULT CURRENT_DATE,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE departments (
  id      SERIAL PRIMARY KEY,
  name    TEXT UNIQUE NOT NULL,
  budget  NUMERIC(12,2),
  head_id INTEGER REFERENCES employees(id)
);

-- Sample data
INSERT INTO employees (first_name, last_name, email, department, salary, is_active, hired_on)
VALUES
  ('Alice',   'Kumar',  'alice@co.com',  'Engineering', 95000, TRUE,  '2019-03-10'),
  ('Bob',     'Singh',  'bob@co.com',    'Engineering', 78000, TRUE,  '2021-07-22'),
  ('Carol',   'Mehta',  'carol@co.com',  'HR',          62000, TRUE,  '2020-01-15'),
  ('Dave',    'Patel',  'dave@co.com',   'Finance',     84000, FALSE, '2018-11-05'),
  ('Eva',     'Joshi',  'eva@co.com',    'HR',          58000, TRUE,  '2022-09-01'),
  ('Frank',   'Nair',   'frank@co.com',  'Engineering', 105000,TRUE,  '2017-04-18'),
  ('Grace',   'Reddy',  'grace@co.com',  'Finance',     91000, TRUE,  '2020-06-30'),
  ('Hank',    'Das',    'hank@co.com',   'Marketing',   67000, FALSE, '2019-12-12'),
  ('Irene',   'Sen',    'irene@co.com',  'Engineering', 88000, TRUE,  '2021-02-28'),
  ('Jay',     'Roy',    'jay@co.com',    'Marketing',   72000, TRUE,  '2023-05-14'),
  ('Kim',     'Bose',   'kim@co.com',    NULL,          55000, TRUE,  '2022-11-01'),
  ('Leo',     'Ghosh',  'leo@co.com',    'Engineering', 112000,TRUE,  '2016-08-09');
```

---

## <span style="color:#1565C0">5.1 SELECT Clause - Expressions and Aliases</span>

> **Definition:** The `SELECT` clause defines what appears in the output - which columns, computed expressions, literals, and function results. Every item in the output list can be renamed with an alias using `AS`.

---

### <span style="color:#2E86AB">Selecting Columns</span>

```sql
-- All columns (wildcard)
SELECT * FROM employees;

-- Specific columns
SELECT first_name, last_name, salary FROM employees;

-- Order in SELECT controls output column order (independent of table definition)
SELECT salary, email, first_name FROM employees;
```

---

### <span style="color:#2E86AB">Column Aliases with AS</span>

> **Definition:** An alias renames a column or expression in the query output. It does not affect the underlying table. Use `AS` keyword (optional but recommended for clarity).

```sql
-- Rename columns in output
SELECT
  first_name  AS "First Name",
  last_name   AS "Last Name",
  salary      AS annual_salary,
  department  AS dept
FROM employees;

-- AS is optional but recommended
SELECT first_name name, salary pay FROM employees;  -- works but less readable

-- Aliases with spaces require double-quotes
SELECT first_name AS "Employee First Name" FROM employees;

-- Aliases are case-sensitive when quoted
SELECT salary AS "Salary", salary AS salary_col FROM employees;
-- "Salary" and salary_col are different alias names
```

---

### <span style="color:#2E86AB">Expressions in SELECT</span>

```sql
-- Arithmetic expressions
SELECT
  first_name,
  salary,
  salary / 12                      AS monthly_salary,
  salary * 1.10                    AS with_10pct_raise,
  ROUND(salary * 0.18, 2)          AS tax_at_18pct,
  salary - ROUND(salary * 0.18, 2) AS take_home
FROM employees;

-- String expressions
SELECT
  first_name || ' ' || last_name          AS full_name,
  UPPER(first_name) || '.' || UPPER(last_name) AS initials_style,
  LENGTH(first_name || last_name)         AS name_char_count
FROM employees;

-- Date expressions
SELECT
  first_name,
  hired_on,
  CURRENT_DATE - hired_on                  AS days_employed,
  EXTRACT(YEAR FROM AGE(hired_on))::INT    AS years_employed,
  DATE_TRUNC('year', hired_on)::DATE       AS hire_year
FROM employees;

-- Boolean expressions
SELECT
  first_name,
  salary,
  salary > 80000             AS is_high_earner,
  department = 'Engineering' AS is_engineer,
  is_active AND salary > 70000 AS active_and_paid_well
FROM employees;

-- Literal values in SELECT
SELECT
  id,
  first_name,
  'Company Inc'    AS company,       -- string literal
  2024             AS report_year,   -- integer literal
  TRUE             AS is_current     -- boolean literal
FROM employees;

-- Conditional expression (CASE - full coverage in 5.14)
SELECT
  first_name,
  salary,
  CASE
    WHEN salary >= 100000 THEN 'Senior'
    WHEN salary >=  75000 THEN 'Mid-Level'
    ELSE 'Junior'
  END AS seniority_band
FROM employees;
```

---

### <span style="color:#2E86AB">SELECT DISTINCT in the SELECT Clause</span>

```sql
-- All unique departments (including NULL)
SELECT DISTINCT department FROM employees ORDER BY department;

-- Covered fully in section 5.9
```

---

### <span style="color:#2E86AB">Using Aliases Downstream</span>

```sql
-- Aliases defined in SELECT are available in ORDER BY
SELECT salary * 12 AS annual FROM employees ORDER BY annual DESC;

-- But NOT available in WHERE (WHERE runs before SELECT)
SELECT salary * 12 AS annual FROM employees WHERE annual > 100000; -- ERROR

-- And NOT in GROUP BY either (use the expression again, or column position)
SELECT department, AVG(salary) AS avg_sal FROM employees
GROUP BY department ORDER BY avg_sal DESC;  -- alias OK in ORDER BY

-- Column position reference (1-based) - works in ORDER BY and GROUP BY
SELECT department, AVG(salary) FROM employees GROUP BY 1 ORDER BY 2 DESC;
-- 1 = department, 2 = AVG(salary)
```

---

## <span style="color:#1565C0">5.2 FROM Clause</span>

> **Definition:** The `FROM` clause identifies the source of rows for the query - one or more tables, subqueries, CTEs, functions, or values. Every `SELECT` that reads data must have a `FROM`.

```sql
-- Single table
SELECT * FROM employees;

-- With schema prefix
SELECT * FROM public.employees;

-- From a subquery (inline view / derived table)
SELECT avg_data.department, avg_data.avg_salary
FROM (
  SELECT department, AVG(salary) AS avg_salary
  FROM employees
  GROUP BY department
) AS avg_data
WHERE avg_data.avg_salary > 75000;

-- From a function (returns rows)
SELECT * FROM generate_series(1, 5) AS gs(n);
-- n: 1, 2, 3, 4, 5

-- From a VALUES list (inline table)
SELECT *
FROM (VALUES
  ('Engineering', 100000),
  ('HR',           65000),
  ('Finance',      80000)
) AS dept_targets(department, target_salary);
-- Acts as a small inline table

-- Multiple tables (implicit cross join - usually an accident, avoid without WHERE)
SELECT * FROM employees, departments;  -- every employee × every department row
```

---

## <span style="color:#1565C0">5.3 WHERE Clause</span>

> **Definition:** The `WHERE` clause filters rows returned by `FROM`. Only rows where the condition evaluates to `TRUE` are included in the result. Rows where the condition is `FALSE` or `NULL` are excluded.

---

### <span style="color:#2E86AB">Comparison Operators</span>

| Operator | Meaning | Example |
|:---:|:---|:---|
| `=` | Equal to | `salary = 80000` |
| `<>` | Not equal to (SQL standard) | `department <> 'HR'` |
| `!=` | Not equal to (same as `<>`) | `department != 'HR'` |
| `<` | Less than | `salary < 70000` |
| `>` | Greater than | `salary > 90000` |
| `<=` | Less than or equal | `salary <= 75000` |
| `>=` | Greater than or equal | `hired_on >= '2020-01-01'` |

```sql
-- Equal to
SELECT * FROM employees WHERE department = 'Engineering';

-- Not equal (both <> and != work the same in PostgreSQL)
SELECT * FROM employees WHERE department <> 'HR';
SELECT * FROM employees WHERE department != 'HR';

-- Less than / greater than
SELECT * FROM employees WHERE salary < 70000;
SELECT * FROM employees WHERE salary > 90000;

-- Less/greater than or equal
SELECT * FROM employees WHERE salary <= 75000;
SELECT * FROM employees WHERE hired_on >= '2020-01-01';

-- String comparison (lexicographic / alphabetical)
SELECT * FROM employees WHERE last_name >= 'N';  -- last names N onwards

-- Date comparison
SELECT * FROM employees WHERE hired_on > '2021-01-01';
SELECT * FROM employees WHERE hired_on = '2019-03-10';
```

---

### <span style="color:#2E86AB">Logical Operators - AND, OR, NOT</span>

```sql
-- AND - both conditions must be TRUE
SELECT * FROM employees
WHERE department = 'Engineering'
  AND salary > 90000;
-- Alice (95000) ✓, Frank (105000) ✓, Leo (112000) ✓

-- OR - at least one condition must be TRUE
SELECT * FROM employees
WHERE department = 'HR'
   OR department = 'Finance';
-- Carol, Eva (HR), Dave, Grace (Finance)

-- NOT - inverts the condition
SELECT * FROM employees
WHERE NOT department = 'Engineering';
-- Everyone except Engineering staff

-- Combining AND, OR, NOT
SELECT * FROM employees
WHERE (department = 'Engineering' OR department = 'Finance')
  AND salary > 80000
  AND is_active = TRUE;

-- NOT with IS NULL
SELECT * FROM employees WHERE NOT department IS NULL;
-- Same as: WHERE department IS NOT NULL
```

---

### <span style="color:#2E86AB">Operator Precedence</span>

> **Definition:** When multiple logical operators appear in a `WHERE` clause, PostgreSQL evaluates them in a fixed order - NOT first, then AND, then OR. Misunderstanding this causes subtle bugs.

```
Precedence (highest to lowest):
  1. NOT
  2. AND
  3. OR
```

```sql
-- This might not do what you expect:
SELECT * FROM employees
WHERE department = 'HR' OR department = 'Finance' AND salary > 80000;

-- PostgreSQL reads it as:
SELECT * FROM employees
WHERE department = 'HR' OR (department = 'Finance' AND salary > 80000);
-- Returns ALL HR employees (any salary)
-- AND Finance employees with salary > 80000

-- What you probably wanted:
SELECT * FROM employees
WHERE (department = 'HR' OR department = 'Finance') AND salary > 80000;
-- Returns HR AND Finance employees, both with salary > 80000
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 580px;">

### <span style="color:#D4A017">Always use parentheses when mixing AND and OR. Never rely on implicit precedence - it makes code harder to read and introduces subtle bugs.</span>

</div>

---

```sql
-- More precedence examples
SELECT * FROM employees
WHERE NOT is_active AND department = 'Marketing';
-- NOT binds first: (NOT is_active) AND department = 'Marketing'
-- Inactive Marketing employees

SELECT * FROM employees
WHERE NOT (is_active AND department = 'Marketing');
-- Negates the whole group: employees who are NOT (active AND in Marketing)
-- Very different result!
```

---

## <span style="color:#1565C0">5.4 BETWEEN ... AND ...</span>

> **Definition:** `BETWEEN` tests whether a value falls within an inclusive range. `value BETWEEN low AND high` is exactly equivalent to `value >= low AND value <= high`. Both endpoints are **included**.

---

### <span style="color:#2E86AB">Numeric BETWEEN</span>

```sql
-- Find employees earning between 70000 and 90000 (inclusive)
SELECT first_name, salary
FROM employees
WHERE salary BETWEEN 70000 AND 90000;
-- Returns: Bob (78000), Dave (84000), Irene (88000), Jay (72000)

-- Equivalent expression
SELECT first_name, salary
FROM employees
WHERE salary >= 70000 AND salary <= 90000;  -- identical result

-- NOT BETWEEN - outside the range
SELECT first_name, salary
FROM employees
WHERE salary NOT BETWEEN 70000 AND 90000;
-- Returns those earning < 70000 OR > 90000
```

---

### <span style="color:#2E86AB">Date BETWEEN</span>

```sql
-- Employees hired in 2020
SELECT first_name, hired_on
FROM employees
WHERE hired_on BETWEEN '2020-01-01' AND '2020-12-31';
-- Returns: Carol (2020-01-15), Grace (2020-06-30)

-- Employees hired in the last 3 years
SELECT first_name, hired_on
FROM employees
WHERE hired_on BETWEEN CURRENT_DATE - INTERVAL '3 years' AND CURRENT_DATE;

-- TIMESTAMPTZ BETWEEN - be careful with end-of-day
SELECT * FROM orders
WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31';
-- DANGER: '2024-01-31' = '2024-01-31 00:00:00' - misses rest of Jan 31

-- Safer for timestamps: use < instead of <=
SELECT * FROM orders
WHERE created_at >= '2024-01-01'
  AND created_at  < '2024-02-01';  -- strictly before Feb 1
```

---

### <span style="color:#2E86AB">String BETWEEN</span>

```sql
-- Alphabetical range
SELECT first_name FROM employees
WHERE first_name BETWEEN 'A' AND 'F';
-- Returns names starting with A, B, C, D, E (F not included unless exact match)
-- String comparison is lexicographic (character by character)

-- More predictable: use LIKE or >= / <=
SELECT first_name FROM employees WHERE first_name >= 'D' AND first_name < 'G';
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 580px;">

### <span style="color:#D4A017">BETWEEN is always INCLUSIVE on both ends. BETWEEN 1 AND 10 includes both 1 and 10. For timestamps, prefer >= start AND &lt; next_day to avoid midnight-cutoff surprises.</span>

</div>

---

## <span style="color:#1565C0">5.5 IN and NOT IN</span>

> **Definition:** `IN` tests whether a value matches any value in a given list or subquery. It is a cleaner, more readable alternative to multiple `OR` conditions.

---

### <span style="color:#2E86AB">IN with a Value List</span>

```sql
-- Employees in specific departments
SELECT first_name, department
FROM employees
WHERE department IN ('Engineering', 'Finance', 'HR');

-- Equivalent OR expression (more verbose)
SELECT first_name, department
FROM employees
WHERE department = 'Engineering'
   OR department = 'Finance'
   OR department = 'HR';

-- IN with numbers
SELECT * FROM employees WHERE id IN (1, 3, 5, 7, 9);

-- IN with dates
SELECT * FROM employees
WHERE hired_on IN ('2019-03-10', '2020-01-15', '2021-07-22');

-- IN with a single value (valid but pointless - use = instead)
SELECT * FROM employees WHERE department IN ('Engineering');
```

---

### <span style="color:#2E86AB">NOT IN</span>

```sql
-- Everyone NOT in these departments
SELECT first_name, department
FROM employees
WHERE department NOT IN ('Marketing', 'HR');

-- NOT IN with a list
SELECT * FROM employees WHERE id NOT IN (4, 8);
```

---

### <span style="color:#2E86AB">IN with a Subquery</span>

```sql
-- Find employees in departments that have more than 2 employees
SELECT first_name, department
FROM employees
WHERE department IN (
  SELECT department
  FROM employees
  WHERE department IS NOT NULL
  GROUP BY department
  HAVING COUNT(*) > 2
);

-- Find employees whose salary is above the average of any department
SELECT first_name, salary
FROM employees
WHERE salary IN (
  SELECT MAX(salary) FROM employees GROUP BY department
);
-- Returns the top earner from each department
```

---

### <span style="color:#2E86AB">The NULL Trap with NOT IN</span>

```sql
-- The employees table has Kim with department = NULL

-- IN works fine with NULLs in the list
SELECT first_name FROM employees WHERE department IN ('HR', NULL);
-- Returns Carol and Eva (the HR employees)
-- NULL in the IN list is simply ignored in matching

-- NOT IN with NULLs in the list - RETURNS NO ROWS
SELECT first_name FROM employees WHERE department NOT IN ('HR', NULL);
-- Returns NOTHING - because x NOT IN (..., NULL)
-- is equivalent to: x <> 'HR' AND x <> NULL
-- x <> NULL is always NULL, and NULL makes the whole condition NULL

-- The cause: NOT IN (list_with_null) → always evaluates to NULL (never TRUE)

-- CORRECT approach - handle NULLs explicitly
SELECT first_name FROM employees
WHERE department NOT IN ('HR')
  AND department IS NOT NULL;

-- Or use NOT EXISTS / EXCEPT (covered in Phase 7/8) for NULL-safe exclusion
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 580px;">

### <span style="color:#D4A017">Critical NULL Trap: If the subquery or list inside NOT IN contains even ONE NULL, the entire NOT IN condition returns NULL for every row - giving you zero results. Always add IS NOT NULL when using NOT IN with subqueries.</span>

</div>

```sql
-- Safe pattern for NOT IN with subquery
SELECT first_name FROM employees
WHERE department NOT IN (
  SELECT department FROM some_table WHERE department IS NOT NULL  -- guard here
);
```

---

## <span style="color:#1565C0">5.6 LIKE - Pattern Matching with Wildcards</span>

> **Definition:** `LIKE` performs pattern matching on text strings using two special wildcard characters. It is case-sensitive in PostgreSQL.

---

### <span style="color:#2E86AB">Wildcard Characters</span>

| Wildcard | Meaning | Example | Matches |
|:---:|:---|:---|:---|
| `%` | Zero or more of any characters | `'A%'` | `Alice`, `AB`, `A` |
| `_` | Exactly one of any character | `'_ob'` | `Bob`, `Job`, `Rob` |

---

### <span style="color:#2E86AB">LIKE Pattern Examples</span>

```sql
-- % at the end - starts with
SELECT * FROM employees WHERE first_name LIKE 'A%';
-- Alice

-- % at the start - ends with
SELECT * FROM employees WHERE email LIKE '%@co.com';
-- All employees (all emails end with @co.com)

-- % on both sides - contains
SELECT * FROM employees WHERE email LIKE '%kumar%';
-- Nothing - LIKE is case-sensitive! 'kumar' doesn't match 'Kumar'

-- Exact match with no wildcards (same as =)
SELECT * FROM employees WHERE first_name LIKE 'Alice';  -- same as = 'Alice'

-- _ wildcard - exactly one character
SELECT * FROM employees WHERE first_name LIKE '_ob';
-- Matches: Bob, Job, Rob etc.

SELECT * FROM employees WHERE first_name LIKE '___';
-- Matches names with exactly 3 characters: Kim, Jay, Bob, Eva, Leo

SELECT * FROM employees WHERE first_name LIKE '_a%';
-- Names where second character is 'a': Dave, Jay, Hank, Carol, Grace

-- Mixing % and _
SELECT * FROM employees WHERE email LIKE '_a%@%.com';
-- Email where second char is 'a'

-- NOT LIKE
SELECT * FROM employees WHERE first_name NOT LIKE 'A%';
-- Everyone except Alice
```

---

### <span style="color:#2E86AB">Escaping Wildcard Characters</span>

```sql
-- If your data contains literal % or _, use ESCAPE
-- Find values that contain a literal percent sign

SELECT * FROM discounts WHERE description LIKE '%50\%%' ESCAPE '\';
-- Matches descriptions containing '50%' as a literal string

-- Default escape is backslash in PostgreSQL
SELECT * FROM file_paths WHERE path LIKE '%\_backup%';
-- Matches paths containing '_backup' as a literal underscore
```

---

## <span style="color:#1565C0">5.7 ILIKE - Case-Insensitive Pattern Matching (PostgreSQL-Specific)</span>

> **Definition:** `ILIKE` is PostgreSQL's case-insensitive version of `LIKE`. It behaves identically to `LIKE` but ignores case differences. It is not part of the SQL standard.

```sql
-- LIKE is case-sensitive
SELECT * FROM employees WHERE first_name LIKE 'alice';  -- 0 rows
SELECT * FROM employees WHERE first_name LIKE 'Alice';  -- 1 row (Alice)

-- ILIKE is case-insensitive
SELECT * FROM employees WHERE first_name ILIKE 'alice';  -- returns Alice
SELECT * FROM employees WHERE first_name ILIKE 'ALICE';  -- returns Alice
SELECT * FROM employees WHERE first_name ILIKE 'aLiCe';  -- returns Alice

-- Common use case: case-insensitive search in applications
SELECT first_name, email FROM employees
WHERE email ILIKE '%engineering%';

-- User search - match regardless of how they typed it
SELECT * FROM employees
WHERE first_name ILIKE '%' || 'frank' || '%';
-- Returns Frank

-- NOT ILIKE
SELECT * FROM employees WHERE department NOT ILIKE '%engineering%';

-- ILIKE with _ wildcard (case-insensitive)
SELECT * FROM employees WHERE first_name ILIKE '_ob';
-- Matches: Bob, Job, Rob, bob, BOB, jOb ...
```

---

### <span style="color:#2E86AB">LIKE vs ILIKE vs SIMILAR TO vs Regex</span>

| Operator | Case-sensitive | Standard SQL | Pattern Type |
|:---|:---:|:---:|:---|
| `LIKE` | ✓ | ✓ | Simple `%` and `_` wildcards |
| `ILIKE` | ✗ | ✗ (PostgreSQL only) | Simple `%` and `_` wildcards |
| `SIMILAR TO` | ✓ | ✓ | SQL regex (mix of LIKE and regex) |
| `~` | ✓ | ✗ | POSIX regular expressions |
| `~*` | ✗ | ✗ | Case-insensitive POSIX regex |

```sql
-- SIMILAR TO (SQL regex - limited)
SELECT * FROM employees WHERE first_name SIMILAR TO '(Alice|Bob|Carol)';

-- POSIX regex ~ (case-sensitive)
SELECT * FROM employees WHERE first_name ~ '^[A-F]';  -- starts with A-F

-- POSIX regex ~* (case-insensitive)
SELECT * FROM employees WHERE email ~* 'KUMAR|SINGH|MEHTA';
```

---

## <span style="color:#1565C0">5.8 IS NULL / IS NOT NULL</span>

> **Definition:** `IS NULL` tests whether a value is NULL. `IS NOT NULL` tests whether a value is not NULL. These are the only correct operators for NULL checking - using `= NULL` always returns NULL, never TRUE.

```sql
-- Find employees with no department assigned
SELECT first_name, department
FROM employees
WHERE department IS NULL;
-- Returns: Kim (her department is NULL)

-- Find employees who have a department
SELECT first_name, department
FROM employees
WHERE department IS NOT NULL;
-- Returns all except Kim

-- WRONG - this returns 0 rows, not NULLs
SELECT * FROM employees WHERE department = NULL;    -- always empty
SELECT * FROM employees WHERE department != NULL;   -- always empty

-- IS NULL in combination with other conditions
SELECT first_name, salary, department
FROM employees
WHERE department IS NULL
   OR salary < 60000;

-- Checking multiple NULLable columns
SELECT * FROM employees
WHERE department IS NULL
  AND salary IS NOT NULL;
```

---

### <span style="color:#2E86AB">IS DISTINCT FROM and IS NOT DISTINCT FROM</span>

> **Definition:** `IS DISTINCT FROM` is a NULL-safe comparison operator. Unlike `<>`, it treats two NULLs as equal and NULL vs non-NULL as not equal. Useful when comparing nullable columns.

```sql
-- Normal <> fails to detect NULL vs non-NULL differences
SELECT NULL <> NULL;    -- NULL (not TRUE or FALSE)
SELECT NULL <> 'HR';    -- NULL

-- IS DISTINCT FROM is NULL-safe
SELECT NULL IS DISTINCT FROM NULL;     -- FALSE (they are the same - both NULL)
SELECT NULL IS DISTINCT FROM 'HR';     -- TRUE  (NULL and 'HR' are different)
SELECT 'HR' IS DISTINCT FROM 'HR';     -- FALSE (same value)
SELECT 'HR' IS DISTINCT FROM 'IT';     -- TRUE  (different values)

-- Practical use: find rows where department changed (including NULL changes)
SELECT * FROM employees e
JOIN employees_history h ON e.id = h.id
WHERE e.department IS DISTINCT FROM h.department;
-- Catches: 'HR' → 'IT', NULL → 'HR', 'HR' → NULL, NULL → NULL (no change)
```

---

## <span style="color:#1565C0">5.9 DISTINCT - Removing Duplicates</span>

> **Definition:** `DISTINCT` removes duplicate rows from the result set, returning only unique rows. It applies to the entire row (all selected columns combined).

---

### <span style="color:#2E86AB">SELECT DISTINCT</span>

```sql
-- All unique departments (including NULL as one unique value)
SELECT DISTINCT department
FROM employees
ORDER BY department;
-- Engineering, Finance, HR, Marketing, NULL

-- DISTINCT on multiple columns - unique combinations
SELECT DISTINCT department, is_active
FROM employees
ORDER BY department;
-- Engineering/TRUE, Finance/FALSE, Finance/TRUE, HR/TRUE, Marketing/FALSE, Marketing/TRUE, NULL/TRUE

-- Without DISTINCT (shows duplicates)
SELECT department FROM employees ORDER BY department;
-- Engineering, Engineering, Engineering, Engineering, Engineering, Finance, Finance, HR, HR, Marketing, Marketing, NULL
```

---

### <span style="color:#2E86AB">DISTINCT vs DISTINCT ON (PostgreSQL-specific)</span>

> **Definition:** `DISTINCT ON (expression)` is a PostgreSQL extension that returns one row per unique value of the specified expression - the first row encountered within each group (controlled by `ORDER BY`). It is a concise alternative to window functions for "first row per group" queries.

```sql
-- Standard DISTINCT - unique combinations of ALL selected columns
SELECT DISTINCT department, salary FROM employees;
-- Returns every unique (department, salary) pair

-- DISTINCT ON - one row per department, picking the highest-paid employee
SELECT DISTINCT ON (department)
  department,
  first_name,
  salary
FROM employees
ORDER BY department, salary DESC;
-- For each department, returns the row with the highest salary
-- Engineering → Leo (112000)
-- Finance     → Grace (91000)
-- HR          → Carol (62000)
-- Marketing   → Jay (72000)
-- NULL        → Kim (55000)
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 580px;">

### <span style="color:#D4A017">DISTINCT ON rule: The ORDER BY must begin with the same column(s) used in DISTINCT ON. The first row in each group (per ORDER BY) is the one returned.</span>

</div>

```sql
-- DISTINCT ON syntax
SELECT DISTINCT ON (col1) col1, col2, col3
FROM table
ORDER BY col1, col2 DESC;
--        ^^^^  ← DISTINCT ON col must be first in ORDER BY

-- Earliest hire per department
SELECT DISTINCT ON (department)
  department, first_name, hired_on
FROM employees
ORDER BY department, hired_on ASC;
-- Returns the first person hired in each department
```

---

### <span style="color:#2E86AB">COUNT DISTINCT</span>

```sql
-- Count unique departments
SELECT COUNT(DISTINCT department) FROM employees;
-- Returns 4 (Engineering, Finance, HR, Marketing - NULL is excluded)

-- Count unique values in any column
SELECT COUNT(DISTINCT salary) FROM employees;
```

---

## <span style="color:#1565C0">5.10 ORDER BY</span>

> **Definition:** `ORDER BY` sorts the result rows. Without `ORDER BY`, the order of rows returned by PostgreSQL is **not guaranteed** - it can change between queries as data changes. Always use `ORDER BY` when row order matters.

---

### <span style="color:#2E86AB">ASC and DESC</span>

```sql
-- Ascending order (default - lowest to highest, A to Z, oldest to newest)
SELECT first_name, salary FROM employees ORDER BY salary ASC;
SELECT first_name, salary FROM employees ORDER BY salary;  -- ASC is the default

-- Descending order (highest to lowest, Z to A, newest to oldest)
SELECT first_name, salary FROM employees ORDER BY salary DESC;

-- Sort by text column
SELECT first_name FROM employees ORDER BY first_name ASC;   -- alphabetical A→Z
SELECT first_name FROM employees ORDER BY first_name DESC;  -- Z→A

-- Sort by date
SELECT first_name, hired_on FROM employees ORDER BY hired_on ASC;   -- oldest first
SELECT first_name, hired_on FROM employees ORDER BY hired_on DESC;  -- newest first
```

---

### <span style="color:#2E86AB">Multi-Column ORDER BY</span>

```sql
-- Sort by department first (A→Z), then by salary within each department (high→low)
SELECT first_name, department, salary
FROM employees
ORDER BY department ASC, salary DESC;

-- Sort by active status (TRUE first), then by name
SELECT first_name, is_active
FROM employees
ORDER BY is_active DESC, first_name ASC;
-- Active employees (TRUE) first, alphabetical within each group

-- Three-level sort
SELECT department, is_active, first_name, salary
FROM employees
ORDER BY department ASC, is_active DESC, salary DESC;

-- Mix of column name and expression
SELECT first_name, salary
FROM employees
ORDER BY LENGTH(first_name) ASC, salary DESC;
-- Shortest names first; same-length names sorted by salary descending
```

---

### <span style="color:#2E86AB">ORDER BY Column Position</span>

```sql
-- Reference output columns by position (1-based)
SELECT department, salary, first_name
FROM employees
ORDER BY 1, 2 DESC;
-- ORDER BY department ASC, salary DESC

-- Useful when the expression is long
SELECT department, AVG(salary), COUNT(*)
FROM employees
GROUP BY department
ORDER BY 2 DESC;  -- sort by AVG(salary) descending
-- Cleaner than repeating AVG(salary)
```

---

### <span style="color:#2E86AB">NULLS FIRST / NULLS LAST</span>

> **Definition:** By default in PostgreSQL, `NULL` values sort after all non-NULL values in `ASC` order, and before all non-NULL values in `DESC` order. You can override this with `NULLS FIRST` and `NULLS LAST`.

```sql
-- Default NULL behaviour
SELECT first_name, department FROM employees ORDER BY department ASC;
-- Engineering, Engineering, Finance, Finance, HR, HR, Marketing, Marketing, NULL ← NULL last by default in ASC

SELECT first_name, department FROM employees ORDER BY department DESC;
-- NULL ← NULL first by default in DESC, then Marketing, Marketing, HR, HR...

-- Explicit control
SELECT first_name, department FROM employees
ORDER BY department ASC NULLS FIRST;
-- NULL first, then alphabetical

SELECT first_name, department FROM employees
ORDER BY department ASC NULLS LAST;
-- Alphabetical, NULL at very end

SELECT first_name, department FROM employees
ORDER BY department DESC NULLS LAST;
-- Reverse alphabetical, NULL at very end (not the default!)
```

| Sort Direction | Default NULL position | To change |
|:---|:---|:---|
| `ASC` | NULLS LAST | Use `ASC NULLS FIRST` |
| `DESC` | NULLS FIRST | Use `DESC NULLS LAST` |

---

### <span style="color:#2E86AB">ORDER BY Expression</span>

```sql
-- Sort by computed value
SELECT first_name, salary
FROM employees
ORDER BY salary * 0.82 DESC;  -- sort by after-tax salary

-- Sort by string length
SELECT first_name FROM employees ORDER BY LENGTH(first_name);

-- Sort by extracted date part
SELECT first_name, hired_on
FROM employees
ORDER BY EXTRACT(MONTH FROM hired_on), first_name;
-- Group by hire month, then alphabetical within each month

-- Random order (useful for sampling or shuffling)
SELECT * FROM employees ORDER BY RANDOM() LIMIT 3;
-- Returns 3 random employees
```

---

## <span style="color:#1565C0">5.11 LIMIT & OFFSET - Pagination</span>

> **Definition:** `LIMIT` restricts the number of rows returned. `OFFSET` skips a specified number of rows before starting to return rows. Together they implement pagination.

---

### <span style="color:#2E86AB">LIMIT</span>

```sql
-- Return only the first 5 rows
SELECT * FROM employees ORDER BY id LIMIT 5;

-- Return the top 3 highest-paid employees
SELECT first_name, salary FROM employees ORDER BY salary DESC LIMIT 3;
-- Leo (112000), Frank (105000), Alice (95000)

-- LIMIT 1 - get a single row (most common with ORDER BY)
SELECT first_name, salary FROM employees ORDER BY salary DESC LIMIT 1;
-- Leo - the highest-paid employee

-- LIMIT ALL - same as no limit (returns all rows)
SELECT * FROM employees ORDER BY id LIMIT ALL;

-- LIMIT 0 - returns no rows but includes column headers (useful for structure inspection)
SELECT * FROM employees LIMIT 0;
```

---

### <span style="color:#2E86AB">OFFSET</span>

```sql
-- Skip the first 5 rows, return the next rows
SELECT * FROM employees ORDER BY id OFFSET 5;
-- Returns rows 6, 7, 8, 9, 10, 11, 12

-- Skip 0 rows (default behaviour - same as no OFFSET)
SELECT * FROM employees ORDER BY id OFFSET 0;
```

---

### <span style="color:#2E86AB">LIMIT + OFFSET Together - Pagination</span>

```sql
-- Page 1: rows 1-4   (OFFSET 0, LIMIT 4)
SELECT first_name, salary FROM employees ORDER BY id LIMIT 4 OFFSET 0;

-- Page 2: rows 5-8   (OFFSET 4, LIMIT 4)
SELECT first_name, salary FROM employees ORDER BY id LIMIT 4 OFFSET 4;

-- Page 3: rows 9-12  (OFFSET 8, LIMIT 4)
SELECT first_name, salary FROM employees ORDER BY id LIMIT 4 OFFSET 8;

-- General formula:
-- OFFSET = (page_number - 1) * page_size
-- LIMIT  = page_size

-- Example: page 5, showing 10 items per page
SELECT * FROM employees
ORDER BY id
LIMIT 10 OFFSET 40;   -- (5-1) * 10 = 40
```

---

### <span style="color:#2E86AB">Pagination Performance Considerations</span>

```sql
-- OFFSET gets slower as it grows - PostgreSQL must scan and discard all skipped rows
-- OFFSET 1000000 LIMIT 10 scans 1,000,010 rows just to return 10

-- Keyset / cursor pagination - better alternative for large datasets
-- Instead of OFFSET, remember where you left off using the last seen id

-- Page 1
SELECT * FROM employees ORDER BY id LIMIT 10;
-- Last id returned: 10

-- Page 2 - use the last seen id as the filter
SELECT * FROM employees WHERE id > 10 ORDER BY id LIMIT 10;
-- Last id returned: 20

-- Page 3
SELECT * FROM employees WHERE id > 20 ORDER BY id LIMIT 10;

-- This always scans from the "cursor" position - stays fast even at deep pages
-- Works best with a unique, indexed, ordered column (id, created_at, etc.)
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 580px;">

### <span style="color:#D4A017">OFFSET-based pagination degrades as page numbers grow. For user-facing pagination on large tables, prefer keyset/cursor pagination using WHERE id > last_seen_id LIMIT n.</span>

</div>

---

## <span style="color:#1565C0">5.12 FETCH - SQL Standard Alternative to LIMIT</span>

> **Definition:** `FETCH` is the SQL standard (ISO/IEC) syntax for limiting rows. It is equivalent to `LIMIT` / `OFFSET` but uses a more verbose, standards-compliant form. PostgreSQL supports both.

---

### <span style="color:#2E86AB">FETCH Syntax</span>

```sql
SELECT ...
FROM ...
ORDER BY ...
OFFSET n ROWS
FETCH FIRST m ROWS ONLY;
```

```sql
-- FETCH equivalent of LIMIT 5
SELECT first_name, salary
FROM employees
ORDER BY salary DESC
FETCH FIRST 5 ROWS ONLY;

-- FETCH with OFFSET (page 2, 4 items per page)
SELECT first_name, salary
FROM employees
ORDER BY salary DESC
OFFSET 4 ROWS
FETCH NEXT 4 ROWS ONLY;
-- NEXT and FIRST are interchangeable

-- FETCH FIRST 1 ROW ONLY
SELECT first_name, salary
FROM employees
ORDER BY salary DESC
FETCH FIRST 1 ROW ONLY;   -- ROW (singular) also valid

-- FETCH ALL - all rows (like LIMIT ALL)
SELECT * FROM employees ORDER BY id FETCH FIRST ALL ROWS ONLY;
```

---

### <span style="color:#2E86AB">LIMIT vs FETCH Comparison</span>

| Feature | `LIMIT / OFFSET` | `FETCH / OFFSET` |
|:---|:---|:---|
| PostgreSQL support | ✓ | ✓ |
| SQL standard | ✗ | ✓ |
| Brevity | More concise | More verbose |
| `OFFSET` position | After `LIMIT` | Before `FETCH` |
| Portability | PostgreSQL, MySQL, SQLite | Oracle, SQL Server, DB2, PostgreSQL |

```sql
-- These are exactly equivalent:
SELECT * FROM employees ORDER BY id LIMIT 5 OFFSET 10;
SELECT * FROM employees ORDER BY id OFFSET 10 ROWS FETCH NEXT 5 ROWS ONLY;
```

---

## <span style="color:#1565C0">5.13 Set Operations</span>

> **Definition:** Set operations combine the results of two or more `SELECT` queries into a single result set. The queries being combined must have the same number of columns, and corresponding columns must have compatible data types.

---

### <span style="color:#2E86AB">Rules for All Set Operations</span>

```
Rule 1: Same number of columns in both SELECT statements
Rule 2: Corresponding columns must be compatible types
Rule 3: Column names in the result come from the FIRST query
Rule 4: ORDER BY applies to the final combined result (goes at the very end)
```

```sql
-- Setup: two query shapes that are compatible
-- Query A: Engineering employees (id, name, department)
-- Query B: Finance employees (id, name, department)
-- Both have 3 columns of types INTEGER, TEXT, TEXT → compatible ✓
```

---

### <span style="color:#2E86AB">UNION vs UNION ALL</span>

> **Definition:** `UNION` combines results of two queries and **removes duplicate rows** from the final set. `UNION ALL` combines results and **keeps all rows including duplicates**. `UNION ALL` is faster since it skips deduplication.

```sql
-- UNION - deduplicates the combined result
SELECT first_name, department FROM employees WHERE department = 'Engineering'
UNION
SELECT first_name, department FROM employees WHERE salary > 80000;
-- If Alice is in Engineering AND has salary > 80000, she appears ONCE

-- UNION ALL - keeps all rows including duplicates
SELECT first_name, department FROM employees WHERE department = 'Engineering'
UNION ALL
SELECT first_name, department FROM employees WHERE salary > 80000;
-- Alice would appear TWICE (once from each query)

-- Practical UNION: combine active and archived employees into one list
SELECT id, first_name, 'active' AS status FROM employees WHERE is_active = TRUE
UNION ALL
SELECT id, first_name, 'inactive' AS status FROM employees WHERE is_active = FALSE;

-- UNION with different tables
SELECT first_name AS name, 'Employee' AS type FROM employees
UNION ALL
SELECT name, 'Department' AS type FROM departments;
-- Combines employee names and department names into one column

-- UNION with ORDER BY - ORDER BY goes at the very end, after all queries
SELECT first_name, salary FROM employees WHERE department = 'Engineering'
UNION ALL
SELECT first_name, salary FROM employees WHERE department = 'Finance'
ORDER BY salary DESC;   -- sorts the combined result
```

---

### <span style="color:#2E86AB">INTERSECT</span>

> **Definition:** `INTERSECT` returns only the rows that appear in **both** result sets. It is the SQL equivalent of a set intersection. Like `UNION`, it deduplicates by default.

```sql
-- INTERSECT: rows that appear in BOTH queries
SELECT department FROM employees WHERE salary > 80000
INTERSECT
SELECT department FROM employees WHERE is_active = TRUE;
-- Departments that have BOTH: employees earning > 80k AND active employees

-- Example: employees who are both high earners AND recently hired
SELECT first_name FROM employees WHERE salary > 80000
INTERSECT
SELECT first_name FROM employees WHERE hired_on > '2020-01-01';
-- Returns employees who match BOTH conditions
-- (though a single WHERE with AND would be simpler for this case)

-- INTERSECT ALL - keep duplicates
SELECT first_name FROM employees WHERE salary > 80000
INTERSECT ALL
SELECT first_name FROM employees WHERE is_active = TRUE;
```

---

### <span style="color:#2E86AB">EXCEPT</span>

> **Definition:** `EXCEPT` returns rows from the **first** query that do **not** appear in the **second** query. It is the SQL equivalent of set difference. `EXCEPT ALL` keeps duplicates.

```sql
-- EXCEPT: rows in first query but NOT in second
SELECT department FROM employees  -- all departments
EXCEPT
SELECT department FROM employees WHERE salary > 90000;
-- Departments that have NO employee earning over 90000

-- Employees in Engineering who are NOT high earners
SELECT first_name FROM employees WHERE department = 'Engineering'
EXCEPT
SELECT first_name FROM employees WHERE salary > 100000;
-- Returns Engineering employees who earn <= 100000

-- EXCEPT ALL - keeps duplicates from first query not matched in second
SELECT first_name FROM employees WHERE department = 'Engineering'
EXCEPT ALL
SELECT first_name FROM employees WHERE salary > 100000;

-- Real-world: find users who signed up but never placed an order
SELECT user_id FROM signups
EXCEPT
SELECT DISTINCT user_id FROM orders;

-- EXCEPT is directional - order matters
SELECT 'A' UNION SELECT 'B' EXCEPT SELECT 'B';  -- result: A
SELECT 'B' EXCEPT SELECT 'A' UNION SELECT 'B';  -- result: B, B (UNION after EXCEPT)
```

---

### <span style="color:#2E86AB">Set Operation Summary</span>

| Operation | Returns | Deduplicates |
|:---|:---|:---:|
| `UNION` | All rows from both queries | ✓ |
| `UNION ALL` | All rows from both queries | ✗ |
| `INTERSECT` | Rows in **both** queries | ✓ |
| `INTERSECT ALL` | Rows in **both** queries | ✗ |
| `EXCEPT` | Rows in **first** but NOT in second | ✓ |
| `EXCEPT ALL` | Rows in **first** but NOT in second | ✗ |

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 580px;">

### <span style="color:#D4A017">Prefer UNION ALL over UNION whenever you know there are no duplicates or don't care about them. UNION performs an implicit DISTINCT which requires a sort or hash - UNION ALL just concatenates and is always faster.</span>

</div>

---

## <span style="color:#1565C0">5.14 Expressions in SELECT</span>

> **Definition:** The `SELECT` clause can contain any valid expression - arithmetic, string operations, date functions, conditional logic, or subqueries. These produce computed columns in the result.

---

### <span style="color:#2E86AB">Arithmetic Expressions</span>

```sql
SELECT
  first_name,
  salary,
  salary / 12                        AS monthly,
  salary * 12                        AS check_annual,   -- same as salary for NUMERIC
  salary * 1.10                      AS after_10pct_raise,
  ROUND(salary * 0.18, 2)            AS tax,
  salary - ROUND(salary * 0.18, 2)   AS net_salary,
  salary * 1.10 - salary             AS raise_amount
FROM employees;

-- Integer division vs float division
SELECT 7 / 2;           -- 3  (integer division - truncates)
SELECT 7.0 / 2;         -- 3.5
SELECT 7::FLOAT / 2;    -- 3.5
SELECT 7 / 2.0;         -- 3.5

-- Modulo
SELECT 17 % 5;          -- 2 (remainder)
SELECT MOD(17, 5);      -- 2

-- Power and square root
SELECT POWER(2, 10);    -- 1024
SELECT SQRT(144);       -- 12
SELECT 2 ^ 10;          -- 1024 (PostgreSQL-specific exponent operator)
```

---

### <span style="color:#2E86AB">String Concatenation Expressions</span>

```sql
SELECT
  first_name || ' ' || last_name          AS full_name,
  UPPER(first_name) || ' ' || UPPER(last_name) AS upper_name,
  INITCAP(first_name || ' ' || last_name) AS proper_name,
  first_name || ' (' || department || ')' AS name_with_dept,
  LEFT(first_name, 1) || '.' || LEFT(last_name, 1) || '.' AS initials,
  department || ' - ' || salary::TEXT     AS dept_salary_label
FROM employees;

-- CONCAT function (NULL-safe - ignores NULLs, || with NULL returns NULL)
SELECT CONCAT(first_name, ' ', last_name)     AS full_name FROM employees;
SELECT first_name || ' ' || last_name         AS full_name FROM employees;
-- Kim has department = NULL:
SELECT first_name || ' - ' || department      FROM employees WHERE first_name = 'Kim';
-- Returns NULL (because NULL || anything = NULL)
SELECT CONCAT(first_name, ' - ', department)  FROM employees WHERE first_name = 'Kim';
-- Returns 'Kim - ' (CONCAT skips NULLs)

-- COALESCE to handle NULL in concatenation
SELECT first_name || ' - ' || COALESCE(department, 'No Dept') AS label
FROM employees;
```

---

### <span style="color:#2E86AB">CASE Expression</span>

> **Definition:** `CASE` is a conditional expression that returns different values based on conditions. It is the SQL equivalent of if-else or switch. It can appear anywhere an expression is valid - `SELECT`, `WHERE`, `ORDER BY`, `GROUP BY`.

#### <span style="color:#5B8DB8">Searched CASE (most common)</span>

```sql
-- Salary band classification
SELECT
  first_name,
  salary,
  CASE
    WHEN salary >= 100000 THEN 'Senior'
    WHEN salary >=  80000 THEN 'Mid-Senior'
    WHEN salary >=  65000 THEN 'Mid-Level'
    ELSE 'Junior'
  END AS seniority_band
FROM employees;

-- Department label
SELECT
  first_name,
  department,
  CASE
    WHEN department IN ('Engineering', 'Product') THEN 'Tech'
    WHEN department IN ('HR', 'Finance')           THEN 'Support'
    WHEN department = 'Marketing'                  THEN 'Growth'
    WHEN department IS NULL                        THEN 'Unassigned'
    ELSE 'Other'
  END AS dept_group
FROM employees;

-- CASE in WHERE (conditional filtering)
SELECT first_name, salary, department
FROM employees
WHERE CASE
  WHEN department = 'Engineering' THEN salary > 90000
  WHEN department = 'HR'          THEN salary > 60000
  ELSE salary > 70000
END;
-- Different salary threshold per department
```

#### <span style="color:#5B8DB8">Simple CASE (switch-style)</span>

```sql
-- Switch on a specific value
SELECT
  first_name,
  department,
  CASE department
    WHEN 'Engineering' THEN 'Tech Team'
    WHEN 'HR'          THEN 'People Team'
    WHEN 'Finance'     THEN 'Money Team'
    WHEN 'Marketing'   THEN 'Growth Team'
    ELSE 'Other'
  END AS team_name
FROM employees;
```

#### <span style="color:#5B8DB8">CASE in ORDER BY (custom sort)</span>

```sql
-- Custom sort: Engineering first, then HR, then Finance, then everything else
SELECT first_name, department
FROM employees
ORDER BY
  CASE department
    WHEN 'Engineering' THEN 1
    WHEN 'HR'          THEN 2
    WHEN 'Finance'     THEN 3
    ELSE 4
  END,
  first_name ASC;
```

---

### <span style="color:#2E86AB">Date Expressions</span>

```sql
SELECT
  first_name,
  hired_on,
  CURRENT_DATE                              AS today,
  CURRENT_DATE - hired_on                   AS days_employed,
  EXTRACT(YEAR FROM AGE(hired_on))::INT     AS years_employed,
  EXTRACT(MONTH FROM hired_on)              AS hire_month,
  TO_CHAR(hired_on, 'Month DD, YYYY')       AS formatted_date,
  DATE_TRUNC('year', hired_on)::DATE        AS hire_year_start,
  hired_on + INTERVAL '1 year'              AS first_anniversary
FROM employees;
```

---

## <span style="color:#1565C0">5.15 Table Aliases</span>

> **Definition:** A table alias is a short, temporary name given to a table within a query. Table aliases simplify long table names, are essential when joining a table to itself, and are required when the same table is used more than once in a query.

---

### <span style="color:#2E86AB">Basic Table Alias Syntax</span>

```sql
-- Full syntax with AS keyword
SELECT e.first_name, e.salary
FROM employees AS e
WHERE e.salary > 80000;

-- Short syntax without AS (more common in practice)
SELECT e.first_name, e.salary
FROM employees e
WHERE e.salary > 80000;

-- Table alias qualifies column names: alias.column_name
SELECT
  e.first_name,
  e.last_name,
  e.department,
  e.salary
FROM employees e
WHERE e.is_active = TRUE
ORDER BY e.salary DESC;
```

---

### <span style="color:#2E86AB">Why Table Aliases Are Important</span>

```sql
-- 1. Avoiding ambiguity when column names exist in multiple tables
SELECT e.first_name, d.name AS dept_name
FROM employees e
JOIN departments d ON e.department = d.name;
-- Without aliases, 'name' would be ambiguous - which table's 'name'?

-- 2. Self-join - must have two aliases for the same table
SELECT
  e.first_name    AS employee,
  mgr.first_name  AS manager
FROM employees e
JOIN employees mgr ON e.manager_id = mgr.id;
-- Without aliases, PostgreSQL can't tell which "employees" is which

-- 3. Subquery alias - required for all subqueries in FROM
SELECT avg_data.department, avg_data.avg_salary
FROM (
  SELECT department, AVG(salary) AS avg_salary
  FROM employees
  GROUP BY department
) AS avg_data           -- subquery MUST have an alias
WHERE avg_data.avg_salary > 80000;

-- 4. Shortening long table or schema-qualified names
SELECT p.id, p.name
FROM public.products_catalog_v2 p
WHERE p.is_available = TRUE;
```

---

### <span style="color:#2E86AB">Alias Scope Rules</span>

```sql
-- Table alias is valid for the entire query after it is defined
SELECT e.first_name
FROM employees e                -- alias 'e' defined here
WHERE e.salary > 80000          -- valid use
ORDER BY e.last_name;           -- valid use

-- Once you alias a table, you MUST use the alias to qualify columns
-- (you can no longer use the original table name)
SELECT employees.first_name     -- ERROR: table reference 'employees' not valid
FROM employees e;               -- must use 'e.first_name'
```

---

### <span style="color:#2E86AB">Column Aliases vs Table Aliases</span>

| Alias Type | Syntax | Purpose | Scope |
|:---|:---|:---|:---|
| Column alias | `SELECT col AS alias` | Rename output column | Available in ORDER BY only |
| Table alias | `FROM table AS t` | Rename table reference | Entire query |

---

## <span style="color:#1565C0">Phase 5 - Quick Reference Card</span>

### <span style="color:#2E86AB">WHERE Operator Cheatsheet</span>

```sql
WHERE col = val                  -- equal
WHERE col <> val                 -- not equal
WHERE col != val                 -- not equal (same as <>)
WHERE col BETWEEN low AND high   -- inclusive range
WHERE col NOT BETWEEN low AND high
WHERE col IN (v1, v2, v3)        -- matches any value in list
WHERE col NOT IN (v1, v2)        -- matches none  ← watch NULL trap
WHERE col LIKE 'pat%'            -- case-sensitive pattern
WHERE col ILIKE 'pat%'           -- case-insensitive pattern (PostgreSQL)
WHERE col IS NULL                -- is NULL
WHERE col IS NOT NULL            -- is not NULL
WHERE col IS DISTINCT FROM val   -- NULL-safe not-equal
WHERE expr AND expr              -- both must be true
WHERE expr OR expr               -- at least one must be true
WHERE NOT expr                   -- inverts the condition
```

### <span style="color:#2E86AB">LIKE / ILIKE Wildcard Summary</span>

| Pattern | Meaning |
|:---|:---|
| `'A%'` | Starts with A |
| `'%z'` | Ends with z |
| `'%abc%'` | Contains abc anywhere |
| `'_bc'` | Any one char, then bc |
| `'A_C'` | A, any one char, C |
| `'___'` | Exactly 3 characters |

### <span style="color:#2E86AB">ORDER BY Quick Patterns</span>

```sql
ORDER BY col ASC              -- smallest first (default)
ORDER BY col DESC             -- largest first
ORDER BY col ASC NULLS FIRST  -- NULLs before real values
ORDER BY col ASC NULLS LAST   -- NULLs after real values (ASC default)
ORDER BY col DESC NULLS LAST  -- largest first, NULLs last
ORDER BY col1 ASC, col2 DESC  -- multi-level sort
ORDER BY RANDOM()             -- shuffle rows
```

### <span style="color:#2E86AB">Pagination Pattern</span>

```sql
-- OFFSET-based (simple, gets slow at high pages)
SELECT * FROM t ORDER BY id LIMIT :page_size OFFSET (:page - 1) * :page_size;

-- Keyset-based (fast at any page depth)
SELECT * FROM t WHERE id > :last_seen_id ORDER BY id LIMIT :page_size;
```

### <span style="color:#2E86AB">Set Operations Summary</span>

```sql
SELECT ... UNION     SELECT ...  -- combined, deduped
SELECT ... UNION ALL SELECT ...  -- combined, with duplicates (faster)
SELECT ... INTERSECT SELECT ...  -- rows in both
SELECT ... EXCEPT    SELECT ...  -- rows in first, not in second
```

### <span style="color:#2E86AB">CASE Template</span>

```sql
-- Searched CASE
CASE
  WHEN condition1 THEN result1
  WHEN condition2 THEN result2
  ELSE default_result
END

-- Simple CASE
CASE column
  WHEN val1 THEN result1
  WHEN val2 THEN result2
  ELSE default_result
END
```