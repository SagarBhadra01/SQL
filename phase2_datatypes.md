<div align="center">

# <span style="color:#0A2FA8">PostgreSQL : Phase 2</span>

<sub>Data Types </sub>

</div>

---

## <span style="color:#1565C0">2.1 Numeric Types</span>

> **Definition:** Numeric types store numbers - whole integers, decimals with exact precision, or floating-point approximations. Choosing the right numeric type matters for storage size, precision, and performance.

---

### <span style="color:#2E86AB">Integer Types</span>

Integer types store whole numbers (no decimal part). PostgreSQL provides three sizes:

| Type | Storage | Range | Use When |
|:---|:---:|:---|:---|
| `SMALLINT` | 2 bytes | -32,768 to 32,767 | Tiny values - age, status codes, small counters |
| `INTEGER` / `INT` | 4 bytes | -2,147,483,648 to 2,147,483,647 | General purpose integers - IDs, counts, quantities |
| `BIGINT` | 8 bytes | -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807 | Very large values - timestamps in ms, large counters |

```sql
-- Declaring integer columns
CREATE TABLE products (
  id          INTEGER,
  year        SMALLINT,
  view_count  BIGINT
);

-- INT and INTEGER are identical - both work
CREATE TABLE orders (
  id INT
);

-- Checking ranges
SELECT
  32767::SMALLINT,            -- max SMALLINT
  2147483647::INTEGER,        -- max INTEGER
  9223372036854775807::BIGINT -- max BIGINT
;

-- Overflow error example (will throw an error)
SELECT 32768::SMALLINT;  -- ERROR: smallint out of range
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 560px;">

### <span style="color:#D4A017">Rule: Default to INTEGER. Use BIGINT for IDs in high-volume tables (10M+ rows). Use SMALLINT only when you are certain the range is tiny.</span>

</div>

---

### <span style="color:#2E86AB">Exact Numeric Types - DECIMAL and NUMERIC</span>

> **Definition:** `DECIMAL` and `NUMERIC` are exact-precision types - they store numbers with no rounding error. They are identical in PostgreSQL. Use them whenever accuracy is non-negotiable (money, measurements, scientific data).

```sql
NUMERIC(precision, scale)
DECIMAL(precision, scale)
```

| Term | Meaning | Example for NUMERIC(8,2) |
|:---|:---|:---|
| `precision` | Total number of significant digits (both sides of decimal) | 8 digits total |
| `scale` | Number of digits after the decimal point | 2 digits after `.` |

```sql
-- NUMERIC(8, 2) can store: 123456.78  (6 before + 2 after = 8 total)
CREATE TABLE invoices (
  id          SERIAL PRIMARY KEY,
  amount      NUMERIC(10, 2),   -- up to 99999999.99
  tax_rate    NUMERIC(5, 4),    -- up to 9.9999 (e.g. 0.1875)
  quantity    NUMERIC           -- no precision limit specified
);

-- Exact arithmetic - no floating point errors
SELECT 0.1 + 0.2;               -- returns 0.3 (exact, as NUMERIC)
SELECT 0.1::FLOAT + 0.2::FLOAT; -- returns 0.30000000000000004 (floating point!)

-- Inserting values
INSERT INTO invoices (amount, tax_rate) VALUES (1999.99, 0.1800);

-- Scale is enforced - extra decimals are rounded
INSERT INTO invoices (amount) VALUES (99.999);   -- stored as 100.00
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 560px;">

### <span style="color:#D4A017">Always use NUMERIC for money and financial values. Never use FLOAT for currency - floating-point rounding errors will cause incorrect calculations.</span>

</div>

---

### <span style="color:#2E86AB">Floating-Point Types - REAL and DOUBLE PRECISION</span>

> **Definition:** Floating-point types store approximate numeric values. They are fast and use less storage than `NUMERIC`, but they are **inexact** - small rounding errors occur. Use them for scientific or statistical values where tiny imprecision is acceptable.

| Type | Storage | Precision | Alias |
|:---|:---:|:---|:---|
| `REAL` | 4 bytes | ~6 decimal digits of precision | `FLOAT4` |
| `DOUBLE PRECISION` | 8 bytes | ~15 decimal digits of precision | `FLOAT8`, `FLOAT` |

```sql
CREATE TABLE sensor_readings (
  id          SERIAL PRIMARY KEY,
  temperature REAL,              -- e.g. 36.6
  latitude    DOUBLE PRECISION,  -- e.g. 19.07283491823
  longitude   DOUBLE PRECISION
);

-- Floating-point imprecision demonstration
SELECT 0.1::REAL + 0.2::REAL;
-- Result: 0.3 (looks fine, but internally it's 0.30000001192092896)

SELECT 0.1::DOUBLE PRECISION + 0.2::DOUBLE PRECISION;
-- Result: 0.30000000000000004

-- Special float values
SELECT 'Infinity'::FLOAT;    -- positive infinity
SELECT '-Infinity'::FLOAT;   -- negative infinity
SELECT 'NaN'::FLOAT;         -- Not a Number (result of 0.0/0.0)
```

---

### <span style="color:#2E86AB">Auto-Increment Types - SERIAL and BIGSERIAL</span>

> **Definition:** `SERIAL` and `BIGSERIAL` are shorthand notations in PostgreSQL that automatically create a **sequence** and use its `nextval()` as the default value for a column. They generate unique, incrementing integers - commonly used for primary keys.

| Type | Underlying Type | Range | Storage |
|:---|:---|:---|:---:|
| `SMALLSERIAL` | `SMALLINT` | 1 to 32,767 | 2 bytes |
| `SERIAL` | `INTEGER` | 1 to 2,147,483,647 | 4 bytes |
| `BIGSERIAL` | `BIGINT` | 1 to 9,223,372,036,854,775,807 | 8 bytes |

```sql
-- Using SERIAL
CREATE TABLE users (
  id    SERIAL PRIMARY KEY,
  name  TEXT NOT NULL
);

-- What PostgreSQL actually does internally when you write SERIAL:
-- 1. Creates a sequence:  CREATE SEQUENCE users_id_seq
-- 2. Sets column default: DEFAULT nextval('users_id_seq')
-- 3. Marks NOT NULL

-- INSERT without specifying id - it auto-fills
INSERT INTO users (name) VALUES ('Alice');  -- id = 1
INSERT INTO users (name) VALUES ('Bob');    -- id = 2
INSERT INTO users (name) VALUES ('Charlie');-- id = 3

-- Verify the sequence
SELECT currval('users_id_seq');  -- returns 3

-- BIGSERIAL for high-volume tables
CREATE TABLE events (
  id         BIGSERIAL PRIMARY KEY,
  event_name TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);
```

#### <span style="color:#5B8DB8">SERIAL vs GENERATED ALWAYS AS IDENTITY (Modern Approach)</span>

PostgreSQL 10+ introduced the SQL-standard `IDENTITY` column, which is preferred over `SERIAL`:

```sql
-- Old way (SERIAL) - still works, widely used
CREATE TABLE users (
  id SERIAL PRIMARY KEY
);

-- New way (IDENTITY) - SQL standard, recommended for new projects
CREATE TABLE users (
  id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);

-- GENERATED BY DEFAULT allows manual override
CREATE TABLE users (
  id INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY
);

-- Difference:
-- GENERATED ALWAYS   → PostgreSQL rejects manual id inserts (safer)
-- GENERATED BY DEFAULT → allows overriding id manually if needed
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 560px;">

### <span style="color:#D4A017">SERIAL gaps are normal - if you INSERT then ROLLBACK, the sequence advances but the row doesn't exist. IDs will have gaps. This is expected and not a bug.</span>

</div>

---

### <span style="color:#2E86AB">Numeric Types - Full Comparison</span>

| Type | Exact? | Storage | Best For |
|:---|:---:|:---:|:---|
| `SMALLINT` | ✓ | 2 bytes | Tiny whole numbers |
| `INTEGER` | ✓ | 4 bytes | General-purpose IDs and counts |
| `BIGINT` | ✓ | 8 bytes | Large IDs, epoch timestamps |
| `NUMERIC(p,s)` | ✓ | Variable | Money, financial, exact precision |
| `REAL` | ✗ | 4 bytes | Scientific data, approximate floats |
| `DOUBLE PRECISION` | ✗ | 8 bytes | High-range approximate floats |
| `SERIAL` | ✓ | 4 bytes | Auto-increment primary keys |
| `BIGSERIAL` | ✓ | 8 bytes | Auto-increment for high-volume tables |

---

## <span style="color:#1565C0">2.2 Text & Character Types</span>

> **Definition:** Character types store text strings. PostgreSQL provides three main text types: fixed-length `CHAR(n)`, variable-length `VARCHAR(n)`, and unlimited `TEXT`.

---

### <span style="color:#2E86AB">CHAR(n) - Fixed Length</span>

> **Definition:** `CHAR(n)` (also `CHARACTER(n)`) stores a string of **exactly n characters**. If the value is shorter than n, it is **right-padded with spaces** to fill the length.

```sql
CREATE TABLE test_char (
  code CHAR(5)
);

INSERT INTO test_char VALUES ('AB');     -- stored as 'AB   ' (3 spaces added)
INSERT INTO test_char VALUES ('ABCDE');  -- stored as 'ABCDE' (exact)
INSERT INTO test_char VALUES ('ABCDEF'); -- ERROR: value too long for type character(5)

-- Trailing spaces are stripped on comparison
SELECT * FROM test_char WHERE code = 'AB';     -- ✓ matches 'AB   '
SELECT * FROM test_char WHERE code = 'AB   ';  -- ✓ also matches

-- Checking actual stored length
SELECT length(code) FROM test_char;  -- returns 2 (spaces are ignored by length())
SELECT char_length(code) FROM test_char; -- also returns 2
```

**When to use `CHAR(n)`:** Rarely in modern PostgreSQL. Mainly for fixed-format codes where every value is guaranteed to be the same length - country codes (`CHAR(2)`), currency codes (`CHAR(3)`), fixed-width legacy formats.

---

### <span style="color:#2E86AB">VARCHAR(n) - Variable Length with Limit</span>

> **Definition:** `VARCHAR(n)` (also `CHARACTER VARYING(n)`) stores strings of **up to n characters**. Shorter strings are stored as-is with no padding. Values exceeding n characters throw an error.

```sql
CREATE TABLE users (
  username   VARCHAR(50),   -- max 50 characters
  email      VARCHAR(255),  -- max 255 characters
  bio        VARCHAR(500)   -- max 500 characters
);

INSERT INTO users (username) VALUES ('alice');        -- stored as 'alice' (5 chars)
INSERT INTO users (username) VALUES ('a');            -- stored as 'a' (1 char)
INSERT INTO users (username) VALUES (repeat('x', 60)); -- ERROR: value too long

-- No padding - what you insert is what you get
SELECT length(username) FROM users WHERE username = 'alice'; -- returns 5
```

---

### <span style="color:#2E86AB">TEXT - Unlimited Length</span>

> **Definition:** `TEXT` stores strings of **unlimited length**. There is no upper size limit (up to 1 GB per value in practice). It is the most flexible text type in PostgreSQL.

```sql
CREATE TABLE articles (
  id      SERIAL PRIMARY KEY,
  title   VARCHAR(200),   -- short, bounded
  content TEXT            -- could be megabytes of text
);

INSERT INTO articles (title, content)
VALUES ('Hello World', 'This is a very long article content...');
```

---

### <span style="color:#2E86AB">CHAR vs VARCHAR vs TEXT - Key Differences</span>

| Feature | `CHAR(n)` | `VARCHAR(n)` | `TEXT` |
|:---|:---:|:---:|:---:|
| Max length enforced | ✓ (fixed) | ✓ (user-defined) | ✗ (unlimited) |
| Pads with spaces | ✓ | ✗ | ✗ |
| Storage | Fixed - always n bytes | Variable - actual length | Variable - actual length |
| Performance | Same | Same | Same |
| Trailing space in comparison | Ignored | Not ignored | Not ignored |

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 560px;">

### <span style="color:#D4A017">PostgreSQL Performance Note: CHAR, VARCHAR, and TEXT have identical performance internally. Unlike other databases, there is NO performance penalty for using TEXT over VARCHAR.</span>

</div>

```sql
-- PostgreSQL recommendation for most use cases:
-- Use TEXT for everything, add CHECK constraints for length limits if needed

CREATE TABLE users (
  username TEXT CHECK (char_length(username) <= 50),
  email    TEXT CHECK (char_length(email) <= 255)
);

-- Or simply use VARCHAR(n) for self-documenting intent:
CREATE TABLE users (
  username VARCHAR(50),   -- clearly communicates: max 50 chars expected
  email    VARCHAR(255)
);
```

---

## <span style="color:#1565C0">2.3 Boolean</span>

> **Definition:** The `BOOLEAN` type stores one of three values: `TRUE`, `FALSE`, or `NULL`. It uses 1 byte of storage.

---

### <span style="color:#2E86AB">Accepted Input Values</span>

PostgreSQL is flexible in what it accepts as boolean input:

| Accepted as TRUE | Accepted as FALSE |
|:---:|:---:|
| `TRUE` | `FALSE` |
| `true` | `false` |
| `'t'` | `'f'` |
| `'yes'` | `'no'` |
| `'y'` | `'n'` |
| `'on'` | `'off'` |
| `'1'` | `'0'` |

```sql
CREATE TABLE settings (
  id           SERIAL PRIMARY KEY,
  is_active    BOOLEAN DEFAULT TRUE,
  is_verified  BOOLEAN DEFAULT FALSE,
  is_deleted   BOOLEAN
);

-- All of these are valid inserts
INSERT INTO settings (is_active, is_verified) VALUES (TRUE, FALSE);
INSERT INTO settings (is_active, is_verified) VALUES ('yes', 'no');
INSERT INTO settings (is_active, is_verified) VALUES ('on', 'off');
INSERT INTO settings (is_active, is_verified) VALUES (1, 0);
```

---

### <span style="color:#2E86AB">Boolean in WHERE Clauses</span>

```sql
-- Standard way
SELECT * FROM settings WHERE is_active = TRUE;
SELECT * FROM settings WHERE is_active = FALSE;

-- Shorthand (cleaner)
SELECT * FROM settings WHERE is_active;      -- same as = TRUE
SELECT * FROM settings WHERE NOT is_active;  -- same as = FALSE

-- NULL behavior - IS NULL must be used, not = NULL
SELECT * FROM settings WHERE is_deleted IS NULL;
SELECT * FROM settings WHERE is_deleted IS NOT NULL;

-- NULL is neither TRUE nor FALSE
SELECT NULL = TRUE;   -- returns NULL (not FALSE!)
SELECT NULL = FALSE;  -- returns NULL (not TRUE!)
```

---

### <span style="color:#2E86AB">Boolean NULL Behavior - The Three-Value Logic</span>

| Operation | Result |
|:---|:---:|
| `TRUE AND TRUE` | `TRUE` |
| `TRUE AND FALSE` | `FALSE` |
| `TRUE AND NULL` | `NULL` |
| `FALSE AND NULL` | `FALSE` |
| `TRUE OR FALSE` | `TRUE` |
| `TRUE OR NULL` | `TRUE` |
| `FALSE OR NULL` | `NULL` |
| `NOT TRUE` | `FALSE` |
| `NOT FALSE` | `TRUE` |
| `NOT NULL` | `NULL` |

---

## <span style="color:#1565C0">2.4 Date & Time Types</span>

> **Definition:** PostgreSQL provides a rich set of date/time types for storing temporal data. Understanding the difference between timezone-aware and timezone-naive types is critical.

---

### <span style="color:#2E86AB">DATE</span>

> **Definition:** `DATE` stores a calendar date (year, month, day) with **no time and no timezone** information. Storage: 4 bytes.

```sql
CREATE TABLE employees (
  id         SERIAL PRIMARY KEY,
  name       TEXT,
  birth_date DATE,
  hire_date  DATE DEFAULT CURRENT_DATE
);

-- Inserting dates (ISO 8601 format recommended)
INSERT INTO employees (name, birth_date) VALUES ('Alice', '1995-08-14');
INSERT INTO employees (name, birth_date) VALUES ('Bob', '1990-03-22');

-- Other accepted formats
SELECT '2024-12-25'::DATE;   -- ISO format (recommended)
SELECT 'December 25, 2024'::DATE;
SELECT '25/12/2024'::DATE;

-- Date arithmetic
SELECT CURRENT_DATE;                        -- today
SELECT CURRENT_DATE + 7;                    -- 7 days from now
SELECT CURRENT_DATE - '2000-01-01'::DATE;   -- days since year 2000
SELECT AGE('1995-08-14');                   -- age as interval

-- Date range
SELECT '0001-01-01'::DATE;  -- minimum date
SELECT '9999-12-31'::DATE;  -- maximum date
```

---

### <span style="color:#2E86AB">TIME and TIME WITH TIME ZONE</span>

> **Definition:** `TIME` stores a time of day (hours, minutes, seconds) without a date or timezone. `TIMETZ` (TIME WITH TIME ZONE) also stores the UTC offset, but is rarely recommended.

| Type | Storage | Timezone | Typical Use |
|:---|:---:|:---:|:---|
| `TIME` | 8 bytes | ✗ | Store "clock time" - opening hours, schedules |
| `TIME WITH TIME ZONE` / `TIMETZ` | 12 bytes | ✓ | Rarely used - prefer TIMESTAMPTZ instead |

```sql
CREATE TABLE store_hours (
  id           SERIAL PRIMARY KEY,
  day_of_week  TEXT,
  opens_at     TIME,
  closes_at    TIME
);

INSERT INTO store_hours (day_of_week, opens_at, closes_at)
VALUES ('Monday', '09:00:00', '18:30:00');

-- TIME precision
SELECT '14:30'::TIME;             -- 14:30:00
SELECT '14:30:45.123'::TIME;      -- 14:30:45.123
SELECT '2:30 PM'::TIME;           -- 14:30:00

-- TIME arithmetic
SELECT '14:30:00'::TIME + INTERVAL '2 hours';   -- 16:30:00
SELECT '18:30:00'::TIME - '09:00:00'::TIME;     -- 09:30:00

-- Range: 00:00:00 to 24:00:00
```

---

### <span style="color:#2E86AB">TIMESTAMP and TIMESTAMPTZ</span>

> **Definition:** `TIMESTAMP` stores both date and time together, with no timezone information. `TIMESTAMPTZ` (TIMESTAMP WITH TIME ZONE) also records the timezone offset, storing values internally as UTC.

| Type | Storage | Timezone-aware | Alias |
|:---|:---:|:---:|:---|
| `TIMESTAMP` | 8 bytes | ✗ (naive) | `TIMESTAMP WITHOUT TIME ZONE` |
| `TIMESTAMPTZ` | 8 bytes | ✓ (aware) | `TIMESTAMP WITH TIME ZONE` |

```sql
-- TIMESTAMP - no timezone
CREATE TABLE local_events (
  id         SERIAL PRIMARY KEY,
  event_name TEXT,
  event_time TIMESTAMP
);

-- TIMESTAMPTZ - recommended for most applications
CREATE TABLE orders (
  id         SERIAL PRIMARY KEY,
  placed_at  TIMESTAMPTZ DEFAULT NOW()
);

-- How TIMESTAMPTZ works
-- Input: '2024-06-15 10:30:00+05:30'  (IST timezone)
-- Stored internally as: '2024-06-15 05:00:00 UTC'
-- Displayed as: '2024-06-15 10:30:00+05:30' (converted to your session timezone)

-- Set session timezone
SET timezone = 'Asia/Kolkata';
SELECT NOW();  -- shows in IST

SET timezone = 'UTC';
SELECT NOW();  -- shows in UTC (same moment, different display)

-- Inserting timestamps
INSERT INTO orders DEFAULT VALUES;  -- uses NOW() as default

INSERT INTO orders (placed_at) VALUES ('2024-06-15 10:30:00');
INSERT INTO orders (placed_at) VALUES ('2024-06-15 10:30:00+05:30');
INSERT INTO orders (placed_at) VALUES (NOW());

-- Range: 4713 BC to 294276 AD
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 560px;">

### <span style="color:#D4A017">Always use TIMESTAMPTZ (not TIMESTAMP) in real applications. It handles daylight saving time, timezone conversions, and global users correctly. TIMESTAMP is only for truly local, timezone-free data.</span>

</div>

---

### <span style="color:#2E86AB">INTERVAL</span>

> **Definition:** `INTERVAL` stores a span of time - a duration, not a point in time. It can represent years, months, days, hours, minutes, and seconds.

```sql
-- Declaring INTERVAL
SELECT INTERVAL '1 year 2 months 3 days';
SELECT INTERVAL '5 hours 30 minutes';
SELECT INTERVAL '90 seconds';
SELECT INTERVAL '2 weeks';
SELECT INTERVAL '-3 days';   -- negative interval

-- INTERVAL in table columns
CREATE TABLE subscriptions (
  id         SERIAL PRIMARY KEY,
  user_id    INT,
  starts_at  TIMESTAMPTZ,
  duration   INTERVAL
);

INSERT INTO subscriptions (user_id, starts_at, duration)
VALUES (1, NOW(), INTERVAL '1 year');

-- INTERVAL arithmetic with dates
SELECT NOW() + INTERVAL '7 days';          -- 7 days from now
SELECT NOW() - INTERVAL '1 month';         -- 1 month ago
SELECT '2024-12-25'::DATE + INTERVAL '10 days'; -- Jan 4, 2025
SELECT AGE(NOW(), '2000-01-01');           -- your age as interval

-- Interval units accepted
-- microseconds, milliseconds, seconds, minutes, hours
-- days, weeks, months, years, decades, centuries, millennia

-- Extracting parts from an interval
SELECT EXTRACT(hours FROM INTERVAL '2 hours 30 minutes');  -- 2
SELECT EXTRACT(minutes FROM INTERVAL '2 hours 30 minutes'); -- 30
```

---

### <span style="color:#2E86AB">Date & Time Types Summary</span>

| Type | What it stores | Timezone | Best For |
|:---|:---|:---:|:---|
| `DATE` | Year, month, day only | ✗ | Birthdays, deadlines, calendar dates |
| `TIME` | Hours, minutes, seconds only | ✗ | Store-opening times, recurring schedules |
| `TIMETZ` | Time + offset | ✓ | Rarely used |
| `TIMESTAMP` | Date + time | ✗ | Local/naive timestamps |
| `TIMESTAMPTZ` | Date + time + timezone | ✓ | Almost all real-world timestamps |
| `INTERVAL` | Duration / span of time | N/A | Expiry windows, age, durations |

---

## <span style="color:#1565C0">2.5 UUID</span>

> **Definition:** UUID (Universally Unique Identifier) is a 128-bit identifier formatted as 32 hexadecimal digits in groups: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`. It is globally unique across space and time - no two systems will ever generate the same UUID.

Storage: 16 bytes (stored as raw binary internally)

---

### <span style="color:#2E86AB">UUID vs SERIAL</span>

| Feature | `SERIAL` / `BIGSERIAL` | `UUID` |
|:---|:---|:---|
| Type | Sequential integer | 128-bit random string |
| Uniqueness | Unique within one table | Globally unique across all systems |
| Predictability | Predictable (1, 2, 3...) | Not guessable |
| Size | 4 or 8 bytes | 16 bytes |
| Merge/sync safe | ✗ (conflicts when merging DBs) | ✓ (safe to merge) |
| URL security | ✗ `/users/1` is guessable | ✓ `/users/550e8400-e29b-41d4-a716-446655440000` |
| Index performance | Slightly faster (sequential) | Slightly slower (random, causes index fragmentation) |

---

### <span style="color:#2E86AB">Using UUID in PostgreSQL</span>

```sql
-- Enable the uuid-ossp extension (provides generation functions)
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Or use the built-in gen_random_uuid() (PostgreSQL 13+, no extension needed)
SELECT gen_random_uuid();
-- Output: a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11

-- Create a table with UUID primary key
CREATE TABLE users (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name       TEXT NOT NULL,
  email      TEXT UNIQUE NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- INSERT without specifying id - auto-generates UUID
INSERT INTO users (name, email)
VALUES ('Alice', 'alice@example.com');

-- Check the generated UUID
SELECT id, name FROM users;
-- id: a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11  | name: Alice

-- Manually providing a UUID
INSERT INTO users (id, name, email)
VALUES ('550e8400-e29b-41d4-a716-446655440000', 'Bob', 'bob@example.com');

-- Casting string to UUID
SELECT '550e8400-e29b-41d4-a716-446655440000'::UUID;

-- uuid-ossp functions (if extension installed)
SELECT uuid_generate_v1();  -- time-based UUID
SELECT uuid_generate_v4();  -- random UUID (most common)
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 560px;">

### <span style="color:#D4A017">Use UUID when: you need distributed systems, you expose IDs in URLs, you merge data from multiple sources, or you're building microservices where auto-increment integers would conflict.</span>

</div>

---

## <span style="color:#1565C0">2.6 Arrays</span>

> **Definition:** PostgreSQL allows any data type to be stored as a multi-value array in a single column. Arrays are ordered collections of elements of the same type.

---

### <span style="color:#2E86AB">Declaring Array Columns</span>

```sql
-- Two ways to declare arrays
CREATE TABLE students (
  id        SERIAL PRIMARY KEY,
  name      TEXT,
  scores    INTEGER[],         -- array of integers
  tags      TEXT[],            -- array of text
  matrix    INTEGER[][],       -- 2D array
  notes     TEXT ARRAY         -- alternative syntax (same result)
);
```

---

### <span style="color:#2E86AB">Inserting Arrays</span>

```sql
-- Using ARRAY constructor
INSERT INTO students (name, scores, tags)
VALUES ('Alice', ARRAY[85, 92, 78, 95], ARRAY['math', 'science']);

-- Using curly-brace literal syntax
INSERT INTO students (name, scores, tags)
VALUES ('Bob', '{72, 68, 88}', '{"english", "history"}');

-- NULL inside arrays
INSERT INTO students (name, scores)
VALUES ('Charlie', ARRAY[90, NULL, 85]);

-- 2D array
INSERT INTO students (name, matrix)
VALUES ('Diana', ARRAY[[1,2],[3,4]]);
```

---

### <span style="color:#2E86AB">Accessing Array Elements</span>

> **Note:** PostgreSQL arrays are **1-indexed** (first element is index 1, not 0).

```sql
-- Access single element (1-indexed)
SELECT scores[1] FROM students WHERE name = 'Alice';  -- 85
SELECT scores[2] FROM students WHERE name = 'Alice';  -- 92

-- Slicing - array[start:end]
SELECT scores[1:3] FROM students WHERE name = 'Alice'; -- {85, 92, 78}

-- Array length
SELECT array_length(scores, 1) FROM students; -- 1 = first dimension
-- Returns 4 for Alice

-- Cardinality (total elements in all dimensions)
SELECT cardinality(scores) FROM students WHERE name = 'Alice'; -- 4
```

---

### <span style="color:#2E86AB">Querying with Arrays</span>

```sql
-- Check if a value exists in an array
SELECT * FROM students WHERE 92 = ANY(scores);   -- rows where 92 is in scores
SELECT * FROM students WHERE 'math' = ANY(tags); -- rows where 'math' is in tags

-- ALL - condition must be true for all elements
SELECT * FROM students WHERE 70 < ALL(scores);  -- all scores > 70

-- Contains operator @> - does array contain this set?
SELECT * FROM students WHERE tags @> ARRAY['math'];
-- Finds students who have 'math' in their tags

-- Is contained by <@
SELECT * FROM students WHERE ARRAY['math'] <@ tags;
-- Same result as @> but reversed

-- Overlap operator && - do arrays share any elements?
SELECT * FROM students WHERE tags && ARRAY['math', 'english'];
-- Students who have either math or english tag

-- WHERE array element equals value
SELECT * FROM students WHERE scores[1] = 85;  -- first score is 85
```

---

### <span style="color:#2E86AB">Array Manipulation Functions</span>

```sql
-- unnest() - expand array into rows (very useful)
SELECT name, unnest(scores) AS score FROM students;
-- Alice | 85
-- Alice | 92
-- Alice | 78
-- Alice | 95

-- array_append - add to end
SELECT array_append(ARRAY[1,2,3], 4);  -- {1,2,3,4}

-- array_prepend - add to front
SELECT array_prepend(0, ARRAY[1,2,3]);  -- {0,1,2,3}

-- array_cat - concatenate two arrays
SELECT array_cat(ARRAY[1,2], ARRAY[3,4]);  -- {1,2,3,4}

-- || operator - concatenate
SELECT ARRAY[1,2] || ARRAY[3,4];  -- {1,2,3,4}
SELECT ARRAY[1,2] || 3;           -- {1,2,3}   (append single element)

-- array_remove - remove all occurrences of a value
SELECT array_remove(ARRAY[1,2,3,2,1], 2);  -- {1,3,1}

-- array_replace
SELECT array_replace(ARRAY[1,2,3], 2, 99);  -- {1,99,3}

-- array_position - find index of first occurrence (1-indexed)
SELECT array_position(ARRAY['a','b','c'], 'b');  -- 2

-- array_positions - all positions of a value
SELECT array_positions(ARRAY[1,2,3,2,1], 2);  -- {2,4}
```

---

## <span style="color:#1565C0">2.7 JSON and JSONB</span>

> **Definition:** PostgreSQL provides two types for storing JSON data: `JSON` stores the raw text exactly as provided, while `JSONB` (JSON Binary) stores data in a parsed binary format that supports indexing, operators, and faster querying.

---

### <span style="color:#2E86AB">JSON vs JSONB - Core Differences</span>

| Feature | `JSON` | `JSONB` |
|:---|:---|:---|
| Storage format | Raw text (exact copy) | Parsed binary |
| Whitespace preserved | ✓ | ✗ |
| Duplicate keys | Last value kept | ✗ |
| Key order preserved | ✓ | ✗ |
| Indexing (GIN) | ✗ | ✓ |
| Query operators | Limited | Full set |
| Write speed | Faster | Slightly slower |
| Read/Query speed | Slower | Faster |

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 560px;">

### <span style="color:#D4A017">Always use JSONB in practice. It is faster to query, supports indexing, and has a richer operator set. JSON is only useful if you need to preserve exact whitespace or duplicate keys (very rare).</span>

</div>

---

### <span style="color:#2E86AB">Creating Tables with JSONB</span>

```sql
CREATE TABLE products (
  id         SERIAL PRIMARY KEY,
  name       TEXT,
  attributes JSONB
);

-- Insert JSON data
INSERT INTO products (name, attributes) VALUES
  ('T-Shirt', '{"color": "red", "sizes": ["S","M","L"], "price": 29.99}'),
  ('Laptop',  '{"brand": "Dell", "specs": {"ram": "16GB", "ssd": "512GB"}, "price": 899}'),
  ('Phone',   '{"brand": "Apple", "colors": ["black","white"], "price": 999}');
```

---

### <span style="color:#2E86AB">JSON/JSONB Operators</span>

| Operator | Returns | Meaning |
|:---:|:---:|:---|
| `->` | `JSON` / `JSONB` | Get value by key (as JSON) |
| `->>` | `TEXT` | Get value by key (as text) |
| `#>` | `JSON` / `JSONB` | Get nested value by path (as JSON) |
| `#>>` | `TEXT` | Get nested value by path (as text) |
| `@>` | `BOOLEAN` | Does left contain right? |
| `<@` | `BOOLEAN` | Is left contained in right? |
| `?` | `BOOLEAN` | Does key exist? |
| `?|` | `BOOLEAN` | Does any of these keys exist? |
| `?&` | `BOOLEAN` | Do all these keys exist? |

```sql
-- -> returns a JSONB value
SELECT attributes -> 'color' FROM products WHERE name = 'T-Shirt';
-- Result: "red"   (as JSONB, with quotes)

-- ->> returns text
SELECT attributes ->> 'color' FROM products WHERE name = 'T-Shirt';
-- Result: red   (as plain text, no quotes)

-- Access array element by index
SELECT attributes -> 'sizes' -> 0 FROM products WHERE name = 'T-Shirt';
-- Result: "S"

-- Nested key access with #>
SELECT attributes #> '{specs, ram}' FROM products WHERE name = 'Laptop';
-- Result: "16GB"

-- #>> returns as text
SELECT attributes #>> '{specs, ram}' FROM products WHERE name = 'Laptop';
-- Result: 16GB

-- @> containment check
SELECT * FROM products WHERE attributes @> '{"brand": "Apple"}';
-- Returns the Phone row

-- ? key existence check
SELECT * FROM products WHERE attributes ? 'color';
-- Returns T-Shirt (has a 'color' key)

-- ?| any of these keys
SELECT * FROM products WHERE attributes ?| ARRAY['color', 'brand'];
-- Returns T-Shirt and Laptop and Phone

-- ?& all of these keys must exist
SELECT * FROM products WHERE attributes ?& ARRAY['brand', 'price'];
-- Returns Laptop and Phone
```

---

### <span style="color:#2E86AB">Filtering and Using JSONB in WHERE</span>

```sql
-- Filter by a JSON value (note: use ->> for text comparison)
SELECT * FROM products WHERE attributes ->> 'brand' = 'Apple';

-- Numeric comparison (cast required)
SELECT * FROM products
WHERE (attributes ->> 'price')::NUMERIC > 100;

-- Containment filter
SELECT * FROM products
WHERE attributes @> '{"specs": {"ram": "16GB"}}';

-- Check array contains value
SELECT * FROM products
WHERE attributes -> 'sizes' ? 'L';    -- does sizes array contain "L" as key?
-- For arrays use @>:
SELECT * FROM products
WHERE attributes -> 'sizes' @> '"L"'; -- does sizes array contain "L"?
```

---

### <span style="color:#2E86AB">Modifying JSONB</span>

```sql
-- jsonb_set(target, path, new_value, create_missing)
UPDATE products
SET attributes = jsonb_set(attributes, '{price}', '34.99')
WHERE name = 'T-Shirt';

-- Add a new key
UPDATE products
SET attributes = jsonb_set(attributes, '{in_stock}', 'true', true)
WHERE name = 'T-Shirt';

-- Remove a key using - operator
UPDATE products
SET attributes = attributes - 'color'
WHERE name = 'T-Shirt';

-- Remove nested key using #- operator
UPDATE products
SET attributes = attributes #- '{specs, ram}'
WHERE name = 'Laptop';

-- Concatenate/merge two JSONB objects with ||
UPDATE products
SET attributes = attributes || '{"discount": 0.10}'::JSONB
WHERE name = 'Laptop';
```

---

### <span style="color:#2E86AB">Useful JSONB Functions</span>

```sql
-- Convert a row to JSON
SELECT row_to_json(products) FROM products WHERE id = 1;

-- jsonb_each - expand top-level keys into rows
SELECT key, value FROM jsonb_each(
  '{"name": "Alice", "age": 30}'::JSONB
);
-- key  | value
-- name | "Alice"
-- age  | 30

-- jsonb_object_keys - get all keys
SELECT jsonb_object_keys('{"a":1,"b":2,"c":3}'::JSONB);
-- a
-- b
-- c

-- jsonb_array_elements - expand JSON array into rows
SELECT jsonb_array_elements('["S","M","L"]'::JSONB);
-- "S"
-- "M"
-- "L"

-- Build a JSON object from scratch
SELECT jsonb_build_object('name', 'Alice', 'age', 30);
-- {"name": "Alice", "age": 30}

-- json_agg - aggregate rows into a JSON array
SELECT json_agg(name) FROM products;
-- ["T-Shirt", "Laptop", "Phone"]
```

---

## <span style="color:#1565C0">2.8 ENUM - Custom Enumerated Types</span>

> **Definition:** An `ENUM` (enumerated type) is a custom data type consisting of a fixed, ordered list of allowed string values. Once defined, only values from that list can be stored in an ENUM column.

---

### <span style="color:#2E86AB">Creating and Using ENUM</span>

```sql
-- Step 1: Create the ENUM type
CREATE TYPE order_status AS ENUM (
  'pending',
  'confirmed',
  'shipped',
  'delivered',
  'cancelled'
);

CREATE TYPE user_role AS ENUM ('admin', 'editor', 'viewer');

-- Step 2: Use it in a table
CREATE TABLE orders (
  id         SERIAL PRIMARY KEY,
  customer   TEXT,
  status     order_status DEFAULT 'pending',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE users (
  id    SERIAL PRIMARY KEY,
  name  TEXT,
  role  user_role DEFAULT 'viewer'
);

-- Inserting valid ENUM values
INSERT INTO orders (customer, status) VALUES ('Alice', 'pending');
INSERT INTO orders (customer, status) VALUES ('Bob', 'shipped');

-- Invalid value → ERROR
INSERT INTO orders (customer, status) VALUES ('Charlie', 'refunded');
-- ERROR: invalid input value for enum order_status: "refunded"
```

---

### <span style="color:#2E86AB">ENUM Ordering</span>

ENUMs have an implicit order based on the sequence they were defined in:

```sql
-- ENUM values are ordered as declared
SELECT * FROM orders WHERE status > 'pending';
-- Returns rows where status comes after 'pending' in the defined list
-- (confirmed, shipped, delivered, cancelled)

SELECT * FROM orders ORDER BY status;
-- Orders by ENUM order, not alphabetically
```

---

### <span style="color:#2E86AB">Modifying ENUM Types</span>

```sql
-- Add a new value
ALTER TYPE order_status ADD VALUE 'refunded';                  -- adds at end
ALTER TYPE order_status ADD VALUE 'processing' AFTER 'confirmed'; -- insert at position
ALTER TYPE order_status ADD VALUE 'on_hold' BEFORE 'shipped';

-- Rename a value (PostgreSQL 10+)
ALTER TYPE order_status RENAME VALUE 'cancelled' TO 'canceled';

-- List all ENUM values
SELECT unnest(enum_range(NULL::order_status));
-- pending
-- confirmed
-- processing
-- shipped
-- delivered
-- cancelled

-- You CANNOT remove a value from an ENUM without rebuilding it
-- (requires creating a new type and migrating)
```

---

### <span style="color:#2E86AB">ENUM vs VARCHAR with CHECK Constraint</span>

| Feature | `ENUM` | `VARCHAR + CHECK` |
|:---|:---|:---|
| Storage | Compact (4 bytes as OID) | Full string |
| Enforced at DB level | ✓ | ✓ |
| Adding new values | `ALTER TYPE` (easy) | `ALTER TABLE ... CHECK` |
| Removing values | Difficult (rebuild required) | Easy |
| Ordering | Implicit, defined at creation | Alphabetical or custom |
| Portability | PostgreSQL-specific | Standard SQL |

---

## <span style="color:#1565C0">2.9 Monetary Type - MONEY</span>

> **Definition:** The `MONEY` type stores a currency amount with a fixed fractional precision. It displays with the currency symbol based on the database locale (`lc_monetary` setting). Storage: 8 bytes.

```sql
CREATE TABLE expenses (
  id          SERIAL PRIMARY KEY,
  description TEXT,
  amount      MONEY
);

-- Insert money values
INSERT INTO expenses (description, amount) VALUES ('Lunch', 12.50);
INSERT INTO expenses (description, amount) VALUES ('Coffee', '$3.75');
INSERT INTO expenses (description, amount) VALUES ('Taxi', '₹250.00');

-- Arithmetic with MONEY
SELECT amount * 1.18 FROM expenses;   -- multiply by 1.18 (adds 18% tax)
SELECT SUM(amount) FROM expenses;     -- total

-- Check the locale
SHOW lc_monetary;  -- e.g., en_US.UTF-8 → displays as $12.50
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 560px;">

### <span style="color:#D4A017">Caution: MONEY is locale-dependent. The same value will display differently on different servers. Prefer NUMERIC(10,2) for financial data in production - it is portable, precise, and locale-independent.</span>

</div>

---

## <span style="color:#1565C0">2.10 Binary Type - BYTEA</span>

> **Definition:** `BYTEA` (byte array) stores raw binary data - images, files, PDFs, encrypted blobs, or any sequence of bytes. The name means "byte array."

```sql
CREATE TABLE documents (
  id       SERIAL PRIMARY KEY,
  filename TEXT,
  content  BYTEA
);

-- Inserting binary data using hex escape
INSERT INTO documents (filename, content)
VALUES ('hello.txt', '\x48656c6c6f');  -- hex for "Hello"

-- Using escape syntax
INSERT INTO documents (filename, content)
VALUES ('null.bin', '\000\001\002');  -- raw bytes 0, 1, 2

-- PostgreSQL functions for BYTEA
SELECT length(content) FROM documents;   -- length in bytes
SELECT encode(content, 'hex') FROM documents;   -- convert to hex string
SELECT encode(content, 'base64') FROM documents; -- convert to base64
SELECT decode('48656c6c6f', 'hex');              -- hex string to bytea
```

**When to use BYTEA:**
- Small binary data that must stay in the database (encrypted tokens, small images, certificates)
- For larger files (photos, videos, PDFs), store them in object storage (S3, GCS) and store only the URL in PostgreSQL

---

## <span style="color:#1565C0">2.11 NULL - What NULL Really Means</span>

> **Definition:** `NULL` in SQL means **the absence of a value** - it does not mean zero, empty string, or false. It represents unknown or inapplicable data. NULL is not a value; it is a marker for missing information.

---

### <span style="color:#2E86AB">NULL Behavior in Comparisons</span>

The most important rule: **any comparison with NULL returns NULL, not TRUE or FALSE**.

```sql
-- These all return NULL (not TRUE, not FALSE):
SELECT NULL = NULL;     -- NULL (unknown)
SELECT NULL = 1;        -- NULL
SELECT NULL != 1;       -- NULL
SELECT NULL > 0;        -- NULL
SELECT NULL < 0;        -- NULL
SELECT 1 + NULL;        -- NULL
SELECT 'hello' || NULL; -- NULL

-- The ONLY correct way to check for NULL:
SELECT NULL IS NULL;       -- TRUE  ✓
SELECT NULL IS NOT NULL;   -- FALSE ✓
SELECT 1 IS NULL;          -- FALSE ✓
SELECT 1 IS NOT NULL;      -- TRUE  ✓
```

---

### <span style="color:#2E86AB">NULL in WHERE Clauses</span>

```sql
CREATE TABLE users (
  id    SERIAL PRIMARY KEY,
  name  TEXT,
  phone TEXT   -- some users may not have a phone
);

INSERT INTO users (name, phone) VALUES ('Alice', '9999999999');
INSERT INTO users (name, phone) VALUES ('Bob', NULL);
INSERT INTO users (name, phone) VALUES ('Charlie', NULL);

-- WRONG - this returns NOTHING, because NULL != NULL is NULL, not TRUE
SELECT * FROM users WHERE phone != '9999999999';

-- CORRECT - explicitly handle NULL
SELECT * FROM users WHERE phone != '9999999999' OR phone IS NULL;

-- Find rows with NULL
SELECT * FROM users WHERE phone IS NULL;

-- Find rows where phone exists
SELECT * FROM users WHERE phone IS NOT NULL;

-- COUNT behavior with NULL
SELECT COUNT(*)     FROM users;  -- 3 (counts all rows)
SELECT COUNT(phone) FROM users;  -- 1 (only counts non-NULL values)
```

---

### <span style="color:#2E86AB">NULL in Arithmetic</span>

```sql
-- Any arithmetic with NULL produces NULL
SELECT 10 + NULL;   -- NULL
SELECT 10 * NULL;   -- NULL
SELECT 10 / NULL;   -- NULL
SELECT NULL / 0;    -- NULL (not a division-by-zero error!)

-- Aggregate functions ignore NULL (except COUNT(*))
SELECT AVG(phone_length)   -- ignores NULLs in calculation
SELECT SUM(score)          -- ignores NULLs
SELECT MIN(score)          -- ignores NULLs

-- To treat NULL as zero in arithmetic, use COALESCE
SELECT 10 + COALESCE(NULL, 0);  -- 10
SELECT COALESCE(score, 0) FROM tests; -- replaces NULL scores with 0
```

---

### <span style="color:#2E86AB">NULL in Sorting</span>

```sql
-- By default, NULLs sort LAST in ASC, FIRST in DESC
SELECT name, phone FROM users ORDER BY phone ASC;
-- Alice (9999999999), Bob (NULL), Charlie (NULL)

SELECT name, phone FROM users ORDER BY phone DESC;
-- Bob (NULL), Charlie (NULL), Alice (9999999999)

-- Control NULL position explicitly
SELECT name, phone FROM users ORDER BY phone ASC NULLS FIRST;
SELECT name, phone FROM users ORDER BY phone ASC NULLS LAST;
```

---

### <span style="color:#2E86AB">NULL in UNIQUE Constraints</span>

```sql
CREATE TABLE demo (
  value TEXT UNIQUE
);

-- Multiple NULLs are allowed in a UNIQUE column
-- Because NULL != NULL, so they are considered distinct
INSERT INTO demo VALUES (NULL);  -- OK
INSERT INTO demo VALUES (NULL);  -- OK - no violation!
INSERT INTO demo VALUES ('abc'); -- OK
INSERT INTO demo VALUES ('abc'); -- ERROR - 'abc' already exists
```

<div align="center" style="border: 2px solid #D4A017; border-radius: 8px; padding: 14px 24px; margin: 12px auto; max-width: 560px;">

### <span style="color:#D4A017">The Three Rules of NULL: 1) NULL is not a value - it's the absence of one. 2) Any comparison or arithmetic with NULL returns NULL. 3) Always use IS NULL / IS NOT NULL, never = NULL.</span>

</div>

---

## <span style="color:#1565C0">2.12 Type Casting - :: Syntax and CAST()</span>

> **Definition:** Type casting is the process of converting a value from one data type to another. PostgreSQL provides two equivalent syntaxes for casting: the SQL-standard `CAST()` function and the PostgreSQL-specific `::` shorthand operator.

---

### <span style="color:#2E86AB">The Two Casting Syntaxes</span>

```sql
-- PostgreSQL shorthand :: (preferred for its brevity)
SELECT '42'::INTEGER;
SELECT '3.14'::NUMERIC;
SELECT '2024-01-15'::DATE;
SELECT 'true'::BOOLEAN;

-- SQL standard CAST() function (more portable)
SELECT CAST('42' AS INTEGER);
SELECT CAST('3.14' AS NUMERIC);
SELECT CAST('2024-01-15' AS DATE);
SELECT CAST('true' AS BOOLEAN);

-- Both are identical in behaviour
SELECT '42'::INTEGER = CAST('42' AS INTEGER);  -- TRUE
```

---

### <span style="color:#2E86AB">Common Casting Examples</span>

```sql
-- String to number
SELECT '100'::INT;            -- 100
SELECT '3.14'::FLOAT;         -- 3.14
SELECT '99.99'::NUMERIC(5,2); -- 99.99

-- Number to string
SELECT 42::TEXT;              -- '42'
SELECT 3.14::VARCHAR;         -- '3.14'

-- String to date/time
SELECT '2024-12-25'::DATE;
SELECT '14:30:00'::TIME;
SELECT '2024-12-25 14:30:00'::TIMESTAMP;
SELECT '2024-12-25 14:30:00+05:30'::TIMESTAMPTZ;

-- Date/time to string (use TO_CHAR for formatting, not just cast)
SELECT NOW()::TEXT;           -- '2024-06-15 10:30:00.123456+05:30'
SELECT NOW()::DATE;           -- '2024-06-15'  (drops time part)
SELECT NOW()::TIME;           -- '10:30:00.123456'

-- Boolean casts
SELECT 'yes'::BOOLEAN;        -- TRUE
SELECT 1::BOOLEAN;            -- TRUE
SELECT 0::BOOLEAN;            -- FALSE
SELECT TRUE::TEXT;            -- 'true'
SELECT TRUE::INTEGER;         -- 1

-- JSON to JSONB and back
SELECT '{"a":1}'::JSONB;
SELECT '{"a":1}'::JSON;
SELECT '{"a":1}'::JSONB::TEXT;  -- back to text

-- Integer to float
SELECT 7::FLOAT / 2;           -- 3.5   (float division)
SELECT 7 / 2;                  -- 3     (integer division - truncates!)
SELECT 7::FLOAT / 2::FLOAT;    -- 3.5
```

---

### <span style="color:#2E86AB">Implicit vs Explicit Casting</span>

> **Definition:** **Implicit casting** happens automatically when PostgreSQL can safely convert between types without losing data. **Explicit casting** is when you manually specify the target type using `::` or `CAST()`.

```sql
-- Implicit cast - PostgreSQL converts automatically
SELECT 1 + 1.5;    -- 2.5  (INT implicitly cast to NUMERIC)
SELECT 'hello' = 'hello'::VARCHAR;  -- TRUE

-- When implicit cast fails - you must cast explicitly
CREATE TABLE test (score NUMERIC(5,2));
INSERT INTO test VALUES ('99.99');  -- OK - implicit TEXT to NUMERIC

-- Type mismatch error
SELECT '2024-01-01' - '2023-01-01';  -- ERROR: operator does not exist
SELECT '2024-01-01'::DATE - '2023-01-01'::DATE;  -- OK: returns 366 (days)
```

---

### <span style="color:#2E86AB">Safe Casting - Handling Failures</span>

```sql
-- Regular cast throws an error on bad input
SELECT 'abc'::INTEGER;  -- ERROR: invalid input syntax for type integer

-- PostgreSQL 14+ TRY_CAST equivalent using a function
-- Use a custom function or CASE + REGEXP for safe casting:

-- Safe integer check
SELECT CASE
  WHEN '123' ~ '^\d+$' THEN '123'::INTEGER
  ELSE NULL
END;

-- More robust: use a try-catch in PL/pgSQL function
CREATE OR REPLACE FUNCTION safe_cast_int(p_val TEXT)
RETURNS INTEGER AS $$
BEGIN
  RETURN p_val::INTEGER;
EXCEPTION WHEN OTHERS THEN
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;

SELECT safe_cast_int('123');   -- 123
SELECT safe_cast_int('abc');   -- NULL (instead of error)
```

---

### <span style="color:#2E86AB">Casting in Practice - Common Patterns</span>

```sql
-- Extract year as integer from a date
SELECT EXTRACT(YEAR FROM NOW())::INTEGER;

-- Convert aggregated JSON to text for display
SELECT jsonb_build_object('count', COUNT(*))::TEXT FROM users;

-- Divide two integers and get a decimal result
SELECT (total_sales::FLOAT / total_orders) AS avg_order_value
FROM summary;

-- Format a number with to_char (not just cast)
SELECT TO_CHAR(9999.5, 'FM99,999.00');   -- '9,999.50'
SELECT TO_CHAR(NOW(), 'DD Mon YYYY');    -- '15 Jun 2024'

-- Cast a BOOLEAN result to integer (0/1)
SELECT is_active::INTEGER FROM users;    -- 1 or 0

-- Cast for JSON extraction (always TEXT, then cast to target type)
SELECT (attributes ->> 'price')::NUMERIC FROM products;
SELECT (attributes ->> 'in_stock')::BOOLEAN FROM products;
```

---

## <span style="color:#1565C0">Phase 2 - Quick Reference Card</span>

### <span style="color:#2E86AB">Choosing the Right Type</span>

| Need | Use |
|:---|:---|
| Whole numbers (general) | `INTEGER` |
| Very large IDs / counters | `BIGINT` |
| Auto-increment primary key | `SERIAL` / `GENERATED ALWAYS AS IDENTITY` |
| Money / exact decimals | `NUMERIC(p, s)` |
| Scientific / approximate | `DOUBLE PRECISION` |
| Short strings with limit | `VARCHAR(n)` |
| Long or unlimited text | `TEXT` |
| True/False flags | `BOOLEAN` |
| Calendar date only | `DATE` |
| Date + time (no tz) | `TIMESTAMP` |
| Date + time (timezone-aware) | `TIMESTAMPTZ` ← use this by default |
| Duration / span | `INTERVAL` |
| Globally unique ID | `UUID` |
| Multiple values in one column | `TEXT[]`, `INTEGER[]` etc. |
| Flexible structured data | `JSONB` |
| Fixed allowed values | `ENUM` |
| Raw binary / files | `BYTEA` |

### <span style="color:#2E86AB">NULL Quick Rules</span>

| Situation | Correct Syntax |
|:---|:---|
| Check if value is NULL | `col IS NULL` |
| Check if value is not NULL | `col IS NOT NULL` |
| Replace NULL with a default | `COALESCE(col, default)` |
| Return NULL if two values are equal | `NULLIF(col, value)` |
| Arithmetic with possible NULL | Wrap in `COALESCE(col, 0)` |

### <span style="color:#2E86AB">Casting Quick Reference</span>

| From | To | Syntax |
|:---|:---|:---|
| Text → Integer | `'42'::INT` | |
| Text → Date | `'2024-01-01'::DATE` | |
| Text → JSONB | `'{"a":1}'::JSONB` | |
| Integer → Float | `7::FLOAT / 2` | |
| Any → Text | `value::TEXT` | |
| JSON → value | `(col->>'key')::TYPE` | |

---

