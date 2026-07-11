# sqlc + goose — Type-Safe SQL & Migrations, Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I have never used a code generator or a migration tool in Go" to "I can evolve a production Postgres schema with goose, generate a fully type-safe Go data layer from hand-written SQL with sqlc, run atomic money transfers inside transactions, lint my queries against a real database in CI, and coexist sanely with an ORM" — entirely **offline**. Every concept is explained in **prose first** (what it is, *why* it works that way, when and how to use it, the key parameters, best practices, and the security/injection angle), and only **then** demonstrated with heavily commented, runnable-looking code. Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **sqlc v1.31+** and **goose v3.27+** (both current in 2026), talking to **PostgreSQL** through **pgx v5**, on **Go 1.25 / 1.26**. It shows how sqlc and goose slot into a real production stack alongside **Gin v1.10+**, **golang-jwt/jwt v5**, **Argon2id** (`golang.org/x/crypto/argon2`), **Ent v0.14+**, and **Air** (live reload). The core ideas — *SQL is the source of truth*, *migrations own schema evolution*, *code generation gives you compile-time safety* — have been stable for years. Where a fast-moving API or flag differs across releases, the section is flagged with **⚡**. Always cross-check **docs.sqlc.dev** and **github.com/pressly/goose** for the very latest. The author is on **Windows 11**, so cross-platform notes (path separators, shells, the `sqlc` binary) are called out where relevant.
>
> **This guide's place in the library:** sqlc and goose sit *between* your Go program and PostgreSQL. For the engine itself read **[PostgreSQL](POSTGRESQL_GUIDE.md)**; for the theory the schema encodes (keys, normalization, indexes) read **[Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md)**; for the migration tool in depth read **[Goose migrations](GO_GOOSE_MIGRATIONS_GUIDE.md)**; for the ORM this guide coexists with read **[Go ent ORM](GO_ENT_ORM_GUIDE.md)**; for the HTTP layer read **[Go Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md)**; and for the auth used in the worked example read **[Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md)**. This guide assumes you can read Go and write basic SQL. It teaches the *toolchain*, not the language or the query language from scratch.

---

## Table of Contents

1. [Philosophy: Three Ways to Talk to Postgres in Go](#1-philosophy-three-ways-to-talk-to-postgres-in-go) **[B]**
2. [Why Pair sqlc WITH goose — One Source of Truth](#2-why-pair-sqlc-with-goose--one-source-of-truth) **[B]**
3. [Install & Project Layout — First End-to-End Loop](#3-install--project-layout--first-end-to-end-loop) **[B]**
4. [`sqlc.yaml` in Depth — Every Option That Matters](#4-sqlcyaml-in-depth--every-option-that-matters) **[B/I]**
5. [Writing Queries: Annotations, Params, Joins, Pagination, Upserts](#5-writing-queries-annotations-params-joins-pagination-upserts) **[I]**
6. [The Generated Code Explained](#6-the-generated-code-explained) **[I]**
7. [Transactions with sqlc + pgx](#7-transactions-with-sqlc--pgx) **[I/A]**
8. [The Development Loop & CI: `generate`, `vet`, `diff`](#8-the-development-loop--ci-generate-vet-diff) **[I/A]**
9. [Coexisting with Ent — The Polyglot Data Layer](#9-coexisting-with-ent--the-polyglot-data-layer) **[A]**
10. [Banking-Grade with sqlc](#10-banking-grade-with-sqlc) **[A]**
11. [Complete Worked Example: an Accounts + Ledger Service](#11-complete-worked-example-an-accounts--ledger-service) **[I/A]**
12. [Gotchas & Best Practices](#12-gotchas--best-practices) **[A]**
13. [Study Path & Build-to-Learn Projects](#13-study-path--build-to-learn-projects)

---

## 1. Philosophy: Three Ways to Talk to Postgres in Go

### 1.1 The problem, stated plainly **[B]**

A relational database speaks **SQL** and stores **rows** in tables. A Go program speaks **structs**, **slices**, and **methods**. Every database-backed Go application therefore contains, somewhere, a *translation layer*: something that turns a `SELECT` row into a `struct`, a `struct` into an `INSERT`, and a chain of Go values into a parameterized `WHERE`. The entire design space of Go database libraries is just *different answers to the question: who writes that translation layer, and when?*

There are three broad answers, and understanding the trade-offs among them is the single most important thing in this section — because sqlc only makes sense once you feel the pain it removes.

### 1.2 Option A — raw `database/sql` or raw pgx **[B]**

The most direct approach is to write SQL as Go strings and scan the results by hand. With pgx v5 (the modern, high-performance Postgres driver) it looks like this:

```go
// Raw pgx: you write the SQL, you scan every column, by hand.
row := pool.QueryRow(ctx,
	`SELECT id, owner_id, balance, currency, created_at
	   FROM accounts WHERE id = $1`, accountID)

var a Account
if err := row.Scan(&a.ID, &a.OwnerID, &a.Balance, &a.Currency, &a.CreatedAt); err != nil {
	return nil, err
}
```

**Why people love it:** it is the fastest possible path — no reflection, no abstraction, the SQL is exactly what runs. You have total control. pgx exposes every Postgres feature (arrays, `COPY`, `LISTEN/NOTIFY`, custom types) directly.

**Why it hurts at scale:** notice everything the compiler *cannot* help you with. The SQL is a **string** — a typo in a column name (`blance`) compiles fine and fails at runtime. The `Scan` argument order must exactly match the `SELECT` column order — swap two and you silently write the balance into the currency field, a *catastrophe* in banking code. Add a column to the table and every `Scan` that used `SELECT *` (or an explicit list) silently breaks or drifts. There is no compile-time link between the shape of the query and the shape of the struct. You are, in effect, hand-maintaining a mapping that a machine could verify.

This is safe *SQL-injection-wise* (parameters are separate from the query text), but *type-wise* it is completely unchecked. Every mistake is a runtime mistake.

### 1.3 Option B — an ORM like Ent (or GORM) **[B]**

The opposite extreme hides SQL entirely behind a Go API. **Ent** (covered fully in **[Go ent ORM](GO_ENT_ORM_GUIDE.md)**) lets you write:

```go
// Ent: no SQL in sight — a type-safe query builder generated from your schema.
acct, err := client.Account.Query().
	Where(account.IDEQ(accountID)).
	Only(ctx)
```

**Why people love it:** it is extremely productive for CRUD and for *graph* traversal ("give me this user's posts and each post's comments"). The query builder is type-safe (Ent generates it from your schema), it handles relations, eager loading, and it removes almost all boilerplate. For an app that is mostly "load an entity, load its related entities, save it back," an ORM is a joy.

**Why it hurts sometimes:** an ORM is an abstraction *over* SQL, and abstractions leak. When you need a window function, a lateral join, a recursive CTE, a carefully hand-tuned index-only scan, or a bulk `INSERT ... ON CONFLICT` with a specific plan, you are now fighting the DSL, reading generated SQL to figure out what it *actually* ran, and sometimes dropping to raw SQL anyway. GORM specifically does its mapping via **runtime reflection**, so a misspelled column in a raw fragment is a runtime error. Ent is generated and type-safe, but it still *builds* SQL at runtime and you do not, at a glance, control the exact query text. For the hottest, most performance-critical, most correctness-critical paths, "I can't see the SQL" is a real cost.

### 1.4 Option C — sqlc: write real SQL, get type-safe Go **[B]**

sqlc takes a genuinely different stance, and it is the reason this guide exists:

> **You write ordinary, hand-authored SQL in `.sql` files. sqlc reads your schema and your queries at *build time*, understands them the way Postgres would, and generates concrete, fully type-safe Go — real structs, real methods, correct scan order — for you. There is no runtime reflection and no query DSL. The SQL that runs is the SQL you wrote.**

You write this query file:

```sql
-- name: GetAccount :one
SELECT id, owner_id, balance, currency, created_at
FROM accounts
WHERE id = $1;
```

…and sqlc generates this Go for you (you never edit it):

```go
// Generated by sqlc. DO NOT EDIT.
func (q *Queries) GetAccount(ctx context.Context, id uuid.UUID) (Account, error) {
	row := q.db.QueryRow(ctx, getAccount, id)
	var i Account
	err := row.Scan(&i.ID, &i.OwnerID, &i.Balance, &i.Currency, &i.CreatedAt)
	return i, err
}
```

Look at what you *got for free*: the parameter type (`uuid.UUID`) was inferred from the schema; the return type (`Account`) is a struct whose fields exactly match the columns; the `Scan` order is guaranteed correct because a machine wrote it against the actual `SELECT`. If you rename `balance` to `balance_cents` in the schema and regenerate, the `Account` struct changes and every caller that referenced `.Balance` becomes a **compile error** — the mistake surfaces at build time, not at 2 a.m. in production.

sqlc gives you the two things the other options each give up:

- Unlike raw pgx, **the mapping is machine-verified** — no scan-order bugs, no stringly-typed columns in your Go, compile-time safety when the schema changes.
- Unlike an ORM, **you keep the full expressive power of SQL and total visibility into what runs** — window functions, CTEs, `FOR UPDATE`, hand-tuned joins, exactly the query you wrote.

### 1.5 The "SQL is the source of truth" model — why it fits banking **[B/I]**

sqlc's core philosophy is that **SQL is the source of truth** and Go is generated *from* it. This inverts the ORM model (where Go schema code is the source of truth and SQL is generated from it). Three consequences make this model an outstanding fit for correctness- and performance-critical systems like a ledger:

1. **Every query is static and parameterized.** Because sqlc parses your SQL at build time, the query text is fixed; the only things that vary at runtime are the `$1, $2 …` parameters, which pgx sends *separately* from the SQL text. **SQL injection is structurally impossible** — there is no string concatenation anywhere in the hot path. (We return to this in §10.)
2. **The generated types match the database exactly.** In banking you care deeply that `balance` is a `numeric`/decimal and never a `float64`. sqlc will map the column to whatever type you configure (see `overrides`, §4.6) and *fail to generate* if a query is inconsistent with the schema — the type system enforces your money-handling policy.
3. **Performance is predictable.** There is no reflection, no DSL translation, no surprise query plan. What you read in the `.sql` file is what the planner sees. For a hot balance-lookup executed millions of times a day, that predictability is worth a great deal.

### 1.6 When to choose which — an honest decision table **[I]**

| You are… | Reach for | Because |
|---|---|---|
| Writing a hot, perf-critical, or SQL-heavy read path (reports, ledgers, analytics, window functions) | **sqlc** | You want the exact SQL, compile-time safety, zero reflection. |
| Doing lots of relation-graph CRUD (load entity + its edges, RBAC traversal, admin panels) | **Ent** | The graph traversal & eager-loading API is where an ORM shines. |
| Prototyping throwaway code, one-off scripts, or a tiny app | **raw pgx** | The ceremony of codegen isn't worth it for 3 queries. |
| Building a serious service and unsure | **sqlc first** | Most services are 90% straightforward queries; sqlc covers them with safety, and you can always drop to raw pgx for the exotic 10% since sqlc uses the same pool. |

The key realization: **these are not mutually exclusive.** sqlc, Ent, and raw pgx can all share one `*pgxpool.Pool` in the same process. §9 shows exactly how to run sqlc and Ent side by side. But *most* applications should pick **one primary** data layer and only reach for a second when a concrete need appears. Do not adopt two just because you can.

### 1.7 Where does goose fit in this picture? **[B]**

Everything above is about *reading and writing rows*. None of it *creates the tables in the first place*, nor evolves them over time (add a column, add an index, backfill data). That job — **schema evolution** — belongs to a **migration tool**, and that is **goose**. sqlc and goose are not competitors; they are the two halves of one workflow, and §2 explains why pairing them specifically is the whole point of this guide.

---

## 2. Why Pair sqlc WITH goose — One Source of Truth

### 2.1 The chicken-and-egg problem sqlc has **[B]**

sqlc generates Go from your SQL. To do that, it must **know your schema**: to generate `GetAccount`, sqlc has to know that `accounts` has columns `id uuid`, `owner_id uuid`, `balance numeric`, `currency text`, `created_at timestamptz`. Only then can it infer that the parameter is a `uuid.UUID` and the result struct has those five typed fields.

So sqlc needs a description of your schema — the `CREATE TABLE` statements. The question is: **where does that schema description live, and who keeps it accurate as the database changes?**

The naive answer is: keep a file, say `schema.sql`, full of `CREATE TABLE` statements, and point sqlc at it. That works on day one. But on day fifty you have run twenty migrations against production — added columns, dropped others, created indexes, renamed things — and now `schema.sql` is a *hand-maintained fiction* that must be kept in sync with reality by discipline alone. The moment `schema.sql` drifts from what migrations actually did, **sqlc generates code that matches a schema your database doesn't have**, and you get runtime errors that the whole point of sqlc was to prevent. This is the classic *schema drift* problem.

### 2.2 goose owns schema evolution **[B]**

**goose** is a database migration tool (covered fully in **[Goose migrations](GO_GOOSE_MIGRATIONS_GUIDE.md)**). A goose migration is a `.sql` file with two labeled halves — the forward change and its reverse:

```sql
-- db/migrations/00001_create_accounts.sql
-- +goose Up
CREATE TABLE accounts (
	id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
	owner_id   uuid NOT NULL,
	balance    numeric(20,4) NOT NULL DEFAULT 0,
	currency   text NOT NULL,
	created_at timestamptz NOT NULL DEFAULT now()
);

-- +goose Down
DROP TABLE accounts;
```

Running `goose up` applies every not-yet-applied migration's **Up** section in filename order and records what it did in a `goose_db_version` tracking table, so it never runs the same migration twice. Running `goose down` runs the latest migration's **Down** section to roll back. The set of migration files, applied in order, **is** the true definition of your schema — it is the exact sequence of DDL that produced the live database. There is no separate description to drift, because *the migrations are the description*.

### 2.3 The killer synergy: sqlc reads the goose files directly **[B]**

Here is the insight that makes this pairing special, and it is worth stating as loudly as possible:

> **sqlc can read goose migration files natively as its schema source.** You point sqlc's `schema:` setting at the *goose migrations directory*. sqlc parses the `-- +goose Up` / `-- +goose Down` annotations, ignores the Down halves, applies the Up statements *in filename order* — including `ALTER TABLE`s from later migrations — and builds the complete, current schema in memory, exactly as goose would produce it in the real database. **The migrations are the single source of truth for both the database and the generated code.**

The consequence is beautiful and is the beating heart of this guide:

- You write a new migration `00007_add_account_status.sql` that does `ALTER TABLE accounts ADD COLUMN status text NOT NULL DEFAULT 'active';`.
- You run `goose up` — the live database now has the `status` column.
- You run `sqlc generate` **with no config change whatsoever** — sqlc re-reads the same migrations directory, sees the `ALTER TABLE`, and now the `Account` struct has a `Status string` field and every query can reference `status`.

The database and the generated Go **cannot** drift, because they are produced from the identical set of files. One source of truth, verified by both tools. That is the entire reason to pair sqlc with goose rather than with a hand-written `schema.sql`.

### 2.4 The complete loop, drawn out **[B]**

```
          ┌─────────────────────────────────────────────────────┐
          │            db/migrations/*.sql  (goose)             │
          │   -- +goose Up / -- +goose Down  (the ONE truth)    │
          └───────────────┬─────────────────────┬───────────────┘
                          │                     │
             goose up ────┘                     └──── sqlc reads Up stmts
                          │                          as its schema source
                          ▼                                 │
                 ┌──────────────────┐                       ▼
                 │  live PostgreSQL │              ┌──────────────────┐
                 │      database    │              │  sqlc generate   │
                 └──────────────────┘              │  reads db/query  │
                          ▲                        │  + the schema    │
                          │                        └────────┬─────────┘
                          │                                 ▼
                          │                        ┌──────────────────┐
                          │  runtime: the typed    │  db/gen/*.go     │
                          └──────────────────────  │  (type-safe Go)  │
                             Go calls the DB via    └──────────────────┘
                             generated methods
```

Read it as a cycle: **edit migrations → `goose up` (schema changes in DB) → `sqlc generate` (types change in Go) → write/adjust queries → compile (mistakes caught) → run.** Because the top box feeds both the database and the codegen, the left and right sides can never disagree.

### 2.5 "Can't I just use one `schema.sql` file?" — yes, but don't **[I]**

sqlc absolutely *supports* pointing `schema:` at a single DDL file (or a directory of plain `CREATE TABLE` files with no goose annotations). Many sqlc tutorials do exactly that. It is simpler on paper.

But it re-introduces the drift problem: now you must keep `schema.sql` and your migrations in agreement by hand. Every migration must be mirrored into `schema.sql` in the same PR, forever, correctly. Miss one and your generated code lies about the database. Pointing sqlc at the goose directory **eliminates an entire class of bugs** for the price of one config line. For any project that will live longer than a weekend — and certainly for a banking system — read the goose files. That is the recommended, drift-proof setup, and it is what this guide uses throughout.

> **⚡ Version note:** goose-annotation parsing in sqlc has been stable and reliable for years, but sqlc parses the DDL itself (it is not calling goose). That means every Up statement in your migrations must be **standalone-valid DDL that sqlc's parser understands**. A handful of exotic constructs are discussed as gotchas in §12.7. In practice, ordinary `CREATE TABLE` / `ALTER TABLE` / `CREATE INDEX` / `CREATE TYPE` migrations parse cleanly.

---

## 3. Install & Project Layout — First End-to-End Loop

### 3.1 Installing the tools **[B]**

Both tools are single Go binaries. There are two install styles, and you should understand the difference.

**Install with `go install`** (simple, gets you going):

```bash
# sqlc — the code generator (current: v1.31.x in 2026)
go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest

# goose — the migration runner (current: v3.27.x in 2026)
go install github.com/pressly/goose/v3/cmd/goose@latest
```

On Windows these land in `%USERPROFILE%\go\bin` (e.g. `C:\Users\Phoniex\go\bin`). Make sure that directory is on your `PATH` — in PowerShell you can check with `where.exe sqlc` and `where.exe goose`. If they aren't found, add the folder to `PATH` via *System Properties → Environment Variables*, or run them by full path.

**Pin the versions** (recommended for teams / CI). `@latest` means "whatever is newest today," which is non-reproducible — two developers can generate different code. Pin exact versions instead:

```bash
go install github.com/sqlc-dev/sqlc/cmd/sqlc@v1.31.0
go install github.com/pressly/goose/v3/cmd/goose@v3.27.0
```

> **Best practice — `tools.go`:** record the tool versions in your module so `go.mod` pins them and everyone gets the same binaries. Create a build-tag-guarded file:
>
> ```go
> //go:build tools
> package tools
>
> import (
> 	_ "github.com/sqlc-dev/sqlc/cmd/sqlc"
> 	_ "github.com/pressly/goose/v3/cmd/goose"
> )
> ```
>
> Now `go.mod` tracks exact versions, and CI can `go run github.com/sqlc-dev/sqlc/cmd/sqlc generate` to use the pinned one. (Go 1.24+ also supports `go tool` directives in `go.mod` — either mechanism works; pick one and be consistent.)

**Verify the install:**

```bash
sqlc version    # -> v1.31.0
goose -version  # -> goose version: v3.27.0
```

### 3.2 The canonical project layout **[B]**

This layout is used by the whole guide. It cleanly separates the three concerns: **migrations** (schema truth), **queries** (your SQL), and **generated code** (never hand-edited).

```
myledger/
├── go.mod
├── sqlc.yaml                 # sqlc configuration (version "2")
├── .air.toml                 # live-reload config (see §8.5)
├── Makefile                  # or Taskfile.yml — the dev-loop commands
├── cmd/
│   └── api/
│       └── main.go           # wires pgxpool + Gin + the generated Queries
├── db/
│   ├── migrations/           # goose migrations — the ONE source of truth
│   │   ├── 00001_create_accounts.sql
│   │   ├── 00002_create_entries.sql
│   │   └── 00003_create_transfers.sql
│   ├── query/                # your hand-written .sql query files (sqlc input)
│   │   ├── accounts.sql
│   │   ├── entries.sql
│   │   └── transfers.sql
│   └── gen/                  # sqlc OUTPUT — generated Go, do not edit by hand
│       ├── db.go             # DBTX interface + New()
│       ├── models.go         # struct per table
│       ├── accounts.sql.go   # methods from accounts.sql
│       ├── entries.sql.go
│       └── transfers.sql.go
└── internal/
    └── ...                   # handlers, auth, services
```

Two rules that save pain:

- **`db/gen/` is machine-owned.** Never edit generated files. Commit them (so builds don't require sqlc installed), but treat them as read-only artifacts. Every file starts with `// Code generated by sqlc. DO NOT EDIT.`
- **`db/migrations/` is append-mostly and immutable once shipped.** Once a migration has run in any shared environment, you never edit it — you write a *new* migration to make further changes. (This immutability is why it can safely be the source of truth.)

### 3.3 The minimal `sqlc.yaml` **[B]**

Create `sqlc.yaml` at the project root. This is the exact `version: "2"` configuration for pgx/v5, and it is the config the whole guide builds on. Note the critical line: **`schema:` points at the goose migrations directory.**

```yaml
version: "2"
sql:
  - engine: "postgresql"
    queries: "./db/query"          # where YOUR .sql query files live
    schema: "./db/migrations"      # <-- the goose migrations dir = schema truth
    gen:
      go:
        package: "db"              # Go package name for generated code
        out: "./db/gen"            # where generated .go files are written
        sql_package: "pgx/v5"      # generate pgx v5 code (not database/sql)
        emit_json_tags: true       # add `json:"..."` tags to model structs
        emit_pointers_for_null_types: true  # nullable cols -> *T instead of pgtype.T
        emit_interface: true       # also emit a Querier interface (for mocking)
        emit_prepared_queries: false
        emit_exact_table_names: false
```

Every one of these options is dissected in §4. For now, `schema: "./db/migrations"` is the line that wires goose and sqlc together.

### 3.4 First migration with goose **[B]**

goose can create timestamped or sequential migration files. Sequential numbering (`00001_`, `00002_`) is easier to read and is what we use. Create the first one:

```bash
# Create a new sequential SQL migration in db/migrations
goose -dir db/migrations create create_accounts sql
# -> Created new file: db/migrations/00001_create_accounts.sql
```

Open it and fill in both halves. **Both halves matter** — the Up builds the schema (and sqlc reads it), the Down lets you roll back:

```sql
-- db/migrations/00001_create_accounts.sql
-- +goose Up
-- +goose StatementBegin
CREATE TABLE accounts (
	id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
	owner_id   uuid        NOT NULL,
	balance    numeric(20,4) NOT NULL DEFAULT 0,
	currency   text        NOT NULL,
	created_at timestamptz NOT NULL DEFAULT now()
);
-- +goose StatementEnd

-- +goose Down
-- +goose StatementBegin
DROP TABLE accounts;
-- +goose StatementEnd
```

> **What are `StatementBegin`/`StatementEnd`?** goose splits a migration into individual statements on semicolons so it can run them one at a time. But some statements *contain* semicolons that are not statement terminators — most notably `CREATE FUNCTION` bodies and `DO $$ ... $$` blocks. Wrapping a statement in `-- +goose StatementBegin` / `-- +goose StatementEnd` tells goose "treat everything in here as ONE statement, don't split on the inner semicolons." For simple `CREATE TABLE`s the markers are optional, but they are harmless and habitual, and they are essential the moment a function body appears. sqlc understands these markers too. See **[Goose migrations](GO_GOOSE_MIGRATIONS_GUIDE.md)** for the full treatment.

### 3.5 Applying the migration **[B]**

goose needs a database connection. It takes the driver name and a connection string. For Postgres via the standard-library driver goose bundles, use `postgres`:

```bash
# Windows PowerShell: set the connection string once, then run goose
$env:GOOSE_DRIVER = "postgres"
$env:GOOSE_DBSTRING = "host=localhost port=5432 user=postgres password=secret dbname=myledger sslmode=disable"

goose -dir db/migrations up
# -> OK   00001_create_accounts.sql (12.3ms)
# -> goose: successfully migrated database to version: 1
```

Or pass them inline (bash syntax):

```bash
goose -dir db/migrations postgres \
  "host=localhost port=5432 user=postgres password=secret dbname=myledger sslmode=disable" up
```

Useful goose subcommands you will use constantly:

| Command | What it does |
|---|---|
| `goose ... up` | Apply all pending migrations. |
| `goose ... up-by-one` | Apply exactly the next one migration. |
| `goose ... down` | Roll back the most recently applied migration. |
| `goose ... redo` | `down` then `up` the latest — handy while iterating. |
| `goose ... status` | Show which migrations are applied vs pending. |
| `goose ... version` | Print the current schema version number. |

After `goose up`, the `accounts` table exists in the live database.

### 3.6 Writing the first query **[B]**

Now create a query file. sqlc scans everything under `queries:` (`./db/query`). Each query gets a **magic comment annotation** telling sqlc the query's *name* and its *cardinality* (`:one`, `:many`, `:exec`, …):

```sql
-- db/query/accounts.sql

-- name: CreateAccount :one
INSERT INTO accounts (owner_id, currency)
VALUES ($1, $2)
RETURNING *;

-- name: GetAccount :one
SELECT * FROM accounts
WHERE id = $1;

-- name: ListAccountsByOwner :many
SELECT * FROM accounts
WHERE owner_id = $1
ORDER BY created_at DESC;
```

The comment format is exact: `-- name: <MethodName> :<cardinality>`. `CreateAccount` becomes a Go method named `CreateAccount`; `:one` means "returns exactly one row → returns `(Account, error)`"; `:many` means "returns a slice → `([]Account, error)`". Full annotation table in §5.1.

### 3.7 Generating the Go code **[B]**

From the project root, with `sqlc.yaml` present:

```bash
sqlc generate
```

sqlc now: reads `sqlc.yaml` → reads `./db/migrations` and builds the schema (applying the goose Up statements) → reads `./db/query` and parses each annotated query → *type-checks each query against the schema* → writes Go into `./db/gen`. If a query references a non-existent column, sqlc **errors out here, at generate time**, before your code ever runs. That early failure is the whole value proposition.

You now have (abbreviated):

```go
// db/gen/models.go — generated
type Account struct {
	ID        uuid.UUID          `json:"id"`
	OwnerID   uuid.UUID          `json:"owner_id"`
	Balance   pgtype.Numeric     `json:"balance"`
	Currency  string             `json:"currency"`
	CreatedAt pgtype.Timestamptz `json:"created_at"`
}
```

```go
// db/gen/accounts.sql.go — generated (abbreviated)
func (q *Queries) GetAccount(ctx context.Context, id uuid.UUID) (Account, error) { /* ... */ }
func (q *Queries) CreateAccount(ctx context.Context, arg CreateAccountParams) (Account, error) { /* ... */ }
func (q *Queries) ListAccountsByOwner(ctx context.Context, ownerID uuid.UUID) ([]Account, error) { /* ... */ }
```

### 3.8 Calling the typed method **[B]**

Finally, wire a pgx pool and call the generated method. This is the whole loop paying off:

```go
package main

import (
	"context"
	"log"

	"github.com/jackc/pgx/v5/pgxpool"
	db "myledger/db/gen" // the generated package
)

func main() {
	ctx := context.Background()

	// A connection POOL — the recommended pgx primitive for servers.
	pool, err := pgxpool.New(ctx, "postgres://postgres:secret@localhost:5432/myledger")
	if err != nil {
		log.Fatal(err)
	}
	defer pool.Close()

	// db.New accepts anything satisfying DBTX; a *pgxpool.Pool does.
	q := db.New(pool)

	// Fully type-checked. ownerID must be a uuid.UUID; the return is (Account, error).
	acct, err := q.CreateAccount(ctx, db.CreateAccountParams{
		OwnerID:  someOwnerUUID,
		Currency: "USD",
	})
	if err != nil {
		log.Fatal(err)
	}
	log.Printf("created account %s for owner %s", acct.ID, acct.OwnerID)
}
```

That is the entire beginner arc: **migration → `goose up` → query → `sqlc generate` → typed call.** Everything else in this guide deepens one of those four steps.

---

## 4. `sqlc.yaml` in Depth — Every Option That Matters

### 4.1 Structure of the config **[B]**

`sqlc.yaml` uses `version: "2"` (the current format; version 1 is legacy and you should not use it for new projects). The top-level `sql:` key holds a **list** — each entry describes one code-generation target (one engine, one set of queries, one output package). Most projects have exactly one entry, but you can have several (e.g. one for Postgres, one for a separate analytics database).

```yaml
version: "2"
sql:
  - engine: "postgresql"       # the database engine (also: mysql, sqlite)
    queries: "./db/query"      # dir or file(s) of your query .sql
    schema:  "./db/migrations" # dir or file(s) defining the schema (goose dir!)
    gen:                       # what to generate
      go:
        # ... all the Go options, below
    # database:                # optional: a live DB for `sqlc vet` (see §8.3)
    # rules:                   # optional: sqlc vet lint rules (see §8.3)
```

`queries` and `schema` each accept a single path or a **list** of paths/globs. `schema` pointed at a directory reads *every* `.sql` file in it, in filename order — which is exactly why the goose directory works as a schema source: the numbered files apply in sequence.

### 4.2 The complete, annotated `gen.go` block **[B/I]**

Here is a fully annotated production configuration. Read the comments — each option changes the *shape* of the code you get, and getting them right up front saves painful churn later.

```yaml
version: "2"
sql:
  - engine: "postgresql"
    queries: "./db/query"
    schema:  "./db/migrations"
    gen:
      go:
        # --- identity / output ---
        package: "db"                 # package name at the top of every generated file
        out: "./db/gen"               # output directory
        sql_package: "pgx/v5"         # target driver: "pgx/v5" | "pgx/v4" | "database/sql"

        # --- struct decoration ---
        emit_json_tags: true          # add `json:"col_name"` to model fields (great for APIs)
        json_tags_case_style: "snake" # "snake" | "camel" | "pascal" | "none" for the json tags
        emit_db_tags: false           # add `db:"col_name"` tags too (for other scanners)

        # --- null handling (see §4.5) ---
        emit_pointers_for_null_types: true   # nullable col -> *string, *int64 (else pgtype.Text…)

        # --- interfaces & mocking (see §6.6) ---
        emit_interface: true          # also generate a `Querier` interface listing every method
        emit_methods_with_db_argument: false  # methods take db as a param instead of holding it

        # --- result-shape tweaks ---
        emit_result_struct_pointers: false   # :many returns []*T instead of []T when true
        emit_params_struct_pointers: false   # pass *Params instead of Params when true
        emit_exact_table_names: false        # struct name = singularized table name (accounts->Account)
        emit_empty_slices: true              # :many returns [] (not nil) when no rows — nicer for JSON

        # --- prepared statements ---
        emit_prepared_queries: false  # generate a Prepare() path caching prepared stmts (rarely needed w/ pgx)

        # --- type & name mapping (see §4.6, §4.7) ---
        overrides:
          - db_type: "uuid"
            go_type: "github.com/google/uuid.UUID"
          - db_type: "timestamptz"
            go_type: "time.Time"
          - db_type: "numeric"
            go_type:
              import: "github.com/jackc/pgx/v5/pgtype"
              type: "Numeric"
        rename:
          id: "ID"           # force a specific Go field name for a column
          api_key: "APIKey"  # sqlc's default would be ApiKey; override to APIKey
```

### 4.3 `sql_package` — why `pgx/v5` **[B/I]**

`sql_package` decides which driver's types and call style the generated code uses. Your realistic choices:

- **`pgx/v5`** — modern, high-performance native Postgres driver. Generates code that uses pgx types (`pgtype.Numeric`, `pgtype.Timestamptz`), pgx's `QueryRow`/`Query`/`Exec`, and crucially the **pgx batch and `CopyFrom`** features (`:batch*`, `:copyfrom` annotations). This is the right choice for new Postgres services and is what this guide uses everywhere.
- **`pgx/v4`** — same idea, older major version. Only if you're stuck on v4.
- **`database/sql`** — generates code against the standard library's `*sql.DB`, using `sql.NullString` etc. for nullables. Use only if you must use a `database/sql`-based driver or share a `*sql.DB` with libraries that require it. You *lose* the pgx batch/copyfrom generation.

With `sql_package: "pgx/v5"`, the generated `New()` takes a **`DBTX`** interface (detailed in §6.2) that a `*pgxpool.Pool`, a `*pgx.Conn`, and a `pgx.Tx` all satisfy — which is what makes the transaction pattern in §7 so clean.

### 4.4 `emit_json_tags`, naming, and struct decoration **[B]**

- **`emit_json_tags: true`** adds `json:"..."` struct tags so you can return generated models straight from a Gin handler as JSON without a separate DTO. `json_tags_case_style` controls the tag casing (`snake` keeps `owner_id`; `camel` gives `ownerId`). For an API consumed by a JS/TS frontend, `camel` is often nice; for internal services `snake` matches the DB.
- **`emit_exact_table_names: false`** makes sqlc singularize table names for struct names (`accounts` → `Account`, `entries` → `Entry`). Set to `true` to keep the exact table name (`accounts` → `Accounts`). Singular structs read better (`var a Account`), so `false` is the common choice.
- **`emit_empty_slices: true`** makes `:many` queries return an empty non-nil slice when there are no rows, instead of `nil`. This serializes to `[]` in JSON rather than `null`, which frontends universally prefer. Recommended `true` for API services.

### 4.5 Null handling — `emit_pointers_for_null_types` (the big one) **[I]**

This is the option people most often get wrong, so slow down here.

In Postgres a column is either `NOT NULL` or nullable. sqlc must represent a *nullable* column with a Go type that can express "no value." There are two strategies:

1. **Default (option off): pgtype wrapper types.** A nullable `text` becomes `pgtype.Text` — a struct `{ String string; Valid bool }`. A nullable `bigint` becomes `pgtype.Int8` (`{ Int64 int64; Valid bool }`), nullable `timestamptz` → `pgtype.Timestamptz`, nullable `uuid` → `pgtype.UUID`. You check `.Valid` before reading `.String`/`.Int64`/etc. This is explicit and lossless, and it round-trips cleanly through pgx, but it is verbose and does not serialize to clean JSON without help.

2. **`emit_pointers_for_null_types: true`: Go pointers.** A nullable `text` becomes `*string`, nullable `bigint` → `*int64`, nullable `timestamptz` → `*time.Time`. `nil` means SQL `NULL`; a non-nil pointer holds the value. This is idiomatic Go, serializes to JSON perfectly (`null` or the value), and reads naturally (`if a.ClosedAt != nil`).

> **The critical rule:** this affects **nullable columns only.** A `NOT NULL text` column is *always* a plain `string` regardless of this setting — there is no null to represent. So the setting is really "how do I want my *nullable* columns to look." For an API-facing service, **`true` (pointers)** is usually the better ergonomics. The tradeoff: pointer fields mean more nil-checks and a small allocation per non-null value, and some people prefer the explicit `.Valid` of pgtype. Pick one policy and keep it consistent across the whole project — mixing is confusing.

There is also `emit_pointers_for_null_types`'s cousin behavior with `sqlc.narg` (nullable named params, §5.3): a `sqlc.narg('status')` parameter becomes `*string` (with pointers on) or `pgtype.Text` (off).

### 4.6 `overrides` — mapping DB types to the Go types you actually want **[I]**

By default sqlc picks reasonable Go types, but "reasonable" isn't always "what your codebase already uses." `overrides` lets you say "every column of DB type X should be Go type Y." This is essential for three types in particular:

- **`uuid`** → by default sqlc (with pgx/v5) uses `pgtype.UUID` (a `[16]byte` + `Valid`). Most codebases prefer **`github.com/google/uuid.UUID`**, which prints as the familiar hyphenated string and is what the rest of your app uses. Override it.
- **`numeric`** (money!) → maps to `pgtype.Numeric` by default. That is *correct* — it's an arbitrary-precision decimal, never a float. You may override to a dedicated decimal type like `github.com/shopspg/decimal.Decimal` or `github.com/jackc/pgx/v5/pgtype.Numeric` explicitly. **Never** override money to `float64`. (See §10.3.)
- **`timestamptz`** → maps to `pgtype.Timestamptz` by default; many teams override to plain **`time.Time`** for `NOT NULL` timestamp columns for ergonomics (pgx knows how to scan into `time.Time`).

An `overrides` entry can target a **DB type** (applies everywhere that type appears) or a **specific column** (`column: "accounts.balance"`), and can control the nullable variant separately:

```yaml
overrides:
  # Every uuid column -> google/uuid.UUID (the friendly type)
  - db_type: "uuid"
    go_type: "github.com/google/uuid.UUID"

  # Nullable uuid -> a pointer to it, so NULL is representable
  - db_type: "uuid"
    nullable: true
    go_type:
      import: "github.com/google/uuid"
      type: "UUID"
      pointer: true

  # Every timestamptz (NOT NULL) -> time.Time
  - db_type: "timestamptz"
    go_type: "time.Time"

  # Money: keep arbitrary precision. NEVER float. Here we make it explicit.
  - db_type: "numeric"
    go_type:
      import: "github.com/jackc/pgx/v5/pgtype"
      type: "Numeric"

  # Target one specific column by name (wins over db_type rules)
  - column: "accounts.currency"
    go_type: "myledger/internal/money.Currency"
```

The `go_type` can be a short string (`"time.Time"`, or a full import path + type like `"github.com/google/uuid.UUID"`) or the long form (`import` / `type` / `pointer`) when you need the pointer flag or an aliased import. Keep your overrides **identical** across every `sql:` entry in a multi-target config, or the same column will have different Go types in different packages — a maddening bug (see §12.6).

> **Requirement:** an overridden Go type must be scannable by pgx (implement the right decoding) and, for parameters, encodable. `google/uuid.UUID`, `time.Time`, and `pgtype.Numeric` all satisfy pgx out of the box. A custom type must implement pgx's scan/value interfaces or be registered on the pool's type map.

### 4.7 `rename` — fixing individual field names **[I]**

sqlc turns snake_case columns into Go PascalCase, applying a list of common initialisms (`id`→`ID`, `url`→`URL`, `http`→`HTTP`). Occasionally it guesses "wrong" for your taste — `api_key` becomes `ApiKey`, but you want `APIKey`. `rename` overrides a specific name:

```yaml
rename:
  api_key: "APIKey"
  ip: "IP"
  oauth: "OAuth"
```

`rename` keys are the column (or enum) names; values are the exact Go identifier you want. Use sparingly, for genuine initialisms.

### 4.8 Full config reference table **[I]**

| Option | Type | Default | What it does |
|---|---|---|---|
| `engine` | string | — | `postgresql`, `mysql`, or `sqlite`. |
| `queries` | path/list | — | Where your query `.sql` files are. |
| `schema` | path/list | — | Schema source. **Point at the goose migrations dir.** |
| `gen.go.package` | string | — | Package name of generated code. |
| `gen.go.out` | path | — | Output directory. |
| `gen.go.sql_package` | string | `database/sql` | `pgx/v5` recommended for Postgres. |
| `emit_json_tags` | bool | `false` | Add `json:"…"` tags. |
| `json_tags_case_style` | string | `none` | `snake`/`camel`/`pascal`/`none`. |
| `emit_db_tags` | bool | `false` | Add `db:"…"` tags. |
| `emit_pointers_for_null_types` | bool | `false` | Nullable cols → `*T` instead of `pgtype.T`. |
| `emit_interface` | bool | `false` | Emit a `Querier` interface for mocking. |
| `emit_exact_table_names` | bool | `false` | Keep exact (plural) table names for structs. |
| `emit_empty_slices` | bool | `false` | `:many` returns `[]` not `nil` when empty. |
| `emit_result_struct_pointers` | bool | `false` | `:many` returns `[]*T`. |
| `emit_params_struct_pointers` | bool | `false` | Params passed as `*T`. |
| `emit_prepared_queries` | bool | `false` | Generate a prepared-statement path. |
| `emit_methods_with_db_argument` | bool | `false` | Methods take `db` arg instead of holding it. |
| `overrides` | list | — | Map DB types/columns → Go types. |
| `rename` | map | — | Override individual generated field names. |

### 4.9 A note on multiple targets and shared config **[A]**

If you have two databases (say, a transactional Postgres and a read-replica analytics store), you add two entries under `sql:`. Each generates into its own `out` package. Keep `overrides` blocks in sync between them if the same types appear (money, uuid). sqlc also supports YAML anchors to avoid repetition; but for two targets, honestly, copy-paste-and-keep-in-sync is clearer than clever anchors.

---

## 5. Writing Queries: Annotations, Params, Joins, Pagination, Upserts

### 5.1 The query annotation — name + cardinality **[B]**

Every query in a `db/query/*.sql` file is preceded by a magic comment of the form:

```
-- name: <GoMethodName> :<cardinality>
```

`<GoMethodName>` becomes the exact Go method name (so use PascalCase). `<cardinality>` tells sqlc what shape of result to generate. This is the master reference table:

| Annotation | Generated return | Use when | Notes |
|---|---|---|---|
| `:one` | `(T, error)` | Query returns exactly one row | Returns `pgx.ErrNoRows` if zero rows. |
| `:many` | `([]T, error)` | Query returns zero-or-more rows | Slice; empty vs nil per `emit_empty_slices`. |
| `:exec` | `error` | `INSERT`/`UPDATE`/`DELETE` you don't need results from | Simplest write. |
| `:execrows` | `(int64, error)` | You need the affected-row count | Great for "did the UPDATE hit anything?" |
| `:execresult` | `(pgconn.CommandTag, error)` | You need the raw command tag | Advanced; rarely needed. |
| `:execlastid` | `(int64, error)` | Get the last inserted id | MySQL/SQLite mainly; Postgres prefers `RETURNING`. |
| `:batchexec` | a `*Batch` you iterate | Bulk write via pgx batch | pgx/v5 only. |
| `:batchmany` | batch of `:many`s | Bulk read via pgx batch | pgx/v5 only. |
| `:batchone` | batch of `:one`s | Bulk single-row ops | pgx/v5 only. |
| `:copyfrom` | `(int64, error)` | High-throughput bulk INSERT | Uses Postgres `COPY`; pgx/v5 only. |

The distinction between `:one` and `:many` is not cosmetic — it changes the generated code from `QueryRow`+single `Scan` to `Query`+loop. Choose based on how many rows the query *can* return, not how many you expect.

### 5.2 Positional vs named parameters **[B]**

sqlc supports two ways to write parameters. Both are safe (all parameters are sent separately from SQL — no injection).

- **Positional `$1, $2, …`** — the raw Postgres style. sqlc names the generated Go params from context when it can (`GetAccount($1)` → param `id` because the `WHERE id = $1`), but for multi-param inserts it bundles them into a `<Method>Params` struct with best-guess names. Fine for simple queries.

```sql
-- name: GetEntry :one
SELECT * FROM entries WHERE id = $1;
```

- **Named `sqlc.arg(name)`** — you name the parameter explicitly, and sqlc uses that exact name in the generated Params struct. This is far more readable for anything with more than one parameter, and it is self-documenting.

```sql
-- name: CreateEntry :one
INSERT INTO entries (account_id, amount, reason)
VALUES (sqlc.arg(account_id), sqlc.arg(amount), sqlc.arg(reason))
RETURNING *;
```

generates a `CreateEntryParams{ AccountID, Amount, Reason }` struct — clean and obvious.

### 5.3 `sqlc.arg` vs `sqlc.narg` — NOT NULL vs nullable params **[I]**

This distinction is subtle and important:

- **`sqlc.arg(name)`** produces a **non-nullable** Go parameter. If the underlying column is `NOT NULL`, the param is a plain `string`/`int64`/etc.
- **`sqlc.narg(name)`** produces a **nullable** Go parameter — `*string` (with pointers on) or `pgtype.Text` (off). Use it when the parameter is genuinely optional, most commonly in a filtered query where "not provided" should mean "don't filter on this."

The canonical use of `sqlc.narg` is an **optional filter with COALESCE**, which is sqlc's idiomatic answer to "sort-of dynamic WHERE":

```sql
-- name: SearchEntries :many
-- If the caller passes NULL for a filter, COALESCE falls back to the column itself,
-- which makes that condition always true — i.e. the filter is skipped.
SELECT * FROM entries
WHERE account_id = sqlc.arg(account_id)
  AND reason   = COALESCE(sqlc.narg(reason), reason)
  AND amount  >= COALESCE(sqlc.narg(min_amount), amount)
ORDER BY created_at DESC
LIMIT sqlc.arg(page_size);
```

Here `account_id` and `page_size` are required (`sqlc.arg`), while `reason` and `min_amount` are optional (`sqlc.narg`) — pass `nil` to skip that filter. This is the standard workaround for sqlc's inability to build a truly dynamic WHERE (see §12.1).

### 5.4 `sqlc.embed` — nesting a full struct in a join **[I]**

When you join two tables, you often want the result to contain *both* full rows as nested structs, not a flat mash of columns. `sqlc.embed(table)` does exactly that:

```sql
-- name: GetEntryWithAccount :one
SELECT sqlc.embed(entries), sqlc.embed(accounts)
FROM entries
JOIN accounts ON accounts.id = entries.account_id
WHERE entries.id = $1;
```

generates a result struct like:

```go
type GetEntryWithAccountRow struct {
	Entry   Entry    // the full entries row as a nested struct
	Account Account  // the full accounts row as a nested struct
}
```

This is far nicer than a flat struct with 12 columns you have to mentally re-group. Use it whenever a join's result naturally decomposes into whole entities.

### 5.5 `:one` and `:many` in practice **[B]**

```sql
-- name: GetAccountForUpdate :one
-- FOR UPDATE locks the row for the duration of the transaction (see §7).
SELECT * FROM accounts
WHERE id = $1
FOR UPDATE;

-- name: ListEntriesByAccount :many
SELECT * FROM entries
WHERE account_id = $1
ORDER BY created_at DESC
LIMIT $2 OFFSET $3;
```

Generated signatures (abbreviated), with pointers/overrides applied:

```go
func (q *Queries) GetAccountForUpdate(ctx context.Context, id uuid.UUID) (Account, error)

type ListEntriesByAccountParams struct {
	AccountID uuid.UUID
	Limit     int32
	Offset    int32
}
func (q *Queries) ListEntriesByAccount(ctx context.Context, arg ListEntriesByAccountParams) ([]Entry, error)
```

### 5.6 LIMIT/OFFSET vs keyset pagination **[I]**

Two ways to paginate, and the difference matters at scale.

**OFFSET pagination** (`LIMIT $1 OFFSET $2`) is easy but degrades: to return page 10,000 the database must scan and discard 10,000×pageSize rows. It also *skips or duplicates* rows if data is inserted between page loads. Fine for small admin lists; bad for large, growing tables like a ledger.

**Keyset (cursor) pagination** uses the last-seen sort key as the cursor and asks for rows *after* it. It uses an index range scan — O(pageSize) regardless of how deep you are — and is stable under concurrent inserts. It is the correct choice for a ledger's entries. The trick is a composite comparison on `(created_at, id)` to break ties deterministically:

```sql
-- name: ListEntriesKeyset :many
-- Cursor = the (created_at, id) of the last row on the previous page.
-- We want the NEXT rows in descending order, so we ask for rows "before" the cursor.
SELECT * FROM entries
WHERE account_id = sqlc.arg(account_id)
  AND (created_at, id) < (sqlc.arg(cursor_created_at), sqlc.arg(cursor_id))
ORDER BY created_at DESC, id DESC
LIMIT sqlc.arg(page_size);
```

For the *first* page, pass a sentinel far-future cursor (e.g. `now()` or `'infinity'::timestamptz` and the max UUID) so every row qualifies. Back this with an index `CREATE INDEX ON entries (account_id, created_at DESC, id DESC);` — see **[PostgreSQL](POSTGRESQL_GUIDE.md)** for index tuning.

### 5.7 Joins, aggregates, and GROUP BY **[I]**

sqlc handles arbitrary SQL, including aggregates. The generated result struct adapts to the exact `SELECT` list, so aggregate expressions need explicit aliases (that alias becomes the Go field name):

```sql
-- name: AccountBalanceSummary :one
SELECT
	COUNT(*)                          AS entry_count,
	COALESCE(SUM(amount), 0)::numeric AS total_amount,
	MAX(created_at)                   AS last_entry_at
FROM entries
WHERE account_id = $1;
```

generates:

```go
type AccountBalanceSummaryRow struct {
	EntryCount  int64
	TotalAmount pgtype.Numeric
	LastEntryAt pgtype.Timestamptz
}
```

Two gotchas visible here: **alias every computed column** (`AS entry_count`) so sqlc has a name; and **cast when the type is ambiguous** — `SUM(amount)` over an empty set is `NULL`, and `COALESCE(..., 0)` keeps it non-null, while the `::numeric` cast pins the type so sqlc infers `pgtype.Numeric` rather than a surprising type.

### 5.8 RETURNING and upserts (`ON CONFLICT`) **[I]**

Postgres's `RETURNING` clause lets an `INSERT`/`UPDATE`/`DELETE` hand back the affected rows — pair it with `:one`/`:many` to get the result without a second round-trip. This is the idiomatic way to create a row and immediately get its generated `id`/`created_at`.

An **upsert** ("insert or update if it already exists") uses `INSERT ... ON CONFLICT`. Classic use: an idempotency-key table, or a settings row.

```sql
-- name: UpsertAccountBalance :one
-- Insert a starting balance; if the account already has one, add to it.
INSERT INTO account_balances (account_id, balance)
VALUES (sqlc.arg(account_id), sqlc.arg(delta))
ON CONFLICT (account_id)
DO UPDATE SET balance = account_balances.balance + EXCLUDED.balance
RETURNING *;
```

`EXCLUDED` refers to the row that *would* have been inserted, so `EXCLUDED.balance` is the `delta`. `RETURNING *` with `:one` gives you the final row. Upserts are a workhorse for idempotent writes (§10.7).

### 5.9 Batch operations (`:batchexec`, `:batchmany`, `:batchone`) **[A]**

pgx v5 supports **pipelining** many statements in a single network round-trip via a batch. sqlc exposes this with the `:batch*` annotations. This is dramatically faster than a loop of individual queries when you have many small operations (e.g. inserting 500 ledger entries).

```sql
-- name: InsertEntryBatch :batchexec
INSERT INTO entries (account_id, amount, reason)
VALUES (sqlc.arg(account_id), sqlc.arg(amount), sqlc.arg(reason));
```

Generated usage (abbreviated) — you queue many, then execute the pipeline:

```go
params := []db.InsertEntryBatchParams{ /* ...500 of them... */ }
br := q.InsertEntryBatch(ctx, params) // returns a *InsertEntryBatchBatchResults
defer br.Close()
br.Exec(func(i int, err error) {
	if err != nil {
		log.Printf("row %d failed: %v", i, err)
	}
})
```

`:batchone`/`:batchmany` work similarly but let each queued statement return rows. Batches run within whatever `DBTX` you gave `New()` — so you can batch inside a transaction (§7).

### 5.10 `:copyfrom` — the fastest bulk insert **[A]**

For truly high-volume ingestion (tens of thousands of rows), even batching is slower than Postgres's `COPY` protocol. `:copyfrom` generates code using pgx's `CopyFrom`, which streams rows in a compact binary format:

```sql
-- name: BulkInsertEntries :copyfrom
INSERT INTO entries (account_id, amount, reason)
VALUES ($1, $2, $3);
```

```go
rows := []db.BulkInsertEntriesParams{ /* ...50,000 of them... */ }
inserted, err := q.BulkInsertEntries(ctx, rows) // one COPY stream
```

Caveats: `COPY` does **not** run per-row triggers the same way, does not support `ON CONFLICT`, and returns only a count (no `RETURNING`). It is the tool for bulk data loads (imports, backfills), not for everyday inserts. But when you need it, nothing is faster.

### 5.11 Query-writing best practices **[I]**

- **One query per method; keep them focused.** sqlc rewards small, named, single-purpose queries. Composition happens in Go, not in mega-queries.
- **Never `SELECT *` in production queries.** It couples your struct to column order/count and breaks intent. List columns explicitly. (You can even *ban* `SELECT *` with a `sqlc vet` rule — §8.3.) In this guide we use `*` in a few teaching snippets for brevity, but real code should enumerate columns.
- **Alias every computed/aggregate column.** No alias → no Go field name → confusing generated code.
- **Name multi-param queries with `sqlc.arg`.** Positional params in a 5-arg insert produce guessy field names; named params are self-documenting.
- **Push filtering/sorting/limiting into SQL.** Don't fetch 10k rows and filter in Go; let Postgres and its indexes do the work.

---

## 6. The Generated Code Explained

### 6.1 What sqlc writes, file by file **[B]**

After `sqlc generate`, `db/gen/` contains a small, predictable set of files. Knowing what each is demystifies the whole thing:

- **`db.go`** — the plumbing: the `DBTX` interface, the `Queries` struct, `New()`, and `WithTx()`. This file is the same shape in every sqlc project.
- **`models.go`** — one Go struct per table in your schema (`Account`, `Entry`, `Transfer`), plus Go types for any Postgres `enum`s. These mirror the tables sqlc read from your goose migrations.
- **`<file>.sql.go`** — for each query file (`accounts.sql` → `accounts.sql.go`): one method per query, each Params/Row struct, and the raw SQL as a `const` string.
- **`querier.go`** — only if `emit_interface: true`: the `Querier` interface listing every method (for mocking, §6.6).
- **`batch.go`** — only if you used any `:batch*` annotation.
- **`copyfrom.go`** — only if you used any `:copyfrom` annotation.

### 6.2 The `DBTX` interface — the key abstraction **[B/I]**

The single most important generated type is `DBTX`. With `sql_package: "pgx/v5"` it looks like this:

```go
// db/gen/db.go — generated
type DBTX interface {
	Exec(context.Context, string, ...interface{}) (pgconn.CommandTag, error)
	Query(context.Context, string, ...interface{}) (pgx.Rows, error)
	QueryRow(context.Context, string, ...interface{}) pgx.Row
}

type Queries struct {
	db DBTX
}

func New(db DBTX) *Queries {
	return &Queries{db: db}
}
```

The genius here is that **three different pgx types all satisfy `DBTX`**:

- `*pgxpool.Pool` — a connection pool (what you use for normal, autocommitted queries).
- `*pgx.Conn` — a single dedicated connection.
- `pgx.Tx` — a transaction handle.

Because all three implement `Exec`/`Query`/`QueryRow`, you can construct a `Queries` around *any* of them. `db.New(pool)` gives you a pool-backed `Queries` for everyday use; `db.New(tx)` (or, more ergonomically, `q.WithTx(tx)`) gives you a transaction-backed one. **The generated methods don't know or care which** — that is what makes transactions in §7 so clean.

### 6.3 The models **[B]**

`models.go` has one struct per table. With our overrides (uuid→google/uuid, numeric→pgtype.Numeric, timestamptz→time.Time) and `emit_json_tags: true`:

```go
// db/gen/models.go — generated
type Account struct {
	ID        uuid.UUID      `json:"id"`
	OwnerID   uuid.UUID      `json:"owner_id"`
	Balance   pgtype.Numeric `json:"balance"`
	Currency  string         `json:"currency"`
	CreatedAt time.Time      `json:"created_at"`
}

type Entry struct {
	ID        int64          `json:"id"`
	AccountID uuid.UUID      `json:"account_id"`
	Amount    pgtype.Numeric `json:"amount"`
	Reason    string         `json:"reason"`
	CreatedAt time.Time      `json:"created_at"`
}
```

The field types are the *result of your config*: change an override or toggle `emit_pointers_for_null_types` and these structs change on the next `generate`. That is the feedback loop — config drives shape.

### 6.4 A generated method, dissected **[I]**

Here is a full generated `:one` method so you can see there is no magic — just the SQL, a `QueryRow`, and a correct `Scan`:

```go
// db/gen/accounts.sql.go — generated
const getAccount = `-- name: GetAccount :one
SELECT id, owner_id, balance, currency, created_at FROM accounts
WHERE id = $1`

func (q *Queries) GetAccount(ctx context.Context, id uuid.UUID) (Account, error) {
	row := q.db.QueryRow(ctx, getAccount, id)   // q.db is the DBTX (pool OR tx)
	var i Account
	err := row.Scan(
		&i.ID,        // scan order is machine-generated to match the SELECT list
		&i.OwnerID,   // — impossible to get wrong
		&i.Balance,
		&i.Currency,
		&i.CreatedAt,
	)
	return i, err
}
```

And a `:many`, which loops:

```go
func (q *Queries) ListAccountsByOwner(ctx context.Context, ownerID uuid.UUID) ([]Account, error) {
	rows, err := q.db.Query(ctx, listAccountsByOwner, ownerID)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	items := []Account{}
	for rows.Next() {
		var i Account
		if err := rows.Scan(&i.ID, &i.OwnerID, &i.Balance, &i.Currency, &i.CreatedAt); err != nil {
			return nil, err
		}
		items = append(items, i)
	}
	if err := rows.Err(); err != nil {   // pgx: always check rows.Err() after the loop
		return nil, err
	}
	return items, nil
}
```

That is *exactly* the boilerplate you'd otherwise write by hand for every query, generated correctly every time. The value is not that it does something you couldn't — it's that it does it *without the bugs*.

### 6.5 Params structs and the "single arg vs struct" rule **[I]**

sqlc's convention: a query with **one** parameter generates a method taking that param directly (`GetAccount(ctx, id)`); a query with **two or more** parameters bundles them into a `<Method>Params` struct (`CreateAccount(ctx, CreateAccountParams{...})`). The struct form is a deliberate choice — it makes call sites self-documenting (you see the field names) and prevents "which arg was which" mistakes. You can force pointer-to-struct passing with `emit_params_struct_pointers`, but the value form is fine and idiomatic.

### 6.6 `emit_interface` and the `Querier` — mocking your data layer **[I]**

With `emit_interface: true`, sqlc also writes:

```go
// db/gen/querier.go — generated
type Querier interface {
	CreateAccount(ctx context.Context, arg CreateAccountParams) (Account, error)
	GetAccount(ctx context.Context, id uuid.UUID) (Account, error)
	ListAccountsByOwner(ctx context.Context, ownerID uuid.UUID) ([]Account, error)
	// ... one line per query ...
}
// *Queries implements Querier.
```

Why this matters: your service layer can depend on the `Querier` **interface** rather than the concrete `*Queries`. In tests you substitute a mock/fake implementation — no database required for unit tests of business logic. Generate a mock with `counterfeiter`/`mockery`/`gomock`, or hand-write a fake:

```go
// A hand fake for a unit test — implements only what the test touches.
type fakeQ struct{ db.Querier }
func (f fakeQ) GetAccount(_ context.Context, id uuid.UUID) (db.Account, error) {
	return db.Account{ID: id, Currency: "USD"}, nil
}

func TestTransferService(t *testing.T) {
	svc := NewTransferService(fakeQ{})   // no real DB
	// ... assert on business logic ...
}
```

> **Caveat:** the `Querier` interface is great for *unit* testing business logic, but it does **not** model transactions well (the `WithTx` return is `*Queries`, not `Querier`). For code that must run in a transaction, prefer **integration tests against a real Postgres** (e.g. via `testcontainers-go` or a disposable Docker Postgres) — mocking a transactional money transfer gives false confidence. Use both: interface mocks for pure logic, real DB for transactional paths.

### 6.7 `emit_prepared_queries` — do you need it? **[A]**

`emit_prepared_queries: true` generates a `Prepare(ctx, db)` method that pre-prepares every query and a `Queries` variant that uses the prepared statements. With `database/sql` this can save re-parsing on every call. **With pgx/v5 you almost never need it** — pgx automatically caches prepared statements per connection by default (statement cache), so you get the benefit without the extra generated surface and the lifecycle management of prepared statements across a pool. Leave it `false` for pgx unless you have measured a specific need.

---

## 7. Transactions with sqlc + pgx

### 7.1 Why transactions are the whole point in banking **[B]**

A **transaction** groups several statements so they either *all* succeed or *all* roll back, with no partial state visible to anyone else. In a ledger this is non-negotiable: a money transfer must debit one account **and** credit another **and** write two ledger entries — atomically. If the process crashes after the debit but before the credit, money vanishes. Transactions make that impossible: either the whole transfer commits or none of it does.

sqlc + pgx makes this remarkably clean, thanks to the `DBTX` interface (§6.2): since a `pgx.Tx` satisfies `DBTX`, you can run the *same generated methods* inside a transaction just by handing the `Queries` a tx instead of the pool.

### 7.2 The `WithTx` pattern **[B/I]**

The generated `db.go` includes:

```go
// db/gen/db.go — generated
func (q *Queries) WithTx(tx pgx.Tx) *Queries {
	return &Queries{db: tx}   // a new Queries whose DBTX is the transaction
}
```

So the pattern is: begin a tx on the pool, wrap your base `Queries` with `WithTx(tx)`, run all the queries through that tx-bound `qtx`, then commit (or roll back on any error).

### 7.3 The defer-rollback idiom — the correct, safe skeleton **[I]**

The single most important transaction pattern in Go+pgx is the **defer-rollback** idiom. You `defer tx.Rollback(ctx)` immediately after `Begin`. If the function returns early with an error, the deferred rollback undoes everything. If you reach `Commit` successfully, the later `Rollback` is a harmless no-op (pgx returns `ErrTxClosed`, which you ignore). This guarantees you never leak an open transaction, even on a panic or an early return you forgot about.

```go
func (s *Store) doInTx(ctx context.Context, fn func(*db.Queries) error) error {
	tx, err := s.pool.Begin(ctx)
	if err != nil {
		return fmt.Errorf("begin tx: %w", err)
	}
	// Deferred rollback: if we return before Commit for ANY reason, the tx is undone.
	// After a successful Commit, this Rollback is a no-op (tx already closed).
	defer tx.Rollback(ctx)

	qtx := s.q.WithTx(tx)     // same methods, now bound to the transaction
	if err := fn(qtx); err != nil {
		return err            // deferred Rollback fires -> nothing persisted
	}
	return tx.Commit(ctx)     // all-or-nothing commit
}
```

This `doInTx` helper is worth having in every project: it centralizes the correct pattern so individual operations just write the body.

### 7.4 A complete atomic money transfer **[I/A]**

Now the real thing: transfer money from account A to account B. The operation must, atomically: **lock both accounts**, verify the sender has funds, **decrement** A, **increment** B, write two **ledger entries**, and record a **transfer** row. Every step goes through `qtx` so it's all in one transaction.

A subtle but critical detail: **lock the two accounts in a consistent order** (e.g. by sorted UUID) to avoid a **deadlock** when two concurrent transfers touch the same pair in opposite directions.

```go
// TransferParams — the inputs to a transfer.
type TransferParams struct {
	FromAccountID uuid.UUID
	ToAccountID   uuid.UUID
	Amount        pgtype.Numeric // never float — arbitrary-precision money
	Reason        string
}

// TransferResult — everything the operation produced.
type TransferResult struct {
	Transfer    db.Transfer
	FromEntry   db.Entry
	ToEntry     db.Entry
	FromAccount db.Account
	ToAccount   db.Account
}

func (s *Store) Transfer(ctx context.Context, p TransferParams) (TransferResult, error) {
	var res TransferResult

	err := s.doInTx(ctx, func(qtx *db.Queries) error {
		// 1) Lock BOTH accounts in a deterministic order to prevent deadlocks.
		//    We use SELECT ... FOR UPDATE (the GetAccountForUpdate query, §5.5).
		firstID, secondID := p.FromAccountID, p.ToAccountID
		if firstID.String() > secondID.String() {
			firstID, secondID = secondID, firstID
		}
		if _, err := qtx.GetAccountForUpdate(ctx, firstID); err != nil {
			return fmt.Errorf("lock first account: %w", err)
		}
		if _, err := qtx.GetAccountForUpdate(ctx, secondID); err != nil {
			return fmt.Errorf("lock second account: %w", err)
		}

		// 2) Debit the sender. AddAccountBalance does
		//    UPDATE accounts SET balance = balance + $amount WHERE id = $id RETURNING *,
		//    with a CHECK/guard that balance stays >= 0 (enforced in SQL, see §11).
		from, err := qtx.AddAccountBalance(ctx, db.AddAccountBalanceParams{
			ID:     p.FromAccountID,
			Amount: negate(p.Amount), // subtract
		})
		if err != nil {
			return fmt.Errorf("debit sender: %w", err)
		}
		res.FromAccount = from

		// 3) Credit the receiver.
		to, err := qtx.AddAccountBalance(ctx, db.AddAccountBalanceParams{
			ID:     p.ToAccountID,
			Amount: p.Amount,
		})
		if err != nil {
			return fmt.Errorf("credit receiver: %w", err)
		}
		res.ToAccount = to

		// 4) Write the two ledger entries (double-entry bookkeeping).
		res.FromEntry, err = qtx.CreateEntry(ctx, db.CreateEntryParams{
			AccountID: p.FromAccountID, Amount: negate(p.Amount), Reason: p.Reason,
		})
		if err != nil {
			return err
		}
		res.ToEntry, err = qtx.CreateEntry(ctx, db.CreateEntryParams{
			AccountID: p.ToAccountID, Amount: p.Amount, Reason: p.Reason,
		})
		if err != nil {
			return err
		}

		// 5) Record the transfer itself.
		res.Transfer, err = qtx.CreateTransfer(ctx, db.CreateTransferParams{
			FromAccountID: p.FromAccountID,
			ToAccountID:   p.ToAccountID,
			Amount:        p.Amount,
		})
		return err
	})

	return res, err
}
```

Everything inside the closure runs on `qtx`. If any step returns an error, `doInTx`'s deferred `Rollback` fires and **nothing** persists — no half-transfer is ever possible. If all steps succeed, `Commit` makes them all durable at once.

### 7.5 Transaction isolation levels **[A]**

pgx lets you choose the isolation level when you begin. The default (`ReadCommitted`) is fine for the `FOR UPDATE`-locked approach above. For stricter guarantees — e.g. to prevent *phantom* anomalies in balance calculations that read many rows — use `Serializable`, and be prepared to **retry** on serialization failures (`40001`), which are normal and expected under Serializable.

```go
tx, err := s.pool.BeginTx(ctx, pgx.TxOptions{
	IsoLevel:   pgx.Serializable,
	AccessMode: pgx.ReadWrite,
})
```

> **Best practice:** for a ledger, the pragmatic recipe is **`ReadCommitted` + explicit `SELECT ... FOR UPDATE`** on the affected accounts (as in §7.4). It is easy to reason about, avoids most serialization retries, and the row locks provide the needed mutual exclusion. Reserve `Serializable` for read-heavy invariant checks where you can't enumerate the rows to lock, and wrap those in a retry loop.

### 7.6 Retry-on-serialization-failure helper **[A]**

If you do use `Serializable`, wrap the tx in a bounded retry, because `40001` is a *transient* error that succeeds on retry:

```go
func (s *Store) doInSerializableTx(ctx context.Context, fn func(*db.Queries) error) error {
	const maxRetries = 5
	for attempt := 0; attempt < maxRetries; attempt++ {
		tx, err := s.pool.BeginTx(ctx, pgx.TxOptions{IsoLevel: pgx.Serializable})
		if err != nil {
			return err
		}
		err = func() error {
			defer tx.Rollback(ctx)
			if err := fn(s.q.WithTx(tx)); err != nil {
				return err
			}
			return tx.Commit(ctx)
		}()
		if err == nil {
			return nil
		}
		var pgErr *pgconn.PgError
		if errors.As(err, &pgErr) && pgErr.Code == "40001" { // serialization_failure
			continue // transient: retry
		}
		return err // real error: give up
	}
	return errors.New("transaction failed after max retries")
}
```

### 7.7 Nested operations and passing `qtx` around **[I]**

A common design question: how do service methods compose transactions? The clean answer is **the transaction boundary lives at the top (the use-case/service layer), and lower functions accept a `*db.Queries`** — which may be pool-bound *or* tx-bound, they can't tell. So write helpers as `func(qtx *db.Queries, ...)`. This lets the same helper run standalone (pass `s.q`) or inside a larger transaction (pass the `qtx`), without duplicating logic. Do **not** begin a transaction inside a helper that might already be inside one — pgx does not nest real transactions (it offers *savepoints* via `tx.Begin()` on a tx, an advanced tool for partial rollback within a tx).

---

## 8. The Development Loop & CI: `generate`, `vet`, `diff`

### 8.1 The everyday inner loop **[B]**

Your minute-to-minute workflow is a tight cycle:

1. **Change the schema** — write a new goose migration and `goose up`.
2. **Regenerate** — `sqlc generate` (the models/methods now reflect the new schema).
3. **Write/adjust queries** — edit `db/query/*.sql`, `sqlc generate` again.
4. **Build** — `go build ./...`; any query/struct mismatch is now a compile error.
5. **Run** — via `go run` or, better, Air (§8.5) for auto-reload.

The invariant to internalize: **after every migration, regenerate.** If you add a column and forget to regenerate, your Go doesn't know about it (best case) or, worse, references stale columns. Automate this so you can't forget (§8.4, §8.5).

### 8.2 `sqlc generate` failure modes are your friend **[B]**

`sqlc generate` type-checks every query against the schema, so it catches whole classes of mistakes *before* runtime:

- Referencing a column that doesn't exist → error naming the column.
- A query the parser can't understand → error with the location.
- A type mismatch (e.g. comparing a `uuid` to an `int`) → error.

Treat a failing `sqlc generate` the way you treat a failing compile: fix it now. It is cheaper than any test.

### 8.3 `sqlc vet` — linting your SQL, including against a real database **[I/A]**

`sqlc vet` runs **lint rules** over your queries. There are two flavors, and the second is a genuine superpower for banking.

**(a) Static rules** are written in **CEL** (Common Expression Language) and inspect each query's structure — its SQL text, its parameters, its result columns. You define them in `sqlc.yaml` and can enforce policy across the whole codebase:

```yaml
version: "2"
rules:
  - name: no-select-star
    message: "SELECT * is banned — list columns explicitly."
    rule: "query.sql.matches('(?i)select\\\\s+\\\\*')"

  - name: no-delete-without-where
    message: "DELETE without WHERE is forbidden."
    rule: "query.sql.matches('(?i)^\\\\s*delete\\\\s+from') && !query.sql.matches('(?i)where')"

  - name: no-pg-float-for-money
    message: "Do not use float/double columns; money must be numeric."
    rule: "query.sql.matches('(?i)::(float|double)')"

sql:
  - engine: "postgresql"
    queries: "./db/query"
    schema:  "./db/migrations"
    rules:
      - no-select-star
      - no-delete-without-where
      - no-pg-float-for-money
    gen:
      go: { package: "db", out: "./db/gen", sql_package: "pgx/v5" }
```

**(b) Database-backed rules** are the killer feature. If you give sqlc a live database connection via a `database:` block, `sqlc vet` will **prepare (and can EXPLAIN) every query against the real database**. Preparing catches errors even static analysis misses; EXPLAIN lets you write rules on the *query plan* — e.g. "fail if any query does a sequential scan on `entries`" (i.e. is missing an index), which is exactly the kind of policy a bank wants enforced automatically.

```yaml
sql:
  - engine: "postgresql"
    queries: "./db/query"
    schema:  "./db/migrations"
    # A managed connection sqlc uses ONLY for vet's prepare/EXPLAIN.
    database:
      uri: "${DATABASE_URL}"   # env var; a throwaway CI Postgres with the schema applied
    rules:
      - sqlc/db-prepare         # built-in: prepare every query against the DB
      - no-seqscan-on-entries
    gen:
      go: { package: "db", out: "./db/gen", sql_package: "pgx/v5" }

rules:
  - name: no-seqscan-on-entries
    message: "Query does a Seq Scan on entries — add/why-missing an index."
    # EXPLAIN output is available to the rule when a database is configured.
    rule: >
      has(postgresql.explain) &&
      postgresql.explain.plan.matches('Seq Scan on entries')
```

The `sqlc/db-prepare` built-in rule catches "this query is valid against the schema sqlc parsed, but does it actually prepare on a real server?" — surfacing subtle issues (missing extensions, function signature mismatches). Run `sqlc vet` in CI against a fresh Postgres that has had `goose up` applied, so the vet DB matches the schema exactly.

> **⚡ Version note:** the exact CEL variables available (`query.sql`, `query.params`, `query.cmd`, and the `postgresql.explain` object when a `database:` is set) have grown across sqlc releases. Check `sqlc vet` docs for your version; the *concept* — static CEL rules plus optional DB-backed prepare/EXPLAIN — is stable.

### 8.4 `sqlc diff` — the stale-codegen gate **[I]**

`sqlc diff` regenerates in memory and compares against the committed files **without writing anything**. If they differ, it prints the diff and exits non-zero. This is the perfect **CI gate**: it fails the build if someone changed a query or a migration but forgot to run `sqlc generate` and commit the result. It guarantees the checked-in `db/gen/` always matches the checked-in SQL — no stale generated code ever reaches `main`.

```bash
# In CI, after checkout and (optionally) applying migrations to a scratch DB:
sqlc diff    # exit 0 = generated code is up to date; non-zero = someone forgot to regenerate
```

Pair `sqlc diff` (is the code fresh?) with `sqlc vet` (are the queries sane?) and you have a robust two-part gate.

### 8.5 A Makefile that ties it together **[I]**

Centralize the commands so the whole team runs them identically. On Windows you can use `make` via Git Bash / MSYS2, or translate these to a `Taskfile.yml` (Task runs natively on Windows). A representative `Makefile`:

```makefile
# Makefile
GOOSE_DIR   := db/migrations
DB_URL      := postgres://postgres:secret@localhost:5432/myledger?sslmode=disable

.PHONY: migrate-up migrate-down migrate-status generate vet diff check dev

migrate-up:      ## apply all pending migrations
	goose -dir $(GOOSE_DIR) postgres "$(DB_URL)" up

migrate-down:    ## roll back the latest migration
	goose -dir $(GOOSE_DIR) postgres "$(DB_URL)" down

migrate-status:  ## show migration status
	goose -dir $(GOOSE_DIR) postgres "$(DB_URL)" status

new-migration:   ## make NAME=create_foo new-migration
	goose -dir $(GOOSE_DIR) create $(NAME) sql

generate:        ## regenerate type-safe Go from SQL
	sqlc generate

vet:             ## lint queries (static + DB-backed if configured)
	sqlc vet

diff:            ## CI gate: fail if generated code is stale
	sqlc diff

# The one-shot after a schema change: migrate, regenerate, then build.
check: migrate-up generate vet diff
	go build ./...

dev:             ## live-reload dev server
	air
```

`make check` after any schema change does the whole safe sequence. In CI, run `migrate-up` against a scratch database, then `generate`/`vet`/`diff`/`go build`/`go test`.

### 8.6 Air integration — regenerate + reload **[I]**

**Air** watches your files and rebuilds/reruns the server on change. You want two things: (1) rebuild Go on `.go` changes, and (2) *regenerate sqlc* when you change a `.sql` query file, so the new method exists before the rebuild. Air's `pre_cmd` hook (run before each build) is the clean way to do this. A `.air.toml`:

```toml
# .air.toml
root = "."
tmp_dir = "tmp"

[build]
  # Regenerate sqlc BEFORE each build so query changes are reflected.
  pre_cmd = ["sqlc generate"]
  cmd = "go build -o ./tmp/api.exe ./cmd/api"   # .exe for Windows
  bin = "tmp/api.exe"
  # Watch SQL files too, so editing a query triggers pre_cmd (regen) + rebuild.
  include_ext = ["go", "sql", "yaml", "tmpl", "html"]
  exclude_dir = ["tmp", "db/gen", "vendor"]     # don't loop on generated output
  delay = 300     # ms debounce

[log]
  time = true
```

Two important details: **include `sql` in `include_ext`** so query edits trigger a regen, and **exclude `db/gen`** so Air doesn't loop forever reacting to its own generated files. Now: edit a query → Air runs `sqlc generate` → rebuilds → restarts, all automatically. Migrations still run manually via `make migrate-up` (you generally don't want auto-migrations on file save — that's a deliberate act).

> **Windows note:** Air needs the binary name to end in `.exe` (`bin = "tmp/api.exe"`, `cmd` outputs `api.exe`). Install Air with `go install github.com/air-verse/air@latest`.

### 8.7 The full CI pipeline sketch **[A]**

Putting it in a CI workflow (see **[GitHub Actions CI/CD](GITHUB_ACTIONS_CICD_GUIDE.md)** for the full syntax), the job:

1. Spin up a Postgres service container.
2. `goose up` the migrations against it (this is also a *migration smoke test* — do they even apply?).
3. `sqlc diff` — fail if generated code is stale.
4. `sqlc vet` (with `database: ${DATABASE_URL}` pointing at that Postgres) — prepare/EXPLAIN every query.
5. `go build ./...` and `go test ./...` (integration tests hit the same Postgres).

That pipeline makes it structurally impossible to merge: a migration that doesn't apply, generated code that's out of date, a query that doesn't prepare, or code that doesn't compile. That is the safety the sqlc+goose pairing buys you.

---

## 9. Coexisting with Ent — The Polyglot Data Layer

### 9.1 First, the honest question: do you even need both? **[A]**

This section shows how to run **sqlc and Ent in one process**, but before the "how," the "whether." **Most applications should pick one primary data layer.** Adding a second doubles the concepts your team must hold, and it means two code generators, two mental models, and two places a bug can hide. Reach for both **only** when you have a concrete, felt need — typically:

- Your app is **graph-shaped** in one area (users, roles, permissions, org hierarchies — lots of relation traversal and RBAC), where **Ent's edge traversal and eager loading** are a genuine productivity win; **and**
- It has **hot, SQL-heavy paths** elsewhere (a ledger, reports, analytics, precise `FOR UPDATE` money movement) where you want **exactly the SQL sqlc gives you**, with zero DSL between you and the planner.

If your app is *all* graph CRUD, use Ent alone (see **[Go ent ORM](GO_ENT_ORM_GUIDE.md)**). If it's *all* precise SQL, use sqlc alone. The polyglot setup below is for the genuine middle: **Ent for the relation-graph/RBAC surface, sqlc for the hot/complex/perf-critical surface** — a deliberate division of labor, not a default.

### 9.2 The non-negotiable rule: goose owns the schema **[A]**

The one thing that makes coexistence sane: **there is exactly one schema authority, and it is goose.** Both Ent and sqlc are *readers/consumers* of the schema goose defines; neither one *owns* it.

- **sqlc** reads the goose migrations directory as its schema source (as this whole guide describes). ✔ already aligned.
- **Ent** normally *wants* to own migrations (its Atlas-based auto/versioned migration). **You turn that off.** Ent's schema (its Go `ent/schema/*.go`) must be written to *match* what goose already created, and you never let Ent migrate the database. Goose is the single writer of DDL.

This avoids the disaster of two tools both trying to evolve the schema and fighting each other. One writer (goose), two readers (sqlc, Ent). (Practically: keep Ent's schema definitions consistent with the goose DDL by hand, and let `sqlc vet`/tests catch drift. Some teams even generate the Ent schema from the DB once and then freeze migrations off.)

### 9.3 One pgxpool, two query layers **[A]**

The elegant part: **both layers share a single `*pgxpool.Pool`.** One pool, one set of connections, one place to configure limits — used by both sqlc (natively) and Ent (via an adapter). This is efficient (no double pooling) and correct (both see the same database, honor the same limits).

- **sqlc** takes the pool directly: `db.New(pool)`.
- **Ent** needs a `database/sql`-style `*sql.DB`. pgx provides `stdlib.OpenDBFromPool(pool)` to wrap a `*pgxpool.Pool` as a `*sql.DB` **that shares the same underlying pool**. You then hand that to Ent via `entsql.OpenDB(dialect.Postgres, sqlDB)`.

```go
package main

import (
	"context"
	"log"

	"entgo.io/ent/dialect"
	entsql "entgo.io/ent/dialect/sql"
	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/jackc/pgx/v5/stdlib"

	"myledger/ent"           // Ent's generated client
	db "myledger/db/gen"     // sqlc's generated package
)

type App struct {
	Pool *pgxpool.Pool
	Q    *db.Queries   // sqlc query layer (hot/complex SQL)
	Ent  *ent.Client   // Ent client (relation-graph/RBAC CRUD)
}

func NewApp(ctx context.Context, dsn string) (*App, error) {
	// 1) The ONE pool. Both layers will use it.
	pool, err := pgxpool.New(ctx, dsn)
	if err != nil {
		return nil, err
	}

	// 2) sqlc gets the pool directly.
	q := db.New(pool)

	// 3) Ent gets the SAME pool, wrapped as *sql.DB, then as an ent driver.
	sqlDB := stdlib.OpenDBFromPool(pool)                 // shares pool, no new connections pool
	drv := entsql.OpenDB(dialect.Postgres, sqlDB)        // ent dialect driver over that *sql.DB
	entClient := ent.NewClient(ent.Driver(drv))

	// NOTE: we deliberately DO NOT call entClient.Schema.Create(ctx) — Ent's
	// migration is OFF. goose owns the schema. Ent only READS/WRITES rows.

	return &App{Pool: pool, Q: q, Ent: entClient}, nil
}
```

Now in a handler you choose the right tool per operation: `app.Ent.User.Query()...` for the RBAC/graph stuff, `app.Q.Transfer(...)` for the money movement. Same database, same pool, same transaction boundaries possible.

### 9.4 Sharing a transaction across both layers (advanced) **[A]**

Occasionally you need one transaction that spans an Ent write *and* a sqlc write. This is possible but fiddly, because Ent transacts over `*sql.DB` and sqlc transacts over `pgx.Tx` — two different transaction handles. The clean approaches, in order of preference:

1. **Don't.** Keep each business operation within one layer's transaction. If money movement is sqlc's job, do the whole atomic unit in sqlc's `WithTx` (§7). This is almost always the right call.
2. If you truly must span both, drive the transaction from the `database/sql` side (`sqlDB.BeginTx`), give Ent that tx (`entClient.Tx(ctx)` / `ent.NewClient(ent.Driver(entsql.OpenDB(..., tx)))`), and for sqlc use a `pgx.Tx` obtained from the *same* connection — which requires pulling a single `*pgx.Conn` from the pool and using it for both, bypassing the shared-pool convenience. This is genuinely advanced and error-prone; isolate it behind a well-tested helper and comment it heavily.

The pragmatic guidance: **design your aggregate boundaries so a single business transaction lives in a single layer.** Cross-layer transactions are a smell that your responsibilities are split at the wrong seam.

### 9.5 A clean division-of-labor example **[A]**

A concrete split for the ledger app:

| Concern | Layer | Why |
|---|---|---|
| User signup, profile, roles, permissions, "who can access account X" | **Ent** | Relation graph + RBAC traversal; Ent's edges and privacy policies shine. |
| Listing a user's accounts (simple) | **Ent** | Trivial CRUD; use whichever is already loaded. |
| Get account balance (hot path, called constantly) | **sqlc** | Perf-critical, want exact indexed query. |
| Atomic money transfer (`FOR UPDATE`, double-entry) | **sqlc** | Needs precise SQL + transaction control. |
| Ledger reports (window functions, running balances) | **sqlc** | SQL-heavy; ORM DSL would fight you. |
| Bulk import of historical entries | **sqlc** (`:copyfrom`) | `COPY` throughput. |

The seam is clear: **Ent handles the identity/authorization graph; sqlc handles the money.** Both over one pool, schema owned by goose.

---

## 10. Banking-Grade with sqlc

Everything in this section is about turning sqlc's properties into *policy* for a system where mistakes cost money and trust.

### 10.1 SQL injection is structurally impossible — and why **[A]**

The number-one web vulnerability class, SQL injection, comes from **building SQL by concatenating untrusted input into the query text**. sqlc makes this impossible by construction:

- Every query is **written ahead of time** in a `.sql` file and parsed at build time. The query *text* is a fixed Go `const` string. Nothing the user sends is ever concatenated into it.
- User input arrives only as **parameters** (`$1, $2, …`), which pgx sends to Postgres **separately** from the query text over the wire protocol. The database treats them strictly as *values*, never as SQL. A parameter of `'; DROP TABLE accounts; --` is just a string that gets compared to a column — it cannot become a statement.

So the very shape of sqlc — static SQL + separate parameters — eliminates the vulnerability rather than mitigating it. There is no "remember to escape" step to forget. (The only way to reintroduce injection is to *leave* sqlc and build dynamic SQL yourself with string concatenation — see §12.1 for how to handle dynamic needs *without* doing that.)

### 10.2 Row-level authorization baked into the query **[A]**

A subtle, expensive class of bugs is **broken object-level authorization** — user A fetching user B's account because the handler forgot to check ownership. sqlc lets you make the ownership check part of the *query itself*, so it's impossible to forget:

```sql
-- name: GetAccountOwned :one
-- The WHERE owner_id = $2 means a caller can ONLY ever read their own account.
-- If the owner doesn't match, the query returns pgx.ErrNoRows — not the row.
SELECT * FROM accounts
WHERE id = $1 AND owner_id = $2;
```

Always pass the *authenticated* owner id (from the verified JWT, never from the request body) as `$2`. Now authorization is enforced at the data layer, not left to a check a handler might skip. Do the same for `UPDATE`/`DELETE`: `... WHERE id = $1 AND owner_id = $2`. This is defense in depth — even if the HTTP-layer check is bypassed, the query returns nothing.

### 10.3 Money is `numeric`, NEVER float **[A]**

This is the cardinal rule of financial code, and sqlc + Postgres make it easy to honor. **Floating-point types (`float4`/`float8`/Go `float64`) cannot represent decimal money exactly** — `0.1 + 0.2 != 0.3` in binary floating point, and rounding errors accumulate into real discrepancies. Money must use an **exact decimal** type end to end:

- In Postgres: **`numeric(precision, scale)`** — e.g. `numeric(20,4)` for up to 16 integer digits and 4 fractional (enough for sub-cent). It is arbitrary-precision and exact.
- In Go via pgx: **`pgtype.Numeric`** (the default mapping) or a dedicated decimal library type via an override (§4.6). Both are exact.
- **Never** `overrides` a money column to `float64`. If you see `float` anywhere near money, that's a bug — and you can ban it with a `sqlc vet` rule (§8.3, the `no-pg-float-for-money` example).

Doing arithmetic on money is best done **in SQL** (`balance = balance + $1`) where Postgres does exact decimal math, rather than pulling into Go, converting, computing, and writing back (which invites both a conversion bug and a lost-update race).

### 10.4 `timestamptz` for all time **[A]**

Always store time as **`timestamptz`** (timestamp with time zone), which Postgres stores as UTC and converts on the way in/out. Never `timestamp` (without zone) for events — it's ambiguous across zones and DST. sqlc maps `timestamptz` to `pgtype.Timestamptz` or (via override) `time.Time`. For a ledger, every `created_at`, `posted_at`, `settled_at` is `timestamptz NOT NULL DEFAULT now()`.

### 10.5 Least-privilege database roles **[A]**

sqlc doesn't manage roles, but banking-grade deployment demands them, and it composes with the sqlc/goose split naturally:

- **Migration role** — the identity goose connects as. Owns the schema, can `CREATE`/`ALTER`/`DROP`. Used only during deploys/migrations.
- **Application role** — the identity the app's pgxpool connects as. Has *only* `SELECT`/`INSERT`/`UPDATE`/`DELETE` on the specific tables it needs, and **cannot** alter the schema. If the app is compromised, the blast radius is data, not structure.
- Optionally a **read-only role** for reporting/analytics connections.

Because sqlc's queries are static, you can even enumerate exactly which tables/columns the app touches and grant precisely those. See **[PostgreSQL](POSTGRESQL_GUIDE.md)** and **[Database Server Admin](DATABASE_SERVER_ADMIN_GUIDE.md)** for `GRANT`/`REVOKE` details.

### 10.6 Optimistic locking and `FOR UPDATE` **[A]**

Two accounts, two concurrent operations — how do you prevent a **lost update** (both read balance 100, both write 100−x, one write lost)? Two techniques, both expressible in sqlc:

- **Pessimistic — `SELECT ... FOR UPDATE`** (used in §7.4): the transaction locks the row; other transactions touching it block until you commit. Simple, correct, the right default for money movement.
- **Optimistic — a `version` column**: read the row with its version; on update, `... SET balance = $1, version = version + 1 WHERE id = $2 AND version = $3`. Use `:execrows`; if it affected **0 rows**, someone else updated it first — abort and retry. Great for low-contention rows where you'd rather not hold a lock.

```sql
-- name: UpdateAccountOptimistic :execrows
UPDATE accounts
SET balance = sqlc.arg(balance), version = version + 1
WHERE id = sqlc.arg(id) AND version = sqlc.arg(expected_version);
```

```go
n, err := q.UpdateAccountOptimistic(ctx, params)
if err != nil { return err }
if n == 0 { return ErrConcurrentModification } // caller retries
```

### 10.7 Idempotency keys **[A]**

Network retries mean the same "transfer $50" request can arrive twice. Charging twice is unacceptable. An **idempotency key** — a client-supplied unique id per logical operation — makes retries safe:

- The client sends `Idempotency-Key: <uuid>` with the transfer.
- The server, **inside the transfer transaction**, inserts that key into an `idempotency_keys` table with a `UNIQUE` constraint (or upserts). If the insert **conflicts**, the operation already ran — return the *stored* prior result instead of doing it again.

```sql
-- name: ClaimIdempotencyKey :one
-- Returns the row; ON CONFLICT DO NOTHING means a duplicate claim inserts nothing.
-- Use :one + check pgx.ErrNoRows to detect "already claimed".
INSERT INTO idempotency_keys (key, owner_id, request_hash)
VALUES (sqlc.arg(key), sqlc.arg(owner_id), sqlc.arg(request_hash))
ON CONFLICT (key) DO NOTHING
RETURNING *;
```

If `ClaimIdempotencyKey` returns `pgx.ErrNoRows`, the key was already used — look up and return the earlier response. Because the claim and the transfer are in the **same transaction**, they commit or roll back together: no way to consume the key without doing the work, or vice versa.

### 10.8 Audit columns and append-only ledgers **[A]**

A ledger's integrity rests on being **append-only**: you never `UPDATE` or `DELETE` an `entries` row — a correction is a *new*, compensating entry. Enforce and record:

- **Audit columns** on mutable tables: `created_at`, `updated_at`, `created_by`, `updated_by` — the `_by` from the authenticated JWT subject.
- **Immutable entries**: grant the app role `INSERT`/`SELECT` but **not** `UPDATE`/`DELETE` on `entries`. The database enforces append-only regardless of application bugs.
- **A running-balance invariant**: the sum of an account's entries must equal its `balance`. Assert it in tests and, periodically, with a reconciliation query.

### 10.9 The banking checklist **[A]**

| Control | How, with sqlc/goose |
|---|---|
| No SQL injection | Inherent — static SQL + separate params. |
| Object-level authz | `WHERE owner_id = $auth` in every read/write query. |
| Exact money | `numeric` columns, `pgtype.Numeric`/decimal in Go, never float. |
| Correct time | `timestamptz`, UTC. |
| No lost updates | `FOR UPDATE` or a `version` column (`:execrows`). |
| Atomicity | pgx transactions via `WithTx` + defer-rollback. |
| Idempotency | Unique idempotency-key table, claimed in the same tx. |
| Append-only ledger | Revoke `UPDATE`/`DELETE` on `entries`; compensating entries. |
| Least privilege | Separate migration vs app DB roles. |
| Enforced policy | `sqlc vet` CEL rules (ban `SELECT *`, ban float, require WHERE). |
| No stale code | `sqlc diff` gate in CI. |
| Query-plan discipline | DB-backed `sqlc vet` EXPLAIN rules (no unexpected seq scans). |

---

## 11. Complete Worked Example: an Accounts + Ledger Service

This section assembles everything into one coherent, runnable-looking service: goose migrations, sqlc queries, a pgxpool, Argon2id + JWT auth for the account owner, and Gin handlers — including the atomic transfer.

### 11.1 The schema, as goose migrations **[I]**

Three migrations. Note: `numeric` money, `timestamptz` time, a `CHECK` keeping balances non-negative, indexes for the keyset pagination, and an idempotency table.

```sql
-- db/migrations/00001_create_users.sql
-- +goose Up
-- +goose StatementBegin
CREATE EXTENSION IF NOT EXISTS pgcrypto;   -- for gen_random_uuid()

CREATE TABLE users (
	id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
	email         text NOT NULL UNIQUE,
	password_hash text NOT NULL,            -- Argon2id encoded hash (see §11.4)
	created_at    timestamptz NOT NULL DEFAULT now()
);
-- +goose StatementEnd

-- +goose Down
-- +goose StatementBegin
DROP TABLE users;
-- +goose StatementEnd
```

```sql
-- db/migrations/00002_create_accounts.sql
-- +goose Up
-- +goose StatementBegin
CREATE TABLE accounts (
	id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
	owner_id   uuid NOT NULL REFERENCES users(id),
	balance    numeric(20,4) NOT NULL DEFAULT 0 CHECK (balance >= 0),
	currency   text NOT NULL,
	version    bigint NOT NULL DEFAULT 0,        -- for optimistic locking
	created_at timestamptz NOT NULL DEFAULT now(),
	updated_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_accounts_owner ON accounts (owner_id);
-- +goose StatementEnd

-- +goose Down
-- +goose StatementBegin
DROP TABLE accounts;
-- +goose StatementEnd
```

```sql
-- db/migrations/00003_create_ledger.sql
-- +goose Up
-- +goose StatementBegin
-- Double-entry ledger: every money movement writes entries and a transfer row.
CREATE TABLE entries (
	id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
	account_id uuid NOT NULL REFERENCES accounts(id),
	amount     numeric(20,4) NOT NULL,     -- signed: negative = debit, positive = credit
	reason     text NOT NULL,
	created_at timestamptz NOT NULL DEFAULT now()
);
-- Index supporting keyset pagination (§5.6): newest-first within an account.
CREATE INDEX idx_entries_account_created ON entries (account_id, created_at DESC, id DESC);

CREATE TABLE transfers (
	id              bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
	from_account_id uuid NOT NULL REFERENCES accounts(id),
	to_account_id   uuid NOT NULL REFERENCES accounts(id),
	amount          numeric(20,4) NOT NULL CHECK (amount > 0),
	created_at      timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE idempotency_keys (
	key          uuid PRIMARY KEY,
	owner_id     uuid NOT NULL REFERENCES users(id),
	request_hash text NOT NULL,
	created_at   timestamptz NOT NULL DEFAULT now()
);
-- +goose StatementEnd

-- +goose Down
-- +goose StatementBegin
DROP TABLE idempotency_keys;
DROP TABLE transfers;
DROP TABLE entries;
-- +goose StatementEnd
```

Apply them: `goose -dir db/migrations postgres "$DB_URL" up`. sqlc will read these same files as its schema.

### 11.2 The queries **[I]**

```sql
-- db/query/users.sql

-- name: CreateUser :one
INSERT INTO users (email, password_hash)
VALUES (sqlc.arg(email), sqlc.arg(password_hash))
RETURNING id, email, created_at;

-- name: GetUserByEmail :one
SELECT id, email, password_hash, created_at
FROM users
WHERE email = $1;
```

```sql
-- db/query/accounts.sql

-- name: CreateAccount :one
INSERT INTO accounts (owner_id, currency)
VALUES (sqlc.arg(owner_id), sqlc.arg(currency))
RETURNING id, owner_id, balance, currency, version, created_at, updated_at;

-- name: GetAccountOwned :one
-- Object-level authorization baked in: only the owner can read it (§10.2).
SELECT id, owner_id, balance, currency, version, created_at, updated_at
FROM accounts
WHERE id = sqlc.arg(id) AND owner_id = sqlc.arg(owner_id);

-- name: GetAccountForUpdate :one
-- Row lock for the transfer transaction (§7.4). No owner check here — the
-- caller (the transfer service) has already authorized via the owned lookup.
SELECT id, owner_id, balance, currency, version, created_at, updated_at
FROM accounts
WHERE id = $1
FOR UPDATE;

-- name: AddAccountBalance :one
-- Exact decimal math done IN SQL. The CHECK(balance >= 0) constraint makes an
-- overdraft fail at the database, aborting the transaction (§10.3, §10.6).
UPDATE accounts
SET balance = balance + sqlc.arg(amount),
    updated_at = now(),
    version = version + 1
WHERE id = sqlc.arg(id)
RETURNING id, owner_id, balance, currency, version, created_at, updated_at;

-- name: ListAccountsByOwner :many
SELECT id, owner_id, balance, currency, version, created_at, updated_at
FROM accounts
WHERE owner_id = $1
ORDER BY created_at DESC;
```

```sql
-- db/query/ledger.sql

-- name: CreateEntry :one
INSERT INTO entries (account_id, amount, reason)
VALUES (sqlc.arg(account_id), sqlc.arg(amount), sqlc.arg(reason))
RETURNING id, account_id, amount, reason, created_at;

-- name: CreateTransfer :one
INSERT INTO transfers (from_account_id, to_account_id, amount)
VALUES (sqlc.arg(from_account_id), sqlc.arg(to_account_id), sqlc.arg(amount))
RETURNING id, from_account_id, to_account_id, amount, created_at;

-- name: ListEntriesKeyset :many
-- Keyset pagination (§5.6): stable, index-backed, O(page) at any depth.
SELECT id, account_id, amount, reason, created_at
FROM entries
WHERE account_id = sqlc.arg(account_id)
  AND (created_at, id) < (sqlc.arg(cursor_created_at), sqlc.arg(cursor_id))
ORDER BY created_at DESC, id DESC
LIMIT sqlc.arg(page_size);

-- name: ClaimIdempotencyKey :one
INSERT INTO idempotency_keys (key, owner_id, request_hash)
VALUES (sqlc.arg(key), sqlc.arg(owner_id), sqlc.arg(request_hash))
ON CONFLICT (key) DO NOTHING
RETURNING key, owner_id, request_hash, created_at;
```

Run `sqlc generate`. You now have `db.Queries` with all these methods, `db.Account`, `db.Entry`, `db.Transfer`, `db.User`, and their Params/Row structs.

### 11.3 The config for this project **[I]**

```yaml
# sqlc.yaml
version: "2"
sql:
  - engine: "postgresql"
    queries: "./db/query"
    schema:  "./db/migrations"
    database:
      uri: "${DATABASE_URL}"   # used by `sqlc vet` in CI for prepare/EXPLAIN
    rules:
      - sqlc/db-prepare
    gen:
      go:
        package: "db"
        out: "./db/gen"
        sql_package: "pgx/v5"
        emit_json_tags: true
        json_tags_case_style: "snake"
        emit_pointers_for_null_types: true
        emit_interface: true
        emit_empty_slices: true
        overrides:
          - db_type: "uuid"
            go_type: "github.com/google/uuid.UUID"
          - db_type: "timestamptz"
            go_type: "time.Time"
          - db_type: "numeric"
            go_type:
              import: "github.com/jackc/pgx/v5/pgtype"
              type: "Numeric"
```

### 11.4 Auth: Argon2id password hashing + JWT **[I/A]**

Full treatment is in **[Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md)**; here is the essential wiring used by the handlers. **Argon2id** is the memory-hard password hash; we store the full encoded string (params + salt + hash) in `users.password_hash`.

```go
// internal/auth/password.go
package auth

import (
	"crypto/rand"
	"crypto/subtle"
	"encoding/base64"
	"errors"
	"fmt"
	"strings"

	"golang.org/x/crypto/argon2"
)

// Tune per your hardware; these are sane 2026 defaults for a server.
const (
	argonTime    = 3         // iterations
	argonMemory  = 64 * 1024 // 64 MB
	argonThreads = 2
	argonKeyLen  = 32
	argonSaltLen = 16
)

func HashPassword(password string) (string, error) {
	salt := make([]byte, argonSaltLen)
	if _, err := rand.Read(salt); err != nil {
		return "", err
	}
	key := argon2.IDKey([]byte(password), salt, argonTime, argonMemory, argonThreads, argonKeyLen)
	// Standard PHC-style encoded string.
	return fmt.Sprintf("$argon2id$v=%d$m=%d,t=%d,p=%d$%s$%s",
		argon2.Version, argonMemory, argonTime, argonThreads,
		base64.RawStdEncoding.EncodeToString(salt),
		base64.RawStdEncoding.EncodeToString(key),
	), nil
}

func VerifyPassword(password, encoded string) (bool, error) {
	parts := strings.Split(encoded, "$")
	if len(parts) != 6 || parts[1] != "argon2id" {
		return false, errors.New("bad hash format")
	}
	var mem, t uint32
	var p uint8
	if _, err := fmt.Sscanf(parts[3], "m=%d,t=%d,p=%d", &mem, &t, &p); err != nil {
		return false, err
	}
	salt, err := base64.RawStdEncoding.DecodeString(parts[4])
	if err != nil {
		return false, err
	}
	want, err := base64.RawStdEncoding.DecodeString(parts[5])
	if err != nil {
		return false, err
	}
	got := argon2.IDKey([]byte(password), salt, t, mem, p, uint32(len(want)))
	// Constant-time compare to avoid timing attacks.
	return subtle.ConstantTimeCompare(got, want) == 1, nil
}
```

```go
// internal/auth/jwt.go
package auth

import (
	"time"

	"github.com/golang-jwt/jwt/v5"
	"github.com/google/uuid"
)

type Claims struct {
	jwt.RegisteredClaims
	UserID uuid.UUID `json:"uid"`
}

func IssueToken(secret []byte, userID uuid.UUID, ttl time.Duration) (string, error) {
	now := time.Now()
	claims := Claims{
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   userID.String(),
			IssuedAt:  jwt.NewNumericDate(now),
			ExpiresAt: jwt.NewNumericDate(now.Add(ttl)),
		},
		UserID: userID,
	}
	tok := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return tok.SignedString(secret)
}

func ParseToken(secret []byte, tokenStr string) (*Claims, error) {
	claims := &Claims{}
	_, err := jwt.ParseWithClaims(tokenStr, claims, func(t *jwt.Token) (interface{}, error) {
		if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
			return nil, jwt.ErrTokenSignatureInvalid // reject alg confusion
		}
		return secret, nil
	})
	if err != nil {
		return nil, err
	}
	return claims, nil
}
```

### 11.5 Wiring pgxpool + the store + Gin **[I/A]**

```go
// internal/store/store.go
package store

import (
	"context"
	"errors"
	"fmt"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgtype"
	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/google/uuid"

	db "myledger/db/gen"
)

type Store struct {
	pool *pgxpool.Pool
	q    *db.Queries
}

func New(pool *pgxpool.Pool) *Store {
	return &Store{pool: pool, q: db.New(pool)}
}

// Q exposes the pool-bound queries for simple, non-transactional reads.
func (s *Store) Q() *db.Queries { return s.q }

// doInTx — the defer-rollback transaction helper (§7.3).
func (s *Store) doInTx(ctx context.Context, fn func(*db.Queries) error) error {
	tx, err := s.pool.Begin(ctx)
	if err != nil {
		return fmt.Errorf("begin: %w", err)
	}
	defer tx.Rollback(ctx)
	if err := fn(s.q.WithTx(tx)); err != nil {
		return err
	}
	return tx.Commit(ctx)
}
```

```go
// internal/store/transfer.go
package store

import (
	"context"
	"errors"
	"fmt"

	"github.com/google/uuid"
	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgtype"

	db "myledger/db/gen"
)

var ErrDuplicateRequest = errors.New("idempotent replay")

type TransferInput struct {
	IdempotencyKey uuid.UUID
	OwnerID        uuid.UUID      // authenticated user (from JWT)
	FromAccountID  uuid.UUID
	ToAccountID    uuid.UUID
	Amount         pgtype.Numeric // exact decimal
	RequestHash    string
}

type TransferOutput struct {
	Transfer  db.Transfer
	FromEntry db.Entry
	ToEntry   db.Entry
}

func (s *Store) Transfer(ctx context.Context, in TransferInput) (TransferOutput, error) {
	var out TransferOutput

	err := s.doInTx(ctx, func(q *db.Queries) error {
		// (a) Idempotency: claim the key inside the tx (§10.7).
		_, err := q.ClaimIdempotencyKey(ctx, db.ClaimIdempotencyKeyParams{
			Key:         in.IdempotencyKey,
			OwnerID:     in.OwnerID,
			RequestHash: in.RequestHash,
		})
		if errors.Is(err, pgx.ErrNoRows) {
			return ErrDuplicateRequest // already processed; handler returns the prior result
		}
		if err != nil {
			return fmt.Errorf("claim idempotency: %w", err)
		}

		// (b) Authorize: the sender account must belong to the caller.
		if _, err := q.GetAccountOwned(ctx, db.GetAccountOwnedParams{
			ID: in.FromAccountID, OwnerID: in.OwnerID,
		}); err != nil {
			return fmt.Errorf("sender not owned/found: %w", err)
		}

		// (c) Lock both accounts in deterministic order (deadlock avoidance, §7.4).
		a, b := in.FromAccountID, in.ToAccountID
		if a.String() > b.String() {
			a, b = b, a
		}
		if _, err := q.GetAccountForUpdate(ctx, a); err != nil {
			return err
		}
		if _, err := q.GetAccountForUpdate(ctx, b); err != nil {
			return err
		}

		// (d) Debit sender (CHECK(balance>=0) aborts on overdraft).
		neg := negateNumeric(in.Amount)
		if _, err := q.AddAccountBalance(ctx, db.AddAccountBalanceParams{ID: in.FromAccountID, Amount: neg}); err != nil {
			return fmt.Errorf("debit: %w", err)
		}
		// (e) Credit receiver.
		if _, err := q.AddAccountBalance(ctx, db.AddAccountBalanceParams{ID: in.ToAccountID, Amount: in.Amount}); err != nil {
			return fmt.Errorf("credit: %w", err)
		}

		// (f) Double-entry ledger rows.
		if out.FromEntry, err = q.CreateEntry(ctx, db.CreateEntryParams{
			AccountID: in.FromAccountID, Amount: neg, Reason: "transfer",
		}); err != nil {
			return err
		}
		if out.ToEntry, err = q.CreateEntry(ctx, db.CreateEntryParams{
			AccountID: in.ToAccountID, Amount: in.Amount, Reason: "transfer",
		}); err != nil {
			return err
		}

		// (g) Record the transfer.
		out.Transfer, err = q.CreateTransfer(ctx, db.CreateTransferParams{
			FromAccountID: in.FromAccountID, ToAccountID: in.ToAccountID, Amount: in.Amount,
		})
		return err
	})

	return out, err
}
```

The `negateNumeric` helper flips a `pgtype.Numeric`'s sign (exact — no float):

```go
func negateNumeric(n pgtype.Numeric) pgtype.Numeric {
	out := n
	if out.Int != nil {
		neg := new(big.Int).Neg(out.Int) // exact integer mantissa negation
		out.Int = neg
	}
	return out
}
```

### 11.6 The Gin handlers **[I]**

Full Gin patterns are in **[Go Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md)**; here are the ledger-specific handlers, including the auth middleware that puts the verified user id in the context.

```go
// internal/api/router.go
package api

import (
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/google/uuid"
	"github.com/jackc/pgx/v5/pgtype"

	"myledger/internal/auth"
	"myledger/internal/store"
)

type Server struct {
	st     *store.Store
	secret []byte
}

func NewServer(st *store.Store, secret []byte) *gin.Engine {
	s := &Server{st: st, secret: secret}
	r := gin.New()
	r.Use(gin.Recovery(), gin.Logger())

	r.POST("/signup", s.signup)
	r.POST("/login", s.login)

	authed := r.Group("/", s.requireAuth())
	authed.POST("/accounts", s.createAccount)
	authed.GET("/accounts/:id", s.getAccount)
	authed.GET("/accounts/:id/entries", s.listEntries)
	authed.POST("/transfers", s.createTransfer)
	return r
}

// requireAuth verifies the Bearer JWT and stashes the user id in the context.
func (s *Server) requireAuth() gin.HandlerFunc {
	return func(c *gin.Context) {
		h := c.GetHeader("Authorization")
		const p = "Bearer "
		if len(h) <= len(p) || h[:len(p)] != p {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "missing token"})
			return
		}
		claims, err := auth.ParseToken(s.secret, h[len(p):])
		if err != nil {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid token"})
			return
		}
		c.Set("userID", claims.UserID)
		c.Next()
	}
}

func currentUser(c *gin.Context) uuid.UUID { return c.MustGet("userID").(uuid.UUID) }
```

```go
// internal/api/handlers.go
package api

import (
	"errors"
	"net/http"

	"github.com/gin-gonic/gin"
	"github.com/google/uuid"
	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgtype"

	"myledger/internal/auth"
	"myledger/internal/store"
	db "myledger/db/gen"
)

func (s *Server) signup(c *gin.Context) {
	var body struct {
		Email    string `json:"email" binding:"required,email"`
		Password string `json:"password" binding:"required,min=12"`
	}
	if err := c.ShouldBindJSON(&body); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	hash, err := auth.HashPassword(body.Password)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "hash failed"})
		return
	}
	u, err := s.st.Q().CreateUser(c, db.CreateUserParams{Email: body.Email, PasswordHash: hash})
	if err != nil {
		c.JSON(http.StatusConflict, gin.H{"error": "email taken"})
		return
	}
	c.JSON(http.StatusCreated, u)
}

func (s *Server) createTransfer(c *gin.Context) {
	var body struct {
		IdempotencyKey uuid.UUID `json:"idempotency_key" binding:"required"`
		FromAccountID  uuid.UUID `json:"from_account_id" binding:"required"`
		ToAccountID    uuid.UUID `json:"to_account_id" binding:"required"`
		Amount         string    `json:"amount" binding:"required"` // decimal string, never float
	}
	if err := c.ShouldBindJSON(&body); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	var amt pgtype.Numeric
	if err := amt.Scan(body.Amount); err != nil { // parse the decimal string exactly
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid amount"})
		return
	}

	out, err := s.st.Transfer(c, store.TransferInput{
		IdempotencyKey: body.IdempotencyKey,
		OwnerID:        currentUser(c), // authenticated — NOT from the body
		FromAccountID:  body.FromAccountID,
		ToAccountID:    body.ToAccountID,
		Amount:         amt,
		RequestHash:    hashRequest(body),
	})
	switch {
	case errors.Is(err, store.ErrDuplicateRequest):
		c.JSON(http.StatusOK, gin.H{"status": "already processed"})
	case err != nil:
		c.JSON(http.StatusUnprocessableEntity, gin.H{"error": err.Error()})
	default:
		c.JSON(http.StatusCreated, out)
	}
}

func (s *Server) getAccount(c *gin.Context) {
	id, err := uuid.Parse(c.Param("id"))
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "bad id"})
		return
	}
	// Object-level authorization is IN the query (§10.2).
	acct, err := s.st.Q().GetAccountOwned(c, db.GetAccountOwnedParams{
		ID: id, OwnerID: currentUser(c),
	})
	if errors.Is(err, pgx.ErrNoRows) {
		c.JSON(http.StatusNotFound, gin.H{"error": "not found"}) // also covers "not yours"
		return
	}
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "db error"})
		return
	}
	c.JSON(http.StatusOK, acct)
}
```

### 11.7 `main.go` — pgxpool tuning + graceful start **[I]**

```go
// cmd/api/main.go
package main

import (
	"context"
	"log"
	"os"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"

	"myledger/internal/api"
	"myledger/internal/store"
)

func main() {
	ctx := context.Background()

	cfg, err := pgxpool.ParseConfig(os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Fatal(err)
	}
	// Pool tuning for a service. Match MaxConns to your DB's connection budget.
	cfg.MaxConns = 20
	cfg.MinConns = 2
	cfg.MaxConnLifetime = time.Hour
	cfg.MaxConnIdleTime = 30 * time.Minute
	cfg.HealthCheckPeriod = time.Minute

	pool, err := pgxpool.NewWithConfig(ctx, cfg)
	if err != nil {
		log.Fatal(err)
	}
	defer pool.Close()
	if err := pool.Ping(ctx); err != nil {
		log.Fatal("db unreachable: ", err)
	}

	st := store.New(pool)
	r := api.NewServer(st, []byte(os.Getenv("JWT_SECRET")))

	log.Println("listening on :8080")
	if err := r.Run(":8080"); err != nil {
		log.Fatal(err)
	}
}
```

### 11.8 Running it end to end **[I]**

```bash
# 1) Migrate (schema truth) — sqlc reads these same files.
$env:GOOSE_DRIVER   = "postgres"
$env:GOOSE_DBSTRING = "host=localhost user=postgres password=secret dbname=myledger sslmode=disable"
goose -dir db/migrations up

# 2) Generate the type-safe Go.
sqlc generate

# 3) Vet the queries (optionally DB-backed).
sqlc vet

# 4) Run (Air for reload, or plain go run).
$env:DATABASE_URL = "postgres://postgres:secret@localhost:5432/myledger"
$env:JWT_SECRET   = "change-me-32-bytes-minimum-secret"
air     # or: go run ./cmd/api
```

You now have a coherent, banking-shaped service: goose-owned schema, sqlc-generated type-safe access, atomic idempotent transfers with row locks and exact decimal money, Argon2id+JWT auth, all over one tuned pgxpool.

---

## 12. Gotchas & Best Practices

### 12.1 sqlc cannot do dynamic SQL — and the right workarounds **[A]**

This is the single most important limitation to internalize. Because sqlc parses each query at build time, **it cannot generate code for SQL that isn't known until runtime**:

- **Dynamic column or table names** — `ORDER BY <user-chosen column>`, `SELECT FROM <dynamic table>`. sqlc has no query to parse.
- **Fully dynamic WHERE** — an advanced-search form where the user checks any subset of 10 filters, producing 2¹⁰ possible WHERE clauses. You cannot write 1024 queries.

The wrong "fix" is to concatenate SQL strings yourself — that throws away sqlc's injection safety and type checking. The right workarounds, in order of preference:

1. **`sqlc.narg` + `COALESCE`** (§5.3) — for optional *filters* with a fixed set of columns. Pass `NULL` to skip a filter. Covers most "advanced search" needs cleanly and stays fully type-safe. This handles the large majority of "dynamic WHERE" cases in practice.
2. **A small fixed set of named queries** — for a handful of sort options, write `ListByCreated`, `ListByAmount`, etc., and `switch` in Go. Explicit, safe, and the query plans are individually inspectable.
3. **Whitelist + safe interpolation for dynamic identifiers only** — if you truly must order by a user-chosen column, map the user's string through a **hard-coded allowlist** to a fixed column name (`map[string]string{"amount": "amount", "date": "created_at"}`), and only ever inject the *allowlisted constant*, never the raw input. Combine with sqlc for the parameterized parts.
4. **Drop to a query builder (`squirrel`) or raw pgx for the genuinely dynamic 5%** — using the *same pool*. sqlc is not all-or-nothing; use it for the 95% of static queries and a builder for the rare dynamic one. Keep the builder code small, reviewed, and parameterized.

The mental model: **sqlc owns your static queries (almost everything); a narrow, well-guarded escape hatch owns the truly dynamic few.** Do not contort sqlc into dynamic SQL, and do not abandon it because 5% of queries are dynamic.

### 12.2 ALWAYS regenerate after a migration **[A]**

The most common day-to-day mistake: change a migration, run `goose up`, then forget `sqlc generate`. Now your Go compiles against a stale schema view. Symptoms range from "the new column isn't on the struct" to subtle type mismatches. **Defenses:** (1) the `make check` target that always runs migrate→generate→build together; (2) Air's `pre_cmd = ["sqlc generate"]` so query edits auto-regenerate; (3) most importantly, the **`sqlc diff` CI gate** (§8.4) that fails the build if committed generated code doesn't match the committed SQL. With the CI gate, a forgotten regenerate can never reach `main`.

### 12.3 Migration files must be standalone-valid DDL for sqlc's parser **[A]**

sqlc parses your goose migrations with its *own* SQL parser (it does not call Postgres). That means every Up statement must be something sqlc understands. Almost all ordinary DDL parses fine, but watch for:

- **`CREATE FUNCTION` / `DO $$ ... $$` blocks** — wrap them in `-- +goose StatementBegin/End` (which you should do for goose anyway), and be aware sqlc may not deeply parse the function *body* (it doesn't need to — it needs the schema shape). Keep application-relevant DDL (tables, columns, types) outside opaque function bodies.
- **Very new/exotic Postgres syntax** — if you use bleeding-edge DDL that sqlc's parser version doesn't yet know, `sqlc generate` errors. Usually a sqlc upgrade fixes it; occasionally you refactor the migration.
- **Data-only migrations** (`INSERT`/`UPDATE` seed data, backfills) — these change data, not schema. sqlc ignores their schema effect. That's fine, but don't put a `CREATE TABLE` your queries depend on *only* inside a conditionally-run block sqlc can't see.

> **⚡ Version note:** sqlc's parser coverage of Postgres DDL improves every release. If a valid migration fails to parse, first try upgrading sqlc.

### 12.4 goose-annotation parsing quirks in the schema dir **[A]**

When sqlc reads the goose directory, it applies **only the `-- +goose Up` sections**, in **filename order**. Consequences to keep in mind:

- **Filename order is schema order.** Sequential prefixes (`00001_`, `00002_`) sort correctly; timestamp prefixes also sort correctly. Don't mix styles in one directory or ordering gets confusing.
- **A later migration's `ALTER`/`DROP` is honored.** If `00007` drops a column, sqlc's built schema won't have it either — matching reality. Good.
- **`Down` sections are ignored by sqlc** (they'd un-build the schema). Keep them correct for goose's sake, but know sqlc doesn't read them.
- **Non-`.sql` files** in the directory (e.g. Go-based goose migrations) are **not** read by sqlc as schema. If you use goose *Go migrations* for data backfills, keep all *DDL* in `.sql` migrations so sqlc sees it.

### 12.5 pgx v5 null-handling surprises **[A]**

- A `NOT NULL` column is a plain Go type; a nullable column is `pgtype.X` or `*X`. Mixing them up in your head leads to "why is this a pointer?" confusion — check the migration's `NOT NULL`.
- `pgtype.Numeric`'s zero value is **not** `0`; it's an *invalid/null* Numeric. To represent zero money you must set it explicitly (`n.Scan("0")` or build it). Don't assume a zero-valued struct means "0.00".
- Scanning a SQL `NULL` into a non-nullable target errors at runtime. If a column *can* be null, make sure the schema says so (nullable) so sqlc gives you a nullable Go type.
- `COALESCE(x, y)` changes nullability: `COALESCE(nullable_col, 0)` is non-null, so sqlc infers a non-null Go type for that result column. Use this deliberately to simplify result types.

### 12.6 Keep `overrides` consistent everywhere **[A]**

If you have multiple `sql:` targets, an `overrides` block in one but not another means the *same* DB type maps to different Go types in different packages — e.g. `uuid` is `uuid.UUID` in one and `pgtype.UUID` in another. Passing values between them then requires conversions and invites bugs. **Copy the overrides block verbatim into every target**, or factor a shared config. Also: an override's Go type must be pgx-scannable/encodable; a typo'd import path fails at `sqlc generate`.

### 12.7 `emit_pointers_for_null_types` tradeoffs, revisited **[A]**

Turning pointers on gives clean JSON and idiomatic nil-checks, but: (1) every nullable field is now a pointer, so you write `*p.Field` (with nil-checks) instead of `p.Field.Int64`; (2) constructing params for nullable fields means taking addresses (`ptr("x")` helper); (3) a nil-pointer deref is a panic if you forget to check. Turning pointers off (pgtype) is more verbose but harder to crash. Neither is wrong — **decide once, project-wide, and document it.** Flipping it later is a big regenerate-and-fix-callers churn.

### 12.8 Don't `SELECT *` in shipped queries **[A]**

`SELECT *` couples your generated struct to the *current* column set and order. Add a column via migration and every `SELECT *` result struct silently gains a field (usually harmless, occasionally surprising in JSON output or when a column is large/sensitive like `password_hash` leaking into an API response). **List columns explicitly** in production queries; reserve `*` for quick experiments. Enforce with a `sqlc vet` rule (§8.3). (Teaching snippets in early sections use `*` for brevity; §11's real queries enumerate columns.)

### 12.9 Commit generated code; don't hand-edit it **[A]**

Commit `db/gen/` so builds and CI don't require sqlc installed and so diffs are reviewable. But treat it as read-only — every file says `DO NOT EDIT`. If you want different output, change the config or the SQL and regenerate. Hand-edits are silently destroyed on the next `sqlc generate`. The `sqlc diff` gate also enforces "committed == generated."

### 12.10 Transaction and locking pitfalls **[A]**

- **Forgetting `defer tx.Rollback`** leaks a connection on the error path. Always use the `doInTx` helper (§7.3).
- **Inconsistent lock ordering** across concurrent transfers causes deadlocks. Always lock accounts in a deterministic order (§7.4).
- **Holding a transaction open across slow work** (HTTP calls, heavy computation) holds row locks and starves the pool. Do slow work *outside* the transaction; keep transactions short.
- **Doing money math in Go** (read balance → compute in Go → write back) races and risks float bugs. Prefer `SET balance = balance + $1` *in SQL* inside the locked transaction.

### 12.11 Best-practices summary **[A]**

| Do | Don't |
|---|---|
| Point `schema:` at the goose migrations dir (one truth). | Maintain a separate `schema.sql` that drifts. |
| `sqlc generate` after every migration; gate with `sqlc diff`. | Forget to regenerate and ship stale code. |
| List columns explicitly. | `SELECT *` in shipped queries. |
| Use `numeric`/`pgtype.Numeric` for money. | Ever use `float` for money. |
| Bake `WHERE owner_id = $auth` authz into queries. | Rely solely on a handler-layer check. |
| Use `WithTx` + defer-rollback for atomicity. | Leave transactions open across slow work. |
| Lock rows in a deterministic order. | Risk deadlocks with ad-hoc lock order. |
| Keep `overrides` identical across targets. | Let the same DB type map to different Go types. |
| Use `sqlc.narg`+`COALESCE` for optional filters. | Concatenate dynamic SQL strings by hand. |
| Pick one primary data layer; add a second only on real need. | Adopt sqlc + Ent reflexively. |

---

## 13. Study Path & Build-to-Learn Projects

### 13.1 A staged study path **[B→A]**

**Stage 1 — Beginner (get the loop in your fingers).**
- Install sqlc and goose; verify versions (§3.1).
- Create the canonical layout (§3.2) and the minimal `sqlc.yaml` (§3.3).
- Write one goose migration for a `notes(id, body, created_at)` table; `goose up`; check `goose status`.
- Point `schema:` at `db/migrations`; write `CreateNote :one`, `GetNote :one`, `ListNotes :many`; `sqlc generate`.
- Call the generated methods from a tiny `main.go` over a `pgxpool.Pool`. *Goal: internalize migration → generate → typed call.*

**Stage 2 — Intermediate (the real toolchain).**
- Add a migration that `ALTER`s the table (add a `tags text[]` column); regenerate; watch the struct change with zero config change (§2.3).
- Learn every annotation: convert some queries to `:exec`, `:execrows`, add a `:many` with keyset pagination (§5.6).
- Turn on `emit_json_tags`, `emit_pointers_for_null_types`, `emit_interface`; observe the output differences (§4).
- Add `overrides` for `uuid`/`timestamptz`/`numeric` (§4.6).
- Wire a `WithTx` transaction that does two writes atomically (§7).
- Add a `Makefile` and Air with `pre_cmd = ["sqlc generate"]` (§8.5–8.6).

**Stage 3 — Advanced (production & banking).**
- Add `sqlc vet` static CEL rules (ban `SELECT *`, ban float) and a DB-backed `sqlc/db-prepare` rule; run in CI (§8.3).
- Add the `sqlc diff` gate to CI (§8.4).
- Build the full accounts+ledger service (§11): numeric money, `FOR UPDATE`, idempotency keys, Argon2id+JWT, deterministic lock order.
- Add least-privilege DB roles (migration role vs app role) (§10.5).
- (Optional) Stand up the polyglot setup: add Ent for the user/RBAC graph over the *same* pool, Ent migrations OFF (§9).

### 13.2 Build-to-learn projects **[I/A]**

1. **URL shortener with analytics** *(warm-up).* goose migrations for `links` and `clicks`; sqlc queries for create/resolve; a keyset-paginated `clicks` list; `:copyfrom` to bulk-import a click log. Practice: annotations, pagination, bulk insert.

2. **The accounts + ledger service** *(the core project).* Build §11 from scratch: double-entry ledger, atomic idempotent transfers, exact decimal money, row locks, Argon2id+JWT auth, `sqlc vet` policy rules, `sqlc diff` CI gate. This single project exercises nearly everything in the guide. Add a reconciliation endpoint that asserts `SUM(entries.amount) == balance` per account.

3. **Multi-tenant SaaS billing** *(advanced).* Every table has a `tenant_id`; every query enforces `WHERE tenant_id = $auth` (object-level authz at scale, §10.2). Add optimistic-locking (`version` column) on the subscription row (§10.6). Add a nightly backfill migration (goose Go migration for data + `.sql` migration for schema) and confirm sqlc still parses the schema correctly (§12.4).

4. **Reporting read-model** *(advanced SQL).* Add heavy analytical queries — running balances with window functions, monthly rollups with `GROUP BY`, a recursive CTE for a transfer chain. Prove these are *painful in an ORM but natural in sqlc*. Add a DB-backed `sqlc vet` EXPLAIN rule that fails if any report query does a seq scan on `entries` (§8.3).

5. **The polyglot cutover** *(architecture).* Start with an Ent-only app (users, roles, accounts). Introduce sqlc for the hot balance-lookup and transfer paths over the *same pgxpool* (§9). Measure the latency difference on the hot path. Decide honestly, per §9.1, which paths deserve sqlc and which stay on Ent — and write down the seam.

### 13.3 Where to go next in this library **[B]**

- **[Goose migrations](GO_GOOSE_MIGRATIONS_GUIDE.md)** — the deep dive on goose: Go migrations, `StatementBegin/End`, out-of-order migrations, embedding migrations in the binary, no-transaction migrations for `CREATE INDEX CONCURRENTLY`.
- **[Go ent ORM](GO_ENT_ORM_GUIDE.md)** — the ORM you coexist with in §9; edges, eager loading, privacy policies.
- **[PostgreSQL](POSTGRESQL_GUIDE.md)** — the engine: indexes, `EXPLAIN`, isolation levels, roles/grants, `numeric` internals.
- **[Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md)** — the schema theory your migrations encode: keys, normalization, constraints.
- **[Go Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md)** — the HTTP layer wrapping your queries; middleware, binding, uploads.
- **[Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md)** — the auth used in §11: token issuance/verification, password hashing, session security.

### 13.4 The one-paragraph summary to remember **[B]**

**goose owns schema evolution; sqlc reads the very same goose migration files as its schema source, so your hand-written SQL is turned into type-safe Go that always matches the live database.** You write real SQL (keeping full expressive power and zero runtime reflection), goose evolves the schema (one immutable source of truth), and the two can never drift because they're generated from the identical files. Wrap it in pgx transactions for atomic money movement, enforce policy with `sqlc vet`, gate staleness with `sqlc diff`, and you have a correctness-first, banking-grade data layer. Pick this pairing whenever SQL correctness and performance matter — which, in a ledger, is always.
