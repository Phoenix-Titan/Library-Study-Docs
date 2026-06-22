# SQLite3 — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Developers who want to *truly* understand SQLite — the most widely deployed database engine on Earth — from `CREATE TABLE` to WAL-mode concurrency, type affinity, FTS5, JSON, and embedding it inside Node.js and Go services. You do not need a server, an account, or even an internet connection: SQLite is a single C library and your database is one file. Every concept is taught in prose first (what it is, *why* it works that way, when to use it) and then shown in heavily commented, runnable code. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **SQLite 3.x as current in 2026** (the 3.45–3.50 line). Modern features assumed available and covered here include: **STRICT tables** (3.37+, type enforcement), the **`->` / `->>` JSON operators** and the full JSON suite now built in by default (`SQLITE_ENABLE_JSON1` is on in standard builds since 3.38), **FTS5** full-text search, **window functions** (3.25+), **`UPSERT`** (`ON CONFLICT … DO UPDATE`, 3.24+), **`RETURNING`** (3.35+), **generated columns** (3.31+), and **math functions** (3.35+, when built with `SQLITE_ENABLE_MATH_FUNCTIONS`). On the application side, **Node.js now ships a built-in `node:sqlite` module** (stable/maturing through Node 22→24 in 2025–2026) alongside the long-standing **`better-sqlite3`**, and in Go the **pure-Go, cgo-free `modernc.org/sqlite`** driver has become a default choice next to **`mattn/go-sqlite3`**. Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**, so platform notes are called out. Confirm exact APIs against the official docs (`sqlite.org`) and `SELECT sqlite_version();`.
>
> **See also:** For the client/server relational counterpart, cross-reference **`POSTGRESQL_GUIDE.md`** (when to graduate from SQLite to Postgres is discussed in §15). For schema modelling, normalization, keys, and relationships that apply to *any* SQL database, cross-reference **`RELATIONAL_DB_DESIGN_GUIDE.md`**.

---

## Table of Contents

1. [What SQLite Is & When To Use It](#1-what-sqlite-is--when-to-use-it) **[B]**
2. [Getting Started: the CLI & Dot-Commands](#2-getting-started-the-cli--dot-commands) **[B]**
3. [Type Affinity & Dynamic Typing (the big quirk)](#3-type-affinity--dynamic-typing-the-big-quirk) **[B/I]**
4. [CRUD & Schema — CREATE/INSERT/UPDATE/DELETE/SELECT](#4-crud--schema) **[B]**
5. [Querying — Joins, Aggregates, Subqueries, CTEs, Windows](#5-querying) **[B/I]**
6. [Indexing & EXPLAIN QUERY PLAN](#6-indexing--explain-query-plan) **[I]**
7. [Constraints & Foreign Keys](#7-constraints--foreign-keys) **[I]**
8. [Transactions & the Concurrency Model (WAL)](#8-transactions--the-concurrency-model-wal) **[I/A]**
9. [PRAGMAs That Matter](#9-pragmas-that-matter) **[I]**
10. [JSON Support](#10-json-support) **[I]**
11. [Full-Text Search (FTS5)](#11-full-text-search-fts5) **[I/A]**
12. [Other Features — Generated Columns, Views, Triggers, Dates, Math](#12-other-features) **[I]**
13. [Node.js Usage — better-sqlite3 & node:sqlite (Express/NestJS)](#13-nodejs-usage) **[I/A]**
14. [Go Usage — modernc.org/sqlite & mattn/go-sqlite3 (Gin)](#14-go-usage) **[I/A]**
15. [Backup, Maintenance & Operations](#15-backup-maintenance--operations) **[I/A]**
16. [Performance Tuning](#16-performance-tuning) **[A]**
17. [Anti-Patterns & Gotchas](#17-anti-patterns--gotchas) **[A]**
18. [Study Path & Build-to-Learn Projects](#18-study-path--build-to-learn-projects)

---

## 1. What SQLite Is & When To Use It

**[B]**

### 1.1 The one-sentence definition

SQLite is an **embedded, serverless, single-file, zero-configuration, transactional SQL database engine**. Unpack each word, because each one is doing real work:

- **Embedded:** SQLite is not a program you run and connect to. It is a **C library** (a few hundred kilobytes) that you *link into your own application*. When your Node.js or Go process opens a database, the SQLite engine runs **inside that process** — there is no separate database server, no daemon, no network hop. Reading and writing the database is a function call, not a socket round-trip.
- **Serverless:** Because the engine lives in your process, there is **no server process to start, stop, configure, secure, or keep alive**. Contrast this with PostgreSQL or MySQL, where a long-running server process owns the data files and every client talks to it over TCP or a Unix socket. SQLite has no such middleman. (Note: "serverless" here means "no database *server process*". It is unrelated to cloud "serverless functions", though SQLite is popular in those too.)
- **Single-file:** An *entire* SQLite database — all your tables, indexes, triggers, views, and the schema itself — lives in **one ordinary file** on disk (commonly `.db`, `.sqlite`, or `.sqlite3`; the extension is just a convention). You can copy it, email it, version it, or `scp` it like any other file. There is also a small temporary "rollback journal" or "WAL" sidecar file during writes (more in §8).
- **Zero-configuration:** There is nothing to install on a "server", no `postgresql.conf`, no users/roles to create, no listening port to open. You point the library at a filename and you have a database. If the file does not exist, it is created.
- **Transactional (ACID):** Despite its simplicity, SQLite is fully **ACID-compliant** — Atomic, Consistent, Isolated, Durable. A power loss or crash mid-write will not corrupt your database; either a transaction fully committed or it did not. This is *not* a toy guarantee; SQLite's durability is famously rigorous and heavily tested.

### 1.2 How this differs from client/server databases (Postgres, MySQL)

This is the mental model that everything else depends on, so it is worth being explicit.

| Aspect | SQLite | PostgreSQL / MySQL (client-server) |
|---|---|---|
| **Architecture** | Library linked into *your* process | Separate server process; clients connect over the network |
| **"Connection"** | Opening a file handle (a function call) | A TCP/socket session, authentication, a backend process/thread |
| **Where queries run** | In your app's process & thread | In the server, possibly on another machine |
| **Concurrency** | Many concurrent readers; effectively **one writer at a time** for the whole database | Many concurrent readers *and* writers via row-level locking & MVCC |
| **Scaling model** | Scale *up* (faster disk/CPU); one machine | Scale *out*; replicas, connection pools, clustering |
| **Configuration** | Filename + a few PRAGMAs | Users, roles, network, memory, replication, WAL archiving… |
| **Network access** | None built in — it's local to the file | First-class; that's the whole point |
| **Operational overhead** | Essentially zero | A real ongoing job (backups, upgrades, tuning, security) |
| **Data types** | Dynamic *type affinity* (§3) | Static, strict types |
| **Best at** | Embedded/local/edge, single-machine workloads | Multi-user, networked, high-write-concurrency systems |

The single most important consequence: **SQLite has no network protocol and no per-row write concurrency.** It serializes writes at the *database* level. That is a feature, not a bug — it is what lets the whole thing be a single file with no server — but it shapes *where* SQLite is the right tool.

### 1.3 Why it's the most-deployed database in the world

You almost certainly carry thousands of SQLite databases in your pocket right now. SQLite is built into:

- **Every Android and iOS device** — it backs app storage, contacts, messages, settings.
- **Every major web browser** — Chrome, Firefox, Safari use it (and the browser-side via the OPFS/WASM build).
- **macOS, Windows, and most Linux distros** ship it; countless desktop apps embed it.
- **Aircraft, IoT devices, cars, set-top boxes** — its tiny footprint and rock-solid durability make it the default "application file format" for structured data.

SQLite's authors describe it not just as a database but as **"a replacement for `fopen()`"** — i.e., the right way to store *application data* on a single device, instead of inventing your own file format. Its public-domain license, exhaustive test suite (the test code dwarfs the library code), and decades of stability are why it is everywhere.

### 1.4 Where SQLite shines

- **Mobile apps** (Android/iOS local storage).
- **Edge & IoT** — runs anywhere C compiles, tiny memory footprint.
- **Desktop applications** — the modern, queryable "application file format".
- **Browsers / local-first web apps** — via the official WASM build with OPFS persistence; sync engines (e.g. CRDT-based local-first stacks) often build on SQLite.
- **Testing** — spin up a real SQL database in-memory (`:memory:`) per test, with zero setup and teardown.
- **Caching / config / metadata stores** embedded inside a larger service.
- **Small-to-medium websites** — for read-heavy sites, a single SQLite file on the web server is astonishingly capable. Modern tooling (e.g. Litestream/LiteFS-style streaming replication, and read-replica patterns) has pushed SQLite well into production web territory.
- **Data analysis & shipping datasets** — a `.db` (or `.sqlite`) file is a great way to ship a queryable dataset.

### 1.5 Where SQLite is NOT the right tool

- **High write concurrency / many simultaneous writers.** Only one writer touches the database at a time. A workload with hundreds of clients all trying to *write* constantly will serialize and contend. (Reads scale far better, especially in WAL mode — see §8.)
- **Large multi-user, networked systems** that need a database accessible by many machines over the network. SQLite has no network layer; bolting one on means you've essentially rebuilt a server (and should just use Postgres).
- **Massive datasets that exceed one machine.** SQLite handles surprisingly large databases (terabytes are technically supported), but if you need horizontal scaling, sharding, or replication across nodes, that's client-server territory.
- **Fine-grained, per-user network access control.** Access control in SQLite is *filesystem* permissions on the file; there are no SQL users/roles/`GRANT`.

> **Rule of thumb:** If the data lives on *one machine* and is accessed by *one application* (even a multi-threaded one), SQLite is probably perfect. The moment you need *many machines* to share *live* data, or *sustained high write concurrency from many clients*, reach for PostgreSQL (see **`POSTGRESQL_GUIDE.md`**) — and §15 covers how to migrate.

---

## 2. Getting Started: the CLI & Dot-Commands

**[B]**

### 2.1 Installing the `sqlite3` command-line shell

The library is everywhere, but to *interactively* poke at databases you want the `sqlite3` CLI shell.

```bash
# Windows 11 (PowerShell): the easiest route is winget or scoop
winget install SQLite.SQLite        # installs the sqlite3.exe shell
# or download the "sqlite-tools" zip from sqlite.org/download.html and put sqlite3.exe on your PATH

# macOS: preinstalled; or via Homebrew for a newer version
brew install sqlite

# Debian/Ubuntu
sudo apt-get install sqlite3
```

Check the version (matters for the modern features in this guide):

```bash
sqlite3 --version            # prints e.g. 3.46.0 2024-...   (you want 3.37+ for STRICT, 3.45+ ideally)
```

You can also confirm from *inside* SQL (from any app, not just the CLI):

```sql
SELECT sqlite_version();     -- e.g. '3.46.0'
```

### 2.2 Opening / creating a database

A SQLite database is just a file. Opening one that doesn't exist **creates** it (lazily — the file isn't actually written until you create a table or write data).

```bash
sqlite3 app.db          # open (or create) app.db and drop into the SQL prompt
sqlite3                 # open a *transient in-memory* database (lost on exit)
sqlite3 :memory:        # explicit in-memory database
```

At the `sqlite>` prompt you type SQL (terminated by `;`) or **dot-commands** (shell directives, *not* SQL, that start with `.` and need no semicolon).

### 2.3 The essential dot-commands

Dot-commands configure the shell itself — output format, importing/exporting, inspecting schema. They are a feature of the **CLI**, not of SQLite-the-engine, so they do nothing inside your app code.

```text
.help                 -- list all dot-commands with descriptions
.databases            -- show attached databases and their files
.tables               -- list all tables (and views) in the database
.schema               -- print the CREATE statements for the WHOLE database
.schema users         -- print just the CREATE statement(s) for table `users`
.indexes              -- list indexes
.fullschema           -- schema + the contents of sqlite_stat tables (for query-plan debugging)

.mode column          -- pretty, aligned columns (great for reading)
.mode box             -- Unicode box-drawing table (very readable; modern shells)
.mode table           -- Markdown-ish table
.mode csv             -- comma-separated (for export)
.mode json            -- emit each row as a JSON object
.mode insert tbl      -- emit results as INSERT statements into `tbl`
.headers on           -- show column names as a header row
.nullvalue NULL       -- print this string for SQL NULLs (default is empty)

.import data.csv users   -- import data.csv into table `users` (respects .mode csv)
.once out.csv         -- send the NEXT query's output to a file
.output out.txt       -- redirect ALL subsequent output to a file (.output to reset)

.dump                 -- emit the entire DB as a SQL script (schema + INSERTs) — see §15
.dump users           -- dump just one table
.backup main back.db  -- make a safe live backup copy of the DB (see §15)

.read script.sql      -- execute SQL from a file
.timer on             -- show wall-clock time for each query (handy for tuning)
.expert               -- suggest indexes for a query (analysis helper)
.quit  / .exit        -- leave the shell  (Ctrl-D on unix also works)
```

### 2.4 A complete first session (heavily commented)

```bash
$ sqlite3 demo.db          # create/open demo.db
```

```sql
-- We're now at the sqlite> prompt. First, make output readable:
.mode box                  -- pretty Unicode tables
.headers on                -- show column names

-- Create a table. (Details of CREATE TABLE are in §4; this is just to get moving.)
CREATE TABLE fruit (
  id    INTEGER PRIMARY KEY,   -- becomes the rowid alias (see §3.6)
  name  TEXT NOT NULL,
  price REAL                   -- a floating-point number
);

-- Insert a few rows.
INSERT INTO fruit (name, price) VALUES ('apple', 0.50), ('banana', 0.25), ('cherry', 2.00);

-- Query.
SELECT * FROM fruit WHERE price < 1.00 ORDER BY price;
-- ┌────┬────────┬───────┐
-- │ id │  name  │ price │
-- ├────┼────────┼───────┤
-- │ 2  │ banana │ 0.25  │
-- │ 1  │ apple  │ 0.5   │
-- └────┴────────┴───────┘

-- Inspect the schema we just made:
.schema fruit
-- CREATE TABLE fruit ( id INTEGER PRIMARY KEY, name TEXT NOT NULL, price REAL );

-- List tables:
.tables
-- fruit

.quit                      -- done; demo.db now exists on disk with our data
```

### 2.5 Running SQL non-interactively

You don't have to be interactive — the CLI is scriptable, which is handy in CI, cron jobs, and shell pipelines.

```bash
# Run one statement and exit:
sqlite3 demo.db "SELECT count(*) FROM fruit;"

# Pipe a whole script in:
sqlite3 demo.db < setup.sql

# Use the .read dot-command from the command line via -cmd:
sqlite3 demo.db -cmd ".mode csv" "SELECT * FROM fruit;" > fruit.csv

# Open read-only (won't create/modify the file) — great for inspecting prod copies:
sqlite3 -readonly demo.db "SELECT count(*) FROM fruit;"
```

> **Windows note:** In PowerShell, `<` input redirection isn't supported the same way as in bash. Use `Get-Content setup.sql | sqlite3 demo.db` or `sqlite3 demo.db ".read setup.sql"`.

---

## 3. Type Affinity & Dynamic Typing (the big quirk)

**[B/I]**

This section is the one that surprises people coming from Postgres or MySQL. **Read it carefully** — most "SQLite is weird" complaints trace back to not understanding type affinity.

### 3.1 The core idea: SQLite is dynamically typed

In almost every other SQL database, **types belong to columns**: a column declared `INTEGER` can *only* ever hold integers, and the database rejects anything else. This is *static* (or *rigid* / *manifest*) typing.

SQLite is different. By default, **types belong to *values*, not columns.** Any column (with one exception, the `INTEGER PRIMARY KEY`) can hold a value of *any* type, regardless of how the column was declared. The declared "type" of a column is more of a *hint* — called its **affinity** — that influences how values are *converted* on the way in, but does not strictly *enforce* what may be stored.

This is intentional. SQLite was designed to be flexible and forgiving, and this dynamic typing is part of its "replacement for `fopen()`" philosophy. But it means a column you declared as `INTEGER` can, by default, end up storing the text `'banana'` if you insert it — SQLite will happily keep it.

### 3.2 The 5 storage classes

Every *value* stored in SQLite has one of exactly **five storage classes** (this is the value's actual physical type):

| Storage class | What it holds |
|---|---|
| **NULL** | The SQL NULL (absence of a value). |
| **INTEGER** | A signed integer, stored in 1–8 bytes depending on magnitude. |
| **REAL** | An 8-byte IEEE 754 floating-point number. |
| **TEXT** | A text string (stored in the database encoding: UTF-8, UTF-16LE, or UTF-16BE). |
| **BLOB** | A blob of bytes, stored exactly as input (no interpretation). |

Note what's *missing*: there is no native `BOOLEAN`, no native `DATE`/`DATETIME`, no `DECIMAL`/`NUMERIC` money type, no `VARCHAR(n)` length enforcement. Those concepts are *expressed* using the five classes (booleans as 0/1 integers, dates as TEXT/INTEGER/REAL — see §12.4), but they are not first-class storage types.

You can ask any value its storage class with the `typeof()` function:

```sql
SELECT typeof(123),       -- 'integer'
       typeof(1.5),       -- 'real'
       typeof('hi'),      -- 'text'
       typeof(NULL),      -- 'null'
       typeof(x'1234');   -- 'blob'   (x'...' is a hex blob literal)
```

### 3.3 Column affinity and the rules

When you declare a column with a type name, SQLite assigns the column an **affinity** — its *preferred* storage class. There are **five affinities**: `TEXT`, `NUMERIC`, `INTEGER`, `REAL`, and `BLOB`.

The affinity is determined from the declared type name using these rules (checked in order):

1. If the declared type contains **`INT`** → **INTEGER** affinity. (So `INTEGER`, `INT`, `BIGINT`, `TINYINT` all get INTEGER affinity.)
2. Else if it contains **`CHAR`**, **`CLOB`**, or **`TEXT`** → **TEXT** affinity. (`VARCHAR(255)`, `NVARCHAR`, `TEXT`, `CHARACTER(20)`.)
3. Else if it contains **`BLOB`**, or **no type is declared at all** → **BLOB** affinity (no conversion happens; values are stored as-is).
4. Else if it contains **`REAL`**, **`FLOA`**, or **`DOUB`** → **REAL** affinity. (`REAL`, `FLOAT`, `DOUBLE`.)
5. Otherwise → **NUMERIC** affinity. (This is the catch-all — `NUMERIC`, `DECIMAL`, `BOOLEAN`, `DATE`, `DATETIME` all land here.)

**What affinity does:** when you store a value into a column, SQLite tries to *convert* the value to the column's preferred class **if it can do so losslessly / sensibly**, then stores it:

- **TEXT** affinity: numeric values are converted to text before storing.
- **NUMERIC** / **INTEGER** / **REAL** affinity: text that *looks like* a number is converted to a number; text that doesn't look numeric is stored as text unchanged. INTEGER prefers integer storage; a value like `4.0` may be stored as the integer `4`.
- **BLOB** affinity: no conversion at all — store whatever you give it.

### 3.4 Why a "TEXT" column can hold a number (and vice-versa)

Because affinity only *prefers* a class — it doesn't *forbid* others. If a conversion would lose information or doesn't apply, SQLite stores the original value as-is. This is the source of the classic surprise:

```sql
CREATE TABLE demo (n INTEGER, t TEXT, anything);  -- last column: no type => BLOB affinity

-- Insert "wrong" types and watch what SQLite actually stores:
INSERT INTO demo (n, t, anything) VALUES ('banana', 42, 99);

SELECT typeof(n),        -- 'text'    <- 'banana' couldn't become an integer, kept as text!
       typeof(t),        -- 'text'    <- 42 was converted to the text '42' (TEXT affinity)
       typeof(anything)  -- 'integer' <- BLOB affinity does no conversion; 99 stayed an integer
FROM demo;

-- A number-looking string DOES get converted in a NUMERIC/INTEGER column:
INSERT INTO demo (n) VALUES ('100');
SELECT typeof(n) FROM demo WHERE n = 100;   -- 'integer'  ('100' -> 100)
```

The practical danger: comparisons and indexes behave according to *storage class*, and `'42'` (text) does **not** equal `42` (integer) in a sort/comparison sense. If half your column is text-`'42'` and half is integer-`42`, ordering and `=` get confusing. Hence STRICT tables (next).

### 3.5 STRICT tables — enforce types like a grown-up database **[I]**

> **⚡ Version note:** `STRICT` tables require **SQLite 3.37 (late 2021)** or newer. They are the modern, recommended way to get predictable typing.

If you *want* the safety of rigid typing (and you usually do for application schemas), append the **`STRICT`** keyword to `CREATE TABLE`. In a STRICT table:

- Every column **must** declare one of exactly these types: `INT`, `INTEGER`, `REAL`, `TEXT`, `BLOB`, or `ANY`.
- Inserting a value of the wrong type that *cannot* be losslessly converted raises an **error** instead of being silently stored.
- An `INTEGER` column will *not* silently keep a non-numeric string; you get an error.
- The special `ANY` type lets a column hold any storage class (useful escape hatch) — and notably, in a STRICT table `ANY` does *not* apply affinity, so values are stored exactly as given.

```sql
CREATE TABLE account (
  id      INTEGER PRIMARY KEY,
  email   TEXT NOT NULL,
  balance INTEGER NOT NULL DEFAULT 0,   -- store money as integer cents (see §3.7)
  meta    ANY                            -- can hold any type, stored as-is
) STRICT;

INSERT INTO account (email, balance) VALUES ('a@b.com', 1500);   -- OK

INSERT INTO account (email, balance) VALUES ('c@d.com', 'oops'); -- ERROR:
-- Runtime error: cannot store TEXT value in INTEGER column account.balance
```

> **Recommendation:** Use `STRICT` for every application table you control. You get the file-format simplicity of SQLite *and* the type safety of a normal database. The only reason not to is when you genuinely need dynamic typing (rare) or must support pre-3.37 engines.

### 3.6 `rowid` and `INTEGER PRIMARY KEY` — the one column that's special **[B/I]**

Every ordinary SQLite table has a hidden 64-bit signed integer key called the **`rowid`** (also accessible as `_rowid_` or `oid`). It uniquely identifies each row and is how rows are physically stored and looked up — the table is essentially a B-tree keyed by `rowid`.

There is one magic declaration: **`INTEGER PRIMARY KEY`**. When a column is declared *exactly* as `INTEGER PRIMARY KEY` (the type must be `INTEGER`, not `INT` or `BIGINT`), that column becomes an **alias for the rowid**. This is the most efficient primary key you can have — lookups by it are direct B-tree lookups with no secondary index needed.

```sql
CREATE TABLE post (
  id    INTEGER PRIMARY KEY,   -- THIS column IS the rowid. Auto-assigned if you omit it.
  title TEXT NOT NULL
);

INSERT INTO post (title) VALUES ('Hello');   -- id auto-fills with the next rowid (1)
INSERT INTO post (id, title) VALUES (100, 'Jump'); -- you can also set it explicitly
SELECT id, rowid, title FROM post;           -- id and rowid are the SAME value
```

Two subtleties:

- **`INTEGER PRIMARY KEY` vs `INT PRIMARY KEY`:** Only the exact spelling `INTEGER` triggers the rowid-alias behaviour. `INT PRIMARY KEY` does *not* alias the rowid (it creates a normal-ish primary key with INTEGER affinity). Always write `INTEGER PRIMARY KEY` when you want the efficient alias.
- **`WITHOUT ROWID` tables** (advanced): you can declare a table `WITHOUT ROWID` to make it a clustered B-tree keyed by a non-integer primary key (e.g. a TEXT key). This saves space and a lookup hop when your natural key isn't an integer, but it's a specialized optimization; the default rowid table is right the vast majority of the time.

### 3.7 Practical typing patterns

- **Booleans:** there is no boolean type. Use an `INTEGER` storing `0` (false) or `1` (true). The keywords `TRUE`/`FALSE` exist and evaluate to `1`/`0`.
- **Money / currency:** do **not** use `REAL` (floating point) for money — `0.1 + 0.2 != 0.3`. Store integer **cents** (or the smallest currency unit) in an `INTEGER` and divide in the app, or store as `TEXT` and use exact decimal math in the app. (Postgres has `NUMERIC`; SQLite does not — see **`POSTGRESQL_GUIDE.md`** for the contrast.)
- **Dates/times:** no native date type. Store as ISO-8601 `TEXT` (`'2026-06-22T10:30:00Z'`), Unix epoch `INTEGER` seconds, or Julian-day `REAL`. ISO-8601 text sorts correctly lexicographically and is human-readable — usually the best default. See §12.4.
- **UUIDs:** store as `TEXT` (36-char canonical) for readability, or as a 16-byte `BLOB` for compactness.

---

## 4. CRUD & Schema

**[B]**

### 4.1 `CREATE TABLE` — columns, constraints, defaults

`CREATE TABLE` defines a relation: its columns, their affinities (§3), and the **constraints** that keep the data honest. Constraints are the database doing your validation *for* you, atomically and reliably, so bad data can never get in regardless of which client wrote it.

Constraints come in two forms: **column constraints** (written after a single column) and **table constraints** (written as separate clauses, used when a constraint spans multiple columns, e.g. a composite primary key).

```sql
CREATE TABLE IF NOT EXISTS users (
  id         INTEGER PRIMARY KEY,                 -- rowid alias (§3.6)
  email      TEXT    NOT NULL UNIQUE,             -- NOT NULL: required; UNIQUE: no duplicates
  username   TEXT    NOT NULL,
  age        INTEGER CHECK (age IS NULL OR age >= 0),  -- CHECK: a boolean rule that must hold
  role       TEXT    NOT NULL DEFAULT 'member',   -- DEFAULT: value used when not supplied
  is_active  INTEGER NOT NULL DEFAULT 1,          -- boolean as 0/1
  created_at TEXT    NOT NULL DEFAULT (datetime('now')),  -- expression default (note the parens)

  -- A table-level constraint spanning multiple columns:
  UNIQUE (username, role)                         -- (username, role) pair must be unique
) STRICT;                                          -- enforce types (§3.5)
```

What each constraint *is* and *why* you'd use it:

- **`PRIMARY KEY`** — the unique identifier of a row. There can be only one per table (it may be composite). With `INTEGER PRIMARY KEY` it aliases the rowid (efficient). It implies uniqueness; combine with `NOT NULL` in STRICT tables for clarity. See **`RELATIONAL_DB_DESIGN_GUIDE.md`** for natural vs surrogate key trade-offs.
- **`NOT NULL`** — the column must always have a value. Prevents the "missing data" bugs that NULLs cause downstream.
- **`UNIQUE`** — no two rows may share this value (or, for composite, this combination). Implemented with a unique index, so it's also fast to look up by.
- **`CHECK (expr)`** — an arbitrary boolean expression evaluated on insert/update; the row is rejected if it's false. Great for domain rules (`price >= 0`, `status IN ('open','closed')`).
- **`DEFAULT`** — value substituted when an `INSERT` omits the column. Can be a literal or a *parenthesized* expression like `(datetime('now'))`.
- **`FOREIGN KEY`** — references another table's key; covered in depth in §7 (including the crucial fact that enforcement is *off by default*).

> **`DEFAULT (datetime('now'))` vs `CURRENT_TIMESTAMP`:** SQLite also accepts the bare keywords `CURRENT_TIMESTAMP`, `CURRENT_TIME`, `CURRENT_DATE` as defaults. `CURRENT_TIMESTAMP` yields `'YYYY-MM-DD HH:MM:SS'` in **UTC**. Using `datetime('now')` is equivalent and more explicit. Be aware both are UTC, not local time.

### 4.2 `INSERT`

```sql
-- Basic insert, naming columns (always name them — robust to schema changes):
INSERT INTO users (email, username) VALUES ('ann@x.com', 'ann');

-- Multi-row insert (one statement, one transaction — far faster than N statements, see §16):
INSERT INTO users (email, username) VALUES
  ('bob@x.com', 'bob'),
  ('cara@x.com', 'cara'),
  ('dan@x.com', 'dan');

-- Insert the result of a query:
INSERT INTO users_archive (email, username)
SELECT email, username FROM users WHERE is_active = 0;
```

### 4.3 `UPDATE` and `DELETE`

```sql
-- UPDATE: always have a WHERE unless you really mean "every row".
UPDATE users SET role = 'admin', is_active = 1 WHERE email = 'ann@x.com';

-- UPDATE using a value derived from the row itself:
UPDATE users SET username = lower(username);

-- DELETE: again, mind the WHERE.
DELETE FROM users WHERE is_active = 0;

-- "Truncate" the table (SQLite has no TRUNCATE; this is optimized internally):
DELETE FROM users;          -- removes all rows
```

> **Safety gotcha:** A missing `WHERE` on `UPDATE`/`DELETE` hits *every row*. The CLI does not warn you. Wrap risky changes in a transaction (`BEGIN; … ;` then verify with a `SELECT` before `COMMIT;` — §8) so you can `ROLLBACK`.

### 4.4 `SELECT` basics

```sql
SELECT id, email, role FROM users;                  -- specific columns (preferred)
SELECT * FROM users;                                -- all columns (fine ad-hoc, avoid in app code)
SELECT DISTINCT role FROM users;                    -- unique values
SELECT count(*) AS n, role FROM users GROUP BY role; -- aggregate (more in §5)
```

### 4.5 `ON CONFLICT` / UPSERT — insert-or-update **[I]**

> **⚡ Version note:** UPSERT (`ON CONFLICT … DO UPDATE`) requires **SQLite 3.24+ (2018)**.

Often you want "insert this row, but if it would violate a UNIQUE/PK constraint, update the existing row instead." That's an **upsert**. SQLite expresses it with an `ON CONFLICT` clause that targets the conflicting constraint.

```sql
-- Track page-view counts; first view inserts, subsequent views increment.
CREATE TABLE page_views (
  path  TEXT PRIMARY KEY,
  views INTEGER NOT NULL DEFAULT 0,
  last  TEXT    NOT NULL
) STRICT;

INSERT INTO page_views (path, views, last)
VALUES ('/home', 1, datetime('now'))
ON CONFLICT(path) DO UPDATE SET            -- if `path` already exists:
  views = views + 1,                       -- bump the count
  last  = excluded.last;                   -- `excluded` = the row we TRIED to insert
```

Key concepts:

- **`ON CONFLICT(column)`** names the **conflict target** — the UNIQUE/PK constraint that would be violated. (You can also write `ON CONFLICT` with no target to mean "any conflict".)
- **`excluded`** is a special pseudo-table referring to the values from the row you attempted to insert. Use it to pull the new values into the update.
- **`DO NOTHING`** is the other option — silently skip the insert on conflict (an "insert if not exists").

```sql
-- Insert only if not present (ignore duplicates):
INSERT INTO users (email, username) VALUES ('ann@x.com', 'ann')
ON CONFLICT(email) DO NOTHING;
```

There's also the older, blunter **`INSERT OR …`** syntax (conflict *resolution* algorithms): `INSERT OR IGNORE`, `INSERT OR REPLACE`, `INSERT OR ABORT`, etc. `INSERT OR REPLACE` (often written just `REPLACE`) *deletes* the conflicting row and inserts a new one — which can fire `ON DELETE` cascades and reset the rowid. Prefer the explicit `ON CONFLICT … DO UPDATE` for upserts; reach for `INSERT OR IGNORE` only for true "skip duplicates".

### 4.6 `RETURNING` — get back what changed **[I]**

> **⚡ Version note:** `RETURNING` requires **SQLite 3.35+ (2021)**.

Normally `INSERT`/`UPDATE`/`DELETE` return no rows — to learn the new `id` you'd query again. `RETURNING` lets the write statement *return* the affected rows, in one round trip. This is invaluable for getting auto-generated keys and timestamps.

```sql
INSERT INTO users (email, username) VALUES ('eve@x.com', 'eve')
RETURNING id, created_at;         -- returns the new row's generated id and default timestamp

UPDATE users SET role = 'admin' WHERE email = 'eve@x.com'
RETURNING id, email, role;        -- see exactly what you changed

DELETE FROM users WHERE is_active = 0
RETURNING id;                     -- get the ids you just deleted
```

### 4.7 `AUTOINCREMENT` — and why you usually don't need it **[I]**

A column declared `INTEGER PRIMARY KEY` already auto-assigns increasing integer ids (it picks "one larger than the largest existing rowid", or reuses a gap after deletes). Adding the **`AUTOINCREMENT`** keyword changes the algorithm so that a rowid is **never reused**, even after rows are deleted — it monotonically increases for the life of the table, tracked in an internal `sqlite_sequence` table.

```sql
CREATE TABLE invoice (
  id INTEGER PRIMARY KEY AUTOINCREMENT,   -- ids never reused, even after deletion
  total INTEGER NOT NULL
) STRICT;
```

Why you usually **don't** want it:

- It adds overhead (an extra table read/write per insert to maintain `sqlite_sequence`).
- Plain `INTEGER PRIMARY KEY` is already monotonic in practice unless you delete the highest row *and* keep inserting — and even then, reusing a freed id is harmless for most applications.
- The only real reason to use `AUTOINCREMENT` is when you must *guarantee* an id is never reused for the lifetime of the database (e.g. ids leak into external systems / receipts and reuse would be confusing or dangerous).

> **Recommendation:** Use plain `INTEGER PRIMARY KEY`. Reach for `AUTOINCREMENT` only when non-reuse is a hard requirement.

### 4.8 `ALTER TABLE` — limited but improving

SQLite's `ALTER TABLE` is more limited than Postgres's. You can:

```sql
ALTER TABLE users ADD COLUMN phone TEXT;             -- add a column (with optional DEFAULT)
ALTER TABLE users RENAME TO app_users;               -- rename the table
ALTER TABLE app_users RENAME COLUMN phone TO mobile; -- rename a column (3.25+)
ALTER TABLE app_users DROP COLUMN mobile;            -- drop a column (3.35+)
```

You **cannot** directly change a column's type or add a constraint to an existing column. The official pattern for such changes is the **12-step "create new table, copy, drop, rename"** dance, ideally wrapped in a transaction with `PRAGMA foreign_keys=OFF` around it. Most migration tools do this for you.

---

## 5. Querying

**[B/I]**

SQLite's query language is a substantial subset of standard SQL — and a *superset* in a few places. If you know Postgres SQL you'll feel at home; the differences are mostly about what's *absent* (no `FULL OUTER JOIN` until 3.39, no `RIGHT JOIN` until 3.39 — both now present in modern versions).

### 5.1 `WHERE`, `ORDER BY`, `LIMIT`, `OFFSET`

```sql
SELECT name, price
FROM fruit
WHERE price BETWEEN 0.25 AND 1.00      -- inclusive range
  AND name LIKE 'a%'                    -- LIKE: % = any chars, _ = one char (case-insensitive for ASCII)
  AND name IN ('apple', 'apricot')      -- membership
ORDER BY price DESC, name ASC           -- multi-key sort
LIMIT 10 OFFSET 20;                      -- pagination: skip 20, take 10
```

Notes:
- `LIKE` is case-insensitive for ASCII letters by default; `GLOB` uses Unix-glob syntax and *is* case-sensitive.
- For NULL checks you **must** use `IS NULL` / `IS NOT NULL` — `= NULL` is never true.
- **Keyset pagination** (`WHERE id > :last_id ORDER BY id LIMIT 10`) scales far better than large `OFFSET`s, which must scan-and-discard.

### 5.2 Joins

A **join** combines rows from multiple tables based on a relationship (usually a foreign key matching a primary key). See **`RELATIONAL_DB_DESIGN_GUIDE.md`** for the modelling theory; here's the SQL.

```sql
-- Setup for examples:
CREATE TABLE author (id INTEGER PRIMARY KEY, name TEXT NOT NULL) STRICT;
CREATE TABLE book   (id INTEGER PRIMARY KEY, title TEXT NOT NULL,
                     author_id INTEGER REFERENCES author(id)) STRICT;

-- INNER JOIN: only rows that match on BOTH sides.
SELECT b.title, a.name
FROM book AS b
JOIN author AS a ON a.id = b.author_id;

-- LEFT (OUTER) JOIN: every book; author columns are NULL if no match.
SELECT b.title, a.name
FROM book b
LEFT JOIN author a ON a.id = b.author_id;

-- RIGHT and FULL OUTER joins (SQLite 3.39+, 2022):
SELECT a.name, b.title
FROM book b
RIGHT JOIN author a ON a.id = b.author_id;     -- every author, even with no books

SELECT a.name, b.title
FROM book b
FULL OUTER JOIN author a ON a.id = b.author_id; -- every author AND every book

-- CROSS JOIN: cartesian product (every combination) — rarely what you want.
SELECT a.name, b.title FROM author a CROSS JOIN book b;
```

> **⚡ Version note:** `RIGHT JOIN` and `FULL OUTER JOIN` arrived in **SQLite 3.39 (2022)**. Older engines only had `INNER`, `LEFT`, and `CROSS`. Modern 2026 builds have all of them.

### 5.3 Aggregates & `GROUP BY`

Aggregate functions collapse many rows into one summary value. `GROUP BY` splits rows into buckets and applies the aggregate per bucket; `HAVING` filters *after* aggregation (whereas `WHERE` filters *before*).

```sql
SELECT a.name              AS author,
       count(*)            AS book_count,    -- count of rows in each group
       min(b.title)        AS first_title,
       group_concat(b.title, '; ') AS titles -- concatenate values in the group
FROM author a
JOIN book b ON b.author_id = a.id
GROUP BY a.id                                 -- one row per author
HAVING count(*) >= 2                           -- only authors with 2+ books
ORDER BY book_count DESC;
```

Built-in aggregates: `count`, `sum`, `avg`, `min`, `max`, `total` (like sum but always returns REAL), `group_concat`/`string_agg`, and (3.44+) you can add `FILTER (WHERE …)` to any aggregate to count a subset:

```sql
SELECT count(*) AS total,
       count(*) FILTER (WHERE role = 'admin') AS admins   -- 3.44+
FROM users;
```

> **Gotcha:** SQLite is *lenient* about selecting non-aggregated, non-grouped columns (it picks an arbitrary row's value — a misfeature inherited for compatibility). There's one *useful* special case: a "bare columns" `SELECT *, max(score) … GROUP BY g` returns the *whole row* that had the max. Otherwise, only select columns you grouped by or aggregated.

### 5.4 Subqueries

A subquery is a `SELECT` nested inside another statement — used as a value, a derived table, or an existence test.

```sql
-- Scalar subquery (returns one value):
SELECT title, (SELECT name FROM author WHERE id = book.author_id) AS author
FROM book;

-- Subquery as a derived table (FROM clause):
SELECT author, n FROM (
  SELECT author_id AS author, count(*) AS n FROM book GROUP BY author_id
) WHERE n > 1;

-- Correlated EXISTS (efficient "does a related row exist?"):
SELECT name FROM author a
WHERE EXISTS (SELECT 1 FROM book b WHERE b.author_id = a.id);
```

### 5.5 CTEs (Common Table Expressions) — `WITH` **[I]**

A CTE is a named, temporary result set defined with `WITH`, used to make complex queries readable by naming intermediate steps. It reads top-to-bottom like variables.

```sql
WITH active AS (                       -- name a subquery
  SELECT * FROM users WHERE is_active = 1
),
by_role AS (
  SELECT role, count(*) AS n FROM active GROUP BY role
)
SELECT * FROM by_role WHERE n > 5;     -- reference the CTEs like tables
```

### 5.6 Recursive CTEs — hierarchies & sequences **[I/A]**

A **recursive** CTE references *itself*, letting you walk tree/graph structures (org charts, category trees, threaded comments) or generate sequences. The shape is always: an *anchor* (base case) `UNION ALL` a *recursive* member that builds on the previous step, terminating when the recursive member returns no rows.

```sql
-- Generate numbers 1..10 (the classic "tally" generator):
WITH RECURSIVE seq(n) AS (
  SELECT 1                       -- anchor: start at 1
  UNION ALL
  SELECT n + 1 FROM seq WHERE n < 10   -- recursive: each step adds 1, stop at 10
)
SELECT n FROM seq;               -- 1,2,3,...,10

-- Walk an org chart (employee -> manager):
CREATE TABLE emp (id INTEGER PRIMARY KEY, name TEXT, manager_id INTEGER) STRICT;
INSERT INTO emp VALUES (1,'CEO',NULL),(2,'VP',1),(3,'Lead',2),(4,'Dev',3);

WITH RECURSIVE chain(id, name, manager_id, depth) AS (
  SELECT id, name, manager_id, 0 FROM emp WHERE id = 4   -- anchor: start at 'Dev'
  UNION ALL
  SELECT e.id, e.name, e.manager_id, c.depth + 1         -- recursive: climb to each manager
  FROM emp e JOIN chain c ON e.id = c.manager_id
)
SELECT depth, name FROM chain;   -- Dev(0) -> Lead(1) -> VP(2) -> CEO(3)
```

### 5.7 Window functions **[I/A]**

> **⚡ Version note:** Window functions require **SQLite 3.25+ (2018)** — present in all modern builds.

A **window function** computes a value across a set of rows *related to the current row* (the "window") **without collapsing them** the way `GROUP BY` does. You keep every row *and* get aggregate/ranking context. Use them for running totals, rankings, "previous row" comparisons, and top-N-per-group.

The syntax is `func() OVER (PARTITION BY … ORDER BY … frame)`:
- **`PARTITION BY`** splits rows into independent groups (like `GROUP BY` but rows are kept).
- **`ORDER BY`** orders rows within each partition (needed for ranking and running totals).

```sql
CREATE TABLE sale (region TEXT, day INTEGER, amount INTEGER) STRICT;
INSERT INTO sale VALUES ('east',1,100),('east',2,150),('east',3,120),
                        ('west',1,200),('west',2,80);

SELECT region, day, amount,
  -- Running total within each region, ordered by day:
  sum(amount) OVER (PARTITION BY region ORDER BY day) AS running_total,
  -- Rank by amount within region (1 = highest); RANK leaves gaps on ties:
  rank()       OVER (PARTITION BY region ORDER BY amount DESC) AS amt_rank,
  -- Previous day's amount in the same region (NULL for the first day):
  lag(amount)  OVER (PARTITION BY region ORDER BY day) AS prev_day,
  -- Region's average across all its rows, attached to every row:
  avg(amount)  OVER (PARTITION BY region) AS region_avg
FROM sale
ORDER BY region, day;
```

Common window functions: `row_number()`, `rank()`, `dense_rank()`, `ntile(n)`, `lag()`, `lead()`, `first_value()`, `last_value()`, plus any aggregate (`sum`, `avg`, `count`…) used with `OVER`.

### 5.8 `CASE` — inline conditionals

`CASE` is SQL's if/else expression — it returns a value based on conditions.

```sql
SELECT name,
  CASE
    WHEN price < 0.50 THEN 'cheap'
    WHEN price < 2.00 THEN 'mid'
    ELSE 'pricey'
  END AS tier
FROM fruit;

-- "Simple" CASE compares one expression to values:
SELECT CASE role WHEN 'admin' THEN 'staff' WHEN 'mod' THEN 'staff' ELSE 'user' END
FROM users;
```

### 5.9 Set operations

```sql
SELECT email FROM users
UNION         -- combine, removing duplicates (UNION ALL keeps duplicates, faster)
SELECT email FROM users_archive;

SELECT email FROM users
INTERSECT     -- rows in both
SELECT email FROM newsletter;

SELECT email FROM users
EXCEPT        -- rows in the first but not the second
SELECT email FROM unsubscribed;
```

---

## 6. Indexing & EXPLAIN QUERY PLAN

**[I]**

### 6.1 What an index is and why it matters

Without an index, finding rows matching a `WHERE` clause means a **full table scan**: SQLite reads every row and checks each one — O(n). An **index** is a separate, sorted B-tree data structure that maps column values to the rows that contain them, so SQLite can *seek* directly to matching rows in O(log n) instead. The trade-off: indexes take disk space and slow down writes slightly (every insert/update must also update the index). You index the columns you frequently **filter on (`WHERE`), join on, or sort by (`ORDER BY`)**.

Remember from §3.6: the `INTEGER PRIMARY KEY` (the rowid) is *already* the table's clustering key, so lookups by it are inherently fast and need no extra index. `UNIQUE` and `PRIMARY KEY` constraints create indexes automatically.

### 6.2 Creating indexes

```sql
-- Single-column index — speeds up WHERE email = ? and ORDER BY email.
CREATE INDEX idx_users_email ON users(email);

-- Composite (multi-column) index — order matters! This helps queries that filter on
-- (role) or (role, created_at), following the "leftmost prefix" rule.
CREATE INDEX idx_users_role_created ON users(role, created_at);

-- Unique index — also enforces uniqueness (same as a UNIQUE constraint).
CREATE UNIQUE INDEX idx_users_email_u ON users(email);

-- Partial index — only indexes rows matching a condition. Smaller & faster when you
-- only ever query that subset (e.g. only active users).
CREATE INDEX idx_active_users ON users(email) WHERE is_active = 1;

-- Expression index — index the result of an expression (e.g. case-insensitive search).
CREATE INDEX idx_users_lower_email ON users(lower(email));
-- Now this query can use the index:  SELECT * FROM users WHERE lower(email) = 'a@b.com';

DROP INDEX idx_users_email;       -- remove an index
```

### 6.3 The leftmost-prefix rule (composite indexes)

A composite index on `(a, b, c)` can be used for queries filtering on `a`, on `a,b`, or on `a,b,c` — i.e. any **leftmost prefix**. It generally *cannot* be used to satisfy a query that filters only on `b` or only on `c`. Order your composite index columns by how you query: equality-filtered columns first, then the range/sort column last.

### 6.4 Covering indexes

A **covering index** is one that contains *all* the columns a query needs — both the filtered columns *and* the selected columns. SQLite can then answer the query entirely from the index B-tree **without touching the table** at all (an "index-only scan"), which is very fast.

```sql
-- This index "covers" a query that selects email & role filtered by role,
-- because both columns are in the index:
CREATE INDEX idx_cover ON users(role, email);

-- Covered: SQLite reads only the index, never the table rows.
SELECT email FROM users WHERE role = 'admin';
```

### 6.5 `EXPLAIN QUERY PLAN` — reading what SQLite will do

This is your single most important tuning tool. Prefix any query with `EXPLAIN QUERY PLAN` (often abbreviated **EQP**) to see, in plain language, *how* SQLite intends to execute it — which tables it scans, which indexes it uses, and the join order.

```sql
EXPLAIN QUERY PLAN
SELECT b.title, a.name
FROM book b JOIN author a ON a.id = b.author_id
WHERE a.name = 'Tolkien';
```

How to read the output (key phrases):

- **`SCAN table`** — a full table scan (reads every row). Fine for tiny tables; a red flag for large ones in a hot query. If you see `SCAN` on a big table that you filter on, you probably need an index.
- **`SEARCH table USING INDEX idx (...)`** — good: it's using an index to seek directly to rows.
- **`SEARCH table USING INTEGER PRIMARY KEY (rowid=?)`** — best: a direct rowid lookup.
- **`USING COVERING INDEX`** — excellent: answered from the index alone (§6.4).
- **`USE TEMP B-TREE FOR ORDER BY`** — SQLite had to build a temporary structure to sort; an index matching the `ORDER BY` could eliminate it.
- The order of lines usually reflects the **join order** the planner chose.

> **Note:** plain `EXPLAIN` (without `QUERY PLAN`) dumps the low-level VDBE bytecode — rarely what you want. Use `EXPLAIN QUERY PLAN`.

### 6.6 `ANALYZE` — give the planner statistics

The query planner makes better choices when it knows the *distribution* of your data. `ANALYZE` gathers statistics into internal `sqlite_stat1` tables; the planner uses them to pick indexes and join orders. Run it after bulk loads or significant data changes.

```sql
ANALYZE;                 -- gather stats for the whole database
ANALYZE users;           -- just one table/index
```

### 6.7 When indexes *don't* help (or hurt)

- On **tiny tables**, a scan is as fast as an index seek — don't bother.
- On **low-cardinality** columns (e.g. a boolean with two values) a plain index helps little; consider a *partial* index instead.
- Every index **slows writes** and uses space; don't index columns you never filter/sort/join on.
- A function applied to a column in `WHERE` (`WHERE lower(email) = …`) defeats a plain index on `email` — use an *expression index* (§6.2) to match.
- Leading **wildcard** `LIKE '%term'` can't use a normal index (no usable prefix); use FTS5 (§11) for text search.

---

## 7. Constraints & Foreign Keys

**[I]**

### 7.1 What a foreign key is

A **foreign key (FK)** is a column (or set of columns) in one table that *references* the primary key (or a unique key) of another table, declaring a relationship and enforcing **referential integrity**: you cannot reference a parent row that doesn't exist, and (depending on the rules) you cannot delete a parent that still has children. This is how the database guarantees your relationships stay consistent. See **`RELATIONAL_DB_DESIGN_GUIDE.md`** for one-to-many / many-to-many modelling.

```sql
CREATE TABLE author (id INTEGER PRIMARY KEY, name TEXT NOT NULL) STRICT;

CREATE TABLE book (
  id        INTEGER PRIMARY KEY,
  title     TEXT NOT NULL,
  author_id INTEGER NOT NULL
            REFERENCES author(id)   -- column-level FK (shorthand)
            ON DELETE CASCADE        -- referential action (below)
            ON UPDATE CASCADE
) STRICT;

-- Equivalent table-level form (required for composite FKs):
-- FOREIGN KEY (author_id) REFERENCES author(id) ON DELETE CASCADE
```

### 7.2 The crucial gotcha: FK enforcement is OFF by default

Here is the single most surprising operational fact about SQLite: **foreign-key constraints are NOT enforced unless you turn them on, per connection, every time.**

```sql
PRAGMA foreign_keys = ON;   -- MUST be run on every new connection, BEFORE you rely on FKs
```

**Why is it off by default?** For **backward compatibility**. FK enforcement was added in SQLite 3.6.19 (2009), long after countless databases and applications already existed that assumed it was off. Turning it on by default would have broken them. So SQLite keeps the historical default (off) and asks you to opt in.

Critical details:
- It's a **connection-level** setting (not stored in the database file), so you must set it on **every** connection your app opens.
- It **cannot** be changed inside a transaction (set it immediately after opening the connection).
- If it's off, `ON DELETE CASCADE` etc. silently do nothing, and you can insert orphaned rows. This causes subtle data-integrity bugs that pass in tests (where you forgot to enable it too).
- Application drivers differ: some enable it for you, some don't. **Always set it explicitly** (the §13/§14 app examples do).

### 7.3 Referential actions

When a referenced (parent) row is deleted or its key updated, the **referential action** decides what happens to child rows:

| Action | On parent DELETE/UPDATE |
|---|---|
| **`NO ACTION`** (default) | Reject the change if children exist (checked at end of statement). |
| **`RESTRICT`** | Reject immediately if children exist. |
| **`CASCADE`** | Delete/update the children too (the change propagates). |
| **`SET NULL`** | Set the child FK column(s) to NULL. |
| **`SET DEFAULT`** | Set the child FK column(s) to their DEFAULT. |

```sql
CREATE TABLE comment (
  id      INTEGER PRIMARY KEY,
  post_id INTEGER REFERENCES post(id) ON DELETE CASCADE,  -- delete post -> delete its comments
  author  INTEGER REFERENCES users(id) ON DELETE SET NULL  -- delete user -> keep comment, null author
) STRICT;
```

### 7.4 Deferred constraints

By default FK checks happen at the end of each *statement* (`IMMEDIATE`). You can mark a constraint `DEFERRABLE INITIALLY DEFERRED` so it's only checked at `COMMIT` — useful when you must temporarily create a cyclic/inconsistent state inside a transaction (e.g. inserting two mutually-referencing rows).

```sql
-- ... author_id INTEGER REFERENCES author(id) DEFERRABLE INITIALLY DEFERRED ...
```

You can verify integrity at any time with `PRAGMA foreign_key_check;` (lists rows whose FK points nowhere) and `PRAGMA integrity_check;` (§15).

---

## 8. Transactions & the Concurrency Model (WAL)

**[I/A]**

This section explains the heart of how SQLite behaves under concurrent access — and why **WAL mode** is the setting you almost always want.

### 8.1 Transactions: `BEGIN` / `COMMIT` / `ROLLBACK`

A **transaction** groups multiple statements into one atomic unit: either *all* of them take effect (`COMMIT`) or *none* of them do (`ROLLBACK`). This gives you the **A** and **I** of ACID — atomicity and isolation. SQLite is fully transactional; even a single bare statement runs in an implicit (auto-commit) transaction.

```sql
BEGIN;                                    -- start a transaction
  UPDATE account SET balance = balance - 100 WHERE id = 1;
  UPDATE account SET balance = balance + 100 WHERE id = 2;
COMMIT;                                   -- both updates persist together (atomic transfer)

BEGIN;
  DELETE FROM users;                      -- oops
ROLLBACK;                                 -- undo everything since BEGIN; nothing changed
```

**Transaction types** (the `BEGIN` variants matter for concurrency):
- **`BEGIN`** / `BEGIN DEFERRED` (default) — doesn't lock anything yet; a read lock is taken on first read, a write lock on first write. The deferred write lock is where "database is locked" surprises happen under concurrency.
- **`BEGIN IMMEDIATE`** — acquires the write lock right away. **Use this** when you know the transaction will write, to avoid mid-transaction lock-upgrade failures and `SQLITE_BUSY`.
- **`BEGIN EXCLUSIVE`** — locks out other connections entirely (rarely needed in WAL mode).

> **Best practice for write transactions in a concurrent app:** use `BEGIN IMMEDIATE`. It fails fast at the start (where you can retry cleanly) instead of failing partway through after you've done work.

**Savepoints** are nested, named sub-transactions:
```sql
SAVEPOINT sp1;
  UPDATE account SET balance = 0 WHERE id = 1;
ROLLBACK TO sp1;     -- undo to the savepoint without aborting the whole transaction
RELEASE sp1;         -- commit the savepoint into the enclosing transaction
```

### 8.2 The locking & concurrency model — the single-writer reality

This is the defining constraint of SQLite. Because the database is a single file with no server coordinating clients, SQLite serializes writes at the **whole-database** level:

- **Many readers can read concurrently.**
- **Only ONE writer can write at a time**, and (in the classic rollback-journal mode) a writer blocks all readers for the duration of the write.

There is **no row-level or table-level locking** like Postgres's MVCC (see **`POSTGRESQL_GUIDE.md`** §13 for the contrast). The lock is on the entire database file. If a second connection tries to write while another write is in progress, it gets **`SQLITE_BUSY`** ("database is locked").

This is fine — even great — for the workloads SQLite targets (one app, mostly reads, modest write rate). It's the wrong tool when many clients must write constantly (§1.5).

### 8.3 Rollback journal vs WAL — the two journal modes

To be crash-safe, SQLite must be able to undo a half-finished write. It does this with a **journal**, and there are two strategies, selected by `PRAGMA journal_mode`:

**Rollback journal (the historical default, `DELETE` mode):**
- Before modifying a page, SQLite copies the *original* page into a separate `-journal` file.
- On commit, the journal is deleted; on crash, the journal is replayed to *roll back* to the pre-write state.
- **Consequence:** a writer needs an exclusive lock on the DB, so **writers block readers and readers block writers.** Concurrency is poor.

**Write-Ahead Logging (`WAL` mode) — what you want:**
- Instead of overwriting pages in the main file, new/changed pages are *appended* to a separate **`-wal` file**. The main database file is left untouched until a "checkpoint" folds the WAL back into it.
- **Consequence:** because readers read the (unchanged) main file plus a consistent snapshot, **readers do NOT block the writer and the writer does NOT block readers.** You still have only one writer at a time, but reads and the single write proceed concurrently. This is a massive concurrency improvement.
- It's also generally **faster** for writes (sequential appends), and commits are cheaper.

### 8.4 Enabling WAL (and why you almost always want it)

```sql
PRAGMA journal_mode = WAL;     -- switch this database to WAL mode
-- returns: wal
```

Key facts about WAL:
- **It's persistent.** Unlike most PRAGMAs (which are per-connection), `journal_mode = WAL` is stored *in the database file* — you set it once and it stays. (Confirm with `PRAGMA journal_mode;`.)
- It creates two sidecar files alongside `app.db`: **`app.db-wal`** (the log) and **`app.db-shm`** (shared memory index). These are normal and expected.
- **Checkpointing** periodically transfers WAL contents back into the main DB. SQLite auto-checkpoints (default ~1000 pages), or you can force it: `PRAGMA wal_checkpoint(TRUNCATE);`.
- **Caveat:** WAL requires the database to be on a filesystem that supports shared memory / memory-mapping. It does **not** work over most **network filesystems (NFS, SMB)** — keep the DB on local disk. (This is one reason SQLite-over-a-network-share is an anti-pattern; §17.)
- To go back: `PRAGMA journal_mode = DELETE;`.

> **Recommendation:** For virtually every application database, set `journal_mode = WAL` once. It removes the reader/writer blocking that bites people and speeds up writes. The combination **WAL + `busy_timeout` + `synchronous = NORMAL`** (next) is the standard production trio.

### 8.5 `busy_timeout` — handle "database is locked" gracefully

When a connection can't get the lock it needs, it gets `SQLITE_BUSY` *immediately* by default. Setting a **busy timeout** tells SQLite to instead *wait and retry* (sleeping) for up to N milliseconds before giving up — which smooths over brief contention (one writer finishing while another wants to start).

```sql
PRAGMA busy_timeout = 5000;    -- wait up to 5000 ms for a lock before raising SQLITE_BUSY
```

Set this (e.g. 5000 ms) on *every* connection in a concurrent app. Even in WAL mode, two would-be writers still serialize, so a writer may briefly wait — `busy_timeout` turns "instant error you must handle" into "tiny pause that usually resolves itself." You may *still* need a retry loop for the rare timeout, but the timeout handles most cases.

### 8.6 `synchronous` — the durability/speed dial

`PRAGMA synchronous` controls how aggressively SQLite forces (fsyncs) data to disk:
- **`FULL` (3)** — fsync at critical moments; safe against power loss. The default in rollback mode.
- **`NORMAL` (1)** — fewer fsyncs; in **WAL mode this is safe** (a crash can't corrupt the DB; you might lose the *very last* committed transaction only on OS/power crash, not on app crash). The standard WAL choice.
- **`OFF` (0)** — no fsyncs; fastest but a power loss *can* corrupt the database. Use only for throwaway/rebuildable data.

```sql
PRAGMA synchronous = NORMAL;   -- recommended companion to WAL
```

---

## 9. PRAGMAs That Matter

**[I]**

A **PRAGMA** is a special SQLite command that queries or sets engine settings (or inspects the schema). Most are **per-connection** and must be re-applied each time you open a connection; a few (like `journal_mode = WAL`) are persisted in the file. Here are the ones that matter for a real app, with recommended settings.

| PRAGMA | What it does | Recommended for an app |
|---|---|---|
| `foreign_keys = ON` | Enforce FK constraints (off by default!). Per-connection. | **`ON`** always (§7.2). |
| `journal_mode = WAL` | Use write-ahead logging; readers don't block the writer. Persisted. | **`WAL`** (§8.4). |
| `synchronous = NORMAL` | Durability vs speed. NORMAL is safe under WAL. Per-connection. | **`NORMAL`** with WAL (§8.6). |
| `busy_timeout = 5000` | Wait (ms) for locks before `SQLITE_BUSY`. Per-connection. | **`5000`** (§8.5). |
| `cache_size = -64000` | Page cache size. Negative = KiB (here ~64 MB); positive = pages. Per-connection. | A negative value sized to your working set (e.g. `-64000`). |
| `temp_store = MEMORY` | Keep temp tables/indexes in RAM. Per-connection. | `MEMORY` for speed if RAM allows. |
| `mmap_size = 268435456` | Memory-map up to N bytes of the DB for faster reads (here 256 MB). | Set on read-heavy DBs that fit; 0 to disable. |
| `foreign_key_check` | Report rows whose FK points nowhere (diagnostic). | Run in maintenance/CI. |
| `integrity_check` | Verify the database isn't corrupt (§15). | Run periodically / before backups. |
| `optimize` | Let SQLite run recommended maintenance (e.g. targeted ANALYZE). | Run `PRAGMA optimize;` before closing long-lived connections. |

### 9.1 The standard "open a connection" PRAGMA preamble

Run this immediately after opening *every* connection in an application (the §13/§14 examples do exactly this):

```sql
PRAGMA journal_mode = WAL;       -- (persisted, but cheap to re-assert)
PRAGMA busy_timeout = 5000;      -- wait for locks instead of erroring instantly
PRAGMA foreign_keys = ON;        -- enforce referential integrity
PRAGMA synchronous = NORMAL;     -- safe + fast under WAL
PRAGMA cache_size = -64000;      -- ~64 MB page cache
PRAGMA temp_store = MEMORY;      -- temp data in RAM
```

### 9.2 Schema-inspection PRAGMAs (handy)

```sql
PRAGMA table_info(users);        -- columns: name, type, notnull, default, pk
PRAGMA table_list;               -- list all tables/views (3.37+)
PRAGMA index_list(users);        -- indexes on a table
PRAGMA foreign_key_list(book);   -- FKs declared on a table
PRAGMA database_list;            -- attached databases & file paths
```

---

## 10. JSON Support

**[I]**

> **⚡ Version note:** JSON functions (the historical "JSON1" extension) are **built into standard SQLite by default since 3.38 (2022)**, including the `->` and `->>` operators. Modern 2026 builds have the full suite without any special flags.

### 10.1 Why JSON in a relational database?

Sometimes data is genuinely semi-structured or schema-flexible — config blobs, third-party API payloads, per-row metadata that varies. Rather than over-normalize or add dozens of nullable columns, you can store a JSON document in a `TEXT` column and query *into* it with SQL. SQLite stores JSON as **TEXT** (or, with `jsonb` functions, a compact binary form internally — 3.45+), and validates/queries it with built-in functions.

> **Use JSON sparingly.** If a field is queried, filtered, or joined often, make it a real column — that's what relational databases are good at (and indexable). Reserve JSON for genuinely variable or rarely-queried structure. See **`RELATIONAL_DB_DESIGN_GUIDE.md`** on not abusing document storage in a relational schema.

### 10.2 Extracting values: `->`, `->>`, and `json_extract`

```sql
CREATE TABLE event (id INTEGER PRIMARY KEY, data TEXT) STRICT;
INSERT INTO event (data) VALUES
  ('{"type":"click","user":{"id":7,"name":"Ann"},"tags":["a","b"]}');

-- json_extract: the explicit function. Path uses $ for root, .key, [index].
SELECT json_extract(data, '$.type')      AS type,    -- 'click'
       json_extract(data, '$.user.id')   AS uid,     -- 7  (returns a SQL integer)
       json_extract(data, '$.tags[0]')   AS first_tag -- 'a'
FROM event;

-- The operators (Postgres-compatible, 3.38+):
--   ->   returns a JSON representation (still JSON; e.g. a quoted string)
--   ->>  returns a SQL scalar (text/number) — usually what you want
SELECT data ->> '$.type'      AS type,      -- 'click'  (SQL text)
       data ->  '$.user'      AS user_json, -- '{"id":7,"name":"Ann"}' (JSON text)
       data ->> '$.user.name' AS name       -- 'Ann'
FROM event;

-- You can omit the leading $ with the operators: data ->> 'type' also works for top-level keys.
```

### 10.3 Building JSON: `json_object`, `json_array`

```sql
SELECT json_object('id', id, 'type', data ->> '$.type') FROM event;
-- {"id":1,"type":"click"}

SELECT json_array(1, 'two', 3.0, NULL);   -- [1,"two",3.0,null]

-- Aggregate rows into a JSON array/object:
SELECT json_group_array(email) FROM users;          -- ["a@b.com","c@d.com",...]
SELECT json_group_object(email, role) FROM users;   -- {"a@b.com":"member",...}
```

### 10.4 Iterating JSON: `json_each` / `json_tree` (table-valued functions)

`json_each` is a **table-valued function**: it turns the elements of a JSON array/object into *rows* you can join against and filter — bridging documents back into relational queries.

```sql
-- Expand the tags array into rows, one per tag:
SELECT e.id, t.value AS tag
FROM event e, json_each(e.data, '$.tags') AS t;
-- (1,'a'), (1,'b')

-- Find events that have a specific tag:
SELECT DISTINCT e.id
FROM event e, json_each(e.data, '$.tags') t
WHERE t.value = 'a';
```

### 10.5 Validating, updating, and indexing JSON

```sql
SELECT json_valid('{"a":1}');                       -- 1 (valid)
UPDATE event SET data = json_set(data, '$.seen', json('true')) WHERE id = 1;  -- add/replace a key
UPDATE event SET data = json_remove(data, '$.tags[0]') WHERE id = 1;          -- delete a path

-- Index a JSON field for fast lookups (expression index over the extracted value):
CREATE INDEX idx_event_type ON event(json_extract(data, '$.type'));
-- Now this is index-assisted:
SELECT * FROM event WHERE json_extract(data, '$.type') = 'click';
```

> **⚡ Version note:** SQLite 3.45+ added **`jsonb`** functions storing a compact binary JSON in BLOBs (`jsonb_extract`, `jsonb_set`, …), faster to query than re-parsing text each time. The `json_*` (text) functions remain the portable default; reach for `jsonb_*` when JSON access is hot.

---

## 11. Full-Text Search (FTS5)

**[I/A]**

### 11.1 What FTS5 is and when to use it

A normal `LIKE '%word%'` query cannot use an index and scans every row — fine for tiny tables, hopeless for searching thousands of documents. **FTS5** is SQLite's built-in **full-text search** engine, implemented as a *virtual table* that maintains an inverted index (word → documents). It gives you fast keyword search, phrase queries, boolean operators, prefix matching, and relevance **ranking** — all inside your single SQLite file, no external search server (Elasticsearch, etc.) needed. Use it for searching articles, product descriptions, notes, logs, documentation.

### 11.2 Creating an FTS5 table

A virtual FTS5 table stores the searchable text columns. A common pattern is the **"external content"** table that mirrors a real table to avoid duplicating data, but the simplest start is a standalone FTS table.

```sql
-- Simple standalone FTS5 table (it stores the text itself):
CREATE VIRTUAL TABLE docs USING fts5(title, body);

INSERT INTO docs (title, body) VALUES
  ('SQLite intro', 'SQLite is an embedded serverless database engine.'),
  ('Postgres intro', 'PostgreSQL is a powerful client server database.'),
  ('WAL mode', 'Write ahead logging lets readers and a writer work concurrently.');
```

### 11.3 Searching with `MATCH` and ranking

The `MATCH` operator runs a full-text query. FTS5 supports a rich query syntax; results can be ordered by **`rank`** (a built-in relevance score using bm25; ordering by `rank` ascending puts the best matches first).

```sql
-- Basic keyword search (any row containing the term):
SELECT title FROM docs WHERE docs MATCH 'database';

-- Phrase search (exact sequence):
SELECT title FROM docs WHERE docs MATCH '"serverless database"';

-- Boolean operators: AND, OR, NOT  (AND is implicit between terms):
SELECT title FROM docs WHERE docs MATCH 'database NOT server';

-- Prefix search (terms starting with "data"):
SELECT title FROM docs WHERE docs MATCH 'data*';

-- Column filter (search only the title column):
SELECT title FROM docs WHERE docs MATCH 'title:intro';

-- Ranked results, best first (bm25 relevance; rank ascending = best first):
SELECT title, rank FROM docs WHERE docs MATCH 'database' ORDER BY rank;

-- Highlighted snippets around the match (auxiliary functions):
SELECT highlight(docs, 1, '[', ']') AS snippet     -- wrap matches in the body (column index 1)
FROM docs WHERE docs MATCH 'serverless';
-- 'SQLite is an embedded [serverless] database engine.'

SELECT snippet(docs, 1, '<b>', '</b>', '…', 8) FROM docs WHERE docs MATCH 'concurrently';
```

### 11.4 External-content FTS (don't duplicate your data) **[A]**

When you already have a real `article` table, you can make FTS5 index it *without storing a second copy* by using `content=`, and keep them in sync with triggers (§12.3).

```sql
CREATE TABLE article (id INTEGER PRIMARY KEY, title TEXT, body TEXT) STRICT;

-- FTS index that points back at `article`; `content_rowid` ties FTS rows to article.id.
CREATE VIRTUAL TABLE article_fts USING fts5(
  title, body,
  content='article',
  content_rowid='id'
);

-- Keep the index in sync via triggers:
CREATE TRIGGER article_ai AFTER INSERT ON article BEGIN
  INSERT INTO article_fts(rowid, title, body) VALUES (new.id, new.title, new.body);
END;
CREATE TRIGGER article_ad AFTER DELETE ON article BEGIN
  INSERT INTO article_fts(article_fts, rowid, title, body) VALUES('delete', old.id, old.title, old.body);
END;
CREATE TRIGGER article_au AFTER UPDATE ON article BEGIN
  INSERT INTO article_fts(article_fts, rowid, title, body) VALUES('delete', old.id, old.title, old.body);
  INSERT INTO article_fts(rowid, title, body) VALUES (new.id, new.title, new.body);
END;

-- Search, then join back for full rows:
SELECT a.* FROM article_fts f JOIN article a ON a.id = f.rowid
WHERE article_fts MATCH 'serverless' ORDER BY f.rank;
```

> **Tip:** Use the `porter` tokenizer for English stemming (so "running" matches "run"): `CREATE VIRTUAL TABLE docs USING fts5(body, tokenize='porter');`. There's also `trigram` (3.34+) for substring/`LIKE`-style and `unicode61` (default, Unicode-aware).

---

## 12. Other Features

**[I]**

### 12.1 Generated (computed) columns

> **⚡ Version note:** Generated columns require **SQLite 3.31+ (2020)**.

A **generated column** computes its value from other columns via an expression, so you never store it inconsistently. Two flavours:
- **`VIRTUAL`** (default) — not stored; computed on read. Costs nothing on disk, a little on read.
- **`STORED`** — computed on write and saved to disk; costs space, faster on read, and can be indexed directly.

```sql
CREATE TABLE invoice (
  id       INTEGER PRIMARY KEY,
  qty      INTEGER NOT NULL,
  unit     INTEGER NOT NULL,            -- price in cents
  total    INTEGER GENERATED ALWAYS AS (qty * unit) VIRTUAL,  -- computed each read
  total_st INTEGER GENERATED ALWAYS AS (qty * unit) STORED    -- computed on write, on disk
) STRICT;

INSERT INTO invoice (qty, unit) VALUES (3, 500);   -- total -> 1500 automatically
SELECT total, total_st FROM invoice;               -- 1500, 1500

-- You can index a generated column (especially STORED) just like a normal one:
CREATE INDEX idx_invoice_total ON invoice(total);
```

### 12.2 Views

A **view** is a saved `SELECT` you can query like a table. It stores no data — it's a named query that runs each time you select from it. Use views to encapsulate complex joins/filters behind a simple name and present a stable interface.

```sql
CREATE VIEW active_admins AS
  SELECT id, email FROM users WHERE is_active = 1 AND role = 'admin';

SELECT * FROM active_admins;     -- runs the underlying query

DROP VIEW active_admins;
```

Views are **read-only** in SQLite by default; to make them "writable", attach `INSTEAD OF` triggers (below).

### 12.3 Triggers

A **trigger** is SQL that runs automatically in response to `INSERT`, `UPDATE`, or `DELETE` on a table (`BEFORE`, `AFTER`, or `INSTEAD OF` the event). Use them for audit logs, maintaining derived/denormalized data, enforcing complex rules, and keeping FTS indexes in sync (§11.4). Inside a trigger, `NEW` refers to the new row (insert/update) and `OLD` to the prior row (update/delete).

```sql
-- Maintain an updated_at timestamp automatically on every UPDATE.
CREATE TABLE note (
  id INTEGER PRIMARY KEY, body TEXT, updated_at TEXT
) STRICT;

CREATE TRIGGER note_touch
AFTER UPDATE ON note
FOR EACH ROW
BEGIN
  UPDATE note SET updated_at = datetime('now') WHERE id = NEW.id;
END;

-- Audit log: record every deletion.
CREATE TABLE note_audit (note_id INTEGER, deleted_at TEXT) STRICT;
CREATE TRIGGER note_log_delete
AFTER DELETE ON note
BEGIN
  INSERT INTO note_audit (note_id, deleted_at) VALUES (OLD.id, datetime('now'));
END;
```

> **Caution:** triggers run *implicitly*; overusing them makes behaviour hard to follow. Keep them simple and well-documented.

### 12.4 Dates and times (and the lack of a native date type)

SQLite has **no dedicated date/time storage type** (recall §3.2). Dates are stored as one of:
- **TEXT** in ISO-8601 (`'2026-06-22 10:30:00'` or with `T`/`Z`). **Recommended default** — sorts correctly, human-readable.
- **INTEGER** Unix epoch seconds (`1750000000`). Compact; good for arithmetic.
- **REAL** Julian day numbers. Rarely used directly.

A family of **date/time functions** parses and formats these:

```sql
SELECT date('now');                       -- '2026-06-22'  (UTC date)
SELECT time('now');                       -- '10:30:00'
SELECT datetime('now');                   -- '2026-06-22 10:30:00'  (UTC)
SELECT datetime('now', 'localtime');      -- convert to local time
SELECT strftime('%Y-%m-%d %H:%M', 'now'); -- custom format
SELECT unixepoch('now');                  -- Unix seconds (3.38+); or strftime('%s','now')
SELECT julianday('now');                  -- Julian day number

-- Date arithmetic via modifiers:
SELECT date('now', '+7 days');            -- a week from now
SELECT datetime('now', '-1 month', 'start of month');  -- first of last month
SELECT date('2026-06-22', 'weekday 0');   -- next Sunday

-- Difference between two timestamps (e.g. days):
SELECT julianday('2026-12-25') - julianday('now') AS days_until_xmas;
```

> **Always store UTC** and convert to local only for display. `datetime('now')` and `CURRENT_TIMESTAMP` are UTC. Storing local times invites DST bugs.

### 12.5 Math, string, and other built-in functions

> **⚡ Version note:** Math functions (`sqrt`, `pow`, `ln`, `sin`, `floor`, …) require the build flag `SQLITE_ENABLE_MATH_FUNCTIONS`, **on by default in standard 3.35+ builds and the CLI**. Confirm with `SELECT sqrt(2);`.

```sql
-- Math:
SELECT abs(-5), round(3.14159, 2), ceil(2.1), floor(2.9), pow(2, 10), sqrt(2), pi();

-- Strings:
SELECT upper('hi'), lower('HI'), length('abc'), substr('abcdef', 2, 3),  -- 'bcd'
       replace('a-b-c', '-', '_'), trim('  x  '), instr('abc', 'b'),     -- 'a_b_c', 'x', 2
       'Hello' || ' ' || 'World',                                        -- || is concat
       printf('%s is %d', 'x', 7),                                       -- formatted string
       format('%.2f', 3.14159);                                          -- '3.14' (alias of printf)

-- Null handling:
SELECT coalesce(NULL, NULL, 'fallback'),   -- first non-NULL -> 'fallback'
       ifnull(NULL, 0),                     -- 0
       nullif(5, 5);                        -- NULL when equal

-- Misc:
SELECT random(),            -- random integer
       hex(x'4869'),        -- 'Hi' bytes -> hex
       quote('O''Brien');   -- safely quote for SQL
```

---

## 13. Node.js Usage

**[I/A]**

Node has **two** good ways to use SQLite in 2026. Both run the engine *in your Node process* (no server), which is exactly the SQLite model.

1. **`better-sqlite3`** — the long-standing, extremely popular npm package. It is **synchronous** and **fast**. Synchronous sounds wrong for Node, but for an embedded, in-process database it's actually ideal: a local SQLite query takes microseconds, so the overhead of going async (callbacks/promises, event-loop hops) is *larger* than the query itself. Synchronous code is also simpler and makes transactions trivial (no interleaving). It's a native addon (compiled C++), so it needs a build toolchain or prebuilt binaries.
2. **`node:sqlite`** — the **built-in** module that ships *with Node itself* (no install). It landed experimentally in Node 22 and has been stabilizing through Node 23/24 into 2026. Its API is deliberately similar to better-sqlite3 (synchronous, prepared statements). Use it when you want zero dependencies.

> **⚡ Version note:** `node:sqlite` is new and was marked *experimental* in Node 22 (you needed `--experimental-sqlite`); it has been moving toward stable in Node 23/24. Check `node --version` and the docs for your release — on older Node you must pass the flag, and the API surface keeps growing. `better-sqlite3` remains the safe, battle-tested choice for production today.

### 13.1 Installing

```bash
npm install better-sqlite3          # native addon; prebuilt binaries exist for common platforms
# node:sqlite needs NO install — it's built in:
#   const { DatabaseSync } = require('node:sqlite');
```

> **Windows note:** `better-sqlite3` ships prebuilt binaries for current Node LTS on Windows x64, so `npm install` usually just works. If it tries to compile, install the build tools (`npm install --global windows-build-tools` historically, or the "Desktop development with C++" workload in Visual Studio Build Tools).

### 13.2 A shared database module (the PRAGMA preamble in code)

Centralize opening the DB so the standard PRAGMAs (§9.1) are applied exactly once, consistently. This is `db.js`:

```js
// db.js — better-sqlite3 version
const Database = require('better-sqlite3');

// Open (creates the file if missing). verbose logs every statement in dev.
const db = new Database('app.db', {
  // verbose: console.log,   // uncomment to trace SQL during development
});

// --- The standard connection preamble (see §9.1) ---
db.pragma('journal_mode = WAL');     // readers don't block the single writer
db.pragma('busy_timeout = 5000');    // wait up to 5s for a lock instead of erroring
db.pragma('foreign_keys = ON');      // ENFORCE foreign keys (off by default! §7.2)
db.pragma('synchronous = NORMAL');   // safe + fast under WAL (§8.6)
db.pragma('cache_size = -64000');    // ~64 MB page cache
db.pragma('temp_store = MEMORY');    // temp tables/indexes in RAM

module.exports = db;
```

> **Connection model in Node:** because better-sqlite3 is synchronous and single-threaded, you typically open **one** `Database` object for the whole process and reuse it everywhere. There is no connection pool to manage (the engine is in-process). If you use Node Worker Threads, give *each worker its own* connection — a `Database` handle is not shareable across threads.

### 13.3 Schema & migrations

Create the schema once at startup (idempotently). For real projects use a migration tool, but `IF NOT EXISTS` plus a `user_version` PRAGMA is a fine lightweight scheme.

```js
// schema.js
const db = require('./db');

db.exec(`
  CREATE TABLE IF NOT EXISTS task (
    id          INTEGER PRIMARY KEY,
    title       TEXT    NOT NULL,
    done        INTEGER NOT NULL DEFAULT 0 CHECK (done IN (0, 1)),  -- boolean as 0/1
    created_at  TEXT    NOT NULL DEFAULT (datetime('now'))
  ) STRICT;

  CREATE INDEX IF NOT EXISTS idx_task_done ON task(done);
`);
```

### 13.4 Prepared statements — the core API

A **prepared statement** is a compiled, reusable query with **bound parameters** (placeholders) instead of values spliced into the SQL string. You should *always* use them because they (a) prevent **SQL injection** — user input can never be interpreted as SQL — and (b) are faster when reused, since SQLite compiles the SQL once and you just rebind values.

better-sqlite3 gives each prepared statement three execution methods:
- **`.run(params)`** — for `INSERT`/`UPDATE`/`DELETE`/DDL; returns `{ changes, lastInsertRowid }`.
- **`.get(params)`** — returns the **first** row (or `undefined`).
- **`.all(params)`** — returns **all** rows as an array of objects.
- **`.iterate(params)`** — returns an iterator (stream large result sets without loading all into memory).

Parameters can be **positional** (`?`) or **named** (`@name`, `:name`, `$name`).

```js
const db = require('./db');

// Prepare once (e.g. at module load); reuse many times.
const insertTask = db.prepare(
  'INSERT INTO task (title) VALUES (@title) RETURNING id, created_at'  // named param + RETURNING
);
const getTask    = db.prepare('SELECT * FROM task WHERE id = ?');       // positional param
const listTasks  = db.prepare('SELECT * FROM task ORDER BY id');
const completeTask = db.prepare('UPDATE task SET done = 1 WHERE id = ?');
const deleteTask = db.prepare('DELETE FROM task WHERE id = ?');

// Create:
const row = insertTask.get({ title: 'Write the SQLite guide' }); // .get because of RETURNING
console.log(row);  // { id: 1, created_at: '2026-06-22 10:30:00' }

// Read:
const one  = getTask.get(1);     // { id: 1, title: '...', done: 0, created_at: '...' }
const many = listTasks.all();    // [ {...}, {...} ]

// Update / Delete:
const upd = completeTask.run(1); // { changes: 1, lastInsertRowid: 0n }
const del = deleteTask.run(1);   // { changes: 1, ... }
```

> **Note:** `lastInsertRowid` is a **BigInt** in better-sqlite3 (SQLite rowids are 64-bit). Convert with `Number(...)` if it's small enough; prefer `RETURNING id` to get it cleanly.

### 13.5 Transactions — the safe, fast wrapper

For multiple related writes (and for bulk inserts — see §16), wrap them in a transaction. better-sqlite3 has an elegant `db.transaction()` helper: it returns a function that runs your callback inside `BEGIN`/`COMMIT`, automatically `ROLLBACK`-ing if the callback throws. Nested calls become savepoints automatically.

```js
const db = require('./db');

const insertOne = db.prepare('INSERT INTO task (title) VALUES (?)');

// Build a transaction that inserts MANY tasks atomically and fast.
const insertManyTasks = db.transaction((titles) => {
  for (const t of titles) insertOne.run(t);
  return titles.length;             // the return value is passed back out
});

// Calling it runs everything in ONE transaction:
const n = insertManyTasks(['a', 'b', 'c']);  // all-or-nothing; if any throws, all roll back
console.log(`inserted ${n}`);

// A money-transfer example (atomicity matters):
const debit  = db.prepare('UPDATE account SET balance = balance - ? WHERE id = ?');
const credit = db.prepare('UPDATE account SET balance = balance + ? WHERE id = ?');
const transfer = db.transaction((from, to, amount) => {
  const r = debit.run(amount, from);
  if (r.changes !== 1) throw new Error('debit failed');     // throwing => automatic ROLLBACK
  credit.run(amount, to);
});
transfer(1, 2, 100);   // both updates commit together, or neither does
```

### 13.6 Worked example: a Tasks REST API with **Express**

A complete, idiomatic Express service using `db.js`/`schema.js` above. Note how prepared statements are created once and reused per request, and how a transaction guards a bulk endpoint.

```js
// server.js — Express + better-sqlite3
const express = require('express');
const db = require('./db');
require('./schema');                  // ensure tables exist

const app = express();
app.use(express.json());              // parse JSON request bodies

// Prepare statements ONCE at startup (not per-request) for speed:
const stmts = {
  list:     db.prepare('SELECT * FROM task ORDER BY id'),
  get:      db.prepare('SELECT * FROM task WHERE id = ?'),
  create:   db.prepare('INSERT INTO task (title) VALUES (@title) RETURNING *'),
  complete: db.prepare('UPDATE task SET done = 1 WHERE id = ? RETURNING *'),
  remove:   db.prepare('DELETE FROM task WHERE id = ?'),
};

// Bulk insert as a transaction (see §13.5):
const createMany = db.transaction((titles) =>
  titles.map((title) => stmts.create.get({ title }))
);

app.get('/tasks', (req, res) => {
  res.json(stmts.list.all());         // synchronous call — returns instantly
});

app.get('/tasks/:id', (req, res) => {
  const task = stmts.get.get(Number(req.params.id));
  if (!task) return res.status(404).json({ error: 'not found' });
  res.json(task);
});

app.post('/tasks', (req, res) => {
  const { title } = req.body;
  if (!title) return res.status(400).json({ error: 'title required' });
  const task = stmts.create.get({ title });   // .get because of RETURNING *
  res.status(201).json(task);
});

app.post('/tasks/bulk', (req, res) => {
  const { titles } = req.body;        // ["a","b",...]
  if (!Array.isArray(titles)) return res.status(400).json({ error: 'titles[] required' });
  res.status(201).json(createMany(titles));    // one transaction for all inserts
});

app.patch('/tasks/:id/complete', (req, res) => {
  const task = stmts.complete.get(Number(req.params.id));
  if (!task) return res.status(404).json({ error: 'not found' });
  res.json(task);
});

app.delete('/tasks/:id', (req, res) => {
  const r = stmts.remove.run(Number(req.params.id));
  if (r.changes === 0) return res.status(404).json({ error: 'not found' });
  res.status(204).end();
});

app.listen(3000, () => console.log('http://localhost:3000'));
```

### 13.7 The same with the built-in `node:sqlite`

The API is intentionally close to better-sqlite3, so porting is mostly mechanical. Methods differ slightly: `db.prepare(sql)` returns a statement with `.run()`, `.get()`, `.all()`; binding uses positional or named params.

```js
// db-builtin.js — using Node's built-in module (no npm install)
// On Node 22 you may need: node --experimental-sqlite server.js
const { DatabaseSync } = require('node:sqlite');

const db = new DatabaseSync('app.db');

// Apply the same PRAGMA preamble. node:sqlite uses .exec() for raw SQL:
db.exec('PRAGMA journal_mode = WAL');
db.exec('PRAGMA busy_timeout = 5000');
db.exec('PRAGMA foreign_keys = ON');
db.exec('PRAGMA synchronous = NORMAL');

db.exec(`
  CREATE TABLE IF NOT EXISTS task (
    id    INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    done  INTEGER NOT NULL DEFAULT 0
  ) STRICT;
`);

const insert = db.prepare('INSERT INTO task (title) VALUES (?)');
const list   = db.prepare('SELECT * FROM task ORDER BY id');

const info = insert.run('learn node:sqlite');   // { changes, lastInsertRowid }
console.log(info.lastInsertRowid);
console.log(list.all());                         // [ { id, title, done } ]

// node:sqlite has no built-in db.transaction() helper — use explicit SQL:
db.exec('BEGIN');
try {
  insert.run('a');
  insert.run('b');
  db.exec('COMMIT');
} catch (e) {
  db.exec('ROLLBACK');
  throw e;
}

module.exports = db;
```

> **Choosing between them:** Use **`better-sqlite3`** for production today — mature, fast, rich API (`db.transaction`, user-defined functions, backups, etc.). Use **`node:sqlite`** when you want zero dependencies, are on a recent Node, and your needs are basic. Both are synchronous and follow the same mental model.

### 13.8 Using it from **NestJS**

In NestJS, wrap the database in an injectable **provider** so the rest of the app depends on an abstraction, and apply the PRAGMA preamble in the provider's lifecycle. The pattern: a `DatabaseService` that owns the `better-sqlite3` instance, plus feature services that prepare and run statements.

```ts
// database.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import Database from 'better-sqlite3';

@Injectable()
export class DatabaseService implements OnModuleInit, OnModuleDestroy {
  public readonly db = new Database(process.env.DB_PATH ?? 'app.db');

  onModuleInit() {
    // Standard preamble + schema, once when the module initializes:
    this.db.pragma('journal_mode = WAL');
    this.db.pragma('busy_timeout = 5000');
    this.db.pragma('foreign_keys = ON');
    this.db.pragma('synchronous = NORMAL');
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS task (
        id INTEGER PRIMARY KEY, title TEXT NOT NULL, done INTEGER NOT NULL DEFAULT 0
      ) STRICT;
    `);
  }

  onModuleDestroy() {
    this.db.pragma('optimize');  // recommended before closing a long-lived connection (§9)
    this.db.close();
  }
}
```

```ts
// tasks.service.ts
import { Injectable } from '@nestjs/common';
import { DatabaseService } from './database.service';
import type { Statement } from 'better-sqlite3';

@Injectable()
export class TasksService {
  private readonly listStmt: Statement;
  private readonly createStmt: Statement;
  private readonly createMany: (titles: string[]) => unknown[];

  constructor(private readonly database: DatabaseService) {
    const db = this.database.db;
    // Prepare statements once in the constructor (singleton service => prepared once):
    this.listStmt = db.prepare('SELECT * FROM task ORDER BY id');
    this.createStmt = db.prepare('INSERT INTO task (title) VALUES (@title) RETURNING *');
    this.createMany = db.transaction((titles: string[]) =>
      titles.map((title) => this.createStmt.get({ title })),
    );
  }

  findAll() { return this.listStmt.all(); }
  create(title: string) { return this.createStmt.get({ title }); }
  bulk(titles: string[]) { return this.createMany(titles); }
}
```

```ts
// tasks.controller.ts
import { Body, Controller, Get, Post } from '@nestjs/common';
import { TasksService } from './tasks.service';

@Controller('tasks')
export class TasksController {
  constructor(private readonly tasks: TasksService) {}

  @Get()  findAll() { return this.tasks.findAll(); }
  @Post() create(@Body('title') title: string) { return this.tasks.create(title); }
  @Post('bulk') bulk(@Body('titles') titles: string[]) { return this.tasks.bulk(titles); }
}
```

> Because better-sqlite3 is synchronous, NestJS controller methods can simply `return` the result — no `async`/`await` needed for the DB call itself. (You can still mark them `async` if other async work is involved.)

---

## 14. Go Usage

**[I/A]**

Go talks to SQLite through the standard **`database/sql`** package plus a *driver*. There are two main drivers, and the choice has real consequences:

1. **`modernc.org/sqlite`** — a **pure-Go**, **cgo-free** translation of SQLite to Go. **Why cgo-free matters:** with cgo, your build needs a C compiler, cross-compilation becomes painful (you need a C toolchain for each target OS/arch), builds are slower, and you can't produce a fully static binary as easily. A pure-Go driver means `CGO_ENABLED=0 go build` just works, you cross-compile trivially (great for shipping a single static binary to Alpine/scratch containers or to ARM edge devices), and there's no C toolchain dependency. The trade-off: it's a re-implementation, so it can lag the C library slightly and is marginally slower in some workloads — but for the vast majority of apps it's the pragmatic default in 2026.
2. **`mattn/go-sqlite3`** — the long-established driver that **binds the real C SQLite via cgo**. It's fast and battle-tested and always tracks the canonical engine, but it **requires cgo** (a C compiler at build time) and complicates cross-compilation and static linking.

> **Recommendation:** Default to **`modernc.org/sqlite`** for the cgo-free build and deployment simplicity. Choose **`mattn/go-sqlite3`** when you need a specific C-level feature, maximum raw performance, or to load C extensions. The `database/sql` code below is *identical* for both — only the import and driver name change.

### 14.1 Installing

```bash
go get modernc.org/sqlite           # pure-Go, cgo-free (driver name: "sqlite")
# or:
go get github.com/mattn/go-sqlite3  # cgo-based (driver name: "sqlite3"); needs a C compiler
```

### 14.2 Opening the database with the PRAGMA preamble

`database/sql` manages a **connection pool**. This matters enormously for SQLite: since there is only one writer, letting Go open *many* pooled connections that all try to write leads to `SQLITE_BUSY` contention. A common, robust pattern is to **set `MaxOpenConns(1)`** (serialize all access through one connection) for the writer, *or* configure WAL + busy_timeout and cap connections low. Crucially, **PRAGMAs are per-connection**, and the pool may open several connections lazily — so you pass the important PRAGMAs in the **DSN (connection string)** so *every* pooled connection gets them.

```go
package main

import (
	"database/sql"
	"log"

	_ "modernc.org/sqlite" // register the pure-Go driver under the name "sqlite"
	// _ "github.com/mattn/go-sqlite3" // alternative; driver name would be "sqlite3"
)

func openDB(path string) (*sql.DB, error) {
	// PRAGMAs passed via the DSN so EVERY pooled connection applies them.
	// modernc.org/sqlite uses "_pragma=NAME(value)" query params.
	dsn := path +
		"?_pragma=journal_mode(WAL)" +
		"&_pragma=busy_timeout(5000)" +
		"&_pragma=foreign_keys(ON)" +
		"&_pragma=synchronous(NORMAL)"
	// (mattn/go-sqlite3 uses a slightly different DSN syntax, e.g.
	//  "file:app.db?_journal_mode=WAL&_busy_timeout=5000&_foreign_keys=on")

	db, err := sql.Open("sqlite", dsn) // "sqlite3" for the mattn driver
	if err != nil {
		return nil, err
	}

	// WAL allows concurrent readers + one writer. We let reads use the pool but
	// keep writes from thundering: a single open connection is the simplest safe choice.
	db.SetMaxOpenConns(1) // serialize access; bump up only if you've measured & use WAL carefully
	db.SetMaxIdleConns(1)

	if err := db.Ping(); err != nil { // force-open a connection so PRAGMAs apply now
		return nil, err
	}
	return db, nil
}

func migrate(db *sql.DB) error {
	_, err := db.Exec(`
		CREATE TABLE IF NOT EXISTS task (
			id         INTEGER PRIMARY KEY,
			title      TEXT    NOT NULL,
			done       INTEGER NOT NULL DEFAULT 0 CHECK (done IN (0,1)),
			created_at TEXT    NOT NULL DEFAULT (datetime('now'))
		) STRICT;
		CREATE INDEX IF NOT EXISTS idx_task_done ON task(done);
	`)
	return err
}

func main() {
	db, err := openDB("app.db")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()
	if err := migrate(db); err != nil {
		log.Fatal(err)
	}
	log.Println("db ready")
}
```

### 14.3 Queries & prepared statements in `database/sql`

`database/sql` always uses bound parameters (`?`), so you get SQL-injection safety for free. The key methods:
- **`db.Exec(sql, args...)`** — for `INSERT`/`UPDATE`/`DELETE`/DDL; returns a `Result` (`LastInsertId`, `RowsAffected`).
- **`db.QueryRow(sql, args...)`** — fetch a single row; `.Scan(&dest...)` into variables.
- **`db.Query(sql, args...)`** — fetch many rows; iterate with `rows.Next()` + `rows.Scan(...)`.
- **`db.Prepare(sql)`** — explicit prepared statement for heavy reuse (the pool also caches prepared statements behind the scenes).

```go
type Task struct {
	ID        int64
	Title     string
	Done      bool
	CreatedAt string
}

// Create, returning the new row via RETURNING + QueryRow:
func createTask(db *sql.DB, title string) (Task, error) {
	var t Task
	err := db.QueryRow(
		`INSERT INTO task (title) VALUES (?) RETURNING id, title, done, created_at`,
		title,
	).Scan(&t.ID, &t.Title, &t.Done, &t.CreatedAt) // SQLite 0/1 scans cleanly into a bool
	return t, err
}

// List many rows:
func listTasks(db *sql.DB) ([]Task, error) {
	rows, err := db.Query(`SELECT id, title, done, created_at FROM task ORDER BY id`)
	if err != nil {
		return nil, err
	}
	defer rows.Close() // ALWAYS close rows to release the connection back to the pool

	var out []Task
	for rows.Next() {
		var t Task
		if err := rows.Scan(&t.ID, &t.Title, &t.Done, &t.CreatedAt); err != nil {
			return nil, err
		}
		out = append(out, t)
	}
	return out, rows.Err() // check for iteration errors
}
```

### 14.4 Transactions (and why they're essential for bulk writes)

A `*sql.Tx` runs statements on one connection inside `BEGIN`/`COMMIT`. For bulk inserts this is the difference between thousands of separate disk syncs and one — a 50–100x speedup (§16). The idiom is `defer tx.Rollback()` (a no-op after a successful `Commit`).

```go
func insertManyTasks(db *sql.DB, titles []string) error {
	tx, err := db.Begin() // for a known-write txn you can use BeginTx with options
	if err != nil {
		return err
	}
	defer tx.Rollback() // safe: becomes a no-op once we Commit successfully

	stmt, err := tx.Prepare(`INSERT INTO task (title) VALUES (?)`) // prepared within the txn
	if err != nil {
		return err
	}
	defer stmt.Close()

	for _, title := range titles {
		if _, err := stmt.Exec(title); err != nil {
			return err // deferred Rollback undoes everything
		}
	}
	return tx.Commit() // all inserts persist together
}
```

### 14.5 Worked example: the Tasks API with **Gin**

This mirrors the Node/Express service from §13.6 — same endpoints, same transaction-for-bulk pattern, using Gin for routing.

```go
package main

import (
	"database/sql"
	"errors"
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"
	_ "modernc.org/sqlite"
)

type Task struct {
	ID        int64  `json:"id"`
	Title     string `json:"title"`
	Done      bool   `json:"done"`
	CreatedAt string `json:"created_at"`
}

var db *sql.DB

func main() {
	var err error
	db, err = openDB("app.db") // from §14.2 (WAL + busy_timeout + foreign_keys via DSN)
	if err != nil {
		panic(err)
	}
	defer db.Close()
	if err := migrate(db); err != nil {
		panic(err)
	}

	r := gin.Default()

	// LIST
	r.GET("/tasks", func(c *gin.Context) {
		tasks, err := listTasks(db)
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		c.JSON(http.StatusOK, tasks)
	})

	// GET one
	r.GET("/tasks/:id", func(c *gin.Context) {
		id, _ := strconv.ParseInt(c.Param("id"), 10, 64)
		var t Task
		err := db.QueryRow(
			`SELECT id, title, done, created_at FROM task WHERE id = ?`, id,
		).Scan(&t.ID, &t.Title, &t.Done, &t.CreatedAt)
		if errors.Is(err, sql.ErrNoRows) {
			c.JSON(http.StatusNotFound, gin.H{"error": "not found"})
			return
		} else if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		c.JSON(http.StatusOK, t)
	})

	// CREATE
	r.POST("/tasks", func(c *gin.Context) {
		var body struct {
			Title string `json:"title"`
		}
		if err := c.ShouldBindJSON(&body); err != nil || body.Title == "" {
			c.JSON(http.StatusBadRequest, gin.H{"error": "title required"})
			return
		}
		t, err := createTask(db, body.Title)
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		c.JSON(http.StatusCreated, t)
	})

	// BULK CREATE (one transaction)
	r.POST("/tasks/bulk", func(c *gin.Context) {
		var body struct {
			Titles []string `json:"titles"`
		}
		if err := c.ShouldBindJSON(&body); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": "titles[] required"})
			return
		}
		if err := insertManyTasks(db, body.Titles); err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		c.JSON(http.StatusCreated, gin.H{"inserted": len(body.Titles)})
	})

	// COMPLETE
	r.PATCH("/tasks/:id/complete", func(c *gin.Context) {
		id, _ := strconv.ParseInt(c.Param("id"), 10, 64)
		res, err := db.Exec(`UPDATE task SET done = 1 WHERE id = ?`, id)
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		if n, _ := res.RowsAffected(); n == 0 {
			c.JSON(http.StatusNotFound, gin.H{"error": "not found"})
			return
		}
		c.Status(http.StatusNoContent)
	})

	// DELETE
	r.DELETE("/tasks/:id", func(c *gin.Context) {
		id, _ := strconv.ParseInt(c.Param("id"), 10, 64)
		res, _ := db.Exec(`DELETE FROM task WHERE id = ?`, id)
		if n, _ := res.RowsAffected(); n == 0 {
			c.JSON(http.StatusNotFound, gin.H{"error": "not found"})
			return
		}
		c.Status(http.StatusNoContent)
	})

	r.Run(":8080") // http://localhost:8080
}
```

> **Cross-reference:** For the broader Gin REST/file-upload patterns (middleware, binding, validation), see **`GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`**. For Postgres from Go (pgx) and the ORM route, see **`POSTGRESQL_GUIDE.md`** §21 and **`GO_ENT_ORM_GUIDE.md`**.

---

## 15. Backup, Maintenance & Operations

**[I/A]**

### 15.1 Backups — the right and wrong ways

Because the database is a single file, you might think `cp app.db backup.db` is a backup. **It is not safe while the database is in use** — a concurrent write (and, in WAL mode, the `-wal`/`-shm` sidecars) can leave you with a torn, inconsistent copy. Use a method that takes a consistent snapshot:

**`.backup` (online backup) — the safest live backup:**

```bash
# From the CLI, copies a consistent snapshot even while the DB is being written:
sqlite3 app.db ".backup backup.db"
# or from inside the shell:  .backup main backup.db
```

This uses SQLite's **online backup API**, which copies pages while holding appropriate locks, producing a valid standalone database. better-sqlite3 exposes `db.backup('backup.db')`; in Go you can run `VACUUM INTO` (below) or use the C backup API via the driver.

**`VACUUM INTO` — snapshot to a new, compacted file (3.27+):**

```sql
VACUUM INTO 'backup.db';   -- writes a fresh, defragmented copy; safe online snapshot
```

**`.dump` — logical (SQL text) backup:**

```bash
sqlite3 app.db ".dump" > dump.sql            # entire DB as a SQL script (schema + INSERTs)
sqlite3 restored.db < dump.sql               # rebuild from the script
```

A `.dump` is portable (plain SQL), great for version control of schema/seed data and for moving between SQLite versions, but slower to restore than a binary copy for large databases.

> **⚡ Continuous replication (2026):** For production web use, tools like **Litestream** stream the WAL to object storage (S3) for point-in-time recovery, and **LiteFS** replicates a SQLite file across nodes. These turn SQLite into a far more operationally robust backend without changing your app's SQL. Worth knowing about for "SQLite in production on the web."

### 15.2 `VACUUM` — reclaim space and defragment

When you delete data, SQLite doesn't shrink the file — it keeps the freed pages for reuse (so the file only grows). Over time the file can become fragmented and bloated. **`VACUUM`** rebuilds the database file from scratch: it reclaims unused space, defragments, and can shrink the file.

```sql
VACUUM;                          -- rebuild & compact the whole database (needs free disk space ~= DB size)
PRAGMA auto_vacuum = INCREMENTAL; -- set at creation; lets you reclaim incrementally without full VACUUM
PRAGMA incremental_vacuum;       -- reclaim some free pages (if auto_vacuum = INCREMENTAL)
```

`VACUUM` takes an exclusive lock and rewrites the file, so run it during a maintenance window, not on a hot path. It also resets the rowid layout (harmless for `INTEGER PRIMARY KEY`).

### 15.3 `ANALYZE` and `PRAGMA optimize`

As in §6.6, `ANALYZE` refreshes the planner's statistics. The modern convenience is **`PRAGMA optimize`**, which runs whatever maintenance SQLite currently recommends (typically a targeted `ANALYZE` on tables that changed a lot). Run it periodically — e.g. before closing a long-lived connection.

```sql
PRAGMA optimize;   -- recommended to run on connection close or on a schedule
```

### 15.4 `integrity_check` — detect corruption

To verify the file isn't corrupt (after a crash, a suspicious copy, or bad hardware):

```sql
PRAGMA integrity_check;       -- 'ok' if healthy, else a list of problems (thorough, slower)
PRAGMA quick_check;           -- faster, less exhaustive variant
PRAGMA foreign_key_check;     -- separately, find rows whose FK points nowhere (§7.4)
```

Run `integrity_check` before taking a backup you intend to rely on, or as a periodic health check.

### 15.5 File portability

A SQLite file is **byte-for-byte portable across operating systems and CPU architectures** (it has a defined, endian-independent on-disk format). You can create a `.db` on Windows x64 and open it on a Linux ARM device unchanged. Keep these caveats in mind:
- Move the `-wal` along with the main file if the DB hasn't been checkpointed, or checkpoint first (`PRAGMA wal_checkpoint(TRUNCATE)`) so all data is in the main file before copying.
- The file format is stable and backward-compatible: newer SQLite reads older files; older SQLite reads newer files unless they use newer features (e.g. you can't read a STRICT-table DB on a pre-3.37 engine).

### 15.6 Concurrent access caveats (operational)

- **Local disk only for WAL.** Do not put a WAL-mode SQLite file on NFS/SMB or a Dropbox/OneDrive-synced folder; the locking/shared-memory assumptions break and corruption can result (§8.4, §17).
- **One process is happiest.** Multiple *processes* on the same machine can share a SQLite file (with WAL + busy_timeout), but it's smoother to funnel writes through a single process/service.
- **Don't hold transactions open.** A long-running write transaction blocks other writers for its whole duration and prevents WAL checkpointing — keep transactions short.

### 15.7 When to migrate to PostgreSQL

SQLite is the right answer far more often than people assume, but graduate to Postgres (see **`POSTGRESQL_GUIDE.md`**) when you hit these signals:

- **Sustained high write concurrency from many clients** — the single-writer model becomes the bottleneck (queue depth grows, `SQLITE_BUSY`/timeouts appear despite WAL).
- **Multiple application servers must share live data** over the network — SQLite has no network layer; you'd be reinventing a server.
- **You need features SQLite lacks:** real `NUMERIC`/`DECIMAL` types, rich native types (arrays, ranges, `jsonb` indexing depth, PostGIS), materialized views, partitioning, row-level security, server-side roles/`GRANT`, or `pgvector` for large-scale vector search.
- **Horizontal scaling / replication topologies** beyond what Litestream/LiteFS offer.

Migration path: a `.dump` gets you the schema + data as SQL, but you'll adapt types (`INTEGER PRIMARY KEY` → `BIGSERIAL`/`GENERATED`, affinity columns → strict Postgres types, `datetime('now')` → `now()`), turn implicit booleans (0/1) into real `boolean`, and replace SQLite-specific functions. Tools like `pgloader` automate much of it.

---

## 16. Performance Tuning

**[A]**

SQLite is *fast* by default, but a few techniques make order-of-magnitude differences. In rough priority order:

### 16.1 Wrap bulk writes in a transaction (the biggest win)

Every standalone `INSERT` runs in its own implicit transaction, which means a **disk sync (fsync) per statement**. Inserting 10,000 rows one-by-one can mean 10,000 syncs — agonizingly slow. Wrap them in **one** explicit transaction and you get **one** sync at commit: commonly a **50–100x speedup**. This is the single most important SQLite performance lesson.

```sql
BEGIN;                                  -- one transaction...
INSERT INTO t (x) VALUES (1);
INSERT INTO t (x) VALUES (2);
-- ... 10,000 of these ...
COMMIT;                                 -- ...one fsync. Massive speedup vs 10,000 auto-commits.
```

In code, use the transaction wrappers from §13.5 (better-sqlite3 `db.transaction`) and §14.4 (Go `*sql.Tx`). Also batch with **multi-row `INSERT`** (`VALUES (...),(...),(...)`) to cut statement overhead further.

### 16.2 Enable WAL + tune synchronous

`PRAGMA journal_mode = WAL` (§8.4) plus `PRAGMA synchronous = NORMAL` (§8.6) speeds up writes and removes reader/writer blocking. This is part of the standard preamble for a reason.

### 16.3 Reuse prepared statements

Compiling SQL has a cost. **Prepare once, execute many** (§13.4, §14.3). In a request handler, prepare statements at startup, not per request. SQLite caches compiled statements internally too, but explicit reuse avoids re-parsing entirely.

### 16.4 Index what you query

The right index turns an O(n) scan into an O(log n) seek (§6). Use `EXPLAIN QUERY PLAN` to confirm. Add **covering indexes** for hot read paths. Run `ANALYZE`/`PRAGMA optimize` so the planner chooses well.

### 16.5 Size the cache and consider mmap

```sql
PRAGMA cache_size = -64000;     -- ~64 MB of page cache (negative = KiB)
PRAGMA mmap_size = 268435456;   -- memory-map up to 256 MB for faster reads (read-heavy DBs)
PRAGMA temp_store = MEMORY;     -- temp B-trees in RAM
```

A larger page cache keeps hot pages in memory; `mmap` lets reads avoid copying through the OS read buffer. Size to your working set and available RAM.

### 16.6 Schema choices that help

- Use **`INTEGER PRIMARY KEY`** so the primary key is the efficient rowid (§3.6).
- Use **`STRICT`** tables — predictable types avoid surprise conversions and make indexes effective (§3.5).
- Prefer **keyset pagination** over large `OFFSET` (§5.1).
- Avoid storing **huge BLOBs** inline if you frequently read rows without the blob; keep big blobs in a side table or on the filesystem with a path reference (§17).
- For batch loads, consider `PRAGMA journal_mode = OFF` / `synchronous = OFF` *only* on a throwaway/rebuildable import DB, then switch back.

### 16.7 Measure

Use `.timer on` and `EXPLAIN QUERY PLAN` in the CLI. In `better-sqlite3`, the `verbose` option logs every statement. Profile real queries against realistic data volumes — SQLite's behaviour on 100 rows tells you little about 10 million.

---

## 17. Anti-Patterns & Gotchas

**[A]**

A consolidated checklist of the mistakes that bite people. Most are "you didn't know SQLite's defaults."

1. **Foreign keys silently not enforced.** FK enforcement is **off by default** and is per-connection. If you forget `PRAGMA foreign_keys = ON`, cascades do nothing and orphans accumulate. Set it on every connection (§7.2). Tests often forget it too, hiding the bug.
2. **Type-affinity surprises.** A non-STRICT `INTEGER` column will happily store the text `'banana'`; `'42'` and `42` don't compare/sort the same. Use **STRICT tables** (§3.5) for application schemas.
3. **Treating it as a multi-writer server.** Only one writer at a time, database-wide. High concurrent-write workloads will serialize and throw `SQLITE_BUSY`. Mitigate with WAL + `busy_timeout`, but if writes are genuinely heavy and concurrent, you've outgrown SQLite (§1.5, §15.7).
4. **Not using transactions for bulk writes.** N individual inserts = N fsyncs = ~100x slower than one transaction. Always batch (§16.1).
5. **One connection per query (or a big pool of writers).** Opening/closing a connection per query wastes work and, in Go, a large `MaxOpenConns` of writers causes lock contention. Reuse a long-lived connection; in Go cap writers (often `MaxOpenConns(1)`) (§14.2).
6. **`cp` of a live database.** Copying the file while it's being written (especially with `-wal` not checkpointed) yields a corrupt backup. Use `.backup`, `VACUUM INTO`, or checkpoint first (§15.1).
7. **SQLite on a network filesystem.** NFS/SMB/cloud-sync folders break SQLite's file locking; WAL needs local shared memory. Keep the DB on **local disk** (§8.4, §15.6).
8. **Storing huge BLOBs inline.** Multi-MB blobs in a row you read often bloat the page cache and slow scans. Store large binaries on the filesystem (or object storage) and keep only a path/reference in SQLite — or isolate them in a dedicated table.
9. **Floating-point money.** `REAL` can't represent `0.10` exactly. Store integer cents (§3.7).
10. **Local time in storage.** Store UTC ISO-8601; convert at display time (§12.4).
11. **`SELECT *` in application code.** Fragile to schema changes and defeats covering indexes. Name your columns.
12. **`AUTOINCREMENT` by reflex.** It adds overhead and is rarely needed; plain `INTEGER PRIMARY KEY` is enough (§4.7).
13. **Long-open transactions.** They block other writers and stall WAL checkpointing. Keep write transactions short.
14. **Forgetting `rows.Close()` (Go) / leaking statements.** In `database/sql`, an unclosed `*sql.Rows` holds a pooled connection hostage. Always `defer rows.Close()` (§14.3).
15. **Relying on lenient `GROUP BY`.** SQLite lets you select non-grouped columns and picks an arbitrary value; this hides bugs. Group/aggregate every selected column deliberately (§5.3).
16. **Assuming `LIKE '%x%'` is fast.** Leading-wildcard searches can't use a normal index. Use FTS5 for text search (§11), or a trigram index.

---

## 18. Study Path & Build-to-Learn Projects

**[All levels]**

### Suggested learning order

1. **Day 1 — Foundations (Beginner):** Install the `sqlite3` CLI. Create a `.db`, live in the shell, master the dot-commands (`.tables`, `.schema`, `.mode box`, `.import`, `.dump`) (§2). Do CRUD by hand: `CREATE TABLE`, `INSERT`, `SELECT`, `UPDATE`, `DELETE` (§4).
2. **Day 2 — The big quirk (Beginner→Intermediate):** Internalize **type affinity** and the five storage classes; prove to yourself that an `INTEGER` column can hold text, then adopt **STRICT tables** so it can't (§3). Understand `INTEGER PRIMARY KEY` / rowid.
3. **Day 3 — Querying (Intermediate):** Joins (all kinds), `GROUP BY`/aggregates, subqueries, CTEs and recursive CTEs, window functions, `CASE` (§5). Build queries against a dataset you care about.
4. **Day 4 — Performance fundamentals (Intermediate):** Create indexes; read `EXPLAIN QUERY PLAN` until `SCAN` vs `SEARCH` is instinctive; build a covering index; run `ANALYZE` (§6). Measure a slow query before and after.
5. **Day 5 — Integrity & transactions (Intermediate→Advanced):** Foreign keys (and remember to *enable* them!), referential actions (§7). Transactions, the single-writer model, and especially **WAL mode** + `busy_timeout` + `synchronous` (§8, §9).
6. **Day 6 — Modern features (Intermediate):** JSON (`->>`, `json_each`, indexing JSON) (§10); FTS5 full-text search with ranking (§11); generated columns, views, triggers, dates (§12).
7. **Day 7 — Embed it (Advanced):** Wire SQLite into a real service — Node with **better-sqlite3** (and try **node:sqlite**) behind Express/NestJS (§13), or Go with **modernc.org/sqlite** behind Gin (§14). Apply the PRAGMA preamble; use prepared statements and a transaction for a bulk endpoint.
8. **Ongoing — Operations & tuning (Advanced):** Backups (`.backup`, `VACUUM INTO`), `VACUUM`, `integrity_check`, portability (§15); the performance playbook (§16); and the anti-patterns checklist (§17). Know the signals for migrating to Postgres (§15.7).

### Build-to-learn projects

1. **CLI notes app (Beginner):** A command-line tool that stores notes in a single `.db`. Tables for `note` and `tag` (many-to-many via a join table — see **`RELATIONAL_DB_DESIGN_GUIDE.md`**). Add full-text search over note bodies with **FTS5** (§11) and tag filtering. Practice prepared statements and the standard PRAGMA preamble. Ship it as a single static Go binary using `modernc.org/sqlite` (cgo-free) so it runs anywhere.
2. **Local-first task tracker (Intermediate):** The Tasks API from §13/§14, expanded: projects → tasks (FK with `ON DELETE CASCADE`), `done` boolean, due dates stored as UTC ISO-8601, an `updated_at` trigger, a JSON `metadata` column, and a bulk-import endpoint wrapped in a transaction. Enable **WAL** and load-test concurrent reads + a single writer; watch `busy_timeout` absorb contention. Add a `.backup`-based nightly snapshot.
3. **Embedded cache / metadata store for a web service (Advanced):** Use SQLite as the local store *inside* a larger app — e.g. an HTTP cache or rate-limiter state, or a config/feature-flag store. Compare its role with Redis (see **`REDIS_GUIDE.md`**): SQLite gives you durable, queryable, transactional local state with zero extra infrastructure. Tune `cache_size`/`mmap_size`, index hot lookups, and run `PRAGMA optimize` on a schedule.
4. **"Ship a dataset" project (Intermediate):** Take an open dataset (CSV), `.import` it, design proper STRICT tables and indexes, add views and a couple of analytical window-function queries, and distribute the resulting `.db` as a self-contained, queryable artifact. Verify it opens unchanged on a different OS/arch (file portability, §15.5).
5. **Production-hardening drill (Advanced):** Take project #2 and add continuous backup with **Litestream** to object storage; simulate a crash and restore to a point in time. Then deliberately push it past its limits (hundreds of concurrent writers) to *feel* the single-writer ceiling — and write down the exact signals that would make you migrate to **PostgreSQL** (§15.7).

### Cheat-sheet of "look it up later" commands

```text
sqlite3 app.db                      -- open/create a database
.tables / .schema / .mode box       -- inspect & format (CLI)
.backup main b.db / VACUUM INTO 'b.db'  -- safe online backup
.dump > dump.sql                    -- logical (SQL) backup
PRAGMA journal_mode = WAL;          -- concurrency: readers don't block the writer
PRAGMA busy_timeout = 5000;         -- wait for locks instead of erroring
PRAGMA foreign_keys = ON;           -- ENFORCE FKs (off by default!)
PRAGMA synchronous = NORMAL;        -- safe + fast under WAL
EXPLAIN QUERY PLAN <query>;          -- see SCAN vs SEARCH (index usage)
ANALYZE; PRAGMA optimize;           -- refresh planner stats
PRAGMA integrity_check;             -- detect corruption
INSERT ... ON CONFLICT(col) DO UPDATE SET ...  -- upsert
INSERT ... RETURNING id;            -- get generated keys in one round trip
BEGIN; ...many writes...; COMMIT;   -- batch writes = huge speedup
CREATE VIRTUAL TABLE x USING fts5(...);  -- full-text search
SELECT json_extract(data,'$.k'); data ->> '$.k';  -- JSON
```

> **Final advice:** SQLite rewards understanding its *defaults*. The three settings that fix 90% of "SQLite is weird/slow/broken" complaints are: **turn foreign keys ON**, **switch to WAL mode**, and **wrap bulk writes in a transaction**. Use **STRICT tables** so types behave, **prepared statements** everywhere, and `EXPLAIN QUERY PLAN` whenever something is slow. Reach for SQLite first for anything that lives on one machine — it is astonishingly capable, requires zero operations, and is the most thoroughly tested software most of us will ever use. When you genuinely need many machines to share live data or sustained high write concurrency, that's your cue to move to **PostgreSQL** — and you'll carry every SQL skill from this guide with you.

### 14.6 Go concurrency reminder

Even though Go makes it trivial to launch hundreds of goroutines, **SQLite still has one writer.** With WAL + `busy_timeout` set in the DSN, concurrent *reads* scale fine, and brief write contention is absorbed by the busy timeout. The `SetMaxOpenConns(1)` approach makes writes strictly serial and `SQLITE_BUSY`-free at the cost of read parallelism on that pool; a common refinement is **two pools** — a read-only pool (`mode=ro`, several connections) and a single-connection write pool — but only reach for that once you've measured a need.
