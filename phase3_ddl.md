
<div align="center">

# <span style="color:#0A2FA8">PostgreSQL : Phase 3</span>

<sub>DDL - Data Definition Language</sub>

</div>

---

## <span style="color:#1565C0">3.1 CREATE TABLE - Full Syntax</span>

> **Definition:** `CREATE TABLE` is the DDL command used to define a new table - its name, columns, data types, and constraints. It is the foundation of a relational database schema.

---

### <span style="color:#2E86AB">Basic Syntax</span>

```sql
CREATE TABLE table_name (
  column_name  data_type  [column_constraints],
  column_name  data_type  [column_constraints],
  ...
  [table_constraints]
);
```

---

### <span style="color:#2E86AB">Simple Table Example</span>

```sql
CREATE TABLE users (
  id         SERIAL PRIMARY KEY,
  name       VARCHAR(100) NOT NULL,
  email      VARCHAR(255) UNIQUE NOT NULL,
  age        SMALLINT CHECK (age >= 0 AND age <= 150),
  is_active  BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

### <span style="color:#2E86AB">CREATE TABLE IF NOT EXISTS</span>

```sql
-- Does nothing (no error) if the table already exists
CREATE TABLE IF NOT EXISTS users (
  id   SERIAL PRIMARY KEY,
  name TEXT NOT NULL
);
```

---

### <span style="color:#2E86AB">CREATE TABLE AS - From a Query</span>

> **Definition:** `CREATE TABLE AS` creates a new table and populates it with the result of a SELECT query. The table structure (column names and types) is inferred from the query.

```sql
-- Create a table from a query result
CREATE TABLE active_users AS
SELECT id, name, email
FROM users
WHERE is_active = TRUE;

-- Create with no rows (structure only)
CREATE TABLE users_backup AS
SELECT * FROM users WHERE FALSE;

-- WITH NO DATA - explicit syntax
CREATE TABLE users_backup AS
TABLE users
WITH NO DATA;
```

---

### <span style="color:#2E86AB">CREATE TABLE LIKE - Copy Structure</span>

> **Definition:** `CREATE TABLE ... LIKE` creates a new empty table with the same column definitions as an existing table. You can optionally include constraints, defaults, indexes, etc.

```sql
-- Copy column structure only
CREATE TABLE users_archive (LIKE users);

-- Copy with defaults and constraints
CREATE TABLE users_archive (
  LIKE users INCLUDING DEFAULTS INCLUDING CONSTRAINTS
);

-- Include everything
CREATE TABLE users_archive (
  LIKE users INCLUDING ALL
);

-- Exclude specific things
CREATE TABLE users_archive (
  LIKE users INCLUDING ALL EXCLUDING INDEXES
);
```

| Option | What it copies |
|:---|:---|
| `INCLUDING DEFAULTS` | Column `DEFAULT` values |
| `INCLUDING CONSTRAINTS` | `CHECK`, `NOT NULL` constraints |
| `INCLUDING INDEXES` | All indexes including PRIMARY KEY |
| `INCLUDING STORAGE` | Storage parameters |
| `INCLUDING COMMENTS` | Column/table comments |
| `INCLUDING ALL` | Everything above |

---

### <span style="color:#2E86AB">Full Column Definition Syntax</span>

```sql
column_name  data_type
  [DEFAULT expr]
  [NOT NULL | NULL]
  [CHECK (condition)]
  [UNIQUE]
  [PRIMARY KEY]
  [REFERENCES other_table (other_col)
    [ON DELETE action]
    [ON UPDATE action]
  ]
```

---

## <span style="color:#1565C0">3.2 Column Constraints</span>

> **Definition:** Column constraints are rules applied to individual columns to enforce data integrity. They are declared inline with the column definition.

---

### <span style="color:#2E86AB">NOT NULL</span>

> **Definition:** `NOT NULL` ensures that a column cannot store a NULL value. Every INSERT or UPDATE on that column must provide a non-NULL value.

```sql
CREATE TABLE employees (
  id         SERIAL PRIMARY KEY,
  first_name TEXT NOT NULL,       -- required
  last_name  TEXT NOT NULL,       -- required
  email      TEXT NOT NULL,       -- required
  nickname   TEXT                 -- optional (NULL allowed)
);

-- This INSERT fails:
INSERT INTO employees (first_name, last_name)
VALUES (NULL, 'Smith');
-- ERROR: null value in column "first_name" violates not-null constraint

-- This is fine:
INSERT INTO employees (first_name, last_name, email)
VALUES ('John', 'Smith', 'john@example.com');
-- nickname defaults to NULL - that is allowed
```

---

### <span style="color:#2E86AB">UNIQUE</span>

> **Definition:** `UNIQUE` ensures all values in a column are distinct across all rows. Duplicate values are rejected. Unlike `PRIMARY KEY`, a `UNIQUE` column can contain NULL (and multiple NULLs are allowed, since NULL ≠ NULL).

```sql
CREATE TABLE users (
  id       SERIAL PRIMARY KEY,
  email    TEXT UNIQUE NOT NULL,
  username TEXT UNIQUE NOT NULL,
  phone    TEXT UNIQUE          -- NULLs allowed - multiple users can have no phone
);

-- Duplicate email fails:
INSERT INTO users (email, username) VALUES ('a@b.com', 'alice');
INSERT INTO users (email, username) VALUES ('a@b.com', 'bob');
-- ERROR: duplicate key value violates unique constraint

-- Multiple NULLs in a UNIQUE column are allowed:
INSERT INTO users (email, username, phone) VALUES ('x@x.com', 'x', NULL);
INSERT INTO users (email, username, phone) VALUES ('y@y.com', 'y', NULL);
-- Both succeed - NULL is not considered a duplicate
```

---

### <span style="color:#2E86AB">DEFAULT</span>

> **Definition:** `DEFAULT` specifies a value that is automatically used when no value is provided for a column during INSERT.

```sql
CREATE TABLE orders (
  id           SERIAL PRIMARY KEY,
  status       TEXT        DEFAULT 'pending',
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  updated_at   TIMESTAMPTZ DEFAULT NOW(),
  is_deleted   BOOLEAN     DEFAULT FALSE,
  retry_count  INTEGER     DEFAULT 0,
  priority     SMALLINT    DEFAULT 1,
  region       TEXT        DEFAULT 'IN'
);

-- INSERT without specifying defaulted columns
INSERT INTO orders DEFAULT VALUES;
-- All columns get their defaults

-- Explicit DEFAULT keyword in INSERT
INSERT INTO orders (status, priority)
VALUES (DEFAULT, 2);
-- status uses DEFAULT = 'pending', priority = 2

-- DEFAULT can use:
--   Literal values: DEFAULT 0, DEFAULT 'active', DEFAULT TRUE
--   Functions:      DEFAULT NOW(), DEFAULT gen_random_uuid()
--   Expressions:    DEFAULT CURRENT_DATE + INTERVAL '30 days'
```

---

### <span style="color:#2E86AB">CHECK</span>

> **Definition:** `CHECK` defines a boolean condition that every row must satisfy. Any INSERT or UPDATE that would violate the condition is rejected.

```sql
CREATE TABLE products (
  id           SERIAL PRIMARY KEY,
  name         TEXT NOT NULL,
  price        NUMERIC(10, 2) CHECK (price > 0),
  discount     NUMERIC(5, 2)  CHECK (discount >= 0 AND discount <= 100),
  stock        INTEGER        CHECK (stock >= 0),
  rating       NUMERIC(3, 2)  CHECK (rating BETWEEN 0 AND 5),
  size         TEXT           CHECK (size IN ('XS','S','M','L','XL','XXL'))
);

-- Valid insert
INSERT INTO products (name, price, discount, stock)
VALUES ('T-Shirt', 29.99, 10.00, 100);

-- Violates CHECK (price > 0)
INSERT INTO products (name, price) VALUES ('Freebie', -1);
-- ERROR: new row for relation "products" violates check constraint

-- Multi-column CHECK - must use table-level constraint (see 3.5)
CREATE TABLE events (
  id         SERIAL PRIMARY KEY,
  start_date DATE NOT NULL,
  end_date   DATE NOT NULL,
  CHECK (end_date >= start_date)   -- table-level: references both columns
);

-- CHECK with NULL - NULL passes any CHECK constraint by default
-- Because NULL makes the CHECK expression evaluate to NULL (unknown), not FALSE
INSERT INTO products (name, price, stock) VALUES ('Mystery', NULL, 10);
-- This PASSES even though price has CHECK (price > 0)!
-- Solution: combine with NOT NULL
CREATE TABLE products2 (
  price NUMERIC(10,2) NOT NULL CHECK (price > 0)
);
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 560px;">

### <span style="color:#D4A017">Important: NULL silently passes CHECK constraints. If a column must be non-null AND satisfy a condition, always combine NOT NULL with CHECK.</span>

</div>

---

## <span style="color:#1565C0">3.3 PRIMARY KEY</span>

> **Definition:** A `PRIMARY KEY` is a column (or combination of columns) that uniquely identifies each row in a table. It enforces both `UNIQUE` and `NOT NULL` simultaneously. Every table should have exactly one primary key.

---

### <span style="color:#2E86AB">Single Column Primary Key</span>

```sql
-- Inline column-level syntax (most common)
CREATE TABLE users (
  id    SERIAL PRIMARY KEY,
  name  TEXT NOT NULL
);

-- Table-level syntax (equivalent)
CREATE TABLE users (
  id    SERIAL,
  name  TEXT NOT NULL,
  PRIMARY KEY (id)
);

-- Using IDENTITY (modern, preferred)
CREATE TABLE users (
  id    INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name  TEXT NOT NULL
);

-- Using UUID as primary key
CREATE TABLE sessions (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id    INTEGER NOT NULL,
  started_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

### <span style="color:#2E86AB">Composite Primary Key</span>

> **Definition:** A composite (compound) primary key uses **two or more columns together** to uniquely identify a row. No single column is unique on its own - only the combination is unique.

```sql
-- Junction table for many-to-many relationship
CREATE TABLE student_courses (
  student_id  INTEGER NOT NULL,
  course_id   INTEGER NOT NULL,
  enrolled_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (student_id, course_id)  -- combination must be unique
);

-- This is valid (same student in different course):
INSERT INTO student_courses (student_id, course_id) VALUES (1, 10);
INSERT INTO student_courses (student_id, course_id) VALUES (1, 20);

-- This is valid (different student in same course):
INSERT INTO student_courses (student_id, course_id) VALUES (2, 10);

-- This fails (duplicate combination):
INSERT INTO student_courses (student_id, course_id) VALUES (1, 10);
-- ERROR: duplicate key value violates unique constraint

-- Another composite key example
CREATE TABLE order_items (
  order_id    INTEGER NOT NULL,
  product_id  INTEGER NOT NULL,
  quantity    INTEGER NOT NULL DEFAULT 1,
  unit_price  NUMERIC(10,2) NOT NULL,
  PRIMARY KEY (order_id, product_id)
);
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 560px;">

### <span style="color:#D4A017">A PRIMARY KEY automatically creates a UNIQUE index on that column(s). You do not need to add a separate UNIQUE or index - it is implied.</span>

</div>

---

### <span style="color:#2E86AB">Natural Key vs Surrogate Key</span>

| Type | Description | Example |
|:---|:---|:---|
| Natural Key | A real-world value that is inherently unique | `email`, `national_id`, `isbn` |
| Surrogate Key | An artificial ID generated just to be a PK | `SERIAL id`, `UUID id` |

```sql
-- Natural key (email as PK)
CREATE TABLE users (
  email  TEXT PRIMARY KEY,
  name   TEXT NOT NULL
);

-- Surrogate key (auto-increment id as PK - more common in practice)
CREATE TABLE users (
  id    SERIAL PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  name  TEXT NOT NULL
);
```

**Why surrogate keys are usually preferred:** Natural keys can change (email address changes), are often longer (performance), and create tight coupling between business logic and DB design.

---

## <span style="color:#1565C0">3.4 FOREIGN KEY</span>

> **Definition:** A `FOREIGN KEY` is a column (or set of columns) in one table that references the `PRIMARY KEY` (or `UNIQUE` column) of another table. It enforces **referential integrity** - you cannot insert a value in the FK column that doesn't exist in the referenced table.

---

### <span style="color:#2E86AB">Basic Foreign Key</span>

```sql
-- Parent table
CREATE TABLE customers (
  id    SERIAL PRIMARY KEY,
  name  TEXT NOT NULL
);

-- Child table with foreign key
CREATE TABLE orders (
  id          SERIAL PRIMARY KEY,
  customer_id INTEGER NOT NULL,
  total       NUMERIC(10,2),
  FOREIGN KEY (customer_id) REFERENCES customers(id)
);

-- Inline column-level syntax (shorthand)
CREATE TABLE orders (
  id          SERIAL PRIMARY KEY,
  customer_id INTEGER NOT NULL REFERENCES customers(id),
  total       NUMERIC(10,2)
);

-- What this enforces:
INSERT INTO orders (customer_id, total) VALUES (999, 100.00);
-- ERROR: insert or update on table "orders" violates foreign key constraint
-- Detail: Key (customer_id)=(999) is not present in table "customers"
```

---

### <span style="color:#2E86AB">ON DELETE Actions</span>

> **Definition:** `ON DELETE` specifies what happens to rows in the **child table** when the referenced row in the **parent table** is deleted.

| Action | Behaviour |
|:---|:---|
| `NO ACTION` | Default. Raises an error if child rows exist |
| `RESTRICT` | Same as NO ACTION, checked immediately |
| `CASCADE` | Automatically deletes child rows too |
| `SET NULL` | Sets the FK column to NULL in child rows |
| `SET DEFAULT` | Sets the FK column to its DEFAULT value in child rows |

```sql
-- ON DELETE CASCADE - most common
-- Delete the customer → all their orders are deleted automatically
CREATE TABLE orders (
  id          SERIAL PRIMARY KEY,
  customer_id INTEGER REFERENCES customers(id) ON DELETE CASCADE,
  total       NUMERIC(10,2)
);

DELETE FROM customers WHERE id = 1;
-- Automatically deletes all orders where customer_id = 1

-- ON DELETE SET NULL
-- Delete the customer → orders remain, but customer_id becomes NULL
CREATE TABLE orders (
  id          SERIAL PRIMARY KEY,
  customer_id INTEGER REFERENCES customers(id) ON DELETE SET NULL,
  total       NUMERIC(10,2)
);

DELETE FROM customers WHERE id = 1;
-- Orders still exist, customer_id = NULL

-- ON DELETE RESTRICT (strict - checked immediately, not at end of transaction)
CREATE TABLE orders (
  id          SERIAL PRIMARY KEY,
  customer_id INTEGER REFERENCES customers(id) ON DELETE RESTRICT,
  total       NUMERIC(10,2)
);

DELETE FROM customers WHERE id = 1;
-- ERROR: cannot delete a row because child rows exist in "orders"

-- ON DELETE SET DEFAULT
CREATE TABLE orders (
  id          SERIAL PRIMARY KEY,
  customer_id INTEGER DEFAULT 0 REFERENCES customers(id) ON DELETE SET DEFAULT,
  total       NUMERIC(10,2)
);
-- Requires customer with id=0 to exist in customers table
```

---

### <span style="color:#2E86AB">ON UPDATE Actions</span>

> **Definition:** `ON UPDATE` specifies what happens to FK columns in child rows when the referenced primary key value in the parent changes.

```sql
-- ON UPDATE CASCADE - keep child rows in sync with parent PK changes
CREATE TABLE orders (
  id          SERIAL PRIMARY KEY,
  customer_id INTEGER REFERENCES customers(id)
    ON DELETE CASCADE
    ON UPDATE CASCADE
);

-- Changing customer id propagates to all orders
UPDATE customers SET id = 99 WHERE id = 1;
-- All orders with customer_id = 1 now have customer_id = 99
```

| Combined Pattern | Meaning |
|:---|:---|
| `ON DELETE CASCADE ON UPDATE CASCADE` | Full sync - child follows parent |
| `ON DELETE SET NULL ON UPDATE CASCADE` | Keep rows, nullify if deleted |
| `ON DELETE RESTRICT ON UPDATE RESTRICT` | Strict referential integrity |
| `ON DELETE NO ACTION` | Default - error on delete with children |

---

### <span style="color:#2E86AB">DEFERRABLE Constraints</span>

> **Definition:** By default, foreign key checks happen immediately on each statement. `DEFERRABLE` constraints can be postponed until the end of a transaction - useful when inserting into both tables in the same transaction.

```sql
CREATE TABLE employees (
  id         SERIAL PRIMARY KEY,
  name       TEXT,
  manager_id INTEGER REFERENCES employees(id)
    DEFERRABLE INITIALLY DEFERRED  -- check at COMMIT, not on each statement
);

BEGIN;
  INSERT INTO employees (id, name, manager_id) VALUES (1, 'CEO', NULL);
  INSERT INTO employees (id, name, manager_id) VALUES (2, 'Manager', 1);
  -- No FK error even though row 1 was inserted before row 2 references it
COMMIT;
```

---

### <span style="color:#2E86AB">Self-Referencing Foreign Key</span>

```sql
-- A table that references itself (e.g. org hierarchy)
CREATE TABLE employees (
  id         SERIAL PRIMARY KEY,
  name       TEXT NOT NULL,
  manager_id INTEGER REFERENCES employees(id) ON DELETE SET NULL
);

-- CEO has no manager
INSERT INTO employees (name) VALUES ('Alice CEO');        -- id = 1

-- Reports to CEO
INSERT INTO employees (name, manager_id) VALUES ('Bob VP', 1);     -- id = 2
INSERT INTO employees (name, manager_id) VALUES ('Carol VP', 1);   -- id = 3

-- Reports to Bob
INSERT INTO employees (name, manager_id) VALUES ('Dave Mgr', 2);   -- id = 4
```

---

## <span style="color:#1565C0">3.5 Table-Level vs Column-Level Constraints</span>

> **Definition:** Constraints can be defined either **inline** with a column (column-level) or separately at the bottom of the `CREATE TABLE` statement (table-level). The difference matters when a constraint involves multiple columns.

---

### <span style="color:#2E86AB">Column-Level Constraints</span>

Defined immediately after the column's data type. Apply to that single column only.

```sql
CREATE TABLE users (
  id       SERIAL     PRIMARY KEY,           -- column-level PK
  email    TEXT       UNIQUE NOT NULL,       -- column-level UNIQUE + NOT NULL
  age      SMALLINT   CHECK (age > 0),       -- column-level CHECK
  role     TEXT       DEFAULT 'viewer'       -- column-level DEFAULT
);
```

---

### <span style="color:#2E86AB">Table-Level Constraints</span>

Defined after all columns. **Required** when a constraint spans multiple columns.

```sql
CREATE TABLE order_items (
  order_id    INTEGER NOT NULL,
  product_id  INTEGER NOT NULL,
  quantity    INTEGER NOT NULL CHECK (quantity > 0),
  unit_price  NUMERIC(10,2) NOT NULL CHECK (unit_price > 0),

  -- Table-level constraints (after all column definitions)
  PRIMARY KEY (order_id, product_id),         -- composite PK
  FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
  FOREIGN KEY (product_id) REFERENCES products(id),
  CHECK (quantity * unit_price < 1000000)     -- multi-column CHECK
);
```

---

### <span style="color:#2E86AB">Side-by-Side Comparison</span>

| Constraint Type | Column-Level | Table-Level |
|:---|:---:|:---:|
| `NOT NULL` | ✓ | ✗ (must be column-level) |
| `DEFAULT` | ✓ | ✗ (must be column-level) |
| `UNIQUE` (single col) | ✓ | ✓ |
| `PRIMARY KEY` (single col) | ✓ | ✓ |
| `CHECK` (single col) | ✓ | ✓ |
| `FOREIGN KEY` (single col) | ✓ | ✓ |
| Composite `PRIMARY KEY` | ✗ | ✓ (must be table-level) |
| Composite `UNIQUE` | ✗ | ✓ (must be table-level) |
| Multi-column `CHECK` | ✗ | ✓ (must be table-level) |
| Multi-column `FOREIGN KEY` | ✗ | ✓ (must be table-level) |

---

### <span style="color:#2E86AB">Composite UNIQUE Constraint</span>

```sql
-- Ensure (first_name + last_name) combination is unique
CREATE TABLE authors (
  id          SERIAL PRIMARY KEY,
  first_name  TEXT NOT NULL,
  last_name   TEXT NOT NULL,
  UNIQUE (first_name, last_name)    -- table-level composite unique
);

-- Same first name or same last name is fine - only combo must be unique
INSERT INTO authors (first_name, last_name) VALUES ('John', 'Smith');  -- OK
INSERT INTO authors (first_name, last_name) VALUES ('John', 'Doe');    -- OK (different last)
INSERT INTO authors (first_name, last_name) VALUES ('Jane', 'Smith');  -- OK (different first)
INSERT INTO authors (first_name, last_name) VALUES ('John', 'Smith');  -- ERROR: duplicate
```

---

## <span style="color:#1565C0">3.6 Naming Constraints Explicitly</span>

> **Definition:** By default, PostgreSQL auto-generates constraint names (e.g. `users_email_key`, `orders_pkey`). You can give constraints your own explicit names using `CONSTRAINT name` - which makes error messages clearer and makes it easier to `ALTER` or `DROP` specific constraints later.

---

### <span style="color:#2E86AB">CONSTRAINT Naming Syntax</span>

```sql
-- Column-level named constraint
CREATE TABLE users (
  id     SERIAL,
  email  TEXT,
  age    INTEGER,

  CONSTRAINT users_pkey         PRIMARY KEY (id),
  CONSTRAINT users_email_unique UNIQUE (email),
  CONSTRAINT users_age_positive CHECK (age > 0)
);

-- Inline named constraint (column-level)
CREATE TABLE users (
  id    SERIAL  CONSTRAINT users_pkey PRIMARY KEY,
  email TEXT    CONSTRAINT users_email_unique UNIQUE NOT NULL,
  age   INTEGER CONSTRAINT users_age_positive CHECK (age > 0)
);
```

---

### <span style="color:#2E86AB">Naming Foreign Keys</span>

```sql
CREATE TABLE orders (
  id          SERIAL,
  customer_id INTEGER NOT NULL,
  product_id  INTEGER NOT NULL,

  CONSTRAINT orders_pkey
    PRIMARY KEY (id),

  CONSTRAINT orders_customer_fk
    FOREIGN KEY (customer_id)
    REFERENCES customers(id)
    ON DELETE CASCADE,

  CONSTRAINT orders_product_fk
    FOREIGN KEY (product_id)
    REFERENCES products(id)
    ON DELETE RESTRICT
);
```

---

### <span style="color:#2E86AB">Why Explicit Names Matter</span>

```sql
-- Auto-named constraint error message (hard to understand):
-- ERROR: insert violates foreign key constraint "orders_customer_id_fkey"

-- vs named constraint error message (clear):
-- ERROR: insert violates foreign key constraint "orders_customer_fk"

-- Dropping a named constraint is straightforward:
ALTER TABLE orders DROP CONSTRAINT orders_customer_fk;

-- vs dropping an auto-named one (must look up the name first):
SELECT conname FROM pg_constraint WHERE conrelid = 'orders'::regclass;
ALTER TABLE orders DROP CONSTRAINT orders_customer_id_fkey;
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 560px;">

### <span style="color:#D4A017">Naming Convention: table_column_constraint. Examples: users_email_unique, orders_customer_fk, products_price_positive. Consistent names make migrations, error handling, and team collaboration much easier.</span>

</div>

---

## <span style="color:#1565C0">3.7 ALTER TABLE</span>

> **Definition:** `ALTER TABLE` modifies the structure of an existing table - adding/removing columns, changing types, adding/dropping constraints, and renaming. It is the primary DDL command for evolving your schema over time.

---

### <span style="color:#2E86AB">ADD COLUMN</span>

```sql
-- Add a simple column
ALTER TABLE users ADD COLUMN phone TEXT;

-- Add with constraint and default
ALTER TABLE users ADD COLUMN is_verified BOOLEAN DEFAULT FALSE NOT NULL;

-- Add with a foreign key
ALTER TABLE orders ADD COLUMN shipping_id INTEGER REFERENCES shippings(id);

-- Add multiple columns (separate statements)
ALTER TABLE products
  ADD COLUMN weight    NUMERIC(8,2),
  ADD COLUMN color     TEXT,
  ADD COLUMN in_stock  BOOLEAN DEFAULT TRUE;

-- Note: Adding a NOT NULL column to a table with existing data
-- Option 1: Add with DEFAULT first (populates existing rows), then remove default
ALTER TABLE users ADD COLUMN score INTEGER NOT NULL DEFAULT 0;
ALTER TABLE users ALTER COLUMN score DROP DEFAULT;

-- Option 2: Add as nullable, backfill data, then set NOT NULL
ALTER TABLE users ADD COLUMN score INTEGER;
UPDATE users SET score = 0 WHERE score IS NULL;
ALTER TABLE users ALTER COLUMN score SET NOT NULL;
```

---

### <span style="color:#2E86AB">DROP COLUMN</span>

```sql
-- Drop a column (irreversible!)
ALTER TABLE users DROP COLUMN phone;

-- Safe version - no error if column doesn't exist
ALTER TABLE users DROP COLUMN IF EXISTS phone;

-- CASCADE - also drops objects that depend on this column
-- (views, foreign keys, indexes that reference this column)
ALTER TABLE users DROP COLUMN phone CASCADE;

-- RESTRICT (default) - fails if anything depends on the column
ALTER TABLE users DROP COLUMN phone RESTRICT;
```

---

### <span style="color:#2E86AB">RENAME COLUMN</span>

```sql
-- Rename a column
ALTER TABLE users RENAME COLUMN username TO handle;

-- Rename in a specific schema
ALTER TABLE public.users RENAME COLUMN username TO handle;
```

---

### <span style="color:#2E86AB">ALTER COLUMN TYPE</span>

```sql
-- Change a column's data type
ALTER TABLE products ALTER COLUMN price TYPE NUMERIC(12, 2);

-- Change type with explicit CAST (when implicit cast isn't available)
ALTER TABLE users ALTER COLUMN age TYPE TEXT
  USING age::TEXT;       -- convert existing INTEGER values to TEXT

-- Convert string date to actual DATE type
ALTER TABLE events ALTER COLUMN event_date TYPE DATE
  USING event_date::DATE;

-- Resize a VARCHAR
ALTER TABLE users ALTER COLUMN username TYPE VARCHAR(100);
```

---

### <span style="color:#2E86AB">SET / DROP DEFAULT</span>

```sql
-- Set a default on an existing column
ALTER TABLE users ALTER COLUMN is_active SET DEFAULT TRUE;
ALTER TABLE orders ALTER COLUMN status SET DEFAULT 'pending';

-- Remove a default
ALTER TABLE users ALTER COLUMN is_active DROP DEFAULT;
```

---

### <span style="color:#2E86AB">SET / DROP NOT NULL</span>

```sql
-- Make a column required (fails if any existing rows have NULL in that column)
ALTER TABLE users ALTER COLUMN email SET NOT NULL;

-- Make a column optional again
ALTER TABLE users ALTER COLUMN phone DROP NOT NULL;
```

---

### <span style="color:#2E86AB">ADD CONSTRAINT</span>

```sql
-- Add a named PRIMARY KEY
ALTER TABLE users ADD CONSTRAINT users_pkey PRIMARY KEY (id);

-- Add a UNIQUE constraint
ALTER TABLE users ADD CONSTRAINT users_email_unique UNIQUE (email);

-- Add a CHECK constraint
ALTER TABLE products ADD CONSTRAINT products_price_positive CHECK (price > 0);

-- Add a FOREIGN KEY
ALTER TABLE orders
  ADD CONSTRAINT orders_customer_fk
  FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE CASCADE;

-- Add composite UNIQUE
ALTER TABLE authors
  ADD CONSTRAINT authors_fullname_unique UNIQUE (first_name, last_name);
```

---

### <span style="color:#2E86AB">DROP CONSTRAINT</span>

```sql
-- Drop a named constraint
ALTER TABLE users DROP CONSTRAINT users_email_unique;

-- Safe version
ALTER TABLE users DROP CONSTRAINT IF EXISTS users_email_unique;

-- Drop and cascade (also drops dependent objects)
ALTER TABLE orders DROP CONSTRAINT orders_customer_fk CASCADE;

-- Find constraint names if you forgot them
SELECT conname, contype
FROM pg_constraint
WHERE conrelid = 'users'::regclass;

-- contype meanings:
-- p = primary key
-- u = unique
-- c = check
-- f = foreign key
-- n = not null
```

---

### <span style="color:#2E86AB">RENAME TABLE</span>

```sql
-- Rename a table
ALTER TABLE users RENAME TO app_users;

-- Rename in a specific schema
ALTER TABLE public.users RENAME TO app_users;
```

---

### <span style="color:#2E86AB">SET SCHEMA - Move Table to Another Schema</span>

```sql
-- Move table to a different schema
ALTER TABLE users SET SCHEMA archive;
-- Table is now archive.users
```

---

### <span style="color:#2E86AB">ENABLE / DISABLE Triggers and Rules</span>

```sql
-- Disable all triggers on a table (useful for bulk imports)
ALTER TABLE orders DISABLE TRIGGER ALL;

-- Re-enable
ALTER TABLE orders ENABLE TRIGGER ALL;
```

---

### <span style="color:#2E86AB">ALTER TABLE Quick Reference</span>

| Operation | Syntax |
|:---|:---|
| Add column | `ALTER TABLE t ADD COLUMN col TYPE` |
| Drop column | `ALTER TABLE t DROP COLUMN col` |
| Rename column | `ALTER TABLE t RENAME COLUMN old TO new` |
| Change type | `ALTER TABLE t ALTER COLUMN col TYPE newtype` |
| Set default | `ALTER TABLE t ALTER COLUMN col SET DEFAULT val` |
| Drop default | `ALTER TABLE t ALTER COLUMN col DROP DEFAULT` |
| Set NOT NULL | `ALTER TABLE t ALTER COLUMN col SET NOT NULL` |
| Drop NOT NULL | `ALTER TABLE t ALTER COLUMN col DROP NOT NULL` |
| Add constraint | `ALTER TABLE t ADD CONSTRAINT name ...` |
| Drop constraint | `ALTER TABLE t DROP CONSTRAINT name` |
| Rename table | `ALTER TABLE t RENAME TO newname` |
| Change schema | `ALTER TABLE t SET SCHEMA newschema` |

---

## <span style="color:#1565C0">3.8 DROP TABLE - DROP vs TRUNCATE vs DELETE</span>

> **Definition:** `DROP TABLE` permanently removes a table and all its data, indexes, constraints, and triggers from the database. It is irreversible.

---

### <span style="color:#2E86AB">DROP TABLE Syntax</span>

```sql
-- Drop a table
DROP TABLE users;

-- Safe version - no error if table doesn't exist
DROP TABLE IF EXISTS users;

-- DROP with RESTRICT (default) - fails if anything depends on this table
DROP TABLE customers RESTRICT;

-- DROP with CASCADE - also drops dependent objects
-- (foreign keys from other tables, views built on this table, etc.)
DROP TABLE customers CASCADE;

-- Drop multiple tables at once
DROP TABLE IF EXISTS orders, order_items, shipments;
```

---

### <span style="color:#2E86AB">DROP vs TRUNCATE vs DELETE - Full Comparison</span>

| Feature | `DELETE` | `TRUNCATE` | `DROP` |
|:---|:---|:---|:---|
| Removes rows | ✓ (all or filtered) | ✓ (all rows) | ✓ (all rows + structure) |
| Removes table structure | ✗ | ✗ | ✓ |
| WHERE clause support | ✓ | ✗ | ✗ |
| Triggers fired | ✓ (row-level) | ✓ (statement-level only) | ✗ |
| Transaction-safe (rollback) | ✓ | ✓ | ✓ |
| Resets SERIAL/IDENTITY | ✗ | Optional (RESTART IDENTITY) | N/A |
| Speed on large tables | Slow (row-by-row) | Very fast | Very fast |
| WAL logging | Full (every row) | Minimal | Minimal |
| Foreign key check | ✓ | ✓ | CASCADE needed |
| Type | DML | DDL (behaves like DDL) | DDL |

```sql
-- DELETE - removes specific rows, table remains
DELETE FROM orders WHERE created_at < '2020-01-01';
DELETE FROM orders;  -- removes all rows, but slowly

-- TRUNCATE - removes all rows fast, table structure remains
TRUNCATE TABLE orders;

-- DROP - removes table entirely (structure + data)
DROP TABLE orders;
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 560px;">

### <span style="color:#D4A017">Rule of Thumb: Use DELETE when you need WHERE conditions or triggers. Use TRUNCATE to quickly empty a table. Use DROP to completely remove a table from the schema.</span>

</div>

---

## <span style="color:#1565C0">3.9 TRUNCATE TABLE</span>

> **Definition:** `TRUNCATE` removes all rows from a table very efficiently - much faster than `DELETE` for large tables - by deallocating data pages directly instead of deleting row by row.

---

### <span style="color:#2E86AB">TRUNCATE Syntax</span>

```sql
-- Basic TRUNCATE
TRUNCATE TABLE orders;

-- Truncate multiple tables at once
TRUNCATE TABLE orders, order_items, shipments;

-- RESTART IDENTITY - resets all SERIAL / IDENTITY sequences back to 1
TRUNCATE TABLE orders RESTART IDENTITY;

-- CONTINUE IDENTITY - keeps sequence at current value (default)
TRUNCATE TABLE orders CONTINUE IDENTITY;

-- CASCADE - also truncates tables that have FK references to this table
TRUNCATE TABLE customers CASCADE;
-- This also truncates orders (which has FK to customers)

-- Combine options
TRUNCATE TABLE customers RESTART IDENTITY CASCADE;
```

---

### <span style="color:#2E86AB">TRUNCATE and Transactions</span>

```sql
-- TRUNCATE is transaction-safe - it can be rolled back
BEGIN;
  TRUNCATE TABLE test_data;
  -- Oops, I didn't mean to
ROLLBACK;
-- All rows are restored
```

---

### <span style="color:#2E86AB">TRUNCATE vs DELETE Performance</span>

```sql
-- TRUNCATE on 10 million rows - takes milliseconds
TRUNCATE TABLE big_table;

-- DELETE on 10 million rows - takes minutes (row-by-row WAL logging)
DELETE FROM big_table;

-- But DELETE can be filtered:
DELETE FROM big_table WHERE created_at < NOW() - INTERVAL '1 year';
-- TRUNCATE cannot do this
```

---

## <span style="color:#1565C0">3.10 Sequences</span>

> **Definition:** A Sequence is a database object that generates a series of unique integer values, typically used to auto-populate primary key columns. Sequences are independent objects - they exist separately from tables.

---

### <span style="color:#2E86AB">CREATE SEQUENCE</span>

```sql
-- Basic sequence (starts at 1, increments by 1)
CREATE SEQUENCE user_id_seq;

-- Full syntax with all options
CREATE SEQUENCE order_number_seq
  START WITH 1000         -- first value
  INCREMENT BY 1          -- step size
  MINVALUE 1000           -- minimum value
  MAXVALUE 9999999        -- maximum value
  CYCLE                   -- restart after reaching MAXVALUE
  CACHE 10;               -- pre-allocate 10 values for performance

-- Sequence for negative progression
CREATE SEQUENCE countdown_seq
  START WITH -1
  INCREMENT BY -1
  MAXVALUE -1
  MINVALUE -1000000
  NO CYCLE;               -- stop at MINVALUE instead of cycling
```

---

### <span style="color:#2E86AB">Sequence Functions</span>

```sql
-- nextval - advance sequence and return next value (most common)
SELECT nextval('user_id_seq');  -- 1 (first call)
SELECT nextval('user_id_seq');  -- 2
SELECT nextval('user_id_seq');  -- 3

-- currval - return current value without advancing (must call nextval first)
SELECT currval('user_id_seq');  -- 3 (last value returned by nextval in this session)

-- lastval - return last value returned by ANY sequence in this session
SELECT lastval();               -- 3

-- setval - manually set sequence to a specific value
SELECT setval('user_id_seq', 100);        -- next nextval returns 101
SELECT setval('user_id_seq', 100, false); -- next nextval returns 100

-- Using nextval as a column default
CREATE TABLE orders (
  id          INTEGER PRIMARY KEY DEFAULT nextval('order_number_seq'),
  customer_id INTEGER NOT NULL
);
```

---

### <span style="color:#2E86AB">Sequence Ownership</span>

```sql
-- Attach sequence to a column (so it's dropped with the table)
ALTER SEQUENCE order_number_seq OWNED BY orders.id;

-- Remove ownership
ALTER SEQUENCE order_number_seq OWNED BY NONE;

-- List sequences in current database
SELECT sequencename, start_value, increment_by, max_value, last_value
FROM pg_sequences
WHERE schemaname = 'public';
```

---

### <span style="color:#2E86AB">ALTER SEQUENCE</span>

```sql
-- Restart a sequence from its start value
ALTER SEQUENCE user_id_seq RESTART;

-- Restart from a specific value
ALTER SEQUENCE user_id_seq RESTART WITH 500;

-- Change increment
ALTER SEQUENCE user_id_seq INCREMENT BY 5;

-- Change max
ALTER SEQUENCE user_id_seq MAXVALUE 999999;
```

---

### <span style="color:#2E86AB">DROP SEQUENCE</span>

```sql
DROP SEQUENCE user_id_seq;
DROP SEQUENCE IF EXISTS user_id_seq;
DROP SEQUENCE user_id_seq CASCADE;  -- also drops columns that use it as default
```

---

### <span style="color:#2E86AB">SERIAL vs SEQUENCE vs IDENTITY - Comparison</span>

| Method | Example | Notes |
|:---|:---|:---|
| `SERIAL` | `id SERIAL PRIMARY KEY` | Shorthand - creates sequence automatically. Widely used. |
| Manual SEQUENCE | `DEFAULT nextval('seq')` | Full control - use when you need custom sequence behavior |
| `GENERATED ALWAYS AS IDENTITY` | SQL standard (PG 10+). Prevents manual overrides. Preferred for new code. | |
| `GENERATED BY DEFAULT AS IDENTITY` | Like above but allows manual inserts when needed. | |

```sql
-- GENERATED ALWAYS AS IDENTITY (cannot manually insert id)
CREATE TABLE users (
  id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);
INSERT INTO users DEFAULT VALUES;         -- OK
INSERT INTO users (id) VALUES (99);       -- ERROR: cannot insert into identity column

-- GENERATED BY DEFAULT AS IDENTITY (can override if needed)
CREATE TABLE users (
  id INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY
);
INSERT INTO users DEFAULT VALUES;         -- OK
INSERT INTO users (id) VALUES (99);       -- OK
```

---

### <span style="color:#2E86AB">Important: Sequence Gaps are Normal</span>

```sql
-- Sequences never rollback - even if your transaction does
BEGIN;
  INSERT INTO users (name) VALUES ('Alice');  -- nextval used: 1
ROLLBACK;  -- row deleted, but sequence stays at 1

BEGIN;
  INSERT INTO users (name) VALUES ('Bob');    -- nextval returns 2 (not 1)
COMMIT;
-- id 1 is permanently "used up" - the gap (1, 3, 4...) is normal and expected

-- This is by design: guarantees uniqueness under concurrent access
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 560px;">

### <span style="color:#D4A017">Sequence gaps are normal and expected. Never rely on sequential, gap-free IDs in application logic. If you need gap-free numbers (invoice numbers), implement them with a transaction and locking - not just a sequence.</span>

</div>

---

## <span style="color:#1565C0">3.11 Temporary Tables</span>

> **Definition:** A Temporary Table exists only for the duration of a database session (or transaction). It is automatically dropped when the session ends. Temp tables are private - each session has its own copy and cannot see another session's temp tables.

---

### <span style="color:#2E86AB">CREATE TEMP TABLE</span>

```sql
-- Session-scoped temp table (dropped when session disconnects)
CREATE TEMP TABLE temp_results (
  id     SERIAL,
  value  NUMERIC,
  label  TEXT
);

-- TEMPORARY and TEMP are identical keywords
CREATE TEMPORARY TABLE staging_data (
  raw_line  TEXT,
  processed BOOLEAN DEFAULT FALSE
);

-- Temp table from a query
CREATE TEMP TABLE recent_orders AS
SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '7 days';

-- ON COMMIT DROP - drop automatically when transaction ends (not session)
CREATE TEMP TABLE batch_work (
  item_id INTEGER
) ON COMMIT DROP;

-- ON COMMIT DELETE ROWS - keep structure, delete rows at transaction end
CREATE TEMP TABLE session_cache (
  key   TEXT,
  value TEXT
) ON COMMIT DELETE ROWS;

-- ON COMMIT PRESERVE ROWS (default) - rows survive until session ends
CREATE TEMP TABLE working_set (
  id INTEGER
) ON COMMIT PRESERVE ROWS;
```

---

### <span style="color:#2E86AB">When to Use Temp Tables</span>

```sql
-- 1. Store intermediate results for complex multi-step queries
CREATE TEMP TABLE step1 AS
SELECT user_id, SUM(amount) AS total_spend
FROM orders
WHERE created_at > NOW() - INTERVAL '1 year'
GROUP BY user_id;

CREATE TEMP TABLE step2 AS
SELECT s.user_id, s.total_spend, u.email
FROM step1 s
JOIN users u ON u.id = s.user_id;

SELECT * FROM step2 WHERE total_spend > 1000;

-- 2. Staging area for data import/transformation
CREATE TEMP TABLE import_staging (
  raw_data TEXT
);

COPY import_staging FROM '/tmp/upload.csv';

-- Validate and process
INSERT INTO final_table
SELECT cleaned_column(raw_data) FROM import_staging
WHERE raw_data IS NOT NULL;

-- 3. Replace repeated subqueries in reports
CREATE TEMP TABLE monthly_summary AS
SELECT ...complex query...;

-- Now use monthly_summary multiple times without re-running the heavy query
SELECT * FROM monthly_summary WHERE region = 'North';
SELECT * FROM monthly_summary WHERE revenue > 50000;
```

---

## <span style="color:#1565C0">3.12 Table Inheritance (PostgreSQL-Specific)</span>

> **Definition:** Table Inheritance allows a child table to inherit all columns and constraints from a parent table. The child table can add its own columns. Queries on the parent table automatically include rows from child tables.

---

### <span style="color:#2E86AB">Creating Inherited Tables</span>

```sql
-- Parent table
CREATE TABLE vehicles (
  id           SERIAL PRIMARY KEY,
  make         TEXT NOT NULL,
  model        TEXT NOT NULL,
  year         SMALLINT NOT NULL,
  color        TEXT,
  registered_at DATE DEFAULT CURRENT_DATE
);

-- Child tables inherit all parent columns + add their own
CREATE TABLE cars (
  num_doors  SMALLINT DEFAULT 4,
  trunk_size NUMERIC(5,2)
) INHERITS (vehicles);

CREATE TABLE motorcycles (
  has_sidecar BOOLEAN DEFAULT FALSE,
  engine_cc   INTEGER
) INHERITS (vehicles);

CREATE TABLE trucks (
  payload_kg  NUMERIC(10,2),
  num_axles   SMALLINT DEFAULT 2
) INHERITS (vehicles);
```

---

### <span style="color:#2E86AB">Querying Inherited Tables</span>

```sql
-- Insert into child tables
INSERT INTO cars (make, model, year, num_doors)
VALUES ('Toyota', 'Camry', 2022, 4);

INSERT INTO motorcycles (make, model, year, engine_cc)
VALUES ('Honda', 'CBR', 2021, 600);

-- Query parent - returns ALL rows from all child tables
SELECT * FROM vehicles;
-- Returns both the Toyota Camry AND the Honda CBR

-- Query child only
SELECT * FROM cars;       -- only cars
SELECT * FROM motorcycles; -- only motorcycles

-- ONLY keyword - query JUST the parent, exclude children
SELECT * FROM ONLY vehicles;
-- Returns rows inserted directly into vehicles, not from cars/motorcycles
```

---

### <span style="color:#2E86AB">Caveats of Table Inheritance</span>

| Feature | Supported |
|:---|:---|
| Inherits columns | ✓ |
| Inherits CHECK constraints | ✓ |
| Inherits indexes | ✗ (must create on each child) |
| Inherits UNIQUE constraints | ✗ (not enforced across child tables) |
| Inherits FOREIGN KEY from parent | ✗ |
| FOREIGN KEY can reference child rows | ✗ (only parent or child, not both) |

```sql
-- Must create indexes on each child separately
CREATE INDEX ON cars (make, model);
CREATE INDEX ON motorcycles (make, model);
CREATE INDEX ON trucks (make, model);

-- PRIMARY KEY on parent does NOT prevent duplicate IDs in children
-- Each child manages its own sequence independently by default
```

> **Note:** Table inheritance has limitations that make it less common in modern PostgreSQL. For partitioning use cases, **declarative partitioning** (Phase 3.13) is preferred. Inheritance is mainly useful for polymorphic schemas.

---

## <span style="color:#1565C0">3.13 Partitioning Tables</span>

> **Definition:** Table Partitioning divides a logically single large table into smaller physical pieces (partitions) based on a column's values. PostgreSQL routes queries and writes to the correct partition automatically. This improves performance and manageability for very large tables.

---

### <span style="color:#2E86AB">Why Partition?</span>

| Benefit | How |
|:---|:---|
| Query performance | Partition pruning - queries scan only relevant partitions |
| Faster deletes | Drop a whole partition instead of row-by-row DELETE |
| Easier archiving | Detach old partitions and archive them |
| Index efficiency | Smaller per-partition indexes |

**Use partitioning when a table has millions+ of rows and queries consistently filter by a specific column (dates, regions, status).**

---

### <span style="color:#2E86AB">Range Partitioning</span>

> **Definition:** Range partitioning divides rows based on a range of values in a column. Rows where the column falls within the range go into that partition. Most commonly used with date/time columns.

```sql
-- Step 1: Create the parent (partitioned) table
CREATE TABLE orders (
  id          BIGSERIAL,
  customer_id INTEGER NOT NULL,
  total       NUMERIC(10,2),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Step 2: Create partitions for each range
CREATE TABLE orders_2022 PARTITION OF orders
  FOR VALUES FROM ('2022-01-01') TO ('2023-01-01');

CREATE TABLE orders_2023 PARTITION OF orders
  FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders
  FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Step 3: Create a default partition for values outside all ranges
CREATE TABLE orders_default PARTITION OF orders DEFAULT;

-- All inserts are routed automatically
INSERT INTO orders (customer_id, total, created_at)
VALUES (1, 99.99, '2023-06-15');
-- Goes into orders_2023 automatically

-- Query the parent - PostgreSQL uses only relevant partitions (partition pruning)
SELECT * FROM orders WHERE created_at BETWEEN '2023-01-01' AND '2023-12-31';
-- Only scans orders_2023 - orders_2022 and orders_2024 are not touched

-- Each partition can have its own indexes
CREATE INDEX ON orders_2023 (customer_id);
CREATE INDEX ON orders_2024 (customer_id);

-- To archive old data: detach and drop
ALTER TABLE orders DETACH PARTITION orders_2022;
DROP TABLE orders_2022;  -- instant, no row-by-row deletion
```

---

### <span style="color:#2E86AB">List Partitioning</span>

> **Definition:** List partitioning assigns rows to partitions based on exact matching values of a column. Best for categorical data with a known, limited set of values.

```sql
-- Partition by country/region
CREATE TABLE customers (
  id       BIGSERIAL,
  name     TEXT NOT NULL,
  country  TEXT NOT NULL,
  email    TEXT
) PARTITION BY LIST (country);

CREATE TABLE customers_india PARTITION OF customers
  FOR VALUES IN ('IN', 'India');

CREATE TABLE customers_usa PARTITION OF customers
  FOR VALUES IN ('US', 'USA', 'United States');

CREATE TABLE customers_europe PARTITION OF customers
  FOR VALUES IN ('GB', 'DE', 'FR', 'IT', 'ES');

CREATE TABLE customers_other PARTITION OF customers DEFAULT;

-- Insert - automatically routed
INSERT INTO customers (name, country) VALUES ('Alice', 'IN');  -- → customers_india
INSERT INTO customers (name, country) VALUES ('Bob', 'US');    -- → customers_usa
INSERT INTO customers (name, country) VALUES ('Carlos', 'ES'); -- → customers_europe
```

---

### <span style="color:#2E86AB">Hash Partitioning</span>

> **Definition:** Hash partitioning computes a hash of the partition key column and distributes rows evenly across a fixed number of partitions. Use when you want even data distribution without a natural range or list, and when queries don't filter by the partition key.

```sql
-- Partition by user_id hash - 4 equal partitions
CREATE TABLE events (
  id       BIGSERIAL,
  user_id  INTEGER NOT NULL,
  action   TEXT,
  ts       TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY HASH (user_id);

CREATE TABLE events_p0 PARTITION OF events
  FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE events_p1 PARTITION OF events
  FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE events_p2 PARTITION OF events
  FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE events_p3 PARTITION OF events
  FOR VALUES WITH (MODULUS 4, REMAINDER 3);

-- Rows are distributed by: hash(user_id) % 4
-- Each partition gets roughly 25% of rows
```

---

### <span style="color:#2E86AB">Partitioning - Key Rules and Constraints</span>

```sql
-- PRIMARY KEY and UNIQUE must include the partition key column
CREATE TABLE orders (
  id         BIGSERIAL,
  customer_id INTEGER NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (id, created_at)   -- created_at must be included (partition key)
) PARTITION BY RANGE (created_at);

-- Indexes on parent are inherited by partitions (PostgreSQL 11+)
CREATE INDEX ON orders (customer_id);
-- Automatically creates the same index on all partitions

-- FOREIGN KEYS from other tables can't reference a partitioned table
-- unless they reference a specific partition

-- Attaching an existing table as a partition
CREATE TABLE orders_2025 (LIKE orders);  -- create standalone table
ALTER TABLE orders ATTACH PARTITION orders_2025
  FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- Detaching a partition (makes it a standalone table again)
ALTER TABLE orders DETACH PARTITION orders_2022;
```

---

### <span style="color:#2E86AB">Partitioning Strategy Summary</span>

| Strategy | Partition Key | Best For |
|:---|:---|:---|
| Range | `created_at`, `date`, `id` range | Time-series data, logs, events |
| List | `country`, `status`, `region` | Categorical, regional data |
| Hash | Any high-cardinality column | Even distribution, no natural grouping |

---

## <span style="color:#1565C0">Phase 3 - Quick Reference Card</span>

### <span style="color:#2E86AB">Constraint Summary</span>

| Constraint | Purpose | Allows NULL |
|:---|:---|:---:|
| `NOT NULL` | Column must have a value | ✗ |
| `UNIQUE` | All values must be distinct | ✓ (multiple NULLs OK) |
| `PRIMARY KEY` | Unique + Not Null - identifies each row | ✗ |
| `FOREIGN KEY` | References another table's PK/UNIQUE | ✓ (unless NOT NULL added) |
| `CHECK` | Custom boolean condition | ✓ (NULL passes CHECK!) |
| `DEFAULT` | Fallback value when not provided | N/A |

### <span style="color:#2E86AB">ALTER TABLE Quick Reference</span>

```sql
ALTER TABLE t ADD COLUMN col TYPE [constraints];
ALTER TABLE t DROP COLUMN col [CASCADE];
ALTER TABLE t RENAME COLUMN old TO new;
ALTER TABLE t ALTER COLUMN col TYPE newtype [USING expr];
ALTER TABLE t ALTER COLUMN col SET DEFAULT val;
ALTER TABLE t ALTER COLUMN col DROP DEFAULT;
ALTER TABLE t ALTER COLUMN col SET NOT NULL;
ALTER TABLE t ALTER COLUMN col DROP NOT NULL;
ALTER TABLE t ADD CONSTRAINT name ...;
ALTER TABLE t DROP CONSTRAINT name [CASCADE];
ALTER TABLE t RENAME TO newname;
ALTER TABLE t SET SCHEMA newschema;
```

### <span style="color:#2E86AB">DROP vs TRUNCATE vs DELETE</span>

| | `DELETE` | `TRUNCATE` | `DROP` |
|:---|:---:|:---:|:---:|
| Removes rows | ✓ | ✓ | ✓ |
| Keeps structure | ✓ | ✓ | ✗ |
| WHERE filter | ✓ | ✗ | ✗ |
| Rollback safe | ✓ | ✓ | ✓ |
| Speed | Slow | Fast | Fast |

### <span style="color:#2E86AB">ON DELETE Action Reference</span>

```sql
REFERENCES parent(id) ON DELETE CASCADE      -- child deleted with parent
REFERENCES parent(id) ON DELETE SET NULL     -- child.fk becomes NULL
REFERENCES parent(id) ON DELETE SET DEFAULT  -- child.fk gets its DEFAULT
REFERENCES parent(id) ON DELETE RESTRICT     -- error if children exist
REFERENCES parent(id) ON DELETE NO ACTION    -- same as RESTRICT (default)
```

---


