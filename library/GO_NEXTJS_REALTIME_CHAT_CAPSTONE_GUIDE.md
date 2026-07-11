# Real-Time Chat Capstone (Gin · Ent · sqlc · goose · pgx · coder/websocket · JWT/Argon2 · Air + Next.js) — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Developers who have met each tool in this stack on its own and now want to see them **composed into one real, production-shaped, banking-grade real-time chat application** — the "put it all together" reference. On the backend we run a **Go API** (Gin for HTTP, `golang-jwt/jwt` v5 + `x/crypto/argon2` for auth, **Ent** for the relation graph, **sqlc** for hot-path type-safe SQL, **goose** for migrations, **pgx v5** as the one shared pool, **Air** for the dev loop, and **`github.com/coder/websocket`** for realtime). On the frontend we run **Next.js 16 + React 19 + TanStack Query v5 + TypeScript**. Together they deliver the full feature set of a modern chat product: **register/login, rooms & channels, direct messages, presence, typing indicators, message history with keyset pagination, unread counts, and read receipts** — with realtime fan-out that scales across many server nodes via a **Redis pub/sub backplane**. This is an **explain-first** guide: every concept is taught in prose first (what it is, *why* the architecture pushes you this way, when to reach for it, how it works, the key parameters, best practices, and the **security implications**) and only then shown as heavily-commented, runnable code. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced. It **assumes and cross-references** the single-tool guides but is self-contained enough to follow top to bottom.
>
> **Version note:** Targets **Go 1.25 / 1.26**, **`github.com/gin-gonic/gin` v1.10+**, **`github.com/coder/websocket` v1.8.x**, **`entgo.io/ent` v0.14+**, **`sqlc` v1.31+** (config `version: "2"`, `sql_package: "pgx/v5"`), **`github.com/pressly/goose/v3` v3.27+**, **`github.com/jackc/pgx/v5`**, **`golang-jwt/jwt` v5**, **`golang.org/x/crypto/argon2`**, and **Air** for live reload. Frontend: **Next.js 16 (App Router)** on **React 19**, **`@tanstack/react-query` v5**, **TypeScript 5.x**. Infra: **PostgreSQL** (Supabase or plain), **Redis 7/8**, **Nginx**, **Docker** — all current as of **2026**. Fast-moving APIs are flagged with **⚡**. The author is on **Windows 11**, so cross-platform notes (`.exe`, path separators, shells) are called out where they bite.
>
> **This guide's place in the library — it is the integration capstone that composes the whole Go + Next.js realtime stack.** It does not re-teach each tool from scratch; it teaches you to **wire them together** and shows the integration glue the single-tool guides leave out (one pool feeding two query layers, the WebSocket auth handshake from a browser, the Hub + Redis backplane, the optimistic cache with idempotent replay). Read each component guide alongside the section that uses it:
> - [Coder WebSocket](GO_CODER_WEBSOCKETS_GUIDE.md) — the Accept/Dial API, read/write pumps, the one-reader rule, CSWSH defense.
> - [Goose migrations](GO_GOOSE_MIGRATIONS_GUIDE.md) — SQL migrations, embedding, advisory-locked runners.
> - [sqlc + goose](GO_SQLC_GOOSE_GUIDE.md) — type-safe queries generated from the goose schema.
> - [Go ent ORM](GO_ENT_ORM_GUIDE.md) — schema-as-code, edges, eager loading, hooks.
> - [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md) — the password-hashing & token threat models in full.
> - [Go Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) — the router, `*gin.Context`, middleware, binding.
> - [Go (Gin+Ent) + Next.js Full-Stack](GO_GIN_NEXTJS_REALTIME_FULLSTACK_GUIDE.md) — the sibling capstone (CRUD + realtime dashboard) this one deepens.
> - [Next.js 16](NEXTJS_16_GUIDE.md) · [React 19](REACT_19_GUIDE.md) · [TanStack Query](TANSTACK_QUERY_GUIDE.md) — the frontend half.
> - [PostgreSQL](POSTGRESQL_GUIDE.md) · [Redis](REDIS_GUIDE.md) · [Nginx](NGINX_GUIDE.md) · [Docker](DOCKER_GUIDE.md) — the data tier and the edge.

---

## Table of Contents

1. [What We're Building & Why This Stack](#1-what-were-building--why-this-stack) **[B]**
2. [The Data-Layer Division of Labor — One Pool, Two Query Layers](#2-the-data-layer-division-of-labor--one-pool-two-query-layers) **[I/A]**
3. [Project Layout, Config & Graceful Shutdown](#3-project-layout-config--graceful-shutdown) **[B/I]**
4. [The Database Schema via goose Migrations](#4-the-database-schema-via-goose-migrations) **[I]**
5. [Ent Schema — The Relation Graph](#5-ent-schema--the-relation-graph) **[I]**
6. [sqlc — Type-Safe Hot-Path Queries](#6-sqlc--type-safe-hot-path-queries) **[I/A]**
7. [Authentication — Argon2id, JWT Access & Refresh Rotation](#7-authentication--argon2id-jwt-access--refresh-rotation) **[I/A]**
8. [RBAC & Authorization — Room Roles and IDOR Defense](#8-rbac--authorization--room-roles-and-idor-defense) **[I/A]**
9. [The REST API with Gin](#9-the-rest-api-with-gin) **[I]**
10. [Realtime with coder/websocket — The Hub Architecture](#10-realtime-with-coderwebsocket--the-hub-architecture) **[A]**
11. [The Realtime Protocol — Envelope & Events](#11-the-realtime-protocol--envelope--events) **[I/A]**
12. [The Message Send Flow End-to-End](#12-the-message-send-flow-end-to-end) **[A]**
13. [Presence & Typing Indicators](#13-presence--typing-indicators) **[A]**
14. [Read Receipts & Unread Counts](#14-read-receipts--unread-counts) **[I/A]**
15. [Scaling Out — The Redis Pub/Sub Backplane & Nginx](#15-scaling-out--the-redis-pubsub-backplane--nginx) **[A]**
16. [Next.js Frontend — Setup & Auth](#16-nextjs-frontend--setup--auth) **[B/I]**
17. [Frontend REST Data Layer with TanStack Query](#17-frontend-rest-data-layer-with-tanstack-query) **[I]**
18. [The Frontend WebSocket Client](#18-the-frontend-websocket-client) **[A]**
19. [The Chat UI with React 19](#19-the-chat-ui-with-react-19) **[I/A]**
20. [Frontend RBAC — Mirroring the Server](#20-frontend-rbac--mirroring-the-server) **[I]**
21. [Banking-Grade Security Hardening](#21-banking-grade-security-hardening) **[A]**
22. [Observability & Testing](#22-observability--testing) **[A]**
23. [Deployment — Docker, Compose & Migrations-on-Deploy](#23-deployment--docker-compose--migrations-on-deploy) **[A]**
24. [Maintainability — Contracts & Keeping Tools in Sync](#24-maintainability--contracts--keeping-tools-in-sync) **[I/A]**
25. [End-to-End Trace — A User Sends a Message](#25-end-to-end-trace--a-user-sends-a-message) **[A]**
26. [Gotchas & Best Practices](#26-gotchas--best-practices) **[A]**
27. [Study Path & Build-to-Learn Extensions](#27-study-path--build-to-learn-extensions)

---

## 1. What We're Building & Why This Stack

### 1.1 The product, in one paragraph **[B]**

We are building **one real-time chat application** — think a self-hosted Slack/Discord core — split into two separately deployed programs: a **Go API service** and a **Next.js browser client**. A user registers and logs in; they see a list of **rooms** (public channels and private groups) and **direct-message conversations** they belong to; they open a room and see its **message history** loaded newest-first with infinite scroll; they type and everyone in the room sees the message **instantly**; they see **who is online** (presence) and **who is typing**; the sidebar shows **unread badges**; and when they scroll to the bottom the app sends a **read receipt** so others can see their message was seen and the unread count resets. Everything that a user *initiates and waits for* goes over **REST**; everything that must be **pushed** to them the instant it happens goes over a single **WebSocket**.

### 1.2 The feature set we will actually implement **[B]**

| Feature | Transport | Backed by | Section |
|---|---|---|---|
| Register / login / refresh / logout | REST | Argon2id + JWT + `refresh_tokens` table | [§7](#7-authentication--argon2id-jwt-access--refresh-rotation) |
| List rooms & DMs you belong to | REST | Ent (membership graph) | [§9](#9-the-rest-api-with-gin) |
| Create room, invite/add members, set roles | REST | Ent (owner/admin/member) | [§8](#8-rbac--authorization--room-roles-and-idor-defense) |
| Message history (newest-first, infinite scroll) | REST | **sqlc keyset pagination** | [§6](#6-sqlc--type-safe-hot-path-queries), [§9](#9-the-rest-api-with-gin) |
| Send a message (persist + fan-out) | WebSocket | sqlc insert in a tx + Hub broadcast | [§12](#12-the-message-send-flow-end-to-end) |
| Presence (who's online) | WebSocket | in-memory + Redis TTL keys | [§13](#13-presence--typing-indicators) |
| Typing indicators | WebSocket | ephemeral (never persisted) | [§13](#13-presence--typing-indicators) |
| Unread counts | REST + WS | **sqlc** count vs. last-read | [§14](#14-read-receipts--unread-counts) |
| Read receipts | WebSocket | `read_receipts` table + broadcast | [§14](#14-read-receipts--unread-counts) |
| Cluster-wide broadcast | internal | **Redis pub/sub backplane** | [§15](#15-scaling-out--the-redis-pubsub-backplane--nginx) |

### 1.3 The system in one picture **[B]**

```
┌──────────────────────────────┐         ┌───────────────────────────────────────┐
│  Next.js 16 Chat Client       │         │   Go API Service (Gin)                │
│  (the browser app)            │  HTTPS  │  ┌─────────────────────────────────┐  │
│  • Login / Register           │ ──REST──▶  │ Gin router + Auth/RBAC middleware│  │
│  • Room list + unread badges  │ ◀──JSON─┤  │ Ent (users, rooms, memberships) │  │
│  • Message list (infinite)    │         │  │ sqlc (messages, history, unread)│  │
│  • Composer + typing          │   WSS   │  │ Argon2id + JWT (golang-jwt v5)  │  │
│  • Presence dots              │ ◀═realtime═ │ coder/websocket Hub             │  │
│  • TanStack Query cache       │  events │  └──────┬──────────────────┬────────┘  │
└──────────────────────────────┘         └─────────┼──────────────────┼───────────┘
                                                    │ pgx v5 pool      │ redis pub/sub
                                                    ▼                  ▼
                                          ┌──────────────────┐  ┌──────────────┐
                                          │  PostgreSQL      │  │  Redis 7/8   │
                                          │  (goose schema)  │  │ backplane+   │
                                          └──────────────────┘  │ presence TTL │
                                                                └──────────────┘
```

Two channels connect client to server, and it is worth being crisp about the split because *the entire architecture flows from it*:

1. **REST over HTTPS** — the request/response channel for anything the user initiates and can wait a round-trip for: authentication, listing rooms, loading a page of history, creating a room, inviting a member. Stateless; every request carries a short-lived **access token**.
2. **A single WebSocket over WSS** — one long-lived, full-duplex connection per open tab. The client sends *live* events over it (a new message, "I'm typing", "I've read up to here") and the server **pushes** events the user did not ask for at that instant (someone else's message, a presence change, a read receipt). This is what makes chat feel *live*.

The database and Redis are reached **only by the Go service**. The browser never holds a DB credential — a non-negotiable trust boundary we return to constantly and formalize in [§21](#21-banking-grade-security-hardening).

### 1.4 Why REST *and* WebSocket, not one or the other **[B]**

A recurring beginner instinct is "it's realtime, put everything on the WebSocket." Resist it. History loading is a paginated, cacheable, retry-able **query** — exactly what HTTP and TanStack Query are superb at (dedupe, caching, infinite queries, back/forward). Message *sending and receiving* is a push problem HTTP is bad at. The right design uses each transport for what it is good at and **keeps them consistent through one cache**: REST populates the TanStack Query cache; WebSocket events *patch* the same cache. The UI reads only from the cache and therefore can never show REST and WS disagreeing. We implement exactly this in [§17](#17-frontend-rest-data-layer-with-tanstack-query)–[§18](#18-the-frontend-websocket-client).

### 1.5 Why this exact backend stack **[B]**

Each tool earns its place; the honest one-line justification and its deep-dive guide:

| Concern | Choice | Why (short) | Deep-dive |
|---|---|---|---|
| HTTP framework | **Gin** | Fast radix router, middleware chain, binding/validation — the Go REST default | [Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) |
| Migrations / schema truth | **goose** | Plain SQL migrations, embeddable, advisory-lockable; the single source of truth for the schema | [Goose](GO_GOOSE_MIGRATIONS_GUIDE.md) |
| Hot-path queries | **sqlc** | Generates type-safe Go from SQL you write and review; perfect for the message insert, keyset history, unread counts | [sqlc + goose](GO_SQLC_GOOSE_GUIDE.md) |
| Relation-graph CRUD | **Ent** | Schema-as-Go-code + codegen; graph traversal & eager-loading for users/rooms/memberships/invites | [Ent](GO_ENT_ORM_GUIDE.md) |
| DB driver / pool | **pgx v5** | The fastest, most correct Postgres driver for Go; **one** `*pgxpool.Pool` shared by sqlc *and* Ent | [PostgreSQL](POSTGRESQL_GUIDE.md) |
| Password hashing | **Argon2id** | Memory-hard PHC-winner; current best practice | [JWT+Argon2](GO_JWT_ARGON2_GUIDE.md) |
| Auth tokens | **JWT access + rotating refresh** | Access in memory, refresh in an HttpOnly cookie with reuse detection | [JWT+Argon2](GO_JWT_ARGON2_GUIDE.md) |
| Realtime | **coder/websocket** | Modern, context-first WS library; concurrency-safe writes; CSWSH-safe by default | [Coder WS](GO_CODER_WEBSOCKETS_GUIDE.md) |
| Cross-node fan-out | **Redis pub/sub** | Lets every WS node broadcast to the whole cluster; also holds presence TTLs | [Redis](REDIS_GUIDE.md) |
| Dev loop | **Air** | Recompiles on save; we hook `sqlc generate` + `goose up` into it | this guide, [§23](#23-deployment--docker-compose--migrations-on-deploy) |

> **⚡ Version note — coder/websocket vs. gorilla/websocket.** The sibling capstone uses `gorilla/websocket`; this one deliberately uses **`github.com/coder/websocket`** (formerly `nhooyr.io/websocket`) to teach the modern, context-first API. The key behavioral differences you must internalize: `Accept` **blocks cross-origin requests by default** (you *opt in* to origins via `AcceptOptions.OriginPatterns`), every method except `Read`/`Reader` is concurrency-safe, and JSON helpers live in a separate `wsjson` package. See [§10](#10-realtime-with-coderwebsocket--the-hub-architecture).

### 1.6 Why this exact frontend stack **[B]**

**Next.js 16** gives us the App Router, a fast dev server, and a production build/deploy story; we use it mostly as a **client-rendered SPA with route guards** because a chat app is intensely interactive and session-bound (there is little to server-render for a logged-in socket app). **React 19** brings the modern hooks and `use`/transitions we lean on for the composer and optimistic sends. **TanStack Query v5** is the linchpin: it is the single cache that both REST and the WebSocket write into, giving us infinite history, background refetch on reconnect, and optimistic mutations for free. **TypeScript** keeps the wire contract (the message envelope, the DTOs) honest across the network boundary — the same field names appear in Go structs and TS interfaces, and drift is the #1 source of realtime bugs ([§24](#24-maintainability--contracts--keeping-tools-in-sync)).

### 1.7 The domain model we will use everywhere **[B]**

The *same* entities, identities, and envelope appear in every section. Memorize this; we never deviate.

```
users                 rooms                    room_members              messages
─────────             ─────────                ─────────────             ─────────
id (uuid pk)          id (uuid pk)             room_id  (fk)             id (uuid pk)
email (unique)        kind (channel|dm)        user_id  (fk)            room_id (fk)
username              name (null for dm)       role (owner|admin|member) sender_id (fk)
password_hash         created_by (fk users)    joined_at                 body (text)
created_at            created_at               last_read_message_id      client_msg_id (uuid)
                                                                          created_at

refresh_tokens                     read_receipts
─────────                          ─────────
id (uuid pk)                       room_id  (fk)      ← composite pk (room_id,user_id)
user_id (fk)                       user_id  (fk)
token_hash (sha-256)               message_id (fk)   ← last message this user has read
family_id (uuid)                   read_at
expires_at, revoked_at
```

Two `kind`s of room unify channels and DMs: a **channel** has a `name` and many members; a **direct message** is just a `room` with `kind='dm'`, `name=NULL`, and exactly two members. This is the single most important modeling decision in the app — it means *every* realtime code path (send, history, unread, receipts, presence) works identically for channels and DMs, and we never write a parallel "DM system." We revisit it in [§4.4](#44-why-dms-are-just-rooms).

### 1.8 The realtime envelope, once **[B]**

Every WebSocket frame in either direction is a JSON object with the same shape. We define it precisely in [§11](#11-the-realtime-protocol--envelope--events); here is the shape to hold in your head:

```jsonc
{
  "v": 1,                       // protocol version — bump on breaking changes
  "type": "message.new",        // the event name (dot-namespaced)
  "room_id": "0f8c…",           // which room this concerns (omitted for global events)
  "client_msg_id": "b1a2…",     // client-generated UUID for idempotency (send/ack only)
  "payload": { /* type-specific */ }
}
```

Events: `message.new`, `message.ack`, `message.error`, `presence.update`, `typing.start`, `typing.stop`, `receipt.read`. That is the entire realtime vocabulary of the app.

### 1.9 How to read this guide **[B]**

Sections 2–15 are the **backend**, front to back: data layer → schema → auth → REST → realtime → scale. Sections 16–20 are the **frontend**. Sections 21–24 are cross-cutting production concerns. [§25](#25-end-to-end-trace--a-user-sends-a-message) walks a single message through *every* layer both directions — read it twice, once now for the map and once at the end for the detail. If you have already read the component guides, you can skim 4–6 (schema/queries) and spend your energy on 10–15 (the Hub, the protocol, the send flow, presence, the backplane), which is where the integration lessons live.

---

## 2. The Data-Layer Division of Labor — One Pool, Two Query Layers

This is the **central architecture lesson** of the guide and the reason it exists, so we teach it before any schema. Most tutorials pick *one* data-access tool and marry it. Real systems frequently mix them, and doing so *cleanly* is a skill. We will run **goose, sqlc, and Ent together over one pgx pool**, and — importantly — we will be honest that **many apps need only one of these**. The chat app is a case where each tool's strength is genuinely load-bearing, which is what justifies the extra moving parts.

### 2.1 The four tools and their non-overlapping jobs **[I]**

Think of it as four responsibilities, each owned by exactly one tool:

- **goose = the schema's single source of truth.** Every table, column, index, constraint, enum, and trigger is defined in versioned SQL migration files. Nothing else is allowed to create or alter schema. Not sqlc (it only *reads* the schema to generate types), not Ent (its auto-migration is **turned off**). If it isn't in a goose migration, it does not exist. This gives you one linear, reviewable, forward-and-back history of your database, deployable with an advisory lock so a rolling deploy of N app instances migrates exactly once. See [§4](#4-the-database-schema-via-goose-migrations) and the [Goose guide](GO_GOOSE_MIGRATIONS_GUIDE.md).

- **sqlc = the hot path and the hard queries.** The message `INSERT`, the keyset-paginated history read, the unread-count aggregate, the "recent conversations" list — these are performance- and correctness-critical, and you want to *see and review the exact SQL* and get a compile error if a column changes. sqlc reads your goose migrations as the schema and your `.sql` query files, and generates Go functions with typed params and rows. No reflection, no query builder, no surprises in the plan. See [§6](#6-sqlc--type-safe-hot-path-queries) and the [sqlc + goose guide](GO_SQLC_GOOSE_GUIDE.md).

- **Ent = the relation graph and RBAC CRUD.** Users, rooms, memberships, roles, invitations — data whose *shape is a graph* and whose operations are "load this user with their rooms and each room's members," "add a member with role admin," "is this user a member of that room?" Ent's typed edges and eager-loading (`WithMembers`, `WithRooms`) make graph traversal and the authorization checks in [§8](#8-rbac--authorization--room-roles-and-idor-defense) clean and hard to get wrong. See [§5](#5-ent-schema--the-relation-graph) and the [Ent guide](GO_ENT_ORM_GUIDE.md).

- **pgx v5 = the one connection pool everything shares.** There is exactly **one** `*pgxpool.Pool` in the process. sqlc talks to it directly. Ent talks to the *same* pool through a thin `database/sql` shim. goose runs its migrations over the same pool. One pool means one place to size connections, one health check, one set of metrics, and no double-counting against Postgres's connection limit.

### 2.2 Why not just one tool? An honest accounting **[I]**

You could build this whole app with **only Ent** (it can do raw SQL for the hard queries) or **only sqlc** (you'd hand-write the graph queries and joins). Both are legitimate, and for a smaller app you should pick one and move on. We combine them because the chat domain splits cleanly along the exact seam where each shines:

- The **graph/RBAC** side (who is in what room with which role, invitations, "list my conversations with their members") is *tedious and error-prone in hand-written SQL* — lots of joins, N+1 traps, and authorization logic. Ent removes that pain.
- The **hot path** (insert a message; read the last 50 before a cursor; count unread across a user's rooms) is *exactly where an ORM's abstraction hurts*: you want a specific index used, a specific keyset predicate, a batch insert, and a `numeric` that never becomes a float. sqlc gives you the SQL verbatim with types bolted on.

The cost is real: two code-gen steps, two mental models, and a rule everyone must remember — **schema changes go in goose, then regenerate both**. We make that cheap by wiring `goose up` + `sqlc generate` + `ent generate` into the [Air](#23-deployment--docker-compose--migrations-on-deploy) dev loop so they never drift. If your team won't hold that discipline, use one tool. For this guide, the two-layer split *is* one of the lessons.

> **Best practice — draw the line by *ownership of the write*, not by table.** A table can be *read* by both layers, but each row's *writes* should have one owner. Here: Ent owns writes to `users`, `rooms`, `room_members` (membership/roles/invites). sqlc owns writes to `messages` and `read_receipts` (the hot path). Both may *read* any table. This keeps invariants (e.g., "a membership role is valid") enforced in one place while letting the fast path stay fast.

### 2.3 One pool, two query layers — the wiring **[I/A]**

Here is the code that makes it real: build one `pgxpool.Pool`, hand it to sqlc directly, and hand the *same* pool to Ent through `stdlib.OpenDBFromPool`. Ent's schema migration is off because goose owns the schema.

```go
// internal/db/db.go
package db

import (
	"context"
	"fmt"
	"time"

	"entgo.io/ent/dialect"
	entsql "entgo.io/ent/dialect/sql"
	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/jackc/pgx/v5/stdlib" // bridges a pgxpool to database/sql

	"chat/ent"          // Ent generated client
	gen "chat/db/gen"   // sqlc generated package
)

// Store bundles the two query layers over the ONE shared pool.
type Store struct {
	Pool *pgxpool.Pool // the single source of connections
	Q    *gen.Queries  // sqlc: hot-path & complex read queries
	Ent  *ent.Client   // Ent: relation-graph CRUD & RBAC
}

func Open(ctx context.Context, dsn string) (*Store, error) {
	// 1) Configure and open the ONE pool. Everything else borrows from it.
	cfg, err := pgxpool.ParseConfig(dsn)
	if err != nil {
		return nil, fmt.Errorf("parse dsn: %w", err)
	}
	cfg.MaxConns = 20                       // size for your Postgres + node count
	cfg.MinConns = 2                        // keep a couple warm
	cfg.MaxConnLifetime = time.Hour         // recycle to avoid stale server-side state
	cfg.MaxConnIdleTime = 5 * time.Minute
	cfg.HealthCheckPeriod = time.Minute

	pool, err := pgxpool.NewWithConfig(ctx, cfg)
	if err != nil {
		return nil, fmt.Errorf("open pool: %w", err)
	}
	if err := pool.Ping(ctx); err != nil { // fail fast at boot
		pool.Close()
		return nil, fmt.Errorf("ping: %w", err)
	}

	// 2) sqlc gets the pool DIRECTLY. gen.New accepts a DBTX, which *pgxpool.Pool
	//    satisfies (Exec/Query/QueryRow). For a tx you call q.WithTx(pgx.Tx).
	queries := gen.New(pool)

	// 3) Ent gets the SAME pool via a database/sql shim. stdlib.OpenDBFromPool
	//    wraps the pgxpool as a *sql.DB WITHOUT opening new connections of its own.
	sqlDB := stdlib.OpenDBFromPool(pool)
	drv := entsql.OpenDB(dialect.Postgres, sqlDB)
	entClient := ent.NewClient(ent.Driver(drv))
	// NOTE: we deliberately DO NOT call entClient.Schema.Create(...) — goose owns
	// the schema. Ent only issues DML (inserts/updates/selects), never DDL.

	return &Store{Pool: pool, Q: queries, Ent: entClient}, nil
}

// Close releases everything. Closing the Ent client closes its *sql.DB wrapper,
// but the underlying pgxpool must be closed exactly once — we own it here.
func (s *Store) Close() {
	_ = s.Ent.Close() // closes the sql.DB shim (does not double-close the pool)
	s.Pool.Close()    // the real shutdown of the connections
}
```

Two subtleties worth calling out now because they cause confusing bugs later:

1. **`stdlib.OpenDBFromPool(pool)` does not create a second pool.** It wraps your existing `pgxpool` so `database/sql` (and thus Ent) draws from the same connections. If you instead called `sql.Open("pgx", dsn)` you would have *two* independent pools both counting against Postgres's `max_connections` — the classic "why am I out of connections at half the load I expected" bug.
2. **Transactions do not span the two layers.** A `pgx.Tx` you begin for a sqlc insert is not visible to Ent, and vice versa. That is fine here because our write-ownership rule ([§2.2](#22-why-not-just-one-tool-an-honest-accounting)) means a single business operation writes through *one* layer. The one place we take a transaction — the message send — is entirely sqlc ([§12](#12-the-message-send-flow-end-to-end)). If you ever truly need a cross-layer transaction, you must drop to a shared `pgx.Tx` and give it to *both* `q.WithTx(tx)` and an Ent client built on that tx — possible but a smell; prefer to keep operations single-layer.

### 2.4 The mental model to keep **[I]**

> **goose writes the schema. Ent walks the graph. sqlc runs the hot path. pgx owns the sockets.** When you add a feature, ask: *is this a schema change?* → goose migration first. *Is this graph/RBAC CRUD?* → Ent. *Is this a performance-critical or gnarly query?* → sqlc. *Where do connections come from?* → the one pool. Hold that and the rest of the guide slots into place.

---

## 3. Project Layout, Config & Graceful Shutdown

### 3.1 The monorepo layout **[B]**

One repository, two top-level apps: `/server` (Go) and `/web` (Next.js). Keeping them together means one PR can change a wire contract on both sides atomically — critical when the message envelope evolves.

```
chat/
├─ server/
│  ├─ cmd/
│  │  └─ api/main.go            # entrypoint: wire config→store→hub→router→http.Server
│  ├─ internal/
│  │  ├─ config/config.go       # typed env config, fail-fast
│  │  ├─ db/db.go               # the Store (pool + sqlc + Ent) from §2.3
│  │  ├─ auth/                  # argon2, jwt, refresh rotation, middleware (§7)
│  │  ├─ rbac/                  # room-role checks (§8)
│  │  ├─ rest/                  # gin handlers + DTOs (§9)
│  │  ├─ ws/                    # hub, client, read/write pumps, protocol (§10–§13)
│  │  ├─ presence/              # presence + typing, redis-backed (§13)
│  │  └─ backplane/             # redis pub/sub fan-out (§15)
│  ├─ db/
│  │  ├─ migrations/            # goose *.sql — THE schema (§4)
│  │  ├─ query/                 # sqlc *.sql query files (§6)
│  │  └─ gen/                   # sqlc-generated Go (do not edit)
│  ├─ ent/
│  │  ├─ schema/                # Ent schema-as-code (§5)
│  │  └─ *.go                   # ent-generated (do not edit)
│  ├─ sqlc.yaml                 # sqlc config (§6.1)
│  ├─ .air.toml                 # dev loop (§23.6)
│  ├─ Dockerfile
│  └─ go.mod
├─ web/                          # Next.js 16 app (§16–§20)
│  ├─ app/                       # App Router routes
│  ├─ lib/                       # api client, ws client, query keys
│  ├─ components/
│  └─ package.json
├─ docker-compose.yml            # api, postgres, redis, web, nginx (§23)
└─ nginx/chat.conf               # one-origin reverse proxy (§15.6)
```

> **Why `internal/`?** Go refuses to let packages outside the module import anything under `internal/`. It is a compiler-enforced "these are private implementation packages" boundary — exactly right for handlers, the hub, and auth internals you never want another module reaching into.

### 3.2 Typed configuration, fail-fast **[B/I]**

Read every secret and tunable from the environment **once at boot**, into a typed struct, and **crash immediately** if anything required is missing or malformed. A server that boots with a blank JWT secret is a security incident waiting to happen; better to never start.

```go
// internal/config/config.go
package config

import (
	"fmt"
	"os"
	"time"
)

type Config struct {
	Env             string        // "dev" | "prod"
	HTTPAddr        string        // ":8080"
	DatabaseURL     string        // pgx DSN
	RedisURL        string        // "redis://…"
	JWTSecret       []byte        // HS256 signing key (>=32 bytes)
	AccessTTL       time.Duration // e.g. 15m
	RefreshTTL      time.Duration // e.g. 720h (30d)
	AllowedOrigins  []string      // for CORS + WS OriginPatterns
	CookieDomain    string
	CookieSecure    bool          // true in prod (HTTPS)
	MaxMessageBytes int64         // per-message size cap (e.g. 8192)
}

func must(key string) string {
	v := os.Getenv(key)
	if v == "" {
		panic(fmt.Sprintf("missing required env %s", key))
	}
	return v
}

func Load() (*Config, error) {
	secret := must("JWT_SECRET")
	if len(secret) < 32 {
		return nil, fmt.Errorf("JWT_SECRET must be >= 32 bytes, got %d", len(secret))
	}
	env := os.Getenv("APP_ENV")
	if env == "" {
		env = "dev"
	}
	return &Config{
		Env:             env,
		HTTPAddr:        orDefault("HTTP_ADDR", ":8080"),
		DatabaseURL:     must("DATABASE_URL"),
		RedisURL:        must("REDIS_URL"),
		JWTSecret:       []byte(secret),
		AccessTTL:       15 * time.Minute,
		RefreshTTL:      30 * 24 * time.Hour,
		AllowedOrigins:  []string{orDefault("WEB_ORIGIN", "http://localhost:3000")},
		CookieDomain:    orDefault("COOKIE_DOMAIN", "localhost"),
		CookieSecure:    env == "prod",
		MaxMessageBytes: 8192,
	}, nil
}

func orDefault(key, def string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return def
}
```

> **Security note.** Never hardcode `JWT_SECRET` or `DATABASE_URL` in code or commit them. In prod they come from your secrets manager (Docker/K8s secrets, cloud SM). The `>= 32 bytes` check exists because an HS256 key shorter than the hash output materially weakens the signature. See [§21.9](#21-banking-grade-security-hardening).

### 3.3 The entrypoint & graceful shutdown skeleton **[I]**

The realtime nature of the app makes graceful shutdown *matter*: on deploy you have live WebSocket connections mid-conversation. A blunt `os.Exit` drops them with a TCP RST and users see "disconnected." Instead we: stop accepting new HTTP/WS, tell the Hub to send each client a **going-away** close frame (so the browser reconnects to a new node rather than erroring), drain in-flight, then close the pool and Redis.

```go
// cmd/api/main.go
package main

import (
	"context"
	"errors"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"chat/internal/config"
	"chat/internal/db"
	"chat/internal/backplane"
	"chat/internal/ws"
	"chat/internal/rest"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
	slog.SetDefault(logger)

	cfg, err := config.Load()
	if err != nil {
		logger.Error("config", "err", err)
		os.Exit(1)
	}

	// Root context cancelled on SIGINT/SIGTERM — the signal that a deploy is happening.
	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	// 1) Schema up-to-date BEFORE serving (advisory-locked; §4.6). In prod this is
	//    often a separate job, but running it here (idempotent, locked) is safe.
	if err := db.MigrateUp(ctx, cfg.DatabaseURL); err != nil {
		logger.Error("migrate", "err", err)
		os.Exit(1)
	}

	// 2) The one pool + two query layers (§2.3).
	store, err := db.Open(ctx, cfg.DatabaseURL)
	if err != nil {
		logger.Error("db", "err", err)
		os.Exit(1)
	}
	defer store.Close()

	// 3) Redis backplane for cross-node fan-out + presence (§13, §15).
	bp, err := backplane.New(ctx, cfg.RedisURL)
	if err != nil {
		logger.Error("redis", "err", err)
		os.Exit(1)
	}
	defer bp.Close()

	// 4) The Hub owns all live connections and rooms (§10).
	hub := ws.NewHub(store, bp, cfg)
	go hub.Run(ctx)      // the Hub's event loop
	go bp.Consume(ctx, hub) // deliver cluster messages into this node's hub

	// 5) The Gin router (REST) + the WS upgrade endpoint.
	router := rest.NewRouter(cfg, store, hub)

	srv := &http.Server{
		Addr:              cfg.HTTPAddr,
		Handler:           router,
		ReadHeaderTimeout: 10 * time.Second,
		// NOTE: do NOT set WriteTimeout/IdleTimeout so tight they kill the long-lived
		// WS handler. coder/websocket hijacks the conn; we manage its own deadlines.
	}

	go func() {
		logger.Info("listening", "addr", cfg.HTTPAddr)
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			logger.Error("serve", "err", err)
			os.Exit(1)
		}
	}()

	<-ctx.Done() // a signal arrived: begin graceful shutdown
	logger.Info("shutting down")

	// Give the Hub a moment to send going-away frames to every client so browsers
	// reconnect to another node instead of seeing an abrupt drop.
	hub.Drain()

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
	defer cancel()
	if err := srv.Shutdown(shutdownCtx); err != nil {
		logger.Error("http shutdown", "err", err)
	}
	logger.Info("bye")
}
```

We flesh out `hub.Drain()`, `MigrateUp`, and `backplane` in their sections. The point here is the **shape**: config → migrate → store → redis → hub → router → serve → signal → drain → shutdown. Every capstone-scale Go service looks like this.

> **Gotcha — HTTP server timeouts kill WebSockets.** `http.Server.WriteTimeout` and `IdleTimeout` apply to the whole connection. If you set them (sensible for REST) they will *silently sever* long-lived WebSocket connections at the timeout. Because `coder/websocket` hijacks the connection and manages its own read/write deadlines, leave those two server timeouts unset (or very large) and keep `ReadHeaderTimeout` to defend the *handshake* against slowloris. This bites everyone once.

---
## 4. The Database Schema via goose Migrations

goose owns the schema ([§2.1](#21-the-four-tools-and-their-non-overlapping-jobs)). Every table below is created by a numbered SQL migration in `server/db/migrations/`. We embed those files into the binary and run them, advisory-locked, at boot. This section is both the schema reference for the whole app and a worked goose example; for the tool in depth see the [Goose migrations guide](GO_GOOSE_MIGRATIONS_GUIDE.md) and [sqlc + goose](GO_SQLC_GOOSE_GUIDE.md).

### 4.1 goose migration anatomy **[I]**

A goose SQL migration is one file with an **Up** and a **Down** half, delimited by annotation comments. The critical rules:

- `-- +goose Up` / `-- +goose Down` mark the two halves. goose runs statements in a transaction **per migration** by default, so a failed migration rolls back cleanly.
- `-- +goose StatementBegin` / `-- +goose StatementEnd` wrap any statement that itself contains semicolons (a `CREATE FUNCTION` body, a `DO $$…$$`, a trigger) so goose doesn't split it at the wrong `;`.
- `-- +goose NO TRANSACTION` at the top of a file opts that migration *out* of the wrapping transaction — required for `CREATE INDEX CONCURRENTLY`, which cannot run inside a transaction.

Files are ordered by their numeric prefix; we use zero-padded sequence numbers so ordering is obvious and merge conflicts are visible.

### 4.2 Migration 1 — extensions, users, refresh tokens **[I]**

```sql
-- server/db/migrations/00001_init_auth.sql
-- +goose Up
-- +goose StatementBegin
CREATE EXTENSION IF NOT EXISTS pgcrypto; -- gen_random_uuid()
-- +goose StatementEnd

CREATE TABLE users (
    id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    email         text NOT NULL,
    username      text NOT NULL,
    password_hash text NOT NULL,          -- Argon2id PHC string (§7.2); NEVER a plaintext
    created_at    timestamptz NOT NULL DEFAULT now()
);
-- Case-insensitive uniqueness so "Alice@x.com" and "alice@x.com" can't both register.
CREATE UNIQUE INDEX users_email_lower_key ON users (lower(email));
CREATE UNIQUE INDEX users_username_lower_key ON users (lower(username));

-- Refresh-token FAMILIES enable rotation + reuse detection (§7.4). We store only a
-- SHA-256 hash of the opaque token, never the token itself.
CREATE TABLE refresh_tokens (
    id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id    uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    family_id  uuid NOT NULL,             -- all rotations of one login share a family
    token_hash bytea NOT NULL,            -- sha256(raw token)
    expires_at timestamptz NOT NULL,
    revoked_at timestamptz,               -- set when rotated or on reuse detection
    created_at timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX refresh_tokens_hash_key ON refresh_tokens (token_hash);
CREATE INDEX refresh_tokens_user_idx ON refresh_tokens (user_id);
CREATE INDEX refresh_tokens_family_idx ON refresh_tokens (family_id);

-- +goose Down
DROP TABLE refresh_tokens;
DROP TABLE users;
```

### 4.3 Migration 2 — rooms and memberships **[I]**

The `room_members.role` column uses a Postgres `ENUM`; the `rooms.kind` distinguishes channels from DMs. Note `last_read_message_id` lives on the *membership* — it is per-user-per-room, exactly where unread bookkeeping belongs.

```sql
-- server/db/migrations/00002_rooms.sql
-- +goose Up
-- +goose StatementBegin
DO $$ BEGIN
    CREATE TYPE room_role AS ENUM ('owner', 'admin', 'member');
    CREATE TYPE room_kind AS ENUM ('channel', 'dm');
EXCEPTION WHEN duplicate_object THEN NULL;
END $$;
-- +goose StatementEnd

CREATE TABLE rooms (
    id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    kind       room_kind NOT NULL DEFAULT 'channel',
    name       text,                       -- NULL for DMs (§4.4)
    created_by uuid NOT NULL REFERENCES users(id),
    created_at timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT channel_needs_name CHECK (kind <> 'channel' OR name IS NOT NULL)
);

CREATE TABLE room_members (
    room_id              uuid NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
    user_id              uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role                 room_role NOT NULL DEFAULT 'member',
    joined_at            timestamptz NOT NULL DEFAULT now(),
    last_read_message_id uuid,             -- newest message this user has read (§14)
    PRIMARY KEY (room_id, user_id)          -- a user is in a room at most once
);
-- Fast "what rooms is this user in?" — the query behind the sidebar and every authz check.
CREATE INDEX room_members_user_idx ON room_members (user_id);

-- +goose Down
DROP TABLE room_members;
DROP TABLE rooms;
-- +goose StatementBegin
DROP TYPE IF EXISTS room_kind;
DROP TYPE IF EXISTS room_role;
-- +goose StatementEnd
```

### 4.4 Why DMs are just rooms **[I]**

A direct message is a `room` with `kind='dm'`, `name=NULL`, and exactly two `room_members`. We do **not** build a separate `direct_messages` table. The payoff is enormous: message insert, history pagination, unread counting, read receipts, presence, and the entire WebSocket send path are written **once** against `rooms`/`messages` and work for both channels and DMs unchanged. The only DM-specific logic is *creation* — "find or create the unique 2-person dm room for users A and B" — which we handle with a deterministic lookup (below) and a partial unique index to prevent duplicate DM rooms racing into existence.

```sql
-- server/db/migrations/00003_dm_uniqueness.sql
-- +goose Up
-- A canonical "pair key" for a DM: the two member UUIDs sorted and concatenated.
-- We materialize it so a UNIQUE index can prevent two dm rooms for the same pair.
ALTER TABLE rooms ADD COLUMN dm_key text;  -- NULL for channels
CREATE UNIQUE INDEX rooms_dm_key_uidx ON rooms (dm_key) WHERE kind = 'dm';

-- +goose Down
DROP INDEX rooms_dm_key_uidx;
ALTER TABLE rooms DROP COLUMN dm_key;
```

When creating a DM we compute `dm_key = least(a,b) || ':' || greatest(a,b)` in Go and `INSERT … ON CONFLICT (dm_key) WHERE kind='dm' DO NOTHING RETURNING …`, then fetch the existing room if the insert was a no-op. Two users clicking "message" simultaneously converge on one room.

### 4.5 Migration 4 — messages and read receipts **[I]**

`messages` is the highest-volume table and the hot path. Two details are load-bearing:

- **`client_msg_id`** — a UUID the *client* generates before sending, unique per `(room_id, sender_id)`. It gives us **idempotency**: if a flaky network causes the client to resend, the second insert conflicts and we return the already-stored row instead of creating a duplicate. This is how "send exactly once" survives retries and reconnects ([§12.4](#12-the-message-send-flow-end-to-end)).
- **The composite index `(room_id, created_at, id)`** — this is the index the **keyset pagination** query rides. History reads "the newest N messages in room R older than cursor (created_at, id)"; that predicate is a range scan on exactly this index. Get it wrong and history reads do sequential scans that melt under load.

```sql
-- server/db/migrations/00004_messages.sql
-- +goose Up
CREATE TABLE messages (
    id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    room_id       uuid NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
    sender_id     uuid NOT NULL REFERENCES users(id),
    body          text NOT NULL,
    client_msg_id uuid NOT NULL,           -- idempotency key from the client (§12.4)
    created_at    timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT body_not_blank CHECK (length(btrim(body)) > 0),
    CONSTRAINT body_len CHECK (length(body) <= 8192)
);
-- Idempotency: one client_msg_id per sender per room.
CREATE UNIQUE INDEX messages_idem_uidx ON messages (room_id, sender_id, client_msg_id);
-- THE keyset-pagination index. Order matches the ORDER BY in the history query.
CREATE INDEX messages_room_created_idx ON messages (room_id, created_at DESC, id DESC);

-- Read receipts: the newest message each user has read in each room. We keep it as a
-- single row per (room,user) rather than per-message so "mark read up to X" is one UPSERT.
CREATE TABLE read_receipts (
    room_id    uuid NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
    user_id    uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    message_id uuid NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
    read_at    timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (room_id, user_id)
);

-- +goose Down
DROP TABLE read_receipts;
DROP TABLE messages;
```

> **Design note — receipts vs. `last_read_message_id`.** We store the "last read" pointer in *two* places: `room_members.last_read_message_id` (used to compute *your own* unread count) and `read_receipts` (used to show *others* your read state, e.g. the "seen" checkmark). They are updated together in one statement ([§14](#14-read-receipts--unread-counts)). Keeping both is a deliberate denormalization: one drives the sidebar badge for you, the other drives the per-message receipt broadcast for everyone else.

### 4.6 The embedded, advisory-locked runner **[A]**

We embed the migration files into the binary (`//go:embed`) so a deployed container needs no external files, and we run them behind a **Postgres advisory lock** so that when N app instances start during a rolling deploy, exactly one runs the migrations and the rest wait, then no-op. We use the modern goose *provider* API over a `*sql.DB` obtained from the shared pool.

```go
// internal/db/migrate.go
package db

import (
	"context"
	"embed"
	"fmt"
	"log/slog"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/jackc/pgx/v5/stdlib"
	"github.com/pressly/goose/v3"
)

//go:embed migrations/*.sql
var migrationsFS embed.FS

const advisoryLockKey = 727274 // any stable app-wide constant

// MigrateUp brings the schema to the latest version. Safe to call from every
// instance concurrently: the advisory lock serializes them.
func MigrateUp(ctx context.Context, dsn string) error {
	pool, err := pgxpool.New(ctx, dsn) // a short-lived pool just for migration
	if err != nil {
		return fmt.Errorf("migrate pool: %w", err)
	}
	defer pool.Close()

	// Take a session-level advisory lock so only one instance migrates at a time.
	conn, err := pool.Acquire(ctx)
	if err != nil {
		return err
	}
	defer conn.Release()
	if _, err := conn.Exec(ctx, "SELECT pg_advisory_lock($1)", advisoryLockKey); err != nil {
		return fmt.Errorf("advisory lock: %w", err)
	}
	defer conn.Exec(ctx, "SELECT pg_advisory_unlock($1)", advisoryLockKey)

	// goose needs a *sql.DB. Wrap the SAME pool (no new connections).
	sqlDB := stdlib.OpenDBFromPool(pool)
	defer sqlDB.Close()

	provider, err := goose.NewProvider(
		goose.DialectPostgres,
		sqlDB,
		migrationsFS,
		goose.WithSlog(slog.Default()),
	)
	if err != nil {
		return fmt.Errorf("goose provider: %w", err)
	}
	results, err := provider.Up(ctx)
	if err != nil {
		return fmt.Errorf("goose up: %w", err)
	}
	for _, r := range results {
		slog.Info("migrated", "version", r.Source.Version, "name", r.Source.Path)
	}
	return nil
}
```

> **Gotcha — the advisory lock must be on a *dedicated connection* you hold for the whole migration.** `pg_advisory_lock` is session-scoped; if the connection that took the lock is returned to the pool and a *different* connection runs the migrations, you have no mutual exclusion. That is why we `Acquire` a specific `conn`, lock on it, and hold it until unlock. Using a transaction-scoped `pg_advisory_xact_lock` inside goose's per-migration transaction would release the lock between migrations — not what we want. See the [Goose guide](GO_GOOSE_MIGRATIONS_GUIDE.md) for the deploy-as-separate-job alternative.

---

## 5. Ent Schema — The Relation Graph

Ent owns the **graph and RBAC CRUD**: users, rooms, memberships, roles, invitations. We define the schema as Go code, run codegen, and get a typed client that makes "load this user with their rooms and each room's members" a one-liner — and, crucially, makes the authorization checks in [§8](#8-rbac--authorization--room-roles-and-idor-defense) hard to get wrong. Ent's **auto-migration is off**; goose already created these tables in [§4](#4-the-database-schema-via-goose-migrations). Ent must therefore describe the *same* columns goose created. For Ent in depth see the [Ent ORM guide](GO_ENT_ORM_GUIDE.md).

### 5.1 The mental split: Ent describes tables goose owns **[I]**

This is the subtle part of running Ent alongside goose. Normally Ent generates and applies its own DDL. Here goose is the source of truth, so Ent's schema files are a **hand-maintained mirror** of the goose tables — same names, same columns, same types — used only to generate the *query/CRUD* code, never to create tables. If the two drift (you add a column in goose but not in Ent), Ent will error at runtime when it selects a column that its struct doesn't know, or silently ignore one it expects. The discipline: **every goose column that Ent reads or writes must appear in the Ent schema**, and Ent never introduces a column goose doesn't have. [§24](#24-maintainability--contracts--keeping-tools-in-sync) covers keeping them in sync.

### 5.2 The User schema **[I]**

```go
// ent/schema/user.go
package schema

import (
	"time"

	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/index"
	"github.com/google/uuid"
)

type User struct{ ent.Schema }

func (User) Fields() []ent.Field {
	return []ent.Field{
		field.UUID("id", uuid.UUID{}).
			Default(uuid.New), // matches gen_random_uuid() semantics on inserts we do via Ent
		field.String("email").NotEmpty(),
		field.String("username").NotEmpty(),
		field.String("password_hash").Sensitive(), // Sensitive() omits it from Ent's String()/logs
		field.Time("created_at").Default(time.Now).Immutable(),
	}
}

func (User) Edges() []ent.Edge {
	return []ent.Edge{
		// A user has many memberships; each membership points at a room (§5.4).
		edge.To("memberships", Membership.Type),
		// Rooms this user created.
		edge.To("created_rooms", Room.Type),
	}
}

func (User) Indexes() []ent.Index {
	return []ent.Index{
		// Mirror goose's case-insensitive unique indexes (informational for Ent).
		index.Fields("email").Unique(),
		index.Fields("username").Unique(),
	}
}
```

> **`Sensitive()` and `password_hash`.** Marking the field `Sensitive()` makes Ent redact it from the entity's generated `String()` method and from any accidental structured log of the whole entity — a cheap defense against leaking hashes into logs. We still only ever *read* it in the login path ([§7.3](#7-authentication--argon2id-jwt-access--refresh-rotation)).

### 5.3 The Room schema **[I]**

```go
// ent/schema/room.go
package schema

import (
	"time"

	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
	"github.com/google/uuid"
)

type Room struct{ ent.Schema }

func (Room) Fields() []ent.Field {
	return []ent.Field{
		field.UUID("id", uuid.UUID{}).Default(uuid.New),
		field.Enum("kind").Values("channel", "dm").Default("channel"),
		field.String("name").Optional().Nillable(), // NULL for DMs
		field.String("dm_key").Optional().Nillable(),
		field.UUID("created_by", uuid.UUID{}),
		field.Time("created_at").Default(time.Now).Immutable(),
	}
}

func (Room) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("members", Membership.Type),
		// The inverse of User.created_rooms.
		edge.From("creator", User.Type).Ref("created_rooms").
			Field("created_by").Unique().Required(),
	}
}
```

### 5.4 The Membership schema — an edge with data **[I]**

`room_members` is a classic **join table with extra columns** (`role`, `joined_at`, `last_read_message_id`). In Ent you model this as its own entity (`Membership`) sitting between `User` and `Room`, because the relationship *carries data*. This is the entity every RBAC check reads.

```go
// ent/schema/membership.go
package schema

import (
	"time"

	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/index"
	"github.com/google/uuid"
)

type Membership struct{ ent.Schema }

func (Membership) Fields() []ent.Field {
	return []ent.Field{
		field.UUID("room_id", uuid.UUID{}),
		field.UUID("user_id", uuid.UUID{}),
		field.Enum("role").Values("owner", "admin", "member").Default("member"),
		field.Time("joined_at").Default(time.Now).Immutable(),
		field.UUID("last_read_message_id", uuid.UUID{}).Optional().Nillable(),
	}
}

func (Membership) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("room", Room.Type).Field("room_id").Unique().Required(),
		edge.To("user", User.Type).Field("user_id").Unique().Required(),
	}
}

func (Membership) Indexes() []ent.Index {
	return []ent.Index{
		// Composite uniqueness mirrors goose's PRIMARY KEY (room_id, user_id).
		index.Fields("room_id", "user_id").Unique(),
	}
}
```

### 5.5 Generating and using the client **[I]**

```bash
# From server/, regenerate the Ent client whenever a schema file changes.
go generate ./ent
# (ent/generate.go contains: //go:generate go run -mod=mod entgo.io/ent/cmd/ent generate ./schema)
```

Typed queries the RBAC layer will lean on — note the eager-loading that eliminates N+1:

```go
// Is this user a member of this room, and with what role? One indexed query.
m, err := store.Ent.Membership.Query().
	Where(membership.RoomID(roomID), membership.UserID(userID)).
	Only(ctx) // returns *ent.NotFoundError if not a member — that IS the authz answer

// The sidebar: all rooms a user belongs to, each with its members eager-loaded.
memberships, err := store.Ent.Membership.Query().
	Where(membership.UserID(userID)).
	WithRoom(func(rq *ent.RoomQuery) {
		rq.WithMembers(func(mq *ent.MembershipQuery) {
			mq.WithUser() // so we can render member avatars/names
		})
	}).
	All(ctx)
```

> **Why Ent here and sqlc for messages?** Notice the query above walks *three levels of relationship* (membership → room → members → user) and Ent eager-loads it in a handful of batched queries with zero hand-written SQL. That is Ent's sweet spot. A message-history read, by contrast, is a single-table keyset scan where we want to *control the exact predicate and index* — sqlc's sweet spot ([§6](#6-sqlc--type-safe-hot-path-queries)). Same pool, right tool per job.

---

## 6. sqlc — Type-Safe Hot-Path Queries

sqlc reads your **goose migrations** as the schema and your **`.sql` query files** and generates Go functions with typed parameters and typed result rows — no reflection, no query builder, the exact SQL you wrote and reviewed. We use it for the four hottest/gnarliest queries in the app: **insert a message (idempotent)**, **keyset-paginate history**, **count unread**, and **list recent conversations**. See the [sqlc + goose guide](GO_SQLC_GOOSE_GUIDE.md) for the tool in depth.

### 6.1 sqlc.yaml — pointing schema at the goose dir **[I]**

The single most important line: `schema:` points at the **goose migrations directory**, not a separate schema file. sqlc understands goose's `-- +goose Up/Down` annotations natively and builds its type model from the same migrations that create your real tables — so sqlc's types can never drift from production DDL.

```yaml
# server/sqlc.yaml
version: "2"
sql:
  - engine: "postgresql"
    schema: "db/migrations"     # THE goose migrations = the schema (single source of truth)
    queries: "db/query"          # our hand-written .sql query files
    gen:
      go:
        package: "gen"
        out: "db/gen"
        sql_package: "pgx/v5"    # generate pgx v5 code (uses the pool / pgx.Tx as DBTX)
        emit_pointers_for_null_types: true
        overrides:
          - db_type: "uuid"
            go_type: "github.com/google/uuid.UUID"
          - db_type: "timestamptz"
            go_type: "time.Time"
```

> **⚡ Version note.** `version: "2"` and `sql_package: "pgx/v5"` are the current, correct settings (sqlc v1.31+). With `pgx/v5`, the generated `New(db DBTX)` accepts anything implementing `Exec/Query/QueryRow` — which `*pgxpool.Pool` and `pgx.Tx` both do — so the same `Queries` value works on the pool for autocommit reads and, via `q.WithTx(tx)`, inside a transaction for the send flow.

### 6.2 The message queries **[I/A]**

Query annotations tell sqlc what to generate: `:one` (single row), `:many` (slice), `:exec` (no rows), `:execrows` (rows-affected). Parameters are `$1, $2, …`; `sqlc.arg(name)` names them; `sqlc.narg(name)` marks a *nullable* arg (used for the keyset cursor's "first page" case).

```sql
-- server/db/query/messages.sql

-- name: InsertMessage :one
-- Idempotent insert. If this (room, sender, client_msg_id) already exists — a retry —
-- the ON CONFLICT DO NOTHING makes RETURNING empty, and we fall back to SelectByIdem.
INSERT INTO messages (room_id, sender_id, body, client_msg_id)
VALUES ($1, $2, $3, $4)
ON CONFLICT (room_id, sender_id, client_msg_id) DO NOTHING
RETURNING id, room_id, sender_id, body, client_msg_id, created_at;

-- name: SelectMessageByIdem :one
SELECT id, room_id, sender_id, body, client_msg_id, created_at
FROM messages
WHERE room_id = $1 AND sender_id = $2 AND client_msg_id = $3;

-- name: ListRoomHistory :many
-- Keyset pagination: newest-first, everything strictly OLDER than the cursor
-- (created_at, id). On the first page pass NULLs and the predicate degrades to "no cursor".
-- Rides the messages_room_created_idx index exactly.
SELECT id, room_id, sender_id, body, client_msg_id, created_at
FROM messages
WHERE room_id = $1
  AND (
    sqlc.narg(before_created_at)::timestamptz IS NULL
    OR (created_at, id) < (sqlc.narg(before_created_at)::timestamptz, sqlc.narg(before_id)::uuid)
  )
ORDER BY created_at DESC, id DESC
LIMIT $2;
```

**Why keyset, not `OFFSET`.** `OFFSET 10000 LIMIT 50` makes Postgres read and discard 10,000 rows every deep page — cost grows with page depth, and rows shift under you as new messages arrive, causing skips/dupes. Keyset pagination carries a **cursor** (the `(created_at, id)` of the last row you saw) and asks for rows *strictly older than the cursor*. Cost is constant regardless of depth, and because the cursor is an absolute position, concurrent inserts never cause skips. The `(created_at, id)` tuple (not `created_at` alone) breaks ties when two messages share a timestamp — essential for correctness.

### 6.3 Unread counts and recent conversations **[I/A]**

Unread for a room = messages newer than the user's `last_read_message_id` (excluding the user's own). We compute it against the same keyset index. The "recent conversations" query powers the sidebar: each room the user is in, its last message, and its unread count, ordered by most-recent activity.

```sql
-- server/db/query/unread.sql

-- name: CountUnreadForRoom :one
-- Count messages in a room newer than the user's last-read pointer, not sent by them.
SELECT count(*) AS unread
FROM messages m
LEFT JOIN room_members rm
       ON rm.room_id = m.room_id AND rm.user_id = sqlc.arg(user_id)
WHERE m.room_id = sqlc.arg(room_id)
  AND m.sender_id <> sqlc.arg(user_id)
  AND (
    rm.last_read_message_id IS NULL
    OR m.created_at > (
        SELECT created_at FROM messages WHERE id = rm.last_read_message_id
    )
  );

-- name: MarkRoomRead :execrows
-- UPSERT both the membership pointer AND the read_receipts row in the SAME logical op.
-- We only ever move the pointer FORWARD (never backward) via the GREATEST-style guard.
WITH target AS (
    SELECT created_at FROM messages WHERE id = sqlc.arg(message_id) AND room_id = sqlc.arg(room_id)
)
UPDATE room_members rm
SET last_read_message_id = sqlc.arg(message_id)
FROM target
WHERE rm.room_id = sqlc.arg(room_id)
  AND rm.user_id = sqlc.arg(user_id)
  AND (
    rm.last_read_message_id IS NULL
    OR (SELECT created_at FROM messages WHERE id = rm.last_read_message_id) < target.created_at
  );

-- name: UpsertReadReceipt :exec
INSERT INTO read_receipts (room_id, user_id, message_id, read_at)
VALUES ($1, $2, $3, now())
ON CONFLICT (room_id, user_id)
DO UPDATE SET message_id = EXCLUDED.message_id, read_at = EXCLUDED.read_at
WHERE read_receipts.message_id <> EXCLUDED.message_id;
```

```sql
-- server/db/query/conversations.sql

-- name: ListRecentConversations :many
-- One row per room the user belongs to, with the last message preview and unread count.
-- DISTINCT ON picks the newest message per room cheaply given the room/created index.
SELECT
    r.id            AS room_id,
    r.kind          AS kind,
    r.name          AS name,
    last.body       AS last_body,
    last.created_at AS last_at
FROM room_members rm
JOIN rooms r ON r.id = rm.room_id
LEFT JOIN LATERAL (
    SELECT body, created_at
    FROM messages m
    WHERE m.room_id = r.id
    ORDER BY m.created_at DESC, m.id DESC
    LIMIT 1
) last ON true
WHERE rm.user_id = sqlc.arg(user_id)
ORDER BY last.created_at DESC NULLS LAST;
```

### 6.4 Generated code, and using it with and without a transaction **[I]**

`sqlc generate` produces `db/gen/` with a `Queries` struct, a `New(db DBTX)` constructor, and one method per query with typed params/results. You call it on the pool for autocommit reads, and inside a transaction via `WithTx` for the send flow.

```go
// A plain read on the shared pool (autocommit): the history endpoint.
rows, err := store.Q.ListRoomHistory(ctx, gen.ListRoomHistoryParams{
	RoomID:          roomID,
	Limit:           50,
	BeforeCreatedAt: cursorTime, // *time.Time — nil on the first page
	BeforeID:        cursorID,   // *uuid.UUID — nil on the first page
})

// Inside a transaction: the message send (persist + read-back on idempotent retry).
tx, err := store.Pool.Begin(ctx)
if err != nil { /* ... */ }
defer tx.Rollback(ctx) // no-op after a successful Commit
qtx := store.Q.WithTx(tx) // a Queries bound to THIS transaction
msg, err := qtx.InsertMessage(ctx, gen.InsertMessageParams{ /* ... */ })
// ... on empty (conflict), qtx.SelectMessageByIdem(...) ...
if err := tx.Commit(ctx); err != nil { /* ... */ }
```

> **Gotcha — regenerate after every schema change.** Because sqlc's types come from the goose migrations, adding a column in a goose file and forgetting `sqlc generate` means your Go still compiles against the *old* row shape and silently ignores the new column. Wire `sqlc generate` into the [Air](#23-deployment--docker-compose--migrations-on-deploy) loop and CI so it can never lag. The same discipline applies to `go generate ./ent` — one schema change, two regenerations.

---
## 7. Authentication — Argon2id, JWT Access & Refresh Rotation

Auth is the gate to everything, and it is subtle enough that we cross-reference the full threat model in the [JWT + Argon2 guide](GO_JWT_ARGON2_GUIDE.md) and implement the practical, correct subset here. The design: **passwords hashed with Argon2id**; a short-lived **JWT access token** (15 min, held in memory on the client); a long-lived **opaque refresh token** (30 days, in an HttpOnly cookie) that **rotates on every use** with **reuse detection** via token families. This same access token also authenticates the WebSocket upgrade ([§10.3](#10-realtime-with-coderwebsocket--the-hub-architecture)).

### 7.1 Why this shape **[I]**

- **Argon2id for passwords** because it is *memory-hard*: an attacker with GPUs/ASICs can't parallelize guesses cheaply the way they can against fast hashes. It won the Password Hashing Competition and is the current best practice.
- **A stateless JWT access token** so the vast majority of requests (and every WS frame's authorization) verify identity with a **signature check and no database round-trip**. It is short-lived so a stolen access token expires quickly.
- **An opaque, rotating refresh token in an HttpOnly cookie** because it must survive page reloads (memory can't) yet be unreadable to JavaScript (defends against XSS token theft). We store only its **hash**, so a database leak doesn't yield usable tokens. Rotation + families give us **reuse detection**: if an old refresh token is ever presented again, we know it was stolen and nuke the whole family.

### 7.2 Argon2id register **[I/A]**

We hash into the standard **PHC string** (`$argon2id$v=19$m=…,t=…,p=…$salt$hash`) so the parameters travel with the hash and can be upgraded later without a schema change. Parameters below are a sane 2026 baseline; tune `memory` upward on capable hardware.

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

// Tunables. memory is in KiB (64 MiB here). Raise memory/time on stronger hosts.
const (
	argonTime    = 3
	argonMemory  = 64 * 1024
	argonThreads = 2
	argonKeyLen  = 32
	argonSaltLen = 16
)

func HashPassword(plain string) (string, error) {
	salt := make([]byte, argonSaltLen)
	if _, err := rand.Read(salt); err != nil { // CSPRNG salt, unique per password
		return "", err
	}
	key := argon2.IDKey([]byte(plain), salt, argonTime, argonMemory, argonThreads, argonKeyLen)
	// Encode as a PHC string so params + salt are self-describing.
	return fmt.Sprintf("$argon2id$v=%d$m=%d,t=%d,p=%d$%s$%s",
		argon2.Version, argonMemory, argonTime, argonThreads,
		base64.RawStdEncoding.EncodeToString(salt),
		base64.RawStdEncoding.EncodeToString(key),
	), nil
}

// VerifyPassword recomputes the hash with the stored params and compares in
// constant time. subtle.ConstantTimeCompare defends against timing attacks.
func VerifyPassword(plain, phc string) (bool, error) {
	var m, t uint32
	var p uint8
	parts := strings.Split(phc, "$")
	if len(parts) != 6 || parts[1] != "argon2id" {
		return false, errors.New("bad hash format")
	}
	if _, err := fmt.Sscanf(parts[3], "m=%d,t=%d,p=%d", &m, &t, &p); err != nil {
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
	got := argon2.IDKey([]byte(plain), salt, t, m, p, uint32(len(want)))
	return subtle.ConstantTimeCompare(got, want) == 1, nil
}
```

The register handler creates the user through **Ent** (users are graph data), lowercasing email/username for the case-insensitive unique indexes, and mapping the unique-violation to a clean 409.

```go
// internal/auth/service.go (excerpt)
func (s *Service) Register(ctx context.Context, email, username, password string) (*ent.User, error) {
	hash, err := HashPassword(password)
	if err != nil {
		return nil, err
	}
	u, err := s.ent.User.Create().
		SetEmail(strings.ToLower(email)).
		SetUsername(strings.ToLower(username)).
		SetPasswordHash(hash).
		Save(ctx)
	if ent.IsConstraintError(err) {
		return nil, ErrEmailTaken // handler → 409 Conflict
	}
	return u, err
}
```

> **Security note — do not leak which field collided, and rate-limit registration.** Return a generic "email or username already in use" rather than "email taken" vs. "username taken" to avoid an *enumeration oracle*. And rate-limit the endpoint ([§21.5](#21-banking-grade-security-hardening)) — Argon2id is deliberately expensive, which is also a mild DoS vector if an attacker floods registrations.

### 7.3 JWT access tokens & login **[I/A]**

The access token is a signed JWT carrying the user id (`sub`), an expiry, and nothing sensitive. We sign with HS256 and — critically — **assert the algorithm on verify** to prevent `alg:none` and algorithm-confusion attacks.

```go
// internal/auth/jwt.go
package auth

import (
	"errors"
	"time"

	"github.com/golang-jwt/jwt/v5"
	"github.com/google/uuid"
)

type Claims struct {
	jwt.RegisteredClaims
	UserID uuid.UUID `json:"uid"`
}

func (s *Service) IssueAccess(userID uuid.UUID) (string, error) {
	now := time.Now()
	claims := Claims{
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   userID.String(),
			IssuedAt:  jwt.NewNumericDate(now),
			ExpiresAt: jwt.NewNumericDate(now.Add(s.accessTTL)),
			Issuer:    "chat-api",
			Audience:  []string{"chat-web"},
		},
		UserID: userID,
	}
	tok := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return tok.SignedString(s.jwtSecret)
}

func (s *Service) ParseAccess(raw string) (*Claims, error) {
	claims := &Claims{}
	_, err := jwt.ParseWithClaims(raw, claims, func(t *jwt.Token) (any, error) {
		return s.jwtSecret, nil
	},
		jwt.WithValidMethods([]string{"HS256"}), // reject anything but HS256 — no alg confusion
		jwt.WithIssuer("chat-api"),
		jwt.WithAudience("chat-web"),
	)
	if err != nil {
		return nil, err
	}
	if claims.UserID == uuid.Nil {
		return nil, errors.New("missing uid")
	}
	return claims, nil
}
```

Login verifies the password, then issues an access token *and* a fresh refresh token (new family):

```go
func (s *Service) Login(ctx context.Context, email, password string) (access string, refresh string, err error) {
	u, err := s.ent.User.Query().Where(user.EmailEQ(strings.ToLower(email))).Only(ctx)
	if err != nil {
		// Verify against a DUMMY hash anyway so response time doesn't reveal
		// whether the email exists (timing-based user enumeration defense).
		_, _ = VerifyPassword(password, dummyHash)
		return "", "", ErrInvalidCredentials
	}
	ok, err := VerifyPassword(password, u.PasswordHash)
	if err != nil || !ok {
		return "", "", ErrInvalidCredentials
	}
	access, err = s.IssueAccess(u.ID)
	if err != nil {
		return "", "", err
	}
	refresh, err = s.issueRefresh(ctx, u.ID, uuid.New()) // brand-new family
	return access, refresh, err
}
```

### 7.4 Refresh rotation with reuse detection **[A]**

The refresh token is a random 256-bit opaque string. We store only `sha256(token)`. Each token belongs to a **family** (one login = one family). On refresh: look up the presented token by hash; if not found or expired → 401; if found but **already revoked** → this is a *reuse* of a rotated token, meaning it was likely stolen, so we **revoke the entire family** and force re-login; otherwise revoke the presented token and issue a new one *in the same family*.

```go
// internal/auth/refresh.go
package auth

import (
	"context"
	"crypto/rand"
	"crypto/sha256"
	"encoding/base64"
	"time"

	"github.com/google/uuid"
)

func newOpaque() (raw string, hash []byte) {
	b := make([]byte, 32)
	_, _ = rand.Read(b)
	raw = base64.RawURLEncoding.EncodeToString(b)
	h := sha256.Sum256([]byte(raw))
	return raw, h[:]
}

func (s *Service) issueRefresh(ctx context.Context, userID, family uuid.UUID) (string, error) {
	raw, hash := newOpaque()
	_, err := s.ent.RefreshToken.Create().
		SetUserID(userID).
		SetFamilyID(family).
		SetTokenHash(hash).
		SetExpiresAt(time.Now().Add(s.refreshTTL)).
		Save(ctx)
	return raw, err
}

// Rotate consumes the presented refresh token and returns a new access+refresh pair.
func (s *Service) Rotate(ctx context.Context, presented string) (access, refresh string, err error) {
	h := sha256.Sum256([]byte(presented))
	rt, err := s.ent.RefreshToken.Query().Where(refreshtoken.TokenHash(h[:])).Only(ctx)
	if err != nil {
		return "", "", ErrInvalidRefresh // unknown token
	}
	if rt.RevokedAt != nil {
		// REUSE DETECTED: a rotated (already-revoked) token was presented again.
		// Assume theft; burn the whole family so neither party can continue.
		_, _ = s.ent.RefreshToken.Update().
			Where(refreshtoken.FamilyID(rt.FamilyID), refreshtoken.RevokedAtIsNil()).
			SetRevokedAt(time.Now()).Save(ctx)
		return "", "", ErrRefreshReuse
	}
	if time.Now().After(rt.ExpiresAt) {
		return "", "", ErrInvalidRefresh
	}
	// Happy path: revoke the presented token, mint a replacement in the SAME family.
	if _, err := s.ent.RefreshToken.UpdateOne(rt).SetRevokedAt(time.Now()).Save(ctx); err != nil {
		return "", "", err
	}
	if access, err = s.IssueAccess(rt.UserID); err != nil {
		return "", "", err
	}
	refresh, err = s.issueRefresh(ctx, rt.UserID, rt.FamilyID)
	return access, refresh, err
}
```

The cookie helpers set the refresh token as **HttpOnly, Secure, SameSite** with a path scoped to the refresh endpoint so it isn't sent on every request:

```go
func (s *Service) SetRefreshCookie(c *gin.Context, raw string) {
	c.SetSameSite(http.SameSiteStrictMode) // or None+Secure if cross-site; see §15.6
	c.SetCookie(
		"refresh_token", raw,
		int(s.refreshTTL.Seconds()),
		"/api/auth", // path-scoped: only sent to auth endpoints
		s.cookieDomain,
		s.cookieSecure, // true in prod (HTTPS only)
		true,           // HttpOnly: unreadable to JS — XSS can't exfiltrate it
	)
}
```

### 7.5 The Gin auth middleware & context identity **[I]**

REST handlers read the access token from the `Authorization: Bearer …` header, verify it, and stash the user id on the request context. Handlers then read identity from context, never from the request body.

```go
// internal/auth/middleware.go
func (s *Service) RequireAuth() gin.HandlerFunc {
	return func(c *gin.Context) {
		raw := strings.TrimPrefix(c.GetHeader("Authorization"), "Bearer ")
		if raw == "" {
			c.AbortWithStatusJSON(401, gin.H{"error": "missing token"})
			return
		}
		claims, err := s.ParseAccess(raw)
		if err != nil {
			c.AbortWithStatusJSON(401, gin.H{"error": "invalid token"})
			return
		}
		c.Set("userID", claims.UserID) // downstream handlers read this
		c.Next()
	}
}

// Helper handlers use to fetch the authenticated identity.
func UserID(c *gin.Context) uuid.UUID { return c.MustGet("userID").(uuid.UUID) }
```

> **Gotcha — the WebSocket cannot use this header.** Browsers cannot set an `Authorization` header on the WebSocket handshake. The WS endpoint therefore authenticates via a **subprotocol** carrying the same access token, verified at `Accept` time — [§10.3](#10-realtime-with-coderwebsocket--the-hub-architecture). Same token, different delivery channel. Do not try to reuse `RequireAuth` on the upgrade route.

---

## 8. RBAC & Authorization — Room Roles and IDOR Defense

Authentication says *who you are*; **authorization** says *what you may do*. In a chat app the authorization questions are almost all **row-level**: may *this* user read *this* room's history? post to it? delete *this* message? promote *that* member? Getting this right is the difference between a chat app and a data breach — the classic failure is **IDOR** (Insecure Direct Object Reference): a user changes a room id in a URL and reads a room they were never invited to.

### 8.1 The authorization model **[I]**

Two layers, checked in order:

1. **Membership** — are you a `room_member` of the room at all? If not, the room does not exist *as far as you're concerned* (we return 404, not 403, so we don't even confirm the room exists — see the note below). This single check defends every read and the ability to post.
2. **Role** — within a room you hold `owner`, `admin`, or `member`. Roles gate *management* actions: only `owner`/`admin` may add/remove members, change roles, rename or delete the room; `owner` is additionally protected (you can't remove the last owner).

| Action | member | admin | owner |
|---|:--:|:--:|:--:|
| Read history / post messages | ✅ | ✅ | ✅ |
| Add / remove members | ❌ | ✅ | ✅ |
| Change a member's role | ❌ | ✅ (not to/from owner) | ✅ |
| Rename room | ❌ | ✅ | ✅ |
| Delete room | ❌ | ❌ | ✅ |
| Delete *any* message (moderation) | ❌ | ✅ | ✅ |
| Delete *own* message | ✅ | ✅ | ✅ |

> **Return 404, not 403, for non-members.** If a non-member requests `/rooms/{id}`, returning 403 *confirms the room exists*. Returning 404 leaks nothing. Reserve 403 for the case where you *are* a member but lack the *role* for a management action (there, the resource's existence is already known to you). This nuance matters for private rooms/DMs.

### 8.2 Enforcing it in BOTH layers **[I/A]**

The rule from [§2.2](#22-why-not-just-one-tool-an-honest-accounting) — Ent owns membership, sqlc owns messages — means authorization must be enforced in *both* query layers, consistently. Ent-side we query the `Membership`; sqlc-side we fold the membership predicate directly into the `WHERE` clause so a non-member's query returns zero rows *at the database*, not filtered in Go.

The Ent-side check, reused by every room route:

```go
// internal/rbac/rbac.go
package rbac

import (
	"context"
	"errors"

	"github.com/google/uuid"
	"chat/ent"
	"chat/ent/membership"
)

var (
	ErrNotMember = errors.New("not a member")   // → 404
	ErrForbidden = errors.New("insufficient role") // → 403
)

type Role = string

// Membership returns the caller's role in the room, or ErrNotMember.
func Get(ctx context.Context, c *ent.Client, roomID, userID uuid.UUID) (Role, error) {
	m, err := c.Membership.Query().
		Where(membership.RoomID(roomID), membership.UserID(userID)).
		Only(ctx)
	if ent.IsNotFound(err) {
		return "", ErrNotMember
	}
	if err != nil {
		return "", err
	}
	return m.Role.String(), nil
}

// RequireRole asserts the caller holds at least one of the allowed roles.
func RequireRole(ctx context.Context, c *ent.Client, roomID, userID uuid.UUID, allowed ...Role) error {
	role, err := Get(ctx, c, roomID, userID)
	if err != nil {
		return err // ErrNotMember → 404
	}
	for _, a := range allowed {
		if role == a {
			return nil
		}
	}
	return ErrForbidden // member, but wrong role → 403
}
```

And the sqlc-side belt-and-braces: even the history query itself only returns rows if you're a member. We add a membership `EXISTS` guard so an attacker who somehow reaches the query with a foreign room id gets nothing:

```sql
-- name: ListRoomHistoryAuthz :many
-- Identical to ListRoomHistory but returns rows ONLY if the caller is a member.
-- Defense in depth: authz is also enforced at the handler, but the query refuses too.
SELECT m.id, m.room_id, m.sender_id, m.body, m.client_msg_id, m.created_at
FROM messages m
WHERE m.room_id = sqlc.arg(room_id)
  AND EXISTS (
      SELECT 1 FROM room_members rm
      WHERE rm.room_id = m.room_id AND rm.user_id = sqlc.arg(user_id)
  )
  AND (
    sqlc.narg(before_created_at)::timestamptz IS NULL
    OR (m.created_at, m.id) < (sqlc.narg(before_created_at)::timestamptz, sqlc.narg(before_id)::uuid)
  )
ORDER BY m.created_at DESC, m.id DESC
LIMIT sqlc.arg(lim);
```

> **Why enforce twice?** The handler check gives a clean 404/403 and avoids the query when possible; the in-query `EXISTS` is a **second wall** so a future refactor that forgets the handler check still can't leak data. Defense in depth is not paranoia here — a single missed membership check is a reportable breach. The realtime path enforces the *same* membership check before every broadcast ([§12.2](#12-the-message-send-flow-end-to-end)).

### 8.3 Management actions and the last-owner guard **[I/A]**

Role changes and deletions run through Ent with explicit guards. The two rules that trip people up: you cannot remove or demote the **last owner** (or the room becomes unmanageable), and an `admin` cannot grant or revoke `owner` (privilege-escalation defense — only an owner mints owners).

```go
func (s *RoomService) RemoveMember(ctx context.Context, roomID, actorID, targetID uuid.UUID) error {
	if err := rbac.RequireRole(ctx, s.ent, roomID, actorID, "owner", "admin"); err != nil {
		return err
	}
	target, err := rbac.Get(ctx, s.ent, roomID, targetID)
	if err != nil {
		return err
	}
	if target == "owner" {
		// Count owners; refuse to remove the last one.
		owners, _ := s.ent.Membership.Query().
			Where(membership.RoomID(roomID), membership.RoleEQ(membership.RoleOwner)).Count(ctx)
		if owners <= 1 {
			return ErrLastOwner
		}
	}
	_, err = s.ent.Membership.Delete().
		Where(membership.RoomID(roomID), membership.UserID(targetID)).Exec(ctx)
	return err
}
```

> **Frontend mirrors, never enforces.** The UI hides the "Delete room" button for non-owners ([§20](#20-frontend-rbac--mirroring-the-server)), but that is UX only. Every rule above is enforced on the server; the client is never the security boundary. We repeat this because it is the single most violated principle in real apps.

---

## 9. The REST API with Gin

REST carries everything the user initiates and can await: auth, room CRUD, membership/invites, and **message history** (the one high-traffic read, keyset-paginated via sqlc). Realtime *sending* is on the WebSocket ([§12](#12-the-message-send-flow-end-to-end)); REST is the query and command surface around it. For Gin itself — router, `*gin.Context`, binding, middleware — see the [Gin guide](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md).

### 9.1 Router assembly **[I]**

We group routes by auth requirement: a public group (register/login/refresh) and a protected group behind `RequireAuth`. The WS upgrade route is separate because it authenticates differently ([§10.3](#10-realtime-with-coderwebsocket--the-hub-architecture)).

```go
// internal/rest/router.go
package rest

import (
	"github.com/gin-gonic/gin"
	"chat/internal/auth"
	"chat/internal/config"
	"chat/internal/db"
	"chat/internal/ws"
)

func NewRouter(cfg *config.Config, store *db.Store, hub *ws.Hub) *gin.Engine {
	r := gin.New()
	r.Use(gin.Recovery(), RequestLogger(), CORS(cfg.AllowedOrigins), SecurityHeaders())

	authSvc := auth.NewService(store, cfg)
	h := &Handlers{store: store, auth: authSvc, hub: hub, cfg: cfg}

	api := r.Group("/api")
	{
		pub := api.Group("/auth")
		pub.POST("/register", h.Register)
		pub.POST("/login", RateLimit(5, time.Minute), h.Login) // throttle credential stuffing
		pub.POST("/refresh", h.Refresh)
		pub.POST("/logout", h.Logout)

		p := api.Group("")
		p.Use(authSvc.RequireAuth())
		p.GET("/me", h.Me)
		p.GET("/conversations", h.ListConversations)     // sidebar (sqlc)
		p.POST("/rooms", h.CreateRoom)                    // Ent
		p.GET("/rooms/:id", h.GetRoom)                    // Ent + membership
		p.GET("/rooms/:id/messages", h.ListMessages)      // sqlc keyset
		p.POST("/rooms/:id/members", h.AddMember)         // Ent + RBAC
		p.DELETE("/rooms/:id/members/:uid", h.RemoveMember)
		p.POST("/dms/:userId", h.OpenDM)                  // find-or-create dm room
		p.POST("/rooms/:id/read", h.MarkRead)             // read receipt (also over WS)
	}

	// The WebSocket upgrade — auth handled inside via subprotocol (§10.3).
	r.GET("/ws", h.ServeWS)
	return r
}
```

### 9.2 DTOs — wire shapes separate from entities **[I]**

Never serialize an Ent entity or a sqlc row straight to the client. Define explicit **DTOs**: they are the public contract, they omit sensitive fields (`password_hash`!), and they decouple the API from internal schema changes. The same field names appear in the TypeScript types ([§17](#17-frontend-rest-data-layer-with-tanstack-query)).

```go
// internal/rest/dto.go
package rest

import (
	"time"
	"github.com/google/uuid"
)

type MessageDTO struct {
	ID          uuid.UUID `json:"id"`
	RoomID      uuid.UUID `json:"roomId"`
	SenderID    uuid.UUID `json:"senderId"`
	Body        string    `json:"body"`
	ClientMsgID uuid.UUID `json:"clientMsgId"`
	CreatedAt   time.Time `json:"createdAt"`
}

type ConversationDTO struct {
	RoomID   uuid.UUID  `json:"roomId"`
	Kind     string     `json:"kind"`             // "channel" | "dm"
	Name     *string    `json:"name"`             // null for dm
	LastBody *string    `json:"lastBody"`
	LastAt   *time.Time `json:"lastAt"`
	Unread   int64      `json:"unread"`
}

type HistoryPageDTO struct {
	Messages   []MessageDTO `json:"messages"`
	NextCursor *string      `json:"nextCursor"` // opaque; null when no older pages
}
```

### 9.3 The keyset history endpoint **[I/A]**

The cursor is **opaque** to the client — we base64-encode `created_at|id` so the frontend treats it as a token and can't fabricate positions. The handler decodes it, runs the sqlc authz'd query, and emits the next cursor from the last row.

```go
// internal/rest/messages.go
func (h *Handlers) ListMessages(c *gin.Context) {
	userID := auth.UserID(c)
	roomID, err := uuid.Parse(c.Param("id"))
	if err != nil {
		c.JSON(400, gin.H{"error": "bad room id"})
		return
	}
	limit := clampLimit(c.Query("limit"), 50, 100) // default 50, hard cap 100

	var beforeAt *time.Time
	var beforeID *uuid.UUID
	if cur := c.Query("cursor"); cur != "" {
		beforeAt, beforeID, err = decodeCursor(cur)
		if err != nil {
			c.JSON(400, gin.H{"error": "bad cursor"})
			return
		}
	}

	rows, err := h.store.Q.ListRoomHistoryAuthz(c, gen.ListRoomHistoryAuthzParams{
		RoomID:          roomID,
		UserID:          userID, // the EXISTS guard: non-members get zero rows
		BeforeCreatedAt: beforeAt,
		BeforeID:        beforeID,
		Lim:             int32(limit),
	})
	if err != nil {
		c.JSON(500, gin.H{"error": "server error"})
		return
	}

	page := HistoryPageDTO{Messages: make([]MessageDTO, 0, len(rows))}
	for _, m := range rows {
		page.Messages = append(page.Messages, toMessageDTO(m))
	}
	if len(rows) == limit { // a full page implies there may be more
		last := rows[len(rows)-1]
		cur := encodeCursor(last.CreatedAt, last.ID)
		page.NextCursor = &cur
	}
	c.JSON(200, page)
}

func encodeCursor(t time.Time, id uuid.UUID) string {
	raw := fmt.Sprintf("%d|%s", t.UnixNano(), id)
	return base64.RawURLEncoding.EncodeToString([]byte(raw))
}
```

> **Note — the empty membership result is the 404.** Because `ListRoomHistoryAuthz` returns zero rows for a non-member, a naive handler would return an empty page (200) instead of 404. For a private room you may prefer to first assert membership via `rbac.Get` and 404 if absent, *then* query. We do the explicit check on `GetRoom` and rely on the in-query guard for the message list; pick one policy and apply it consistently ([§8.2](#8-rbac--authorization--room-roles-and-idor-defense)).

### 9.4 Consistent errors, status codes & validation **[I]**

Every error response uses one envelope `{ "error": "message", "code": "SNAKE_CASE" }` so the frontend can branch on `code` while showing `error`. Status codes are disciplined: 400 validation, 401 unauthenticated, 403 authenticated-but-forbidden, 404 not-found/not-a-member, 409 conflict, 422 semantic validation, 429 rate-limited, 500 server. Binding uses Gin's `ShouldBindJSON` with `binding` tags for structural validation, and we add semantic checks (e.g., "can't DM yourself") in the handler.

```go
type createRoomReq struct {
	Name string `json:"name" binding:"required,min=1,max=80"`
}

func (h *Handlers) CreateRoom(c *gin.Context) {
	var req createRoomReq
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(400, gin.H{"error": err.Error(), "code": "VALIDATION"})
		return
	}
	userID := auth.UserID(c)
	room, err := h.rooms.CreateChannel(c, userID, req.Name) // Ent: room + owner membership in a tx
	if err != nil {
		c.JSON(500, gin.H{"error": "could not create room", "code": "INTERNAL"})
		return
	}
	c.JSON(201, toRoomDTO(room))
}
```

Creating a channel also makes the creator its **owner** membership — one Ent transaction so a room never exists without an owner:

```go
func (s *RoomService) CreateChannel(ctx context.Context, creator uuid.UUID, name string) (*ent.Room, error) {
	tx, err := s.ent.Tx(ctx)
	if err != nil {
		return nil, err
	}
	room, err := tx.Room.Create().
		SetKind("channel").SetName(name).SetCreatedBy(creator).Save(ctx)
	if err != nil {
		_ = tx.Rollback()
		return nil, err
	}
	if _, err := tx.Membership.Create().
		SetRoomID(room.ID).SetUserID(creator).SetRole("owner").Save(ctx); err != nil {
		_ = tx.Rollback()
		return nil, err
	}
	return room, tx.Commit()
}
```

---
## 10. Realtime with coder/websocket — The Hub Architecture

This is the heart of the app. We use **`github.com/coder/websocket`** — a modern, context-first library. This section builds the **Hub** (the in-memory registry of connections and room subscriptions), the **Client** (one live connection with a read pump and a write pump), authenticates the upgrade, and defends against slow consumers and cross-site hijacking. Read the [Coder WebSocket guide](GO_CODER_WEBSOCKETS_GUIDE.md) alongside — it covers the Accept/Dial API and the one-reader rule in full; here we assemble them into a production hub.

### 10.1 The Hub / Client / room-topic model **[A]**

A chat server must answer one question fast, thousands of times a second: *"a message just landed in room R — which live connections should receive it?"* The **Hub** is the data structure that answers it. It holds:

- a set of **Clients** (one per open WebSocket), and
- a map from **room id → set of Clients** subscribed to that room (the "topics").

When a message is persisted, the Hub looks up the room's client set and enqueues the frame on each client's **send channel**. It never writes to sockets directly — each Client has a single **write-pump goroutine** that drains its send channel and writes to the socket. This is the crucial concurrency discipline: **one goroutine reads a connection, one goroutine writes it**, and the Hub communicates with clients only through channels. The Hub itself runs a single event loop (register/unregister/broadcast) so its maps need no locks — everything funnels through channels. (An alternative is a mutex-guarded map; the single-goroutine loop is easier to reason about and is what we use.)

```
                         ┌──────────────────── Hub (one goroutine) ─────────────────┐
   register/unregister → │  clients:  set[*Client]                                   │
   broadcast(roomID,env) │  rooms:    map[roomID] set[*Client]                       │
                         └──────┬────────────────────────────────────────┬──────────┘
                                │ enqueue frame on each subscriber's      │
                                ▼ bounded send channel                    ▼
                    ┌─────────── Client A ───────────┐        ┌────────── Client B ──────────┐
   socket ↔ readPump│ reads frames → Hub.Inbound     │        │ …                            │
   socket ↔ writePump│ drains send chan → conn.Write  │        │                              │
                    └────────────────────────────────┘        └──────────────────────────────┘
```

### 10.2 The Hub implementation **[A]**

```go
// internal/ws/hub.go
package ws

import (
	"context"
	"log/slog"

	"github.com/google/uuid"
	"chat/internal/backplane"
	"chat/internal/config"
	"chat/internal/db"
)

type Hub struct {
	store *db.Store
	bp    *backplane.Backplane
	cfg   *config.Config

	clients    map[*Client]struct{}
	rooms      map[uuid.UUID]map[*Client]struct{} // roomID → subscribers on THIS node
	byUser     map[uuid.UUID]map[*Client]struct{} // userID → their connections (multi-tab)

	register   chan *Client
	unregister chan *Client
	// broadcast delivers an already-encoded envelope to a room's local subscribers.
	broadcast  chan roomFrame
	drain      chan struct{}
}

type roomFrame struct {
	room  uuid.UUID
	data  []byte // pre-marshaled envelope JSON
	// skip lets the sender avoid echoing to a specific connection when desired.
	skip  *Client
}

func NewHub(store *db.Store, bp *backplane.Backplane, cfg *config.Config) *Hub {
	return &Hub{
		store: store, bp: bp, cfg: cfg,
		clients:    make(map[*Client]struct{}),
		rooms:      make(map[uuid.UUID]map[*Client]struct{}),
		byUser:     make(map[uuid.UUID]map[*Client]struct{}),
		register:   make(chan *Client),
		unregister: make(chan *Client),
		broadcast:  make(chan roomFrame, 256),
		drain:      make(chan struct{}),
	}
}

// Run is the single event loop. Because all mutation of the maps happens here,
// the maps need no locks.
func (h *Hub) Run(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			return
		case c := <-h.register:
			h.clients[c] = struct{}{}
			addTo(h.byUser, c.userID, c)
			for _, rid := range c.rooms { // subscribe to the rooms they belong to
				addTo(h.rooms, rid, c)
			}
			slog.Debug("ws register", "user", c.userID, "rooms", len(c.rooms))
		case c := <-h.unregister:
			h.removeClient(c)
		case rf := <-h.broadcast:
			for c := range h.rooms[rf.room] {
				if c == rf.skip {
					continue
				}
				h.send(c, rf.data)
			}
		case <-h.drain:
			for c := range h.clients {
				c.close(websocketGoingAway, "server shutting down")
			}
		}
	}
}

// send enqueues without blocking the Hub. If a client's buffer is full it is a
// slow consumer — we drop it rather than let it stall the whole hub (§10.5).
func (h *Hub) send(c *Client, data []byte) {
	select {
	case c.out <- data:
	default:
		slog.Warn("slow client evicted", "user", c.userID)
		h.removeClient(c)
		c.close(websocketTryAgainLater, "too slow")
	}
}

func (h *Hub) removeClient(c *Client) {
	if _, ok := h.clients[c]; !ok {
		return
	}
	delete(h.clients, c)
	removeFrom(h.byUser, c.userID, c)
	for _, rid := range c.rooms {
		removeFrom(h.rooms, rid, c)
	}
	close(c.out) // signals the write pump to finish
}

// BroadcastLocal fans out to this node's subscribers only. Cluster-wide fan-out
// goes through the backplane (§15), which calls this on every node.
func (h *Hub) BroadcastLocal(room uuid.UUID, data []byte, skip *Client) {
	h.broadcast <- roomFrame{room: room, data: data, skip: skip}
}

func (h *Hub) Drain() { close(h.drain) }

// tiny set helpers omitted for brevity: addTo / removeFrom / etc.
```

### 10.3 Authenticating the upgrade — JWT via subprotocol **[A]**

Browsers **cannot** set an `Authorization` header on a WebSocket handshake, so we cannot reuse the REST middleware. The two viable browser-compatible options are: (a) a short-lived **query-param ticket**, or (b) smuggling the token through the **`Sec-WebSocket-Protocol`** header, which the browser *can* set via the `new WebSocket(url, protocols)` second argument. We use the subprotocol approach because the token never lands in server access logs or the browser history the way a query param does.

The client sends two subprotocols: a marker (`"chat.v1"`) and `"bearer.<accessJWT>"`. The server verifies the JWT, and — importantly — **must echo back an accepted subprotocol** or the browser rejects the connection. We echo `"chat.v1"`.

```go
// internal/ws/serve.go
package ws

import (
	"net/http"
	"strings"
	"time"

	"github.com/coder/websocket"
	"github.com/gin-gonic/gin"
)

func (h *Handlers) ServeWS(c *gin.Context) {
	// Pull the bearer token out of the requested subprotocols. coder/websocket
	// has no Subprotocols() helper (that is a gorilla API), so parse the
	// comma-separated Sec-WebSocket-Protocol request header ourselves.
	var token string
	for _, p := range strings.Split(c.Request.Header.Get("Sec-WebSocket-Protocol"), ",") {
		if p = strings.TrimSpace(p); strings.HasPrefix(p, "bearer.") {
			token = strings.TrimPrefix(p, "bearer.")
		}
	}
	claims, err := h.auth.ParseAccess(token)
	if err != nil {
		// Reject the handshake BEFORE Accept — never upgrade an unauthenticated conn.
		c.AbortWithStatus(http.StatusUnauthorized)
		return
	}

	conn, err := websocket.Accept(c.Writer, c.Request, &websocket.AcceptOptions{
		Subprotocols:   []string{"chat.v1"},      // we MUST select one we support
		OriginPatterns: h.cfg.AllowedOrigins,      // CSWSH defense — see §10.4
		// CompressionMode: websocket.CompressionDisabled, // optional
	})
	if err != nil {
		return // Accept already wrote the error response
	}
	conn.SetReadLimit(h.cfg.MaxMessageBytes + 512) // envelope overhead headroom

	// Load the user's room memberships so the Hub knows what topics to subscribe.
	rooms, err := h.rooms.RoomIDsForUser(c, claims.UserID)
	if err != nil {
		conn.Close(websocket.StatusInternalError, "load rooms")
		return
	}

	client := newClient(h.hub, conn, claims.UserID, rooms, h.cfg.MaxMessageBytes)
	h.hub.register <- client

	// Run the two pumps. serveWS returns when the connection ends.
	client.run(c.Request.Context())
	h.hub.unregister <- client
}
```

> **Gotcha — you MUST select a subprotocol you were offered.** If the client offers `["chat.v1", "bearer.xyz"]` and your `AcceptOptions.Subprotocols` doesn't include one of them, `coder/websocket` completes the handshake with *no* agreed subprotocol and the browser's `WebSocket` fires an error/close. Always advertise `"chat.v1"` server-side and have the client offer it first. Do **not** put the token itself in `AcceptOptions.Subprotocols`.

### 10.4 CSWSH and OriginPatterns **[A]**

**Cross-Site WebSocket Hijacking (CSWSH)** is the WebSocket cousin of CSRF: a malicious page in the user's browser opens a WebSocket to *your* server; because cookies (and here, if you used a cookie/ticket) ride along automatically, the attacker's page could drive an authenticated socket. The defense is to **verify the `Origin` header** against an allowlist. `coder/websocket`'s `Accept` is safe-by-default: it **rejects cross-origin handshakes unless you set `OriginPatterns`**. We set it to our known web origins from config. Because our WS auth is a token in the subprotocol (not an ambient cookie), CSWSH is already weaker against us, but we keep the Origin allowlist as defense in depth.

```go
// AcceptOptions.OriginPatterns supports wildcards like "*.example.com".
OriginPatterns: []string{"chat.example.com", "localhost:3000"},
```

> **Do not "fix" a blocked connection with `InsecureSkipVerify` or an allow-all pattern.** The equivalent mistake in gorilla is `CheckOrigin: return true`. Setting `OriginPatterns: []string{"*"}` re-opens CSWSH. If your dev client is blocked, add its *exact* origin to the allowlist instead.

### 10.5 The Client: read pump, write pump, heartbeat, backpressure **[A]**

Each `Client` runs exactly two goroutines. The **read pump** is the single reader (`coder/websocket` allows only one reader at a time) — it reads envelopes and hands them to the Hub/handlers. The **write pump** is the single writer — it drains the bounded `out` channel and also sends periodic **Pings** to detect dead peers. Backpressure is enforced by the *bounded* `out` channel: if the Hub can't enqueue because the buffer is full, that client is a **slow consumer** and gets evicted ([§10.2](#10-realtime-with-coderwebsocket--the-hub-architecture) `send`).

```go
// internal/ws/client.go
package ws

import (
	"context"
	"encoding/json"
	"time"

	"github.com/coder/websocket"
	"github.com/google/uuid"
)

const (
	writeWait      = 10 * time.Second
	pingPeriod     = 30 * time.Second
	sendBuffer     = 64 // bounded: slow clients fill this and get dropped
	websocketGoingAway     = websocket.StatusGoingAway
	websocketTryAgainLater = websocket.StatusTryAgainLater
)

type Client struct {
	hub    *Hub
	conn   *websocket.Conn
	userID uuid.UUID
	rooms  []uuid.UUID     // rooms this connection is subscribed to
	out    chan []byte     // bounded outbound queue
	maxMsg int64
}

func newClient(h *Hub, conn *websocket.Conn, uid uuid.UUID, rooms []uuid.UUID, maxMsg int64) *Client {
	return &Client{hub: h, conn: conn, userID: uid, rooms: rooms, out: make(chan []byte, sendBuffer), maxMsg: maxMsg}
}

func (c *Client) close(code websocket.StatusCode, reason string) {
	_ = c.conn.Close(code, reason)
}

// run starts both pumps and blocks until either ends.
func (c *Client) run(ctx context.Context) {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	go c.writePump(ctx, cancel)
	c.readPump(ctx, cancel) // the read pump runs on this goroutine
}

// readPump is the SINGLE reader. It reads envelopes, enforces size, and dispatches.
func (c *Client) readPump(ctx context.Context, cancel context.CancelFunc) {
	defer cancel()
	for {
		typ, data, err := c.conn.Read(ctx)
		if err != nil {
			// Normal close, going away, or a real error — all end the pump.
			return
		}
		if typ != websocket.MessageText {
			c.close(websocket.StatusUnsupportedData, "text only")
			return
		}
		var env Envelope
		if err := json.Unmarshal(data, &env); err != nil {
			c.sendError("", "BAD_ENVELOPE", "malformed json")
			continue
		}
		c.hub.dispatch(ctx, c, &env) // route by env.Type (§12)
	}
}

// writePump is the SINGLE writer. It drains out and sends Pings on a timer.
// coder/websocket serializes writes internally, but a single writer keeps
// backpressure and ordering under OUR control.
func (c *Client) writePump(ctx context.Context, cancel context.CancelFunc) {
	defer cancel()
	ticker := time.NewTicker(pingPeriod)
	defer ticker.Stop()
	for {
		select {
		case <-ctx.Done():
			return
		case data, ok := <-c.out:
			if !ok { // Hub closed the channel: we've been unregistered
				c.close(websocket.StatusNormalClosure, "bye")
				return
			}
			wctx, wcancel := context.WithTimeout(ctx, writeWait)
			err := c.conn.Write(wctx, websocket.MessageText, data)
			wcancel()
			if err != nil {
				return
			}
		case <-ticker.C:
			// Ping must run concurrently with the reader; it does (separate goroutine).
			pctx, pcancel := context.WithTimeout(ctx, writeWait)
			err := c.conn.Ping(pctx)
			pcancel()
			if err != nil {
				return // peer didn't pong in time — dead connection
			}
		}
	}
}

func (c *Client) enqueue(data []byte) { c.hub.send(c, data) }
```

> **Why a single write pump if `coder/websocket` writes are concurrency-safe?** The library *does* serialize concurrent `Write` calls, so you *can* write from many goroutines safely. But you still want a **single writer draining a bounded channel** for three reasons the library can't give you: (1) **backpressure** — a bounded queue lets you detect and evict slow consumers deterministically; (2) **ordering** — messages leave in the order they were enqueued; (3) **lifecycle** — closing the channel is a clean shutdown signal. Concurrency-safety and backpressure are different problems; the library solves the first, your pump solves the second.

> **⚡ Version note — `SetReadLimit` default is 32768.** `coder/websocket` caps inbound frames at 32 KiB by default and returns `StatusMessageTooBig` beyond it. We set it explicitly to our per-message cap plus envelope overhead so an attacker can't send a 10 MB frame to exhaust memory. Enforce the *business* size (`body` ≤ 8 KiB) separately in the handler; `SetReadLimit` is the transport-level backstop.

### 10.6 Graceful drain on shutdown **[A]**

When `main` receives a signal it calls `hub.Drain()` ([§3.3](#3-project-layout-config--graceful-shutdown)), which sends each client a **`StatusGoingAway` (1001)** close frame. The frontend treats 1001 specially: it does *not* show an error, it just **reconnects** — landing on another node behind the load balancer. This turns a deploy from "everyone sees Disconnected" into a sub-second, invisible reconnection. We wire the frontend side in [§18.4](#18-the-frontend-websocket-client).

---

## 11. The Realtime Protocol — Envelope & Events

Before the send flow we pin down the **wire protocol**: one envelope, a fixed vocabulary of event types, and versioning. A disciplined protocol is what keeps the frontend and backend from drifting and makes the socket debuggable — you can read a frame and know exactly what it means.

### 11.1 The envelope **[I]**

Every frame in either direction is a JSON object with the same top-level fields. Keeping the shape identical in both directions means one parser, one type, one mental model.

```go
// internal/ws/protocol.go
package ws

import (
	"encoding/json"
	"github.com/google/uuid"
)

type Envelope struct {
	V           int             `json:"v"`                     // protocol version (1)
	Type        string          `json:"type"`                  // e.g. "message.new"
	RoomID      *uuid.UUID      `json:"room_id,omitempty"`     // omitted for global events
	ClientMsgID *uuid.UUID      `json:"client_msg_id,omitempty"` // send/ack idempotency
	Payload     json.RawMessage `json:"payload,omitempty"`     // type-specific body
}

func encode(typ string, room *uuid.UUID, cmid *uuid.UUID, payload any) ([]byte, error) {
	p, err := json.Marshal(payload)
	if err != nil {
		return nil, err
	}
	return json.Marshal(Envelope{V: 1, Type: typ, RoomID: room, ClientMsgID: cmid, Payload: p})
}
```

We deliberately keep `Payload` as `json.RawMessage` so the read pump can cheaply route on `Type` and only unmarshal the payload once the handler knows the concrete type — avoiding a giant tagged union parse on every frame.

### 11.2 The event vocabulary **[I]**

| `type` | Direction | Payload | Meaning |
|---|---|---|---|
| `message.send` | client → server | `{ body }` + `client_msg_id` | user posts a message to `room_id` |
| `message.new` | server → clients | `MessageDTO` | a new message exists in `room_id` (fan-out) |
| `message.ack` | server → sender | `{ id, created_at }` + `client_msg_id` | the sender's message was persisted |
| `message.error` | server → sender | `{ code, reason }` + `client_msg_id` | the send was rejected (authz/validation) |
| `typing.start` / `typing.stop` | both | `{ user_id }` | typing indicator (ephemeral) |
| `presence.update` | server → clients | `{ user_id, status }` | a user went online/offline |
| `receipt.read` | both | `{ user_id, message_id }` | user read up to a message |

That is the *entire* realtime surface. Notice the asymmetry: the client sends `message.send`; the server responds to the *sender* with `message.ack` (or `message.error`) **and** broadcasts `message.new` to *everyone in the room including the sender's other tabs*. The sender's own tab that initiated the send reconciles its optimistic bubble via the `ack`'s `client_msg_id` and ignores the echoed `message.new` for that id ([§18.5](#18-the-frontend-websocket-client)).

### 11.3 Versioning **[I]**

The `v` field and the `chat.v1` subprotocol are your evolution levers. Rules we follow:

- **Additive changes don't bump `v`.** Adding an optional field to a payload, or a new event type, is backward-compatible: old clients ignore what they don't understand. Design payloads as open maps, not fixed tuples.
- **Breaking changes bump `v` and the subprotocol** (`chat.v2`). During a migration the server can accept both `chat.v1` and `chat.v2`, translating v1 frames internally, until old clients drain. This is why the subprotocol carries the version, not just the auth token.
- **Never reuse a `type` name with new semantics.** Add `message.new2` or a versioned payload rather than silently changing what `message.new` means. Debuggability depends on names being stable.

> **Best practice — validate the envelope at the boundary.** The read pump rejects any frame with `V != 1`, an unknown `Type`, or (for room events) a `RoomID` the sender isn't a member of. Treat every inbound frame as hostile until validated; the socket is an attacker-controlled input just like an HTTP body ([§21.4](#21-banking-grade-security-hardening)).

---

## 12. The Message Send Flow End-to-End

Now we assemble [§6](#6-sqlc--type-safe-hot-path-queries) (the idempotent insert), [§8](#8-rbac--authorization--room-roles-and-idor-defense) (membership authz), [§10](#10-realtime-with-coderwebsocket--the-hub-architecture) (the Hub), and [§11](#11-the-realtime-protocol--envelope--events) (the protocol) into the single most important code path: a user sends a message and everyone in the room sees it, exactly once, durably.

### 12.1 The steps, named **[A]**

When a `message.send` envelope arrives on a client's read pump, the server:

1. **Authorizes** — is `sender` a member of `room_id`? (Membership check; non-members are silently dropped or error'd — never broadcast.)
2. **Validates** — non-empty after trim, within the size cap, correct types.
3. **Persists idempotently** — `InsertMessage` inside a `pgx.Tx`, keyed by `client_msg_id`; on conflict, read back the existing row (a retry).
4. **Acks the sender** — `message.ack` with the server id + timestamp, keyed by `client_msg_id`, so the sender's optimistic bubble becomes confirmed.
5. **Fans out** — `message.new` to every member's connections across the whole cluster via the Redis backplane ([§15](#15-scaling-out--the-redis-pubsub-backplane--nginx)), which lands as `BroadcastLocal` on each node.
6. **Bumps unread** — recipients' unread counts increase; the frontend derives this from the new message vs. their last-read pointer ([§14](#14-read-receipts--unread-counts)).

### 12.2 The dispatch router **[A]**

`Hub.dispatch` routes an envelope by `Type` to the right handler. It is the server-side mirror of the protocol table.

```go
// internal/ws/dispatch.go
package ws

import (
	"context"
	"encoding/json"
)

func (h *Hub) dispatch(ctx context.Context, c *Client, env *Envelope) {
	switch env.Type {
	case "message.send":
		h.handleSend(ctx, c, env)
	case "typing.start":
		h.handleTyping(ctx, c, env, true)
	case "typing.stop":
		h.handleTyping(ctx, c, env, false)
	case "receipt.read":
		h.handleReceipt(ctx, c, env)
	default:
		c.sendError(env.ClientMsgID, "UNKNOWN_TYPE", "unsupported event type")
	}
}
```

### 12.3 The send handler **[A]**

This is the code that ties everything together. Read the comments — every numbered step from [§12.1](#12-the-message-send-flow-end-to-end) is here.

```go
// internal/ws/send.go
package ws

import (
	"context"
	"encoding/json"
	"strings"

	"github.com/google/uuid"
	gen "chat/db/gen"
)

type sendPayload struct {
	Body string `json:"body"`
}

func (h *Hub) handleSend(ctx context.Context, c *Client, env *Envelope) {
	// --- guard: room + client_msg_id are required for a send ---
	if env.RoomID == nil || env.ClientMsgID == nil {
		c.sendError(env.ClientMsgID, "BAD_SEND", "room_id and client_msg_id required")
		return
	}
	roomID, cmid := *env.RoomID, *env.ClientMsgID

	// (1) AUTHORIZE — must be a member of the room. This is the realtime IDOR guard,
	// mirroring the REST membership check (§8). We check against the connection's
	// known rooms first (cheap), then trust the DB constraint as backstop.
	if !c.inRoom(roomID) {
		c.sendError(&cmid, "FORBIDDEN", "not a member")
		return
	}

	// (2) VALIDATE
	var p sendPayload
	if err := json.Unmarshal(env.Payload, &p); err != nil {
		c.sendError(&cmid, "BAD_PAYLOAD", "invalid payload")
		return
	}
	body := strings.TrimSpace(p.Body)
	if body == "" || int64(len(body)) > c.maxMsg {
		c.sendError(&cmid, "INVALID_BODY", "empty or too long")
		return
	}

	// (3) PERSIST IDEMPOTENTLY inside a transaction.
	tx, err := h.store.Pool.Begin(ctx)
	if err != nil {
		c.sendError(&cmid, "INTERNAL", "db")
		return
	}
	defer tx.Rollback(ctx) // no-op after Commit
	q := h.store.Q.WithTx(tx)

	row, err := q.InsertMessage(ctx, gen.InsertMessageParams{
		RoomID:      roomID,
		SenderID:    c.userID,
		Body:        body,
		ClientMsgID: cmid,
	})
	if err != nil {
		c.sendError(&cmid, "INTERNAL", "insert")
		return
	}
	// InsertMessage RETURNs nothing on ON CONFLICT DO NOTHING (a retry). Read it back
	// so a resend still gets a correct ack with the ORIGINAL id — exactly-once effect.
	if row.ID == uuid.Nil {
		row, err = q.SelectMessageByIdem(ctx, gen.SelectMessageByIdemParams{
			RoomID: roomID, SenderID: c.userID, ClientMsgID: cmid,
		})
		if err != nil {
			c.sendError(&cmid, "INTERNAL", "readback")
			return
		}
	}
	if err := tx.Commit(ctx); err != nil {
		c.sendError(&cmid, "INTERNAL", "commit")
		return
	}

	dto := toWSMessageDTO(row)

	// (4) ACK the sender (keyed by client_msg_id → reconciles the optimistic bubble).
	if ackData, err := encode("message.ack", &roomID, &cmid, ackPayload{
		ID: row.ID, CreatedAt: row.CreatedAt,
	}); err == nil {
		c.enqueue(ackData)
	}

	// (5) FAN OUT message.new to the whole cluster. The backplane publishes to Redis;
	// every node (including THIS one) receives it and calls BroadcastLocal to its
	// local room subscribers. We skip re-sending message.new to the origin connection
	// only if you prefer ack-only reconciliation; here we DO send it (other tabs need it)
	// and the origin tab dedupes by client_msg_id (§18.5).
	newData, _ := encode("message.new", &roomID, &cmid, dto)
	h.bp.Publish(ctx, roomID, newData) // → backplane → BroadcastLocal on every node
}
```

### 12.4 Idempotency, exactly-once, and ordering **[A]**

Three properties this flow guarantees, and *how*:

- **Exactly-once persistence** comes from the unique index `(room_id, sender_id, client_msg_id)` plus `ON CONFLICT DO NOTHING` + read-back. A client that resends after a dropped ack gets the *same* stored row and the *same* ack — no duplicate message. The client generates `client_msg_id` (a UUID) *once* per composed message and reuses it across retries ([§18.5](#18-the-frontend-websocket-client)).
- **At-least-once delivery to recipients** is what the fan-out provides; combined with the recipient deduping by message `id`, the *effect* is exactly-once display. A recipient that was briefly disconnected and missed the `message.new` fills the gap by **refetching history on reconnect** ([§18.4](#18-the-frontend-websocket-client)) — the socket is a freshness optimization, never the source of truth.
- **Per-room ordering** is defined by `(created_at, id)` — the same tuple the keyset index and cursor use. Because the DB assigns `created_at` at insert and the fan-out carries the persisted row, all clients converge on the DB's order even if frames arrive slightly reordered; the client inserts by `(created_at, id)`, not arrival order.

> **Gotcha — never broadcast before commit.** If you fan out `message.new` and *then* the transaction fails, recipients saw a message that doesn't exist. Persist-then-commit-then-broadcast, in that order. The code above only publishes after `tx.Commit` returns nil. This is the realtime equivalent of "don't send the email inside the transaction."

---
## 13. Presence & Typing Indicators

Presence ("who is online") and typing indicators ("Alice is typing…") are **ephemeral** state — they are true for seconds, change constantly, and losing them on a restart is harmless. That single observation dictates the whole design: **never persist them to Postgres.** They live in memory per node, with **Redis** as the cross-node aggregator and a **TTL** as the self-healing garbage collector. This section builds both on top of the [§10](#10-realtime-with-coderwebsocket--the-hub-architecture) Hub and the [§11](#11-the-realtime-protocol--envelope--events) envelope.

### 13.1 Why ephemeral state is a different problem **[I]**

A message is a durable fact: it happened, it must survive forever, and correctness demands exactly-once storage. Presence is the opposite: it is a *continuously-refreshed guess* about the current instant. If you tried to store presence in Postgres you would get a write storm (every connect/disconnect/tab-switch is a row change), you would leak "ghost" online users whenever a process crashed without cleaning up, and you would still be wrong the moment you read it. The correct tools for "true-for-now, self-expiring" state are **in-memory sets** (fast, local) plus **Redis keys with TTL** (shared, self-cleaning). The TTL is the key insight: instead of trusting every client to send a clean "I'm leaving," you require every online client to **refresh** a short-lived key; if a node dies, its keys simply *expire* and presence heals itself with no cleanup code.

### 13.2 Typing indicators — pure fan-out, no storage **[A]**

Typing is the simplest realtime feature and a good warm-up: it is a **pass-through broadcast** with zero persistence. When a client sends `typing.start`, the server verifies membership and rebroadcasts `typing.start` (carrying the typist's `user_id`) to the *other* members of the room. The frontend shows the indicator and auto-clears it after a couple of seconds if no further `typing.start` arrives, so a dropped `typing.stop` never leaves a stuck indicator.

```go
// internal/ws/typing.go
package ws

import (
	"context"
	"encoding/json"

	"github.com/google/uuid"
)

type typingPayload struct {
	UserID uuid.UUID `json:"user_id"`
}

func (h *Hub) handleTyping(ctx context.Context, c *Client, env *Envelope, start bool) {
	if env.RoomID == nil || !c.inRoom(*env.RoomID) {
		return // silently ignore: typing is best-effort, never error the client for it
	}
	typ := "typing.stop"
	if start {
		typ = "typing.start"
	}
	// The payload carries WHO is typing (the server's trusted identity, not client-claimed).
	data, err := encode(typ, env.RoomID, nil, typingPayload{UserID: c.userID})
	if err != nil {
		return
	}
	// Broadcast to the room but SKIP the typist's own connection — you don't need to
	// see your own typing indicator. Cluster-wide via the backplane (§15).
	h.bp.PublishSkipUser(ctx, *env.RoomID, data, c.userID)
}
```

Two design choices worth internalizing: the server stamps the `user_id` from `c.userID` (the authenticated identity), **never** from client-supplied payload — otherwise Mallory could forge "Alice is typing." And we throttle on the *client* side (send at most one `typing.start` per ~2s while the composer has focus and content) so a fast typist doesn't emit a frame per keystroke; the server just relays.

### 13.3 Presence — in-memory per node + Redis TTL **[A]**

Presence must answer "is user U online *anywhere in the cluster*?" No single node knows the whole answer, so we aggregate through Redis. The scheme:

- On connect, the node adds the user to its **local** online set and writes a Redis key `presence:{userID}` with a **TTL** (say 45s), then publishes a `presence.update{status:"online"}` to the user's rooms.
- Every online connection **refreshes** that TTL on a heartbeat (piggy-backed on the Ping timer, ~every 30s < 45s TTL). As long as *any* of the user's tabs on *any* node is alive, the key stays warm.
- On disconnect, the node checks whether the user still has *any* local connection; if not, it does **not** immediately mark them offline (another node might have them) — it lets the Redis TTL decide. When the key finally expires (no node refreshed it), a keyspace-notification or a lazy check on next read declares them offline and publishes `presence.update{status:"offline"}`.

```go
// internal/presence/presence.go
package presence

import (
	"context"
	"fmt"
	"time"

	"github.com/google/uuid"
	"github.com/redis/go-redis/v9"
)

const presenceTTL = 45 * time.Second

type Tracker struct {
	rdb *redis.Client
}

func New(rdb *redis.Client) *Tracker { return &Tracker{rdb: rdb} }

func key(u uuid.UUID) string { return fmt.Sprintf("presence:%s", u) }

// Online marks the user present and (re)arms the TTL. Called on connect AND on each
// heartbeat. SET with EX is idempotent — refreshing is just another SET.
func (t *Tracker) Online(ctx context.Context, u uuid.UUID) error {
	return t.rdb.Set(ctx, key(u), "1", presenceTTL).Err()
}

// IsOnline is a cheap existence check for rendering another user's presence dot.
func (t *Tracker) IsOnline(ctx context.Context, u uuid.UUID) (bool, error) {
	n, err := t.rdb.Exists(ctx, key(u)).Result()
	return n == 1, err
}

// OnlineSet returns which of a set of users are currently online (for a room roster).
// Uses a pipeline so one round-trip covers the whole room.
func (t *Tracker) OnlineSet(ctx context.Context, users []uuid.UUID) (map[uuid.UUID]bool, error) {
	pipe := t.rdb.Pipeline()
	cmds := make([]*redis.IntCmd, len(users))
	for i, u := range users {
		cmds[i] = pipe.Exists(ctx, key(u))
	}
	if _, err := pipe.Exec(ctx); err != nil {
		return nil, err
	}
	out := make(map[uuid.UUID]bool, len(users))
	for i, u := range users {
		out[u] = cmds[i].Val() == 1
	}
	return out, nil
}
```

We wire `Online` into the connect path and the write pump's ping tick, and we broadcast a `presence.update` when a user *transitions* (their first connection online, or their last connection gone). The transition detection uses the Hub's `byUser` map ([§10.2](#10-realtime-with-coderwebsocket--the-hub-architecture)): if `len(byUser[uid])` goes 0→1 we publish online; if a disconnect takes it 1→0 we schedule an offline check.

```go
// In the Hub, on register:
if first := len(h.byUser[c.userID]) == 1; first { // just became 1 → transitioned online
	_ = h.presence.Online(ctx, c.userID)
	h.publishPresence(ctx, c, "online")
}
// On the write pump's ping tick, refresh the TTL so we never expire while connected:
_ = h.presence.Online(ctx, c.userID)
```

### 13.4 Broadcasting presence to the right people **[A]**

You do not broadcast a presence change to the *whole cluster* — only to users who **share a room** with the changed user (they're the only ones who can see their dot). We compute the union of the user's rooms and publish `presence.update` to each, letting the backplane fan it out. This keeps presence traffic proportional to social graph density, not user count.

```go
func (h *Hub) publishPresence(ctx context.Context, c *Client, status string) {
	payload := map[string]any{"user_id": c.userID, "status": status}
	for _, roomID := range c.rooms {
		data, _ := encode("presence.update", &roomID, nil, payload)
		h.bp.Publish(ctx, roomID, data)
	}
}
```

> **Gotcha — presence and the "ghost user."** Without a TTL, a node that crashes (OOM, `kill -9`) never sends "offline," and that user appears online forever. The TTL is the *only* reliable cleanup: it assumes nothing about clean shutdown. Set the refresh interval comfortably below the TTL (30s refresh, 45s TTL) so a single missed heartbeat doesn't flap the user offline. If you need instant, precise offline (e.g., "last seen"), layer Redis keyspace notifications (`notify-keyspace-events Ex`) to fire the moment a key expires — see the [Redis guide](REDIS_GUIDE.md).

---

## 14. Read Receipts & Unread Counts

Unread counts and read receipts are two views of the same fact — **how far each user has read in each room** — surfaced differently: unread counts drive *your own* sidebar badges; receipts show *others* that you've seen their messages. Both are anchored by the per-membership pointer `room_members.last_read_message_id` and the `read_receipts` table from [§4.5](#4-the-database-schema-via-goose-migrations), and both use the sqlc queries from [§6.3](#6-sqlc--type-safe-hot-path-queries).

### 14.1 The model: one pointer, two consumers **[I]**

Rather than store a row per (user, message) marking each as read — which explodes in volume — we store a single **high-water mark** per (room, user): the id of the newest message that user has read. Everything derives from it:

- **Your unread count** for a room = messages newer than *your* pointer, not sent by you (`CountUnreadForRoom`).
- **What others have read** = *their* pointers; the "seen" checkmark under a message appears when some other member's pointer is at-or-past that message.

Moving the pointer is monotonic (forward only) — scrolling up and back down must never *decrease* what you've read. The `MarkRoomRead` query enforces this with a guard comparing timestamps ([§6.3](#6-sqlc--type-safe-hot-path-queries)).

### 14.2 Marking read — REST and realtime **[I/A]**

A client marks read when the newest message becomes visible at the bottom of the viewport. It can do so two ways, and we support both: a REST `POST /rooms/:id/read` (reliable, used on initial open and on blur) and a `receipt.read` WebSocket frame (instant, used while actively viewing). Both funnel into the same service method, which updates the pointer, upserts the receipt, and broadcasts `receipt.read` to the room so other members' "seen" state updates live.

```go
// internal/ws/receipt.go
package ws

import (
	"context"
	"encoding/json"

	"github.com/google/uuid"
	gen "chat/db/gen"
)

type receiptPayload struct {
	MessageID uuid.UUID `json:"message_id"`
}

func (h *Hub) handleReceipt(ctx context.Context, c *Client, env *Envelope) {
	if env.RoomID == nil || !c.inRoom(*env.RoomID) {
		return
	}
	var p receiptPayload
	if err := json.Unmarshal(env.Payload, &p); err != nil {
		return
	}
	roomID := *env.RoomID

	// Move the high-water mark forward (idempotent, monotonic) and upsert the receipt.
	// Both writes are sqlc — this is the hot path, not graph CRUD.
	if _, err := h.store.Q.MarkRoomRead(ctx, gen.MarkRoomReadParams{
		RoomID: roomID, UserID: c.userID, MessageID: p.MessageID,
	}); err != nil {
		return
	}
	_ = h.store.Q.UpsertReadReceipt(ctx, gen.UpsertReadReceiptParams{
		RoomID: roomID, UserID: c.userID, MessageID: p.MessageID,
	})

	// Broadcast so OTHER members see the "seen" update live. The server stamps user_id
	// from the authenticated identity — never trust a client-claimed reader.
	data, _ := encode("receipt.read", &roomID, nil, map[string]any{
		"user_id": c.userID, "message_id": p.MessageID,
	})
	h.bp.PublishSkipUser(ctx, roomID, data, c.userID)
}
```

### 14.3 Delivering unread counts to the client **[I]**

Unread counts appear in two places with two mechanisms, and it's important to see why:

1. **On load**, the sidebar's `GET /conversations` returns each room with its `unread` count (the `ListRecentConversations` + `CountUnreadForRoom` queries). This is the authoritative snapshot.
2. **Live**, the frontend does *not* re-query on every message. Instead, when a `message.new` arrives for a room the user is **not currently viewing**, the client **increments** that room's unread badge locally in the TanStack Query cache ([§18.6](#18-the-frontend-websocket-client)); when they open/scroll-to-bottom of a room, it **resets** to zero and sends a `receipt.read`. The server count and the client's derived count reconcile on the next `GET /conversations` (e.g., on reconnect).

This "authoritative snapshot + local increments + periodic reconciliation" pattern is the standard way to keep counters live without hammering the database. The database is the source of truth; the live count is a cheap local optimization that self-corrects.

### 14.4 Reconciliation and edge cases **[A]**

- **Multi-device.** You read a room on your phone; your laptop's badge must clear too. Because the pointer is server-side per (room, user), the laptop learns the new pointer on its next `GET /conversations` or when it receives the `receipt.read` for its own `user_id` (we *do* deliver `receipt.read` to the reader's *other* connections, just not the initiating one — a small tweak to `PublishSkipUser` to skip only the origin connection, not all of the user's). Simplest: on receiving a `receipt.read` for *your own* user id, set that room's unread to 0 in the cache.
- **Counting your own messages.** Sending a message must not increase your own unread; `CountUnreadForRoom` excludes `sender_id = user`, and the client resets on send.
- **The pointer references a deleted message.** `last_read_message_id` has `ON DELETE SET NULL` semantics if you allow deletes; treat NULL as "unread from the beginning" and let the next read re-anchor it.

> **Best practice — never let the badge be the source of truth.** The badge is derived UI. If it ever disagrees with reality (a missed frame, a race), the next `GET /conversations` overwrites it from the database. Design counters so a full refetch always *corrects* them; never accumulate state the server can't reproduce.

---

## 15. Scaling Out — The Redis Pub/Sub Backplane & Nginx

Everything so far works on **one** node. The moment you run a second API instance behind a load balancer, a message sent to a client on node A must reach a member connected to node B — and node A's in-memory Hub has never heard of node B's connections. The fix is a **backplane**: a shared bus every node publishes to and subscribes from. We use **Redis pub/sub**. This section is the full worked implementation plus the Nginx config that puts REST, WebSocket, and the frontend behind **one origin** (which keeps cookies and CORS sane).

### 15.1 The single-node ceiling **[A]**

A single Go process can hold a very large number of idle WebSocket connections (tens of thousands, memory permitting) — Go's goroutine-per-connection model is cheap. So why scale out at all? Three reasons: **redundancy** (one node's crash or deploy shouldn't drop everyone), **CPU** (message fan-out, JSON encoding, and TLS termination are CPU-bound and one core saturates), and **rolling deploys** (you need ≥2 nodes to deploy without downtime). Once you have ≥2 nodes, cross-node broadcast is mandatory — hence the backplane.

### 15.2 The backplane design **[A]**

The contract is simple: when any node wants to broadcast to room R, it **publishes** the (already-encoded) envelope to a Redis channel named for that room (`room:{roomID}`). Every node **subscribes** to the channels for rooms it currently has local subscribers in, and when it receives a published message it calls `hub.BroadcastLocal` to fan out to *its* clients in that room. The Hub's local fan-out ([§10.2](#10-realtime-with-coderwebsocket--the-hub-architecture)) doesn't change at all — the backplane simply feeds it from the network in addition to feeding it locally.

```
   node A                        Redis                        node B
   handleSend ──publish room:R──▶ PUBLISH ──deliver to───────▶ subscriber
                                  channel   all subscribers    │
   subscriber ◀──deliver──────────                             ▼
        │                                              BroadcastLocal(R)
        ▼                                                      │
   BroadcastLocal(R) → node A's clients in R          node B's clients in R
```

A subtlety: node A publishes *and* subscribes, so node A receives its **own** published message back and fans it out locally. That means `handleSend` should **not** also call `BroadcastLocal` directly — it publishes, and the fan-out happens uniformly when the message comes back through the subscription. One code path for both local and remote delivery; no double-sends.

### 15.3 The backplane implementation **[A]**

We subscribe to a **single pattern** `room:*` rather than managing per-room subscribe/unsubscribe churn as clients come and go. For very high room counts you'd switch to targeted `SUBSCRIBE`/`UNSUBSCRIBE` keyed by which rooms a node actually holds; the pattern approach is simpler and fine to a large scale because a node ignores messages for rooms it has no local subscribers in.

```go
// internal/backplane/backplane.go
package backplane

import (
	"context"
	"encoding/json"
	"fmt"

	"github.com/google/uuid"
	"github.com/redis/go-redis/v9"
)

type Backplane struct {
	rdb *redis.Client
}

// envelope wrapper carried over Redis: the room, the pre-encoded frame, and an
// optional skip (origin connection or user) so we don't echo where we shouldn't.
type busMsg struct {
	Room     uuid.UUID `json:"room"`
	Data     []byte    `json:"data"`      // the already-encoded WS envelope
	SkipUser uuid.UUID `json:"skip_user"` // uuid.Nil = skip nobody
}

func New(ctx context.Context, url string) (*Backplane, error) {
	opt, err := redis.ParseURL(url)
	if err != nil {
		return nil, err
	}
	rdb := redis.NewClient(opt)
	if err := rdb.Ping(ctx).Err(); err != nil {
		return nil, fmt.Errorf("redis ping: %w", err)
	}
	return &Backplane{rdb: rdb}, nil
}

func channel(room uuid.UUID) string { return "room:" + room.String() }

// Publish sends a frame to every node subscribed to the room's channel (incl. self).
func (b *Backplane) Publish(ctx context.Context, room uuid.UUID, data []byte) {
	msg, _ := json.Marshal(busMsg{Room: room, Data: data})
	_ = b.rdb.Publish(ctx, channel(room), msg).Err()
}

// PublishSkipUser is like Publish but tells each node to skip a given user's conns
// (used for typing/receipts so the actor doesn't get their own echo).
func (b *Backplane) PublishSkipUser(ctx context.Context, room uuid.UUID, data []byte, skip uuid.UUID) {
	msg, _ := json.Marshal(busMsg{Room: room, Data: data, SkipUser: skip})
	_ = b.rdb.Publish(ctx, channel(room), msg).Err()
}

// LocalFanout is the interface the Hub satisfies — decouples backplane from ws pkg.
type LocalFanout interface {
	FanoutLocal(room uuid.UUID, data []byte, skipUser uuid.UUID)
}

// Consume subscribes to all room channels and feeds every received frame into the
// local Hub. Runs for the process lifetime; called as `go bp.Consume(ctx, hub)`.
func (b *Backplane) Consume(ctx context.Context, hub LocalFanout) {
	sub := b.rdb.PSubscribe(ctx, "room:*")
	defer sub.Close()
	ch := sub.Channel()
	for {
		select {
		case <-ctx.Done():
			return
		case m, ok := <-ch:
			if !ok {
				return
			}
			var bm busMsg
			if err := json.Unmarshal([]byte(m.Payload), &bm); err != nil {
				continue
			}
			hub.FanoutLocal(bm.Room, bm.Data, bm.SkipUser) // → §10.2, feeds local clients
		}
	}
}

func (b *Backplane) Close() error { return b.rdb.Close() }
```

And the Hub grows a `FanoutLocal` that respects `skipUser` (skipping *all* of a user's local connections for typing/receipts):

```go
// internal/ws/hub.go (addition)
func (h *Hub) FanoutLocal(room uuid.UUID, data []byte, skipUser uuid.UUID) {
	h.broadcast <- roomFrame{room: room, data: data, skipUserID: skipUser}
}
// In Run's broadcast case, extend the skip test:
//   for c := range h.rooms[rf.room] {
//       if rf.skipUserID != uuid.Nil && c.userID == rf.skipUserID { continue }
//       h.send(c, rf.data)
//   }
```

> **Gotcha — Redis pub/sub is fire-and-forget.** A node that is momentarily disconnected from Redis *misses* messages published during the gap — pub/sub does not buffer. That's acceptable for chat because the **durable** copy is in Postgres and clients **refetch history on reconnect** ([§18.4](#18-the-frontend-websocket-client)); the backplane only accelerates delivery. If you need guaranteed delivery (e.g., cross-node presence you can't reconstruct), use Redis Streams (`XADD`/`XREADGROUP`) instead — see the [Redis guide](REDIS_GUIDE.md). For this app, pub/sub + Postgres-as-truth is the right, simple choice.

### 15.4 Presence at scale **[A]**

Presence already scales because we put it in Redis in [§13.3](#13-presence--typing-indicators): the TTL keys are shared, so *any* node can answer "is U online?" without knowing which node holds U's connection. Nothing changes when you add nodes — that's the payoff of designing presence around Redis from the start rather than an in-process map.

### 15.5 Stateless nodes & load balancing **[A]**

With the backplane and Redis-backed presence, **API nodes are effectively stateless** for correctness: any client can connect to any node and receive all their room traffic. This means you do **not** need sticky sessions for correctness — a reconnecting client can land on a different node freely. You may still enable connection-level stickiness for minor efficiency (keeping a client on one node reduces reconnect churn), but it's an optimization, not a requirement. Because auth rides in the JWT/subprotocol (not a server session), there's no session affinity to preserve. Scale is then just "add nodes behind the balancer"; the ceiling moves to Redis throughput and Postgres write capacity, both far higher than a single API node.

### 15.6 Nginx: one origin for REST + WS + frontend **[A]**

The cleanest production topology serves the Next.js app, the REST API, and the WebSocket **from one origin** (e.g. `https://chat.example.com`). One origin means **no CORS** for same-origin fetches and — critically — the refresh **cookie is first-party**, so `SameSite=Strict` works and you avoid the `SameSite=None; Secure` cross-site dance. Nginx reverse-proxies by path: `/api/*` and `/ws` to the Go service, everything else to Next.js. The WebSocket location needs the `Upgrade`/`Connection` headers and long timeouts.

```nginx
# nginx/chat.conf
map $http_upgrade $connection_upgrade {   # required for WS upgrade proxying
    default upgrade;
    ''      close;
}

upstream chat_api { server api:8080; }     # docker-compose service name
upstream chat_web { server web:3000; }

server {
    listen 443 ssl http2;
    server_name chat.example.com;

    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    # --- WebSocket: MUST proxy Upgrade headers and disable read timeout ---
    location /ws {
        proxy_pass http://chat_api;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 3600s;   # allow long-lived idle connections (Pings keep it warm)
        proxy_send_timeout 3600s;
    }

    # --- REST API ---
    location /api/ {
        proxy_pass http://chat_api;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # --- Everything else → the Next.js app ---
    location / {
        proxy_pass http://chat_web;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Redirect http → https so wss/https are always used.
server {
    listen 80;
    server_name chat.example.com;
    return 301 https://$host$request_uri;
}
```

> **Gotcha — the two headers that make or break WS through Nginx.** `proxy_http_version 1.1` plus `Upgrade`/`Connection` headers are mandatory; omit them and the upgrade silently degrades to a plain HTTP request that your handler rejects, and you get a mystifying "WebSocket connection failed." Also raise `proxy_read_timeout` well above your Ping period, or Nginx will sever idle-looking connections mid-conversation. See the [Nginx guide](NGINX_GUIDE.md) for TLS tuning and rate limiting at the edge.

> **⚡ Version note — sharing an origin makes the whole auth story simpler.** With one origin, the browser's `WebSocket` URL is `wss://chat.example.com/ws`, fetches are relative (`/api/...`), and the refresh cookie is first-party. If you *must* split origins (API on `api.example.com`, web on `app.example.com`), you then need CORS with `AllowCredentials`, an exact-origin allowlist, and `SameSite=None; Secure` on the cookie — all covered in the sibling [full-stack capstone](GO_GIN_NEXTJS_REALTIME_FULLSTACK_GUIDE.md). Prefer one origin.

---
## 16. Next.js Frontend — Setup & Auth

We now cross the network boundary. The frontend is a **Next.js 16 App Router** app run largely as a **client-rendered SPA**: a logged-in chat app is intensely interactive and session-bound, so there is little to server-render — most routes are Client Components guarded by an auth check. The auth model mirrors the backend contract from [§7](#7-authentication--argon2id-jwt-access--refresh-rotation): the **access token lives in memory**, the **refresh token is an HttpOnly cookie** the browser sends automatically, and a **silent-refresh interceptor** transparently renews the access token when it expires. See [Next.js 16](NEXTJS_16_GUIDE.md) and [React 19](REACT_19_GUIDE.md) for the framework specifics.

### 16.1 The token-storage decision, restated for the client **[I]**

This is the single most consequential security choice on the frontend, so we state it plainly. **The access token goes in a JavaScript variable (memory), never `localStorage`.** `localStorage` is readable by any script on the page, so one XSS bug turns into full account takeover. Memory is wiped on reload — which is *fine*, because on load we call `/api/auth/refresh`; the browser sends the HttpOnly refresh cookie (which JS cannot read), the server rotates it and returns a fresh access token, and we're authenticated again without the access token ever touching disk or JS-readable storage. The refresh cookie survives reloads *and* resists XSS. This split — memory access + HttpOnly refresh — is the whole game.

### 16.2 The in-memory token store and API client **[I/A]**

A tiny module holds the access token and an optional "on unauthenticated" callback (to redirect to login). The `apiFetch` wrapper attaches the token, and on a `401` it triggers a **single, shared** refresh (coalescing concurrent 401s so a burst of requests refreshes once, not N times).

```ts
// web/lib/auth-store.ts
let accessToken: string | null = null;
let onLogout: (() => void) | null = null;

export const authStore = {
  get: () => accessToken,
  set: (t: string | null) => { accessToken = t; },
  setOnLogout: (fn: () => void) => { onLogout = fn; },
  logout: () => { accessToken = null; onLogout?.(); },
};
```

```ts
// web/lib/api.ts — the fetch wrapper with silent refresh
import { authStore } from "./auth-store";

const BASE = ""; // same-origin (§15.6): fetch("/api/...") — no CORS

let refreshing: Promise<boolean> | null = null;

// Coalesce concurrent refreshes into ONE in-flight request.
async function refreshAccess(): Promise<boolean> {
  if (!refreshing) {
    refreshing = (async () => {
      const res = await fetch(`${BASE}/api/auth/refresh`, {
        method: "POST",
        credentials: "include", // send the HttpOnly refresh cookie
      });
      if (!res.ok) return false;
      const { accessToken } = await res.json();
      authStore.set(accessToken);
      return true;
    })().finally(() => { refreshing = null; });
  }
  return refreshing;
}

export async function apiFetch<T>(path: string, init: RequestInit = {}): Promise<T> {
  const doFetch = () =>
    fetch(`${BASE}${path}`, {
      ...init,
      credentials: "include",
      headers: {
        "Content-Type": "application/json",
        ...(authStore.get() ? { Authorization: `Bearer ${authStore.get()}` } : {}),
        ...init.headers,
      },
    });

  let res = await doFetch();
  if (res.status === 401) {
    // Access token expired/absent → try one silent refresh, then retry once.
    const ok = await refreshAccess();
    if (!ok) { authStore.logout(); throw new ApiError(401, "unauthenticated"); }
    res = await doFetch();
  }
  if (!res.ok) {
    const body = await res.json().catch(() => ({}));
    throw new ApiError(res.status, body.error ?? res.statusText, body.code);
  }
  return res.status === 204 ? (undefined as T) : res.json();
}

export class ApiError extends Error {
  constructor(public status: number, message: string, public code?: string) { super(message); }
}
```

> **Gotcha — the refresh-stampede.** When the access token expires, a chat screen can fire many requests at once (history, conversations, mark-read), each getting a 401. Without coalescing, each fires its own `/refresh`, and because refresh **rotates** the token, the parallel refreshes invalidate each other and trigger the reuse-detection ([§7.4](#7-authentication--argon2id-jwt-access--refresh-rotation)), logging the user out. The shared `refreshing` promise above ensures exactly one refresh runs; all callers await it. This is a mandatory pattern with rotating refresh tokens.

### 16.3 Bootstrapping auth on load & route guards **[I]**

On app mount we attempt a silent refresh to restore the session, and until it resolves we render nothing (or a splash). A guard hook redirects unauthenticated users to `/login`.

```tsx
// web/app/providers.tsx (Client Component)
"use client";
import { useEffect, useState } from "react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { useRouter } from "next/navigation";
import { authStore } from "@/lib/auth-store";

const queryClient = new QueryClient({
  defaultOptions: { queries: { staleTime: 30_000, refetchOnWindowFocus: false } },
});

export function Providers({ children }: { children: React.ReactNode }) {
  const router = useRouter();
  const [ready, setReady] = useState(false);

  useEffect(() => {
    authStore.setOnLogout(() => router.replace("/login"));
    // Try to restore the session from the refresh cookie on first paint.
    fetch("/api/auth/refresh", { method: "POST", credentials: "include" })
      .then((r) => (r.ok ? r.json() : null))
      .then((d) => { if (d?.accessToken) authStore.set(d.accessToken); })
      .finally(() => setReady(true));
  }, [router]);

  if (!ready) return <Splash />;
  return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>;
}
```

```tsx
// web/app/(chat)/layout.tsx — guards the whole chat area
"use client";
import { useEffect } from "react";
import { useRouter } from "next/navigation";
import { authStore } from "@/lib/auth-store";

export default function ChatLayout({ children }: { children: React.ReactNode }) {
  const router = useRouter();
  useEffect(() => {
    if (!authStore.get()) router.replace("/login"); // memory empty after refresh failed
  }, [router]);
  return <div className="chat-shell">{children}</div>;
}
```

### 16.4 Login & register pages **[B/I]**

The login form posts credentials, stores the returned access token in memory, and navigates to the chat. The refresh cookie is set by the server's `Set-Cookie` — the client never touches it.

```tsx
// web/app/login/page.tsx (excerpt)
"use client";
import { useState } from "react";
import { useRouter } from "next/navigation";
import { authStore } from "@/lib/auth-store";

export default function LoginPage() {
  const router = useRouter();
  const [err, setErr] = useState<string | null>(null);

  async function onSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const form = new FormData(e.currentTarget);
    const res = await fetch("/api/auth/login", {
      method: "POST",
      credentials: "include", // so Set-Cookie (refresh) is stored
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email: form.get("email"), password: form.get("password") }),
    });
    if (!res.ok) { setErr("Invalid email or password"); return; }
    const { accessToken } = await res.json();
    authStore.set(accessToken);
    router.replace("/");
  }

  return (
    <form onSubmit={onSubmit}>
      <input name="email" type="email" autoComplete="email" required />
      <input name="password" type="password" autoComplete="current-password" required />
      {err && <p role="alert">{err}</p>}
      <button type="submit">Sign in</button>
    </form>
  );
}
```

> **Security note.** Never render the server's raw error text into the DOM without care, and never reveal whether the email or the password was wrong (enumeration). Keep the generic "Invalid email or password." Autocomplete hints (`current-password`) help password managers; they don't weaken security.

---

## 17. Frontend REST Data Layer with TanStack Query

TanStack Query v5 is the **cache that sits between the UI and the REST API**, and — as promised in [§1.4](#1-what-were-building--why-this-stack) — it is also the cache the **WebSocket writes into** ([§18](#18-the-frontend-websocket-client)). Getting the query keys and the infinite-history query right here is what makes the realtime integration clean later. See the [TanStack Query guide](TANSTACK_QUERY_GUIDE.md) for the library in depth.

### 17.1 Query keys — the addressing scheme **[I]**

Every cached thing needs a stable, structured key. We centralize them so the WS handler and the components refer to the *same* keys (a typo here silently breaks realtime updates because the WS would patch a different cache entry than the UI reads).

```ts
// web/lib/query-keys.ts
export const qk = {
  me: ["me"] as const,
  conversations: ["conversations"] as const,           // sidebar list + unread
  messages: (roomId: string) => ["messages", roomId] as const, // infinite history
  room: (roomId: string) => ["room", roomId] as const,          // members, meta
};
```

### 17.2 The wire types — mirror the Go DTOs **[I]**

These TypeScript interfaces are the *exact* mirror of the Go DTOs from [§9.2](#9-the-rest-api-with-gin) and the WS payloads from [§11](#11-the-realtime-protocol--envelope--events). Keeping them in one file makes the contract auditable; [§24](#24-maintainability--contracts--keeping-tools-in-sync) discusses generating them.

```ts
// web/lib/types.ts
export interface Message {
  id: string;
  roomId: string;
  senderId: string;
  body: string;
  clientMsgId: string;
  createdAt: string; // ISO-8601
}
export interface Conversation {
  roomId: string;
  kind: "channel" | "dm";
  name: string | null;
  lastBody: string | null;
  lastAt: string | null;
  unread: number;
}
export interface HistoryPage {
  messages: Message[];
  nextCursor: string | null;
}
// The realtime envelope — identical shape to the Go Envelope (§11.1).
export interface Envelope<P = unknown> {
  v: number;
  type: string;
  room_id?: string;
  client_msg_id?: string;
  payload?: P;
}
```

### 17.3 Conversations (sidebar) query **[I]**

A plain query backs the sidebar. `unread` counts arrive in this snapshot and are then adjusted live by the WS handler ([§18.6](#18-the-frontend-websocket-client)).

```ts
// web/lib/queries.ts
import { useQuery } from "@tanstack/react-query";
import { apiFetch } from "./api";
import { qk } from "./query-keys";
import type { Conversation } from "./types";

export function useConversations() {
  return useQuery({
    queryKey: qk.conversations,
    queryFn: () => apiFetch<Conversation[]>("/api/conversations"),
  });
}
```

### 17.4 Message history as an infinite keyset query **[I/A]**

History is the showcase for `useInfiniteQuery`: each page is a keyset slice from [§9.3](#9-the-rest-api-with-gin), and `getNextPageParam` reads the server's opaque `nextCursor`. Because history is **newest-first**, we render the page arrays reversed at display time (oldest at top, newest at bottom) and "load older" fetches the *next* page.

```ts
// web/lib/queries.ts (continued)
import { useInfiniteQuery } from "@tanstack/react-query";
import type { HistoryPage } from "./types";

export function useMessages(roomId: string) {
  return useInfiniteQuery({
    queryKey: qk.messages(roomId),
    initialPageParam: null as string | null,
    queryFn: ({ pageParam }) => {
      const q = new URLSearchParams({ limit: "50" });
      if (pageParam) q.set("cursor", pageParam);
      return apiFetch<HistoryPage>(`/api/rooms/${roomId}/messages?${q}`);
    },
    // Return the cursor for the NEXT (older) page, or undefined to stop.
    getNextPageParam: (last) => last.nextCursor ?? undefined,
    staleTime: Infinity, // history doesn't go stale; live edits come via WS, not refetch
  });
}
```

Two decisions that pay off in [§18](#18-the-frontend-websocket-client): `staleTime: Infinity` means TanStack Query won't background-refetch history (which would fight the WS updates); and the cache shape is `{ pages: HistoryPage[], pageParams }`, which the WS handler patches by pushing a new message into the **first** page (the newest one). We flatten and reverse for rendering:

```ts
// A selector the message list uses to get a flat, oldest→newest array.
export function flattenMessages(data?: { pages: HistoryPage[] }): Message[] {
  if (!data) return [];
  // pages are newest-first; concat then reverse to oldest→newest for display.
  return data.pages.flatMap((p) => p.messages).slice().reverse();
}
```

> **Best practice — one cache, never two.** It is tempting to keep messages in React state and use TanStack Query only for the initial load. Don't. Keep the messages *in the query cache* and have both REST and WS write there. Then the message list is a pure function of the cache, optimistic sends and server confirmations reconcile in one place, and reconnection refetch just repopulates the same structure. Splitting state across `useState` and the query cache is the root cause of "the message shows twice / disappears on reload" bugs.

---

## 18. The Frontend WebSocket Client

This is where realtime comes alive on the client: open one WebSocket, authenticate it with the access token via a **subprotocol**, reconnect with **exponential backoff**, and — the crux — **write every incoming event into the TanStack Query cache** so the UI (which reads only from the cache, [§17.4](#17-frontend-rest-data-layer-with-tanstack-query)) updates automatically. We also implement **optimistic send** with `client_msg_id` idempotency and rollback. Pair with [§10](#10-realtime-with-coderwebsocket--the-hub-architecture) (the server side) and the [Coder WebSocket guide](GO_CODER_WEBSOCKETS_GUIDE.md).

### 18.1 Opening the socket with the JWT subprotocol **[A]**

The browser `WebSocket` constructor's second argument is the list of subprotocols — the *only* way a browser can pass data on the handshake (it cannot set headers). We send `["chat.v1", "bearer.<accessToken>"]`, matching what the server parses in [§10.3](#10-realtime-with-coderwebsocket--the-hub-architecture). The server must echo `chat.v1` or the browser rejects the connection.

```ts
// web/lib/ws.ts
import { authStore } from "./auth-store";

export function openSocket(): WebSocket {
  const token = authStore.get();
  const scheme = location.protocol === "https:" ? "wss" : "ws";
  // Same origin (§15.6): wss://host/ws. Subprotocols carry the marker + bearer token.
  return new WebSocket(`${scheme}://${location.host}/ws`, ["chat.v1", `bearer.${token}`]);
}
```

> **Gotcha — the token is in the subprotocol, and subprotocols are visible.** The `Sec-WebSocket-Protocol` header is not secret from the network layer, but under **wss** (TLS) it is encrypted in transit like any header. Never send it over plain `ws`. Because the access token is short-lived (15 min) and refreshed in memory, exposure risk is bounded; still, always use `wss` in production ([§21.1](#21-banking-grade-security-hardening)).

### 18.2 The connection manager with reconnect & resubscribe **[A]**

We wrap the socket in a manager that owns the lifecycle: connect, dispatch incoming envelopes to registered handlers, send outgoing envelopes (queuing if not yet open), and **reconnect with exponential backoff + jitter** on unexpected close. On every (re)connect it triggers a **history/conversations refetch** to fill any gap of messages missed while disconnected — the socket is a freshness optimization, Postgres is truth ([§12.4](#12-the-message-send-flow-end-to-end)).

```ts
// web/lib/ws-manager.ts
import { openSocket } from "./ws";
import type { Envelope } from "./types";

type Handler = (env: Envelope) => void;

export class WSManager {
  private ws: WebSocket | null = null;
  private handlers = new Set<Handler>();
  private queue: string[] = [];
  private retries = 0;
  private closedByUser = false;
  private onReconnect?: () => void; // e.g. refetch history/conversations

  constructor(onReconnect?: () => void) { this.onReconnect = onReconnect; }

  connect() {
    this.closedByUser = false;
    const ws = openSocket();
    this.ws = ws;

    ws.onopen = () => {
      this.retries = 0;
      // Flush anything queued while we were offline.
      for (const m of this.queue) ws.send(m);
      this.queue = [];
      this.onReconnect?.(); // refill the gap: refetch on (re)connect
    };
    ws.onmessage = (e) => {
      let env: Envelope;
      try { env = JSON.parse(e.data); } catch { return; }
      for (const h of this.handlers) h(env);
    };
    ws.onclose = (e) => {
      // 1000 normal / 1001 going-away (deploy) → reconnect quietly. Others too, backed off.
      if (this.closedByUser) return;
      this.scheduleReconnect();
    };
    ws.onerror = () => { /* onclose will follow; let it handle reconnect */ };
  }

  private scheduleReconnect() {
    this.retries++;
    // Exponential backoff capped at 15s, with jitter to avoid thundering herds
    // when a whole node's clients reconnect at once after a deploy.
    const base = Math.min(15_000, 500 * 2 ** this.retries);
    const delay = base / 2 + Math.random() * (base / 2);
    setTimeout(() => this.connect(), delay);
  }

  send(env: Envelope) {
    const data = JSON.stringify(env);
    if (this.ws?.readyState === WebSocket.OPEN) this.ws.send(data);
    else this.queue.push(data); // buffered until reconnect
  }

  subscribe(h: Handler) { this.handlers.add(h); return () => this.handlers.delete(h); }

  close() { this.closedByUser = true; this.ws?.close(1000, "bye"); }
}
```

### 18.3 A React provider that owns the single socket **[A]**

One socket per tab, created once, shared via context. On reconnect we invalidate the conversations and the active room's history so TanStack Query refetches the authoritative state.

```tsx
// web/lib/ws-provider.tsx
"use client";
import { createContext, useContext, useEffect, useRef } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { WSManager } from "./ws-manager";
import { applyEnvelope } from "./ws-apply";
import { qk } from "./query-keys";

const Ctx = createContext<WSManager | null>(null);
export const useWS = () => useContext(Ctx)!;

export function WSProvider({ children }: { children: React.ReactNode }) {
  const qc = useQueryClient();
  const ref = useRef<WSManager | null>(null);

  if (!ref.current) {
    ref.current = new WSManager(() => {
      // On (re)connect: refetch the durable truth to fill any missed-message gap.
      qc.invalidateQueries({ queryKey: qk.conversations });
      qc.invalidateQueries({ queryKey: ["messages"] });
    });
  }

  useEffect(() => {
    const mgr = ref.current!;
    const unsub = mgr.subscribe((env) => applyEnvelope(qc, env)); // WS → cache
    mgr.connect();
    return () => { unsub(); mgr.close(); };
  }, [qc]);

  return <Ctx.Provider value={ref.current}>{children}</Ctx.Provider>;
}
```

### 18.4 Reconnection & the missed-message gap **[A]**

While the socket is down (network blip, node deploy sending `1001`), the client misses `message.new` frames. Rather than build a fragile replay protocol, we rely on the durable store: **on every reconnect, refetch.** `invalidateQueries` marks history/conversations stale and TanStack Query refetches the newest page, which contains any messages that arrived during the gap. Combined with the dedupe-by-id in the cache patcher ([§18.5](#18-the-frontend-websocket-client)), the result is correct regardless of how many frames were missed. This is the client half of "the WebSocket is a freshness optimization, not the source of truth" from [§12.4](#12-the-message-send-flow-end-to-end).

### 18.5 Applying events to the cache (the heart of it) **[A]**

`applyEnvelope` is the single function that turns a WS event into a cache mutation. It is **idempotent** — applying the same `message.new` twice (e.g., the echo of your own send plus a reconnect refetch) must not duplicate — which we get by **deduping on message `id`** and reconciling optimistic bubbles by `client_msg_id`.

```ts
// web/lib/ws-apply.ts
import type { QueryClient } from "@tanstack/react-query";
import { qk } from "./query-keys";
import type { Envelope, HistoryPage, Message, Conversation } from "./types";

export function applyEnvelope(qc: QueryClient, env: Envelope) {
  switch (env.type) {
    case "message.new":   return onMessageNew(qc, env);
    case "message.ack":   return onMessageAck(qc, env);
    case "message.error": return onMessageError(qc, env);
    case "presence.update": return onPresence(qc, env);
    case "typing.start":  return onTyping(qc, env, true);
    case "typing.stop":   return onTyping(qc, env, false);
    case "receipt.read":  return onReceipt(qc, env);
  }
}

function onMessageNew(qc: QueryClient, env: Envelope) {
  const msg = env.payload as Message;
  const key = qk.messages(msg.roomId);
  qc.setQueryData<{ pages: HistoryPage[]; pageParams: unknown[] }>(key, (old) => {
    if (!old) return old; // room not open/loaded — the badge path handles it below
    const first = old.pages[0];
    // Dedupe: if this id OR this clientMsgId already exists, replace in place (idempotent).
    const exists = first.messages.some(
      (m) => m.id === msg.id || (m.clientMsgId && m.clientMsgId === msg.clientMsgId),
    );
    const messages = exists
      ? first.messages.map((m) =>
          m.clientMsgId === msg.clientMsgId || m.id === msg.id ? { ...msg } : m,
        )
      : [msg, ...first.messages]; // newest-first: prepend to page 0
    return { ...old, pages: [{ ...first, messages }, ...old.pages.slice(1)] };
  });
  bumpConversation(qc, msg); // update sidebar preview + unread (§18.6)
}

function onMessageAck(qc: QueryClient, env: Envelope) {
  const { id, created_at } = env.payload as { id: string; created_at: string };
  const cmid = env.client_msg_id!;
  const key = qk.messages(env.room_id!);
  // Promote the optimistic bubble (status "sending") to confirmed with the server id/time.
  qc.setQueryData<{ pages: HistoryPage[] }>(key, (old) => {
    if (!old) return old;
    const pages = old.pages.map((p) => ({
      ...p,
      messages: p.messages.map((m) =>
        m.clientMsgId === cmid ? { ...m, id, createdAt: created_at, status: "sent" } : m,
      ),
    }));
    return { ...old, pages };
  });
}

function onMessageError(qc: QueryClient, env: Envelope) {
  const cmid = env.client_msg_id!;
  const key = qk.messages(env.room_id!);
  // Rollback: mark the optimistic bubble failed so the UI shows a retry affordance.
  qc.setQueryData<{ pages: HistoryPage[] }>(key, (old) => {
    if (!old) return old;
    const pages = old.pages.map((p) => ({
      ...p,
      messages: p.messages.map((m) => (m.clientMsgId === cmid ? { ...m, status: "failed" } : m)),
    }));
    return { ...old, pages };
  });
}
```

### 18.6 Optimistic send, unread bumps, presence & typing in the cache **[A]**

Sending is optimistic: we generate a `client_msg_id` (via `crypto.randomUUID()`), insert a bubble with `status:"sending"` into the cache *immediately*, then send the `message.send` frame. The server's `message.ack` promotes it; a `message.error` marks it failed. This is the client half of the exactly-once story from [§12.4](#12-the-message-send-flow-end-to-end): the same `client_msg_id` is reused if the user hits retry, so the server dedupes.

```ts
// web/lib/send.ts
import type { QueryClient } from "@tanstack/react-query";
import { qk } from "./query-keys";
import type { WSManager } from "./ws-manager";

export function sendMessage(qc: QueryClient, ws: WSManager, roomId: string, body: string, meId: string) {
  const clientMsgId = crypto.randomUUID(); // ONE id, reused across retries → idempotency
  const optimistic = {
    id: `tmp-${clientMsgId}`, roomId, senderId: meId, body, clientMsgId,
    createdAt: new Date().toISOString(), status: "sending" as const,
  };
  // 1) Optimistically place the bubble at the bottom (page 0, newest-first).
  qc.setQueryData<{ pages: any[] }>(qk.messages(roomId), (old) =>
    old ? { ...old, pages: [{ ...old.pages[0], messages: [optimistic, ...old.pages[0].messages] }, ...old.pages.slice(1)] } : old,
  );
  // 2) Fire the frame. If offline, WSManager queues it until reconnect.
  ws.send({ v: 1, type: "message.send", room_id: roomId, client_msg_id: clientMsgId, payload: { body } });
}
```

Unread bumps, presence, and typing patch small, separate slices of the cache. Unread increments only when the message's room is **not** the currently-open one:

```ts
// web/lib/ws-apply.ts (continued)
function bumpConversation(qc: QueryClient, msg: Message) {
  qc.setQueryData<Conversation[]>(qk.conversations, (old) =>
    (old ?? []).map((c) =>
      c.roomId === msg.roomId
        ? { ...c, lastBody: msg.body, lastAt: msg.createdAt,
            unread: isRoomOpen(msg.roomId) ? 0 : c.unread + 1 }
        : c,
    ),
  );
}

function onPresence(qc: QueryClient, env: Envelope) {
  const { user_id, status } = env.payload as { user_id: string; status: string };
  qc.setQueryData(["presence"], (m: Record<string, boolean> = {}) => ({ ...m, [user_id]: status === "online" }));
}

// Typing lives in ephemeral React state keyed by room (a small store), NOT the query
// cache, because it self-expires; see §19.5 for the auto-clear timer.
function onTyping(qc: QueryClient, env: Envelope, start: boolean) {
  const { user_id } = env.payload as { user_id: string };
  typingBus.emit(env.room_id!, user_id, start);
}

function onReceipt(qc: QueryClient, env: Envelope) {
  const { user_id, message_id } = env.payload as { user_id: string; message_id: string };
  qc.setQueryData(["receipts", env.room_id], (m: Record<string, string> = {}) => ({ ...m, [user_id]: message_id }));
  // If the receipt is for MY OWN user (multi-device), clear this room's unread (§14.4).
  if (user_id === currentUserId()) {
    qc.setQueryData<Conversation[]>(qk.conversations, (old) =>
      (old ?? []).map((c) => (c.roomId === env.room_id ? { ...c, unread: 0 } : c)));
  }
}
```

> **Best practice — the cache is the single source of truth for the UI.** Every event type resolves to a `setQueryData`/store update; components never read from the socket directly. This is what makes the app correct under reconnection, optimistic sends, and multi-tab: there is exactly one place state lives, and both REST and WS converge on it. If you find yourself keeping messages in `useState` next to the query cache, stop — that duplication is the bug.

---
## 19. The Chat UI with React 19

With the cache-driven data layer in place ([§17](#17-frontend-rest-data-layer-with-tanstack-query)–[§18](#18-the-frontend-websocket-client)), the UI is almost entirely a **pure function of the cache**. This section wires the visible pieces: the room list with unread badges, the paginated message list, the composer with typing, presence dots, and read receipts — with **XSS-safe rendering** throughout. See [React 19](REACT_19_GUIDE.md) for hooks specifics.

### 19.1 The layout & the open-room store **[I]**

The shell is a sidebar (conversations) plus a main pane (the open room). "Which room is open" is UI navigation state — we keep it in the URL (`/rooms/[id]`) so it's linkable and survives reload, and a tiny module exposes `isRoomOpen(id)` used by the unread-bump logic ([§18.6](#18-the-frontend-websocket-client)).

```tsx
// web/app/(chat)/rooms/[id]/page.tsx
"use client";
import { useParams } from "next/navigation";
import { RoomView } from "@/components/RoomView";
import { setOpenRoom } from "@/lib/open-room";
import { useEffect } from "react";

export default function RoomPage() {
  const { id } = useParams<{ id: string }>();
  useEffect(() => { setOpenRoom(id); return () => setOpenRoom(null); }, [id]);
  return <RoomView roomId={id} />;
}
```

### 19.2 The room list with unread badges **[I]**

The sidebar reads `useConversations()` ([§17.3](#17-frontend-rest-data-layer-with-tanstack-query)); each item shows the room name (or the other user's name for a DM), a last-message preview, and an unread badge driven entirely by the cache value the WS handler maintains ([§18.6](#18-the-frontend-websocket-client)). No socket code here — just cache reads.

```tsx
// web/components/Sidebar.tsx
"use client";
import Link from "next/link";
import { useConversations } from "@/lib/queries";

export function Sidebar() {
  const { data: convos = [] } = useConversations();
  return (
    <nav className="sidebar">
      {convos.map((c) => (
        <Link key={c.roomId} href={`/rooms/${c.roomId}`} className="convo">
          {/* text content is auto-escaped by React — never dangerouslySetInnerHTML here */}
          <span className="name">{c.name ?? "Direct message"}</span>
          <span className="preview">{c.lastBody ?? "No messages yet"}</span>
          {c.unread > 0 && <span className="badge" aria-label={`${c.unread} unread`}>{c.unread}</span>}
        </Link>
      ))}
    </nav>
  );
}
```

### 19.3 The message list — paginated, ordered, virtualized **[I/A]**

The list reads the infinite query ([§17.4](#17-frontend-rest-data-layer-with-tanstack-query)), flattens to oldest→newest, and renders. Two realtime-specific behaviors: **"load older"** at the top calls `fetchNextPage` (keyset), and **auto-scroll to bottom** on a new message *only if the user is already near the bottom* (so we don't yank them away while they read history). For large rooms, virtualize with a windowing library so only visible rows mount — but keep the data in the cache, not the virtualizer.

```tsx
// web/components/MessageList.tsx
"use client";
import { useEffect, useRef } from "react";
import { useMessages, flattenMessages } from "@/lib/queries";
import { MessageBubble } from "./MessageBubble";

export function MessageList({ roomId }: { roomId: string }) {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useMessages(roomId);
  const messages = flattenMessages(data);
  const bottomRef = useRef<HTMLDivElement>(null);
  const nearBottom = useRef(true);

  // Auto-scroll to newest ONLY when the user is near the bottom already.
  useEffect(() => {
    if (nearBottom.current) bottomRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages.length]);

  return (
    <div
      className="messages"
      onScroll={(e) => {
        const el = e.currentTarget;
        nearBottom.current = el.scrollHeight - el.scrollTop - el.clientHeight < 120;
        if (el.scrollTop < 80 && hasNextPage && !isFetchingNextPage) fetchNextPage();
      }}
    >
      {hasNextPage && <button onClick={() => fetchNextPage()}>Load older</button>}
      {messages.map((m) => <MessageBubble key={m.clientMsgId || m.id} m={m} />)}
      <div ref={bottomRef} />
    </div>
  );
}
```

### 19.4 XSS-safe message rendering **[A]**

Chat bodies are **untrusted user input** — the highest-risk XSS surface in the whole app, because a message from one user is rendered in every other user's browser. The rule: **render `body` as text, never as HTML.** React escapes text children by default, so `{m.body}` is safe. **Never** pass user content to `dangerouslySetInnerHTML`. If you want clickable links or markdown, do **not** hand-roll it: parse to a safe AST and render *elements*, or sanitize the resulting HTML with a vetted allowlist sanitizer — but the default and correct choice for a chat app is plain escaped text with linkification done by constructing `<a>` elements from matched URL spans, never by injecting HTML strings.

```tsx
// web/components/MessageBubble.tsx
"use client";
import type { Message } from "@/lib/types";

export function MessageBubble({ m }: { m: Message & { status?: string } }) {
  return (
    <div className={`bubble ${m.status ?? "sent"}`} data-mid={m.id}>
      {/* {m.body} is auto-escaped by React. This is the XSS-safe default. */}
      <p className="body">{m.body}</p>
      <time dateTime={m.createdAt}>{formatTime(m.createdAt)}</time>
      {m.status === "sending" && <span className="tick">…</span>}
      {m.status === "sent" && <span className="tick">✓</span>}
      {m.status === "failed" && <button className="retry">retry</button>}
    </div>
  );
}
```

> **Security note — the server validates too.** Client-side escaping protects *this* renderer, but another client (or a malicious API caller) is the real threat. The server caps size and rejects blank bodies ([§12.3](#12-the-message-send-flow-end-to-end)); it does **not** need to strip HTML because we never render bodies as HTML anywhere. If you ever add server-rendered emails or a non-React client, revisit this — the safety currently depends on React's text escaping. See [§21.6](#21-banking-grade-security-hardening).

### 19.5 The composer, typing, presence & receipts **[I/A]**

The composer sends optimistically ([§18.6](#18-the-frontend-websocket-client)) and emits throttled `typing.start` while the user types. Typing indicators from others live in an ephemeral store with an **auto-clear timer** so a lost `typing.stop` can't leave a stuck "typing…". Presence dots read the `["presence"]` cache slice; receipts read `["receipts", roomId]`.

```tsx
// web/components/Composer.tsx
"use client";
import { useRef } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { useWS } from "@/lib/ws-provider";
import { sendMessage } from "@/lib/send";
import { currentUserId } from "@/lib/me";

export function Composer({ roomId }: { roomId: string }) {
  const qc = useQueryClient();
  const ws = useWS();
  const lastTyping = useRef(0);

  function onInput() {
    const now = Date.now();
    if (now - lastTyping.current > 2000) { // throttle: ≤1 typing.start per 2s
      ws.send({ v: 1, type: "typing.start", room_id: roomId });
      lastTyping.current = now;
    }
  }
  function onSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const input = e.currentTarget.elements.namedItem("body") as HTMLInputElement;
    const body = input.value.trim();
    if (!body) return;
    sendMessage(qc, ws, roomId, body, currentUserId());
    ws.send({ v: 1, type: "typing.stop", room_id: roomId });
    input.value = "";
  }
  return (
    <form onSubmit={onSubmit} className="composer">
      <input name="body" autoComplete="off" onInput={onInput} placeholder="Message" />
      <button type="submit">Send</button>
    </form>
  );
}
```

```tsx
// web/components/TypingIndicator.tsx — auto-expiring typing state
"use client";
import { useEffect, useState } from "react";
import { typingBus } from "@/lib/typing-bus";

export function TypingIndicator({ roomId }: { roomId: string }) {
  const [typers, setTypers] = useState<Record<string, number>>({}); // userId → expiry ts
  useEffect(() => {
    return typingBus.on(roomId, (userId, start) => {
      setTypers((t) => {
        if (!start) { const { [userId]: _, ...rest } = t; return rest; }
        return { ...t, [userId]: Date.now() + 4000 }; // auto-clear 4s after last signal
      });
    });
  }, [roomId]);
  // Sweep expired entries so a dropped typing.stop can't stick.
  useEffect(() => {
    const iv = setInterval(() => setTypers((t) => {
      const now = Date.now();
      return Object.fromEntries(Object.entries(t).filter(([, exp]) => exp > now));
    }), 1000);
    return () => clearInterval(iv);
  }, []);
  const names = Object.keys(typers);
  if (names.length === 0) return null;
  return <div className="typing">{names.length === 1 ? "typing…" : `${names.length} typing…`}</div>;
}
```

Read receipts render as a "Seen" marker under the newest message a peer's pointer has reached. When the local user scrolls to the bottom (the newest message becomes visible), the room view fires a `receipt.read` ([§14.2](#14-read-receipts--unread-counts)) and resets the unread badge.

```tsx
// Inside RoomView: mark read when the newest message is visible.
useEffect(() => {
  if (newest && nearBottom.current) {
    ws.send({ v: 1, type: "receipt.read", room_id: roomId, payload: { message_id: newest.id } });
  }
}, [newest?.id]);
```

---

## 20. Frontend RBAC — Mirroring the Server

The frontend mirrors the server's room roles ([§8](#8-rbac--authorization--room-roles-and-idor-defense)) to produce good UX — hiding buttons a user can't use, guarding routes — while never being the security boundary. This section is short by design, because **the important half of RBAC is on the server** and we already built it. The client's job is cosmetic.

### 20.1 The golden rule, once more **[I]**

> **The client is never the security boundary.** Every rule that matters — who can read a room, post, delete a message, remove a member — is enforced by the Go service ([§8](#8-rbac--authorization--room-roles-and-idor-defense), [§12.3](#12-the-message-send-flow-end-to-end)). The frontend hides the "Delete room" button for non-owners *only so the UI isn't confusing*. A determined user can unhide it, call the API directly, or craft a WS frame — and the server will still return 403. If you ever catch yourself thinking "the button is hidden, so I don't need to check on the server," stop: that is exactly the IDOR/privilege-escalation hole attackers look for.

### 20.2 Deriving the caller's role and gating UI **[I]**

The room detail query ([§17.1](#17-frontend-rest-data-layer-with-tanstack-query) `qk.room`) includes the caller's membership role. A small `useRole` hook plus a `Can` helper mirror the server's permission table so components can conditionally render.

```ts
// web/lib/rbac.ts — MIRRORS the server table in §8.1 (UX only, not enforcement)
export type Role = "owner" | "admin" | "member";
export const Can = {
  manageMembers: (r: Role) => r === "owner" || r === "admin",
  changeRole:    (r: Role) => r === "owner" || r === "admin",
  renameRoom:    (r: Role) => r === "owner" || r === "admin",
  deleteRoom:    (r: Role) => r === "owner",
  moderate:      (r: Role) => r === "owner" || r === "admin", // delete any message
};
```

```tsx
// web/components/RoomHeader.tsx (excerpt)
import { Can } from "@/lib/rbac";
export function RoomHeader({ room, myRole }: { room: Room; myRole: Role }) {
  return (
    <header>
      <h1>{room.name ?? "Direct message"}</h1>
      {Can.manageMembers(myRole) && <button>Manage members</button>}
      {Can.deleteRoom(myRole)   && <button className="danger">Delete room</button>}
    </header>
  );
}
```

### 20.3 Route guards & the failing-closed default **[I]**

Route-level guards (the chat layout in [§16.3](#16-nextjs-frontend--setup--auth)) redirect the unauthenticated. For role-gated *views* (e.g., a room-settings page), the component checks the role and renders a "not allowed" state if it's insufficient — but again, the settings *mutations* are also blocked server-side. Default to **failing closed**: if the role is unknown/loading, render nothing rather than the privileged UI, so a slow role fetch never briefly flashes an admin button to a member.

> **Best practice — keep the two tables in sync deliberately.** `web/lib/rbac.ts` (client) and `internal/rbac` (server) encode the same permission matrix. When you change one, change the other in the same PR, and let the server be the one that's *tested* ([§22](#22-observability--testing)). Drift here is only a UX bug (a button shows that shouldn't), never a security bug, because the server enforces regardless — but it's still worth keeping tidy. [§24](#24-maintainability--contracts--keeping-tools-in-sync) covers contract-sync discipline.

---

## 21. Banking-Grade Security Hardening

Chat apps are a high-value target: they carry private conversations, they're multi-tenant (every user's data in one database), and they expose a long-lived, attacker-reachable socket. This section consolidates the security posture — much of it built inline earlier — into a checklist you can audit against. Nothing here is optional for production. Cross-reference the [JWT + Argon2 guide](GO_JWT_ARGON2_GUIDE.md) for the auth threat models in full.

### 21.1 TLS / wss everywhere **[A]**

All traffic — REST, the WebSocket, and the database/Redis connections — must be encrypted in transit. Terminate TLS at Nginx ([§15.6](#15-scaling-out--the-redis-pubsub-backplane--nginx)) and serve `https`/`wss` only; redirect `http`→`https`. The access token rides in the WS subprotocol, so plain `ws` would expose it — never allow it. Postgres and Redis connections use TLS (`sslmode=require` for pgx; `rediss://`) when they cross a network boundary. HSTS (`Strict-Transport-Security`) forces browsers to `https` and defends against SSL-strip.

### 21.2 CSWSH & Origin enforcement **[A]**

`coder/websocket`'s `Accept` blocks cross-origin handshakes unless you set `OriginPatterns` ([§10.4](#10-realtime-with-coderwebsocket--the-hub-architecture)); we set it to our exact web origins — never `["*"]`. Because our WS auth is a token in the subprotocol rather than an ambient cookie, a hijacking page can't silently ride an existing session, but the Origin allowlist stays as defense in depth. The REST side uses a matching CORS allowlist (or, with one origin, no CORS at all).

### 21.3 Authentication on upgrade + per-message authorization **[A]**

Two independent gates, both mandatory. **On upgrade** we verify the JWT *before* calling `Accept` and reject unauthenticated handshakes ([§10.3](#10-realtime-with-coderwebsocket--the-hub-architecture)). **Per message** we re-check membership on *every* `message.send`/`typing`/`receipt` ([§12.3](#12-the-message-send-flow-end-to-end), [§13.2](#13-presence--typing-indicators)) — because a connection authenticated as user U must still be prevented from posting to a room U was removed from mid-session. Never assume "authenticated once" means "authorized for everything." The connection's subscribed-rooms set is refreshed on membership changes, and the DB constraints are the final backstop.

### 21.4 IDOR & row-level ownership **[A]**

The cardinal chat vulnerability: reading or posting to a room you don't belong to by supplying its id. We defend in **three** layers: the handler membership check ([§8.2](#8-rbac--authorization--room-roles-and-idor-defense)), the in-query `EXISTS` membership guard so even a query with a foreign room id returns nothing, and the realtime `inRoom` check before any broadcast. Return **404 (not 403)** to non-members so you don't confirm a room's existence ([§8.1](#8-rbac--authorization--room-roles-and-idor-defense)). Message deletes check `sender_id = caller` OR moderator role; never trust an id from the client without an ownership predicate.

### 21.5 Input validation, message size & rate limiting **[A]**

Every inbound value is validated at the boundary. Transport-level: `SetReadLimit` caps WS frame size ([§10.5](#10-realtime-with-coderwebsocket--the-hub-architecture)); business-level: bodies are trimmed, non-empty, and ≤ 8 KiB ([§12.3](#12-the-message-send-flow-end-to-end)), enforced *both* in the handler and by a Postgres `CHECK` ([§4.5](#4-the-database-schema-via-goose-migrations)). Rate limits apply per user/connection at several granularities:

- **Login/register** — a few attempts per minute per IP+account (credential-stuffing/enumeration defense; [§9.1](#9-the-rest-api-with-gin)).
- **Message send** — e.g. ≤ 5/second/connection with a small burst, enforced in the WS dispatch with a token bucket; excess earns a `message.error{code:"RATE_LIMITED"}`.
- **Connection count** — cap concurrent sockets per user (e.g. 10) so one account can't open thousands and exhaust memory ([§21.9](#21-banking-grade-security-hardening)).

```go
// A token-bucket rate limit inside the WS dispatch (per-connection).
if !c.limiter.Allow() { // golang.org/x/time/rate, e.g. rate.NewLimiter(5, 10)
	c.sendError(env.ClientMsgID, "RATE_LIMITED", "slow down")
	return
}
```

### 21.6 XSS **[A]**

Message bodies are rendered as escaped **text** by React, never as HTML; we never use `dangerouslySetInnerHTML` on user content ([§19.4](#19-the-chat-ui-with-react-19)). This is the primary XSS defense, and it's structural (the framework escapes) rather than a fragile sanitizer. A CSP ([§21.8](#21-banking-grade-security-hardening)) is the second layer: even if an injection slipped through, a strict `script-src` blocks execution.

### 21.7 CSRF for the cookie-based refresh **[A]**

The refresh token is an HttpOnly cookie, which the browser sends automatically — so the refresh endpoint is CSRF-relevant. Three mitigations, layered: (1) `SameSite=Strict` (or `Lax`) on the cookie so a cross-site page can't trigger an authenticated refresh in the first place — trivial with the one-origin topology ([§15.6](#15-scaling-out--the-redis-pubsub-backplane--nginx)); (2) the refresh endpoint accepts only `POST` and checks `Origin`/`Referer`; (3) refresh **rotates** the token, so even a successful CSRF gains a token the legitimate client's next refresh will invalidate via reuse detection ([§7.4](#7-authentication--argon2id-jwt-access--refresh-rotation)). Access-token-bearing requests aren't CSRF-able because the token is in a header (memory), not a cookie.

### 21.8 CSP & security headers **[A]**

Every response carries a hardened header set (middleware in [§9.1](#9-the-rest-api-with-gin), and Next.js config for the app shell):

```go
// internal/rest/security_headers.go
func SecurityHeaders() gin.HandlerFunc {
	return func(c *gin.Context) {
		h := c.Writer.Header()
		h.Set("Content-Security-Policy",
			"default-src 'self'; connect-src 'self' wss://chat.example.com; "+
				"img-src 'self' data:; script-src 'self'; style-src 'self' 'unsafe-inline'; "+
				"frame-ancestors 'none'; base-uri 'self'")
		h.Set("X-Content-Type-Options", "nosniff")
		h.Set("X-Frame-Options", "DENY")             // clickjacking
		h.Set("Referrer-Policy", "strict-origin-when-cross-origin")
		h.Set("Strict-Transport-Security", "max-age=63072000; includeSubDomains; preload")
		c.Next()
	}
}
```

Note `connect-src` must include your `wss://` origin or the browser blocks the WebSocket. `frame-ancestors 'none'` and `X-Frame-Options: DENY` prevent your chat being embedded in a hostile iframe (clickjacking/UI-redress).

### 21.9 Secrets, encryption at rest, PII & audit logging **[A]**

- **Secrets** (`JWT_SECRET`, DB/Redis creds) come from a secrets manager, never source control or images ([§3.2](#3-project-layout-config--graceful-shutdown)); rotate the JWT secret with a key-id (`kid`) grace period if you need zero-downtime rotation.
- **Encryption at rest**: Postgres/Redis volumes are encrypted at the disk/managed-service level. Passwords are Argon2id hashes ([§7.2](#7-authentication--argon2id-jwt-access--refresh-rotation)); refresh tokens are stored only as SHA-256 hashes ([§7.4](#7-authentication--argon2id-jwt-access--refresh-rotation)) — a DB leak yields no usable credentials.
- **PII**: emails and message bodies are personal data. Minimize what you log (never log bodies or tokens), support account deletion (cascades via FKs), and treat `password_hash` as `Sensitive()` in Ent ([§5.2](#5-ent-schema--the-relation-graph)).
- **Audit logging**: record security-relevant events (login success/failure, refresh reuse detected, room-membership changes, message deletes) with actor, target, and timestamp — but *not* message contents. These logs are your incident-response trail.

### 21.10 DoS, max-connections & graceful shutdown **[A]**

The socket is a resource-exhaustion surface. Defenses: per-user connection caps and per-connection send rate limits ([§21.5](#21-banking-grade-security-hardening)); bounded send channels that **evict slow consumers** rather than buffering unboundedly ([§10.5](#10-realtime-with-coderwebsocket--the-hub-architecture)); `ReadHeaderTimeout` against slowloris on the handshake ([§3.3](#3-project-layout-config--graceful-shutdown)); and Nginx-level connection/rate limits at the edge. **Graceful shutdown** ([§10.6](#10-realtime-with-coderwebsocket--the-hub-architecture)) sends `1001 GoingAway` so clients reconnect elsewhere instead of hammering a dying node.

### 21.11 Dependency hygiene & the trust boundary **[A]**

Run `govulncheck` and `npm audit` in CI; pin versions; keep `coder/websocket`, `pgx`, `golang-jwt`, Ent, and Next.js current (the **⚡** flags mark the fast-movers). Above all, hold the **trust boundary** in mind: **the Go service is the only thing the database trusts, and every input from the browser — REST body, query param, WS frame, cookie, subprotocol — is hostile until validated.** The frontend is a convenience layer; delete it and the server must still be secure. Audit against that sentence and most classes of vulnerability have already been designed out.

---
## 22. Observability & Testing

A realtime system fails in ways a request/response app doesn't — a slow-consumer eviction, a backplane hiccup, a reconnect storm — and you can only diagnose what you can see. This section covers structured logging, the metrics that matter for chat, and how to test each layer, including the tricky bits: `Dial`-based WebSocket tests and transaction-rollback DB tests.

### 22.1 Structured logging with slog **[A]**

We use Go's standard `log/slog` with a JSON handler ([§3.3](#3-project-layout-config--graceful-shutdown)) so logs are queryable in aggregation. Every log line carries context — a `request_id` for REST (from middleware) and a `conn_id`/`user_id` for WS. **Never log** message bodies, tokens, or password hashes ([§21.9](#21-banking-grade-security-hardening)); log *events and identifiers*, not *content*.

```go
// A per-connection logger threads user + conn id through the WS lifecycle.
clogger := slog.With("conn_id", connID, "user_id", claims.UserID)
clogger.Info("ws connected", "rooms", len(rooms))
// ... on eviction:
clogger.Warn("slow consumer evicted", "buffered", cap(c.out))
```

### 22.2 Metrics that matter for chat **[A]**

Export Prometheus metrics (or your platform's equivalent) for the signals that predict user-visible pain:

- **`ws_connections` (gauge)** — current live sockets per node; the capacity signal.
- **`ws_messages_total{type}` (counter)** — send/new/ack/error rates; a spike in `message.error` means authz or validation is rejecting traffic.
- **`ws_slow_consumer_evictions_total` (counter)** — non-zero means clients can't keep up (or an attack).
- **`message_persist_seconds` (histogram)** — the DB insert latency on the hot path; the leading indicator of Postgres saturation.
- **`backplane_publish_seconds` / `backplane_lag`** — Redis fan-out health.
- **`http_request_duration_seconds{route,status}`** — standard REST latency/error SLOs.

Wrap the send handler and history query with timers; alert on p99 `message_persist_seconds` and any sustained eviction rate.

### 22.3 Testing the REST layer with httptest **[A]**

REST handlers test cleanly with `net/http/httptest` against the Gin router, using a real (throwaway) Postgres so the sqlc/Ent queries actually run. The pattern: build the router with a test `Store`, issue a request, assert status + body.

```go
// internal/rest/messages_test.go
func TestListMessages_NonMemberGets404(t *testing.T) {
	store := testStore(t)            // spins a temp DB, runs goose migrations
	router := NewRouter(testCfg, store, testHub(t))

	alice := seedUser(t, store, "alice")
	bob := seedUser(t, store, "bob")
	room := seedChannel(t, store, alice) // alice is owner; bob is NOT a member

	req := httptest.NewRequest("GET", "/api/rooms/"+room.ID.String()+"/messages", nil)
	req.Header.Set("Authorization", "Bearer "+accessFor(bob)) // bob is authenticated…
	rec := httptest.NewRecorder()
	router.ServeHTTP(rec, req)

	// …but not a member → the membership guard yields an empty/forbidden result.
	if rec.Code != http.StatusOK || len(decodeHistory(t, rec).Messages) != 0 {
		t.Fatalf("non-member must not see messages; got %d", rec.Code)
	}
}
```

### 22.4 Dial-based WebSocket tests **[A]**

The realtime path is testable end-to-end by starting the real server with `httptest.NewServer` and **dialing** it with `coder/websocket`'s `Dial`. This exercises the actual Accept, subprotocol auth, Hub registration, and broadcast — not mocks. The canonical test: two clients in a room, one sends, the other receives.

```go
// internal/ws/send_test.go
func TestBroadcast_TwoClients(t *testing.T) {
	store := testStore(t)
	hub := NewHub(store, testBackplane(t), testCfg)
	go hub.Run(context.Background())

	srv := httptest.NewServer(NewRouter(testCfg, store, hub))
	defer srv.Close()
	wsURL := "ws" + strings.TrimPrefix(srv.URL, "http") + "/ws"

	alice, bob := seedUser(t, store, "a"), seedUser(t, store, "b")
	room := seedChannelWith(t, store, alice, bob) // both members

	// Dial two authenticated sockets via the bearer subprotocol (§10.3).
	ca := dial(t, wsURL, accessFor(alice))
	cb := dial(t, wsURL, accessFor(bob))
	defer ca.Close(websocket.StatusNormalClosure, "")
	defer cb.Close(websocket.StatusNormalClosure, "")

	cmid := uuid.New()
	// Alice sends message.send.
	wsjson.Write(context.Background(), ca, Envelope{
		V: 1, Type: "message.send", RoomID: &room.ID, ClientMsgID: &cmid,
		Payload: mustJSON(map[string]string{"body": "hi bob"}),
	})

	// Bob must receive message.new with the same body.
	var got Envelope
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()
	if err := wsjson.Read(ctx, cb, &got); err != nil {
		t.Fatal(err)
	}
	if got.Type != "message.new" {
		t.Fatalf("want message.new, got %s", got.Type)
	}
}

func dial(t *testing.T, url, token string) *websocket.Conn {
	c, _, err := websocket.Dial(context.Background(), url, &websocket.DialOptions{
		Subprotocols: []string{"chat.v1", "bearer." + token},
	})
	if err != nil {
		t.Fatal(err)
	}
	return c
}
```

### 22.5 Transaction-rollback DB tests **[A]**

To test a query in isolation without polluting the database, run each test inside a transaction and **roll it back** at the end. Because sqlc's `Queries` accepts any `DBTX`, you hand it a `pgx.Tx` and never commit. This makes DB tests fast and hermetic — no cleanup, perfect isolation, parallelizable.

```go
// internal/db/tx_test.go
func TestInsertMessage_Idempotent(t *testing.T) {
	tx, err := testPool.Begin(context.Background())
	if err != nil { t.Fatal(err) }
	defer tx.Rollback(context.Background()) // ALWAYS rollback — never persists

	q := gen.New(tx) // sqlc bound to the tx
	room, sender := seedRoomTx(t, tx), seedUserTx(t, tx)
	cmid := uuid.New()

	m1, _ := q.InsertMessage(ctx, gen.InsertMessageParams{RoomID: room, SenderID: sender, Body: "x", ClientMsgID: cmid})
	// Second insert with the SAME client_msg_id conflicts (RETURNING empty).
	m2, _ := q.InsertMessage(ctx, gen.InsertMessageParams{RoomID: room, SenderID: sender, Body: "x", ClientMsgID: cmid})
	if m2.ID != uuid.Nil {
		t.Fatal("expected ON CONFLICT DO NOTHING to return no row on retry")
	}
	// Read-back returns the ORIGINAL row — the exactly-once guarantee (§12.4).
	back, _ := q.SelectMessageByIdem(ctx, gen.SelectMessageByIdemParams{RoomID: room, SenderID: sender, ClientMsgID: cmid})
	if back.ID != m1.ID {
		t.Fatal("read-back must return the original message")
	}
}
```

### 22.6 Frontend testing notes **[I]**

- **Cache-patch logic** (`applyEnvelope`, `sendMessage`) is pure-ish and the highest-value thing to unit-test: feed it a `QueryClient` and an envelope, assert the cache. This is where realtime bugs (dupes, lost acks) live.
- **Component tests** with Testing Library assert the UI renders from the cache: seed the query cache, render, check the message appears; dispatch a `message.new` through `applyEnvelope`, check it updates.
- **WS integration** can use a mock `WebSocket` (or `mock-socket`) to drive `WSManager` through connect/reconnect/backoff without a real server.
- **E2E** (Playwright) is worth it for the two-browser sync test: open two contexts, send in one, assert it appears in the other — the same scenario as the Go `Dial` test, from the user's side.

> **Best practice — test the seams, not the tools.** You don't need to re-test that Gin routes or Ent inserts; those are covered by their own libraries. Spend your test budget on the *integration seams* this guide is about: the membership authz (does a non-member get nothing?), the idempotent send (does a retry dupe?), the broadcast (do both clients get it?), and the cache patch (does the UI reconcile ack + echo without duplicating?). Those are where *this system* breaks.

---

## 23. Deployment — Docker, Compose & Migrations-on-Deploy

We ship the whole stack as containers behind one Nginx origin: the Go API, Postgres, Redis, the Next.js app, and Nginx. Migrations run advisory-locked on deploy. Dev uses **Air** with `sqlc generate` + `goose up` wired into the reload loop. See [Docker](DOCKER_GUIDE.md) and [Nginx](NGINX_GUIDE.md).

### 23.1 The Go API Dockerfile (multi-stage) **[A]**

A multi-stage build compiles a static binary in a full Go image, then copies just the binary into a tiny runtime image. The migrations are embedded ([§4.6](#4-the-database-schema-via-goose-migrations)) so no files need copying; the generated sqlc/Ent code is compiled in.

```dockerfile
# server/Dockerfile
# ---- build stage ----
FROM golang:1.26 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
# Codegen must already be committed (sqlc + ent). Build a static binary.
RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o /out/api ./cmd/api

# ---- runtime stage ----
FROM gcr.io/distroless/static-debian12
COPY --from=build /out/api /api
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/api"]
```

### 23.2 The Next.js Dockerfile (standalone) **[A]**

Next.js 16's `output: "standalone"` produces a minimal server bundle. Multi-stage: install+build, then copy the standalone output.

```dockerfile
# web/Dockerfile
FROM node:22 AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build          # next.config: output: "standalone"

FROM node:22-slim AS run
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static
COPY --from=build /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

### 23.3 docker-compose — the whole system **[A]**

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_USER: chat
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
      POSTGRES_DB: chat
    volumes: ["pgdata:/var/lib/postgresql/data"]
    secrets: ["db_password"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U chat"]
      interval: 5s
      retries: 10

  redis:
    image: redis:8
    command: ["redis-server", "--appendonly", "yes", "--notify-keyspace-events", "Ex"]
    volumes: ["redisdata:/data"]

  api:
    build: ./server
    environment:
      APP_ENV: prod
      HTTP_ADDR: ":8080"
      DATABASE_URL: "postgres://chat@postgres:5432/chat?sslmode=disable"
      REDIS_URL: "redis://redis:6379/0"
      WEB_ORIGIN: "https://chat.example.com"
      COOKIE_DOMAIN: "chat.example.com"
    secrets: ["jwt_secret"]     # mounted; config reads JWT_SECRET from the file's content
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_started }
    deploy: { replicas: 2 }     # ≥2 → backplane earns its keep; rolling deploys work

  web:
    build: ./web
    environment:
      NODE_ENV: production
    depends_on: ["api"]

  nginx:
    image: nginx:1.27
    ports: ["443:443", "80:80"]
    volumes:
      - ./nginx/chat.conf:/etc/nginx/conf.d/default.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    depends_on: ["api", "web"]

volumes: { pgdata: {}, redisdata: {} }
secrets:
  db_password: { file: ./secrets/db_password.txt }
  jwt_secret:  { file: ./secrets/jwt_secret.txt }
```

### 23.4 Migrations on deploy **[A]**

Two safe patterns, both using the advisory-locked runner from [§4.6](#4-the-database-schema-via-goose-migrations):

1. **In-process at boot** (what our `main` does): every replica calls `MigrateUp`; the advisory lock ensures exactly one runs the migrations while the others wait then no-op. Simple; fine for small fleets.
2. **A dedicated migration job** that runs to completion *before* new app replicas start (a `compose run --rm api migrate` step, or a Kubernetes `Job`/init-container). Preferred at scale because schema changes land once, deterministically, and app pods start only against the new schema.

Either way, **migrations must be backward-compatible with the currently-running version** during a rolling deploy (both old and new code briefly run against the new schema). Follow the expand/contract pattern: add columns/tables (expand), deploy code that uses them, then remove the old ones in a later migration (contract) — never rename-in-place in one step. See the [Goose guide](GO_GOOSE_MIGRATIONS_GUIDE.md).

### 23.5 The `.air.toml` dev loop with codegen **[A]**

In development, Air watches for changes and rebuilds. We hook codegen in so editing a goose migration or a sqlc query regenerates before the rebuild — the two-tool discipline from [§2.2](#22-why-not-just-one-tool-an-honest-accounting) made automatic. A pre-build command runs `sqlc generate`; `goose up` runs at app start via `MigrateUp`. Ent codegen runs on schema edits.

```toml
# server/.air.toml
root = "."
tmp_dir = "tmp"

[build]
# Regenerate type-safe query code before each rebuild so sqlc/goose never drift.
pre_cmd = ["sqlc generate"]
cmd = "go build -o ./tmp/api ./cmd/api"
bin = "./tmp/api"
# Watch Go, the goose migrations, and the sqlc query files.
include_ext = ["go", "sql", "toml"]
include_dir = ["cmd", "internal", "db", "ent"]
exclude_dir = ["tmp", "db/gen"]   # don't loop on generated output
delay = 400

[log]
time = true
```

> **⚡ Version note — commit generated code.** Both sqlc (`db/gen`) and Ent (`ent/*`) output is committed to the repo and compiled in the Docker build ([§23.1](#23-deployment--docker-compose--migrations-on-deploy)), so CI/CD needs no codegen step and builds are reproducible. Air regenerates locally on save; CI verifies the committed output is up to date (run `sqlc generate` + `go generate ./ent` and fail if `git diff` is non-empty). This catches "forgot to regenerate" before it merges.

### 23.6 Dev vs. prod config differences **[I]**

| Concern | Dev | Prod |
|---|---|---|
| TLS | plain `http`/`ws` on localhost | `https`/`wss` via Nginx, HSTS |
| Cookie `Secure` | `false` (no HTTPS) | `true` |
| Cookie `SameSite` | `Lax` | `Strict` (one origin) |
| Origins | `localhost:3000` | your exact domain |
| Migrations | in-process at boot | dedicated job, expand/contract |
| Replicas | 1 | ≥2 (backplane, rolling deploy) |
| Logs | text, debug | JSON, info+, aggregated |

---

## 24. Maintainability — Contracts & Keeping Tools in Sync

This system has more moving parts than a single-tool app, so maintainability is a first-class concern, not an afterthought. Three disciplines keep it healthy over time: clean **layering**, a single honest **TS↔Go contract**, and a routine for keeping **goose/sqlc/Ent** and the **WS protocol** in sync.

### 24.1 Layering & where logic lives **[I]**

Each layer has one job, and dependencies point one direction (handlers → services → data; never back):

- **Transport** (`rest/` handlers, `ws/` dispatch) — parse/validate input, call a service, shape the response/envelope. No business rules here beyond validation.
- **Services** (`auth/`, `rbac/`, room service) — the business rules: authorization, the last-owner guard, DM find-or-create, refresh rotation. This is where invariants live and what tests target.
- **Data** (`db/` with sqlc + Ent over the one pool) — queries only. The write-ownership rule ([§2.2](#22-why-not-just-one-tool-an-honest-accounting)) keeps each table's mutations in one layer.

The realtime handlers are just another transport that calls the same services — the send handler ([§12.3](#12-the-message-send-flow-end-to-end)) validates then persists via the data layer exactly as a REST handler would. Keeping the Hub thin (routing + fan-out) and pushing rules into services means realtime and REST can't diverge in behavior.

### 24.2 The TS↔Go contract **[I/A]**

The DTOs ([§9.2](#9-the-rest-api-with-gin)) and the envelope/payloads ([§11](#11-the-realtime-protocol--envelope--events)) are a contract spanning two languages. A renamed Go field silently breaks the frontend at runtime — the #1 integration bug. Options, in increasing order of safety:

1. **Discipline** — one `web/lib/types.ts` mirroring the Go structs, changed in the same PR. Workable for a small team; what we've shown.
2. **Generate the TypeScript from Go** — tools like `tygo` emit `.ts` interfaces from tagged Go structs, so the wire types can't drift. Recommended once the surface grows.
3. **A shared schema** (OpenAPI for REST, a JSON-schema or protobuf for the envelope) as the single source both sides generate from. Heaviest, strongest.

Whichever you pick, the rule is: **the wire shape has exactly one definition**, and both sides derive from it. Field names use the same casing convention (we use `camelCase` in JSON via Go tags to match idiomatic TS).

### 24.3 Keeping goose / sqlc / Ent in sync **[I/A]**

The one workflow to internalize, every schema change:

1. **Write a goose migration** (`db/migrations/NNNNN_*.sql`) — the schema changes *here first, always*.
2. **Regenerate sqlc** (`sqlc generate`) — because `schema:` points at the goose dir, sqlc rebuilds its types from the new DDL; any query now inconsistent fails to generate (a good early error).
3. **Update the Ent schema** to mirror the new columns ([§5.1](#5-ent-schema--the-relation-graph)) and **`go generate ./ent`** — only for tables Ent touches.
4. **Update DTOs and TS types** if the change is visible on the wire.

CI enforces steps 2–3 by failing if regenerated output differs from committed ([§23.5](#23-deployment--docker-compose--migrations-on-deploy)). This is the tax for the two-layer data design; the Air `pre_cmd` and CI check make it nearly free. If your team can't hold this routine, that's the signal to collapse to one data tool ([§2.2](#22-why-not-just-one-tool-an-honest-accounting)).

### 24.4 Versioning the WS protocol over time **[I]**

The `v` field and `chat.vN` subprotocol ([§11.3](#11-the-realtime-protocol--envelope--events)) are your evolution mechanism. In practice: add fields and event types freely (additive, no bump); for a breaking change, ship the server accepting both `chat.v1` and `chat.v2` and translating, deploy the new client, then retire `v1` once telemetry shows old clients have drained. Because clients auto-reconnect and refetch, a rolling protocol upgrade is invisible to users. Keep a short `CHANGELOG` of protocol changes next to `protocol.go` and `types.ts` so the two sides' history is auditable.

> **Best practice — make the right thing the easy thing.** Every sync discipline above is enforced by automation (Air regen, CI diff-check, one types file) rather than memory. A maintainable system isn't one where everyone remembers the rules; it's one where forgetting the rules fails the build. Invest the afternoon to wire those checks and the two-tool data layer stops being a burden.

---
## 25. End-to-End Trace — A User Sends a Message

Everything in this guide converges on one moment. Here we walk a single message — **Alice types "ship it" in room #42 and Bob, on a different server node, sees it** — through every layer, first front-to-back (the send), then back-to-front (the delivery). Read it with the section links; it is the map of the whole system.

### 25.1 Front-to-back: Alice sends **[A]**

1. **Composer (browser).** Alice types and hits Enter. `Composer.onSubmit` ([§19.5](#19-the-chat-ui-with-react-19)) calls `sendMessage(qc, ws, "42", "ship it", aliceId)`.
2. **Optimistic insert.** `sendMessage` ([§18.6](#18-the-frontend-websocket-client)) generates `clientMsgId = crypto.randomUUID()` **once**, writes an optimistic bubble `{ id: "tmp-…", status: "sending", clientMsgId }` into the front page of the `["messages","42"]` query cache. The UI, being a pure function of the cache ([§17.4](#17-frontend-rest-data-layer-with-tanstack-query)), renders the bubble with a "…" tick **immediately** — zero perceived latency.
3. **Frame on the wire.** `ws.send({ v:1, type:"message.send", room_id:"42", client_msg_id: clientMsgId, payload:{ body:"ship it" } })`. If the socket is momentarily down, `WSManager` queues it and flushes on reconnect ([§18.2](#18-the-frontend-websocket-client)). Over `wss`, the frame is TLS-encrypted.
4. **Nginx.** The `wss://chat.example.com/ws` connection was proxied with `Upgrade`/`Connection` headers to Alice's API node (node A) ([§15.6](#15-scaling-out--the-redis-pubsub-backplane--nginx)). The frame arrives on node A.
5. **Read pump (node A).** Alice's `Client.readPump` — the single reader ([§10.5](#10-realtime-with-coderwebsocket--the-hub-architecture)) — reads the frame, size-checked against `SetReadLimit`, unmarshals the `Envelope`, and calls `hub.dispatch`.
6. **Dispatch → send handler.** `dispatch` routes `message.send` to `handleSend` ([§12.3](#12-the-message-send-flow-end-to-end)).
7. **Authorize.** `c.inRoom("42")` confirms Alice is a member — the realtime IDOR guard ([§21.4](#21-banking-grade-security-hardening)). A non-member would get `message.error{FORBIDDEN}` and stop here.
8. **Validate.** Body trimmed, non-empty, ≤ 8 KiB ([§21.5](#21-banking-grade-security-hardening)); rate limiter checked.
9. **Persist idempotently.** Inside a `pgx.Tx`, `qtx.InsertMessage` ([§6.2](#6-sqlc--type-safe-hot-path-queries)) inserts keyed by `(room_id, sender_id, client_msg_id)`. First time → a row with a server `id` and `created_at`. (A retry with the same `clientMsgId` would `ON CONFLICT DO NOTHING`, and the read-back returns the original — exactly-once, [§12.4](#12-the-message-send-flow-end-to-end).) `tx.Commit`.
10. **Ack the sender.** Node A sends `message.ack{ id, created_at }` (keyed by `clientMsgId`) down Alice's socket. `applyEnvelope`→`onMessageAck` ([§18.5](#18-the-frontend-websocket-client)) promotes her optimistic bubble to `status:"sent"` (✓) with the real id/time. Her own message is now confirmed.
11. **Fan out to the cluster.** Node A `bp.Publish("room:42", message.new{...})` to Redis ([§15.3](#15-scaling-out--the-redis-pubsub-backplane--nginx)). *Only now*, post-commit — never before ([§12.4](#12-the-message-send-flow-end-to-end)).

### 25.2 Back-to-front: Bob receives **[A]**

12. **Redis delivers to every node.** Both node A and node B are `PSubscribe`d to `room:*`. Redis delivers the `message.new` bus message to each ([§15.2](#15-scaling-out--the-redis-pubsub-backplane--nginx)).
13. **Backplane → local Hub.** On node B, `Backplane.Consume` unmarshals the bus message and calls `hub.FanoutLocal("42", data, skip=nil)` ([§15.3](#15-scaling-out--the-redis-pubsub-backplane--nginx)).
14. **Hub fan-out (node B).** The Hub's event loop looks up `rooms["42"]` — the set of local clients subscribed to room 42 — and enqueues the frame on each one's bounded `out` channel via the non-blocking `send` ([§10.2](#10-realtime-with-coderwebsocket--the-hub-architecture)). Bob's client is in that set. (A slow Bob whose buffer is full would be evicted, not block the hub — [§10.5](#10-realtime-with-coderwebsocket--the-hub-architecture).)
15. **Write pump (node B).** Bob's single write-pump goroutine drains `out` and `conn.Write`s the `message.new` frame to his browser over `wss` ([§10.5](#10-realtime-with-coderwebsocket--the-hub-architecture)).
16. **Bob's WSManager.** `onmessage` parses the envelope and calls `applyEnvelope(qc, env)` ([§18.3](#18-the-frontend-websocket-client)).
17. **Cache patch.** `onMessageNew` ([§18.5](#18-the-frontend-websocket-client)) checks Bob's `["messages","42"]` cache: the message id/clientMsgId isn't present, so it **prepends** the message to page 0. It also calls `bumpConversation` — if Bob isn't currently viewing room 42, the sidebar's `unread` for room 42 **increments** ([§18.6](#18-the-frontend-websocket-client), [§14.3](#14-read-receipts--unread-counts)).
18. **Render.** Bob's `MessageList` re-renders from the cache; "ship it" appears, XSS-safely escaped as text ([§19.4](#19-the-chat-ui-with-react-19)). If he was near the bottom, it auto-scrolls into view.
19. **Read + receipt.** Bob is looking at room 42, so the newest-message-visible effect fires `receipt.read{message_id}` ([§19.5](#19-the-chat-ui-with-react-19)). Node B's `handleReceipt` ([§14.2](#14-read-receipts--unread-counts)) moves Bob's `last_read_message_id` forward, upserts his receipt, and broadcasts `receipt.read` to the room. Alice's client receives it and renders a "Seen" marker under her message ([§19.5](#19-the-chat-ui-with-react-19)).

### 25.3 What made each guarantee hold **[A]**

- **Instant feel** — the optimistic bubble (step 2) rendered before any network round-trip.
- **Exactly-once storage** — the `client_msg_id` unique index + read-back (step 9); a dropped ack and resend can't duplicate.
- **Exactly-once display** — recipient dedupe by id/clientMsgId (step 17); the echo of Alice's own `message.new` plus any reconnect refetch can't double a bubble.
- **Cross-node delivery** — the Redis backplane (steps 11–13); Alice on node A reached Bob on node B with node A knowing nothing about Bob's connection.
- **Ordering** — `(created_at, id)` from the DB (step 9), the same tuple the keyset index and cursor use ([§6.2](#6-sqlc--type-safe-hot-path-queries)); all clients converge on DB order regardless of frame arrival order.
- **Durability & gap-fill** — the row is in Postgres before fan-out; a disconnected Bob refetches on reconnect ([§18.4](#18-the-frontend-websocket-client)) and the same dedupe makes it seamless.
- **Security** — auth on upgrade (the socket existed only because Alice's JWT verified, [§10.3](#10-realtime-with-coderwebsocket--the-hub-architecture)) and per-message membership authz (step 7).

That single trace exercises goose (the schema), sqlc (the insert), the pool (the tx), the Hub (fan-out), coder/websocket (the frames), the backplane (cross-node), JWT (the gate), and TanStack Query (the cache) — every tool in the stack, doing exactly its one job. That is the capstone.

---

## 26. Gotchas & Best Practices

A consolidated, scannable reference of the traps this guide sets you up to avoid, grouped by area. Each links to the section with the fix.

### 26.1 Data layer (goose / sqlc / Ent / pgx) **[A]**

| Gotcha | Fix | § |
|---|---|---|
| Two pools (one for Ent, one for sqlc) exhaust Postgres connections | `stdlib.OpenDBFromPool(pool)` wraps the *one* pgxpool | [§2.3](#2-the-data-layer-division-of-labor--one-pool-two-query-layers) |
| Ent auto-migrate fights goose over the schema | Turn Ent migration **off**; goose is the sole schema owner | [§2.1](#2-the-data-layer-division-of-labor--one-pool-two-query-layers) |
| Advisory lock on a pooled conn gives no mutual exclusion | Acquire a dedicated conn, lock it, hold until unlock | [§4.6](#4-the-database-schema-via-goose-migrations) |
| `OFFSET` pagination slows with depth and skips rows | Keyset pagination on `(created_at, id)` | [§6.2](#6-sqlc--type-safe-hot-path-queries) |
| Editing generated sqlc/Ent code | Never; change the source (`.sql`/schema) and regenerate | [§6.4](#6-sqlc--type-safe-hot-path-queries) |
| Schema change without regen → silent stale types | Wire `sqlc generate` + `go generate ./ent` into Air/CI | [§23.5](#23-deployment--docker-compose--migrations-on-deploy) |
| `numeric` read as float loses precision | Override to `pgtype.Numeric`; never float for money-like data | [§6.1](#6-sqlc--type-safe-hot-path-queries) |

### 26.2 Realtime (coder/websocket & the Hub) **[A]**

| Gotcha | Fix | § |
|---|---|---|
| `OriginPatterns: ["*"]` re-opens CSWSH | Exact-origin allowlist from config | [§10.4](#10-realtime-with-coderwebsocket--the-hub-architecture) |
| Not echoing an offered subprotocol → browser rejects | Advertise `chat.v1` server-side; client offers it first | [§10.3](#10-realtime-with-coderwebsocket--the-hub-architecture) |
| `Authorization` header on the WS handshake (browsers can't) | Token via subprotocol `bearer.<jwt>` or a ticket | [§10.3](#10-realtime-with-coderwebsocket--the-hub-architecture) |
| More than one reader per conn (library forbids) | One read pump; one write pump | [§10.5](#10-realtime-with-coderwebsocket--the-hub-architecture) |
| Slow client blocks the whole hub | Bounded send channel + evict on full | [§10.5](#10-realtime-with-coderwebsocket--the-hub-architecture) |
| No heartbeat → dead conns accumulate | Ping on a timer from the write pump | [§10.5](#10-realtime-with-coderwebsocket--the-hub-architecture) |
| Default 32 KiB read limit / no limit → memory DoS | `SetReadLimit` to your cap; enforce body size in handler too | [§10.5](#10-realtime-with-coderwebsocket--the-hub-architecture) |
| Broadcasting before the tx commits | Persist → commit → **then** publish | [§12.4](#12-the-message-send-flow-end-to-end) |
| HTTP `WriteTimeout`/`IdleTimeout` silently kills WS | Leave them unset; keep only `ReadHeaderTimeout` | [§3.3](#3-project-layout-config--graceful-shutdown) |
| One Hub, multiple nodes → broadcasts don't cross | Redis pub/sub backplane | [§15](#15-scaling-out--the-redis-pubsub-backplane--nginx) |
| Nginx drops WS (missing upgrade headers / short timeout) | `proxy_http_version 1.1` + Upgrade/Connection + long `proxy_read_timeout` | [§15.6](#15-scaling-out--the-redis-pubsub-backplane--nginx) |
| Presence "ghost users" after a crash | Redis TTL keys refreshed on heartbeat; never trust clean disconnect | [§13.4](#13-presence--typing-indicators) |
| Trusting client-claimed `user_id` in typing/receipts | Stamp identity from the authenticated connection | [§13.2](#13-presence--typing-indicators) |

### 26.3 Auth & security **[A]**

| Gotcha | Fix | § |
|---|---|---|
| Access token in `localStorage` (XSS → takeover) | Memory for access; HttpOnly cookie for refresh | [§16.1](#16-nextjs-frontend--setup--auth) |
| Refresh stampede logs users out | Coalesce concurrent refreshes into one promise | [§16.2](#16-nextjs-frontend--setup--auth) |
| Not asserting the JWT algorithm (`alg:none`) | `jwt.WithValidMethods(["HS256"])` | [§7.3](#7-authentication--argon2id-jwt-access--refresh-rotation) |
| Enforcing RBAC only on the frontend | Server enforces; UI hides buttons for UX only | [§20.1](#20-frontend-rbac--mirroring-the-server) |
| IDOR: reading a room by guessing its id | Membership check in handler + in-query `EXISTS` + `inRoom` on WS; 404 not 403 | [§8.2](#8-rbac--authorization--room-roles-and-idor-defense) |
| Rendering user message as HTML (XSS) | React text escaping; never `dangerouslySetInnerHTML` on user data | [§19.4](#19-the-chat-ui-with-react-19) |
| CSRF on the cookie refresh endpoint | `SameSite`, Origin check, rotation + reuse detection | [§21.7](#21-banking-grade-security-hardening) |
| Missing `connect-src wss://…` in CSP blocks the socket | Include your wss origin in the CSP | [§21.8](#21-banking-grade-security-hardening) |
| Logging tokens / message bodies | Log events + ids only; `Sensitive()` the hash | [§21.9](#21-banking-grade-security-hardening) |

### 26.4 Frontend & cache **[I/A]**

| Gotcha | Fix | § |
|---|---|---|
| Messages in `useState` beside the query cache (dupes/loss) | One cache; both REST and WS write to it | [§17.4](#17-frontend-rest-data-layer-with-tanstack-query) |
| Duplicate bubble from echo + reconnect refetch | Dedupe on `id`/`clientMsgId` in `onMessageNew` | [§18.5](#18-the-frontend-websocket-client) |
| Optimistic bubble never resolves | `message.ack` promotes by `clientMsgId`; `message.error` rolls back | [§18.5](#18-the-frontend-websocket-client) |
| Reconnect storm after a deploy | Exponential backoff **+ jitter** | [§18.2](#18-the-frontend-websocket-client) |
| Missed messages while disconnected | Refetch history/conversations on reconnect | [§18.4](#18-the-frontend-websocket-client) |
| Auto-scroll yanks user reading history | Only auto-scroll when already near bottom | [§19.3](#19-the-chat-ui-with-react-19) |
| Stuck "typing…" from a lost `typing.stop` | Client-side auto-expire timer | [§19.5](#19-the-chat-ui-with-react-19) |
| Unread badge drifts from reality | Treat `GET /conversations` as truth; badges self-correct | [§14.4](#14-read-receipts--unread-counts) |

### 26.5 Best practices, distilled **[A]**

- **One pool, right tool per job** — goose owns schema, Ent walks the graph, sqlc runs the hot path, pgx owns the sockets ([§2.4](#2-the-data-layer-division-of-labor--one-pool-two-query-layers)).
- **The cache is the single source of truth for the UI** — every REST result and WS event resolves to a `setQueryData` ([§18.5](#18-the-frontend-websocket-client)).
- **The WebSocket is a freshness optimization; Postgres is truth** — always able to reconstruct via refetch ([§12.4](#12-the-message-send-flow-end-to-end)).
- **Persist → commit → broadcast**, never reorder ([§12.4](#12-the-message-send-flow-end-to-end)).
- **Authenticate on upgrade AND authorize every message** ([§21.3](#21-banking-grade-security-hardening)).
- **The client is never the security boundary** — mirror rules for UX, enforce on the server ([§20.1](#20-frontend-rbac--mirroring-the-server)).
- **Make the right thing the easy thing** — automate codegen sync and contract checks so forgetting fails the build ([§24.4](#24-maintainability--contracts--keeping-tools-in-sync)).

---

## 27. Study Path & Build-to-Learn Extensions

### 27.1 How to work through this guide **[B]**

This capstone assumes you've met each tool alone. The efficient path:

1. **The data layer first**, because the whole architecture rests on it: [Goose migrations](GO_GOOSE_MIGRATIONS_GUIDE.md) → [sqlc + goose](GO_SQLC_GOOSE_GUIDE.md) → [Go ent ORM](GO_ENT_ORM_GUIDE.md), then re-read [§2](#2-the-data-layer-division-of-labor--one-pool-two-query-layers) until the "one pool, two query layers" split is reflex.
2. **Auth**: [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md) before [§7](#7-authentication--argon2id-jwt-access--refresh-rotation)–[§8](#8-rbac--authorization--room-roles-and-idor-defense); this guide implements the practical subset, that one explains every threat.
3. **HTTP**: [Go Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) before [§9](#9-the-rest-api-with-gin).
4. **Realtime**: [Coder WebSocket](GO_CODER_WEBSOCKETS_GUIDE.md) before [§10](#10-realtime-with-coderwebsocket--the-hub-architecture)–[§15](#15-scaling-out--the-redis-pubsub-backplane--nginx) — the Accept/Dial API, the one-reader rule, and the Hub come from there; [Redis](REDIS_GUIDE.md) for the backplane and presence.
5. **Frontend**: [React 19](REACT_19_GUIDE.md) → [Next.js 16](NEXTJS_16_GUIDE.md) → [TanStack Query](TANSTACK_QUERY_GUIDE.md) before [§16](#16-nextjs-frontend--setup--auth)–[§20](#20-frontend-rbac--mirroring-the-server).
6. **Operate it**: [Nginx](NGINX_GUIDE.md) and [Docker](DOCKER_GUIDE.md) for [§15.6](#15-scaling-out--the-redis-pubsub-backplane--nginx) and [§23](#23-deployment--docker-compose--migrations-on-deploy).
7. **Then build the whole thing once, top to bottom**, typing every line. Reading it is not the same as wiring the WS auth handshake, the idempotent send, the backplane, and the optimistic cache yourself. Finish by re-reading the [end-to-end trace](#25-end-to-end-trace--a-user-sends-a-message) — it should now read as obvious.

### 27.2 Build-to-learn extensions **[B/I/A]**

Each adds a real-world layer. Do them roughly in order; every one reuses the exact schema, envelope, and cache patterns you've built.

1. **[B] The minimum loop.** Strip to one channel, no roles, no presence: register/login, load history, send + receive one message across two browsers. Get the **JWT-over-subprotocol handshake** and the **optimistic send + ack** working — the two hardest first hurdles.
2. **[I] Rooms, DMs & RBAC.** Add channel creation, the owner/admin/member roles with the last-owner guard, and DMs as `kind='dm'` rooms ([§4.4](#4-the-database-schema-via-goose-migrations)). Prove the server enforces authz by hitting the API and the socket directly as a non-member and confirming 404/`FORBIDDEN`.
3. **[I/A] Presence, typing, unread & receipts.** Add the ephemeral features ([§13](#13-presence--typing-indicators)–[§14](#14-read-receipts--unread-counts)). Open three tabs and verify presence heals after you `kill -9` a node (the TTL).
4. **[A] Scale out.** Run two API nodes behind Nginx with the Redis backplane ([§15](#15-scaling-out--the-redis-pubsub-backplane--nginx)); confirm a message crosses nodes. Load-test with many concurrent sockets and watch the eviction metric.
5. **[A] Reactions.** Add an `emoji reaction` to a message: a new `reactions` table (goose), a sqlc upsert/count, a `reaction.add`/`reaction.remove` event pair, and a cache patch. Reuse every pattern — it should feel mechanical, which is the point.
6. **[A] File attachments.** Add uploads: a presigned-URL flow (the browser uploads to object storage directly), an `attachment` reference on the message, and size/type validation. Cross-reference the [Gin file-upload guide](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md).
7. **[A] Message edits & deletes.** Add `edited_at`/soft-delete columns (expand/contract migration), `message.edit`/`message.delete` events, moderator-vs-owner authz ([§8.1](#8-rbac--authorization--room-roles-and-idor-defense)), and idempotent cache patches. Handle the `last_read`/receipt pointer when a referenced message is deleted ([§14.4](#14-read-receipts--unread-counts)).
8. **[A] Full-text search.** Add a Postgres `tsvector` index and a sqlc search query with keyset pagination over rank; a REST `/search` endpoint scoped to the caller's rooms (the same membership guard).
9. **[A] Moderation.** Add rate-limit tiers, per-room mutes/bans (a membership status), an audit log of moderator actions ([§21.9](#21-banking-grade-security-hardening)), and slow-mode.
10. **[A] End-to-end encryption.** The deep end: client-side key exchange so the server stores only ciphertext bodies. This changes what search/preview/receipts can do (the server can no longer read bodies) — a genuine architecture exercise in the trade-off between features and privacy.

### 27.3 Where to go next **[A]**

- **A different realtime shape**: compare raw WebSockets against GraphQL subscriptions or Server-Sent Events; each has a place. This guide's Hub/backplane pattern transfers to all of them.
- **The sibling capstone**: the [Go (Gin+Ent) + Next.js Full-Stack](GO_GIN_NEXTJS_REALTIME_FULLSTACK_GUIDE.md) guide builds a CRUD-plus-realtime dashboard with gorilla/websocket and a single data tool — read it to feel *when* the extra machinery here (two query layers, a backplane) is and isn't worth it.
- **Operate it like a company**: add CI/CD, blue-green deploys, and SLO-based alerting on the metrics from [§22.2](#22-observability--testing).

You've built a complete, secure, horizontally-scalable, real-time chat system that composes goose, sqlc, Ent, pgx, coder/websocket, JWT/Argon2, Gin, Redis, and Nginx on the backend with Next.js, React, and TanStack Query on the frontend — and, more importantly, you understand the *seams* that hold them together: the one pool feeding two query layers, the JWT-over-subprotocol handshake, the persist-then-broadcast discipline, the backplane that lets nodes forget about each other, and the single cache that keeps REST and realtime from ever disagreeing. That understanding is what separates "I followed a tutorial" from "I can architect this." Now go build the next one from an empty folder.

---

*Part of the offline study library. This is the integration capstone for the Go + Next.js realtime stack — pair it with the single-tool guides it composes: [Coder WebSocket](GO_CODER_WEBSOCKETS_GUIDE.md), [Goose migrations](GO_GOOSE_MIGRATIONS_GUIDE.md), [sqlc + goose](GO_SQLC_GOOSE_GUIDE.md), [Go ent ORM](GO_ENT_ORM_GUIDE.md), [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md), [Go Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md), [Go (Gin+Ent) + Next.js Full-Stack](GO_GIN_NEXTJS_REALTIME_FULLSTACK_GUIDE.md), [Next.js 16](NEXTJS_16_GUIDE.md), [React 19](REACT_19_GUIDE.md), [TanStack Query](TANSTACK_QUERY_GUIDE.md), [PostgreSQL](POSTGRESQL_GUIDE.md), [Redis](REDIS_GUIDE.md), [Nginx](NGINX_GUIDE.md), [Docker](DOCKER_GUIDE.md). Built for 2026; confirm fast-moving package APIs (flagged ⚡) against each tool's dedicated guide and official docs before production.*
