# ent — The Go Entity Framework & ORM: Complete Reference Guide

> **Version note:** This guide targets **ent v0.14+** (2025–2026) with Go 1.22+. ent is a stable, production-grade framework maintained by Meta and the open-source community. Where APIs differ across minor versions, sections are flagged with **⚡ Version note**. Always cross-check the official docs at **entgo.io** for the very latest. The core concepts — schema-as-code, code generation, type-safe query builders — have been stable for years.

---

## Table of Contents

1. [What ent Is](#1-what-ent-is)
2. [Setup & Initialization](#2-setup--initialization)
3. [Defining Schema: Fields](#3-defining-schema-fields)
4. [Edges (Relations)](#4-edges-relations)
5. [Indexes & Mixins](#5-indexes--mixins)
6. [Code Generation](#6-code-generation)
7. [Client & Migrations](#7-client--migrations)
8. [CRUD Operations](#8-crud-operations)
9. [Predicates, Filtering, Ordering & Pagination](#9-predicates-filtering-ordering--pagination)
10. [Eager Loading & Edge Traversal](#10-eager-loading--edge-traversal)
11. [Transactions](#11-transactions)
12. [Hooks & Privacy](#12-hooks--privacy)
13. [Full Worked Example: Blog (User, Post, Comment)](#13-full-worked-example-blog)
14. [GraphQL, net/http & Gin Integration](#14-graphql-nethttp--gin-integration)
15. [Tips, Tricks & Gotchas](#15-tips-tricks--gotchas)
16. [Study Path](#16-study-path)

---

## 1. What ent Is

**ent** (pronounced "entity") is a Go entity framework / ORM developed at Meta (Facebook) and open-sourced in 2019. Its core philosophy is: **define your data schema as Go code, then generate all the database access layer from it.**

### The Three Pillars

| Pillar | What it means |
|---|---|
| **Schema-as-Go-code** | Entities, fields, and relations live in `.go` files under `ent/schema/`. No YAML, no JSON, no SQL DDL. |
| **Code generation** | Running `go generate ./ent` produces a full, type-safe client — builders, predicates, query methods, mutation methods. |
| **Type-safe queries** | No `interface{}`, no raw strings for column names. The compiler catches typos. |

### How ent Differs from GORM and Prisma

| Feature | ent | GORM | Prisma (Go client) |
|---|---|---|---|
| Schema language | Go structs | Go structs (tags) | Prisma schema file (.prisma) |
| Query API | Generated type-safe builders | Chainable with `interface{}` | Generated, type-safe |
| Code generation | Full client generated | Partial (no generation) | Full client generated |
| Graph traversal | First-class (`With...()`, `QueryEdge()`) | Manual joins | Manual |
| Migrations | Auto + Atlas versioned | Auto, GORM migrate | Prisma migrate |
| Hooks | Schema + runtime hooks | Callbacks | Limited |
| Privacy layer | Built-in policy engine | None | None |
| Language | Go only | Go only | Multiple (Go, JS, Python…) |
| Maturity | Production (Meta-backed) | Production (community) | Varies by language |

### Supported Databases

| Database | Status |
|---|---|
| PostgreSQL | Full support |
| MySQL / MariaDB | Full support |
| SQLite | Full support |
| CockroachDB | Full support |
| TiDB | Full support |
| Gremlin (AWS Neptune) | Experimental graph support |

> **⚡ Version note:** CockroachDB and TiDB support was added in v0.11 and has matured significantly through v0.13–v0.14.

---

## 2. Setup & Initialization

### Prerequisites

```bash
# Go 1.22+ required
go version   # e.g. go version go1.22.4 linux/amd64
```

### Create a new module and install ent

```bash
mkdir myblog && cd myblog
go mod init github.com/you/myblog

# Add ent as a dependency
go get entgo.io/ent@latest
```

### Initialize the first schema (User entity)

```bash
# This runs the ent CLI tool inline (no global install needed)
go run -mod=mod entgo.io/ent/cmd/ent new User
```

This creates:

```
ent/
├── generate.go          # contains //go:generate directive
└── schema/
    └── user.go          # your User schema skeleton
```

Add more entities:

```bash
go run -mod=mod entgo.io/ent/cmd/ent new Post Comment Tag
```

### Project layout after generation

```
myblog/
├── go.mod
├── go.sum
├── main.go
└── ent/
    ├── generate.go          # //go:generate go run -mod=mod entgo.io/ent/cmd/ent generate ./schema
    ├── schema/
    │   ├── user.go          # YOU write these
    │   ├── post.go
    │   └── comment.go
    │   
    │   # Everything below is GENERATED — do not edit manually
    ├── client.go            # ent.Client, ent.Tx
    ├── config.go
    ├── context.go
    ├── ent.go               # shared types, Node interface
    ├── enttest/             # helpers for tests
    ├── hook/                # hook types
    ├── migrate/             # migration schema
    ├── predicate/           # predicate types
    ├── user.go              # User entity type
    ├── user_create.go       # UserCreate builder
    ├── user_update.go       # UserUpdate / UserUpdateOne builders
    ├── user_delete.go       # UserDelete builder
    ├── user_query.go        # UserQuery builder
    └── ...                  # same pattern for Post, Comment, etc.
```

### The generate.go file

```go
// ent/generate.go
package ent

//go:generate go run -mod=mod entgo.io/ent/cmd/ent generate ./schema
```

---

## 3. Defining Schema: Fields

The `Fields()` method on your schema struct returns a slice of `ent.Field` descriptors. ent uses Go types directly — no reflection hacks, no string-based column definitions.

### Basic field types

```go
// ent/schema/user.go
package schema

import (
    "time"

    "entgo.io/ent"
    "entgo.io/ent/schema/field"
)

// User holds the schema definition for the User entity.
type User struct {
    ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
    return []ent.Field{
        // String field — required by default (NOT NULL in SQL)
        field.String("name"),

        // Optional string — allows NULL
        field.String("bio").
            Optional(),

        // Unique constraint (adds UNIQUE index)
        field.String("email").
            Unique(),

        // Default value (applied at the Go layer before insert)
        field.String("role").
            Default("member"),

        // SchemaType lets you specify the underlying SQL type
        // (useful for ENUM, TINYINT, etc.)
        field.String("status").
            Default("active").
            SchemaType(map[string]string{
                dialect.MySQL:    "enum('active','banned','pending')",
                dialect.Postgres: "varchar(10)",
            }),

        // Integer with range validator
        field.Int("age").
            Optional().
            Nillable().               // generates *int in the struct (nil-able pointer)
            Min(0).
            Max(150),

        // Float
        field.Float("score").
            Default(0.0),

        // Bool with default
        field.Bool("active").
            Default(true),

        // Time — ent.CreatedAt / UpdatedAt annotations for auto-management
        field.Time("created_at").
            Default(time.Now).        // called once at insert
            Immutable(),              // cannot be updated after creation

        field.Time("updated_at").
            Default(time.Now).
            UpdateDefault(time.Now),  // automatically updated on mutation

        // UUID primary key (override the default int ID)
        // See note below about UUID IDs
        // field.UUID("id", uuid.UUID{}).Default(uuid.New),

        // Enum using Go iota-style constants
        field.Enum("gender").
            Values("male", "female", "other").
            Optional(),

        // JSON field (stored as JSONB in Postgres, TEXT in SQLite)
        field.JSON("metadata", map[string]interface{}{}).
            Optional(),

        // Bytes
        field.Bytes("avatar").
            Optional(),
    }
}
```

> **⚡ Version note:** `field.UUID` became stable in v0.9. For PostgreSQL, ent maps it to the native `uuid` type. For SQLite/MySQL it stores as a 16-byte blob or varchar(36).

### Field Options Reference

| Option | Effect |
|---|---|
| `.Optional()` | Column is nullable; Go field is a value type (zero value on nil) |
| `.Nillable()` | Makes the Go struct field a pointer (`*string`, `*int`, …) — requires `.Optional()` |
| `.Unique()` | Adds a UNIQUE constraint/index |
| `.Immutable()` | Field can only be set on Create, not Update |
| `.Default(v)` | Default value; can be a literal or a `func() T` |
| `.UpdateDefault(v)` | Value applied automatically on Update (useful for `updated_at`) |
| `.Min(n)` / `.Max(n)` | Numeric range validators |
| `.MinLen(n)` / `.MaxLen(n)` | String length validators |
| `.Match(regexp)` | Regex validator |
| `.Validate(func(T)error)` | Custom validator function |
| `.Comment("…")` | SQL column comment |
| `.StorageKey("col_name")` | Override the SQL column name |
| `.Sensitive()` | Field is omitted from JSON marshaling and logs |
| `.SchemaType(map)` | Override the SQL type per dialect |

### Custom Validators

```go
field.String("username").
    MinLen(3).
    MaxLen(20).
    Match(regexp.MustCompile(`^[a-zA-Z0-9_]+$`)).
    Validate(func(s string) error {
        // custom logic — reserved names, profanity filter, etc.
        if s == "admin" {
            return errors.New("username 'admin' is reserved")
        }
        return nil
    }),
```

### Annotations

Annotations attach extra metadata to fields (used by extensions like entgql, entoas, etc.).

```go
import "entgo.io/contrib/entgql"

field.String("name").
    Annotations(entgql.OrderField("NAME")),
```

---

## 4. Edges (Relations)

Edges define relationships between entities. ent uses **directed** edges with explicit `From` / `To` declarations.

### Edge direction rule

- `edge.To("name", TargetType.Type)` — this entity **owns** the relation (the FK lives on the target side for O2M, or a join table for M2M).
- `edge.From("name", TargetType.Type).Ref("to_edge_name")` — this entity references the **inverse** of a `To` edge.

### One-to-Many (O2M): User has many Posts

```go
// ent/schema/user.go
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        // One User → Many Posts
        // The FK "user_id" lives on the posts table
        edge.To("posts", Post.Type),
    }
}

// ent/schema/post.go
func (Post) Edges() []ent.Edge {
    return []ent.Edge{
        // Inverse: each Post belongs to one User
        // Ref("posts") must match the To name above
        edge.From("author", User.Type).
            Ref("posts").
            Unique().       // makes the FK side (post.author) return a single User
            Required(),     // NOT NULL — post must have an author
    }
}
```

### One-to-One (O2O): User has one Profile

```go
// ent/schema/user.go
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("profile", Profile.Type).
            Unique(),   // unique on User side → O2O
    }
}

// ent/schema/profile.go
func (Profile) Edges() []ent.Edge {
    return []ent.Edge{
        edge.From("owner", User.Type).
            Ref("profile").
            Unique().
            Required(),
    }
}
```

### Many-to-Many (M2M): Posts and Tags

```go
// ent/schema/post.go
func (Post) Edges() []ent.Edge {
    return []ent.Edge{
        // M2M: ent generates a "post_tags" join table automatically
        edge.To("tags", Tag.Type),
    }
}

// ent/schema/tag.go
func (Tag) Edges() []ent.Edge {
    return []ent.Edge{
        edge.From("posts", Post.Type).
            Ref("tags"),
    }
}
```

### Self-referential Edge: User follows Users

```go
// ent/schema/user.go
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        // Followers/Following — self-referential M2M
        edge.To("following", User.Type),
        edge.From("followers", User.Type).
            Ref("following"),
    }
}
```

### Edge with a Through/Join Entity (edge schema)

When your M2M join table needs extra columns (e.g., a "joined_at" timestamp), use an **edge schema**:

```go
// ent/schema/usergroup.go  — the join entity
type UserGroup struct {
    ent.Schema
}

func (UserGroup) Fields() []ent.Field {
    return []ent.Field{
        field.Time("joined_at").Default(time.Now),
    }
}

func (UserGroup) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("user", User.Type).Required().Unique(),
        edge.To("group", Group.Type).Required().Unique(),
    }
}

// ent/schema/user.go
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("groups", Group.Type).
            Through("user_groups", UserGroup.Type),
    }
}
```

> **⚡ Version note:** Edge schemas (`.Through()`) were introduced in v0.10 and are the recommended way to handle M2M with extra join-table data.

### Edge Options Reference

| Option | Effect |
|---|---|
| `.Unique()` | Makes the edge return a single entity (not a slice) — required for O2O and the FK side of O2M |
| `.Required()` | The edge FK is NOT NULL; entity cannot be created without it |
| `.Immutable()` | Edge cannot be changed after creation |
| `.StorageKey(edge.Column("col"))` | Override the FK column name |
| `.Annotations(...)` | Attach extension metadata |
| `.Comment("…")` | Documentation |

---

## 5. Indexes & Mixins

### Indexes

Define indexes in the `Indexes()` method:

```go
// ent/schema/user.go
import "entgo.io/ent/schema/index"

func (User) Indexes() []ent.Index {
    return []ent.Index{
        // Single-field unique index (alternative to field.Unique())
        index.Fields("email").Unique(),

        // Composite index (for multi-column WHERE clauses)
        index.Fields("first_name", "last_name"),

        // Composite unique index
        index.Fields("tenant_id", "username").Unique(),

        // Index spanning a field AND an edge FK
        index.Fields("name").Edges("owner").Unique(),
    }
}
```

### Mixins

A **Mixin** is a reusable set of fields, edges, indexes, and hooks that you can embed into multiple schemas — similar to Go embedding.

```go
// ent/schema/mixin/timestamps.go
package mixin

import (
    "time"

    "entgo.io/ent"
    "entgo.io/ent/schema/field"
    "entgo.io/ent/schema/mixin"
)

// TimeMixin provides created_at and updated_at to any schema.
type TimeMixin struct {
    mixin.Schema
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

// SoftDeleteMixin adds a nullable deleted_at field.
type SoftDeleteMixin struct {
    mixin.Schema
}

func (SoftDeleteMixin) Fields() []ent.Field {
    return []ent.Field{
        field.Time("deleted_at").
            Optional().
            Nillable(),
    }
}
```

Embed in any schema:

```go
// ent/schema/post.go
func (Post) Mixin() []ent.Mixin {
    return []ent.Mixin{
        mixin.TimeMixin{},       // gets created_at, updated_at
        mixin.SoftDeleteMixin{}, // gets deleted_at
    }
}
```

> Mixins can also define Hooks, Interceptors, Indexes, and Policy — making them extremely powerful for cross-cutting concerns like audit trails, soft-delete, and multi-tenancy.

---

## 6. Code Generation

### Running generation

```bash
# From your project root
go generate ./ent
```

Or equivalently:

```bash
go run -mod=mod entgo.io/ent/cmd/ent generate ./ent/schema
```

### What gets generated

For each entity (e.g., `User`) ent generates:

| File | Contents |
|---|---|
| `ent/user.go` | `User` struct with typed fields, `String()`, `Unwrap()` |
| `ent/user_query.go` | `UserQuery` builder with `Where`, `Order`, `Limit`, `Offset`, `With*`, `All`, `Only`, `First`, `Count`, `IDs`, `Exist` |
| `ent/user_create.go` | `UserCreate` / `UserCreateBulk` builders |
| `ent/user_update.go` | `UserUpdate` / `UserUpdateOne` builders |
| `ent/user_delete.go` | `UserDelete` / `UserDeleteOne` builders |
| `ent/user/user.go` | Package-level field names (`user.FieldName`, `user.Label`, etc.) |
| `ent/user/where.go` | All predicate functions (`user.NameEQ`, `user.AgeGT`, …) |
| `ent/client.go` | `Client` struct with `User`, `Post`, etc. sub-clients |
| `ent/migrate/schema.go` | Migration schema definition |

### Customizing generation with EntGo config

Create `ent/entc.go` to configure code generation (add extensions, change features):

```go
//go:build ignore

package main

import (
    "log"

    "entgo.io/ent/entc"
    "entgo.io/ent/entc/gen"
)

func main() {
    err := entc.Generate("./schema", &gen.Config{
        // Add generated code to a custom package path
        // Header: "// Custom license header\n",
        Features: []gen.Feature{
            gen.FeaturePrivacy,       // enable privacy layer
            gen.FeatureSnapshot,      // save schema snapshot for drift detection
            gen.FeatureIntercept,     // enable interceptors
            gen.FeatureVersionedMigration, // Atlas versioned migrations
            gen.FeatureUpsert,        // enable Upsert builder
        },
    })
    if err != nil {
        log.Fatal("running ent codegen:", err)
    }
}
```

Then update `ent/generate.go`:

```go
package ent

//go:generate go run -mod=mod entc.go
```

> **⚡ Version note:** `gen.FeatureUpsert` (upsert/insert-or-update) was stabilized in v0.12. `gen.FeatureVersionedMigration` for Atlas integration was added in v0.11.

---

## 7. Client & Migrations

### Opening a connection

```go
package main

import (
    "context"
    "log"

    _ "github.com/lib/pq"           // Postgres driver
    // _ "github.com/mattn/go-sqlite3" // SQLite driver
    // _ "github.com/go-sql-driver/mysql" // MySQL driver

    "github.com/you/myblog/ent"
)

func main() {
    // Open a PostgreSQL connection
    client, err := ent.Open("postgres", "host=localhost port=5432 user=postgres dbname=myblog sslmode=disable")
    if err != nil {
        log.Fatalf("failed connecting to postgres: %v", err)
    }
    defer client.Close()

    ctx := context.Background()

    // Run auto-migration (creates/alters tables to match schema)
    // Safe for development; use Atlas versioned migrations in production
    if err := client.Schema.Create(ctx); err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }

    // Your app code here...
}
```

### SQLite (great for local dev & tests)

```go
import _ "github.com/mattn/go-sqlite3"

client, err := ent.Open("sqlite3", "file:myblog.db?cache=shared&_fk=1")
```

> Always include `_fk=1` for SQLite — it enforces foreign keys (disabled by default in SQLite).

### Auto-migration options

```go
// Create tables, add columns, add indexes — but NEVER drops columns/tables
client.Schema.Create(ctx)

// Drop-and-recreate everything (dev only!)
client.Schema.Create(ctx, migrate.WithDropTable(true), migrate.WithDropColumn(true))

// Inspect what SQL would be generated (dry run)
client.Schema.WriteTo(ctx, os.Stdout)
```

### Atlas Versioned Migrations (production recommended)

```bash
# Install Atlas CLI
go install ariga.io/atlas/cmd/atlas@latest

# Generate a versioned migration file from your ent schema diff
atlas migrate diff my_migration \
    --dir "file://ent/migrate/migrations" \
    --to "ent://ent/schema" \
    --dev-url "sqlite://file?mode=memory&_fk=1"

# Apply pending migrations
atlas migrate apply \
    --dir "file://ent/migrate/migrations" \
    --url "postgres://user:pass@localhost/myblog"
```

This produces timestamped SQL files in `ent/migrate/migrations/` that you can review, version-control, and apply safely.

> **⚡ Version note:** The ent + Atlas integration (versioned migrations as a first-class feature) was introduced in v0.11 via the `gen.FeatureVersionedMigration` code-gen feature. It is the strongly recommended migration strategy for production.

---

## 8. CRUD Operations

All operations use the generated client. Every builder follows the pattern: **configure → execute**.

### Create

```go
// Create a single user
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
fmt.Println("created user:", u.ID, u.Name)

// Create with edge (set author when creating a post)
post, err := client.Post.
    Create().
    SetTitle("Hello ent").
    SetBody("ent is great").
    SetAuthorID(u.ID).   // SetXID for FK fields on Required edges
    Save(ctx)

// Bulk create
users, err := client.User.CreateBulk(
    client.User.Create().SetName("Bob").SetEmail("bob@example.com"),
    client.User.Create().SetName("Carol").SetEmail("carol@example.com"),
).Save(ctx)
```

### Query (Read)

```go
// Get all users
users, err := client.User.Query().All(ctx)

// Get one user by ID (returns error if not found)
u, err := client.User.Get(ctx, 42)

// Get first match (returns error if none)
u, err := client.User.Query().
    Where(user.Email("alice@example.com")).
    Only(ctx)  // errors if 0 or >1 results

// First (no error if none — returns nil, nil)
u, err := client.User.Query().
    Where(user.NameContains("ali")).
    First(ctx)

// Count
n, err := client.User.Query().
    Where(user.ActiveEQ(true)).
    Count(ctx)

// Check existence
exists, err := client.User.Query().
    Where(user.Email("alice@example.com")).
    Exist(ctx)

// Select only specific fields (reduces data transfer)
names, err := client.User.Query().
    Select(user.FieldName, user.FieldEmail).
    All(ctx)

// Get only IDs
ids, err := client.User.Query().IDs(ctx)
```

### Update

```go
// Update one by ID
u, err := client.User.UpdateOneID(42).
    SetName("Alice Updated").
    SetAge(31).
    ClearBio().          // set nullable field to NULL
    Save(ctx)

// Update one fetched entity
u, err = client.User.UpdateOne(u).
    AddAge(1).           // increment numeric fields
    Save(ctx)

// Bulk update with predicate
n, err := client.User.Update().
    Where(user.RoleEQ("member")).
    SetRole("subscriber").
    Save(ctx)
fmt.Println("updated", n, "users")
```

### Delete

```go
// Delete one by ID
err := client.User.DeleteOneID(42).Exec(ctx)

// Delete fetched entity
err = client.User.DeleteOne(u).Exec(ctx)

// Bulk delete with predicate
n, err := client.User.Delete().
    Where(user.ActiveEQ(false)).
    Exec(ctx)
fmt.Println("deleted", n, "inactive users")
```

### Upsert (insert or update)

```go
// Requires gen.FeatureUpsert in codegen config
err := client.User.Create().
    SetEmail("alice@example.com").
    SetName("Alice").
    OnConflict(
        sql.ConflictColumns(user.FieldEmail),
    ).
    UpdateNewValues(). // update all fields to new values on conflict
    Exec(ctx)
```

---

## 9. Predicates, Filtering, Ordering & Pagination

### Predicate functions (generated per field)

For a `String` field `name`, ent generates:

| Predicate | SQL equivalent |
|---|---|
| `user.NameEQ("Alice")` | `name = 'Alice'` |
| `user.NameNEQ("Alice")` | `name != 'Alice'` |
| `user.NameIn("Alice","Bob")` | `name IN ('Alice','Bob')` |
| `user.NameNotIn("Alice")` | `name NOT IN ('Alice')` |
| `user.NameContains("li")` | `name LIKE '%li%'` |
| `user.NameHasPrefix("Al")` | `name LIKE 'Al%'` |
| `user.NameHasSuffix("ce")` | `name LIKE '%ce'` |
| `user.NameContainsFold("ali")` | case-insensitive LIKE |
| `user.NameEqualFold("alice")` | case-insensitive = |
| `user.NameIsNil()` | `name IS NULL` |
| `user.NameNotNil()` | `name IS NOT NULL` |

For `Int` / numeric fields (`age`):

| Predicate | SQL |
|---|---|
| `user.AgeEQ(30)` | `age = 30` |
| `user.AgeNEQ(30)` | `age != 30` |
| `user.AgeGT(18)` | `age > 18` |
| `user.AgeGTE(18)` | `age >= 18` |
| `user.AgeLT(65)` | `age < 65` |
| `user.AgeLTE(65)` | `age <= 65` |
| `user.AgeIn(25,30,35)` | `age IN (25,30,35)` |

### Combining predicates

```go
import (
    "github.com/you/myblog/ent/predicate"
    "github.com/you/myblog/ent/user"
)

// AND (default when chaining multiple Where calls)
users, err := client.User.Query().
    Where(
        user.AgeGTE(18),
        user.ActiveEQ(true),
    ).All(ctx)

// Explicit AND / OR / NOT
users, err = client.User.Query().
    Where(
        entql.Or(
            user.RoleEQ("admin"),
            user.RoleEQ("moderator"),
        ),
    ).All(ctx)

users, err = client.User.Query().
    Where(
        entql.And(
            user.AgeGTE(18),
            entql.Not(user.NameEQ("banned_user")),
        ),
    ).All(ctx)

// Predicate on related edge (has-edge / has-edge-with)
// Users who have at least one post
users, err = client.User.Query().
    Where(user.HasPosts()).
    All(ctx)

// Users who have a post with a specific title
users, err = client.User.Query().
    Where(
        user.HasPostsWith(post.TitleContains("ent")),
    ).All(ctx)
```

### Ordering

```go
import "github.com/you/myblog/ent/user"

// Order by single field
users, err := client.User.Query().
    Order(ent.Asc(user.FieldCreatedAt)).
    All(ctx)

// Order by multiple fields
users, err = client.User.Query().
    Order(
        ent.Desc(user.FieldScore),
        ent.Asc(user.FieldName),
    ).All(ctx)
```

### Limit, Offset & Pagination

```go
const pageSize = 20

// Page 1
users, err := client.User.Query().
    Order(ent.Asc(user.FieldID)).
    Limit(pageSize).
    Offset(0).
    All(ctx)

// Page 2
users, err = client.User.Query().
    Order(ent.Asc(user.FieldID)).
    Limit(pageSize).
    Offset(pageSize). // skip first 20
    All(ctx)

// Cursor-based pagination (more efficient for large datasets)
// Use the last-seen ID as a cursor
users, err = client.User.Query().
    Where(user.IDGT(lastSeenID)).
    Order(ent.Asc(user.FieldID)).
    Limit(pageSize).
    All(ctx)
```

> For GraphQL Relay-style cursor pagination, use the `entgql` extension which generates `Connection` types automatically.

---

## 10. Eager Loading & Edge Traversal

### WithX — eager loading (JOIN / separate query)

By default, querying a `User` does not load its `posts`. Call `WithPosts()` to eager-load.

```go
// Load user with their posts
u, err := client.User.Query().
    Where(user.IDEQ(42)).
    WithPosts(). // eager load posts
    Only(ctx)

// Access the loaded edge (nil-safe; returns empty slice if not loaded)
for _, p := range u.Edges.Posts {
    fmt.Println(p.Title)
}

// Eager load with filtering/ordering on the edge
u, err = client.User.Query().
    Where(user.IDEQ(42)).
    WithPosts(func(q *ent.PostQuery) {
        q.Where(post.PublishedEQ(true)).
            Order(ent.Desc(post.FieldCreatedAt)).
            Limit(5)
    }).
    Only(ctx)

// Load multiple edges at once
users, err := client.User.Query().
    WithPosts().
    WithProfile().
    All(ctx)
```

### Traversing edges (graph traversal)

ent generates `QueryX()` methods on entities to traverse from one entity to related entities:

```go
// Get all posts by a specific user (traverse from user to posts)
posts, err := u.QueryPosts().
    Where(post.PublishedEQ(true)).
    All(ctx)

// Get the author of a post (traverse the inverse edge)
author, err := p.QueryAuthor().Only(ctx)

// Chain traversals: user → posts → comments
comments, err := u.QueryPosts().
    QueryComments().
    All(ctx)

// Count without loading entities
commentCount, err := u.QueryPosts().QueryComments().Count(ctx)
```

### Querying through edges directly on the client

```go
// All comments on posts by user 42
comments, err := client.Comment.Query().
    Where(
        comment.HasPostWith(
            post.HasAuthorWith(user.IDEQ(42)),
        ),
    ).All(ctx)
```

---

## 11. Transactions

### Basic transaction

```go
func CreateUserWithPost(ctx context.Context, client *ent.Client) error {
    // Start a transaction
    tx, err := client.Tx(ctx)
    if err != nil {
        return fmt.Errorf("starting transaction: %w", err)
    }

    // Create user within the transaction
    u, err := tx.User.Create().
        SetName("Transactional Alice").
        SetEmail("tx-alice@example.com").
        Save(ctx)
    if err != nil {
        // Always rollback on error
        tx.Rollback()
        return fmt.Errorf("creating user: %w", err)
    }

    // Create post within same transaction
    _, err = tx.Post.Create().
        SetTitle("First Post").
        SetAuthor(u).
        Save(ctx)
    if err != nil {
        tx.Rollback()
        return fmt.Errorf("creating post: %w", err)
    }

    // Commit on success
    return tx.Commit()
}
```

### WithTx helper (cleaner, handles rollback automatically)

```go
func WithTx(ctx context.Context, client *ent.Client, fn func(tx *ent.Tx) error) error {
    tx, err := client.Tx(ctx)
    if err != nil {
        return err
    }
    defer func() {
        if v := recover(); v != nil {
            tx.Rollback()
            panic(v) // re-panic after rollback
        }
    }()
    if err := fn(tx); err != nil {
        if rerr := tx.Rollback(); rerr != nil {
            err = fmt.Errorf("%w: rolling back transaction: %v", err, rerr)
        }
        return err
    }
    if err := tx.Commit(); err != nil {
        return fmt.Errorf("committing transaction: %w", err)
    }
    return nil
}

// Usage
err := WithTx(ctx, client, func(tx *ent.Tx) error {
    u, err := tx.User.Create().SetName("Bob").SetEmail("bob@example.com").Save(ctx)
    if err != nil {
        return err
    }
    _, err = tx.Post.Create().SetTitle("Bob's Post").SetAuthor(u).Save(ctx)
    return err
})
```

> **⚡ Version note:** ent v0.14 ships a built-in `client.WithTx(ctx, fn)` method that wraps this pattern — check the docs for the exact signature in your version.

### Accessing the client from within a transaction

The `tx.Client()` method returns an `*ent.Client` scoped to the transaction — useful when passing a client to sub-functions that should participate in the same transaction.

```go
func CreateUser(ctx context.Context, c *ent.Client, name, email string) (*ent.User, error) {
    return c.User.Create().SetName(name).SetEmail(email).Save(ctx)
}

// Works both inside and outside a transaction:
err = WithTx(ctx, client, func(tx *ent.Tx) error {
    _, err := CreateUser(ctx, tx.Client(), "Dave", "dave@example.com")
    return err
})
```

---

## 12. Hooks & Privacy

### Schema Hooks

Schema hooks are defined inside the schema and run on every mutation of that entity type.

```go
// ent/schema/user.go
func (User) Hooks() []ent.Hook {
    return []ent.Hook{
        // Hook runs before every User Create operation
        hook.On(
            func(next ent.Mutator) ent.Mutator {
                return hook.UserFunc(func(ctx context.Context, m *ent.UserMutation) (ent.Value, error) {
                    // Normalize email to lowercase before saving
                    if email, ok := m.Email(); ok {
                        m.SetEmail(strings.ToLower(email))
                    }
                    return next.Mutate(ctx, m)
                })
            },
            ent.OpCreate,
        ),

        // Audit hook: log all updates
        hook.On(
            func(next ent.Mutator) ent.Mutator {
                return hook.UserFunc(func(ctx context.Context, m *ent.UserMutation) (ent.Value, error) {
                    log.Printf("user update: fields=%v", m.Fields())
                    return next.Mutate(ctx, m)
                })
            },
            ent.OpUpdate|ent.OpUpdateOne,
        ),
    }
}
```

### Runtime Hooks (registered on the client)

Runtime hooks are useful for cross-cutting concerns that you don't want embedded in the schema (e.g., per-tenant filtering, request-context logging):

```go
// Register a hook on all Create operations for all entity types
client.Use(func(next ent.Mutator) ent.Mutator {
    return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
        start := time.Now()
        defer func() {
            log.Printf("op=%s type=%s duration=%s", m.Op(), m.Type(), time.Since(start))
        }()
        return next.Mutate(ctx, m)
    })
})

// Register on specific types only
client.User.Use(myUserHook)
```

### Privacy Policies (brief overview)

Requires `gen.FeaturePrivacy` in codegen config.

```go
// ent/schema/post.go
import "entgo.io/ent/entql"
import "github.com/you/myblog/ent/privacy"
import "github.com/you/myblog/rule"

func (Post) Policy() ent.Policy {
    return privacy.Policy{
        Mutation: privacy.MutationPolicy{
            // Only the author can delete their post
            rule.AllowIfAdmin(),
            rule.DenyIfNotAuthor(),
        },
        Query: privacy.QueryPolicy{
            // Everyone can read published posts
            rule.AllowIfPublished(),
            // Authors can always read their own
            rule.AllowIfAuthor(),
            privacy.AlwaysDenyRule(),
        },
    }
}
```

Privacy rules are evaluated in order; first `Allow` or `Deny` wins. This is ent's answer to row-level security at the application layer.

---

## 13. Full Worked Example: Blog

This section builds a complete mini-blog: `User`, `Post`, `Comment`.

### Schema: User

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
        field.String("email").Unique().Match(regexp.MustCompile(`^[^@]+@[^@]+\.[^@]+$`)),
        field.String("bio").Optional().MaxLen(500),
        field.Bool("active").Default(true),
        field.Time("created_at").Immutable().Default(time.Now),
    }
}

func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("posts", Post.Type),
        edge.To("comments", Comment.Type),
    }
}

func (User) Indexes() []ent.Index {
    return []ent.Index{
        index.Fields("email").Unique(),
    }
}
```

### Schema: Post

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
        field.Text("body"),
        field.Bool("published").Default(false),
        field.Time("created_at").Immutable().Default(time.Now),
        field.Time("updated_at").Default(time.Now).UpdateDefault(time.Now),
    }
}

func (Post) Edges() []ent.Edge {
    return []ent.Edge{
        // Post belongs to one User (author)
        edge.From("author", User.Type).
            Ref("posts").
            Unique().
            Required(),

        // Post has many Comments
        edge.To("comments", Comment.Type),
    }
}
```

### Schema: Comment

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
        // Comment belongs to one Post
        edge.From("post", Post.Type).
            Ref("comments").
            Unique().
            Required(),

        // Comment belongs to one User (author)
        edge.From("author", User.Type).
            Ref("comments").
            Unique().
            Required(),
    }
}
```

### Generate the code

```bash
go generate ./ent
```

### main.go — wire it all together

```go
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
    client, err := ent.Open("sqlite3", "file:blog.db?cache=shared&_fk=1")
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()

    ctx := context.Background()

    // Auto-migrate
    if err := client.Schema.Create(ctx); err != nil {
        log.Fatal(err)
    }

    // --- CREATE ---
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
        SetBody("ent is a fantastic Go ORM...").
        SetPublished(true).
        SetAuthor(alice).
        Save(ctx)
    if err != nil {
        log.Fatal(err)
    }

    _, err = client.Comment.Create().
        SetBody("Great post, Alice!").
        SetPost(p).
        SetAuthor(bob).
        Save(ctx)
    if err != nil {
        log.Fatal(err)
    }

    // --- QUERY: published posts with author and comments ---
    posts, err := client.Post.Query().
        Where(post.PublishedEQ(true)).
        WithAuthor().
        WithComments(func(q *ent.CommentQuery) {
            q.WithAuthor().
                Order(ent.Asc("created_at"))
        }).
        Order(ent.Desc(post.FieldCreatedAt)).
        All(ctx)
    if err != nil {
        log.Fatal(err)
    }

    for _, p := range posts {
        fmt.Printf("Post: %s (by %s)\n", p.Title, p.Edges.Author.Name)
        for _, c := range p.Edges.Comments {
            fmt.Printf("  Comment by %s: %s\n", c.Edges.Author.Name, c.Body)
        }
    }

    // --- QUERY: users who have published posts ---
    authors, err := client.User.Query().
        Where(
            user.HasPostsWith(post.PublishedEQ(true)),
        ).
        All(ctx)
    if err != nil {
        log.Fatal(err)
    }
    for _, a := range authors {
        fmt.Println("Author:", a.Name)
    }

    // --- UPDATE ---
    _, err = client.Post.UpdateOne(p).
        SetTitle("Why I love ent (updated)").
        Save(ctx)
    if err != nil {
        log.Fatal(err)
    }

    // --- TRANSACTION: safe user + post creation ---
    err = withTx(ctx, client, func(tx *ent.Tx) error {
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
    })
    if err != nil {
        log.Fatal(err)
    }

    fmt.Println("Done!")
}

func withTx(ctx context.Context, client *ent.Client, fn func(*ent.Tx) error) error {
    tx, err := client.Tx(ctx)
    if err != nil {
        return err
    }
    defer func() {
        if v := recover(); v != nil {
            tx.Rollback()
            panic(v)
        }
    }()
    if err := fn(tx); err != nil {
        tx.Rollback()
        return err
    }
    return tx.Commit()
}
```

### Testing with enttest

```go
package blog_test

import (
    "context"
    "testing"

    _ "github.com/mattn/go-sqlite3"

    "github.com/you/myblog/ent/enttest"
    "github.com/you/myblog/ent/user"
)

func TestCreateUser(t *testing.T) {
    // enttest.Open opens an in-memory SQLite DB, runs migrations, returns client
    client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
    defer client.Close()

    ctx := context.Background()

    u, err := client.User.Create().
        SetName("Test User").
        SetEmail("test@example.com").
        Save(ctx)
    if err != nil {
        t.Fatalf("failed to create user: %v", err)
    }

    // Verify
    got, err := client.User.Query().
        Where(user.Email("test@example.com")).
        Only(ctx)
    if err != nil {
        t.Fatal(err)
    }
    if got.ID != u.ID {
        t.Errorf("got ID %d, want %d", got.ID, u.ID)
    }
}
```

---

## 14. GraphQL, net/http & Gin Integration

### entgql — GraphQL integration

`entgql` is the official ent extension for [GraphQL](https://graphql.org/) via [gqlgen](https://gqlgen.com/).

```bash
go get entgo.io/contrib/entgql
```

**What it provides:**
- Auto-generates GraphQL schema types from ent schemas
- Relay cursor-based connection types (pagination)
- `OrderBy` input types
- `Where` filter input types (maps to ent predicates)
- `Node` interface support

**Setup sketch:**

```go
// ent/entc.go
//go:build ignore
package main

import (
    "log"
    "entgo.io/contrib/entgql"
    "entgo.io/ent/entc"
    "entgo.io/ent/entc/gen"
)

func main() {
    ex, err := entgql.NewExtension(
        entgql.WithSchemaGenerator(),
        entgql.WithSchemaPath("./graph/ent.graphql"),
        entgql.WithConfigPath("./gqlgen.yml"),
        entgql.WithWhereInputs(true),
    )
    if err != nil {
        log.Fatal(err)
    }
    if err := entc.Generate("./schema", &gen.Config{
        Features: []gen.Feature{gen.FeatureVersionedMigration},
    }, entc.Extensions(ex)); err != nil {
        log.Fatal(err)
    }
}
```

After generation, wire the ent client into the gqlgen resolver:

```go
// graph/resolver.go
type Resolver struct {
    client *ent.Client
}

func (r *queryResolver) Users(ctx context.Context, after *entgql.Cursor[int], first *int, ...) (*ent.UserConnection, error) {
    return r.client.User.Query().
        Paginate(ctx, after, first, before, last)
}
```

> **⚡ Version note:** entgql v0.14+ supports federation (Apollo Federation v2) via `entgql.WithFederation()`. Check `entgo.io/contrib` for the current version matching your ent version.

### net/http integration

ent has no HTTP framework dependency — just pass the client around:

```go
package main

import (
    "encoding/json"
    "net/http"

    "github.com/you/myblog/ent"
    "github.com/you/myblog/ent/user"
)

type Server struct {
    client *ent.Client
}

func (s *Server) ListUsers(w http.ResponseWriter, r *http.Request) {
    users, err := s.client.User.Query().
        Where(user.ActiveEQ(true)).
        Order(ent.Asc(user.FieldName)).
        All(r.Context())
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func main() {
    client, _ := ent.Open("sqlite3", "file:blog.db?_fk=1")
    defer client.Close()

    s := &Server{client: client}
    http.HandleFunc("/users", s.ListUsers)
    http.ListenAndServe(":8080", nil)
}
```

### Gin integration

```go
package main

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/you/myblog/ent"
    "github.com/you/myblog/ent/post"
)

func main() {
    client, _ := ent.Open("postgres", "host=localhost dbname=myblog sslmode=disable")
    defer client.Close()

    r := gin.Default()

    // Middleware: attach ent client to context
    r.Use(func(c *gin.Context) {
        c.Set("ent", client)
        c.Next()
    })

    r.GET("/posts", func(c *gin.Context) {
        db := c.MustGet("ent").(*ent.Client)

        posts, err := db.Post.Query().
            Where(post.PublishedEQ(true)).
            WithAuthor().
            Order(ent.Desc(post.FieldCreatedAt)).
            Limit(20).
            All(c.Request.Context())
        if err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
            return
        }
        c.JSON(http.StatusOK, posts)
    })

    r.POST("/posts", func(c *gin.Context) {
        db := c.MustGet("ent").(*ent.Client)

        var body struct {
            Title    string `json:"title"`
            Content  string `json:"body"`
            AuthorID int    `json:"author_id"`
        }
        if err := c.ShouldBindJSON(&body); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
            return
        }

        p, err := db.Post.Create().
            SetTitle(body.Title).
            SetBody(body.Content).
            SetAuthorID(body.AuthorID).
            Save(c.Request.Context())
        if err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
            return
        }
        c.JSON(http.StatusCreated, p)
    })

    r.Run(":8080")
}
```

---

## 15. Tips, Tricks & Gotchas

### Always regenerate after schema changes

```bash
# This is the most important workflow habit with ent.
# After ANY change to a file in ent/schema/, run:
go generate ./ent

# If you skip this, you'll get compile errors (the generated code is stale)
# or, worse, runtime panics if you're using older generated files.
```

### Edge direction confusion

The most common ent beginner mistake:

```
WRONG mental model: "To" means "the FK is on this table."
CORRECT mental model:
  - edge.To("posts", Post.Type) on User → "User OWNS the posts edge"
    → FK lives on the POSTS table (user_id in posts)
  - edge.From("author", User.Type).Ref("posts") on Post → "Post references the User"
    → This is the INVERSE declaration — it doesn't add a new FK
```

Remember: **only one FK is created per relation**, always on the "many" side (for O2M) or on a join table (for M2M). The `From` declaration just gives you the reverse traversal API.

### Nillable vs Optional

```go
// Optional() alone: field is nullable in DB, but Go struct field is a VALUE type (zero value if NULL)
field.String("bio").Optional()
// u.Bio is a string — empty string if DB is NULL. You can't tell if it was NULL or ""

// Optional() + Nillable(): field is nullable AND Go struct field is a POINTER
field.String("bio").Optional().Nillable()
// u.Bio is *string — nil if DB is NULL, &"" if empty string
// This is the correct choice when NULL and empty string are semantically different
```

### The ID field

By default, ent uses an `int` auto-increment `id` primary key. For UUIDs:

```go
import "github.com/google/uuid"

field.UUID("id", uuid.UUID{}).
    Default(uuid.New).
    Immutable(),
```

You must add `github.com/google/uuid` to your `go.mod`. UUID keys are recommended for distributed systems but have performance implications on large indexed tables (random UUIDs fragment B-tree indexes).

### Migration safety in production

```
Auto-migration (client.Schema.Create) is fine for development.
In production:
  1. Use Atlas versioned migrations (atlas migrate diff + atlas migrate apply).
  2. Always review generated SQL before applying.
  3. Test migrations on a staging DB first.
  4. Never use WithDropTable(true) or WithDropColumn(true) in production — data loss!
```

### Don't forget SQLite foreign keys

```go
// SQLite has foreign key enforcement DISABLED by default.
// Always use this DSN flag when using ent with SQLite:
ent.Open("sqlite3", "file:mydb.db?_fk=1")
//                                       ^^^^ THIS IS CRITICAL
// Without it, edge Required() constraints are silently ignored.
```

### Query returns zero results vs error

```go
// .All(ctx) — returns empty slice ([]T{}) + nil error when there are no matches
// .Only(ctx) — returns nil + not-found error when there are no matches (also errors on >1 result)
// .First(ctx) — returns nil + nil when there are no matches (no error!)

// Always check the ent.IsNotFound error type when appropriate:
u, err := client.User.Get(ctx, id)
if ent.IsNotFound(err) {
    http.Error(w, "user not found", http.StatusNotFound)
    return
}
```

### Sensitive fields and JSON

```go
// Fields marked .Sensitive() are excluded from the entity's String() output
// and from default JSON marshaling. Use this for passwords, tokens, secrets.
field.String("password_hash").Sensitive(),
```

### Upsert requires codegen feature flag

```go
// If you try to use .OnConflict() without enabling gen.FeatureUpsert in entc.go,
// the generated code won't have those methods. Add to your config:
Features: []gen.Feature{gen.FeatureUpsert},
// Then re-run go generate ./ent
```

### Avoid N+1 queries

```go
// BAD — N+1 problem: queries users, then queries posts for EACH user
users, _ := client.User.Query().All(ctx)
for _, u := range users {
    posts, _ := u.QueryPosts().All(ctx) // separate query per user!
}

// GOOD — eager load with WithPosts()
users, _ := client.User.Query().
    WithPosts().  // single additional query for all posts, then joined in Go
    All(ctx)
for _, u := range users {
    _ = u.Edges.Posts // already loaded
}
```

### Use `enttest` for all tests — never mock the DB

ent's generated code is tightly coupled to real SQL semantics. Mocking the client is painful and error-prone. Instead, use SQLite in-memory DBs for tests:

```go
client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
// Fast, isolated, auto-cleaned up when the test ends.
```

### Schema comments become SQL comments

```go
field.String("name").Comment("The user's display name — shown in the UI"),
// Generates: `name TEXT NOT NULL -- The user's display name — shown in the UI`
// Atlas picks this up and it appears in migration SQL for documentation.
```

---

## 16. Study Path

Work through these in order. Each step builds on the previous and produces working code you can show.

### Phase 1: Core fundamentals (Week 1)

1. **Read the official "Getting Started" guide** at `entgo.io/docs/getting-started`. Follow it with SQLite so you can skip database setup.
2. **Build: URL Shortener** — Two entities: `Link` (url, slug, click_count) and `Click` (ip, user_agent, created_at). O2M relation. Practice: Create, Query, Update (increment click_count), Delete.
3. **Drills: Field options** — Rebuild the `Link` schema three times adding: validators, default values, a UUID primary key.

### Phase 2: Relations & generation (Week 2)

4. **Build: Task Manager** — Entities: `User`, `Project`, `Task`, `Label`. Relations: User O2M Tasks, Project O2M Tasks, Tasks M2M Labels. Practice every edge type.
5. **Drills: Predicates** — Write 10 different `Where()` clauses. Practice `HasEdgeWith`, `And`, `Or`, `Not`, `Fold` (case-insensitive).
6. **Study: Code generation internals** — Open a generated `_query.go` file and read it. Understand what `entql` generates so you can debug unfamiliar errors.

### Phase 3: Production patterns (Week 3)

7. **Add Atlas migrations** to your Task Manager. Generate migration files, apply them, make a schema change, generate a new migration.
8. **Add Mixins** — Create a `TimestampMixin` and `SoftDeleteMixin`. Apply to all entities. Observe how generated code changes.
9. **Build: Multi-tenant SaaS skeleton** — Add a `Tenant` entity. Add a runtime hook that injects tenant filtering on every query based on a value in `context.Context`.
10. **Add Transactions** — Wrap every "business operation" (create project + first task, complete task + update project status) in a transaction with the `withTx` helper.

### Phase 4: Advanced & integration (Week 4)

11. **Add Hooks** — Schema hook: auto-lowercase email on User create/update. Runtime hook: audit log every mutation to a file.
12. **Build: REST API with Gin** — Wrap your Task Manager in a Gin HTTP API. Practice passing the ent client through Gin middleware and context.
13. **Explore entgql** — Add GraphQL to your blog project. Generate the schema, wire gqlgen resolvers, test cursor pagination with the generated `Connection` types.
14. **Study Privacy** — Enable `gen.FeaturePrivacy`. Write a simple policy for Posts: only the author can update/delete their post. Test it.

### Reference projects to study

| Project | What to learn |
|---|---|
| `github.com/ent/contrib/examples` | Official examples: GraphQL, gRPC, real-world patterns |
| `github.com/hedwigz/entviz` | Visualize your ent schema as a diagram |
| Any open-source Go SaaS on GitHub that uses ent | Real-world schema design decisions |

### Key resources (offline-friendly)

| Resource | URL |
|---|---|
| Official docs | `https://entgo.io/docs/getting-started` |
| entgo GitHub | `https://github.com/ent/ent` |
| ent contrib (GraphQL, gRPC, etc.) | `https://github.com/ent/contrib` |
| Atlas migration docs | `https://atlasgo.io/docs` |
| ent Discord | `https://discord.gg/qZmPgTE6RX` |

---

> **Final note:** ent's superpower is that your schema IS your documentation, your migration source of truth, and your query API — all in one place, all in Go, all checked by the compiler. Invest time up front in getting your schema right: it pays off every day you don't write raw SQL or fight type assertions.
