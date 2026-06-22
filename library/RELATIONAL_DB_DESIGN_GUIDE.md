# Relational Database Design — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Developers, backend engineers, and aspiring data architects who want to *design* relational databases well — not just write SQL, but decide what tables should exist, how they relate, which keys to use, how far to normalize, and which schema patterns fit the problem. This is a **design (data modeling)** guide: it assumes you can already write basic `CREATE TABLE` / `SELECT` (if not, read the SQL chapters of `POSTGRESQL_GUIDE.md` and `SQLITE3_GUIDE.md` first). Every concept is taught in three parts — **what it is and why**, **what it's for / when**, and **how to use it** — followed by heavily commented examples. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note (2026):** SQL examples use **PostgreSQL** dialect (current stable is **PostgreSQL 17**, with PG 18 features like `uuidv7()` called out), because it is the most standards-faithful open-source RDBMS and the one most worth learning design on. **The *design principles* here are RDBMS-agnostic** — they apply equally to MySQL 8.4+, SQL Server 2022, Oracle 23ai, MariaDB, and (with caveats) SQLite. Where a feature is Postgres-specific, it is flagged, and a portable alternative is given. Application-layer examples show **both** a Node.js backend (NestJS/Express with **Prisma** and the raw **`pg`** driver) **and** a Go backend (**Gin** with **pgx** / **sqlc** and **GORM**) so you can see how a well-designed schema is consumed from real production stacks.
>
> **Cross-references:** For SQL syntax depth, indexing internals, `EXPLAIN`, MVCC, partitioning mechanics, and replication ops, see **`POSTGRESQL_GUIDE.md`**. For the embedded/single-file case and its design constraints, see **`SQLITE3_GUIDE.md`**. For ORM specifics see **`PRISMA_ORM_GUIDE.md`** and **`GO_ENT_ORM_GUIDE.md`**. For the Go web layer see **`GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`**; for the Node web layer see **`NESTJS_GUIDE.md`** / **`NODEJS_GUIDE.md`**.

---

## Table of Contents

1. [The Relational Model](#1-the-relational-model) **[B]**
2. [Keys — Primary, Candidate, Composite, Foreign, Unique](#2-keys--primary-candidate-composite-foreign-unique) **[B/I]**
3. [Data Types, Domains, NULLs & Constraints](#3-data-types-domains-nulls--constraints) **[B/I]**
4. [Entity-Relationship (ER) Modeling](#4-entity-relationship-er-modeling) **[B/I]**
5. [Implementing Relationships in Tables](#5-implementing-relationships-in-tables) **[B/I]**
6. [Normalization (and Denormalization)](#6-normalization-and-denormalization) **[I]**
7. [Indexing for Design](#7-indexing-for-design) **[I]**
8. [Schema Design Patterns](#8-schema-design-patterns) **[I/A]**
9. [Referential Actions & Integrity](#9-referential-actions--integrity) **[I]**
10. [Transactions & Concurrency in Design](#10-transactions--concurrency-in-design) **[I/A]**
11. [Complete Worked Example — E-commerce](#11-complete-worked-example--e-commerce) **[I/A]**
12. [Migrations & Schema Evolution](#12-migrations--schema-evolution) **[I/A]**
13. [Performance & Scaling Design](#13-performance--scaling-design) **[A]**
14. [Anti-Patterns & Gotchas](#14-anti-patterns--gotchas) **[All]**
15. [Study Path & Build-to-Learn Projects](#15-study-path--build-to-learn-projects)

---

## 1. The Relational Model

**[B]**

### 1.1 What the relational model is — and the logic behind it

A **relational database** organizes data into **relations** — what almost everyone calls **tables**. This is not a casual naming choice; it descends from a precise piece of mathematics. In 1970, Edgar F. Codd (an IBM researcher) published *"A Relational Model of Data for Large Shared Data Banks."* His insight was to model data using **set theory and first-order predicate logic** instead of the tangled pointer-chasing structures (hierarchical and network databases) that dominated at the time. That mathematical grounding is *why* relational databases are so durable: queries have well-defined meaning, the system can reorder and optimize operations freely, and correctness is provable rather than hopeful.

Here is the vocabulary, with the formal term, the everyday term, and what it actually *is*:

| Formal (Codd) | Everyday term | What it is |
|---|---|---|
| **Relation** | Table | A set of rows over a fixed set of columns |
| **Tuple** | Row / record | One member of the set — one fact about one thing |
| **Attribute** | Column / field | A named, typed slot every tuple has |
| **Domain** | Data type (+ constraints) | The set of legal values an attribute may take |
| **Relation schema** | Table definition | The table name + its attributes and their domains |
| **Degree** | (column count) | How many attributes the relation has |
| **Cardinality** | (row count) | How many tuples the relation currently holds |

The single most important consequence of "a relation is a **set**" is this: **a true relation has no duplicate rows and no inherent row order.** Sets are unordered and contain no duplicates. Real SQL tables relax this — SQL is technically based on *multisets* (bags), so a table *can* contain duplicate rows if you let it — but **good design treats every table as a set** by giving it a key (Section 2) so duplicates are impossible. If you ever find yourself relying on "the order rows happen to come back in," you have stepped outside the model and your code is fragile; use `ORDER BY`.

### 1.2 Why the relational model won

Three competing approaches existed in the 1970s–80s, and relational beat them all for general-purpose data:

- **Hierarchical model (e.g. IMS):** data is a tree. Fast for tree-shaped access, but a record can have only one parent — modeling a many-to-many relationship (a student in many courses, a course with many students) is painful, and querying "sideways" against the tree is hard.
- **Network model (CODASYL):** data is a graph of records linked by explicit pointers. More flexible than hierarchical, but the *application* had to navigate the pointers manually. Change the access pattern and you rewrite navigation code.
- **Relational model:** data is sets of tuples linked by **values** (a foreign key holds the *value* of another row's key), not pointers. You **declare what you want** with SQL and the optimizer figures out *how* to get it. This **physical/logical independence** is the killer feature: you can add an index, repartition, or change storage and *not touch a single query*.

The relational model's pairing with **SQL** (a declarative query language) and **ACID transactions** (Section 10) made it the default for systems that must not lose or corrupt data — finance, commerce, inventory, identity. NoSQL systems trade some of these guarantees for scale or schema flexibility; they are tools for specific jobs, not replacements. In 2026, the relational database remains the correct default for the *system of record* in almost every application.

### 1.3 What an RDBMS gives you

An **RDBMS** (Relational Database Management System) is the software that stores relations and enforces the model. Beyond storage, it provides:

- **A declarative query engine + optimizer** — you say *what*, it decides *how*.
- **Constraints** — the database itself refuses to store data that violates your rules (Section 3, 9). This is the difference between "we hope the app validates it" and "it is *impossible* to store a bad row."
- **Transactions (ACID)** — groups of changes that all succeed or all fail, with isolation between concurrent users (Section 10).
- **Concurrency control** — many users reading/writing at once without corrupting each other's view (locking, or MVCC in Postgres).
- **Durability & recovery** — committed data survives crashes (write-ahead logging).

Popular RDBMSs in 2026 and where they shine:

| RDBMS | Sweet spot | Notes for designers |
|---|---|---|
| **PostgreSQL 17/18** | General default; rich types, extensible, strict | Best standards compliance; JSONB, arrays, partial/expression indexes, `CHECK`, exclusion constraints |
| **MySQL 8.4 / MariaDB** | Web apps, read-heavy, huge ecosystem | InnoDB engine; weaker `CHECK` history (enforced since 8.0.16); clustered PK matters a lot for design |
| **SQL Server 2022** | Enterprise / .NET shops | Clustered indexes, strong tooling, `MERGE`, temporal tables built in |
| **Oracle 23ai** | Large enterprise | Mature, expensive; many proprietary features |
| **SQLite** | Embedded, mobile, single-file, tests | One writer at a time; flexible typing; great for local-first (see `SQLITE3_GUIDE.md`) |

> **Design takeaway:** The model is about **structuring facts as related sets and letting the database enforce truth.** Everything in this guide is, ultimately, about *putting each fact in exactly one obvious place* and *making illegal states impossible to store*.

### 1.4 Relational algebra — the operations your SQL compiles to

You don't write relational algebra, but knowing it exists clarifies *why* SQL works and why the optimizer is free to rewrite your queries. **Relational algebra** is the set of operations that take relations as input and produce relations as output (so they compose). The core operators:

- **Selection (σ)** — keep rows matching a predicate. This is SQL `WHERE`.
- **Projection (π)** — keep only certain columns (and drop duplicates, since the result is a set). This is SQL `SELECT col1, col2`.
- **Union / Intersection / Difference** — set operations on two union-compatible relations. SQL `UNION` / `INTERSECT` / `EXCEPT`.
- **Cartesian product (×)** — every row of A paired with every row of B. SQL `CROSS JOIN`.
- **Join (⋈)** — a product filtered to matching rows; the workhorse for combining related tables via key values. SQL `JOIN ... ON`.
- **Rename (ρ)** — alias a relation/attribute. SQL `AS`.

Because every operation's input and output is a relation, operations **nest and reorder freely** — and that is precisely the mathematical license the query optimizer uses to transform `SELECT ... WHERE ... JOIN ...` into the cheapest equivalent plan (push a filter below a join, swap join order, pick an index). Your declarative query has a single *meaning* but many possible *executions*; algebra is why they're provably equivalent.

```sql
-- This SQL...
SELECT u.email, o.total_cents
FROM users u JOIN "order" o ON o.user_id = u.id
WHERE o.total_cents > 10000;

-- ...corresponds to:  π_{email,total_cents} ( σ_{total_cents>10000} ( users ⋈ order ) )
-- The optimizer may instead apply σ BEFORE the join (scan only big orders first) — same result, faster.
```

---

## 2. Keys — Primary, Candidate, Composite, Foreign, Unique

**[B/I]**

Keys are the backbone of relational design. They are how you guarantee uniqueness (no accidental duplicates) and how you link tables together (relationships). Get keys right and the rest of the design tends to fall into place; get them wrong and you fight the database forever.

### 2.1 Candidate keys and the primary key — what & why

A **candidate key** is *any* minimal set of columns whose values uniquely identify a row. "Minimal" matters: if `{email}` alone is unique, then `{email, name}` is *not* a candidate key (it's not minimal — `name` is dead weight). A table can have several candidate keys; for a `users` table, both `{id}` and `{email}` might each uniquely identify a row, so both are candidate keys.

The **primary key (PK)** is the one candidate key you *elect* to be the row's official identity. The database enforces that it is **unique** and **NOT NULL** (a primary key can never be unknown — identity must always exist). You then use the PK as the target that foreign keys point at.

**Why you need one on every table:** Without a key, you cannot reliably address a single row to update or delete it, you can accumulate duplicate rows, replication tools and ORMs misbehave, and (in clustered-index databases like MySQL/SQL Server) the engine invents a hidden one anyway. **Rule: every table gets a primary key.**

```sql
-- A candidate key that is also the natural identity:
CREATE TABLE country (
    iso_code  CHAR(2) PRIMARY KEY,   -- 'US', 'NG', 'JP' — naturally unique & stable
    name      TEXT NOT NULL
);

-- Multiple candidate keys: id (surrogate) is PK, email is an alternate (UNIQUE) candidate key.
CREATE TABLE users (
    id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY, -- elected primary key
    email  TEXT NOT NULL UNIQUE,                             -- alternate candidate key
    name   TEXT NOT NULL
);
```

### 2.2 Natural vs surrogate keys — the central debate

This is one of the most argued-about decisions in data modeling, so understand both sides rather than parroting a rule.

A **natural key** is a key made of real-world data that already identifies the entity: a country's ISO code, a book's ISBN, a user's email, a US Social Security Number. A **surrogate key** is a synthetic, meaningless identifier the database generates purely to be the key: an auto-incrementing integer, a UUID, a ULID. It has no business meaning — its only job is to be unique and stable.

**The case for natural keys:** No extra column; the key is meaningful so joins and foreign keys read naturally; if two systems independently record the same real entity they'll agree on the key.

**The case against natural keys (and why surrogates usually win):**
- **Natural identifiers change.** Emails change, people change names, "permanent" product codes get reissued, and "this will never change" famously always changes. If your PK changes, every foreign key referencing it must cascade-update — expensive and risky.
- **Natural keys leak and are sensitive.** Using an SSN or email as a PK spreads PII across every child table and into URLs (`/orders/by-user/jane@x.com`).
- **Natural keys are often not actually unique or not always known.** "Surely full name + birthdate is unique" — until it isn't.
- **Composite natural keys are wide and awkward** to propagate as foreign keys into many tables.

**Pragmatic consensus (2026):** Use a **surrogate primary key** on almost every table for *identity and joins*, **and** add a `UNIQUE` constraint on the **natural key** to enforce the real-world uniqueness rule. You get the stability of a surrogate *and* the data-integrity guarantee of the natural key. The natural key remains a candidate key — just not the PK.

```sql
CREATE TABLE product (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY, -- surrogate: stable identity for FKs & URLs
    sku         TEXT NOT NULL UNIQUE,   -- natural business key: still enforced unique
    name        TEXT NOT NULL
);
-- Best of both: orders reference product.id (never changes), but the business
-- rule "SKU is unique" is still guaranteed by the database.
```

> **Exception:** Pure **junction tables** (Section 5.3) and small fixed **lookup tables** (Section 8.5) are good places for natural/composite keys — a `tag_id, post_id` composite PK on a join table is idiomatic and correct.

### 2.3 Composite keys — what & when

A **composite key** is a key made of **more than one column**. Its uniqueness is on the *combination*, not the individual columns. The classic, correct use is a **junction table** for a many-to-many relationship, where the pair of foreign keys *is* the identity of the relationship row.

```sql
-- Many-to-many: a student enrolls in many courses; a course has many students.
CREATE TABLE enrollment (
    student_id  BIGINT NOT NULL REFERENCES student(id),
    course_id   BIGINT NOT NULL REFERENCES course(id),
    enrolled_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- The COMPOSITE primary key: a given student can enroll in a given course only once.
    -- Column ORDER matters for the underlying index (Section 7): put the column you
    -- filter by most first. Here we expect "courses for a student" queries, so student_id leads.
    PRIMARY KEY (student_id, course_id)
);
```

A composite key's **column order** defines the order of the backing index, which affects which queries it can accelerate (Section 7.4). Choose the order to match your most common access pattern.

### 2.4 Foreign keys & referential integrity — the heart of "relational"

A **foreign key (FK)** is a column (or columns) in one table whose values must match an existing primary/unique key value in another table (or be `NULL`). This is how relationships are *physically represented*: the child row stores the *value* of the parent's key.

The constraint the FK creates is **referential integrity**: the database **guarantees there are no orphans** — you cannot insert an `order` for a `user_id` that doesn't exist, and (depending on the referential action, Section 9) you cannot delete a `user` who still has orders. This is enormous: it means a whole class of bugs ("dangling reference," "ghost record") is *impossible by construction*, not merely "unlikely if the app is careful."

```sql
CREATE TABLE "order" (
    id       BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id  BIGINT NOT NULL                       -- the FK column
             REFERENCES users(id)                   -- must match an existing users.id ...
             ON DELETE RESTRICT,                    -- ... and you can't delete a user with orders (Section 9)
    total_cents BIGINT NOT NULL CHECK (total_cents >= 0)
);
```

> **Critical, easily-missed point:** A foreign key does **NOT** create an index on the child column. The *parent* side is indexed (it's a PK/unique), but the *child* `user_id` is not. You almost always want to **index every FK column yourself** — otherwise "find all orders for this user" does a full table scan, and (in most engines) deleting/updating a parent row scans every child table to check the constraint. See Section 7.2.

### 2.5 Unique keys (alternate keys)

A **unique constraint** enforces that a column or combination is unique, *without* making it the primary key. Use it for every additional candidate key (emails, usernames, SKUs, slugs). Unlike a PK, a unique column **may allow NULLs** — and here lurks a famous gotcha: by the SQL standard, `NULL` is "unknown," so multiple `NULL`s are *not* considered equal and are all allowed in a `UNIQUE` column. PostgreSQL follows the standard (PG 15+ lets you opt into `UNIQUE NULLS NOT DISTINCT` to forbid more than one NULL). Different engines differ — know your engine.

```sql
CREATE TABLE account (
    id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    username  TEXT NOT NULL UNIQUE,                  -- alternate candidate key, no NULLs (NOT NULL)
    phone     TEXT UNIQUE,                           -- nullable: many rows MAY have NULL phone (standard SQL)
    -- Force at-most-one NULL phone instead (PG15+): UNIQUE NULLS NOT DISTINCT (phone)
    UNIQUE (username)                                -- (redundant here; shown for the multi-column form)
);
```

### 2.6 Choosing a surrogate key type — auto-increment vs UUID vs ULID

When you go surrogate (almost always), you still must pick *what kind* of value. This choice has real performance and architecture consequences.

| Type | What it is | Pros | Cons |
|---|---|---|---|
| **Auto-increment / IDENTITY** (`BIGINT`) | Sequential integers from a sequence | Tiny (8 bytes), fast, naturally ordered → great index locality; human-readable | Reveals row counts & order (`/orders/1002` ⇒ guess `/orders/1003`); requires a central allocator (hard to generate offline / pre-merge); leaks business volume |
| **UUIDv4** (`UUID`, 16 bytes) | Random 128-bit value | Globally unique, generate anywhere (client, offline, before insert), no central allocator, no enumeration leak | **Random ⇒ terrible B-tree insert locality** (writes scatter across the index → page splits, bloat, cache misses); 16 bytes vs 8; ugly in URLs; not time-sortable |
| **UUIDv7** (`UUID`, 16 bytes) | Time-ordered UUID (timestamp prefix + randomness) | Globally unique like v4 **but monotonic** → restores index locality; sortable by creation time; standard (RFC 9562, 2024) | Slightly leaks creation time; still 16 bytes; needs PG 18 `uuidv7()` or a library on older versions |
| **ULID** | 128-bit, time-sorted, Crockford-base32 (26 chars) | Time-sortable, URL-friendly text, generate anywhere | Usually stored as text/UUID; library-dependent; same "leaks time" caveat |

**How to choose:**
- **Single database, internal IDs, you control inserts → `BIGINT IDENTITY`.** Smallest, fastest, simplest. Default choice.
- **Distributed inserts, client-generated IDs, microservices that must mint IDs before the row hits the DB, or you want to merge data from multiple sources → time-ordered UUID (`UUIDv7`) or ULID.** You get global uniqueness *and* decent index performance.
- **Avoid random `UUIDv4` as a clustered/primary key on write-heavy tables** unless you genuinely need the unguessability and have measured the write amplification. If you must, consider keeping a `BIGINT` PK for internal joins and a separate `UUID` "public id" column for external exposure.

```sql
-- Default: sequential surrogate. GENERATED ALWAYS prevents apps from injecting ids by accident.
CREATE TABLE invoice (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);

-- Distributed / client-mintable, but keep good index locality with v7 (PG 18+):
CREATE TABLE event (
    id UUID PRIMARY KEY DEFAULT uuidv7()   -- PG18 builtin; pre-18 use pgcrypto's gen_random_uuid() (v4) or a lib
);

-- Common hybrid: internal BIGINT for joins, public UUID for the outside world.
CREATE TABLE customer (
    id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- fast internal joins/FKs
    public_id UUID NOT NULL UNIQUE DEFAULT gen_random_uuid()    -- safe to put in URLs/APIs
);
```

> **Gotcha:** Sequences (and `IDENTITY`) **leave gaps** — a rolled-back transaction still "uses up" the number. Never assume IDs are contiguous; never use "max id" as a row count.

---

## 3. Data Types, Domains, NULLs & Constraints

**[B/I]**

Choosing the right *type* for each column is design, not trivia. The type is the column's **domain** — the set of legal values — and it is your first and cheapest line of integrity defense. A correct type makes whole categories of bad data impossible, saves storage, enables the right indexes, and communicates intent.

### 3.1 Choosing data types — principles

1. **Store data as the thing it is, not as text.** A date is a `DATE`, not `'2026-06-21'` in a `TEXT` column. Storing typed data lets the database validate it, sort/compare it correctly, do arithmetic, and index it efficiently. Text-as-everything ("stringly typed") defeats all of that.
2. **Use the narrowest type that comfortably fits the *lifetime* range.** But don't be penny-wise: use `BIGINT` for surrogate keys even if `INT` "would fit for now" — running out of IDs in production is a brutal migration.
3. **Prefer exact types for money and identifiers.** Never use `FLOAT`/`DOUBLE` for currency (Section 14) — binary floating point can't represent `0.10` exactly. Use `NUMERIC(precision, scale)` or, very commonly, an **integer count of the smallest unit (cents)**.
4. **Use the database's rich types instead of reinventing them.** Postgres gives you `BOOLEAN`, `TIMESTAMPTZ`, `INET`, `UUID`, `JSONB`, arrays, ranges, `ENUM` — using them is clearer and safer than encoding everything in strings/ints.

| Need | Good Postgres type | Why |
|---|---|---|
| Surrogate id | `BIGINT` / `UUID` | Won't overflow / globally unique |
| Money | `NUMERIC(19,4)` or `BIGINT` cents | Exact decimal arithmetic |
| Whole numbers | `INTEGER`, `BIGINT`, `SMALLINT` | Exact, efficient |
| True/false | `BOOLEAN` | Self-documenting vs `0/1` ints |
| Short/long text | `TEXT` (Postgres) / `VARCHAR(n)` | In PG `TEXT` and `VARCHAR` are equally fast; use a length `CHECK` if you need a limit |
| Timestamp | `TIMESTAMPTZ` | Stores an absolute instant; **always prefer over `TIMESTAMP`** |
| Date only | `DATE` | Birthdays, due dates |
| Enumerated set | `ENUM` *or* a lookup table | See Section 8.5 trade-off |
| Semi-structured | `JSONB` | Indexable, validated JSON; *don't* use it to avoid modeling (Section 14) |
| Unique id from anywhere | `UUID` | 16-byte native type |

> **Timezone trap:** Use `TIMESTAMPTZ` (timestamp *with* time zone) for "a moment in time." Despite the name it does **not** store a zone — it stores a UTC instant and converts on input/output using the session zone. Plain `TIMESTAMP` (without zone) stores wall-clock with no instant, which causes ambiguous-time and DST bugs. Reach for `TIMESTAMP` only for true "floating" local times (e.g., a store's "opens at 09:00 local" rule).

### 3.2 NULL semantics and the three-valued-logic trap **[I, critical]**

`NULL` is not a value — it is a marker meaning **"unknown / not applicable / absent."** This single fact causes more subtle bugs than almost anything else in SQL, because it changes logic from two-valued (TRUE/FALSE) to **three-valued (TRUE / FALSE / UNKNOWN)**.

The rules that bite people:

- **Any comparison with NULL yields `UNKNOWN`, not TRUE/FALSE.** `NULL = NULL` is `UNKNOWN`. `NULL = 5` is `UNKNOWN`. `NULL <> NULL` is `UNKNOWN`. So `WHERE x = NULL` matches **nothing** — you must write `WHERE x IS NULL`.
- **A `WHERE` clause keeps a row only if the predicate is `TRUE`** (not `UNKNOWN`). So a row where the test is `UNKNOWN` is silently dropped.
- **This silently breaks "give me everything not equal to X."** `WHERE status <> 'shipped'` excludes rows where `status IS NULL`, because `NULL <> 'shipped'` is `UNKNOWN`. You usually meant to include them: `WHERE status <> 'shipped' OR status IS NULL`, or use `WHERE status IS DISTINCT FROM 'shipped'`.
- **Aggregates ignore NULLs** (except `COUNT(*)`). `AVG(col)`, `SUM(col)` skip NULL rows; `COUNT(col)` counts non-NULLs while `COUNT(*)` counts rows.
- **`UNIQUE` lets multiple NULLs coexist** (Section 2.5), because two unknowns aren't "equal."

```sql
SELECT NULL = NULL;                 -- NULL (i.e. UNKNOWN), NOT true!
SELECT NULL IS NULL;                -- true  (use IS NULL / IS NOT NULL)
SELECT 5 IS DISTINCT FROM NULL;     -- true  (NULL-safe "not equal")
SELECT NULL IS NOT DISTINCT FROM NULL; -- true (NULL-safe "equal")

-- The classic bug:
-- DELETE FROM task WHERE status <> 'done';   -- ⚠ leaves rows where status IS NULL!
DELETE FROM task WHERE status IS DISTINCT FROM 'done'; -- ✔ also removes NULL-status rows

-- COALESCE supplies a fallback for NULL:
SELECT COALESCE(nickname, full_name, 'Anonymous') FROM person;
```

**Design implication:** Decide *deliberately* whether each column may be NULL, and prefer `NOT NULL` unless absence is a genuine, meaningful state. A `NOT NULL` column with a sensible `DEFAULT` removes a whole class of three-valued-logic surprises. Reserve NULL for "we truly don't know yet" (e.g., `shipped_at` before shipping), not for "empty string" or "zero."

### 3.3 Defaults

A **default** supplies a value when an `INSERT` omits the column. Use defaults to encode invariants ("new accounts start `active`", "rows are created `now()`") so the app can't forget them.

```sql
CREATE TABLE account (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status      TEXT NOT NULL DEFAULT 'active',     -- new accounts are active unless stated
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(), -- server clock, not app clock
    balance_cents BIGINT NOT NULL DEFAULT 0
);
```

### 3.4 Constraints — making illegal states impossible **[I]**

Constraints are the database enforcing your business rules so that *no* code path — not the app, not a migration, not a manual `psql` session — can store invalid data. They are the most underused integrity tool. The five kinds:

- **`NOT NULL`** — the column must have a value.
- **`UNIQUE`** — no duplicate values/combinations (Section 2.5).
- **`PRIMARY KEY`** — `UNIQUE` + `NOT NULL`, the elected identity.
- **`FOREIGN KEY`** — references a valid parent row (Section 2.4, 9).
- **`CHECK`** — an arbitrary boolean expression that must hold for every row.

`CHECK` is the workhorse for domain rules. Note the three-valued logic interaction: a `CHECK` *passes* if the expression is TRUE **or** UNKNOWN — i.e., `CHECK (price > 0)` does **not** reject a `NULL` price (the NULL makes it UNKNOWN, which passes). Add `NOT NULL` if you mean it.

```sql
CREATE TABLE product (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name        TEXT   NOT NULL CHECK (length(trim(name)) > 0),   -- no blank names
    price_cents BIGINT NOT NULL CHECK (price_cents >= 0),         -- no negative prices
    -- A row-level CHECK can span columns:
    sale_cents  BIGINT CHECK (sale_cents IS NULL OR sale_cents <= price_cents),
    status      TEXT   NOT NULL DEFAULT 'draft'
                 CHECK (status IN ('draft','active','archived'))  -- a poor-man's enum (Section 8.5)
);

-- Named constraints give readable error messages and let you ALTER them later:
ALTER TABLE product
  ADD CONSTRAINT product_price_nonneg CHECK (price_cents >= 0);
```

### 3.5 Domains (reusable named types) **[I]**

A **`DOMAIN`** is a named, reusable type = a base type + constraints. Define a rule once (e.g., "an email is text matching this pattern and never NULL") and apply it across many tables. It centralizes the rule so it can't drift between tables.

```sql
-- Define once:
CREATE DOMAIN email AS TEXT
  CHECK (VALUE ~ '^[^@\s]+@[^@\s]+\.[^@\s]+$');   -- simplistic but illustrative

CREATE DOMAIN positive_cents AS BIGINT
  CHECK (VALUE >= 0);

-- Reuse everywhere — every email/amount column now shares the same guarantee:
CREATE TABLE contact (
    id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email email NOT NULL,
    quota positive_cents NOT NULL DEFAULT 0
);
```

### 3.6 Generated (computed) columns **[I]**

A **generated column** is computed from other columns by the database, so it can never disagree with its inputs. Postgres `STORED` generated columns are written to disk (can be indexed); PG 18 adds `VIRTUAL` (computed on read, no storage). Use them for derived values you query often (a full name, a total, a normalized search key) instead of trusting the app to keep a denormalized copy in sync.

```sql
CREATE TABLE order_line (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    qty         INT    NOT NULL CHECK (qty > 0),
    unit_cents  BIGINT NOT NULL CHECK (unit_cents >= 0),
    -- Always correct; cannot drift from its inputs; can be indexed (STORED):
    line_total_cents BIGINT GENERATED ALWAYS AS (qty * unit_cents) STORED
);

-- A normalized search column kept perfectly in sync with `name`:
CREATE TABLE city (
    name        TEXT NOT NULL,
    name_lower  TEXT GENERATED ALWAYS AS (lower(name)) STORED
);
-- CREATE INDEX ON city (name_lower);  -- now case-insensitive lookups are index-backed
```

---

## 4. Entity-Relationship (ER) Modeling

**[B/I]**

Before you write a single `CREATE TABLE`, you model the *concepts* and how they relate. **Entity-Relationship (ER) modeling** is the standard technique: it produces a diagram of the problem domain that translates almost mechanically into tables. Designing on paper first is dramatically cheaper than discovering structural mistakes after you have data and code depending on the schema.

### 4.1 The three building blocks

- **Entity** — a *thing* the business cares about and stores facts about: a `Customer`, a `Product`, an `Order`. Each entity becomes (roughly) a **table**, and each *instance* of the entity (one specific customer) becomes a **row**.
- **Attribute** — a *property* of an entity: a customer's `email`, an order's `placed_at`. Each becomes a **column**. Attributes have domains (Section 3). One (or a combination) is the **key** (Section 2).
- **Relationship** — an *association* between entities: a customer *places* orders; an order *contains* products. Relationships become **foreign keys** (1:1, 1:N) or **junction tables** (M:N) — Section 5.

Attributes come in flavors worth naming because each has a modeling implication:
- **Simple vs composite:** `birth_date` is simple; an `address` is composite (street, city, zip) — you usually break composites into separate columns, or a separate table if reused.
- **Single-valued vs multi-valued:** `email` (one) vs `phone numbers` (many). **A multi-valued attribute is a red flag that you actually have a separate entity / a one-to-many relationship.** You do *not* store many phones in one CSV column (Section 14); you make a `phone` table. This is the seed of 1NF (Section 6).
- **Stored vs derived:** `age` is derived from `birth_date` — don't store derivable data unless you have a measured reason (then use a generated column, Section 3.6).

### 4.2 Cardinality — the shape of a relationship

**Cardinality** answers "how many of B can relate to one A, and vice-versa?" There are three shapes, and recognizing which one you have *is* the design decision:

- **One-to-one (1:1):** one A relates to at most one B and vice versa. Rare. Example: a `user` and that user's `user_profile` (heavy/optional details split off). Often a sign the two should just be one table — justify the split.
- **One-to-many (1:N):** one A relates to many B, each B to one A. The most common shape. Example: one `customer` has many `orders`; each order belongs to one customer.
- **Many-to-many (M:N):** many A relate to many B. Example: a `student` takes many `courses`; a `course` has many `students`. **You cannot represent M:N directly with a foreign key** — it requires a third (junction) table (Section 5.3).

You also record **optionality / participation** (the "minimum" side): must every order have a customer (mandatory) or can an order exist without one (optional)? Mandatory means a `NOT NULL` foreign key; optional means a nullable FK.

### 4.3 Reading ER notations (Crow's Foot)

The most common diagram style is **Crow's Foot**, where the symbol at each *end* of a line encodes (minimum, maximum):

```text
Crow's-foot end symbols:
   --||      exactly one        (one and only one)
   --o|      zero or one        (optional, max one)
   --|<      one or many        (mandatory, max many)   "<" is the crow's foot
   --o<      zero or many       (optional, max many)

So a 1:N "a CUSTOMER places zero-or-many ORDERS, each ORDER belongs to exactly one CUSTOMER":

   CUSTOMER ||--------o< ORDER
            ^               ^
            exactly one     zero-or-many
   (read each end toward the entity it touches)
```

A small worked ER sketch for a blog, in ASCII:

```text
            places                      contains
  USER ||----------o< POST ||----------o< COMMENT
   |                  |                     ^
   | id (PK)          | id (PK)             | id (PK)
   | email (UQ)       | user_id (FK)        | post_id (FK)
   | name             | title               | user_id (FK)  (commenter)
                      | body                | body
                      | published_at        | created_at

                 M:N via junction
       POST >o----------< TAG
            \           /
             \         /
              POST_TAG          (post_id FK, tag_id FK, PK(post_id, tag_id))
```

This reads: a USER places zero-or-many POSTs; each POST belongs to exactly one USER. A POST has zero-or-many COMMENTs. POSTs and TAGs are many-to-many, resolved by the `POST_TAG` junction.

### 4.4 Translating an ER model to tables — the mechanical recipe

Once the ER model is right, conversion is almost mechanical:

1. **Each strong entity becomes a table.** Add a surrogate PK (Section 2.2).
2. **Each single-valued attribute becomes a column** with an appropriate type (Section 3) and nullability.
3. **Each 1:N relationship becomes a foreign key on the *many* side**, pointing at the *one* side's PK. Nullable if and only if the relationship is optional. (Section 5.2)
4. **Each M:N relationship becomes a junction table** holding the two foreign keys, with a composite PK of the pair (plus any relationship attributes). (Section 5.3)
5. **Each 1:1 relationship becomes a foreign key with a `UNIQUE` constraint** on the side that "owns" it, or the two share a PK. (Section 5.1)
6. **Each multi-valued attribute becomes its own table** with a FK back to the owner (it was a hidden 1:N).
7. **Apply constraints** (`NOT NULL`, `CHECK`, `UNIQUE`) to encode the business rules you discovered while modeling.

The next section shows steps 3–5 in concrete SQL.

---

## 5. Implementing Relationships in Tables

**[B/I]**

Relationships are represented by **shared key values**, never by pointers. The pattern differs by cardinality. Get these three patterns into muscle memory — almost every schema is built from them.

### 5.1 One-to-one (1:1)

A 1:1 says each row in A pairs with at most one row in B. Two common reasons to *split* what could be one table:
- **Optional/rarely-present heavy data:** keep the hot `users` table lean and move large, seldom-read details (a bio, KYC documents) to `user_profile`, fetched only when needed.
- **Different security/access:** isolate sensitive columns (payment details) in a separate table with tighter permissions.

Implement it by putting a FK to the parent on the child **and marking it `UNIQUE`** — the uniqueness is what turns a 1:N into a 1:1 (each parent appears at most once on the child side).

```sql
CREATE TABLE users (
    id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email TEXT NOT NULL UNIQUE
);

-- 1:1 — each user has at most one profile; each profile belongs to exactly one user.
CREATE TABLE user_profile (
    user_id BIGINT PRIMARY KEY                       -- using the FK itself AS the PK enforces 1:1 elegantly:
            REFERENCES users(id) ON DELETE CASCADE,  -- profile dies with the user (Section 9)
    bio     TEXT,
    avatar_url TEXT
);
-- Because user_id is the PRIMARY KEY of user_profile, a user can appear at most once: a clean 1:1.
-- (Alternative: separate surrogate PK + `user_id BIGINT NOT NULL UNIQUE REFERENCES users(id)`.)
```

### 5.2 One-to-many (1:N) — FK on the many side

The most important and common pattern. **The foreign key always lives on the "many" side**, pointing at the "one" side. Intuition: each `order` has exactly one `customer`, so an order can store *its* single `customer_id`; but a customer has many orders, so a customer row could not store a single `order_id`.

```sql
CREATE TABLE customer (
    id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE "order" (                          -- "order" is a SQL keyword; quote it (or name it orders)
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id BIGINT NOT NULL                 -- FK on the MANY side; NOT NULL means every order must have a customer
                REFERENCES customer(id)
                ON DELETE RESTRICT,             -- don't allow deleting a customer who has orders (Section 9)
    placed_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Index the FK so "all orders for a customer" is fast and parent deletes are cheap (Section 7.2):
CREATE INDEX idx_order_customer ON "order"(customer_id);
```

If the relationship is *optional* ("an order may be placed by a guest with no account"), make the FK **nullable** — but think hard: a nullable FK introduces three-valued logic (Section 3.2) into your joins and is often a sign you should model the optional case differently.

### 5.3 Many-to-many (M:N) — the junction (join) table

You **cannot** store M:N with a single FK on either side, because each side relates to *many* of the other. The solution is a **junction table** (also: join table, associative entity, bridge table, link table) whose rows are the *pairings*. Each pairing row carries two FKs — one to each side — and the **composite PK of the two FKs** ensures a pair appears at most once.

```sql
CREATE TABLE student (
    id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL
);
CREATE TABLE course (
    id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    code TEXT NOT NULL UNIQUE
);

-- The junction resolves M:N into two 1:N relationships (student 1:N enrollment, course 1:N enrollment):
CREATE TABLE enrollment (
    student_id BIGINT NOT NULL REFERENCES student(id) ON DELETE CASCADE,
    course_id  BIGINT NOT NULL REFERENCES course(id)  ON DELETE CASCADE,
    PRIMARY KEY (student_id, course_id)   -- composite PK: one enrollment per (student, course) pair
);

-- You need an index for the OTHER access direction too. The composite PK index serves
-- "courses of a student" (student_id leads). For "students in a course", add:
CREATE INDEX idx_enrollment_course ON enrollment(course_id);
```

### 5.4 M:N *with attributes on the relationship* **[I]**

A pure junction is just two FKs. But the relationship itself often has data: *when* the student enrolled, their *grade*, the *price* paid at the time of an order line. These attributes belong on the junction row — this is sometimes called an **associative entity** and is one of the most common real-world structures.

```sql
CREATE TABLE enrollment (
    student_id  BIGINT NOT NULL REFERENCES student(id) ON DELETE CASCADE,
    course_id   BIGINT NOT NULL REFERENCES course(id)  ON DELETE CASCADE,
    enrolled_at TIMESTAMPTZ NOT NULL DEFAULT now(),     -- attribute OF the relationship
    grade       TEXT CHECK (grade IN ('A','B','C','D','F') OR grade IS NULL),
    PRIMARY KEY (student_id, course_id)
);
```

> **When the relationship gains its own identity** (e.g., the pairing needs to be referenced by *other* tables, or the same pair can recur over time), promote the junction to a full entity with its own surrogate PK and a `UNIQUE(student_id, course_id)` where appropriate. An `order_item` in Section 11 is exactly this: a richly-attributed associative entity between orders and products.

---

## 6. Normalization (and Denormalization)

**[I]**

**Normalization** is a systematic process for organizing columns into tables so that each fact is stored **once, in one place**, eliminating redundancy. The payoff is **update integrity**: if a fact lives in exactly one cell, it cannot become inconsistent. The cost is more tables and more joins. Understanding *which problem each normal form solves* matters far more than memorizing definitions — so each form below leads with the **anomaly it prevents**.

### 6.1 Why redundancy is the enemy — the three anomalies

Consider a deliberately bad, un-normalized table storing orders *and* customer info together:

```text
order_bad(order_id, customer_email, customer_name, customer_city, product, qty)
   1001,  jane@x.com, Jane Doe,   Lagos,  Keyboard, 2
   1002,  jane@x.com, Jane Doe,   Lagos,  Mouse,    1
   1003,  jane@x.com, Jane Dough, Lagos,  Monitor,  1   <-- name disagrees! which is right?
```

Storing Jane's details on every order row causes three classic **anomalies**:
- **Update anomaly:** Jane changes her name. You must update *every* order row. Miss one (row 1003) and the data is now contradictory — the database can't tell you which name is correct.
- **Insertion anomaly:** You can't record a new customer until they place an order, because customer facts only exist embedded in order rows.
- **Deletion anomaly:** Delete Jane's last order and you accidentally erase the only record that Jane (and the fact she's in Lagos) ever existed.

Normalization removes these by splitting `customer` facts into their own table, stored once, and linking by `customer_id`.

### 6.2 Functional dependencies — the underlying theory

A **functional dependency (FD)** `X -> Y` means "the value of X determines the value of Y" — knowing X, there is exactly one Y. In `order(order_id, ...)`, `order_id -> placed_at` (each order has one placed-at time). In the bad table, `customer_email -> customer_name` (an email determines a name). Normalization is, formally, *arranging tables so that every non-key attribute depends on the key, the whole key, and nothing but the key* — that pithy phrase is literally the definition of being in 3NF/BCNF. FDs are the tool you reason with to find where a fact "really belongs."

### 6.3 First Normal Form (1NF) — atomic values, no repeating groups

**Problem it solves:** multi-valued / repeating data in a single cell or row, which you cannot query, constrain, or index properly.

A table is in **1NF** when every cell holds a single **atomic** value (no lists, no nested structures) and there are no "repeating groups" of columns (`phone1, phone2, phone3`). Each row is identified by a key.

```sql
-- NOT 1NF: a list crammed into one column, and repeating-group columns.
CREATE TABLE contact_bad (
    id      BIGINT PRIMARY KEY,
    name    TEXT,
    phones  TEXT,        -- '+234..., +1...'  (CSV — unqueryable, unconstrainable; see Section 14)
    tag1 TEXT, tag2 TEXT, tag3 TEXT   -- repeating group; what about a 4th tag?
);

-- 1NF: the multi-valued attribute becomes its own table (it was a hidden 1:N, Section 4.1).
CREATE TABLE contact (
    id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL
);
CREATE TABLE contact_phone (
    contact_id BIGINT NOT NULL REFERENCES contact(id) ON DELETE CASCADE,
    phone      TEXT   NOT NULL,
    PRIMARY KEY (contact_id, phone)   -- atomic, queryable, unique per contact
);
```

### 6.4 Second Normal Form (2NF) — no partial dependency on a composite key

**Problem it solves:** in a table with a *composite* key, an attribute that depends on only *part* of the key is stored redundantly.

A table is in **2NF** if it is in 1NF **and** every non-key attribute depends on the *whole* primary key, not just part of it. 2NF only has teeth when the PK is composite (with a single-column PK there are no "partial" dependencies).

```text
NOT 2NF — key is (order_id, product_id):
   order_item_bad(order_id, product_id, qty, product_name, product_price)
   FDs:  (order_id, product_id) -> qty             (depends on whole key  OK)
         product_id -> product_name, product_price (depends on PART of key: partial dependency!)
   => product_name/price repeat for every order containing that product.
```

```sql
-- 2NF: attributes that depend only on product_id move to the product table.
CREATE TABLE product (
    id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name  TEXT NOT NULL,
    price_cents BIGINT NOT NULL
);
CREATE TABLE order_item (
    order_id   BIGINT NOT NULL REFERENCES "order"(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL REFERENCES product(id),
    qty        INT NOT NULL CHECK (qty > 0),          -- depends on the WHOLE key — stays
    PRIMARY KEY (order_id, product_id)
);
```

> Note: order lines often *intentionally* copy the price at time of sale (`unit_price_cents`) — that is a deliberate, justified denormalization (a price is a *historical fact* of the order), not a 2NF violation. See Section 6.8 and the e-commerce example.

### 6.5 Third Normal Form (3NF) — no transitive dependency

**Problem it solves:** a non-key attribute that depends on *another non-key attribute* rather than directly on the key — so it's stored redundantly.

A table is in **3NF** if it is in 2NF **and** no non-key attribute depends on another non-key attribute (no **transitive** dependency `key -> A -> B`). The folk version: *every non-key column depends on **the key, the whole key, and nothing but the key.***

```text
NOT 3NF:
   employee_bad(emp_id, name, dept_id, dept_name)
   FDs:  emp_id -> dept_id        (OK direct)
         dept_id -> dept_name     (transitive: dept_name depends on dept_id, not on emp_id)
   => a department's name repeats on every employee in it; rename the dept -> update many rows.
```

```sql
-- 3NF: department name lives once in the department table.
CREATE TABLE department (
    id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL UNIQUE
);
CREATE TABLE employee (
    id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name    TEXT NOT NULL,
    dept_id BIGINT NOT NULL REFERENCES department(id)   -- only the key reference, no dept_name copy
);
```

### 6.6 Boyce-Codd Normal Form (BCNF) — the stricter 3NF

**Problem it solves:** rare cases where a table is in 3NF but still has an FD whose left side is not a candidate key, usually when there are *multiple overlapping candidate keys*.

A table is in **BCNF** if **for every functional dependency `X -> Y`, `X` is a (super)key.** 3NF tolerates a tiny exception (a non-key attribute determining part of a candidate key); BCNF removes it. In practice, **most well-designed schemas that reach 3NF are already in BCNF.** BCNF matters when you have several candidate keys that overlap.

```text
Classic BCNF example — course scheduling:
   booking(student, course, teacher)
   Rules: each course is taught by exactly one teacher  (teacher -> course)
          a student takes a course with that course's teacher
   Candidate keys: (student, course) and (student, teacher).
   FD  teacher -> course  has a LEFT side (teacher) that is NOT a superkey => violates BCNF (though it can be 3NF).
   Fix: split so the teacher->course fact lives once:
        teaches(teacher, course)            -- teacher -> course stored once
        enrolment(student, teacher)         -- who studies under whom
```

### 6.7 How far should you normalize? When to stop

**Target 3NF (≈BCNF) for your transactional/OLTP system of record.** It is the sweet spot: it eliminates the damaging anomalies while keeping joins reasonable. Going further (4NF/5NF, which deal with multi-valued and join dependencies) is occasionally warranted but rarely changes a typical app schema. The practical rule:

> **Normalize until it hurts (3NF), then denormalize until it works (only where measurement proves you must).**

OLTP (lots of small writes, integrity-critical) means normalize. Analytics/reporting/OLAP (read-heavy, few writes, query speed paramount) often means deliberately *de*normalized (star/snowflake schemas, wide tables) because there the cost-benefit flips.

### 6.8 Denormalization — when and why to break the rules **[I/A]**

**Denormalization** is *deliberately* introducing redundancy to improve read performance, accepting the burden of keeping the copies in sync. It is a *trade-off made with eyes open*, never an excuse to skip modeling. Do it only when a *measured* read path is too slow and you've exhausted indexing.

Common, legitimate denormalizations:

- **Historical snapshots (not really denormalization):** copy `unit_price` onto `order_item` because the price *at time of sale* is a distinct fact from the product's current price. This is correct modeling, not a hack.
- **Cached aggregates / counters:** store `comment_count` on `post` to avoid `COUNT(*)` on every page load. Keep it in sync with a **trigger** or in the same transaction as the insert. Trade-off: the count can drift if a write path forgets to update it — triggers reduce that risk.
- **Read models / materialized views:** precompute a join-heavy report into a materialized view (`POSTGRESQL_GUIDE.md` §15) and refresh it on a schedule. The source stays normalized; the read model is derived.
- **Duplicated lookup labels** in a wide reporting table to avoid joins at query time.

```sql
-- Denormalized counter kept correct by a trigger (so no write path can forget it):
ALTER TABLE post ADD COLUMN comment_count INT NOT NULL DEFAULT 0;

CREATE FUNCTION bump_comment_count() RETURNS trigger AS $$
BEGIN
  IF (TG_OP = 'INSERT') THEN
    UPDATE post SET comment_count = comment_count + 1 WHERE id = NEW.post_id;
  ELSIF (TG_OP = 'DELETE') THEN
    UPDATE post SET comment_count = comment_count - 1 WHERE id = OLD.post_id;
  END IF;
  RETURN NULL;
END; $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_comment_count
AFTER INSERT OR DELETE ON comment
FOR EACH ROW EXECUTE FUNCTION bump_comment_count();
```

**Costs you accept with any denormalization:** extra write work, the risk of divergence (the cached value disagrees with the truth), more complex writes, and more storage. Always keep the *normalized* source of truth so you can rebuild the derived copy. Document *why* each denormalization exists.

---

## 7. Indexing for Design

**[I]**

Indexing is where logical design meets physical performance. You don't design a schema in a vacuum — you design it *for queries*, and indexes are how the database answers those queries without scanning whole tables. This section is about the *design decisions* (which columns, which order, which kind); for the internals (B-tree vs GIN/GiST/BRIN, `EXPLAIN`/`EXPLAIN ANALYZE`, index-only scans, bloat) see **`POSTGRESQL_GUIDE.md` §11–12**.

### 7.1 What an index is, and the fundamental trade-off

An **index** is an auxiliary data structure (usually a **B-tree**) that maps column values to the rows that contain them, kept sorted so the database can binary-search instead of scanning every row. Without an index, `WHERE email = 'x'` reads the whole table (a *sequential scan*); with one, it jumps near-instantly to the match.

The trade-off is the entire game: **indexes make matching reads faster but make every write slower and use disk**, because every `INSERT`/`UPDATE`/`DELETE` must also update each index on the affected columns. So indexing is *balancing read speedups against write cost and storage*. The skill is indexing the columns that queries actually filter, join, and sort on — and **not** indexing everything "just in case" (Section 7.6).

### 7.2 Index your foreign keys — almost always **[critical]**

As noted in Section 2.4, a FK does **not** auto-create an index on the child column. You should create one yourself in two situations (which together cover nearly all FKs):

1. **You query "children of a parent"** (`SELECT * FROM orders WHERE customer_id = ?`). Extremely common; without the index it's a full scan.
2. **The parent rows get updated or deleted.** To enforce the FK, the engine must check "are there child rows?" On delete/update of a parent, an *unindexed* child FK forces a full scan of the child table — and (in Postgres) can take heavier locks. On big child tables this turns a quick delete into a multi-second stall.

```sql
CREATE INDEX idx_order_customer_id ON "order"(customer_id);   -- index every FK like this
```

> This is the single most common indexing omission in real schemas. Make "index the FK" automatic.

### 7.3 What else to index

- **Columns in `WHERE` equality/range filters** you run often (`status`, `created_at`).
- **Columns you `JOIN` on** (usually the FKs — see 7.2).
- **Columns you `ORDER BY`** for large result sets / pagination (a B-tree is already sorted).
- **Uniqueness rules** — a `UNIQUE` constraint *is* a unique index; it enforces the rule *and* speeds lookups.
- **Lookups by natural/business key** (`sku`, `slug`, `email`) — these are usually `UNIQUE` anyway.

Don't index: tiny tables (a scan is already fast), very low-cardinality columns *alone* (a boolean — though a **partial** index can help, below), or columns you never filter/sort/join on.

### 7.4 Composite indexes & the column-order rule **[I]**

A **composite (multi-column) index** indexes a *tuple* of columns in order. The order is the crux: a composite index on `(a, b)` is sorted by `a`, then by `b` within each `a`. This is the **leftmost-prefix rule**: the index can serve queries that filter on `a`, or on `a AND b`, but **not on `b` alone** (the b-values are scattered across every a-group). Design the order to match your query: put the column used in **equality** filters first, then range/sort columns.

```sql
-- Query: "a user's orders, newest first" — filter by user_id (equality), sort by placed_at (range/sort).
CREATE INDEX idx_order_user_placed ON "order"(user_id, placed_at DESC);
--   serves:  WHERE user_id = ?                          (leftmost prefix)         OK
--   serves:  WHERE user_id = ? ORDER BY placed_at DESC  (filter + sort from one)  OK
--   does NOT serve:  WHERE placed_at > ?  (alone)        (not the leftmost column) NO
```

### 7.5 Covering, partial, and unique indexes **[I]**

- **Covering index (`INCLUDE`):** add non-key columns to the index so a query can be answered *entirely from the index* without touching the table (an **index-only scan**). Great for hot read paths.
- **Partial index (`WHERE`):** index only the rows matching a predicate — smaller, faster, cheaper to maintain. Perfect for "mostly-irrelevant" values (e.g., index only `active` rows, or only non-NULLs).
- **Unique index:** enforces a constraint *and* indexes; a **partial unique index** enforces conditional uniqueness (e.g., "one active subscription per user").

```sql
-- Covering: serve "email + name for a user id" from the index alone.
CREATE INDEX idx_users_id_incl ON users(id) INCLUDE (email, name);

-- Partial: only index unshipped orders (the queue you actually scan); ignore the millions of shipped ones.
CREATE INDEX idx_order_pending ON "order"(placed_at) WHERE status = 'pending';

-- Partial UNIQUE: enforce "at most one ACTIVE subscription per user" (others may repeat).
CREATE UNIQUE INDEX uq_one_active_sub ON subscription(user_id) WHERE status = 'active';
```

### 7.6 Over-indexing pitfalls

Too many indexes is its own problem: every write maintains them all (slower inserts/updates, more WAL, more bloat), they consume disk, and *redundant* indexes (e.g., both `(a)` and `(a, b)` — the first is covered by the second's leftmost prefix) are pure waste. Periodically check `pg_stat_user_indexes` for **never-used indexes** and drop them. Index deliberately, from real query patterns, and verify with `EXPLAIN ANALYZE` (see `POSTGRESQL_GUIDE.md` §12) that the index is actually used.

```sql
-- Find unused indexes (idx_scan = 0 means the planner never chose it):
SELECT relname AS table, indexrelname AS index, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY relname;
```

---

## 8. Schema Design Patterns

**[I/A]**

This is the advanced centerpiece. Beyond the mechanics of keys and relationships, real systems repeatedly need a handful of *structural patterns*. For each: what it is, when to use it, the trade-offs, and SQL. Knowing these by name lets you reach for the right one instead of reinventing (often badly).

### 8.1 Audit columns (created_at / updated_at / created_by)

**What & why:** Almost every table benefits from knowing *when* a row was created and last changed, and *who* did it. These "audit columns" are the cheapest form of traceability and are invaluable for debugging, sorting, sync, and compliance. The subtlety is keeping `updated_at` honest: apps forget to set it, so enforce it in the database with a trigger.

```sql
CREATE TABLE article (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title      TEXT NOT NULL,
    body       TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),  -- set once, by the DB clock
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),  -- maintained by the trigger below
    created_by BIGINT REFERENCES users(id),         -- who created it (nullable for system rows)
    updated_by BIGINT REFERENCES users(id)
);

-- A reusable trigger function: stamp updated_at on every UPDATE so the app can't forget it.
CREATE FUNCTION set_updated_at() RETURNS trigger AS $$
BEGIN
    NEW.updated_at := now();   -- always overwrite, regardless of what the app sent
    RETURN NEW;
END; $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_article_updated_at
BEFORE UPDATE ON article
FOR EACH ROW EXECUTE FUNCTION set_updated_at();   -- reuse this trigger fn on every audited table
```

**Trade-off:** trivial storage cost for huge operational value. Use `TIMESTAMPTZ` (Section 3.1). For *full* change history (old/new values of every column), you need a history table (8.10) or `pgaudit`/temporal tables — audit columns only tell you the *latest* change.

### 8.2 Soft deletes

**What & why:** A **soft delete** marks a row as deleted (`deleted_at` timestamp or `is_deleted` flag) instead of physically removing it. Use it when you must retain data for audit/recovery/legal reasons, support "undo," or preserve referential history (an order whose product was "deleted" must still show what was bought). The cost is that *every* query must now remember to exclude soft-deleted rows — forget once and deleted data leaks into the UI.

```sql
ALTER TABLE product ADD COLUMN deleted_at TIMESTAMPTZ;  -- NULL = live; non-NULL = soft-deleted (when)

-- "Delete" = stamp it:
UPDATE product SET deleted_at = now() WHERE id = 42;

-- Every read must filter — easy to forget, so encapsulate it in a VIEW:
CREATE VIEW product_live AS SELECT * FROM product WHERE deleted_at IS NULL;
-- ...then apps SELECT FROM product_live and can't accidentally see deleted rows.

-- Gotcha: a UNIQUE(sku) now blocks re-using a soft-deleted SKU. Make uniqueness apply to LIVE rows only:
CREATE UNIQUE INDEX uq_product_sku_live ON product(sku) WHERE deleted_at IS NULL;
```

**Trade-offs / pitfalls:** tables grow forever (consider archiving); unique constraints must be made partial (above); FKs still see the "deleted" row (often desirable); and you must be disciplined everywhere. If you don't truly need recovery, a real `DELETE` plus a history table is cleaner.

### 8.3 Hierarchical / tree data — four approaches compared **[A]**

Trees (categories with subcategories, org charts, comment threads, file systems) are a recurring modeling challenge because SQL is set-oriented, not recursive by nature. There are four canonical approaches, each with a different read/write trade-off. **Choosing among them is a classic senior-level design decision**, so understand all four.

| Approach | Read subtree | Insert / move node | Storage | Best when |
|---|---|---|---|---|
| **Adjacency list** (each row stores `parent_id`) | Hard (recursion needed) | Trivial | Tiny | Default; shallow trees; you have recursive CTEs |
| **Materialized path** (store `'1/4/9/'` path string) | Easy (`LIKE 'path%'`) | Moderate (rewrite descendants on move) | Small | Read-heavy; path is meaningful (URLs, breadcrumbs) |
| **Nested set** (each row has `lft`/`rgt` numbers) | Very easy & fast (range query) | Painful (renumber many rows) | Small | Read-mostly, rarely-changing trees |
| **Closure table** (separate table of all ancestor→descendant pairs) | Easy & fast (join) | Moderate (insert N rows per node) | Larger | Frequent reads *and* writes; arbitrary queries; the robust general answer |

**(a) Adjacency list** — the natural, simplest model. Modern Postgres makes reads tractable with a **recursive CTE**.

```sql
CREATE TABLE category (
    id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name      TEXT NOT NULL,
    parent_id BIGINT REFERENCES category(id)   -- NULL parent = a root node
);
CREATE INDEX idx_category_parent ON category(parent_id);  -- index the FK (Section 7.2)

-- Read the whole subtree under category 4 with a recursive CTE:
WITH RECURSIVE subtree AS (
    SELECT id, name, parent_id, 1 AS depth
    FROM category WHERE id = 4              -- anchor: the starting node
  UNION ALL
    SELECT c.id, c.name, c.parent_id, s.depth + 1
    FROM category c JOIN subtree s ON c.parent_id = s.id   -- recurse to children
)
SELECT * FROM subtree;
```

**(b) Materialized path** — store the ancestry as a string; reads become prefix matches.

```sql
CREATE TABLE category_mp (
    id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    path TEXT NOT NULL    -- e.g. '1/4/9/' ; the node's own id is the last segment
);
-- All descendants of '1/4/' (subtree read is a single index-friendly prefix scan):
SELECT * FROM category_mp WHERE path LIKE '1/4/%';
-- Moving a subtree means rewriting the path prefix of every descendant. (PG's `ltree` type does this natively.)
```

**(c) Nested set** — number nodes by a depth-first walk; a node's subtree is everything whose numbers fall between its `lft` and `rgt`.

```sql
CREATE TABLE category_ns (
    id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    lft  INT NOT NULL,   -- assigned by a pre-order traversal
    rgt  INT NOT NULL
);
-- Read a subtree with NO recursion — just a range (very fast for reads):
SELECT child.* FROM category_ns parent
JOIN category_ns child ON child.lft BETWEEN parent.lft AND parent.rgt
WHERE parent.id = 4;
-- BUT inserting/moving a node requires renumbering lft/rgt of many rows — write-hostile.
```

**(d) Closure table** — the robust general solution: a side table holding *every* ancestor-descendant pair (including self), so any tree query is a simple join.

```sql
CREATE TABLE node (
    id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL
);
-- One row per (ancestor, descendant) pair, plus depth. A node has a (self,self,0) row.
CREATE TABLE node_closure (
    ancestor_id   BIGINT NOT NULL REFERENCES node(id) ON DELETE CASCADE,
    descendant_id BIGINT NOT NULL REFERENCES node(id) ON DELETE CASCADE,
    depth         INT    NOT NULL,
    PRIMARY KEY (ancestor_id, descendant_id)
);
CREATE INDEX idx_closure_desc ON node_closure(descendant_id);

-- Descendants of node 4: trivial join, no recursion.
SELECT n.* FROM node_closure c JOIN node n ON n.id = c.descendant_id WHERE c.ancestor_id = 4;
-- Ancestors of node 9 (its breadcrumb): symmetric query on descendant_id.
-- Cost: inserting a node writes one closure row per ancestor (depth rows); moves rewrite a subtree's rows.
```

> **Rule of thumb:** start with **adjacency list + recursive CTEs** (simplest, fine for most apps). Switch to a **closure table** when trees are large and you need fast arbitrary ancestor/descendant queries with ongoing writes. Use **nested set** only for nearly-static, read-only hierarchies.

### 8.4 Polymorphic associations — the problem and the alternatives **[A]**

**What it is:** A "polymorphic association" is when one table needs to reference *one of several* parent types — e.g., a `comment` that can attach to a `post` *or* a `photo` *or* a `video`. The tempting (and problematic) pattern stores a type discriminator plus a generic id:

```sql
-- ANTI-PATTERN (illustrative): one FK column that points to "whatever type_name says".
CREATE TABLE comment_bad (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    commentable_type TEXT   NOT NULL,   -- 'post' | 'photo' | 'video'
    commentable_id   BIGINT NOT NULL,   -- id within THAT table
    body          TEXT NOT NULL
);
-- Why it's bad: you CANNOT put a real FOREIGN KEY on commentable_id (it points at different tables),
-- so the database can't guarantee referential integrity — orphan comments become possible.
-- Joins require CASE/UNION gymnastics, and the planner can't use FK-based optimizations.
```

The core defect: **you lose the foreign key**, the very thing that makes the data relational and trustworthy. Three sound alternatives:

```sql
-- ALTERNATIVE 1: separate join/child tables per type — each gets a REAL foreign key.
CREATE TABLE comment (
    id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    body TEXT NOT NULL
);
CREATE TABLE post_comment  (post_id  BIGINT REFERENCES post(id),   comment_id BIGINT REFERENCES comment(id), PRIMARY KEY(post_id, comment_id));
CREATE TABLE photo_comment (photo_id BIGINT REFERENCES photo(id),  comment_id BIGINT REFERENCES comment(id), PRIMARY KEY(photo_id, comment_id));
-- Integrity is enforced; the cost is one join table per parent type.

-- ALTERNATIVE 2: a shared supertype the children all reference (table inheritance via a common parent).
CREATE TABLE commentable (                      -- every "thing that can be commented on" gets a row here
    id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    kind TEXT NOT NULL CHECK (kind IN ('post','photo','video'))
);
CREATE TABLE post  (commentable_id BIGINT PRIMARY KEY REFERENCES commentable(id), title TEXT);
CREATE TABLE photo (commentable_id BIGINT PRIMARY KEY REFERENCES commentable(id), url   TEXT);
CREATE TABLE comment2 (
    id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    commentable_id BIGINT NOT NULL REFERENCES commentable(id),  -- ONE real FK, to the supertype
    body TEXT NOT NULL
);
-- Now a single real FK works because everything commentable shares one identity table.

-- ALTERNATIVE 3: multiple nullable FKs + a CHECK that exactly one is set (good for a small fixed set).
CREATE TABLE comment3 (
    id       BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    post_id  BIGINT REFERENCES post(id),
    photo_id BIGINT REFERENCES photo(id),
    body     TEXT NOT NULL,
    CHECK (num_nonnulls(post_id, photo_id) = 1)  -- exactly one parent; both real FKs, both enforced
);
```

> **Guidance:** prefer Alternative 2 (shared supertype) or 3 (exclusive-arc with `CHECK`) for a *small, known* set of parent types — they keep real FKs. Use the discriminator anti-pattern only as a last resort (large/open-ended type set), and then enforce integrity in the app and reconcile with periodic checks.

### 8.5 Lookup / reference tables vs ENUMs **[I]**

For a column with a fixed small set of allowed values (`status`, `role`, `currency`), you have three choices, and the decision hinges on *how often the set changes* and *whether the values need attributes*.

| Approach | How | Pros | Cons |
|---|---|---|---|
| **`CHECK (x IN (...))`** | inline constraint | simplest; no extra table | adding a value = `ALTER TABLE` (DDL/migration); no metadata per value |
| **Native `ENUM` type** | `CREATE TYPE ... AS ENUM` | type-safe, compact, ordered | adding/reordering values is awkward (`ALTER TYPE`); Postgres-specific; can't carry attributes; can't easily remove a value |
| **Lookup table** + FK | a `status(code PK, label, ...)` table | values are *data* (add via INSERT, no migration); can carry attributes (label, sort order, active flag); real FK integrity; join to get display text | extra table + join; slightly more ceremony |

```sql
-- ENUM: great when the set is truly fixed and you want a compact, ordered type.
CREATE TYPE order_status AS ENUM ('pending','paid','shipped','delivered','cancelled');
CREATE TABLE order_e (id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY, status order_status NOT NULL DEFAULT 'pending');

-- Lookup table: prefer when values change over time or need metadata. Adding a status = an INSERT, not a migration.
CREATE TABLE order_status_ref (
    code      TEXT PRIMARY KEY,                 -- 'pending','paid',...  natural key is fine for a lookup
    label     TEXT NOT NULL,                    -- human-friendly display text
    sort_order INT  NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE
);
CREATE TABLE order_l (
    id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status TEXT NOT NULL REFERENCES order_status_ref(code) DEFAULT 'pending'  -- FK gives integrity + joinable labels
);
```

> **Default advice (2026):** reach for a **lookup table** for business-domain value sets (statuses, types, categories) because they evolve and benefit from metadata; reserve **`ENUM`** for truly immutable, code-level sets where compactness matters. `CHECK IN (...)` is fine for tiny, stable, attribute-free sets.

### 8.6 Status / state machines

**What & why:** Many entities have a lifecycle (`order`: pending → paid → shipped → delivered, with cancellation paths). Modeling this as a free-text column lets impossible transitions slip in (`delivered` → `pending`). Encode the *allowed transitions* so the database refuses illegal ones, and record the history of transitions.

```sql
-- The set of legal transitions, as DATA (easy to extend without code changes):
CREATE TABLE order_transition (
    from_status TEXT NOT NULL,
    to_status   TEXT NOT NULL,
    PRIMARY KEY (from_status, to_status)
);
INSERT INTO order_transition VALUES
  ('pending','paid'), ('paid','shipped'), ('shipped','delivered'),
  ('pending','cancelled'), ('paid','cancelled');

-- A trigger enforces "you may only move along a defined edge":
CREATE FUNCTION enforce_order_transition() RETURNS trigger AS $$
BEGIN
  IF NEW.status IS DISTINCT FROM OLD.status THEN          -- only check on an actual status change
    IF NOT EXISTS (SELECT 1 FROM order_transition
                   WHERE from_status = OLD.status AND to_status = NEW.status) THEN
      RAISE EXCEPTION 'illegal order status transition: % -> %', OLD.status, NEW.status;
    END IF;
  END IF;
  RETURN NEW;
END; $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_order_transition
BEFORE UPDATE OF status ON order_l
FOR EACH ROW EXECUTE FUNCTION enforce_order_transition();

-- And log every transition for an audit trail (the "history" pattern, 8.10):
CREATE TABLE order_status_log (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id   BIGINT NOT NULL REFERENCES order_l(id),
    from_status TEXT, to_status TEXT NOT NULL,
    changed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    changed_by BIGINT REFERENCES users(id)
);
```

### 8.7 Multi-tenancy — three strategies **[A]**

**What & why:** A multi-tenant SaaS serves many customer organizations ("tenants") from shared infrastructure. The central design question is **how strongly to isolate tenants' data**, trading isolation/compliance against operational simplicity and cost. Three models:

| Model | Isolation | Cost / ops | Cross-tenant queries | Best when |
|---|---|---|---|---|
| **Shared schema, `tenant_id` column** | Logical only (one bug leaks data) | Cheapest, simplest, scales to many tenants | Easy (it's all one DB) | SaaS with many small/medium tenants; default |
| **Schema-per-tenant** (one PG schema each) | Stronger (separate namespaces) | Medium; migrations run per-schema | Harder | Moderate tenant count; per-tenant customization |
| **Database-per-tenant** | Strongest (physical) | Highest; many DBs to migrate/back up | Hardest | Few large/enterprise tenants; strict compliance |

The shared-schema model is most common. Its danger is a forgotten `WHERE tenant_id = ?` leaking one tenant's data to another — so back it with **Row-Level Security (RLS)** so the database *itself* enforces isolation (see `POSTGRESQL_GUIDE.md` §19).

```sql
-- Shared-schema multi-tenancy: every tenant-owned table carries tenant_id, and it leads composite indexes.
CREATE TABLE tenant (id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY, name TEXT NOT NULL);

CREATE TABLE project (
    id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id BIGINT NOT NULL REFERENCES tenant(id),
    name      TEXT NOT NULL,
    UNIQUE (tenant_id, name)            -- names unique WITHIN a tenant, not globally
);
CREATE INDEX idx_project_tenant ON project(tenant_id);  -- tenant_id leads every index

-- Defense in depth: let the DB enforce tenant isolation regardless of app bugs.
ALTER TABLE project ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON project
  USING (tenant_id = current_setting('app.tenant_id')::BIGINT);  -- app sets this per request/connection
-- The app does:  SET app.tenant_id = '42';  and can then NEVER see another tenant's rows.
```

### 8.8 Versioning / history tables (temporal data) **[A]**

**What & why:** Sometimes you must answer "what did this record look like *last March*?" — for audit, compliance, or showing edit history. Keep the *current* row in the main table and append immutable snapshots to a **history table** on every change (via a trigger), or use a **bitemporal** design with validity ranges.

```sql
-- Main (current) table:
CREATE TABLE policy (
    id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    holder    TEXT NOT NULL,
    premium_cents BIGINT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- History table: one row per version, with the window during which it was the truth.
CREATE TABLE policy_history (
    id          BIGINT NOT NULL,                  -- the policy's id (not unique here)
    holder      TEXT NOT NULL,
    premium_cents BIGINT NOT NULL,
    valid_from  TIMESTAMPTZ NOT NULL,
    valid_to    TIMESTAMPTZ NOT NULL DEFAULT 'infinity',  -- 'infinity' = still current
    PRIMARY KEY (id, valid_from)
);
-- A BEFORE UPDATE trigger closes the old version (set valid_to = now()) and inserts the new one.
-- Postgres range types + an EXCLUDE constraint can prevent overlapping validity windows entirely.
```

> Postgres lacks built-in `SYSTEM VERSIONING` (SQL Server/MariaDB have it as "temporal tables"); the trigger-driven history table above is the portable pattern. See also 8.10 for a generic audit-log variant.

### 8.9 Slowly Changing Dimensions (SCD) — for data warehouses **[A]**

**What & why:** In analytics/warehouse design (star schemas), *dimension* tables (customer, product) change over time, and you must decide what happens to history. The standard taxonomy:

- **SCD Type 1 — overwrite:** just `UPDATE` the dimension row; no history kept. Use when old values don't matter (fixing a typo).
- **SCD Type 2 — add a new row:** keep the old row, insert a new version with `valid_from`/`valid_to` (or `effective`/`is_current`) so facts join to the value that was true *at the time*. The workhorse for "track history."
- **SCD Type 3 — add a column:** keep `previous_value` alongside `current_value`. Limited (only one step of history); niche.

```sql
-- SCD Type 2 dimension: each customer change creates a new versioned row; facts reference the surrogate key.
CREATE TABLE dim_customer (
    customer_key BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- surrogate per VERSION (not natural id)
    customer_id  BIGINT NOT NULL,                                  -- the stable business id
    name         TEXT NOT NULL,
    city         TEXT NOT NULL,
    valid_from   DATE NOT NULL,
    valid_to     DATE NOT NULL DEFAULT '9999-12-31',
    is_current   BOOLEAN NOT NULL DEFAULT TRUE
);
-- A fact row stores the customer_key that was current at the event time, freezing the dimension's history.
```

### 8.10 The EAV anti-pattern — why it tempts and why to avoid **[A]**

**What it is:** **Entity-Attribute-Value (EAV)** stores attributes as *rows* instead of *columns*: a generic `(entity_id, attribute_name, value)` table that can hold "any" attribute of "any" entity. It tempts whenever requirements say "users can define their own custom fields" or "every product type has totally different attributes."

```sql
-- EAV (use with great caution):
CREATE TABLE entity_attr (
    entity_id BIGINT NOT NULL,
    attribute TEXT   NOT NULL,        -- 'color', 'weight_kg', 'screen_size'
    value     TEXT   NOT NULL,        -- everything stored as text => no real typing
    PRIMARY KEY (entity_id, attribute)
);
```

**Why it's usually an anti-pattern:** you throw away nearly everything the relational model gives you. There are **no real data types** (everything is text), **no per-attribute constraints or FKs**, **no easy multi-attribute query** (each attribute is another self-join — "products that are red AND under 2kg AND in stock" becomes a monstrous query), the optimizer can't reason about it, and reconstructing one entity's full record means pivoting many rows.

**When it's *legitimately* tempting and the better alternatives:**
- **Truly dynamic, user-defined fields:** prefer a **`JSONB` column** (Postgres) — you keep one row per entity, can index inside it (GIN), validate with `CHECK`, and query with JSON operators. This is the modern answer to "arbitrary custom attributes."
- **A fixed-but-large set of optional attributes:** just use nullable columns, or a per-subtype table (class-table inheritance).
- **A small number of well-known attribute groups:** model them as real tables.

```sql
-- Modern alternative to EAV for custom fields: a typed-ish JSONB blob with validation + indexing.
CREATE TABLE product_dyn (
    id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name  TEXT NOT NULL,
    attrs JSONB NOT NULL DEFAULT '{}'::jsonb
            CHECK (jsonb_typeof(attrs) = 'object')   -- enforce it's an object, not a scalar/array
);
CREATE INDEX idx_product_attrs ON product_dyn USING GIN (attrs);  -- index arbitrary keys/values
-- Query: products whose custom attr "color" = 'red'
SELECT * FROM product_dyn WHERE attrs @> '{"color":"red"}';
```

> **Bottom line:** if you can name the attributes, make them columns. If they're genuinely open-ended, use `JSONB`, not EAV. Reserve true EAV for systems whose entire purpose is user-defined schemas (and accept the query pain).

---

## 9. Referential Actions & Integrity

**[I]**

A foreign key guarantees no orphans (Section 2.4). But what should happen to *child* rows when the *parent* they point to is deleted or its key updated? That is the **referential action**, declared with `ON DELETE` / `ON UPDATE`. Choosing it correctly is a real design decision — it encodes a business rule about what data may outlive what.

### 9.1 The available actions

| Action | On parent DELETE/UPDATE, the child... | Use when |
|---|---|---|
| `NO ACTION` (default) | ...blocks it (checked at end of statement/txn) | default; you want an explicit error |
| `RESTRICT` | ...blocks it immediately | "you must not delete a customer who has orders" |
| `CASCADE` | ...is deleted/updated too | child is *part of* the parent and meaningless without it |
| `SET NULL` | ...has its FK set to NULL | the link is optional; child survives, orphaned-on-purpose |
| `SET DEFAULT` | ...has its FK set to its column default | reassign to a fallback parent (rare) |

```sql
-- CASCADE: order_items are PART OF an order — delete the order, the lines go with it.
CREATE TABLE order_item (
    order_id   BIGINT NOT NULL REFERENCES "order"(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL REFERENCES product(id) ON DELETE RESTRICT,  -- but DON'T let a product vanish from history
    PRIMARY KEY (order_id, product_id)
);

-- RESTRICT/NO ACTION: protect referenced master data.
CREATE TABLE "order" (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customer(id) ON DELETE RESTRICT  -- can't delete a customer with orders
);

-- SET NULL: an author's posts survive if the author is deleted (FK becomes NULL ⇒ must be nullable).
CREATE TABLE post (
    id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    author_id BIGINT REFERENCES users(id) ON DELETE SET NULL,   -- column MUST be nullable for SET NULL
    title     TEXT NOT NULL
);
```

### 9.2 Choosing the right action — the reasoning

Ask: **"Does the child have meaning without this parent?"**
- **No, it's a part of the parent** (order line, profile, image of a product) → `CASCADE`. Deleting the whole aggregate is correct.
- **Yes, it's an independent entity that merely references the parent** (an order references a customer; a post references its author) → `RESTRICT` (forbid the delete) or `SET NULL` (keep the child, drop the link). Cascading here would silently destroy independent data — a classic accidental-mass-deletion footgun.

> **Caution with `CASCADE`:** it is convenient but dangerous — a single `DELETE FROM customer WHERE id = 1` can ripple through many tables and erase thousands of rows with no warning. Use it only for genuine parent/child *composition*, and combine with soft deletes (8.2) where recovery matters.

### 9.3 Enforcing invariants beyond simple FKs **[I]**

Real rules are often richer than "no orphans." Push them into the database so they're inviolable:

- **`CHECK` constraints** for intra-row rules (`sale_price <= price`, Section 3.4).
- **Partial unique indexes** for conditional uniqueness ("one primary address per user", Section 7.5).
- **Exclusion constraints** (Postgres) for "no two overlapping" rules — e.g., no double-booking a room.
- **Triggers** for multi-row/multi-table invariants that constraints can't express (state machines, 8.6).
- **`DEFERRABLE` constraints** when you must temporarily violate integrity *within* a transaction (e.g., inserting two rows that reference each other) and have it checked at `COMMIT`.

```sql
-- Exclusion constraint: prevent overlapping bookings for the same room (needs btree_gist extension).
CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE TABLE booking (
    id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id BIGINT NOT NULL REFERENCES room(id),
    during  TSTZRANGE NOT NULL,                       -- the time window as a range type
    EXCLUDE USING gist (room_id WITH =, during WITH &&)  -- no two rows with same room AND overlapping (&&) range
);

-- Deferred FK: allow temporary inconsistency inside a transaction, validated at COMMIT.
ALTER TABLE node ADD CONSTRAINT fk_parent FOREIGN KEY (parent_id)
  REFERENCES node(id) DEFERRABLE INITIALLY DEFERRED;
```

---

## 10. Transactions & Concurrency in Design

**[I/A]**

Schema design and concurrency are intertwined: *how* you structure data and *what guarantees you need under concurrent access* shape each other. You don't have to be a locking expert, but a designer must understand the levers. For the deep mechanics (MVCC, lock types, `SKIP LOCKED`, deadlocks) see `POSTGRESQL_GUIDE.md` §13.

### 10.1 ACID and why it shapes design

A **transaction** is a unit of work that is **A**tomic (all-or-nothing), **C**onsistent (leaves the DB valid w.r.t. constraints), **I**solated (concurrent transactions don't see each other's partial work), and **D**urable (committed work survives crashes). The design implication: **anything that must be true together should be writable together in one transaction.** This is the concept of an **aggregate** — a cluster of rows (an order + its items + a payment) that changes as a unit. Design tables so that the natural transaction boundary matches the consistency boundary (you'll see this in Section 11's "create order with items" transaction).

### 10.2 Isolation levels affect what your schema must defend against

Higher isolation prevents more anomalies but costs concurrency. Postgres defaults to **Read Committed**; **Repeatable Read** and **Serializable** are stronger. The design takeaway: if you rely on a *read-then-write* invariant ("check stock, then decrement"), `Read Committed` alone can let two transactions both pass the check — a **race condition**. You must defend it with either a stronger isolation level (and a retry loop on serialization failures, SQLSTATE `40001`), explicit row locks (`SELECT ... FOR UPDATE`), or an atomic conditional write.

```sql
-- Atomic conditional decrement avoids the read-then-write race WITHOUT extra locking ceremony:
UPDATE product
SET    stock = stock - 1
WHERE  id = 42 AND stock >= 1     -- the WHERE makes the check-and-decrement a single atomic step
RETURNING stock;                  -- 0 rows returned ⇒ out of stock; the app reacts accordingly
```

### 10.3 Optimistic locking with a version column **[I]**

**What & why:** When two users edit the same record, the last writer can silently clobber the first ("lost update"). **Optimistic locking** detects this without holding locks: add a `version` (or use `updated_at`) column; on update, require the version to still match what you read, and bump it. If zero rows update, someone changed it first — the app retries or warns.

```sql
ALTER TABLE article ADD COLUMN version INT NOT NULL DEFAULT 0;

-- The app read version = 7 earlier. Now it writes ONLY IF nobody else changed it since:
UPDATE article
SET    body = $new_body, version = version + 1
WHERE  id = $id AND version = 7;        -- matches only if still version 7
-- rows affected = 1 ⇒ success;  = 0 ⇒ a concurrent edit happened ⇒ reload & retry (don't blindly overwrite)
```

This is "optimistic" (assume conflicts are rare, detect at write time) versus "pessimistic" (`SELECT ... FOR UPDATE` locks the row up front). Optimistic suits low-contention, user-facing edits; pessimistic suits hot rows with frequent conflicts.

### 10.4 Idempotency keys **[I/A]**

**What & why:** Networks retry. A user double-clicks "Pay"; a client resends a request after a timeout that actually succeeded. Without protection you charge twice or create duplicate orders. An **idempotency key** is a client-supplied unique token stored with a `UNIQUE` constraint, so a retried request with the same key is recognized and the original result returned instead of re-executed. This is a *schema* solution to an *operational* problem.

```sql
CREATE TABLE payment (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id        BIGINT NOT NULL REFERENCES "order"(id),
    amount_cents    BIGINT NOT NULL CHECK (amount_cents > 0),
    idempotency_key TEXT NOT NULL UNIQUE,        -- client sends a UUID per logical attempt
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- The insert with ON CONFLICT turns a duplicate retry into a harmless no-op (return the existing row):
INSERT INTO payment (order_id, amount_cents, idempotency_key)
VALUES ($1, $2, $3)
ON CONFLICT (idempotency_key) DO NOTHING       -- second identical request inserts nothing
RETURNING id;                                   -- empty ⇒ already processed; look up & return the original
```

---

## 11. Complete Worked Example — E-commerce

**[I/A]**

Now we put everything together: design a realistic e-commerce schema from ER reasoning through normalized DDL and indexes, then *consume* it from **both** a Node.js (NestJS + Prisma) and a Go (Gin + pgx) backend. Watch how a clean schema makes the application code straightforward.

### 11.1 ER reasoning

The entities and the relationships between them:

- **users** place **orders** (1:N) and write **reviews** (1:N).
- A user has many **addresses** (1:N); an order references a shipping and a billing address.
- **products** belong to **categories**. A product is M:N with categories (a shirt is in "Apparel" *and* "Sale"), so we need a junction `product_category`. Categories form a **tree** (adjacency list, 8.3a).
- A product has many **product_variants** (size/color SKUs) — 1:N. *Stock and price live on the variant*, because that's the thing actually sold.
- A user has one active **cart** (1:1-ish) holding **cart_items** (M:N user↔variant with quantity = associative entity).
- An **order** has many **order_items** (associative entity between order and variant) that **snapshot price at purchase time** (justified denormalization, 6.8). An order has **payments** (1:N to allow retries/partial payments) using idempotency keys (10.4) and follows a status state machine (8.6).
- Users write **reviews** of products (1:N from each side; a user reviews a product at most once → `UNIQUE(user_id, product_id)`).

```text
USER ||--o< ADDRESS                 USER ||--o< ORDER >|--|| ADDRESS (ship/bill)
USER ||--o< REVIEW >o--|| PRODUCT   ORDER ||--o< ORDER_ITEM >|--|| PRODUCT_VARIANT
USER ||--o| CART ||--o< CART_ITEM >|--|| PRODUCT_VARIANT
PRODUCT ||--o< PRODUCT_VARIANT      PRODUCT >o----< CATEGORY  (via PRODUCT_CATEGORY)
CATEGORY ||--o< CATEGORY (parent_id, self-referential tree)
ORDER ||--o< PAYMENT
```

### 11.2 The normalized DDL (PostgreSQL)

```sql
-- =========================================================================
-- E-COMMERCE SCHEMA — PostgreSQL 17/18. Money is stored as BIGINT cents (Section 14).
-- Surrogate BIGINT identity PKs; natural keys enforced with UNIQUE; FKs all indexed.
-- =========================================================================

-- Reusable updated_at trigger (Section 8.1)
CREATE FUNCTION set_updated_at() RETURNS trigger AS $$
BEGIN NEW.updated_at := now(); RETURN NEW; END; $$ LANGUAGE plpgsql;

-- ---- users -------------------------------------------------------------
CREATE TABLE users (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    public_id   UUID NOT NULL UNIQUE DEFAULT gen_random_uuid(),  -- safe to expose in URLs/APIs (Section 2.6)
    email       TEXT NOT NULL UNIQUE,                            -- natural key, enforced unique
    password_hash TEXT NOT NULL,
    full_name   TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TRIGGER trg_users_updated BEFORE UPDATE ON users
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- ---- addresses (1:N from users) ---------------------------------------
CREATE TABLE address (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id     BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,  -- addresses die with the user
    line1       TEXT NOT NULL,
    line2       TEXT,
    city        TEXT NOT NULL,
    region      TEXT,
    postal_code TEXT NOT NULL,
    country     CHAR(2) NOT NULL,                 -- ISO-3166-1 alpha-2
    is_default  BOOLEAN NOT NULL DEFAULT FALSE
);
CREATE INDEX idx_address_user ON address(user_id);            -- index the FK (Section 7.2)
-- Only ONE default address per user (partial unique index, Section 7.5):
CREATE UNIQUE INDEX uq_one_default_addr ON address(user_id) WHERE is_default;

-- ---- category (self-referential tree, adjacency list 8.3a) ------------
CREATE TABLE category (
    id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    parent_id BIGINT REFERENCES category(id) ON DELETE RESTRICT,  -- NULL = root; don't orphan children
    name      TEXT NOT NULL,
    slug      TEXT NOT NULL UNIQUE
);
CREATE INDEX idx_category_parent ON category(parent_id);

-- ---- product ----------------------------------------------------------
CREATE TABLE product (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    slug        TEXT NOT NULL UNIQUE,             -- URL key, natural & unique
    title       TEXT NOT NULL CHECK (length(trim(title)) > 0),
    description TEXT NOT NULL DEFAULT '',
    is_active   BOOLEAN NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TRIGGER trg_product_updated BEFORE UPDATE ON product
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- ---- product_category (M:N junction, 5.3) -----------------------------
CREATE TABLE product_category (
    product_id  BIGINT NOT NULL REFERENCES product(id)  ON DELETE CASCADE,
    category_id BIGINT NOT NULL REFERENCES category(id) ON DELETE CASCADE,
    PRIMARY KEY (product_id, category_id)         -- one link per pair
);
CREATE INDEX idx_prodcat_category ON product_category(category_id);  -- for "products in a category"

-- ---- product_variant (the actually-sellable unit; price & stock live here) ----
CREATE TABLE product_variant (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_id  BIGINT NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    sku         TEXT NOT NULL UNIQUE,             -- natural business key
    name        TEXT NOT NULL,                    -- e.g. 'Red / L'
    price_cents BIGINT NOT NULL CHECK (price_cents >= 0),
    stock       INT    NOT NULL DEFAULT 0 CHECK (stock >= 0),  -- never negative
    UNIQUE (product_id, name)
);
CREATE INDEX idx_variant_product ON product_variant(product_id);

-- ---- cart (one active cart per user) + cart_item ----------------------
CREATE TABLE cart (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id    BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX uq_cart_user ON cart(user_id);   -- 1:1 active cart per user
CREATE TABLE cart_item (
    cart_id    BIGINT NOT NULL REFERENCES cart(id) ON DELETE CASCADE,
    variant_id BIGINT NOT NULL REFERENCES product_variant(id) ON DELETE RESTRICT,
    qty        INT NOT NULL CHECK (qty > 0),
    PRIMARY KEY (cart_id, variant_id)             -- one line per variant per cart
);

-- ---- order + order_item + payment -------------------------------------
CREATE TYPE order_status AS ENUM ('pending','paid','shipped','delivered','cancelled');

CREATE TABLE "order" (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    public_id       UUID NOT NULL UNIQUE DEFAULT gen_random_uuid(),
    user_id         BIGINT NOT NULL REFERENCES users(id) ON DELETE RESTRICT,  -- keep order history
    status          order_status NOT NULL DEFAULT 'pending',
    ship_address_id BIGINT NOT NULL REFERENCES address(id) ON DELETE RESTRICT,
    bill_address_id BIGINT NOT NULL REFERENCES address(id) ON DELETE RESTRICT,
    -- total is derived from items; store it as a snapshot for fast reads & immutable history:
    total_cents     BIGINT NOT NULL DEFAULT 0 CHECK (total_cents >= 0),
    placed_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_order_user_placed ON "order"(user_id, placed_at DESC);  -- "a user's orders, newest first"
CREATE INDEX idx_order_pending ON "order"(placed_at) WHERE status = 'pending';  -- the fulfillment queue

CREATE TABLE order_item (
    order_id          BIGINT NOT NULL REFERENCES "order"(id) ON DELETE CASCADE,  -- part of the order
    variant_id        BIGINT NOT NULL REFERENCES product_variant(id) ON DELETE RESTRICT, -- keep what was bought
    qty               INT    NOT NULL CHECK (qty > 0),
    -- SNAPSHOT the price & name at purchase time — a historical fact, NOT a 3NF violation (6.8):
    unit_price_cents  BIGINT NOT NULL CHECK (unit_price_cents >= 0),
    variant_name      TEXT   NOT NULL,
    line_total_cents  BIGINT GENERATED ALWAYS AS (qty * unit_price_cents) STORED, -- always correct (3.6)
    PRIMARY KEY (order_id, variant_id)
);
CREATE INDEX idx_orderitem_variant ON order_item(variant_id);

CREATE TABLE payment (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id        BIGINT NOT NULL REFERENCES "order"(id) ON DELETE CASCADE,
    amount_cents    BIGINT NOT NULL CHECK (amount_cents > 0),
    status          TEXT NOT NULL DEFAULT 'captured' CHECK (status IN ('captured','failed','refunded')),
    idempotency_key TEXT NOT NULL UNIQUE,         -- prevents double-charge on retries (10.4)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_payment_order ON payment(order_id);

-- ---- review (1 review per user per product) ---------------------------
CREATE TABLE review (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id    BIGINT NOT NULL REFERENCES users(id)   ON DELETE CASCADE,
    product_id BIGINT NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    rating     SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    body       TEXT NOT NULL DEFAULT '',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, product_id)                  -- a user reviews a product at most once
);
CREATE INDEX idx_review_product ON review(product_id);  -- "all reviews for a product"
```

Every design idea from earlier appears here: surrogate PKs with public UUIDs (2.6), natural keys as `UNIQUE` (2.2), every FK indexed (7.2), composite/partial indexes matched to queries (7.4–7.5), a junction with composite PK (5.3), an associative entity with snapshotted price (5.4 + 6.8), generated column (3.6), `CHECK`/`NOT NULL` invariants (3.4), an enum status (8.5/8.6), and idempotency keys (10.4).

### 11.3 Consuming it from Node.js — NestJS + Prisma

First, the **Prisma schema** that maps to (a subset of) the DDL above. Prisma is introspection-friendly: you can `prisma db pull` from the existing DB, but here we write it explicitly so the mapping is clear.

```prisma
// prisma/schema.prisma  — see PRISMA_ORM_GUIDE.md for full Prisma depth.
generator client { provider = "prisma-client-js" }
datasource db { provider = "postgresql"; url = env("DATABASE_URL") }

model User {
  id           BigInt    @id @default(autoincrement())
  publicId     String    @unique @default(uuid()) @map("public_id") @db.Uuid
  email        String    @unique
  passwordHash String    @map("password_hash")
  fullName     String    @map("full_name")
  orders       Order[]                                  // 1:N relation (the FK lives on Order)
  createdAt    DateTime  @default(now()) @map("created_at")
  updatedAt    DateTime  @updatedAt @map("updated_at")
  @@map("users")                                        // map model -> actual table name
}

model ProductVariant {
  id         BigInt      @id @default(autoincrement())
  sku        String      @unique
  name       String
  priceCents BigInt      @map("price_cents")
  stock      Int
  orderItems OrderItem[]
  @@map("product_variant")
}

model Order {
  id            BigInt      @id @default(autoincrement())
  publicId      String      @unique @default(uuid()) @map("public_id") @db.Uuid
  userId        BigInt      @map("user_id")
  user          User        @relation(fields: [userId], references: [id])  // FK on the many side (5.2)
  status        String      @default("pending")
  shipAddressId BigInt      @map("ship_address_id")
  billAddressId BigInt      @map("bill_address_id")
  totalCents    BigInt      @default(0) @map("total_cents")
  placedAt      DateTime    @default(now()) @map("placed_at")
  items         OrderItem[]                            // 1:N to order items
  @@index([userId, placedAt(sort: Desc)])              // mirrors idx_order_user_placed
  @@map("order")
}

model OrderItem {
  orderId        BigInt         @map("order_id")
  order          Order          @relation(fields: [orderId], references: [id], onDelete: Cascade)
  variantId      BigInt         @map("variant_id")
  variant        ProductVariant @relation(fields: [variantId], references: [id])
  qty            Int
  unitPriceCents BigInt         @map("unit_price_cents")
  variantName    String         @map("variant_name")
  @@id([orderId, variantId])                           // composite PK (5.3)
  @@map("order_item")
}
```

Now a **NestJS service** that creates an order with items **atomically in a transaction**, decrements stock safely, and snapshots prices. This is the canonical "aggregate written as one unit" (10.1).

```typescript
// orders.service.ts  (NestJS) — see NESTJS_GUIDE.md for module/DI setup.
import { Injectable, BadRequestException } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

interface NewItem { variantId: bigint; qty: number; }

@Injectable()
export class OrdersService {
  async createOrder(
    userId: bigint, shipAddressId: bigint, billAddressId: bigint, items: NewItem[],
  ) {
    // $transaction runs everything atomically: if ANY step throws, the whole order rolls back.
    return prisma.$transaction(async (tx) => {
      let total = 0n;

      // 1) Create the order shell first so order_items have an order_id to reference.
      const order = await tx.order.create({
        data: { userId, shipAddressId, billAddressId, status: 'pending' },
      });

      // 2) For each line: atomically decrement stock (the conditional-UPDATE race fix, 10.2),
      //    then snapshot price/name onto the order_item.
      for (const it of items) {
        // updateMany with a stock>=qty guard returns count=0 if not enough stock — atomic check+decrement.
        const dec = await tx.productVariant.updateMany({
          where: { id: it.variantId, stock: { gte: it.qty } },
          data:  { stock: { decrement: it.qty } },
        });
        if (dec.count === 0) {
          // Throwing here rolls back the transaction (no partial order is left behind).
          throw new BadRequestException(`variant ${it.variantId} out of stock`);
        }

        const v = await tx.productVariant.findUniqueOrThrow({ where: { id: it.variantId } });
        await tx.orderItem.create({
          data: {
            orderId: order.id, variantId: v.id, qty: it.qty,
            unitPriceCents: v.priceCents,    // SNAPSHOT current price (6.8)
            variantName: v.name,
          },
        });
        total += v.priceCents * BigInt(it.qty);
      }

      // 3) Persist the computed total on the order header (denormalized snapshot for fast reads).
      return tx.order.update({ where: { id: order.id }, data: { totalCents: total } });
    });
  }

  // Query one order WITH its items and the user, in a single round-trip via Prisma `include`.
  async getOrder(orderId: bigint) {
    return prisma.order.findUniqueOrThrow({
      where: { id: orderId },
      include: { items: { include: { variant: true } }, user: true },
    });
  }

  // List a user's orders, newest first — uses the (user_id, placed_at DESC) index.
  async listUserOrders(userId: bigint) {
    return prisma.order.findMany({
      where: { userId },
      orderBy: { placedAt: 'desc' },
      take: 50,
    });
  }
}
```

If you prefer the **raw `pg` driver** (no ORM), the same create-order transaction is explicit SQL — useful to see what Prisma generates:

```typescript
// orders.pg.ts — using node-postgres (`pg`) directly. See NODEJS_GUIDE.md / POSTGRESQL_GUIDE.md §21.
import { Pool } from 'pg';
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

export async function createOrderRaw(userId: number, ship: number, bill: number,
                                     items: { variantId: number; qty: number }[]) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');                                    // start transaction
    const { rows: [order] } = await client.query(
      `INSERT INTO "order"(user_id, ship_address_id, bill_address_id)
       VALUES ($1,$2,$3) RETURNING id`, [userId, ship, bill]);

    let total = 0;
    for (const it of items) {
      // Atomic check-and-decrement; RETURNING tells us if it succeeded.
      const dec = await client.query(
        `UPDATE product_variant SET stock = stock - $2
         WHERE id = $1 AND stock >= $2 RETURNING price_cents, name`,
        [it.variantId, it.qty]);
      if (dec.rowCount === 0) throw new Error(`variant ${it.variantId} out of stock`);
      const { price_cents, name } = dec.rows[0];
      await client.query(
        `INSERT INTO order_item(order_id, variant_id, qty, unit_price_cents, variant_name)
         VALUES ($1,$2,$3,$4,$5)`,
        [order.id, it.variantId, it.qty, price_cents, name]);
      total += Number(price_cents) * it.qty;
    }
    await client.query(`UPDATE "order" SET total_cents = $2 WHERE id = $1`, [order.id, total]);
    await client.query('COMMIT');                                   // all-or-nothing
    return order.id;
  } catch (e) {
    await client.query('ROLLBACK');                                 // undo everything on any error
    throw e;
  } finally {
    client.release();                                               // return connection to the pool
  }
}
```

### 11.4 Consuming it from Go — Gin + pgx

The same operations in Go using **pgx** (the modern Postgres driver). See `GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md` for the Gin web layer and `POSTGRESQL_GUIDE.md` §21 for pgx setup.

```go
// orders.go — Gin handlers backed by pgx v5. Transaction = one consistent aggregate write.
package main

import (
	"context"
	"errors"
	"net/http"

	"github.com/gin-gonic/gin"
	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"
)

type NewItem struct {
	VariantID int64 `json:"variant_id"`
	Qty       int   `json:"qty"`
}
type CreateOrderReq struct {
	UserID        int64     `json:"user_id"`
	ShipAddressID int64     `json:"ship_address_id"`
	BillAddressID int64     `json:"bill_address_id"`
	Items         []NewItem `json:"items"`
}

// createOrder writes the order + items + stock decrement atomically inside a single tx.
func createOrder(pool *pgxpool.Pool) gin.HandlerFunc {
	return func(c *gin.Context) {
		var req CreateOrderReq
		if err := c.ShouldBindJSON(&req); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}
		ctx := c.Request.Context()

		// Begin a transaction; defer Rollback is a no-op after a successful Commit.
		tx, err := pool.Begin(ctx)
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		defer tx.Rollback(ctx) // safety net: rolls back if we return early on any error

		// 1) Insert the order header, get its id.
		var orderID int64
		err = tx.QueryRow(ctx,
			`INSERT INTO "order"(user_id, ship_address_id, bill_address_id)
			 VALUES ($1,$2,$3) RETURNING id`,
			req.UserID, req.ShipAddressID, req.BillAddressID).Scan(&orderID)
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}

		// 2) Each line: atomic stock check+decrement, then snapshot price/name.
		var total int64
		for _, it := range req.Items {
			var price int64
			var name string
			err = tx.QueryRow(ctx,
				`UPDATE product_variant SET stock = stock - $2
				 WHERE id = $1 AND stock >= $2
				 RETURNING price_cents, name`, it.VariantID, it.Qty).Scan(&price, &name)
			if errors.Is(err, pgx.ErrNoRows) { // 0 rows ⇒ insufficient stock
				c.JSON(http.StatusConflict, gin.H{"error": "out of stock", "variant": it.VariantID})
				return // defer Rollback undoes the partial order
			} else if err != nil {
				c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
				return
			}
			if _, err = tx.Exec(ctx,
				`INSERT INTO order_item(order_id, variant_id, qty, unit_price_cents, variant_name)
				 VALUES ($1,$2,$3,$4,$5)`,
				orderID, it.VariantID, it.Qty, price, name); err != nil {
				c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
				return
			}
			total += price * int64(it.Qty)
		}

		// 3) Store the snapshot total and commit.
		if _, err = tx.Exec(ctx, `UPDATE "order" SET total_cents = $2 WHERE id = $1`, orderID, total); err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		if err = tx.Commit(ctx); err != nil { // make it all durable, or nothing
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		c.JSON(http.StatusCreated, gin.H{"order_id": orderID, "total_cents": total})
	}
}

// getOrder returns an order with its line items (one query for the header, one for the items).
type OrderItemDTO struct {
	VariantID      int64  `json:"variant_id"`
	VariantName    string `json:"variant_name"`
	Qty            int    `json:"qty"`
	UnitPriceCents int64  `json:"unit_price_cents"`
}
func getOrder(pool *pgxpool.Pool) gin.HandlerFunc {
	return func(c *gin.Context) {
		ctx := c.Request.Context()
		id := c.Param("id")

		var userID, total int64
		var status string
		err := pool.QueryRow(ctx,
			`SELECT user_id, status, total_cents FROM "order" WHERE id = $1`, id).
			Scan(&userID, &status, &total)
		if errors.Is(err, pgx.ErrNoRows) {
			c.JSON(http.StatusNotFound, gin.H{"error": "order not found"})
			return
		} else if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}

		// Fetch the items; the (order_id) PK prefix makes this an index lookup.
		rows, err := pool.Query(ctx,
			`SELECT variant_id, variant_name, qty, unit_price_cents
			 FROM order_item WHERE order_id = $1 ORDER BY variant_id`, id)
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		defer rows.Close()

		var items []OrderItemDTO
		for rows.Next() {
			var it OrderItemDTO
			if err := rows.Scan(&it.VariantID, &it.VariantName, &it.Qty, &it.UnitPriceCents); err != nil {
				c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
				return
			}
			items = append(items, it)
		}
		c.JSON(http.StatusOK, gin.H{
			"id": id, "user_id": userID, "status": status, "total_cents": total, "items": items,
		})
	}
}

// listUserOrders uses the (user_id, placed_at DESC) index for cheap, sorted retrieval.
func listUserOrders(pool *pgxpool.Pool) gin.HandlerFunc {
	return func(c *gin.Context) {
		ctx := c.Request.Context()
		rows, err := pool.Query(ctx,
			`SELECT id, status, total_cents, placed_at
			 FROM "order" WHERE user_id = $1
			 ORDER BY placed_at DESC LIMIT 50`, c.Param("userId"))
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		defer rows.Close()
		// pgx.CollectRows + RowToStructByName is the ergonomic way to scan many rows (omitted for brevity).
		c.JSON(http.StatusOK, gin.H{"ok": true})
	}
}
```

> **sqlc note:** Many Go teams write the SQL *first* and generate type-safe Go with **sqlc** (you author `queries.sql`, sqlc emits structs + methods). This pairs beautifully with a well-designed schema because the generated types mirror your tables exactly. **GORM** (the popular ORM) is also viable — define structs with `gorm` tags — but for transaction-heavy commerce code, explicit pgx/sqlc keeps the SQL (and the locking) visible, which is usually what you want. See `GO_ENT_ORM_GUIDE.md` for the Ent alternative.

### 11.5 What this example demonstrates

Notice that the application code in *both* languages is short and obvious — because the schema did the hard work: the FK constraints prevent orphans, the `CHECK`s prevent negative stock/prices, the composite PKs prevent duplicate lines, the unique idempotency key prevents double-charges, and the indexes make the queries fast. **A good schema makes correct application code the path of least resistance.**

---

## 12. Migrations & Schema Evolution

**[I/A]**

A schema is never "done" — it evolves with the product. **Migrations** are versioned, ordered, repeatable scripts that take the schema from one state to the next. The design discipline is making changes **safely**, ideally with **zero downtime** while old and new application code run simultaneously during a deploy.

### 12.1 What migrations are and why they're versioned

A migration is a small, ordered unit of DDL (and sometimes DML) checked into source control alongside your code. Tools track which migrations have run (in a metadata table) so each environment applies exactly the pending ones, in order, exactly once. This gives you a **reproducible, auditable history** of how the schema reached its current shape — the database equivalent of git for your tables.

Common tools in 2026:

| Tool | Ecosystem | Style |
|---|---|---|
| **Prisma Migrate** | Node/TS | Declarative: edit `schema.prisma`, generate SQL migration, apply |
| **golang-migrate** | Go (any lang) | Imperative: numbered `up.sql`/`down.sql` pairs |
| **Flyway** | JVM (any DB) | Versioned `V1__*.sql` SQL scripts; very widely used |
| **Liquibase** | JVM | XML/YAML/SQL changesets, rich rollback |
| **Atlas** | Go/any | Declarative desired-state diffing (HCL/SQL) |

```sql
-- golang-migrate style: 000004_add_review.up.sql
CREATE TABLE review (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    rating SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    UNIQUE (user_id, product_id)
);
-- 000004_add_review.down.sql  (the reverse, so you can roll back)
DROP TABLE review;
```

```bash
# Prisma: edit schema.prisma, then generate + apply a migration
npx prisma migrate dev --name add_review     # dev: create + apply + regenerate client
npx prisma migrate deploy                     # prod: apply pending migrations only

# golang-migrate CLI
migrate -path ./migrations -database "$DATABASE_URL" up      # apply all pending
migrate -path ./migrations -database "$DATABASE_URL" down 1  # roll back one
```

### 12.2 Backward-compatible changes — the golden rule

During a rolling deploy, **two versions of your app run at once** against **one** database. So a migration must not break the *currently running* (old) code. **Additive changes are safe; destructive changes are not.**

- **Safe (backward-compatible):** add a *nullable* column (or one with a default), add a new table, add an index `CONCURRENTLY`, add a *non-validated* constraint then validate later.
- **Dangerous (breaks old code or locks the table):** drop/rename a column or table, change a type, add a `NOT NULL` column without a default, add a constraint that rewrites/locks the table.

```sql
-- SAFE: nullable add — old code ignores it, new code populates it.
ALTER TABLE users ADD COLUMN phone TEXT;

-- DANGEROUS as one step: renaming breaks old code that still selects "full_name".
-- ALTER TABLE users RENAME COLUMN full_name TO name;   -- DON'T do this in a live rolling deploy
```

### 12.3 Zero-downtime migrations — the expand/contract pattern **[A]**

The professional technique for *breaking* changes is **expand/contract** (a.k.a. parallel change): never change in place — add the new shape, migrate to it across multiple deploys, then remove the old shape once nothing uses it.

Renaming `users.full_name` to `users.name` with zero downtime:

```sql
-- DEPLOY 1 (EXPAND): add the new column; keep BOTH in sync.
ALTER TABLE users ADD COLUMN name TEXT;                 -- nullable add = safe
UPDATE users SET name = full_name WHERE name IS NULL;   -- backfill (batch large tables!)
-- App code now WRITES BOTH columns (or a trigger copies full_name -> name) and still READS full_name.

-- DEPLOY 2 (MIGRATE READS): app code now READS `name`, still WRITES both. Verify all instances updated.

-- DEPLOY 3 (CONTRACT): nothing reads or writes full_name anymore — now it's safe to drop.
ALTER TABLE users DROP COLUMN full_name;
```

Other zero-downtime essentials:
- **Create indexes without locking writes:** `CREATE INDEX CONCURRENTLY` (Postgres) — never bare `CREATE INDEX` on a big live table.
- **Add `NOT NULL` in two steps:** add the column nullable + backfill; add a `CHECK (col IS NOT NULL) NOT VALID`; `VALIDATE CONSTRAINT` (scans without a long exclusive lock); then optionally convert to a real `NOT NULL`.
- **Add FKs without a long lock:** `ADD CONSTRAINT ... NOT VALID`, then `VALIDATE CONSTRAINT` separately.
- **Backfill in batches** (e.g., 10k rows per transaction) so you don't lock millions of rows or bloat WAL.

```sql
-- Add a validated FK to a large table without holding a heavy lock for the full scan:
ALTER TABLE "order" ADD CONSTRAINT fk_order_user
  FOREIGN KEY (user_id) REFERENCES users(id) NOT VALID;   -- fast: only checks NEW rows
ALTER TABLE "order" VALIDATE CONSTRAINT fk_order_user;    -- separate, lighter-lock full check
```

> See `POSTGRESQL_GUIDE.md` for the locking specifics of each DDL operation; lock behavior differs sharply between Postgres versions and other engines (MySQL's online-DDL rules differ).

---

## 13. Performance & Scaling Design

**[A]**

When a single well-indexed table or instance is no longer enough, you reach for *structural* scaling techniques. The ordering matters: **exhaust indexing and query tuning first**, then caching, then read replicas, then partitioning, and only finally sharding. Each step adds operational complexity, so adopt the *least* that solves your measured problem.

### 13.1 Partitioning — splitting one big table into pieces

**What & why:** **Partitioning** physically divides one logical table into multiple sub-tables ("partitions") by a key, while queries still target the single parent. The wins: the planner can **prune** irrelevant partitions (scan only this month's data), `VACUUM`/index maintenance works per-partition, and you can drop old data instantly by dropping a partition (vs a slow mass `DELETE`). Use it when a table is very large (hundreds of millions of rows) *and* queries/retention naturally align to a key (usually time).

Three partitioning strategies:
- **Range** — by a continuous value, classically a date (`events` by month). Most common.
- **List** — by a discrete value (`orders` by region/country).
- **Hash** — by a hash of the key, to spread evenly when there's no natural range (distribute load).

```sql
-- RANGE partition events by month. Queries with a date filter scan only matching partitions (pruning).
CREATE TABLE event (
    id        BIGINT GENERATED ALWAYS AS IDENTITY,
    occurred_at TIMESTAMPTZ NOT NULL,
    payload   JSONB NOT NULL,
    PRIMARY KEY (id, occurred_at)        -- partition key must be part of the PK
) PARTITION BY RANGE (occurred_at);

CREATE TABLE event_2026_06 PARTITION OF event
  FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');   -- one partition per month
CREATE TABLE event_2026_07 PARTITION OF event
  FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');
-- Retention: DROP TABLE event_2026_06;  -- instant vs DELETE ... WHERE occurred_at < ...
```

> Partitioning constrains design: the **partition key must be in every unique/primary key**, which can force composite keys. See `POSTGRESQL_GUIDE.md` §16 for partition-pruning verification.

### 13.2 Read replicas — scaling reads

**What & why:** A **read replica** is a streaming copy of the primary that serves *read-only* queries, offloading the primary. It scales read-heavy workloads (reporting, dashboards, most web traffic) horizontally. The design caveat is **replication lag**: a replica is milliseconds-to-seconds behind, so a read right after a write may not see it ("read-your-writes" violation). Route reads that tolerate staleness to replicas; route reads that must see the just-written value (and all writes) to the primary.

```text
            writes + fresh reads          stale-tolerant reads
   App ───────────────────────► PRIMARY ═══(stream)═══► REPLICA ◄─────── App (reports)
```

### 13.3 Sharding — scaling writes **[A]**

**What & why:** When *write* volume or data size exceeds one machine, **sharding** splits data **across multiple independent databases** by a **shard key** (e.g., `tenant_id`, `user_id`). Each shard holds a disjoint subset; the app (or a proxy like Citus/Vitess) routes each query to the right shard. This is the heaviest tool — it sacrifices cross-shard joins, complicates transactions (no easy multi-shard atomicity), and makes the shard-key choice nearly irreversible. **Reach for it last**, when replicas + partitioning are exhausted.

Design consequences of sharding:
- **Choose the shard key to keep related data co-located** (everything for one tenant on one shard) and to **avoid hotspots** (don't shard by a key where one value gets 90% of traffic).
- **Cross-shard queries become application-level scatter/gather** — design so the common queries stay within one shard.
- **Globally unique IDs** matter more (UUIDv7/ULID, Section 2.6) since a central sequence no longer works across shards.

> Multi-tenant SaaS often *naturally* shards by `tenant_id` — the shared-schema model (8.7) is "logical sharding" you can later promote to physical shards. Plan the tenant_id everywhere early and physical sharding stays an option.

### 13.4 Caching and connection pooling — design adjacents

Two scaling levers that touch design without being part of the schema:

- **Caching:** A cache (Redis, or an in-process layer) absorbs repeated reads so they never hit the database. The design implication is **invalidation**: stale caches and the DB disagree, so you must decide *when* to invalidate (on write, by TTL, or via change-data-capture). Cache *derived* read models, not the source of truth, and keep the DB authoritative. Predictable cache keys come from stable surrogate IDs (Section 2.6) — another reason to expose a `public_id`. See `REDIS_GUIDE.md`.
- **Connection pooling:** Every DB connection costs memory on the server; thousands of app instances each opening connections will exhaust the server. A **pooler** (PgBouncer for Postgres, or the driver's built-in pool — `pgxpool` in Go, the `pg` `Pool`/Prisma's pool in Node) multiplexes many app requests over a small number of real connections. Design implication: **keep transactions short** (a long-running transaction holds a pooled connection and, in Postgres, blocks `VACUUM`), and prefer `transaction`-mode pooling for high concurrency. This is *why* the §11 transactions do the minimum work and commit promptly.

### 13.5 Scaling decision ladder

```text
Slow / overloaded? Climb this ladder in order — stop at the first rung that solves it:
  1. Add/fix INDEXES; rewrite the query (EXPLAIN ANALYZE).          (cheapest, do first)
  2. CACHE hot reads (Redis / read models).
  3. READ REPLICAS for stale-tolerant read traffic.
  4. PARTITION the giant table (by time/region) for pruning + cheap retention.
  5. SHARD by a well-chosen key — only when writes/size exceed one box.  (last, hardest)
```

---

## 14. Anti-Patterns & Gotchas

**[All]**

A catalog of the mistakes that bite hardest. Most are violations of a single principle: *let the database enforce truth, and store each fact once, as the thing it is.*

| Anti-pattern | Why it's bad | Do instead |
|---|---|---|
| **No foreign keys** ("we'll enforce it in the app") | Orphans and dangling references creep in; no app is perfect across every code path, migration, and manual fix | Declare FKs; the DB makes orphans impossible (Section 2.4) |
| **Missing index on a FK** | Slow "children of parent" queries; parent deletes/updates do full child scans and take heavy locks | Index every FK column (Section 7.2) |
| **God table** (one giant table with 80 columns and a `type` flag) | Lots of NULLs, unclear semantics, lock contention, every read touches everything | Split by entity; use subtype tables or proper relations |
| **Storing CSV/lists in a column** (`tags = 'a,b,c'`) | Can't index, constrain, join, or query a member; violates 1NF | A child table or junction (Section 6.3); or a real array/JSONB *only* if you never need relational integrity on the members |
| **Floats for money** (`price FLOAT`) | Binary FP can't represent `0.10`; rounding errors accumulate; `0.1+0.2 != 0.3` | `NUMERIC(p,s)` or integer cents (Section 3.1) |
| **NULL misuse** | NULL-as-"empty"/"zero" triggers three-valued-logic bugs; `WHERE x <> 'a'` drops NULLs silently | Use `NOT NULL` + sensible defaults; reserve NULL for genuinely unknown (Section 3.2) |
| **Premature denormalization** | Redundant copies drift out of sync; complex writes; bugs | Normalize to 3NF first; denormalize only with measured proof (Section 6.8) |
| **EAV for everything** | Loses types, constraints, FKs, sane queries | Real columns; or `JSONB` for truly dynamic fields (Section 8.10) |
| **Natural key as PK that changes** | A changed PK cascades through every FK | Surrogate PK + UNIQUE on the natural key (Section 2.2) |
| **`SELECT *` in app code coupling to column order** | Adding/reordering columns breaks brittle code | Select named columns |
| **No `updated_at`/audit columns** | Can't debug, sort, or sync; no traceability | Add audit columns + trigger (Section 8.1) |
| **Unbounded `VARCHAR`/no length sanity** | Lets garbage in; one huge value can bloat rows | `TEXT` + a length `CHECK` where a real limit exists |
| **Timestamps without zone (`TIMESTAMP`)** | DST/ambiguity bugs; can't compare across zones | `TIMESTAMPTZ` for instants (Section 3.1) |
| **Enum as `CHECK IN(...)` that changes weekly** | Every value change is a migration | Lookup table with FK (Section 8.5) |
| **Wide composite PK propagated everywhere** | Fat FKs in many child tables, slow joins | Surrogate PK on the entity; composite only on junctions |

### 14.1 A few subtle gotchas worth internalizing

- **A `CHECK` constraint passes on NULL.** `CHECK (price > 0)` does *not* reject a NULL price (it's UNKNOWN, which passes). Add `NOT NULL`.
- **`UNIQUE` allows multiple NULLs** by default (Section 2.5). Use `NULLS NOT DISTINCT` (PG15+) or a partial index if you need at-most-one NULL.
- **`COUNT(column)` ignores NULLs; `COUNT(*)` doesn't.** They answer different questions.
- **Booleans beat status-as-int.** `is_active BOOLEAN` is clearer than `status INT` where `1` means active (which everyone forgets).
- **Don't reuse a column for two meanings** ("`note` holds the cancellation reason *if* cancelled"). Add a dedicated column; overloaded columns rot.
- **Beware `ON DELETE CASCADE` chains** — one delete can silently erase thousands of rows across many tables (Section 9.2).

---

## 15. Study Path & Build-to-Learn Projects

**[All levels]**

### Suggested learning order

1. **Week 1 — Foundations (Beginner):** Internalize the relational model and vocabulary (§1). Master keys: PK/FK/unique, natural vs surrogate, and surrogate-key types (§2). Learn to pick data types, understand NULL three-valued logic cold, and write constraints (§3). Practice by modeling and creating a handful of related tables by hand.
2. **Week 2 — Modeling (Beginner→Intermediate):** Do ER modeling on paper for three different domains (§4). Implement 1:1, 1:N, and M:N relationships until the patterns are automatic (§5). For each table you create, ask "what's the key, what are the FKs, are they indexed?"
3. **Week 3 — Normalization & indexing (Intermediate):** Work the normalization examples by hand: take a messy spreadsheet and drive it to 3NF, naming the anomaly each step removes (§6). Learn which columns to index and why FK indexes matter; read `EXPLAIN` in `POSTGRESQL_GUIDE.md` §12 (§7).
4. **Week 4 — Patterns (Intermediate→Advanced):** Implement each design pattern at least once: audit columns, soft deletes, a tree (all four approaches), a state machine, a lookup table, multi-tenancy with RLS, a history table (§8). Practice choosing referential actions deliberately (§9).
5. **Week 5 — Concurrency & the worked example (Advanced):** Study how isolation/locking shape schema; implement optimistic locking and idempotency keys (§10). Then build the full e-commerce schema and consume it from both a Node and a Go backend, writing the create-order transaction yourself (§11).
6. **Week 6 — Evolution & scale (Advanced):** Practice migrations with a real tool; perform an expand/contract rename with zero downtime (§12). Set up a partitioned table and a read replica; reason about when sharding is justified (§13). Audit a real schema against the anti-pattern table (§14).

### Build-to-learn projects (design *and* build)

- **Blog / CMS (start here):** Model `users`, `posts`, `comments`, `tags` (M:N), `categories` (tree). Add audit columns + an `updated_at` trigger, soft deletes on posts, a `comment_count` denormalized counter kept correct by a trigger, and full-text search. Consume it from Node (Prisma) or Go (pgx). Goal: cement 1:N, M:N, trees, denormalization, and indexing.
- **Booking / reservation system:** Model `resources` (rooms), `bookings`, `users`. The headline challenge: **prevent double-booking** with an exclusion constraint over a time range (§9.3). Add a booking state machine (pending → confirmed → cancelled, §8.6) and optimistic locking on edits. Goal: cement integrity constraints and concurrency.
- **Multi-tenant SaaS schema:** Pick a domain (project management). Implement shared-schema multi-tenancy with `tenant_id` on every table, names unique *within* a tenant, and **Row-Level Security** enforcing isolation (§8.7). Add idempotency keys to the write API. Verify a tenant cannot read another's rows even with a buggy missing `WHERE`. Goal: cement multi-tenancy, RLS, and defense-in-depth, and set up for future sharding by tenant.
- **E-commerce (the capstone):** Build §11 end to end — products, variants, categories (tree + M:N), carts, orders, order_items (with price snapshots), payments (idempotent), reviews. Add migrations with a real tool, then do a zero-downtime expand/contract change. Partition the `event`/audit-log table by month. Goal: integrate *everything*.

### Quick reference: the design checklist for every table

```text
[ ] Does it have a PRIMARY KEY? (surrogate BIGINT/UUID by default)
[ ] Are natural/business keys enforced with UNIQUE?
[ ] Is every FK declared, with a deliberate ON DELETE action?
[ ] Is every FK column INDEXED?
[ ] Is each column the right TYPE? (money = cents/NUMERIC, instant = TIMESTAMPTZ)
[ ] Is each column NOT NULL unless absence is genuinely meaningful?
[ ] Are business rules encoded as CHECK / UNIQUE / EXCLUDE constraints?
[ ] Is it in 3NF? Any redundancy is either justified (snapshot) or removed.
[ ] Are there audit columns (created_at/updated_at) where useful?
[ ] Do the indexes match the actual query patterns (filter/join/sort)?
[ ] Will this change safely under a rolling deploy (additive)?
```

> **The one sentence to remember:** *Put each fact in exactly one obvious place, give every table a key, let the database enforce every rule it can, and design for the queries you actually run.*

