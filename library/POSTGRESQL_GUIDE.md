# PostgreSQL — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Developers and aspiring DBAs who want to learn PostgreSQL deeply — from `CREATE TABLE` to query-plan tuning, MVCC, replication, and AI vector search — without needing to look anything up online. This guide is written **explain-first**: every concept is introduced in prose (what it *is*, the *logic/why*, what it's *for* and *when* to reach for it, *how* to use it, the key parameters, best practices, and security recommendations), and only *then* shown as runnable, heavily commented SQL you can paste straight into `psql` and execute. Read it top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** Beginner, **[I]** Intermediate, or **[A]** Advanced so you can navigate by skill level.
>
> **Version note:** This guide targets **PostgreSQL 17 and 18** (**PG 18** is the current stable release as of June 2026, GA September 2025; PG 17 remains extremely common and everything here applies to both). PG 17 brought `JSON_TABLE` and a fuller SQL/JSON suite, incremental physical backups (`pg_basebackup --incremental` + `pg_combinebackup`), a faster and lower-memory `VACUUM` (the new TID store), improved logical-replication slot synchronization (failover slots), and `MERGE ... RETURNING`. Where **PostgreSQL 18** (released late 2025) is relevant — e.g. asynchronous I/O, `uuidv7()`, virtual generated columns, and skip-scan for B-tree indexes — it is called out with **⚡ Version note**. Always confirm exact behaviour against the official docs for your installed `SELECT version();`.
>
> **How this fits the library:** PostgreSQL is the *engine*; three sibling guides cover the surrounding craft. **`RELATIONAL_DB_DESIGN_GUIDE.md`** teaches the theory you should apply *before* you type `CREATE TABLE` — normalization, keys, relationships, ER modelling — and this guide assumes you will lean on it for schema design decisions. **`PRISMA_ORM_GUIDE.md`** shows the type-safe ORM most TypeScript apps use on top of Postgres; we cross-reference it wherever raw SQL meets application code. **`GO_GUIDE.md`** is the language reference for the Go (`pgx`) examples in §21. Cross-references appear inline as **→ see RELATIONAL_DB_DESIGN_GUIDE.md**, etc.

---

## Table of Contents

1. [Installation, psql & First Connection](#1-installation-psql--first-connection) **[B]**
2. [Data Types in Depth](#2-data-types-in-depth) **[B]**
3. [DDL — Tables, Constraints, Schemas, Sequences](#3-ddl--tables-constraints-schemas-sequences) **[B]**
4. [DML & Queries — INSERT, UPDATE, DELETE, SELECT](#4-dml--queries--insert-update-delete-select) **[B]**
5. [Joins & Set Operations](#5-joins--set-operations) **[I]**
6. [Aggregation — GROUP BY, HAVING, GROUPING SETS](#6-aggregation--group-by-having-grouping-sets) **[I]**
7. [Subqueries & CTEs (incl. Recursive)](#7-subqueries--ctes-incl-recursive) **[I]**
8. [Window Functions in Depth](#8-window-functions-in-depth) **[I]**
9. [Working with JSON & JSONB](#9-working-with-json--jsonb) **[I]**
10. [Arrays & Full-Text Search](#10-arrays--full-text-search) **[I]**
11. [Indexing Deeply](#11-indexing-deeply) **[A]**
12. [EXPLAIN, Query Plans & Finding Slow Queries](#12-explain-query-plans--finding-slow-queries) **[A]**
13. [Transactions, MVCC & Concurrency](#13-transactions-mvcc--concurrency) **[A]**
14. [Functions, Procedures & Triggers (PL/pgSQL)](#14-functions-procedures--triggers-plpgsql) **[A]**
15. [Views & Materialized Views](#15-views--materialized-views) **[A]**
16. [Performance Tuning, VACUUM & Partitioning](#16-performance-tuning-vacuum--partitioning) **[A]**
17. [Backup & Restore (incl. PITR & Incremental)](#17-backup--restore-incl-pitr--incremental) **[A]**
18. [Replication & High Availability](#18-replication--high-availability) **[A]**
19. [Security — Roles, Privileges, RLS, SSL](#19-security--roles-privileges-rls-ssl) **[A]**
20. [Extensions — pgvector, PostGIS, pg_trgm & more](#20-extensions--pgvector-postgis-pg_trgm--more) **[A]**
21. [Connecting from Apps — Node.js & Go](#21-connecting-from-apps--nodejs--go) **[A]**
22. [Gotchas & Best Practices](#22-gotchas--best-practices) **[All]**
23. [Study Path & Build-to-Learn Projects](#23-study-path--build-to-learn-projects)

---

## 1. Installation, psql & First Connection

**[Beginner]**

### What PostgreSQL is — and *why* it dominates

PostgreSQL ("Postgres") is a free, open-source, **object-relational** database management system (RDBMS). "Relational" means data lives in tables of rows and columns related by keys (the theory of which is **→ see RELATIONAL_DB_DESIGN_GUIDE.md**); "object-" means Postgres extends the relational model with rich user-definable types, inheritance, and a pluggable extension system. It is known for three things that, together, make it the default choice for new projects in 2026:

1. **Correctness and standards compliance.** Postgres implements the SQL standard more faithfully than most rivals, enforces constraints rigorously, and gives you full **ACID** transactions (explained in §13). It would rather reject bad data than silently corrupt it.
2. **Extensibility.** You can add new data types, operators, index methods, procedural languages, and whole feature sets (`pgvector` for AI embeddings, `PostGIS` for maps) without forking the database. This is why Postgres keeps absorbing capabilities that used to require separate systems.
3. **A world-class query planner.** Postgres's cost-based optimizer (§12) can rewrite and re-order your query into an efficient execution plan, which is what lets you write *declarative* SQL ("what I want") and trust the engine to figure out *how*.

**The mental model you must hold:** a running PostgreSQL is a **server process** (historically the *postmaster*, today usually just called the `postgres` process) that manages a **cluster** — one physical data directory containing one or more **databases**. Each database contains **schemas** (namespaces), and schemas contain **tables**, **views**, **functions**, and so on. You never touch the files directly; you talk to the server with a **client** over a connection. Clients include the built-in `psql` terminal, GUIs (pgAdmin, DBeaver, TablePlus), and your application's **driver** (the `pg` library in Node, `pgx` in Go — §21). Everything in this guide is "send SQL/commands to the server, read results back."

**When to choose Postgres vs alternatives:** reach for Postgres for almost any transactional application — web/mobile backends, SaaS, analytics that fit on one big box, anything needing JSON + relational in one place, or vector search for AI. Reach for **SQLite** (→ see SQLITE3_GUIDE.md) when you want a zero-server embedded file database (mobile, desktop, tests). Reach for a **document store like MongoDB** (→ see MONGODB_GUIDE.md) only when your data is genuinely schemaless *and* you don't need joins/transactions across documents — and even then, Postgres's JSONB (§9) often covers the same ground while keeping relational power.

### Install with Docker (recommended for learning)

Docker is the fastest way to get a clean, throwaway PostgreSQL without modifying your operating system. The logic: the official `postgres` image is a pre-built, correctly configured server; you give it a password and a port and it runs. When you're done you can delete it and your machine is untouched — perfect for experimenting where you might make a mess. (For the full Docker treatment → see DOCKER_GUIDE.md.)

```bash
# Pull and run PostgreSQL 17 in the background.
#  --name        : container name so you can reference it later
#  -e POSTGRES_PASSWORD : sets the password for the default 'postgres' superuser (REQUIRED — the
#                         image refuses to start without it, a safety default)
#  -e POSTGRES_USER     : (optional) name of the superuser to create (default: postgres)
#  -e POSTGRES_DB       : (optional) name of a database to create on first boot
#  -p 5432:5432         : map container port 5432 to your host's 5432 (host:container)
#  -v pgdata:/var/lib/postgresql/data : persist data in a named volume so it survives container removal
#  -d                   : detached (run in the background)
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

# Tail the server logs (the FIRST place to look when connections or auth fail)
docker logs -f pg17

# Stop / start / remove
docker stop pg17
docker start pg17
docker rm -f pg17          # remove the container (named volume 'pgdata' is KEPT — data survives!)
docker volume rm pgdata    # only THIS deletes the persisted data
```

> **Security note (don't ship this):** `POSTGRES_PASSWORD=secret` and a port published to `0.0.0.0` are fine on your laptop but dangerous in production — a Postgres exposed to the internet with a weak password is a classic breach. For real deployments use a strong, secret-managed password, bind only to a private network, and require TLS (§19).

A reproducible `docker-compose.yml` (preferable once you have more than one service) — the `healthcheck` matters because it lets dependent containers wait until Postgres is *actually* accepting connections, not just "started":

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
    # Health check so dependent services wait until PG is actually ready, not merely launched:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 5s
      timeout: 3s
      retries: 5
volumes:
  pgdata:
```

### Install natively

A native install runs Postgres as a normal OS service. This is what you'll do on a server. The key difference from Docker is that an OS user named `postgres` is created and owns the data directory; the conventional way to get your first superuser shell is to *become* that OS user (`sudo -u postgres`), because the default authentication for local connections trusts the matching OS identity (peer auth).

```bash
# --- Debian / Ubuntu (uses the PGDG apt repo for the latest version) ---
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh   # adds the official PGDG repo
sudo apt install -y postgresql-17
# The 'postgres' OS user is created; switch to it and run psql:
sudo -u postgres psql

# --- macOS (Homebrew) ---
brew install postgresql@17
brew services start postgresql@17   # run as a background service that restarts on login
psql postgres                       # connects as your macOS username (peer auth)

# --- Windows ---
# Use the EDB installer from enterprisedb.com, or:
#   winget install PostgreSQL.PostgreSQL.17
# Then use the bundled "SQL Shell (psql)" Start-menu item, or pgAdmin.
```

After install, the server listens on TCP port **5432** by default. The default superuser is **`postgres`**, and a default database **`postgres`** exists (used as a neutral place to connect when you haven't created your own database yet).

### Connecting with psql — your primary tool

`psql` is the standard interactive terminal client, and you should become fluent in it even if you usually use a GUI — it's always available, scriptable, and the fastest way to inspect a database. It accepts two kinds of input: ordinary **SQL** (terminated by `;`) and **meta-commands** that start with a backslash (`\`), which are psql features (not SQL) for inspecting and controlling the session.

There are several equivalent ways to specify *what* to connect to; pick whichever is convenient. Understanding the pieces — host, port, user, database — matters because connection problems are almost always one of these four being wrong, or a `pg_hba.conf` rule rejecting the combination (see the gotcha below).

```bash
# Connection options (any combination):
#   -h host   -p port   -U user   -d database   -W (force a password prompt)
psql -h localhost -p 5432 -U app -d appdb

# Connection string / URI form (handy for one-liners and matches what apps use):
psql "postgresql://app:secret@localhost:5432/appdb"
psql "postgresql://app@localhost/appdb?sslmode=require"

# Environment variables psql/libpq honor (avoids retyping flags every time):
export PGHOST=localhost PGPORT=5432 PGUSER=app PGDATABASE=appdb PGPASSWORD=secret
psql   # now connects with no flags
```

> **Security tip:** Don't put passwords in `PGPASSWORD` on shared machines — environment variables can leak into process listings and shell history. Use a `~/.pgpass` file (with `chmod 600` so only you can read it) containing lines like `hostname:port:database:username:password`; libpq reads it automatically. This keeps secrets out of your command line and CI logs.

### Essential psql meta-commands

These are your "look around the database" toolkit. The pattern is consistent: `\d` describes, lowercase letters narrow to an object type (`\dt` tables, `\di` indexes, `\df` functions…), and a trailing `+` adds detail. Memorizing a dozen of these will make you dramatically faster than clicking through a GUI.

```text
\?              -- list ALL psql meta-commands (your cheat sheet — start here when stuck)
\h              -- SQL help index;  \h CREATE TABLE  -- syntax help for a specific statement
\l   (\list)    -- list databases
\c dbname       -- connect to another database (\connect)
\c dbname user  -- connect as a different user
\dn             -- list schemas
\dt             -- list tables in the search_path
\dt schema.*    -- list tables in a specific schema
\d tablename    -- describe a table (columns, types, indexes, constraints, triggers)
\d+ tablename   -- describe with MORE detail (storage, description, stats target)
\di             -- list indexes
\dv             -- list views
\dm             -- list materialized views
\ds             -- list sequences
\df             -- list functions
\dx             -- list installed extensions
\du             -- list roles/users and their attributes
\dp  (\z)       -- list table access privileges (who can do what)
\sf func_name   -- show the source of a function
\timing         -- toggle showing how long each query took (turn this on while learning)
\x              -- toggle expanded (vertical) output — great for wide rows
\e              -- open the last query in your $EDITOR (edit a long query comfortably)
\i file.sql     -- run SQL from a file
\o out.txt      -- send query output to a file
\copy ...       -- client-side bulk import/export (see DML section §4)
\set            -- list/define psql variables
\conninfo       -- show details of the current connection (am I where I think I am?)
\q              -- quit
```

A few quality-of-life settings for `~/.psqlrc` (psql auto-loads this file on start), with comments on *why* each helps:

```text
\set QUIET 1                       -- suppress the noise printed while reading .psqlrc itself
\timing on                         -- always show query durations — builds performance intuition
\x auto                            -- auto-switch to vertical layout only when rows are too wide
\set COMP_KEYWORD_CASE upper       -- tab-completion produces UPPERCASE keywords (readable SQL)
\set HISTSIZE 5000                 -- remember more history across sessions
-- Make NULLs visible in output (default shows them as an empty string, indistinguishable from ''):
\pset null '(null)'
\set QUIET 0
```

### Creating databases, users, and roles — the role model

This is the single most important conceptual point about Postgres security: there is **no separate "user" object**. The unifying concept is the **role**. A role is just a named entity that can own objects and hold privileges. A *user* is simply a role that has the `LOGIN` attribute (so it can authenticate and open a connection); a *group* is simply a role you `GRANT` other roles *into* so they inherit its privileges. `CREATE USER` is literally an alias for `CREATE ROLE ... WITH LOGIN`. Internalizing this makes the security chapter (§19) click: you design a *hierarchy* of roles — broad group roles holding privilege sets, narrow login roles that are members of those groups — and you grant privileges to the groups.

```sql
-- Create a login role (a "user") with a password:
CREATE ROLE app_user WITH LOGIN PASSWORD 'changeme';

-- Equivalent shorthand (CREATE USER == CREATE ROLE ... WITH LOGIN):
CREATE USER report_user WITH PASSWORD 'changeme';

-- A role with the power to create databases and other roles, but NOT a full superuser
-- (least privilege: give CREATEDB/CREATEROLE rather than SUPERUSER whenever possible):
CREATE ROLE devops WITH LOGIN PASSWORD 'x' CREATEDB CREATEROLE;

-- A group role (no login) you grant membership into for shared privileges:
CREATE ROLE readonly;                 -- no LOGIN attribute = cannot connect directly
GRANT readonly TO report_user;        -- report_user now INHERITS readonly's privileges

-- Create a database owned by a specific role:
CREATE DATABASE shop
  OWNER app_user
  ENCODING 'UTF8'                      -- always UTF8 for modern apps
  LC_COLLATE 'en_US.UTF-8'             -- text sort order (locale-dependent)
  LC_CTYPE  'en_US.UTF-8'              -- character classification (what's a letter, digit…)
  TEMPLATE template0;                  -- template0 lets you choose a different locale/encoding

-- Alter / drop:
ALTER ROLE app_user WITH PASSWORD 'newsecret';
ALTER ROLE app_user VALID UNTIL '2027-01-01';   -- password expiry (forces rotation)
ALTER ROLE app_user CONNECTION LIMIT 20;        -- cap concurrent connections for this role
DROP ROLE report_user;
DROP DATABASE shop;                              -- must disconnect everyone first
DROP DATABASE IF EXISTS shop WITH (FORCE);       -- PG13+: terminate other sessions, then drop
```

Command-line equivalents (thin wrappers around the SQL above, convenient in scripts):

```bash
createdb shop                 # create a database
createuser --interactive app  # create a role, prompting for attributes
dropdb shop
dropuser app
```

> **Gotcha — two gates, not one:** Connecting requires passing **both** (1) a role with `LOGIN` and the correct password, *and* (2) a matching line in `pg_hba.conf` (Host-Based Authentication) that permits that user/database/source-IP/auth-method combination. The classic confusing symptom is "I can connect locally but not over TCP" — that's almost always `pg_hba.conf` not allowing the host, or `listen_addresses` in `postgresql.conf` still set to `localhost`. Authentication is covered fully in §19.

---

## 2. Data Types in Depth

**[Beginner]**

Choosing the right type is one of the most consequential decisions you make, and it's easy to underrate because a wrong type often "works" until it bites you. The type determines three things: **correctness** (can invalid data even be stored?), **storage size** (which affects how much fits in cache and how fast scans are), and **index behaviour** (some types index beautifully, others not at all). The guiding principle is **let the database enforce as much as possible** — a column typed `numeric(12,2) NOT NULL CHECK (total >= 0)` makes whole classes of bugs impossible, whereas a `text` column "containing a number" lets garbage in. This is defence in depth: even if application code has a bug, the type system is a second wall. For the *design* reasoning about which attributes deserve which types, **→ see RELATIONAL_DB_DESIGN_GUIDE.md**.

### Numeric types

The decision tree: use `integer` for ordinary whole numbers (IDs, counts), `bigint` when you might exceed ~2 billion (large IDs, byte counters, anything that grows forever), `numeric` for money and any value where exact decimal arithmetic matters, and the floating types *only* for measurements where tiny rounding error is acceptable (scientific data, percentages used for display). The reason money must never be a float is fundamental: binary floating point cannot represent `0.10` exactly, so sums drift — a payments system that loses a cent per thousand transactions is a real, expensive bug. `numeric` stores decimal digits exactly at the cost of being slower and larger.

| Type | Size | Range / notes |
|---|---|---|
| `smallint` (`int2`) | 2 bytes | -32,768 … 32,767 |
| `integer` (`int`, `int4`) | 4 bytes | ~±2.1 billion. The default choice for IDs/counts. |
| `bigint` (`int8`) | 8 bytes | ~±9.2 quintillion. Use for large IDs/counters. |
| `numeric(p, s)` / `decimal` | variable | **Exact** decimal; `p` total digits, `s` digits after the point. Use for money. |
| `real` (`float4`) | 4 bytes | Inexact, ~6 significant digits. |
| `double precision` (`float8`) | 8 bytes | Inexact, ~15 significant digits. |
| `smallserial`/`serial`/`bigserial` | — | Auto-incrementing integer (legacy; prefer `GENERATED ... AS IDENTITY`, see §3). |

```sql
-- NEVER store money in float — rounding errors WILL accumulate and bite you:
CREATE TABLE invoice (
  total numeric(12, 2) NOT NULL   -- exact: up to 10 digits before the point, 2 after
);
INSERT INTO invoice (total) VALUES (0.1 + 0.2);  -- stored exactly as 0.30, NOT 0.30000000004

-- Casting and rounding:
SELECT 10.0 / 3;                  -- numeric division: 3.3333333333333333
SELECT (10.0 / 3)::numeric(10,2); -- 3.33  (the :: operator casts/converts a value)
SELECT round(2.5), round(3.5);    -- 3, 4  (round-half-away-from-zero for numeric)
```

> **Why `serial` is discouraged:** `serial` is not a real type — it's sugar that creates an `integer` plus a sequence and a default. It has historical ownership quirks (the sequence isn't cleanly tied to the column) and predates the SQL-standard `GENERATED ... AS IDENTITY`, which is cleaner and what you should use today (§3).

### Character / text types

A point that surprises people coming from other databases: **in PostgreSQL, `text` and `varchar` have identical performance**. There is no speed penalty for using unlimited `text`. Therefore the rule is: use `text` by default, and reach for `varchar(n)` *only* when you have a genuine business reason to enforce a maximum length (e.g. an external system rejects codes longer than 10 characters). `char(n)` is almost always a mistake — it blank-pads values to the fixed length, wasting space and causing comparison surprises.

| Type | Notes |
|---|---|
| `text` | Unlimited length. **Preferred** for almost everything. |
| `varchar(n)` | Variable length with a max of `n` characters. Raises an error if you exceed `n`. |
| `char(n)` | Fixed length, **blank-padded** to `n`. Rarely a good idea. |

```sql
-- In PostgreSQL there is NO performance penalty for `text` vs `varchar`.
-- Use varchar(n) only when you genuinely need to ENFORCE a max length.
CREATE TABLE person (
  name  text NOT NULL,
  code  varchar(10)        -- e.g. an external code with a hard 10-char limit
);

-- Useful string functions (Postgres has a huge library — these are the daily ones):
SELECT length('héllo');                 -- 5 (characters, not bytes — Unicode-aware)
SELECT octet_length('héllo');           -- 6 (bytes in UTF-8, because é is 2 bytes)
SELECT upper('abc'), lower('ABC'), initcap('the quick fox');
SELECT trim('  hi  '), ltrim('xxhi','x'), rtrim('hixx','x');
SELECT substring('postgresql' FROM 1 FOR 4);    -- 'post'
SELECT split_part('a,b,c', ',', 2);             -- 'b' (split on ',', take the 2nd piece)
SELECT 'Hello, ' || 'world';                    -- '||' concatenates -> 'Hello, world'
SELECT format('User %s has %s points', 'Ana', 42);  -- printf-style; %I quotes identifiers safely
SELECT 'abc' ILIKE 'A%';                         -- case-insensitive LIKE -> true
SELECT regexp_replace('a1b2c3', '\d', '#', 'g'); -- 'a#b#c#'  ('g' = replace all matches)
SELECT 'foo bar baz' ~ '\mbar\M';                -- POSIX regex word-boundary match -> true
```

> **Security note on `format()`:** when you must build dynamic SQL inside functions, use `format()` with `%I` (identifier) and `%L` (literal) placeholders — they quote correctly and prevent SQL injection. Never assemble SQL by gluing strings with `||` and untrusted input. The general injection rule (parameterize from app code) is in §21; inside the database, `format(... %I %L ...)` plus `quote_ident`/`quote_literal` are the equivalents.

> **Tip:** For case-insensitive *columns* (emails, usernames), see the `citext` extension in §20 — it makes equality and uniqueness case-insensitive automatically, which is cleaner than sprinkling `lower()` everywhere.

### Boolean

A genuine three-state type, which matters: a boolean column can be `TRUE`, `FALSE`, or `NULL` (unknown). This is Postgres's three-valued logic, and forgetting the third state is a common source of bugs (a `WHERE active = true` filter silently drops rows where `active IS NULL`). Postgres accepts many textual spellings on input for convenience.

```sql
-- 'true' inputs: TRUE, 't', 'true', 'y', 'yes', 'on', '1'
-- 'false' inputs: FALSE, 'f', 'false', 'n', 'no', 'off', '0'
CREATE TABLE flag_demo (active boolean NOT NULL DEFAULT true);
SELECT true AND false, true OR false, NOT true;   -- f, t, f
-- Three-valued logic: comparisons with NULL yield NULL (treated as "unknown"):
SELECT true AND NULL;   -- NULL  (not false!)
SELECT NULL IS NULL;    -- t     (use IS / IS NOT for NULL checks, never = NULL)
```

### Date / time — and timezone handling (read this carefully, it's the #1 source of date bugs)

The most important rule in this entire section: **prefer `timestamptz` for any moment in time.** Despite its name, `timestamptz` does *not* store a timezone — it stores an absolute UTC instant. The mechanics: on **input**, the value is interpreted using the session's `TimeZone` setting and converted to UTC for storage; on **output**, the stored UTC is converted back to the session's `TimeZone` for display. This means two clients in different zones both see the *correct local rendering of the same instant* — exactly what you want. By contrast, plain `timestamp` (without time zone) stores the literal digits with no zone awareness; it's a "wall clock photo" that means different absolute moments to different observers, which is almost never what you want for events like "order placed at."

| Type | Stores | Notes |
|---|---|---|
| `date` | calendar date | no time, no zone |
| `time` | time of day | no zone |
| `timestamp` (without time zone) | date+time | **no zone info** — a "wall clock" reading |
| `timestamptz` (with time zone) | date+time | stored as UTC; converted to/from session TZ |
| `interval` | a span of time | e.g. `'2 days 3 hours'` |

```sql
SHOW TimeZone;                       -- the session timezone (e.g. 'UTC' or 'America/New_York')
SET TimeZone = 'UTC';                -- set for this session (storing UTC end-to-end is simplest)

-- Demonstrate the difference between naive and aware:
SELECT
  TIMESTAMP        '2026-06-21 12:00:00' AS naive,    -- "12:00" with no zone meaning
  TIMESTAMPTZ      '2026-06-21 12:00:00-04' AS aware; -- absolute instant (16:00 UTC)

-- "now" functions (know which you need):
SELECT now();              -- timestamptz; TRANSACTION start time (stable within a txn)
SELECT clock_timestamp();  -- actual wall clock; changes even within a single transaction
SELECT current_date, current_time, current_timestamp;

-- Arithmetic with intervals:
SELECT now() + interval '7 days';
SELECT now() - timestamptz '2026-01-01';        -- an interval (elapsed time between instants)
SELECT age(timestamptz '2000-01-01');           -- symbolic age in years/months/days

-- Truncation and extraction (the backbone of time-series reporting):
SELECT date_trunc('month', now());              -- first instant of this month
SELECT extract(dow FROM now());                 -- day of week (0=Sun..6=Sat)
SELECT extract(epoch FROM now());               -- Unix seconds as a double
SELECT to_char(now(), 'YYYY-MM-DD HH24:MI');    -- formatted text for display

-- Convert between zones explicitly with AT TIME ZONE:
SELECT now() AT TIME ZONE 'America/New_York';   -- timestamptz -> local wall clock (timestamp)
SELECT (TIMESTAMP '2026-06-21 09:00') AT TIME ZONE 'America/New_York';  -- naive -> timestamptz
```

> **Gotcha — `AT TIME ZONE` is overloaded.** Applied to a `timestamptz` it returns a `timestamp` (the wall clock in that zone). Applied to a `timestamp` it returns a `timestamptz` (it *interprets* the naive value as being in that zone). Read it in plain English as two different operations: "render this instant *in* zone X" versus "this wall-clock *is in* zone X, give me the absolute instant." Mixing these up produces times that are off by your UTC offset.

### UUID

A UUID is a 128-bit globally-unique identifier. The reason to choose it over a sequential integer key: you can generate it **anywhere** (in the client, in another service, offline) without coordinating with the database, and it's **non-guessable** so it doesn't leak how many rows exist or let an attacker enumerate `/users/1`, `/users/2`. The trade-off is the dark side of random v4 UUIDs: because they're random, inserts land all over the B-tree index, causing fragmentation and worse cache behaviour than a sequential key. UUIDv7 fixes this (see the version note).

```sql
-- UUIDs are ideal when you need globally-unique, non-guessable, client-generatable keys.
CREATE TABLE account (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid()  -- gen_random_uuid built-in since PG13
);
SELECT gen_random_uuid();   -- a random v4 UUID
```

⚡ **Version note (PG 18):** PostgreSQL 18 adds a built-in **`uuidv7()`**. UUIDv7 embeds a timestamp prefix, so values are *time-sortable* — new rows insert near the "end" of the index like a sequence would, dramatically reducing B-tree fragmentation versus random v4 UUIDs. This gives you the distribution/non-coordination benefits of UUIDs with near-sequential insert locality. On PG17 you can use the `pg_uuidv7` extension or generate v7 in your application.

### JSON vs JSONB

Postgres can store JSON documents directly, which is what lets it serve as a "document database when you need one" without giving up relational power. There are two types, and the choice is almost always `jsonb`. The difference is *how* it's stored: `json` keeps the raw text verbatim (preserving whitespace, key order, and duplicate keys), while `jsonb` parses it into a binary tree (canonicalizing it — dropping whitespace, sorting keys, removing duplicates). The binary form is the whole point: it can be queried efficiently and, crucially, **indexed with GIN** (§9, §11). Use plain `json` only in the rare case where you must reproduce the exact original bytes (e.g. verifying a cryptographic signature over the payload).

| Type | Storage | Preserves order/whitespace/dupes? | Indexable? | Speed |
|---|---|---|---|---|
| `json` | exact text copy | yes | no (only as text) | fast write, slow read |
| `jsonb` | parsed binary | no (canonicalized, dupes dropped) | **yes (GIN)** | slower write, fast read/query |

```sql
-- Use jsonb 99% of the time. Full operators and indexing are covered in §9.
CREATE TABLE event (id bigserial PRIMARY KEY, payload jsonb NOT NULL);
INSERT INTO event (payload) VALUES ('{"type":"click","x":10,"x":20}');
SELECT payload FROM event;   -- {"x": 20, "type": "click"}  (dup key dropped, keys reordered)
```

### Arrays

PostgreSQL lets any type be an array — a genuine first-class array column, not a serialized blob. This is occasionally exactly right (a small, bounded list of tags on an article) and occasionally a normalization smell (if you find yourself querying "rows where the array contains X *and* joining on it," a separate child table — **→ see RELATIONAL_DB_DESIGN_GUIDE.md** on many-to-many — is usually cleaner). Use arrays for short, mostly-read-as-a-whole lists; reach for a junction table when the elements are entities in their own right.

```sql
-- Any type can be an array. Declared with a [] suffix.
CREATE TABLE article (
  id    bigserial PRIMARY KEY,
  title text NOT NULL,
  tags  text[]                  -- a 1-D array of text
);
INSERT INTO article (title, tags) VALUES
  ('Postgres tips', ARRAY['sql','db','postgres']),
  ('Hello',         '{greeting,intro}');           -- alternative array-literal syntax

SELECT title FROM article WHERE 'sql' = ANY(tags); -- ANY: true if any element equals 'sql'
SELECT title FROM article WHERE tags @> ARRAY['db','sql'];  -- '@>' contains ALL of these
SELECT tags[1] FROM article;                       -- arrays are 1-INDEXED (not 0)!
SELECT array_length(tags, 1) FROM article;         -- length of the 1st dimension
SELECT unnest(tags) FROM article WHERE id = 1;     -- expand an array into one row per element
SELECT array_agg(title) FROM article;              -- the inverse: collapse rows into an array
SELECT cardinality(tags) FROM article;             -- total element count across all dimensions
```

### Enums

An enum is a custom type with a fixed, *ordered* set of allowed text values. It's a good fit for a small, stable set of categories where you want type-level enforcement and meaningful ordering (a status that progresses `pending → paid → shipped`). The ordering is by *declaration order*, not alphabetical, which is exactly what you want for status-like data. The catch is rigidity: changing an enum's values is awkward (you can add, but reordering or removing requires recreating the type). So the rule of thumb: use an enum when the set is small and rarely changes; use a **lookup table with a foreign key** when the set grows or changes often, or when you need to attach extra attributes to each value (**→ see RELATIONAL_DB_DESIGN_GUIDE.md** on lookup tables).

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

> **Gotcha:** You cannot easily `DELETE` a value or change a value's *position*, and removing a value requires recreating the type and migrating dependent columns. If your set of categories changes frequently, a lookup table with a foreign key is far more flexible than an enum.

### Ranges

Range types represent a continuous span with a lower and upper bound — a powerful, underused feature. The killer application is **preventing overlaps** declaratively: combined with an exclusion constraint, the *database itself* guarantees that no two bookings for the same room can overlap in time, an invariant that is notoriously hard and race-prone to enforce in application code. Built-in ranges: `int4range`, `int8range`, `numrange`, `tsrange`, `tstzrange`, `daterange`.

```sql
-- Range types represent a span with bounds.
CREATE TABLE reservation (
  room   int NOT NULL,
  during tstzrange NOT NULL,
  -- EXCLUDE constraint: the DATABASE prevents overlapping bookings for the same room.
  -- Requires the btree_gist extension so '=' on int can participate in a GiST index.
  EXCLUDE USING gist (room WITH =, during WITH &&)   -- '&&' is the "overlaps" operator
);
-- Bound notation: '[)' = inclusive lower, exclusive upper (the usual half-open convention,
-- which makes adjacent ranges meet without overlapping):
INSERT INTO reservation VALUES (1, tstzrange('2026-06-21 09:00', '2026-06-21 10:00', '[)'));
SELECT int4range(1, 10) @> 5;          -- '@>' contains 5? -> true
SELECT numrange(1,5) && numrange(4,9); -- '&&' overlap? -> true
SELECT upper(int4range(1,10)), lower(int4range(1,10));  -- 10, 1 (the bounds)
```

### Network address types

Dedicated types for IP addresses and networks. The value over storing them as `text`: they *validate* on input (a malformed IP is rejected), they sort and compare correctly, and they support network operators like "is this address inside that subnet" — which a text column cannot do.

```sql
CREATE TABLE access_log (
  client inet,        -- IPv4 or IPv6 host (optionally with a subnet mask)
  net    cidr,        -- a network block; validates that host bits are zero
  mac    macaddr      -- a hardware MAC address
);
INSERT INTO access_log VALUES ('192.168.1.50', '192.168.1.0/24', '08:00:2b:01:02:03');
SELECT inet '192.168.1.50' << cidr '192.168.1.0/24';  -- '<<' is contained by network? -> true
SELECT host(inet '10.0.0.1/8'), masklen(inet '10.0.0.1/8');  -- '10.0.0.1', 8
```

### bytea (binary data)

`bytea` stores raw bytes — hashes, small thumbnails, encrypted blobs. The important design guidance: **don't store large files in the database.** Big blobs bloat backups, blow out the cache, and make the row store inefficient. The standard architecture is to put large objects in object storage (S3, etc.) and keep only a URL/key (and maybe a content hash) in the database.

```sql
-- bytea stores raw binary. For LARGE files, store them in object storage and keep only a URL here.
CREATE TABLE file_blob (id bigserial PRIMARY KEY, data bytea);
INSERT INTO file_blob (data) VALUES ('\xDEADBEEF');     -- hex input format
SELECT encode(data, 'hex'), encode(data, 'base64') FROM file_blob;
SELECT length(data) FROM file_blob;   -- byte length
```

### Generated columns

A generated column is computed from other columns in the same row, so you never write to it directly and it can never get out of sync. Use it to materialize a derived value you query or index frequently (a `subtotal = price * qty`, or a `tsvector` for full-text search, §10). On PG17 generated columns are `STORED` (physically written, costing disk and write time but free to read); PG18 adds `VIRTUAL` (computed on read).

```sql
-- A STORED generated column is computed from other columns and physically stored.
CREATE TABLE product (
  price    numeric(10,2) NOT NULL,
  qty      int NOT NULL,
  subtotal numeric(12,2) GENERATED ALWAYS AS (price * qty) STORED
);
INSERT INTO product (price, qty) VALUES (9.99, 3);
SELECT subtotal FROM product;   -- 29.97, computed automatically; writing to it is an error
```

⚡ **Version note (PG 18):** PG 18 adds **`VIRTUAL`** generated columns (computed on read, not stored), now the default for `GENERATED ALWAYS AS`. They save disk and write cost in exchange for compute on each read — a good trade for rarely-read derived values. On PG17 only `STORED` exists, and you must write `STORED` explicitly.

---

## 3. DDL — Tables, Constraints, Schemas, Sequences

**[Beginner]**

DDL (Data Definition Language) is the subset of SQL that defines *structure* — `CREATE`, `ALTER`, `DROP`. Where DML (§4) manipulates rows, DDL shapes the containers those rows live in. Two PostgreSQL facts make DDL especially pleasant and safe. First, **DDL is transactional**: you can wrap `CREATE`/`ALTER`/`DROP` in `BEGIN ... COMMIT` and roll the whole thing back if any step fails — a property MySQL lacks, and one that makes migrations dramatically safer (a half-applied migration can be cleanly undone). Second, constraints declared in DDL are enforced by the engine on *every* write, no matter which client or code path made it — the strongest place to encode your business invariants. The *art* of deciding which tables, columns, keys, and relationships to create is database design proper, covered in **→ see RELATIONAL_DB_DESIGN_GUIDE.md**; this section is the *mechanics* of expressing that design in PostgreSQL.

### CREATE TABLE with all the common constraints

A constraint is a rule the database guarantees can never be violated. Understanding each one — and *why* you'd reach for it — is foundational, because constraints are how you make illegal states unrepresentable:

- **`NOT NULL`** — the column must always have a value. Use it aggressively; a nullable column is a promise that "absent" is a meaningful state, and most columns don't need that ambiguity.
- **`UNIQUE`** — no two rows may share the value. Postgres enforces it with an automatically-created index (so it's also fast to look up). NULLs are considered distinct from each other by default (multiple NULLs allowed).
- **`DEFAULT`** — the value used when an `INSERT` omits the column. Great for `created_at DEFAULT now()` and status defaults.
- **`CHECK`** — an arbitrary boolean expression that must hold for every row. This is your general-purpose invariant enforcer (`total >= 0`, `status IN (...)`, `email LIKE '%@%'`).
- **`PRIMARY KEY`** — the canonical unique identifier of a row: `UNIQUE` + `NOT NULL`, plus it's the default join target for foreign keys. Every table should have one.
- **`FOREIGN KEY`** — enforces *referential integrity*: a value here must exist in the referenced table, so you can never have an order pointing at a customer that doesn't exist.

```sql
CREATE TABLE customer (
  -- IDENTITY column: the modern, SQL-standard replacement for serial.
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,

  -- NOT NULL: column must always have a value.
  email       text NOT NULL,

  -- UNIQUE: no two rows may share this value (NULLs are allowed & considered distinct).
  -- A UNIQUE constraint automatically creates a supporting index used for lookups too.
  CONSTRAINT customer_email_key UNIQUE (email),

  -- DEFAULT: value used when an INSERT omits the column.
  created_at  timestamptz NOT NULL DEFAULT now(),

  -- CHECK: a boolean expression that must hold for every row.
  age         int CHECK (age IS NULL OR age >= 0),
  status      text NOT NULL DEFAULT 'active'
              CHECK (status IN ('active', 'suspended', 'closed')),

  -- A NAMED CHECK constraint produces clearer error messages and is easier to drop later.
  CONSTRAINT email_has_at CHECK (email LIKE '%@%')
);

CREATE TABLE orders (
  id           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id  bigint NOT NULL,
  total        numeric(12,2) NOT NULL CHECK (total >= 0),
  placed_at    timestamptz NOT NULL DEFAULT now(),

  -- FOREIGN KEY: enforces referential integrity. The ON DELETE clause decides what
  -- happens to children when a parent is deleted — choose deliberately:
  --   ON DELETE CASCADE  : deleting a customer deletes their orders (good for owned data)
  --   ON DELETE RESTRICT : block deleting a customer that still has orders (protect history)
  --   ON DELETE SET NULL : set customer_id to NULL instead (orphan but keep the order)
  CONSTRAINT orders_customer_fk
    FOREIGN KEY (customer_id) REFERENCES customer(id) ON DELETE CASCADE
);
```

> **`GENERATED ALWAYS AS IDENTITY` vs `BY DEFAULT`:** `ALWAYS` forbids manually inserting a value into the column (you'd have to say `OVERRIDING SYSTEM VALUE`), which protects the surrogate key from accidental tampering and keeps the sequence authoritative. `BY DEFAULT` lets you supply your own value — handy during data migration, but it risks desyncing the underlying sequence so the next auto-value collides. Prefer `ALWAYS` for true surrogate keys; use `BY DEFAULT` only when importing existing IDs.

> **Always index your foreign keys — Postgres does NOT do it for you.** Postgres automatically indexes the *referenced* (parent) primary key, but *not* the *referencing* (child) FK column. Without an index on `orders.customer_id`, two things get slow: deleting a customer must scan the entire `orders` table to check for references, and any join from customer to orders is a sequential scan. This is the single most common avoidable performance bug in real schemas — see §11 and §22.

### Primary keys, composite keys, and natural vs surrogate

Every table needs a primary key — the value that uniquely identifies a row and that other tables reference. The recurring design decision is **surrogate vs natural**. A *surrogate* key is a meaningless generated number/UUID (`id bigint GENERATED ... IDENTITY`); a *natural* key is real-world data that happens to be unique (an ISBN, an email). Surrogate keys are usually preferred because real-world "unique" values turn out not to be (people change emails, "unique" external codes get reused), and a stable meaningless ID never needs to change. A **composite key** uses two or more columns together — the classic case is a junction table for a many-to-many relationship, where the pair `(student_id, course_id)` is the natural identity of an enrollment. The full surrogate-vs-natural and many-to-many discussion is in **→ see RELATIONAL_DB_DESIGN_GUIDE.md**.

```sql
-- Composite primary key (a junction / join table for a many-to-many relationship):
CREATE TABLE enrollment (
  student_id  bigint NOT NULL REFERENCES customer(id),
  course_id   bigint NOT NULL,
  enrolled_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (student_id, course_id)   -- the PAIR must be unique (one enrollment per student/course)
);
```

### ALTER TABLE — evolving your schema safely

Schemas change as applications grow; `ALTER TABLE` is how you evolve them. The critical thing to understand is **locking**: many alterations need an `ACCESS EXCLUSIVE` lock, which blocks *all* reads and writes for the duration. On a tiny table that's instantaneous; on a multi-million-row table a careless `ALTER` can cause an outage. Postgres has steadily reduced which operations need full rewrites (e.g. since PG11, adding a column with a constant default is fast), and offers patterns like `NOT VALID` then `VALIDATE` to add constraints with only a brief lock. Always think "how big is this table and what lock does this take?" before altering production.

```sql
ALTER TABLE customer ADD COLUMN phone text;                 -- add a nullable column (fast)
ALTER TABLE customer ADD COLUMN country text NOT NULL DEFAULT 'US';  -- PG11+ fast for constant defaults
ALTER TABLE customer DROP COLUMN phone;                     -- drop a column (fast; space reclaimed later)
ALTER TABLE customer RENAME COLUMN email TO email_address;  -- rename (metadata-only, fast)
ALTER TABLE customer ALTER COLUMN status SET DEFAULT 'active';
ALTER TABLE customer ALTER COLUMN status DROP DEFAULT;
ALTER TABLE customer ALTER COLUMN age SET NOT NULL;         -- adds NOT NULL (must scan to validate)
ALTER TABLE customer ALTER COLUMN age TYPE bigint;          -- change type (may REWRITE the whole table)

-- Add/drop constraints after the fact:
ALTER TABLE orders ADD CONSTRAINT total_positive CHECK (total > 0);
ALTER TABLE orders DROP CONSTRAINT total_positive;

-- The safe pattern for adding a FK/CHECK to a HUGE table: skip the initial full-table check,
-- then validate later under a weaker lock that doesn't block writes:
ALTER TABLE orders ADD CONSTRAINT orders_customer_fk
  FOREIGN KEY (customer_id) REFERENCES customer(id) NOT VALID;  -- doesn't check existing rows yet
ALTER TABLE orders VALIDATE CONSTRAINT orders_customer_fk;      -- validate now, weaker (SHARE UPDATE) lock

ALTER TABLE customer RENAME TO client;                     -- rename the table (metadata-only)
```

> **Gotcha (locking):** Operations that rewrite the table — changing a column type incompatibly, or (in older PG) adding a volatile default — hold `ACCESS EXCLUSIVE` for the entire rewrite, blocking everything. Use the `NOT VALID` → `VALIDATE` pattern for constraints, build indexes with `CONCURRENTLY` (§11), and schedule unavoidable rewrites for low-traffic windows. For zero-downtime migrations on large tables, tools and patterns exist (expand/contract migrations) — the key is to never assume an `ALTER` is free.

### DROP and TRUNCATE

`DROP TABLE` removes the table and its data entirely. `TRUNCATE` keeps the table but deletes *all* rows extremely fast — it doesn't scan and delete row-by-row, it just discards the data files. The trade-off that catches people: `TRUNCATE` cannot be filtered with a `WHERE`, and it does **not** fire per-row `DELETE` triggers (it fires statement-level truncate triggers instead), so it is *not* a drop-in replacement for `DELETE` when triggers or auditing matter.

```sql
DROP TABLE IF EXISTS enrollment;            -- remove a table (IF EXISTS = no error if absent)
DROP TABLE orders CASCADE;                  -- also drop dependents (views, FKs) that reference it
TRUNCATE orders;                            -- delete ALL rows fast; no per-row triggers, no WHERE
TRUNCATE orders RESTART IDENTITY CASCADE;   -- also reset the identity sequence & truncate referencing tables
```

### Schemas (namespaces)

A schema is a namespace *inside* a database — a folder for your tables, views, and functions. The default schema is `public`. Schemas serve three purposes: **organization** (group related objects, e.g. `auth.users`, `billing.invoices`), **access control** (grant privileges per schema), and **multi-tenancy** (a schema per tenant, though this scales only to moderate tenant counts). The `search_path` setting determines which schemas are consulted, and in what order, when you write an *unqualified* name like `invoice` instead of `billing.invoice`.

```sql
CREATE SCHEMA billing;
CREATE SCHEMA billing AUTHORIZATION app_user;   -- create AND set owner in one statement

CREATE TABLE billing.invoice (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY);

-- The search_path determines which schemas are searched for unqualified names:
SHOW search_path;                  -- default: "$user", public
SET search_path TO billing, public;
SELECT * FROM invoice;             -- resolves to billing.invoice (first match wins)

DROP SCHEMA billing CASCADE;       -- drop the schema and EVERYTHING in it (careful!)
```

> **Security note on `search_path`:** a misconfigured `search_path` is a real privilege-escalation vector — if a function runs with `SECURITY DEFINER` and an attacker can create a function/table earlier in the path, they can hijack calls. Best practice for `SECURITY DEFINER` functions: set an explicit `search_path` on the function (`SET search_path = pg_catalog, public`) so name resolution can't be tricked. More in §14 and §19.

### Sequences and identity columns

A sequence is a special object that hands out increasing numbers, atomically and concurrently safely — it's the machinery behind `IDENTITY` and `serial`. You'll occasionally use one directly (e.g. a human-facing ticket number with a custom start/increment). The one behaviour everyone must internalize: **sequences are not gap-free.** Because a sequence value is fetched outside transactional rollback (so that concurrent transactions never block each other waiting for the next number), a transaction that rolls back still "used up" the number it grabbed. IDs will therefore have gaps. They guarantee *uniqueness* and *monotonic increase*, never contiguity — never build logic that assumes "no gaps."

```sql
-- A sequence generates increasing numbers. IDENTITY columns use one under the hood.
CREATE SEQUENCE order_no_seq START 1000 INCREMENT 1;
SELECT nextval('order_no_seq');    -- 1000, and advances the sequence (this is the only "consume" op)
SELECT currval('order_no_seq');    -- 1000 (last value THIS session fetched; errors if none yet)
SELECT setval('order_no_seq', 5000);  -- reset the counter (admin operation)

-- Attach a sequence as a column default manually:
CREATE TABLE ticket (
  number bigint NOT NULL DEFAULT nextval('order_no_seq')
);

-- Inspect/fix an IDENTITY column's underlying sequence (e.g. after a bulk load with explicit IDs):
SELECT pg_get_serial_sequence('orders', 'id');   -- discover the backing sequence name
SELECT setval(pg_get_serial_sequence('orders','id'),
              (SELECT max(id) FROM orders));      -- resync so the next auto-id won't collide
```

> **Gotcha:** Sequences are **not** transactional for gap-freeness — a rolled-back transaction still consumes the numbers it fetched, and concurrent inserts interleave. Never assume sequence values are contiguous or that "the latest id" equals "the row count." If you need a truly gap-free human-facing number (invoice numbering for tax/legal reasons), generate it separately with explicit locking and accept the reduced concurrency.

---

## 4. DML & Queries — INSERT, UPDATE, DELETE, SELECT

**[Beginner]**

DML (Data Manipulation Language) is the part of SQL you use most: putting rows in (`INSERT`), changing them (`UPDATE`), removing them (`DELETE`), and reading them (`SELECT`). The mental shift from imperative programming is that SQL is **declarative and set-based** — you describe the set of rows you want and the operation to apply, and the engine processes them all at once rather than looping. "Update every Engineering salary by 10%" is one statement, not a loop. Internalizing the set-based mindset is what separates SQL that scales from SQL that issues a thousand round-trips.

Let's build a small dataset to query throughout the rest of the guide.

```sql
CREATE TABLE employee (
  id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name       text NOT NULL,
  department text NOT NULL,
  salary     numeric(10,2) NOT NULL,
  manager_id bigint REFERENCES employee(id),   -- self-reference models an org chart
  hired_at   date NOT NULL DEFAULT current_date
);
```

### INSERT — single, multi-row, RETURNING, upsert

`INSERT` adds rows. Three things to learn beyond the basics. First, **multi-row inserts** (many value tuples in one statement) are far faster than many separate `INSERT`s because each statement has fixed overhead (parsing, planning, a network round-trip) — batching amortizes it. Second, **`RETURNING`** is a PostgreSQL superpower: it lets the same statement hand back generated values (the new `id`, a `DEFAULT`-computed `created_at`) so you don't need a second query to find out what you just inserted. Third, **`INSERT ... SELECT`** lets you populate a table from a query, transforming data in flight.

```sql
-- Single row:
INSERT INTO employee (name, department, salary)
VALUES ('Ada', 'Engineering', 120000);

-- Multi-row (one statement, far faster than many separate INSERTs):
INSERT INTO employee (name, department, salary, manager_id) VALUES
  ('Grace',     'Engineering', 110000, 1),
  ('Linus',     'Engineering',  95000, 1),
  ('Margaret',  'Design',       90000, NULL),
  ('Katherine', 'Design',       88000, 4);

-- RETURNING: get back generated/computed values without a second round-trip:
INSERT INTO employee (name, department, salary)
VALUES ('Alan', 'Research', 130000)
RETURNING id, name, hired_at;

-- INSERT ... SELECT (copy/transform rows from another query):
INSERT INTO employee (name, department, salary)
SELECT name || ' (copy)', department, salary FROM employee WHERE department = 'Design';
```

### UPSERT with ON CONFLICT

"Upsert" = insert if new, otherwise update. You need it constantly — counters, caches, idempotent imports where the same row might arrive twice. The mechanism is `ON CONFLICT`, and the key requirement is that there must be a `UNIQUE` or `PRIMARY KEY` constraint for Postgres to *detect* the conflict on. The special table `EXCLUDED` refers to the row you *tried* to insert, so you can reference the new values inside the update branch. The two common shapes are `DO UPDATE` (merge in the new data) and `DO NOTHING` (ignore duplicates silently — perfect for idempotent inserts).

```sql
-- Requires a UNIQUE or PRIMARY KEY constraint to detect the conflict:
CREATE TABLE inventory (
  sku        text PRIMARY KEY,
  qty        int NOT NULL,
  updated_at timestamptz NOT NULL DEFAULT now()
);

-- "Insert, or if the sku already exists, ADD to the existing quantity":
INSERT INTO inventory (sku, qty) VALUES ('ABC-1', 10)
ON CONFLICT (sku) DO UPDATE
  SET qty = inventory.qty + EXCLUDED.qty,   -- EXCLUDED = the row we attempted to insert
      updated_at = now();

-- "Insert, but do nothing if it already exists" (idempotent insert):
INSERT INTO inventory (sku, qty) VALUES ('ABC-1', 5)
ON CONFLICT (sku) DO NOTHING;

-- Conditional upsert: only apply the update when a WHERE on the existing row holds:
INSERT INTO inventory (sku, qty) VALUES ('ABC-1', 3)
ON CONFLICT (sku) DO UPDATE SET qty = EXCLUDED.qty
WHERE inventory.qty < EXCLUDED.qty;   -- only overwrite if the new qty is larger
```

⚡ **Version note:** PG15+ has `MERGE`, the SQL-standard multi-action upsert; **PG17** adds `MERGE ... RETURNING` and a `WHEN NOT MATCHED BY SOURCE` clause. Use `ON CONFLICT` for plain upserts (it's simpler and handles concurrency cleanly); reach for `MERGE` when you need INSERT *and* UPDATE *and* DELETE branches in a single statement (e.g. reconciling a target table against a source feed).

```sql
-- MERGE example (PG15+): reconcile inventory against an incoming batch.
MERGE INTO inventory AS t
USING (VALUES ('ABC-1', 7)) AS s(sku, qty) ON t.sku = s.sku
WHEN MATCHED THEN UPDATE SET qty = t.qty + s.qty
WHEN NOT MATCHED THEN INSERT (sku, qty) VALUES (s.sku, s.qty)
RETURNING merge_action(), t.*;   -- PG17: merge_action() returns 'INSERT'/'UPDATE'/'DELETE'
```

### UPDATE & DELETE

`UPDATE` and `DELETE` change/remove rows matching a `WHERE`. The number-one safety rule: **never run an `UPDATE` or `DELETE` without a `WHERE`** unless you genuinely mean "every row." A common safe habit is to write the `WHERE` first, run it as a `SELECT` to confirm the affected rows, then convert it to the mutation. Postgres extends both with a `FROM`/`USING` clause so you can drive the change from a join to another table, and (like `INSERT`) both support `RETURNING` so you can see exactly what changed.

```sql
-- UPDATE with WHERE (forgetting the WHERE updates the ENTIRE table — a classic disaster):
UPDATE employee SET salary = salary * 1.10 WHERE department = 'Engineering';

-- UPDATE ... FROM: update using a join/subquery against other data:
UPDATE employee e
SET salary = salary + 5000
FROM (SELECT department FROM employee GROUP BY department HAVING avg(salary) < 100000) low
WHERE e.department = low.department
RETURNING e.id, e.name, e.salary;   -- RETURNING works on UPDATE/DELETE too

-- DELETE:
DELETE FROM employee WHERE department = 'Research' RETURNING *;
DELETE FROM employee USING orders WHERE employee.id = orders.customer_id;  -- DELETE ... USING join
```

> **Safety habit:** in production, set a `statement_timeout` and consider wrapping risky bulk mutations in an explicit transaction so you can `ROLLBACK` if the row count looks wrong (`BEGIN; DELETE ...; -- check, then COMMIT or ROLLBACK`). DDL and DML are both transactional in Postgres, so this is cheap insurance.

### SELECT — the core query

`SELECT` is where you'll spend most of your SQL life. Beyond column selection and `WHERE` filtering, the pieces you must know cold are: `ORDER BY` (with explicit `NULLS FIRST/LAST` because NULL ordering surprises people), `LIMIT`/`OFFSET` (pagination — but see the keyset-pagination warning in §22, because deep `OFFSET` is slow), `DISTINCT` and the PostgreSQL-specific `DISTINCT ON`, and the conditional expressions `CASE`, `COALESCE`, and `NULLIF`. A recurring NULL rule appears here too: comparisons against NULL must use `IS NULL` / `IS NOT NULL`, never `= NULL` (which yields NULL/unknown, matching nothing).

```sql
-- WHERE: filter rows. Operators: = <> < > <= >= BETWEEN IN LIKE ILIKE IS NULL
SELECT name, salary FROM employee
WHERE department = 'Engineering'
  AND salary BETWEEN 90000 AND 130000
  AND name ILIKE 'a%'              -- case-insensitive prefix match
  AND manager_id IS NOT NULL;      -- NULL checks MUST use IS / IS NOT, never = NULL

-- IN and NOT IN:
SELECT * FROM employee WHERE department IN ('Design', 'Engineering');

-- ORDER BY (ASC is the default); NULLS FIRST/LAST controls where NULLs land:
SELECT name, salary FROM employee
ORDER BY salary DESC, name ASC NULLS LAST;

-- LIMIT / OFFSET — pagination (OFFSET is slow for deep pages; see keyset pagination in §22):
SELECT * FROM employee ORDER BY id LIMIT 10 OFFSET 20;   -- rows 21..30
-- SQL-standard equivalent:
SELECT * FROM employee ORDER BY id FETCH FIRST 10 ROWS ONLY;

-- DISTINCT and DISTINCT ON:
SELECT DISTINCT department FROM employee;          -- unique departments
-- DISTINCT ON (a Postgres extension): keep ONE row per department, the highest-paid one:
SELECT DISTINCT ON (department) department, name, salary
FROM employee
ORDER BY department, salary DESC;   -- ORDER BY MUST start with the DISTINCT ON column(s)

-- Column & table aliases, and computed columns:
SELECT e.name AS employee, e.salary / 12 AS monthly_pay
FROM employee AS e
WHERE e.salary > 100000;

-- CASE expression (inline conditional, like if/else in a query):
SELECT name,
  CASE
    WHEN salary >= 120000 THEN 'senior'
    WHEN salary >= 95000  THEN 'mid'
    ELSE 'junior'
  END AS band
FROM employee;

-- COALESCE (first non-NULL argument) and NULLIF (NULL if the two args are equal):
SELECT name, COALESCE(manager_id::text, 'no manager') AS mgr FROM employee;
SELECT NULLIF(salary, 0) FROM employee;   -- returns NULL when salary = 0 (avoid divide-by-zero etc.)
```

### Bulk import/export with COPY

For loading or extracting large amounts of data, `COPY` is the right tool — it streams rows in a tight loop and is typically **10–100× faster** than row-by-row `INSERT`. There are two flavours and the difference matters for permissions: server-side `COPY` reads/writes files on the *database server's* filesystem and therefore needs elevated privileges (superuser or the `pg_read_server_files`/`pg_write_server_files` roles), while psql's client-side `\copy` reads/writes files on *your* machine and needs no special database privilege. On your own laptop you'll usually use `\copy`.

```sql
-- Server-side COPY (file lives on the SERVER; needs superuser or pg_read_server_files):
COPY employee (name, department, salary) FROM '/data/emp.csv' WITH (FORMAT csv, HEADER true);
COPY (SELECT * FROM employee) TO '/data/out.csv' WITH (FORMAT csv, HEADER true);
```

```text
-- Client-side \copy in psql (file lives on YOUR machine; no special DB perms needed):
\copy employee(name,department,salary) FROM 'emp.csv' WITH (FORMAT csv, HEADER true)
\copy (SELECT * FROM employee) TO 'out.csv' WITH (FORMAT csv, HEADER true)
```

> **Tip:** For programmatic bulk loads from application code, drivers expose the COPY protocol directly — `pgx`'s `CopyFrom` in Go and the `pg-copy-streams` package in Node — which is dramatically faster than looping inserts (§21). After a big load, run `ANALYZE` (§12) so the planner has fresh statistics.

---

## 5. Joins & Set Operations

**[Intermediate]**

Joins are how relational databases earn their name: data is deliberately split across tables (normalization — **→ see RELATIONAL_DB_DESIGN_GUIDE.md**) to avoid duplication, and joins recombine it at query time. A join pairs rows from two tables based on a matching condition (usually a foreign-key relationship). The *type* of join determines what happens to rows that have **no** match on the other side — and getting that right is the crux of most join bugs. Picture two overlapping circles (a Venn diagram): an **inner** join returns only the overlap; a **left** join returns the whole left circle (filling NULLs where the right is missing); a **full** join returns both circles entirely. Choosing the wrong one silently drops or duplicates rows, so always ask "what should happen to unmatched rows?" before writing the join.

```sql
-- Sample data we'll reuse:
-- employee(id, name, department, salary, manager_id, hired_at)
-- orders(id, customer_id, total, placed_at)  -- from §3

-- INNER JOIN: only rows that have a match in BOTH tables (the overlap).
SELECT e.name, m.name AS manager
FROM employee e
INNER JOIN employee m ON e.manager_id = m.id;   -- a SELF-join: the table joined to itself

-- LEFT (OUTER) JOIN: ALL rows from the LEFT table; NULLs where there's no right-side match.
SELECT e.name, m.name AS manager
FROM employee e
LEFT JOIN employee m ON e.manager_id = m.id;    -- includes employees who have no manager

-- Find rows with NO match (an ANTI-JOIN) — the classic "orphans" / "missing" query:
SELECT e.name
FROM employee e
LEFT JOIN employee m ON e.manager_id = m.id
WHERE m.id IS NULL;        -- keep only the left rows that found nothing on the right

-- RIGHT JOIN: ALL rows from the RIGHT table (a mirror of LEFT — usually just flip the tables instead).
SELECT e.name, m.name AS manager
FROM employee m
RIGHT JOIN employee e ON e.manager_id = m.id;

-- FULL OUTER JOIN: all rows from BOTH sides; NULLs wherever either side lacks a match.
SELECT COALESCE(a.department, b.department) AS dept
FROM employee a
FULL JOIN employee b ON a.department = b.department;

-- CROSS JOIN: the Cartesian product (every left row paired with every right row).
SELECT e.name, d.day
FROM employee e
CROSS JOIN (VALUES ('Mon'), ('Tue')) AS d(day);   -- duplicates each employee once per day

-- USING shorthand (when the join columns share the SAME name — collapses them into one output column):
SELECT * FROM orders JOIN customer USING (id);    -- equivalent to ON orders.id = customer.id

-- Multiple joins chain naturally:
SELECT c.name AS customer, o.id AS order_id, o.total
FROM customer c
JOIN orders o ON o.customer_id = c.id
WHERE o.total > 100;
```

> **Gotcha — the most common join bug:** putting a filter on the *right* table of a `LEFT JOIN` in the `WHERE` clause silently turns it into an inner join (because `WHERE right.col = x` excludes the NULL-filled unmatched rows). If you want to filter the right side *while preserving* unmatched left rows, put the condition in the `ON` clause instead: `LEFT JOIN orders o ON o.customer_id = c.id AND o.total > 100`.

### LATERAL joins — "join to a subquery that references the left row"

`LATERAL` removes a fundamental restriction: normally a subquery in the `FROM` clause is self-contained and can't see the other tables in the same `FROM`. `LATERAL` lets it reference columns from the tables listed *before* it — so the subquery runs once *per left row*, parameterized by that row. The canonical use is **top-N per group** ("the 2 highest-paid employees in *each* department"), which is awkward to express any other way. Think of it as a `for` loop: "for each department row, run this correlated subquery."

```sql
-- For each department, get its 2 highest-paid employees:
SELECT d.department, top.name, top.salary
FROM (SELECT DISTINCT department FROM employee) d
CROSS JOIN LATERAL (
  SELECT name, salary
  FROM employee e
  WHERE e.department = d.department    -- references the OUTER 'd' — only legal because of LATERAL
  ORDER BY salary DESC
  LIMIT 2
) AS top;
```

### Set operations

Where joins combine tables *side by side* (adding columns), set operations stack query results *on top of each other* (combining rows), like the mathematical set operators. The three are `UNION` (everything in either, **deduplicated**), `INTERSECT` (only rows in both), and `EXCEPT` (rows in the first but not the second). Two rules govern all of them: the queries must have the same number of columns with compatible types, and **columns are matched by position, not by name**. The performance note worth memorizing: `UNION` does a deduplication pass (a sort or hash), so if you know there are no duplicates — or you actually want them — use `UNION ALL`, which is markedly cheaper.

```sql
-- UNION: combine result sets and REMOVE duplicates. UNION ALL keeps duplicates (and is faster).
SELECT name FROM employee WHERE department = 'Engineering'
UNION
SELECT name FROM employee WHERE salary > 100000;

SELECT department FROM employee
UNION ALL                       -- keep dups; use when you know there are none, or you want counts
SELECT department FROM orders_archive;   -- columns matched by POSITION; types must be compatible

-- INTERSECT: rows present in BOTH queries.
SELECT name FROM employee WHERE department = 'Engineering'
INTERSECT
SELECT name FROM employee WHERE salary > 100000;

-- EXCEPT: rows in the first query but NOT the second (set difference).
SELECT name FROM employee
EXCEPT
SELECT name FROM employee WHERE department = 'Design';
```

> **Gotcha:** `UNION` performs a deduplication sort/hash that you pay for even if there are no duplicates — reach for `UNION ALL` unless you specifically need duplicate removal. And remember set operations align columns **by position**: if the two `SELECT`s list columns in a different order, you'll silently mix data.

---

## 6. Aggregation — GROUP BY, HAVING, GROUPING SETS

**[Intermediate]**

Aggregation collapses many rows into summary values — counts, sums, averages, min/max. The model has two stages and getting them straight prevents most aggregation confusion: **`WHERE` filters individual rows *before* grouping**, then **`GROUP BY` collapses the survivors into groups**, then **`HAVING` filters the *groups* by their aggregate values**. So "departments hired since 2020 whose average salary exceeds 95k" is `WHERE hired_at >= '2020'` (per-row) + `GROUP BY department` + `HAVING avg(salary) > 95000` (per-group). A second rule the engine enforces: every column in the `SELECT` list must either be in the `GROUP BY` or be wrapped in an aggregate function — because for a non-grouped column there'd be many values per group and no way to pick one. Aggregates also silently **skip NULLs** (`avg`, `sum`, `count(col)` all ignore them; only `count(*)` counts every row), which is usually what you want but occasionally surprising.

```sql
-- Core aggregate functions in one query:
SELECT
  count(*)                   AS num_rows,       -- counts ALL rows (including those with NULLs)
  count(manager_id)          AS num_with_mgr,   -- counts only NON-NULL manager_id values
  count(DISTINCT department) AS num_depts,
  sum(salary)                AS payroll,
  avg(salary)                AS avg_salary,
  min(salary), max(salary),
  string_agg(name, ', ' ORDER BY name) AS everyone,        -- concatenate text across the group
  array_agg(salary ORDER BY salary DESC) AS salaries       -- collect values into an array
FROM employee;

-- GROUP BY: one output row per distinct group.
SELECT department, count(*) AS headcount, round(avg(salary), 2) AS avg_salary
FROM employee
GROUP BY department
ORDER BY avg_salary DESC;

-- HAVING vs WHERE — the two-stage filter:
SELECT department, avg(salary) AS avg_salary
FROM employee
WHERE hired_at >= '2020-01-01'   -- WHERE: filters INPUT rows (before grouping)
GROUP BY department
HAVING avg(salary) > 95000;      -- HAVING: filters GROUPS by their aggregate (after grouping)
```

### FILTER — conditional aggregation

`FILTER` restricts which rows a *single* aggregate sees, letting you compute several differently-scoped aggregates in one pass over the data. It's cleaner and faster than the old trick of putting `CASE` expressions inside aggregates, and it reads naturally. Use it for "pivot-style" summaries — totals, plus sub-counts by category, side by side.

```sql
-- FILTER scopes each aggregate independently — one scan, many conditional measures:
SELECT
  count(*)                                      AS total,
  count(*) FILTER (WHERE salary > 100000)       AS high_earners,
  count(*) FILTER (WHERE department = 'Design') AS designers,
  avg(salary) FILTER (WHERE department = 'Engineering') AS eng_avg
FROM employee;
```

### GROUPING SETS, ROLLUP, CUBE — multi-level subtotals

These extensions produce multiple grouping levels in a single query — exactly what reports and dashboards need (per-department totals *and* a grand total, in one result set, without `UNION`ing several queries). `ROLLUP` gives hierarchical subtotals (think a drill-down: by (dept, manager), then by dept, then overall). `CUBE` gives *every* combination of the listed columns (a full cross-tab). `GROUPING SETS` lets you list exactly the groupings you want. In all of them, subtotal/grand-total rows appear with `NULL` in the rolled-up columns — and the `GROUPING()` function tells you whether a `NULL` is a genuine value or a subtotal placeholder, so you can label them.

```sql
-- ROLLUP: hierarchical subtotals + grand total in one query.
-- Produces rows for: each (department, manager_id), each department, and the overall total.
SELECT department, manager_id, count(*) AS n, sum(salary) AS payroll
FROM employee
GROUP BY ROLLUP (department, manager_id)
ORDER BY department NULLS LAST, manager_id NULLS LAST;
-- A NULL in a grouped column marks a subtotal / grand-total line for that level.

-- CUBE: subtotals for EVERY combination of the listed columns (full cross-tabulation).
SELECT department, manager_id, sum(salary)
FROM employee
GROUP BY CUBE (department, manager_id);

-- GROUPING SETS: explicitly list exactly the groupings you want.
SELECT department, manager_id, count(*)
FROM employee
GROUP BY GROUPING SETS ((department), (manager_id), ());  -- () = the grand total

-- GROUPING() distinguishes a real NULL value from a subtotal placeholder (1 = placeholder):
SELECT
  CASE WHEN grouping(department) = 1 THEN 'ALL DEPARTMENTS' ELSE department END AS dept,
  sum(salary)
FROM employee
GROUP BY ROLLUP (department);
```

---

## 7. Subqueries & CTEs (incl. Recursive)

**[Intermediate]**

A subquery is a query nested inside another. They come in flavours worth naming because each behaves differently: a **scalar** subquery returns a single value and can sit anywhere a value is expected; an **`IN`/`ANY`** subquery returns a column to test membership against; a **correlated** subquery references the outer row and therefore conceptually re-runs for each outer row; and an **`EXISTS`** subquery just tests whether any matching row exists. The big practical guidance is to prefer `EXISTS`/`NOT EXISTS` over `IN`/`NOT IN` when checking presence — `EXISTS` is NULL-safe and the planner handles it well, whereas `NOT IN` has a notorious NULL trap (below).

### Subqueries

```sql
-- Scalar subquery (returns exactly one value, usable inline):
SELECT name, salary,
  salary - (SELECT avg(salary) FROM employee) AS diff_from_avg
FROM employee;

-- IN subquery (membership test against a returned column):
SELECT name FROM employee
WHERE department IN (SELECT department FROM employee GROUP BY department HAVING count(*) > 2);

-- Correlated subquery (references the outer row; conceptually runs per outer row):
SELECT e.name, e.salary
FROM employee e
WHERE e.salary > (SELECT avg(salary) FROM employee x WHERE x.department = e.department);

-- EXISTS / NOT EXISTS (often faster and ALWAYS NULL-safe vs IN/NOT IN):
SELECT c.name FROM customer c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);      -- customers WITH orders
SELECT c.name FROM customer c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);  -- customers with NO orders
```

> **Gotcha — the `NOT IN` NULL trap:** `NOT IN (subquery)` returns *no rows at all* if the subquery yields even a single `NULL`. The reason is three-valued logic: `x NOT IN (1, NULL)` becomes `x <> 1 AND x <> NULL`, and `x <> NULL` is `NULL` (unknown), which is never true. This bug is silent and data-dependent (it appears only once a NULL sneaks into the list). Always prefer `NOT EXISTS`, which has no such surprise.

### Common Table Expressions (CTEs / WITH)

A CTE is a named subquery defined with `WITH` that you reference like a table in the main query. The primary value is **readability**: you can decompose a gnarly query into named, logically-ordered steps that read top-to-bottom, instead of deeply nested subqueries. You can chain multiple CTEs (each can reference the previous), and — a uniquely powerful Postgres feature — a **data-modifying CTE** lets you `INSERT`/`UPDATE`/`DELETE` with `RETURNING` inside a `WITH` and feed the affected rows into a following statement, all atomically (e.g. "delete old orders and insert them into an archive" in one statement).

```sql
-- A CTE names a subquery, improving readability and enabling reuse within the statement.
WITH dept_stats AS (
  SELECT department, avg(salary) AS avg_sal, count(*) AS n
  FROM employee
  GROUP BY department
)
SELECT e.name, e.salary, ds.avg_sal
FROM employee e
JOIN dept_stats ds ON ds.department = e.department
WHERE e.salary > ds.avg_sal;     -- above-average earners within their own department

-- Multiple CTEs, chained (each step builds on the prior):
WITH
  recent AS (SELECT * FROM orders WHERE placed_at > now() - interval '30 days'),
  big    AS (SELECT * FROM recent WHERE total > 500)
SELECT customer_id, count(*) FROM big GROUP BY customer_id;

-- Data-modifying CTE: archive-then-delete ATOMICALLY in one statement:
WITH moved AS (
  DELETE FROM orders WHERE placed_at < '2020-01-01' RETURNING *
)
INSERT INTO orders_archive SELECT * FROM moved;
```

⚡ **Version note:** Since PG12, CTEs are **inlined** by default — the optimizer can fold a CTE into the main query and optimize across the boundary — *unless* the CTE is recursive, referenced more than once, or marked `MATERIALIZED`. Before PG12 every CTE was an "optimization fence" (always computed separately), which people sometimes relied on. Use `WITH x AS MATERIALIZED (...)` to force the fence (handy to compute an expensive CTE exactly once), or `NOT MATERIALIZED` to force inlining.

### Recursive CTEs — tree & graph traversal

This is the tool for hierarchical and graph-shaped data: org charts, category trees, bills of materials, "all descendants/ancestors," shortest reachability. A recursive CTE has two parts joined by `UNION [ALL]`: the **anchor** (the starting rows — e.g. top-level managers) and the **recursive term** (which joins the table back to the rows produced so far, expanding the frontier each iteration until no new rows appear). The mental model is breadth-first expansion: start with the seed set, repeatedly find the next level, accumulate. For graphs that may contain cycles you must guard against infinite loops — either track a visited-path array yourself, or use the `CYCLE` clause (PG14+).

```sql
-- Walk the org chart from the top down, tracking depth and a readable path.
WITH RECURSIVE org AS (
  -- ANCHOR: the starting rows (top-level managers, who have no boss).
  SELECT id, name, manager_id, 1 AS depth, name::text AS path
  FROM employee
  WHERE manager_id IS NULL

  UNION ALL

  -- RECURSIVE term: join the table back to the rows produced so far ('org').
  SELECT e.id, e.name, e.manager_id, o.depth + 1, o.path || ' > ' || e.name
  FROM employee e
  JOIN org o ON e.manager_id = o.id     -- 'org' here = everything produced in prior iterations
)
SELECT repeat('  ', depth - 1) || name AS tree, depth, path
FROM org
ORDER BY path;

-- A numeric series via recursion (generate_series is usually better, but this shows the pattern):
WITH RECURSIVE nums(n) AS (
  SELECT 1
  UNION ALL
  SELECT n + 1 FROM nums WHERE n < 10    -- the WHERE is the termination condition
)
SELECT n FROM nums;   -- 1..10

-- Graph traversal with explicit cycle protection (PG14+ also offers a CYCLE clause):
WITH RECURSIVE reachable AS (
  SELECT id, name, ARRAY[id] AS visited FROM employee WHERE id = 1
  UNION ALL
  SELECT e.id, e.name, r.visited || e.id
  FROM employee e
  JOIN reachable r ON e.manager_id = r.id
  WHERE e.id <> ALL(r.visited)   -- don't revisit nodes -> prevents infinite loops on cycles
)
SELECT * FROM reachable;
```

> **Gotcha:** Use `UNION ALL` (not `UNION`) in recursive CTEs unless you specifically need per-iteration deduplication — `UNION` adds a dedup pass each round. And always ensure a termination condition (a `WHERE` that eventually stops producing rows, or cycle protection on graphs), or the query runs forever.

---

## 8. Window Functions in Depth

**[Intermediate]**

Window functions are the feature that, once it clicks, replaces dozens of awkward self-joins and subqueries. The defining difference from `GROUP BY`: aggregation *collapses* rows into one per group, whereas a window function computes a value across a set of related rows **while keeping every original row**. Each output row "looks out a window" at its neighbours and reports something about them — its rank within its group, the running total up to it, the previous row's value, its share of the group total. You get the detail rows *and* the cross-row computation in one pass.

Every window function is followed by an `OVER (...)` clause with up to three parts, and understanding them is the whole game:

- **`PARTITION BY`** divides the rows into independent groups; the function restarts for each partition (like `GROUP BY` but without collapsing). Omit it and the whole result is one partition.
- **`ORDER BY`** (inside `OVER`) orders the rows *within* each partition — essential for anything sequential (ranking, running totals, `LAG`/`LEAD`).
- **The frame clause** (`ROWS`/`RANGE`/`GROUPS BETWEEN ... AND ...`) defines, *for the current row*, which subset of the partition the function sees. This is the subtle part that trips people up (see below).

### Ranking

Ranking functions assign positions within each partition. The three you must distinguish: `row_number()` gives a strict 1,2,3 with no ties (arbitrary tiebreak); `rank()` lets ties share a number then *skips* (1,2,2,4); `dense_rank()` lets ties share but does *not* skip (1,2,2,3). `ntile(n)` slices the partition into n roughly-equal buckets (quartiles, percentiles).

```sql
SELECT
  name, department, salary,
  -- ROW_NUMBER: a unique 1,2,3... within each partition (ties broken arbitrarily).
  row_number() OVER w AS rn,
  -- RANK: ties share a rank, then the next rank SKIPS (1,2,2,4).
  rank()       OVER w AS rnk,
  -- DENSE_RANK: ties share a rank, NO gap (1,2,2,3).
  dense_rank() OVER w AS dense,
  -- NTILE(4): split each partition into 4 buckets (quartiles).
  ntile(4)     OVER w AS quartile,
  -- PERCENT_RANK: relative standing 0..1 within the partition.
  round(percent_rank() OVER w * 100, 1) AS pct_rank
FROM employee
WINDOW w AS (PARTITION BY department ORDER BY salary DESC);  -- a reusable NAMED window
```

### LAG / LEAD — compare to neighbouring rows

`lag()` reaches *backwards* to a previous row and `lead()` reaches *forwards* — the standard way to compute deltas and period-over-period change without a self-join. The rows must be ordered (`ORDER BY` in the window) for "previous/next" to mean anything. `first_value`/`last_value` grab the boundary rows of the frame.

```sql
-- Month-over-month change in revenue.
WITH monthly AS (
  SELECT date_trunc('month', placed_at) AS month, sum(total) AS revenue
  FROM orders GROUP BY 1
)
SELECT
  month,
  revenue,
  lag(revenue)  OVER (ORDER BY month) AS prev_month,             -- the previous row's revenue
  revenue - lag(revenue) OVER (ORDER BY month) AS delta,         -- this month minus last month
  lead(revenue) OVER (ORDER BY month) AS next_month,             -- the next row's revenue
  first_value(revenue) OVER (ORDER BY month) AS first_month,
  last_value(revenue)  OVER (ORDER BY month
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_month  -- need full frame!
FROM monthly
ORDER BY month;
```

### Running totals & moving averages (the frame clause, explained properly)

This is where precision matters. The **frame** is the slice of the partition the aggregate considers for the current row. The default frame, if you write `ORDER BY` but no explicit frame, is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — and that "RANGE" default has a sharp edge: it treats all rows with the *same ORDER BY value* as a single peer group, lumping ties together. For a true row-by-row running total you almost always want `ROWS` instead, which counts physical rows. The distinctions:

- **`ROWS`** — a literal count of physical rows (e.g. "the current row and the 2 before it"). Predictable; use this for moving averages and running totals.
- **`RANGE`** — peers sharing the same `ORDER BY` value are all included at once (so ties are grouped). The default; surprising with duplicate sort keys.
- **`GROUPS`** (PG11+) — counts *peer groups* rather than rows.

```sql
SELECT
  name, department, salary, hired_at,
  -- Running total of salary in hire order, within each department:
  sum(salary) OVER (
    PARTITION BY department ORDER BY hired_at
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW   -- ROWS frame = physical rows (predictable)
  ) AS running_payroll,
  -- 3-row moving average (current row plus the 2 preceding):
  round(avg(salary) OVER (
    PARTITION BY department ORDER BY hired_at
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
  ), 2) AS moving_avg,
  -- Each row's salary as a % of its department total (no ORDER BY => the WHOLE partition is the frame):
  round(100.0 * salary / sum(salary) OVER (PARTITION BY department), 1) AS pct_of_dept
FROM employee;
```

> **ROWS vs RANGE — the practical rule:** if your `ORDER BY` column can have duplicate values and you want a strict per-row running total, write `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` explicitly. Relying on the `RANGE` default will lump equal-valued rows together and inflate intermediate totals.

### Top-N per group with window functions

A clean alternative to the `LATERAL` approach from §5: rank within each partition, then filter by the rank in an outer query. This also gives you the rank as a usable column.

```sql
-- Cleaner than LATERAL when you also want the rank number itself:
SELECT * FROM (
  SELECT name, department, salary,
         row_number() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
  FROM employee
) ranked
WHERE rn <= 2;     -- the top 2 earners per department
```

> **Why you can't filter on a window function in WHERE:** window functions are computed *after* `WHERE`/`GROUP BY`/`HAVING`, so `WHERE rn <= 2` is illegal in the same query level — the `rn` column doesn't exist yet at `WHERE` time. The subquery (or a CTE) is how you make the computed rank available to an outer filter.

⚡ **Version note (PG18):** The syntax above is stable across PG14–18. PG18's general planner/executor improvements speed up window aggregates over large partitions, but you don't change how you write them.

---

## 9. Working with JSON & JSONB

**[Intermediate]**

JSONB is what lets PostgreSQL absorb the "I need a document store" use case without giving up relations, transactions, and joins. The design question to settle first, though, is *when* to use it. JSONB shines for genuinely **variable or schemaless** data: sparse attributes that differ per row, user-defined custom fields, or storing an external API's payload verbatim. It is the **wrong** tool for data that has a stable shape — if every row has the same fields, real columns are faster, type-checked, constrainable, and indexable per-field. A very common and healthy pattern is **hybrid**: model the core, queried, constrained attributes as normal columns, and add one `jsonb` "extras" column for the long tail of optional fields. (When you find yourself heavily querying *inside* JSONB, that's often a signal the data wanted to be relational — **→ see RELATIONAL_DB_DESIGN_GUIDE.md**.)

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

The operator zoo is the part to memorize, because the *return type* matters. The single-arrow family extracts: `->` returns JSONB (so you can chain further), `->>` returns text (the final value you actually use). The `#>` / `#>>` variants take a path array for digging into nested structures. The containment/existence operators (`@>`, `?`, `?|`, `?&`) are special because **they can use a GIN index** — they're how you query JSONB efficiently at scale.

```sql
SELECT data -> 'name'              FROM doc;   -- '->'  get a field as JSONB   -> "Ada"
SELECT data ->> 'name'            FROM doc;   -- '->>' get a field as TEXT    -> Ada
SELECT data -> 'skills' -> 0      FROM doc;   -- array element as JSONB (0-indexed here)
SELECT data -> 'skills' ->> 0     FROM doc;   -- array element as text
SELECT data #> '{address,city}'   FROM doc;   -- '#>'  nested value as JSONB (path as an array)
SELECT data #>> '{address,city}'  FROM doc;   -- '#>>' nested value as text

-- Containment & existence (these can be accelerated by a GIN index — see below):
SELECT * FROM doc WHERE data @> '{"active":true}';        -- '@>' does data CONTAIN this JSON?
SELECT * FROM doc WHERE data ? 'address';                 -- '?'  does this top-level key exist?
SELECT * FROM doc WHERE data ?| array['email','phone'];   -- '?|' does ANY of these keys exist?
SELECT * FROM doc WHERE data ?& array['name','skills'];   -- '?&' do ALL of these keys exist?
SELECT * FROM doc WHERE data -> 'skills' @> '["sql"]';    -- array contains the element 'sql'
```

### jsonb functions & path queries

Beyond operators, a rich function set inspects, expands, and modifies JSONB. Note that the modification functions return a *new* JSONB value — to persist a change you wrap them in an `UPDATE`. The SQL/JSON **path language** (`jsonb_path_query`, PG12+) is a compact mini-language (like XPath for JSON) for complex extraction and filtering.

```sql
SELECT jsonb_pretty(data) FROM doc WHERE id = 1;          -- human-readable formatting
SELECT jsonb_typeof(data -> 'skills') FROM doc;           -- 'array', 'object', 'string', ...
SELECT jsonb_array_length(data -> 'skills') FROM doc;     -- 2
SELECT jsonb_object_keys(data) FROM doc WHERE id = 1;     -- one row per top-level key

-- Expand a JSON array into rows (so you can join/aggregate over elements):
SELECT id, skill
FROM doc, jsonb_array_elements_text(data -> 'skills') AS skill;

-- Expand a JSON object into key/value rows:
SELECT key, value FROM doc, jsonb_each(data) WHERE id = 1;

-- Modify JSONB (each returns a NEW value; wrap in UPDATE to persist):
UPDATE doc SET data = jsonb_set(data, '{address,country}', '"UK"') WHERE id = 1;  -- set/insert at path
UPDATE doc SET data = data || '{"verified":true}'::jsonb WHERE id = 1;            -- '||' merge/concat
UPDATE doc SET data = data - 'active' WHERE id = 2;                               -- '-' delete a key
UPDATE doc SET data = data #- '{address,zip}' WHERE id = 1;                       -- '#-' delete at path

-- SQL/JSON path language (PG12+):
SELECT jsonb_path_query(data, '$.skills[*]') FROM doc WHERE id = 1;       -- each skill element
SELECT jsonb_path_query_array(data, '$.skills[*] ? (@ starts with "g")') FROM doc;  -- filtered
SELECT data @? '$.skills[*] ? (@ == "sql")' FROM doc;    -- '@?' does the path match anything?
SELECT data @@ '$.active == true' FROM doc;              -- '@@' evaluate a path predicate to boolean
```

### JSON_TABLE — turn JSON into a relational result (PG17)

⚡ **Version note (PG17):** `JSON_TABLE` is the SQL-standard way to "shred" a JSON document into a proper relational table of columns and rows — a big ergonomic and performance win over manually chaining `jsonb_array_elements` and extraction operators. Define a row pattern (which part of the document becomes rows) and a `COLUMNS` list (how to project fields into typed columns).

```sql
-- Flatten each doc's skills array into a clean relational result:
SELECT d.id, jt.skill, jt.idx
FROM doc d,
  JSON_TABLE(
    d.data,
    '$.skills[*]'                       -- row pattern: produce one output row per skill
    COLUMNS (
      idx   FOR ORDINALITY,             -- a 1-based position counter
      skill text PATH '$'               -- the scalar value at this path
    )
  ) AS jt;

-- Extract nested fields into named, typed columns:
SELECT jt.*
FROM doc d,
  JSON_TABLE(d.data, '$'
    COLUMNS (
      name   text    PATH '$.name',
      city   text    PATH '$.address.city',
      active boolean PATH '$.active'
    )
  ) AS jt;
```

### Indexing JSONB with GIN

To query JSONB at scale you index it. The default **GIN** index supports the containment and existence operators (`@>`, `?`, `?|`, `?&`). The `jsonb_path_ops` variant is smaller and faster but supports *only* `@>` (containment) — choose it when containment is all you need. For a single frequently-queried field, a plain **B-tree expression index** on the extracted value gives you fast equality/range on that one field, often cheaper than a full GIN. Match the index to the query shape (and confirm with `EXPLAIN`, §12).

```sql
-- Default GIN: supports @>, ?, ?|, ?& and key/value lookups.
CREATE INDEX idx_doc_data ON doc USING gin (data);

-- jsonb_path_ops: smaller & faster, but ONLY supports @> (containment):
CREATE INDEX idx_doc_data_path ON doc USING gin (data jsonb_path_ops);

-- B-tree EXPRESSION index on one hot field — best for equality/range on that single field:
CREATE INDEX idx_doc_name ON doc ((data ->> 'name'));
SELECT * FROM doc WHERE data ->> 'name' = 'Ada';   -- can use idx_doc_name
SELECT * FROM doc WHERE data @> '{"active":true}'; -- can use idx_doc_data
```

### Building JSON output

The reverse direction — assembling JSON from relational rows — is invaluable for API responses, letting the database return ready-to-serialize nested structures and avoiding N+1 round-trips from the app. `jsonb_build_object`/`jsonb_build_array` construct values; `jsonb_agg` and `json_object_agg` aggregate rows into JSON arrays/objects; `to_jsonb(row)` turns a whole row into an object.

```sql
-- Build a JSON object per row (great for shaping API responses in the DB):
SELECT jsonb_build_object('id', id, 'name', name, 'salary', salary) FROM employee;

-- Aggregate many rows into a single JSON array of objects:
SELECT jsonb_agg(jsonb_build_object('name', name, 'salary', salary)) AS team
FROM employee WHERE department = 'Engineering';

-- Whole row as JSON, plus nested aggregation (one row per dept carrying its members array):
SELECT department,
       jsonb_agg(to_jsonb(e) - 'department' ORDER BY salary DESC) AS members
FROM employee e
GROUP BY department;

-- json_agg vs jsonb_agg: json_agg preserves input order/formatting; jsonb_agg returns canonical jsonb.
SELECT json_object_agg(name, salary) FROM employee;   -- {"Ada":120000, ...}
```

> **Security note:** building JSON in the database is safe from injection as long as the *values* come from columns/parameters, not from string-concatenated user input. If you ever construct JSON keys from user input, validate them — don't interpolate raw strings into a JSON text literal.

---

## 10. Arrays & Full-Text Search

**[Intermediate]**

### Array querying recap & indexing

Arrays (introduced in §2) become powerful when you query *inside* them with the set operators and back those queries with a GIN index. The three containment operators mirror set logic: `&&` (overlap — any common element), `@>` (contains all of these), `<@` (is contained by). A GIN index makes all three fast, which is what makes a `text[] tags` column a viable design for moderate tag sets.

```sql
-- (article table from §2 has a tags text[] column)
SELECT * FROM article WHERE tags && ARRAY['sql','python'];  -- '&&' overlaps (shares ANY element)
SELECT * FROM article WHERE tags @> ARRAY['sql'];           -- '@>' contains ALL listed elements
SELECT * FROM article WHERE tags <@ ARRAY['sql','db','x'];  -- '<@' is contained by this set

-- A GIN index makes @>, <@, && fast on array columns:
CREATE INDEX idx_article_tags ON article USING gin (tags);

-- Aggregate operations across the elements of all rows:
SELECT array_agg(DISTINCT t) AS all_tags FROM article, unnest(tags) AS t;
SELECT t, count(*) FROM article, unnest(tags) AS t GROUP BY t ORDER BY count(*) DESC;  -- tag cloud
```

### Full-text search (FTS) — what it is and why not just `LIKE`

`LIKE '%word%'` is naive: it can't stem ("running" won't match "run"), can't ignore stop words ("the", "a"), can't rank by relevance, and can't use an index for a leading-wildcard pattern. Full-text search solves all of this with a proper linguistic pipeline. It converts documents into a **`tsvector`** — a sorted set of normalized *lexemes* (stemmed word roots) with positions — and queries into a **`tsquery`** of search terms combined with boolean operators, then matches them with the `@@` operator. A language configuration (`'english'`) drives the stemming and stop-word rules.

```sql
-- 1) Inspect the building blocks to see the normalization happen:
SELECT to_tsvector('english', 'The cats are running quickly');
--   'cat':2 'quick':5 'run':4   (stop words removed, words stemmed, positions retained)
SELECT to_tsquery('english', 'cat & run');          -- 'cat' AND 'run'
SELECT plainto_tsquery('english', 'running cats');  -- forgiving: plain text -> 'run' & 'cat'
SELECT websearch_to_tsquery('english', '"data science" -python');  -- web-style: phrases & negation

-- 2) Match a document against a query:
SELECT to_tsvector('english','quick brown fox') @@ to_tsquery('english','fox & quick');  -- true
```

### A searchable table the right way

The production pattern: store a **generated `tsvector` column** so the search index stays automatically in sync with the source text (no triggers to maintain), apply **weights** so matches in the title outrank matches in the body, and put a **GIN index** on the tsvector. Then queries match with `@@`, rank with `ts_rank`, and can even return highlighted snippets with `ts_headline`.

```sql
CREATE TABLE post (
  id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  title text NOT NULL,
  body  text NOT NULL,
  -- A GENERATED tsvector column keeps the search index in sync automatically (PG12+).
  -- Weights 'A'/'B' make title matches rank higher than body matches.
  search tsvector GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title,'')), 'A') ||
    setweight(to_tsvector('english', coalesce(body, '')), 'B')
  ) STORED
);

-- A GIN index over the tsvector column = fast search:
CREATE INDEX idx_post_search ON post USING gin (search);

INSERT INTO post (title, body) VALUES
  ('PostgreSQL indexing', 'Learn about B-tree, GIN, and GiST indexes.'),
  ('Cooking with cast iron', 'A guide to seasoning and maintaining pans.');

-- Search, rank by relevance, and return a highlighted snippet:
SELECT id, title,
       ts_rank(search, q) AS rank,
       ts_headline('english', body, q) AS snippet   -- highlights the matched terms
FROM post, websearch_to_tsquery('english', 'index') q
WHERE search @@ q
ORDER BY rank DESC;
```

> **Trigram alternative (`pg_trgm`):** FTS is built for *words* in *documents*. For typo-tolerant matching, substring search, autocomplete, or making `ILIKE '%term%'` fast on short strings (names, SKUs), use the `pg_trgm` extension (§20) with a GIN/GiST trigram index instead — it indexes 3-character sequences and supports similarity scoring. Choose FTS for prose, trigrams for short identifiers and fuzzy matching.

---

## 11. Indexing Deeply

**[Advanced]**

An index is a separate, ordered data structure that lets the database find rows without scanning the whole table — the difference between flipping to a book's index versus reading every page. The fundamental trade-off is always the same: indexes **speed up reads** but **slow down writes** (every `INSERT`/`UPDATE`/`DELETE` must also maintain every relevant index) and **consume storage**. So indexing is an optimization you apply deliberately, guided by your actual query patterns and verified with `EXPLAIN` (§12) — not by reflexively indexing everything. The other crucial idea: the planner *chooses* whether to use an index based on cost estimates from table statistics; an index that exists but isn't selective enough (or whose stats are stale) simply won't be used.

### The index types and when each applies

PostgreSQL offers several index *methods*, each suited to different data and operators. Choosing the right type is what turns a slow query fast. The mental shortcuts: **B-tree** for ordered scalar data and the vast majority of cases; **GIN** for "many values inside one row" (arrays, JSONB, full-text); **GiST** for ranges, geometry, and nearest-neighbour; **BRIN** for enormous naturally-ordered tables (time-series) where a tiny index is worth a coarse one.

| Index | Best for | Notes |
|---|---|---|
| **B-tree** (default) | `=`, `<`, `>`, `BETWEEN`, `IN`, `ORDER BY`, prefix `LIKE 'abc%'` | The workhorse. Multicolumn, unique, ordered scans. |
| **Hash** | equality `=` only | Smaller for pure equality; crash-safe & WAL-logged since PG10. Rarely beats B-tree. |
| **GIN** | "many values per row": arrays, JSONB, FTS (`tsvector`), `pg_trgm` | Fast lookups, slower/heavier writes (mitigated by the fastupdate pending list). |
| **GiST** | geometric/range/nearest-neighbour, FTS, exclusion constraints | Extensible; supports `&&`, `<->` (KNN), ranges, PostGIS. |
| **SP-GiST** | space-partitioned data: quadtrees, IP/text-prefix trees | Good for non-balanced, naturally partitioned structures. |
| **BRIN** | huge, naturally-ordered tables (time-series append) | Tiny index storing per-block min/max. Cheap; great for date columns. |

```sql
-- B-tree (default — you can omit USING btree):
CREATE INDEX idx_emp_dept ON employee (department);
CREATE INDEX idx_emp_salary ON employee (salary DESC NULLS LAST);

-- Multicolumn B-tree: column ORDER matters (the "leftmost-prefix" rule). This index serves
-- filters on (department) and on (department, salary), but NOT salary alone:
CREATE INDEX idx_emp_dept_salary ON employee (department, salary);

-- Unique index (can also be declared as a constraint); here it enforces case-insensitive uniqueness:
CREATE UNIQUE INDEX idx_customer_email ON customer (lower(email));

-- PARTIAL index: index only the subset of rows you actually query — smaller, faster, cheaper to maintain:
CREATE INDEX idx_orders_unpaid ON orders (placed_at) WHERE total > 0 AND placed_at IS NOT NULL;
CREATE INDEX idx_active_users ON customer (id) WHERE status = 'active';   -- only "active" rows indexed

-- EXPRESSION (functional) index: index the RESULT of an expression so a matching query can use it:
CREATE INDEX idx_emp_lower_name ON employee (lower(name));
SELECT * FROM employee WHERE lower(name) = 'ada';   -- can use idx_emp_lower_name

-- COVERING index with INCLUDE (PG11+): store extra columns in the leaf so a query can be
-- answered from the index ALONE (an index-only scan), never touching the table heap:
CREATE INDEX idx_orders_cust_inc ON orders (customer_id) INCLUDE (total, placed_at);

-- BRIN for a big append-only time-series table (tiny index, coarse but cheap):
CREATE INDEX idx_events_time_brin ON event USING brin (created_at) WITH (pages_per_range = 32);

-- GIN for JSONB / arrays / FTS (see §9, §10).
```

### Why multicolumn column order matters (the leftmost-prefix rule)

A multicolumn B-tree is sorted by the first column, then the second within ties, and so on — like a phone book sorted by last name then first name. So it can satisfy a query that filters on a *leftmost prefix* of its columns, but not one that skips the leading column. `(department, salary)` helps `WHERE department = ?` and `WHERE department = ? AND salary > ?`, but not `WHERE salary > ?` alone. The design rule: put the column(s) used for **equality** first, the column used for **range/sort** last, and order by selectivity when in doubt.

### Index-only scans & visibility

An **index-only scan** is the fastest read pattern: the query is answered entirely from the index without fetching the table heap. Two conditions must hold: (1) every column the query needs is in the index (use `INCLUDE` to add payload columns), and (2) the rows are marked all-visible in the **visibility map**, which `VACUUM` maintains. If you see non-zero "Heap Fetches" in `EXPLAIN ANALYZE`, the visibility map is stale and Postgres had to visit the heap anyway — a sign the table needs vacuuming (§16).

```sql
-- Verify with EXPLAIN whether you actually get an index-only scan:
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, total FROM orders WHERE customer_id = 42;
-- Look for "Index Only Scan using idx_orders_cust_inc" AND "Heap Fetches: 0".
```

### Maintenance & inspection

Two operational essentials. First, **build and rebuild indexes `CONCURRENTLY` in production** — a plain `CREATE INDEX` takes a lock that blocks writes for the whole build; `CONCURRENTLY` builds online at the cost of being slower and not running inside a transaction. Second, **find and drop unused indexes** — they cost write overhead and storage for no benefit; `pg_stat_user_indexes` tracks how often each is used.

```sql
-- Build/drop without locking out writes (slower, online; must run OUTSIDE a transaction):
CREATE INDEX CONCURRENTLY idx_emp_hired ON employee (hired_at);
DROP INDEX CONCURRENTLY idx_emp_hired;

-- Rebuild a bloated index online (PG12+):
REINDEX INDEX CONCURRENTLY idx_emp_dept;
REINDEX TABLE CONCURRENTLY employee;

-- Find UNUSED indexes (candidates for removal — pure write overhead):
SELECT relname AS table, indexrelname AS index, idx_scan AS times_used,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC, pg_relation_size(indexrelid) DESC;

-- Inspect an index's size:
SELECT pg_size_pretty(pg_relation_size('idx_emp_dept'));
```

⚡ **Version note (PG18):** PG18 adds B-tree **"skip scan,"** letting a multicolumn index be used even when the leading column is absent from the `WHERE` clause (the executor skips through the leading column's distinct values). This relaxes the strict leftmost-prefix rule for *low-cardinality* leading columns — but deliberately ordering columns for your real query mix still matters.

> **Indexing rules of thumb:** index columns used in `WHERE`, `JOIN`, and `ORDER BY`; **index every foreign key** (Postgres doesn't, §3); put equality/most-selective columns first in multicolumn indexes; use partial indexes for skewed predicates (e.g. `WHERE status = 'active'`); add `INCLUDE` columns to enable index-only scans on hot read paths; don't over-index (each one taxes writes and bloats storage). Always confirm the planner actually uses your index with `EXPLAIN` — an unused index is worse than none.

---

## 12. EXPLAIN, Query Plans & Finding Slow Queries

**[Advanced]**

When a query is slow, **don't guess — ask the planner.** `EXPLAIN` prints the execution plan the optimizer chose and its *estimated* costs without running the query; `EXPLAIN ANALYZE` actually executes it and reports *real* timings and row counts alongside the estimates. The gap between estimated and actual rows is the single most valuable diagnostic signal you have: when they diverge wildly, the planner is working from bad statistics and is likely choosing a poor plan. Reading plans fluently is the core skill of database performance work, and it's learnable — every plan is a tree of a small set of node types.

```sql
EXPLAIN SELECT * FROM employee WHERE department = 'Engineering';
-- Shows the ESTIMATED plan only (does NOT run the query).

EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT)
SELECT e.name, m.name AS manager
FROM employee e JOIN employee m ON e.manager_id = m.id
WHERE e.salary > 100000;
--  ANALYZE : actually execute and show real time/rows (so you can compare to estimates)
--  BUFFERS : show shared/local buffer hits & reads (actual I/O) — essential for tuning
--  VERBOSE : show output columns and schema-qualified names
--  FORMAT  : TEXT (default), JSON, YAML, XML
```

> **Caution:** `EXPLAIN ANALYZE` *runs* the statement — for `INSERT`/`UPDATE`/`DELETE` it will modify data. Wrap it in a transaction you roll back: `BEGIN; EXPLAIN ANALYZE UPDATE ...; ROLLBACK;`.

### Reading a plan

Plans are trees, read **inside-out / bottom-up**: the deepest, most-indented nodes run first and feed their parents. Each node line reports:

- **`cost=startup..total`** — abstract cost units the planner uses to compare plans (lower total wins). "startup" is the cost before the first row emerges; "total" is for all rows.
- **`rows=`** — the *estimated* row count. Under `ANALYZE` you also get `actual rows`. **A large estimate-vs-actual gap is the #1 sign of a planning problem** (stale stats, or correlated columns the planner assumes are independent).
- **`actual time=startup..total`** (ms) and **`loops=`** — how many times the node executed; the per-loop time is multiplied by loops, so a cheap-looking node run 100,000 times is expensive.

### Scan types

| Node | Meaning | When good / bad |
|---|---|---|
| **Seq Scan** | read the entire table | Fine for small tables or when most rows match; bad on big tables with a selective filter. |
| **Index Scan** | walk the index, then fetch matching heap rows | Good for selective predicates. |
| **Index Only Scan** | answer from the index alone, no heap | Best — needs a covering index + fresh visibility map. |
| **Bitmap Index/Heap Scan** | collect many matches via index(es), then fetch heap in physical order | Good for medium selectivity and combining multiple indexes. |

### Join algorithms

| Join | How it works | Best when |
|---|---|---|
| **Nested Loop** | for each outer row, probe the inner side (ideally via an index) | Small outer side and an indexed inner. Catastrophic if outer is large and inner unindexed. |
| **Hash Join** | build a hash table on the smaller input, probe it with the larger | Large unsorted inputs, equality joins. Uses `work_mem`. |
| **Merge Join** | sort both inputs, then merge in order | Both inputs already sorted (indexed columns) or cheaply sortable. |

### A worked diagnosis

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 42;
-- BAD plan symptom:
--   Seq Scan on orders  (cost=0.00..18500 rows=1) (actual rows=1 loops=1)
--     Filter: (customer_id = 42)
--     Rows Removed by Filter: 999999          <-- scanned a MILLION rows to return one!
-- FIX: give the planner an index so it can do an Index Scan instead:
CREATE INDEX idx_orders_customer ON orders (customer_id);
-- Re-run EXPLAIN; you should now see "Index Scan using idx_orders_customer" and far less I/O.
```

### The cost model & statistics

The planner estimates how many rows each step produces using **statistics** gathered by `ANALYZE` (a histogram and most-common-values list per column). Wrong estimates → wrong plans. Two levers: raise the statistics target on a skewed, important column to get a finer histogram; and create **extended statistics** to teach the planner about *correlated* columns (by default it assumes columns are independent, which badly underestimates selectivity when, say, `city` and `country` move together).

```sql
ANALYZE employee;                       -- refresh stats for one table
ANALYZE;                                -- the whole database

-- Finer stats on a skewed/important column (default 100 histogram buckets):
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 1000;
ANALYZE orders;

-- Extended statistics for CORRELATED columns (planner otherwise assumes independence):
CREATE STATISTICS s_city_country (dependencies, ndistinct)
  ON city, country FROM customer;
ANALYZE customer;

-- Key cost knobs (session or global). On SSD/NVMe, lower random_page_cost so index scans win:
SET random_page_cost = 1.1;             -- default 4.0 was tuned for spinning disks
SET effective_cache_size = '12GB';      -- a HINT about total OS+PG cache (no memory is allocated)
```

### Finding slow queries with pg_stat_statements

You can't `EXPLAIN` what you haven't identified. **`pg_stat_statements`** is the indispensable extension that records aggregate execution stats per normalized query, so you can rank queries by total time spent — the real hotspots are usually "fast but run millions of times," not the one slow report. Pair it with `pg_stat_activity` (what's running *right now*) to catch live problems.

```sql
-- One-time setup: add to shared_preload_libraries in postgresql.conf, then restart:
--   shared_preload_libraries = 'pg_stat_statements'
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top 10 queries by TOTAL time (the genuine hotspots, accounting for frequency):
SELECT
  round(total_exec_time::numeric, 1) AS total_ms,
  calls,
  round(mean_exec_time::numeric, 2)  AS mean_ms,
  round(100 * total_exec_time / sum(total_exec_time) OVER (), 1) AS pct,
  query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Reset counters after a deploy to measure the new code fresh:
SELECT pg_stat_statements_reset();

-- See currently-running queries and what each is waiting on:
SELECT pid, state, wait_event_type, wait_event,
       now() - query_start AS running_for, query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY running_for DESC;

-- Stop a runaway query (cancel) or close its connection (terminate):
SELECT pg_cancel_backend(12345);     -- politely cancel the current query in that backend
SELECT pg_terminate_backend(12345);  -- forcibly close the whole connection
```

> **`auto_explain`** logs the plan of any query exceeding a duration threshold automatically — set `auto_explain.log_min_duration = '500ms'` to capture slow production queries with their plans, no manual `EXPLAIN` needed. Invaluable for intermittent slowness you can't reproduce on demand.

---

## 13. Transactions, MVCC & Concurrency

**[Advanced]**

This is the chapter that separates "I can write SQL" from "I can build a correct concurrent system." A transaction groups statements so they succeed or fail as a unit, and Postgres guarantees the **ACID** properties: **A**tomicity (all-or-nothing), **C**onsistency (constraints always hold), **I**solation (concurrent transactions don't corrupt each other's view), and **D**urability (a committed transaction survives a crash). The reason to wrap related writes in a transaction is correctness under failure: a money transfer must debit *and* credit, or neither — never one without the other.

```sql
BEGIN;                                  -- start a transaction
UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;
-- Atomicity: if anything between BEGIN and COMMIT fails, NONE of it is applied.
COMMIT;                                 -- make all changes durable and visible to others
-- or
ROLLBACK;                               -- discard every change made since BEGIN

-- Savepoints let you partially roll back WITHIN a transaction:
BEGIN;
INSERT INTO employee (name, department, salary) VALUES ('Temp', 'X', 1);
SAVEPOINT sp1;
UPDATE employee SET salary = -5 WHERE name = 'Temp';   -- suppose this is a mistake
ROLLBACK TO SAVEPOINT sp1;             -- undo back to the savepoint; the transaction continues
COMMIT;
```

> **Gotcha — the aborted transaction state:** once *any* statement errors inside a transaction, the whole transaction enters an *aborted* state and every subsequent command fails with "current transaction is aborted" until you `ROLLBACK` (or `ROLLBACK TO SAVEPOINT`). Application drivers usually handle this for you, but it surprises people working interactively in psql.

### MVCC explained — the engine of Postgres concurrency

PostgreSQL's concurrency model is **Multi-Version Concurrency Control (MVCC)**, and understanding it explains almost every concurrency behaviour you'll meet. The core idea: **readers never block writers and writers never block readers.** Instead of overwriting a row in place, an `UPDATE` writes a *new version* of the row and marks the old version as expired; a `DELETE` just marks a row expired. Every transaction sees a consistent **snapshot** of the database — it sees row versions that were committed as of its snapshot point and ignores newer ones. Each row carries hidden system columns (`xmin` = the transaction id that created this version, `xmax` = the one that expired it) that the visibility rules use to decide which version each transaction should see.

The price of this elegance is **dead tuples**: old, expired row versions pile up and must be reclaimed by `VACUUM` (§16). This is *the* reason vacuuming matters in Postgres — without it, tables bloat with invisible dead versions. It's also why a long-running transaction is harmful: it holds an old snapshot, so `VACUUM` cannot remove dead tuples that the old snapshot might still need to see, and bloat accumulates.

```sql
-- Every row carries hidden system columns revealing its version:
SELECT xmin, xmax, ctid, * FROM employee LIMIT 1;
-- xmin = txid that created this version; ctid = the physical (page, offset) location of the row.
SELECT txid_current();   -- the current transaction's id
```

### Isolation levels

The isolation level controls *how much* concurrent transactions are shielded from each other's in-flight changes — a dial trading strictness against concurrency. The anomalies it prevents are: **dirty read** (seeing uncommitted data — Postgres never allows this), **non-repeatable read** (re-reading a row and getting a different committed value), **phantom read** (a re-run query gaining/losing rows), and **serialization anomaly** (the result differs from any serial order).

| Level | Dirty read | Non-repeatable read | Phantom read | Serialization anomaly |
|---|---|---|---|---|
| Read Uncommitted | (treated as Read Committed in PG) | possible | possible | possible |
| **Read Committed** (default) | no | possible | possible | possible |
| **Repeatable Read** | no | no | no* | possible |
| **Serializable** | no | no | no | no |

\* In PostgreSQL, Repeatable Read also prevents phantom reads (it implements true snapshot isolation), which is *stronger* than the SQL standard requires.

The practical guidance: **Read Committed** (the default) is right for most OLTP — each *statement* sees a fresh snapshot of committed data. Step up to **Repeatable Read** when a single transaction must see one stable snapshot across multiple statements (a consistent report, a multi-read calculation). Use **Serializable** when correctness requires that concurrent transactions behave *as if* they ran one at a time (financial invariants, inventory that must never oversell) — accepting that it may abort transactions with a serialization error you must retry.

```sql
-- Read Committed (default): each STATEMENT sees rows committed before that statement began.
-- Repeatable Read: the whole TRANSACTION sees one snapshot frozen at its first statement.
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT sum(balance) FROM account;       -- snapshot is fixed here
-- ... even if other transactions commit changes now, this txn keeps seeing the same data ...
SELECT sum(balance) FROM account;       -- identical result, guaranteed
COMMIT;

-- Serializable: as if transactions executed serially. May abort with SQLSTATE 40001 —
-- the application MUST catch this and retry the whole transaction.
BEGIN ISOLATION LEVEL SERIALIZABLE;
-- ... business logic ...
COMMIT;   -- might raise: "could not serialize access due to read/write dependencies"
```

> **The serializable retry pattern is mandatory, not optional.** Any transaction at Serializable (and sometimes Repeatable Read) can legitimately abort with `40001`. The *correct* response is to roll back and re-run the entire transaction from the top, not to log-and-ignore. Build a small retry loop (with a few attempts and a touch of backoff) around such transactions in your application code (§21 shows the shape in Go/Node).

### Locks

MVCC means you rarely need explicit locks for reads, but writes that must coordinate (read-modify-write on the same row) need them. `SELECT ... FOR UPDATE` locks the selected rows so no other transaction can update them until you commit — preventing lost updates. The modifiers `NOWAIT` (error immediately if locked) and `SKIP LOCKED` (silently skip locked rows) are powerful for queue-like workloads.

```sql
-- Row-level lock: prevent concurrent updates to the rows you're about to modify.
BEGIN;
SELECT * FROM account WHERE id = 1 FOR UPDATE;   -- lock this row against other writers
UPDATE account SET balance = balance - 100 WHERE id = 1;
COMMIT;

-- Lock strengths: FOR UPDATE (strongest), FOR NO KEY UPDATE, FOR SHARE, FOR KEY SHARE (weakest).
-- Behaviour when rows are already locked:
SELECT * FROM job WHERE status = 'queued' FOR UPDATE NOWAIT;       -- error out if any row is locked
SELECT * FROM job WHERE status = 'queued' FOR UPDATE SKIP LOCKED;  -- skip rows others hold

-- Inspect locks that are currently being waited on:
SELECT locktype, relation::regclass, mode, granted, pid
FROM pg_locks WHERE NOT granted;
```

### Job queue with SELECT ... FOR UPDATE SKIP LOCKED

A genuinely beautiful pattern: a reliable work queue with *no external broker*, just a table. Multiple workers each run the same statement; `SKIP LOCKED` ensures they grab *different* jobs without blocking on each other, and the row lock guarantees no job is processed twice. This is production-grade and worth committing to memory.

```sql
CREATE TABLE job (
  id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  payload   jsonb NOT NULL,
  status    text NOT NULL DEFAULT 'queued',
  locked_at timestamptz
);

-- Each worker runs this to atomically claim exactly ONE job:
WITH next_job AS (
  SELECT id FROM job
  WHERE status = 'queued'
  ORDER BY id
  FOR UPDATE SKIP LOCKED          -- skip rows other workers have already claimed
  LIMIT 1
)
UPDATE job
SET status = 'processing', locked_at = now()
FROM next_job
WHERE job.id = next_job.id
RETURNING job.*;                  -- this worker gets its job; concurrent workers get different ones
```

### Deadlocks

A deadlock is a cycle: transaction A holds a lock B wants, while B holds a lock A wants — neither can proceed. PostgreSQL detects deadlocks automatically and resolves them by aborting one transaction with a deadlock error. You can't make deadlocks impossible in general, but you can prevent the common kind by **always acquiring locks in a consistent order** across your code paths (e.g. always lock the lower account id first), so a cycle can never form.

```sql
-- Prevent deadlocks by acquiring locks in a CONSISTENT, deterministic order everywhere:
BEGIN;
SELECT * FROM account WHERE id = least(1, 2) FOR UPDATE;     -- always the smaller id first
SELECT * FROM account WHERE id = greatest(1, 2) FOR UPDATE;  -- then the larger
-- ... perform the transfer ...
COMMIT;
```

### Advisory locks — application-level mutexes

Advisory locks aren't tied to any row or table — *you* choose the integer key and Postgres just tracks who holds it. They're the right tool for "only one worker may run this job at a time" coordination across processes. Prefer the **transaction-scoped** variants (`pg_advisory_xact_lock`), which auto-release at COMMIT/ROLLBACK and therefore can't leak if your code forgets to unlock.

```sql
-- Not tied to any data; the key is yours to define. Useful as a distributed mutex.
SELECT pg_advisory_lock(42);        -- blocks until acquired (SESSION-scoped — you must unlock)
SELECT pg_try_advisory_lock(42);    -- non-blocking; returns true/false immediately
SELECT pg_advisory_unlock(42);      -- release a session-scoped lock
-- Transaction-scoped (auto-released at COMMIT/ROLLBACK — safer, no leak risk):
SELECT pg_advisory_xact_lock(42);
```

---

## 14. Functions, Procedures & Triggers (PL/pgSQL)

**[Advanced]**

PostgreSQL lets you run code *inside* the database. The reason to do this — and it's a genuine architectural choice with trade-offs — is to enforce invariants and perform logic right next to the data, where it runs atomically within the transaction and can't be bypassed by any client. The cost is that database-side logic is harder to test, version, and debug than application code, and it scales with the (expensive) database server. The pragmatic stance: use functions/triggers for things that *must* hold no matter who writes (audit trails, `updated_at` maintenance, integrity that a CHECK can't express), and keep ordinary business logic in the application. A function's **volatility** label (`IMMUTABLE`/`STABLE`/`VOLATILE`) is important: it tells the planner whether the function's result can be cached or used in an index, so labelling it correctly is both a performance and a correctness matter.

### SQL functions

The simplest functions are written in plain SQL. If marked `IMMUTABLE` (same inputs always give the same output, no database access), the planner can inline them and even use them in expression indexes.

```sql
-- A simple, inlinable SQL function. The volatility label guides the planner:
--   IMMUTABLE = same inputs => same output, no DB reads (eligible for indexes/constant-folding)
--   STABLE    = consistent within a single statement (e.g. reads tables, uses now())
--   VOLATILE  = may change every call / has side effects (the default; e.g. random(), writes)
CREATE OR REPLACE FUNCTION full_name(first text, last text)
RETURNS text
LANGUAGE sql
IMMUTABLE
AS $$
  SELECT first || ' ' || last;
$$;
SELECT full_name('Ada', 'Lovelace');

-- A set-returning SQL function (returns rows, usable in FROM):
CREATE OR REPLACE FUNCTION emps_in(dept text)
RETURNS SETOF employee
LANGUAGE sql STABLE
AS $$ SELECT * FROM employee WHERE department = dept; $$;
SELECT * FROM emps_in('Engineering');
```

### PL/pgSQL — the procedural language

When you need variables, control flow, loops, and exception handling, use **PL/pgSQL**. It adds imperative constructs around SQL: `DECLARE` variables, `SELECT ... INTO` to capture query results, `IF`/`CASE`/`LOOP`/`FOR`/`WHILE`, and `RAISE` to emit messages or throw errors (which abort and roll back the transaction). The `$$ ... $$` "dollar quoting" is just a way to write the function body as a string literal without escaping inner quotes.

```sql
CREATE OR REPLACE FUNCTION give_raise(emp_id bigint, pct numeric)
RETURNS numeric           -- returns the new salary
LANGUAGE plpgsql
AS $$
DECLARE
  current_salary numeric;
  new_salary     numeric;
BEGIN
  -- Capture a query result into a variable:
  SELECT salary INTO current_salary FROM employee WHERE id = emp_id;

  -- Control flow + raising an error (aborts and rolls back the transaction):
  IF current_salary IS NULL THEN
    RAISE EXCEPTION 'Employee % not found', emp_id;
  END IF;

  new_salary := current_salary * (1 + pct / 100.0);
  UPDATE employee SET salary = new_salary WHERE id = emp_id;

  RAISE NOTICE 'Raised % from % to %', emp_id, current_salary, new_salary;  -- log, doesn't abort
  RETURN new_salary;
END;
$$;
SELECT give_raise(1, 10);
```

```sql
-- Loops, iterating over a query, and per-iteration exception handling:
CREATE OR REPLACE FUNCTION normalize_salaries()
RETURNS int LANGUAGE plpgsql AS $$
DECLARE
  rec     record;
  updated int := 0;
BEGIN
  FOR rec IN SELECT id, salary FROM employee WHERE salary < 50000 LOOP
    BEGIN
      UPDATE employee SET salary = 50000 WHERE id = rec.id;
      updated := updated + 1;
    EXCEPTION WHEN others THEN
      -- A nested BEGIN/EXCEPTION catches per-row errors so one bad row doesn't kill the loop:
      RAISE WARNING 'Skipping % due to %', rec.id, SQLERRM;
    END;
  END LOOP;

  -- Other loop forms:  WHILE condition LOOP ... END LOOP;   FOR i IN 1..10 LOOP ... END LOOP;
  RETURN updated;
END;
$$;
```

> **Security note — SQL injection inside functions:** if a function builds dynamic SQL from its arguments and runs it with `EXECUTE`, it is just as injectable as application code. Never concatenate arguments into the SQL string; use `EXECUTE format('... %I ... %L ...', ident, value)` (`%I` quotes identifiers, `%L` quotes literals) or `EXECUTE '...' USING value` for parameters. For `SECURITY DEFINER` functions (which run with the *definer's* privileges), always pin `SET search_path = pg_catalog, public` on the function so an attacker can't shadow a table/function name — a classic privilege-escalation route.

### Procedures (can manage their own transactions)

The key difference from a function: a **procedure** (PG11+) can `COMMIT` and `ROLLBACK` *inside* its body, so it can process work in batches, committing each chunk to keep transactions short (important — long transactions block VACUUM, §16). Procedures are invoked with `CALL`, not `SELECT`.

```sql
-- A procedure can COMMIT mid-body — ideal for chunked maintenance that avoids one giant transaction.
CREATE OR REPLACE PROCEDURE batch_archive(batch int)
LANGUAGE plpgsql AS $$
DECLARE moved int;
BEGIN
  LOOP
    WITH del AS (
      DELETE FROM orders
      WHERE id IN (SELECT id FROM orders WHERE placed_at < '2020-01-01' LIMIT batch)
      RETURNING *
    )
    INSERT INTO orders_archive SELECT * FROM del;
    GET DIAGNOSTICS moved = ROW_COUNT;   -- how many rows the last statement affected
    EXIT WHEN moved = 0;
    COMMIT;                              -- commit each batch to keep transactions short
  END LOOP;
END;
$$;
CALL batch_archive(1000);               -- procedures are invoked with CALL
```

### Triggers

A trigger runs a function automatically in response to `INSERT`/`UPDATE`/`DELETE` on a table — the database's event hooks. The trigger function returns type `trigger` and accesses the special row variables `NEW` (the incoming row) and `OLD` (the prior row), plus `TG_OP`/`TG_TABLE_NAME` metadata. Timing matters: a **`BEFORE`** trigger can *modify* `NEW` before it's written (used for `updated_at`); an **`AFTER`** trigger fires once the write is done (used for auditing/side effects); an **`INSTEAD OF`** trigger makes a view writable. Triggers can fire per row (`FOR EACH ROW`) or once per statement (`FOR EACH STATEMENT`), and a `WHEN` clause can make them conditional.

```sql
-- 1) updated_at auto-maintenance — the single most common trigger:
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  NEW.updated_at := now();   -- modify the row being written (only meaningful in a BEFORE trigger)
  RETURN NEW;                -- returning NEW lets the operation proceed with the modification
END;
$$;

ALTER TABLE customer ADD COLUMN updated_at timestamptz NOT NULL DEFAULT now();
CREATE TRIGGER trg_customer_updated
BEFORE UPDATE ON customer
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();

-- 2) Audit log via an AFTER trigger (records every change as JSONB):
CREATE TABLE audit_log (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  table_name text, op text, row_data jsonb, changed_at timestamptz DEFAULT now()
);
CREATE OR REPLACE FUNCTION audit_changes()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  INSERT INTO audit_log (table_name, op, row_data)
  VALUES (TG_TABLE_NAME, TG_OP, to_jsonb(COALESCE(NEW, OLD)));  -- NEW for ins/upd, OLD for delete
  RETURN NULL;   -- AFTER triggers ignore the return value
END;
$$;
CREATE TRIGGER trg_customer_audit
AFTER INSERT OR UPDATE OR DELETE ON customer
FOR EACH ROW EXECUTE FUNCTION audit_changes();

-- 3) Conditional trigger with WHEN (fire only for large orders):
CREATE TRIGGER trg_only_big
AFTER UPDATE ON orders
FOR EACH ROW
WHEN (NEW.total > 10000)
EXECUTE FUNCTION audit_changes();

-- INSTEAD OF triggers make a VIEW writable:
-- CREATE TRIGGER ... INSTEAD OF INSERT ON some_view FOR EACH ROW EXECUTE FUNCTION ...;
```

> **Trigger gotchas:** keep trigger logic light — it runs inside the caller's transaction and multiplies every write's cost. Beware *recursive* triggers (a trigger that updates its own table re-fires itself). And beware "spooky action at a distance": logic hidden in triggers is hard for the next developer to discover, so reserve triggers for cross-cutting invariants (auditing, timestamps) and keep discoverable business rules in application code.

---

## 15. Views & Materialized Views

**[Advanced]**

A **view** is a stored query that behaves like a virtual table — it holds no data; every time you select from it, the underlying query runs fresh. Views serve three purposes: **abstraction** (hide a complex join behind a simple name), **reuse** (define the query once), and **security** (grant access to a view that exposes only certain columns/rows, while revoking access to the base table — a real, useful privilege boundary, §19). A **materialized view**, by contrast, *stores* the query result physically: reads are fast because the work is precomputed, but the data is stale until you `REFRESH` it. The choice is a freshness-vs-speed trade: plain views for always-live data and abstraction; materialized views for expensive aggregates that are read far more often than the underlying data changes (dashboards, periodic reports).

```sql
-- A VIEW is a saved query — stores no data, runs fresh on every reference.
CREATE OR REPLACE VIEW dept_summary AS
SELECT department, count(*) AS headcount, round(avg(salary), 2) AS avg_salary
FROM employee
GROUP BY department;
SELECT * FROM dept_summary;          -- query it like a read-only table

-- Simple single-table views are automatically updatable; WITH CHECK OPTION enforces the
-- view's WHERE on writes (so you can't insert/update a row that would fall outside the view):
CREATE VIEW eng AS
SELECT id, name, salary FROM employee WHERE department = 'Engineering'
WITH CHECK OPTION;

-- A MATERIALIZED VIEW stores the RESULT physically — fast to read, stale until refreshed.
CREATE MATERIALIZED VIEW dept_report AS
SELECT department, count(*) AS headcount, sum(salary) AS payroll
FROM employee GROUP BY department
WITH DATA;                            -- populate immediately (WITH NO DATA = create empty)

-- Refresh strategies:
REFRESH MATERIALIZED VIEW dept_report;             -- recompute; takes an EXCLUSIVE lock (blocks reads)
-- CONCURRENTLY avoids blocking readers, but REQUIRES a UNIQUE index on the matview:
CREATE UNIQUE INDEX ON dept_report (department);
REFRESH MATERIALIZED VIEW CONCURRENTLY dept_report;  -- readers keep seeing old data during refresh

-- You can index a materialized view like a real table (you can't index a plain view — index its base tables):
CREATE INDEX idx_dept_report_payroll ON dept_report (payroll DESC);
```

| | View | Materialized View |
|---|---|---|
| Storage | none (query only) | stores result rows |
| Freshness | always live | stale until `REFRESH` |
| Read speed | depends on the underlying query | fast (precomputed) |
| Indexable | no (index the base tables) | yes |
| Use when | abstraction, reuse, a security boundary | expensive aggregates read far more than they change |

> **Refresh automation:** there is no built-in auto-refresh — *you* decide the cadence based on how stale the data may acceptably be. Schedule `REFRESH MATERIALIZED VIEW CONCURRENTLY` with the `pg_cron` extension (in-database scheduler) or an external scheduler. Always use `CONCURRENTLY` in production so reporting reads aren't blocked during the refresh.

---

## 16. Performance Tuning, VACUUM & Partitioning

**[Advanced]**

### VACUUM, ANALYZE & autovacuum — the consequence of MVCC

Because MVCC (§13) leaves dead row versions behind on every update and delete, PostgreSQL needs a garbage-collection process, and that's **`VACUUM`**. Plain `VACUUM` reclaims dead-tuple space *for reuse by the same table* (the file doesn't shrink, but new rows fill the freed space) and updates the visibility map that enables index-only scans. `ANALYZE` refreshes the planner statistics. **`autovacuum`** runs both automatically in the background based on how many rows have changed — and for the most part you should let it, while understanding it well enough to tune it for hot tables and to recognize when it's falling behind. The one form to use sparingly is `VACUUM FULL`: it actually rewrites the table and returns space to the OS, but takes an `ACCESS EXCLUSIVE` lock that blocks everything — never run it casually on a live table; use `pg_repack` for online compaction instead.

```sql
VACUUM employee;                      -- reclaim dead tuples for reuse (space stays in the file)
VACUUM (VERBOSE, ANALYZE) employee;   -- vacuum + refresh stats + print a report
VACUUM FULL employee;                 -- REWRITE the table, return space to the OS — but locks EVERYTHING
ANALYZE employee;                     -- statistics only

-- Inspect dead tuples and when each table was last (auto)vacuumed/analyzed:
SELECT relname, n_live_tup, n_dead_tup,
       last_vacuum, last_autovacuum, last_analyze, last_autoanalyze
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;

-- Tune autovacuum to be MORE aggressive on a hot, frequently-updated table:
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor  = 0.02,   -- vacuum once 2% of rows are dead (default 0.2 = 20%)
  autovacuum_analyze_scale_factor = 0.01
);
```

> **Bloat:** tables and indexes with many dead tuples grow larger than the live data warrants, which slows scans and wastes disk. Symptoms: steadily growing size, slowing queries. Remedies: make sure autovacuum keeps up (tune scale factors and `autovacuum_max_workers`), `REINDEX CONCURRENTLY` bloated indexes, or use `pg_repack` to rewrite a bloated table *online* (vs the fully-locking `VACUUM FULL`).

⚡ **Version note (PG17):** VACUUM gained a new memory-efficient dead-tuple store (the **TID store**), using far less memory and running faster on large tables. PG18 continues with parallelism and async-I/O improvements. The syntax doesn't change — you just get faster vacuums.

> **Transaction ID wraparound — the classic emergency:** Postgres uses a 32-bit transaction id counter, and `VACUUM` must "freeze" old rows before that counter wraps around, or the server will *stop accepting writes* to protect your data. Healthy autovacuum handles this invisibly, but a misconfigured system that lets autovacuum fall behind can hit it. Monitor `SELECT datname, age(datfrozenxid) FROM pg_database;` — if `age` climbs toward ~2 billion, vacuuming is behind. This is the single most famous cause of avoidable Postgres outages, so put it on your monitoring dashboard.

### Key configuration parameters

These are the handful of settings that actually move the needle. The mental model: `shared_buffers` is Postgres's own cache of table/index pages; `effective_cache_size` is a *hint* (no allocation) telling the planner how much total cache (PG + OS) exists, which nudges it toward index scans; `work_mem` is memory per *sort/hash operation* and is the most dangerous to set high (see the warning); `random_page_cost` should drop to ~1.1 on SSD/NVMe because random reads are no longer dramatically slower than sequential.

| Parameter | What it does | Typical starting point |
|---|---|---|
| `shared_buffers` | PG's own page cache | 25% of RAM |
| `effective_cache_size` | planner's estimate of OS+PG cache (no allocation) | 50–75% of RAM |
| `work_mem` | memory per sort/hash *operation* (many per query possible!) | 16–64MB; raise locally for analytics |
| `maintenance_work_mem` | memory for VACUUM, CREATE INDEX, etc. | 256MB–1GB |
| `max_connections` | connection cap | keep modest; use a pooler |
| `wal_compression` | compress WAL records | `on` often helps |
| `random_page_cost` | cost of a random page read | 1.1 on SSD/NVMe (default 4.0) |
| `max_wal_size` | how much WAL before a checkpoint | larger = fewer, bigger checkpoints |

```sql
SHOW shared_buffers;
ALTER SYSTEM SET work_mem = '64MB';   -- writes to postgresql.auto.conf (persists across restarts)
SELECT pg_reload_conf();              -- reload config (some params still need a full restart)
SELECT name, setting, pending_restart FROM pg_settings WHERE name = 'shared_buffers';
```

> **`work_mem` is per-operation, not per-query or per-connection.** A single complex query may run several sorts/hashes simultaneously, each allowed up to `work_mem`, across every concurrent connection. Worst case ≈ `work_mem × operations_per_query × connections`. Set it conservatively at the global level and raise it *locally* for known-heavy analytics queries with `SET LOCAL work_mem = '256MB';` inside a transaction.

### Connection pooling (PgBouncer)

Each PostgreSQL connection is a full OS process consuming several megabytes; thousands of direct app connections will exhaust memory and degrade everything. The standard fix is a **connection pooler** like **PgBouncer**, which sits in front of Postgres and multiplexes a large number of client connections onto a small pool of real server connections. In `transaction` pooling mode (the most common), a server connection is assigned to a client only for the duration of a transaction, maximizing reuse — at the cost that you can't rely on session-level state (session-scoped advisory locks, some prepared-statement patterns, `SET`) persisting between transactions.

```ini
; pgbouncer.ini
[databases]
appdb = host=127.0.0.1 port=5432 dbname=appdb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256          ; use SCRAM, never trust/md5 (security, §19)
auth_file = /etc/pgbouncer/userlist.txt
; transaction pooling: a server conn is held only for the duration of a transaction.
; -> best concurrency, but DON'T use session features (session advisory locks, SET, etc.)
pool_mode = transaction
default_pool_size = 20
max_client_conn = 1000
```

> Apps then connect to PgBouncer on 6432 instead of Postgres on 5432. With transaction pooling, modern drivers handle server-side prepared statements correctly (`pgx`, recent `node-postgres`); **Prisma** needs `?pgbouncer=true` in the URL (→ see PRISMA_ORM_GUIDE.md). Cloud-hosted Postgres (e.g. Supabase, → see SUPABASE_GUIDE.md) typically bundles a pooler endpoint you connect to instead.

### Partitioning (range / list / hash)

Partitioning splits one logical table into many physical child tables under the hood, transparent to queries. The payoffs: **partition pruning** (a query with a key predicate scans only the relevant partitions, not the whole dataset), **instant bulk deletion** (drop a whole partition instead of a massive `DELETE`+`VACUUM`), and per-partition maintenance (vacuum/index one month at a time). It is a tool for *large* tables (millions of rows and up) — for small tables it adds complexity with no benefit. The three strategies match the data: **RANGE** for ordered/continuous keys (time-series by month is the classic), **LIST** for discrete categories (by region), **HASH** for even distribution when there's no natural range or list.

```sql
-- RANGE partitioning by month (the classic for time-series / event data):
CREATE TABLE measurement (
  id       bigint GENERATED ALWAYS AS IDENTITY,
  taken_at timestamptz NOT NULL,
  device   int NOT NULL,
  value    numeric NOT NULL,
  PRIMARY KEY (id, taken_at)        -- the partition key MUST be part of any PK/unique constraint
) PARTITION BY RANGE (taken_at);

-- Child partitions hold the actual rows:
CREATE TABLE measurement_2026_06 PARTITION OF measurement
  FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
CREATE TABLE measurement_2026_07 PARTITION OF measurement
  FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');
CREATE TABLE measurement_default PARTITION OF measurement DEFAULT;  -- catch-all for out-of-range rows

-- Inserts route to the right partition automatically; queries PRUNE to the relevant ones:
EXPLAIN SELECT * FROM measurement WHERE taken_at >= '2026-07-15';  -- scans only _2026_07

-- LIST partitioning (by discrete values, e.g. region):
CREATE TABLE sales (region text, amount numeric) PARTITION BY LIST (region);
CREATE TABLE sales_eu PARTITION OF sales FOR VALUES IN ('DE','FR','ES');
CREATE TABLE sales_us PARTITION OF sales FOR VALUES IN ('US','CA');

-- HASH partitioning (even spread when there's no natural range/list):
CREATE TABLE big_users (id bigint, name text) PARTITION BY HASH (id);
CREATE TABLE big_users_0 PARTITION OF big_users FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE big_users_1 PARTITION OF big_users FOR VALUES WITH (MODULUS 4, REMAINDER 1);
-- ... remainders 2 and 3 ...

-- Drop old data INSTANTLY by detaching/dropping a partition (no huge DELETE, no bloat):
ALTER TABLE measurement DETACH PARTITION measurement_2026_06 CONCURRENTLY;
DROP TABLE measurement_2026_06;
```

> **Partitioning tips:** partition only genuinely large tables; keep the partition count reasonable (hundreds, not tens of thousands — too many partitions slows planning); include the partition key in unique constraints and in most `WHERE` clauses so pruning actually fires; automate partition creation and retention with the `pg_partman` extension rather than by hand.

---

## 17. Backup & Restore (incl. PITR & Incremental)

**[Advanced]**

A database without tested backups is a disaster waiting to happen, and Postgres gives you two complementary strategies that solve different problems. **Logical backups** (`pg_dump`) export the data as SQL or a portable archive — they're version- and architecture-independent, support selective restore (one table, one schema), and are ideal for moving data, cloning a database, or keeping a portable snapshot. **Physical backups** (`pg_basebackup`) copy the actual data files byte-for-byte — they're the basis for full-cluster recovery, streaming replication, and **Point-in-Time Recovery (PITR)**, but require the same major version and architecture to restore. The decisive operational truth to absorb now: **an untested backup is not a backup.** Schedule regular *restore drills*, because the only way to know a backup works is to restore it.

### Logical backups: pg_dump / pg_restore

```bash
# Plain SQL dump (human-readable; restore by piping to psql):
pg_dump -h localhost -U app -d appdb -f appdb.sql

# Custom format (compressed; enables SELECTIVE and PARALLEL restore — RECOMMENDED default):
pg_dump -h localhost -U app -d appdb -Fc -f appdb.dump

# Directory format enables a PARALLEL dump (-j) — fastest for big databases:
pg_dump -d appdb -Fd -j 4 -f appdb_dir

# Dump only schema, only data, or specific tables:
pg_dump -d appdb --schema-only -f schema.sql
pg_dump -d appdb --data-only   -f data.sql
pg_dump -d appdb -t public.employee -t public.orders -Fc -f two_tables.dump

# Dump ALL databases plus roles/tablespaces (cluster-wide; pg_dump alone misses roles):
pg_dumpall -h localhost -U postgres -f cluster.sql

# Restore:
psql -d newdb -f appdb.sql                          # for plain-SQL dumps
createdb newdb
pg_restore -d newdb -Fc appdb.dump                  # custom/dir formats
pg_restore -d newdb -j 4 appdb_dir                  # parallel restore
pg_restore -d newdb -t employee appdb.dump          # restore just one table
pg_restore -d newdb --clean --if-exists appdb.dump  # drop existing objects first
```

> **Security note:** dump files contain all your data in the clear — treat them as the sensitive assets they are. Encrypt them at rest, restrict who can read the backup location, and never commit a dump to version control. Roles' passwords are *not* included in a plain dump (they live in `pg_authid`, captured by `pg_dumpall --roles-only`), so a full DR plan needs both.

### Physical backups: pg_basebackup

A physical base backup is an exact copy of the cluster's data directory, taken while the server runs. It's the foundation for both replication (§18) and PITR.

```bash
# Take a base backup of the whole cluster:
pg_basebackup -h localhost -U replicator -D /backup/base -Ft -z -P
#  -D dir : output directory   -Ft : tar format   -z : gzip   -P : show progress
```

### Point-in-Time Recovery (PITR)

PITR is the answer to "someone ran a bad `DELETE` an hour ago — restore to the moment *before* it." It works by combining a base backup with **continuous WAL archiving**: the Write-Ahead Log records every change, and if you archive every completed WAL segment, you can restore the base backup and then *replay* the WAL forward to any chosen instant. The cost is operational discipline (you must reliably archive WAL and monitor that archiving keeps up), but the payoff is the ability to recover to any second, not just to your last full backup.

```ini
# postgresql.conf — enable WAL archiving (the prerequisite for PITR):
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'   # archive each completed WAL segment
```

```bash
# Recovery procedure (conceptual):
# 1) Restore the base backup into a fresh data directory.
# 2) The archived WAL segments accumulate in /archive.
# 3) Configure recovery (below), drop a 'recovery.signal' file, and start Postgres —
#    it replays WAL up to your target time, then promotes.
```

```ini
# postgresql.conf on the restore target (recovery settings live here since PG12):
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2026-06-21 14:25:00+00'   # stop replaying WAL at this exact instant
recovery_target_action = 'promote'                 # become a normal primary when the target is reached
# Then create an empty file named recovery.signal in the data dir and start the server.
```

### Incremental backup (PG17)

⚡ **Version note (PG17):** PostgreSQL 17 added **native incremental physical backups**. With WAL summarization enabled, `pg_basebackup --incremental` captures only the data blocks changed since a prior backup (dramatically smaller and faster than a full base backup), and `pg_combinebackup` reconstructs a complete backup from the chain (full + incrementals) when you need to restore.

```ini
# postgresql.conf — enable change tracking that incrementals depend on:
summarize_wal = on
```

```bash
# 1) A full base backup:
pg_basebackup -D /backup/full -c fast

# 2) Later, an incremental relative to the full backup's manifest (only changed blocks):
pg_basebackup -D /backup/incr1 --incremental=/backup/full/backup_manifest

# 3) Reconstruct a restorable full backup by combining the chain:
pg_combinebackup /backup/full /backup/incr1 -o /backup/restored
```

> **Backup best practices:** automate backups; **test restores regularly** (an untested backup is fiction); keep copies off-site (the 3-2-1 rule: 3 copies, 2 media, 1 off-site); monitor that WAL archiving isn't lagging; and write down your **RPO** (how much data you can afford to lose) and **RTO** (how fast you must be back). For production, use a backup manager — **pgBackRest** or **Barman** — which wrap base backups, WAL, PITR, and incrementals with retention policies, compression, parallelism, and verification.

---

## 18. Replication & High Availability

**[Advanced]**

Replication keeps one or more copies of your database on other servers, for two distinct goals that are easy to conflate: **high availability** (if the primary dies, promote a replica and keep serving — minimizing downtime) and **read scaling** (offload read-heavy traffic like reports to replicas). PostgreSQL provides the replication *primitives*; turning them into automatic failover requires external orchestration (Patroni and friends, below). There are two replication mechanisms with different strengths: **physical/streaming** replication ships the raw WAL to byte-identical standbys (whole-cluster, same version), while **logical** replication publishes row-level changes for selected tables (flexible, cross-version, subset-able).

### Streaming (physical) replication

A **primary** continuously streams its WAL to one or more **standby** servers, which replay it to stay byte-identical. Standbys can serve read-only queries and can be promoted to primary on failover. Use a **replication slot** so the primary retains the WAL a lagging standby still needs (rather than guessing with `wal_keep_size`).

```ini
# On the PRIMARY (postgresql.conf):
wal_level = replica
max_wal_senders = 10
wal_keep_size = '1GB'         # retain some WAL so a briefly-disconnected standby can catch up
# (better: a replication slot retains WAL until the standby has consumed it)
```

```sql
-- Create a replication role and a physical slot on the PRIMARY:
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'secret';
SELECT pg_create_physical_replication_slot('standby1');  -- prevents needed WAL from being recycled
```

```bash
# On the STANDBY: clone the primary and configure it to follow:
pg_basebackup -h primary_host -U replicator -D /var/lib/postgresql/data -R -P -Xs
#  -R writes the connection settings and creates 'standby.signal' so it starts as a replica.
```

```sql
-- Monitor replication FROM THE PRIMARY:
SELECT client_addr, state, sent_lsn, replay_lsn,
       pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS replay_lag
FROM pg_stat_replication;

-- ON THE STANDBY, check recovery status and lag:
SELECT pg_is_in_recovery();                         -- true on a standby
SELECT now() - pg_last_xact_replay_timestamp() AS lag;
```

The synchronous-vs-asynchronous choice is the durability dial:

- **Asynchronous** (default): the primary commits without waiting for any standby. Fast, but a primary crash can lose the last few transactions that hadn't reached the standby yet.
- **Synchronous** (set `synchronous_standby_names`): the primary waits for standby confirmation before reporting commit. Zero data loss on failover, at the cost of higher commit latency.

### Logical replication

Logical replication operates at the **row level**: you `CREATE PUBLICATION` for chosen tables on the source and `CREATE SUBSCRIPTION` on the target, which copies the initial data and then streams ongoing changes. Because it's logical (not raw WAL), it works **across major versions**, can replicate a **subset** of tables, and supports **many-to-one consolidation** — making it the tool for zero-downtime major-version upgrades and selective data distribution.

```sql
-- On the SOURCE (publisher): requires wal_level = logical.
CREATE PUBLICATION mypub FOR TABLE employee, orders;
-- or:  CREATE PUBLICATION mypub FOR ALL TABLES;

-- On the TARGET (subscriber):
CREATE SUBSCRIPTION mysub
  CONNECTION 'host=source_host dbname=appdb user=replicator password=secret'
  PUBLICATION mypub;
-- The subscriber copies existing data, then streams subsequent changes continuously.
```

⚡ **Version note (PG17):** logical replication gained **failover slots** (slot state can sync to physical standbys, so logical replication survives a failover instead of breaking) and the ability to `pg_upgrade` a subscriber without losing replication state — major operational wins. PG18 further improves logical replication of sequences and conflict handling.

### High availability overview

PostgreSQL gives you replication; it does **not** elect a new primary by itself. **Automatic failover** — detecting that the primary is dead and promoting a standby without split-brain — is the job of external tooling:

- **Patroni** (backed by a consensus store: etcd/Consul/ZooKeeper) — the most widely-used open-source HA orchestrator.
- **repmgr**, **pg_auto_failover** — alternatives.
- A **load balancer / virtual IP / PgBouncer** layer routes clients to whichever node is currently primary, and spreads reads across replicas.

> **Replication lag and "read-your-own-writes":** an asynchronous read replica may not yet contain a row your app just wrote to the primary, so reading it back from a replica returns stale/missing data — a subtle, confusing bug. Mitigations: route writes and any immediate read-after-write to the primary, or use synchronous replication for the critical paths that must be consistent. Always design with lag in mind when you add read replicas.

---

## 19. Security — Roles, Privileges, RLS, SSL

**[Advanced]**

Database security rests on **least privilege**: every role should have exactly the permissions it needs and nothing more, so that a compromised application account or a SQL-injection foothold can do limited damage. This chapter is where the role model from §1, the parameterization rule from §21, and the constraint enforcement from §3 come together into a defense-in-depth posture. The layers are: *authentication* (who may connect, proven how — `pg_hba.conf` + SCRAM + TLS), *authorization* (what each role may do to which objects — `GRANT`/`REVOKE`), *row-level* control (which rows a role sees — RLS), and *encryption* (TLS in transit). Treat them as cumulative, not alternatives.

### Privileges: GRANT / REVOKE

PostgreSQL privileges are granted per object (database, schema, table, column, sequence, function). The practical pattern is to grant broad privilege *sets* to group roles and make login roles members of those groups, then to grant the *minimum* each application needs. Note the two-step nature of access: a role needs `CONNECT` on the database, `USAGE` on the schema, *and* the relevant privilege on the table — missing any one denies access.

```sql
-- Database & schema level (the outer gates):
GRANT CONNECT ON DATABASE appdb TO readonly;
GRANT USAGE ON SCHEMA public TO readonly;          -- needed to "see into" the schema at all

-- Table privileges:
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
GRANT SELECT, INSERT, UPDATE, DELETE ON employee TO app_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_user;  -- required for IDENTITY/serial inserts

-- COLUMN-level privileges (e.g. let a role read names but NOT salaries):
GRANT SELECT (id, name) ON employee TO readonly;

-- REVOKE takes privileges away:
REVOKE INSERT ON employee FROM app_user;

-- DEFAULT PRIVILEGES auto-grant on FUTURE objects (so new tables aren't accidentally inaccessible):
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT ON TABLES TO readonly;

-- Inspect who can do what:
\dp employee
SELECT grantee, privilege_type FROM information_schema.role_table_grants
WHERE table_name = 'employee';
```

> **The PUBLIC pitfall:** the pseudo-role `PUBLIC` means *every* role. Historically `PUBLIC` had `CONNECT` on new databases and `CREATE`/`USAGE` on the `public` schema, which is too open for a secure baseline. Since PG15 the `public` schema no longer grants `CREATE` to `PUBLIC` by default, but you should still harden: `REVOKE ALL ON DATABASE appdb FROM PUBLIC;` and lock down `public`. Grant connection and access deliberately, not by default.

> **Never run your application as a superuser.** A superuser bypasses all permission checks *and* RLS — if that connection is compromised, the attacker owns the cluster. Create a narrow `app_user` with only the table privileges it needs, and reserve superuser for migrations/administration.

### Row-Level Security (RLS)

RLS pushes per-row access control *into the database itself*: with RLS enabled on a table, a role sees and modifies only the rows that satisfy the policies you define — enforced no matter what query the application sends. This is the robust foundation for **multi-tenancy** (tenant A can never read tenant B's rows even if the app has a bug) and per-user data isolation. A policy's `USING` clause filters which rows are *visible* to reads/updates/deletes; its `WITH CHECK` clause constrains which rows may be *written* (so a user can't insert a row owned by someone else).

```sql
CREATE TABLE document (
  id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  owner   text NOT NULL,        -- which user owns this row
  content text NOT NULL
);

-- 1) Enable RLS (until you do, policies are inert):
ALTER TABLE document ENABLE ROW LEVEL SECURITY;
-- FORCE makes even the table OWNER subject to policies (owners bypass RLS otherwise):
ALTER TABLE document FORCE ROW LEVEL SECURITY;

-- 2) A policy: a user may see/modify ONLY their own rows.
CREATE POLICY owner_isolation ON document
  USING (owner = current_user)            -- read/update/delete see only matching rows
  WITH CHECK (owner = current_user);      -- writes must satisfy this (can't insert for others)

-- Per-command policies are possible too:
CREATE POLICY read_all  ON document FOR SELECT USING (true);
CREATE POLICY write_own ON document FOR INSERT WITH CHECK (owner = current_user);

-- Multi-tenant pattern driven by a per-request session variable the app sets:
-- the app runs:  SET app.tenant_id = '42';   (or SELECT set_config('app.tenant_id','42',true))
CREATE POLICY tenant_isolation ON document
  USING (owner = current_setting('app.tenant_id', true));
```

> **RLS gotchas:** the table *owner* and *superusers* bypass RLS unless you add `FORCE ROW LEVEL SECURITY` — so the very account you test with might not be subject to the policy, hiding mistakes. Also, every policy adds a filter to every query against the table, so keep policy expressions indexable (e.g. index `tenant_id`/`owner`) or they become a performance drag. Test isolation by connecting *as* a restricted role, not as a superuser.

### Authentication & SSL

`pg_hba.conf` is the front door: each line says "for this connection *type*, *database*, *user*, and source *address*, require this *method*." Rules are matched top-to-bottom, first match wins, so order them from specific to general and end with a `reject`. Always use **`scram-sha-256`** (modern salted password auth), never `trust` (no password!) or the obsolete `md5`. For any connection crossing a network, require **TLS** so credentials and data can't be sniffed, and have clients use `sslmode=verify-full` so they also verify the server's certificate and hostname (preventing man-in-the-middle).

```text
# pg_hba.conf — TYPE  DATABASE  USER     ADDRESS          METHOD   (first match wins, top to bottom)
local   all       all                       scram-sha-256   # unix socket connections
host    appdb     app_user 10.0.0.0/8       scram-sha-256   # password (hashed) from the private net
hostssl appdb     app_user 0.0.0.0/0        scram-sha-256   # require SSL for this rule
host    all       all      0.0.0.0/0        reject          # deny everything else explicitly
```

```ini
# postgresql.conf — enable TLS on the server:
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file  = 'server.key'
```

```bash
# Clients enforce TLS via sslmode. verify-full = encrypted + server cert + hostname verified:
psql "postgresql://app@host/appdb?sslmode=verify-full&sslrootcert=ca.crt"
```

> **Security checklist:** use `scram-sha-256` (never `trust`/`md5`); require SSL (`hostssl` + client `sslmode=verify-full`) for all remote connections; apply least privilege and never connect apps as superuser; rotate credentials and set `VALID UNTIL`; keep Postgres patched; restrict `listen_addresses` to the interfaces you actually serve; parameterize every query to kill SQL injection (§21); and for compliance, enable audit logging with the `pgaudit` extension. Defense in depth means each of these is a layer the attacker must defeat.

---

## 20. Extensions — pgvector, PostGIS, pg_trgm & more

**[Advanced]**

Extensions are the mechanism behind Postgres's famous extensibility: a packaged bundle of types, functions, operators, and even index methods that you load into a database with `CREATE EXTENSION`. This is *the* reason Postgres keeps absorbing capabilities that used to demand separate systems — instead of bolting on a vector database, a geospatial database, and a fuzzy-search engine, you add three extensions and keep everything (transactions, joins, backups, security) in one place. Extensions are installed per-database, and which ones are *available* depends on what's installed on the server (the `contrib` package and third-party packages).

```sql
SELECT * FROM pg_available_extensions ORDER BY name;   -- what you can install on this server
\dx                                                    -- what is installed in THIS database
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

### pgvector — vector similarity search (AI / RAG, central in 2026)

`pgvector` adds a `vector` type and nearest-neighbour search, turning Postgres into a vector database for embeddings — the backbone of semantic search and retrieval-augmented generation (RAG). The compelling reason to use it over a dedicated vector store: your embeddings live *next to* your relational data, so you can combine semantic similarity with ordinary SQL filters in one query (find the most similar document chunks *that also* belong to this tenant and were created this year) — a "hybrid search" that's awkward when vectors and metadata live in separate systems. For large tables you add an approximate-nearest-neighbour (ANN) index: **HNSW** (best recall/speed, more memory/build time) or **IVFFlat** (faster to build, needs its `lists` parameter tuned).

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE chunk (
  id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  content   text NOT NULL,
  embedding vector(1536)        -- dimension must match your embedding model (e.g. 1536)
);

-- Distance operators:  <-> L2 (Euclidean),  <#> negative inner product,  <=> cosine distance.
-- Find the 5 chunks most similar to a query embedding (ORDER BY the distance = nearest neighbours):
SELECT id, content, embedding <=> '[0.01, 0.02, ...]' AS cosine_distance
FROM chunk
ORDER BY embedding <=> '[0.01, 0.02, ...]'
LIMIT 5;

-- ANN indexes for speed on large tables:
-- HNSW (best recall/latency, more memory):
CREATE INDEX ON chunk USING hnsw (embedding vector_cosine_ops);
-- IVFFlat (faster build; tune 'lists' ~ rows/1000; ANALYZE the table first):
CREATE INDEX ON chunk USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

-- HYBRID search — the Postgres superpower: vector similarity + a relational filter, together:
SELECT content FROM chunk
WHERE content ILIKE '%postgres%'                 -- relational/keyword filter
ORDER BY embedding <=> '[...]' LIMIT 5;          -- semantic ranking
```

> **Security note:** embeddings can encode sensitive source text; protect the `chunk` table with the same least-privilege and (if multi-tenant) RLS policies as any other data — a vector column is not magically anonymous.

### PostGIS — geospatial

`PostGIS` is the gold-standard geospatial extension: geometry/geography types, hundreds of spatial functions, and GiST spatial indexing. Use it for "what's near here," routing, regions, and any map-backed feature.

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE TABLE place (
  id   bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name text,
  geom geography(Point, 4326)     -- WGS84 longitude/latitude
);
INSERT INTO place (name, geom) VALUES ('HQ', 'SRID=4326;POINT(-0.1276 51.5072)');
CREATE INDEX idx_place_geom ON place USING gist (geom);   -- spatial index
-- Everything within 1 km of a point:
SELECT name FROM place
WHERE ST_DWithin(geom, 'SRID=4326;POINT(-0.13 51.51)'::geography, 1000);
```

### pg_trgm — fuzzy / substring search

`pg_trgm` indexes 3-character sequences (trigrams), which makes `ILIKE '%term%'` and similarity matching fast — the right tool for autocomplete, typo tolerance, and short-string fuzzy search (where full-text search, §10, is overkill or wrong).

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
-- A trigram GIN index accelerates ILIKE '%...%' and similarity():
CREATE INDEX idx_emp_name_trgm ON employee USING gin (name gin_trgm_ops);
SELECT name, similarity(name, 'adya') AS sim
FROM employee WHERE name % 'adya'        -- '%' = "similar enough" (typo-tolerant match)
ORDER BY sim DESC;
SELECT * FROM employee WHERE name ILIKE '%ace%';   -- now index-assisted, not a full scan
```

### Other commonly-used extensions

```sql
-- uuid-ossp: extra UUID generators (gen_random_uuid is built-in now, so often unnecessary).
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
SELECT uuid_generate_v4();

-- pgcrypto: hashing & encryption. Use it for password hashing if you hash in the DB:
CREATE EXTENSION IF NOT EXISTS pgcrypto;
SELECT crypt('mypassword', gen_salt('bf'));      -- bcrypt password hash (salted)
SELECT digest('hello', 'sha256');                -- SHA-256 digest

-- citext: case-INsensitive text (clean way to make emails/usernames case-insensitive & unique):
CREATE EXTENSION IF NOT EXISTS citext;
CREATE TABLE account2 (email citext UNIQUE);     -- 'A@x.com' and 'a@x.com' now collide

-- hstore: simple flat key/value string store (predates JSONB; JSONB is usually preferred now).
CREATE EXTENSION IF NOT EXISTS hstore;
SELECT 'a=>1, b=>2'::hstore -> 'a';              -- '1'

-- Other notables: pg_stat_statements (§12), btree_gist/btree_gin (mixed exclusion constraints),
-- pg_cron (in-DB job scheduler), pg_partman (partition automation), postgres_fdw (query remote PGs).
```

> **Security note on `pgcrypto` for passwords:** `crypt()` with `gen_salt('bf')` is a legitimate bcrypt, but hashing passwords *in the database* means the plaintext travels to the server and may appear in logs. Many teams prefer to hash in the application with Argon2id (→ see GO_JWT_ARGON2_GUIDE.md for the Go approach). Whichever you choose, never store plaintext and always use a slow, salted KDF.

---

## 21. Connecting from Apps — Node.js & Go

**[Advanced]**

This is where the database meets your application code, and it has one non-negotiable rule that overrides everything else: **never build SQL by string concatenation — always use parameterized queries.** The reasoning is both security and performance. Security: when you send the SQL text with placeholders (`$1`, `$2`) and pass the values *separately*, the driver and server treat the values as pure data that can never be interpreted as SQL — this *structurally eliminates* SQL injection, the most damaging and common database vulnerability, where input like `'; DROP TABLE users; --` would otherwise become executable. Performance: the server can recognize and cache the plan for a repeated parameterized query. There is no scenario where gluing user input into a query string is acceptable; if you need a *dynamic identifier* (a column or table name, which can't be a parameter), validate it against an allow-list of known-good names — never against raw user input.

A second universal rule: **use a connection pool, created once per process, never per request.** Opening a Postgres connection is expensive (it forks an OS process server-side, §16), so the pool keeps a set of warm connections and hands them out — but you must return each connection to the pool when done, and run all statements of a transaction on the *same* dedicated connection.

### Node.js with `pg` (node-postgres)

```js
// npm install pg
import pg from "pg";
const { Pool } = pg;

// A Pool reuses connections — create ONE per process, share it everywhere, never per request.
const pool = new Pool({
  connectionString: process.env.DATABASE_URL, // postgres://user:pass@host:5432/db
  max: 20,                 // max connections this pool will open
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,
  // ssl: { rejectUnauthorized: true, ca: fs.readFileSync("ca.crt") }, // enforce TLS in prod
});

// Parameterized query: $1 is a placeholder; the value is passed separately and is INJECTION-SAFE.
async function findByEmail(email) {
  const { rows } = await pool.query(
    "SELECT id, name FROM customer WHERE email = $1", // NEVER: `... = '${email}'`
    [email]
  );
  return rows[0] ?? null;
}

// Transaction: you MUST use a single dedicated client so every statement runs on one connection.
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
    await client.query("ROLLBACK"); // undo everything on any error
    throw err;
  } finally {
    client.release();               // ALWAYS return the client to the pool, success or failure
  }
}

// Graceful shutdown so in-flight work finishes and sockets close cleanly:
process.on("SIGTERM", () => pool.end());
```

> **Prisma note (→ see PRISMA_ORM_GUIDE.md):** Prisma is the higher-level, type-safe ORM most TypeScript teams use on top of a driver. It auto-parameterizes *everything* (so it's injection-safe by construction), manages its own pool, generates fully-typed query methods from your schema, and exposes `$queryRaw\`...\`` (a tagged template, which is parameterized) for raw SQL when you need it. Behind a PgBouncer in transaction mode, add `?pgbouncer=true` to the URL. Choose `pg` directly when you want full hand-written SQL control; choose Prisma for productivity, migrations, and end-to-end types. (For Go, the ORM analog is Ent — → see GO_ENT_ORM_GUIDE.md.)

### Go with `pgx` v5 (the modern driver)

The Go examples use `pgx` v5 with `pgxpool`; the language itself is covered in → see GO_GUIDE.md. The same two rules apply — parameterize, and share one pool. `pgx`'s `BeginFunc` helper is the idiomatic transaction pattern: it commits if your callback returns `nil` and automatically rolls back on any error or panic, so you can't forget to clean up.

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

	// pgxpool is the connection pool — create ONE and share it across the program.
	pool, err := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Fatal(err)
	}
	defer pool.Close()

	// Parameterized query ($1, $2). pgx sends args separately => INJECTION-SAFE.
	var id int64
	var name string
	err = pool.QueryRow(ctx,
		"SELECT id, name FROM customer WHERE email = $1", // NEVER concatenate the value into the SQL
		"ada@example.com",
	).Scan(&id, &name)
	if errors.Is(err, pgx.ErrNoRows) {
		log.Println("not found")
	} else if err != nil {
		log.Fatal(err)
	}
	log.Printf("found %d %s", id, name)

	// Query many rows; pgx v5 has CollectRows helpers, but the manual loop shows the pattern:
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
	if err := rows.Err(); err != nil { // ALWAYS check rows.Err() after the loop
		log.Fatal(err)
	}

	// Transaction via the helper that auto-rolls-back on error or panic:
	err = pgx.BeginFunc(ctx, pool, func(tx pgx.Tx) error {
		if _, err := tx.Exec(ctx,
			"UPDATE account SET balance = balance - $1 WHERE id = $2", 100, 1); err != nil {
			return err // returning a non-nil error triggers an automatic ROLLBACK
		}
		if _, err := tx.Exec(ctx,
			"UPDATE account SET balance = balance + $1 WHERE id = $2", 100, 2); err != nil {
			return err
		}
		return nil // returning nil triggers COMMIT
	})
	if err != nil {
		log.Fatal(err)
	}
}
```

> **Driver notes:** `pgx` v5 is the modern, actively-developed Go driver — use `pgxpool` for pooling. It can also expose the standard `database/sql` interface via its `stdlib` package if a library requires that. `pgx` surfaces Postgres-specific power the generic `lib/pq` (now in maintenance mode) doesn't: the COPY protocol via `pool.CopyFrom(...)` for fast bulk loads (far faster than looping `INSERT`), `LISTEN`/`NOTIFY`, and rich type mapping for arrays/JSONB/ranges.

> **SQL injection — the one rule, restated:** in *every* language the pattern is identical — send the query text with placeholders, pass the values as a separate argument. Never `"... WHERE name = '" + userInput + "'"`. The retry loop for `SERIALIZABLE` transactions (§13) also lives here in app code: catch SQLSTATE `40001`, and re-run the whole transaction a few times before giving up.

---
