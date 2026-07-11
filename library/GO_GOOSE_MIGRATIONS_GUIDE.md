# Goose (github.com/pressly/goose) — Database Migrations, Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Any Go developer who has *never run a database migration in their life* and wants to reach the point where they can own a production PostgreSQL schema safely — author SQL and Go migrations, embed them in a binary, run them programmatically, guard them with advisory locks, and evolve a live schema with zero downtime — all learnable **offline**, with no internet. This guide is deliberately **explain-first**: every concept leads with prose — *what it is, why it exists, when to reach for it, how to use it, the best practice, and the gotcha* — and only then shows heavily-commented code. Migrations are the one part of your stack that touches *durable state you cannot un-break*; a migration you ran but did not understand is a data-loss incident waiting for its moment. So the "why" matters at least as much as the "how." Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **goose v3.27+** (current in 2026) on **Go 1.25 / 1.26**, driving **PostgreSQL** through **pgx v5**, alongside **Ent v0.14+**, **Gin v1.10+**, **golang-jwt/jwt v5**, **Argon2id** (`golang.org/x/crypto/argon2`), and **Air** for hot-reload in development. goose has two overlapping APIs: the older **global functions** (`goose.Up(db, dir)`, `goose.SetDialect(...)`, `goose.AddMigrationContext(...)`) that carry package-level state, and the modern **`goose.NewProvider(...)`** API (no global state, thread-safe, testable). Everywhere the two diverge — and it matters — the modern API is marked with **⚡**. The author is on **Windows 11**, so shell commands are shown for PowerShell and POSIX where they differ. Always cross-check the current signatures at **pkg.go.dev/github.com/pressly/goose/v3** before shipping; migrations are not a place to guess.
>
> **This guide's place in the library:** goose owns the *shape* of your database — the DDL, the versioned evolution of tables, indexes, and constraints. It sits alongside several siblings you will want open: [Go ent ORM](GO_ENT_ORM_GUIDE.md) generates your type-safe data-access layer **on top of** the schema goose creates (§9 explains the exact division of labour); [sqlc + goose](GO_SQLC_GOOSE_GUIDE.md) is the type-safe-queries counterpart where you keep hand-written SQL; [PostgreSQL](POSTGRESQL_GUIDE.md) is the engine goose talks to and where DDL semantics (transactional DDL, lock levels) actually live; [Go Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) and [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md) are the application this guide's worked example wires migrations into; [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md) is the *theory* (normalization, keys, indexes) your migrations encode; and [Database Server Admin](DATABASE_SERVER_ADMIN_GUIDE.md) covers backups, roles, and the operational side of the database you migrate against. This guide assumes you can read Go and know what a `CREATE TABLE` is; it teaches goose.

---

## Table of Contents

1. [What Migrations Are — and Why AutoMigrate Is Not Enough](#1-what-migrations-are--and-why-automigrate-is-not-enough) **[B]**
2. [Install goose & Your First Migration](#2-install-goose--your-first-migration) **[B]**
3. [SQL Migrations In Depth](#3-sql-migrations-in-depth) **[B/I]**
4. [The Full Command Reference](#4-the-full-command-reference) **[B/I]**
5. [Go Migrations: When SQL Isn't Enough](#5-go-migrations-when-sql-isnt-enough) **[I]**
6. [Embedding Migrations with embed.FS](#6-embedding-migrations-with-embedfs) **[I]**
7. [The Modern Programmatic API: NewProvider](#7-the-modern-programmatic-api-newprovider) **[I/A]**
8. [pgx Integration: One Pool for Migrations and App](#8-pgx-integration-one-pool-for-migrations-and-app) **[I/A]**
9. [Coexisting with Ent: Goose as the Single Source of Truth](#9-coexisting-with-ent-goose-as-the-single-source-of-truth) **[I/A]**
10. [Production Workflow — Banking-Grade Discipline](#10-production-workflow--banking-grade-discipline) **[A]**
11. [The Full Stack: Gin + Ent + pgx + JWT + Argon2 + Air](#11-the-full-stack-gin--ent--pgx--jwt--argon2--air) **[I/A]**
12. [Gotchas & Best Practices](#12-gotchas--best-practices) **[A]**
13. [Study Path & Build-to-Learn Projects](#13-study-path--build-to-learn-projects)

---

## 1. What Migrations Are — and Why AutoMigrate Is Not Enough

### 1.1 The core problem, stated plainly **[B]**

Your application's source code lives in Git. When a teammate clones the repo, checks out a branch, or a CI runner spins up, they get *exactly* the code that produced any given commit. You can `git bisect` a bug to a single line. You can roll back a deploy by checking out the previous tag. The code is **reproducible, versioned, and reviewable** — that is the entire point of source control.

Your **database schema** — the tables, columns, indexes, constraints, functions, and triggers — is *also* code. It is arguably the most important code you have, because unlike your Go binary, **the database holds state you cannot regenerate**. If you deploy a bad binary you roll it back; if you run a bad `ALTER TABLE ... DROP COLUMN` you have destroyed data that no rollback restores. And yet, without a migration tool, the schema lives *nowhere* that Git can see. It lives inside a running PostgreSQL server, mutated by whoever last typed `ALTER TABLE` into `psql`. Nobody can tell you what the schema was at commit `abc123`. Two developers' local databases drift apart. Staging doesn't match production. This divergence has a name — **schema drift** — and it is the source of an enormous fraction of "works on my machine" outages.

A **database migration tool** solves this by making the schema *a versioned, ordered sequence of change scripts stored in your repository.* Each script — a **migration** — describes one incremental change: "create the `users` table," "add a `deleted_at` column," "create an index on `sessions.token`." The scripts are numbered so their order is unambiguous. The tool records, *inside the database itself*, which migrations have already been applied. To bring any database — a fresh clone, CI, staging, production — to the current schema, you run the same command (`goose up`), and the tool applies exactly the migrations that database is missing, in order. The schema becomes reproducible from the repo, reviewable in a pull request, and diffable across time. That is the whole idea, and everything else in this guide is a consequence of it.

### 1.2 What a migration tool actually gives you **[B]**

Spelled out, a real migration tool provides five guarantees that hand-editing a database never can:

| Guarantee | What it means | Why it matters |
|---|---|---|
| **Reproducibility** | Running the migrations from empty produces the exact same schema everywhere. | Dev == CI == staging == prod. "Works on my machine" dies. |
| **Ordering** | Migrations apply in a defined, total order and never re-apply. | Migration 5 can rely on the table migration 3 created. No ambiguity. |
| **A recorded history** | The DB stores which versions are applied (goose uses a `goose_db_version` table). | The tool knows *exactly* what any database is missing and applies only that. |
| **Reviewability** | Each schema change is a file in a pull request. | A DBA or teammate reviews the `DROP COLUMN` *before* it touches production. |
| **Forward + backward paths** | Each migration has an "up" (apply) and, ideally, a "down" (undo). | You can reason about rollback; you have a tested escape hatch. |

Notice the word *incremental*. A migration is not "here is the full current schema." It is "here is the **delta** from the previous state to the next state." This is what lets a production database that is three versions behind catch up by applying exactly three deltas, without touching the data already there.

### 1.3 Forward and backward: "up" and "down" **[B]**

Every goose migration file has two halves, marked by annotations goose reads from SQL comments:

- **Up** — the change you want: `CREATE TABLE users (...)`. Applied when you run `goose up`.
- **Down** — the inverse that undoes it: `DROP TABLE users`. Applied when you run `goose down`.

The down half exists so that during development you can freely apply and un-apply a migration while you iterate, and so that — *in principle* — you have a defined rollback. We will spend real time in §10 on why "in principle" is doing heavy lifting there: in production you usually **roll forward** (write a new migration that fixes the problem) rather than **roll back** (run the down migration), because a down migration that drops a column silently destroys any data written to it since. But even so, writing and testing the down half is a discipline that forces you to think about reversibility, and it is invaluable in dev and CI. Write your downs. Just do not treat them as a magic prod undo button.

### 1.4 Why NOT `AutoMigrate` / `Schema.Create` in production **[B/I]**

This is the single most important argument in the chapter, so we will be blunt. Many ORMs offer an **automatic migration** feature: GORM's `db.AutoMigrate(&User{})`, Ent's `client.Schema.Create(ctx)`, Django's implicit sync, and so on. You define your models in code, call one function on startup, and the ORM inspects the current database, diffs it against your models, and issues whatever `CREATE`/`ALTER` statements it thinks will reconcile them. It feels magical in a tutorial. **It is a liability in production**, and here is precisely why:

1. **It is uncontrolled.** *You* did not decide the SQL — the ORM's diff engine did, at runtime, based on a heuristic. You do not know in advance whether it will issue a cheap `ADD COLUMN` or a catastrophic table rewrite that locks the table for ten minutes. On a banking table with 200 million rows, "the ORM decided" is not an acceptable answer to "why was the service down."

2. **It is unreviewable.** There is no artifact. No file in a PR. No line for a DBA to comment "this ALTER takes an ACCESS EXCLUSIVE lock, do it off-hours." The change exists only as an emergent behavior of the ORM's version, your struct tags, and the current DB state — three moving parts that produce different SQL in different combinations.

3. **It is destructive in ways you don't see.** Auto-migration engines are conservative about *adding* but wildly inconsistent about *removing*. Some will happily drop a column you renamed in code (because to the diff engine, "renamed" is indistinguishable from "old column deleted, new column added") — taking the data with it. Others refuse to drop anything, so your schema silently accumulates dead columns forever. Neither is what you want, and you cannot tell which you'll get without reading the ORM's source.

4. **It has no history and no rollback.** There is no recorded sequence of changes, so you cannot answer "what changed between last Tuesday and today," cannot bisect a schema regression, and cannot cleanly undo.

5. **It couples schema changes to deploys in the worst way.** If auto-migrate runs on app boot, then *every* instance that starts tries to migrate, concurrently — the split-brain problem we solve with advisory locks in §10 — and a schema change can only ship coupled to a code deploy, never independently, never gated, never in a controlled maintenance window.

The migration-tool philosophy is the opposite of all five: **you** write the exact SQL, it lives in a **file**, that file is **reviewed** in a PR, applied in a **defined order**, recorded in **history**, and run as a **deliberate, gated step** — not a side effect of a process starting. This is why every serious shop, and unconditionally every regulated one (banking, health, payments), uses explicit migrations. `AutoMigrate` is fine for a weekend prototype. It has no place in a system where losing a row is an incident.

> **The honest nuance (foreshadowing §9):** Ent *can* generate migration SQL for you (via Atlas) — it can diff your Ent schema against the DB and emit a migration *file* you then review and run with a tool. That versioned mode is legitimate. What is dangerous is the *runtime* `client.Schema.Create(ctx)` that applies changes on boot with no file and no review. In this guide, **goose owns the schema** and Ent is configured to *never* auto-migrate at runtime. §9 makes the boundary exact.

### 1.5 Migrations vs ORM auto-migration — the mental model **[I]**

Hold these two mental models side by side:

- **Auto-migration:** "The schema is a *projection* of my code models. The ORM keeps the DB in sync with my structs." The source of truth is the code models; the DDL is implicit and derived at runtime.
- **Explicit migrations (goose):** "The schema is *itself* source code — an ordered list of hand-authored, reviewed deltas. The DB is the sum of applied deltas." The source of truth is the migration files; the ORM's model definitions must be kept *consistent with* them (§9), but they do not *drive* the DDL.

goose is firmly in the second camp, and this guide teaches you to live there comfortably. The trade is a little more typing (you write the `ALTER TABLE` yourself) for a lot more control (you know exactly what runs, when, reviewed by whom). In production, that trade is not close.

---

## 2. Install goose & Your First Migration

### 2.1 What goose is, concretely **[B]**

goose is a single, dependency-light migration tool that comes in two forms from the same module (`github.com/pressly/goose/v3`):

- A **command-line tool** (`goose`) you install and run from your terminal: `goose up`, `goose status`, `goose create`. Great for local development, ad-hoc operations, and simple deploy scripts.
- A **Go library** you import into your own program to run migrations from code — either the older global functions (`goose.Up(db, "migrations")`) or the modern `goose.NewProvider(...)` (§7). This is how you embed migrations in your binary, run them from a `cmd/migrate` command, and integrate them into a real service.

Both forms read the same migration files and use the same `goose_db_version` tracking table, so you can create a migration with the CLI, apply it with the library in production, and check its status with the CLI again. They are interchangeable views of the same system.

goose supports many SQL dialects — **postgres, mysql, sqlite3, mssql, clickhouse, tidb, vertica, ydb** and more — because the *tracking* logic is dialect-agnostic and only a small amount of dialect-specific SQL (how to create the version table, how to query it) differs. This guide is PostgreSQL-throughout, since that is our stack, but the concepts port directly.

### 2.2 Installing the CLI **[B]**

The CLI installs like any Go tool, via `go install`, which compiles it and drops the binary into `$GOBIN` (usually `~/go/bin`, or `%USERPROFILE%\go\bin` on Windows). Make sure that directory is on your `PATH`.

```bash
# Install the goose CLI (the /cmd/goose subpackage is the binary; the module root is the library).
go install github.com/pressly/goose/v3/cmd/goose@latest

# Verify it's on PATH and see the version.
goose -version
```

> **⚡ Version note:** `@latest` pulls the newest tagged release (v3.27+ in 2026). For **reproducible builds** — CI, team consistency — pin an exact version instead: `go install github.com/pressly/goose/v3/cmd/goose@v3.27.0`. Even better, add goose to your project as a **tool dependency** so its version is recorded in `go.mod` and everyone gets the same one. In Go 1.24+ that is `go get -tool github.com/pressly/goose/v3/cmd/goose`, after which `go tool goose ...` runs the pinned version. Pinning matters: a migration tool that behaves subtly differently across developer machines is exactly the kind of drift migrations exist to eliminate.

> **⚡ Build tags for a lean binary:** goose's CLI, by default, links drivers for many databases, which bloats the binary and its dependency tree. If you only use PostgreSQL you can build a slimmer CLI with build tags: `go install -tags='no_mysql no_sqlite3 no_mssql no_clickhouse no_vertica no_ydb no_libsql' github.com/pressly/goose/v3/cmd/goose@latest`. This is optional but nice for CI images. It has no effect on the *library* API you import.

Confirm the install:

```text
$ goose -version
goose version: v3.27.0
```

### 2.3 The DSN — how goose reaches the database **[B]**

To do anything, goose needs to connect to a database. The CLI takes a **driver name** and a **connection string** (DSN — Data Source Name). For PostgreSQL the driver word is `postgres` and the DSN is a standard libpq/pgx URL or key-value string:

```bash
# URL form (most common). Note ?sslmode=... — required against real servers.
postgres://appuser:secret@localhost:5432/appdb?sslmode=disable

# Key-value form (also accepted by the pg drivers).
host=localhost port=5432 user=appuser password=secret dbname=appdb sslmode=disable
```

**Never hardcode a DSN with a real password into a script committed to Git.** Read it from an environment variable. goose respects a few env vars natively — most usefully `GOOSE_DBSTRING` (the DSN), `GOOSE_DRIVER` (the dialect), and `GOOSE_MIGRATION_DIR` (the migrations folder) — so you can set them once and drop the positional args:

```bash
# POSIX / Git Bash
export GOOSE_DRIVER=postgres
export GOOSE_DBSTRING='postgres://appuser:secret@localhost:5432/appdb?sslmode=disable'
export GOOSE_MIGRATION_DIR=./migrations
goose status          # no need to repeat driver/dsn/dir
```

```powershell
# PowerShell (Windows 11)
$env:GOOSE_DRIVER   = 'postgres'
$env:GOOSE_DBSTRING = 'postgres://appuser:secret@localhost:5432/appdb?sslmode=disable'
$env:GOOSE_MIGRATION_DIR = './migrations'
goose status
```

For clarity, most examples below show the **explicit** form (`goose -dir ./migrations postgres "<dsn>" <command>`) so you can see every moving part. In a real project, use the env vars.

### 2.4 Creating the migrations directory and your first migration **[B]**

By convention, migrations live in a `migrations/` directory at the project root. Create it, then use `goose create` to scaffold your first migration. `goose create <name> sql` generates an empty, correctly-annotated SQL file with a **timestamped** name (`YYYYMMDDhhmmss_name.sql`) so that two developers creating migrations at the same time on different branches get non-colliding filenames.

```bash
mkdir migrations

# Scaffold a SQL migration named "create_users_table".
goose -dir ./migrations create create_users_table sql
# => Created new file: migrations/20260711103000_create_users_table.sql
```

Open the generated file. goose pre-fills the annotation skeleton:

```sql
-- +goose Up
-- +goose StatementBegin
SELECT 'up SQL query';
-- +goose StatementEnd

-- +goose Down
-- +goose StatementBegin
SELECT 'down SQL query';
-- +goose StatementEnd
```

Those `-- +goose ...` lines are **annotations**: ordinary SQL comments that goose parses to understand the file. `-- +goose Up` marks where the "apply" SQL begins; `-- +goose Down` marks the "undo" SQL. (We cover `StatementBegin/End` fully in §3 — for simple single statements you can even omit them.) Replace the placeholders with real DDL:

```sql
-- +goose Up
CREATE TABLE users (
	id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
	email        CITEXT NOT NULL UNIQUE,        -- CITEXT = case-insensitive text; emails compare case-insensitively
	password_hash TEXT   NOT NULL,              -- the Argon2id PHC string (see GO_JWT_ARGON2_GUIDE.md), NEVER the password
	created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
	updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- +goose Down
DROP TABLE users;
```

> **Note on `CITEXT`:** it is an extension type. If your DB doesn't have the extension yet, this migration will fail — which is a *good* demonstration of ordering: you'd write an earlier migration `CREATE EXTENSION IF NOT EXISTS citext;`. We do exactly that in the §11 worked example. For now, if you're following along on a bare database, use `TEXT` instead of `CITEXT` for `email`.

### 2.5 Running it: `up`, `status`, and the version table **[B]**

Apply all pending migrations with `goose up`:

```bash
goose -dir ./migrations postgres "postgres://appuser:secret@localhost:5432/appdb?sslmode=disable" up
```

```text
2026/07/11 10:35:02 OK   20260711103000_create_users_table.sql (14.2ms)
2026/07/11 10:35:02 goose: successfully migrated database to version: 20260711103000
```

Two things just happened. First, goose ran your `-- +goose Up` SQL, creating the `users` table. Second — and this is the mechanism that makes the whole system work — goose created a **tracking table** (default name `goose_db_version`) if it didn't exist, and inserted a row recording that version `20260711103000` is now applied. That table is how goose knows, on the next run, what to skip.

Inspect it. `goose status` reads the tracking table and the migrations directory and shows you the state of each migration — applied (with when) or pending:

```bash
goose -dir ./migrations postgres "<dsn>" status
```

```text
    Applied At                  Migration
    =======================================
    2026-07-11 10:35:02 +0000   20260711103000_create_users_table.sql
```

Peek at the tracking table directly to demystify it:

```sql
SELECT * FROM goose_db_version ORDER BY id;
```

```text
 id |  version_id   | is_applied |          tstamp
----+---------------+------------+----------------------------
  1 |             0 | t          | 2026-07-11 10:35:02.1+00     -- the baseline "version 0" row goose seeds
  2 | 20260711103000| t          | 2026-07-11 10:35:02.2+00     -- your migration
```

The columns: `version_id` is the numeric version parsed from the filename; `is_applied` is a boolean (goose writes an `is_applied=false` history row conceptually when rolling down, though the current version is derived from the max applied `version_id`); `tstamp` is when it happened. You rarely touch this table by hand — but knowing it is *just a table* removes all the mystery. There is no hidden state; goose's entire memory is these rows.

Now undo it with `goose down` (applies the `-- +goose Down` SQL of the most recent applied migration):

```bash
goose -dir ./migrations postgres "<dsn>" down
```

```text
2026/07/11 10:37:10 OK   20260711103000_create_users_table.sql (8.1ms)
```

The `users` table is gone and its row removed from `goose_db_version`. Run `up` again and you're back. **That loop — create, up, status, down — is the entire beginner workflow.** Everything else in this guide makes it safe, embeddable, and production-grade.

### 2.6 `goose fix` — from timestamps to sequence numbers **[B/I]**

`goose create` names files with a **timestamp** (`20260711103000_...`) so parallel work on branches doesn't collide. But timestamps have a subtle ordering hazard: suppose Alice creates a migration at 10:30 on her branch and Bob creates one at 10:31 on his; if Bob's merges first and *then* Alice's, the applied order in your merged history is 10:30-before-10:31 by filename, but the *review/merge* order was the opposite. On most changes this is harmless, but for dependent changes it can bite.

The convention that removes the ambiguity is: **develop with timestamps, then before merging/shipping, convert to sequential numbers** with `goose fix`. It renames the timestamped files to zero-padded sequential ones — `00001_...`, `00002_...` — in timestamp order, giving you a clean, obviously-ordered, monotonic sequence:

```bash
goose -dir ./migrations fix
# renames 20260711103000_create_users_table.sql -> 00001_create_users_table.sql, etc.
```

Run `goose fix` as part of preparing a branch for merge (many teams enforce it in CI: "the migrations dir must be sequentially numbered"). §12 revisits the branch-ordering problem in full. For now internalize the rule: **timestamps while iterating, `goose fix` before it ships.**

---

## 3. SQL Migrations In Depth

SQL migrations are the workhorse — the great majority of your migrations will be plain SQL files. This section teaches every part of the format in depth, because the annotations are subtle and the failure modes (a function body split at a semicolon, an index that locks a table for ten minutes) are exactly the ones that cause outages.

### 3.1 Anatomy of a SQL migration file **[B]**

A goose SQL migration is a single `.sql` file with two labelled regions. goose reads the file, splits it at the annotations, and runs the appropriate region for the direction you asked for:

```sql
-- +goose Up
-- ... SQL that moves the schema FORWARD goes here ...

-- +goose Down
-- ... SQL that moves the schema BACKWARD (undoes Up) goes here ...
```

The annotations are SQL comments (`--`) beginning with `+goose`. Any SQL between `-- +goose Up` and `-- +goose Down` is the up-migration; anything after `-- +goose Down` is the down-migration. goose executes the statements in order. **By default, each migration file runs inside a single database transaction** (on PostgreSQL, which has transactional DDL), so if any statement in the file fails, the whole migration rolls back and the schema is left untouched — a crucial safety property we return to in §3.5 and §12.

### 3.2 Statement splitting, and why `StatementBegin/End` exists **[B/I]**

Here is a subtlety that trips up everyone once. goose needs to send statements to the database, and to do that it splits your SQL into individual statements. Its default splitting heuristic is: **for simple migrations, treat each `;`-terminated line as a statement.** For 90% of DDL (`CREATE TABLE ...;`, `ALTER TABLE ...;`, `CREATE INDEX ...;`) this works perfectly and you write nothing special.

But some SQL statements *contain semicolons inside them* — most notably **PL/pgSQL function and trigger bodies**, and anything using **dollar-quoting** (`$$ ... $$`). If goose naively split on `;`, it would chop a function body into fragments mid-statement and send garbage to the server. To tell goose "treat everything between these markers as ONE statement, do not split it," you wrap it in:

```sql
-- +goose StatementBegin
... a single statement that internally contains semicolons ...
-- +goose StatementEnd
```

**Rule of thumb:** if a single SQL statement contains a semicolon that is *not* the statement terminator — i.e. functions, procedures, triggers, `DO $$ ... $$` blocks, dollar-quoted string bodies — wrap it in `StatementBegin/StatementEnd`. Plain `CREATE TABLE`/`ALTER`/`CREATE INDEX` do **not** need it. Forgetting the wrapper around a function is the single most common goose authoring bug; the symptom is a syntax error near the middle of your function body.

Here is a complete, realistic example — a PL/pgSQL function plus a trigger that auto-updates an `updated_at` column, which *requires* the wrapper because the function body is full of semicolons:

```sql
-- +goose Up
-- The function body contains semicolons, so it MUST be wrapped so goose sends it whole.
-- +goose StatementBegin
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
	NEW.updated_at = now();   -- this semicolon is INSIDE the body — must not be split
	RETURN NEW;               -- and so is this one
END;
$$ LANGUAGE plpgsql;
-- +goose StatementEnd

-- A trigger definition is a single statement with no interior semicolons, so no wrapper needed.
CREATE TRIGGER users_set_updated_at
	BEFORE UPDATE ON users
	FOR EACH ROW
	EXECUTE FUNCTION set_updated_at();

-- +goose Down
DROP TRIGGER IF EXISTS users_set_updated_at ON users;
-- Dropping the function is also a single simple statement.
DROP FUNCTION IF EXISTS set_updated_at();
```

Read that carefully. The `CREATE FUNCTION` is wrapped because its body has interior semicolons; the `CREATE TRIGGER` and both `DROP`s are single statements and are left unwrapped. Notice the `Down` uses `IF EXISTS` — an idempotency habit we discuss in §3.6.

### 3.3 `NO TRANSACTION` — for DDL that cannot run in a transaction **[I]**

We said each migration runs in a transaction by default. That default is a *safety* feature — a failed migration rolls back cleanly. But a handful of PostgreSQL operations **cannot run inside a transaction block at all**, and the most important one for real systems is:

```sql
CREATE INDEX CONCURRENTLY ...
```

`CREATE INDEX` (non-concurrent) takes an `ACCESS EXCLUSIVE`-ish lock that **blocks writes to the table for the entire duration of the build** — on a large, hot table that can be minutes of downtime. `CREATE INDEX CONCURRENTLY` builds the index without blocking writes, which is what you want in production. But `CONCURRENTLY` is *specifically forbidden* inside a transaction block, and since goose wraps migrations in a transaction by default, a `CONCURRENTLY` migration will error with `CREATE INDEX CONCURRENTLY cannot run inside a transaction block`.

The fix is the file-level annotation **`-- +goose NO TRANSACTION`**, placed at the *top* of the file. It tells goose "run this migration's statements *without* wrapping them in a transaction." That is exactly what non-transactional DDL needs:

```sql
-- +goose NO TRANSACTION
-- ^ MUST be at the very top. Disables goose's transaction wrapper for THIS file.

-- +goose Up
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_sessions_user_id
	ON sessions (user_id);

-- +goose Down
DROP INDEX CONCURRENTLY IF EXISTS idx_sessions_user_id;
```

**The trade-off you must understand:** with `NO TRANSACTION`, you lose the automatic all-or-nothing rollback. If this file has *two* statements and the second fails, the first is already committed — the migration is now *partially applied*, and goose will mark it as failed, leaving you to clean up by hand. Therefore the discipline is: **keep `NO TRANSACTION` migrations to a single statement** (or statements that are each independently safe and idempotent), so there is no partial-application ambiguity. One `CREATE INDEX CONCURRENTLY` per `NO TRANSACTION` file is the canonical pattern.

> **⚡ Gotcha specific to `CONCURRENTLY`:** if a `CREATE INDEX CONCURRENTLY` *itself* fails midway (e.g. a unique violation while building a unique index), PostgreSQL leaves behind an **invalid** index that still occupies space and must be dropped manually (`DROP INDEX CONCURRENTLY idx_name;`) before you retry. This is a PostgreSQL behavior, not a goose one, but it's the kind of thing that bites during a migration. Always `DROP INDEX CONCURRENTLY IF EXISTS` first in a retry, or use `IF NOT EXISTS` on the create.

### 3.4 One logical change per migration **[B/I]**

A migration should represent **one logical, atomic schema change**: "create the sessions table," "add the `mfa_enabled` column," "add the unique index on email." Resist the urge to cram a week of schema evolution into one giant file. The reasons are practical:

- **Reviewability.** A reviewer can reason about "add one nullable column" instantly; a 400-line file mixing five concerns hides the dangerous line among the safe ones.
- **Rollback granularity.** If change B is bad but change A (in the same file) is good, you cannot undo B without also undoing A.
- **Bisecting.** When a schema change breaks something, small migrations let you pinpoint *which* change.
- **Zero-downtime staging.** The expand-contract pattern (§10) explicitly requires *separate* migrations for "add column," "backfill," "add constraint," "drop old" — often shipped across multiple deploys. You cannot do that if they're fused into one file.

The counter-force is not *too* granular: a migration that logically requires two statements together (create a table *and* its index; add a column *and* its check constraint that must exist atomically) belongs in one file. The rule is **one logical change**, which may be one statement or a few tightly-coupled ones — not "one statement per file" and not "one release per file."

### 3.5 Transactions per migration (PostgreSQL's superpower) **[I]**

Worth stating explicitly because it distinguishes PostgreSQL from MySQL and shapes how safe your migrations are: **PostgreSQL has transactional DDL.** `CREATE TABLE`, `ALTER TABLE`, `DROP`, `CREATE INDEX` (non-concurrent), adding constraints — all of these can run inside a transaction, and if the transaction rolls back, **the schema change is completely undone as if it never happened.** goose leans on this: by default it wraps each migration file in `BEGIN ... COMMIT`, so a migration that fails on its third statement leaves your database *exactly* as it was before the migration started. No half-created tables, no orphaned columns.

This is why a failed `goose up` on PostgreSQL is usually a non-event: you read the error, fix the SQL, and re-run — the database is clean. Contrast MySQL, where most DDL is *not* transactional and auto-commits: a migration that fails halfway leaves the earlier statements applied, and you must manually reconcile. If you are on PostgreSQL (as this guide assumes), treasure transactional DDL and **do not casually reach for `NO TRANSACTION`** — you only give up this safety when a specific operation (`CONCURRENTLY`, `ALTER TYPE ... ADD VALUE` in older PG, `VACUUM`, etc.) forces you to.

### 3.6 Idempotency and defensive DDL **[I]**

An *idempotent* operation is one that has the same effect whether run once or many times. Migrations, in the normal case, are *not* re-run (goose skips applied versions), so idempotency isn't strictly required. But defensive `IF EXISTS` / `IF NOT EXISTS` clauses make migrations far more robust against the messy realities of production:

- A `NO TRANSACTION` migration that partially applied and must be safely retried.
- A migration that races another instance in the window before advisory locks are in place.
- A hotfix applied manually to production that a later migration then also tries to apply.

So, as a habit:

```sql
-- +goose Up
CREATE TABLE IF NOT EXISTS audit_log (...);           -- won't error if a prior partial run created it
CREATE INDEX IF NOT EXISTS idx_audit_created_at ON audit_log (created_at);

-- +goose Down
DROP INDEX IF EXISTS idx_audit_created_at;            -- won't error if already gone
DROP TABLE IF EXISTS audit_log;
```

Two caveats. First, `IF NOT EXISTS` is *not* a substitute for correctness — if the table exists with a *different* shape than you expect, `IF NOT EXISTS` silently skips creation and you now have a schema mismatch that no error warned you about. Use it for robustness, not to paper over drift. Second, `CREATE INDEX CONCURRENTLY IF NOT EXISTS` will *skip* (not rebuild) an existing *invalid* index left by a prior failed concurrent build — so in a retry you often want an explicit `DROP INDEX CONCURRENTLY IF EXISTS` first, as noted in §3.3.

### 3.7 Ordering, dependencies, and the version number **[B/I]**

goose applies migrations in **ascending version order** — the number parsed from the filename prefix. That number is the *only* thing that determines order; the rest of the filename is a human-readable label. Consequences:

- A migration can safely depend on anything created by a *lower-numbered* migration. `00005_add_index_on_sessions.sql` can index `sessions` because `00003_create_sessions.sql` ran first.
- A migration must **never** depend on a *higher-numbered* one. If you find yourself needing that, your numbering is wrong — renumber.
- **Never renumber an already-applied migration**, and **never insert a migration "in the middle" of applied history.** If `00003` is applied in production and you add `00002.5`, production will *never* run it (its version is below the current max), while a fresh dev DB *will* — instant drift. New changes always get a *new, higher* number appended at the end. §12 hammers this.

### 3.8 A realistic multi-statement migration **[I]**

Putting §3 together — here is a production-shaped migration creating a `sessions` table (refresh-token sessions for the JWT flow in [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md)), with a foreign key, constraints, and indexes, all in one transactional file because they form one logical unit:

```sql
-- +goose Up
CREATE TABLE sessions (
	id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),  -- gen_random_uuid needs pgcrypto or PG13+
	user_id       BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
	refresh_hash  TEXT   NOT NULL,             -- store a HASH of the refresh token, never the token itself
	user_agent    TEXT,
	ip            INET,
	expires_at    TIMESTAMPTZ NOT NULL,
	revoked_at    TIMESTAMPTZ,                 -- NULL = active; set = revoked (soft state)
	created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
	CONSTRAINT sessions_expires_after_created CHECK (expires_at > created_at)
);

-- Index the FK column (Postgres does NOT auto-index FKs) so cascade deletes and lookups are fast.
CREATE INDEX idx_sessions_user_id ON sessions (user_id);
-- Partial index: only active, unexpired sessions — the rows we query on every refresh.
CREATE INDEX idx_sessions_active ON sessions (user_id)
	WHERE revoked_at IS NULL;

-- +goose Down
DROP TABLE sessions;   -- dropping the table drops its indexes and FK automatically
```

Everything here is transactional (no `CONCURRENTLY`), so it's all-or-nothing: if the second `CREATE INDEX` had a typo, the table and first index roll back too, and you re-run after fixing.

---

## 4. The Full Command Reference

goose's command surface is small and orthogonal. This section is the reference table plus a prose explanation of *when* each command is the right one — because knowing that `reset` exists is useless without knowing it is a dev-only footgun.

### 4.1 The command table **[B/I]**

Every command below is invoked as `goose -dir <dir> <driver> "<dsn>" <command> [args]` (or with `GOOSE_*` env vars set, just `goose <command>`):

| Command | What it does | When to use | Danger |
|---|---|---|---|
| `up` | Apply **all** pending migrations, in order. | The default "bring DB to latest." Dev, CI, deploy. | Low (transactional per file). |
| `up-by-one` | Apply exactly **one** pending migration. | Step forward carefully; test migrations one at a time. | Low. |
| `up-to VERSION` | Apply pending migrations up to and **including** `VERSION`. | Deploy to a *specific* schema version; phased rollout. | Low. |
| `down` | Roll back the **most recent** applied migration (its Down). | Undo the last change in dev while iterating. | **Medium** — Down may drop data. |
| `down-to VERSION` | Roll back migrations **down to** `VERSION` (exclusive). | Rewind several steps in dev. | **Medium/High** — multiple Downs. |
| `redo` | Roll back the last migration, then re-apply it (`down` then `up-by-one`). | Iterate on the migration you're currently writing. | Medium — runs a Down. |
| `reset` | Roll back **ALL** migrations (every Down, back to version 0). | Wipe a **dev** schema to nothing. | **DESTRUCTIVE — dev only.** |
| `status` | Print each migration and whether/when it's applied. | Inspect state. Read-only. | None. |
| `version` | Print the **current** DB version (max applied). | Scripting, health checks, CI assertions. | None. |
| `create NAME sql\|go` | Scaffold a new timestamped migration file. | Author a new migration. | None (touches files, not DB). |
| `fix` | Rename timestamped files to sequential `00001_...`. | Before merging/shipping a branch. | None (renames files). |
| `validate` | Parse & sanity-check migration files without running them. | CI lint step; catch malformed files early. | None (read-only, no DB writes). |

### 4.2 The "up" family in depth **[B/I]**

`up` is what you run 95% of the time. It reads `goose_db_version`, computes which migrations are missing, and applies them all in order. Idempotent in the sense that running `up` on an already-current database does nothing and reports "no migrations to run."

`up-by-one` applies only the *next* pending migration. Its value is **precision**: when you're testing a risky migration against a copy of production, you step one migration at a time and verify state between steps. It's also how a `cmd/migrate` tool can offer a "single step" mode.

`up-to VERSION` applies everything up to and including a named version, then stops. This is the tool for **phased / decoupled deploys**: suppose you have three pending migrations but only want the first two live now (the third pairs with code that isn't deployed yet). `goose up-to 00012` applies through version 12 and leaves 13 pending. This is central to expand-contract (§10), where the schema change and the code change ship in separate steps.

```bash
goose -dir ./migrations postgres "<dsn>" up          # all pending
goose -dir ./migrations postgres "<dsn>" up-by-one   # exactly one
goose -dir ./migrations postgres "<dsn>" up-to 00012 # through v12 inclusive
```

### 4.3 The "down" family — respect it **[B/I]**

`down` rolls back the single most recent migration by running its `-- +goose Down`. In development this is a convenience: you applied a migration, realized the column name is wrong, `goose down`, edit... wait — **do not edit an applied migration** (§12). The correct iterate loop for a migration you *just created and haven't shared* is `redo` (down + up) after editing; for one that's been merged, write a new migration.

`down-to VERSION` rolls back repeatedly until the DB is at `VERSION` (that version stays applied; everything above it is undone). `reset` is `down-to 0` — it undoes *everything*. Both run real Down SQL, which may `DROP TABLE`/`DROP COLUMN` and **destroy data**.

> **⚠️ `reset` and `down` in production:** Treat `reset` as a **development-only** command — running it against production drops every table your migrations created, which is to say, your entire application schema and all its data. There is essentially no legitimate production use. Even plain `down` in production is a red flag: it runs a Down migration that may drop a column populated with new data since the Up ran, silently losing it. The production philosophy (§10) is **roll forward, not back**: to reverse a bad change, write and apply a *new* migration that corrects it, under review, rather than running a Down. Some teams delete or stub out Down bodies for destructive migrations specifically to prevent an accidental `down` in prod from nuking data. We revisit this in §10 and §12.

### 4.4 `redo` — the authoring loop helper **[I]**

`redo` runs `down` then `up-by-one` on the most recent migration — i.e. it re-runs the migration you're currently working on. This is the *correct* way to iterate on a migration **that you have created but not yet merged/shared**: edit the file, `goose redo`, inspect, repeat. Because the migration hasn't left your branch, re-running it changes no shared history and no other database has applied the old version. Once a migration is merged and possibly applied elsewhere, `redo` (which edits behavior of an "applied" version) is off-limits — you're back to "write a new migration."

### 4.5 `status`, `version`, `validate` — the read-only trio **[B/I]**

`status` is your first diagnostic: it lists every migration file and, for each, whether it's applied and when. Run it before and after any migration operation, and in CI to assert "no pending migrations" on a freshly-migrated DB.

`version` prints just the current numeric version (the max applied). It's for scripting: health checks that assert the DB is at the expected version, deploy gates that refuse to start the app if `version` doesn't match what the code expects.

`validate` parses the migration files and checks them for structural problems — malformed annotations, missing Up/Down markers, duplicate version numbers, unparseable filenames — **without connecting to a database or running anything.** It's the perfect CI lint step: catch a broken migration file in the pull request, before it ever reaches a database. (§10 shows it in a CI pipeline.)

```bash
goose -dir ./migrations postgres "<dsn>" status
goose -dir ./migrations postgres "<dsn>" version
goose -dir ./migrations validate     # no DB needed
```

### 4.6 A note on `-allow-missing` and out-of-order migrations **[A]**

By default goose applies migrations strictly in order and will refuse to apply a migration whose version is *lower* than the current max applied version — it considers it "missing/out of order" and treats it as an error, protecting you from the branch-merge hazard of §3.7. Occasionally, in a controlled multi-team setting, you genuinely need to apply a lower-numbered migration that arrived late (it was merged after a higher-numbered one already ran). The `-allow-missing` flag (CLI) / `WithAllowMissing()` provider option (§7) permits this. **Use it sparingly and knowingly** — it is an escape hatch, and reaching for it routinely means your team's branching/numbering discipline (use `goose fix`, coordinate migration numbers) has broken down. In banking-grade setups, prefer to fix the ordering than to normalize out-of-order application.

---

## 5. Go Migrations: When SQL Isn't Enough

### 5.1 Why Go migrations exist **[I]**

Most schema changes are pure DDL and belong in SQL migrations. But some migrations need to *transform existing data* in ways SQL alone expresses poorly or not at all — and that's where **Go migrations** come in. A Go migration is a migration whose Up and Down are Go functions instead of SQL text, giving you the full language: loops, conditionals, calls into your own packages, cryptography, HTTP, parsing.

The canonical use cases:

- **Data backfills that need application logic.** You add a `password_algo` column and want to backfill it by *parsing* each existing `password_hash` PHC string to detect its algorithm — a job for Go string parsing, not SQL.
- **Re-hashing / re-encrypting data.** You're rotating an encryption key or upgrading a hash; you must read each row, run it through a Go crypto function, and write it back. Pure SQL cannot call Argon2.
- **Complex transforms.** Splitting a `full_name` column into `first`/`last` with real name-parsing rules; normalizing denormalized JSON; migrating data between differently-shaped tables with business logic in between.
- **Conditional migrations.** "Only do X if the environment is Y," or branching on runtime-computed state.

The guideline: **use SQL migrations for schema (DDL) and simple set-based data changes (`UPDATE ... WHERE`), and reach for Go migrations only when you need real code** — application logic that SQL can't express. Do not use a Go migration to run SQL you could have put in a `.sql` file; the SQL file is simpler, reviewable as SQL, and needs no compilation.

### 5.2 Scaffolding and the function signature **[I]**

`goose create <name> go` scaffolds a Go migration file. Inside, you *register* your up and down functions with goose in the file's `init()` so goose discovers them:

```bash
goose -dir ./migrations create backfill_password_algo go
# => Created new file: migrations/20260711120000_backfill_password_algo.go
```

The generated file, filled in for a realistic backfill:

```go
package migrations

import (
	"context"
	"database/sql"
	"strings"

	"github.com/pressly/goose/v3"
)

// init registers this migration's up/down functions with goose.
// AddMigrationContext ties them to THIS file's version (parsed from the filename).
func init() {
	goose.AddMigrationContext(upBackfillPasswordAlgo, downBackfillPasswordAlgo)
}

// The signature for a transactional Go migration: it receives a context and a *sql.Tx.
// goose has already opened the transaction; you use tx for all queries; goose commits
// (or rolls back on error) around your function. Return an error to abort + roll back.
func upBackfillPasswordAlgo(ctx context.Context, tx *sql.Tx) error {
	// Read every user's hash so we can classify its algorithm.
	rows, err := tx.QueryContext(ctx, `SELECT id, password_hash FROM users`)
	if err != nil {
		return err
	}
	defer rows.Close()

	// Collect first, then update — don't UPDATE while iterating an open cursor on the same conn.
	type row struct {
		id   int64
		hash string
	}
	var batch []row
	for rows.Next() {
		var r row
		if err := rows.Scan(&r.id, &r.hash); err != nil {
			return err
		}
		batch = append(batch, r)
	}
	if err := rows.Err(); err != nil {
		return err
	}

	// Application logic SQL can't easily do: parse the PHC string prefix to detect the algorithm.
	for _, r := range batch {
		algo := "unknown"
		switch {
		case strings.HasPrefix(r.hash, "$argon2id$"):
			algo = "argon2id"
		case strings.HasPrefix(r.hash, "$2a$"), strings.HasPrefix(r.hash, "$2b$"):
			algo = "bcrypt"
		}
		if _, err := tx.ExecContext(ctx,
			`UPDATE users SET password_algo = $1 WHERE id = $2`, algo, r.id); err != nil {
			return err
		}
	}
	return nil
}

func downBackfillPasswordAlgo(ctx context.Context, tx *sql.Tx) error {
	// The inverse: clear what we set. (The COLUMN itself is added/dropped by a separate SQL migration.)
	_, err := tx.ExecContext(ctx, `UPDATE users SET password_algo = NULL`)
	return err
}
```

Key points about the signature `func(ctx context.Context, tx *sql.Tx) error`:

- The `ctx` is goose's context — respect it (pass it to every query) so cancellation/timeouts propagate.
- The `tx` is a transaction goose already opened. Everything you do goes through `tx`. **Do not open your own connection or transaction** — use the one goose gave you, so your work commits atomically with goose's version-table update. goose commits on `nil` return, rolls back on non-`nil` error.
- Returning an error aborts and rolls back the whole migration (on PostgreSQL, cleanly).

> **⚡ v3 signature note:** goose v3 uses the **context-aware** registration functions (`AddMigrationContext`, `AddMigrationNoTxContext`) and the `func(ctx, tx)` signature. Older code and blog posts use the non-context `AddMigration(up, down)` with `func(tx *sql.Tx) error`; those still exist for compatibility but are legacy. Always use the `...Context` variants in new code.

### 5.3 Non-transactional Go migrations **[I]**

Just as SQL migrations sometimes need `NO TRANSACTION`, a Go migration sometimes must run outside a transaction — most importantly a **large batched backfill** where wrapping millions of `UPDATE`s in one transaction would bloat the WAL, hold locks forever, and risk timeouts. For that, register with **`AddMigrationNoTxContext`**, whose functions receive a `*sql.DB` (a connection pool) instead of a `*sql.Tx`:

```go
func init() {
	// NoTx variant: functions get *sql.DB, and goose does NOT wrap them in a transaction.
	goose.AddMigrationNoTxContext(upBigBackfill, downBigBackfill)
}

// Note the parameter type: *sql.DB, not *sql.Tx. YOU control transactionality now.
func upBigBackfill(ctx context.Context, db *sql.DB) error {
	// Backfill in bounded batches so each is a short, committed transaction.
	// This keeps locks short and lets the migration resume if interrupted.
	const batchSize = 5000
	for {
		res, err := db.ExecContext(ctx, `
			WITH batch AS (
				SELECT id FROM users
				WHERE normalized_email IS NULL
				LIMIT $1
				FOR UPDATE SKIP LOCKED       -- don't block/overlap with concurrent workers
			)
			UPDATE users u
			SET normalized_email = lower(u.email)
			FROM batch
			WHERE u.id = batch.id`, batchSize)
		if err != nil {
			return err
		}
		n, _ := res.RowsAffected()
		if n == 0 {
			break // no more rows to backfill — done
		}
	}
	return nil
}

func downBigBackfill(ctx context.Context, db *sql.DB) error {
	_, err := db.ExecContext(ctx, `UPDATE users SET normalized_email = NULL`)
	return err
}
```

The batched pattern above is the production-correct way to backfill a large table: each `Exec` is its own short transaction, locks are held briefly, and if the migration is killed halfway it resumes from where it left off on the next run (because it only touches rows still `NULL`). This is a §10 zero-downtime technique, shown here because it *requires* the NoTx Go-migration form.

### 5.4 Mixing SQL and Go migrations **[I]**

SQL and Go migrations coexist freely in the same directory — goose orders them all by version number regardless of type. A common and clean pattern for a data-dependent change spans multiple migrations of both kinds:

```text
migrations/
  00010_add_password_algo_column.sql   -- SQL: ALTER TABLE users ADD COLUMN password_algo TEXT
  00011_backfill_password_algo.go      -- Go:  parse hashes, populate password_algo
  00012_password_algo_not_null.sql     -- SQL: ALTER TABLE users ALTER COLUMN password_algo SET NOT NULL
```

This is exactly the **expand → backfill → constrain** sequence: add the column nullable (10), fill it with Go logic (11), then enforce `NOT NULL` once every row has a value (12). Splitting it across three migrations is not incidental — it's what makes the change safe to apply to a live table.

> **⚡ Go migrations require a compiled binary.** A subtle but important operational fact: the standalone `goose` CLI you `go install`ed is a *pre-built* binary — it does **not** know about your Go migration functions, because those are compiled into *your* program, not into goose. So Go migrations only run when executed from a binary that imports your `migrations` package (your `cmd/migrate` tool — §7/§11). If you `goose up` with the generic CLI against a directory containing `.go` migrations, it will process the SQL ones but cannot run the Go ones. **The practical rule: if you use Go migrations, run migrations through your own binary, not the generic CLI.** This nudges every serious project toward the programmatic API of §7 anyway.

---

## 6. Embedding Migrations with embed.FS

### 6.1 Why embed migrations **[I]**

By default goose reads migration files from a directory on disk. In production that means your deployed artifact must *include* the `migrations/` folder alongside the binary, in the right relative path, and keep them in sync. That's a fragile coupling: a Docker image that copies the binary but forgets the migrations, a version skew where the binary is v2 but the shipped migrations are v1 — both are real incidents.

Go's `embed` package (standard library since Go 1.16) eliminates the coupling by **compiling the migration files into the binary itself.** The `//go:embed` directive reads files at *build* time and bakes them into an `embed.FS` (an in-memory read-only filesystem). Your binary is then fully self-contained: one file to ship, migrations guaranteed to match the code they were built with, no external directory to lose. For containerized and single-binary deploys this is the standard approach.

### 6.2 How to embed and point goose at it **[I]**

Two steps: embed the files into an `embed.FS`, then tell goose to read from that FS instead of disk with **`goose.SetBaseFS(embedFS)`** (global API) or by passing the FS to `NewProvider` (§7).

```go
package migrations

import "embed"

// The //go:embed directive bakes every .sql (and this package's compiled .go migrations)
// into the binary at build time. The path is relative to THIS source file.
//
//go:embed *.sql
var Embedded embed.FS
```

Then in your migrate command, with the **global API**:

```go
import (
	"database/sql"

	"github.com/pressly/goose/v3"
	"yourapp/migrations" // the package holding the embed.FS above
)

func runMigrations(db *sql.DB) error {
	// Tell goose to read migration files from the embedded FS, not the disk.
	goose.SetBaseFS(migrations.Embedded)

	if err := goose.SetDialect("postgres"); err != nil {
		return err
	}
	// "." is the directory WITHIN the embed.FS (the files are at its root because we embedded *.sql
	// from the same package directory).
	return goose.Up(db, ".")
}
```

A few practical notes:

- `//go:embed *.sql` embeds only SQL files. Go migrations are compiled into the binary anyway (they're `.go` code), and their `init()` registration is what wires them in — so embedding `*.sql` plus importing the package (which triggers the `init()`s) covers a mixed SQL+Go set. Ensure the migrations package is imported so its `init()`s run (a blank import `_ "yourapp/migrations"` if you don't otherwise reference it).
- The embed path is **relative to the Go source file** containing the directive, and cannot climb out with `..`. Keep the `embed.go` file in the same directory as the migration files (or a parent that includes them, e.g. `//go:embed all:migrations`).
- Once you call `SetBaseFS`, the `-dir` flag / directory argument is interpreted *within* the embedded FS, not the real disk.

> **⚡ Modern API preference:** `SetBaseFS` is the *global-API* way. With `NewProvider` (§7) you pass the `fs.FS` as a direct argument — no global mutation, which is why the provider is the recommended modern path. The embedding concept is identical; only the wiring differs.

---

## 7. The Modern Programmatic API: NewProvider

### 7.1 Two APIs, and why the provider wins **[I/A]**

goose ships two library APIs, and understanding the split is important because most older documentation shows the wrong one for new code:

- **The global API** — `goose.SetDialect("postgres")`, `goose.SetBaseFS(fs)`, `goose.Up(db, dir)`, `goose.AddMigrationContext(...)`. These functions carry **package-level (global) state**: the dialect, the base FS, the registered Go migrations all live in package variables. That's convenient for a quick script but has real drawbacks — it's **not thread-safe** (two goroutines calling `SetDialect` race), it's **hard to test** (global state leaks between tests), and you can't cleanly run two providers with different configs in one process.

- **⚡ The provider API** — `goose.NewProvider(dialect, db, fsys, opts...)` returns a `*goose.Provider` object that **holds all its own state** (dialect, DB, filesystem, table name, logger, registered migrations) with **no globals**. It's thread-safe, trivially testable (construct one per test), and configured explicitly via functional options. **This is the recommended modern API**, and what you should use in any new application. The global API remains for backward compatibility and quick CLIs.

We'll build with the provider from here on.

### 7.2 Constructing a provider **[I/A]**

`NewProvider` takes the dialect constant, an open `*sql.DB`, an `fs.FS` holding the migrations, and zero or more options. It validates everything up front (unknown dialect, missing migrations, duplicate versions) and returns an error rather than failing later:

```go
package db

import (
	"context"
	"database/sql"
	"fmt"
	"log/slog"

	"github.com/pressly/goose/v3"
	"yourapp/migrations"
)

// NewMigrationProvider builds a configured goose Provider for our app.
func NewMigrationProvider(sqlDB *sql.DB, logger *slog.Logger) (*goose.Provider, error) {
	provider, err := goose.NewProvider(
		goose.DialectPostgres,     // the dialect CONSTANT (not the string "postgres")
		sqlDB,                     // an open *sql.DB (see §8 for opening it over pgx)
		migrations.Embedded,       // the embed.FS from §6 — no global SetBaseFS needed
		// --- functional options ---
		goose.WithVerbose(true),               // log each migration as it runs
		goose.WithSlog(logger),                // ⚡ structured logging via *slog.Logger
		goose.WithTableName("schema_migrations"), // customize the tracking table name (default goose_db_version)
	)
	if err != nil {
		return nil, fmt.Errorf("build goose provider: %w", err)
	}
	return provider, nil
}
```

Notable options (there are more; check pkg.go.dev):

| Option | Purpose |
|---|---|
| `WithSlog(*slog.Logger)` | ⚡ Route goose's output through Go's structured logger — JSON logs, levels, fields. |
| `WithVerbose(bool)` | Log each migration applied/rolled back. |
| `WithTableName(string)` | Use a custom tracking-table name (e.g. `schema_migrations`) instead of `goose_db_version`. |
| `WithAllowMissing()` | Permit applying out-of-order (lower-version) migrations (the §4.6 escape hatch). |
| `WithStore(store)` | Supply a custom store implementation for the tracking table (advanced; usually unneeded). |
| `WithGoMigrations(...)` | Register Go migrations explicitly on the provider (instead of relying on global `init()` registration). |
| `WithDisableGlobalRegistry(bool)` | Ignore globally-registered Go migrations — full isolation from package globals. |

> **⚡ Version note:** the exact set of provider options grows across releases. `WithSlog` and `WithTableName` are stable and central; verify newer ones (`WithLogger`, `WithExcludeNames`, disable-versioning knobs) against your installed version. The provider constructor validates duplicate versions and missing migrations *at construction*, which is a nice fail-fast: a malformed migration set won't even build a provider.

### 7.3 Running migrations through the provider **[I/A]**

The provider methods mirror the CLI commands, all context-aware, all returning structured results:

```go
func Migrate(ctx context.Context, provider *goose.Provider) error {
	// Up applies all pending migrations and returns a slice of results — one per applied migration.
	results, err := provider.Up(ctx)
	if err != nil {
		return fmt.Errorf("goose up: %w", err)
	}
	for _, r := range results {
		// Each result has Source (file), Direction, Duration, Empty, etc. Great for structured logs.
		slog.Info("applied migration",
			"version", r.Source.Version,
			"path", r.Source.Path,
			"duration", r.Duration)
	}
	return nil
}
```

The full provider method set, matching §4:

| Provider method | CLI equivalent | Returns |
|---|---|---|
| `provider.Up(ctx)` | `up` | `[]*MigrationResult`, error |
| `provider.UpByOne(ctx)` | `up-by-one` | `*MigrationResult`, error |
| `provider.UpTo(ctx, version)` | `up-to` | `[]*MigrationResult`, error |
| `provider.Down(ctx)` | `down` | `*MigrationResult`, error |
| `provider.DownTo(ctx, version)` | `down-to` | `[]*MigrationResult`, error |
| `provider.Status(ctx)` | `status` | `[]*MigrationStatus`, error |
| `provider.GetDBVersion(ctx)` | `version` | `int64`, error |
| `provider.HasPending(ctx)` | (deploy gate) | `bool`, error |
| `provider.ListSources()` | (introspection) | `[]*Source` (no DB call) |

`HasPending` deserves a special mention: it answers "are there migrations this DB hasn't applied?" without applying them — perfect for a **deploy gate** ("refuse to start the app if the DB isn't fully migrated") or a health check.

```go
// A read-only check you can run at app startup to REFUSE to serve on a stale schema.
func assertSchemaCurrent(ctx context.Context, p *goose.Provider) error {
	pending, err := p.HasPending(ctx)
	if err != nil {
		return err
	}
	if pending {
		return errors.New("database has unapplied migrations; refusing to start (run the migrate step first)")
	}
	return nil
}
```

### 7.4 Where to run migrations: dedicated binary vs on-boot **[A]**

You now *can* run migrations from application code — the question is *should* you, and *when*. There are three patterns, in increasing production-safety:

1. **On app boot, unconditionally.** The server process calls `provider.Up(ctx)` before serving. Simplest, and fine for a single-instance hobby app. **Dangerous at scale**: every instance that boots races to migrate (split-brain), coupling schema changes to every restart, with no gate. Avoid in production without at least the advisory lock of §10.

2. **A dedicated `cmd/migrate` binary (or subcommand).** Migrations run as a *separate, explicit step* — a distinct binary or a subcommand (`myapp migrate up`) executed by your deploy pipeline (a Kubernetes `Job`, an init container, a CI deploy stage) *before* the new app version rolls out. The app server itself only *checks* (`HasPending`) and refuses to start if the schema is stale, but never migrates. **This is the banking-grade default** — schema changes are a deliberate, gated, observable step, decoupled from serving traffic.

3. **A gated, advisory-locked boot runner.** A middle ground: the app *may* run migrations on boot, but only after acquiring a Postgres advisory lock so exactly one instance migrates while others wait, then all proceed. Acceptable when a separate deploy step is impractical. §10 implements the lock.

We build pattern 2 (with a nod to 3) in §11. The provider API is what makes all three clean, because it's just a method call on an object you construct wherever you need it.

---

## 8. pgx Integration: One Pool for Migrations and App

### 8.1 The impedance: goose speaks database/sql, your app speaks pgx **[I/A]**

Your application uses **pgx v5**, the best-in-class PostgreSQL driver — either directly via `*pgxpool.Pool` or under Ent. But goose is built on Go's standard **`database/sql`** abstraction: it wants a `*sql.DB`. So there's a small impedance to bridge: how do you give goose a `*sql.DB` that talks to the same PostgreSQL, ideally the *same connection pool*, as your pgx-based app? pgx provides two clean bridges through its **`stdlib`** subpackage.

### 8.2 Bridge 1: open a fresh `*sql.DB` over pgx's stdlib driver **[I/A]**

pgx registers a `database/sql` driver named `"pgx"` via a blank import. Import it for side effects, then `sql.Open("pgx", dsn)` gives you a `*sql.DB` backed by pgx. This is the simplest bridge and ideal for a *dedicated migrate binary* that opens its own short-lived connection, migrates, and exits:

```go
package main

import (
	"database/sql"

	_ "github.com/jackc/pgx/v5/stdlib" // registers the "pgx" database/sql driver (blank import for side effects)
	"github.com/pressly/goose/v3"
)

func openForMigrations(dsn string) (*sql.DB, error) {
	// "pgx" is the driver name registered by the stdlib blank import above.
	db, err := sql.Open("pgx", dsn)
	if err != nil {
		return nil, err
	}
	// sql.Open is lazy — it doesn't connect. Ping to fail fast if the DSN/credentials are wrong.
	if err := db.Ping(); err != nil {
		return nil, err
	}
	return db, nil
}

func main() {
	db, err := openForMigrations("postgres://appuser:secret@localhost:5432/appdb?sslmode=require")
	if err != nil {
		panic(err)
	}
	defer db.Close()

	provider, err := goose.NewProvider(goose.DialectPostgres, db, nil /* fsys */)
	_ = provider
	_ = err
	// ... provider.Up(ctx) ...
}
```

### 8.3 Bridge 2: share your existing `*pgxpool.Pool` **[I/A]**

Often your app already has a `*pgxpool.Pool` — the connection pool the whole service uses. Rather than open a *second*, separate connection just for migrations, you can wrap the existing pool as a `*sql.DB` with **`stdlib.OpenDBFromPool(pool)`**. This is the "**one pool, migrations + app**" story: a single pool of connections, configured once (TLS, timeouts, pool size, credentials), used by both your Ent/pgx queries *and* goose. Fewer connections, one configuration surface, one place to reason about.

```go
package db

import (
	"context"
	"database/sql"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/jackc/pgx/v5/stdlib"
)

// Connect builds the app's single pgxpool.Pool AND a database/sql view of it for goose.
func Connect(ctx context.Context, dsn string) (*pgxpool.Pool, *sql.DB, error) {
	cfg, err := pgxpool.ParseConfig(dsn)
	if err != nil {
		return nil, nil, err
	}
	// Tune the pool once, here — both the app and goose inherit it.
	cfg.MaxConns = 10
	cfg.MinConns = 2

	pool, err := pgxpool.NewWithConfig(ctx, cfg)
	if err != nil {
		return nil, nil, err
	}
	if err := pool.Ping(ctx); err != nil {
		pool.Close()
		return nil, nil, err
	}

	// Wrap the SAME pool as a *sql.DB for goose. Both talk to one set of connections.
	sqlDB := stdlib.OpenDBFromPool(pool)

	return pool, sqlDB, nil
}
```

> **⚡ Lifecycle caveat with `OpenDBFromPool`:** the returned `*sql.DB` is a *view* over the pool. Close the **pool** (`pool.Close()`) to release connections; the `*sql.DB` wrapper doesn't own them independently. Don't double-close, and be deliberate about shutdown order. For a *dedicated migrate binary* that just runs and exits, Bridge 1 (its own `sql.Open("pgx", dsn)`) is often cleaner precisely because its lifecycle is trivial; Bridge 2 shines in a long-lived app that also runs a gated boot-migration (§10).

### 8.4 Dialect setup recap **[I]**

Whichever bridge you use, tell goose it's talking to PostgreSQL:

- **Provider API (⚡ recommended):** pass `goose.DialectPostgres` as the first argument to `NewProvider`. Done — no global mutation.
- **Global API:** call `goose.SetDialect("postgres")` once before running. Note it takes the *string* `"postgres"`; the provider takes the *constant* `goose.DialectPostgres`. (Also accepted historically: `"pgx"` as an alias in some versions — but `"postgres"` is canonical.)

That's the whole integration. goose doesn't care that pgx is underneath; it issues standard SQL through `database/sql`, and pgx executes it. Your app gets pgx's performance for queries; goose gets a `*sql.DB` for migrations; and with Bridge 2 they share one pool.

---

## 9. Coexisting with Ent: Goose as the Single Source of Truth

### 9.1 The tension, stated honestly **[I/A]**

This is the section that trips up teams using both Ent and goose, so we'll be precise. Ent (see [Go ent ORM](GO_ENT_ORM_GUIDE.md)) has its *own* schema story: you define entities as Go code under `ent/schema/`, and Ent can create/alter the database tables to match — either at runtime via `client.Schema.Create(ctx)` (auto-migration) or via Atlas-generated versioned migration files. Meanwhile goose *also* wants to own the schema, as an ordered set of SQL migrations. **Both cannot be the source of truth.** If Ent auto-creates a `users` table and goose also has a `CREATE TABLE users` migration, they'll fight — duplicate objects, drift, confusion about "what actually made this column."

The resolution this guide adopts — and the one appropriate for banking-grade systems — is unambiguous: **goose is the single source of truth for the database schema. Ent is a *read model* of that schema, used only to generate the type-safe query layer, and is never allowed to modify the database at runtime.**

### 9.2 The division of labour **[I/A]**

| Concern | Owner | Mechanism |
|---|---|---|
| The actual DDL that runs against the DB | **goose** | Hand-authored, reviewed SQL migrations in `migrations/` |
| Table/column/index existence in prod | **goose** | `goose up` as a gated deploy step |
| Type-safe Go query builders | **Ent** | `ent/schema/*.go` → `go generate ./ent` |
| Keeping Ent's schema *definitions* consistent with the DB | **You** (discipline / codegen) | Mirror goose's DDL in Ent schema, or generate Ent from the migrated DB |
| Runtime `Schema.Create` / auto-migrate | **Nobody** | **Explicitly disabled in production** |

So: you write a goose migration that creates a column; you *also* update the corresponding Ent schema field so Ent's generated code knows the column exists; you run `go generate ./ent` to regenerate the query layer. Ent never touches the database structure — it only *reads and writes rows* through the schema goose built.

### 9.3 Disabling Ent's auto-migration in production **[I/A]**

The single most important operational rule: **do not call `client.Schema.Create(ctx)` in production.** That call is Ent's runtime auto-migration — exactly the uncontrolled, unreviewable mechanism §1.4 warned against. In a goose-owned world, it must not run. Concretely, your Ent client setup simply *omits* the `Schema.Create` call that Ent tutorials show:

```go
package data

import (
	"context"
	"database/sql"

	"entgo.io/ent/dialect"
	entsql "entgo.io/ent/dialect/sql"
	"yourapp/ent"
)

// NewEntClient builds an Ent client over an EXISTING *sql.DB (the same one goose used, §8).
// Crucially: it does NOT call client.Schema.Create — goose owns the schema.
func NewEntClient(sqlDB *sql.DB) *ent.Client {
	// Wrap the shared *sql.DB as an Ent driver. Ent reuses our pgx-backed pool.
	drv := entsql.OpenDB(dialect.Postgres, sqlDB)
	client := ent.NewClient(ent.Driver(drv))

	// NOTE: NO client.Schema.Create(ctx) here. That would let Ent mutate the schema at runtime.
	// The schema already exists because `goose up` ran (as a deploy step) BEFORE this app started.

	return client
}
```

If you want a defensive belt-and-suspenders, some teams gate any accidental `Schema.Create` behind an environment check (`if os.Getenv("ALLOW_ENT_AUTOMIGRATE") == "1"`), so it can be used in a throwaway local test DB but is impossible in prod. Even then, prefer goose everywhere for consistency.

### 9.4 Keeping Ent's schema in sync with goose **[I/A]**

Since goose writes the DDL and Ent needs to *know* that DDL to generate correct code, you must keep the two consistent. Three viable strategies:

1. **Hand-mirror (most common, most control).** When you write a goose migration adding `users.mfa_enabled BOOLEAN NOT NULL DEFAULT false`, you also add the field to `ent/schema/user.go`:
   ```go
   func (User) Fields() []ent.Field {
       return []ent.Field{
           field.String("email").Unique(),
           field.String("password_hash").Sensitive(), // .Sensitive() keeps it out of logs/JSON
           field.Bool("mfa_enabled").Default(false),   // mirrors the goose migration
       }
   }
   ```
   Then `go generate ./ent`. The DDL lives in goose; the *type definition* lives in Ent; you keep them matched by discipline (and CI checks — §10 — that fail if they diverge). This is the recommended default: it's explicit and reviewable on both sides.

2. **Generate Ent *from* the migrated database (`entimport`).** Ariga's `entimport` tool reads an existing database (the one goose migrated) and generates Ent schema files from it. You run goose first, then `entimport` to derive the Ent schema. This makes goose *strictly* the source of truth and Ent purely derived — nice, but `entimport` doesn't capture everything (some constraints, some types) and you often hand-tweak. Good for bootstrapping a large existing schema; verify the output.

3. **Let Ent generate migration *files* (Atlas) instead of goose.** The opposite choice — Ent's Atlas integration diffs your Ent schema and emits versioned SQL files you review and run. This is legitimate but it's *Ent-owns-schema*, not *goose-owns-schema*; it's outside this guide's chosen architecture. Mentioned for completeness — pick one owner, not both.

> **Cross-reference:** For the query side of a goose-owned schema *without* an ORM — hand-written SQL made type-safe by codegen — see [sqlc + goose](GO_SQLC_GOOSE_GUIDE.md). sqlc and goose are natural partners: goose owns the schema, sqlc reads it (or your queries against it) and generates type-safe Go. The choice between "Ent + goose" and "sqlc + goose" is the choice between an ORM and typed hand-SQL; goose's role is identical in both.

### 9.5 Why this division is worth the effort **[A]**

Maintaining Ent's schema *and* goose's migrations feels like duplicated effort — you describe the `mfa_enabled` column twice. It's a real cost, and worth it, because the two describe *different things*: goose describes **how the database changed over time** (the audited, ordered, reviewable delta), while Ent describes **the current shape for type-safe access** (the query builder). A regulated system needs the former as a first-class, immutable artifact — you cannot get "which reviewed change added this column, applied when, by whom" from Ent's current-state model. goose provides exactly that history; Ent provides the ergonomics. The small duplication buys you an audit trail you're often legally required to have.

---

## 10. Production Workflow — Banking-Grade Discipline

This is the section that separates "I can run `goose up`" from "I can own a production schema without causing an outage." Everything here is about *safety under adversarial conditions*: multiple instances, live traffic, large tables, regulatory audit, and the certainty that anything that can go wrong eventually will.

### 10.1 Run migrations as a separate, gated step — not silently on boot **[A]**

We touched this in §7.4; here is the rule and the reasoning. **Migrations must be a deliberate, observable, gated step — not a side effect of a process starting.** Concretely:

- **Deploy pipeline runs migrations explicitly** as its own stage: a Kubernetes `Job` / init container / CI deploy step that runs `myapp migrate up` (or the `cmd/migrate` binary) and must *succeed* before the new app version is rolled out. This step is logged, timed, alertable, and rollback-aware.
- **The app server never migrates.** On startup it only *verifies* the schema is current (`provider.HasPending`) and **refuses to start** if it isn't (§7.3). This decouples "change the schema" from "start serving," so you can migrate in a maintenance window, migrate ahead of a deploy, or hold a deploy while a long migration runs — none of which is possible if migration is glued to boot.

Why so strict? Because a schema change is the one operation that can take a table-level lock, run for minutes, and be impossible to cleanly abort. You want a human (or a well-understood pipeline) to *decide* when that happens — not have it triggered by an autoscaler adding a pod at peak traffic.

### 10.2 Postgres advisory locks — defeating split-brain **[A]**

Even with a dedicated migrate step, you can end up with *two* migration runners at once — two CI jobs, a retry overlapping the original, or (in the gated-boot pattern) two app instances booting together. Two processes running `goose up` concurrently against one database is **split-brain**: both read "version 10 is current," both try to apply migration 11, and you get duplicate-object errors at best, corruption or a half-applied change at worst.

PostgreSQL's **advisory locks** are the fix. An advisory lock is an application-defined, database-wide mutex: `pg_advisory_lock(key)` blocks until this session holds the lock for `key`; only one session can hold a given key at a time. Wrap your migration run so that a runner **acquires the lock, migrates, releases** — any second runner blocks at the lock until the first finishes, then finds nothing pending and proceeds harmlessly.

```go
package db

import (
	"context"
	"database/sql"
	"fmt"

	"github.com/pressly/goose/v3"
)

// A fixed, arbitrary 64-bit key unique to THIS app's migrations. Pick once; keep forever.
// (Any constant works, as long as every migrator uses the same one.)
const migrationLockKey int64 = 0x6D6967726174696F // "migratio" as bytes, for memorability

// MigrateWithLock acquires a Postgres advisory lock so exactly one migrator runs at a time.
func MigrateWithLock(ctx context.Context, sqlDB *sql.DB, provider *goose.Provider) error {
	// Grab a SINGLE connection and hold it for the whole operation — session-level advisory
	// locks are tied to the connection that took them. Using the pool directly would risk
	// lock and unlock landing on different connections.
	conn, err := sqlDB.Conn(ctx)
	if err != nil {
		return err
	}
	defer conn.Close()

	// Block until we hold the lock. A concurrent migrator waits here instead of racing us.
	if _, err := conn.ExecContext(ctx, `SELECT pg_advisory_lock($1)`, migrationLockKey); err != nil {
		return fmt.Errorf("acquire migration lock: %w", err)
	}
	// ALWAYS release — even on panic/error — so a crash doesn't wedge future deploys.
	defer func() {
		_, _ = conn.ExecContext(context.WithoutCancel(ctx),
			`SELECT pg_advisory_unlock($1)`, migrationLockKey)
	}()

	// Now, holding the lock, run the migrations. The second runner is still blocked above;
	// when we release, it acquires, sees nothing pending, and returns cleanly.
	if _, err := provider.Up(ctx); err != nil {
		return fmt.Errorf("run migrations: %w", err)
	}
	return nil
}
```

Notes that matter: use a **single connection** (`sqlDB.Conn`) for the lock+migrate+unlock, since session advisory locks are connection-scoped. Prefer `pg_advisory_lock` (blocking, waits) over `pg_try_advisory_lock` (returns immediately) when you *want* the second runner to wait rather than fail — usually the case for deploys. And always release in a `defer` using a non-cancellable context so a cancelled parent context doesn't skip the unlock and wedge every future migration.

> **⚡ goose can lock for you.** Recent goose versions include session-locking support via provider options / a `lock` package (e.g. a Postgres session locker you attach so `Up` takes the advisory lock itself). If your version offers it, prefer the built-in locker over hand-rolling — but the manual version above is portable, explicit, and shows exactly what's happening. Check pkg.go.dev for `WithSessionLocker` / `lock.NewPostgresSessionLocker` in your version.

### 10.3 CI checks for migrations **[A]**

Migrations are code; they get CI. A banking-grade pipeline enforces at least:

1. **`goose validate`** — every migration file parses and is structurally sound. Fails the PR on a malformed file *before* it can reach any database.
2. **Apply-from-empty on a throwaway DB** — spin up a scratch PostgreSQL (a CI service container), run `goose up` from zero, assert it succeeds. This proves the full migration chain applies cleanly on a fresh database (catches a migration that only worked because your dev DB happened to already have something).
3. **Status must be clean after up** — after `up`, `goose status` shows no pending, and (optionally) `down` then `up` round-trips, proving your Down migrations actually work.
4. **No editing already-applied migrations** — a check that migration files present in the base branch are byte-identical in the PR (you may *add* files, never *modify* an existing one). This mechanically enforces the cardinal rule of §12.
5. **Sequentially numbered** — assert filenames are the `NNNNN_` sequential form (i.e. `goose fix` was run), catching branch-ordering hazards.
6. **Ent-in-sync** (if using Ent) — run `go generate ./ent` and fail if it produces a diff, proving the Ent schema matches what's checked in (and, by §9 discipline, the goose DDL).

A sketch of the GitHub Actions job (see [GitHub Actions CI/CD] patterns in your CI guide):

```yaml
migrations:
  runs-on: ubuntu-latest
  services:
    postgres:
      image: postgres:17
      env: { POSTGRES_PASSWORD: ci, POSTGRES_DB: appdb }
      ports: ["5432:5432"]
      options: >-
        --health-cmd "pg_isready -U postgres" --health-interval 5s
        --health-timeout 5s --health-retries 5
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-go@v5
      with: { go-version: "1.26" }
    - name: Install goose
      run: go install github.com/pressly/goose/v3/cmd/goose@v3.27.0
    - name: Validate migration files (no DB)
      run: goose -dir ./migrations validate
    - name: Apply from empty
      env:
        GOOSE_DRIVER: postgres
        GOOSE_DBSTRING: postgres://postgres:ci@localhost:5432/appdb?sslmode=disable
        GOOSE_MIGRATION_DIR: ./migrations
      run: |
        goose up
        goose status
        # Assert nothing pending (grep for the absence of "Pending" — tune to your output).
    - name: Reject edits to applied migrations
      run: |
        # Fail if any migration file that exists on main was modified in this PR.
        git fetch origin main
        if git diff --name-status origin/main -- migrations/ | grep -E '^M'; then
          echo "ERROR: an existing migration was modified. Add a NEW migration instead."; exit 1
        fi
```

### 10.4 Zero-downtime & expand-contract **[A]**

The hardest migrations are the ones that run against a **live** database serving traffic, where the old and new application code both run simultaneously during a rolling deploy. The governing pattern is **expand-contract** (a.k.a. parallel-change): never change a thing in place; instead *add* the new thing, *migrate* to it, then *remove* the old thing — across multiple deploys so that at every instant the schema is compatible with *both* the currently-running old code and the incoming new code.

The canonical example — you want to make a new `NOT NULL` column `password_algo`:

| Step | Migration | App code | Why it's safe |
|---|---|---|---|
| **Expand** | Add `password_algo TEXT` **nullable** | (old code still runs, ignores it) | Adding a nullable column takes a trivial lock, doesn't rewrite the table, and old code doesn't care. |
| **Backfill** | Go/SQL migration fills `password_algo` in **batches** | (deploy new code that *writes* the column going forward) | Batched backfill holds only brief locks; new writes populate it; nulls shrink to zero. |
| **Constrain** | `ALTER TABLE ... SET NOT NULL` (after verifying no nulls) | new code relies on it | Once every row has a value, adding NOT NULL is validated cheaply (esp. with a prior `CHECK ... NOT VALID` + `VALIDATE`). |
| **Contract** | Drop the *old* column/table this replaced (a later deploy) | old code fully gone | Only drop after no running code references the old thing. |

The ironclad rules that fall out of this:

- **Never rename a column in place.** A rename is instantly incompatible with old code still reading the old name. Instead: add the new column, backfill, dual-write, switch reads, drop old — expand-contract. Renames are the classic zero-downtime killer.
- **Add columns nullable (or with a default carefully).** In modern PostgreSQL, `ADD COLUMN ... DEFAULT <const>` is fast (metadata-only) for constant defaults, but a *volatile* default rewrites the table. When unsure, add nullable, backfill, then set the default/constraint.
- **Build indexes `CONCURRENTLY`** (with `-- +goose NO TRANSACTION`, §3.3) so index creation never blocks writes on a hot table.
- **Avoid long-held locks and table rewrites.** Know which `ALTER`s rewrite the table (changing a column type, some default changes) or take `ACCESS EXCLUSIVE`. On a large table, prefer the additive path: new column of the new type, backfill, swap.
- **Batch backfills** (§5.3) so you never wrap millions of rows in one lock-holding transaction.
- **Add constraints in two phases.** `ADD CONSTRAINT ... CHECK (...) NOT VALID` (fast, takes a brief lock, enforces for *new* rows) then, in a separate migration, `VALIDATE CONSTRAINT` (scans existing rows without an exclusive lock). Same two-phase idea applies to foreign keys.

### 10.5 Roll forward, not back **[A]**

We keep returning to this because it's the philosophical core. In production you **do not run `goose down` to fix a bad migration.** Reasons: a Down that drops a column destroys data written to it since the Up; a Down assumes the schema is exactly the shape the Up left it, which may not hold after other changes; and a Down is itself an un-reviewed-in-context operation run under incident pressure. Instead you **roll forward**: write a *new* migration that corrects the problem (drop the bad index, fix the constraint, migrate the mangled data back), review it, and apply it as the next version. Your history stays append-only and every change stays audited.

This is *why* teams sometimes deliberately leave the Down bodies of destructive migrations empty or `SELECT 'no automatic rollback';` — to make an accidental `goose down` in production a no-op rather than a data-loss event. Write real Downs for *dev iteration* value where they're safe; for genuinely destructive prod migrations, consider making the Down refuse to run.

### 10.6 The other non-negotiables **[A]**

- **Back up before destructive changes.** Before any migration that drops or rewrites data, take a verified backup / snapshot (see [Database Server Admin](DATABASE_SERVER_ADMIN_GUIDE.md)). A `DROP TABLE` with a fresh, restore-tested backup is recoverable; without one it's an incident.
- **Never put secrets or PII in migration files.** Migrations are committed to Git and live forever in history. No passwords, API keys, real customer data, or PII in migration SQL — not even in a data-seeding migration. Seed reference data (country codes, roles), never personal data. Reference secrets via runtime config, never bake them into a migration.
- **Migrations are audited artifacts.** Each migration is reviewed (PR approval), immutable once applied, timestamped in `goose_db_version`, and traceable to an author and a review. In a regulated environment that audit trail is a compliance requirement, and it's *free* if you follow the discipline above — which is the whole point.
- **Test Downs in dev/CI, distrust them in prod.** A Down you never ran is a lie. Round-trip them in CI (`up` → `down` → `up`). But per §10.5, prefer roll-forward in production regardless.

---

## 11. The Full Stack: Gin + Ent + pgx + JWT + Argon2 + Air

Now we assemble everything into a coherent production-shaped project where **goose owns the schema** for a real Gin API using Ent (over pgx), JWT auth, and Argon2id password hashing — with Air running migrations before the server in development and a dedicated migrate step in production.

### 11.1 Project layout **[I]**

```text
myapp/
├── cmd/
│   ├── server/main.go        # the Gin API server (verifies schema is current; never migrates)
│   └── migrate/main.go       # the DEDICATED migrate binary (runs goose; the deploy step calls this)
├── migrations/
│   ├── embed.go              # //go:embed *.sql -> embed.FS
│   ├── 00001_extensions.sql
│   ├── 00002_create_users.sql
│   ├── 00003_users_updated_at_trigger.sql
│   ├── 00004_create_sessions.sql
│   ├── 00005_create_accounts.sql
│   └── 00006_sessions_active_index.sql   # CONCURRENTLY, NO TRANSACTION
├── ent/                      # Ent schema + generated code (mirrors goose DDL; never Schema.Create in prod)
│   └── schema/...
├── internal/
│   ├── db/db.go              # pgxpool + sql.DB bridge + goose provider + advisory-lock runner
│   └── auth/...              # Argon2id + JWT (see GO_JWT_ARGON2_GUIDE.md)
├── .air.toml                 # dev hot-reload: migrate THEN run server
├── go.mod
└── go.sum
```

The two-binary split (`cmd/server`, `cmd/migrate`) is the §10.1 architecture in file form: the server serves, the migrate binary migrates, and they never overlap responsibilities.

### 11.2 The migration set — a coherent worked schema **[I]**

Six migrations that build a real auth-plus-domain schema. Read them in order; each is one logical change.

`migrations/00001_extensions.sql` — extensions first, because later migrations depend on them:

```sql
-- +goose Up
CREATE EXTENSION IF NOT EXISTS citext;    -- case-insensitive email column type
CREATE EXTENSION IF NOT EXISTS pgcrypto;  -- gen_random_uuid() for session/account IDs

-- +goose Down
DROP EXTENSION IF EXISTS pgcrypto;
DROP EXTENSION IF EXISTS citext;
```

`migrations/00002_create_users.sql` — the users table with the Argon2id hash column:

```sql
-- +goose Up
CREATE TABLE users (
	id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
	email         CITEXT NOT NULL UNIQUE,        -- unique, case-insensitive
	password_hash TEXT   NOT NULL,               -- Argon2id PHC string, e.g. $argon2id$v=19$m=65536,t=3,p=2$...
	role          TEXT   NOT NULL DEFAULT 'user' -- 'user' | 'admin'; enforced by CHECK below
		CHECK (role IN ('user', 'admin')),
	mfa_enabled   BOOLEAN NOT NULL DEFAULT false,
	created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
	updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- +goose Down
DROP TABLE users;
```

`migrations/00003_users_updated_at_trigger.sql` — the trigger from §3.2 (needs `StatementBegin/End`):

```sql
-- +goose Up
-- +goose StatementBegin
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
	NEW.updated_at = now();
	RETURN NEW;
END;
$$ LANGUAGE plpgsql;
-- +goose StatementEnd

CREATE TRIGGER users_set_updated_at
	BEFORE UPDATE ON users
	FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- +goose Down
DROP TRIGGER IF EXISTS users_set_updated_at ON users;
DROP FUNCTION IF EXISTS set_updated_at();
```

`migrations/00004_create_sessions.sql` — refresh-token sessions for the JWT flow:

```sql
-- +goose Up
CREATE TABLE sessions (
	id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
	user_id      BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
	refresh_hash TEXT   NOT NULL,               -- SHA-256 of the refresh token; never the token itself
	user_agent   TEXT,
	ip           INET,
	expires_at   TIMESTAMPTZ NOT NULL,
	revoked_at   TIMESTAMPTZ,
	created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
	CONSTRAINT sessions_expires_after_created CHECK (expires_at > created_at)
);
CREATE INDEX idx_sessions_user_id ON sessions (user_id);

-- +goose Down
DROP TABLE sessions;
```

`migrations/00005_create_accounts.sql` — a domain table (banking-flavored) referencing users:

```sql
-- +goose Up
CREATE TABLE accounts (
	id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
	owner_id     BIGINT NOT NULL REFERENCES users(id) ON DELETE RESTRICT, -- keep accounts if user deletion attempted
	currency     CHAR(3) NOT NULL,                     -- ISO 4217, e.g. 'USD'
	-- Money as an integer of minor units (cents) — NEVER float for money.
	balance_cents BIGINT NOT NULL DEFAULT 0 CHECK (balance_cents >= 0),
	created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
	updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_accounts_owner_id ON accounts (owner_id);

-- +goose Down
DROP TABLE accounts;
```

`migrations/00006_sessions_active_index.sql` — a concurrent index, demonstrating `NO TRANSACTION`:

```sql
-- +goose NO TRANSACTION
-- +goose Up
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_sessions_active
	ON sessions (user_id)
	WHERE revoked_at IS NULL;   -- partial index over active sessions only

-- +goose Down
DROP INDEX CONCURRENTLY IF EXISTS idx_sessions_active;
```

### 11.3 Embedding the migrations **[I]**

`migrations/embed.go`:

```go
package migrations

import "embed"

//go:embed *.sql
var Embedded embed.FS
```

### 11.4 The db package: pool, bridge, provider, locked runner **[I/A]**

`internal/db/db.go` ties §8 and §10 together:

```go
package db

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"log/slog"

	"entgo.io/ent/dialect"
	entsql "entgo.io/ent/dialect/sql"
	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/jackc/pgx/v5/stdlib"
	"github.com/pressly/goose/v3"

	"yourapp/ent"
	"yourapp/migrations"
)

const migrationLockKey int64 = 0x6D6967726174696F

type DB struct {
	Pool  *pgxpool.Pool
	SQL   *sql.DB
	Ent   *ent.Client
	goose *goose.Provider
}

// Open builds the shared pool, the database/sql bridge, the Ent client (NO Schema.Create),
// and the goose provider over embedded migrations.
func Open(ctx context.Context, dsn string, logger *slog.Logger) (*DB, error) {
	cfg, err := pgxpool.ParseConfig(dsn)
	if err != nil {
		return nil, fmt.Errorf("parse dsn: %w", err)
	}
	cfg.MaxConns = 10

	pool, err := pgxpool.NewWithConfig(ctx, cfg)
	if err != nil {
		return nil, fmt.Errorf("open pool: %w", err)
	}
	if err := pool.Ping(ctx); err != nil {
		pool.Close()
		return nil, fmt.Errorf("ping: %w", err)
	}

	// One pool, two consumers: Ent (queries) and goose (migrations), via the stdlib bridge.
	sqlDB := stdlib.OpenDBFromPool(pool)

	// Ent over the shared *sql.DB — NO Schema.Create anywhere (goose owns the schema).
	entClient := ent.NewClient(ent.Driver(entsql.OpenDB(dialect.Postgres, sqlDB)))

	provider, err := goose.NewProvider(
		goose.DialectPostgres,
		sqlDB,
		migrations.Embedded,
		goose.WithSlog(logger),
		goose.WithVerbose(true),
	)
	if err != nil {
		pool.Close()
		return nil, fmt.Errorf("goose provider: %w", err)
	}

	return &DB{Pool: pool, SQL: sqlDB, Ent: entClient, goose: provider}, nil
}

// MigrateUp runs all pending migrations under an advisory lock (§10.2).
func (d *DB) MigrateUp(ctx context.Context) error {
	conn, err := d.SQL.Conn(ctx)
	if err != nil {
		return err
	}
	defer conn.Close()

	if _, err := conn.ExecContext(ctx, `SELECT pg_advisory_lock($1)`, migrationLockKey); err != nil {
		return fmt.Errorf("acquire migration lock: %w", err)
	}
	defer func() {
		_, _ = conn.ExecContext(context.WithoutCancel(ctx), `SELECT pg_advisory_unlock($1)`, migrationLockKey)
	}()

	results, err := d.goose.Up(ctx)
	if err != nil {
		return fmt.Errorf("goose up: %w", err)
	}
	for _, r := range results {
		slog.Info("migrated", "version", r.Source.Version, "path", r.Source.Path, "dur", r.Duration)
	}
	return nil
}

// AssertCurrent refuses to proceed if the schema is stale — the server calls this on boot.
func (d *DB) AssertCurrent(ctx context.Context) error {
	pending, err := d.goose.HasPending(ctx)
	if err != nil {
		return err
	}
	if pending {
		return errors.New("schema not current: unapplied migrations exist; run the migrate step first")
	}
	return nil
}

func (d *DB) Close() { d.Pool.Close() }
```

### 11.5 The dedicated migrate binary **[I/A]**

`cmd/migrate/main.go` — a tiny CLI the deploy pipeline (or `air`) invokes. It supports `up`, `status`, and `version` so ops has what it needs:

```go
package main

import (
	"context"
	"log/slog"
	"os"

	"yourapp/internal/db"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
	ctx := context.Background()

	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		logger.Error("DATABASE_URL not set")
		os.Exit(1)
	}

	database, err := db.Open(ctx, dsn, logger)
	if err != nil {
		logger.Error("open db", "err", err)
		os.Exit(1)
	}
	defer database.Close()

	cmd := "up"
	if len(os.Args) > 1 {
		cmd = os.Args[1]
	}

	switch cmd {
	case "up":
		if err := database.MigrateUp(ctx); err != nil {
			logger.Error("migrate up", "err", err)
			os.Exit(1)
		}
		logger.Info("migrations applied")
	default:
		logger.Error("unknown command", "cmd", cmd)
		os.Exit(2)
	}
}
```

Run it locally or as a deploy step:

```bash
DATABASE_URL='postgres://appuser:secret@localhost:5432/appdb?sslmode=require' go run ./cmd/migrate up
```

### 11.6 The server binary — verify, never migrate **[I]**

`cmd/server/main.go` boots Gin but only *checks* the schema:

```go
package main

import (
	"context"
	"log/slog"
	"net/http"
	"os"

	"github.com/gin-gonic/gin"
	"yourapp/internal/db"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
	ctx := context.Background()

	database, err := db.Open(ctx, os.Getenv("DATABASE_URL"), logger)
	if err != nil {
		logger.Error("open db", "err", err)
		os.Exit(1)
	}
	defer database.Close()

	// Refuse to serve on a stale schema. The migrate STEP must have run first (§10.1).
	if err := database.AssertCurrent(ctx); err != nil {
		logger.Error("schema check failed", "err", err)
		os.Exit(1)
	}

	r := gin.New()
	r.Use(gin.Recovery())
	// ... register /auth/register, /auth/login (Argon2id + JWT), /accounts handlers using database.Ent ...
	r.GET("/healthz", func(c *gin.Context) { c.JSON(http.StatusOK, gin.H{"status": "ok"}) })

	logger.Info("listening", "addr", ":8080")
	_ = r.Run(":8080")
}
```

The auth handlers (register hashing with Argon2id, login issuing JWTs, refresh via the `sessions` table) are the subject of [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md); here the point is that they query through `database.Ent` over the schema **goose** built.

### 11.7 Air: migrate before serving in dev **[I]**

Air hot-reloads your server on file changes. You want *migrations to run before the server starts* in development, so a schema change you just wrote is applied automatically. Configure `.air.toml` to build both binaries and run migrate then serve via a small script, or use Air's pre-run hook:

`.air.toml`:

```toml
root = "."
tmp_dir = "tmp"

[build]
  # Build the server binary that Air will run.
  cmd = "go build -o ./tmp/server ./cmd/server"
  bin = "./tmp/server"
  # Run the migrate binary BEFORE (re)starting the server, so new migrations apply on reload.
  # full_bin runs this instead of bin; we chain migrate then server.
  full_bin = "sh -c 'go run ./cmd/migrate up && ./tmp/server'"
  include_ext = ["go", "sql", "toml"]   # reload on .sql too — new migrations trigger a rebuild
  exclude_dir = ["tmp", "ent/gen"]
  delay = 200

[log]
  time = true
```

> **Windows note:** the `sh -c '...'` form assumes a POSIX shell (Git Bash / WSL). On plain PowerShell, use `full_bin = "cmd /c \"go run ./cmd/migrate up && tmp\\server.exe\""` or a small `.ps1` wrapper. The concept is identical: **migrate, then serve.** Including `sql` in `include_ext` means adding a new migration file triggers a reload that applies it — a smooth dev loop.

### 11.8 The production deploy sequence **[A]**

Putting §10.1 into a concrete order for a rolling deploy:

1. **CI** runs `goose validate`, applies migrations from empty on a scratch DB, checks status clean, verifies Ent is in sync, and rejects edits to applied migrations (§10.3).
2. **Deploy pipeline** runs the migrate step: a Job / init-container executes `migrate up` (which takes the advisory lock, applies pending migrations, releases). This must **succeed** before proceeding. Migrations were authored expand-contract, so the new schema is compatible with the *currently running* old code.
3. **Rollout** starts new app pods. Each calls `AssertCurrent` on boot and starts serving; old pods drain. Because the schema was already migrated (compatibly), old and new pods coexist safely during the rollout.
4. **Later deploy** ships the contract step (drop old columns) once no old code remains.

That sequence — CI gate, gated migrate step, schema-current check, expand-contract compatibility — is the whole banking-grade discipline made operational.

---

## 12. Gotchas & Best Practices

A dedicated catalog of the mistakes that bite, each with the rule that prevents it.

### 12.1 Never edit an already-applied migration **[A]**

**The cardinal rule.** Once a migration has been applied *anywhere it matters* (a teammate's DB, staging, prod, CI's baseline), it is **immutable**. If you edit its SQL, databases that already applied it will *not* re-run it (goose sees the version as done) — so they keep the *old* behavior — while fresh databases run the *new* version. Instant, silent drift between environments. Worse, goose may detect the change as a checksum/version mismatch and error, or (if it doesn't) you get two divergent schemas with the same version number. **The rule: to change something a migration did, write a NEW migration.** The old one is history; you append a correction. Enforce it in CI (§10.3, the "reject edits" check).

### 12.2 Timestamp vs sequential ordering across branches **[A]**

Two branches each `goose create` a migration; both get timestamps. When both merge, their relative order is by timestamp, which may not match merge order or intended dependency order — and if one depends on the other, you can get a migration that runs before the table it needs exists. **Rule: run `goose fix` to convert to sequential numbers before merging, coordinate migration numbering across in-flight branches, and let CI assert sequential numbering.** For the rare legitimate late-arriving lower migration, `-allow-missing` / `WithAllowMissing()` exists (§4.6) — but treat needing it as a smell.

### 12.3 Forgetting `StatementBegin/End` around functions **[I]**

Symptom: a `CREATE FUNCTION`/`CREATE TRIGGER`/`DO $$...$$` fails with a syntax error somewhere in the *middle* of the body. Cause: goose split the statement on an interior semicolon. **Rule: wrap any statement containing interior semicolons (PL/pgSQL bodies, dollar-quoted blocks) in `-- +goose StatementBegin` / `-- +goose StatementEnd`** (§3.2). Plain DDL doesn't need it.

### 12.4 `NO TRANSACTION` caveats **[A]**

`-- +goose NO TRANSACTION` is required for `CREATE INDEX CONCURRENTLY` and other non-transactional DDL, but it forfeits the all-or-nothing rollback (§3.3). **Rules:** keep such files to a *single* statement so there's no partial-apply ambiguity; make the statement idempotent (`IF NOT EXISTS` / drop-invalid-first) so a retry after failure is clean; and remember a failed `CONCURRENTLY` leaves an *invalid* index you must drop manually. Never put `NO TRANSACTION` on a multi-statement migration unless every statement is independently safe.

### 12.5 Down migrations that silently drop data **[A]**

A Down that `DROP COLUMN`s or `DROP TABLE`s destroys data written since the Up. Running `goose down` (or `down-to`, or `reset`) in production can therefore silently lose data. **Rules:** roll *forward* in prod, not back (§10.5); consider making Downs of destructive migrations a no-op/refusal so an accidental `down` can't nuke data; back up before anything destructive; and never rely on Down as a prod safety net — it's a dev-iteration convenience.

### 12.6 Running `reset` in production **[A]**

`goose reset` runs every Down back to version 0 — it *drops your entire schema and data*. There is no legitimate production use. **Rule: `reset` is dev-only.** Guard against it operationally: the prod migrate binary in §11 doesn't even expose a `reset` command. Don't hand a `reset`-capable tool to production credentials.

### 12.7 Mixing dialects / DSN mistakes **[I]**

Pointing goose at the wrong dialect (or a MySQL DSN with `postgres` driver) produces confusing errors, and worse, migrations written for one dialect's SQL won't run on another. **Rules:** set the dialect once, correctly (`goose.DialectPostgres` / `SetDialect("postgres")`); keep migrations dialect-specific and don't try to share a migrations dir across Postgres and MySQL; and store the DSN in `GOOSE_DBSTRING`/env, never hardcoded, so dev/staging/prod differ only by environment.

### 12.8 Checksum / version drift **[A]**

If goose (or your CI) reports a version/checksum mismatch, it means a migration file's content changed after it was applied, or two files share a version number, or migrations were applied out of order. **Rule: the migrations directory is append-only and sequentially numbered; never renumber, never edit applied files, never duplicate a version.** The CI checks in §10.3 catch all three before they reach a database.

### 12.9 Transaction-per-migration behavior **[I]**

On PostgreSQL, each migration file runs in a transaction by default, so a *failed* migration rolls back completely — your schema is untouched and you just fix-and-rerun (§3.5). Don't be surprised that a partially-written failing migration left *nothing* behind; that's the feature. The exceptions are `NO TRANSACTION` files and MySQL (non-transactional DDL), where a failure can leave a partial state you must reconcile by hand.

### 12.10 Go migrations and the generic CLI **[I]**

The pre-built `goose` CLI cannot run your `.go` migrations — those only exist inside a binary that imports your migrations package (§5.4). **Rule: if you use Go migrations, run migrations through your own `cmd/migrate` binary, not the generic CLI.** Otherwise your Go migrations silently don't run and your schema is subtly incomplete.

### 12.11 Best-practice checklist **[A]**

| Do | Don't |
|---|---|
| One logical change per migration | Cram a release into one giant file |
| Write and CI-test Down migrations | Trust Downs as a prod undo button |
| Append new migrations; keep the dir immutable | Edit or renumber applied migrations |
| `goose fix` before merge; sequential numbers | Ship timestamped, branch-ordered migrations |
| `NO TRANSACTION` + single statement for `CONCURRENTLY` | Put `CONCURRENTLY` in a transactional/multi-stmt file |
| Wrap function/trigger bodies in `StatementBegin/End` | Let goose split PL/pgSQL on interior `;` |
| Run migrations as a gated deploy step, advisory-locked | Auto-migrate on every app boot, unlocked |
| Verify schema-current on server boot, refuse if stale | Serve traffic against an unmigrated schema |
| Expand-contract; add nullable, backfill, constrain | Rename/alter in place on a live table |
| Roll forward to fix; back up before destructive DDL | `goose down`/`reset` in production |
| Keep goose the single schema owner; Ent read-only | Call `ent.Schema.Create` in production |
| Store DSN in env; keep secrets/PII out of migrations | Hardcode credentials or seed PII |

---

## 13. Study Path & Build-to-Learn Projects

### 13.1 Suggested study path

Work these in order; each builds on the previous.

1. **[B] The loop.** Install the CLI (§2), create a `users` migration, `up`, `status`, inspect `goose_db_version` in `psql`, `down`, `up` again. Do it until the create → up → status → down cycle is muscle memory and the version table holds no mystery.
2. **[B/I] SQL depth.** Author the §11 migration set by hand: extensions, users, the `updated_at` trigger (feel the `StatementBegin/End` requirement by omitting it once and reading the error), sessions with an FK, a `CONCURRENTLY` index with `NO TRANSACTION`. Run `goose fix`.
3. **[B/I] Commands.** Practice `up-by-one`, `up-to`, `down-to`, `redo`, `validate`, `version`. Deliberately break a migration and watch PostgreSQL's transactional DDL roll it back cleanly.
4. **[I] Go migrations.** Add a `password_algo` column (SQL), backfill it by parsing the hash prefix (Go migration, §5.2), then `SET NOT NULL` (SQL) — the expand→backfill→constrain trio. Then convert the backfill to a batched `NoTx` version (§5.3).
5. **[I] Embed + provider.** Move to `embed.FS` (§6) and rewrite your runner with `NewProvider`, `WithSlog`, `HasPending` (§7). Compare against the old global `goose.Up`.
6. **[I/A] pgx + Ent.** Share one `pgxpool.Pool` via `stdlib.OpenDBFromPool` (§8), build an Ent client over it with **no** `Schema.Create` (§9), and mirror a new goose column into the Ent schema + regenerate.
7. **[A] Production hardening.** Add the advisory-locked runner (§10.2), the `cmd/migrate` binary, the server's `AssertCurrent` boot check, and the CI job (§10.3). Simulate split-brain by running two migrators at once and confirm one waits.
8. **[A] Zero-downtime.** Do a real expand-contract rename against a table with data: add new column, dual-write, backfill in batches, switch reads, drop old — across "two deploys," verifying compatibility at each step (§10.4).

### 13.2 Build-to-learn projects

- **Project 1 — "Migrate a real schema from zero" [B/I].** Take the §11 six-migration auth+accounts schema and stand it up on a fresh PostgreSQL from empty using only `goose up`. Add three more migrations evolving it (add an `audit_log` table, a partial index, an MFA-secrets table), running `goose fix` before each "merge." Success = a teammate can `git clone` and `goose up` to an identical schema.

- **Project 2 — "The migrate service" [I/A].** Build the full two-binary layout (`cmd/server`, `cmd/migrate`) with `embed.FS`, `NewProvider`, the advisory-locked runner, and a server that refuses to boot on a stale schema. Wire Air to migrate-then-serve in dev. Success = adding a `.sql` file and saving triggers a reload that applies it, and starting the server against an unmigrated DB fails loudly.

- **Project 3 — "goose + Ent, one schema owner" [I/A].** Layer Ent over the goose-owned schema: mirror every goose table into `ent/schema/`, generate the type-safe client, build a Gin handler that creates a user (Argon2id hash) and an account through Ent — with `Schema.Create` nowhere in the codebase. Add a CI check that fails if `go generate ./ent` produces a diff. Success = the DB shape comes 100% from goose, the query layer 100% from Ent, and CI proves they agree.

- **Project 4 — "Zero-downtime rename under load" [A].** With a table holding a million rows and a script writing to it continuously, rename a column with **no failed writes**: expand (add new column), deploy dual-writing code, batched backfill, switch reads, contract (drop old) — each as its own migration/deploy. Success = the write script never errors across the entire multi-step migration.

- **Project 5 — "Break it and recover" [A].** Deliberately create incidents and practice roll-forward: apply a migration that adds a bad constraint, then *fix it with a new migration* (not `down`). Apply a `CONCURRENTLY` index that fails partway (unique violation), find the invalid index, and clean up. Take a backup, `DROP` a table, restore from backup. Success = you can recover from each without ever running `goose down`/`reset` in the "prod" DB.

### 13.3 Cross-references

- [Go ent ORM](GO_ENT_ORM_GUIDE.md) — the type-safe query layer that sits on top of the goose-owned schema; §9 defines the boundary.
- [sqlc + goose](GO_SQLC_GOOSE_GUIDE.md) — the hand-SQL, codegen counterpart to Ent; goose plays the identical schema-owner role.
- [PostgreSQL](POSTGRESQL_GUIDE.md) — transactional DDL, lock levels, `CONCURRENTLY`, `CHECK ... NOT VALID`/`VALIDATE`; the semantics your migrations exploit.
- [Go Gin REST API + File Upload](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) — the API server the §11 example wires migrations into.
- [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md) — the auth flow using the `users`/`sessions` tables goose creates here.
- [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md) — normalization, keys, indexes: the theory your migration DDL encodes.
- [Database Server Admin](DATABASE_SERVER_ADMIN_GUIDE.md) — backups, restores, roles; the operational safety net around destructive migrations.

---

*End of guide. goose's power is not in its command list — it is in the discipline it lets you impose: schema as reviewed, ordered, immutable, audited code. Master the discipline (§10, §12) and the commands become trivia. Own your schema; do not let it own you.*
