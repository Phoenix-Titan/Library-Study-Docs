# Prisma ORM Complete Guide (v6+)

A full, study-friendly reference for **Prisma ORM** — the type-safe database toolkit for TypeScript and Node.js. Covers schema design, migrations, every Client API method, relations, transactions, raw queries, extensions, seeding, Next.js integration, and production best practices.

> **What it is:** Prisma is a next-generation ORM for Node.js and TypeScript that replaces hand-written SQL and traditional ORMs (like Sequelize, TypeORM). It has three main parts: **Prisma Client** (auto-generated, type-safe query builder), **Prisma Migrate** (declarative schema migrations), and **Prisma Studio** (visual DB browser). Your database schema is the single source of truth, defined in a `.prisma` file, and Prisma generates fully-typed TypeScript code from it — so your editor knows every column name and type at compile time.

---

## Table of Contents
1. [What Prisma Is & Its Parts](#1-what-prisma-is--its-parts)
2. [Setup & Initialization](#2-setup--initialization)
3. [The Schema File](#3-the-schema-file)
4. [Relations](#4-relations)
5. [Migrations](#5-migrations)
6. [Prisma Client CRUD](#6-prisma-client-crud)
7. [Querying: Filters, Select, Include, Pagination](#7-querying-filters-select-include-pagination)
8. [Nested Writes](#8-nested-writes)
9. [Aggregations, GroupBy & Count](#9-aggregations-groupby--count)
10. [Transactions](#10-transactions)
11. [Raw Queries](#11-raw-queries)
12. [Client Extensions (and Legacy Middleware)](#12-client-extensions-and-legacy-middleware)
13. [Seeding the Database](#13-seeding-the-database)
14. [Generated Types & Type Safety](#14-generated-types--type-safety)
15. [Using Prisma in Next.js / Serverless](#15-using-prisma-in-nextjs--serverless)
16. [Prisma Studio](#16-prisma-studio)
17. [Tips, Tricks & Gotchas](#17-tips-tricks--gotchas)
18. [Study Path](#18-study-path)

---

## 1. What Prisma Is & Its Parts

### The Three Pillars

| Part | What it does | CLI command |
|---|---|---|
| **Prisma Client** | Auto-generated, type-safe query builder you import in code | `npx prisma generate` |
| **Prisma Migrate** | Tracks schema changes, writes SQL migration files, applies them | `npx prisma migrate dev` |
| **Prisma Studio** | Visual browser/editor for your database rows | `npx prisma studio` |

### Supported Databases

| Database | Prisma identifier | Notes |
|---|---|---|
| PostgreSQL | `postgresql` | Full feature support; recommended for production |
| MySQL / MariaDB | `mysql` | Full feature support |
| SQLite | `sqlite` | Great for local dev / embedded apps |
| Microsoft SQL Server | `sqlserver` | Full feature support |
| CockroachDB | `cockroachdb` | PostgreSQL-compatible distributed DB |
| MongoDB | `mongodb` | Supported via `db push` (no migrate); uses `ObjectId` |
| PlanetScale | `mysql` | Serverless MySQL; use `db push` + `relationMode = "prisma"` |

### How it all fits together

```
prisma/schema.prisma   ← you write this
       │
       ├── npx prisma migrate dev   → creates SQL migration files + applies them
       ├── npx prisma generate      → generates node_modules/@prisma/client
       └── npx prisma studio        → opens GUI at localhost:5555

import { PrismaClient } from "@prisma/client"   ← fully typed from your schema
const prisma = new PrismaClient()
```

⚡ **Version note:** Prisma 6 (released late 2024) introduced **driver adapters** (stable), the new `prisma-client` generator alias, and significant performance improvements to relation queries. The examples below target Prisma 6+.

---

## 2. Setup & Initialization

### Install

```bash
# Install CLI (dev dependency) and the generated client (runtime dependency)
npm install prisma --save-dev
npm install @prisma/client

# Or with pnpm / yarn
pnpm add -D prisma
pnpm add @prisma/client
```

### Initialize

```bash
# Creates prisma/schema.prisma and .env with DATABASE_URL placeholder
npx prisma init

# Init targeting a specific provider
npx prisma init --datasource-provider postgresql
npx prisma init --datasource-provider sqlite
npx prisma init --datasource-provider mysql
```

This creates:
```
your-project/
├── prisma/
│   └── schema.prisma    ← define all models here
├── .env                 ← DATABASE_URL goes here (git-ignored)
└── node_modules/
    └── @prisma/client/  ← generated after `prisma generate`
```

### Set your DATABASE_URL in `.env`

```bash
# PostgreSQL
DATABASE_URL="postgresql://USER:PASSWORD@HOST:5432/DBNAME?schema=public"

# SQLite (file-based, great for dev/learning)
DATABASE_URL="file:./dev.db"

# MySQL
DATABASE_URL="mysql://USER:PASSWORD@HOST:3306/DBNAME"

# MongoDB
DATABASE_URL="mongodb+srv://USER:PASSWORD@cluster.mongodb.net/DBNAME"
```

### Generate the client (after any schema change)

```bash
npx prisma generate
```

⚡ **Version note (Prisma 6):** You can now use the shorter generator name `prisma-client` instead of `prisma-client-js`. Both work; `prisma-client-js` remains default for backwards compatibility.

---

## 3. The Schema File

The `prisma/schema.prisma` file is written in Prisma Schema Language (PSL). It has three block types: `datasource`, `generator`, and `model`.

### Full annotated example

```prisma
// prisma/schema.prisma

// ── 1. GENERATOR ────────────────────────────────────────────────────────────
generator client {
  provider = "prisma-client-js"   // generates @prisma/client
  // previewFeatures = ["fullTextSearch"]  // opt-in to preview features
}

// ── 2. DATASOURCE ───────────────────────────────────────────────────────────
datasource db {
  provider = "postgresql"         // postgresql | mysql | sqlite | mongodb | sqlserver
  url      = env("DATABASE_URL")  // reads from .env
}

// ── 3. ENUMS ────────────────────────────────────────────────────────────────
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
  // --- Scalar fields ---
  id        Int      @id @default(autoincrement())   // primary key, auto-int
  // Alternative: cuid / uuid
  // id     String   @id @default(cuid())
  // id     String   @id @default(uuid())

  email     String   @unique                        // unique constraint
  username  String   @unique @db.VarChar(50)        // native type modifier
  name      String?                                 // ? = optional (nullable)
  bio       String?  @db.Text                       // long text
  age       Int?
  score     Float    @default(0.0)
  isActive  Boolean  @default(true)
  role      Role     @default(USER)                 // enum field

  createdAt DateTime @default(now())                // auto-set on create
  updatedAt DateTime @updatedAt                     // auto-set on every update

  // --- Relation fields (no column in DB, just for the client) ---
  posts     Post[]                                  // one-to-many
  profile   Profile?                                // one-to-one
  comments  Comment[]

  // --- Map to a different table name in the DB ---
  @@map("users")                                    // table name in DB is "users"
}

model Profile {
  id        Int     @id @default(autoincrement())
  bio       String?
  avatarUrl String?

  // One-to-one: the FK lives on this side
  user      User    @relation(fields: [userId], references: [id])
  userId    Int     @unique                         // @unique enforces 1-to-1

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

  // FK back to User
  author      User       @relation(fields: [authorId], references: [id])
  authorId    Int

  // Many-to-many with Tag (implicit join table)
  tags        Tag[]

  // One-to-many with Comment
  comments    Comment[]

  // Compound unique constraint
  @@unique([title, authorId])
  // Compound index
  @@index([status, createdAt])
  @@map("posts")
}

model Tag {
  id    Int    @id @default(autoincrement())
  name  String @unique
  posts Post[]                                      // implicit many-to-many

  @@map("tags")
}

model Comment {
  id        Int      @id @default(autoincrement())
  content   String
  createdAt DateTime @default(now())

  author    User     @relation(fields: [authorId], references: [id])
  authorId  Int

  post      Post     @relation(fields: [postId], references: [id])
  postId    Int

  @@map("comments")
}
```

### Scalar Types Reference

| Prisma Type | TypeScript Type | Maps to (Postgres) |
|---|---|---|
| `String` | `string` | `TEXT` / `VARCHAR` |
| `Boolean` | `boolean` | `BOOLEAN` |
| `Int` | `number` | `INTEGER` |
| `BigInt` | `bigint` | `BIGINT` |
| `Float` | `number` | `DOUBLE PRECISION` |
| `Decimal` | `Decimal` (Prisma class) | `DECIMAL` |
| `DateTime` | `Date` | `TIMESTAMP` |
| `Json` | `JsonValue` | `JSONB` |
| `Bytes` | `Buffer` | `BYTEA` |

### Field Modifiers

| Modifier | Meaning | Example |
|---|---|---|
| `?` | Optional / nullable | `name String?` |
| `[]` | List / array | `posts Post[]` |
| (none) | Required | `email String` |

### Field Attributes

| Attribute | Description | Example |
|---|---|---|
| `@id` | Primary key | `id Int @id` |
| `@default(val)` | Default value | `@default(now())` / `@default(0)` |
| `@unique` | Unique constraint | `email String @unique` |
| `@relation(...)` | Define FK relation | `@relation(fields: [userId], references: [id])` |
| `@map("col")` | Map to a different column name | `firstName String @map("first_name")` |
| `@updatedAt` | Auto-set on update | `updatedAt DateTime @updatedAt` |
| `@ignore` | Exclude field from client | `legacyField String @ignore` |
| `@db.VarChar(n)` | Native DB type | `username String @db.VarChar(50)` |

### Block Attributes (on the model)

| Attribute | Description | Example |
|---|---|---|
| `@@id([a,b])` | Composite primary key | `@@id([userId, postId])` |
| `@@unique([a,b])` | Compound unique | `@@unique([title, authorId])` |
| `@@index([a,b])` | Compound index | `@@index([status, createdAt])` |
| `@@map("table")` | Map to different table name | `@@map("users")` |
| `@@ignore` | Exclude whole model from client | `@@ignore` |

---

## 4. Relations

### One-to-One

One `User` has one `Profile`. The FK (`userId`) lives on the "child" model (`Profile`). The `@unique` on the FK column enforces the 1-to-1 constraint.

```prisma
model User {
  id      Int      @id @default(autoincrement())
  profile Profile?                                  // no FK here
}

model Profile {
  id     Int  @id @default(autoincrement())
  user   User @relation(fields: [userId], references: [id])
  userId Int  @unique                               // UNIQUE enforces 1-to-1
}
```

### One-to-Many

One `User` writes many `Post`s. The FK (`authorId`) lives on the "many" side (`Post`).

```prisma
model User {
  id    Int    @id @default(autoincrement())
  posts Post[]                                      // virtual list, no column
}

model Post {
  id       Int  @id @default(autoincrement())
  author   User @relation(fields: [authorId], references: [id])
  authorId Int                                      // the actual FK column
}
```

### Many-to-Many — Implicit Join Table

Prisma manages the join table automatically. Use this when you don't need extra columns on the join record.

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
// Prisma creates a hidden "_PostToTag" table in the DB
```

### Many-to-Many — Explicit Join Table

Use this when you need extra data on the join record (e.g., a `role` on `UserTeam`).

```prisma
model User {
  id    Int        @id @default(autoincrement())
  teams UserTeam[]
}

model Team {
  id      Int        @id @default(autoincrement())
  name    String
  members UserTeam[]
}

// Explicit join model
model UserTeam {
  user      User     @relation(fields: [userId], references: [id])
  userId    Int
  team      Team     @relation(fields: [teamId], references: [id])
  teamId    Int
  role      String   @default("MEMBER")             // extra column!
  joinedAt  DateTime @default(now())

  @@id([userId, teamId])                            // composite PK
}
```

### Self-Relations

A model that references itself — e.g., employees with managers, or comment threads.

```prisma
model Employee {
  id         Int        @id @default(autoincrement())
  name       String
  managerId  Int?
  manager    Employee?  @relation("EmployeeManager", fields: [managerId], references: [id])
  reports    Employee[] @relation("EmployeeManager")
}

model Comment {
  id        Int       @id @default(autoincrement())
  content   String
  parentId  Int?
  parent    Comment?  @relation("CommentThread", fields: [parentId], references: [id])
  replies   Comment[] @relation("CommentThread")
}
```

### Referential Actions (cascade behaviour)

```prisma
model Post {
  author   User @relation(fields: [authorId], references: [id],
    onDelete: Cascade,     // delete post when user is deleted
    onUpdate: Cascade      // update FK when user PK changes
  )
  authorId Int
}
```

| Action | Behaviour |
|---|---|
| `Cascade` | Propagate delete/update to children |
| `Restrict` | Error if referenced record is deleted |
| `SetNull` | Set FK to null (field must be optional) |
| `SetDefault` | Set FK to field's default value |
| `NoAction` | DB decides (usually like Restrict) |

---

## 5. Migrations

### Development workflow

```bash
# Create a migration file AND apply it to the dev DB
npx prisma migrate dev

# Give it a descriptive name (shows in the migration filename)
npx prisma migrate dev --name add_user_role

# What it does:
# 1. Diffs schema.prisma vs the current DB state
# 2. Generates prisma/migrations/20240101_add_user_role/migration.sql
# 3. Applies the SQL to your dev database
# 4. Runs `prisma generate` to rebuild the client
```

### Production deployment

```bash
# Apply all pending migrations (never creates new ones)
# Safe to run in CI/CD
npx prisma migrate deploy
```

### Prototype / quick iteration (no migration files)

```bash
# Pushes schema directly to DB — no migration files created
# Great for rapid prototyping or MongoDB
npx prisma db push

# Force-reset the DB (DROPS ALL DATA, dev only!)
npx prisma migrate reset
```

### Inspect an existing database

```bash
# Pull the DB schema into schema.prisma (reverse engineer)
npx prisma db pull
```

### Migration file structure

```
prisma/
└── migrations/
    ├── 20240101120000_init/
    │   └── migration.sql        ← raw SQL Prisma generated
    ├── 20240110150000_add_user_role/
    │   └── migration.sql
    └── migration_lock.toml      ← locks the DB provider; do NOT edit
```

### Manual migration (custom SQL)

```bash
# 1. Create an empty migration file
npx prisma migrate dev --name seed_categories --create-only

# 2. Edit the generated migration.sql with your custom SQL
# 3. Apply it
npx prisma migrate dev
```

### Baseline an existing database

```bash
# When you have an existing DB and want to start using Prisma Migrate
npx prisma migrate diff \
  --from-empty \
  --to-schema-datamodel prisma/schema.prisma \
  --script > prisma/migrations/0_init/migration.sql

npx prisma migrate resolve --applied 0_init
```

---

## 6. Prisma Client CRUD

### Setup

```typescript
// lib/prisma.ts  (see Section 15 for the singleton pattern)
import { PrismaClient } from "@prisma/client"
export const prisma = new PrismaClient()
```

### CREATE

```typescript
// create — single record
const user = await prisma.user.create({
  data: {
    email: "alice@example.com",
    username: "alice",
    name: "Alice Smith",
    role: "ADMIN",
  },
})
// Returns: User object with all fields

// createMany — multiple records in one query (much faster than a loop)
const result = await prisma.user.createMany({
  data: [
    { email: "bob@example.com", username: "bob" },
    { email: "carol@example.com", username: "carol" },
  ],
  skipDuplicates: true,  // silently skip if unique constraint violated
})
// Returns: { count: 2 }

// createManyAndReturn — get back the created records (Prisma 5.14+)
// ⚡ Version note: available in Prisma 5.14+ / 6+
const users = await prisma.user.createManyAndReturn({
  data: [
    { email: "dave@example.com", username: "dave" },
  ],
})
// Returns: User[]
```

### READ

```typescript
// findUnique — by @id or @unique field; returns null if not found
const user = await prisma.user.findUnique({
  where: { email: "alice@example.com" },
})
// user is User | null

// findUniqueOrThrow — throws PrismaClientKnownRequestError if not found
const user = await prisma.user.findUniqueOrThrow({
  where: { id: 1 },
})

// findFirst — first matching record (with any where filter)
const draft = await prisma.post.findFirst({
  where: { status: "DRAFT" },
  orderBy: { createdAt: "desc" },
})

// findFirstOrThrow — same but throws if nothing found
const post = await prisma.post.findFirstOrThrow({
  where: { authorId: 1 },
})

// findMany — list of records
const posts = await prisma.post.findMany({
  where: { published: true },
  orderBy: { createdAt: "desc" },
  take: 10,   // LIMIT 10
  skip: 0,    // OFFSET 0
})
// Returns: Post[]
```

### UPDATE

```typescript
// update — single record by unique identifier
const updated = await prisma.user.update({
  where: { id: 1 },
  data: { name: "Alice Updated", role: "MODERATOR" },
})

// updateMany — update multiple records; returns count
const result = await prisma.post.updateMany({
  where: { authorId: 1, published: false },
  data: { status: "ARCHIVED" },
})
// result = { count: 3 }

// Atomic number operations (avoids race conditions)
const post = await prisma.post.update({
  where: { id: 42 },
  data: {
    viewCount: { increment: 1 },   // viewCount += 1
    // decrement: 1
    // multiply: 2
    // divide: 2
    // set: 100
  },
})
```

### UPSERT

```typescript
// upsert — update if exists, create if not
const user = await prisma.user.upsert({
  where: { email: "alice@example.com" },
  update: { name: "Alice Updated" },     // if found, run this
  create: {                               // if not found, run this
    email: "alice@example.com",
    username: "alice",
    name: "Alice Updated",
  },
})
```

### DELETE

```typescript
// delete — single record (throws if not found)
const deleted = await prisma.user.delete({
  where: { id: 1 },
})

// deleteMany — multiple records; returns count
const result = await prisma.post.deleteMany({
  where: { status: "ARCHIVED" },
})
// result = { count: 5 }
```

---

## 7. Querying: Filters, Select, Include, Pagination

### Where Filters

```typescript
// Equality (default)
prisma.user.findMany({ where: { role: "ADMIN" } })

// Operators
prisma.user.findMany({
  where: {
    age:      { gt: 18 },            // greater than
    score:    { gte: 4.5 },          // greater than or equal
    name:     { lt: "M" },           // less than (alphabetical)
    viewCount:{ lte: 1000 },         // less than or equal
    email:    { contains: "@gmail" }, // LIKE '%@gmail%'
    username: { startsWith: "a" },   // LIKE 'a%'
    bio:      { endsWith: ".dev" },   // LIKE '%.dev'
    name:     { not: null },          // IS NOT NULL
    id:       { in: [1, 2, 3] },     // IN (1,2,3)
    id:       { notIn: [4, 5] },     // NOT IN (4,5)
  },
})

// Logical combinators
prisma.post.findMany({
  where: {
    AND: [
      { published: true },
      { createdAt: { gte: new Date("2024-01-01") } },
    ],
  },
})

prisma.post.findMany({
  where: {
    OR: [
      { status: "PUBLISHED" },
      { status: "DRAFT", authorId: 1 },
    ],
  },
})

prisma.post.findMany({
  where: {
    NOT: { status: "ARCHIVED" },
  },
})

// Filter on relation fields
prisma.post.findMany({
  where: {
    author: { role: "ADMIN" },   // posts written by admins
  },
})

// some / every / none (for list relations)
prisma.user.findMany({
  where: {
    posts: {
      some: { published: true },   // at least one published post
      // every: { published: true } // all posts published
      // none: { status: "ARCHIVED" } // no archived posts
    },
  },
})
```

### String filter modes (case insensitive)

```typescript
prisma.user.findMany({
  where: {
    name: {
      contains: "alice",
      mode: "insensitive",   // PostgreSQL only
    },
  },
})
```

### Select (pick specific fields)

```typescript
// Returns a plain object with ONLY the selected fields (not a full User)
const user = await prisma.user.findUnique({
  where: { id: 1 },
  select: {
    id: true,
    email: true,
    name: true,
    // role: false (or just omit it)
  },
})
// TypeScript knows: { id: number; email: string; name: string | null }
```

### Include (eager-load relations)

```typescript
// Fetch user with all their posts
const user = await prisma.user.findUnique({
  where: { id: 1 },
  include: {
    posts: true,
    profile: true,
  },
})
// user.posts is Post[], user.profile is Profile | null

// Include with nested filtering
const user = await prisma.user.findUnique({
  where: { id: 1 },
  include: {
    posts: {
      where: { published: true },
      orderBy: { createdAt: "desc" },
      take: 5,
    },
  },
})

// Nested include (posts with their comments)
const user = await prisma.user.findUnique({
  where: { id: 1 },
  include: {
    posts: {
      include: {
        comments: {
          include: { author: true },
        },
      },
    },
  },
})
```

> **Note:** You cannot use `select` and `include` at the top level simultaneously. Use `select` with nested `select`/`include` inside instead.

```typescript
// Use select at top level, then include via nested select
const user = await prisma.user.findUnique({
  where: { id: 1 },
  select: {
    email: true,
    posts: {                       // relation inside select = include
      select: { title: true },
    },
  },
})
```

### OrderBy

```typescript
// Single field
prisma.post.findMany({ orderBy: { createdAt: "desc" } })

// Multiple fields
prisma.post.findMany({
  orderBy: [
    { status: "asc" },
    { createdAt: "desc" },
  ],
})

// Order by relation field
prisma.post.findMany({
  orderBy: { author: { name: "asc" } },
})

// Order by aggregated relation count
prisma.user.findMany({
  orderBy: { posts: { _count: "desc" } },
})
```

### Pagination

```typescript
// Offset pagination (skip + take)
const page = 2
const pageSize = 10

const posts = await prisma.post.findMany({
  skip: (page - 1) * pageSize,   // OFFSET
  take: pageSize,                 // LIMIT
  orderBy: { id: "asc" },
})

// Cursor-based pagination (better performance on large datasets)
// First page (no cursor)
const firstPage = await prisma.post.findMany({
  take: 10,
  orderBy: { id: "asc" },
})

// Next page: use the last item's id as the cursor
const lastId = firstPage[firstPage.length - 1].id

const secondPage = await prisma.post.findMany({
  take: 10,
  skip: 1,             // skip the cursor item itself
  cursor: { id: lastId },
  orderBy: { id: "asc" },
})
```

---

## 8. Nested Writes

Nested writes let you create/update related records in a single atomic operation.

### create (create a new related record)

```typescript
// Create user and profile in one query
const user = await prisma.user.create({
  data: {
    email: "alice@example.com",
    username: "alice",
    profile: {
      create: {
        bio: "Full-stack developer",
        avatarUrl: "https://example.com/avatar.jpg",
      },
    },
    posts: {
      create: [
        { title: "Hello World", content: "My first post" },
        { title: "Prisma is great", content: "Here is why..." },
      ],
    },
  },
  include: { profile: true, posts: true },
})
```

### connect (link to an existing record)

```typescript
// Create a post and connect it to an existing tag by its unique field
const post = await prisma.post.create({
  data: {
    title: "Learning Prisma",
    author: { connect: { id: 1 } },          // connect to existing user
    tags: {
      connect: [
        { name: "typescript" },              // connect by @unique field
        { id: 5 },                           // connect by @id
      ],
    },
  },
})
```

### connectOrCreate (connect if exists, create if not)

```typescript
const post = await prisma.post.create({
  data: {
    title: "New Post",
    author: { connect: { id: 1 } },
    tags: {
      connectOrCreate: [
        {
          where: { name: "prisma" },
          create: { name: "prisma" },
        },
        {
          where: { name: "orm" },
          create: { name: "orm" },
        },
      ],
    },
  },
})
```

### Nested update

```typescript
const user = await prisma.user.update({
  where: { id: 1 },
  data: {
    name: "Updated Name",
    profile: {
      upsert: {
        create: { bio: "New bio" },
        update: { bio: "Updated bio" },
      },
    },
    posts: {
      // update a specific related post
      update: {
        where: { id: 10 },
        data: { published: true },
      },
      // delete a specific related post
      delete: { id: 11 },
      // disconnect a tag from all posts (many-to-many)
      disconnect: [{ id: 5 }],
    },
  },
})
```

### set (replace all related records)

```typescript
// Replace all tags on a post with a new set
const post = await prisma.post.update({
  where: { id: 1 },
  data: {
    tags: {
      set: [{ id: 1 }, { id: 3 }],  // replaces ALL existing tag connections
    },
  },
})
```

---

## 9. Aggregations, GroupBy & Count

### count

```typescript
// Count all users
const total = await prisma.user.count()

// Count with a filter
const adminCount = await prisma.user.count({
  where: { role: "ADMIN" },
})

// Count a relation (number of posts per user)
const user = await prisma.user.findUnique({
  where: { id: 1 },
  include: {
    _count: {
      select: { posts: true, comments: true },
    },
  },
})
// user._count.posts: number
// user._count.comments: number
```

### aggregate

```typescript
const stats = await prisma.post.aggregate({
  where: { published: true },
  _count: true,              // total count
  _avg: { viewCount: true }, // average view count
  _sum: { viewCount: true }, // sum of view counts
  _min: { viewCount: true }, // minimum view count
  _max: { viewCount: true }, // maximum view count
})

console.log(stats._count)          // total posts
console.log(stats._avg.viewCount)  // avg views
```

### groupBy

```typescript
// Count posts grouped by status
const grouped = await prisma.post.groupBy({
  by: ["status"],
  _count: { id: true },
  _avg: { viewCount: true },
  orderBy: { _count: { id: "desc" } },
})
// [
//   { status: "PUBLISHED", _count: { id: 45 }, _avg: { viewCount: 234 } },
//   { status: "DRAFT",     _count: { id: 12 }, _avg: { viewCount: 0   } },
// ]

// HAVING equivalent: filter after grouping
const popularStatuses = await prisma.post.groupBy({
  by: ["status"],
  _count: { id: true },
  having: {
    id: { _count: { gt: 10 } },  // only groups with more than 10 posts
  },
})
```

---

## 10. Transactions

### Sequential transactions (most common)

All operations share one connection and commit atomically. If any fails, all roll back.

```typescript
// Pass an array of Prisma Client promises
const [user, post] = await prisma.$transaction([
  prisma.user.create({
    data: { email: "tx@example.com", username: "txuser" },
  }),
  prisma.post.create({
    data: { title: "Transaction post", authorId: 1 },
  }),
])

// Transfer example: debit one account, credit another
const [debit, credit] = await prisma.$transaction([
  prisma.account.update({
    where: { id: fromId },
    data: { balance: { decrement: amount } },
  }),
  prisma.account.update({
    where: { id: toId },
    data: { balance: { increment: amount } },
  }),
])
```

### Interactive transactions (for conditional logic)

Use when you need to read data and then decide what to write — e.g., checking a balance before debiting.

```typescript
const result = await prisma.$transaction(async (tx) => {
  // tx is a Prisma Client bound to this transaction
  const account = await tx.account.findUniqueOrThrow({
    where: { id: fromId },
  })

  if (account.balance < amount) {
    throw new Error("Insufficient funds")   // triggers rollback
  }

  const updated = await tx.account.update({
    where: { id: fromId },
    data: { balance: { decrement: amount } },
  })

  await tx.account.update({
    where: { id: toId },
    data: { balance: { increment: amount } },
  })

  return updated
})

// Transaction options
await prisma.$transaction(async (tx) => {
  // ...
}, {
  maxWait: 5000,   // max ms to wait to acquire a connection (default 2000)
  timeout: 10000,  // max ms for the transaction to complete (default 5000)
  isolationLevel: "Serializable",  // Prisma.TransactionIsolationLevel
})
```

---

## 11. Raw Queries

Use raw queries when Prisma's query builder can't express what you need (complex SQL, DB-specific functions).

### $queryRaw — SELECT (returns rows)

```typescript
import { Prisma } from "@prisma/client"

// Always use Prisma.sql template tag for parameterization — NEVER interpolate!
const email = "alice@example.com"

const users = await prisma.$queryRaw<{ id: number; email: string }[]>(
  Prisma.sql`SELECT id, email FROM users WHERE email = ${email}`
  //                                                    ^^^^^^^ safely parameterized
)

// With multiple params
const users = await prisma.$queryRaw<User[]>(
  Prisma.sql`
    SELECT * FROM users
    WHERE role = ${role}
    AND created_at > ${since}
    ORDER BY name ASC
    LIMIT ${limit}
  `
)
```

### $executeRaw — INSERT / UPDATE / DELETE (returns count)

```typescript
const count = await prisma.$executeRaw(
  Prisma.sql`
    UPDATE posts
    SET view_count = view_count + 1
    WHERE id = ${postId}
  `
)
// count = number of affected rows
```

### $queryRawUnsafe — DANGER: SQL injection risk

```typescript
// Only use when you control the entire string (no user input)
// Useful for dynamic column names that can't be parameterized
const results = await prisma.$queryRawUnsafe(
  `SELECT ${columnName} FROM users WHERE id = $1`,
  userId
)
// Even here: use numbered placeholders ($1, $2) for values
```

> **Safety rule:** Always prefer `Prisma.sql` template literals. The tagged template automatically parameterizes values. Never concatenate user input into SQL strings.

---

## 12. Client Extensions (and Legacy Middleware)

### Client Extensions (Prisma 4.16+ / stable in 5+)

Extensions are the modern, type-safe way to add custom logic to Prisma Client.

```typescript
import { PrismaClient } from "@prisma/client"

// ── Model extension: add custom methods to a model ───────────────────────────
const prisma = new PrismaClient().$extends({
  model: {
    user: {
      // Custom static method: prisma.user.findByEmail(email)
      async findByEmail(email: string) {
        return prisma.user.findUnique({ where: { email } })
      },
    },
    post: {
      async publish(id: number) {
        return prisma.post.update({
          where: { id },
          data: { published: true, status: "PUBLISHED" },
        })
      },
    },
  },
})

// Usage
const user = await prisma.user.findByEmail("alice@example.com")
await prisma.post.publish(42)


// ── Result extension: add computed fields to query results ───────────────────
const prismaWithComputed = new PrismaClient().$extends({
  result: {
    user: {
      fullName: {
        needs: { name: true, username: true },
        compute(user) {
          return user.name ?? user.username
        },
      },
    },
  },
})

const user = await prismaWithComputed.user.findUnique({ where: { id: 1 } })
console.log(user?.fullName)  // computed, not in DB


// ── Query extension: intercept / modify queries ──────────────────────────────
const softDeleteClient = new PrismaClient().$extends({
  query: {
    user: {
      async findMany({ args, query }) {
        // Automatically filter out soft-deleted records
        args.where = { ...args.where, deletedAt: null }
        return query(args)
      },
    },
  },
})
```

### Legacy Middleware ($use) — deprecated

```typescript
// ⚡ Version note: $use middleware is deprecated in Prisma 5+ and
// will be removed in a future version. Prefer extensions above.
prisma.$use(async (params, next) => {
  const before = Date.now()
  const result = await next(params)
  const after = Date.now()
  console.log(`Query ${params.model}.${params.action} took ${after - before}ms`)
  return result
})
```

---

## 13. Seeding the Database

### Setup seed script

```typescript
// prisma/seed.ts
import { PrismaClient } from "@prisma/client"
const prisma = new PrismaClient()

async function main() {
  // Upsert so re-running the seed is idempotent
  const admin = await prisma.user.upsert({
    where: { email: "admin@example.com" },
    update: {},
    create: {
      email: "admin@example.com",
      username: "admin",
      name: "Admin User",
      role: "ADMIN",
    },
  })

  const alice = await prisma.user.upsert({
    where: { email: "alice@example.com" },
    update: {},
    create: {
      email: "alice@example.com",
      username: "alice",
      name: "Alice Smith",
    },
  })

  // Seed posts
  await prisma.post.createMany({
    data: [
      { title: "Hello World", authorId: alice.id, published: true, status: "PUBLISHED" },
      { title: "Draft Post",  authorId: alice.id, published: false, status: "DRAFT" },
    ],
    skipDuplicates: true,
  })

  console.log("Seeded:", { admin, alice })
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect())
```

### Register the seed command in package.json

```json
{
  "prisma": {
    "seed": "ts-node --compiler-options {\"module\":\"CommonJS\"} prisma/seed.ts"
  }
}
```

Or with `tsx` (ESM-friendly, faster):

```json
{
  "prisma": {
    "seed": "tsx prisma/seed.ts"
  }
}
```

### Run the seed

```bash
# Run seed directly
npx prisma db seed

# migrate reset also runs seed automatically
npx prisma migrate reset
```

---

## 14. Generated Types & Type Safety

Prisma generates TypeScript types into `@prisma/client` based on your schema.

### Model types

```typescript
import { User, Post, Role } from "@prisma/client"
//       ^^^^  ^^^^  ^^^^
//       These are generated from your schema

const user: User = await prisma.user.findUniqueOrThrow({ where: { id: 1 } })
// user.id: number
// user.email: string
// user.role: Role   (enum)
// user.posts:       ← NOT here! Not included unless you use include:
```

### The Prisma namespace (utility types)

```typescript
import { Prisma } from "@prisma/client"

// Input types for create / update
type UserCreateInput = Prisma.UserCreateInput
type PostUpdateInput = Prisma.PostUpdateInput

// Where filter types
type UserWhereInput = Prisma.UserWhereUniqueInput

// Select return types
type UserWithPosts = Prisma.UserGetPayload<{
  include: { posts: true }
}>
// Equivalent to: User & { posts: Post[] }

type PostWithAuthorAndTags = Prisma.PostGetPayload<{
  include: { author: true; tags: true }
}>

// Select payload
type UserEmailOnly = Prisma.UserGetPayload<{
  select: { id: true; email: true }
}>
// { id: number; email: string }
```

### Reusable query argument types

```typescript
// Define a reusable include shape
const userWithPostsArgs = {
  include: { posts: true, profile: true },
} satisfies Prisma.UserArgs

// Helper function with correct return type
async function getUserWithPosts(id: number) {
  return prisma.user.findUniqueOrThrow({
    where: { id },
    ...userWithPostsArgs,
  })
}
// Return type is automatically: User & { posts: Post[]; profile: Profile | null }
```

### Enum types

```typescript
import { Role } from "@prisma/client"

function promoteToAdmin(userId: number) {
  return prisma.user.update({
    where: { id: userId },
    data: { role: Role.ADMIN },  // type-safe, no magic strings
  })
}
```

### Error types

```typescript
import { Prisma } from "@prisma/client"

try {
  await prisma.user.create({ data: { email: "dup@example.com", username: "x" } })
} catch (e) {
  if (e instanceof Prisma.PrismaClientKnownRequestError) {
    if (e.code === "P2002") {
      // Unique constraint violation
      console.log("Duplicate:", e.meta?.target)
    }
    if (e.code === "P2025") {
      // Record not found (for findUniqueOrThrow, etc.)
      console.log("Not found")
    }
  }
}
```

### Common Prisma Error Codes

| Code | Meaning |
|---|---|
| `P2000` | Value too long for field |
| `P2001` | Record not found in where clause |
| `P2002` | Unique constraint violation |
| `P2003` | Foreign key constraint violation |
| `P2025` | Record not found (delete / update) |
| `P2034` | Transaction conflict (retry-able) |

---

## 15. Using Prisma in Next.js / Serverless

### The problem: too many connections

In serverless environments (Next.js API routes, Vercel Functions, AWS Lambda), each cold start creates a new `PrismaClient` instance, each of which holds its own connection pool. Under load, this exhausts your database's connection limit.

### The singleton pattern (the standard fix)

```typescript
// lib/prisma.ts
import { PrismaClient } from "@prisma/client"

// In development, Next.js hot-reloads which re-runs this module repeatedly.
// We store the client on globalThis to reuse it across hot reloads.
// In production (one long-running process or true serverless), this is a no-op.

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient }

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === "development"
      ? ["query", "error", "warn"]
      : ["error"],
  })

if (process.env.NODE_ENV !== "production") {
  globalForPrisma.prisma = prisma
}
```

### Using in Next.js App Router (Server Components & Route Handlers)

```typescript
// app/users/page.tsx — Server Component, no boilerplate needed
import { prisma } from "@/lib/prisma"

export default async function UsersPage() {
  const users = await prisma.user.findMany({
    select: { id: true, name: true, email: true },
    orderBy: { createdAt: "desc" },
  })

  return (
    <ul>
      {users.map(u => <li key={u.id}>{u.name} — {u.email}</li>)}
    </ul>
  )
}
```

```typescript
// app/api/users/route.ts — Route Handler
import { prisma } from "@/lib/prisma"
import { NextResponse } from "next/server"

export async function GET() {
  const users = await prisma.user.findMany()
  return NextResponse.json(users)
}

export async function POST(req: Request) {
  const body = await req.json()
  const user = await prisma.user.create({ data: body })
  return NextResponse.json(user, { status: 201 })
}
```

### Connection pooling options

| Option | Description |
|---|---|
| **Prisma Accelerate** | Prisma's global edge cache + connection pooler (recommended for serverless / edge) |
| **PgBouncer** | External pooler for PostgreSQL; set `?pgbouncer=true` in DATABASE_URL |
| **Supabase connection pooler** | Built into Supabase dashboard; use the pooler URL |
| **Railway / Neon / PlanetScale** | Each has their own built-in pooling |

### Prisma Accelerate (Prisma's connection pooler + edge cache)

```bash
npm install @prisma/extension-accelerate
```

```typescript
// lib/prisma.ts — with Accelerate
import { PrismaClient } from "@prisma/client"
import { withAccelerate } from "@prisma/extension-accelerate"

export const prisma = new PrismaClient().$extends(withAccelerate())

// Use cacheStrategy on individual queries
const users = await prisma.user.findMany({
  cacheStrategy: {
    ttl: 60,        // cache for 60 seconds
    swr: 30,        // stale-while-revalidate 30 seconds
  },
})
```

```bash
# DATABASE_URL for Accelerate uses prisma:// protocol
DATABASE_URL="prisma://accelerate.prisma-data.net/?api_key=your_api_key"
```

### Always disconnect in scripts

```typescript
// In standalone scripts (not in a long-running server), disconnect when done
async function main() {
  const users = await prisma.user.findMany()
  console.log(users)
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect())   // ← always do this in scripts
```

---

## 16. Prisma Studio

Prisma Studio is a visual, browser-based GUI for your database. It requires no additional installation.

```bash
# Start Studio (opens at http://localhost:5555)
npx prisma studio
```

### What you can do in Studio

| Feature | Description |
|---|---|
| Browse tables | See all models as tabbed tables |
| View records | Paginated row viewer with type-aware display |
| Create records | Form-based record creation with validation |
| Edit records | Inline cell editing |
| Delete records | Single or bulk delete |
| Filter records | Basic filter by field value |
| Navigate relations | Click through to related records |
| Export | Copy to clipboard as JSON |

### Studio limitations

- No raw SQL editor (use `prisma db execute` or a DB client for that)
- No schema editor (edit `schema.prisma` in your code editor)
- Not designed for production use — it uses your `DATABASE_URL` directly

```bash
# Run Studio against a specific schema file
npx prisma studio --schema=./path/to/schema.prisma

# Run on a different port
npx prisma studio --port 5556
```

---

## 17. Tips, Tricks & Gotchas

### N+1 Problem

The most common Prisma performance mistake. Happens when you run a query in a loop:

```typescript
// ❌ BAD: N+1 — 1 query for users, then N queries for posts
const users = await prisma.user.findMany()
for (const user of users) {
  // This runs a SEPARATE query for every user!
  const posts = await prisma.post.findMany({ where: { authorId: user.id } })
}

// ✅ GOOD: 2 queries total via include
const users = await prisma.user.findMany({
  include: { posts: true },
})
// Now user.posts is already loaded for every user

// ✅ ALSO GOOD: If you only need IDs, use _count
const users = await prisma.user.findMany({
  include: { _count: { select: { posts: true } } },
})
```

### Global Client Pattern (never `new PrismaClient()` inline)

```typescript
// ❌ BAD: Creates a new connection pool on every import / request
const prisma = new PrismaClient()

// ✅ GOOD: Use the singleton from lib/prisma.ts everywhere
import { prisma } from "@/lib/prisma"
```

### Changing Enums in Migrations

Renaming or removing enum values is a breaking change. Do it in steps:

```
Step 1: Add the new value, keep the old one
Step 2: Migrate existing data to use the new value
Step 3: Remove the old value in a separate migration
```

Avoid renaming enum values directly — Prisma Migrate generates a DROP + CREATE which can fail if rows use the old value.

### select vs include — you can't use both at the top level

```typescript
// ❌ Type error — can't mix select and include at top level
await prisma.user.findUnique({
  where: { id: 1 },
  select: { email: true },
  include: { posts: true },  // Error!
})

// ✅ Use select everywhere, including for relations
await prisma.user.findUnique({
  where: { id: 1 },
  select: {
    email: true,
    posts: { select: { title: true } },   // select inside select = include
  },
})
```

### Always await Prisma queries

```typescript
// ❌ Forgot await — silently returns a Promise, not data
const user = prisma.user.findUnique({ where: { id: 1 } })
console.log(user.email)  // undefined

// ✅ Always await
const user = await prisma.user.findUnique({ where: { id: 1 } })
```

### findUnique vs findFirst

```typescript
// findUnique — ONLY works with @id or @unique fields; Prisma can optimize it
prisma.user.findUnique({ where: { email: "..." } })

// findFirst — works with ANY where clause; less optimized
prisma.user.findFirst({ where: { name: "Alice" } })
// Gotcha: if multiple Alices exist, you get one arbitrarily
```

### Decimal vs Float precision

```typescript
// Float loses precision for financial data
// ❌ AVOID for money
score Float   // JavaScript number — floating point imprecision

// ✅ USE for money / exact decimals
price Decimal @db.Decimal(10, 2)   // exact 10 digits, 2 decimal places
```

### JSON fields

```typescript
// Store arbitrary JSON
model Config {
  id       Int   @id @default(autoincrement())
  settings Json
}

// Write
await prisma.config.create({
  data: { settings: { theme: "dark", lang: "en" } },
})

// Filter inside JSON (PostgreSQL path operator)
await prisma.config.findMany({
  where: {
    settings: {
      path: ["theme"],
      equals: "dark",
    },
  },
})
```

### Auto-generate vs manual IDs

```typescript
// Auto integer (simple, but leaks count info)
id Int @id @default(autoincrement())

// CUID (collision-resistant, safe for URLs, sortable)
id String @id @default(cuid())

// UUID (industry standard, universal uniqueness)
id String @id @default(uuid())

// UUID v7 (sortable UUID, Prisma 5.8+)
// ⚡ Version note: available in Prisma 5.8+ / 6+
id String @id @default(uuid(7))
```

### Soft deletes pattern

```prisma
// Add to model
deletedAt DateTime?
```

```typescript
// Wrap all queries with a Client Extension (Section 12) to auto-filter
// Or use a helper
async function softDelete(id: number) {
  return prisma.user.update({
    where: { id },
    data: { deletedAt: new Date() },
  })
}

// Query only non-deleted
prisma.user.findMany({ where: { deletedAt: null } })
```

### Database connection URL safety

```bash
# .env — always in .gitignore
DATABASE_URL="postgresql://user:pass@host:5432/db"

# For special characters in passwords, percent-encode them:
# @ → %40, # → %23, : → %3A
DATABASE_URL="postgresql://user:p%40ssw0rd@host:5432/db"
```

### Migrate vs db push

| Command | Use for | Creates migration files | Safe for prod |
|---|---|---|---|
| `migrate dev` | Development | Yes | No |
| `migrate deploy` | Production CI/CD | No (applies existing) | Yes |
| `db push` | Prototyping / MongoDB | No | No (destructive!) |

### Prisma generate must run after schema changes

```bash
# In package.json scripts: add postinstall to ensure client is always generated
{
  "scripts": {
    "postinstall": "prisma generate"
  }
}
```

### Logging slow queries in development

```typescript
const prisma = new PrismaClient({
  log: [
    { level: "query", emit: "event" },
    { level: "warn",  emit: "stdout" },
    { level: "error", emit: "stdout" },
  ],
})

prisma.$on("query", (e) => {
  console.log(`Query: ${e.query}`)
  console.log(`Params: ${e.params}`)
  console.log(`Duration: ${e.duration}ms`)
})
```

---

## 18. Study Path

Work through these in order. Each builds on the last, and the projects make the concepts stick.

### Phase 1 — Foundations (Week 1)
1. Read Section 1 — understand the three parts and the schema-first approach
2. Follow Section 2 — set up a real project with SQLite (no server needed)
3. Study Section 3 — write a schema with 3–4 models, enums, and all field attribute types
4. Run `prisma migrate dev` and inspect the generated SQL
5. **Project:** Model a blog: `User`, `Post`, `Tag`, `Comment` — with enums for roles and status

### Phase 2 — Querying (Week 2)
6. Study Section 6 — implement all CRUD operations for your blog models
7. Study Section 7 — practice every filter operator; build a search function using `OR` + `contains`
8. Study Section 4 — add relations to your schema; practice all four relation types
9. Study Section 8 — create a post with nested tags and comments in one call
10. **Project:** REST API (Express or Hono) with full CRUD + filters + pagination for the blog

### Phase 3 — Advanced Features (Week 3)
11. Study Section 9 — build a dashboard endpoint that uses `groupBy` and `aggregate`
12. Study Section 10 — implement a "transfer" feature using interactive transactions
13. Study Section 11 — write a complex report query in raw SQL; use `Prisma.sql` safely
14. Study Section 14 — use `Prisma.PostGetPayload` to type your API response shapes
15. **Project:** Add an analytics endpoint: posts per day for the last 30 days using `groupBy`

### Phase 4 — Next.js & Production (Week 4)
16. Study Section 15 — integrate the singleton pattern into a Next.js App Router project
17. Study Section 5 — practice the full migration workflow: dev → prod with `migrate deploy`
18. Study Section 13 — write an idempotent seed script; test it with `prisma migrate reset`
19. Study Section 12 — add a soft-delete extension using `$extends`
20. **Project:** Full-stack Next.js blog with Server Components, Route Handlers, Prisma, and seed data

### Phase 5 — Polish & Mastery
21. Study Section 17 — audit your project for N+1 queries using query logging
22. Add Prisma Accelerate or PgBouncer for connection pooling
23. Write a GitHub Actions workflow that runs `prisma migrate deploy` on push to main
24. Explore Prisma's `fullTextSearch` preview feature (PostgreSQL / MySQL)
25. **Capstone Project:** Multi-tenant SaaS starter with: JWT auth, role-based access, soft deletes, migrations, seed, Studio walkthrough, connection pooling configured

### Key reference commands cheatsheet

```bash
npx prisma init                    # scaffold schema + .env
npx prisma generate                # regenerate client after schema change
npx prisma migrate dev             # create + apply migration in dev
npx prisma migrate dev --name foo  # same with a name
npx prisma migrate deploy          # apply pending migrations (prod)
npx prisma migrate reset           # drop DB, re-apply all, seed (dev only!)
npx prisma db push                 # push schema to DB without migration files
npx prisma db pull                 # reverse-engineer DB into schema.prisma
npx prisma db seed                 # run prisma.seed script
npx prisma studio                  # open GUI at localhost:5555
npx prisma format                  # format schema.prisma
npx prisma validate                # validate schema without generating
```
