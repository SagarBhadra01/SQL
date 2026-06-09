<div align="center">

# <span style="color:#0A2FA8">PostgreSQL : Phase 1</span>

<sub>Foundations & Environment Setup </sub>

</div>

---

## <span style="color:#1565C0">1.1 What is Data, Database, and DBMS?</span>

### <span style="color:#2E86AB">What is Data?</span>

> **Definition:** Data is raw, unprocessed facts and figures that on their own have no meaning. Once data is processed, organized, and given context - it becomes **information**.

Examples of data:
- `9920011234` - just a number until labeled as "phone number"
- `2024-06-01` - just a date until labeled as "order date"
- `Mumbai` - just a word until labeled as "city"

| Term | Meaning | Example |
|:---|:---|:---|
| Data | Raw facts | `42`, `"John"`, `true` |
| Information | Processed data with context | "John's age is 42" |
| Knowledge | Derived insights from information | "John is eligible for senior discount" |

---

### <span style="color:#2E86AB">What is a Database?</span>

> **Definition:** A Database is an organized, structured collection of data that is stored and accessed electronically. It allows data to be stored persistently and retrieved efficiently.

Without a database, you'd store data in:
- Plain text files - no structure, hard to search
- Excel sheets - limited rows, no relationships, no concurrency
- CSV files - no data integrity, no querying

A database solves all of these problems.

---

### <span style="color:#2E86AB">What is a DBMS?</span>

> **Definition:** A Database Management System (DBMS) is software that acts as an interface between the user/application and the database. It handles storing, retrieving, updating, and managing data.

```
Application / User
       ↓
    DBMS  ← (the software you interact with)
       ↓
  Physical Storage (disk)
```

**What a DBMS does:**
- Stores and retrieves data efficiently
- Enforces data integrity and constraints
- Manages multiple users accessing data simultaneously (concurrency)
- Handles backup and recovery
- Provides security and access control
- Manages transactions (all-or-nothing operations)

**Popular DBMS examples:**

| DBMS | Type |
|:---|:---|
| PostgreSQL | Relational (RDBMS) |
| MySQL | Relational (RDBMS) |
| Oracle | Relational (RDBMS) |
| MongoDB | Document (NoSQL) |
| Redis | Key-Value (NoSQL) |
| SQLite | Lightweight Relational |

---

## <span style="color:#1565C0">1.2 What is RDBMS - How Relational Databases Work</span>

### <span style="color:#2E86AB">What is an RDBMS?</span>

> **Definition:** A Relational Database Management System (RDBMS) is a type of DBMS that stores data in a structured format using **tables** (called relations) and defines **relationships** between those tables.

It is based on the **Relational Model** proposed by **E.F. Codd** in 1970.

---

### <span style="color:#2E86AB">Core Concepts of the Relational Model</span>

#### <span style="color:#5B8DB8">Tables (Relations)</span>

Data is organized into **tables**. Each table represents one entity (e.g., `users`, `orders`, `products`).

```
TABLE: users
+----+----------+-------------------+--------+
| id | name     | email             | age    |
+----+----------+-------------------+--------+
|  1 | Alice    | alice@mail.com    |  28    |
|  2 | Bob      | bob@mail.com      |  34    |
|  3 | Charlie  | charlie@mail.com  |  22    |
+----+----------+-------------------+--------+
```

| Term | Meaning |
|:---|:---|
| Table / Relation | The whole grid of data |
| Row / Record / Tuple | One horizontal entry (one user) |
| Column / Field / Attribute | One vertical property (e.g., `name`) |
| Cell | Intersection of row and column - one value |

---

#### <span style="color:#5B8DB8">Keys</span>

> **Definition:** A **Primary Key** is a column (or combination of columns) that uniquely identifies every row in a table. No two rows can have the same primary key, and it cannot be NULL.

> **Definition:** A **Foreign Key** is a column in one table that references the Primary Key of another table. It creates the "relationship" between tables.

```
TABLE: orders
+----+---------+-------------+
| id | user_id | total_price |
+----+---------+-------------+
|  1 |    1    |    599.00   |   ← user_id 1 = Alice
|  2 |    2    |    149.00   |   ← user_id 2 = Bob
|  3 |    1    |    299.00   |   ← user_id 1 = Alice again
+----+---------+-------------+
         ↑
    Foreign Key → references users.id
```

---

#### <span style="color:#5B8DB8">Relationships Between Tables</span>

| Relationship | Example |
|:---|:---|
| One-to-One | One person has one passport |
| One-to-Many | One user can have many orders |
| Many-to-Many | One student can enroll in many courses; one course has many students |

Many-to-Many relationships require a **junction / bridge table**:

```
students ←→ enrollments ←→ courses
```

---

#### <span style="color:#5B8DB8">Data Integrity</span>

RDBMS enforces rules to keep data accurate and consistent:

| Rule | Meaning |
|:---|:---|
| Entity Integrity | Primary Key must be unique and NOT NULL |
| Referential Integrity | Foreign Key must point to a real existing row |
| Domain Integrity | Data in a column must match its defined type/constraints |
| User-Defined Integrity | Custom business rules (e.g., age must be > 0) |

---

### <span style="color:#2E86AB">How an RDBMS Processes a Query</span>

```
SQL Query written by user
        ↓
   Query Parser  (checks syntax)
        ↓
   Query Optimizer  (finds the fastest execution plan)
        ↓
   Execution Engine  (runs the plan)
        ↓
   Storage Engine  (reads/writes disk)
        ↓
    Result returned to user
```

---

## <span style="color:#1565C0">1.3 SQL vs NoSQL - Differences, When to Use What</span>

### <span style="color:#2E86AB">What is SQL?</span>

> **Definition:** SQL (Structured Query Language) is the standard language used to interact with relational databases. It covers creating structures, inserting data, querying, updating, and deleting.

SQL databases store data in **tables with fixed schemas**. Every row must follow the same column structure.

---

### <span style="color:#2E86AB">What is NoSQL?</span>

> **Definition:** NoSQL ("Not Only SQL") refers to databases that do not use the traditional relational table model. They store data in a variety of formats - documents, key-value pairs, graphs, or wide columns - and are designed for flexibility and scale.

---

### <span style="color:#2E86AB">Core Comparison</span>

| Feature | SQL (Relational) | NoSQL |
|:---|:---|:---|
| Data model | Tables with rows & columns | Documents, Key-Value, Graph, Column |
| Schema | Fixed schema (defined upfront) | Flexible / Schema-less |
| Relationships | Strong, enforced via foreign keys | Usually handled in application code |
| Query language | SQL (standardized) | Varies by database |
| ACID compliance | Strong (built-in) | Varies (often eventual consistency) |
| Scaling | Vertical scaling (scale up) | Horizontal scaling (scale out) |
| Best for | Structured, relational data | Unstructured, flexible, high-volume data |

---

### <span style="color:#2E86AB">NoSQL Database Types</span>

| Type | Example DBs | Stores data as | Use case |
|:---|:---|:---|:---|
| Document | MongoDB, CouchDB | JSON-like documents | User profiles, product catalogs |
| Key-Value | Redis, DynamoDB | Key → Value pairs | Caching, sessions |
| Column-Family | Cassandra, HBase | Column groups | Time-series, IoT data |
| Graph | Neo4j, Amazon Neptune | Nodes and edges | Social networks, fraud detection |

---

### <span style="color:#2E86AB">When to Use SQL</span>

- Data is structured and well-defined
- Relationships between entities are important
- You need strong data integrity (banking, finance, healthcare)
- Complex queries and aggregations are required
- Reporting and analytics workloads
- Team is familiar with SQL

**Examples:** E-commerce orders, banking transactions, HR systems, inventory management

---

### <span style="color:#2E86AB">When to Use NoSQL</span>

- Data structure is unknown, changes frequently, or is hierarchical
- You need to handle massive scale horizontally
- Speed of reads/writes is critical and you can tolerate eventual consistency
- Storing large amounts of unstructured data (logs, events, user activity)

**Examples:** Real-time feeds, product recommendation engines, session stores, IoT sensor data

---

### <span style="color:#2E86AB">The Reality - They Are Not Mutually Exclusive</span>

Modern applications often use both:
- PostgreSQL for core transactional data (orders, users, payments)
- Redis for caching sessions and hot data
- Elasticsearch for full-text search
- MongoDB for flexible content/catalog storage

PostgreSQL itself blurs this line - it has **JSONB** support, **full-text search**, and **array types**, making it a hybrid in many use cases.

---

## <span style="color:#1565C0">1.4 What is PostgreSQL - History, Why It's Popular</span>

### <span style="color:#2E86AB">History of PostgreSQL</span>

```
1973  → INGRES project started at UC Berkeley (Michael Stonebraker)
1986  → POSTGRES project began at UC Berkeley (post-INGRES)
1994  → Postgres95 released - SQL support added
1996  → Renamed PostgreSQL (to reflect SQL support)
1997  → PostgreSQL 6.0 - first community-managed release
2000  → PostgreSQL 7.0 - Introduced Write-Ahead Logging (WAL) & Foreign Keys
2005  → PostgreSQL 8.0 - First version to natively support Microsoft Windows
2010  → PostgreSQL 9.0 - streaming replication and Hot Standby added
2017  → PostgreSQL 10 - logical replication, partitioning
2019  → PostgreSQL 12 - major performance improvements
2021  → PostgreSQL 14 - improvements for heavy workloads
2023  → PostgreSQL 16 - parallel query improvements, logical replication enhancements
```

> **Definition:** PostgreSQL (also called "Postgres") is a free, open-source, advanced object-relational database system that uses and extends the SQL language. It has been actively developed for over 35 years.

---

### <span style="color:#2E86AB">Why PostgreSQL is Popular</span>

#### <span style="color:#5B8DB8">1. Fully ACID Compliant</span>
Every transaction in PostgreSQL guarantees Atomicity, Consistency, Isolation, and Durability - critical for applications that can't afford data loss or corruption.

#### <span style="color:#5B8DB8">2. Extremely Standards Compliant</span>
PostgreSQL follows the SQL standard more closely than most databases, making it easier to port SQL knowledge from other systems.

#### <span style="color:#5B8DB8">3. Extensible</span>
You can define your own:
- Data types
- Functions and operators
- Index methods
- Procedural languages (PL/pgSQL, PL/Python, PL/JavaScript, etc.)

#### <span style="color:#5B8DB8">4. Advanced Data Types</span>
Beyond standard SQL, PostgreSQL natively supports:
- `JSONB` - binary JSON with indexing
- Arrays
- `hstore` - key-value pairs
- Range types
- Network address types (`INET`, `CIDR`)
- Geometric types
- UUID

#### <span style="color:#5B8DB8">5. Concurrency Without Read Locks</span>
PostgreSQL uses **MVCC (Multi-Version Concurrency Control)** which means readers never block writers and writers never block readers - critical for high-traffic applications.

#### <span style="color:#5B8DB8">6. Full-Text Search Built-In</span>
No external search engine needed for basic search functionality.

#### <span style="color:#5B8DB8">7. Reliable & Battle-Tested</span>
Used by companies like Instagram, Reddit, Spotify, GitHub, Twitch, and thousands more at massive scale.

#### <span style="color:#5B8DB8">8. Open Source & Free</span>
PostgreSQL uses the **PostgreSQL License** (similar to MIT/BSD) - free to use for any purpose, including commercial.

---

### <span style="color:#2E86AB">PostgreSQL vs Other Databases</span>

| Feature | PostgreSQL | MySQL | SQLite | Oracle |
|:---|:---:|:---:|:---:|:---:|
| Open Source | ✓ | ✓ | ✓ | ✗ |
| ACID Compliant | ✓ | ✓ (InnoDB) | ✓ | ✓ |
| JSONB Support | ✓ | Partial | ✗ | ✓ |
| Full-Text Search | ✓ | ✓ | ✗ | ✓ |
| Window Functions | ✓ | ✓ | Partial | ✓ |
| Extensibility | Excellent | Limited | None | Limited |
| Best for | General purpose | Web apps | Embedded | Enterprise |

---

## <span style="color:#1565C0">1.5 Installing PostgreSQL on Your OS</span>

### <span style="color:#2E86AB">What Gets Installed</span>

When you install PostgreSQL, you get:
- `postgres` - the database server (always running as a background service)
- `psql` - the command-line client
- `pg_dump`, `pg_restore` - backup tools
- `pgAdmin` (optional GUI)

---

### <span style="color:#2E86AB">Installing on macOS</span>

**Option 1 - Homebrew (recommended)**
```bash
# Install Homebrew first if not present
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install PostgreSQL
brew install postgresql@16

# Start the service
brew services start postgresql@16

# Verify installation
psql --version
```

**Option 2 - Postgres.app**
Download from [postgresapp.com](https://postgresapp.com) - a menu bar app that manages the server visually. Good for beginners on Mac.

---

### <span style="color:#2E86AB">Installing on Ubuntu / Debian (Linux)</span>

```bash
# Update package list
sudo apt update

# Install PostgreSQL
sudo apt install postgresql postgresql-contrib

# Check service status
sudo systemctl status postgresql

# Start if not running
sudo systemctl start postgresql

# Enable on boot
sudo systemctl enable postgresql

# Verify
psql --version
```

---

### <span style="color:#2E86AB">Installing on Windows</span>

1. Download the installer from [postgresql.org/download/windows](https://www.postgresql.org/download/windows/)
2. Run the `.exe` installer
3. Choose components: PostgreSQL Server, pgAdmin 4, Command Line Tools
4. Set a password for the default `postgres` superuser
5. Set port (default: `5432` - leave as is)
6. Complete installation
7. PostgreSQL will start automatically as a Windows Service

---

### <span style="color:#2E86AB">Default Setup After Install</span>

After installation, PostgreSQL creates:

| Item | Value |
|:---|:---|
| Default superuser | `postgres` |
| Default port | `5432` |
| Default database | `postgres` |
| Data directory (Linux) | `/var/lib/postgresql/<version>/main` |
| Data directory (macOS) | `/usr/local/var/postgresql@16` |
| Config file | `postgresql.conf` (inside data directory) |

---

### <span style="color:#2E86AB">Switching to the postgres User (Linux)</span>

```bash
# Switch to the postgres OS user
sudo -i -u postgres

# Open psql
psql

# Or do both in one command
sudo -u postgres psql
```

On Linux, PostgreSQL uses **peer authentication** by default - meaning you must be the `postgres` OS user to connect as the `postgres` database superuser.

---

## <span style="color:#1565C0">1.6 pgAdmin (GUI) Setup and Walkthrough</span>

### <span style="color:#2E86AB">What is pgAdmin?</span>

> **Definition:** pgAdmin is the official open-source GUI (Graphical User Interface) tool for PostgreSQL. It lets you manage databases, run queries, view table structures, monitor activity, and more - all without using the command line.

---

### <span style="color:#2E86AB">Installing pgAdmin</span>

**macOS / Windows:** Download from [pgadmin.org](https://www.pgadmin.org/download/) or install alongside PostgreSQL.

**Ubuntu/Linux:**
```bash
# Add the pgAdmin repo
curl https://www.pgadmin.org/static/packages_pgadmin_org.pub | sudo apt-key add -
sudo sh -c 'echo "deb https://ftp.postgresql.org/pub/pgadmin/pgadmin4/apt/$(lsb_release -cs) pgadmin4 main" > /etc/apt/sources.list.d/pgadmin4.list'

sudo apt update
sudo apt install pgadmin4-desktop
```

---

### <span style="color:#2E86AB">Connecting to a Server in pgAdmin</span>

```
1. Open pgAdmin
2. Right-click "Servers" in the left panel → Register → Server
3. Fill in:
   - Name:     My Local Server  (any label you want)
   - Host:     localhost
   - Port:     5432
   - Username: postgres
   - Password: (the password you set during install)
4. Click Save
```

---

### <span style="color:#2E86AB">pgAdmin Interface Overview</span>

```
Left Panel (Object Tree)
├── Servers
│   └── My Local Server
│       ├── Databases
│       │   ├── postgres  (default DB)
│       │   └── your_db
│       ├── Login/Group Roles
│       └── Tablespaces
```

| Panel | Purpose |
|:---|:---|
| Object Tree (left) | Navigate server, databases, schemas, tables |
| Query Tool | Write and run SQL queries |
| Properties Tab | View columns, constraints, indexes of a table |
| Statistics Tab | See row counts, table size, cache hits |
| Dashboard | Monitor active connections, transactions |

---

### <span style="color:#2E86AB">Running a Query in pgAdmin</span>

```
1. Expand: Servers → My Local Server → Databases → postgres
2. Click on the database you want to query
3. Click the Query Tool button (or press Alt+Shift+Q)
4. Type SQL in the editor pane
5. Press F5 (or click the Run button ▶) to execute
6. Results appear in the Data Output panel below
```

---

## <span style="color:#1565C0">1.7 psql CLI - Connecting, Basic Usage</span>

### <span style="color:#2E86AB">What is psql?</span>

> **Definition:** `psql` is the official interactive terminal (command-line client) for PostgreSQL. It lets you connect to a PostgreSQL server and run SQL commands, meta-commands, and scripts directly from the terminal.

psql is the most reliable tool for working with PostgreSQL - faster than GUI for scripting, migrations, and administration.

---

### <span style="color:#2E86AB">Connecting with psql</span>

**Basic connection:**
```bash
psql -U username -d database_name
```

**Full connection options:**
```bash
psql -h hostname -p port -U username -d database_name
```

| Flag | Meaning | Default |
|:---:|:---|:---|
| `-h` | Host (server address) | `localhost` |
| `-p` | Port | `5432` |
| `-U` | Username | current OS user |
| `-d` | Database name | same as username |
| `-W` | Force password prompt | - |

**Common connection examples:**
```bash
# Connect as postgres user to default db
psql -U postgres

# Connect to a specific database
psql -U postgres -d myapp

# Connect to a remote server
psql -h 192.168.1.100 -p 5432 -U alice -d production

# Using a connection string (URL format)
psql postgresql://username:password@localhost:5432/mydb
```

---

### <span style="color:#2E86AB">The psql Prompt</span>

After connecting, you see a prompt like:

```
psql (16.2)
Type "help" for help.

postgres=#
```

| Prompt | Meaning |
|:---|:---|
| `postgres=#` | Connected to `postgres` db, you are a superuser |
| `mydb=#` | Connected to `mydb` database |
| `mydb=>` | Connected, you are a regular user (not superuser) |
| `mydb-#` | You typed an incomplete command, press Enter to continue or `;` to end |
| `mydb'#` | Inside an unclosed single quote string |

---

### <span style="color:#2E86AB">Running SQL in psql</span>

SQL statements must end with a **semicolon** `;` to execute:

```sql
-- This will NOT run yet (no semicolon):
SELECT * FROM users

-- This runs immediately:
SELECT * FROM users;

-- Multi-line is fine - only runs when ; is hit:
SELECT
  id,
  name,
  email
FROM users
WHERE age > 25;
```

---

### <span style="color:#2E86AB">Running a SQL File from psql</span>

```bash
# From outside psql (terminal)
psql -U postgres -d mydb -f /path/to/script.sql

# From inside psql
\i /path/to/script.sql
```

---

### <span style="color:#2E86AB">Useful psql Tips</span>

```bash
# Clear screen
\! clear        # Linux/Mac
\! cls          # Windows

# Exit psql
\q
# or press Ctrl+D

# Get help on SQL commands
\h SELECT
\h CREATE TABLE

# Get help on psql meta-commands
\?
```

---

## <span style="color:#1565C0">1.8 psql Meta-Commands</span>

> **Definition:** Meta-commands (also called backslash commands) are psql-specific commands that start with `\`. They are **not** SQL - they are instructions to psql itself and do NOT end with a semicolon.

---

### <span style="color:#2E86AB">Database & Connection Commands</span>

| Command | What it does |
|:---|:---|
| `\l` or `\list` | List all databases |
| `\l+` | List databases with size, owner, description |
| `\c dbname` | Connect (switch) to a database |
| `\c dbname username` | Connect to a db as a different user |
| `\conninfo` | Show current connection info |

```sql
-- List all databases
\l

-- Switch to a database called 'shop'
\c shop

-- Check current connection
\conninfo
-- Output: You are connected to database "shop" as user "postgres" via socket in "/tmp" at port "5432".
```

---

### <span style="color:#2E86AB">Table & Schema Inspection Commands</span>

| Command | What it does |
|:---|:---|
| `\dt` | List all tables in current schema |
| `\dt schema.*` | List tables in a specific schema |
| `\dt+` | List tables with size and description |
| `\d tablename` | Describe a table - columns, types, constraints |
| `\d+ tablename` | Describe with extra detail (storage, stats) |
| `\dn` | List all schemas |
| `\dn+` | List schemas with permissions |
| `\di` | List indexes |
| `\dv` | List views |
| `\dm` | List materialized views |
| `\df` | List functions |
| `\ds` | List sequences |

```sql
-- See all tables
\dt

-- Inspect the 'users' table structure
\d users

-- List all schemas
\dn
```

**Example output of `\d users`:**
```
                        Table "public.users"
   Column   |          Type          | Nullable |      Default
------------+------------------------+----------+-------------------
 id         | integer                | not null | nextval('users_id_seq')
 name       | character varying(100) | not null |
 email      | character varying(255) | not null |
 created_at | timestamp              |          | now()
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
    "users_email_key" UNIQUE CONSTRAINT, btree (email)
```

---

### <span style="color:#2E86AB">User & Role Commands</span>

| Command | What it does |
|:---|:---|
| `\du` | List all roles/users |
| `\du+` | List with extra info |
| `\du username` | Info about a specific user |

```sql
-- List all roles
\du
```

---

### <span style="color:#2E86AB">Output & Formatting Commands</span>

| Command | What it does |
|:---|:---|
| `\x` | Toggle expanded display (vertical format) |
| `\x on` / `\x off` | Explicitly set expanded mode |
| `\a` | Toggle aligned/unaligned output |
| `\t` | Toggle headers and row counts (tuples only) |
| `\o filename` | Send output to a file |
| `\o` | Stop sending to file (back to screen) |
| `\timing` | Toggle query execution time display |
| `\pset` | Set output formatting options |

```sql
-- Turn on expanded mode (better for wide rows)
\x

-- Now each row displays as key: value
SELECT * FROM users WHERE id = 1;
-- Output:
-- -[ RECORD 1 ]--------
-- id         | 1
-- name       | Alice
-- email      | alice@mail.com

-- Show query timing
\timing
SELECT COUNT(*) FROM large_table;
-- Output: Time: 42.318 ms
```

---

### <span style="color:#2E86AB">File & Script Commands</span>

| Command | What it does |
|:---|:---|
| `\i filename` | Execute SQL from a file |
| `\ir filename` | Execute file relative to current file location |
| `\e` | Open current query in your text editor |
| `\ef functionname` | Edit a function in your editor |

---

### <span style="color:#2E86AB">Help Commands</span>

| Command | What it does |
|:---|:---|
| `\?` | List all psql meta-commands |
| `\h` | List all SQL commands |
| `\h COMMAND` | Get help on a specific SQL command |
| `help` | General help |

---

### <span style="color:#2E86AB">Exit</span>

| Command | What it does |
|:---|:---|
| `\q` | Quit psql |
| `Ctrl + D` | Same as `\q` |

---

## <span style="color:#1565C0">1.9 What is a Schema - Database vs Schema vs Table</span>

### <span style="color:#2E86AB">The Three-Level Hierarchy</span>

```
PostgreSQL Server (instance)
└── Database  (e.g., "shop_db")
    └── Schema  (e.g., "public", "inventory", "billing")
        └── Table  (e.g., "users", "products", "orders")
```

Think of it as:
```
Server  =  a city
Database  =  a building in that city
Schema  =  a floor inside that building
Table  =  a room on that floor
```

---

### <span style="color:#2E86AB">Database</span>

> **Definition:** A Database is a completely separate container of data within a PostgreSQL server. Each database is isolated - you cannot directly join tables across different databases.

- A PostgreSQL server can host many databases
- Each database has its own users, schemas, and permissions
- You connect to one database at a time

```sql
-- See your current database
SELECT current_database();

-- Each database is a completely separate environment
-- You CAN'T do this:
SELECT * FROM database1.schema.table JOIN database2.schema.table ...
-- (Cross-database queries require Foreign Data Wrappers)
```

---

### <span style="color:#2E86AB">Schema</span>

> **Definition:** A Schema is a named namespace inside a database. It is used to organize tables, views, functions, and other objects into logical groups. Multiple schemas can exist within one database.

- Every database starts with a default schema: **`public`**
- Every table you create without specifying a schema goes into `public`
- Schemas allow you to organize and separate concerns within the same database

**Common use cases for multiple schemas:**

| Schema | Contains |
|:---|:---|
| `public` | General-purpose tables |
| `inventory` | Stock, warehouse tables |
| `billing` | Invoice, payment tables |
| `auth` | Users, roles, sessions |
| `audit` | Logs, change tracking |

```sql
-- Referencing a table in a specific schema:
SELECT * FROM inventory.products;
SELECT * FROM billing.invoices;

-- vs referencing in the default public schema:
SELECT * FROM products;  -- same as public.products
```

---

### <span style="color:#2E86AB">The search_path</span>

> **Definition:** `search_path` is a PostgreSQL setting that tells the server which schemas to look in (and in what order) when a table name is used without a schema prefix.

```sql
-- Show current search_path
SHOW search_path;
-- Output: "$user", public
-- Meaning: first look in a schema named after the current user, then in public

-- Change search_path for the session
SET search_path TO inventory, public;

-- Now this:
SELECT * FROM products;
-- First looks in inventory.products, then public.products
```

---

### <span style="color:#2E86AB">Table</span>

> **Definition:** A Table is a structured object that stores data as rows and columns within a schema. It is the primary unit of data storage in a relational database.

```
Summary:
  1 Server  →  can hold  → many Databases
  1 Database  →  can hold  → many Schemas
  1 Schema  →  can hold  → many Tables
  1 Table  →  can hold  → many Rows
```

---

## <span style="color:#1565C0">1.10 CREATE DATABASE & DROP DATABASE</span>

### <span style="color:#2E86AB">CREATE DATABASE</span>

```sql
-- Simplest form
CREATE DATABASE myapp;

-- With options
CREATE DATABASE myapp
  OWNER = alice
  ENCODING = 'UTF8'
  LC_COLLATE = 'en_US.UTF-8'
  LC_CTYPE = 'en_US.UTF-8'
  TEMPLATE = template0
  CONNECTION LIMIT = 100;
```

| Option | Meaning | Default |
|:---|:---|:---|
| `OWNER` | Who owns the database | Current user |
| `ENCODING` | Character encoding | Server default (usually UTF8) |
| `LC_COLLATE` | String sort order | Server default |
| `LC_CTYPE` | Character classification | Server default |
| `TEMPLATE` | Template to copy from | `template1` |
| `CONNECTION LIMIT` | Max simultaneous connections | `-1` (unlimited) |

---

### <span style="color:#2E86AB">Templates in PostgreSQL</span>

> **Definition:** When PostgreSQL creates a new database, it **copies** (clones) an existing template database. Two templates exist by default.

| Template | Description |
|:---|:---|
| `template1` | Default template. Any objects you add here appear in all new databases |
| `template0` | Clean, unmodified backup template. Use this when specifying custom encodings |

```sql
-- Always use template0 when you need a specific encoding
CREATE DATABASE french_db
  ENCODING = 'UTF8'
  LC_COLLATE = 'fr_FR.UTF-8'
  LC_CTYPE = 'fr_FR.UTF-8'
  TEMPLATE = template0;
```

---

### <span style="color:#2E86AB">Connecting to a Database</span>

```sql
-- In psql, switch to a different database
\c myapp

-- In psql with a different user
\c myapp alice

-- Verify
SELECT current_database();
```

---

### <span style="color:#2E86AB">Renaming a Database</span>

```sql
-- Rename a database (no one can be connected to it)
ALTER DATABASE myapp RENAME TO myapp_v2;

-- Change the owner
ALTER DATABASE myapp OWNER TO bob;

-- Change connection limit
ALTER DATABASE myapp CONNECTION LIMIT 50;
```

---

### <span style="color:#2E86AB">DROP DATABASE</span>

```sql
-- Drop a database (permanently deletes everything inside)
DROP DATABASE myapp;

-- Safe version - no error if it doesn't exist
DROP DATABASE IF EXISTS myapp;

-- Force close all existing connections before dropping (PostgreSQL 13+)
DROP DATABASE myapp WITH (FORCE);
```

> **Note:** You cannot drop a database you are currently connected to. Switch to another database first (`\c postgres`), then drop it.

---

### <span style="color:#2E86AB">Listing Databases</span>

```sql
-- In psql
\l

-- In SQL
SELECT datname FROM pg_database;

-- With more details
SELECT
  datname AS database_name,
  pg_size_pretty(pg_database_size(datname)) AS size,
  datdba::regrole AS owner,
  pg_encoding_to_char(encoding) AS encoding
FROM pg_database
ORDER BY datname;
```

---

## <span style="color:#1565C0">1.11 CREATE SCHEMA & DROP SCHEMA</span>

### <span style="color:#2E86AB">CREATE SCHEMA</span>

```sql
-- Basic schema creation
CREATE SCHEMA inventory;

-- Create schema with a specific owner
CREATE SCHEMA billing AUTHORIZATION alice;

-- Create schema only if it doesn't exist
CREATE SCHEMA IF NOT EXISTS analytics;

-- Create schema and a table in it in one statement
CREATE SCHEMA shop
  CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    price NUMERIC(10, 2)
  )
  CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL
  );
```

---

### <span style="color:#2E86AB">Using Schemas</span>

```sql
-- Create a table in a specific schema
CREATE TABLE inventory.products (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  stock_count INT DEFAULT 0
);

-- Query with schema prefix
SELECT * FROM inventory.products;

-- Insert into a specific schema's table
INSERT INTO inventory.products (name, stock_count)
VALUES ('Widget A', 100);
```

---

### <span style="color:#2E86AB">The public Schema</span>

```sql
-- These are identical - public is the default schema
CREATE TABLE users (...);
CREATE TABLE public.users (...);

-- Check search_path
SHOW search_path;
-- "$user", public

-- Change search path for current session
SET search_path TO inventory, public;
-- Now unqualified table names look in inventory first

-- Change search path permanently for a user
ALTER ROLE alice SET search_path TO billing, public;

-- Change search path permanently for a database
ALTER DATABASE myapp SET search_path TO myschema, public;
```

---

### <span style="color:#2E86AB">Listing Schemas</span>

```sql
-- In psql
\dn
\dn+   -- with permissions

-- In SQL
SELECT schema_name, schema_owner
FROM information_schema.schemata
ORDER BY schema_name;

-- From pg_namespace (system catalog)
SELECT nspname AS schema_name
FROM pg_namespace
WHERE nspname NOT LIKE 'pg_%'  -- exclude system schemas
  AND nspname <> 'information_schema'
ORDER BY nspname;
```

---

### <span style="color:#2E86AB">ALTER SCHEMA</span>

```sql
-- Rename a schema
ALTER SCHEMA inventory RENAME TO warehouse;

-- Change the owner
ALTER SCHEMA billing OWNER TO finance_team;
```

---

### <span style="color:#2E86AB">DROP SCHEMA</span>

```sql
-- Drop an EMPTY schema only
DROP SCHEMA inventory;

-- Drop schema and everything inside it (tables, views, functions)
DROP SCHEMA inventory CASCADE;

-- Safe version
DROP SCHEMA IF EXISTS inventory CASCADE;
```

> **Warning:** `DROP SCHEMA ... CASCADE` is irreversible and will delete all tables, views, functions, sequences, and other objects inside that schema. Always double-check before running.

---

## <span style="color:#1565C0">1.12 Understanding the PostgreSQL System Catalog</span>

### <span style="color:#2E86AB">What is the System Catalog?</span>

> **Definition:** The System Catalog is a collection of special system tables and views that PostgreSQL uses internally to store metadata - information about databases, tables, columns, indexes, users, permissions, and everything else in the server.

Every time you create a table, add a column, define an index, or grant a permission - PostgreSQL records that information in the system catalog automatically.

There are two ways to access this metadata:

| Access Method | Description |
|:---|:---|
| `pg_*` tables | PostgreSQL-specific native system catalog tables |
| `information_schema` | SQL-standard views, more portable across databases |

---

### <span style="color:#2E86AB">pg_catalog - Native System Tables</span>

These are raw PostgreSQL internal tables. They are faster and more detailed, but PostgreSQL-specific.

#### <span style="color:#5B8DB8">pg_database - All Databases</span>

```sql
SELECT datname, pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

#### <span style="color:#5B8DB8">pg_tables - All Tables</span>

```sql
-- All user-created tables (exclude system tables)
SELECT schemaname, tablename, tableowner
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, tablename;
```

#### <span style="color:#5B8DB8">pg_columns - All Columns</span>

```sql
-- Columns of a specific table
SELECT column_name, data_type, character_maximum_length, is_nullable
FROM information_schema.columns
WHERE table_name = 'users'
ORDER BY ordinal_position;
```

#### <span style="color:#5B8DB8">pg_indexes - All Indexes</span>

```sql
SELECT schemaname, tablename, indexname, indexdef
FROM pg_indexes
WHERE schemaname = 'public'
ORDER BY tablename;
```

#### <span style="color:#5B8DB8">pg_class - Tables, Views, Indexes, Sequences</span>

```sql
-- All tables (relkind = 'r' means regular table)
SELECT relname AS name, relkind AS type
FROM pg_class
WHERE relkind IN ('r', 'v', 'i', 'S')  -- r=table, v=view, i=index, S=sequence
  AND relnamespace = 'public'::regnamespace
ORDER BY relkind, relname;
```

#### <span style="color:#5B8DB8">pg_namespace - All Schemas</span>

```sql
SELECT nspname AS schema_name
FROM pg_namespace
ORDER BY nspname;
```

#### <span style="color:#5B8DB8">pg_roles / pg_user - Users & Roles</span>

```sql
SELECT rolname, rolsuper, rolinherit, rolcreatedb, rolcanlogin
FROM pg_roles
ORDER BY rolname;
```

#### <span style="color:#5B8DB8">pg_stat_activity - Currently Running Queries</span>

```sql
-- See all active database connections and queries
SELECT pid, usename, datname, state, query, query_start
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start;
```

---

### <span style="color:#2E86AB">information_schema - SQL Standard Views</span>

`information_schema` is a set of read-only views that provide metadata in a standardized way (same across PostgreSQL, MySQL, SQL Server).

#### <span style="color:#5B8DB8">information_schema.tables</span>

```sql
-- All user tables
SELECT table_schema, table_name, table_type
FROM information_schema.tables
WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY table_schema, table_name;
```

#### <span style="color:#5B8DB8">information_schema.columns</span>

```sql
-- All columns in a table
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_schema = 'public'
  AND table_name = 'users'
ORDER BY ordinal_position;
```

#### <span style="color:#5B8DB8">information_schema.table_constraints</span>

```sql
-- All constraints (primary keys, foreign keys, unique, check)
SELECT constraint_name, constraint_type, table_name
FROM information_schema.table_constraints
WHERE table_schema = 'public'
ORDER BY table_name, constraint_type;
```

#### <span style="color:#5B8DB8">information_schema.referential_constraints</span>

```sql
-- All foreign key relationships
SELECT
  tc.table_name AS child_table,
  kcu.column_name AS child_column,
  ccu.table_name AS parent_table,
  ccu.column_name AS parent_column
FROM information_schema.table_constraints AS tc
JOIN information_schema.key_column_usage AS kcu
  ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage AS ccu
  ON ccu.constraint_name = tc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
  AND tc.table_schema = 'public';
```

---

### <span style="color:#2E86AB">pg_catalog vs information_schema - When to Use Which</span>

| Use Case | Prefer |
|:---|:---|
| Quick inspection, admin tasks | `pg_catalog` (`pg_tables`, `pg_indexes`) |
| Portable code that works across databases | `information_schema` |
| Detailed PostgreSQL-specific metadata | `pg_catalog` |
| Finding column details | Either - `information_schema.columns` is more readable |
| Monitoring live activity | `pg_stat_activity` (pg_catalog only) |

---

### <span style="color:#2E86AB">Useful System Functions</span>

```sql
-- Current user
SELECT current_user;

-- Current database
SELECT current_database();

-- Current schemas in search_path
SELECT current_schemas(true);

-- PostgreSQL server version
SELECT version();

-- Table size
SELECT pg_size_pretty(pg_total_relation_size('users'));

-- Database size
SELECT pg_size_pretty(pg_database_size('myapp'));

-- Check if a table exists
SELECT EXISTS (
  SELECT 1 FROM information_schema.tables
  WHERE table_schema = 'public'
    AND table_name = 'users'
);
```


