# Go ent ORM — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I have never used an ORM in Go" to "I can design a real ent schema, generate a fully type-safe data layer, write complex graph queries, manage versioned migrations, and reason about the security and performance trade-offs of every decision" — entirely offline. Every concept is explained in **prose first** (what it is, *why* it works that way, when and how to use it, the key parameters, best practices, and security/injection notes), and only **then** demonstrated with heavily-commented, runnable code. Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **ent v0.14+** with **Go 1.23 / 1.24** (current in 2026). ent is a stable, production-grade framework originally built at Meta (Facebook) and now maintained by Ariga (the Atlas team) and the open-source community. The core ideas — *schema-as-Go-code*, *code generation*, *type-safe query builders* — have been stable for years; where APIs differ across minor versions, sections are flagged with **⚡ Version note**. Always cross-check **entgo.io** for the very latest. The author is on **Windows 11**, so cross-platform notes (path separators, shells) are called out where relevant.
>
> **This guide's place in the library:** ent is the ORM layer that sits *on top of* a SQL database. For the database itself, read **`POSTGRESQL_GUIDE.md`** (the engine ent talks to most often in production) and **`RELATIONAL_DB_DESIGN_GUIDE.md`** (normalization, keys, indexes — the theory ent's schema-as-code encodes). For the language ent is written in and that you write schemas in, read **`GO_GUIDE.md`** (the pure-language reference: structs, methods, interfaces, generics, error handling, context). This guide assumes you can read Go and understand a foreign key; it teaches ent. We deliberately avoid re-teaching SQL or Go.

---

## Table of Contents

1. [What ent Is — and When to Choose It](#1-what-ent-is--and-when-to-choose-it) **[B]**
2. [Setup, Installation & the Codegen Workflow](#2-setup-installation--the-codegen-workflow) **[B]**
3. [Defining Schemas: Fields](#3-defining-schemas-fields) **[B/I]**
4. [Edges (Relations): the Graph Model](#4-edges-relations-the-graph-model) **[B/I]**
5. [Indexes & Mixins](#5-indexes--mixins) **[I]**
6. [Code Generation: What Gets Built & How the Type-Safe Builder Works](#6-code-generation-what-gets-built--how-the-type-safe-builder-works) **[I]**
7. [The Client & Connecting to a Database](#7-the-client--connecting-to-a-database) **[B/I]**
8. [CRUD Operations in Depth](#8-crud-operations-in-depth) **[B/I]**
9. [Predicates, Filtering, Ordering & Pagination](#9-predicates-filtering-ordering--pagination) **[I]**
10. [Eager Loading & Edge Traversal — Defeating N+1](#10-eager-loading--edge-traversal--defeating-n1) **[I]**
11. [Aggregations & Grouping](#11-aggregations--grouping) **[I/A]**
12. [Transactions](#12-transactions) **[I/A]**
13. [Hooks & Interceptors](#13-hooks--interceptors) **[A]**
14. [Schema Migrations: Auto vs Versioned (Atlas)](#14-schema-migrations-auto-vs-versioned-atlas) **[I/A]**
15. [The Privacy / Policy Layer](#15-the-privacy--policy-layer) **[A]**
16. [Full Worked Example: a Blog (User, Post, Comment)](#16-full-worked-example-a-blog-user-post-comment) **[I]**
17. [Best Practices & Security](#17-best-practices--security) **[I/A]**
18. [Tips, Tricks & Gotchas](#18-tips-tricks--gotchas) **[A]**
19. [Study Path & Build-to-Learn Projects](#19-study-path--build-to-learn-projects)

---

## 1. What ent Is — and When to Choose It

### 1.1 The core idea in one sentence **[B]**

**ent** (pronounced "entity") is a Go entity framework / ORM whose defining idea is: **you describe your data model as Go code, run a code generator, and get a complete, fully type-safe database access layer in return.** You never hand-write the boilerplate that maps rows to structs, you never type a column name as a string in your query code, and you never pass `interface{}` into a query and hope the runtime accepts it. If a query compiles, it is structurally valid against your schema.

That is the whole pitch, and everything else in this guide is a consequence of it. To understand *why* ent is shaped the way it is, you have to understand the problem it solves.

### 1.2 The problem ORMs solve, and the problem ent solves **[B]**

A relational database speaks SQL and stores rows in tables. A Go program speaks structs, slices, and methods. An **Object-Relational Mapper (ORM)** is the translation layer between those two worlds: it turns a row into a struct, a struct into an `INSERT`, and a chain of method calls into a `SELECT ... WHERE ...`. Without an ORM you write that translation by hand for every table — tedious, repetitive, and a fertile source of bugs (forgotten columns, wrong scan order, SQL injection from string concatenation).

Most Go ORMs (notably **GORM**) do this mapping at **runtime using reflection**. You define a struct with tags, and at runtime GORM inspects the struct, figures out the columns, and builds SQL. This is flexible but has a cost: many mistakes — a misspelled column in a `Where("naem = ?")`, a type mismatch, a relation that doesn't exist — are invisible to the compiler and only blow up when that code path runs in production.

ent takes a fundamentally different route: **it moves all of that work to compile time via code generation.** Instead of reflecting over your structs at runtime, ent reads your schema definitions *before* your program runs, generates concrete Go code (real structs, real methods, real predicate functions named after your actual fields), and you compile against *that*. The result is that the column names, the field types, the relation directions, and the validators all become part of the type system. A typo in a field name is a compile error, not a 2 a.m. page.

### 1.3 The three pillars **[B]**

| Pillar | What it means | Why it matters |
|---|---|---|
| **Schema-as-Go-code** | Entities, fields, indexes, and relations live in plain `.go` files under `ent/schema/`. No YAML, no JSON, no separate DSL, no SQL DDL. | Your schema lives next to your code, is reviewed in the same PR, refactors with your IDE, and is checked by `go vet`. |
| **Code generation** | Running `go generate ./ent` reads those schema files and produces a full, type-safe client: builders, predicates, query/mutation methods, eager-loaders. | The boilerplate is written for you, correctly, every time, and regenerated whenever the schema changes. |
| **Type-safe queries** | No `interface{}`, no string column names, no string-built WHERE clauses in your application code. The generated API is statically typed end to end. | The compiler is your first line of defense against both bugs and SQL injection. |

### 1.4 Why "schema as a graph" — ent's mental model **[B]**

ent does not think of your data as a set of disconnected tables; it thinks of it as a **graph**. Entities are **nodes**; relations are **edges** between nodes. This is not just marketing — it shapes the API. Where a SQL-centric ORM makes you write `JOIN`s, ent gives you *traversal*: from a `User` node you call `.QueryPosts()` to walk the edge to its `Post` nodes, then `.QueryComments()` to walk further. The graph model is why ent's relation API (covered in §4 and §10) reads like navigation rather than like SQL. If you have ever used a graph database conceptually, this will feel natural; if you come from pure SQL, internalizing "edges, not joins" is the single biggest mindset shift.

### 1.5 ent vs GORM vs sqlc vs pgx — choosing the right tool **[I]**

This is the most important decision in the chapter, so we will be concrete. All four are legitimate, widely used choices for Go database access in 2026. They sit on a spectrum from "hand-write SQL" to "never write SQL."

| Tool | What it is | Type safety | Who writes the SQL | Best when |
|---|---|---|---|---|
| **pgx** | A high-performance PostgreSQL driver + toolkit. You write SQL strings and scan results. | Low — query strings are opaque to the compiler | You, by hand | You want maximum control and performance, are PostgreSQL-only, and are comfortable with SQL. |
| **sqlc** | A code generator: you write `.sql` files, sqlc generates type-safe Go functions from them. | High — generated from your SQL | You, by hand (then codegen wraps it) | You love SQL and want type-safe Go wrappers without an ORM's abstraction. |
| **GORM** | A reflection-based, runtime ORM with a chainable API and struct tags. | Low/medium — much resolved at runtime | GORM generates it from method chains | You want the most popular, batteries-included ORM and accept runtime-resolved queries. |
| **ent** | A codegen-based ORM: schema-as-code, generated type-safe builders, graph traversal, hooks, privacy. | **Highest** — the whole API is generated and statically typed | ent generates it from your typed builder calls | You want compile-time-safe queries, rich relations/graph traversal, and built-in hooks/privacy/migrations. |

**The honest trade-offs of choosing ent:**

- **Choose ent when:** your data model has meaningful relationships (users → posts → comments, multi-tenant graphs, social graphs); you want the compiler to catch query mistakes; you value built-in cross-cutting features (hooks, interceptors, a privacy layer, Atlas-backed versioned migrations); your team will benefit from a single source of truth in Go.
- **Be cautious with ent when:** you need a very exotic query that is awkward to express in any ORM (ent does have raw-SQL escape hatches via `sql.Modifier` and the underlying `sql` builder, but if 80% of your queries are hand-tuned SQL, sqlc or pgx is a better fit); your team is allergic to code generation in the build; or your "schema" is a single flat table with no relations (then a full ORM is overkill).
- **The codegen cost is real but small:** `go generate ./ent` adds a step to your workflow, the generated `ent/` directory is large (dozens of files), and you must regenerate after every schema change. In exchange you never write the data-access boilerplate and you get end-to-end type safety. Most teams consider this a clear win for non-trivial models.

> **Security framing up front:** ent's code generation is also its **primary security advantage**. Because your application code never concatenates user input into a query string — it calls generated predicate functions like `user.EmailEQ(input)` that always parameterize the value — the most common database vulnerability, **SQL injection, is structurally prevented** for the entire generated API. We return to this in §17, but keep it in mind from the start: in idiomatic ent, you cannot accidentally write an injectable query because there is no place to type a SQL string.

### 1.6 Supported databases **[B]**

| Database | Status | Notes |
|---|---|---|
| PostgreSQL | Full support | The recommended production target; pairs with `POSTGRESQL_GUIDE.md`. Native `uuid`, `jsonb`, arrays. |
| MySQL / MariaDB | Full support | Mature; ENUM via `SchemaType`. |
| SQLite | Full support | Ideal for local dev and tests; remember foreign keys are off by default (see §7). |
| CockroachDB | Full support | Distributed Postgres-wire-compatible. |
| TiDB | Full support | Distributed MySQL-wire-compatible. |
| Gremlin (AWS Neptune) | Experimental | Graph backend; far less commonly used than the SQL dialects. |

> **⚡ Version note:** CockroachDB and TiDB support matured through v0.11–v0.14. For new projects in 2026, PostgreSQL is the default recommendation unless you have a specific reason otherwise.

---

## 2. Setup, Installation & the Codegen Workflow

### 2.1 The mental model of the workflow **[B]**

Before any commands, understand the **loop** you will repeat for the rest of your ent life, because every other section assumes it:

1. **You write or edit a schema file** under `ent/schema/` (plain Go).
2. **You run `go generate ./ent`** — this invokes ent's code generator, which reads your schema files and (re)writes the generated client under `ent/`.
3. **You write application code** against the freshly generated, type-safe API.
4. **You compile and run.** If you changed a field name in step 1 but forgot step 2, your app code won't compile against the stale generated code — which is a *feature*: the toolchain forces the schema and the code to stay in sync.

The single most common beginner mistake is editing a schema and forgetting to regenerate. Burn step 2 into muscle memory.

### 2.2 Prerequisites **[B]**

```bash
# Go 1.23+ recommended (1.24 ideal). Verify:
go version          # e.g. go version go1.24.0 windows/amd64
```

ent has no separate "install" beyond a Go module dependency — the code generator is itself a Go program you run via `go run`, so there is nothing global to install and no version-skew between a globally installed CLI and your project.

### 2.3 Create a module and add ent **[B]**

```bash
mkdir myblog && cd myblog
go mod init github.com/you/myblog   # creates go.mod; module path is up to you

# Add ent as a dependency
go get entgo.io/ent@latest
```

### 2.4 Scaffold your first schema **[B]**

ent ships a small CLI (`entgo.io/ent/cmd/ent`) that scaffolds schema skeletons so you don't write the boilerplate struct by hand. You run it via `go run` so it always matches your module's ent version:

```bash
# Scaffold a single entity named "User"
go run -mod=mod entgo.io/ent/cmd/ent new User
```

This creates:

```
ent/
├── generate.go          # contains the //go:generate directive (the codegen trigger)
└── schema/
    └── user.go          # your User schema skeleton — YOU edit this
```

Scaffold several entities at once:

```bash
go run -mod=mod entgo.io/ent/cmd/ent new Post Comment Tag
```

> **Why `-mod=mod`?** It tells the Go toolchain it may update `go.mod`/`go.sum` if needed to run the command. In a tidy module it is harmless; without it, a missing tool dependency could fail the command.

### 2.5 The `generate.go` file — the codegen trigger **[B]**

The scaffold writes a tiny file whose only job is to carry a `//go:generate` directive. When you run `go generate ./ent`, Go scans for these directives and executes them.

```go
// ent/generate.go
package ent

//go:generate go run -mod=mod entgo.io/ent/cmd/ent generate ./schema
```

Read that directive literally: "run the ent generator, pointing it at the `./schema` directory." `go generate` is not magic — it just runs the command in the comment. This is the standard Go convention for codegen, the same mechanism `stringer` and `mockgen` use (see `GO_GUIDE.md` on tooling).

### 2.6 Project layout after generation **[B]**

After you fill in a schema and run `go generate ./ent`, the tree looks like this. The crucial thing to internalize: **you own `ent/schema/`; everything else under `ent/` is generated and must never be hand-edited** (your edits would be silently overwritten on the next generate).

```
myblog/
├── go.mod
├── go.sum
├── main.go
└── ent/
    ├── generate.go          # the //go:generate directive
    ├── schema/              # ◄── YOU write these
    │   ├── user.go
    │   ├── post.go
    │   └── comment.go
    │
    │   # ───── everything below is GENERATED — do not edit ─────
    ├── client.go            # ent.Client, ent.Tx — your entry point
    ├── ent.go               # shared types, the Node interface, error helpers
    ├── runtime.go           # wires hooks/defaults/validators into the client
    ├── mutation.go          # mutation types used by hooks
    ├── enttest/             # test helpers (in-memory client)
    ├── hook/                # hook helper types
    ├── migrate/             # migration schema + diff machinery
    ├── predicate/           # predicate types (one per entity)
    ├── user/                # package: user.FieldName, user.NameEQ(...), etc.
    │   ├── user.go          #   constants: field names, edge names, defaults
    │   └── where.go         #   ALL predicate functions for User
    ├── user.go              # the User entity struct (typed fields + Edges)
    ├── user_create.go       # UserCreate / UserCreateBulk builders
    ├── user_update.go       # UserUpdate / UserUpdateOne builders
    ├── user_delete.go       # UserDelete / UserDeleteOne builders
    ├── user_query.go        # UserQuery builder (Where/Order/Limit/With.../All/...)
    └── ...                  # the same set of files for Post, Comment, Tag, ...
```

That is a lot of generated files, and that is normal. You will rarely open them — but knowing what each is (especially `user/where.go`, the home of every predicate) makes debugging unfamiliar compile errors much easier.

> **Best practice:** **Commit the generated `ent/` directory to version control.** It is tempting to `.gitignore` it as "build output," but committing it means a fresh checkout compiles without first running codegen, and code review can see exactly how a schema change rippled through the generated API. Treat the generated diff as a reviewable artifact.

---

## 3. Defining Schemas: Fields

### 3.1 What a schema is, structurally **[B]**

A **schema** in ent is a Go struct that embeds `ent.Schema` and (optionally) overrides a handful of methods. Each method describes one aspect of the entity:

- `Fields()` — the columns/attributes (this section).
- `Edges()` — the relations to other entities (§4).
- `Indexes()` — database indexes (§5).
- `Mixin()` — reusable bundles of the above (§5).
- `Hooks()`, `Interceptors()`, `Policy()` — behavior (§13, §15).

The struct itself holds no data; it is a *description* that the code generator reads. Think of it as a blueprint, not an instance. The entity's actual runtime struct (with your data in it) is the *generated* `ent.User` type, which is different from the `schema.User` you write.

### 3.2 The `Fields()` method — the why and the how **[B]**

`Fields()` returns a `[]ent.Field`. You build each field with a constructor from `entgo.io/ent/schema/field` (`field.String`, `field.Int`, `field.Time`, …) and then chain option methods on it (`.Optional()`, `.Unique()`, `.Default(...)`, validators). The chain reads like a sentence describing the column.

**The most important default to memorize:** a field is **required and non-nullable by default**. `field.String("name")` means a `NOT NULL` column and a non-pointer Go field. You opt *into* nullability with `.Optional()`. This is the opposite of some ORMs and trips up beginners constantly.

Here is a comprehensive, commented `User` schema demonstrating the common field types and options. Read every comment — this single block teaches most of what you need about fields.

```go
// ent/schema/user.go
package schema

import (
	"errors"
	"regexp"
	"time"

	"entgo.io/ent"
	"entgo.io/ent/dialect"
	"entgo.io/ent/schema/field"
)

// User holds the SCHEMA DEFINITION (a blueprint), not user data.
// The generated runtime type is ent.User, defined elsewhere.
type User struct {
	ent.Schema
}

// Fields of the User. Each entry is a column description.
func (User) Fields() []ent.Field {
	return []ent.Field{
		// String field — REQUIRED by default => NOT NULL, non-pointer Go field.
		// NotEmpty() adds a validator: an empty string is rejected before the INSERT.
		field.String("name").
			NotEmpty(),

		// Unique string. Adds a UNIQUE constraint/index at the DB level.
		// Match() attaches a regex validator that runs in Go before the write.
		field.String("email").
			Unique().
			Match(regexp.MustCompile(`^[^@\s]+@[^@\s]+\.[^@\s]+$`)),

		// Optional => the column is NULLABLE.
		// MaxLen() is a string-length validator (checked in Go).
		field.String("bio").
			Optional().
			MaxLen(500),

		// Default value, applied in the GO layer just before INSERT if you
		// did not call SetRole(...). The value here is a literal string.
		field.String("role").
			Default("member"),

		// SchemaType overrides the underlying SQL type PER DIALECT.
		// Use this for ENUMs, TINYINT, custom precision, etc. ent does not
		// otherwise know you want a Postgres-specific type.
		field.String("status").
			Default("active").
			SchemaType(map[string]string{
				dialect.MySQL:    "enum('active','banned','pending')",
				dialect.Postgres: "varchar(16)",
			}),

		// Integer with range validators. Optional + Nillable => *int in the
		// generated struct, so you can distinguish "no age given" (nil) from 0.
		field.Int("age").
			Optional().
			Nillable().
			Min(0).
			Max(150),

		// Float with a default.
		field.Float("score").
			Default(0.0),

		// Bool with a default. In SQL this is BOOLEAN (or its dialect equivalent).
		field.Bool("active").
			Default(true),

		// Time field. Default(time.Now) is a FUNC default: ent calls time.Now()
		// at insert time. Immutable() forbids updating it after creation —
		// exactly what you want for a created_at column.
		field.Time("created_at").
			Default(time.Now).
			Immutable(),

		// UpdateDefault makes ent re-apply the value on EVERY update — the
		// canonical updated_at pattern. (Default handles the initial insert.)
		field.Time("updated_at").
			Default(time.Now).
			UpdateDefault(time.Now),

		// Enum: ent generates a Go type with typed constants and validates that
		// any written value is one of the declared options (defense in depth).
		field.Enum("gender").
			Values("male", "female", "other").
			Optional(),

		// JSON field. Stored as JSONB in Postgres, JSON/TEXT elsewhere. The
		// second argument is the GO type used for (un)marshaling — here a map,
		// but a concrete struct type is usually better for type safety.
		field.JSON("metadata", map[string]any{}).
			Optional(),

		// Sensitive() omits the field from the entity's String() output and
		// from default JSON marshaling. ESSENTIAL for secrets (see security note).
		field.String("password_hash").
			Sensitive().
			NotEmpty(),

		// Custom validator: arbitrary Go logic returning an error to reject.
		field.String("username").
			MinLen(3).
			MaxLen(20).
			Match(regexp.MustCompile(`^[a-zA-Z0-9_]+$`)).
			Validate(func(s string) error {
				if s == "admin" || s == "root" {
					return errors.New("username is reserved")
				}
				return nil
			}),

		// Raw bytes (e.g. a small avatar). Stored as BYTEA/BLOB.
		field.Bytes("avatar").
			Optional(),
	}
}
```

### 3.3 Field options reference **[B]**

| Option | Effect | When to use |
|---|---|---|
| `.Optional()` | Column becomes nullable; Go field stays a **value type** (zero value when NULL). | Any column that may legitimately be absent. |
| `.Nillable()` | Makes the Go field a **pointer** (`*string`, `*int`). Requires `.Optional()`. | When NULL and the zero value are *semantically different* (see §3.4). |
| `.Unique()` | Adds a UNIQUE constraint/index. | Emails, usernames, slugs. |
| `.Immutable()` | Settable on Create only, never on Update. | `created_at`, a UUID id, an owner id. |
| `.Default(v)` | Default value; a literal **or** a `func() T` evaluated per insert. | Timestamps (`time.Now`), status flags, counters. |
| `.UpdateDefault(v)` | Value re-applied automatically on every Update. | `updated_at`. |
| `.Min(n)` / `.Max(n)` | Numeric range validators (run in Go). | Ages, quantities, ratings. |
| `.MinLen(n)` / `.MaxLen(n)` | String length validators. | Names, bios, titles. |
| `.NotEmpty()` | Rejects empty strings. | Any required human text. |
| `.Match(regexp)` | Regex validator. | Emails, slugs, handles. |
| `.Validate(func(T) error)` | Arbitrary custom validator. | Reserved-name checks, cross-field-free business rules. |
| `.Sensitive()` | Excludes the field from `String()` and JSON marshaling. | Passwords, tokens, secrets. |
| `.Comment("…")` | Attaches a comment (surfaces in migration SQL). | Documentation. |
| `.StorageKey("col")` | Override the SQL column name. | Matching a legacy schema. |
| `.SchemaType(map)` | Override the SQL type per dialect. | ENUM, precise decimals, dialect-specific types. |

### 3.4 `Optional` vs `Nillable` — the distinction that confuses everyone **[I]**

This deserves its own paragraph because it is the field concept beginners get wrong most often.

- **`.Optional()` alone:** the database column is nullable, **but the generated Go field is a value type.** A NULL in the DB becomes the Go zero value (`""`, `0`, `false`). You *cannot tell* whether the column was NULL or held the zero value.
- **`.Optional().Nillable()`:** the column is nullable **and the Go field is a pointer.** NULL becomes `nil`; an empty string becomes a pointer to `""`. Now you can distinguish "the user has no bio" (`nil`) from "the user set their bio to empty" (`&""`).

```go
// Value semantics: you lose the NULL-vs-zero distinction.
field.String("bio").Optional()
//  -> generated field: Bio string  (== "" whether NULL or empty)

// Pointer semantics: NULL and "" are distinguishable.
field.String("bio").Optional().Nillable()
//  -> generated field: Bio *string (nil for NULL, &"" for empty)
```

**Rule of thumb:** add `.Nillable()` whenever NULL carries meaning your code must act on (e.g., an optional `deleted_at`, an optional `age`, an optional foreign key). Otherwise plain `.Optional()` is simpler to consume.

### 3.5 Choosing the ID type **[I]**

By default every entity gets an `int` auto-increment primary key named `id`. This is fine for most single-database apps. For distributed systems or public-facing IDs you may prefer UUIDs:

```go
// ent/schema/user.go
import "github.com/google/uuid"

func (User) Fields() []ent.Field {
	return []ent.Field{
		// Override the default int id with a UUID.
		// Default(uuid.New) generates a v4 UUID in Go at insert time.
		field.UUID("id", uuid.UUID{}).
			Default(uuid.New).
			Immutable(),
		// ... other fields ...
	}
}
```

You must `go get github.com/google/uuid`. In PostgreSQL ent maps this to the native `uuid` type. **Trade-off:** random v4 UUIDs fragment B-tree indexes and bloat them versus sequential ints — on very large, write-heavy tables this hurts insert performance and cache locality. If you want UUIDs *and* index locality, use a time-ordered scheme (UUIDv7 or a ULID library) instead of v4. See `RELATIONAL_DB_DESIGN_GUIDE.md` for the key-design theory behind this.

> **⚡ Version note:** `field.UUID` has been stable since v0.9. UUIDv7 support depends on the UUID library you import, not ent itself.

### 3.6 Validators: where, when, and why they are *not* a security boundary **[I]**

Validators (`.Min`, `.MaxLen`, `.Match`, `.Validate`) run **in Go, before the write**, inside the generated mutation code. They are excellent for catching bad data early and returning a clean Go error. But understand their scope precisely:

- They run **only through ent**. If another service writes to the same table directly in SQL, your `.Match` email validator does not apply. Validators are an *application convenience*, not a database constraint. For guarantees that hold no matter who writes, use database-level constraints (UNIQUE, CHECK, NOT NULL) — which ent generates for `.Unique()` and required fields, and which you can add via `SchemaType`/Atlas for CHECK.
- They are **not a substitute for authorization**. A validator answers "is this value well-formed?", not "is this user allowed to set this value?". Authorization belongs in the privacy layer (§15) or your service layer.

> **Security note:** Always pair input validators with database constraints for anything that must be *guaranteed*. A `.Unique()` email field is safe from duplicates even under concurrent inserts because the DB enforces it; a `.Validate` that checks uniqueness by querying first is racy and will let duplicates slip through under load. Let the database enforce invariants; let validators provide friendly early errors.

### 3.7 Annotations **[A]**

Annotations attach extra metadata to a field or schema that *extensions* consume during codegen (GraphQL via `entgql`, OpenAPI via `entoas`, etc.). They do nothing on their own — they are hints for a generator extension.

```go
import "entgo.io/contrib/entgql"

field.String("name").
	Annotations(entgql.OrderField("NAME")), // lets entgql expose ordering by name
```

You will not need annotations until you wire ent into GraphQL or OpenAPI; they are listed here so the term is familiar.

---

## 4. Edges (Relations): the Graph Model

### 4.1 The conceptual model — directed edges **[B]**

This is ent's most distinctive and most misunderstood feature, so we will go slowly.

In ent, a relationship between two entities is an **edge**. Edges are **directed** and are declared with two complementary constructors:

- **`edge.To("name", Target.Type)`** — declares an edge *owned* by this entity, pointing to `Target`. "This entity **has** a `name`."
- **`edge.From("name", Target.Type).Ref("owning_edge")`** — declares the **inverse** view of an edge that some other entity already declared with `To`. The `.Ref(...)` argument names the `To` edge it is the inverse of. "This entity is **referenced by** that `To` edge; give me a way to traverse back."

The critical insight: **`To` and `From` together describe one single relationship.** They are two viewpoints of the same edge, not two separate edges. `From` never creates a new foreign key — it only generates the reverse-traversal API. **Exactly one foreign key (or one join table) is created per relationship**, and it lives where the data model demands (the "many" side for one-to-many, a join table for many-to-many).

### 4.2 The mental model that prevents 90% of edge bugs **[B]**

Beginners assume `To` means "the FK is on this table." **That is wrong.** Here is the correct reading, which you should memorize:

```
edge.To("posts", Post.Type)  on User
   reads as:  "User OWNS the relationship to posts."
   FK location:  on the POSTS table (a user_id column), because a post belongs to a user.

edge.From("author", User.Type).Ref("posts")  on Post
   reads as:  "Post is the inverse side; let me traverse back to the owning User."
   FK location:  unchanged — this declaration adds NO new column. It only enables p.QueryAuthor().
```

So: **`To` = "I own this relationship and define it"; `From` = "I'm the back-reference; let me walk back."** Where the FK physically lives is determined by the *cardinality* (`.Unique()`), not by which side has `To`.

### 4.3 One-to-Many (O2M): User has many Posts **[B]**

A user has many posts; each post belongs to one user. The FK (`user_id`) lives on `posts` (the "many" side). The owning side uses `To`; the belongs-to side uses `From(...).Ref(...).Unique()` — `.Unique()` is what makes the back-reference return a *single* user rather than a slice.

```go
// ent/schema/user.go
import "entgo.io/ent/schema/edge"

func (User) Edges() []ent.Edge {
	return []ent.Edge{
		// One User -> Many Posts. User owns the relationship.
		// FK "user_id" will live on the posts table.
		edge.To("posts", Post.Type),
	}
}
```

```go
// ent/schema/post.go
func (Post) Edges() []ent.Edge {
	return []ent.Edge{
		// Inverse view: each Post belongs to exactly one User.
		// Ref("posts") MUST match the To-edge name on User.
		// .Unique()   => this side returns a single User (cardinality = one).
		// .Required() => the FK is NOT NULL; a post cannot exist without an author.
		edge.From("author", User.Type).
			Ref("posts").
			Unique().
			Required(),
	}
}
```

After codegen you get both directions: `user.QueryPosts()` returns many posts, `post.QueryAuthor()` returns one user, and on the create builder `post.SetAuthor(u)` / `SetAuthorID(id)` sets the FK.

### 4.4 One-to-One (O2O): User has one Profile **[B]**

One-to-one is one-to-many with `.Unique()` on *both* sides. The owning side adds `.Unique()` to cap the cardinality at one; the inverse side does too.

```go
// ent/schema/user.go
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		// Unique() on the owning side caps it at one => O2O.
		edge.To("profile", Profile.Type).
			Unique(),
	}
}
```

```go
// ent/schema/profile.go
func (Profile) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("owner", User.Type).
			Ref("profile").
			Unique().
			Required(), // every profile must belong to a user
	}
}
```

### 4.5 Many-to-Many (M2M): Posts and Tags **[B]**

Many-to-many means each post can have many tags and each tag can label many posts. There is no single column that can hold this — it requires a **join table**. ent creates and manages that join table for you automatically; you never declare it.

```go
// ent/schema/post.go
func (Post) Edges() []ent.Edge {
	return []ent.Edge{
		// M2M owning side. ent auto-creates a "post_tags" join table.
		edge.To("tags", Tag.Type),
	}
}
```

```go
// ent/schema/tag.go
func (Tag) Edges() []ent.Edge {
	return []ent.Edge{
		// Inverse side — no .Unique(), because a tag has MANY posts.
		edge.From("posts", Post.Type).
			Ref("tags"),
	}
}
```

The generated API gives you `post.AddTags(...)`, `post.RemoveTags(...)`, `post.QueryTags()`, and the symmetric methods on `Tag`.

### 4.6 Self-referential edges: User follows Users **[I]**

An edge can point back at the same entity — the classic "followers/following" social graph. It is a normal M2M (or O2M) where the target type is the same schema.

```go
// ent/schema/user.go
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		// Self-referential M2M. "following" is who I follow;
		// "followers" is the inverse — who follows me.
		edge.To("following", User.Type),
		edge.From("followers", User.Type).
			Ref("following"),
	}
}
```

### 4.7 Edge schemas (the "through" table): M2M with extra columns **[I]**

A plain M2M join table holds only the two foreign keys. But real relationships often carry data *about* the relationship — when a user joined a group, what role they have in it, the quantity in an order line. When the join needs its own columns, you promote it to a full entity called an **edge schema** (or "through" entity) and connect it with `.Through(...)`.

```go
// ent/schema/usergroup.go — the JOIN ENTITY (a real schema, with fields)
type UserGroup struct {
	ent.Schema
}

func (UserGroup) Fields() []ent.Field {
	return []ent.Field{
		field.Time("joined_at").Default(time.Now),
		field.Enum("role").Values("member", "moderator", "owner").Default("member"),
	}
}

func (UserGroup) Edges() []ent.Edge {
	return []ent.Edge{
		// The join entity points to BOTH ends; each is unique+required.
		edge.To("user", User.Type).Unique().Required(),
		edge.To("group", Group.Type).Unique().Required(),
	}
}
```

```go
// ent/schema/user.go — declare the M2M, but route it through UserGroup.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("groups", Group.Type).
			Through("user_groups", UserGroup.Type), // <- extra-data join
	}
}
```

Now you can query the relationship rows themselves (`client.UserGroup.Query()...`) to read `joined_at`/`role`, not just the connected entities.

> **⚡ Version note:** Edge schemas via `.Through()` were introduced in v0.10 and are the recommended way to model M2M relationships that carry data. Before that, the only option was a manually shaped join entity with two O2M edges, which lost the M2M traversal sugar.

### 4.8 Edge options reference **[B]**

| Option | Effect | Notes |
|---|---|---|
| `.Unique()` | Caps cardinality at one; the edge returns a single entity, not a slice. | Required on the FK side of O2M and on both sides of O2O. |
| `.Required()` | The FK is `NOT NULL`; the entity cannot be created without the relation. | Use for genuinely mandatory ownership (a post must have an author). |
| `.Immutable()` | The edge cannot be changed after creation. | Use for ownership that should never transfer. |
| `.Ref("toEdgeName")` | On `From` only: names the `To` edge this is the inverse of. | Must exactly match the `To` edge name. |
| `.Through("name", T.Type)` | Routes an M2M through an edge-schema entity. | For join tables with extra columns. |
| `.StorageKey(edge.Column("col"))` | Override the FK column name. | Matching a legacy schema. |
| `.Comment("…")` | Documentation. | Surfaces in migrations. |

### 4.9 Referential actions (ON DELETE) **[I]**

What happens to posts when their author is deleted? By default ent relies on the database's foreign-key behavior. You can request cascading or set-null behavior via annotations so the DDL ent generates includes the right `ON DELETE` clause.

```go
import (
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/entsql"
)

func (Post) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("author", User.Type).
			Ref("posts").
			Unique().
			Required().
			// When the author is deleted, delete their posts too.
			Annotations(entsql.OnDelete(entsql.Cascade)),
	}
}
```

Options include `entsql.Cascade`, `entsql.SetNull`, `entsql.Restrict`, and `entsql.NoAction`. **Choose deliberately:** `Cascade` is convenient but dangerous (deleting one user can wipe thousands of rows); `Restrict` forces you to clean up dependents first, which is often the safer production default. See `RELATIONAL_DB_DESIGN_GUIDE.md` for the trade-offs.

---

## 5. Indexes & Mixins

### 5.1 Indexes: what, why, and the security angle **[I]**

An **index** is a database structure that makes lookups on certain columns fast (turning an O(n) table scan into an O(log n) tree lookup). You define them in the `Indexes()` method. ent already creates an index for every `.Unique()` field and for foreign keys; you add explicit indexes for the columns your `WHERE`, `ORDER BY`, and `JOIN` clauses filter on most.

```go
// ent/schema/user.go
import "entgo.io/ent/schema/index"

func (User) Indexes() []ent.Index {
	return []ent.Index{
		// A single-field UNIQUE index (alternative to field.Unique()).
		index.Fields("email").Unique(),

		// A COMPOSITE index — accelerates queries filtering on both columns,
		// and on the leftmost prefix (first_name alone).
		index.Fields("first_name", "last_name"),

		// A composite UNIQUE index — the canonical multi-tenant uniqueness:
		// usernames must be unique WITHIN a tenant, not globally.
		index.Fields("tenant_id", "username").Unique(),

		// An index spanning a field AND an edge's FK column.
		index.Fields("name").Edges("owner").Unique(),
	}
}
```

> **Why indexes are a security/availability concern, not just performance:** an unindexed column that attackers (or just heavy users) can filter on turns every such query into a full table scan. At scale this is a denial-of-service vector — a handful of expensive queries can saturate the database and take the whole app down. Index the columns exposed to user-controlled filtering, and pair that with query limits (§9). Composite-unique indexes are also a *correctness* guarantee enforced by the database, immune to application-layer race conditions (§3.6).

### 5.2 Mixins: reusable bundles of schema **[I]**

A **Mixin** is a reusable set of fields, edges, indexes, hooks, interceptors, and policy that you embed into multiple schemas — the schema-level analog of Go's struct embedding. The motivation is DRY: almost every table wants `created_at`/`updated_at`; many want soft-delete or a `tenant_id`. Instead of copy-pasting those fields into every schema, you define them once in a mixin and embed it.

```go
// ent/schema/mixin/mixins.go
package mixin

import (
	"time"

	"entgo.io/ent"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/mixin"
)

// TimeMixin provides created_at + updated_at to any schema that embeds it.
type TimeMixin struct {
	mixin.Schema // embed the base mixin to satisfy the interface
}

func (TimeMixin) Fields() []ent.Field {
	return []ent.Field{
		field.Time("created_at").
			Immutable().
			Default(time.Now),
		field.Time("updated_at").
			Default(time.Now).
			UpdateDefault(time.Now),
	}
}

// SoftDeleteMixin adds a nullable deleted_at. The presence of a non-nil
// deleted_at marks a row as "deleted" without physically removing it.
type SoftDeleteMixin struct {
	mixin.Schema
}

func (SoftDeleteMixin) Fields() []ent.Field {
	return []ent.Field{
		field.Time("deleted_at").
			Optional().
			Nillable(), // Nillable so nil clearly means "not deleted"
	}
}
```

Embed mixins via the `Mixin()` method. Mixin fields are prepended to the schema's own fields, in order:

```go
// ent/schema/post.go
import "github.com/you/myblog/ent/schema/mixin"

func (Post) Mixin() []ent.Mixin {
	return []ent.Mixin{
		mixin.TimeMixin{},       // adds created_at, updated_at
		mixin.SoftDeleteMixin{}, // adds deleted_at
	}
}
```

Because mixins can carry **hooks and interceptors too**, they are the idiomatic home for cross-cutting concerns: a `SoftDeleteMixin` can add both the `deleted_at` field *and* an interceptor that automatically filters out soft-deleted rows from every query (§13). That combination — data + behavior in one reusable unit — is what makes mixins powerful rather than just convenient.

---

## 6. Code Generation: What Gets Built & How the Type-Safe Builder Works

### 6.1 Running generation **[B]**

```bash
# From the project root — the normal command:
go generate ./ent

# Equivalent, calling the generator directly:
go run -mod=mod entgo.io/ent/cmd/ent generate ./ent/schema
```

`go generate ./ent` simply executes the `//go:generate` directive in `ent/generate.go` (§2.5). Run it after **every** schema change.

### 6.2 What gets generated, per entity **[I]**

For an entity `User`, ent emits roughly this set. Knowing the map makes errors legible: if the compiler complains about `user.NameEQ`, you know to look in `ent/user/where.go`.

| File / package | Contents | You will touch it... |
|---|---|---|
| `ent/user.go` | The `User` struct: typed fields + an `Edges` holder, plus `String()`, `Unwrap()`. | Read its fields; never edit. |
| `ent/user_query.go` | `UserQuery`: `Where`, `Order`, `Limit`, `Offset`, `With*`, `All`, `Only`, `First`, `Count`, `IDs`, `Exist`, `Aggregate`, `GroupBy`. | Constantly, indirectly. |
| `ent/user_create.go` | `UserCreate` and `UserCreateBulk` builders (`SetX`, `Save`). | Constantly. |
| `ent/user_update.go` | `UserUpdate` (bulk) and `UserUpdateOne` builders. | Constantly. |
| `ent/user_delete.go` | `UserDelete` and `UserDeleteOne` builders. | Constantly. |
| `ent/user/user.go` | Constants: `user.FieldName`, `user.Table`, default values, edge names. | When ordering/selecting by field. |
| `ent/user/where.go` | **Every predicate function**: `user.NameEQ`, `user.AgeGT`, `user.HasPosts`, `user.HasPostsWith`, … | Constantly. |
| `ent/client.go` | The `Client` with sub-clients (`client.User`, `client.Post`, …) and `Tx`. | Constantly. |
| `ent/migrate/schema.go` | The migration schema (table/column/index definitions). | When migrating. |

### 6.3 How the type-safe builder actually works **[I]**

Understanding the *mechanism* turns ent from magic into something you can debug. When you write:

```go
u, err := client.User.Query().
	Where(user.AgeGTE(18)).
	Order(user.ByName()).
	Limit(10).
	All(ctx)
```

each step is an ordinary method on a generated builder struct that **returns the builder** so calls chain (the fluent / builder pattern — see `GO_GUIDE.md` on methods). Crucially:

- `user.AgeGTE(18)` is a **generated function** that exists *because your schema declared an `age` int field*. It returns a `predicate.User` — a closure that, when the query executes, appends a parameterized `age >= $1` to the SQL. The literal `18` is **never interpolated into the SQL string**; it is bound as a parameter. This is the structural reason ent is injection-safe (§17).
- `user.ByName()` and `user.FieldName` exist because you declared a `name` field. If you delete the field and regenerate, these vanish, and any code using them **fails to compile** — the schema and the queries cannot silently drift apart.
- Nothing is resolved by reflection at runtime. The whole query plan is built from typed function calls, then translated to parameterized SQL by ent's `dialect/sql` builder.

So "type-safe" is literal: the set of legal predicates, fields, edges, and setters is exactly what your schema generates, enforced by the Go compiler.

### 6.4 Customizing generation with `entc` (features & extensions) **[I]**

Some capabilities are opt-in **feature flags** you enable in a small generator-config program named `ent/entc.go`. This file is itself a Go program (guarded by `//go:build ignore` so it is not part of your normal build) that calls `entc.Generate`.

```go
//go:build ignore

// ent/entc.go — configures code generation. Run via go generate.
package main

import (
	"log"

	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
)

func main() {
	err := entc.Generate("./schema", &gen.Config{
		Features: []gen.Feature{
			gen.FeaturePrivacy,            // enable the privacy/policy layer (§15)
			gen.FeatureIntercept,          // enable read interceptors (§13)
			gen.FeatureUpsert,             // enable OnConflict/Upsert builders (§8)
			gen.FeatureSnapshot,           // save a schema snapshot for drift detection
			gen.FeatureVersionedMigration, // Atlas versioned migrations (§14)
		},
	})
	if err != nil {
		log.Fatal("running ent codegen:", err)
	}
}
```

Then point `generate.go` at it:

```go
// ent/generate.go
package ent

//go:generate go run -mod=mod ./entc.go
```

> **⚡ Version note:** `gen.FeatureUpsert` was stabilized in v0.12; `gen.FeatureVersionedMigration` (Atlas integration) arrived in v0.11; `gen.FeatureIntercept` (read interceptors) in v0.12. Enable only what you use — each feature enlarges the generated surface.

---

## 7. The Client & Connecting to a Database

### 7.1 Opening a connection **[B]**

`ent.Open(driver, dsn)` opens a connection pool and returns a `*ent.Client` — your entry point for everything. You must import the underlying SQL driver for its side effects (the blank `_` import registers the driver with `database/sql`).

```go
package main

import (
	"context"
	"log"

	_ "github.com/lib/pq" // PostgreSQL driver — registered via side effect

	"github.com/you/myblog/ent"
)

func main() {
	// Open a PostgreSQL connection pool.
	client, err := ent.Open(
		"postgres",
		"host=localhost port=5432 user=postgres password=secret dbname=myblog sslmode=disable",
	)
	if err != nil {
		log.Fatalf("failed connecting to postgres: %v", err)
	}
	defer client.Close() // always close; releases the pool

	ctx := context.Background()

	// Auto-migration: create/alter tables to match the schema.
	// Fine for development; use Atlas versioned migrations in production (§14).
	if err := client.Schema.Create(ctx); err != nil {
		log.Fatalf("failed creating schema resources: %v", err)
	}

	// ... your application code ...
}
```

> **Security note — credentials and TLS:** never hard-code a DSN with a real password in source. Read it from an environment variable or secret manager (`os.Getenv("DATABASE_URL")`). In production set `sslmode=require` (or stricter, `verify-full`) for PostgreSQL so the connection is encrypted and the server certificate verified — `sslmode=disable` is for local dev only. See `POSTGRESQL_GUIDE.md` for the full TLS and role-hardening discussion.

### 7.2 SQLite for local dev and tests **[B]**

SQLite needs no server and is perfect for fast, isolated tests. One flag is mandatory:

```go
import _ "github.com/mattn/go-sqlite3"

client, err := ent.Open("sqlite3", "file:myblog.db?cache=shared&_fk=1")
//                                                            ^^^^ CRITICAL
```

**SQLite disables foreign-key enforcement by default.** Without `_fk=1`, your `.Required()` edges and `ON DELETE` rules are silently ignored, so tests pass that would fail against PostgreSQL. Always include `_fk=1`.

### 7.3 Tuning the connection pool **[I]**

ent wraps `database/sql`, so you tune the pool through the underlying `*sql.DB`. Sensible limits protect the database from connection exhaustion under load.

```go
import (
	"database/sql"
	"time"

	entsql "entgo.io/ent/dialect/sql"
	_ "github.com/lib/pq"
)

func newClient(dsn string) (*ent.Client, error) {
	db, err := sql.Open("postgres", dsn)
	if err != nil {
		return nil, err
	}
	db.SetMaxOpenConns(25)                  // cap total connections
	db.SetMaxIdleConns(10)                  // keep some warm
	db.SetConnMaxLifetime(30 * time.Minute) // recycle to avoid stale conns

	drv := entsql.OpenDB("postgres", db) // wrap the *sql.DB as an ent driver
	return ent.NewClient(ent.Driver(drv)), nil
}
```

> **Best practice:** size `MaxOpenConns` to your database's capacity divided by the number of app instances. An unbounded pool is a classic outage cause: a traffic spike opens thousands of connections and the database falls over.

### 7.4 Auto-migration options **[I]**

```go
import "github.com/you/myblog/ent/migrate"

// Safe additive migration: creates tables, adds columns/indexes.
// NEVER drops anything — non-destructive, fine for dev.
client.Schema.Create(ctx)

// Destructive: also drops removed columns/tables. DEV ONLY — data loss!
client.Schema.Create(ctx,
	migrate.WithDropColumn(true),
	migrate.WithDropIndex(true),
)

// Dry run: write the DDL it WOULD execute to stdout, change nothing.
client.Schema.WriteTo(ctx, os.Stdout)
```

> **Security/safety note:** `WithDropColumn`/`WithDropIndex`/`WithDropTable` are irreversible data-loss switches. They must never run against production. The disciplined alternative — versioned migrations you review before applying — is §14.

---

## 8. CRUD Operations in Depth

Every operation follows the same shape: **start a builder → configure it with `SetX`/`Where` → execute it with a terminal method** (`Save`, `Exec`, `All`, `Only`, …). The builder is just a value you can pass around and conditionally configure before executing.

### 8.1 Create **[B]**

`client.User.Create()` returns a `UserCreate` builder. Each `SetX` is type-checked: `SetAge` takes an `int` because `age` is an int field. `Save(ctx)` executes the INSERT (running defaults, validators, and hooks first) and returns the fully populated entity, including its generated `id`.

```go
// Create one user. Save returns the inserted entity (with its new ID).
u, err := client.User.
	Create().
	SetName("Alice").
	SetEmail("alice@example.com").
	SetAge(30).
	SetRole("admin").
	Save(ctx)
if err != nil {
	return fmt.Errorf("creating user: %w", err)
}
fmt.Println("created:", u.ID, u.Name)

// Create with a relation. On a Required edge you can set the FK directly
// by ID (SetAuthorID) or by passing the entity (SetAuthor).
post, err := client.Post.
	Create().
	SetTitle("Hello ent").
	SetBody("ent is great").
	SetAuthorID(u.ID).
	Save(ctx)
if err != nil {
	return fmt.Errorf("creating post: %w", err)
}

// Bulk create — one round trip, one multi-row INSERT.
users, err := client.User.CreateBulk(
	client.User.Create().SetName("Bob").SetEmail("bob@example.com"),
	client.User.Create().SetName("Carol").SetEmail("carol@example.com"),
).Save(ctx)
```

`Save` vs `SaveX`: every terminal method has an `X` variant (`SaveX`, `OnlyX`, `AllX`) that **panics** instead of returning an error. The `X` forms are handy in tests and `main` setup where a failure should abort; in real request-handling code, always use the error-returning form and handle the error.

### 8.2 Read / Query **[B]**

`client.User.Query()` returns a `UserQuery` you refine with predicates and finish with a terminal. The terminals differ in how they handle "zero" and "many" results — choosing the right one is a correctness decision:

```go
// All: every match. Returns an EMPTY slice + nil error when none match.
users, err := client.User.Query().All(ctx)

// Get: by primary key. Returns a NOT-FOUND error if absent.
u, err := client.User.Get(ctx, 42)

// Only: expects EXACTLY ONE. Errors if 0 (not found) OR >1 (too many).
// Use when a predicate should uniquely identify a row (e.g. by unique email).
u, err = client.User.Query().
	Where(user.EmailEQ("alice@example.com")).
	Only(ctx)

// First: the first match. Returns nil, nil when NONE match — NO error.
// (Watch out: a nil result with a nil error is easy to mishandle.)
u, err = client.User.Query().
	Where(user.NameContains("ali")).
	First(ctx)

// Count, Exist: aggregate-style terminals.
n, err := client.User.Query().Where(user.ActiveEQ(true)).Count(ctx)
exists, err := client.User.Query().Where(user.EmailEQ("alice@example.com")).Exist(ctx)

// Select specific columns only — less data over the wire.
names, err := client.User.Query().Select(user.FieldName, user.FieldEmail).All(ctx)

// IDs only.
ids, err := client.User.Query().IDs(ctx)
```

> **Gotcha to internalize now:** `Only` errors on zero results; `First` returns `nil, nil` on zero results. Mixing these up causes either spurious errors or nil-pointer dereferences. Use `Only` when a row *must* exist and be unique; use `First`/`All` when absence is normal.

### 8.3 Update **[B]**

There are three update entry points:

- `UpdateOneID(id)` — update a single row by primary key.
- `UpdateOne(entity)` — update a single row given an already-loaded entity.
- `Update()` — **bulk** update all rows matching a predicate.

```go
// Update one by ID. ClearBio sets the nullable bio column to NULL.
u, err := client.User.UpdateOneID(42).
	SetName("Alice Updated").
	SetAge(31).
	ClearBio().
	Save(ctx)

// Update from a loaded entity. AddAge atomically increments (age = age + 1)
// in SQL — NOT a read-modify-write in Go, so it's race-safe.
u, err = client.User.UpdateOne(u).
	AddAge(1).
	Save(ctx)

// BULK update: every member becomes a subscriber. Returns the affected count.
n, err := client.User.Update().
	Where(user.RoleEQ("member")).
	SetRole("subscriber").
	Save(ctx)
fmt.Println("updated", n, "users")
```

> **Security note — mass-assignment safety:** ent has no "update from an arbitrary map" footgun. You call explicit `SetX` methods, so a malicious client cannot smuggle a `SetRole("admin")` into an update merely by adding a JSON field — *you* decide which setters run. When building update handlers from request bodies, set only the fields the user is allowed to change; never reflectively copy a request struct into the builder.

### 8.4 Delete **[B]**

```go
// Delete one by ID. Exec returns only an error (no entity to return).
err := client.User.DeleteOneID(42).Exec(ctx)

// Delete a loaded entity.
err = client.User.DeleteOne(u).Exec(ctx)

// Bulk delete by predicate; returns the count deleted.
n, err := client.User.Delete().
	Where(user.ActiveEQ(false)).
	Exec(ctx)
fmt.Println("deleted", n, "inactive users")
```

> **Soft delete:** physical deletes are often the wrong default in production — they destroy audit trails and break references. Prefer the soft-delete mixin (§5.2) plus an interceptor (§13) that filters out `deleted_at != NULL`, so "delete" becomes an update and data is recoverable.

### 8.5 Upsert (insert-or-update) **[I]**

Requires `gen.FeatureUpsert`. An upsert atomically inserts a row or, if it conflicts with an existing unique key, updates it instead — eliminating the racy "SELECT then INSERT-or-UPDATE" pattern.

```go
import "entgo.io/ent/dialect/sql"

err := client.User.Create().
	SetEmail("alice@example.com").
	SetName("Alice").
	OnConflict(
		sql.ConflictColumns(user.FieldEmail), // the unique key to detect conflicts on
	).
	UpdateNewValues(). // on conflict, overwrite with the new values
	Exec(ctx)
```

`UpdateNewValues()` overwrites everything; `Update(func(u *ent.UserUpsert) { u.SetName("...") })` lets you control exactly which columns change on conflict. Upsert is the race-safe way to implement "create if missing, otherwise update."

---

## 9. Predicates, Filtering, Ordering & Pagination

### 9.1 What predicates are and why they are injection-proof **[I]**

A **predicate** is a generated function that represents one condition in a `WHERE` clause. For a `String` field `name`, ent generates `user.NameEQ`, `user.NameContains`, `user.NameHasPrefix`, and so on — each named after your real field. You pass predicates to `Where(...)`, and ent compiles them to **parameterized SQL**: the *structure* (the column, the operator) comes from the generated function; the *value* is always bound as a query parameter, never spliced into the SQL text.

This is the heart of ent's injection safety. There is no `Where("name = '" + input + "'")` API in idiomatic ent — there is only `Where(user.NameEQ(input))`, and `input` is bound, not interpolated. Even `NameContains(userInput)` is safe: ent builds `name LIKE $1` and binds `%userInput%` as the parameter.

### 9.2 The predicate catalog **[I]**

For a **string** field `name`:

| Predicate | SQL |
|---|---|
| `user.NameEQ("Alice")` | `name = $1` |
| `user.NameNEQ("Alice")` | `name <> $1` |
| `user.NameIn("Alice","Bob")` | `name IN ($1,$2)` |
| `user.NameNotIn("Alice")` | `name NOT IN ($1)` |
| `user.NameContains("li")` | `name LIKE '%'||$1||'%'` |
| `user.NameHasPrefix("Al")` | `name LIKE $1||'%'` |
| `user.NameHasSuffix("ce")` | `name LIKE '%'||$1` |
| `user.NameContainsFold("ali")` | case-insensitive contains |
| `user.NameEqualFold("alice")` | case-insensitive `=` |
| `user.NameIsNil()` | `name IS NULL` |
| `user.NameNotNil()` | `name IS NOT NULL` |

For **numeric/comparable** fields `age`:

| Predicate | SQL |
|---|---|
| `user.AgeEQ(30)` | `age = $1` |
| `user.AgeNEQ(30)` | `age <> $1` |
| `user.AgeGT(18)` / `AgeGTE(18)` | `age > $1` / `age >= $1` |
| `user.AgeLT(65)` / `AgeLTE(65)` | `age < $1` / `age <= $1` |
| `user.AgeIn(25,30,35)` | `age IN ($1,$2,$3)` |

### 9.3 Combining predicates: AND / OR / NOT **[I]**

Passing multiple predicates to a single `Where(...)` (or chaining `Where` calls) ANDs them. For OR/NOT and nested boolean logic, use the combinators from the `ent` package (`ent.And`, `ent.Or`, `ent.Not`).

```go
import (
	"github.com/you/myblog/ent"
	"github.com/you/myblog/ent/post"
	"github.com/you/myblog/ent/user"
)

// Implicit AND: adults who are active.
users, err := client.User.Query().
	Where(
		user.AgeGTE(18),
		user.ActiveEQ(true),
	).All(ctx)

// Explicit OR: admins OR moderators.
users, err = client.User.Query().
	Where(
		user.Or(
			user.RoleEQ("admin"),
			user.RoleEQ("moderator"),
		),
	).All(ctx)

// AND + NOT combined.
users, err = client.User.Query().
	Where(
		user.And(
			user.AgeGTE(18),
			user.Not(user.NameEQ("banned_user")),
		),
	).All(ctx)
```

### 9.4 Predicates over edges (the graph filter) **[I]**

This is where ent's graph model shines. For every edge, ent generates a `HasX()` predicate ("has at least one related X") and a `HasXWith(...)` predicate ("has a related X matching these sub-predicates"). These translate to `EXISTS (SELECT ... )` subqueries — you filter parent entities by conditions on their children without writing a JOIN.

```go
// Users who have at least one post.
users, err = client.User.Query().
	Where(user.HasPosts()).
	All(ctx)

// Users who have at least one post whose title contains "ent".
users, err = client.User.Query().
	Where(user.HasPostsWith(post.TitleContains("ent"))).
	All(ctx)

// Deep nesting: comments on posts authored by user 42.
comments, err := client.Comment.Query().
	Where(
		comment.HasPostWith(
			post.HasAuthorWith(user.IDEQ(42)),
		),
	).All(ctx)
```

### 9.5 Ordering **[I]**

```go
// Ascending by a single field. user.FieldCreatedAt is a generated constant.
users, err := client.User.Query().
	Order(ent.Asc(user.FieldCreatedAt)).
	All(ctx)

// Multiple sort keys: highest score first, then name ascending.
users, err = client.User.Query().
	Order(
		ent.Desc(user.FieldScore),
		ent.Asc(user.FieldName),
	).All(ctx)
```

> **⚡ Version note:** newer ent also generates typed order helpers like `user.ByName()` / `user.ByCreatedAt(sql.OrderDesc())` and ordering by aggregated edge counts (`user.ByPostsCount()`). Prefer these where available — they are fully type-checked. The `ent.Asc(user.FieldX)` form remains valid.

### 9.6 Pagination: offset vs cursor **[I]**

```go
const pageSize = 20

// OFFSET pagination — simple but degrades on deep pages (the DB must scan
// and discard all skipped rows) and can skip/duplicate rows under concurrent
// inserts. Fine for small/admin datasets.
page2, err := client.User.Query().
	Order(ent.Asc(user.FieldID)).
	Limit(pageSize).
	Offset(pageSize). // skip the first page
	All(ctx)

// CURSOR (keyset) pagination — efficient and stable. Carry the last seen
// sort key as a cursor; ask for "rows after it". O(log n) via the index,
// and immune to shifting offsets.
afterID := lastSeenID
nextPage, err := client.User.Query().
	Where(user.IDGT(afterID)).
	Order(ent.Asc(user.FieldID)).
	Limit(pageSize).
	All(ctx)
```

> **Security/availability note:** **always set a `Limit` on user-facing list queries.** An endpoint that returns "all rows" is a memory and bandwidth DoS waiting to happen — one client requesting a million-row table can exhaust your server's memory. Default a sane page size, cap the maximum the client may request, and prefer cursor pagination for large datasets. For Relay-style GraphQL pagination, the `entgql` extension generates `Connection` types and a `Paginate(...)` method automatically.

---

## 10. Eager Loading & Edge Traversal — Defeating N+1

### 10.1 The N+1 problem, stated precisely **[I]**

By default, loading a `User` does **not** load its `posts` — edges are lazy. The naive fix is to loop over users and query each one's posts, which issues **1 query for the users + N queries (one per user) for the posts** — the infamous **N+1 problem**. For 1,000 users that is 1,001 round trips, each with network latency. It is the single most common performance disaster in ORM code.

```go
// BAD — N+1: one query for users, then one query PER user for posts.
users, _ := client.User.Query().All(ctx)
for _, u := range users {
	posts, _ := u.QueryPosts().All(ctx) // a separate round trip every iteration!
	_ = posts
}
```

### 10.2 The fix: eager loading with `WithX` **[I]**

`WithPosts()` tells ent to load the posts too. ent issues **one** additional query for *all* the posts of *all* the matched users and stitches them onto each user's `Edges.Posts` in Go — turning N+1 into 2 queries total, regardless of how many users there are.

```go
// GOOD — eager load: 2 queries total no matter how many users.
users, err := client.User.Query().
	WithPosts(). // load each user's posts in one extra query
	All(ctx)
for _, u := range users {
	for _, p := range u.Edges.Posts { // already in memory; no DB call
		fmt.Println(u.Name, "->", p.Title)
	}
}

// Eager-load a single user's posts.
u, err := client.User.Query().
	Where(user.IDEQ(42)).
	WithPosts().
	Only(ctx)
```

### 10.3 Filtering and shaping eager-loaded edges **[I]**

`WithX` accepts an optional function to refine the edge query — filter, order, limit the loaded children. This is how you load "each user's 5 most recent *published* posts" in one shot.

```go
u, err := client.User.Query().
	Where(user.IDEQ(42)).
	WithPosts(func(q *ent.PostQuery) {
		q.Where(post.PublishedEQ(true)).
			Order(ent.Desc(post.FieldCreatedAt)).
			Limit(5)
	}).
	Only(ctx)

// Load multiple edges at once (e.g. posts AND profile).
users, err := client.User.Query().
	WithPosts().
	WithProfile().
	All(ctx)
```

> **Important:** `u.Edges.Posts` is only populated if you called `WithPosts()`. If you forgot, it is an empty slice — not an error, and not lazily loaded. This silent-empty behavior is a common source of "where did my data go?" bugs; if an edge is unexpectedly empty, check that you eager-loaded it.

### 10.4 Traversal: walking the graph **[I]**

When you need related entities of an *already-loaded* entity (and didn't eager-load), use the generated `QueryX()` traversal methods. These execute a fresh query rooted at the current entity — perfect for one-off navigation, but inside a loop they reintroduce N+1, so prefer `WithX` for collections.

```go
// From a user, get their published posts (one query).
posts, err := u.QueryPosts().
	Where(post.PublishedEQ(true)).
	All(ctx)

// Walk the inverse edge: a post's author.
author, err := p.QueryAuthor().Only(ctx)

// Chain traversals: user -> posts -> comments, all in SQL.
comments, err := u.QueryPosts().
	QueryComments().
	All(ctx)

// Count across a traversal without materializing entities.
total, err := u.QueryPosts().QueryComments().Count(ctx)
```

The chained form (`u.QueryPosts().QueryComments()`) compiles to a single SQL query with the appropriate joins/subqueries — it does **not** load posts into Go first. That is the graph model paying off: you express navigation, ent generates efficient SQL.

---

## 11. Aggregations & Grouping

### 11.1 Simple aggregates **[I]**

For whole-table or filtered aggregates without grouping, ent offers `Count`, `Exist`, and an `Aggregate` API:

```go
import "entgo.io/ent/dialect/sql"

// Count active users.
n, err := client.User.Query().Where(user.ActiveEQ(true)).Count(ctx)

// Aggregate without grouping: average age of all users.
var v []struct {
	Avg float64 `json:"avg"`
}
err = client.User.Query().
	Aggregate(ent.As(ent.Mean(user.FieldAge), "avg")).
	Scan(ctx, &v)
fmt.Println("average age:", v[0].Avg)
```

### 11.2 GroupBy + aggregation **[I/A]**

`GroupBy` mirrors SQL `GROUP BY`: you group by one or more fields and compute aggregates per group. The terminal `Scan` decodes the result rows into a slice of a struct you define (with `json` tags matching the selected/aliased columns), or `Strings`/`Ints` for a single column.

```go
// Count posts per author: SELECT user_id, COUNT(*) ... GROUP BY user_id
var byAuthor []struct {
	AuthorID int `json:"author_id"`
	Count    int `json:"count"`
}
err := client.Post.Query().
	GroupBy(post.AuthorColumn).                 // group by the FK column
	Aggregate(ent.Count()).                     // COUNT(*) per group
	Scan(ctx, &byAuthor)

// Group by a single field, return just the values.
roles, err := client.User.Query().
	GroupBy(user.FieldRole).
	Strings(ctx) // distinct role values

// Multiple aggregates per group, with aliases for Scan to map onto.
var stats []struct {
	Role string  `json:"role"`
	Cnt  int     `json:"cnt"`
	Avg  float64 `json:"avg"`
}
err = client.User.Query().
	GroupBy(user.FieldRole).
	Aggregate(
		ent.As(ent.Count(), "cnt"),
		ent.As(ent.Mean(user.FieldAge), "avg"),
	).
	Scan(ctx, &stats)
```

> **Best practice:** the struct field tags in `Scan` must match the selected column names / aliases exactly, or the column silently scans as a zero value. Always alias aggregates with `ent.As(..., "name")` and match the tag. Index the columns you `GroupBy` on for large tables — an unindexed `GROUP BY` is a full scan.

---

## 12. Transactions

### 12.1 What a transaction guarantees, and why you need one **[I]**

A **transaction** groups multiple statements so they either all commit or all roll back — the **atomicity** of ACID. The motivating example: creating a user *and* their first post must both succeed or both fail; a half-done state (user with no post, or worse, a post referencing a user that wasn't created) corrupts your data. Wrapping both writes in a transaction makes the pair atomic.

In ent, `client.Tx(ctx)` starts a transaction and returns a `*ent.Tx` — a transaction-scoped object exposing the same sub-clients (`tx.User`, `tx.Post`, …). Every operation through `tx.*` participates in that transaction. You finish with `tx.Commit()` (persist) or `tx.Rollback()` (discard).

### 12.2 The manual pattern (explicit, verbose) **[I]**

```go
func CreateUserWithPost(ctx context.Context, client *ent.Client) error {
	tx, err := client.Tx(ctx)
	if err != nil {
		return fmt.Errorf("starting tx: %w", err)
	}

	u, err := tx.User.Create().
		SetName("Tx Alice").
		SetEmail("tx-alice@example.com").
		Save(ctx)
	if err != nil {
		_ = tx.Rollback() // discard on ANY error
		return fmt.Errorf("creating user: %w", err)
	}

	if _, err := tx.Post.Create().
		SetTitle("First Post").
		SetAuthor(u).
		Save(ctx); err != nil {
		_ = tx.Rollback()
		return fmt.Errorf("creating post: %w", err)
	}

	return tx.Commit() // commit only if everything succeeded
}
```

The manual pattern is correct but error-prone: forget a single `Rollback()` on an early return and you leak an open transaction (which holds locks and a connection — a serious production bug). That risk motivates the helper.

### 12.3 The `WithTx` helper (the pattern to actually use) **[I/A]**

Wrap the transaction lifecycle once so callers just supply the work. This handles rollback on error, rollback-and-re-panic on panic, and commit on success — eliminating the leak risk.

```go
// WithTx runs fn inside a transaction, committing on success and rolling
// back on error or panic. Use this everywhere instead of hand-rolling Tx.
func WithTx(ctx context.Context, client *ent.Client, fn func(tx *ent.Tx) error) error {
	tx, err := client.Tx(ctx)
	if err != nil {
		return err
	}
	defer func() {
		if v := recover(); v != nil {
			_ = tx.Rollback() // roll back, then re-panic so the panic isn't swallowed
			panic(v)
		}
	}()
	if err := fn(tx); err != nil {
		if rerr := tx.Rollback(); rerr != nil {
			// Surface both the original error and the rollback failure.
			err = fmt.Errorf("%w: rollback failed: %v", err, rerr)
		}
		return err
	}
	if err := tx.Commit(); err != nil {
		return fmt.Errorf("commit: %w", err)
	}
	return nil
}

// Usage: clean, leak-proof.
err := WithTx(ctx, client, func(tx *ent.Tx) error {
	u, err := tx.User.Create().SetName("Bob").SetEmail("bob@example.com").Save(ctx)
	if err != nil {
		return err // triggers rollback
	}
	_, err = tx.Post.Create().SetTitle("Bob's Post").SetAuthor(u).Save(ctx)
	return err
})
```

> **⚡ Version note:** ent v0.14 ships a built-in `client.WithTx(ctx, fn)` with this exact behavior — prefer it if present, but the hand-written helper above is identical and portable across versions.

### 12.4 `tx.Client()` — composing transactional and non-transactional code **[A]**

A common need: a helper function like `CreateUser(c *ent.Client, ...)` that should work *both* standalone and inside a transaction. `tx.Client()` returns an `*ent.Client` bound to the transaction, so you can pass it where a `*ent.Client` is expected and the calls still participate in the tx.

```go
func CreateUser(ctx context.Context, c *ent.Client, name, email string) (*ent.User, error) {
	return c.User.Create().SetName(name).SetEmail(email).Save(ctx)
}

// Same helper, now transactional:
err = WithTx(ctx, client, func(tx *ent.Tx) error {
	_, err := CreateUser(ctx, tx.Client(), "Dave", "dave@example.com")
	return err
})
```

> **Best practices for transactions:** keep them **short** — a transaction holds locks and a connection, so doing slow work (HTTP calls, heavy computation) inside one starves the pool and can deadlock. Do all I/O *before* opening the tx; inside, only the database writes. Be aware of isolation levels (ent uses the database default; pass `sql.TxOptions` for stricter levels via the underlying driver when you need serializable semantics). Never swallow the error from `fn` — returning it is what triggers the rollback.

---

## 13. Hooks & Interceptors

### 13.1 The difference, and when to use each **[A]**

ent gives you two mechanisms to inject cross-cutting behavior into the data layer:

- **Hooks** wrap **mutations** (Create / Update / Delete). They run *around* a write and can inspect and modify the mutation, run side effects, or abort it. Use them for: normalizing input (lowercasing emails), stamping audit fields, emitting events, enforcing write-time invariants.
- **Interceptors** wrap **reads** (queries). They run *around* a `Query` and can modify it (add a filter) or post-process results. Use them for: automatically excluding soft-deleted rows, injecting per-tenant filtering, read-side auditing. Interceptors require `gen.FeatureIntercept`.

Both follow the same middleware shape — a function that takes the "next" handler and returns a wrapping one — identical in spirit to HTTP middleware (see `GO_GUIDE.md`). They can be attached **in the schema** (apply everywhere that type is used) or **on the client at runtime** (apply per process, often using `context.Context` values like the current tenant or user).

### 13.2 Schema hooks **[A]**

Schema hooks live in the entity's `Hooks()` method and run for every mutation of that type. `hook.On(handler, op)` scopes a hook to specific operations.

```go
// ent/schema/user.go
import (
	"context"
	"strings"

	"entgo.io/ent"
	"entgo.io/ent/schema/hook"
)

func (User) Hooks() []ent.Hook {
	return []ent.Hook{
		// On every Create: normalize email to lowercase BEFORE the insert.
		hook.On(
			func(next ent.Mutator) ent.Mutator {
				return hook.UserFunc(func(ctx context.Context, m *ent.UserMutation) (ent.Value, error) {
					if email, ok := m.Email(); ok {
						m.SetEmail(strings.ToLower(strings.TrimSpace(email)))
					}
					return next.Mutate(ctx, m) // proceed with the (modified) mutation
				})
			},
			ent.OpCreate,
		),

		// On Update/UpdateOne: audit-log what changed.
		hook.On(
			func(next ent.Mutator) ent.Mutator {
				return hook.UserFunc(func(ctx context.Context, m *ent.UserMutation) (ent.Value, error) {
					log.Printf("user mutation: op=%s fields=%v", m.Op(), m.Fields())
					return next.Mutate(ctx, m)
				})
			},
			ent.OpUpdate|ent.OpUpdateOne, // bitwise OR to target multiple ops
		),
	}
}
```

The typed `*ent.UserMutation` is the power here: `m.Email()` reads a pending value, `m.SetEmail(...)` rewrites it, `m.Op()` tells you the operation, all type-safe. Returning an error from a hook **aborts the mutation** — a clean way to enforce write rules.

### 13.3 Runtime hooks (client-level) **[A]**

Register hooks on the live client for concerns that depend on runtime context rather than the schema — request logging, metrics, per-tenant guards.

```go
// Apply to ALL mutations of ALL types: time every write.
client.Use(func(next ent.Mutator) ent.Mutator {
	return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
		start := time.Now()
		defer func() {
			log.Printf("op=%s type=%s dur=%s", m.Op(), m.Type(), time.Since(start))
		}()
		return next.Mutate(ctx, m)
	})
})

// Scope to one type only:
client.User.Use(myUserAuditHook)
```

### 13.4 Interceptors: automatic read filtering (soft delete & multi-tenancy) **[A]**

Interceptors are how you make "every query automatically excludes deleted rows" or "every query is scoped to the current tenant" true *by construction*, so no developer can forget the filter. This is both an ergonomics win and a **security control** — a forgotten tenant filter is a cross-tenant data leak; an interceptor closes that gap centrally.

```go
import (
	"context"

	"entgo.io/ent"
	"entgo.io/ent/dialect/sql"
	"entgo.io/ent/intercept"
)

// Soft-delete interceptor: append `deleted_at IS NULL` to every query
// for any entity that has a deleted_at column.
func SoftDeleteInterceptor() ent.Interceptor {
	return intercept.Func(func(ctx context.Context, q intercept.Query) error {
		// Skip the filter when a context flag explicitly asks for deleted rows
		// (e.g. an admin "trash" view or the un-delete path).
		if skip, _ := ctx.Value(skipSoftDelete{}).(bool); skip {
			return nil
		}
		q.WhereP(sql.FieldIsNull("deleted_at"))
		return nil
	})
}

// Register at runtime:
client.Intercept(SoftDeleteInterceptor())
```

> **Security note:** centralizing tenant/soft-delete filters in an interceptor is far safer than relying on every query to remember the `Where`. But interceptors are an *application* control — they only apply to queries that go through ent. For defense in depth against a compromised app or a direct SQL path, combine them with database-level Row-Level Security (`POSTGRESQL_GUIDE.md`) so the isolation holds even outside ent.

---

## 14. Schema Migrations: Auto vs Versioned (Atlas)

### 14.1 The two migration strategies **[I]**

A **migration** evolves your database schema to match your ent schema as the latter changes over time (adding a column, an index, a table). ent offers two strategies, and choosing correctly is a production-readiness decision:

1. **Automatic migration** (`client.Schema.Create(ctx)`) — at startup, ent diffs your schema against the live database and applies additive changes (new tables/columns/indexes) directly. **Pro:** zero ceremony, great for dev and prototyping. **Cons:** it is implicit (no artifact to review), it only does additive changes safely (it won't drop without explicit destructive flags), and running schema changes automatically at app startup is risky in production (no review, no staged rollout, surprises on deploy).
2. **Versioned migration** (via **Atlas**) — ent + Atlas generate timestamped SQL files into a `migrations/` directory. You review them, commit them, and apply them as a deliberate, ordered, checksummed sequence. **This is the production-recommended approach.** Requires `gen.FeatureVersionedMigration`.

### 14.2 Auto-migration recap **[B]**

```go
// Dev workflow: apply additive schema changes at startup.
if err := client.Schema.Create(ctx); err != nil {
	log.Fatal(err)
}
```

Use this for local development and tests. Do **not** use it as your production migration mechanism.

### 14.3 Versioned migrations with Atlas **[I/A]**

The workflow is: (1) change your ent schema, (2) ask Atlas to *diff* the schema against the current migration state and write a new SQL migration file, (3) review the SQL, (4) commit it, (5) apply it (in CI/CD or a deploy step).

```bash
# Install the Atlas CLI (one time).
go install ariga.io/atlas/cmd/atlas@latest

# 1-2-3. Generate a versioned migration named "add_user_status" by diffing
# your ent schema against the existing migrations directory. The dev-url is a
# throwaway database Atlas uses to compute the diff safely.
atlas migrate diff add_user_status \
	--dir "file://ent/migrate/migrations" \
	--to "ent://ent/schema" \
	--dev-url "docker://postgres/16/dev?search_path=public"

# 4. Review and commit the generated SQL file(s) in ent/migrate/migrations/.

# 5. Apply all pending migrations to the target database.
atlas migrate apply \
	--dir "file://ent/migrate/migrations" \
	--url "postgres://user:pass@localhost:5432/myblog?sslmode=disable"
```

Atlas writes human-readable `.sql` files plus an `atlas.sum` checksum file. The checksum is a tamper-evidence guard: if someone edits a previously applied migration, Atlas refuses to proceed, preventing the classic "migration history diverged" disaster.

> **⚡ Version note:** the first-class ent + Atlas versioned-migration integration arrived in v0.11 via `gen.FeatureVersionedMigration` and has been the recommended production strategy since. The `--dev-url` can be SQLite-in-memory for simple cases or a Docker Postgres for full fidelity.

> **Security & safety best practices for migrations:** (1) **Review every generated SQL file** before applying — never apply a diff blind. (2) **Run migrations against staging first.** (3) Use a **dedicated migration database role** with DDL privileges, separate from the application's runtime role (which should only have DML on the tables it needs — principle of least privilege; see `POSTGRESQL_GUIDE.md`). (4) For destructive changes (dropping a column), do it in **multiple deploys** (stop writing the column, deploy; drop it later) so a rollback is always possible. (5) Keep migrations in version control as the single source of truth for the live schema.

---

## 15. The Privacy / Policy Layer

### 15.1 What it is and the threat it addresses **[A]**

ent's **privacy layer** lets you attach **authorization rules directly to a schema** so that *every* query and mutation of that entity is evaluated against them — a form of application-level row-level security. The threat it addresses is the most common real-world data breach: an authorization check forgotten in one handler. If the rule lives on the schema, every code path that touches the entity is covered automatically, no matter who wrote the handler. Requires `gen.FeaturePrivacy`.

A `Policy` has two parts: a **Query policy** (rules for reads) and a **Mutation policy** (rules for writes). Each is an ordered list of rules; each rule can return `privacy.Allow` (permit, stop evaluating), `privacy.Deny` (reject, stop evaluating), or `privacy.Skip` (no opinion, defer to the next rule). The first decisive rule wins; if all skip, the default is to deny.

### 15.2 A policy in practice **[A]**

```go
// ent/schema/post.go
import (
	"context"

	"entgo.io/ent"
	"github.com/you/myblog/ent/privacy" // generated when FeaturePrivacy is on
)

func (Post) Policy() ent.Policy {
	return privacy.Policy{
		Query: privacy.QueryPolicy{
			// Admins can read anything.
			AllowIfAdmin(),
			// Anyone can read published posts.
			AllowIfPublished(),
			// Authors can read their own drafts.
			AllowIfAuthor(),
			// Otherwise deny (explicit, safe default).
			privacy.AlwaysDenyRule(),
		},
		Mutation: privacy.MutationPolicy{
			AllowIfAdmin(),
			DenyIfNotAuthor(), // only the author may modify their post
		},
	}
}

// A rule is a function evaluated in order. It reads the current actor from
// the context (which your auth middleware populates).
func AllowIfAdmin() privacy.QueryMutationRule {
	return privacy.ContextQueryMutationRule(func(ctx context.Context) error {
		v, _ := ctx.Value(viewerKey{}).(*Viewer)
		if v != nil && v.IsAdmin {
			return privacy.Allow // decisive: permit and stop
		}
		return privacy.Skip // no opinion: defer to the next rule
	})
}
```

The viewer/actor is read from `context.Context`, which your authentication middleware sets per request. Because the policy is evaluated inside the generated query/mutation code, a developer literally cannot bypass it by forgetting a check — the check is the data layer.

> **Security note:** the privacy layer is a strong control but it is *application-level* — it protects access **through ent**. It does not protect against direct SQL access or a compromised database role. For the strongest guarantees, layer it on top of database Row-Level Security and least-privilege roles (`POSTGRESQL_GUIDE.md`). Also: always end a policy with an explicit `AlwaysDenyRule()` so the default is deny-by-default, never accidental allow.

---

## 16. Full Worked Example: a Blog (User, Post, Comment)

This section ties everything together into a small but complete application: three entities with realistic relations, codegen, CRUD, eager loading, a transaction, and a test. Use TABS throughout (every `go` block here is tab-indented).

### 16.1 Schema: User

```go
// ent/schema/user.go
package schema

import (
	"regexp"
	"time"

	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/index"
)

type User struct{ ent.Schema }

func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").NotEmpty(),
		field.String("email").
			Unique().
			Match(regexp.MustCompile(`^[^@\s]+@[^@\s]+\.[^@\s]+$`)),
		field.String("bio").Optional().MaxLen(500),
		field.Bool("active").Default(true),
		field.Time("created_at").Immutable().Default(time.Now),
	}
}

func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("posts", Post.Type),       // one user -> many posts
		edge.To("comments", Comment.Type), // one user -> many comments
	}
}

func (User) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("email").Unique(),
	}
}
```

### 16.2 Schema: Post

```go
// ent/schema/post.go
package schema

import (
	"time"

	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
)

type Post struct{ ent.Schema }

func (Post) Fields() []ent.Field {
	return []ent.Field{
		field.String("title").NotEmpty().MaxLen(200),
		field.Text("body"), // Text => no length cap, maps to TEXT
		field.Bool("published").Default(false),
		field.Time("created_at").Immutable().Default(time.Now),
		field.Time("updated_at").Default(time.Now).UpdateDefault(time.Now),
	}
}

func (Post) Edges() []ent.Edge {
	return []ent.Edge{
		// Belongs to one author (inverse of User.posts). FK lives here.
		edge.From("author", User.Type).
			Ref("posts").
			Unique().
			Required(),
		// Has many comments.
		edge.To("comments", Comment.Type),
	}
}
```

### 16.3 Schema: Comment

```go
// ent/schema/comment.go
package schema

import (
	"time"

	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
)

type Comment struct{ ent.Schema }

func (Comment) Fields() []ent.Field {
	return []ent.Field{
		field.Text("body").NotEmpty(),
		field.Time("created_at").Immutable().Default(time.Now),
	}
}

func (Comment) Edges() []ent.Edge {
	return []ent.Edge{
		// A comment belongs to one post and one author.
		edge.From("post", Post.Type).Ref("comments").Unique().Required(),
		edge.From("author", User.Type).Ref("comments").Unique().Required(),
	}
}
```

### 16.4 Generate

```bash
go generate ./ent
```

### 16.5 `main.go` — wire it together

```go
// main.go
package main

import (
	"context"
	"fmt"
	"log"

	_ "github.com/mattn/go-sqlite3"

	"github.com/you/myblog/ent"
	"github.com/you/myblog/ent/post"
	"github.com/you/myblog/ent/user"
)

func main() {
	// SQLite for the demo; note the mandatory _fk=1 flag.
	client, err := ent.Open("sqlite3", "file:blog.db?cache=shared&_fk=1")
	if err != nil {
		log.Fatal(err)
	}
	defer client.Close()

	ctx := context.Background()

	if err := client.Schema.Create(ctx); err != nil {
		log.Fatal(err)
	}

	// ---- CREATE ----
	alice, err := client.User.Create().
		SetName("Alice").
		SetEmail("alice@example.com").
		SetBio("Go enthusiast").
		Save(ctx)
	if err != nil {
		log.Fatal(err)
	}

	bob, err := client.User.Create().
		SetName("Bob").
		SetEmail("bob@example.com").
		Save(ctx)
	if err != nil {
		log.Fatal(err)
	}

	p, err := client.Post.Create().
		SetTitle("Why I love ent").
		SetBody("ent is a fantastic, type-safe Go ORM...").
		SetPublished(true).
		SetAuthor(alice).
		Save(ctx)
	if err != nil {
		log.Fatal(err)
	}

	if _, err := client.Comment.Create().
		SetBody("Great post, Alice!").
		SetPost(p).
		SetAuthor(bob).
		Save(ctx); err != nil {
		log.Fatal(err)
	}

	// ---- QUERY: published posts with author + comments (eager loaded) ----
	posts, err := client.Post.Query().
		Where(post.PublishedEQ(true)).
		WithAuthor(). // 1 extra query for all authors
		WithComments(func(q *ent.CommentQuery) {
			q.WithAuthor().Order(ent.Asc("created_at")) // nested eager load
		}).
		Order(ent.Desc(post.FieldCreatedAt)).
		All(ctx)
	if err != nil {
		log.Fatal(err)
	}
	for _, pp := range posts {
		fmt.Printf("Post: %s (by %s)\n", pp.Title, pp.Edges.Author.Name)
		for _, c := range pp.Edges.Comments {
			fmt.Printf("  %s: %s\n", c.Edges.Author.Name, c.Body)
		}
	}

	// ---- QUERY: graph filter — users who have a published post ----
	authors, err := client.User.Query().
		Where(user.HasPostsWith(post.PublishedEQ(true))).
		All(ctx)
	if err != nil {
		log.Fatal(err)
	}
	for _, a := range authors {
		fmt.Println("Author with a published post:", a.Name)
	}

	// ---- UPDATE ----
	if _, err := client.Post.UpdateOne(p).
		SetTitle("Why I love ent (updated)").
		Save(ctx); err != nil {
		log.Fatal(err)
	}

	// ---- TRANSACTION: create a user + their first post atomically ----
	if err := WithTx(ctx, client, func(tx *ent.Tx) error {
		carol, err := tx.User.Create().
			SetName("Carol").
			SetEmail("carol@example.com").
			Save(ctx)
		if err != nil {
			return err
		}
		_, err = tx.Post.Create().
			SetTitle("Carol's First Post").
			SetBody("Hello, blog!").
			SetAuthor(carol).
			Save(ctx)
		return err
	}); err != nil {
		log.Fatal(err)
	}

	fmt.Println("Done.")
}

// WithTx — the leak-proof transaction helper from §12.
func WithTx(ctx context.Context, client *ent.Client, fn func(*ent.Tx) error) error {
	tx, err := client.Tx(ctx)
	if err != nil {
		return err
	}
	defer func() {
		if v := recover(); v != nil {
			_ = tx.Rollback()
			panic(v)
		}
	}()
	if err := fn(tx); err != nil {
		_ = tx.Rollback()
		return err
	}
	return tx.Commit()
}
```

### 16.6 Testing with `enttest`

ent generates a test helper, `enttest.Open`, that spins up an in-memory SQLite database, runs migrations, and returns a ready client. This is the idiomatic way to test ent code: fast, isolated, no mocking.

```go
// blog_test.go
package main

import (
	"context"
	"testing"

	_ "github.com/mattn/go-sqlite3"

	"github.com/you/myblog/ent/enttest"
	"github.com/you/myblog/ent/user"
)

func TestCreateAndQueryUser(t *testing.T) {
	// In-memory DB; auto-migrated; cleaned up when the test ends.
	client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
	defer client.Close()

	ctx := context.Background()

	want, err := client.User.Create().
		SetName("Test User").
		SetEmail("test@example.com").
		Save(ctx)
	if err != nil {
		t.Fatalf("create: %v", err)
	}

	got, err := client.User.Query().
		Where(user.EmailEQ("test@example.com")).
		Only(ctx)
	if err != nil {
		t.Fatalf("query: %v", err)
	}
	if got.ID != want.ID {
		t.Errorf("got id %v, want %v", got.ID, want.ID)
	}
}
```

> **Why not mock the DB?** ent's generated code is tightly coupled to real SQL semantics (constraints, defaults, transactions, FK enforcement). Mocking the client is brittle and re-implements the database badly. In-memory SQLite gives you real semantics at memory speed — always prefer it for ent tests.

---

## 17. Best Practices & Security

### 17.1 Injection safety — the central guarantee, restated **[I/A]**

The most important security property of idiomatic ent: **your application code never builds a SQL string, so SQL injection is structurally prevented.** Every value you pass to a generated predicate (`user.EmailEQ(input)`), setter (`SetName(input)`), or ordering helper is **parameterized**, not interpolated. An attacker controlling `input` cannot break out of the value into the query structure, because there is no string to break out of.

**Where you can still get injection — the escape hatches.** ent provides raw-SQL escape hatches for queries the builder cannot express: `sql.Modifier`, `(*sql.Selector).Where(sql.P(...))` with hand-written predicates, and the `Querier`/`ExecQuery` raw paths. The moment you concatenate user input into one of these, you reintroduce the vulnerability. The rules:

```go
// SAFE — generated predicate, value is bound as a parameter.
client.User.Query().Where(user.EmailEQ(userInput)).Only(ctx)

// DANGEROUS — raw SQL with concatenated input = classic injection.
// client.User.Query().Where(func(s *sql.Selector) {
//     s.Where(sql.ExprP("email = '" + userInput + "'")) // NEVER do this
// })

// SAFE raw escape hatch — still parameterized with a placeholder + arg.
client.User.Query().Where(func(s *sql.Selector) {
	s.Where(sql.P(func(b *sql.Builder) {
		b.WriteString("email = ").Arg(userInput) // Arg() binds, doesn't interpolate
	}))
}).Only(ctx)
```

> **Rule:** prefer the generated API for everything. When you must drop to raw SQL, **always** bind values via `Arg(...)`/placeholders and never via string concatenation. Treat any `+ userInput` next to SQL as a code-review red flag.

### 17.2 Validation, constraints, and least privilege **[I/A]**

- **Validate in Go for friendly errors, constrain in the DB for guarantees** (§3.6). A `.Match` validator gives a clean error; a `.Unique()` index or a CHECK constraint enforces the invariant even under concurrency and against non-ent writers.
- **Mark secrets `.Sensitive()`** so password hashes and tokens never appear in logs or default JSON output (§3.2). Combine with hashing — never store plaintext credentials; see `GO_JWT_ARGON2_GUIDE.md` for password hashing in Go.
- **Use least-privilege database roles.** The application's runtime role should have only DML (`SELECT/INSERT/UPDATE/DELETE`) on its tables — not DDL. Migrations run under a separate, more privileged role. This limits the blast radius if the app is compromised (`POSTGRESQL_GUIDE.md`).
- **Set query limits** on anything user-facing (§9.6) to prevent memory/bandwidth DoS.
- **Centralize tenant/soft-delete filtering in interceptors** (§13) and back it with database Row-Level Security for defense in depth.

### 17.3 Codegen hygiene & workflow discipline **[I]**

- **Always regenerate after schema changes** (`go generate ./ent`). The mismatch between a changed schema and stale generated code is the #1 source of confusing errors.
- **Commit the generated `ent/` directory** so checkouts build immediately and schema changes are visible in review (§2.6).
- **Pin your ent version** in `go.mod` and bump it deliberately; regenerate and run tests after upgrading, since generated code can change between versions.
- **Use versioned (Atlas) migrations in production**, review every diff, and never run destructive auto-migration flags against real data (§14).

### 17.4 Performance best practices **[I/A]**

- **Eager-load to kill N+1** (`WithX`) instead of querying in loops (§10).
- **`Select` only the columns you need** for read-heavy paths.
- **Index the columns you filter, sort, and group on** (§5.1); profile slow queries with the database's `EXPLAIN` (see `POSTGRESQL_GUIDE.md`).
- **Keep transactions short** and free of network I/O (§12).
- **Tune the connection pool** to your database's capacity (§7.3).
- **Prefer cursor pagination** over deep offsets (§9.6).

---

## 18. Tips, Tricks & Gotchas

### 18.1 Edge direction confusion — the #1 beginner trap

```
WRONG: "edge.To means the FK is on this table."
RIGHT:
  edge.To("posts", Post.Type) on User      => User OWNS the relation;
                                               FK (user_id) is on the POSTS table.
  edge.From("author", User.Type).Ref("posts") on Post
                                            => INVERSE view only; adds NO new FK;
                                               just enables post.QueryAuthor().
Cardinality (.Unique) decides where the FK lives, not which side has To.
```

### 18.2 `Optional` vs `Nillable`

```go
field.String("bio").Optional()           // Bio string  — "" whether NULL or empty
field.String("bio").Optional().Nillable() // Bio *string — nil for NULL, &"" for empty
// Use Nillable whenever NULL carries meaning your code must distinguish.
```

### 18.3 `Only` vs `First` vs `All` return semantics

```go
// All():   empty slice + nil error when no matches.
// Only():  nil + NOT-FOUND error when 0 matches; ALSO errors when >1 match.
// First(): nil + NIL error when 0 matches (no error!) — easy to mishandle.

u, err := client.User.Get(ctx, id)
if ent.IsNotFound(err) { // typed error check
	http.Error(w, "not found", http.StatusNotFound)
	return
}
```

Use `ent.IsNotFound`, `ent.IsConstraintError`, and `ent.IsValidationError` to branch on error categories rather than string-matching messages.

### 18.4 SQLite foreign keys are OFF by default

```go
ent.Open("sqlite3", "file:my.db?_fk=1") // _fk=1 is MANDATORY
// Without it, .Required() edges and ON DELETE rules are silently ignored,
// so tests pass that would fail against PostgreSQL.
```

### 18.5 Upsert needs the feature flag

```go
// .OnConflict()/.UpdateNewValues() only exist if gen.FeatureUpsert is enabled
// in entc.go. Enable it, then re-run go generate ./ent.
```

### 18.6 An eager-loaded edge you forgot to load is silently empty

```go
u, _ := client.User.Query().Only(ctx) // forgot WithPosts()
_ = u.Edges.Posts                       // == empty slice, NOT lazily loaded, NO error
```

### 18.7 Schema comments flow into migration SQL

```go
field.String("name").Comment("Display name shown in the UI")
// Atlas surfaces this in generated migration SQL as documentation.
```

### 18.8 Mass-assignment is not possible by accident

Because you call explicit `SetX` methods, there is no "bind this whole request body into the update" footgun (§8.3). When mapping request data into a builder, set only the fields the actor is authorized to change.

### 18.9 Use `enttest` for tests — never mock

In-memory SQLite via `enttest.Open` gives real SQL semantics at memory speed (§16.6). Mocking the ent client is brittle and re-implements the database poorly.

---

## 19. Study Path & Build-to-Learn Projects

Work through these in order. Each step produces working code and builds on the last. Cross-reference the related library guides as you go: `RELATIONAL_DB_DESIGN_GUIDE.md` for modeling theory, `POSTGRESQL_GUIDE.md` for the production database, and `GO_GUIDE.md` for any language feature you stumble on.

### Phase 1 — Core fundamentals (Week 1) [B]

1. **Follow the official "Getting Started"** at `entgo.io/docs/getting-started` using SQLite (skip DB setup). Internalize the codegen loop (§2.1): edit schema → `go generate ./ent` → write code.
2. **Build: URL shortener.** Entities: `Link` (url, slug, click_count) and `Click` (ip, user_agent, created_at), O2M. Practice Create, Query, atomic increment (`AddClickCount(1)`), Delete.
3. **Drill: field options.** Rebuild `Link` three times adding (a) validators, (b) defaults, (c) a UUID primary key. Observe how the generated setters change.

### Phase 2 — Relations & generation (Week 2) [I]

4. **Build: task manager.** `User`, `Project`, `Task`, `Label`. Relations: User O2M Tasks, Project O2M Tasks, Tasks M2M Labels. Implement every edge type and traverse them.
5. **Drill: predicates.** Write ten different `Where` clauses: `In`, `Contains`, `HasEdgeWith`, `And`/`Or`/`Not`, and a case-insensitive `Fold`.
6. **Study: generated internals.** Open a generated `_query.go` and the entity's `where.go`. Understand how predicates become parameterized SQL (§6.3) so you can debug unfamiliar errors.

### Phase 3 — Production patterns (Week 3) [I/A]

7. **Add Atlas versioned migrations** to the task manager. Generate a migration, apply it, make a schema change, generate the next one, review the SQL diff.
8. **Add mixins.** Create `TimeMixin` and `SoftDeleteMixin`; apply to all entities; observe the generated changes.
9. **Build: multi-tenant skeleton.** Add a `Tenant` entity and a runtime **interceptor** that injects tenant filtering on every query from a `context.Context` value (§13). Confirm it cannot be bypassed.
10. **Add transactions.** Wrap each business operation (create project + first task; complete task + update project status) in the `WithTx` helper (§12).

### Phase 4 — Advanced & integration (Week 4) [A]

11. **Add hooks.** Schema hook: lowercase email on create/update. Runtime hook: audit-log every mutation.
12. **Add a privacy policy.** Enable `gen.FeaturePrivacy`; write a policy where only a post's author can update/delete it; test allow and deny paths (§15).
13. **Aggregations.** Add a dashboard query: posts-per-author and average completion time, using `GroupBy` + `Aggregate` (§11).
14. **Harden it.** Add query limits, least-privilege DB roles, `.Sensitive()` on secrets, and review every raw-SQL escape hatch (there should be none) (§17).

### Reference projects to study

| Project | What to learn |
|---|---|
| `github.com/ent/contrib/examples` | Official examples: GraphQL, gRPC, real-world patterns. |
| `github.com/ent/ent` (the source) | Read a generated package and the `dialect/sql` builder. |
| Any open-source Go SaaS using ent | Real schema-design decisions and multi-tenancy patterns. |

### Key resources (offline-friendly)

| Resource | URL |
|---|---|
| Official docs | `https://entgo.io/docs/getting-started` |
| ent GitHub | `https://github.com/ent/ent` |
| ent contrib (GraphQL, gRPC, …) | `https://github.com/ent/contrib` |
| Atlas migration docs | `https://atlasgo.io/docs` |
| `go doc` offline | `go doc entgo.io/ent/schema/field` |

---

> **Final note:** ent's superpower is that your schema *is* your documentation, your migration source of truth, and your query API — all in one place, all in Go, all checked by the compiler. The two habits that pay off forever: (1) get the schema right up front, because everything generates from it; and (2) stay inside the generated, parameterized API, because that is what keeps your queries both type-safe and injection-safe. Invest in the schema, run `go generate`, and let the compiler carry the weight.
