# Prisma ORM — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I've never used an ORM" to "I design schemas, write efficient type-safe queries, run safe migrations, and ship Prisma to production" — without an internet connection. Every concept comes with prose that explains *what it is*, *the logic / why it works this way*, *what it's for and when you reach for it*, *how to use it*, the key options, best practices, and security recommendations — followed by heavily-commented, runnable code. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Prisma 6** (current through 2026) running on Node.js 20/22+ and TypeScript 5.x. Things worth knowing about modern Prisma:
> - **The Rust-free client direction.** Historically the Prisma Client shipped with a Rust **query engine** binary that ran as a sidecar process and did the heavy lifting of turning your JS calls into SQL. Prisma 6 is steadily moving this work *into TypeScript* (the new `prisma-client` generator + driver adapters), which removes the native binary, slims deployments (smaller serverless bundles, no engine download in CI), improves edge/serverless compatibility, and makes cold starts faster. The classic `prisma-client-js` generator (with the engine) still works and remains the default; the Rust-free `queryCompiler` path is opt-in and maturing. Where the choice matters it is flagged with **⚡ Version note**.
> - **Driver adapters are stable** — you can plug Prisma into a native driver (`pg`, `@neondatabase/serverless`, `better-sqlite3`, etc.) instead of Prisma's bundled connector, which is what makes the Rust-free path and edge runtimes possible.
> - `createManyAndReturn`, `uuid(7)` (sortable UUIDs), `omit` at query level, and typed SQL (`prisma generate --sql`) are all assumed available.
>
> Confirm exact APIs at prisma.io/docs. Cross-references: this guide pairs with **POSTGRESQL_GUIDE.md** (the engine underneath), **RELATIONAL_DB_DESIGN_GUIDE.md** (normalization, keys, indexing theory), **NESTJS_GUIDE.md** (using Prisma as an injectable provider), and **NEXTJS_16_GUIDE.md** (Server Components, Route Handlers, the singleton).

---

## Table of Contents

1. [What an ORM Is & Why/When to Use Prisma](#1-what-an-orm-is--whywhen-to-use-prisma) **[B]**
2. [Setup & Initialization](#2-setup--initialization) **[B]**
3. [The Schema File](#3-the-schema-file) **[B/I]**
4. [Relations In Depth](#4-relations-in-depth) **[I]**
5. [Migrations & the Shadow Database](#5-migrations--the-shadow-database) **[I]**
6. [Prisma Client CRUD](#6-prisma-client-crud) **[B/I]**
7. [Querying: Filters, Select, Include, Pagination](#7-querying-filters-select-include-pagination) **[I]**
8. [Nested Writes](#8-nested-writes) **[I]**
9. [Aggregations, GroupBy & Count](#9-aggregations-groupby--count) **[I]**
10. [Transactions](#10-transactions) **[I/A]**
11. [Raw Queries & Injection Safety](#11-raw-queries--injection-safety) **[A]**
12. [The N+1 Problem & Relation Loading](#12-the-n1-problem--relation-loading) **[I/A]**
13. [Client Extensions (and Legacy Middleware)](#13-client-extensions-and-legacy-middleware) **[A]**
14. [Seeding the Database](#14-seeding-the-database) **[I]**
15. [Generated Types, Type Safety & Error Handling](#15-generated-types-type-safety--error-handling) **[I/A]**
16. [Prisma in Next.js, NestJS & Serverless](#16-prisma-in-nextjs-nestjs--serverless) **[A]**
17. [Connection Pooling & Performance](#17-connection-pooling--performance) **[A]**
18. [Prisma Studio](#18-prisma-studio) **[B]**
19. [Security, Best Practices & Gotchas](#19-security-best-practices--gotchas) **[I/A]**
20. [Study Path & Build-to-Learn Projects](#20-study-path--build-to-learn-projects)

---

## 1. What an ORM Is & Why/When to Use Prisma

### 1.1 What an ORM actually is **[B]**

An **ORM** — Object-Relational Mapper — is a library that lets you talk to a relational database using the data structures of your programming language (objects, arrays, function calls) instead of writing SQL strings by hand. The "object-relational" part names the gap it bridges: your code thinks in **objects** (a `User` with an `email` and a list of `posts`), while the database thinks in **relations** (rows in a `users` table joined to rows in a `posts` table by a foreign key). The ORM translates between the two worlds: it turns `prisma.user.findMany()` into `SELECT * FROM users`, hydrates the rows into typed JavaScript objects, and turns `user.posts` into a JOIN or a follow-up query.

The reason ORMs exist is that hand-writing SQL across a whole application is repetitive and error-prone. You write the same `SELECT … WHERE id = …` a hundred times, you forget to escape a value and open a SQL-injection hole, you rename a column and miss three queries that still reference the old name, and nothing catches it until production. An ORM centralizes that boilerplate, parameterizes values safely by default, and — in Prisma's case especially — gives you compile-time type checking so a renamed column becomes a red squiggle in your editor instead of a 500 at 2 a.m.

### 1.2 What makes Prisma different **[B]**

Most ORMs are *code-first*: you describe tables as classes with decorators (TypeORM) or as model objects (Sequelize), and the library derives the database shape from your code. Prisma is **schema-first**: you describe your data in a dedicated, human-readable file — `prisma/schema.prisma` — written in **Prisma Schema Language (PSL)**. That single file is the source of truth. From it, Prisma generates two things:

1. **Prisma Migrate** reads the schema, diffs it against the database, and writes SQL migration files that evolve the database to match.
2. **Prisma Client** is *code-generated* from the schema into `node_modules` — a fully-typed query builder where every model, field, filter, and relation is a real TypeScript type. There is no `any`. Autocomplete knows your columns. If you `select` three fields, the return type has exactly those three fields and nothing else.

This generated-client approach is Prisma's signature advantage. With Sequelize or a query builder you frequently cast or hope; with Prisma the types are *derived from your actual schema*, so the editor catches mismatches before you run anything.

| Part | What it does | CLI command |
|---|---|---|
| **Prisma Client** | Auto-generated, type-safe query builder you import in code | `npx prisma generate` |
| **Prisma Migrate** | Diffs the schema, writes & applies SQL migration files | `npx prisma migrate dev` |
| **Prisma Studio** | Visual browser/editor for your database rows | `npx prisma studio` |

### 1.3 When to use Prisma vs raw SQL vs a query builder vs `pg` **[I]**

There is no universally correct data-access tool; each trades convenience for control. Pick deliberately.

| Approach | What it is | Strengths | Weaknesses | Reach for it when… |
|---|---|---|---|---|
| **Raw SQL via a driver** (`pg`, `mysql2`) | You write SQL strings and run them through the database driver directly | Maximum control, every DB feature available, zero abstraction overhead, smallest dependency | No type safety on results (you cast), manual parameterization discipline, lots of boilerplate, no migrations | You need an exotic query the ORM can't express, or you're writing a tiny script |
| **Query builder** (Knex, Kysely) | Composable functions that build SQL; you still think in SQL terms | Type-safe (Kysely especially), composable, close to SQL so few surprises | You still hand-model the schema; less ergonomic for deep relations and nested writes | You want type safety *and* SQL-level control, and you're comfortable thinking in JOINs |
| **Prisma (ORM)** | Schema-first, generated typed client + migrations | Best-in-class types, ergonomic relations & nested writes, integrated migrations, great DX, Studio | Generated query language abstracts SQL (a learning curve of its own); very complex analytical SQL still needs raw escape hatches; extra build step | Application development where you model entities and relations and want safety + speed of iteration — the common case for web/API backends |
| **Traditional ORM** (TypeORM, Sequelize) | Code-first model classes | Mature ecosystems, active-record style familiar to some | Weaker types, runtime surprises, heavier "magic" | Legacy projects already invested in them |

**The honest trade-off with Prisma:** you gain enormous safety and speed for the 95% of queries that are CRUD-with-relations, and you accept that the remaining 5% — heavy reporting, window functions, recursive CTEs — you'll drop to **`$queryRaw`** (§11) or typed SQL. That is a healthy split. Prisma deliberately does *not* try to be a full SQL replacement; it gives you a first-class raw escape hatch precisely because it knows where its abstraction ends.

> **Best practice:** choose Prisma for the application layer, learn enough raw SQL (see POSTGRESQL_GUIDE.md) to drop down when needed, and understand relational design (RELATIONAL_DB_DESIGN_GUIDE.md) so your schema is sound regardless of the tool.

### 1.4 Supported databases **[B]**

| Database | Prisma `provider` | Notes |
|---|---|---|
| PostgreSQL | `postgresql` | Full feature support; recommended for production |
| MySQL / MariaDB | `mysql` | Full feature support |
| SQLite | `sqlite` | Zero-setup local dev / embedded apps — ideal for learning |
| Microsoft SQL Server | `sqlserver` | Full feature support |
| CockroachDB | `cockroachdb` | PostgreSQL-compatible distributed DB |
| MongoDB | `mongodb` | Document DB; uses `db push` (no SQL migrations) and `ObjectId` |
| PlanetScale / serverless MySQL | `mysql` | Use `relationMode = "prisma"` (no FK constraints at DB level) |

### 1.5 How the pieces fit together

```
prisma/schema.prisma   ← you write this (single source of truth)
       │
       ├── npx prisma migrate dev   → diff schema vs DB → write SQL migration → apply → regenerate client
       ├── npx prisma generate      → (re)generate node_modules/@prisma/client from the schema
       └── npx prisma studio        → open the GUI at localhost:5555

import { PrismaClient } from "@prisma/client"   ← fully typed FROM your schema
const prisma = new PrismaClient()
```

⚡ **Version note (Prisma 6):** the newer generator is `prisma-client` (Rust-free, generates into a folder you choose), distinct from the classic `prisma-client-js` (ships the Rust query engine, generates into `node_modules`). Both are valid in Prisma 6. This guide uses `prisma-client-js` in most examples because it is still the default and most widely deployed; the Rust-free variant is covered in §2 and §17.

---

## 2. Setup & Initialization

### 2.1 The mental model first

Before commands: Prisma has a **CLI** (the `prisma` package — a dev dependency, used at build/dev time for migrations and code generation) and a **runtime client** (the `@prisma/client` package — a real dependency your app imports). You install both. You write the schema, you run `migrate`/`generate`, and your app imports the generated client. Keep that separation in mind and the workflow stops feeling magical.

### 2.2 Install **[B]**

```bash
# CLI as a dev dependency; generated client as a runtime dependency
npm install prisma --save-dev
npm install @prisma/client

# pnpm / yarn equivalents
pnpm add -D prisma && pnpm add @prisma/client
yarn add -D prisma && yarn add @prisma/client
```

### 2.3 Initialize **[B]**

`prisma init` scaffolds the folder structure and an `.env` file. Choosing the provider up front writes the right `datasource` block so you don't edit it manually.

```bash
npx prisma init                                  # defaults to postgresql
npx prisma init --datasource-provider sqlite     # great for learning — no server needed
npx prisma init --datasource-provider postgresql
npx prisma init --datasource-provider mysql
```

It creates:

```
your-project/
├── prisma/
│   └── schema.prisma    ← define all models here
├── .env                 ← DATABASE_URL goes here (MUST be git-ignored)
└── node_modules/
    └── @prisma/client/  ← generated after `prisma generate`
```

### 2.4 Configure `DATABASE_URL` **[B]**

The connection string lives in `.env`, read by the schema's `env("DATABASE_URL")`. **Why an env var and not a literal in the schema?** Because the URL contains a password and differs per environment (dev/staging/prod). Hard-coding it would leak secrets into git and lock you to one database. The `.env` file is git-ignored; production injects the value through the platform's secret manager.

```bash
# PostgreSQL
DATABASE_URL="postgresql://USER:PASSWORD@HOST:5432/DBNAME?schema=public"

# SQLite (file-based — perfect for tutorials and tests)
DATABASE_URL="file:./dev.db"

# MySQL
DATABASE_URL="mysql://USER:PASSWORD@HOST:3306/DBNAME"

# MongoDB
DATABASE_URL="mongodb+srv://USER:PASSWORD@cluster.mongodb.net/DBNAME"
```

> **Security recommendation:** never commit `.env`. Confirm `.env` is in `.gitignore` (Prisma's `init` adds it). If a password contains URL-special characters, percent-encode them: `@`→`%40`, `#`→`%23`, `:`→`%3A`, `/`→`%2F`. Use the **least-privileged** database user the app actually needs — the app account rarely needs `CREATE DATABASE` or superuser rights. Use a *separate, more-privileged* user for migrations (it needs DDL) than for the running app (which usually needs only DML). See POSTGRESQL_GUIDE.md for role/grant setup.

### 2.5 Generate the client **[B]**

Run this after *every* schema change. It rebuilds the typed client so your editor and runtime match the schema. Forgetting it is the #1 "why doesn't my new field exist?" beginner stumble.

```bash
npx prisma generate
```

> **Best practice:** add `"postinstall": "prisma generate"` to `package.json` scripts so the client is regenerated automatically after `npm install` — essential in CI and on fresh clones where `node_modules` (and the generated client inside it) don't exist yet.

### 2.6 The Rust-free / driver-adapter setup **[A]**

⚡ **Version note (Prisma 6 — the Rust-free direction):** the traditional client downloads a platform-specific Rust **query engine** binary. That binary is what historically made serverless bundles large and edge runtimes awkward. The modern path uses a **driver adapter**: Prisma's query *compiler* (TypeScript) produces SQL and a thin native driver (like `pg`) executes it — no Rust engine. This is the direction Prisma is heading.

```prisma
// schema.prisma — opt into the Rust-free query compiler + driver adapters
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["driverAdapters", "queryCompiler"] // names evolve; check docs for your version
  // The newer generator can also target a custom output dir:
  // provider = "prisma-client"
  // output   = "../src/generated/prisma"
}
```

```typescript
// db.ts — instantiate with a driver adapter instead of Prisma's bundled connector
import { PrismaClient } from "@prisma/client"
import { PrismaPg } from "@prisma/adapter-pg"   // npm i @prisma/adapter-pg pg

const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL })
export const prisma = new PrismaClient({ adapter })
// Same query API as always; the difference is purely how queries reach the DB.
```

> **When to use it:** edge runtimes (Cloudflare Workers, Vercel Edge), tight serverless bundles, or when you want to share one connection pool with other code that uses `pg` directly. **When to skip it:** a plain long-running Node server on the default generator works fine and is the most documented path — don't add adapter complexity you don't need.

---

## 3. The Schema File

The `prisma/schema.prisma` file is written in **Prisma Schema Language (PSL)**, a small declarative language. It has three core block types: `datasource` (where/what the database is), `generator` (what artifacts Prisma emits), and `model` (a table). Plus `enum` blocks. PSL is intentionally readable: a model looks like a struct, attributes start with `@`, and block-level attributes start with `@@`.

### 3.1 The logic of schema-first

Why model in a separate file rather than in code? Because the schema is consumed by *two* tools — Migrate (to shape the DB) and the client generator (to shape the types) — and by *humans* reviewing changes. A diff to `schema.prisma` in a pull request reads like a sentence: "added a `published` boolean to `Post`." That clarity is the payoff. You think about *data shape* in one focused place, then let Prisma propagate it outward.

### 3.2 Full annotated example **[B/I]**

```prisma
// prisma/schema.prisma

// ── 1. GENERATOR ────────────────────────────────────────────────────────────
generator client {
  provider = "prisma-client-js"   // generate the typed client into @prisma/client
  // previewFeatures = ["fullTextSearch"]   // opt into not-yet-stable features
}

// ── 2. DATASOURCE ───────────────────────────────────────────────────────────
datasource db {
  provider = "postgresql"          // postgresql | mysql | sqlite | mongodb | sqlserver | cockroachdb
  url      = env("DATABASE_URL")   // read from .env — never hard-code secrets
  // directUrl = env("DIRECT_URL") // bypass a pooler for migrations (see §17)
}

// ── 3. ENUMS ────────────────────────────────────────────────────────────────
// An enum is a fixed set of allowed string values, enforced by the DB.
// Use enums for closed categories (role, status) — it documents intent AND
// prevents invalid values at the database level (a CHECK-like guarantee).
enum Role {
  USER
  ADMIN
  MODERATOR
}

enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}

// ── 4. MODELS ───────────────────────────────────────────────────────────────
model User {
  // --- Scalar fields (real columns) ---
  id        Int      @id @default(autoincrement())   // primary key, DB auto-increments
  // Alternatives for the PK (see §19 for the trade-offs):
  // id     String   @id @default(cuid())            // collision-resistant, URL-safe
  // id     String   @id @default(uuid(7))           // sortable UUID (Prisma 5.8+)

  email     String   @unique                         // UNIQUE constraint in the DB
  username  String   @unique @db.VarChar(50)         // native type modifier → VARCHAR(50)
  name      String?                                  // ? = optional / nullable column
  bio       String?  @db.Text                        // map to TEXT (unbounded length)
  age       Int?
  score     Float    @default(0.0)
  isActive  Boolean  @default(true)
  role      Role     @default(USER)                  // enum-typed column

  createdAt DateTime @default(now())                 // set once, at insert time
  updatedAt DateTime @updatedAt                      // Prisma sets it on every update

  // --- Relation fields (NO column in the DB; they exist only in the client) ---
  posts     Post[]                                   // one-to-many (User → many Post)
  profile   Profile?                                 // one-to-one (optional side)
  comments  Comment[]

  @@map("users")                                     // physical table name is "users"
}

model Profile {
  id        Int     @id @default(autoincrement())
  bio       String?
  avatarUrl String?

  // One-to-one: the foreign key lives on THIS side.
  user      User    @relation(fields: [userId], references: [id])
  userId    Int     @unique                          // @unique on the FK enforces 1-to-1

  @@map("profiles")
}

model Post {
  id          Int        @id @default(autoincrement())
  title       String
  content     String?    @db.Text
  published   Boolean    @default(false)
  status      PostStatus @default(DRAFT)
  viewCount   Int        @default(0)

  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt

  // Foreign key back to the author. onDelete: Cascade → deleting a user
  // deletes their posts. (See §4.7 for referential actions.)
  author      User       @relation(fields: [authorId], references: [id], onDelete: Cascade)
  authorId    Int

  tags        Tag[]                                  // implicit many-to-many
  comments    Comment[]                              // one-to-many

  @@unique([title, authorId])                        // one author can't reuse a title
  @@index([status, createdAt])                       // speeds up "latest published" queries
  @@map("posts")
}

model Tag {
  id    Int    @id @default(autoincrement())
  name  String @unique
  posts Post[]                                       // implicit M:N (Prisma owns join table)

  @@map("tags")
}

model Comment {
  id        Int      @id @default(autoincrement())
  content   String
  createdAt DateTime @default(now())

  author    User     @relation(fields: [authorId], references: [id])
  authorId  Int
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  postId    Int

  @@map("comments")
}
```

### 3.3 Scalar types reference **[B]**

A **scalar** is a single-value field (not a relation). Prisma maps each PSL scalar to a TypeScript type and a database column type.

| PSL Type | TypeScript Type | Postgres column | Notes |
|---|---|---|---|
| `String` | `string` | `TEXT` / `VARCHAR` | use `@db.VarChar(n)` to cap length |
| `Boolean` | `boolean` | `BOOLEAN` | |
| `Int` | `number` | `INTEGER` | 32-bit |
| `BigInt` | `bigint` | `BIGINT` | values beyond 2^53; serializes as `bigint` |
| `Float` | `number` | `DOUBLE PRECISION` | **never** for money (§19) |
| `Decimal` | `Decimal` (Prisma class) | `DECIMAL` | exact; use for money |
| `DateTime` | `Date` | `TIMESTAMP` | stored UTC; `@db.Date` for date-only |
| `Json` | `JsonValue` | `JSONB` | arbitrary structured data |
| `Bytes` | `Uint8Array` | `BYTEA` | binary blobs |

### 3.4 Field modifiers **[B]**

| Modifier | Meaning | Example |
|---|---|---|
| `?` | Optional / nullable | `name String?` |
| `[]` | List / array (scalar list or relation) | `posts Post[]`, `tags String[]` |
| (none) | Required (NOT NULL) | `email String` |

> **The logic of `?`:** an optional field maps to a nullable column. In the generated client the type becomes `string | null`. This forces you to handle the null case in TypeScript — a feature, not friction. Required fields can never be `null`, so the DB rejects bad inserts and the types reflect that.

### 3.5 Field attributes (`@`) **[B/I]**

| Attribute | What it does | Example |
|---|---|---|
| `@id` | Marks the primary key | `id Int @id` |
| `@default(val)` | Default when omitted on insert | `@default(now())`, `@default(0)`, `@default(uuid())` |
| `@unique` | UNIQUE constraint (also makes the field usable in `findUnique`) | `email String @unique` |
| `@relation(...)` | Defines a relation's FK & references | `@relation(fields: [userId], references: [id])` |
| `@map("col")` | Different physical column name | `firstName String @map("first_name")` |
| `@updatedAt` | Prisma auto-sets to `now()` on every update | `updatedAt DateTime @updatedAt` |
| `@db.X` | Native database type | `@db.VarChar(50)`, `@db.Decimal(10,2)`, `@db.Text` |
| `@ignore` | Exclude the field from the generated client | `legacy String @ignore` |

### 3.6 Block attributes (`@@`) **[I]**

These apply to the whole model, not a single field.

| Attribute | What it does | Example |
|---|---|---|
| `@@id([a, b])` | Composite primary key | `@@id([userId, postId])` |
| `@@unique([a, b])` | Compound unique constraint | `@@unique([title, authorId])` |
| `@@index([a, b])` | Compound index (performance) | `@@index([status, createdAt])` |
| `@@map("table")` | Different physical table name | `@@map("users")` |
| `@@ignore` | Exclude the whole model from the client | `@@ignore` |

> **Best practice — naming convention:** PSL convention is `PascalCase` models and `camelCase` fields; SQL convention is `snake_case` tables/columns. Use `@map`/`@@map` to keep idiomatic names on both sides. Set them once and your SQL DBA and your TypeScript both stay happy.

> **Best practice — index what you filter/sort.** Add `@@index` on columns that appear frequently in `where` and `orderBy`. An index on `(status, createdAt)` makes "latest published posts" fast. Indexes cost write time and disk, so don't blanket-index everything — see RELATIONAL_DB_DESIGN_GUIDE.md for the theory of selectivity and composite-index column order.

### 3.7 Multi-file schemas & validation **[I]**

⚡ Newer Prisma supports splitting the schema across multiple `.prisma` files in a `prisma/schema/` folder (handy for large apps). Useful CLI checks:

```bash
npx prisma format     # auto-format & align the schema (run before committing)
npx prisma validate   # validate without generating — fast feedback in CI
```

---

## 4. Relations In Depth

Relations are where ORMs earn their keep, and where Prisma's design is most worth understanding. **The single most important idea:** in PSL a relation has *two* halves — a **relation field** (a virtual field of the related model's type, e.g. `posts Post[]`, which has **no column** in the database) and a **scalar foreign-key field** (a real column, e.g. `authorId Int`). The `@relation` attribute wires them together by naming which scalar field holds the FK and which column it `references`.

### 4.1 Why two fields? **[I]**

The database only knows about the FK column (`authorId`). It has no concept of "the author object" or "the list of posts." Prisma adds the relation fields purely for *your* ergonomics — so you can write `post.author` and `user.posts` and have Prisma turn those into JOINs or follow-up queries. Keeping them separate also lets you read or set the FK directly (`authorId: 5`) when you already have the id, *or* connect by object when you don't. Understanding "virtual relation field vs real scalar FK" dissolves most relation confusion.

### 4.2 One-to-One **[I]**

One `User` has at most one `Profile`. The FK lives on the side you consider the "child"/owned record (`Profile`). The `@unique` on `userId` is what *makes* it one-to-one — without it, it would be one-to-many.

```prisma
model User {
  id      Int      @id @default(autoincrement())
  profile Profile?                                   // virtual; optional side has no FK
}

model Profile {
  id     Int  @id @default(autoincrement())
  user   User @relation(fields: [userId], references: [id])
  userId Int  @unique                                // UNIQUE → at most one profile per user
}
```

### 4.3 One-to-Many **[I]**

One `User` writes many `Post`s. The FK (`authorId`) lives on the **many** side. This is the most common relation. The "one" side gets a list relation field (`posts Post[]`); the "many" side gets the FK plus a singular relation field (`author User`).

```prisma
model User {
  id    Int    @id @default(autoincrement())
  posts Post[]                                       // virtual list, no column
}

model Post {
  id       Int  @id @default(autoincrement())
  author   User @relation(fields: [authorId], references: [id])
  authorId Int                                       // the real FK column
}
```

### 4.4 Many-to-Many — implicit join table **[I]**

When a `Post` has many `Tag`s and a `Tag` has many `Post`s, you need a join table. **Implicit** M:N lets Prisma create and manage that hidden table for you (named `_PostToTag`). Use it when the relationship itself carries **no extra data**.

```prisma
model Post {
  id   Int   @id @default(autoincrement())
  tags Tag[]
}

model Tag {
  id    Int    @id @default(autoincrement())
  name  String @unique
  posts Post[]
}
// Prisma silently maintains a "_PostToTag" join table. You connect/disconnect
// via the relation fields; you never touch the join table directly.
```

### 4.5 Many-to-Many — explicit join table **[I/A]**

The moment the relationship needs its **own attributes** — a `role` on membership, a `joinedAt` timestamp, a quantity on an order line — you must model the join table yourself as a real model. This is the more common case in real apps than beginners expect.

```prisma
model User {
  id    Int        @id @default(autoincrement())
  teams Membership[]                                 // points at the JOIN model, not Team
}

model Team {
  id      Int        @id @default(autoincrement())
  name    String
  members Membership[]
}

// Explicit join model — a first-class table you can query and extend.
model Membership {
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  userId    Int
  team      Team     @relation(fields: [teamId], references: [id], onDelete: Cascade)
  teamId    Int
  role      String   @default("MEMBER")             // ← the extra data the join carries
  joinedAt  DateTime @default(now())

  @@id([userId, teamId])                            // composite PK = one membership per (user,team)
}
```

> **Best practice:** start implicit; refactor to explicit the day you need a column on the relationship. Going implicit→explicit is a migration; plan for it if a "tag with weight" or "membership with role" is on the horizon.

### 4.6 Self-relations **[I/A]**

A model that references *itself* — managers/reports, comment threads, category trees. You disambiguate the two relation fields with a **named relation** in `@relation("…")` because the model appears on both sides.

```prisma
model Employee {
  id        Int        @id @default(autoincrement())
  name      String
  managerId Int?
  manager   Employee?  @relation("OrgChart", fields: [managerId], references: [id])
  reports   Employee[] @relation("OrgChart")        // same name ties the two ends together
}

model Comment {
  id       Int       @id @default(autoincrement())
  content  String
  parentId Int?
  parent   Comment?  @relation("Thread", fields: [parentId], references: [id])
  replies  Comment[] @relation("Thread")
}
```

### 4.7 Referential actions (cascade behaviour) **[I]**

`onDelete`/`onUpdate` tell the database what to do to *child* rows when a parent is deleted or its key changes. This is a real DB-level constraint (`ON DELETE …`), so it's enforced even outside Prisma. Choosing correctly prevents both orphaned rows and accidental mass deletion.

```prisma
model Post {
  author   User @relation(
    fields: [authorId],
    references: [id],
    onDelete: Cascade,     // delete a user → delete their posts
    onUpdate: Cascade
  )
  authorId Int
}
```

| Action | Behaviour | Use when |
|---|---|---|
| `Cascade` | Delete/update children too | Children are meaningless without the parent (a post's comments) |
| `Restrict` | Error if children exist | You want to force explicit cleanup first |
| `SetNull` | Set the child FK to null (FK must be optional) | Child can outlive parent (post survives author deletion) |
| `SetDefault` | Set FK to its `@default` | Rare; "reassign to a default owner" |
| `NoAction` | Defer to the DB (usually like Restrict) | You manage integrity another way |

> **Security/safety recommendation:** be deliberate with `Cascade`. A cascade on a top-level entity can wipe out huge subtrees with one `delete`. For user-facing "delete account" flows, prefer **soft deletes** (§19) so data is recoverable and audits survive.

### 4.8 `relationMode` (FKs in the app vs the DB) **[A]**

```prisma
datasource db {
  provider     = "mysql"
  url          = env("DATABASE_URL")
  relationMode = "prisma"   // Prisma emulates referential integrity; no DB-level FK constraints
}
```

⚡ Some platforms (PlanetScale, certain Vitess/serverless setups) don't allow foreign-key constraints. `relationMode = "prisma"` makes Prisma enforce relations in the client layer instead and **requires you to add `@@index` on FK columns manually** (the DB won't auto-index them). Default (`"foreignKeys"`) is preferred whenever the DB supports real FKs because the database enforcing integrity is stronger than an app emulating it.

---

## 5. Migrations & the Shadow Database

A **migration** is a versioned, ordered SQL script that evolves your database schema. Prisma Migrate's job is to look at your `schema.prisma`, compare it to the database's current state, and emit the SQL needed to reconcile them — then record that it ran. The set of migration files in `prisma/migrations/` is the *history* of your schema, checked into git, replayed in order on any environment.

### 5.1 Why migrations exist (the logic) **[I]**

Without migrations, two problems bite you. First, **drift**: your laptop's database, your teammate's, and production all diverge because changes were applied ad hoc. Second, **irreversibility of memory**: six months later nobody remembers why a column exists. Migrations solve both — they are a reproducible, reviewable, ordered record. `migrate deploy` replays exactly the same SQL everywhere, so all environments converge to the identical schema.

### 5.2 Development workflow: `migrate dev` **[I]**

```bash
# Diff schema vs DB, generate a migration file, apply it, and regenerate the client
npx prisma migrate dev --name add_user_role
```

What `migrate dev` does, step by step:

1. Checks for **drift** (did someone change the DB outside Prisma?) and that all prior migrations are applied.
2. Uses a **shadow database** (next section) to compute the precise diff between your schema and the migration history.
3. Writes `prisma/migrations/<timestamp>_add_user_role/migration.sql`.
4. Applies that SQL to your development database.
5. Runs `prisma generate` so the client matches.

> **Best practice:** always pass `--name` — the name becomes the folder name and is your changelog. `add_user_role` reads far better in `git log` than an unnamed timestamp.

### 5.3 The shadow database — what & why **[A]**

The **shadow database** is a temporary, throwaway database Prisma creates and drops during `migrate dev`. Why does it need one? To compute a *trustworthy* diff. Prisma replays your existing migration history into the empty shadow DB to reconstruct "what the schema should be according to history," then diffs *that* against your edited `schema.prisma`. This detects two things your dev DB alone can't: (1) **schema drift** — your dev DB no longer matches what the migrations say it should — and (2) **failed/edited migrations**. The shadow DB is purely a dev-time tool; it is never used by `migrate deploy`.

```bash
# If your DB user can't create databases, point Prisma at a shadow DB explicitly:
# datasource db { shadowDatabaseUrl = env("SHADOW_DATABASE_URL") }
```

> **Permissions note:** `migrate dev` needs a user that can create/drop the shadow database. Cloud providers that forbid this (some serverless MySQL) require setting `shadowDatabaseUrl` to a second database the user *can* manage.

### 5.4 Production deployment: `migrate deploy` **[I]**

```bash
# Apply all pending migrations in order. Never creates or edits migrations.
# Idempotent and safe to run in CI/CD on every deploy.
npx prisma migrate deploy
```

`migrate deploy` is the *only* migration command you run in production. It does **not** use a shadow DB, does **not** diff, does **not** prompt — it simply applies any migration files not yet recorded as applied. This is why your migration files must be committed: deploy replays what `migrate dev` authored.

### 5.5 Prototyping: `db push` (no migration files) **[I]**

```bash
# Shove the current schema straight into the DB. No migration file is created.
npx prisma db push
```

`db push` skips the migration history entirely — it diffs schema→DB and applies the change directly. **When to use it:** very early prototyping when the schema changes every five minutes and you don't want a pile of throwaway migrations, or MongoDB (which has no SQL migrations). **When NOT to use it:** anything headed for production, or a shared/team DB — because there's no recorded history, no review artifact, and `db push` may **silently drop columns/data** to make the DB match. Treat it as scaffolding you'll replace with real migrations before the schema stabilizes.

```bash
npx prisma db push --accept-data-loss   # explicitly allow destructive changes (be careful)
```

### 5.6 Resetting & inspecting **[I]**

```bash
# DROP everything, re-apply ALL migrations from scratch, then run the seed. DEV ONLY.
npx prisma migrate reset

# Reverse-engineer an EXISTING database into schema.prisma (introspection)
npx prisma db pull

# Check status: which migrations are applied / pending / failed
npx prisma migrate status
```

> **Safety recommendation:** `migrate reset` and `db push --accept-data-loss` are **destructive**. Guard them — never wire them into a script that could run against production. A good habit: production credentials should belong to a DB user that *cannot* drop tables, so a misfired reset simply errors out.

### 5.7 Migration file structure & custom SQL **[I/A]**

```
prisma/
└── migrations/
    ├── 20260101120000_init/
    │   └── migration.sql        ← the SQL Prisma generated (editable before applying)
    ├── 20260110150000_add_user_role/
    │   └── migration.sql
    └── migration_lock.toml      ← records the provider; commit it, never edit it
```

Sometimes Prisma's generated SQL isn't enough — you need a data backfill, a custom index type (GIN/GiST), a trigger, or a careful column rename that preserves data. Use `--create-only` to author the file *without* applying it, edit the SQL, then apply.

```bash
# 1. Generate the migration file but DON'T run it yet
npx prisma migrate dev --name rename_fullname --create-only

# 2. Edit prisma/migrations/<ts>_rename_fullname/migration.sql, e.g.:
#    -- replace Prisma's DROP+ADD (which loses data) with a non-destructive rename:
#    ALTER TABLE "users" RENAME COLUMN "fullName" TO "name";

# 3. Apply the edited migration
npx prisma migrate dev
```

> **The rename gotcha:** when you rename a field in the schema, Prisma can't tell rename from drop+add, so its default SQL **drops the old column (losing data) and adds a new one**. For any rename on a table with real data, use `--create-only` and hand-edit the SQL to a `RENAME COLUMN`. This is one of the most important production-safety habits in Prisma.

### 5.8 Baselining an existing database **[A]**

To adopt Prisma Migrate on a database that already has data (and was never managed by Prisma), you "baseline": generate an `init` migration representing the current schema and mark it already-applied so deploy doesn't try to recreate existing tables.

```bash
mkdir -p prisma/migrations/0_init
npx prisma migrate diff \
  --from-empty \
  --to-schema-datamodel prisma/schema.prisma \
  --script > prisma/migrations/0_init/migration.sql

npx prisma migrate resolve --applied 0_init   # tell Prisma this migration is already in the DB
```

---

## 6. Prisma Client CRUD

The Prisma Client is the typed query builder you import and call. Every model becomes a property (`prisma.user`, `prisma.post`) exposing the same family of methods. **CRUD** = Create, Read, Update, Delete. The mental model: each method takes one **args object** with keys like `where`, `data`, `select`, `include`, `orderBy`, `take`, `skip` — learn the args object once and it transfers across every method.

### 6.1 Instantiate (preview; full singleton in §16)

```typescript
// lib/prisma.ts
import { PrismaClient } from "@prisma/client"
export const prisma = new PrismaClient()
```

### 6.2 CREATE **[B]**

```typescript
// create — insert one row, returns the full created record
const user = await prisma.user.create({
  data: {
    email: "alice@example.com",
    username: "alice",
    name: "Alice Smith",
    role: "ADMIN",          // enum value as a string literal (type-checked!)
  },
})

// createMany — bulk insert in a single SQL statement (vastly faster than a loop)
const result = await prisma.user.createMany({
  data: [
    { email: "bob@example.com",   username: "bob" },
    { email: "carol@example.com", username: "carol" },
  ],
  skipDuplicates: true,     // silently skip rows that hit a unique constraint
})
// result = { count: 2 }   ← createMany returns a COUNT, not the rows

// createManyAndReturn — bulk insert AND get the rows back (Prisma 5.14+)
const users = await prisma.user.createManyAndReturn({
  data: [{ email: "dave@example.com", username: "dave" }],
})
// users: User[]  ← use this when you need the generated ids of bulk inserts
```

> **Security recommendation — mass assignment.** `data: req.body` is a **mass-assignment vulnerability**: a client could send `{ role: "ADMIN", isActive: true }` and silently escalate privileges. *Never* spread raw request bodies into `data`. Validate and pick fields explicitly (with Zod/Valibot or manual destructuring) so only the fields you intend are writable. See §19 for a Zod example.

### 6.3 READ **[B]**

```typescript
// findUnique — fetch by @id or a @unique field; returns the record or null.
// Prisma can heavily optimize this because it knows the lookup hits at most one row.
const user = await prisma.user.findUnique({ where: { email: "alice@example.com" } })
// user: User | null  ← you MUST handle null

// findUniqueOrThrow — same, but throws P2025 if not found (skip the null check)
const u2 = await prisma.user.findUniqueOrThrow({ where: { id: 1 } })

// findFirst — first row matching ANY where clause (not just unique fields)
const draft = await prisma.post.findFirst({
  where: { status: "DRAFT" },
  orderBy: { createdAt: "desc" },   // ← important: "first" is meaningless without order
})

// findFirstOrThrow — throws if nothing matches
const post = await prisma.post.findFirstOrThrow({ where: { authorId: 1 } })

// findMany — a list
const posts = await prisma.post.findMany({
  where: { published: true },
  orderBy: { createdAt: "desc" },
  take: 10,                          // LIMIT 10
  skip: 0,                           // OFFSET 0
})
// posts: Post[]
```

> **`findUnique` vs `findFirst`:** `findUnique` only accepts unique fields and lets Prisma batch/optimize it (and dedupe under the DataLoader — see §12). `findFirst` accepts any filter but returns an *arbitrary* match if several qualify, so always pair it with `orderBy` to make "first" deterministic.

### 6.4 UPDATE **[B]**

```typescript
// update — one row by unique identifier; throws P2025 if it doesn't exist
const updated = await prisma.user.update({
  where: { id: 1 },
  data: { name: "Alice Updated", role: "MODERATOR" },
})

// updateMany — update every matching row; returns a count (no rows returned)
const res = await prisma.post.updateMany({
  where: { authorId: 1, published: false },
  data: { status: "ARCHIVED" },
})
// res = { count: 3 }

// Atomic number operations — computed in the DB, immune to race conditions
const post = await prisma.post.update({
  where: { id: 42 },
  data: {
    viewCount: { increment: 1 },     // viewCount = viewCount + 1, atomically
    // decrement | multiply | divide | set are the other operators
  },
})
```

> **Why atomic ops matter (the logic):** reading `viewCount`, adding 1 in JS, and writing it back is a *read-modify-write* race — two concurrent requests can both read 10 and both write 11, losing a count. `{ increment: 1 }` pushes the arithmetic into a single SQL `UPDATE … SET viewCount = viewCount + 1`, which the database serializes correctly. Always prefer atomic operators for counters/balances.

### 6.5 UPSERT **[I]**

`upsert` = update-if-exists-else-insert, in one atomic statement. It avoids the find-then-decide race and the "duplicate key" error from a naive create.

```typescript
const user = await prisma.user.upsert({
  where:  { email: "alice@example.com" },   // the unique lookup
  update: { name: "Alice Updated" },          // applied if the row exists
  create: {                                    // applied if it doesn't
    email: "alice@example.com",
    username: "alice",
    name: "Alice Updated",
  },
})
```

### 6.6 DELETE **[B]**

```typescript
// delete — one row by unique id; throws P2025 if not found
const deleted = await prisma.user.delete({ where: { id: 1 } })

// deleteMany — every matching row; returns a count
const res = await prisma.post.deleteMany({ where: { status: "ARCHIVED" } })
// res = { count: 5 }

// deleteMany with no where = delete ALL rows (dangerous — be explicit)
await prisma.comment.deleteMany({})
```

> **Safety recommendation:** prefer **soft deletes** for anything a user or auditor might need back (§19). Reserve hard `delete` for genuinely ephemeral data. When you do hard-delete, rely on `onDelete: Cascade` (or explicit ordered deletes) so you don't leave orphaned child rows that violate FKs.

---

## 7. Querying: Filters, Select, Include, Pagination

This is the heart of day-to-day Prisma. The `where` object expresses SQL `WHERE`; `select`/`include` shape the columns and relations returned; `orderBy`/`take`/`skip`/`cursor` handle sorting and paging. All of it is type-checked against your schema.

### 7.1 `where` filters & operators **[I]**

```typescript
// Equality is the default — bare value means "= value"
prisma.user.findMany({ where: { role: "ADMIN" } })

// Comparison & string operators (each field takes an operator object)
prisma.user.findMany({
  where: {
    age:       { gt: 18 },              // >        (gte, lt, lte too)
    score:     { gte: 4.5 },
    email:     { contains: "@gmail" },  // LIKE '%@gmail%'
    username:  { startsWith: "a" },     // LIKE 'a%'
    bio:       { endsWith: ".dev" },    // LIKE '%.dev'
    name:      { not: null },           // IS NOT NULL
    id:        { in: [1, 2, 3] },       // IN (1,2,3)
  },
})

// Logical combinators — AND / OR / NOT take arrays of conditions
prisma.post.findMany({
  where: {
    AND: [
      { published: true },
      { createdAt: { gte: new Date("2026-01-01") } },
    ],
  },
})
prisma.post.findMany({ where: { OR: [{ status: "PUBLISHED" }, { authorId: 1 }] } })
prisma.post.findMany({ where: { NOT: { status: "ARCHIVED" } } })

// Filter THROUGH a relation (posts whose author is an admin)
prisma.post.findMany({ where: { author: { role: "ADMIN" } } })

// some / every / none — filter on a LIST relation
prisma.user.findMany({
  where: {
    posts: {
      some:  { published: true },       // at least one published post
      // every: { published: true }     // all posts published
      // none:  { status: "ARCHIVED" }  // no archived posts
    },
  },
})
```

> **Security note:** every value you pass to `where` is **parameterized** by Prisma — it is sent to the database separately from the SQL text, so user input in a filter cannot inject SQL. This is the default safe behavior and a major reason to prefer the query API over hand-built SQL strings.

### 7.2 Case-insensitive search **[I]**

```typescript
prisma.user.findMany({
  where: { name: { contains: "alice", mode: "insensitive" } },  // Postgres/Mongo
})
```

> For real search workloads, `contains` (which becomes a `LIKE` and can't use a normal index well) doesn't scale. Use PostgreSQL full-text search (Prisma's `fullTextSearch` preview, or raw `tsvector` queries) or a `pg_trgm` GIN index — see POSTGRESQL_GUIDE.md.

### 7.3 `select` — pick exactly the columns you want **[I]**

`select` returns a *narrowed* object containing only the listed fields — and the TypeScript type narrows with it. This is both a performance win (don't haul back a 5KB `bio` you won't show) and a **security control** (never accidentally serialize a `passwordHash`).

```typescript
const user = await prisma.user.findUnique({
  where: { id: 1 },
  select: { id: true, email: true, name: true },
})
// Type is exactly { id: number; email: string; name: string | null }
// passwordHash is impossible to leak here — it's not in the type or the result.
```

> **Security recommendation:** for any model with sensitive columns (password hashes, tokens, internal flags), **default to `select`** in API responses and explicitly list the safe fields — an allow-list. Relying on "I'll remember to strip it" is how leaks happen. Alternatively use `omit` (below) to subtract sensitive fields globally.

### 7.4 `omit` — subtract fields **[I]**

```typescript
// The inverse of select: return everything EXCEPT the listed fields
const user = await prisma.user.findUnique({
  where: { id: 1 },
  omit: { passwordHash: true },   // available query-level and as a global default (§16)
})
```

### 7.5 `include` — eager-load relations **[I]**

`include` pulls in related records alongside the parent (Prisma issues efficient JOIN-like queries under the hood — see §12). Use it to avoid the N+1 trap.

```typescript
// User with their posts and profile
const user = await prisma.user.findUnique({
  where: { id: 1 },
  include: { posts: true, profile: true },
})
// user.posts: Post[]   user.profile: Profile | null

// include can carry its own where/orderBy/take — filter the related rows
const u = await prisma.user.findUnique({
  where: { id: 1 },
  include: {
    posts: { where: { published: true }, orderBy: { createdAt: "desc" }, take: 5 },
  },
})

// Nested include — posts → their comments → each comment's author
const deep = await prisma.user.findUnique({
  where: { id: 1 },
  include: {
    posts: { include: { comments: { include: { author: true } } } },
  },
})
```

> **`select` vs `include` — you cannot use both at the top level.** They're alternative ways to shape output. To get relations *and* limit scalar fields, use `select` everywhere — a nested relation inside `select` behaves like an include:

```typescript
const user = await prisma.user.findUnique({
  where: { id: 1 },
  select: {
    email: true,
    posts: { select: { title: true } },   // relation inside select == include + narrow
  },
})
```

### 7.6 `orderBy` **[I]**

```typescript
prisma.post.findMany({ orderBy: { createdAt: "desc" } })           // single field
prisma.post.findMany({ orderBy: [{ status: "asc" }, { createdAt: "desc" }] }) // tie-break
prisma.post.findMany({ orderBy: { author: { name: "asc" } } })    // by a relation's field
prisma.user.findMany({ orderBy: { posts: { _count: "desc" } } })  // by related count
```

### 7.7 Pagination: offset vs cursor **[I/A]**

There are two paging strategies and they behave very differently at scale.

**Offset pagination** (`skip` + `take`) is intuitive — "skip 20, take 10 = page 3." It supports jumping to an arbitrary page. Its weakness: the database must *count past* all skipped rows, so `skip: 100000` gets slow, and if rows are inserted/deleted between page loads, items shift (you see a duplicate or miss one).

```typescript
const page = 3, pageSize = 10
const posts = await prisma.post.findMany({
  skip: (page - 1) * pageSize,   // OFFSET 20
  take: pageSize,                 // LIMIT 10
  orderBy: { id: "asc" },
})
```

**Cursor pagination** uses the last item's stable key as a bookmark — "give me 10 rows *after* id 57." It stays fast at any depth (the DB seeks the index to the cursor instead of counting) and is stable under inserts/deletes. Its weakness: you can only go next/previous, not jump to "page 47." This is the right choice for infinite scroll and large feeds.

```typescript
// First page — no cursor
const first = await prisma.post.findMany({ take: 10, orderBy: { id: "asc" } })

// Next page — use the last id as the cursor; skip:1 excludes the cursor row itself
const lastId = first[first.length - 1].id
const next = await prisma.post.findMany({
  take: 10,
  skip: 1,
  cursor: { id: lastId },
  orderBy: { id: "asc" },
})
```

> **Best practice:** use cursor pagination for any feed that can grow large; reserve offset paging for small, bounded admin tables where "jump to page N" matters. Always pair pagination with a stable `orderBy` (a unique or near-unique key) — without it, results aren't deterministic.

---

## 8. Nested Writes

A **nested write** creates or modifies related records in a *single* Prisma call, which Prisma runs inside an implicit transaction (all-or-nothing). This is one of Prisma's most ergonomic features: instead of inserting a user, getting its id, then inserting its profile and posts in three round-trips, you describe the whole graph once and Prisma figures out the order and the FKs.

### 8.1 `create` nested — build a whole object graph **[I]**

```typescript
const user = await prisma.user.create({
  data: {
    email: "alice@example.com",
    username: "alice",
    profile: {
      create: { bio: "Full-stack dev", avatarUrl: "https://x.com/a.jpg" },
    },
    posts: {
      create: [
        { title: "Hello World", content: "First post" },
        { title: "Prisma rocks", content: "Here's why..." },
      ],
    },
  },
  include: { profile: true, posts: true },   // return the created graph
})
// Prisma inserts the user, then the profile & posts with the right FK — atomically.
```

### 8.2 `connect` — link to existing records **[I]**

Use `connect` when the related record already exists and you only need to wire the FK.

```typescript
const post = await prisma.post.create({
  data: {
    title: "Learning Prisma",
    author: { connect: { id: 1 } },          // link to existing user by id
    tags: {
      connect: [
        { name: "typescript" },               // connect by any @unique field
        { id: 5 },
      ],
    },
  },
})
```

### 8.3 `connectOrCreate` — link or create on the fly **[I]**

Perfect for tags: connect to the tag if it exists, otherwise create it — no pre-check race.

```typescript
const post = await prisma.post.create({
  data: {
    title: "New Post",
    author: { connect: { id: 1 } },
    tags: {
      connectOrCreate: [
        { where: { name: "prisma" }, create: { name: "prisma" } },
        { where: { name: "orm" },    create: { name: "orm" } },
      ],
    },
  },
})
```

### 8.4 Nested `update`, `upsert`, `delete`, `disconnect` **[I/A]**

```typescript
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    name: "Updated",
    profile: {
      upsert: {                              // create profile if absent, else update
        create: { bio: "New bio" },
        update: { bio: "Updated bio" },
      },
    },
    posts: {
      update:     { where: { id: 10 }, data: { published: true } }, // edit one related post
      delete:     { id: 11 },                                        // delete one related post
      disconnect: [{ id: 5 }],                                       // unlink (M:N) without deleting
    },
  },
})
```

### 8.5 `set` — replace the whole relation **[I]**

```typescript
// Replace ALL tag connections on a post with exactly this set (others disconnected)
await prisma.post.update({
  where: { id: 1 },
  data: { tags: { set: [{ id: 1 }, { id: 3 }] } },
})
```

> **Best practice:** prefer nested writes over multi-step manual sequences whenever the data forms one logical unit — you get atomicity for free and fewer round-trips. For complex conditional logic *between* writes (read a value, then decide), reach for an interactive transaction (§10) instead.

---

## 9. Aggregations, GroupBy & Count

For dashboards and analytics you summarize rather than list rows. Prisma exposes `count`, `aggregate`, and `groupBy`, mapping to SQL `COUNT`, `SUM/AVG/MIN/MAX`, and `GROUP BY … HAVING`.

### 9.1 `count` **[B]**

```typescript
const total      = await prisma.user.count()                          // all rows
const adminCount = await prisma.user.count({ where: { role: "ADMIN" } }) // filtered

// Count relations alongside a record via _count
const user = await prisma.user.findUnique({
  where: { id: 1 },
  include: { _count: { select: { posts: true, comments: true } } },
})
// user._count.posts: number   user._count.comments: number
```

### 9.2 `aggregate` **[I]**

```typescript
const stats = await prisma.post.aggregate({
  where: { published: true },
  _count: true,                  // total matching rows
  _avg:   { viewCount: true },   // AVG(view_count)
  _sum:   { viewCount: true },   // SUM(view_count)
  _min:   { viewCount: true },
  _max:   { viewCount: true },
})
console.log(stats._count, stats._avg.viewCount)
```

### 9.3 `groupBy` (with `having`) **[I/A]**

```typescript
// Posts per status, with average views, ordered by count
const grouped = await prisma.post.groupBy({
  by: ["status"],
  _count: { id: true },
  _avg:   { viewCount: true },
  orderBy: { _count: { id: "desc" } },
})
// → [{ status: "PUBLISHED", _count: { id: 45 }, _avg: { viewCount: 234 } }, ...]

// HAVING — filter AFTER grouping (only groups with > 10 posts)
const popular = await prisma.post.groupBy({
  by: ["status"],
  _count: { id: true },
  having: { id: { _count: { gt: 10 } } },
})
```

> **Performance note:** aggregations scan and group rows; back them with appropriate indexes on the grouped/filtered columns. For very heavy reporting (time-bucketing, window functions, percentiles), drop to raw SQL (§11) — it's both faster to write and faster to run than forcing the query API to express analytical SQL.

---

## 10. Transactions

A **transaction** groups multiple database operations so they either *all* succeed or *all* roll back, preserving invariants (the classic example: money must never leave one account without arriving in another). Prisma offers two flavors: **sequential** (`$transaction([...])`) and **interactive** (`$transaction(async tx => …)`).

### 10.1 Sequential transactions — an array of operations **[I]**

When you know the operations up front and none depends on another's *result*, pass an array. Prisma runs them in order on one connection and commits atomically; any failure rolls back the lot.

```typescript
const [debit, credit] = await prisma.$transaction([
  prisma.account.update({ where: { id: fromId }, data: { balance: { decrement: amount } } }),
  prisma.account.update({ where: { id: toId },   data: { balance: { increment: amount } } }),
])
// If either update throws, NEITHER is committed.
```

### 10.2 Interactive transactions — read, decide, write **[I/A]**

When a later write depends on an earlier read (check the balance *before* debiting), use the callback form. Prisma hands you a transaction-bound client `tx`; use `tx.*` for every operation inside. Throwing anywhere rolls back.

```typescript
const result = await prisma.$transaction(async (tx) => {
  const account = await tx.account.findUniqueOrThrow({ where: { id: fromId } })

  if (account.balance < amount) {
    throw new Error("Insufficient funds")   // throw → automatic rollback
  }

  await tx.account.update({ where: { id: fromId }, data: { balance: { decrement: amount } } })
  return tx.account.update({ where: { id: toId }, data: { balance: { increment: amount } } })
})
```

### 10.3 Options & isolation levels **[A]**

```typescript
await prisma.$transaction(
  async (tx) => { /* ... */ },
  {
    maxWait: 5000,                   // ms to wait to acquire a connection (default 2000)
    timeout: 10000,                  // ms the whole transaction may run (default 5000)
    isolationLevel: "Serializable",  // strongest isolation — see below
  }
)
```

**Isolation levels** control how concurrent transactions see each other's uncommitted/committed work, trading consistency against concurrency: `ReadCommitted` (default on Postgres) → `RepeatableRead` → `Serializable` (strongest; can reject conflicting transactions with a retryable error). Use `Serializable` for financial invariants where correctness beats throughput.

> **Best practice — keep transactions SHORT.** A transaction holds a connection and locks for its entire duration; long-running interactive transactions starve the pool and cause timeouts under load. Do *no* slow work (HTTP calls, heavy computation, user prompts) inside a transaction — fetch/compute first, then open the transaction only for the writes.

> **Retry on conflict (P2034):** under `Serializable`, conflicting transactions fail with `P2034` and are *meant* to be retried. Wrap critical financial transactions in a small retry loop (e.g. up to 3 attempts with backoff) that re-runs on `P2034`.

---

## 11. Raw Queries & Injection Safety

Sometimes the query API can't express what you need: a window function, a recursive CTE, a DB-specific operator, or a heavily-optimized analytical query. Prisma gives you a **raw escape hatch** — but raw SQL is exactly where SQL-injection vulnerabilities live, so this section is as much about *safety* as syntax.

### 11.1 The injection threat (the why) **[A]**

SQL injection happens when untrusted input is *concatenated into* a SQL string, letting an attacker change the query's meaning. `"SELECT * FROM users WHERE email = '" + input + "'"` with input `' OR '1'='1` returns every user; worse inputs can drop tables. **The fix is parameterization:** send the SQL text and the values *separately*, so the database treats values strictly as data, never as SQL. Prisma's tagged-template raw methods parameterize automatically.

### 11.2 `$queryRaw` — SELECT, safely parameterized **[A]**

```typescript
import { Prisma } from "@prisma/client"

const email = userInput   // untrusted

// Tagged template: ${email} becomes a bound parameter, NOT string concatenation.
const users = await prisma.$queryRaw<{ id: number; email: string }[]>`
  SELECT id, email FROM users WHERE email = ${email}
`
// Prisma sends:  SELECT id, email FROM users WHERE email = $1   with $1 = email value
// Injection is impossible — the value can never be interpreted as SQL.

// Multiple parameters — each interpolation is bound independently
const role = "ADMIN", since = new Date("2026-01-01"), limit = 20
const rows = await prisma.$queryRaw<{ id: number; name: string }[]>`
  SELECT id, name FROM users
  WHERE role = ${role} AND created_at > ${since}
  ORDER BY name ASC
  LIMIT ${limit}
`
```

### 11.3 `$executeRaw` — INSERT/UPDATE/DELETE (returns affected count) **[A]**

```typescript
const count = await prisma.$executeRaw`
  UPDATE posts SET view_count = view_count + 1 WHERE id = ${postId}
`
// count = number of affected rows. Same automatic parameterization as $queryRaw.
```

### 11.4 Composing raw SQL safely with `Prisma.sql` **[A]**

For dynamic fragments (optional filters), build with `Prisma.sql` / `Prisma.join` / `Prisma.empty` — never string concatenation. Values inside `Prisma.sql` stay parameterized.

```typescript
import { Prisma } from "@prisma/client"

const ids = [1, 2, 3]
const onlyPublished = true

const where = Prisma.sql`
  WHERE id IN (${Prisma.join(ids)})
  ${onlyPublished ? Prisma.sql`AND published = true` : Prisma.empty}
`
const rows = await prisma.$queryRaw`SELECT * FROM posts ${where}`
```

### 11.5 `$queryRawUnsafe` — last resort, you own the risk **[A]**

The `*Unsafe` variants take a plain string and do **not** auto-parameterize the template — they exist only for cases where part of the SQL is dynamic in a way templates can't express (e.g. a column or table name chosen at runtime). Even then, never interpolate user *values* — pass them as positional parameters, and **allow-list** any dynamic identifiers.

```typescript
// ✅ Identifier from a strict allow-list; values still passed as bound params
const ALLOWED = { name: "name", created: "created_at" } as const
const sortCol = ALLOWED[userChoice] ?? "name"   // reject anything not allow-listed

const rows = await prisma.$queryRawUnsafe(
  `SELECT id, ${sortCol} FROM users WHERE id = $1`,  // identifier is from allow-list
  userId,                                            // value via positional param ($1)
)
```

> **Security rules for raw SQL — memorize these:**
> 1. **Default to the tagged-template forms** (`$queryRaw\`...\``, `$executeRaw\`...\``) — they parameterize for you.
> 2. **Never** build SQL with `+`/template-string concatenation of user input.
> 3. Use `*Unsafe` only when you must, and then: values → positional params; identifiers → allow-list.
> 4. Raw results are **untyped** at the DB boundary — provide an explicit generic type *and* validate the shape if it flows to clients.
> 5. The least-privileged DB user (no `DROP`, scoped grants) limits the blast radius if something slips through.

### 11.6 Typed SQL (`--sql`) **[A]**

⚡ Prisma can generate fully-typed functions from `.sql` files (`prisma generate --sql`), giving you raw SQL power *with* compile-time-checked parameters and return types — the best of both worlds for complex queries you reuse. Check your version's docs for the exact workflow.

---

## 12. The N+1 Problem & Relation Loading

The **N+1 query problem** is the most common ORM performance bug. It happens when you fetch a list of N parents and then, in a loop, fire one more query per parent to load its children — 1 + N queries where you needed about 2. At N = 1000 that's a thousand round-trips and a slow endpoint.

### 12.1 Recognizing and fixing it **[I/A]**

```typescript
// ❌ N+1: 1 query for users, then ONE query PER user for posts
const users = await prisma.user.findMany()
for (const user of users) {
  const posts = await prisma.post.findMany({ where: { authorId: user.id } }) // fires N times!
}

// ✅ FIX with include: Prisma loads everything in a small, constant number of queries
const usersWithPosts = await prisma.user.findMany({ include: { posts: true } })
// user.posts is already populated for every user.

// ✅ If you only need counts, ask for counts — don't load the rows at all
const usersWithCounts = await prisma.user.findMany({
  include: { _count: { select: { posts: true } } },
})
```

### 12.2 How Prisma loads relations (the logic) **[A]**

Two things help you avoid N+1 even when you don't realize it:

- **`include`/nested `select` are batched** — Prisma fetches the related rows for *all* parents together (an `IN (...)` style query or a JOIN), not one per parent. So `include` is the primary, idiomatic fix.
- **Automatic query batching / DataLoader** — within a single tick of the event loop, multiple identical-shape `findUnique` calls are coalesced into one `WHERE id IN (...)` query. This means even some loop-based code is automatically de-N+1'd. You can rely on it, but `include` is clearer and stronger.

> **Best practice:** turn on query logging in development (§19) and watch the count for an endpoint. If a single request emits dozens of similar `SELECT`s, you have an N+1 — convert the loop to an `include` or a single `findMany` with `where: { id: { in: [...] } }`.

---

## 13. Client Extensions (and Legacy Middleware)

**Client extensions** (`$extends`) are the modern, type-safe way to customize Prisma Client: add model methods, computed result fields, query interceptors, or new top-level helpers. They replace the deprecated `$use` middleware and, crucially, preserve types.

### 13.1 Model extensions — add methods to a model **[A]**

```typescript
import { PrismaClient } from "@prisma/client"

const prisma = new PrismaClient().$extends({
  model: {
    user: {
      // prisma.user.findByEmail("...") becomes a real, typed method
      async findByEmail(email: string) {
        const client = Prisma.getExtensionContext(this) as any
        return client.findUnique({ where: { email } })
      },
    },
    post: {
      async publish(id: number) {
        const client = Prisma.getExtensionContext(this) as any
        return client.update({ where: { id }, data: { published: true, status: "PUBLISHED" } })
      },
    },
  },
})
import { Prisma } from "@prisma/client"
```

### 13.2 Result extensions — computed fields **[A]**

```typescript
const prisma = new PrismaClient().$extends({
  result: {
    user: {
      displayName: {
        needs: { name: true, username: true },   // fields the computation depends on
        compute(user) { return user.name ?? user.username },
      },
    },
  },
})
const u = await prisma.user.findUnique({ where: { id: 1 } })
console.log(u?.displayName)   // computed in JS, not stored in the DB
```

### 13.3 Query extensions — intercept every query (soft delete!) **[A]**

The textbook use: enforce soft-delete filtering globally so no query ever returns deleted rows.

```typescript
const prisma = new PrismaClient().$extends({
  query: {
    user: {
      async findMany({ args, query }) {
        args.where = { ...args.where, deletedAt: null }  // always exclude soft-deleted
        return query(args)
      },
    },
    // $allModels / $allOperations let you apply logic across the whole client
  },
})
```

### 13.4 Legacy `$use` middleware — deprecated **[A]**

```typescript
// ⚡ Version note: $use is deprecated; prefer $extends query extensions above.
prisma.$use(async (params, next) => {
  const start = Date.now()
  const result = await next(params)
  console.log(`${params.model}.${params.action} took ${Date.now() - start}ms`)
  return result
})
```

> **Note:** `$extends` returns a *new* client; assign it back (`const prisma = new PrismaClient().$extends(...)`). Extensions compose — you can chain multiple `$extends` calls.

---

## 14. Seeding the Database

**Seeding** populates a fresh database with baseline or sample data — an admin account, lookup tables, demo content. It makes `migrate reset` give you a usable DB instantly and keeps every developer's environment consistent.

### 14.1 An idempotent seed script **[I]**

```typescript
// prisma/seed.ts
import { PrismaClient } from "@prisma/client"
const prisma = new PrismaClient()

async function main() {
  // upsert → running the seed twice doesn't create duplicates (idempotent)
  const admin = await prisma.user.upsert({
    where: { email: "admin@example.com" },
    update: {},
    create: { email: "admin@example.com", username: "admin", name: "Admin", role: "ADMIN" },
  })

  const alice = await prisma.user.upsert({
    where: { email: "alice@example.com" },
    update: {},
    create: { email: "alice@example.com", username: "alice", name: "Alice" },
  })

  await prisma.post.createMany({
    data: [
      { title: "Hello World", authorId: alice.id, published: true, status: "PUBLISHED" },
      { title: "Draft Post",  authorId: alice.id, published: false, status: "DRAFT" },
    ],
    skipDuplicates: true,
  })

  console.log("Seeded:", { admin: admin.email, alice: alice.email })
}

main()
  .catch((e) => { console.error(e); process.exit(1) })
  .finally(() => prisma.$disconnect())
```

### 14.2 Register & run **[I]**

```json
// package.json
{
  "prisma": { "seed": "tsx prisma/seed.ts" }
}
```

```bash
npx prisma db seed       # run the seed
npx prisma migrate reset # drops, re-migrates, AND runs the seed automatically
```

> **Best practice — make seeds idempotent** (use `upsert`/`skipDuplicates`) so they're safe to re-run. **Security recommendation:** never seed *real* secrets or production passwords; seeded admin credentials are for local/dev only and must differ from production.

---

## 15. Generated Types, Type Safety & Error Handling

Prisma's generated types are the payoff for the schema-first approach. Everything you query is typed *from your schema*, and the `Prisma` namespace gives you utility types to describe inputs, payloads, and errors.

### 15.1 Model & enum types **[I]**

```typescript
import { User, Post, Role } from "@prisma/client"   // generated from your schema

const user: User = await prisma.user.findUniqueOrThrow({ where: { id: 1 } })
// user.role: Role (the enum)   user.posts: ← NOT present unless you include it

// Enums are real values — no magic strings
await prisma.user.update({ where: { id: 1 }, data: { role: Role.ADMIN } })
```

### 15.2 The `Prisma` namespace — input & payload types **[I/A]**

```typescript
import { Prisma } from "@prisma/client"

type UserCreateInput = Prisma.UserCreateInput          // shape accepted by user.create
type UserWhere       = Prisma.UserWhereUniqueInput

// GetPayload — derive the EXACT return type of a query with includes/selects
type UserWithPosts = Prisma.UserGetPayload<{ include: { posts: true } }>
// ≡ User & { posts: Post[] }

type UserEmailOnly = Prisma.UserGetPayload<{ select: { id: true; email: true } }>
// ≡ { id: number; email: string }
```

### 15.3 Reusable, type-safe query args with `satisfies` **[A]**

```typescript
// Define an include shape once; reuse it and keep the precise return type
const withPosts = { include: { posts: true, profile: true } } satisfies Prisma.UserDefaultArgs

async function getUser(id: number) {
  return prisma.user.findUniqueOrThrow({ where: { id }, ...withPosts })
}
// Return type is inferred as User & { posts: Post[]; profile: Profile | null }
```

### 15.4 Error handling & known error codes **[I/A]**

Prisma throws typed errors. The important class is `PrismaClientKnownRequestError`, which carries a `.code` (a stable `Pxxxx` string) and `.meta`. Handle these explicitly to turn DB errors into clean HTTP responses instead of leaking stack traces.

```typescript
import { Prisma } from "@prisma/client"

try {
  await prisma.user.create({ data: { email: "dup@example.com", username: "x" } })
} catch (e) {
  if (e instanceof Prisma.PrismaClientKnownRequestError) {
    if (e.code === "P2002") {                       // unique constraint violation
      // e.meta.target tells you which field(s) collided
      throw new Error(`Already exists: ${e.meta?.target}`)
    }
    if (e.code === "P2025") {                        // record not found (update/delete)
      throw new Error("Not found")
    }
  }
  throw e
}
```

| Code | Meaning | Typical HTTP mapping |
|---|---|---|
| `P2000` | Value too long for the column | 400 |
| `P2002` | Unique constraint violation | 409 Conflict |
| `P2003` | Foreign-key constraint failed | 409 / 400 |
| `P2025` | Record required but not found (update/delete/`*OrThrow`) | 404 |
| `P2034` | Transaction write conflict / deadlock — **retryable** | retry, then 503 |

Other error classes: `PrismaClientValidationError` (you passed an invalid args shape — a bug, not user error), `PrismaClientInitializationError` (couldn't connect), `PrismaClientRustPanicError` (engine crash, rare).

> **Security recommendation:** never forward raw Prisma error messages to API clients — they can reveal column names, constraints, and schema structure. Map codes to generic messages (`409 "email already in use"`) and log the detail server-side only.

---

## 16. Prisma in Next.js, NestJS & Serverless

### 16.1 The connection-explosion problem (the why) **[A]**

`new PrismaClient()` opens its own **connection pool**. In a normal long-running server you instantiate it *once* and that's fine. But two situations break that: (1) **Next.js dev hot-reload** re-executes modules on every save, creating a new client (and pool) each time until your database refuses new connections; (2) **serverless** functions cold-start fresh instances. Both exhaust the database's connection limit. The fix in dev is a **singleton on `globalThis`**; in serverless it's an external pooler (§17).

### 16.2 The singleton pattern (the standard fix) **[A]**

```typescript
// lib/prisma.ts
import { PrismaClient } from "@prisma/client"

const globalForPrisma = globalThis as unknown as { prisma?: PrismaClient }

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === "development" ? ["query", "error", "warn"] : ["error"],
    // omit sensitive fields globally so they can NEVER be returned by accident:
    omit: { user: { passwordHash: true } },
  })

// Cache on globalThis only in dev so hot-reload reuses one client.
// In production each process gets exactly one client (no global needed).
if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma
```

### 16.3 Next.js App Router — Server Components & Route Handlers **[A]**

```typescript
// app/users/page.tsx — Server Component: query the DB directly, no API layer needed
import { prisma } from "@/lib/prisma"

export default async function UsersPage() {
  const users = await prisma.user.findMany({
    select: { id: true, name: true, email: true },   // allow-list fields
    orderBy: { createdAt: "desc" },
  })
  return <ul>{users.map((u) => <li key={u.id}>{u.name} — {u.email}</li>)}</ul>
}
```

```typescript
// app/api/users/route.ts — Route Handler
import { prisma } from "@/lib/prisma"
import { NextResponse } from "next/server"
import { z } from "zod"

const CreateUser = z.object({ email: z.string().email(), username: z.string().min(3) })

export async function GET() {
  return NextResponse.json(await prisma.user.findMany({ select: { id: true, email: true } }))
}

export async function POST(req: Request) {
  // Validate FIRST — never pass req.json() straight into data (mass-assignment, §19)
  const parsed = CreateUser.safeParse(await req.json())
  if (!parsed.success) return NextResponse.json({ error: parsed.error.format() }, { status: 400 })

  const user = await prisma.user.create({ data: parsed.data })  // only validated fields
  return NextResponse.json(user, { status: 201 })
}
```

> See NEXTJS_16_GUIDE.md for caching/`revalidate` interaction with Prisma queries and where to put `"use server"` actions.

### 16.4 NestJS — Prisma as an injectable provider **[A]**

In NestJS you wrap the client in a provider so it participates in DI and lifecycle hooks (connect on init, disconnect on shutdown).

```typescript
// prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from "@nestjs/common"
import { PrismaClient } from "@prisma/client"

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit()    { await this.$connect() }
  async onModuleDestroy() { await this.$disconnect() }
}

// users.service.ts — inject and use it
import { Injectable } from "@nestjs/common"
import { PrismaService } from "./prisma.service"

@Injectable()
export class UsersService {
  constructor(private readonly prisma: PrismaService) {}
  findAll() { return this.prisma.user.findMany() }
}
```

> See NESTJS_GUIDE.md for the full module setup, global `PrismaModule`, and exception filters that translate `Pxxxx` codes to HTTP responses.

### 16.5 Standalone scripts — always disconnect **[B]**

```typescript
async function main() {
  console.log(await prisma.user.count())
}
main()
  .catch((e) => { console.error(e); process.exit(1) })
  .finally(() => prisma.$disconnect())   // long-running servers DON'T need this; scripts DO
```

---

## 17. Connection Pooling & Performance

### 17.1 What a connection pool is & why pooling matters **[A]**

Opening a database connection is expensive (TCP + auth + session setup), so Prisma keeps a **pool** of open connections and reuses them. The pool size defaults to roughly `num_cpus * 2 + 1` and is tunable via the connection string: `?connection_limit=10`. The catch: PostgreSQL itself has a hard cap on total connections (often ~100). Multiply by many serverless instances or app replicas and you blow past it — connections queue or get refused. This is why pooling strategy is a *production* concern, not a detail.

```bash
# Tune Prisma's own pool and timeouts via the URL
DATABASE_URL="postgresql://user:pass@host:5432/db?connection_limit=10&pool_timeout=20"
```

### 17.2 External poolers (PgBouncer, Accelerate, platform poolers) **[A]**

For serverless/edge, the answer is an **external pooler** that sits between your many app instances and the database, multiplexing thousands of client connections onto a small set of real DB connections.

| Option | What it is | Notes |
|---|---|---|
| **PgBouncer** | Battle-tested Postgres pooler | Use **transaction mode**; add `?pgbouncer=true` to the URL so Prisma disables prepared statements that don't survive transaction-mode pooling |
| **Prisma Accelerate** | Prisma's managed global pooler + edge cache | `prisma://` URL; adds `cacheStrategy` to queries |
| **Supabase / Neon / platform poolers** | Built-in poolers | Use the *pooler* connection string for the app |

```prisma
// Migrations need a DIRECT (non-pooled) connection — pooled URLs can't run DDL reliably.
datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")   // pooled URL (PgBouncer) for the running app
  directUrl = env("DIRECT_URL")     // direct URL used by prisma migrate
}
```

> **The directUrl rule:** when the app connects through PgBouncer in *transaction* mode, `prisma migrate` must use a **direct** connection (`directUrl`) because DDL and advisory locks don't behave under transaction pooling. Set both URLs.

### 17.3 Prisma Accelerate query caching **[A]**

```typescript
import { PrismaClient } from "@prisma/client"
import { withAccelerate } from "@prisma/extension-accelerate"   // npm i @prisma/extension-accelerate

export const prisma = new PrismaClient().$extends(withAccelerate())

const users = await prisma.user.findMany({
  cacheStrategy: { ttl: 60, swr: 30 },   // cache 60s; serve stale up to 30s while revalidating
})
```

### 17.4 Performance checklist **[A]**

- **Index** columns used in `where`/`orderBy`/joins (`@@index`); verify with `EXPLAIN ANALYZE` (POSTGRESQL_GUIDE.md).
- **`select`/`omit`** to fetch only needed columns — less I/O, smaller payloads.
- **Batch** with `include` and `createMany` instead of loops (kills N+1).
- **Cursor pagination** for large feeds; never deep `skip`.
- **Atomic operators** (`increment`) instead of read-modify-write.
- **Keep transactions short**; do slow work outside them.
- **Tune `connection_limit`** to fit your DB's cap × number of instances; add a pooler for serverless.
- **Push heavy analytics to raw SQL** rather than contorting the query API.

---

## 18. Prisma Studio

Prisma Studio is a zero-install, browser-based GUI for browsing and editing your data — invaluable while learning and for quick manual fixes in dev.

```bash
npx prisma studio                      # opens http://localhost:5555
npx prisma studio --port 5556          # different port
npx prisma studio --schema=./path/schema.prisma
```

| Feature | Description |
|---|---|
| Browse / view | Models as tabbed, paginated tables with type-aware cells |
| Create / edit / delete | Form-based create, inline edit, single/bulk delete |
| Filter | Basic filtering by field value |
| Navigate relations | Click through to related records |
| Export | Copy rows as JSON |

> **Limitations & safety:** no raw-SQL editor (use a DB client or `prisma db execute`), no schema editing (edit `schema.prisma`). Studio connects with your `DATABASE_URL` directly and bypasses app-level rules — **don't run it against production** with write access casually.

---

## 19. Security, Best Practices & Gotchas

### 19.1 Mass assignment — the top write-side risk **[I/A]**

Spreading untrusted input into `data` lets attackers set fields you never meant to expose (`role`, `isAdmin`, `balance`). Always validate-then-pick.

```typescript
import { z } from "zod"

// Allow-list exactly the fields a user may set — role/isActive are NOT here
const CreateUser = z.object({
  email: z.string().email(),
  username: z.string().min(3).max(50),
  name: z.string().optional(),
})

async function register(body: unknown) {
  const data = CreateUser.parse(body)        // throws on anything unexpected
  return prisma.user.create({ data })        // only validated, allow-listed fields reach the DB
}

// ❌ NEVER: prisma.user.create({ data: req.body })  ← privilege escalation waiting to happen
```

### 19.2 Don't leak sensitive columns — default to allow-lists **[I]**

```typescript
// Global omit so passwordHash can never be returned by accident (§16), or per-query select:
const safeUser = await prisma.user.findUnique({
  where: { id },
  select: { id: true, email: true, name: true, role: true },  // explicit allow-list
})
```

### 19.3 SQL injection — covered, but the rule restated **[A]**

Use the query API (auto-parameterized) and tagged-template raw (`$queryRaw\`...\``). Never concatenate user input into SQL; with `*Unsafe`, use positional params for values and allow-list identifiers (§11).

### 19.4 Least-privilege DB users **[A]**

Run the app as a DB role that can `SELECT/INSERT/UPDATE/DELETE` only the tables it needs — *not* a superuser, *not* able to `DROP`. Give migrations a separate, more-privileged role. This caps the damage of any injection or bug. (Setup in POSTGRESQL_GUIDE.md / RELATIONAL_DB_DESIGN_GUIDE.md.)

### 19.5 Soft deletes **[I]**

```prisma
model User {
  // ...
  deletedAt DateTime?   // null = active; timestamp = soft-deleted
}
```

```typescript
// "Delete" = stamp deletedAt; recover by clearing it. Pair with a query extension (§13.3)
async function softDelete(id: number) {
  return prisma.user.update({ where: { id }, data: { deletedAt: new Date() } })
}
prisma.user.findMany({ where: { deletedAt: null } })   // only active rows
```

> **Why soft delete:** preserves audit trails, supports "undo," avoids breaking FK references, and protects against accidental cascade wipes. The cost: every query must filter `deletedAt: null` — automate that with a query extension so you can't forget.

### 19.6 The rename data-loss gotcha **[I]**

Renaming a schema field makes Prisma generate **DROP + ADD** (losing data). For tables with data, use `migrate dev --create-only` and hand-edit to `ALTER TABLE … RENAME COLUMN` (§5.7). This is the single most important migration-safety habit.

### 19.7 Enum changes are breaking **[I]**

```
Step 1: add the new value (keep the old)
Step 2: migrate existing rows to the new value
Step 3: remove the old value in a SEPARATE migration
```

Renaming/removing enum values directly generates DROP+CREATE that fails if any row still uses the old value. Stage it.

### 19.8 ID strategy trade-offs **[I]**

```prisma
id Int    @id @default(autoincrement())  // simple; BUT leaks row count & is guessable/enumerable
id String @id @default(cuid())           // collision-resistant, URL-safe, not enumerable
id String @id @default(uuid())           // universal standard; random order hurts index locality
id String @id @default(uuid(7))          // sortable UUID (Prisma 5.8+): unguessable AND index-friendly
```

> **Security recommendation:** sequential integer ids are *enumerable* — `/users/1`, `/users/2` lets attackers walk your data and reveals volume (an IDOR enabler). For anything client-facing, prefer `cuid`/`uuid(7)` so ids aren't guessable, and always enforce authorization (don't rely on id obscurity alone).

### 19.9 Decimal vs Float for money **[I]**

```prisma
price Decimal @db.Decimal(10, 2)   // ✅ exact — use for money/currency
score Float                         // ❌ floating-point error — never for money
```

`Decimal` returns a Prisma `Decimal` object (do arithmetic with its methods or a decimal library, not raw JS `+`).

### 19.10 JSON fields **[I]**

```prisma
model Config { id Int @id @default(autoincrement()); settings Json }
```

```typescript
await prisma.config.create({ data: { settings: { theme: "dark", lang: "en" } } })
// Filter inside JSONB (Postgres)
await prisma.config.findMany({ where: { settings: { path: ["theme"], equals: "dark" } } })
```

> JSON columns are unindexed/unvalidated by default — convenient for flexible blobs, dangerous as a dumping ground. Promote frequently-queried keys to real, typed, indexed columns.

### 19.11 Quick gotcha list **[I]**

```typescript
// Always await — a forgotten await returns a Promise, not data
const u = await prisma.user.findUnique({ where: { id: 1 } })

// findFirst needs orderBy to be deterministic; findUnique only takes unique fields
// select + include can't be mixed at the top level — use nested select
// Never `new PrismaClient()` per request — use the singleton (§16)
// Run `prisma generate` after EVERY schema change (postinstall script helps)
```

### 19.12 Query logging in dev (find N+1 & slow queries) **[I/A]**

```typescript
const prisma = new PrismaClient({
  log: [
    { level: "query", emit: "event" },
    { level: "warn",  emit: "stdout" },
    { level: "error", emit: "stdout" },
  ],
})
prisma.$on("query", (e) => {
  console.log(`${e.query} -- params: ${e.params} -- ${e.duration}ms`)
})
```

### 19.13 `migrate` vs `db push` decision table **[I]**

| Command | Use for | Migration files | Safe for prod |
|---|---|---|---|
| `migrate dev` | Development changes | Creates them | No (dev only) |
| `migrate deploy` | Production / CI | Applies existing | **Yes** |
| `db push` | Prototyping / MongoDB | None | No (can drop data) |
| `migrate reset` | Wipe & rebuild dev DB | Re-applies all | **No — destructive** |

---

## 20. Study Path & Build-to-Learn Projects

Work top to bottom; the projects are where the concepts stick. Pair each phase with the cross-referenced guides.

### Phase 1 — Foundations (Week 1) [B]
1. §1 — internalize "ORM," schema-first, and *when Prisma beats raw SQL / a query builder*.
2. §2 — set up a real project with **SQLite** (no server needed); add the `postinstall` generate script.
3. §3 — write a schema with 3–4 models, an enum, every field-attribute type, and `@@index`/`@@map`.
4. Run `migrate dev` and **read the generated `migration.sql`** — connect PSL to actual SQL.
5. **Project:** model a blog — `User`, `Post`, `Tag`, `Comment` — with `Role` and `PostStatus` enums.

### Phase 2 — Querying & Relations (Week 2) [B/I]
6. §6 — implement full CRUD for every blog model; use `createMany`/`createManyAndReturn`.
7. §7 — drill every filter operator; build a search with `OR` + `contains`; do both pagination styles.
8. §4 — add all four relation kinds (1:1, 1:N, implicit & explicit M:N) and a self-relation.
9. §8 — create a post with nested tags + comments in one call; practice `connectOrCreate`.
10. **Project:** a REST API (Fastify/Hono/Express) with CRUD + filters + cursor pagination for the blog.

### Phase 3 — Advanced Features (Week 3) [I/A]
11. §9 — build a dashboard endpoint using `groupBy` + `aggregate` (posts per status, avg views).
12. §10 — implement a money-transfer feature with an **interactive transaction** + P2034 retry.
13. §11 — write a reporting query in **raw SQL** with `$queryRaw` template literals; verify it parameterizes.
14. §12 — induce an N+1 with a loop, observe it in query logs, then fix it with `include`.
15. §15 — type an API response with `Prisma.PostGetPayload`; map `P2002`/`P2025` to HTTP codes.
16. **Project:** analytics endpoint — posts-per-day for 30 days via `groupBy` (or raw SQL date bucketing).

### Phase 4 — Production & Integration (Week 4) [A]
17. §16 — wire the **singleton** into a Next.js App Router project (Server Component + Route Handler).
18. §16 — alternatively expose Prisma as a NestJS provider (`PrismaService`).
19. §5 — practice dev→prod: author migrations with `migrate dev`, deploy with `migrate deploy`; do a safe column rename via `--create-only`.
20. §14 — write an idempotent seed; verify with `migrate reset`.
21. §13 + §19.5 — add a **soft-delete** query extension.
22. **Project:** full-stack Next.js blog — Server Components, Route Handlers, Prisma singleton, seed, soft delete, Zod validation.

### Phase 5 — Mastery
23. §17 — configure a pooler (PgBouncer/Accelerate) with `directUrl`; tune `connection_limit`.
24. §19 — security audit: mass-assignment (Zod allow-lists), no sensitive-column leaks, least-privilege DB user, non-enumerable ids.
25. §11.6 — explore typed SQL (`generate --sql`); §2.6 — try the Rust-free driver-adapter client on an edge runtime.
26. CI: a GitHub Actions job that runs `prisma migrate deploy` on push to main.
27. **Capstone:** multi-tenant SaaS starter — JWT auth + RBAC, soft deletes, migrations + seed, connection pooling, query logging, security checklist, Studio walkthrough. Cross-reference NESTJS_GUIDE.md or NEXTJS_16_GUIDE.md for the app layer and POSTGRESQL_GUIDE.md for the database.

### Cheat-sheet — the commands you'll actually use

```bash
npx prisma init --datasource-provider sqlite   # scaffold schema + .env
npx prisma format                              # format the schema
npx prisma validate                            # validate without generating
npx prisma generate                            # rebuild the typed client (after every schema change)
npx prisma migrate dev --name <name>           # create + apply a migration (dev)
npx prisma migrate dev --create-only --name x  # author SQL without applying (for safe renames)
npx prisma migrate deploy                       # apply pending migrations (prod/CI)
npx prisma migrate status                       # what's applied / pending / failed
npx prisma migrate reset                        # drop + re-apply + seed (DEV ONLY — destructive)
npx prisma db push                              # push schema, no migration file (prototyping)
npx prisma db pull                              # introspect an existing DB into the schema
npx prisma db seed                              # run the seed script
npx prisma studio                               # GUI at localhost:5555
```
