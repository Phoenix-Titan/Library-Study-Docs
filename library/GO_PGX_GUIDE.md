# pgx (github.com/jackc/pgx/v5) — The PostgreSQL Driver & Toolkit for Go, Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Any Go developer who wants to talk to PostgreSQL *directly* — no ORM, no code generator, just you, your SQL, and the wire — and reach the point where they can run a production data layer safely: pool connections correctly, scan rows into structs without boilerplate, handle `NULL`s and PostgreSQL's rich type system, wrap work in transactions, batch and bulk-load for throughput, react to `LISTEN/NOTIFY`, and harden the whole thing to banking-grade. This guide is deliberately **explain-first**: every concept leads with prose — *what it is, why it exists, when to reach for it, how to use it, the best practice, and the gotcha* — and only then shows heavily-commented code. The database driver is the layer where a careless line becomes SQL injection, a leaked connection, or a lock held across a network hiccup; the "why" matters as much as the "how." Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **pgx v5.10+** (`github.com/jackc/pgx/v5`, current in 2026) on **Go 1.25 / 1.26**, talking to **PostgreSQL 17 / 18**. The application scaffolding is **Gin v1.10+** for HTTP, **`github.com/joho/godotenv`** for loading a `.env` file in development, and **Air** for hot-reload while you iterate. pgx v5 is a **major redesign** over v4: the type system moved from `pgtype` value objects you passed around to a codec-based `TypeMap`, the row-collection helpers (`pgx.CollectRows`, `pgx.RowToStructByName[T]`) are new and Go-generics-based, and the module path gained the `/v5` suffix. Everywhere v5 differs from v4 in a way that will trip up old tutorials, it is marked **⚡**. The author is on **Windows 11**, so shell commands are shown for PowerShell and POSIX where they differ. Always cross-check current signatures at **pkg.go.dev/github.com/jackc/pgx/v5** before shipping; the data layer is not a place to guess.
>
> **This guide's place in the library:** pgx is the **foundation** the rest of the Go data stack stands on. [Go ent ORM](GO_ENT_ORM_GUIDE.md) and [sqlc + goose](GO_SQLC_GOOSE_GUIDE.md) both *sit on top of pgx* — Ent runs its generated queries through a pgx pool, and sqlc's recommended driver is `pgx/v5`; this guide teaches the layer underneath them so you understand what they are actually doing and can drop to raw SQL when they can't express something. [Goose — DB migrations](GO_GOOSE_MIGRATIONS_GUIDE.md) owns the *schema* pgx queries against (and §17 shows the `stdlib.OpenDBFromPool` bridge goose needs). [PostgreSQL](POSTGRESQL_GUIDE.md) is the engine — its types, `EXPLAIN`, transactions, and `LISTEN/NOTIFY` are what pgx exposes to Go. [Go Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) is the HTTP framework our worked app is built in, and [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md) is the auth you would bolt onto it. If Ent/sqlc are the comfortable high-level tools, **pgx is the metal** — and knowing the metal is what lets you diagnose the pool exhaustion, the prepared-statement error behind PgBouncer, or the money column silently rounded to a float. This guide assumes you can read Go and know basic SQL (`SELECT`, `INSERT`, `JOIN`); it teaches pgx.

---

## Table of Contents

1. [What pgx Is and Where It Sits](#1-what-pgx-is-and-where-it-sits) **[B]**
2. [Setup: Install, godotenv, Air, First Connection](#2-setup-install-godotenv-air-first-connection) **[B]**
3. [Connecting: Conn vs Pool, Config and Tuning](#3-connecting-conn-vs-pool-config-and-tuning) **[B/I]**
4. [Executing Queries: Exec, Query, QueryRow, Scanning](#4-executing-queries-exec-query-queryrow-scanning) **[B]**
5. [Parameters and SQL Injection](#5-parameters-and-sql-injection) **[B]**
6. [Collecting Rows into Structs — the pgx v5 Superpower](#6-collecting-rows-into-structs--the-pgx-v5-superpower) **[B/I]**
7. [NULLs and the pgtype System](#7-nulls-and-the-pgtype-system) **[I]**
8. [PostgreSQL Data Types in Go](#8-postgresql-data-types-in-go) **[I]**
9. [Errors: PgError, SQLSTATE, ErrNoRows](#9-errors-pgerror-sqlstate-errnorows) **[B/I]**
10. [Transactions](#10-transactions) **[I]**
11. [Batching — Fewer Round-Trips](#11-batching--fewer-round-trips) **[I/A]**
12. [CopyFrom — Bulk Loading](#12-copyfrom--bulk-loading) **[I/A]**
13. [Prepared Statements, the Statement Cache and QueryExecMode](#13-prepared-statements-the-statement-cache-and-queryexecmode) **[A]**
14. [LISTEN / NOTIFY — Postgres Pub/Sub](#14-listen--notify--postgres-pubsub) **[A]**
15. [Context, Timeouts and Cancellation](#15-context-timeouts-and-cancellation) **[I]**
16. [Tracing, Logging and Observability](#16-tracing-logging-and-observability) **[A]**
17. [The database/sql Bridge: Interop with goose, sqlc, Libraries](#17-the-databasesql-bridge-interop-with-goose-sqlc-libraries) **[I]**
18. [Architecture: The Repository Layer in a Gin App](#18-architecture-the-repository-layer-in-a-gin-app) **[I/A]**
19. [Config, .env and Files: godotenv, embed.FS, os/exec](#19-config-env-and-files-godotenv-embedfs-osexec) **[I]**
20. [The Full Worked App: Gin + pgx + godotenv + Air](#20-the-full-worked-app-gin--pgx--godotenv--air) **[I/A]**
21. [Testing: pgxmock, Testcontainers, tx-rollback](#21-testing-pgxmock-testcontainers-tx-rollback) **[A]**
22. [Performance and Scaling: Pool Math, PgBouncer, Replicas](#22-performance-and-scaling-pool-math-pgbouncer-replicas) **[A]**
23. [Banking-Grade Security Checklist](#23-banking-grade-security-checklist) **[A]**
24. [Gotchas and Best Practices](#24-gotchas-and-best-practices) **[A]**
25. [Study Path and Build-to-Learn Projects](#25-study-path-and-build-to-learn-projects)

---

## 1. What pgx Is and Where It Sits

### 1.1 The one-paragraph mental model **[B]**

**pgx is a pure-Go driver *and* toolkit for PostgreSQL.** "Driver" means it knows how to open a TCP (or Unix-socket) connection to a PostgreSQL server, speak the PostgreSQL wire protocol over it, send your SQL, and decode the results back into Go values. "Toolkit" means it is not *only* a driver hidden behind Go's generic `database/sql` interface — it also exposes its own richer, PostgreSQL-specific API (`pgx.Conn`, `pgxpool.Pool`, `pgx.Batch`, `CopyFrom`, `LISTEN/NOTIFY`) that lets you use capabilities the lowest-common-denominator `database/sql` cannot express. You get to choose which face you use, and you can use both at once against the same database.

If you have written Go database code before, you have probably used `database/sql` with a driver like `lib/pq`. pgx is the modern replacement for that whole arrangement: it is faster, it is actively maintained, it supports PostgreSQL's full type system properly, and it gives you the native API when you want it. When people say "use pgx" in 2026, this is what they mean.

### 1.2 The two faces of pgx **[B]**

This is the single most important thing to understand up front, because almost every "which import do I use?" confusion comes from not knowing it. pgx offers **two distinct programming interfaces** to the same underlying engine:

| Face | Import | What you get | When to use |
|---|---|---|---|
| **Native pgx** | `github.com/jackc/pgx/v5` + `.../pgxpool` | `pgx.Conn`, `pgxpool.Pool`, `pgx.Rows`, `Batch`, `CopyFrom`, `LISTEN/NOTIFY`, the generic row-collectors | **Default.** Your application's own code. Full PostgreSQL power, best performance, best ergonomics. |
| **database/sql adapter** | `github.com/jackc/pgx/v5/stdlib` | A standard `*sql.DB` that happens to be driven by pgx | Only when a **third-party library demands `*sql.DB`** — e.g. goose migrations, sqlc's `database/sql` mode, some ORMs, test helpers. |

The rule of thumb: **write your own code against native pgx (`pgxpool.Pool`), and reach for the `stdlib` adapter only at the boundary where some other library insists on `*sql.DB`.** §17 covers the bridge in full. For now, when this guide says "pgx," assume the native API unless it says `stdlib`.

> **⚡ v4 → v5 note:** In pgx v4 the native and database/sql paths were more tangled, and the type system (`pgtype`) leaked into everyday code. v5 cleanly separates them and makes the native path the obvious default. If you find a Stack Overflow answer importing `github.com/jackc/pgx/v4` or passing `*pgtype.Text` values around by hand, it is pre-2022 and the ergonomics have since changed — verify against v5.

### 1.3 pgx vs database/sql + lib/pq vs Ent vs sqlc **[B/I]**

Newcomers to Go-and-Postgres face a confusing menu. Here is the whole landscape on one screen, because knowing what pgx is *not* is how you know when to use it.

| Option | What it is | Trade-off |
|---|---|---|
| **`database/sql` + `lib/pq`** | The classic stdlib interface with the original pure-Go Postgres driver. | `lib/pq` is in **maintenance mode** (the maintainers themselves recommend pgx). `database/sql`'s interface can't express batching, `CopyFrom`, `LISTEN/NOTIFY`, or the full type system. Avoid for new code. |
| **`database/sql` + `pgx/v5/stdlib`** | The stdlib interface, but driven by pgx. | Better driver, same limited interface. Use only when a library needs `*sql.DB`. |
| **native `pgx/v5`** *(this guide)* | pgx's own API. | Full power, best speed, PostgreSQL-only (not portable to MySQL). You write SQL by hand. |
| **`sqlc` (on pgx)** | You write SQL; a generator produces type-safe Go that calls pgx. | Compile-time-checked queries, zero reflection — but a codegen step and less flexibility for dynamic SQL. See [sqlc + goose](GO_SQLC_GOOSE_GUIDE.md). |
| **Ent ORM (on pgx)** | Schema-as-Go-code; a generated, type-safe query builder over pgx. | Great for graph/CRUD and relations; a heavier abstraction. Drops to raw SQL only via an escape hatch. See [ent](GO_ENT_ORM_GUIDE.md). |

The unifying insight: **sqlc and Ent both run on pgx.** They are conveniences layered *on top of* the driver this guide teaches. Learning pgx directly means (a) you can use it standalone when an ORM is overkill, (b) you understand what your ORM is doing on the wire, and (c) you always have the escape hatch — when Ent or sqlc can't express a query, you drop to the pgx pool they're already using and run the SQL yourself.

**When to choose raw pgx for a project:** you want maximum control and performance, your queries are hand-tuned SQL, you don't need cross-database portability, and you're comfortable owning the mapping between rows and structs (which §6 makes nearly free). This describes a huge amount of production Go — high-throughput services, data pipelines, anything where you'd rather read SQL than an ORM's query DSL.

### 1.4 A note on the module layout **[B]**

The `pgx/v5` module ships several packages you'll import. Knowing what each is prevents "which import?" paralysis:

| Package | Purpose |
|---|---|
| `github.com/jackc/pgx/v5` | The core native API: `Conn`, `Rows`, `Batch`, `CollectRows`, query-exec modes, `NamedArgs`. |
| `github.com/jackc/pgx/v5/pgxpool` | The **connection pool** — what your server code holds. `Pool`, `Config`, `ParseConfig`. |
| `github.com/jackc/pgx/v5/pgtype` | The **type system**: `pgtype.Text`, `pgtype.Numeric`, `pgtype.UUID`, `Timestamptz`, the `TypeMap`. Used for `NULL`s and exotic types. |
| `github.com/jackc/pgx/v5/pgconn` | The **low-level connection & errors**: `pgconn.PgError` (SQLSTATE), `CommandTag`, the raw protocol conn. |
| `github.com/jackc/pgx/v5/stdlib` | The **`database/sql` adapter** (§17). |
| `github.com/jackc/pgx/v5/tracelog` | Query **logging/tracing** adapter (§16). |

You will import `pgxpool` in almost every file that touches the DB, `pgconn` whenever you inspect errors, and `pgtype` whenever you deal with nullable or special columns. The rest are situational.

---

## 2. Setup: Install, godotenv, Air, First Connection

### 2.1 Prerequisites and the project skeleton **[B]**

You need Go 1.25+, a running PostgreSQL 17/18 (local install, Docker, or a managed instance), and `psql` handy for sanity checks. Create the project:

```bash
mkdir pgx-bank && cd pgx-bank
go mod init github.com/you/pgx-bank

# The core driver + pool.
go get github.com/jackc/pgx/v5

# godotenv: loads a .env file into the process environment in development.
go get github.com/joho/godotenv

# Gin: the HTTP framework for the worked app (later sections).
go get github.com/gin-gonic/gin
```

`go get github.com/jackc/pgx/v5` pulls the whole module — `pgxpool`, `pgtype`, `pgconn`, `stdlib` all come with it; you don't `go get` them separately, you just `import` the sub-packages and they resolve.

### 2.2 Configuration via .env and godotenv — and why **[B]**

Your database connection string contains a password. It must **never** be hard-coded in source (it would leak into Git history) and it changes per environment (your laptop, CI, staging, prod all point at different databases). The industry-standard answer is the **[twelve-factor](https://12factor.net/config) rule: configuration comes from the environment**, not from the code.

In production, real environment variables are set by your orchestrator (systemd unit, Docker `--env`, Kubernetes `Secret`, the platform's secret manager). But on your laptop, exporting a dozen variables into every shell is tedious and easy to forget. **godotenv** solves the *development* half: it reads a file named `.env` in your project root and loads those `KEY=value` pairs into the process's environment at startup — so `os.Getenv("DATABASE_URL")` works locally exactly as it will in prod, with no code difference.

Create `.env` in the project root:

```bash
# .env — DEVELOPMENT ONLY. This file is git-ignored and never deployed.
DATABASE_URL=postgres://bank:secret@localhost:5432/bank?sslmode=disable
PORT=8080
LOG_LEVEL=debug
```

And — critically — add it to `.gitignore` so a password never reaches your repository:

```bash
# .gitignore
.env
.env.local
/tmp/
bank            # the compiled binary
```

> **⚡ Security rule, stated once and meant permanently:** `.env` is a **development convenience**, not a production secret store. It must be git-ignored, and in production you do **not** ship a `.env` file — you inject real environment variables from a secret manager (Vault, AWS/GCP Secrets Manager, Kubernetes Secrets). godotenv's own docs say the same. The pattern in §2.3 — *try to load `.env`, ignore the error if it's absent* — is exactly what makes the same binary work both ways.

### 2.3 Your first connection **[B]**

Here is the smallest program that connects, verifies the connection, and runs one query. Read the comments — every line is doing something you'll repeat forever.

```go
// cmd/hello/main.go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/joho/godotenv"
)

func main() {
	// 1) Load .env into the environment IF it exists. In production there is no
	//    .env file and Load returns an error — we deliberately ignore it, because
	//    the real env vars are already set by the platform. This one line is why
	//    the same binary works on your laptop and in prod unchanged.
	_ = godotenv.Load()

	// 2) Read config from the environment (never hard-code the DSN).
	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		log.Fatal("DATABASE_URL is not set")
	}

	// 3) A context bounds how long "connecting" is allowed to take. Without a
	//    deadline, a wrong host would hang your startup forever.
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	// 4) Open a POOL (not a single connection). pgxpool.New parses the DSN,
	//    creates the pool, and opens one connection eagerly to validate the DSN.
	//    A *pgxpool.Pool is safe for concurrent use by many goroutines — it is the
	//    thing you hold for the lifetime of the program.
	pool, err := pgxpool.New(ctx, dsn)
	if err != nil {
		log.Fatalf("unable to create connection pool: %v", err)
	}
	defer pool.Close() // returns all connections to the OS on shutdown.

	// 5) Ping proves we can actually reach the server and authenticate. New()
	//    validating the DSN is not the same as a live round-trip; ping confirms it.
	if err := pool.Ping(ctx); err != nil {
		log.Fatalf("cannot reach database: %v", err)
	}

	// 6) Run one query. QueryRow expects exactly one row; Scan copies the columns
	//    into Go variables you pass by pointer.
	var version string
	if err := pool.QueryRow(ctx, "SELECT version()").Scan(&version); err != nil {
		log.Fatalf("query failed: %v", err)
	}

	fmt.Println("Connected to:", version)
}
```

Run it:

```bash
go run ./cmd/hello
# Connected to: PostgreSQL 18.1 on x86_64-pc-linux-gnu, compiled by gcc ...
```

If you see the version string, your whole toolchain — Go, pgx, godotenv, the DSN, PostgreSQL — is wired correctly. Everything else in this guide builds on this six-step spine: **load env → read DSN → bounded context → open pool → ping → query.**

### 2.4 Air for hot-reload in development **[B]**

While developing, you don't want to stop the server, `go build`, and restart on every edit. **Air** watches your source tree, rebuilds, and restarts the binary automatically — the Go equivalent of `nodemon`. It only matters in development; production runs the compiled binary directly.

```bash
# Install once (Go 1.24+ tool style keeps it pinned in go.mod):
go get -tool github.com/air-verse/air
# ...then run it with:  go tool air
# Or the classic global install:
go install github.com/air-verse/air@latest
```

Create `.air.toml` in the project root:

```toml
# .air.toml — dev hot-reload config.
root = "."
tmp_dir = "tmp"

[build]
  # Build the server command into a temp binary, then run it.
  cmd = "go build -o ./tmp/server ./cmd/server"
  bin = "./tmp/server"
  # Watch Go files; ignore the tmp dir and test files to avoid rebuild loops.
  include_ext = ["go", "tmpl", "html", "sql"]
  exclude_dir = ["tmp", "vendor", "node_modules"]
  exclude_regex = ["_test\\.go"]
  # Wait a beat after a change so a burst of saves triggers one rebuild.
  delay = 200 # ms

[misc]
  clean_on_exit = true
```

Now `go tool air` (or `air`) runs your server and restarts it on every save. Note `include_ext` lists `sql` too — so editing an embedded `.sql` file also triggers a rebuild. Air reads the *same* `.env` your app does (it just runs your binary), so godotenv config keeps working under Air with zero extra setup.

> **Gotcha:** Air rebuilds on *any* watched file change, including files your editor writes on save (swap files, format-on-save). If you get a rebuild loop, widen `exclude_regex` or `exclude_dir`. And never run Air in production — it's a dev tool that shells out to the compiler on every change.

---

## 3. Connecting: Conn vs Pool, Config and Tuning

### 3.1 A single connection vs a pool — and why you almost always want the pool **[B]**

pgx gives you two ways to hold a database connection, and choosing wrong is the most common beginner mistake.

- **`pgx.Connect(ctx, dsn)` → `*pgx.Conn`** is a **single physical connection**. It is **not safe for concurrent use** — if two goroutines call methods on the same `*pgx.Conn` at once, you get corruption or a panic. One `*pgx.Conn` = one conversation with the server at a time.
- **`pgxpool.New(ctx, dsn)` → `*pgxpool.Pool`** is a **managed set of connections**. It **is** safe for concurrent use by any number of goroutines. When a goroutine runs a query, the pool hands it an idle connection, the query runs, and the connection returns to the pool automatically.

A web server handles many requests concurrently — each in its own goroutine. If they all shared one `*pgx.Conn`, they'd serialize (or corrupt each other). So **server code always uses a `*pgxpool.Pool`.** You reach for a bare `*pgx.Conn` only in narrow cases: a one-shot CLI/migration script, or a connection you must "own" exclusively for its whole life — most notably a `LISTEN/NOTIFY` listener (§14), where the connection is dedicated to waiting for notifications and must not be shared.

```go
// Single connection — for a short-lived script or a dedicated LISTEN conn.
conn, err := pgx.Connect(ctx, dsn)
if err != nil { /* ... */ }
defer conn.Close(ctx) // note: Conn.Close takes a context; Pool.Close does not.

// A pool — for anything concurrent (a server). Hold ONE for the whole program.
pool, err := pgxpool.New(ctx, dsn)
if err != nil { /* ... */ }
defer pool.Close()
```

> **The cardinal rule:** create **one** pool at startup, store it, share it everywhere. Do **not** open a pool per request — a pool *is* the thing that amortizes connection cost across requests; making one per request defeats its entire purpose and will exhaust the server's connection slots in seconds.

### 3.2 The connection string (DSN) **[B]**

pgx accepts two DSN formats. Both encode the same information; use whichever you find readable.

```text
# 1) URL form (most common):
postgres://USER:PASSWORD@HOST:PORT/DBNAME?sslmode=require&pool_max_conns=10

# 2) Keyword/value form (DSN form):
user=bank password=secret host=localhost port=5432 dbname=bank sslmode=require
```

pgx *also* honors the standard libpq **`PG*` environment variables** (`PGHOST`, `PGPORT`, `PGUSER`, `PGPASSWORD`, `PGDATABASE`, `PGSSLMODE`, …) as fallbacks/overrides — handy in containers. Key parameters you'll set:

| Parameter | Meaning |
|---|---|
| `sslmode` | `disable` (local dev only), `require` (encrypt, don't verify cert), `verify-full` (encrypt **and** verify hostname — use in prod). |
| `pool_max_conns` | Max connections the pool will open (pgxpool reads this straight from the DSN). |
| `pool_min_conns` | Connections kept warm even when idle. |
| `connect_timeout` | Seconds to wait for the TCP+auth handshake. |
| `statement_cache_capacity` | Size of the per-connection prepared-statement cache (§13). |
| `application_name` | Shows up in `pg_stat_activity` — set it so DBAs can see which service a connection belongs to. |

> **⚡ Security:** use `sslmode=verify-full` in production against any database you don't fully control the network path to. `require` encrypts but does **not** stop a man-in-the-middle presenting a forged certificate; `verify-full` checks the server cert against a CA *and* matches the hostname. `disable` sends your password in cleartext — local-only.

### 3.3 Tuning the pool with ParseConfig **[I]**

Passing a DSN string to `pgxpool.New` is the quick path. For real control — pool sizing, lifetimes, per-connection setup — parse the DSN into a `*pgxpool.Config`, adjust its fields, and call `NewWithConfig`. This is the form you'll use in production.

```go
func newPool(ctx context.Context, dsn string) (*pgxpool.Pool, error) {
	cfg, err := pgxpool.ParseConfig(dsn)
	if err != nil {
		return nil, fmt.Errorf("parse pool config: %w", err)
	}

	// ---- Pool sizing ----
	// MaxConns is the single most important knob. Too high and you overwhelm
	// Postgres (each connection is a backend process costing memory); too low and
	// requests queue waiting for a free connection. §22 covers the sizing math.
	cfg.MaxConns = 10
	// MinConns keeps a few connections warm so the first requests after idle don't
	// pay the connect+TLS handshake cost. Keep modest.
	cfg.MinConns = 2

	// ---- Lifetimes (prevent stale connections) ----
	// Recycle a connection this long after it was CREATED, even if healthy. This
	// bounds memory growth on the server side and lets load-balancers rebalance.
	cfg.MaxConnLifetime = time.Hour
	// A little jitter so all connections don't expire at the same instant (which
	// would cause a thundering-herd reconnect). pgx adds jitter automatically, but
	// you can tune the window.
	cfg.MaxConnLifetimeJitter = 5 * time.Minute
	// Close a connection that has sat idle this long — frees server resources
	// during quiet periods.
	cfg.MaxConnIdleTime = 30 * time.Minute
	// How often the pool sweeps for connections to health-check / retire.
	cfg.HealthCheckPeriod = time.Minute

	// ---- Per-connection setup hooks (see 3.4) ----
	cfg.ConnConfig.ConnectTimeout = 5 * time.Second

	return pgxpool.NewWithConfig(ctx, cfg)
}
```

Every field has a sensible default, so you only set the ones you care about. In practice `MaxConns` and the lifetimes are what you tune; the rest you leave alone until a metric tells you otherwise.

### 3.4 Connection lifecycle hooks **[I/A]**

`pgxpool.Config` exposes callbacks that fire at points in a connection's life. They're how you run per-connection setup — register custom types, set session parameters, or gate acquisition.

| Hook | Fires | Typical use |
|---|---|---|
| `BeforeConnect(ctx, *pgx.ConnConfig) error` | Before each new physical connect | Rotate credentials (e.g. fetch a short-lived IAM token per connection). |
| `AfterConnect(ctx, *pgx.Conn) error` | Right after a new connection opens | **Register custom/enum types**, run `SET` statements (e.g. `SET timezone`), prepare named statements. |
| `BeforeAcquire(ctx, *pgx.Conn) bool` | Before a conn is handed out of the pool | Return `false` to discard a connection you deem unhealthy for this checkout. |
| `AfterRelease(*pgx.Conn) bool` | When a conn returns to the pool | Return `false` to destroy (not reuse) a connection — e.g. after you dirtied its session state. |

```go
// Example: after each new connection opens, pin a safe session config.
cfg.AfterConnect = func(ctx context.Context, conn *pgx.Conn) error {
	// A hard server-side ceiling on any single statement. If a query runs longer
	// than this, Postgres cancels it — a crucial DoS/runaway-query guard (§23).
	_, err := conn.Exec(ctx, "SET statement_timeout = '30s'")
	if err != nil {
		return err
	}
	// Kill transactions left idle (a client that BEGINs then hangs holds locks).
	_, err = conn.Exec(ctx, "SET idle_in_transaction_session_timeout = '15s'")
	return err
}
```

`AfterConnect` is also where you register PostgreSQL **enum** and **composite** types so pgx can decode them (§8.6), and where integrations like `github.com/vgarvardt/pgx-google-uuid/v5` register `google/uuid` support on the connection's `TypeMap`.

### 3.5 Acquiring a raw connection from the pool **[I]**

Usually you call query methods directly on the `*pgxpool.Pool` and never think about individual connections. But sometimes you need to run several statements **on the same physical connection** — because they share session state that doesn't survive going back to the pool. Examples: a temporary table, a session-level `SET`, an advisory lock, or a `LISTEN`. For those, `Acquire` checks a connection out of the pool and gives it to you exclusively until you `Release` it.

```go
conn, err := pool.Acquire(ctx)
if err != nil {
	return err
}
defer conn.Release() // ALWAYS release — a leaked acquire permanently shrinks the pool.

// These two run on the SAME connection, so the temp table created by the first
// is visible to the second. Had you run them on the pool directly, they might
// land on different connections and the second would fail.
_, err = conn.Exec(ctx, "CREATE TEMP TABLE scratch (id int)")
// ... use scratch ...
```

> **The `Release` discipline:** every `Acquire` must be paired with exactly one `Release`, and `defer` is how you guarantee it even on an error path. A forgotten `Release` is a **connection leak** — that connection never returns to the pool, so `MaxConns` effectively drops by one. Leak `MaxConns` of them and the pool deadlocks: every request waits forever for a connection that will never come back. This is the single most common way a pgx service falls over in production.

---

## 4. Executing Queries: Exec, Query, QueryRow, Scanning

There are three verbs, and picking the right one is about **how many rows you expect back**.

### 4.1 Exec — statements that return no rows **[B]**

`Exec` runs a statement and returns a **`CommandTag`**, not rows. Use it for `INSERT`/`UPDATE`/`DELETE`/DDL — anything where you care *how many rows changed*, not *what they contain*.

```go
tag, err := pool.Exec(ctx,
	"UPDATE accounts SET balance = balance - $1 WHERE id = $2",
	amount, accountID)
if err != nil {
	return err
}

// CommandTag tells you what happened.
n := tag.RowsAffected() // int64: how many rows the statement touched.
if n == 0 {
	// Zero rows updated → the WHERE matched nothing. Often this means "not found"
	// and you should surface a 404, NOT treat it as success.
	return ErrAccountNotFound
}
// tag also exposes tag.Insert(), tag.Update(), tag.Delete(), tag.String().
```

**Checking `RowsAffected()` is not optional.** An `UPDATE ... WHERE id = $1` that matches no row returns `nil` error and `0` rows affected — the statement *succeeded*, it just didn't do anything. If you don't check, a "transfer money from account 999" (which doesn't exist) looks like success. In a banking context that's a bug that moves phantom money.

### 4.2 QueryRow — exactly one row **[B]**

`QueryRow` is for queries that return **a single row** (a lookup by primary key, an aggregate like `COUNT(*)`, an `INSERT ... RETURNING id`). It never returns an error itself — any error is deferred to `Scan`, so the whole thing is a one-liner.

```go
var (
	id      int64
	email   string
	balance int64
)
err := pool.QueryRow(ctx,
	"SELECT id, email, balance FROM accounts WHERE id = $1", accountID,
).Scan(&id, &email, &balance)

switch {
case errors.Is(err, pgx.ErrNoRows):
	// The query ran fine but returned zero rows → "not found".
	return ErrAccountNotFound
case err != nil:
	// A real error: connection dropped, type mismatch, SQL error.
	return err
}
// id, email, balance are now populated.
```

The three-way branch — **`pgx.ErrNoRows`** (not found, usually a 404), **other error** (a real failure, usually a 500), **success** — is the canonical shape of every single-row read. Memorize it. `Scan` maps result columns **positionally** to the pointers you pass, left to right, so the order of columns in `SELECT` must match the order of `&args` in `Scan`.

### 4.3 Query + Rows — many rows **[B]**

`Query` returns a **`pgx.Rows`** cursor you iterate. This is the low-level, manual form; §6 shows the far nicer generic collectors that replace this boilerplate. But you must understand the manual form first, because the collectors are built on it and the iteration rules are the same.

```go
rows, err := pool.Query(ctx,
	"SELECT id, email, balance FROM accounts WHERE balance > $1 ORDER BY id", minBalance)
if err != nil {
	return nil, err
}
defer rows.Close() // ALWAYS. Releases the underlying connection back to the pool.

var accounts []Account
for rows.Next() { // advances to the next row; returns false at the end OR on error.
	var a Account
	if err := rows.Scan(&a.ID, &a.Email, &a.Balance); err != nil {
		return nil, err // a scan error (type mismatch, etc.)
	}
	accounts = append(accounts, a)
}
// CRUCIAL: rows.Next() returns false BOTH at normal end-of-rows AND on error.
// You MUST check rows.Err() after the loop to tell which happened. Skipping this
// hides mid-iteration failures (e.g. the connection dropping halfway) as if the
// result set simply ended — a silent data-truncation bug.
if err := rows.Err(); err != nil {
	return nil, err
}
return accounts, nil
```

Three rules that the manual loop lives or dies by:

1. **`defer rows.Close()`** — until you close the rows (or fully drain them), the underlying connection is *pinned to this cursor* and cannot serve anyone else. Forgetting it leaks a connection exactly like a forgotten `Release`.
2. **Check `rows.Err()` after the loop** — because `Next()` returning `false` is ambiguous between "done" and "broke."
3. **You cannot run another query on the same connection while `rows` is open** — the connection is busy streaming this result. (With a pool you'd just grab another connection; on a single `Conn` it's an error.)

This boilerplate — declare slice, loop, scan each field by pointer, check `Err` — is identical for every multi-row query, which is exactly why §6 exists to erase it.

### 4.4 The CommandTag in depth **[I]**

`Exec` and the batch/copy APIs return `pgconn.CommandTag`. It's a thin wrapper over the string Postgres sends back (`"UPDATE 3"`, `"INSERT 0 1"`, `"DELETE 0"`):

```go
tag, _ := pool.Exec(ctx, "DELETE FROM sessions WHERE expires_at < now()")
tag.RowsAffected() // 3
tag.Delete()       // true — this was a DELETE
tag.Insert()       // false
tag.String()       // "DELETE 3"
```

`RowsAffected()` is the field you use 99% of the time — for optimistic-concurrency checks ("did my conditional UPDATE actually hit a row?"), for "not found" detection, and for reporting how much a bulk operation changed.

---

## 5. Parameters and SQL Injection

### 5.1 The `$1` placeholder — never, ever concatenate **[B]**

This is the most important security rule in the entire guide, so it gets its own section. **You never build SQL by gluing user input into the query string.** You write **placeholders** — `$1`, `$2`, … — and pass the values as separate arguments. pgx sends the query text and the parameter values to PostgreSQL *separately*, over the extended query protocol. The server compiles the query with the placeholders as typed holes, then binds your values into those holes. The values are **data**, never parsed as SQL. It is structurally impossible for a value to "escape" and become a command.

```go
// ✅ CORRECT — parameterized. email is data; it can contain anything, including
//    "'; DROP TABLE accounts; --", and it will simply be searched for as a literal.
row := pool.QueryRow(ctx, "SELECT id FROM accounts WHERE email = $1", email)

// ❌ CATASTROPHIC — string concatenation. If email is  x' OR '1'='1  the WHERE
//    becomes always-true; if it's  '; DROP TABLE accounts; --  you lose the table.
//    NEVER do this. Not with fmt.Sprintf, not with +, not "just this once".
row := pool.QueryRow(ctx, "SELECT id FROM accounts WHERE email = '"+email+"'")
```

pgx uses PostgreSQL's numbered placeholders (`$1`, `$2`), **not** the `?` you may know from MySQL/`database/sql`. Each number can be reused: `WHERE created_at > $1 AND updated_at > $1` binds `$1` once and uses it twice. There is **no** correct way to interpolate user input into SQL text — if you ever feel you need to, you actually need a parameter (for values) or a strict allow-list (for identifiers; see §5.4).

### 5.2 What can and cannot be a parameter **[B/I]**

Parameters stand in for **values** — the things in `WHERE`, `VALUES`, `SET`, `LIMIT`. They **cannot** stand in for **identifiers** (table names, column names) or **SQL keywords** (`ASC`/`DESC`, `AND`/`OR`). This is a frequent beginner trap:

```go
// ✅ Values are parameterizable.
pool.Query(ctx, "SELECT * FROM accounts WHERE status = $1 LIMIT $2", status, limit)

// ❌ You CANNOT parameterize a column name or sort direction like this:
pool.Query(ctx, "SELECT * FROM accounts ORDER BY $1 $2", column, direction) // WRONG
// Postgres treats $1 as a *value* to sort by (a constant), not a column, so every
// row sorts identically — silently wrong, no error.
```

Handling a dynamic `ORDER BY` safely is §5.4. The rule: if the thing you want to vary is *part of the SQL grammar* (a name, a keyword), a parameter can't do it and you need an allow-list instead.

### 5.3 Named arguments with pgx.NamedArgs **[I]**

Positional `$1, $2, $3, …` gets error-prone once a query has many parameters — miscount by one and everything shifts. pgx v5 offers **`pgx.NamedArgs`**, a `map[string]any` you reference by `@name` in the SQL. pgx rewrites `@name` into the correct positional placeholder before sending — so it's just as injection-safe (still real parameters), only more readable.

```go
tag, err := pool.Exec(ctx,
	`INSERT INTO accounts (owner, email, balance, currency)
	 VALUES (@owner, @email, @balance, @currency)`,
	pgx.NamedArgs{
		"owner":    "Ada Lovelace",
		"email":    "ada@example.com",
		"balance":  0,
		"currency": "USD",
	})
```

Named args shine on long `INSERT`/`UPDATE` statements and when the same value appears several times. They are purely a readability convenience over positional parameters — same protocol, same safety.

### 5.4 Dynamic identifiers: the allow-list pattern **[I/A]**

Sometimes you genuinely need a dynamic column or sort direction — a table UI where the user picks the sort. Since you can't parameterize an identifier, you **map user input through a fixed allow-list of known-safe SQL fragments** and reject anything not in it. The user never supplies SQL; they supply a *key* that *you* translate to SQL you wrote.

```go
// Allow-list: the ONLY sort columns permitted, mapped to trusted SQL text.
var sortColumns = map[string]string{
	"created": "created_at",
	"email":   "email",
	"balance": "balance",
}

func listAccounts(ctx context.Context, pool *pgxpool.Pool, sortKey, dir string) (pgx.Rows, error) {
	col, ok := sortColumns[sortKey]
	if !ok {
		col = "created_at" // safe default; never trust the raw key.
	}
	// Direction is a closed set — normalize to exactly one of two literals.
	order := "ASC"
	if strings.EqualFold(dir, "desc") {
		order = "DESC"
	}
	// col and order are now values WE chose from a fixed set — safe to interpolate.
	// The actual user-supplied *filter values* still go through $1 parameters.
	sql := fmt.Sprintf("SELECT id, email, balance FROM accounts ORDER BY %s %s LIMIT $1", col, order)
	return pool.Query(ctx, sql, 50)
}
```

The safety comes from one property: **every possible interpolated string is one you authored.** The user's input only ever selects *which* of your pre-written fragments to use. If you ever find yourself interpolating a string the user could influence the *content* of, stop — that's the injection you were avoiding.

### 5.5 The IN-list problem **[I]**

A classic snag: `WHERE id IN (?, ?, ?)` with a variable number of ids. You can't parameterize a list as a single `$1`. pgx's idiomatic answer is to pass a **Go slice bound to one parameter as a PostgreSQL array**, using `= ANY($1)`:

```go
ids := []int64{1, 2, 3, 42}
rows, err := pool.Query(ctx,
	"SELECT id, email FROM accounts WHERE id = ANY($1)", ids)
// pgx encodes the []int64 as a Postgres int8[] and ANY matches membership. One
// parameter, any length, fully safe. This is far better than building "IN (...)"
// with N placeholders — no string juggling, no injection surface.
```

`= ANY($1)` with a slice is the pgx way to do "IN a dynamic list." It also sidesteps a subtle bug: an empty `IN ()` is a SQL syntax error, whereas `= ANY('{}')` (empty array) is perfectly valid and correctly matches nothing.

---

## 6. Collecting Rows into Structs — the pgx v5 Superpower

### 6.1 The problem the collectors solve **[B]**

The manual `for rows.Next() { rows.Scan(&a.ID, &a.Email, ...) }` loop from §4.3 is correct but tedious and fragile: every query repeats the same five lines, and every `Scan` re-lists the fields *by position*, so adding a column to the `SELECT` and forgetting to add its pointer to `Scan` is a silent misalignment. pgx v5 introduces **generic row-collector functions** that erase all of it. You describe the destination type once, and pgx does the loop, the scanning, the field mapping, the `Close`, and the `Err` check for you.

This is the biggest ergonomic leap from v4 and the reason raw pgx feels competitive with an ORM for reads. There are two pieces: a **collector** (how many rows) and a **row-mapper** (how one row becomes one value).

### 6.2 CollectRows + RowToStructByName **[B]**

The workhorse. `pgx.CollectRows(rows, mapper)` runs the whole iteration and returns a `[]T`. The mapper `pgx.RowToStructByName[T]` maps each row's **columns to struct fields by name** (case-insensitive; override with a `db:"..."` tag).

```go
// The struct's fields map to columns by name. Exported fields only. The db tag
// overrides the column name when they differ (Go convention vs SQL convention).
type Account struct {
	ID        int64     `db:"id"`
	Owner     string    `db:"owner"`
	Email     string    `db:"email"`
	Balance   int64     `db:"balance"`
	CreatedAt time.Time `db:"created_at"`
}

func listRichAccounts(ctx context.Context, pool *pgxpool.Pool, min int64) ([]Account, error) {
	rows, err := pool.Query(ctx,
		"SELECT id, owner, email, balance, created_at FROM accounts WHERE balance >= $1", min)
	if err != nil {
		return nil, err
	}
	// CollectRows consumes `rows` fully: it loops, maps each row to an Account by
	// matching column names to `db` tags, checks rows.Err(), and closes the rows.
	// The five-line manual loop becomes one line — and it's name-based, so column
	// order in the SELECT no longer has to match field order.
	return pgx.CollectRows(rows, pgx.RowToStructByName[Account])
}
```

Compare to §4.3: no `defer rows.Close()`, no `for`, no per-field `Scan`, no `rows.Err()` — `CollectRows` does all four correctly. And because mapping is **by name**, `SELECT created_at, id, ...` in a different order still works, and an extra column in the struct that isn't selected is the only thing that errors (which `...Lax` relaxes — §6.4).

### 6.3 The mapper family **[I]**

You pass a **row-mapper** as the second argument to `CollectRows`/`CollectOneRow`. Pick by what shape you want each row as:

| Mapper | Each row becomes | Use when |
|---|---|---|
| `pgx.RowToStructByName[T]` | a `T` struct, columns matched to fields **by name** | The default for `SELECT`ing named columns into a struct. |
| `pgx.RowToStructByNameLax[T]` | a `T`, by name, but **extra struct fields are allowed** to be absent from the result | The struct has fields not in this particular `SELECT`. |
| `pgx.RowToAddrOfStructByName[T]` | a `*T` (pointer) | You want `[]*T` (e.g. to avoid copying big structs, or for nil-ability). |
| `pgx.RowToStructByPos[T]` | a `T`, columns matched **by position** | Column order is fixed and you'd rather not tag fields. Fragile — avoid unless deliberate. |
| `pgx.RowTo[T]` | a single scalar `T` | The query selects **one column** (e.g. `SELECT email` → `[]string`). |
| `pgx.RowToMap` | a `map[string]any` | Truly dynamic columns; you don't have a struct. Loses type safety. |

```go
// Single-column query → slice of scalars. No struct needed.
emails, err := pgx.CollectRows(rows, pgx.RowTo[string])

// Slice of pointers, e.g. when structs are large or you want []*Account.
accounts, err := pgx.CollectRows(rows, pgx.RowToAddrOfStructByName[Account])
```

`RowToStructByName` vs `...ByPos` is a real decision: **by-name** is robust to column reordering and self-documenting via `db` tags, at a tiny reflection cost; **by-position** is marginally faster and tag-free but breaks silently if the `SELECT`'s column order changes. Prefer by-name everywhere except the hottest paths where you've measured it matters.

### 6.4 CollectOneRow — exactly one, into a struct **[B/I]**

`QueryRow(...).Scan(...)` is great for scalars but makes you list every field. `pgx.CollectOneRow(rows, mapper)` gives you the struct-mapping ergonomics for the single-row case — and it returns `pgx.ErrNoRows` when there's no row, exactly like `QueryRow`.

```go
func getAccount(ctx context.Context, pool *pgxpool.Pool, id int64) (Account, error) {
	rows, err := pool.Query(ctx,
		"SELECT id, owner, email, balance, created_at FROM accounts WHERE id = $1", id)
	if err != nil {
		return Account{}, err
	}
	acc, err := pgx.CollectOneRow(rows, pgx.RowToStructByName[Account])
	if errors.Is(err, pgx.ErrNoRows) {
		return Account{}, ErrAccountNotFound
	}
	return acc, err
}
```

> **⚡ v5 gotcha — `CollectOneRow` vs `CollectExactlyOneRow`:** `CollectOneRow` returns the first row and ignores any extra rows. If you want to *assert* the query returned exactly one row (erroring if it returned two), use `pgx.CollectExactlyOneRow`, which returns `pgx.ErrTooManyRows` when more than one row comes back. For a primary-key lookup either is fine; for "there should be exactly one match" invariants, the stricter one catches data bugs.

### 6.5 Lax mapping and embedded structs **[I]**

`RowToStructByNameLax` is the escape valve for the common case where **one struct serves several queries** that select different subsets of columns. With the strict mapper, every struct field must have a matching column or you get an error; Lax lets unmatched struct fields stay at their zero value.

```go
type Account struct {
	ID      int64  `db:"id"`
	Email   string `db:"email"`
	Balance int64  `db:"balance"`
	Secret  string `db:"secret_totp"` // not selected by the query below
}

// A "public profile" query that omits Secret. Strict mapping would error because
// Secret has no matching column; Lax leaves Secret == "".
rows, _ := pool.Query(ctx, "SELECT id, email FROM accounts WHERE id = $1", id)
acc, err := pgx.CollectOneRow(rows, pgx.RowToStructByNameLax[Account])
```

pgx's by-name mappers also understand **embedded structs**, flattening them so a shared `type Timestamps struct { CreatedAt, UpdatedAt time.Time }` embedded in many models maps its columns automatically. This keeps model definitions DRY across a large schema.

---

## 7. NULLs and the pgtype System

### 7.1 Why NULL is hard in Go **[B]**

SQL has a value Go's basic types can't represent: **`NULL`**, meaning "no value." A Go `string` is always *some* string (`""` at worst); a Go `int64` is always *some* number (`0`). So when a nullable column comes back `NULL`, where does it go? Scanning `NULL` into a `*string`'s target `string` fails — there's no string to write. You need a type that can represent **"present" vs "absent"** as a first-class distinction. pgx gives you three ways to do it; pick per situation.

### 7.2 Option A — pointers **[B]**

The simplest: scan a nullable column into a **pointer**. `NULL` becomes a `nil` pointer; a value becomes a non-nil pointer to it. Natural and idiomatic for optional fields.

```go
type Profile struct {
	ID       int64   `db:"id"`
	Nickname *string `db:"nickname"` // nullable column → pointer. nil == SQL NULL.
	Age      *int32  `db:"age"`
}
// After scanning: if p.Nickname == nil the column was NULL; otherwise *p.Nickname
// is the value. You must nil-check before dereferencing — the compiler won't.
```

Pointers are clean for structs you marshal to JSON (a nil pointer omits or nulls the field naturally). The cost is nil-checks everywhere you read them, and the mild awkwardness of taking addresses when you *write* them back.

### 7.3 Option B — the pgtype null wrappers **[B/I]**

pgx ships **explicit nullable value types** in `pgtype`: each is a struct with the value plus a `Valid bool`. `Valid == false` means SQL `NULL`. These make presence explicit without pointers and are the most self-documenting option.

```go
import "github.com/jackc/pgx/v5/pgtype"

type Profile struct {
	ID       int64            `db:"id"`
	Nickname pgtype.Text      `db:"nickname"`      // .String + .Valid
	Age      pgtype.Int4      `db:"age"`           // .Int32  + .Valid
	Joined   pgtype.Timestamptz `db:"joined_at"`   // .Time   + .Valid
	Balance  pgtype.Numeric   `db:"balance"`       // exact decimal (§8.3)
}

// Reading:
if p.Nickname.Valid {
	use(p.Nickname.String)
}

// Writing a NULL vs a value:
nick := pgtype.Text{String: "ada", Valid: true} // a value
none := pgtype.Text{Valid: false}               // SQL NULL
pool.Exec(ctx, "UPDATE profiles SET nickname = $1 WHERE id = $2", none, id)
```

The pgtype wrappers are the right default for exact numerics (`pgtype.Numeric` for money — never a float, §8.3), for UUIDs (`pgtype.UUID`), and anywhere the `Valid` flag reads more clearly than a nil pointer. `.Valid` is impossible to forget in the way a nil-deref is easy to hit.

### 7.4 Option C — database/sql null types **[I]**

Go's stdlib has `sql.NullString`, `sql.NullInt64`, `sql.NullTime`, etc. — `{Value, Valid}` structs just like pgtype's. pgx accepts them for backward compatibility, so if you're migrating code or sharing models with a `database/sql` codebase, they work. For new pgx-native code, prefer the `pgtype` wrappers (they cover PostgreSQL types the `sql.Null*` set doesn't, like `Numeric`, `UUID`, arrays, and `jsonb`).

### 7.5 Choosing, in one table **[I]**

| Situation | Best choice |
|---|---|
| Optional field you'll JSON-marshal | Pointer (`*string`) — nil serializes cleanly. |
| Money / exact decimal | `pgtype.Numeric` — never a float, ever (§8.3). |
| UUID column | `pgtype.UUID` (or register `google/uuid`, §8.5). |
| Clear "present vs absent" semantics in domain code | `pgtype.*` wrappers — `.Valid` is explicit. |
| Sharing models with a `database/sql` codebase | `sql.Null*`. |

The meta-rule: **decide per column, not per project.** A single struct freely mixes a plain `int64` (NOT NULL column), a `*string` (nullable, JSON-bound), and a `pgtype.Numeric` (money). Match the Go type to the column's nullability and semantics.

---

## 8. PostgreSQL Data Types in Go

pgx's "toolkit" half shines here: it maps PostgreSQL's rich type system to Go far more completely than `database/sql` + `lib/pq` ever did. This section is a tour of the types you'll actually hit.

### 8.1 The everyday scalar mappings **[B]**

| PostgreSQL | Go (scan into) | Notes |
|---|---|---|
| `int2` / `int4` / `int8` | `int16` / `int32` / `int64` | Use the width that matches; `int` also works for `int4`/`int8`. |
| `text` / `varchar` | `string` | `varchar(n)` and `text` both → `string`. |
| `bool` | `bool` | |
| `float4` / `float8` | `float32` / `float64` | **Never for money** (§8.3). |
| `numeric` / `decimal` | `pgtype.Numeric` | Exact; the money type. |
| `bytea` | `[]byte` | Raw bytes. |
| `timestamptz` | `time.Time` / `pgtype.Timestamptz` | **Always prefer `timestamptz` over `timestamp`** (§8.4). |
| `date` | `time.Time` / `pgtype.Date` | |
| `uuid` | `pgtype.UUID` / `[16]byte` / (registered) `uuid.UUID` | §8.5. |
| `json` / `jsonb` | struct / `[]byte` / `json.RawMessage` | §8.2. |
| `int4[]`, `text[]`, … | `[]int32`, `[]string`, … | §8.6. |

For NOT NULL columns you scan straight into the plain Go type; for nullable ones use one of §7's strategies.

### 8.2 JSON and JSONB **[I]**

PostgreSQL's `jsonb` is a first-class column type, and pgx has a built-in JSON codec, so you can round-trip Go structs through it **transparently** — pass a struct as a parameter and pgx marshals it; scan a `jsonb` column into a struct pointer and pgx unmarshals it. No manual `json.Marshal` at the call site.

```go
type Address struct {
	Street string `json:"street"`
	City   string `json:"city"`
	Zip    string `json:"zip"`
}

// WRITE: pass the Go struct directly; pgx marshals it to jsonb via encoding/json.
addr := Address{Street: "1 Infinite Loop", City: "Cupertino", Zip: "95014"}
_, err := pool.Exec(ctx,
	"UPDATE accounts SET billing_address = $1 WHERE id = $2", addr, id)

// READ: scan the jsonb column straight into a struct pointer; pgx unmarshals.
var got Address
err = pool.QueryRow(ctx,
	"SELECT billing_address FROM accounts WHERE id = $1", id).Scan(&got)
```

If you want the raw bytes instead of unmarshaling (to pass through, log, or defer parsing), scan into `json.RawMessage` or `[]byte`. Note the JSON codec uses the **`json` struct tags**, not the `db` tags — inside a JSON document you're back in `encoding/json` land.

> **jsonb vs json:** use `jsonb` (binary, indexable, deduplicated keys) for storage you'll query; use `json` (text, preserves formatting/key order) only when you must keep the exact input bytes. pgx handles both identically from Go.

### 8.3 numeric for money — the non-negotiable rule **[I/A]**

**Never store or compute money in a floating-point type.** `float64` cannot exactly represent most decimal fractions — `0.1 + 0.2 != 0.3` — and those rounding errors accumulate into real, auditable discrepancies. Every banking guide in this library repeats this because it is the single most common financial-software bug. PostgreSQL's exact type is `numeric` (aka `decimal`); pgx maps it to **`pgtype.Numeric`**.

```go
type Account struct {
	ID      int64          `db:"id"`
	Balance pgtype.Numeric `db:"balance"` // numeric(20,2) in the schema — exact.
}

// pgtype.Numeric holds an arbitrary-precision decimal (a big.Int coefficient + a
// base-10 exponent), so 0.01 is exactly 0.01. Arithmetic is exact.
```

Two practical patterns, both valid:

1. **Store money as `numeric` and carry `pgtype.Numeric`** end-to-end. Most correct; convert to/from a decimal library (e.g. `shopspring/decimal`) at the edges for arithmetic.
2. **Store money as an integer number of minor units** (cents) in a `bigint`, and carry `int64`. Simple and exact for currencies with fixed minor units; you format to dollars only for display. Many of this library's examples use this because a plain `int64` is easy to reason about — just be consistent and document the unit.

Whichever you pick, the forbidden option is `float`. `numeric` (or integer cents), never `float4`/`float8`, for anything representing money.

### 8.4 Timestamps and time zones **[I]**

Use **`timestamptz`** (timestamp *with* time zone) for every point-in-time value, and scan into `time.Time`. Despite the name, `timestamptz` doesn't store a zone — it stores an absolute instant (UTC internally) and converts on the way in/out based on the session `timezone`. `timestamp` (without tz) stores a wall-clock reading with no zone, which is ambiguous the moment two servers in different zones touch it. Store instants as `timestamptz`; convert to a user's local zone only for *display*.

```go
// pgx scans timestamptz into time.Time in UTC (or the session tz). Do time math
// in Go with the time package; store back as timestamptz.
var createdAt time.Time
pool.QueryRow(ctx, "SELECT created_at FROM accounts WHERE id = $1", id).Scan(&createdAt)
```

For nullable timestamps, `pgtype.Timestamptz` (with `.Valid`) or `*time.Time`.

### 8.5 UUIDs **[I]**

PostgreSQL's `uuid` type maps to **`pgtype.UUID`** out of the box (a `{Bytes [16]byte, Valid bool}`). If your codebase uses `github.com/google/uuid` (very common), you'll want to scan straight into `uuid.UUID` rather than converting. Register the integration once, in the pool's `AfterConnect` hook:

```go
import (
	pgxuuid "github.com/vgarvardt/pgx-google-uuid/v5"
	"github.com/jackc/pgx/v5"
)

cfg.AfterConnect = func(ctx context.Context, conn *pgx.Conn) error {
	// Teaches this connection's TypeMap how to encode/decode google/uuid.UUID.
	pgxuuid.Register(conn.TypeMap())
	return nil
}
// After this, a `db:"id"` field typed uuid.UUID scans and binds directly.
```

Without the integration, use `pgtype.UUID` and call `.Bytes`/construct as needed. Either is fine; the google/uuid integration just removes friction if that's the UUID type the rest of your code already uses.

### 8.6 Arrays, enums, and composite types **[I/A]**

**Arrays** map to Go slices directly — `int4[]` ↔ `[]int32`, `text[]` ↔ `[]string`. This is what powers the `= ANY($1)` trick in §5.5 and lets you read/write array columns naturally:

```go
var tags []string
pool.QueryRow(ctx, "SELECT tags FROM posts WHERE id = $1", id).Scan(&tags)
pool.Exec(ctx, "UPDATE posts SET tags = $1 WHERE id = $2", []string{"go", "sql"}, id)
```

**Enums** (`CREATE TYPE role AS ENUM ('user','admin')`) arrive as `string` by default — scan into a `string` (or your own `type Role string`) and it just works. For richer handling you can register the enum type in `AfterConnect`, but treating it as a string is the common, simple path.

**Composite types** and **domains** need registration on the connection's `TypeMap` (via `AfterConnect`) so pgx knows their structure before it can decode them. This is advanced and rarely needed in application code; when you do hit it, the pattern is: `conn.TypeMap().RegisterType(...)` after querying the type's OID. For 95% of apps, scalars + json + arrays cover everything.

---

## 9. Errors: PgError, SQLSTATE, ErrNoRows

### 9.1 The two error categories **[B]**

Every error pgx hands you is one of two kinds, and you handle them differently:

1. **Sentinel errors from pgx itself** — `pgx.ErrNoRows` (a `QueryRow`/`CollectOneRow` found nothing), `pgx.ErrTooManyRows`, `pgx.ErrTxClosed`. You match these with `errors.Is`.
2. **Errors from PostgreSQL** — the server rejected the statement: a unique-constraint violation, a foreign-key failure, a check-constraint breach, a syntax error. These arrive as **`*pgconn.PgError`**, which carries the SQL standard's **SQLSTATE** code plus rich detail. You extract them with `errors.As`.

Handling both correctly is what separates a robust data layer from one that returns "500 internal error" for a user who simply picked a taken username.

### 9.2 pgx.ErrNoRows **[B]**

Covered already in §4.2, but to state the rule crisply: a query that legitimately returns zero rows is **not** an exceptional failure — it's "not found," which is usually a `404` or a domain "does not exist," not a `500`. Always distinguish it:

```go
err := pool.QueryRow(ctx, "SELECT id FROM accounts WHERE email = $1", email).Scan(&id)
switch {
case errors.Is(err, pgx.ErrNoRows):
	return ErrNotFound // → HTTP 404
case err != nil:
	return err        // → HTTP 500
}
```

`pgx.ErrNoRows` is defined to equal `sql.ErrNoRows`, so code that already checks the stdlib sentinel keeps working through the `stdlib` adapter.

### 9.3 PgError and SQLSTATE codes **[I]**

When PostgreSQL rejects a statement, `errors.As(err, &pgErr)` gives you a `*pgconn.PgError` with everything the server said. The key field is **`Code`** — the five-character SQLSTATE. You branch on it to turn database constraints into meaningful domain errors.

```go
import "github.com/jackc/pgx/v5/pgconn"

_, err := pool.Exec(ctx,
	"INSERT INTO accounts (email) VALUES ($1)", email)

var pgErr *pgconn.PgError
if errors.As(err, &pgErr) {
	switch pgErr.Code {
	case "23505": // unique_violation
		// A row with this email already exists. This is a normal, expected outcome
		// (the user picked a taken email) → surface a 409 Conflict, not a 500.
		return fmt.Errorf("%w: %s", ErrEmailTaken, pgErr.ConstraintName)
	case "23503": // foreign_key_violation
		return ErrReferencedRowMissing
	case "23514": // check_violation
		return ErrConstraintFailed
	}
	// pgErr also has: .Message, .Detail, .Hint, .ConstraintName, .TableName,
	// .ColumnName, .SchemaName — everything you need to log or map precisely.
}
```

The SQLSTATE codes you'll actually branch on:

| Code | Name | Typical meaning → response |
|---|---|---|
| `23505` | `unique_violation` | Duplicate key (email/username taken) → **409 Conflict**. |
| `23503` | `foreign_key_violation` | Referenced row missing or still referenced → **400/409**. |
| `23502` | `not_null_violation` | Required column was NULL → **400** (usually a bug). |
| `23514` | `check_violation` | A `CHECK` constraint failed (e.g. `balance >= 0`) → **400/422**. |
| `40001` | `serialization_failure` | Serializable-isolation conflict → **retry the transaction** (§10.5). |
| `40P01` | `deadlock_detected` | Deadlock; one txn was chosen as victim → **retry**. |
| `55P03` | `lock_not_available` | `SELECT ... FOR UPDATE NOWAIT` couldn't lock → **409/back off**. |

Mapping `23505` to a friendly "that email is taken" (409) instead of a generic 500 is the difference between a helpful API and an opaque one. And `40001`/`40P01` are special: they mean "nothing was wrong with your request, the database just couldn't serialize concurrent work — try again," which drives the retry loop in §10.5.

### 9.4 Wrapping and the errors chain **[I]**

Wrap DB errors with `%w` so callers up the stack can still `errors.Is`/`errors.As` through them, but add context so a log line tells you *where* it happened:

```go
if err != nil {
	return fmt.Errorf("insertAccount(%s): %w", email, err)
}
// A caller can STILL do errors.As(err, &pgErr) or errors.Is(err, pgx.ErrNoRows)
// through the wrapper, while your logs show the operation and inputs.
```

Never log the raw DSN or full query with bound parameters at error level in production — parameters can contain PII or secrets. Log the SQLSTATE, constraint name, and a non-sensitive operation label instead (§23).

---

## 10. Transactions

### 10.1 What a transaction guarantees **[B]**

A **transaction** groups several statements so they succeed or fail *as one unit* — the **atomic** "A" in ACID. Transfer money and you must debit one account *and* credit another; if the credit fails after the debit, you cannot leave the money vanished. Wrap both in a transaction and either both commit or neither does. Beyond atomicity you also get **isolation**: concurrent transactions don't see each other's half-finished work.

### 10.2 The Begin / Commit / Rollback shape — and the defer idiom **[B]**

pgx transactions follow a precise pattern. Learn it exactly, because the safety depends on the details.

```go
func transfer(ctx context.Context, pool *pgxpool.Pool, from, to int64, amount int64) error {
	// Begin checks out a connection and issues BEGIN. The returned tx is bound to
	// that one connection for its whole life.
	tx, err := pool.Begin(ctx)
	if err != nil {
		return err
	}
	// THE KEY IDIOM: defer a Rollback immediately. If we return anywhere before
	// Commit — an error, a panic, an early return — this rolls the txn back and
	// releases the connection. Rolling back an ALREADY-committed txn is a harmless
	// no-op (it returns pgx.ErrTxClosed, which we ignore), so this is safe to always
	// defer. This single line guarantees you never leak an open transaction.
	defer tx.Rollback(ctx)

	// All statements go through `tx`, NOT `pool`. Using `pool` here would run on a
	// DIFFERENT connection, OUTSIDE this transaction — a classic silent bug.
	tag, err := tx.Exec(ctx,
		"UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1",
		amount, from)
	if err != nil {
		return err // defer rolls back
	}
	if tag.RowsAffected() == 0 {
		return ErrInsufficientFunds // the WHERE balance >= amount guard failed
	}

	if _, err := tx.Exec(ctx,
		"UPDATE accounts SET balance = balance + $1 WHERE id = $2", amount, to); err != nil {
		return err // defer rolls back — the debit above is undone too
	}

	// Commit makes everything durable and permanent. Only after this returns nil is
	// the transfer real. The deferred Rollback now no-ops.
	return tx.Commit(ctx)
}
```

Three rules the pattern encodes:

1. **`defer tx.Rollback(ctx)` right after `Begin`** — the guaranteed cleanup, safe even after a successful `Commit`.
2. **Every statement runs on `tx`, never `pool`** — mixing them silently runs work outside the transaction.
3. **`Commit` is the only thing that makes it real** — until it returns `nil`, assume nothing persisted.

### 10.3 BeginFunc — the pattern, packaged **[I]**

The defer-rollback/commit dance is boilerplate you'll write for every transaction, and forgetting a piece is dangerous. `pgx.BeginFunc(ctx, pool, fn)` wraps it: it begins, runs your function, and **commits if `fn` returns nil, rolls back if it returns an error** — you can't forget either.

```go
err := pgx.BeginFunc(ctx, pool, func(tx pgx.Tx) error {
	if _, err := tx.Exec(ctx, "UPDATE accounts SET balance = balance - $1 WHERE id = $2", amt, from); err != nil {
		return err
	}
	_, err := tx.Exec(ctx, "UPDATE accounts SET balance = balance + $1 WHERE id = $2", amt, to)
	return err
})
// If fn returned nil → committed. If it returned an error (or panicked) → rolled
// back, and that error is returned to you. No defer, no manual Commit.
```

Prefer `BeginFunc` for straightforward transactions — it's the same semantics with the footguns removed. Use the manual `Begin`/`defer`/`Commit` form when you need finer control (e.g. inspecting the commit error separately, or conditional commit logic).

### 10.4 Isolation levels and access mode **[I/A]**

By default a transaction runs at PostgreSQL's `READ COMMITTED` isolation. For stronger guarantees — or a read-only transaction the server can optimize — pass **`pgx.TxOptions`** to `BeginTx`:

```go
tx, err := pool.BeginTx(ctx, pgx.TxOptions{
	IsoLevel:   pgx.Serializable,  // strongest: as if transactions ran one at a time
	AccessMode: pgx.ReadWrite,     // or pgx.ReadOnly for a pure read txn
})
```

| IsoLevel | Guarantee | Cost |
|---|---|---|
| `pgx.ReadCommitted` (default) | Each statement sees rows committed before *it* started. | Cheapest; non-repeatable reads possible. |
| `pgx.RepeatableRead` | The whole txn sees a single snapshot; re-reads are stable. | Can fail with `40001` on write conflicts. |
| `pgx.Serializable` | As if txns ran serially — no anomalies at all. | Highest conflict rate; **must** retry on `40001`. |

`Serializable` is the correct choice for money-critical invariants (e.g. "the sum of these balances must never go negative") because it makes PostgreSQL detect and reject any concurrent interleaving that would violate serial-equivalence — but it *pushes the burden of retrying onto you* (§10.5). `ReadOnly` lets PostgreSQL skip some bookkeeping and is a nice signal of intent for report queries.

### 10.5 Retrying serialization failures **[A]**

At `RepeatableRead`/`Serializable`, PostgreSQL may abort a transaction with SQLSTATE `40001` (serialization failure) or `40P01` (deadlock) — meaning "your logic was fine, but concurrent activity made this interleaving impossible; run the whole thing again." This is **expected** under those isolation levels, and the correct response is a bounded retry loop.

```go
func withRetry(ctx context.Context, pool *pgxpool.Pool, fn func(pgx.Tx) error) error {
	const maxAttempts = 3
	for attempt := 1; ; attempt++ {
		err := pgx.BeginTxFunc(ctx, pool, pgx.TxOptions{IsoLevel: pgx.Serializable}, fn)
		if err == nil {
			return nil
		}
		var pgErr *pgconn.PgError
		// Only 40001/40P01 are retryable — everything else is a real error.
		if errors.As(err, &pgErr) && (pgErr.Code == "40001" || pgErr.Code == "40P01") && attempt < maxAttempts {
			continue // optionally sleep with backoff+jitter here
		}
		return err
	}
}
```

The retry must wrap the **entire** transaction (a retried txn re-reads fresh snapshots), the function must be **idempotent** with respect to being re-run, and the loop must be **bounded** so a genuinely stuck workload doesn't spin forever. This pattern is how serializable isolation is used in practice — the strong guarantee is only correct *with* the retry loop.

### 10.6 Nested transactions and savepoints **[A]**

PostgreSQL has no true nested transactions, but it has **savepoints** — named points you can roll back *to* without aborting the whole transaction. pgx exposes this transparently: calling `Begin` on an existing `tx` creates a savepoint, and its `Rollback` rolls back to that savepoint. This lets a sub-operation fail and be undone while the outer transaction continues.

```go
tx, _ := pool.Begin(ctx)
defer tx.Rollback(ctx)
// ... outer work ...

sp, _ := tx.Begin(ctx)            // actually a SAVEPOINT
if err := trySomething(ctx, sp); err != nil {
	sp.Rollback(ctx)          // ROLLBACK TO SAVEPOINT — outer tx still alive
} else {
	sp.Commit(ctx)            // RELEASE SAVEPOINT
}
// ... more outer work, then tx.Commit(ctx) ...
```

Savepoints are useful for "try this optional step; if it fails, skip it but keep the rest" logic. Use them sparingly — deeply nested savepoints get hard to reason about, and each adds server-side bookkeeping.

---

## 11. Batching — Fewer Round-Trips

### 11.1 The round-trip problem **[I]**

Every query is a network round-trip: send, wait, receive. Run 100 `INSERT`s in a loop and you pay 100 round-trips — if the database is 1 ms away, that's 100 ms of pure latency doing nothing but waiting. **Batching** sends many queries in *one* round-trip: pgx pipelines them to the server, the server runs them back-to-back, and the results stream back together. For N queries you pay ~1 round-trip instead of N. On a chatty operation this is a 10–100× latency win.

### 11.2 pgx.Batch **[I]**

You queue statements onto a `*pgx.Batch`, send the whole thing with `SendBatch`, then read results **in the same order you queued them**.

```go
batch := &pgx.Batch{}
// Queue as many as you like — mix of INSERT/UPDATE/SELECT, each with its own args.
for _, e := range events {
	batch.Queue("INSERT INTO audit_log (actor, action, at) VALUES ($1, $2, now())",
		e.Actor, e.Action)
}

// SendBatch pipelines the whole batch in one network round-trip.
br := pool.SendBatch(ctx, batch)
defer br.Close() // ALWAYS close the batch results — like rows.Close().

// Read results in queue order. For INSERTs we only care about errors, so Exec().
for range events {
	if _, err := br.Exec(); err != nil {
		return err
	}
}
// Must Close() (deferred above) before the connection can be reused.
```

For queued `SELECT`s you call `br.Query()` / `br.QueryRow()` instead of `br.Exec()`, again **in queue order**. The ordering contract is strict: the *i*-th result you read corresponds to the *i*-th statement you queued.

### 11.3 The QueuedQuery callback form **[I/A]**

`Queue` returns a `*pgx.QueuedQuery` on which you can attach result-handling callbacks, so you write the per-statement result logic right next to the statement instead of in a separate read loop. This keeps a heterogeneous batch readable.

```go
batch := &pgx.Batch{}
batch.Queue("INSERT INTO accounts (email) VALUES ($1) RETURNING id", email).
	QueryRow(func(row pgx.Row) error {
		return row.Scan(&newID) // handle THIS statement's result inline
	})
batch.Queue("UPDATE stats SET signups = signups + 1").
	Exec(func(ct pgconn.CommandTag) error {
		log.Printf("stats updated: %d", ct.RowsAffected())
		return nil
	})

err := pool.SendBatch(ctx, batch).Close() // Close() runs the callbacks & returns the first error
```

> **Batch + transaction:** a batch is **not** automatically a transaction — if statement 3 fails, statements 1–2 may already be committed. To make a batch atomic, run it inside a transaction: `tx.SendBatch(...)`. And note batching trades latency for **all-or-nothing feedback timing** — you find out about a failure only when you read that statement's result, so always drain/close the batch to surface every error.

### 11.4 When to batch **[I/A]**

Batch when you have **many independent statements** to run against the same pool and latency dominates (bulk audit writes, fan-out updates, warming several caches). Do **not** reach for batching when a single set-based SQL statement would do the job better — inserting 1000 rows is usually better as one `INSERT ... SELECT`, `unnest()`, or `CopyFrom` (§12) than 1000 queued `INSERT`s. Batching is for *heterogeneous* or *unavoidably-separate* statements; `CopyFrom` is for *bulk homogeneous inserts*.

---

## 12. CopyFrom — Bulk Loading

### 12.1 Why COPY is special **[I]**

Inserting a large number of rows with individual `INSERT`s — even batched — has per-row protocol overhead. PostgreSQL's **`COPY`** protocol is a dedicated bulk-load path that streams rows in a compact binary format, bypassing most of that overhead. It is *the* fastest way to get many rows into a table, often an order of magnitude faster than row-by-row inserts. pgx exposes it as **`CopyFrom`**.

### 12.2 CopyFrom with a slice **[I]**

You give `CopyFrom` the target table, the column list, and a **source** of rows. The simplest source is `pgx.CopyFromRows` over a `[][]any`:

```go
type Account struct {
	Owner   string
	Email   string
	Balance int64
}

func bulkInsert(ctx context.Context, pool *pgxpool.Pool, accounts []Account) (int64, error) {
	// Build [][]any — each inner slice is one row, values in COLUMN order.
	rows := make([][]any, len(accounts))
	for i, a := range accounts {
		rows[i] = []any{a.Owner, a.Email, a.Balance}
	}

	// CopyFrom(table, columns, source) → number of rows copied.
	return pool.CopyFrom(ctx,
		pgx.Identifier{"accounts"},                 // table name, injection-safe identifier
		[]string{"owner", "email", "balance"},      // columns, matching the row order above
		pgx.CopyFromRows(rows),                     // the source
	)
}
```

`pgx.Identifier{"accounts"}` (or `pgx.Identifier{"public", "accounts"}` for schema-qualified) safely quotes the table name — you don't build it as a string. The return is the row count copied.

### 12.3 CopyFromSlice — streaming without materializing **[I/A]**

If your data comes from a stream (a large file, a paged API) and you don't want to build the whole `[][]any` in memory, **`pgx.CopyFromSlice`** takes a length and a function that produces row *i* on demand:

```go
n, err := pool.CopyFrom(ctx,
	pgx.Identifier{"accounts"},
	[]string{"owner", "email", "balance"},
	pgx.CopyFromSlice(len(accounts), func(i int) ([]any, error) {
		a := accounts[i]
		return []any{a.Owner, a.Email, a.Balance}, nil
	}),
)
```

For a fully streaming source of unknown length, implement the `pgx.CopyFromSource` interface (a `Next()`/`Values()`/`Err()` trio, like `pgx.Rows` in reverse) and feed rows straight from your reader — pgx never holds more than the current row in memory.

### 12.4 CopyFrom's limits and the upsert caveat **[I/A]**

`CopyFrom` is a firehose, and firehoses have constraints you must know:

- **No `ON CONFLICT`.** `COPY` is a raw bulk insert — it can't do upserts. If a row violates a unique constraint, the **whole copy fails**. For "load these, ignoring/updating duplicates," the standard trick is: `COPY` into a `TEMP` table, then `INSERT INTO real SELECT ... FROM temp ON CONFLICT ...` — bulk speed plus conflict handling.
- **All-or-nothing on constraint errors** — a single bad row aborts the batch. Validate before loading, or use the temp-table staging pattern.
- **Triggers and rules still fire** (unless the table is `COPY`-optimized), so a per-row trigger can erase the speed advantage.

Choose `CopyFrom` for the initial bulk load, imports, and ETL. Choose batched or set-based `INSERT ... ON CONFLICT` when you need upsert semantics. They're complementary tools, not competitors.

---

## 13. Prepared Statements, the Statement Cache and QueryExecMode

### 13.1 What "prepared" means and why you get it for free **[A]**

A **prepared statement** is a query the server has already parsed and planned, referenced by a name so subsequent executions skip the parse/plan step and just bind new parameters. It's both a **performance** win (parse once, run many) and the mechanism that makes `$1` parameters safe. In many drivers you manage prepared statements by hand. **pgx does it automatically:** by default, every query you run through the extended protocol is prepared and its plan **cached per connection**, keyed by the SQL text. Run the same SQL again on that connection and pgx reuses the prepared statement transparently. You get the benefit without writing a line of prepare/deallocate code.

This automatic caching is controlled by the **query exec mode**, and understanding the modes matters the day you put a connection pooler (PgBouncer) between your app and PostgreSQL — because that's when the default breaks.

### 13.2 The five QueryExecModes **[A]**

Every `Query`/`QueryRow`/`Exec` runs in one of five modes. You can set a default on the config (`ConnConfig.DefaultQueryExecMode`) or override per-call by passing the mode as the first argument.

| Mode | What it does | Round-trips | Use when |
|---|---|---|---|
| `QueryExecModeCacheStatement` **(default)** | Prepares & **caches** the statement per connection, reused by SQL text. | Fewest (after warm-up) | Direct connection to Postgres. The fast default. |
| `QueryExecModeCacheDescribe` | Caches the parameter/result *description* but not a server-side prepared statement. | Few | Behind a pooler that keeps sessions but not prepared statements. |
| `QueryExecModeDescribeExec` | Describes the statement each time, no caching. | More | Schema may change between calls; safe but slower. |
| `QueryExecModeExec` | Uses the extended protocol but **skips prepared statements** entirely. | Moderate | **Transaction-pooling PgBouncer** — the common cloud setup. |
| `QueryExecModeSimpleProtocol` | Uses the **simple** query protocol; parameters are formatted client-side (still escaped safely by pgx). | Fewest per query, no server prepare | Last resort behind poolers that don't support the extended protocol; also multi-statement strings. |

### 13.3 The PgBouncer problem — the reason this section exists **[A]**

Here is the failure that sends people to this section. **PgBouncer in transaction-pooling mode** multiplexes many client connections onto a few server connections, handing them out *per transaction*. But pgx's default `CacheStatement` mode prepares a statement on "connection X" and expects it to still be there next time — and with transaction pooling, your next query may land on a *different* server connection that never saw that prepared statement. Result: intermittent `prepared statement "stmtcache_..." does not exist` errors that are maddening to debug because they depend on pool timing.

The fix is to tell pgx not to rely on per-connection prepared statements. Set the mode to `QueryExecModeExec` (or `SimpleProtocol`):

```go
cfg, _ := pgxpool.ParseConfig(dsn)
// Behind a transaction-pooling PgBouncer, disable prepared-statement caching so a
// query never depends on a prepared statement living on a specific server conn.
cfg.ConnConfig.DefaultQueryExecMode = pgx.QueryExecModeExec
pool, _ := pgxpool.NewWithConfig(ctx, cfg)
```

You can also do this via the DSN in some setups, but setting it on the config is explicit and reviewable. **Rule of thumb:** direct connection or session-pooling PgBouncer → keep the default `CacheStatement`; **transaction-pooling** PgBouncer → `QueryExecModeExec`. Getting this right is the single most common "pgx works locally but errors in prod" fix, because prod often adds a pooler that dev doesn't have.

### 13.4 Explicit named prepared statements **[A]**

You rarely need them (the auto-cache handles the common case), but you *can* prepare a named statement explicitly on a connection — useful for a hot query you want prepared exactly once at connection setup, via `AfterConnect`:

```go
cfg.AfterConnect = func(ctx context.Context, conn *pgx.Conn) error {
	_, err := conn.Prepare(ctx, "getAccount", "SELECT id, email, balance FROM accounts WHERE id = $1")
	return err
}
// Later, run it by name:
pool.QueryRow(ctx, "getAccount", id).Scan(&a.ID, &a.Email, &a.Balance)
```

This is an optimization for specific hot paths, not something to reach for by default — the automatic statement cache already prepares your queries the first time they run.

---

## 14. LISTEN / NOTIFY — Postgres Pub/Sub

### 14.1 What it is and why it's a superpower **[A]**

PostgreSQL has a built-in publish/subscribe system: a client runs `LISTEN channel_name` and blocks; any client (even in another process) that runs `NOTIFY channel_name, 'payload'` — or calls `pg_notify('channel', 'payload')` — wakes every listener with the payload. This is a **`database/sql` can't-do-it** feature that pgx's native API exposes directly, and it's genuinely useful: cache invalidation, pushing DB changes to a WebSocket hub, cross-instance coordination — all without adding Redis or a message broker, because the database you already have *is* the broker.

### 14.2 The listener — a dedicated connection **[A]**

A listener **blocks** waiting for notifications, so it must own a connection exclusively — you cannot share it with the pool's query traffic. Acquire a connection (or use a dedicated `pgx.Conn`) and dedicate it to listening.

```go
func listen(ctx context.Context, pool *pgxpool.Pool, channel string, handle func(payload string)) error {
	// Take a connection out of the pool and keep it for the listener's whole life.
	conn, err := pool.Acquire(ctx)
	if err != nil {
		return err
	}
	defer conn.Release()

	// Subscribe. LISTEN is a normal statement, but from now on this connection is
	// dedicated to receiving notifications — don't run other queries on it.
	if _, err := conn.Exec(ctx, "LISTEN "+pgx.Identifier{channel}.Sanitize()); err != nil {
		return err
	}

	for {
		// WaitForNotification BLOCKS until a NOTIFY arrives on a subscribed channel
		// or ctx is cancelled. This is why the connection must be dedicated.
		n, err := conn.Conn().WaitForNotification(ctx)
		if err != nil {
			return err // ctx cancelled or connection lost
		}
		// n.Channel, n.Payload (string), n.PID (the notifying backend's process id).
		handle(n.Payload)
	}
}
```

`pgx.Identifier{channel}.Sanitize()` safely quotes the channel name — important because `LISTEN` takes an identifier you can't parameterize (same rule as §5.2).

### 14.3 The notifier — any connection **[A]**

Sending is trivial and needs no dedicated connection — run it from the pool like any statement. Prefer the `pg_notify(...)` **function** over the `NOTIFY` statement because the function takes the payload as a **parameter** (injection-safe), whereas the statement wants a string literal.

```go
// Notify all listeners on "account_changed" with a JSON payload. pg_notify's
// payload is a $2 parameter — safe. (NOTIFY the statement can't parameterize it.)
_, err := pool.Exec(ctx, "SELECT pg_notify($1, $2)", "account_changed",
	fmt.Sprintf(`{"id":%d,"event":"balance_updated"}`, accountID))
```

The most powerful pattern combines this with §10: inside a transaction, do your writes *and* `SELECT pg_notify(...)`, so the notification fires **only if the transaction commits** — subscribers never hear about a change that got rolled back. That transactional coupling is something an external broker can't give you for free.

### 14.4 Production caveats **[A]**

- **Payloads are small** (8 KB limit) — send an *id/event*, not the whole row; listeners fetch details themselves.
- **Notifications are not durable** — a listener that's disconnected when a `NOTIFY` fires *misses it*. LISTEN/NOTIFY is at-most-once, fire-and-forget. If you need guaranteed delivery, use an outbox table + polling, or a real queue.
- **Reconnection matters** — a dropped listener connection loses its subscription; wrap the listener in a supervise-and-reconnect loop, and on reconnect do a full refresh (assume you missed events while down).
- **One listener connection can serve many channels** — issue multiple `LISTEN`s on the same connection and dispatch by `n.Channel`.

This makes LISTEN/NOTIFY perfect for *hints* ("something changed, go look") — like invalidating a cache or nudging a realtime hub — and wrong for anything requiring guaranteed delivery.

---

## 15. Context, Timeouts and Cancellation

### 15.1 Every pgx call takes a context — use it **[I]**

You've seen `ctx` in every example. This is not ceremony: the `context.Context` is how a query is **cancelled** and **time-bounded**. When the context is cancelled or its deadline passes, pgx sends PostgreSQL a cancellation request and the query stops — the goroutine doesn't hang, and the server stops burning CPU on a result nobody's waiting for. In a web server this is the mechanism that stops a slow query from outliving the HTTP request that triggered it.

### 15.2 Per-query timeouts **[I]**

Wrap a query in a context with a deadline so a pathological query can't hang a request forever. This is *client-side* enforcement; pair it with the *server-side* `statement_timeout` from §3.4 for defense in depth.

```go
func getAccount(ctx context.Context, pool *pgxpool.Pool, id int64) (Account, error) {
	// This query gets at most 3 seconds, regardless of the parent ctx's deadline.
	ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
	defer cancel() // ALWAYS cancel to release the timer, even on the happy path.

	var a Account
	err := pool.QueryRow(ctx, "SELECT id, email, balance FROM accounts WHERE id = $1", id).
		Scan(&a.ID, &a.Email, &a.Balance)
	return a, err
}
```

The `defer cancel()` is mandatory — not calling it leaks the timer/context resources until the parent context dies. Every `WithTimeout`/`WithCancel` gets a `defer cancel()`, no exceptions.

### 15.3 Request-scoped context in Gin **[I]**

In a Gin handler, **use the request's context** (`c.Request.Context()`), not `context.Background()`. Gin cancels the request context when the client disconnects — so if a user closes the tab, the in-flight query gets cancelled instead of running to completion for nobody.

```go
func (h *Handler) getAccount(c *gin.Context) {
	// c.Request.Context() is cancelled if the client disconnects or the server
	// shuts down — propagate it all the way to pgx so queries die with the request.
	ctx := c.Request.Context()
	acc, err := h.store.GetAccount(ctx, id)
	// ...
}
```

> **⚡ Gotcha:** don't confuse `c.Request.Context()` (the standard, cancellation-aware request context — **use this**) with Gin's own `*gin.Context` (which also implements `context.Context` but historically had surprising deadline behavior). Passing the request context down to pgx is what wires client-disconnect and server-shutdown cancellation through to the database.

### 15.4 Cancellation vs timeout errors **[I]**

When a query is cut short, the error is `context.Canceled` (someone called `cancel()` / the client left) or `context.DeadlineExceeded` (the timeout fired). Distinguish them from real DB errors so you don't alert on a user simply navigating away:

```go
switch {
case errors.Is(err, context.DeadlineExceeded):
	// The query took too long → 503/504, and worth a metric: something is slow.
case errors.Is(err, context.Canceled):
	// The caller went away (client disconnected). Usually not an error to alert on.
default:
	// A real database error.
}
```

---

## 16. Tracing, Logging and Observability

### 16.1 Seeing what pgx is doing **[A]**

In production you need to answer "what queries ran, how long did they take, which failed?" pgx exposes a **`QueryTracer`** interface — a hook pgx calls at the start and end of every query — and ships an adapter, `tracelog`, that turns those hooks into structured log lines through your logger of choice (`slog`, `zap`, `zerolog`). You can also plug in OpenTelemetry for distributed traces and metrics.

### 16.2 Query logging with tracelog + slog **[A]**

```go
import (
	"github.com/jackc/pgx/v5/tracelog"
)

cfg, _ := pgxpool.ParseConfig(dsn)
cfg.ConnConfig.Tracer = &tracelog.TraceLog{
	// Adapt Go's standard structured logger. tracelog has adapters for slog, zap,
	// zerolog, logrus — pick the one your app already uses.
	Logger:   tracelog.LoggerFunc(func(ctx context.Context, level tracelog.LogLevel, msg string, data map[string]any) {
		slog.Log(ctx, slog.LevelDebug, msg, "pgx", data)
	}),
	// LogLevelDebug logs every query (great in dev, too chatty in prod). Use
	// LogLevelError in production to log only failures, or LogLevelWarn.
	LogLevel: tracelog.LogLevelError,
}
pool, _ := pgxpool.NewWithConfig(ctx, cfg)
```

> **⚡ Security — never log parameters in prod at info/debug.** Query arguments routinely contain PII, tokens, or password hashes. `tracelog` at `LogLevelDebug` logs the SQL *and its args*. Keep production at `LogLevelError` (failures only, and scrub args there too if needed), and reserve arg-logging for local debugging. Logging bound parameters to a shipped log sink is a data-leak waiting to happen.

### 16.3 OpenTelemetry and pool metrics **[A]**

For distributed tracing, community tracers like `github.com/exaring/otelpgx` implement `QueryTracer` to emit OTel spans per query — set `cfg.ConnConfig.Tracer = otelpgx.NewTracer()` and every query becomes a span with SQL, duration, and status, stitched into the request trace. When you need *both* logging and tracing, pgx's `multitracer.Tracer` composes several tracers into one.

Separately, the pool exposes **live stats** via `pool.Stat()` — export these to Prometheus and they're often the first place you'll see trouble:

```go
s := pool.Stat()
s.TotalConns()        // total connections currently in the pool
s.AcquiredConns()     // in use right now
s.IdleConns()         // free and ready
s.EmptyAcquireCount() // times a caller had to WAIT for a connection — if this
                      // climbs, MaxConns is too low or something is leaking conns.
```

`EmptyAcquireCount` and `AcquiredConns` trending toward `MaxConns` are your early-warning signs of pool exhaustion — the leak-a-connection failure from §3.5 shows up here before it takes the service down.

---

## 17. The database/sql Bridge: Interop with goose, sqlc, Libraries

### 17.1 When you need `*sql.DB` **[I]**

Native pgx is your default, but parts of the ecosystem speak only the stdlib `database/sql` interface: **goose** migrations want a `*sql.DB`, **sqlc** in `database/sql` mode generates code against `*sql.DB`, and many general-purpose libraries (some test helpers, some query builders) accept `*sql.DB`. pgx's `stdlib` package bridges the gap — it wraps a pgx pool (or config) as a standard `*sql.DB` driven by pgx underneath. You get the library compatibility without giving up pgx as the actual driver.

### 17.2 stdlib.OpenDBFromPool — one pool, two faces **[I]**

The best pattern is to open **one** `pgxpool.Pool` for your app's native pgx code, then derive a `*sql.DB` from that *same* pool for the library boundary — so you're not running two separate connection pools against the same database.

```go
import (
	"database/sql"
	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/jackc/pgx/v5/stdlib"
)

// Your app holds this native pool for all its own queries.
pool, _ := pgxpool.New(ctx, dsn)

// Derive a *sql.DB backed by the SAME pool — hand THIS to goose / sqlc / etc.
sqlDB := stdlib.OpenDBFromPool(pool)
defer sqlDB.Close()

// e.g. run goose migrations through the standard interface it requires:
// goose.SetDialect("postgres"); goose.Up(sqlDB, "migrations")
```

This is exactly the pattern the [Goose guide](GO_GOOSE_MIGRATIONS_GUIDE.md) §8 and the realtime-chat capstone use: **goose owns the schema (through the `*sql.DB` bridge), your app runs everything else through native pgx (or Ent/sqlc on top of it).** One pool, one set of connections, two API faces.

> **⚡ Note:** there's also the older `sql.Open("pgx", dsn)` form (register the driver by importing `_ "github.com/jackc/pgx/v5/stdlib"`), which creates a *separate* `database/sql`-managed pool. Prefer `OpenDBFromPool` when you already have a `pgxpool.Pool` so the two views share connections; use `sql.Open("pgx", ...)` only when a library needs a `*sql.DB` and you have no native pool to share.

### 17.3 Where pgx sits under Ent and sqlc **[I]**

To close the loop from §1.3: **Ent** connects by opening a `*sql.DB` (often via the pgx stdlib driver) and wrapping it, then runs its generated builders through it; **sqlc** with the `pgx/v5` SQL package generates code that takes a `DBTX` interface satisfied by `*pgxpool.Pool` and `pgx.Tx` — i.e. it calls *native* pgx directly. So when you use sqlc-on-pgx you're still holding a `pgxpool.Pool`, and you can freely mix sqlc-generated calls with hand-written pgx calls on the same pool and even the same transaction. Learning pgx directly is what makes that interop obvious rather than mysterious.

---

## 18. Architecture: The Repository Layer in a Gin App

### 18.1 The DBTX interface — the key abstraction **[I/A]**

Here's the single most useful pattern for structuring pgx code. Both `*pgxpool.Pool` and `pgx.Tx` implement the **same small set of methods** (`Exec`, `Query`, `QueryRow`, `SendBatch`, `CopyFrom`). Define an interface capturing those, and your repository functions can accept *either* — so the exact same `InsertAccount` works standalone (given the pool) **or** inside a transaction (given the tx). This is how you compose small repository operations into larger transactional units without duplicating code.

```go
// DBTX is satisfied by BOTH *pgxpool.Pool AND pgx.Tx. Repository methods take a
// DBTX, so a caller decides whether they run in a transaction or not.
type DBTX interface {
	Exec(ctx context.Context, sql string, args ...any) (pgconn.CommandTag, error)
	Query(ctx context.Context, sql string, args ...any) (pgx.Rows, error)
	QueryRow(ctx context.Context, sql string, args ...any) pgx.Row
}

// One repository operation. It doesn't know or care if db is the pool or a tx.
func InsertAccount(ctx context.Context, db DBTX, owner, email string) (int64, error) {
	var id int64
	err := db.QueryRow(ctx,
		"INSERT INTO accounts (owner, email, balance) VALUES ($1, $2, 0) RETURNING id",
		owner, email).Scan(&id)
	return id, err
}
```

Now the same function composes both ways:

```go
// Standalone — pass the pool.
id, err := InsertAccount(ctx, pool, "Ada", "ada@example.com")

// Inside a transaction — pass the tx, so it's atomic with other operations.
err = pgx.BeginFunc(ctx, pool, func(tx pgx.Tx) error {
	id, err := InsertAccount(ctx, tx, "Ada", "ada@example.com")   // same function!
	if err != nil {
		return err
	}
	_, err = tx.Exec(ctx, "INSERT INTO audit (account_id, event) VALUES ($1, 'created')", id)
	return err
})
```

`DBTX` is the abstraction that makes "should this be transactional?" a *caller's* decision, not something baked into each repository method. It's also exactly the interface sqlc generates against, so the pattern is familiar across the ecosystem.

### 18.2 The Store: wrapping the pool **[I/A]**

Package your pool and repository methods into a `Store` type — one object your handlers depend on, holding the pool and exposing domain operations. This is the seam where HTTP meets data.

```go
type Store struct {
	pool *pgxpool.Pool
}

func NewStore(pool *pgxpool.Pool) *Store {
	return &Store{pool: pool}
}

// A read: struct-mapped, ErrNoRows → domain error.
func (s *Store) GetAccount(ctx context.Context, id int64) (Account, error) {
	rows, err := s.pool.Query(ctx,
		"SELECT id, owner, email, balance, created_at FROM accounts WHERE id = $1", id)
	if err != nil {
		return Account{}, err
	}
	acc, err := pgx.CollectOneRow(rows, pgx.RowToStructByName[Account])
	if errors.Is(err, pgx.ErrNoRows) {
		return Account{}, ErrAccountNotFound
	}
	return acc, err
}

// A write that must be atomic: the transfer from §10, as a Store method.
func (s *Store) Transfer(ctx context.Context, from, to, amount int64) error {
	return pgx.BeginFunc(ctx, s.pool, func(tx pgx.Tx) error {
		tag, err := tx.Exec(ctx,
			"UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1",
			amount, from)
		if err != nil {
			return err
		}
		if tag.RowsAffected() == 0 {
			return ErrInsufficientFunds
		}
		_, err = tx.Exec(ctx,
			"UPDATE accounts SET balance = balance + $1 WHERE id = $2", amount, to)
		return err
	})
}
```

The `Store` keeps SQL out of your HTTP handlers, gives you one place to test the data layer, and is where domain errors (`ErrAccountNotFound`, `ErrInsufficientFunds`) are minted from raw pgx errors — so handlers map *domain* errors to status codes and never touch a `*pgconn.PgError` directly.

### 18.3 Layering: handler → store → pgx **[I/A]**

The clean separation, top to bottom: **handler** (parse/validate the HTTP request, call the store, map domain errors to status codes, write JSON) → **store/repository** (SQL, transactions, map pgx errors to domain errors) → **pgx pool** (the driver). Each layer knows only the one below it. The handler never sees SQL; the store never sees `*gin.Context`; pgx never sees HTTP. §20 assembles all three into a running app.

---

## 19. Config, .env and Files: godotenv, embed.FS, os/exec

This section covers the "everything around the queries" — loading configuration, shipping SQL files with your binary, and shelling out to database tools — because a real pgx app is more than `Query` calls.

### 19.1 A typed config loaded from the environment **[I]**

Read the environment once, at startup, into a typed struct — so the rest of the app depends on validated config, not scattered `os.Getenv` calls. godotenv populates the environment in dev; the struct is the same in every environment.

```go
// internal/platform/config/config.go
package config

import (
	"fmt"
	"os"
	"strconv"

	"github.com/joho/godotenv"
)

type Config struct {
	DatabaseURL string
	Port        string
	LogLevel    string
	MaxConns    int32
}

func Load() (Config, error) {
	// Dev convenience: load .env if present. In prod there's no file → ignored,
	// and the real env vars (from the platform's secret store) are used as-is.
	_ = godotenv.Load()

	cfg := Config{
		DatabaseURL: os.Getenv("DATABASE_URL"),
		Port:        getenv("PORT", "8080"),
		LogLevel:    getenv("LOG_LEVEL", "info"),
		MaxConns:    int32(getenvInt("DB_MAX_CONNS", 10)),
	}
	// Fail fast on missing REQUIRED config — a clear startup error beats a
	// confusing nil-pointer or empty-DSN failure deep in a request later.
	if cfg.DatabaseURL == "" {
		return Config{}, fmt.Errorf("DATABASE_URL is required")
	}
	return cfg, nil
}

func getenv(key, def string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return def
}

func getenvInt(key string, def int) int {
	if v := os.Getenv(key); v != "" {
		if n, err := strconv.Atoi(v); err == nil {
			return n
		}
	}
	return def
}
```

The two disciplines here: **load `.env` but don't require it** (so prod works without a file), and **validate required keys at startup** (fail fast, with a message that names the missing key).

### 19.2 Shipping SQL with embed.FS **[I]**

Rather than scatter SQL as string literals, keep queries and DDL in `.sql` files and embed them into the binary with `//go:embed`. The files ship *inside* the compiled binary — no "where are my .sql files in production?" problem — while staying editable and syntax-highlighted in your editor.

```go
package store

import _ "embed"

// The .sql file is compiled INTO the binary. It exists at build time in your repo
// and is embedded — there's no separate file to deploy.
//go:embed queries/get_account.sql
var getAccountSQL string

func (s *Store) GetAccount(ctx context.Context, id int64) (Account, error) {
	rows, err := s.pool.Query(ctx, getAccountSQL, id) // the embedded SQL text
	// ...
}
```

For a whole directory of SQL (e.g. migrations you run programmatically), embed an `embed.FS`:

```go
import "embed"

//go:embed migrations/*.sql
var migrationsFS embed.FS
// Pass migrationsFS to goose's SetBaseFS (via the stdlib bridge, §17) so
// migrations travel inside the binary too — no files to copy to the server.
```

Embedding is the idiomatic Go answer to "keep SQL in files, but deploy a single self-contained binary." It pairs naturally with the goose guide's embedded-migrations pattern.

### 19.3 Shelling out: psql, pg_dump, migrations via os/exec **[I]**

Sometimes you drive external Postgres tools from Go — run a migration binary on deploy, take a `pg_dump` backup, restore a fixture in a test. Use `os/exec`, and pass the connection info via the **`PG*` environment variables** or a `.pgpass` file rather than putting the password on the command line (command-line args are visible in the process list — a credential leak).

```go
import "os/exec"

func backup(ctx context.Context, dsn, outPath string) error {
	// pg_dump reads the connection from its argument; here we pass the DSN. In a
	// real script prefer PGPASSWORD/env or .pgpass so the secret isn't in argv.
	cmd := exec.CommandContext(ctx, "pg_dump", "--format=custom", "--file="+outPath, dsn)
	// Inherit env so PG* vars / PATH are available; add PGPASSWORD here if needed.
	cmd.Env = os.Environ()
	out, err := cmd.CombinedOutput()
	if err != nil {
		return fmt.Errorf("pg_dump failed: %v: %s", err, out)
	}
	return nil
}
```

`exec.CommandContext` ties the subprocess to a context, so a cancelled/timed-out backup gets killed instead of orphaned. This is the same `os/exec` pattern the [Go FS/OS/CLI guide](GO_FILESYSTEM_OS_CLI_GUIDE.md) covers in depth — here applied to database tooling. Keep secrets out of `argv`; prefer environment or `.pgpass`.

---

## 20. The Full Worked App: Gin + pgx + godotenv + Air

We now assemble everything into a small but production-shaped **accounts API**: create an account, fetch it, list accounts, and transfer money between two atomically. Every piece — config, pool tuning, graceful shutdown, the repository, error mapping, request-scoped context — is something from an earlier section, now wired together.

### 20.1 Project layout **[I]**

```text
pgx-bank/
├── .env                      # dev config (git-ignored)
├── .air.toml                 # hot-reload config
├── go.mod
├── schema.sql                # the DDL (run via goose or psql)
└── cmd/
│   └── server/
│       └── main.go           # wiring: config → pool → store → router → serve
└── internal/
    ├── config/config.go      # §19.1
    ├── store/
    │   ├── store.go          # the pool-backed Store (§18.2)
    │   └── accounts.go       # account queries
    └── api/
        ├── router.go         # Gin routes
        └── accounts.go       # handlers (§18.3)
```

### 20.2 The schema **[I]**

```sql
-- schema.sql — apply with goose (see the Goose guide) or: psql "$DATABASE_URL" -f schema.sql
CREATE TABLE IF NOT EXISTS accounts (
	id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
	owner      text        NOT NULL,
	email      text        NOT NULL UNIQUE,          -- UNIQUE → 23505 on dup (§9.3)
	balance    bigint      NOT NULL DEFAULT 0,        -- money as integer minor units (§8.3)
	created_at timestamptz NOT NULL DEFAULT now(),    -- timestamptz, always (§8.4)
	CONSTRAINT balance_non_negative CHECK (balance >= 0)  -- 23514 if violated
);
```

Note the schema *encodes the invariants*: the `UNIQUE(email)` becomes a `23505` your handler maps to `409`; the `CHECK (balance >= 0)` is a database-level guarantee that no bug can drive a balance negative, backing up the `WHERE balance >= $1` guard in the transfer.

### 20.3 The store **[I/A]**

```go
// internal/store/store.go
package store

import (
	"context"
	"errors"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"
)

var (
	ErrAccountNotFound   = errors.New("account not found")
	ErrEmailTaken        = errors.New("email already registered")
	ErrInsufficientFunds = errors.New("insufficient funds")
)

type Account struct {
	ID        int64     `db:"id"        json:"id"`
	Owner     string    `db:"owner"     json:"owner"`
	Email     string    `db:"email"     json:"email"`
	Balance   int64     `db:"balance"   json:"balance"`
	CreatedAt string    `db:"created_at" json:"created_at"`
}

type Store struct{ pool *pgxpool.Pool }

func New(pool *pgxpool.Pool) *Store { return &Store{pool: pool} }
```

```go
// internal/store/accounts.go
package store

import (
	"context"
	"errors"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgconn"
)

func (s *Store) CreateAccount(ctx context.Context, owner, email string) (Account, error) {
	rows, err := s.pool.Query(ctx,
		`INSERT INTO accounts (owner, email) VALUES ($1, $2)
		 RETURNING id, owner, email, balance, created_at::text`,
		owner, email)
	if err != nil {
		return Account{}, err
	}
	acc, err := pgx.CollectOneRow(rows, pgx.RowToStructByName[Account])
	if err != nil {
		// Map the UNIQUE(email) violation to a domain error → 409 in the handler.
		var pgErr *pgconn.PgError
		if errors.As(err, &pgErr) && pgErr.Code == "23505" {
			return Account{}, ErrEmailTaken
		}
		return Account{}, err
	}
	return acc, nil
}

func (s *Store) GetAccount(ctx context.Context, id int64) (Account, error) {
	rows, err := s.pool.Query(ctx,
		"SELECT id, owner, email, balance, created_at::text FROM accounts WHERE id = $1", id)
	if err != nil {
		return Account{}, err
	}
	acc, err := pgx.CollectOneRow(rows, pgx.RowToStructByName[Account])
	if errors.Is(err, pgx.ErrNoRows) {
		return Account{}, ErrAccountNotFound
	}
	return acc, err
}

func (s *Store) ListAccounts(ctx context.Context, limit int32) ([]Account, error) {
	rows, err := s.pool.Query(ctx,
		"SELECT id, owner, email, balance, created_at::text FROM accounts ORDER BY id LIMIT $1", limit)
	if err != nil {
		return nil, err
	}
	return pgx.CollectRows(rows, pgx.RowToStructByName[Account])
}

// Transfer moves `amount` from → to, atomically. The whole thing is one txn:
// either both balances change or neither does.
func (s *Store) Transfer(ctx context.Context, from, to, amount int64) error {
	if amount <= 0 {
		return errors.New("amount must be positive")
	}
	return pgx.BeginFunc(ctx, s.pool, func(tx pgx.Tx) error {
		// Debit with a guard: the WHERE balance >= amount means the UPDATE hits
		// zero rows if funds are short — RowsAffected()==0 detects it without a
		// separate SELECT (and without a race between checking and updating).
		tag, err := tx.Exec(ctx,
			"UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1",
			amount, from)
		if err != nil {
			return err
		}
		if tag.RowsAffected() == 0 {
			return ErrInsufficientFunds
		}
		// Credit. If the destination id doesn't exist this hits zero rows too.
		tag, err = tx.Exec(ctx,
			"UPDATE accounts SET balance = balance + $1 WHERE id = $2", amount, to)
		if err != nil {
			return err
		}
		if tag.RowsAffected() == 0 {
			return ErrAccountNotFound
		}
		return nil
	})
}
```

### 20.4 The handlers **[I/A]**

```go
// internal/api/accounts.go
package api

import (
	"errors"
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"
	"github.com/you/pgx-bank/internal/store"
)

type Handler struct{ store *store.Store }

func NewHandler(s *store.Store) *Handler { return &Handler{store: s} }

type createReq struct {
	Owner string `json:"owner" binding:"required"`
	Email string `json:"email" binding:"required,email"`
}

func (h *Handler) create(c *gin.Context) {
	var req createReq
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	// c.Request.Context() → the query is cancelled if the client disconnects (§15.3).
	acc, err := h.store.CreateAccount(c.Request.Context(), req.Owner, req.Email)
	switch {
	case errors.Is(err, store.ErrEmailTaken):
		c.JSON(http.StatusConflict, gin.H{"error": "email already registered"}) // 409
	case err != nil:
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"}) // 500
	default:
		c.Header("Location", "/accounts/"+strconv.FormatInt(acc.ID, 10))
		c.JSON(http.StatusCreated, acc) // 201 + Location
	}
}

func (h *Handler) get(c *gin.Context) {
	id, err := strconv.ParseInt(c.Param("id"), 10, 64)
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid id"})
		return
	}
	acc, err := h.store.GetAccount(c.Request.Context(), id)
	switch {
	case errors.Is(err, store.ErrAccountNotFound):
		c.JSON(http.StatusNotFound, gin.H{"error": "account not found"}) // 404
	case err != nil:
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
	default:
		c.JSON(http.StatusOK, acc)
	}
}

func (h *Handler) list(c *gin.Context) {
	accts, err := h.store.ListAccounts(c.Request.Context(), 100)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}
	c.JSON(http.StatusOK, accts)
}

type transferReq struct {
	From   int64 `json:"from"   binding:"required"`
	To     int64 `json:"to"     binding:"required"`
	Amount int64 `json:"amount" binding:"required,gt=0"`
}

func (h *Handler) transfer(c *gin.Context) {
	var req transferReq
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	err := h.store.Transfer(c.Request.Context(), req.From, req.To, req.Amount)
	switch {
	case errors.Is(err, store.ErrInsufficientFunds):
		c.JSON(http.StatusUnprocessableEntity, gin.H{"error": "insufficient funds"}) // 422
	case errors.Is(err, store.ErrAccountNotFound):
		c.JSON(http.StatusNotFound, gin.H{"error": "account not found"})
	case err != nil:
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
	default:
		c.JSON(http.StatusOK, gin.H{"status": "ok"})
	}
}
```

```go
// internal/api/router.go
package api

import "github.com/gin-gonic/gin"

func NewRouter(h *Handler) *gin.Engine {
	r := gin.New()
	r.Use(gin.Logger(), gin.Recovery())

	r.GET("/healthz", func(c *gin.Context) { c.JSON(200, gin.H{"status": "ok"}) })

	a := r.Group("/accounts")
	{
		a.POST("", h.create)
		a.GET("", h.list)
		a.GET("/:id", h.get)
	}
	r.POST("/transfers", h.transfer)
	return r
}
```

### 20.5 main.go — the wiring, with graceful shutdown **[I/A]**

`main` is nothing but wiring: load config → build a tuned pool → build the store → build the router → serve, and on `Ctrl-C` shut down cleanly so in-flight requests finish and the pool drains. Notice the pool tuning (§3.3), the ping (§2.3), and the shutdown dance are all things you've seen — assembled.

```go
// cmd/server/main.go
package main

import (
	"context"
	"errors"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/you/pgx-bank/internal/api"
	"github.com/you/pgx-bank/internal/config"
	"github.com/you/pgx-bank/internal/store"
)

func main() {
	cfg, err := config.Load() // loads .env (dev), validates required keys (§19.1)
	if err != nil {
		log.Fatal(err)
	}

	ctx := context.Background()

	// Build a tuned pool from the DSN (§3.3).
	poolCfg, err := pgxpool.ParseConfig(cfg.DatabaseURL)
	if err != nil {
		log.Fatalf("bad DATABASE_URL: %v", err)
	}
	poolCfg.MaxConns = cfg.MaxConns
	poolCfg.MaxConnLifetime = time.Hour
	poolCfg.MaxConnIdleTime = 30 * time.Minute

	pool, err := pgxpool.NewWithConfig(ctx, poolCfg)
	if err != nil {
		log.Fatalf("connect: %v", err)
	}
	defer pool.Close()

	// Prove the DB is reachable before we start serving (§2.3).
	pingCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()
	if err := pool.Ping(pingCtx); err != nil {
		log.Fatalf("database unreachable: %v", err)
	}

	// Wire the layers (§18.3): store over pool, handler over store, router over handler.
	st := store.New(pool)
	h := api.NewHandler(st)
	router := api.NewRouter(h)

	srv := &http.Server{
		Addr:              ":" + cfg.Port,
		Handler:           router,
		ReadHeaderTimeout: 5 * time.Second, // basic slow-loris protection
	}

	// Serve in a goroutine so main can wait for a shutdown signal.
	go func() {
		log.Printf("listening on :%s", cfg.Port)
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			log.Fatalf("server error: %v", err)
		}
	}()

	// Block until SIGINT/SIGTERM (Ctrl-C, or the orchestrator stopping us).
	stop := make(chan os.Signal, 1)
	signal.Notify(stop, syscall.SIGINT, syscall.SIGTERM)
	<-stop
	log.Println("shutting down...")

	// Give in-flight requests up to 10s to finish, THEN the deferred pool.Close()
	// returns all connections. Order matters: drain HTTP first, then the pool.
	shutdownCtx, cancel2 := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel2()
	if err := srv.Shutdown(shutdownCtx); err != nil {
		log.Printf("graceful shutdown failed: %v", err)
	}
	log.Println("bye")
}
```

### 20.6 Running it **[I]**

```bash
# 1) Apply the schema (or use goose — see the Goose guide).
psql "$DATABASE_URL" -f schema.sql

# 2) Dev: hot-reload with Air (reads the same .env via godotenv).
go tool air     # or: air

# 3) Try it:
curl -X POST localhost:8080/accounts -d '{"owner":"Ada","email":"ada@example.com"}'
# → 201 {"id":1,"owner":"Ada","email":"ada@example.com","balance":0,...}

curl -X POST localhost:8080/accounts -d '{"owner":"Bob","email":"bob@example.com"}'
# → 201 {"id":2,...}

# Re-POST Ada's email → 409 (the 23505 → ErrEmailTaken → Conflict path):
curl -X POST localhost:8080/accounts -d '{"owner":"X","email":"ada@example.com"}'
# → 409 {"error":"email already registered"}

# Transfer 50 from a zero-balance account → 422 (insufficient funds):
curl -X POST localhost:8080/transfers -d '{"from":1,"to":2,"amount":50}'
# → 422 {"error":"insufficient funds"}
```

Every response code traces to a concept in this guide: `201`+`Location` from the create handler, `409` from the `23505` mapping, `422` from the `RowsAffected()==0` guard inside the transaction, `404` from `pgx.ErrNoRows`. That's the whole point — the driver's behaviors become your API's contract.

---

## 21. Testing: pgxmock, Testcontainers, tx-rollback

There are two philosophies for testing pgx code, and mature projects use both: **mock the database** (fast, no Postgres, tests your Go logic against expected SQL) and **run against a real Postgres** (slower, tests that your SQL actually works). Mocks catch logic bugs; real-DB tests catch SQL bugs. You need both because each is blind to the other's failures.

### 21.1 Unit tests with pgxmock **[A]**

`github.com/pashagolub/pgxmock` is the pgx-native equivalent of the well-known `sqlmock`. It implements the pgx interfaces, lets you set expectations ("this SQL will run with these args, return these rows"), and verifies they were met — all with no database. Ideal for testing error-handling branches (like the `23505` → `ErrEmailTaken` mapping) that are painful to trigger against a real DB.

```go
func TestCreateAccount_DuplicateEmail(t *testing.T) {
	mock, err := pgxmock.NewPool()
	if err != nil {
		t.Fatal(err)
	}
	defer mock.Close()

	// Expect the INSERT, and simulate Postgres rejecting it with a 23505.
	mock.ExpectQuery("INSERT INTO accounts").
		WithArgs("Ada", "ada@example.com").
		WillReturnError(&pgconn.PgError{Code: "23505", ConstraintName: "accounts_email_key"})

	// pgxmock.PgxPoolIface has the same methods as *pgxpool.Pool, so a Store written
	// against an interface (not the concrete *pgxpool.Pool) can take the mock.
	s := store.New(mock)
	_, err = s.CreateAccount(context.Background(), "Ada", "ada@example.com")

	// Assert we mapped the DB error to the DOMAIN error the handler expects.
	if !errors.Is(err, store.ErrEmailTaken) {
		t.Fatalf("want ErrEmailTaken, got %v", err)
	}
	if err := mock.ExpectationsWereMet(); err != nil {
		t.Error(err) // fails if the expected SQL never ran
	}
}
```

> **Design implication:** to use pgxmock, your `Store` must depend on a pgx *interface* (e.g. the `DBTX` of §18.1, or pgxmock's `PgxPoolIface`), not the concrete `*pgxpool.Pool`. This is good design anyway — it's the seam that makes the data layer testable. The one caveat: a mock verifies *what SQL you sent*, not *whether Postgres accepts it*. A typo'd column name passes the mock and fails in production — which is why you also need §21.2.

### 21.2 Integration tests with Testcontainers **[A]**

`github.com/testcontainers/testcontainers-go` spins up a **real, throwaway PostgreSQL in Docker** for your test run, gives you its DSN, and tears it down after. Your tests run the actual SQL against the actual engine — catching the typo'd column, the wrong cast, the constraint you forgot. This is the gold standard for data-layer confidence.

```go
func TestAccounts_Integration(t *testing.T) {
	ctx := context.Background()

	// Start a real Postgres 18 container; it's destroyed when the test ends.
	pg, err := postgres.Run(ctx, "postgres:18-alpine",
		postgres.WithDatabase("bank"),
		postgres.WithUsername("test"),
		postgres.WithPassword("test"),
		testcontainers.WithWaitStrategy(
			wait.ForListeningPort("5432/tcp").WithStartupTimeout(30*time.Second)),
	)
	if err != nil {
		t.Fatal(err)
	}
	defer func() { _ = pg.Terminate(ctx) }()

	dsn, _ := pg.ConnectionString(ctx, "sslmode=disable")
	pool, _ := pgxpool.New(ctx, dsn)
	defer pool.Close()

	// Apply the schema, then exercise the REAL queries against the REAL engine.
	if _, err := pool.Exec(ctx, schemaDDL); err != nil {
		t.Fatal(err)
	}
	s := store.New(pool)
	acc, err := s.CreateAccount(ctx, "Ada", "ada@example.com")
	if err != nil || acc.ID == 0 {
		t.Fatalf("create failed: %v", err)
	}
	// A duplicate now triggers a REAL 23505 — proving the mapping end-to-end.
	if _, err := s.CreateAccount(ctx, "X", "ada@example.com"); !errors.Is(err, store.ErrEmailTaken) {
		t.Fatalf("want ErrEmailTaken from real DB, got %v", err)
	}
}
```

Testcontainers tests are slower (a container startup per suite) so you run fewer of them — typically one suite covering the critical paths — while pgxmock covers the many branch permutations cheaply.

### 21.3 The transaction-rollback test pattern **[A]**

A neat trick for fast, isolated real-DB tests without a container per test: run each test **inside a transaction you always roll back**. The test sees a real database, writes real rows, and the rollback wipes them — so tests don't pollute each other and you never clean up.

```go
func withRollback(t *testing.T, pool *pgxpool.Pool, fn func(tx pgx.Tx)) {
	tx, err := pool.Begin(context.Background())
	if err != nil {
		t.Fatal(err)
	}
	defer tx.Rollback(context.Background()) // ALWAYS roll back — test leaves no trace
	fn(tx) // the test runs its queries on tx (which satisfies DBTX)
}
```

Because your repository methods take a `DBTX` (§18.1), you pass the test transaction straight in, and every write vanishes at the end. This gives real-SQL fidelity with test isolation and no cleanup code — the best of both worlds for the bulk of your data-layer tests, with Testcontainers reserved for things that must cross transaction boundaries.

---

## 22. Performance and Scaling: Pool Math, PgBouncer, Replicas

### 22.1 Sizing the pool **[A]**

The instinct is "more connections = faster." It's wrong. Each PostgreSQL connection is a **server-side process** with its own memory; past a point, more connections mean more context-switching and lock contention, and throughput *drops*. The right `MaxConns` is usually **small** — often in the low tens, not hundreds.

A useful starting formula for a CPU-bound workload is `connections ≈ (core_count × 2) + effective_spindle_count`, but the honest answer is **measure**: watch `pool.Stat().EmptyAcquireCount()` (are callers waiting for a connection?) against Postgres's CPU and `pg_stat_activity`. If callers wait *and* Postgres has spare CPU, raise `MaxConns`; if Postgres is saturated, you need a *bigger database or a pooler*, not a bigger pool. And remember the **multiplier**: if you run 10 app instances each with `MaxConns=20`, that's 200 connections hitting Postgres — the per-instance number must be divided by how many instances you run.

### 22.2 PgBouncer and the mode you must set **[A]**

When many app instances would collectively open more connections than Postgres can handle, you put **PgBouncer** (a lightweight connection pooler) in front. Its `transaction` pooling mode is the scalable default — but, as §13.3 detailed, it breaks pgx's default prepared-statement caching. The rule, restated because it's the #1 production gotcha:

- **Direct connection or session-pooling PgBouncer** → keep `QueryExecModeCacheStatement` (the default).
- **Transaction-pooling PgBouncer** → set `cfg.ConnConfig.DefaultQueryExecMode = pgx.QueryExecModeExec`.

Miss this and you get intermittent `prepared statement does not exist` errors that only appear under production's pooler. It's the first thing to check when "it works locally, fails in prod."

### 22.3 Read replicas **[A]**

To scale reads, PostgreSQL replicates to read-only **replicas**. The simplest pgx pattern is **two pools** — one pointed at the primary (writes + read-after-write), one at the replica endpoint (heavy read queries) — and route each query to the right pool:

```go
type DB struct {
	Primary *pgxpool.Pool // writes and consistency-sensitive reads
	Replica *pgxpool.Pool // heavy/analytical reads that tolerate replica lag
}
```

The catch is **replication lag**: a replica is milliseconds-to-seconds behind, so a read immediately after a write may not see it. Route *read-after-write* and consistency-critical reads to the primary; send only lag-tolerant reads (reports, lists, search) to the replica. There's no free lunch — you're trading read throughput for eventual consistency, and you must decide per query which you can tolerate.

### 22.4 Fast paths, recapped **[A]**

The performance tools from earlier sections, as a decision list: **batch** (§11) when you have many separate statements and latency dominates; **`CopyFrom`** (§12) for bulk homogeneous inserts; **prepared statements** (§13, automatic) for repeated queries; **`= ANY($1)`** (§5.5) instead of N-placeholder `IN` lists; and always **`SELECT` only the columns you need** — `SELECT *` ships bytes you'll discard and defeats covering indexes. Profile with `EXPLAIN (ANALYZE, BUFFERS)` (see the [PostgreSQL guide](POSTGRESQL_GUIDE.md)) before optimizing — the bottleneck is usually a missing index, not the driver.

### 22.5 A high-traffic pooling playbook **[A]**

§22.1–§22.4 gave you the knobs and the rules in isolation. This is the consolidated, do-this-in-order playbook for pooling a service under real load — the saturation model, the sizing math, the connection budget, PgBouncer, warmup, and the alerts — because at high traffic the pool is the part that fails first, and it fails in ways that look like "the database is slow" when it isn't.

#### 22.5.1 The saturation model — what a full pool actually does **[A]**

You cannot tune a pool without knowing its behavior at the limit. `pgxpool` has a fixed `MaxConns`. When **every** connection is checked out and another goroutine issues a query, that goroutine **blocks inside the pool's internal `Acquire`** until either a connection frees up **or its context deadline fires**. There is no separate "acquire timeout" setting — **the acquire wait is bounded by the `ctx` you pass, and nothing else.** This single fact drives every high-traffic decision:

```go
func (s *Store) GetAccount(ctx context.Context, id int64) (Account, error) {
	// Under load, if all MaxConns are checked out, THIS call blocks inside the
	// pool waiting for a free connection — up to ctx's deadline, then it returns
	// context.DeadlineExceeded. There is no acquire-timeout knob; the deadline on
	// ctx IS the acquire timeout. So: give every request a bounded context (§15),
	// and a saturated pool sheds load cleanly instead of hanging goroutines forever.
	rows, err := s.pool.Query(ctx, "SELECT id, owner, email, balance, created_at::text FROM accounts WHERE id = $1", id)
	if err != nil {
		return Account{}, err
	}
	acc, err := pgx.CollectOneRow(rows, pgx.RowToStructByName[Account])
	if errors.Is(err, pgx.ErrNoRows) {
		return Account{}, ErrAccountNotFound
	}
	return acc, err
}
```

This behavior is a **feature**: the pool is a bounded queue in front of Postgres. When traffic exceeds capacity, requests queue for a connection and then shed (with `DeadlineExceeded`) once they wait past their deadline — which is far healthier than letting unbounded concurrent queries pile onto the database and collapse it. Two obligations follow: **(1)** every request must carry a deadline-bearing context (the request context in a Gin handler, §15.3), so overload turns into fast 503s instead of hung goroutines; **(2)** the "requests are waiting for a connection" signal is `pool.Stat().EmptyAcquireCount()` climbing and `AcquireDuration()` rising — those are your saturation alarms (§22.5.5).

#### 22.5.2 Sizing with Little's Law — the number is small **[A]**

The instinct to set `MaxConns = 200` for a busy service is almost always wrong. The correct size comes from **Little's Law**: the average number of in-flight queries is the arrival rate times the service time.

> **L = λ × W** — where **L** = concurrent queries (≈ the connections you need), **λ** = queries per second, **W** = average query duration in seconds.

Worked example. Your service does **2,000 queries/sec** and the average query takes **5 ms**:

```text
L = λ × W = 2000 /s × 0.005 s = 10 concurrent queries
```

Ten. Not two hundred. A `MaxConns` of **16** (10 plus ~60% burst headroom) serves that load with room to spare. The counterintuitive lesson: **connection count is driven by query *speed*, not request *volume*.** If a pool feels too small, the first fix is usually **make the queries faster** (an index, fewer round-trips) — halving W halves the connections you need — not adding connections, because each added connection is a Postgres backend process competing for the same CPU and locks. Past the CPU-core count, more connections make throughput *worse* (§22.1). Size for the **average**; let the pool's queue (§22.5.1) absorb bursts, and only raise `MaxConns` when `EmptyAcquireCount` climbs *while Postgres still has spare CPU*.

#### 22.5.3 The connection budget — allocate top-down **[A]**

Postgres enforces a hard **`max_connections`** (default 100; commonly raised to 200–500). Every connection is a backend **process** costing ~5–10 MB of server RAM, so you cannot simply crank it up. Budget from the server down, not from each app up:

| Layer | Quantity | Note |
|---|---|---|
| Postgres `max_connections` | e.g. 200 | Hard ceiling. Raising it costs RAM and doesn't fix contention. |
| − reserved | ~15 | `superuser_reserved_connections`, replication slots, monitoring, `psql`, migrations. |
| = usable by apps | ~185 | Everything your services share. |
| ÷ app instances | 10 instances | **The multiplier trap:** per-instance `MaxConns` × instances must fit here. |
| → per-instance `MaxConns` | **≤ 18** | 10 × 18 = 180 < 185. ✅ 10 × 20 = 200 > 185 → connection-refused storms under load. ❌ |

The failure this prevents is brutal and common: ten instances each configured `MaxConns = 20` quietly agree to open 200 backends against a 200-limit server, leaving zero headroom for `psql`, a migration, or an autoscale event — and the next connection attempt gets `FATAL: sorry, too many clients already`, cascading into failed deploys and health checks. **Per-instance `MaxConns` is a fraction of the server budget, and you must divide by the instance count** (including the extra instances that exist momentarily during a rolling deploy). When that math forces per-instance `MaxConns` uncomfortably low, that is precisely the signal to add PgBouncer.

#### 22.5.4 PgBouncer for real scale — decouple client count from server count **[A]**

When app concurrency (instances × `MaxConns`) genuinely needs to exceed what Postgres can hold — thousands of clients, serverless functions, many services — put **PgBouncer** in transaction-pooling mode between them. PgBouncer multiplexes a large number of cheap client connections onto a **small** set of real Postgres connections, handed out per transaction:

```ini
; pgbouncer.ini — transaction pooling: the scalable default.
[pgbouncer]
pool_mode = transaction
; PgBouncer holds only THIS many real Postgres connections per database, no matter
; how many app clients connect. THIS is the number that must fit max_connections.
default_pool_size = 20
; Room for many cheap client-side connections fanning into that small server pool.
max_client_conn = 5000
```

The budget now inverts: your app pools connect to **PgBouncer** (connecting there is cheap, so per-instance `MaxConns` can be generous), while **Postgres only ever sees `default_pool_size` connections** per database. You've decoupled "how many clients" from "how many backends." Two rules make it work with pgx:

1. **Set `pgx.QueryExecModeExec`** (§13.3). Transaction pooling breaks per-connection prepared statements — this is the #1 "works locally, `prepared statement does not exist` in prod" bug, and it exists *because* prod added the pooler.
2. **Don't use session-level features across transactions** — `LISTEN/NOTIFY` (§14), session `SET`, advisory locks, and `WITH HOLD` cursors need session pooling or a dedicated direct connection, since transaction pooling can move you to a different backend between transactions.

Rule of thumb: **direct-connect until the connection budget (§22.5.3) forces your hand, then PgBouncer transaction mode + `QueryExecModeExec`.**

#### 22.5.5 Warmup, connection storms, and monitoring **[A]**

Two operational hazards remain, both around *change*:

**Cold-start connection storms.** A freshly deployed instance starts with an empty pool. The first burst of traffic makes it open many connections *at once* — and during a rolling deploy, every new instance does this simultaneously, slamming Postgres (or a just-restarted database that's still warming its caches) with a synchronized connection storm. Mitigate with `MinConns` (pre-warm a floor so the first requests don't each pay a connect+TLS handshake and the ramp isn't from zero) and `MaxConnLifetimeJitter` (so the batch of connections opened together doesn't later *expire* together and reconnect in lockstep):

```go
func newHighTrafficPool(ctx context.Context, dsn string) (*pgxpool.Pool, error) {
	cfg, err := pgxpool.ParseConfig(dsn)
	if err != nil {
		return nil, err
	}
	cfg.MaxConns = 16              // from Little's Law (§22.5.2), not guessed high
	cfg.MinConns = 4               // pre-warm a floor: no from-zero storm, no cold first-request latency
	cfg.MaxConnLifetime = 30 * time.Minute
	cfg.MaxConnLifetimeJitter = 5 * time.Minute // desynchronize expiry so reconnects don't thunder
	cfg.MaxConnIdleTime = 15 * time.Minute
	cfg.HealthCheckPeriod = 30 * time.Second    // the sweep that retires stale conns & maintains MinConns
	cfg.ConnConfig.DefaultQueryExecMode = pgx.QueryExecModeExec // if behind a txn-pooling PgBouncer
	return pgxpool.NewWithConfig(ctx, cfg)
}
```

> **⚡ Note on `MinConns`:** the pool maintains the `MinConns` floor lazily, via the `HealthCheckPeriod` sweep, rather than opening them all instantly at `New()`. So `MinConns` smooths the ramp and keeps a warm floor during quiet periods; it is not a hard "N connections exist the instant startup returns" guarantee. For latency-critical services that must be warm before taking traffic, issue a few throwaway `Ping`/`SELECT 1` calls at startup to force connections open before you flip the readiness probe.

**Monitoring — the numbers that predict an outage.** Export `pool.Stat()` to Prometheus and alert on it; these move *before* users feel pain:

```go
// Sample pool.Stat() periodically into Prometheus gauges. (A custom
// prometheus.Collector works too; a sampler is simpler and perfectly adequate.)
func exportPoolMetrics(ctx context.Context, pool *pgxpool.Pool, reg prometheus.Registerer) {
	acquired := prometheus.NewGauge(prometheus.GaugeOpts{Name: "pgx_pool_acquired_conns"})
	idle := prometheus.NewGauge(prometheus.GaugeOpts{Name: "pgx_pool_idle_conns"})
	total := prometheus.NewGauge(prometheus.GaugeOpts{Name: "pgx_pool_total_conns"})
	maxConns := prometheus.NewGauge(prometheus.GaugeOpts{Name: "pgx_pool_max_conns"})
	emptyAcq := prometheus.NewGauge(prometheus.GaugeOpts{Name: "pgx_pool_empty_acquire_total"})
	canceled := prometheus.NewGauge(prometheus.GaugeOpts{Name: "pgx_pool_canceled_acquire_total"})
	reg.MustRegister(acquired, idle, total, maxConns, emptyAcq, canceled)

	go func() {
		ticker := time.NewTicker(15 * time.Second)
		defer ticker.Stop()
		for {
			select {
			case <-ctx.Done():
				return
			case <-ticker.C:
				s := pool.Stat()
				acquired.Set(float64(s.AcquiredConns()))       // in use right now
				idle.Set(float64(s.IdleConns()))               // free and ready
				total.Set(float64(s.TotalConns()))             // open connections
				maxConns.Set(float64(s.MaxConns()))            // the ceiling
				emptyAcq.Set(float64(s.EmptyAcquireCount()))   // times a caller WAITED for a conn
				canceled.Set(float64(s.CanceledAcquireCount())) // acquires abandoned (ctx expired waiting)
			}
		}
	}()
}
```

The alert set that maps to the failure modes above:

| Signal | Condition | What it means → action |
|---|---|---|
| `AcquiredConns / MaxConns` | sustained > ~0.8 | Pool near saturation → optimize queries (§22.5.2) or scale out; raise `MaxConns` only if the connection budget (§22.5.3) allows. |
| rate of `EmptyAcquireCount` | > 0 and rising | Callers are **waiting** for connections → the pool is the bottleneck (too small, or queries too slow). |
| rate of `CanceledAcquireCount` | > 0 | Requests are **shedding** — waiting past their deadline and giving up. Active user-facing pain. |
| `AcquireDuration` (from `Stat`) | p99 climbing | Contention for connections is growing before it's an outage. |

`EmptyAcquireCount` rising is the leading indicator; `CanceledAcquireCount` rising is the lagging one that means users are already getting errors. Watch the first, and you fix the pool before you ever see the second.

---

## 23. Banking-Grade Security Checklist

Everything here has appeared in context above; this is the consolidated checklist to audit a pgx service against before it touches real money or PII.

### 23.1 Injection and query safety **[A]**

- **Always parameterize** with `$1`/`NamedArgs`. **Never** concatenate or `Sprintf` user input into SQL (§5.1). This is non-negotiable and covers the overwhelming majority of real-world database breaches.
- **Dynamic identifiers via allow-list only** (§5.4) — user input selects *which* of your pre-written SQL fragments to use, never supplies SQL text.
- **`pgx.Identifier{...}.Sanitize()`** for any identifier you must inject (table names in `CopyFrom`, channels in `LISTEN`).

### 23.2 Connection and transport **[A]**

- **`sslmode=verify-full`** in production — encrypt *and* verify the server certificate + hostname (§3.2). `require` alone doesn't stop MITM; `disable` is local-only.
- **Least-privilege database role** — the app connects as a role that can `SELECT/INSERT/UPDATE/DELETE` its tables and nothing more. Not a superuser, no `DROP`, no access to other schemas. A compromised app key should be able to do only what the app does.
- **Secrets from the environment / a secret manager**, never in code or the repo. `.env` is dev-only and git-ignored (§2.2); prod injects real env vars.

### 23.3 Resource and DoS limits **[A]**

- **`statement_timeout`** (server-side, via `AfterConnect`, §3.4) so no single query runs unbounded — the ceiling that stops a runaway or malicious query from pinning a backend.
- **`idle_in_transaction_session_timeout`** so a client that `BEGIN`s and hangs can't hold locks forever.
- **Per-query context timeouts** (client-side, §15.2) as defense in depth, using the request context so client disconnects cancel queries.
- **Bounded `MaxConns`** (§22.1) so the app can't open more connections than Postgres can serve — an app-level connection flood is a self-inflicted DoS.

### 23.4 Data integrity **[A]**

- **Money as `numeric` or integer minor units, never `float`** (§8.3).
- **Enforce invariants in the database** — `UNIQUE`, `CHECK (balance >= 0)`, `NOT NULL`, foreign keys — so no application bug can write invalid state. The `CHECK` in §20.2 is a backstop the code cannot bypass.
- **Transactions for multi-step invariants** (§10), with `Serializable` + retry (§10.5) for the strictest money-movement guarantees.
- **Check `RowsAffected()`** on conditional writes (§4.1) — a zero-row `UPDATE` is often a failed precondition, not a success.

### 23.5 Observability and leakage **[A]**

- **Never log bound parameters or the DSN in production** (§16.2) — they carry PII/secrets. Log SQLSTATE, constraint name, and a non-sensitive operation label instead.
- **Map database errors to safe messages** — return "email already registered," not the raw `*pgconn.PgError` with schema/constraint internals that leak your data model to attackers.
- **Monitor `pool.Stat()`** — `EmptyAcquireCount`/`AcquiredConns` climbing toward `MaxConns` is early warning of a leak or undersized pool (§16.3).

---

## 24. Gotchas and Best Practices

| # | Gotcha | Do this instead |
|---|---|---|
| 1 | Sharing one `*pgx.Conn` across goroutines | Use a `*pgxpool.Pool` for anything concurrent; a bare `Conn` is not concurrency-safe (§3.1). |
| 2 | Opening a pool per request | Create **one** pool at startup, share it everywhere (§3.1). |
| 3 | Forgetting `rows.Close()` / `Acquire`→`Release` / `SendBatch`→`Close` | `defer` the close immediately; a leak permanently shrinks the pool until it deadlocks (§3.5, §4.3). |
| 4 | Skipping `rows.Err()` after the loop | `Next()` returning false is *ambiguous* between "done" and "error" — always check `rows.Err()` (§4.3). Or use `CollectRows`, which does it for you. |
| 5 | String-concatenating SQL | Always `$1` parameters. No exceptions (§5.1). |
| 6 | Trying to parameterize a column name / `ASC`/`DESC` | Parameters are for *values*; use an allow-list for identifiers (§5.2, §5.4). |
| 7 | `IN (?, ?, ?)` with a dynamic count | `= ANY($1)` with a slice (§5.5). |
| 8 | Money in `float4`/`float8` | `numeric` (`pgtype.Numeric`) or integer minor units (§8.3). |
| 9 | `timestamp` instead of `timestamptz` | Always `timestamptz`; store instants, convert for display (§8.4). |
| 10 | Not distinguishing `pgx.ErrNoRows` from real errors | Branch: ErrNoRows → 404, other → 500 (§4.2, §9.2). |
| 11 | Treating `23505` as a 500 | Map SQLSTATE to domain errors → 409/400/422 (§9.3). |
| 12 | Running statements on `pool` inside a transaction | Every statement in a txn goes through `tx`, never `pool` (§10.2). |
| 13 | Forgetting to retry `40001`/`40P01` under serializable isolation | Bounded retry loop around the whole transaction (§10.5). |
| 14 | `prepared statement does not exist` behind PgBouncer | Set `QueryExecModeExec` for transaction-pooling PgBouncer (§13.3). |
| 15 | Sharing a `LISTEN` connection with query traffic | Dedicate an acquired connection to the listener (§14.2). |
| 16 | `context.Background()` in a Gin handler | Use `c.Request.Context()` so client disconnect cancels the query (§15.3). |
| 17 | Missing `defer cancel()` on `WithTimeout` | Always `defer cancel()` — it leaks otherwise (§15.2). |
| 18 | Logging query args in production | `LogLevelError` only; never log params/DSN (§16.2). |
| 19 | Depending on the concrete `*pgxpool.Pool` everywhere | Depend on a `DBTX`/pgx interface so code is transaction-composable and testable (§18.1, §21.1). |
| 20 | `CopyFrom` with `ON CONFLICT` expectations | `CopyFrom` can't upsert; stage into a temp table then `INSERT ... ON CONFLICT` (§12.4). |

**Best-practice summary:** one pool, shared; parameters always; `defer` every close/release; branch `ErrNoRows` and SQLSTATE explicitly; transactions via `BeginFunc`; the request context all the way down; money exact; secrets from the environment; and depend on interfaces so you can test. Get these right and pgx is both the fastest and the safest way to talk to Postgres from Go.

---

## 25. Study Path and Build-to-Learn Projects

### 25.1 A staged path through this guide **[B→A]**

1. **Connect and query (B).** §2–§4: get the six-step spine working — load env, pool, ping, `QueryRow`/`Query`/`Exec`. Build: a CLI that connects and prints one table.
2. **Safe parameters and struct mapping (B/I).** §5–§6: never concatenate; make `CollectRows` + `RowToStructByName` your default read path. Build: a `SELECT` that returns `[]Struct`.
3. **NULLs and types (I).** §7–§8: handle a nullable column three ways; store money as `numeric`/cents and a timestamp as `timestamptz`; round-trip a `jsonb` column.
4. **Errors and transactions (I).** §9–§10: map `23505` to a 409; write the atomic transfer with `BeginFunc` and a `balance >= amount` guard.
5. **Throughput (I/A).** §11–§13: batch a fan-out write; `CopyFrom` a bulk load; understand the statement cache and the PgBouncer mode.
6. **Realtime and ops (A).** §14–§16: a `LISTEN/NOTIFY` cache-invalidator; request-scoped timeouts; query logging + `pool.Stat()` metrics.
7. **Structure, test, harden (A).** §17–§23: the `DBTX`-based store, the Gin app, pgxmock + Testcontainers, the security checklist.

### 25.2 Build-to-learn projects **[I→A]**

- **① The accounts API (this guide's §20), extended.** Add pagination (keyset, not `OFFSET`), a `GET /accounts/:id/transactions` history, and a `DELETE` with a soft-delete `deleted_at timestamptz`. Covers §4–§10, §18–§20.
- **② A bulk importer.** Read a large CSV, validate rows, and load them with `CopyFrom` into a staging temp table, then `INSERT ... ON CONFLICT` into the real table. Benchmark it against row-by-row `INSERT` and batched inserts to feel the difference. Covers §11–§12, §19.
- **③ A realtime price ticker.** One process writes price updates and `pg_notify`s a channel inside the same transaction; another process `LISTEN`s and pushes to connected clients. Add reconnect-and-refresh. Covers §10, §14 — and pairs perfectly with the [Coder WebSocket guide](GO_CODER_WEBSOCKETS_GUIDE.md) for the client side.
- **④ A "does it survive prod?" harness.** Put PgBouncer in transaction mode in front of your accounts API, reproduce the `prepared statement does not exist` error, then fix it with `QueryExecModeExec`. Add a `Serializable` transfer under concurrent load and implement the `40001` retry loop. Covers §13, §10.5, §22 — the lessons that only bite in production.
- **⑤ Graduate to the ecosystem.** Re-implement the accounts store with [sqlc + goose](GO_SQLC_GOOSE_GUIDE.md) and again with [Ent](GO_ENT_ORM_GUIDE.md), keeping the same handlers — and notice that both run on the very `pgxpool.Pool` this guide taught you to build. That realization — *the ORM is pgx underneath* — is the point where you've truly understood the layer.

### 25.3 Where to go next **[reference]**

- **The schema layer:** [Goose — DB migrations](GO_GOOSE_MIGRATIONS_GUIDE.md) — own the DDL pgx queries against; the `stdlib.OpenDBFromPool` bridge (§17) is how goose and your pgx pool share connections.
- **Type-safe queries on pgx:** [sqlc + goose](GO_SQLC_GOOSE_GUIDE.md) — write SQL, get compile-checked Go that calls the pgx pool directly.
- **The ORM on pgx:** [Go ent ORM](GO_ENT_ORM_GUIDE.md) — schema-as-code and a type-safe builder when you want the graph handled for you.
- **The engine:** [PostgreSQL](POSTGRESQL_GUIDE.md) — the types, `EXPLAIN ANALYZE`, indexing, MVCC, and `LISTEN/NOTIFY` semantics pgx exposes.
- **The app around it:** [Go Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) and [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md) for the HTTP and auth layers, and the [Real-Time Chat Capstone](GO_NEXTJS_REALTIME_CHAT_CAPSTONE_GUIDE.md) to see pgx-under-Ent, goose, and coder/websocket composed into one production system.

You now know the metal. Everything else in the Go data stack is a convenience built on top of it — and when a convenience can't express what you need, you know exactly how to drop down to the pool and write the SQL yourself.
