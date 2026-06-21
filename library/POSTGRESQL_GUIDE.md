# PostgreSQL — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Developers and aspiring DBAs who want to learn PostgreSQL deeply — from `CREATE TABLE` to query-plan tuning, MVCC, replication, and AI vector search — without needing to look anything up online. Every concept comes with runnable, heavily commented SQL you can paste straight into `psql` and execute. Sections are tagged **[Beginner]**, **[Intermediate]**, or **[Advanced]** so you can navigate by skill level.
>
> **Version note:** This guide targets **PostgreSQL 17** (the current stable release in 2026). PG 17 brought `JSON_TABLE` and a fuller SQL/JSON suite, incremental physical backups (`pg_basebackup --incremental` + `pg_combinebackup`), a faster and lower-memory `VACUUM`, improved logical-replication slot synchronization (failover slots), and `MERGE ... RETURNING`. Where **PostgreSQL 18** (released late 2025) is relevant — e.g. asynchronous I/O, `uuidv7()`, virtual generated columns, and skip-scan for B-tree indexes — it is called out with **⚡ Version note**. Always confirm exact behaviour against the official docs for your installed `SELECT version();`.

---

## Table of Contents

1. [Installation, psql & First Connection](#1-installation-psql--first-connection)
2. [Data Types in Depth](#2-data-types-in-depth)
3. [DDL — Tables, Constraints, Schemas, Sequences](#3-ddl--tables-constraints-schemas-sequences)
4. [DML & Queries — INSERT, UPDATE, DELETE, SELECT](#4-dml--queries--insert-update-delete-select)
5. [Joins & Set Operations](#5-joins--set-operations)
6. [Aggregation — GROUP BY, HAVING, GROUPING SETS](#6-aggregation--group-by-having-grouping-sets)
7. [Subqueries & CTEs (incl. Recursive)](#7-subqueries--ctes-incl-recursive)
8. [Window Functions in Depth](#8-window-functions-in-depth)
9. [Working with JSON & JSONB](#9-working-with-json--jsonb)
10. [Arrays & Full-Text Search](#10-arrays--full-text-search)
11. [Indexing Deeply](#11-indexing-deeply)
12. [EXPLAIN, Query Plans & Finding Slow Queries](#12-explain-query-plans--finding-slow-queries)
13. [Transactions, MVCC & Concurrency](#13-transactions-mvcc--concurrency)
14. [Functions, Procedures & Triggers (PL/pgSQL)](#14-functions-procedures--triggers-plpgsql)
15. [Views & Materialized Views](#15-views--materialized-views)
16. [Performance Tuning, VACUUM & Partitioning](#16-performance-tuning-vacuum--partitioning)
17. [Backup & Restore (incl. PITR & Incremental)](#17-backup--restore-incl-pitr--incremental)
18. [Replication & High Availability](#18-replication--high-availability)
19. [Security — Roles, Privileges, RLS, SSL](#19-security--roles-privileges-rls-ssl)
20. [Extensions — pgvector, PostGIS, pg_trgm & more](#20-extensions--pgvector-postgis-pg_trgm--more)
21. [Connecting from Apps — Node.js & Go](#21-connecting-from-apps--nodejs--go)
22. [Gotchas & Best Practices](#22-gotchas--best-practices)
23. [Study Path & Build-to-Learn Projects](#23-study-path--build-to-learn-projects)

---

## 1. Installation, psql & First Connection

**[Beginner]**

### What PostgreSQL is

PostgreSQL ("Postgres") is a free, open-source, object-relational database management system (RDBMS) known for strict standards compliance, extensibility, and an extremely capable optimizer. It speaks SQL, supports ACID transactions, and ships with advanced features other databases charge for: rich data types (JSONB, arrays, ranges, geometric types), powerful indexing (GIN, GiST, BRIN), window functions, CTEs, full-text search, and a pluggable extension system.

A running PostgreSQL is a **server process** (often called the *postmaster* / *postgres* process) that manages one or more **databases**. You talk to it with a **client** — the built-in `psql` terminal, a GUI like pgAdmin/DBeaver, or your application's driver.

### Install with Docker (recommended for learning)

Docker is the fastest way to get a clean, throwaway PostgreSQL without touching your OS.

```bash
# Pull and run PostgreSQL 17 in the background.
#  --name        : container name so you can reference it later
#  -e POSTGRES_PASSWORD : sets the password for the default 'postgres' superuser (REQUIRED)
#  -e POSTGRES_USER     : (optional) name of the superuser to create (default: postgres)
#  -e POSTGRES_DB       : (optional) name of a database to create on first boot
#  -p 5432:5432         : map container port 5432 to your host's 5432
#  -v pgdata:/var/lib/postgresql/data : persist data in a named volume so it survives restarts
#  -d                   : detached (background)
docker run --name pg17 \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_USER=app \
  -e POSTGRES_DB=appdb \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  -d postgres:17

# Check it's running
docker ps

# Open a psql shell INSIDE the container as the 'app' user against 'appdb'
docker exec -it pg17 psql -U app -d appdb

# Tail the server logs (useful for debugging connection/auth issues)
docker logs -f pg17

# Stop / start / remove
docker stop pg17
docker start pg17
docker rm -f pg17          # remove container (named volume 'pgdata' is kept!)
docker volume rm pgdata    # delete the persisted data
```

A reproducible `docker-compose.yml`:

```yaml
# docker-compose.yml — run with `docker compose up -d`
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    # Health check so dependent services wait until PG is actually accepting connections
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 5s
      timeout: 3s
      retries: 5
volumes:
  pgdata:
```

### Install natively

```bash
# --- Debian / Ubuntu (uses the PGDG apt repo for the latest version) ---
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh   # adds PGDG repo
sudo apt install -y postgresql-17
# The 'postgres' OS user is created; switch to it and run psql:
sudo -u postgres psql

# --- macOS (Homebrew) ---
brew install postgresql@17
brew services start postgresql@17   # run as a background service
psql postgres                       # connects as your macOS username

# --- Windows ---
# Use the EDB installer from enterprisedb.com, or:
#   winget install PostgreSQL.PostgreSQL.17
# Then use the bundled "SQL Shell (psql)" Start-menu item, or pgAdmin.
```

After install, the server listens on TCP port **5432** by default. The default superuser is **`postgres`**, and a default database **`postgres`** exists.

### Connecting with psql

`psql` is the standard interactive terminal. It accepts SQL plus **meta-commands** that start with a backslash (`\`).

```bash
# Connection options (any combination):
#   -h host   -p port   -U user   -d database   -W (force password prompt)
psql -h localhost -p 5432 -U app -d appdb

# Connection string / URI form (handy for one-liners and apps):
psql "postgresql://app:secret@localhost:5432/appdb"
psql "postgresql://app@localhost/appdb?sslmode=require"

# Environment variables psql/libpq honor (avoids retyping flags):
export PGHOST=localhost PGPORT=5432 PGUSER=app PGDATABASE=appdb PGPASSWORD=secret
psql   # now connects with no flags
```

> **Security tip:** Don't put passwords in `PGPASSWORD` on shared machines. Use a `~/.pgpass` file (`chmod 600`) with lines like `hostname:port:database:username:password`, and libpq reads it automatically.

### Essential psql meta-commands

```text
\?              -- list ALL psql meta-commands (your cheat sheet)
\h              -- SQL help index;  \h CREATE TABLE  -- help for a specific statement
\l   (\list)    -- list databases
\c dbname       -- connect to another database (\connect)
\c dbname user  -- connect as a different user
\dn             -- list schemas
\dt             -- list tables in the search_path
\dt schema.*    -- list tables in a specific schema
\d tablename    -- describe a table (columns, types, indexes, constraints)
\d+ tablename   -- describe with MORE detail (storage, description, stats target)
\di             -- list indexes
\dv             -- list views
\dm             -- list materialized views
\ds             -- list sequences
\df             -- list functions
\dx             -- list installed extensions
\du             -- list roles/users and their attributes
\dp  (\z)       -- list table access privileges
\sf func_name   -- show the source of a function
\timing         -- toggle showing how long each query took
\x              -- toggle expanded (vertical) output — great for wide rows
\e              -- open the last query in your $EDITOR
\i file.sql     -- run SQL from a file
\o out.txt      -- send query output to a file
\copy ...       -- client-side bulk import/export (see DML section)
\set            -- list/define psql variables
\conninfo       -- show details of the current connection
\q              -- quit
```

A few quality-of-life settings for `~/.psqlrc` (auto-loaded by psql):

```text
\set QUIET 1
\timing on
\x auto
\set COMP_KEYWORD_CASE upper
\set HISTSIZE 5000
-- Make NULLs visible in output (default shows them as empty string):
\pset null '(null)'
\set QUIET 0
```

### Creating databases, users, and roles

In PostgreSQL a **role** is the unifying concept: a *user* is just a role with the `LOGIN` attribute, and a *group* is a role you grant other roles into. There's no separate "user" vs "group" object internally.

```sql
-- Create a login role (a "user") with a password:
CREATE ROLE app_user WITH LOGIN PASSWORD 'changeme';

-- Equivalent shorthand (CREATE USER == CREATE ROLE ... WITH LOGIN):
CREATE USER report_user WITH PASSWORD 'changeme';

-- A role with the power to create databases and other roles, but NOT a superuser:
CREATE ROLE devops WITH LOGIN PASSWORD 'x' CREATEDB CREATEROLE;

-- A group role (no login) you grant membership into for shared privileges:
CREATE ROLE readonly;                 -- no LOGIN attribute = cannot connect directly
GRANT readonly TO report_user;        -- report_user inherits readonly's privileges

-- Create a database owned by a specific role:
CREATE DATABASE shop
  OWNER app_user
  ENCODING 'UTF8'
  LC_COLLATE 'en_US.UTF-8'   -- text sort order
  LC_CTYPE  'en_US.UTF-8'    -- character classification
  TEMPLATE template0;        -- template0 lets you choose a different locale/encoding

-- Alter / drop:
ALTER ROLE app_user WITH PASSWORD 'newsecret';
ALTER ROLE app_user VALID UNTIL '2027-01-01';   -- password expiry
ALTER ROLE app_user CONNECTION LIMIT 20;        -- max concurrent connections
DROP ROLE report_user;
DROP DATABASE shop;          -- must disconnect everyone first
DROP DATABASE IF EXISTS shop WITH (FORCE);  -- PG13+: terminate other sessions and drop
```

Command-line equivalents (wrappers around the SQL):

```bash
createdb shop                 # create a database
createuser --interactive app  # create a role interactively
dropdb shop
dropuser app
```

> **Gotcha:** Connecting requires both (1) a role with `LOGIN` and the right password, and (2) a matching line in `pg_hba.conf` (Host-Based Authentication) that allows that user/database/source-IP/auth-method combination. If you can authenticate locally but not over TCP, check `pg_hba.conf` and `listen_addresses` in `postgresql.conf`.

---

## 2. Data Types in Depth

**[Beginner]**

Choosing the right type matters for correctness, storage, and index performance. Below are the types you'll actually use.

### Numeric types

| Type | Size | Range / notes |
|---|---|---|
| `smallint` (`int2`) | 2 bytes | -32,768 … 32,767 |
| `integer` (`int`, `int4`) | 4 bytes | ~±2.1 billion. The default choice for IDs/counts. |
| `bigint` (`int8`) | 8 bytes | ~±9.2 quintillion. Use for large IDs/counters. |
| `numeric(p, s)` / `decimal` | variable | **Exact** decimal; `p` total digits, `s` digits after point. Use for money. |
| `real` (`float4`) | 4 bytes | Inexact, 6 sig-digits. |
| `double precision` (`float8`) | 8 bytes | Inexact, 15 sig-digits. |
| `smallserial`/`serial`/`bigserial` | — | Auto-incrementing integer (legacy; prefer `GENERATED ... AS IDENTITY`). |

```sql
-- NEVER store money in float — rounding errors will bite you:
CREATE TABLE invoice (
  total numeric(12, 2) NOT NULL   -- exact: up to 10 digits before the point, 2 after
);
INSERT INTO invoice (total) VALUES (0.1 + 0.2);  -- stored exactly as 0.30, not 0.30000000004

-- Casting and rounding:
SELECT 10.0 / 3;                  -- numeric division: 3.3333333333333333
SELECT (10.0 / 3)::numeric(10,2); -- 3.33
SELECT round(2.5), round(3.5);    -- 3, 4 (round-half-away-from-zero for numeric)
```

### Character / text types

| Type | Notes |
|---|---|
| `text` | Unlimited length. **Preferred** for almost everything. |
| `varchar(n)` | Variable length with a max of `n` chars. Errors if you exceed `n`. |
| `char(n)` | Fixed length, **blank-padded** to `n`. Rarely a good idea. |

```sql
-- In PostgreSQL there is NO performance penalty for `text` vs `varchar`.
-- Use varchar(n) only when you genuinely need to ENFORCE a max length.
CREATE TABLE person (
  name  text NOT NULL,
  code  varchar(10)        -- e.g. an external code with a hard 10-char limit
);

-- Useful string functions:
SELECT length('héllo');                 -- 5 (characters, not bytes)
SELECT octet_length('héllo');           -- 6 (bytes in UTF-8)
SELECT upper('abc'), lower('ABC'), initcap('the quick fox');
SELECT trim('  hi  '), ltrim('xxhi','x'), rtrim('hixx','x');
SELECT substring('postgresql' FROM 1 FOR 4);    -- 'post'
SELECT split_part('a,b,c', ',', 2);             -- 'b'
SELECT 'Hello, ' || 'world';                    -- concatenation -> 'Hello, world'
SELECT format('User %s has %s points', 'Ana', 42);  -- printf-style
SELECT 'abc' ILIKE 'A%';                         -- case-insensitive LIKE -> true
SELECT regexp_replace('a1b2c3', '\d', '#', 'g'); -- 'a#b#c#'
SELECT 'foo bar baz' ~ '\mbar\M';                -- POSIX regex word-boundary match
```

> **Tip:** For case-insensitive *columns* (emails, usernames), see the `citext` extension in §20 — it makes equality and uniqueness case-insensitive automatically.

### Boolean

```sql
-- 'true' inputs: TRUE, 't', 'true', 'y', 'yes', 'on', '1'
-- 'false' inputs: FALSE, 'f', 'false', 'n', 'no', 'off', '0'
CREATE TABLE flag_demo (active boolean NOT NULL DEFAULT true);
SELECT true AND false, true OR false, NOT true;   -- f, t, f
-- Three-valued logic: comparisons with NULL yield NULL (treated as "unknown"):
SELECT true AND NULL;   -- NULL
SELECT NULL IS NULL;    -- t   (use IS / IS NOT for NULL checks)
```

### Date / time — and timezone handling (read this carefully)

| Type | Stores | Notes |
|---|---|---|
| `date` | calendar date | no time, no zone |
| `time` | time of day | no zone |
| `timestamp` (without time zone) | date+time | **no zone info** — a "wall clock" reading |
| `timestamptz` (with time zone) | date+time | stored as UTC; converted to session TZ on display |
| `interval` | a span of time | e.g. `'2 days 3 hours'` |

**The single most important rule:** prefer **`timestamptz`** for any moment in time. Despite the name, `timestamptz` does **not** store a timezone — it stores an absolute UTC instant. On input, the value is converted from the session's `TimeZone` to UTC; on output, UTC is converted back to the session's `TimeZone`. `timestamp` (without tz) stores the literal digits with no awareness of zone — two clients in different zones will misinterpret it.

```sql
SHOW TimeZone;                       -- the session timezone (e.g. 'UTC' or 'America/New_York')
SET TimeZone = 'UTC';                -- set for this session

-- Demonstrate the difference:
SELECT
  TIMESTAMP        '2026-06-21 12:00:00' AS naive,   -- "12:00" with no zone meaning
  TIMESTAMPTZ      '2026-06-21 12:00:00-04' AS aware; -- absolute instant (16:00 UTC)

-- "now" functions:
SELECT now();              -- timestamptz, transaction start time (stable in a txn)
SELECT clock_timestamp();  -- actual wall clock, changes within a transaction
SELECT current_date, current_time, current_timestamp;

-- Arithmetic with intervals:
SELECT now() + interval '7 days';
SELECT now() - timestamptz '2026-01-01';        -- an interval (age between two instants)
SELECT age(timestamptz '2000-01-01');           -- symbolic age in years/months/days

-- Truncation and extraction (great for reporting):
SELECT date_trunc('month', now());              -- first instant of this month
SELECT extract(dow FROM now());                 -- day of week (0=Sun..6=Sat)
SELECT extract(epoch FROM now());               -- Unix seconds as a double
SELECT to_char(now(), 'YYYY-MM-DD HH24:MI');    -- formatted text

-- Convert between zones explicitly with AT TIME ZONE:
SELECT now() AT TIME ZONE 'America/New_York';   -- timestamptz -> local wall clock (timestamp)
SELECT (TIMESTAMP '2026-06-21 09:00') AT TIME ZONE 'America/New_York';  -- naive -> timestamptz
```

> **Gotcha:** `AT TIME ZONE` is overloaded. Applied to a `timestamptz` it returns a `timestamp` (the wall clock in that zone). Applied to a `timestamp` it returns a `timestamptz` (interprets the naive value as being in that zone). Read it as "render this instant in zone X" vs "this wall-clock is in zone X".

### UUID

```sql
-- UUIDs are 128-bit identifiers, ideal when you need globally-unique,
-- non-guessable, client-generatable keys.
CREATE TABLE account (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid()  -- built-in since PG13 (pgcrypto in older)
);
SELECT gen_random_uuid();   -- random v4 UUID
```

⚡ **Version note (PG 18):** PostgreSQL 18 adds a built-in **`uuidv7()`** function. UUIDv7 embeds a timestamp prefix, so values are *time-sortable*. This dramatically reduces B-tree index fragmentation versus random v4 UUIDs (which insert all over the index), giving you UUID benefits with near-sequential insert locality. On PG17, you can use the `pg_uuidv7` extension or generate v7 in your app.

### JSON vs JSONB

| Type | Storage | Preserves key order/whitespace/dupes? | Indexable? | Speed |
|---|---|---|---|---|
| `json` | exact text copy | yes | no (only as text) | fast write, slow read |
| `jsonb` | parsed binary | no (canonicalized, dupes dropped) | **yes (GIN)** | slower write, fast read/query |

**Use `jsonb` 99% of the time.** Use plain `json` only when you must store the document byte-for-byte (e.g. signature verification of the original payload). Full operators and indexing are covered in §9.

```sql
CREATE TABLE event (id bigserial PRIMARY KEY, payload jsonb NOT NULL);
INSERT INTO event (payload) VALUES ('{"type":"click","x":10,"x":20}');
SELECT payload FROM event;   -- {"x": 20, "type": "click"}  (dup key dropped, keys reordered)
```

### Arrays

```sql
-- Any type can be an array. Declared with [] suffix.
CREATE TABLE article (
  id    bigserial PRIMARY KEY,
  title text NOT NULL,
  tags  text[]                  -- a 1-D array of text
);
INSERT INTO article (title, tags) VALUES
  ('Postgres tips', ARRAY['sql','db','postgres']),
  ('Hello',         '{greeting,intro}');           -- array literal syntax

SELECT title FROM article WHERE 'sql' = ANY(tags); -- contains 'sql'
SELECT title FROM article WHERE tags @> ARRAY['db','sql'];  -- contains all of these
SELECT tags[1] FROM article;                       -- arrays are 1-indexed!
SELECT array_length(tags, 1) FROM article;         -- length of 1st dimension
SELECT unnest(tags) FROM article WHERE id = 1;     -- expand array into rows
SELECT array_agg(title) FROM article;              -- collapse rows into an array
SELECT cardinality(tags) FROM article;             -- total element count
```

### Enums

```sql
-- An enum is a custom type with a fixed, ordered set of allowed text values.
CREATE TYPE order_status AS ENUM ('pending', 'paid', 'shipped', 'cancelled');

CREATE TABLE orders (
  id     bigserial PRIMARY KEY,
  status order_status NOT NULL DEFAULT 'pending'
);
INSERT INTO orders (status) VALUES ('paid');
-- Comparison respects the DECLARED order, not alphabetical:
SELECT 'pending'::order_status < 'shipped'::order_status;  -- true

-- Add a new value later (PG supports this without a table rewrite):
ALTER TYPE order_status ADD VALUE 'refunded' AFTER 'shipped';
```

> **Gotcha:** You cannot `DELETE` or `RENAME` an enum value's *position* easily, and removing a value requires recreating the type. If your set changes often, a lookup table with a foreign key is more flexible than an enum.

### Ranges

```sql
-- Range types represent a span with bounds. Built-in: int4range, int8range,
-- numrange, tsrange, tstzrange, daterange.
CREATE TABLE reservation (
  room   int NOT NULL,
  during tstzrange NOT NULL,
  -- EXCLUDE constraint: prevent overlapping bookings for the same room!
  -- Requires the btree_gist extension for the '=' operator on int.
  EXCLUDE USING gist (room WITH =, during WITH &&)
);
-- '[)' = inclusive lower, exclusive upper (the usual convention)
INSERT INTO reservation VALUES (1, tstzrange('2026-06-21 09:00', '2026-06-21 10:00', '[)'));
SELECT int4range(1, 10) @> 5;          -- contains 5? -> true
SELECT numrange(1,5) && numrange(4,9); -- overlap? -> true
SELECT upper(int4range(1,10)), lower(int4range(1,10));  -- 10, 1
```

### Network address types

```sql
CREATE TABLE access_log (
  client inet,        -- IPv4 or IPv6 host (optionally with subnet)
  net    cidr,        -- a network block, validates the mask
  mac    macaddr
);
INSERT INTO access_log VALUES ('192.168.1.50', '192.168.1.0/24', '08:00:2b:01:02:03');
SELECT inet '192.168.1.50' << cidr '192.168.1.0/24';  -- is contained by network? true
SELECT host(inet '10.0.0.1/8'), masklen(inet '10.0.0.1/8');  -- '10.0.0.1', 8
```

### bytea (binary data)

```sql
-- bytea stores raw binary (small blobs, hashes, thumbnails). For large files,
-- prefer object storage (S3) and store only a URL/key in the DB.
CREATE TABLE file_blob (id bigserial PRIMARY KEY, data bytea);
INSERT INTO file_blob (data) VALUES ('\xDEADBEEF');     -- hex input
SELECT encode(data, 'hex'), encode(data, 'base64') FROM file_blob;
SELECT length(data) FROM file_blob;   -- byte length
```

### Generated columns

```sql
-- A STORED generated column is computed from other columns and physically stored.
CREATE TABLE product (
  price    numeric(10,2) NOT NULL,
  qty      int NOT NULL,
  subtotal numeric(12,2) GENERATED ALWAYS AS (price * qty) STORED
);
INSERT INTO product (price, qty) VALUES (9.99, 3);
SELECT subtotal FROM product;   -- 29.97, computed automatically; you can't write to it
```

⚡ **Version note (PG 18):** PG 18 adds **`VIRTUAL`** generated columns (computed on read, not stored), which are now the default for `GENERATED ALWAYS AS`. They save disk and write cost at the price of compute on each read. On PG17 only `STORED` exists.

---

## 3. DDL — Tables, Constraints, Schemas, Sequences

**[Beginner]**

DDL (Data Definition Language) defines structure. DDL in PostgreSQL is **transactional** — you can wrap `CREATE`/`ALTER`/`DROP` in a `BEGIN ... COMMIT` and roll it back if anything fails (a feature MySQL lacks).

### CREATE TABLE with all the common constraints

```sql
CREATE TABLE customer (
  -- IDENTITY column: the modern replacement for serial (SQL-standard).
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,

  -- NOT NULL: column must always have a value.
  email       text NOT NULL,

  -- UNIQUE: no two rows may share this value (NULLs are allowed & considered distinct).
  -- A UNIQUE constraint automatically creates a supporting index.
  CONSTRAINT customer_email_key UNIQUE (email),

  -- DEFAULT: value used when INSERT omits the column.
  created_at  timestamptz NOT NULL DEFAULT now(),

  -- CHECK: a boolean expression that must hold for every row.
  age         int CHECK (age IS NULL OR age >= 0),
  status      text NOT NULL DEFAULT 'active'
              CHECK (status IN ('active', 'suspended', 'closed')),

  -- A named CHECK constraint reads better in error messages.
  CONSTRAINT email_has_at CHECK (email LIKE '%@%')
);

CREATE TABLE orders (
  id           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id  bigint NOT NULL,
  total        numeric(12,2) NOT NULL CHECK (total >= 0),
  placed_at    timestamptz NOT NULL DEFAULT now(),

  -- FOREIGN KEY: enforces referential integrity.
  --   ON DELETE CASCADE  : deleting a customer deletes their orders
  --   ON DELETE RESTRICT : block deleting a customer that still has orders (default-ish)
  --   ON DELETE SET NULL : set customer_id to NULL instead
  CONSTRAINT orders_customer_fk
    FOREIGN KEY (customer_id) REFERENCES customer(id) ON DELETE CASCADE
);
```

> **`GENERATED ALWAYS AS IDENTITY` vs `BY DEFAULT`:** `ALWAYS` forbids manual inserts into the column (you must use `OVERRIDING SYSTEM VALUE` to override). `BY DEFAULT` lets you supply your own value, which is handy for data migration but risks desyncing the sequence. Prefer `ALWAYS` for true surrogate keys.

> **Always index your foreign keys.** PostgreSQL automatically indexes the *referenced* (parent) PK, but **not** the *referencing* (child) FK column. Without an index on `orders.customer_id`, deleting a customer must scan the whole orders table, and joins are slow. See §22.

### Primary keys, composite keys, and natural vs surrogate

```sql
-- Composite primary key (a junction/join table):
CREATE TABLE enrollment (
  student_id bigint NOT NULL REFERENCES customer(id),
  course_id  bigint NOT NULL,
  enrolled_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (student_id, course_id)   -- the pair must be unique
);
```

### ALTER TABLE — evolving your schema

```sql
ALTER TABLE customer ADD COLUMN phone text;                 -- add a column
ALTER TABLE customer ADD COLUMN country text NOT NULL DEFAULT 'US';  -- PG11+ fast for non-volatile defaults
ALTER TABLE customer DROP COLUMN phone;                     -- drop a column
ALTER TABLE customer RENAME COLUMN email TO email_address;  -- rename
ALTER TABLE customer ALTER COLUMN status SET DEFAULT 'active';
ALTER TABLE customer ALTER COLUMN status DROP DEFAULT;
ALTER TABLE customer ALTER COLUMN age SET NOT NULL;         -- add NOT NULL (validates existing rows)
ALTER TABLE customer ALTER COLUMN age TYPE bigint;          -- change type (may rewrite table)

-- Add/drop constraints after the fact:
ALTER TABLE orders ADD CONSTRAINT total_positive CHECK (total > 0);
ALTER TABLE orders DROP CONSTRAINT total_positive;

-- Adding a FK with validation deferred (avoid a long lock on huge tables):
ALTER TABLE orders ADD CONSTRAINT orders_customer_fk
  FOREIGN KEY (customer_id) REFERENCES customer(id) NOT VALID;  -- skips checking existing rows
ALTER TABLE orders VALIDATE CONSTRAINT orders_customer_fk;      -- validate later, weaker lock

ALTER TABLE customer RENAME TO client;                     -- rename the table
```

> **Gotcha (locking):** Many `ALTER TABLE` operations take an `ACCESS EXCLUSIVE` lock, blocking all reads/writes for the duration. Operations that rewrite the table (changing a column type, adding a non-trivial default in older PG) can lock for a long time on big tables. Use the `NOT VALID` then `VALIDATE` pattern, or run schema changes in low-traffic windows.

### DROP and TRUNCATE

```sql
DROP TABLE IF EXISTS enrollment;            -- remove a table
DROP TABLE orders CASCADE;                  -- also drop dependent objects (views, FKs)
TRUNCATE orders;                            -- delete ALL rows fast (no per-row triggers, can't be filtered)
TRUNCATE orders RESTART IDENTITY CASCADE;   -- also reset identity & truncate referencing tables
```

### Schemas (namespaces)

A schema is a namespace inside a database. The default schema is `public`. Use schemas to group objects (`auth.users`, `billing.invoices`) or to isolate tenants.

```sql
CREATE SCHEMA billing;
CREATE SCHEMA billing AUTHORIZATION app_user;   -- owned by app_user

CREATE TABLE billing.invoice (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY);

-- The search_path determines which schemas are searched for unqualified names:
SHOW search_path;                  -- default: "$user", public
SET search_path TO billing, public;
SELECT * FROM invoice;             -- resolves to billing.invoice

DROP SCHEMA billing CASCADE;       -- drop schema and everything in it
```

### Sequences and identity columns

```sql
-- A sequence is a special object that generates increasing numbers.
-- IDENTITY columns use one under the hood; you can also create them manually.
CREATE SEQUENCE order_no_seq START 1000 INCREMENT 1;
SELECT nextval('order_no_seq');    -- 1000, advances the sequence
SELECT currval('order_no_seq');    -- 1000 (last value in THIS session)
SELECT setval('order_no_seq', 5000);  -- reset

-- Attach a sequence as a column default:
CREATE TABLE ticket (
  number bigint NOT NULL DEFAULT nextval('order_no_seq')
);

-- Inspect/fix an IDENTITY column's underlying sequence (e.g. after a bulk load):
SELECT pg_get_serial_sequence('orders', 'id');   -- the sequence name
SELECT setval(pg_get_serial_sequence('orders','id'),
              (SELECT max(id) FROM orders));      -- resync after manual inserts
```

> **Gotcha:** Sequences are **not** transactional for gap-freeness — a rolled-back transaction still consumes the numbers it fetched. IDs will have gaps. Never assume sequence values are contiguous; they only guarantee uniqueness and monotonic increase.

---

## 4. DML & Queries — INSERT, UPDATE, DELETE, SELECT

**[Beginner]**

Let's build a small dataset to query throughout.

```sql
CREATE TABLE employee (
  id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name       text NOT NULL,
  department text NOT NULL,
  salary     numeric(10,2) NOT NULL,
  manager_id bigint REFERENCES employee(id),   -- self-reference for org chart
  hired_at   date NOT NULL DEFAULT current_date
);
```

### INSERT — single, multi-row, RETURNING, upsert

```sql
-- Single row:
INSERT INTO employee (name, department, salary)
VALUES ('Ada', 'Engineering', 120000);

-- Multi-row (one statement, far faster than many INSERTs):
INSERT INTO employee (name, department, salary, manager_id) VALUES
  ('Grace', 'Engineering', 110000, 1),
  ('Linus', 'Engineering',  95000, 1),
  ('Margaret', 'Design',    90000, NULL),
  ('Katherine', 'Design',   88000, 4);

-- RETURNING: get back generated/computed values without a second query:
INSERT INTO employee (name, department, salary)
VALUES ('Alan', 'Research', 130000)
RETURNING id, name, hired_at;

-- INSERT ... SELECT (copy/transform from another query):
INSERT INTO employee (name, department, salary)
SELECT name || ' (copy)', department, salary FROM employee WHERE department = 'Design';
```

### UPSERT with ON CONFLICT

```sql
-- Requires a UNIQUE or PRIMARY KEY constraint to detect the conflict.
CREATE TABLE inventory (
  sku   text PRIMARY KEY,
  qty   int NOT NULL,
  updated_at timestamptz NOT NULL DEFAULT now()
);

-- "Insert, or if the sku already exists, update the quantity":
INSERT INTO inventory (sku, qty) VALUES ('ABC-1', 10)
ON CONFLICT (sku) DO UPDATE
  SET qty = inventory.qty + EXCLUDED.qty,   -- EXCLUDED = the row we tried to insert
      updated_at = now();

-- "Insert, but ignore if it already exists" (no-op on conflict):
INSERT INTO inventory (sku, qty) VALUES ('ABC-1', 5)
ON CONFLICT (sku) DO NOTHING;

-- Conditional upsert with a WHERE on the update:
INSERT INTO inventory (sku, qty) VALUES ('ABC-1', 3)
ON CONFLICT (sku) DO UPDATE SET qty = EXCLUDED.qty
WHERE inventory.qty < EXCLUDED.qty;   -- only update if new qty is larger
```

⚡ **Version note:** PG15+ has `MERGE`, the SQL-standard multi-action upsert; **PG17** adds `MERGE ... RETURNING` and a `WHEN NOT MATCHED BY SOURCE` clause. `ON CONFLICT` is simpler for plain upserts; `MERGE` shines when you need INSERT/UPDATE/DELETE branches in one statement.

```sql
-- MERGE example (PG15+):
MERGE INTO inventory AS t
USING (VALUES ('ABC-1', 7)) AS s(sku, qty) ON t.sku = s.sku
WHEN MATCHED THEN UPDATE SET qty = t.qty + s.qty
WHEN NOT MATCHED THEN INSERT (sku, qty) VALUES (s.sku, s.qty)
RETURNING merge_action(), t.*;   -- PG17: merge_action() returns 'INSERT'/'UPDATE'/'DELETE'
```

### UPDATE & DELETE

```sql
-- UPDATE with WHERE (NEVER forget the WHERE unless you truly mean "all rows"):
UPDATE employee SET salary = salary * 1.10 WHERE department = 'Engineering';

-- UPDATE ... FROM: update using a join to another table:
UPDATE employee e
SET salary = salary + 5000
FROM (SELECT department FROM employee GROUP BY department HAVING avg(salary) < 100000) low
WHERE e.department = low.department
RETURNING e.id, e.name, e.salary;   -- RETURNING works on UPDATE/DELETE too

-- DELETE:
DELETE FROM employee WHERE department = 'Research' RETURNING *;
DELETE FROM employee USING orders WHERE employee.id = orders.customer_id;  -- DELETE ... USING join
```

### SELECT — the core query

```sql
-- WHERE: filter rows. Operators: = <> < > <= >= BETWEEN IN LIKE ILIKE IS NULL
SELECT name, salary FROM employee
WHERE department = 'Engineering'
  AND salary BETWEEN 90000 AND 130000
  AND name ILIKE 'a%'              -- case-insensitive prefix match
  AND manager_id IS NOT NULL;      -- NULL must use IS / IS NOT, never = NULL

-- IN and NOT IN:
SELECT * FROM employee WHERE department IN ('Design', 'Engineering');

-- ORDER BY (asc default), NULLS FIRST/LAST control NULL placement:
SELECT name, salary FROM employee
ORDER BY salary DESC, name ASC NULLS LAST;

-- LIMIT / OFFSET — pagination (OFFSET is slow for deep pages; see keyset pagination in §22):
SELECT * FROM employee ORDER BY id LIMIT 10 OFFSET 20;   -- rows 21..30
-- SQL-standard equivalent:
SELECT * FROM employee ORDER BY id FETCH FIRST 10 ROWS ONLY;

-- DISTINCT and DISTINCT ON:
SELECT DISTINCT department FROM employee;          -- unique departments
-- DISTINCT ON (Postgres extension): one row per department, the highest-paid:
SELECT DISTINCT ON (department) department, name, salary
FROM employee
ORDER BY department, salary DESC;   -- ORDER BY must start with the DISTINCT ON columns

-- Column & table aliases, computed columns:
SELECT e.name AS employee, e.salary / 12 AS monthly_pay
FROM employee AS e
WHERE e.salary > 100000;

-- CASE expression (inline conditional):
SELECT name,
  CASE
    WHEN salary >= 120000 THEN 'senior'
    WHEN salary >= 95000  THEN 'mid'
    ELSE 'junior'
  END AS band
FROM employee;

-- COALESCE (first non-NULL) and NULLIF:
SELECT name, COALESCE(manager_id::text, 'no manager') AS mgr FROM employee;
SELECT NULLIF(salary, 0) FROM employee;   -- returns NULL if salary = 0
```

### Bulk import/export with COPY

```sql
-- Server-side COPY (needs file access on the server; superuser or pg_read_server_files):
COPY employee (name, department, salary) FROM '/data/emp.csv' WITH (FORMAT csv, HEADER true);
COPY (SELECT * FROM employee) TO '/data/out.csv' WITH (FORMAT csv, HEADER true);
```

```text
-- Client-side \copy in psql (reads/writes a file on YOUR machine, no special perms):
\copy employee(name,department,salary) FROM 'emp.csv' WITH (FORMAT csv, HEADER true)
\copy (SELECT * FROM employee) TO 'out.csv' WITH (FORMAT csv, HEADER true)
```

> **Tip:** `COPY` is dramatically faster than row-by-row `INSERT` for loading data. For millions of rows it can be 10–100x faster.

---

## 5. Joins & Set Operations

**[Intermediate]**

Joins combine rows from multiple tables based on a condition.

```sql
-- Sample data we'll reuse:
-- employee(id, name, department, salary, manager_id, hired_at)
-- orders(id, customer_id, total, placed_at)  -- from §3

-- INNER JOIN: only rows with matches in BOTH tables.
SELECT e.name, m.name AS manager
FROM employee e
INNER JOIN employee m ON e.manager_id = m.id;   -- self-join on the same table

-- LEFT (OUTER) JOIN: all rows from the LEFT, NULLs where no match on the right.
SELECT e.name, m.name AS manager
FROM employee e
LEFT JOIN employee m ON e.manager_id = m.id;    -- includes employees with no manager

-- Find rows with NO match (anti-join) — the classic "orphans" query:
SELECT e.name
FROM employee e
LEFT JOIN employee m ON e.manager_id = m.id
WHERE m.id IS NULL;        -- employees whose manager_id points nowhere (or is NULL)

-- RIGHT JOIN: all rows from the RIGHT (mirror of LEFT; rarely used — just flip the tables).
SELECT e.name, m.name AS manager
FROM employee m
RIGHT JOIN employee e ON e.manager_id = m.id;

-- FULL OUTER JOIN: all rows from both; NULLs where either side lacks a match.
SELECT COALESCE(a.department, b.department) AS dept
FROM employee a
FULL JOIN employee b ON a.department = b.department;

-- CROSS JOIN: Cartesian product (every left row paired with every right row).
SELECT e.name, d.day
FROM employee e
CROSS JOIN (VALUES ('Mon'), ('Tue')) AS d(day);   -- duplicate each employee per day

-- USING shorthand (when join columns share a name):
SELECT * FROM orders JOIN customer USING (id);    -- ON orders.id = customer.id (collapses the column)

-- Multiple joins:
SELECT c.name AS customer, o.id AS order_id, o.total
FROM customer c
JOIN orders o ON o.customer_id = c.id
WHERE o.total > 100;
```

### LATERAL joins — "join to a subquery that references the left row"

`LATERAL` lets a subquery in the `FROM` clause reference columns from preceding tables. It's the idiomatic way to do "top-N per group".

```sql
-- For each department, get its 2 highest-paid employees:
SELECT d.department, top.name, top.salary
FROM (SELECT DISTINCT department FROM employee) d
CROSS JOIN LATERAL (
  SELECT name, salary
  FROM employee e
  WHERE e.department = d.department    -- references the outer 'd' — only possible with LATERAL
  ORDER BY salary DESC
  LIMIT 2
) AS top;
```

### Set operations

```sql
-- UNION: combine result sets and REMOVE duplicates. UNION ALL keeps duplicates (faster).
SELECT name FROM employee WHERE department = 'Engineering'
UNION
SELECT name FROM employee WHERE salary > 100000;

SELECT department FROM employee
UNION ALL                       -- keep dups; use when you know there are none, or want counts
SELECT department FROM orders_archive;   -- columns must match in count & compatible types

-- INTERSECT: rows present in BOTH queries.
SELECT name FROM employee WHERE department = 'Engineering'
INTERSECT
SELECT name FROM employee WHERE salary > 100000;

-- EXCEPT: rows in the first query but NOT the second (set difference).
SELECT name FROM employee
EXCEPT
SELECT name FROM employee WHERE department = 'Design';
```

> **Gotcha:** `UNION` performs a deduplication sort/hash; if you don't need it, `UNION ALL` is much cheaper. Also, set operations match columns **by position**, not by name.

---

## 6. Aggregation — GROUP BY, HAVING, GROUPING SETS

**[Intermediate]**

```sql
-- Core aggregate functions:
SELECT
  count(*)                AS num_rows,      -- counts all rows
  count(manager_id)       AS num_with_mgr,  -- counts NON-NULL values only
  count(DISTINCT department) AS num_depts,
  sum(salary)             AS payroll,
  avg(salary)             AS avg_salary,
  min(salary), max(salary),
  string_agg(name, ', ' ORDER BY name) AS everyone,  -- concatenate text
  array_agg(salary ORDER BY salary DESC) AS salaries
FROM employee;

-- GROUP BY: one output row per distinct group.
SELECT department, count(*) AS headcount, round(avg(salary), 2) AS avg_salary
FROM employee
GROUP BY department
ORDER BY avg_salary DESC;

-- HAVING: filter AFTER aggregation (WHERE filters BEFORE).
SELECT department, avg(salary) AS avg_salary
FROM employee
WHERE hired_at >= '2020-01-01'   -- WHERE: filters input rows
GROUP BY department
HAVING avg(salary) > 95000;      -- HAVING: filters groups by their aggregate
```

### FILTER — conditional aggregation

```sql
-- FILTER restricts which rows an aggregate sees — cleaner than CASE inside agg.
SELECT
  count(*)                                   AS total,
  count(*) FILTER (WHERE salary > 100000)    AS high_earners,
  count(*) FILTER (WHERE department = 'Design') AS designers,
  avg(salary) FILTER (WHERE department = 'Engineering') AS eng_avg
FROM employee;
```

### GROUPING SETS, ROLLUP, CUBE — multi-level subtotals

```sql
-- ROLLUP: hierarchical subtotals + grand total in one query.
-- Produces: per (department, manager_id), per department, and overall.
SELECT department, manager_id, count(*) AS n, sum(salary) AS payroll
FROM employee
GROUP BY ROLLUP (department, manager_id)
ORDER BY department NULLS LAST, manager_id NULLS LAST;
-- Rows where a column is NULL = a subtotal/grand-total line for that level.

-- CUBE: subtotals for EVERY combination of the listed columns.
SELECT department, manager_id, sum(salary)
FROM employee
GROUP BY CUBE (department, manager_id);

-- GROUPING SETS: explicitly list the groupings you want.
SELECT department, manager_id, count(*)
FROM employee
GROUP BY GROUPING SETS ((department), (manager_id), ());  -- () = grand total

-- GROUPING() tells you whether a column is a subtotal placeholder (1) or a real value (0):
SELECT
  CASE WHEN grouping(department) = 1 THEN 'ALL DEPARTMENTS' ELSE department END AS dept,
  sum(salary)
FROM employee
GROUP BY ROLLUP (department);
```

---

## 7. Subqueries & CTEs (incl. Recursive)

**[Intermediate]**

### Subqueries

```sql
-- Scalar subquery (returns one value):
SELECT name, salary,
  salary - (SELECT avg(salary) FROM employee) AS diff_from_avg
FROM employee;

-- IN subquery:
SELECT name FROM employee
WHERE department IN (SELECT department FROM employee GROUP BY department HAVING count(*) > 2);

-- Correlated subquery (references the outer row; runs per outer row):
SELECT e.name, e.salary
FROM employee e
WHERE e.salary > (SELECT avg(salary) FROM employee x WHERE x.department = e.department);

-- EXISTS / NOT EXISTS (often faster & NULL-safe vs IN):
SELECT c.name FROM customer c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);  -- customers with orders
SELECT c.name FROM customer c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);  -- with no orders
```

> **Gotcha:** `NOT IN (subquery)` behaves surprisingly with NULLs — if the subquery returns any NULL, `NOT IN` yields no rows at all. Prefer `NOT EXISTS`, which is NULL-safe.

### Common Table Expressions (CTEs / WITH)

```sql
-- A CTE names a subquery, improving readability and enabling reuse.
WITH dept_stats AS (
  SELECT department, avg(salary) AS avg_sal, count(*) AS n
  FROM employee
  GROUP BY department
)
SELECT e.name, e.salary, ds.avg_sal
FROM employee e
JOIN dept_stats ds ON ds.department = e.department
WHERE e.salary > ds.avg_sal;     -- above-average earners in their dept

-- Multiple CTEs, chained:
WITH
  recent AS (SELECT * FROM orders WHERE placed_at > now() - interval '30 days'),
  big    AS (SELECT * FROM recent WHERE total > 500)
SELECT customer_id, count(*) FROM big GROUP BY customer_id;

-- Data-modifying CTEs: move rows in one statement (archive-then-delete atomically):
WITH moved AS (
  DELETE FROM orders WHERE placed_at < '2020-01-01' RETURNING *
)
INSERT INTO orders_archive SELECT * FROM moved;
```

⚡ **Version note:** Since PG12, CTEs are **inlined** by default (the optimizer can merge them into the main query) unless they're recursive, used more than once, or marked `MATERIALIZED`. Before PG12 every CTE was an optimization fence. Use `WITH x AS MATERIALIZED (...)` to force a fence, or `NOT MATERIALIZED` to force inlining.

### Recursive CTEs — tree & graph traversal

```sql
-- Walk the org chart from the top down, tracking depth and path.
WITH RECURSIVE org AS (
  -- ANCHOR: the starting rows (top-level managers, no boss).
  SELECT id, name, manager_id, 1 AS depth, name::text AS path
  FROM employee
  WHERE manager_id IS NULL

  UNION ALL

  -- RECURSIVE term: join the table back to the previous iteration's results.
  SELECT e.id, e.name, e.manager_id, o.depth + 1, o.path || ' > ' || e.name
  FROM employee e
  JOIN org o ON e.manager_id = o.id     -- 'org' here = rows produced so far
)
SELECT repeat('  ', depth - 1) || name AS tree, depth, path
FROM org
ORDER BY path;

-- Numeric series via recursion (generate_series is usually better, but illustrative):
WITH RECURSIVE nums(n) AS (
  SELECT 1
  UNION ALL
  SELECT n + 1 FROM nums WHERE n < 10
)
SELECT n FROM nums;   -- 1..10

-- Graph traversal with cycle protection (PG14+ has CYCLE clause):
WITH RECURSIVE reachable AS (
  SELECT id, name, ARRAY[id] AS visited FROM employee WHERE id = 1
  UNION ALL
  SELECT e.id, e.name, r.visited || e.id
  FROM employee e
  JOIN reachable r ON e.manager_id = r.id
  WHERE e.id <> ALL(r.visited)   -- avoid revisiting -> prevents infinite loops on cycles
)
SELECT * FROM reachable;
```

---

## 8. Window Functions in Depth

**[Intermediate]**

Window functions compute across a set of rows *related to the current row* — without collapsing rows the way `GROUP BY` does. Every output row is preserved; the function looks "out a window" at neighbors.

### Ranking

```sql
SELECT
  name, department, salary,
  -- ROW_NUMBER: unique 1,2,3... within each partition (ties broken arbitrarily).
  row_number() OVER w AS rn,
  -- RANK: ties share a rank, then skip (1,2,2,4).
  rank()       OVER w AS rnk,
  -- DENSE_RANK: ties share a rank, no gaps (1,2,2,3).
  dense_rank() OVER w AS dense,
  -- NTILE(4): split each partition into 4 buckets (quartiles).
  ntile(4)     OVER w AS quartile,
  -- PERCENT_RANK / CUME_DIST: relative standing 0..1.
  round(percent_rank() OVER w * 100, 1) AS pct_rank
FROM employee
WINDOW w AS (PARTITION BY department ORDER BY salary DESC);  -- reusable named window
```

### LAG / LEAD — compare to neighboring rows

```sql
-- Month-over-month change in order totals.
WITH monthly AS (
  SELECT date_trunc('month', placed_at) AS month, sum(total) AS revenue
  FROM orders GROUP BY 1
)
SELECT
  month,
  revenue,
  lag(revenue)  OVER (ORDER BY month) AS prev_month,           -- previous row's value
  revenue - lag(revenue) OVER (ORDER BY month) AS delta,       -- difference
  lead(revenue) OVER (ORDER BY month) AS next_month,           -- next row's value
  first_value(revenue) OVER (ORDER BY month) AS first_month,
  last_value(revenue)  OVER (ORDER BY month
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_month
FROM monthly
ORDER BY month;
```

### Running totals & moving averages (frames)

The **frame clause** defines which rows the aggregate sees relative to the current row. Default frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` (which can surprise you with ties).

```sql
SELECT
  name, department, salary, hired_at,
  -- Running total of salary by hire order within department:
  sum(salary) OVER (
    PARTITION BY department ORDER BY hired_at
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW   -- ROWS frame = physical rows
  ) AS running_payroll,
  -- 3-row moving average (current + 2 preceding):
  round(avg(salary) OVER (
    PARTITION BY department ORDER BY hired_at
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
  ), 2) AS moving_avg,
  -- Each row's salary as a % of its department total (no ORDER BY = whole partition):
  round(100.0 * salary / sum(salary) OVER (PARTITION BY department), 1) AS pct_of_dept
FROM employee;
```

> **ROWS vs RANGE vs GROUPS:** `ROWS` counts physical rows. `RANGE` groups peers with equal ORDER BY values (so all ties are included at once). `GROUPS` (PG11+) counts peer groups. For running totals where ties exist, use `ROWS` to avoid lumping equal-valued rows together.

### Top-N per group with window functions

```sql
-- Cleaner than LATERAL when you want a rank column too:
SELECT * FROM (
  SELECT name, department, salary,
         row_number() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
  FROM employee
) ranked
WHERE rn <= 2;     -- top 2 earners per department
```

⚡ **Version note (PG18):** PG18 adds the SQL-standard `WINDOW` frame option and improvements, but the big runtime win is that window aggregates over large partitions benefit from general planner/executor improvements. The syntax above is stable across PG14–18.

---

## 9. Working with JSON & JSONB

**[Intermediate]**

```sql
CREATE TABLE doc (
  id   bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  data jsonb NOT NULL
);
INSERT INTO doc (data) VALUES
  ('{"name":"Ada","skills":["sql","go"],"address":{"city":"London","zip":"E1"},"active":true}'),
  ('{"name":"Linus","skills":["c","git"],"address":{"city":"Helsinki"},"active":false}');
```

### Operators

```sql
SELECT data -> 'name'              FROM doc;   -- '->'  : get field as JSONB  -> "Ada"
SELECT data ->> 'name'             FROM doc;   -- '->>' : get field as TEXT   -> Ada
SELECT data -> 'skills' -> 0       FROM doc;   -- array element as JSONB
SELECT data -> 'skills' ->> 0      FROM doc;   -- array element as text
SELECT data #> '{address,city}'    FROM doc;   -- '#>'  : get nested as JSONB (path array)
SELECT data #>> '{address,city}'   FROM doc;   -- '#>>' : get nested as text

-- Containment & existence (these can use a GIN index):
SELECT * FROM doc WHERE data @> '{"active":true}';        -- '@>' contains this JSON
SELECT * FROM doc WHERE data ? 'address';                 -- '?'  top-level key exists
SELECT * FROM doc WHERE data ?| array['email','phone'];   -- ANY of these keys exist
SELECT * FROM doc WHERE data ?& array['name','skills'];   -- ALL of these keys exist
SELECT * FROM doc WHERE data -> 'skills' @> '["sql"]';    -- array contains 'sql'
```

### jsonb functions & path queries

```sql
SELECT jsonb_pretty(data) FROM doc WHERE id = 1;          -- human-readable
SELECT jsonb_typeof(data -> 'skills') FROM doc;           -- 'array'
SELECT jsonb_array_length(data -> 'skills') FROM doc;     -- 2
SELECT jsonb_object_keys(data) FROM doc WHERE id = 1;     -- one key per row

-- Expand a JSON array into rows:
SELECT id, skill
FROM doc, jsonb_array_elements_text(data -> 'skills') AS skill;

-- Expand a JSON object into key/value rows:
SELECT key, value FROM doc, jsonb_each(data) WHERE id = 1;

-- Modify JSONB (returns a new value; UPDATE to persist):
UPDATE doc SET data = jsonb_set(data, '{address,country}', '"UK"') WHERE id = 1;  -- set/insert path
UPDATE doc SET data = data || '{"verified":true}'::jsonb WHERE id = 1;            -- merge/concat
UPDATE doc SET data = data - 'active' WHERE id = 2;                               -- delete a key
UPDATE doc SET data = data #- '{address,zip}' WHERE id = 1;                       -- delete nested path

-- SQL/JSON path language (jsonb_path_query), PG12+:
SELECT jsonb_path_query(data, '$.skills[*]') FROM doc WHERE id = 1;       -- each skill
SELECT jsonb_path_query_array(data, '$.skills[*] ? (@ starts with "g")') FROM doc;
SELECT data @? '$.skills[*] ? (@ == "sql")' FROM doc;    -- '@?' : does path match anything?
SELECT data @@ '$.active == true' FROM doc;              -- '@@' : path predicate as boolean
```

### JSON_TABLE — turn JSON into a relational result (PG17)

⚡ **Version note (PG17):** `JSON_TABLE` is the SQL-standard way to shred JSON into columns/rows — a huge ergonomic win over chaining `jsonb_array_elements`.

```sql
-- Flatten each doc's skills into a clean relational table:
SELECT d.id, jt.skill, jt.idx
FROM doc d,
  JSON_TABLE(
    d.data,
    '$.skills[*]'                       -- row pattern: one output row per skill
    COLUMNS (
      idx   FOR ORDINALITY,             -- 1-based position
      skill text PATH '$'               -- the scalar value at this path
    )
  ) AS jt;

-- Extract nested fields into named columns:
SELECT jt.*
FROM doc d,
  JSON_TABLE(d.data, '$'
    COLUMNS (
      name text PATH '$.name',
      city text PATH '$.address.city',
      active boolean PATH '$.active'
    )
  ) AS jt;
```

### Indexing JSONB with GIN

```sql
-- Default GIN: supports @>, ?, ?|, ?& and key/value lookups.
CREATE INDEX idx_doc_data ON doc USING gin (data);

-- jsonb_path_ops: smaller & faster, but ONLY supports @> (containment):
CREATE INDEX idx_doc_data_path ON doc USING gin (data jsonb_path_ops);

-- Expression index on a single hot field (B-tree) for equality/range on that field:
CREATE INDEX idx_doc_name ON doc ((data ->> 'name'));
SELECT * FROM doc WHERE data ->> 'name' = 'Ada';   -- can use idx_doc_name
SELECT * FROM doc WHERE data @> '{"active":true}'; -- can use idx_doc_data
```

### Building JSON output

```sql
-- Build objects/arrays from relational data — great for API responses.
SELECT jsonb_build_object(
  'id', id, 'name', name, 'salary', salary
) FROM employee;

-- Aggregate rows into a JSON array of objects:
SELECT jsonb_agg(jsonb_build_object('name', name, 'salary', salary)) AS team
FROM employee WHERE department = 'Engineering';

-- Whole row as JSON, and nested aggregation (one row per dept with its members):
SELECT department,
       jsonb_agg(to_jsonb(e) - 'department' ORDER BY salary DESC) AS members
FROM employee e
GROUP BY department;

-- json_agg vs jsonb_agg: json_agg preserves input order/format; jsonb_agg returns jsonb.
SELECT json_object_agg(name, salary) FROM employee;   -- {"Ada":120000, ...}
```

> **When to use JSONB:** for genuinely schemaless/variable attributes, sparse data, or storing external API payloads. Do **not** use it to avoid designing a schema — relational columns are faster, type-checked, and easier to constrain/index. A hybrid (core columns + a `jsonb` "extras" column) is often ideal.

---

## 10. Arrays & Full-Text Search

**[Intermediate]**

### Array querying recap & indexing

```sql
-- (article table from §2 has tags text[])
SELECT * FROM article WHERE tags && ARRAY['sql','python'];  -- '&&' overlaps (any common element)
SELECT * FROM article WHERE tags @> ARRAY['sql'];           -- contains
SELECT * FROM article WHERE tags <@ ARRAY['sql','db','x'];  -- contained by

-- GIN index makes @>, <@, && fast on arrays:
CREATE INDEX idx_article_tags ON article USING gin (tags);

-- Aggregate array operations:
SELECT array_agg(DISTINCT t) AS all_tags FROM article, unnest(tags) AS t;
SELECT t, count(*) FROM article, unnest(tags) AS t GROUP BY t ORDER BY count(*) DESC;  -- tag cloud
```

### Full-text search (FTS)

FTS converts documents into normalized `tsvector` (sorted lexemes + positions) and queries into `tsquery`, then matches with `@@`. It handles stemming ("running" → "run"), stop words, and ranking.

```sql
-- 1) Inspect the building blocks:
SELECT to_tsvector('english', 'The cats are running quickly');
--   'cat':2 'quick':5 'run':4   (stop words removed, stemmed, positions kept)
SELECT to_tsquery('english', 'cat & run');         -- 'cat' AND 'run'
SELECT plainto_tsquery('english', 'running cats');  -- forgiving: treats input as plain text -> 'run' & 'cat'
SELECT websearch_to_tsquery('english', '"data science" -python');  -- web-style: phrases & negation

-- 2) Match:
SELECT to_tsvector('english','quick brown fox') @@ to_tsquery('english','fox & quick');  -- true
```

### A searchable table the right way

```sql
CREATE TABLE post (
  id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  title text NOT NULL,
  body  text NOT NULL,
  -- A GENERATED tsvector column keeps the index in sync automatically (PG12+).
  -- 'A'/'B' weights make title matches rank higher than body matches.
  search tsvector GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title,'')), 'A') ||
    setweight(to_tsvector('english', coalesce(body, '')), 'B')
  ) STORED
);

-- GIN index over the tsvector column = fast search:
CREATE INDEX idx_post_search ON post USING gin (search);

INSERT INTO post (title, body) VALUES
  ('PostgreSQL indexing', 'Learn about B-tree, GIN, and GiST indexes.'),
  ('Cooking with cast iron', 'A guide to seasoning and maintaining pans.');

-- Search + rank by relevance:
SELECT id, title,
       ts_rank(search, q) AS rank,
       ts_headline('english', body, q) AS snippet   -- highlight matches
FROM post, websearch_to_tsquery('english', 'index') q
WHERE search @@ q
ORDER BY rank DESC;
```

> **Trigram alternative:** For typo-tolerant / substring / "fuzzy LIKE" search, use the `pg_trgm` extension (§20) with a GIN/GiST index — better than FTS for short strings, autocomplete, and `ILIKE '%term%'`.

---

## 11. Indexing Deeply

**[Advanced]**

Indexes trade write cost and storage for read speed. The optimizer chooses whether to use one based on statistics and cost. Understanding index *types* lets you match the index to the query.

### The index types and when each applies

| Index | Best for | Notes |
|---|---|---|
| **B-tree** (default) | `=`, `<`, `>`, `BETWEEN`, `IN`, `ORDER BY`, prefix `LIKE 'abc%'` | The workhorse. Supports multicolumn, unique, ordered scans. |
| **Hash** | equality `=` only | Smaller for equality-only; crash-safe & WAL-logged since PG10. Rarely beats B-tree. |
| **GIN** | "many values in one row": arrays, JSONB, FTS (`tsvector`), `pg_trgm` | Fast lookups, slower/heavier writes. |
| **GiST** | geometric/range/nearest-neighbor, FTS, exclusion constraints | Extensible; supports `&&`, `<->` (KNN), ranges, PostGIS. |
| **SP-GiST** | space-partitioned data: quadtrees, IP/text-prefix trees | Good for non-balanced, partitioned structures. |
| **BRIN** | huge, naturally-ordered tables (time-series append) | Tiny index storing per-block min/max. Cheap; great for date columns. |

```sql
-- B-tree (default — you can omit USING btree):
CREATE INDEX idx_emp_dept ON employee (department);
CREATE INDEX idx_emp_salary ON employee (salary DESC NULLS LAST);

-- Multicolumn B-tree: column ORDER matters! This serves filters on (dept) and (dept, salary),
-- but NOT a filter on salary alone (leftmost-prefix rule).
CREATE INDEX idx_emp_dept_salary ON employee (department, salary);

-- Unique index (also enforceable as a constraint):
CREATE UNIQUE INDEX idx_customer_email ON customer (lower(email));  -- case-insensitive uniqueness

-- Partial index: index only the rows you query, smaller & faster:
CREATE INDEX idx_orders_unpaid ON orders (placed_at) WHERE total > 0 AND placed_at IS NOT NULL;
-- Great for "active"/"pending" flags where you rarely query the rest:
CREATE INDEX idx_active_users ON customer (id) WHERE status = 'active';

-- Expression (functional) index: index the RESULT of a function:
CREATE INDEX idx_emp_lower_name ON employee (lower(name));
SELECT * FROM employee WHERE lower(name) = 'ada';   -- can use the index

-- Covering index with INCLUDE (PG11+): extra columns stored in the leaf for index-only scans.
CREATE INDEX idx_orders_cust_inc ON orders (customer_id) INCLUDE (total, placed_at);
-- A query selecting only customer_id, total, placed_at can be answered from the index alone.

-- BRIN for a big append-only time-series table:
CREATE INDEX idx_events_time_brin ON event USING brin (created_at) WITH (pages_per_range = 32);

-- GIN for JSONB / arrays / FTS (see §9, §10).
```

### Index-only scans & visibility

An **index-only scan** answers a query entirely from the index without touching the table heap — but only if all selected columns are in the index (use `INCLUDE`) *and* the relevant pages are marked all-visible in the **visibility map**. Keeping the visibility map fresh requires `VACUUM`. If you see "Heap Fetches" in `EXPLAIN ANALYZE`, the visibility map is stale.

```sql
-- Verify with EXPLAIN:
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, total FROM orders WHERE customer_id = 42;
-- Look for "Index Only Scan using idx_orders_cust_inc" and "Heap Fetches: 0".
```

### Maintenance & inspection

```sql
-- Build without locking out writes (longer, but online). Use OUTSIDE a transaction.
CREATE INDEX CONCURRENTLY idx_emp_hired ON employee (hired_at);
DROP INDEX CONCURRENTLY idx_emp_hired;

-- Rebuild a bloated index online (PG12+):
REINDEX INDEX CONCURRENTLY idx_emp_dept;
REINDEX TABLE CONCURRENTLY employee;

-- Find unused indexes (candidates for removal — they only cost write overhead):
SELECT relname AS table, indexrelname AS index, idx_scan AS times_used,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC, pg_relation_size(indexrelid) DESC;

-- Index size:
SELECT pg_size_pretty(pg_relation_size('idx_emp_dept'));
```

⚡ **Version note (PG18):** PG18 adds **B-tree "skip scan"**, letting a multicolumn index be used even when the leading column isn't in the `WHERE` clause (by skipping through its distinct values). This relaxes the strict leftmost-prefix rule for low-cardinality leading columns — but designing the column order for your real queries still matters.

> **Indexing rules of thumb:** index columns used in `WHERE`, `JOIN`, and `ORDER BY`; index every foreign key; put the most selective / equality columns first in multicolumn indexes; use partial indexes for skewed predicates; don't over-index (each index slows writes and bloats). Always confirm with `EXPLAIN`.

---

## 12. EXPLAIN, Query Plans & Finding Slow Queries

**[Advanced]**

`EXPLAIN` shows the optimizer's chosen plan and its *estimated* cost. `EXPLAIN ANALYZE` actually runs the query and shows *real* timings and row counts — invaluable for spotting bad estimates.

```sql
EXPLAIN SELECT * FROM employee WHERE department = 'Engineering';
-- Estimated plan only (does NOT run the query).

EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT)
SELECT e.name, m.name AS manager
FROM employee e JOIN employee m ON e.manager_id = m.id
WHERE e.salary > 100000;
--  ANALYZE  : actually execute & show real time/rows
--  BUFFERS  : show shared/local buffer hits & reads (I/O) — crucial for tuning
--  VERBOSE  : output columns, schema-qualified names
--  FORMAT   : TEXT (default), JSON, YAML, XML
```

> **Caution:** `EXPLAIN ANALYZE` *runs* the statement — for `INSERT/UPDATE/DELETE` it will modify data. Wrap it in a transaction you roll back: `BEGIN; EXPLAIN ANALYZE UPDATE ...; ROLLBACK;`.

### Reading a plan

Plans are trees; read **inside-out, bottom-up**. Each node shows:
- `cost=startup..total` — arbitrary units (estimated). Lower total = preferred.
- `rows=` estimated row count; in `ANALYZE`, also `actual rows`. **A big gap between estimated and actual is the #1 signal of a planning problem** (stale stats, correlated predicates).
- `actual time=startup..total` (ms), `loops=` (how many times this node ran — multiply!).

### Scan types

| Node | Meaning | When good / bad |
|---|---|---|
| **Seq Scan** | read the whole table | Fine for small tables or when fetching most rows; bad on big tables with a selective filter. |
| **Index Scan** | walk index, fetch matching heap rows | Good for selective predicates. |
| **Index Only Scan** | answer from the index alone | Best — no heap fetches (needs covering index + visibility). |
| **Bitmap Index/Heap Scan** | gather many matches via index, then fetch heap in physical order | Good for medium selectivity / multiple indexes combined. |

### Join algorithms

| Join | How | Best when |
|---|---|---|
| **Nested Loop** | for each outer row, probe inner (often via index) | Small outer side; indexed inner. Catastrophic if outer is large & inner unindexed. |
| **Hash Join** | build a hash table on the smaller side, probe with the larger | Large unsorted inputs, equality joins. Needs `work_mem`. |
| **Merge Join** | sort both inputs, merge | Both inputs already sorted (e.g. on indexed columns) or sortable cheaply. |

### A worked diagnosis

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 42;
-- BAD plan symptom:
--   Seq Scan on orders  (cost=0.00..18500 rows=1) (actual rows=1 loops=1)
--     Filter: (customer_id = 42)
--     Rows Removed by Filter: 999999          <-- scanned a million rows to find one!
-- FIX: add an index so the planner can do an Index Scan:
CREATE INDEX idx_orders_customer ON orders (customer_id);
-- Re-run EXPLAIN; you should now see "Index Scan using idx_orders_customer".
```

### The cost model & stats

The planner uses table statistics (collected by `ANALYZE`) to estimate selectivity. If estimates are wrong, plans go wrong.

```sql
ANALYZE employee;                       -- refresh stats for one table
ANALYZE;                                -- whole database
-- Improve estimates on a skewed/important column (default 100 buckets):
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 1000;
ANALYZE orders;

-- Extended statistics for CORRELATED columns (planner assumes independence by default):
CREATE STATISTICS s_city_country (dependencies, ndistinct)
  ON city, country FROM customer;       -- tells planner city<->country are related
ANALYZE customer;

-- Key cost knobs (per session or global): lower random_page_cost for SSDs so index scans win:
SET random_page_cost = 1.1;             -- default 4.0 (tuned for spinning disks)
SET effective_cache_size = '12GB';      -- hint about OS+PG cache size
```

### Finding slow queries with pg_stat_statements

```sql
-- One-time setup (add to shared_preload_libraries in postgresql.conf, then restart):
--   shared_preload_libraries = 'pg_stat_statements'
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top 10 queries by total time spent (the real hotspots):
SELECT
  round(total_exec_time::numeric, 1) AS total_ms,
  calls,
  round(mean_exec_time::numeric, 2)  AS mean_ms,
  round(100 * total_exec_time / sum(total_exec_time) OVER (), 1) AS pct,
  query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Reset the counters after a deploy to measure fresh:
SELECT pg_stat_statements_reset();

-- See currently-running queries and what they're blocked on:
SELECT pid, state, wait_event_type, wait_event,
       now() - query_start AS running_for, query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY running_for DESC;

-- Kill a runaway query (cancel) or its connection (terminate):
SELECT pg_cancel_backend(12345);     -- politely cancel the current query
SELECT pg_terminate_backend(12345);  -- forcibly close the connection
```

> **`auto_explain`** can log the plan of any query slower than a threshold automatically — set `auto_explain.log_min_duration = '500ms'`. Invaluable for catching production slow queries without manual `EXPLAIN`.

---

## 13. Transactions, MVCC & Concurrency

**[Advanced]**

### ACID & basic transactions

A transaction groups statements so they succeed or fail together. PostgreSQL guarantees **A**tomicity, **C**onsistency, **I**solation, **D**urability.

```sql
BEGIN;                                  -- start a transaction
UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;
-- If anything fails, nothing is applied:
COMMIT;                                 -- make changes durable & visible
-- or
ROLLBACK;                               -- discard all changes since BEGIN

-- Savepoints: partial rollback within a transaction.
BEGIN;
INSERT INTO employee (name, department, salary) VALUES ('Temp', 'X', 1);
SAVEPOINT sp1;
UPDATE employee SET salary = -5 WHERE name = 'Temp';   -- oops, violates a CHECK later
ROLLBACK TO SAVEPOINT sp1;             -- undo just to the savepoint; transaction continues
COMMIT;
```

> **Gotcha:** Once any statement in a transaction errors, the whole transaction enters an *aborted* state — every subsequent command fails with "current transaction is aborted" until you `ROLLBACK` (or `ROLLBACK TO SAVEPOINT`). Drivers often handle this, but it surprises people in psql.

### MVCC explained

PostgreSQL uses **Multi-Version Concurrency Control**: writers don't block readers and readers don't block writers. Instead of overwriting a row, an `UPDATE` writes a *new row version* and marks the old one dead. Each transaction sees a consistent **snapshot** based on transaction IDs (`xmin`/`xmax` on every row). The trade-off: dead row versions ("dead tuples") accumulate and must be reclaimed by `VACUUM` (§16).

```sql
-- Every row carries hidden system columns showing its version:
SELECT xmin, xmax, ctid, * FROM employee LIMIT 1;
-- xmin = txid that created this version; ctid = physical (page, offset) location.
SELECT txid_current();   -- the current transaction's id
```

### Isolation levels

| Level | Dirty read | Non-repeatable read | Phantom read | Serialization anomaly |
|---|---|---|---|---|
| Read Uncommitted | (treated as Read Committed in PG) | possible | possible | possible |
| **Read Committed** (default) | no | possible | possible | possible |
| **Repeatable Read** | no | no | no* | possible |
| **Serializable** | no | no | no | no |

\* In PostgreSQL, Repeatable Read also prevents phantom reads (it's snapshot isolation), which is stronger than the SQL standard requires.

```sql
-- Read Committed (default): each STATEMENT sees rows committed before it began.
-- Repeatable Read: the whole TRANSACTION sees one consistent snapshot from its start.
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT sum(balance) FROM account;       -- snapshot taken here
-- ... even if others commit changes, this txn keeps seeing the same data ...
SELECT sum(balance) FROM account;       -- identical result
COMMIT;

-- Serializable: as if transactions ran one-at-a-time. May fail with a
-- serialization error (SQLSTATE 40001) — your app must RETRY the transaction.
BEGIN ISOLATION LEVEL SERIALIZABLE;
-- ... business logic ...
COMMIT;   -- might raise: "could not serialize access due to read/write dependencies"
```

> **Serializable retry pattern:** any transaction at Serializable (or Repeatable Read) can abort with `40001`; the correct response is to **roll back and re-run the whole transaction**, not to ignore it.

### Locks

```sql
-- Row-level: SELECT ... FOR UPDATE locks selected rows against concurrent updates.
BEGIN;
SELECT * FROM account WHERE id = 1 FOR UPDATE;   -- block others from updating this row
UPDATE account SET balance = balance - 100 WHERE id = 1;
COMMIT;

-- Lock modes: FOR UPDATE (exclusive), FOR NO KEY UPDATE, FOR SHARE, FOR KEY SHARE.
-- NOWAIT / SKIP LOCKED control behavior when rows are already locked:
SELECT * FROM job WHERE status = 'queued' FOR UPDATE NOWAIT;       -- error if locked
SELECT * FROM job WHERE status = 'queued' FOR UPDATE SKIP LOCKED;  -- skip locked rows

-- Inspect current locks:
SELECT locktype, relation::regclass, mode, granted, pid
FROM pg_locks WHERE NOT granted;   -- waiting locks
```

### Job queue with SELECT ... FOR UPDATE SKIP LOCKED

A robust, lock-free-feeling work queue — multiple workers grab different jobs without colliding.

```sql
CREATE TABLE job (
  id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  payload jsonb NOT NULL,
  status  text NOT NULL DEFAULT 'queued',
  locked_at timestamptz
);

-- Each worker runs this to claim ONE job atomically:
WITH next_job AS (
  SELECT id FROM job
  WHERE status = 'queued'
  ORDER BY id
  FOR UPDATE SKIP LOCKED          -- skip rows other workers already grabbed
  LIMIT 1
)
UPDATE job
SET status = 'processing', locked_at = now()
FROM next_job
WHERE job.id = next_job.id
RETURNING job.*;                  -- worker gets its job; concurrent workers get different ones
```

### Deadlocks

A deadlock happens when txn A holds a lock B wants while B holds a lock A wants. PostgreSQL detects it and aborts one transaction with a deadlock error.

```sql
-- Prevent deadlocks by always acquiring locks in a CONSISTENT ORDER.
-- e.g. always lock the lower account id first:
BEGIN;
SELECT * FROM account WHERE id = least(1, 2) FOR UPDATE;
SELECT * FROM account WHERE id = greatest(1, 2) FOR UPDATE;
-- ... transfer ...
COMMIT;
```

### Advisory locks — application-level mutexes

```sql
-- Not tied to any row; you choose the key. Useful for "only one worker may run X".
SELECT pg_advisory_lock(42);        -- blocks until acquired (session-level)
SELECT pg_try_advisory_lock(42);    -- non-blocking; returns true/false
SELECT pg_advisory_unlock(42);      -- release
-- Transaction-scoped (auto-released at COMMIT/ROLLBACK — safer, no leak risk):
SELECT pg_advisory_xact_lock(42);
```

---

## 14. Functions, Procedures & Triggers (PL/pgSQL)

**[Advanced]**

### SQL functions

```sql
-- Simple, inlinable SQL function. STABLE/IMMUTABLE/VOLATILE tells the planner about side effects.
CREATE OR REPLACE FUNCTION full_name(first text, last text)
RETURNS text
LANGUAGE sql
IMMUTABLE              -- same inputs => same output, no DB access (best for indexing)
AS $$
  SELECT first || ' ' || last;
$$;
SELECT full_name('Ada', 'Lovelace');

-- A set-returning SQL function:
CREATE OR REPLACE FUNCTION emps_in(dept text)
RETURNS SETOF employee
LANGUAGE sql STABLE
AS $$ SELECT * FROM employee WHERE department = dept; $$;
SELECT * FROM emps_in('Engineering');
```

### PL/pgSQL — procedural language

```sql
CREATE OR REPLACE FUNCTION give_raise(emp_id bigint, pct numeric)
RETURNS numeric           -- returns the new salary
LANGUAGE plpgsql
AS $$
DECLARE
  current_salary numeric;
  new_salary     numeric;
BEGIN
  -- Variables, SELECT INTO:
  SELECT salary INTO current_salary FROM employee WHERE id = emp_id;

  -- Control flow:
  IF current_salary IS NULL THEN
    RAISE EXCEPTION 'Employee % not found', emp_id;   -- raises an error, aborts txn
  END IF;

  new_salary := current_salary * (1 + pct / 100.0);

  UPDATE employee SET salary = new_salary WHERE id = emp_id;

  RAISE NOTICE 'Raised % from % to %', emp_id, current_salary, new_salary;  -- log message
  RETURN new_salary;
END;
$$;
SELECT give_raise(1, 10);
```

```sql
-- Loops, FOR over a query, exception handling:
CREATE OR REPLACE FUNCTION normalize_salaries()
RETURNS int LANGUAGE plpgsql AS $$
DECLARE
  rec       record;
  updated   int := 0;
BEGIN
  FOR rec IN SELECT id, salary FROM employee WHERE salary < 50000 LOOP
    BEGIN
      UPDATE employee SET salary = 50000 WHERE id = rec.id;
      updated := updated + 1;
    EXCEPTION WHEN others THEN
      -- Catch per-row errors so one bad row doesn't abort the whole loop:
      RAISE WARNING 'Skipping % due to %', rec.id, SQLERRM;
    END;
  END LOOP;

  -- WHILE loop example:
  -- WHILE condition LOOP ... END LOOP;
  -- Numeric FOR: FOR i IN 1..10 LOOP ... END LOOP;

  RETURN updated;
END;
$$;
```

### Procedures (can manage their own transactions)

```sql
-- Unlike functions, PROCEDURES can COMMIT/ROLLBACK inside their body (PG11+).
CREATE OR REPLACE PROCEDURE batch_archive(batch int)
LANGUAGE plpgsql AS $$
DECLARE moved int;
BEGIN
  LOOP
    WITH del AS (
      DELETE FROM orders WHERE placed_at < '2020-01-01'
      LIMIT batch RETURNING *           -- (DELETE ... LIMIT requires a CTE/subquery trick in PG)
    )
    INSERT INTO orders_archive SELECT * FROM del;
    GET DIAGNOSTICS moved = ROW_COUNT;
    EXIT WHEN moved = 0;
    COMMIT;                              -- commit each batch to keep transactions short
  END LOOP;
END;
$$;
CALL batch_archive(1000);               -- procedures are invoked with CALL, not SELECT
```

### Triggers

A trigger runs a function automatically on INSERT/UPDATE/DELETE. The function returns type `trigger` and accesses `NEW` (the incoming row) and `OLD` (the prior row).

```sql
-- 1) updated_at auto-maintenance (the most common trigger):
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  NEW.updated_at := now();   -- modify the row being written (only valid in BEFORE triggers)
  RETURN NEW;                -- returning NEW lets the operation proceed with the change
END;
$$;

ALTER TABLE customer ADD COLUMN updated_at timestamptz NOT NULL DEFAULT now();
CREATE TRIGGER trg_customer_updated
BEFORE UPDATE ON customer
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();

-- 2) Audit log via AFTER trigger:
CREATE TABLE audit_log (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  table_name text, op text, row_data jsonb, changed_at timestamptz DEFAULT now()
);
CREATE OR REPLACE FUNCTION audit_changes()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  INSERT INTO audit_log (table_name, op, row_data)
  VALUES (TG_TABLE_NAME, TG_OP,
          to_jsonb(COALESCE(NEW, OLD)));   -- NEW for INSERT/UPDATE, OLD for DELETE
  RETURN NULL;   -- AFTER triggers ignore the return value
END;
$$;
CREATE TRIGGER trg_customer_audit
AFTER INSERT OR UPDATE OR DELETE ON customer
FOR EACH ROW EXECUTE FUNCTION audit_changes();

-- 3) INSTEAD OF trigger: make a VIEW writable.
-- CREATE TRIGGER ... INSTEAD OF INSERT ON some_view FOR EACH ROW EXECUTE FUNCTION ...;

-- Trigger timing & scope:
--   BEFORE / AFTER / INSTEAD OF
--   FOR EACH ROW (per affected row) vs FOR EACH STATEMENT (once per statement)
--   WHEN (condition) to fire conditionally:
CREATE TRIGGER trg_only_big
AFTER UPDATE ON orders
FOR EACH ROW
WHEN (NEW.total > 10000)              -- only fire for large orders
EXECUTE FUNCTION audit_changes();
```

> **Trigger gotchas:** keep trigger logic light — it runs inside the caller's transaction and multiplies write cost. Beware recursive triggers (a trigger that updates the same table re-fires). Avoid heavy business logic in triggers that's hard to discover; prefer explicit application code unless the invariant must hold no matter who writes.

---

## 15. Views & Materialized Views

**[Advanced]**

```sql
-- A VIEW is a saved query — it stores no data, runs fresh each time it's referenced.
CREATE OR REPLACE VIEW dept_summary AS
SELECT department, count(*) AS headcount, round(avg(salary), 2) AS avg_salary
FROM employee
GROUP BY department;
SELECT * FROM dept_summary;          -- behaves like a read-only table

-- Updatable views: simple single-table views are auto-updatable.
CREATE VIEW eng AS SELECT id, name, salary FROM employee WHERE department = 'Engineering'
WITH CHECK OPTION;     -- reject INSERT/UPDATE that would fall outside the view's WHERE
INSERT INTO eng (name, salary) VALUES ('Dev', 90000);  -- works (department defaults? no -> would fail; illustrative)

-- A MATERIALIZED VIEW stores the query RESULT physically — fast reads, but stale until refreshed.
CREATE MATERIALIZED VIEW dept_report AS
SELECT department, count(*) AS headcount, sum(salary) AS payroll
FROM employee GROUP BY department
WITH DATA;                            -- populate now (WITH NO DATA = create empty, refresh later)

-- Refresh strategies:
REFRESH MATERIALIZED VIEW dept_report;             -- recompute (takes an exclusive lock = blocks reads)
-- CONCURRENTLY avoids blocking readers, but REQUIRES a UNIQUE index on the matview:
CREATE UNIQUE INDEX ON dept_report (department);
REFRESH MATERIALIZED VIEW CONCURRENTLY dept_report;  -- readers keep seeing old data during refresh

-- You can index a materialized view like a table:
CREATE INDEX idx_dept_report_payroll ON dept_report (payroll DESC);
```

| | View | Materialized View |
|---|---|---|
| Storage | none (query only) | stores result rows |
| Freshness | always live | stale until `REFRESH` |
| Read speed | depends on underlying query | fast (precomputed) |
| Indexable | no (index base tables) | yes |
| Use when | abstraction, simple reuse, security boundary | expensive aggregates read far more than they change (dashboards, reports) |

> **Refresh automation:** schedule `REFRESH MATERIALIZED VIEW CONCURRENTLY` via `pg_cron` (extension) or an external scheduler. There's no built-in auto-refresh; you decide the cadence based on how stale your data may be.

---

## 16. Performance Tuning, VACUUM & Partitioning

**[Advanced]**

### VACUUM, ANALYZE & autovacuum

Because of MVCC, dead tuples accumulate. `VACUUM` reclaims their space for reuse; `ANALYZE` updates planner statistics. `autovacuum` does both automatically — but you must understand it.

```sql
VACUUM employee;              -- reclaim dead tuples (space stays in the file, reusable)
VACUUM (VERBOSE, ANALYZE) employee;   -- vacuum + refresh stats + report
VACUUM FULL employee;        -- rewrites the table, RETURNS space to OS, but takes ACCESS EXCLUSIVE lock (blocks everything!)
ANALYZE employee;            -- stats only

-- Inspect dead tuples & last (auto)vacuum:
SELECT relname, n_live_tup, n_dead_tup,
       last_vacuum, last_autovacuum, last_analyze, last_autoanalyze
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;

-- Tune autovacuum per-table for hot tables (more aggressive):
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.02,   -- vacuum when 2% of rows are dead (default 0.2 = 20%)
  autovacuum_analyze_scale_factor = 0.01
);
```

> **Bloat:** tables/indexes with many dead tuples grow larger than the live data warrants. Symptoms: growing disk use, slowing scans. Fixes: ensure autovacuum keeps up (tune scale factors / workers), `REINDEX CONCURRENTLY` bloated indexes, or `pg_repack` (extension) to rewrite a table online (vs the locking `VACUUM FULL`).

⚡ **Version note (PG17):** VACUUM got a new memory-efficient dead-tuple store (TID store), using far less memory and running faster on large tables. PG18 continues with parallelization and async I/O improvements. You generally don't change syntax — just enjoy faster vacuums.

> **Transaction ID wraparound:** PostgreSQL must vacuum old transactions before the 32-bit txid counter wraps, or it will shut down writes to protect data. Monitor `SELECT datname, age(datfrozenxid) FROM pg_database;` — if `age` approaches ~2 billion, autovacuum is falling behind. This is rare with healthy autovacuum but is the classic cause of emergency outages.

### Key configuration parameters

| Parameter | What it does | Typical starting point |
|---|---|---|
| `shared_buffers` | PG's own page cache | 25% of RAM |
| `effective_cache_size` | planner's estimate of OS+PG cache (no allocation) | 50–75% of RAM |
| `work_mem` | memory per sort/hash *operation* (can be many per query!) | 16–64MB; raise for analytics |
| `maintenance_work_mem` | memory for VACUUM, CREATE INDEX, etc. | 256MB–1GB |
| `max_connections` | connection cap | keep modest; use a pooler |
| `wal_compression` | compress WAL | `on` often helps |
| `random_page_cost` | cost of random I/O | 1.1 on SSD/NVMe (default 4.0) |
| `max_wal_size` | checkpoint trigger size | larger = fewer checkpoints |

```sql
SHOW shared_buffers;
ALTER SYSTEM SET work_mem = '64MB';   -- writes to postgresql.auto.conf
SELECT pg_reload_conf();              -- reload (some params need a restart)
SELECT name, setting, pending_restart FROM pg_settings WHERE name = 'shared_buffers';
```

> **`work_mem` warning:** it's per *operation*, and a complex query can use several concurrently, across many connections. Total worst case ≈ `work_mem × operations × connections`. Set it conservatively globally and raise it locally (`SET LOCAL work_mem = '256MB';`) for known analytics queries.

### Connection pooling (PgBouncer)

Each PostgreSQL connection is a heavyweight OS process (~several MB). Thousands of app connections will exhaust memory. **PgBouncer** is a lightweight pooler that multiplexes many client connections onto few server connections.

```ini
; pgbouncer.ini
[databases]
appdb = host=127.0.0.1 port=5432 dbname=appdb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
; transaction pooling: a server conn is assigned only for the duration of a transaction
; -> best concurrency, but you CANNOT use session features (prepared statements caveats, SET, advisory session locks)
pool_mode = transaction
default_pool_size = 20
max_client_conn = 1000
```

> Apps then connect to PgBouncer on port 6432 instead of Postgres on 5432. With `transaction` pooling, avoid session-level state and be careful with server-side prepared statements (modern drivers like `pgx` and recent `node-postgres` handle this; check your driver's pooling notes).

### Partitioning (range / list / hash)

Partitioning splits one logical table into many physical child tables. Benefits: faster queries via **partition pruning**, cheap bulk deletes (drop a partition), and per-partition maintenance.

```sql
-- RANGE partitioning by month (classic for time-series / event data):
CREATE TABLE measurement (
  id      bigint GENERATED ALWAYS AS IDENTITY,
  taken_at timestamptz NOT NULL,
  device  int NOT NULL,
  value   numeric NOT NULL,
  PRIMARY KEY (id, taken_at)        -- partition key MUST be part of PK/unique constraints
) PARTITION BY RANGE (taken_at);

-- Create child partitions:
CREATE TABLE measurement_2026_06 PARTITION OF measurement
  FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
CREATE TABLE measurement_2026_07 PARTITION OF measurement
  FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');
-- A catch-all for anything outside defined ranges:
CREATE TABLE measurement_default PARTITION OF measurement DEFAULT;

-- Inserts route automatically; queries prune to relevant partitions:
EXPLAIN SELECT * FROM measurement WHERE taken_at >= '2026-07-15';  -- scans only _2026_07

-- LIST partitioning (by discrete values, e.g. region):
CREATE TABLE sales (region text, amount numeric) PARTITION BY LIST (region);
CREATE TABLE sales_eu PARTITION OF sales FOR VALUES IN ('DE','FR','ES');
CREATE TABLE sales_us PARTITION OF sales FOR VALUES IN ('US','CA');

-- HASH partitioning (even distribution when there's no natural range/list):
CREATE TABLE big_users (id bigint, name text) PARTITION BY HASH (id);
CREATE TABLE big_users_0 PARTITION OF big_users FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE big_users_1 PARTITION OF big_users FOR VALUES WITH (MODULUS 4, REMAINDER 1);
-- ... remainders 2 and 3 ...

-- Drop old data instantly by dropping/detaching a partition (no big DELETE + VACUUM):
ALTER TABLE measurement DETACH PARTITION measurement_2026_06 CONCURRENTLY;
DROP TABLE measurement_2026_06;
```

> **Partitioning tips:** partition only large tables (millions+ of rows); keep the partition count reasonable (hundreds, not tens of thousands); include the partition key in unique constraints and most `WHERE` clauses so pruning works; consider the `pg_partman` extension to automate partition creation/retention.

---

## 17. Backup & Restore (incl. PITR & Incremental)

**[Advanced]**

### Logical backups: pg_dump / pg_restore

Logical backups export SQL/data — portable across versions/architectures, restorable selectively.

```bash
# Plain SQL dump (human-readable, restore with psql):
pg_dump -h localhost -U app -d appdb -f appdb.sql

# Custom format (compressed, allows selective & parallel restore — RECOMMENDED):
pg_dump -h localhost -U app -d appdb -Fc -f appdb.dump

# Directory format enables PARALLEL dump (-j) — fastest for big DBs:
pg_dump -d appdb -Fd -j 4 -f appdb_dir

# Dump only schema, or only data, or specific tables:
pg_dump -d appdb --schema-only -f schema.sql
pg_dump -d appdb --data-only   -f data.sql
pg_dump -d appdb -t public.employee -t public.orders -f two_tables.dump -Fc

# Dump ALL databases + roles/tablespaces (cluster-wide):
pg_dumpall -h localhost -U postgres -f cluster.sql

# Restore:
psql -d newdb -f appdb.sql                       # for plain-SQL dumps
createdb newdb
pg_restore -d newdb -Fc appdb.dump               # custom/dir formats
pg_restore -d newdb -j 4 appdb_dir               # parallel restore
pg_restore -d newdb -t employee appdb.dump       # restore a single table
pg_restore -d newdb --clean --if-exists appdb.dump  # drop existing objects first
```

### Physical backups: pg_basebackup

Physical backups copy the actual data files — exact byte-for-byte cluster, ideal for full-cluster recovery and as a base for replication/PITR. Same major version & architecture required.

```bash
# Take a base backup of the whole cluster:
pg_basebackup -h localhost -U replicator -D /backup/base -Ft -z -P
#  -D dir : output  -Ft tar  -z gzip  -P progress
```

### Point-in-Time Recovery (PITR)

PITR = a base backup + continuous WAL archiving, letting you restore to any moment (e.g. just before a bad `DELETE`).

```ini
# postgresql.conf — enable WAL archiving:
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'   # copy each completed WAL segment
```

```bash
# 1) Base backup (as above) -> /backup/base
# 2) WAL segments accumulate in /archive
# 3) To recover to a point in time, restore the base backup, then create:
#    (recovery settings live in postgresql.conf + a 'recovery.signal' file, PG12+)
```

```ini
# postgresql.conf on the restore target:
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2026-06-21 14:25:00+00'   # stop replaying WAL at this instant
recovery_target_action = 'promote'
# create an empty file named recovery.signal in the data dir, then start postgres.
```

### Incremental backup (PG17)

⚡ **Version note (PG17):** PostgreSQL 17 added **native incremental physical backups**. `pg_basebackup --incremental` captures only blocks changed since a prior backup (tracked via the WAL summarizer); `pg_combinebackup` reconstructs a full backup from a chain.

```ini
# postgresql.conf — enable change tracking for incrementals:
summarize_wal = on
```

```bash
# 1) Full base backup:
pg_basebackup -D /backup/full -c fast

# 2) Later, an incremental relative to the full backup's manifest:
pg_basebackup -D /backup/incr1 --incremental=/backup/full/backup_manifest

# 3) Reconstruct a usable full backup by combining the chain:
pg_combinebackup /backup/full /backup/incr1 -o /backup/restored
```

> **Backup best practices:** automate backups, test **restores** regularly (an untested backup is not a backup), store copies off-site, monitor that WAL archiving is keeping up, and document your RPO (how much data you can lose) and RTO (how fast you must recover). Tools like **pgBackRest** and **Barman** wrap pg_basebackup/WAL/PITR/incrementals with retention, compression, and verification.

---

## 18. Replication & High Availability

**[Advanced]**

### Streaming (physical) replication

A **primary** ships its WAL stream to one or more **standby** (replica) servers, which replay it to stay byte-identical. Standbys can serve read-only queries (read replicas) and be promoted on failover.

```ini
# On the PRIMARY (postgresql.conf):
wal_level = replica
max_wal_senders = 10
wal_keep_size = '1GB'         # keep WAL around so a lagging standby can catch up
# (better: use a replication slot so WAL is retained until consumed)
```

```sql
-- Create a replication role and a slot on the primary:
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'secret';
SELECT pg_create_physical_replication_slot('standby1');  -- prevents needed WAL from being removed
```

```bash
# On the STANDBY: clone the primary and configure it to follow:
pg_basebackup -h primary_host -U replicator -D /var/lib/postgresql/data -R -P -Xs
#  -R writes the connection info + creates 'standby.signal' so it starts as a replica.
```

```sql
-- Monitor replication from the PRIMARY:
SELECT client_addr, state, sent_lsn, replay_lsn,
       pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS replay_lag
FROM pg_stat_replication;

-- On the standby, check if it's in recovery and how far behind:
SELECT pg_is_in_recovery();                         -- true on a standby
SELECT now() - pg_last_xact_replay_timestamp() AS lag;
```

- **Asynchronous** (default): primary commits without waiting for standbys — fast, but a crash can lose the last few transactions on the standby.
- **Synchronous**: set `synchronous_standby_names`; the primary waits for standby confirmation — zero data loss at the cost of latency.

### Logical replication

Replicates at the **row/statement level** by publishing changes from selected tables — works across major versions, allows replicating a subset, and supports many-to-one consolidation.

```sql
-- On the SOURCE (publisher): wal_level = logical required.
CREATE PUBLICATION mypub FOR TABLE employee, orders;
-- or FOR ALL TABLES;

-- On the TARGET (subscriber):
CREATE SUBSCRIPTION mysub
  CONNECTION 'host=source_host dbname=appdb user=replicator password=secret'
  PUBLICATION mypub;
-- The subscriber copies initial data, then streams ongoing changes.
```

⚡ **Version note (PG17):** logical replication gained **failover slots** (slot state can sync to physical standbys so logical replication survives a failover) and the ability to upgrade a subscriber with `pg_upgrade` without losing replication state — major operational improvements. PG18 further improves logical replication of sequences and conflict handling.

### High availability overview

PostgreSQL provides the replication primitives; **automatic failover** (electing a new primary when one dies) is handled by external tooling:
- **Patroni** (with etcd/Consul/ZooKeeper) — the most common open-source HA orchestrator.
- **repmgr**, **pg_auto_failover** — alternatives.
- A **load balancer / VIP / PgBouncer** routes clients to the current primary and read replicas.

> **Read replicas** offload read-heavy traffic (reports, analytics) from the primary. Beware **replication lag**: a read replica may not yet have a row your app just wrote to the primary ("read your own writes" problem). Route writes and read-after-write to the primary, or use synchronous replication for critical reads.

---

## 19. Security — Roles, Privileges, RLS, SSL

**[Advanced]**

### Privileges: GRANT / REVOKE

PostgreSQL privileges are object-level. Grant the minimum needed.

```sql
-- Database & schema level:
GRANT CONNECT ON DATABASE appdb TO readonly;
GRANT USAGE ON SCHEMA public TO readonly;          -- needed to "see into" the schema

-- Table privileges:
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
GRANT SELECT, INSERT, UPDATE, DELETE ON employee TO app_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_user;  -- needed for IDENTITY/serial inserts

-- Column-level privileges:
GRANT SELECT (id, name) ON employee TO readonly;   -- can read only id & name, not salary

-- REVOKE removes privileges:
REVOKE INSERT ON employee FROM app_user;

-- DEFAULT PRIVILEGES: auto-grant on FUTURE objects created by a role:
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT ON TABLES TO readonly;              -- new tables auto-grant SELECT to readonly

-- Inspect privileges:
\dp employee
SELECT grantee, privilege_type FROM information_schema.role_table_grants
WHERE table_name = 'employee';
```

> **The PUBLIC pitfall:** by default `PUBLIC` (every role) has `CONNECT` on new databases and historically `CREATE`/`USAGE` on the `public` schema. Since PG15, the `public` schema no longer grants `CREATE` to PUBLIC by default. Still, `REVOKE ALL ON DATABASE appdb FROM PUBLIC;` and lock down `public` for a secure baseline.

### Row-Level Security (RLS)

RLS filters which *rows* a role can see/modify, enforced by the database itself — perfect for multi-tenancy.

```sql
CREATE TABLE document (
  id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  owner   text NOT NULL,        -- which user owns this row
  content text NOT NULL
);

-- 1) Turn RLS on for the table:
ALTER TABLE document ENABLE ROW LEVEL SECURITY;
-- (Optionally FORCE so even the table owner is subject to policies:)
ALTER TABLE document FORCE ROW LEVEL SECURITY;

-- 2) Define policies. This one: a user can only see/modify their own rows.
CREATE POLICY owner_isolation ON document
  USING (owner = current_user)            -- visibility (SELECT/UPDATE/DELETE see only matching rows)
  WITH CHECK (owner = current_user);      -- writes must satisfy this (can't insert rows for others)

-- Separate read vs write policies, or per-command policies, are possible:
CREATE POLICY read_all ON document FOR SELECT USING (true);
CREATE POLICY write_own ON document FOR INSERT WITH CHECK (owner = current_user);

-- Multi-tenant pattern using a session variable set by your app per request:
-- app runs:  SET app.tenant_id = '42';
CREATE POLICY tenant_isolation ON document
  USING (owner = current_setting('app.tenant_id', true));
```

> **RLS gotcha:** the table *owner* and *superusers* bypass RLS unless you use `FORCE ROW LEVEL SECURITY`. Also, policies add a filter to every query — keep policy expressions indexable (e.g. index `tenant_id`).

### Authentication & SSL

```text
# pg_hba.conf controls who may connect, from where, to which DB, with which method.
# TYPE  DATABASE  USER     ADDRESS          METHOD
local   all       all                       scram-sha-256   # unix socket
host    appdb     app_user 10.0.0.0/8       scram-sha-256   # password (hashed)
hostssl appdb     app_user 0.0.0.0/0        scram-sha-256   # require SSL for this rule
host    all       all      0.0.0.0/0        reject          # deny everything else
```

```ini
# postgresql.conf — enable TLS:
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file  = 'server.key'
```

```bash
# Clients enforce TLS via sslmode (verify-full = encrypted + cert + hostname checked):
psql "postgresql://app@host/appdb?sslmode=verify-full&sslrootcert=ca.crt"
```

> **Security checklist:** use `scram-sha-256` (not `md5`/`trust`); require SSL for remote connections (`sslmode=verify-full`); least-privilege roles; never connect apps as a superuser; rotate credentials; keep PG patched; restrict `listen_addresses`; audit with `pgaudit` if you need compliance logging.

---

## 20. Extensions — pgvector, PostGIS, pg_trgm & more

**[Advanced]**

Extensions add types, functions, and index methods. Installed per-database with `CREATE EXTENSION`.

```sql
SELECT * FROM pg_available_extensions ORDER BY name;   -- what's installable
\dx                                                    -- what's installed
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

### pgvector — vector similarity search (AI/RAG, very relevant in 2026)

`pgvector` stores embedding vectors and does nearest-neighbor search — the backbone of retrieval-augmented generation (RAG) and semantic search, letting you keep your vectors *next to* your relational data.

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE chunk (
  id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  content   text NOT NULL,
  embedding vector(1536)        -- e.g. an OpenAI/Cohere embedding dimension
);

-- Distance operators:  <-> L2,  <#> negative inner product,  <=> cosine distance.
-- Find the 5 most similar chunks to a query embedding:
SELECT id, content, embedding <=> '[0.01, 0.02, ...]' AS cosine_distance
FROM chunk
ORDER BY embedding <=> '[0.01, 0.02, ...]'   -- ORDER BY the distance = nearest neighbors
LIMIT 5;

-- ANN indexes for speed on large tables:
-- HNSW (best recall/speed, more memory):
CREATE INDEX ON chunk USING hnsw (embedding vector_cosine_ops);
-- IVFFlat (faster build, needs 'lists' tuned ~ rows/1000; ANALYZE first):
CREATE INDEX ON chunk USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

-- Hybrid search: combine vector similarity with a SQL filter (a Postgres superpower):
SELECT content FROM chunk
WHERE content ILIKE '%postgres%'                 -- relational/keyword filter
ORDER BY embedding <=> '[...]' LIMIT 5;          -- + semantic ranking
```

### PostGIS — geospatial

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE TABLE place (
  id   bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name text,
  geom geography(Point, 4326)     -- WGS84 lon/lat
);
INSERT INTO place (name, geom) VALUES ('HQ', 'SRID=4326;POINT(-0.1276 51.5072)');
CREATE INDEX idx_place_geom ON place USING gist (geom);   -- spatial index
-- Everything within 1km of a point:
SELECT name FROM place
WHERE ST_DWithin(geom, 'SRID=4326;POINT(-0.13 51.51)'::geography, 1000);
```

### pg_trgm — fuzzy / substring search

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
-- Trigram GIN index makes ILIKE '%term%' and similarity() fast:
CREATE INDEX idx_emp_name_trgm ON employee USING gin (name gin_trgm_ops);
SELECT name, similarity(name, 'adya') AS sim
FROM employee WHERE name % 'adya'        -- '%' = similar enough (typo-tolerant)
ORDER BY sim DESC;
SELECT * FROM employee WHERE name ILIKE '%ace%';   -- now index-assisted
```

### Other commonly-used extensions

```sql
-- uuid-ossp: extra UUID generators (gen_random_uuid is now built-in, so often unnecessary).
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
SELECT uuid_generate_v4();

-- pgcrypto: hashing & encryption.
CREATE EXTENSION IF NOT EXISTS pgcrypto;
SELECT crypt('mypassword', gen_salt('bf'));      -- bcrypt password hash
SELECT digest('hello', 'sha256');                -- SHA-256

-- citext: case-INsensitive text (great for emails/usernames).
CREATE EXTENSION IF NOT EXISTS citext;
CREATE TABLE account2 (email citext UNIQUE);     -- 'A@x.com' and 'a@x.com' collide

-- hstore: simple key/value string store (predates JSONB; JSONB usually preferred now).
CREATE EXTENSION IF NOT EXISTS hstore;
SELECT 'a=>1, b=>2'::hstore -> 'a';              -- '1'

-- Other notables: pg_stat_statements (§12), btree_gist/btree_gin (mixed exclusion constraints),
-- pg_cron (in-DB scheduler), pg_partman (partition automation), postgres_fdw (query remote PGs).
```

---

## 21. Connecting from Apps — Node.js & Go

**[Advanced]**

The golden rule for every language: **never build SQL by string concatenation** — always use parameterized queries so the driver sends values separately from the SQL text. This eliminates SQL injection and lets the server cache plans.

### Node.js with `pg` (node-postgres)

```js
// npm install pg
import pg from "pg";
const { Pool } = pg;

// A Pool reuses connections — create ONE per process, never per request.
const pool = new Pool({
  connectionString: process.env.DATABASE_URL, // postgres://user:pass@host:5432/db
  max: 20,                 // max connections in the pool
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,
  // ssl: { rejectUnauthorized: true, ca: fs.readFileSync("ca.crt") }, // for TLS
});

// Parameterized query: $1, $2 are placeholders; values are passed separately.
// This is INJECTION-SAFE — the email is never interpolated into the SQL string.
async function findByEmail(email) {
  const { rows } = await pool.query(
    "SELECT id, name FROM customer WHERE email = $1", // NEVER: `... = '${email}'`
    [email]
  );
  return rows[0] ?? null;
}

// Transaction: you MUST use a single dedicated client (not pool.query) so all
// statements run on the same connection.
async function transfer(fromId, toId, amount) {
  const client = await pool.connect();
  try {
    await client.query("BEGIN");
    await client.query(
      "UPDATE account SET balance = balance - $1 WHERE id = $2",
      [amount, fromId]
    );
    await client.query(
      "UPDATE account SET balance = balance + $1 WHERE id = $2",
      [amount, toId]
    );
    await client.query("COMMIT");
  } catch (err) {
    await client.query("ROLLBACK"); // undo on any error
    throw err;
  } finally {
    client.release();               // ALWAYS return the client to the pool
  }
}

// Graceful shutdown:
process.on("SIGTERM", () => pool.end());
```

> **Prisma note:** Prisma (covered in its own guide) is a higher-level type-safe ORM that sits on top of a driver. It auto-parameterizes everything (injection-safe), manages its own connection pool, and exposes `$queryRaw\`...\`` (tagged template = parameterized) for raw SQL. With PgBouncer in transaction mode, set `?pgbouncer=true` and disable prepared statements as Prisma's docs describe. Use `pg` directly when you want full SQL control; use Prisma for productivity and end-to-end types.

### Go with `pgx` v5 (the modern driver)

```go
// go get github.com/jackc/pgx/v5/pgxpool
package main

import (
	"context"
	"errors"
	"log"
	"os"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"
)

func main() {
	ctx := context.Background()

	// pgxpool is the connection pool — create ONE and share it.
	pool, err := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Fatal(err)
	}
	defer pool.Close()

	// Parameterized query ($1, $2). pgx sends args separately => injection-safe.
	var id int64
	var name string
	err = pool.QueryRow(ctx,
		"SELECT id, name FROM customer WHERE email = $1", // NEVER concatenate the value
		"ada@example.com",
	).Scan(&id, &name)
	if errors.Is(err, pgx.ErrNoRows) {
		log.Println("not found")
	} else if err != nil {
		log.Fatal(err)
	}
	log.Printf("found %d %s", id, name)

	// Query many rows; pgx v5 has CollectRows helpers:
	rows, _ := pool.Query(ctx, "SELECT id, name FROM customer WHERE salary > $1", 100000)
	defer rows.Close()
	for rows.Next() {
		var cid int64
		var cname string
		if err := rows.Scan(&cid, &cname); err != nil {
			log.Fatal(err)
		}
		log.Println(cid, cname)
	}
	if err := rows.Err(); err != nil { // ALWAYS check after the loop
		log.Fatal(err)
	}

	// Transaction with the helper that auto-rollbacks on error / panic:
	err = pgx.BeginFunc(ctx, pool, func(tx pgx.Tx) error {
		if _, err := tx.Exec(ctx,
			"UPDATE account SET balance = balance - $1 WHERE id = $2", 100, 1); err != nil {
			return err // returning an error triggers automatic ROLLBACK
		}
		if _, err := tx.Exec(ctx,
			"UPDATE account SET balance = balance + $1 WHERE id = $2", 100, 2); err != nil {
			return err
		}
		return nil // nil triggers COMMIT
	})
	if err != nil {
		log.Fatal(err)
	}
}
```

> **Driver notes:** `pgx` v5 is the modern, actively-developed Go driver; use `pgxpool` for pooling. It also implements `database/sql` via `stdlib` if you need that interface. `pgx` exposes Postgres-specific features (COPY protocol via `CopyFrom` for fast bulk loads, listen/notify, rich type mapping) that the generic `lib/pq` (now in maintenance mode) does not. For bulk inserts, `pool.CopyFrom(...)` is far faster than looping `INSERT`.

> **SQL injection — the one rule:** parameterize. In every language the pattern is the same: send the query text with placeholders and the values in a separate argument. Never do `"... WHERE name = '" + userInput + "'"`. If you must build dynamic SQL (e.g. variable column names that can't be parameters), validate them against an allow-list, never against raw user input.

---

## 22. Gotchas & Best Practices

**[All levels]**

### Timezones
- Use **`timestamptz`**, not `timestamp`, for any instant in time. Store UTC; convert at the edges with `AT TIME ZONE`.
- `timestamptz` stores no zone — it's a UTC instant displayed in your session's `TimeZone`.

### NULL handling
- `NULL = NULL` is **NULL** (unknown), not true. Use `IS NULL` / `IS NOT NULL`.
- `NOT IN (subquery)` returns nothing if the subquery yields any NULL — prefer `NOT EXISTS`.
- Aggregates **skip NULLs**: `count(col)` ignores NULLs; `count(*)` doesn't. `avg`/`sum` ignore NULLs.
- `NULL` sorts last by default in `ASC` (use `NULLS FIRST/LAST` to control).
- `COALESCE(x, fallback)` and `col IS DISTINCT FROM other` (NULL-safe `<>`) are your friends.

### Performance & schema
- **Index your foreign keys** — PG doesn't do it for you, and unindexed FKs make parent deletes and joins slow.
- Avoid **N+1 queries** from ORMs: fetch related rows with a `JOIN` or `IN (...)`/`json_agg`, not one query per row.
- Prefer **keyset (seek) pagination** over `OFFSET` for deep pages:

```sql
-- OFFSET 100000 still scans & discards 100000 rows. Keyset jumps straight there:
SELECT * FROM employee WHERE id > 100000 ORDER BY id LIMIT 20;  -- pass last seen id as the cursor
```

- Use `EXPLAIN (ANALYZE, BUFFERS)` before optimizing — measure, don't guess.
- `SELECT *` fetches every column (more I/O, breaks index-only scans, brittle to schema changes) — list columns you need.
- Batch writes (multi-row INSERT, `COPY`) instead of row-by-row.
- Watch `work_mem` × concurrency; keep `max_connections` modest and pool with PgBouncer.

### Correctness
- DDL is transactional — wrap risky migrations in `BEGIN; ... COMMIT;` and test rollback.
- Sequences have gaps; never rely on contiguity.
- `TRUNCATE` skips row triggers and can't be filtered — it's not a `DELETE`.
- Long-running transactions hold back `VACUUM` (dead tuples can't be reclaimed while an old snapshot might need them) — keep transactions short.
- Set statement/lock timeouts in apps to avoid runaway queries holding locks:

```sql
SET statement_timeout = '30s';        -- abort any statement over 30s
SET lock_timeout = '5s';              -- don't wait forever for a lock
SET idle_in_transaction_session_timeout = '60s';  -- kill idle-in-txn sessions (they block VACUUM)
```

### Operational
- Test your **restores**, not just your backups.
- Monitor: dead tuples, table/index bloat, replication lag, `pg_stat_statements`, connection counts, `age(datfrozenxid)` (wraparound), disk space.
- Run `ANALYZE` after big data loads so the planner has fresh stats.
- Use `CREATE INDEX CONCURRENTLY` / `REINDEX CONCURRENTLY` in production to avoid blocking writes.

### Quick reference: useful catalog/admin queries

```sql
-- Database & table sizes:
SELECT pg_size_pretty(pg_database_size(current_database()));
SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) AS total
FROM pg_catalog.pg_statio_user_tables ORDER BY pg_total_relation_size(relid) DESC LIMIT 10;

-- Cache hit ratio (want > 0.99):
SELECT sum(heap_blks_hit) / nullif(sum(heap_blks_hit + heap_blks_read), 0) AS hit_ratio
FROM pg_statio_user_tables;

-- Active connections by state:
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;

-- Blocking tree (who is blocking whom):
SELECT blocked.pid AS blocked_pid, blocking.pid AS blocking_pid,
       blocked.query AS blocked_query, blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

---

## 23. Study Path & Build-to-Learn Projects

**[All levels]**

### Suggested learning order

1. **Week 1 — Foundations (Beginner):** Install via Docker. Live in `psql`. Master meta-commands (§1). Learn data types (§2) and DDL (§3). Create databases, roles, tables with constraints. Do CRUD by hand (§4): INSERT/UPDATE/DELETE/SELECT, upserts, `RETURNING`.
2. **Week 2 — Querying (Intermediate):** Every join type and set operation (§5). Aggregation, `HAVING`, `FILTER`, `ROLLUP` (§6). Subqueries, CTEs, recursive CTEs (§7). Window functions until ranking and running totals are second nature (§8).
3. **Week 3 — Semi-structured & search (Intermediate):** JSONB operators, indexing, `JSON_TABLE`, building JSON output (§9). Arrays and full-text search with ranking (§10).
4. **Week 4 — Performance (Advanced):** Index types and when each applies (§11). Read `EXPLAIN ANALYZE` plans fluently; set up `pg_stat_statements` (§12). Understand MVCC, isolation levels, and locking; build a `SKIP LOCKED` job queue (§13).
5. **Week 5 — Programmability & ops (Advanced):** PL/pgSQL functions, procedures, triggers (§14). Views & materialized views (§15). VACUUM/autovacuum, config tuning, partitioning (§16).
6. **Week 6 — Production (Advanced):** Backups, PITR, incremental backup (§17). Replication & HA (§18). Security: roles, RLS, SSL (§19). Extensions, especially pgvector (§20). Connect from your app safely (§21).

### Build-to-learn projects

- **Mini-blog / CMS:** users, posts, tags (arrays), comments. Add full-text search with ranking and `ts_headline` snippets. Add an `updated_at` trigger and an audit-log trigger.
- **Job queue:** a `job` table consumed by multiple workers using `FOR UPDATE SKIP LOCKED`. Add retry counts, backoff, and a dead-letter status. Load-test with concurrent workers.
- **Analytics dashboard:** an `events` table partitioned by month (range partitioning). Pre-aggregate with materialized views refreshed `CONCURRENTLY`. Practice `EXPLAIN` to confirm partition pruning and index-only scans.
- **Multi-tenant SaaS schema:** enforce tenant isolation with Row-Level Security policies driven by a session variable. Verify a tenant cannot read another's rows.
- **Semantic search / RAG store:** a `chunk` table with `pgvector` embeddings. Build hybrid search (vector `<=>` distance + a SQL `WHERE` filter) and add an HNSW index. Compare recall/latency vs IVFFlat.
- **Money transfer service:** an `account` table; implement transfers as `SERIALIZABLE` transactions with a retry loop on `40001`. Reproduce and then prevent a deadlock by ordering lock acquisition.
- **Backup & recovery drill:** set up WAL archiving, take a base + incremental backup (PG17), simulate a bad `DELETE`, and recover to a point in time. Then stand up a streaming replica and promote it.

### Cheat-sheet of "look it up later" commands

```text
\? \h           -- psql & SQL help
\d \d+ \dt \di  -- describe objects
EXPLAIN (ANALYZE, BUFFERS)   -- diagnose any slow query
\timing on      -- measure query time
pg_stat_statements           -- find the worst queries
pg_stat_activity / pg_locks  -- see what's running / blocked
VACUUM (VERBOSE, ANALYZE)    -- reclaim + restats
pg_dump -Fc / pg_restore     -- backup & restore
```

> **Final advice:** PostgreSQL rewards curiosity. When something is slow, `EXPLAIN ANALYZE` it. When a constraint feels awkward, ask whether the schema models reality. Reach for built-in features (constraints, RLS, partitioning, FTS, JSONB, pgvector) before bolting on external systems — Postgres can usually do it, do it transactionally, and do it well.
