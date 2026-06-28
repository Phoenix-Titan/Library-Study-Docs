# GraphQL with Go — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I have heard of GraphQL but never built a server" to "I can design, secure, optimize, and operate a production-scale GraphQL API in Go." This is a **learn-offline** study document: every concept is explained in *prose first* — what it is, the logic and *why* it exists, what it is for and when to use it, how to use it, the key options, best practices, and **security recommendations** — and only *then* demonstrated with heavily-commented, runnable SDL (`.graphql`) and Go code you can paste into a module. Read it top-to-bottom once; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced. This guide assumes you can already read Go — goroutines, `context`, interfaces, errors, `database/sql`. If any of that is shaky, read **[Go (Golang)](GO_GUIDE.md)** and **[Go — Language & Patterns](GO_LANG_AND_PATTERNS_GUIDE.md)** first. No internet required to learn from this document.
>
> **Version note:** This guide targets **Go 1.25 / 1.26** (current in 2026) and **[gqlgen](https://gqlgen.com)** (`github.com/99designs/gqlgen` **v0.17.x**) as the primary tool — gqlgen is schema-first, code-generating, and type-safe, and is the dominant Go GraphQL library. It mentions `graphql-go/graphql` (runtime, struct-tag) and `graph-gophers/graphql-go` (schema-first, reflection) as alternatives. Data access uses **[pgx](https://github.com/jackc/pgx) v5** against **[PostgreSQL](POSTGRESQL_GUIDE.md)**, **[Redis](REDIS_GUIDE.md)** for caching and the subscription backplane, and **DataLoader** (`github.com/graph-gophers/dataloader` v7 or gqlgen's generated loaders). Fast-moving details are flagged with **⚡ Version note**. The author is on **Windows 11**, so cross-platform notes (paths, `.exe`, shells) are called out where they matter. Always confirm exact APIs at gqlgen.com, pkg.go.dev, and graphql.org.
>
> **This guide's place in the library.** This is the definitive **GraphQL-in-Go** treatment. For the Go *language itself* see **[Go (Golang)](GO_GUIDE.md)** and **[Go — Language & Patterns](GO_LANG_AND_PATTERNS_GUIDE.md)**. For the **REST** alternative and REST-vs-GraphQL trade-offs see **[Go net/http REST API](GO_NET_HTTP_REST_API_GUIDE.md)** and **[Go Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md)**; for **gRPC** (the third API style we compare against) see **[Go gRPC & RPC](GO_GRPC_RPC_GUIDE.md)**. For an ORM you can put behind resolvers see **[Go ent ORM](GO_ENT_ORM_GUIDE.md)** (ent even has a gqlgen integration). For the database and cache layers see **[PostgreSQL](POSTGRESQL_GUIDE.md)** and **[Redis](REDIS_GUIDE.md)**. For the auth primitives reused in §9–§10 see **[Go JWT + Argon2 (Banking-Grade Auth)](GO_JWT_ARGON2_GUIDE.md)**. To compare the same ideas in another ecosystem see **[NestJS GraphQL](NESTJS_GRAPHQL_GUIDE.md)**. For shipping see **[Docker](DOCKER_GUIDE.md)** and **[Networking](NETWORKING_GUIDE.md)**.

---

## Table of Contents

1. [What GraphQL Is & Why — vs REST vs gRPC](#1-what-graphql-is--why--vs-rest-vs-grpc) **[B]**
2. [The Type System & SDL](#2-the-type-system--sdl) **[B/I]**
3. [gqlgen Setup & the Codegen Workflow](#3-gqlgen-setup--the-codegen-workflow) **[B/I]**
4. [Resolvers In Depth](#4-resolvers-in-depth) **[I/A]**
5. [The N+1 Problem & DataLoader](#5-the-n1-problem--dataloader) **[A]**
6. [Mutations & Input Validation](#6-mutations--input-validation) **[I/A]**
7. [Pagination — Offset & Relay Cursor Connections](#7-pagination--offset--relay-cursor-connections) **[I/A]**
8. [Subscriptions — Realtime over WebSocket](#8-subscriptions--realtime-over-websocket) **[A]**
9. [Authentication & Authorization](#9-authentication--authorization) **[A]**
10. [Banking-Grade GraphQL Security](#10-banking-grade-graphql-security) **[A]**
11. [Performance & Scaling to Production](#11-performance--scaling-to-production) **[A]**
12. [Schema Architecture at Scale — Modules & Federation](#12-schema-architecture-at-scale--modules--federation) **[A]**
13. [Testing](#13-testing) **[A]**
14. [Deployment & Observability](#14-deployment--observability) **[A]**
15. [Gotchas & Best Practices](#15-gotchas--best-practices) **[I/A]**
16. [Study Path & Build-to-Learn Projects](#16-study-path--build-to-learn-projects)

---

## 1. What GraphQL Is & Why — vs REST vs gRPC

### 1.1 The one-paragraph definition **[B]**

**GraphQL** is a *query language for APIs* and a *server-side runtime* for executing those queries against your data. Instead of exposing many URLs that each return a fixed shape (REST), a GraphQL server exposes **one endpoint** (`POST /query`) and a **strongly-typed schema** that describes every type of data and every operation a client can perform. The **client** sends a query that names *exactly the fields it wants*, and the server returns *exactly that shape* — no more, no less. The contract is the schema; the client drives the response shape.

That single design choice — *the client declares the response shape, the server fulfills it* — is the source of every advantage and every danger in this guide. The advantages: no over-fetching, no under-fetching, one round trip for a whole screen, a self-describing typed API. The dangers: a client can ask for an arbitrarily deep, arbitrarily expensive query, so you must defend the server (§10). Hold both halves in mind the whole way through.

### 1.2 The problem GraphQL solves — over- and under-fetching **[B]**

Picture a mobile app's "profile screen." It needs the user's name, their last 3 orders, and each order's total. Consider how each API style serves it.

**REST, the classic way.** You probably have `GET /users/1` (returns the user with *all* their columns — over-fetching, you only wanted the name), then `GET /users/1/orders?limit=3` (a second round trip — under-fetching solved by another request), then maybe `GET /orders/{id}` three times to get totals (the dreaded *N+1 round trips* on the client). Three to five HTTP requests, lots of bytes you discard, and the mobile client orchestrates it all over a high-latency radio link. To fix it you build a *custom endpoint* `GET /profile-screen` — and now you have a bespoke endpoint per screen, which is exactly the proliferation REST was supposed to avoid.

**GraphQL, the same screen.** One request:

```graphql
query ProfileScreen {
	me {
		name
		orders(first: 3) {
			id
			total
		}
	}
}
```

One round trip. The response mirrors the query precisely:

```json
{ "data": { "me": { "name": "Ada", "orders": [
	{ "id": "o_1", "total": 4200 },
	{ "id": "o_2", "total": 1990 },
	{ "id": "o_3", "total": 850 }
] } } }
```

No discarded fields, no client-side orchestration, no bespoke endpoint. When a *different* screen needs different fields, it sends a different query against the *same* schema — the server never changes. That is the headline value proposition.

### 1.3 The parts of GraphQL **[B]**

Every GraphQL system is built from a small, fixed vocabulary. Learn these eight words and the rest is detail:

| Term | What it is |
|---|---|
| **Schema (SDL)** | The typed contract. Written in **S**chema **D**efinition **L**anguage. Lists every type, field, and operation. |
| **Query** | A *read* operation. Side-effect-free by convention. Fields run **in parallel**. |
| **Mutation** | A *write* operation (create/update/delete). Top-level fields run **serially**, in order. |
| **Subscription** | A *long-lived* operation that streams updates to the client over a persistent transport (usually WebSocket). |
| **Resolver** | A server function that produces the value for *one field*. The schema says *what*; resolvers say *how*. |
| **Operation** | A single query/mutation/subscription document a client sends. |
| **Scalar** | A leaf value: `Int`, `Float`, `String`, `Boolean`, `ID`, plus custom ones (`Time`, `UUID`). |
| **Introspection** | The built-in ability to query the schema *about itself* (`__schema`, `__type`). Powers tooling — and is a security concern (§10). |

The mental model: **the schema is a graph of types**; a **query is a path (a sub-tree) the client walks**; **resolvers are the functions called at each node** to materialize the data. "Graph" is literal — `User` has `orders`, an `Order` has a `customer` (back to `User`), a `customer` has `orders`… the schema is a cyclic graph, and a query is a finite tree cut out of it.

### 1.4 GraphQL vs REST vs gRPC — when to use each **[B/I]**

These three are the dominant ways to build an API in Go. They are not strictly ranked; each wins a different scenario. (We cover gRPC fully in **[Go gRPC & RPC](GO_GRPC_RPC_GUIDE.md)** and REST in **[Go net/http REST API](GO_NET_HTTP_REST_API_GUIDE.md)**.)

| Dimension | REST | gRPC | GraphQL |
|---|---|---|---|
| **Shape** | Many URLs, resource-oriented | Many methods, RPC-oriented | One endpoint, graph-oriented |
| **Response shape** | Fixed per endpoint (server decides) | Fixed per method (proto decides) | **Client decides** per query |
| **Contract** | OpenAPI (optional, bolt-on) | `.proto` (mandatory, codegen) | **Schema/SDL (mandatory)** |
| **Wire format** | JSON (text) over HTTP/1.1 | Protobuf (binary) over HTTP/2 | JSON over HTTP/1.1 (or WS) |
| **Over/under-fetch** | Common problem | Per-method, less flexible | **Solved by design** |
| **Streaming** | SSE / polling | First-class (4 modes) | Subscriptions (1 mode) |
| **Browser-native** | Yes | No (needs proxy/grpc-web) | **Yes** |
| **Caching (HTTP)** | Excellent (GET + CDN) | Poor | Poor by default (POST); needs APQ/persisted GETs (§11) |
| **Best for** | Public CRUD APIs, cache-heavy, simple | Internal microservices, low-latency, polyglot | Client-driven UIs, aggregating many sources, mobile |

**Rules of thumb:**

- **Public, cache-heavy, resource CRUD with diverse/unknown clients?** REST. HTTP caching and ubiquity win.
- **Service-to-service inside your cluster, latency-sensitive, strongly-typed contracts?** gRPC. Binary + HTTP/2 streaming win.
- **A rich UI (web/mobile) that needs to assemble data from many tables/services in one round trip, where front-end teams iterate fast?** GraphQL. Client-driven fetching wins.

It is common and correct to **mix** them: GraphQL as the *edge/BFF* (backend-for-frontend) that your web/mobile apps talk to, with that GraphQL server calling *internal gRPC services* and *databases* underneath. GraphQL is excellent glue; it is not a database and not a replacement for your service layer.

### 1.5 When NOT to use GraphQL **[B/I]**

GraphQL is not free. Be honest about when it is the wrong tool:

- **Simple CRUD with one known client.** The schema, codegen, resolver, and DataLoader machinery is overhead you will not recoup. Ship REST.
- **You need aggressive HTTP/CDN caching of whole responses.** REST `GET` + `Cache-Control` + a CDN is unbeatable for public read-heavy data. GraphQL's `POST`-by-default defeats HTTP caches (mitigations exist — §11 — but they are extra work).
- **File uploads/downloads as the main job.** GraphQL *can* do multipart uploads, but a plain HTTP endpoint (see **[Go Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md)**) is simpler and streams better.
- **Ultra-low-latency internal RPC.** Binary gRPC beats JSON GraphQL.
- **Your team cannot afford the operational discipline** of depth/complexity limits, DataLoaders, and authorization-per-field (§5, §9, §10). A naively-built GraphQL API is a *denial-of-service vulnerability with a nice query language*. If you cannot commit to the hardening in this guide, do not expose GraphQL publicly.

The honest summary: GraphQL trades *server simplicity* for *client flexibility*. Take the trade only when client flexibility is worth real money to you.

### 1.6 The Go GraphQL library landscape **[B/I]**

Three libraries matter in Go. You should know what each is, but this guide is **gqlgen-centric** because it is what production teams ship.

| Library | Approach | Verdict |
|---|---|---|
| **`github.com/99designs/gqlgen`** | **Schema-first + codegen.** You write SDL; it generates type-safe Go models and resolver interfaces you implement. | **Primary choice.** Type-safe, fast, feature-complete (complexity, federation, dataloader hooks). The default in 2026. |
| **`github.com/graph-gophers/graphql-go`** | Schema-first + **reflection** at runtime (no codegen). You write SDL and Go methods that the library wires by reflection. | Lighter, no build step, but less type-safe and fewer batteries. Fine for small services. |
| **`github.com/graphql-go/graphql`** | **Code-first, runtime.** You build the schema in Go with `graphql.NewObject(...)` and struct tags — no SDL file. | Most verbose, stringly-typed, no codegen safety. Rarely the best choice today. |

⚡ **Version note:** gqlgen reached broad production maturity across the v0.17.x line. The library is pre-1.0 by version number but extremely stable in practice; the team treats it as production-grade. Pin the exact version in `go.mod` and read the release notes before upgrading — generated-code layout and `gqlgen.yml` keys do occasionally shift between minor releases. Always regenerate (`go generate ./...`) after upgrading.

---

## 2. The Type System & SDL

### 2.1 Why a type system is the whole point **[B]**

GraphQL's power is not "ask for the fields you want." It is "ask for the fields you want **against a contract the server guarantees and the client can introspect.**" The schema is a machine-readable type system, which means: editors autocomplete queries, clients generate typed code, the server *validates every query before executing it* (rejecting unknown fields and wrong argument types *before* a single resolver runs), and humans get living documentation. Spend real effort on schema design; it is the most durable part of your API and the hardest to change once clients depend on it.

### 2.2 Scalars — the leaves of the graph **[B]**

A **scalar** is a value that does not decompose further — it has no sub-fields. GraphQL ships five built-in scalars:

| Scalar | Meaning | Go type (gqlgen default) |
|---|---|---|
| `Int` | Signed 32-bit integer | `int` (gqlgen maps to `int`/`int32`/`int64` per config) |
| `Float` | Signed double-precision float | `float64` |
| `String` | UTF-8 text | `string` |
| `Boolean` | `true`/`false` | `bool` |
| `ID` | A unique identifier, serialized as a String but semantically opaque. Use it for primary keys. | `string` |

**`ID` deserves emphasis.** It signals "this is an identifier, treat it as opaque, do not do math on it." Clients should never parse an `ID`; it might be `"42"` today and a base64 cursor tomorrow. Using `ID` (not `Int`) for keys keeps you free to change the underlying representation.

`Int` being **32-bit** is a classic foot-gun. A `bigint` Postgres primary key, a Unix-millis timestamp, or a money-in-cents value can overflow `Int`. Use a **custom scalar** (see §2.9) or model such values as `String`. Never put a 64-bit DB id into a GraphQL `Int`.

### 2.3 Object types & fields **[B]**

An **object type** is a named bag of fields. Each field has a name, a type, and optionally **arguments**. This is where the "graph" lives — a field's type can be another object type, forming edges.

```graphql
# A leading """triple-quoted""" string is a doc comment shown in tooling.
"""A registered user of the system."""
type User {
	id: ID!
	email: String!
	displayName: String!
	# A field with an argument: the client asks for a page of orders.
	orders(first: Int = 20, after: String): OrderConnection!
	createdAt: Time!
}
```

### 2.4 Non-null (`!`) and lists (`[]`) — the modifiers **[B/I]**

Two type modifiers combine to express nullability and cardinality, and getting them right is *the* most common schema-design skill.

- **`T`** — nullable `T`. May be the value or `null`.
- **`T!`** — non-null `T`. The server *guarantees* it is never `null`. If a resolver for a non-null field returns `null` or errors, the **null propagates up** to the nearest nullable ancestor.
- **`[T]`** — nullable list of nullable `T`. The list may be `null`, and elements may be `null`.
- **`[T!]`** — nullable list of non-null `T`. The list may be `null`, but no element is `null`.
- **`[T!]!`** — non-null list of non-null `T`. Neither the list nor any element is `null`. **This is the right default for collections** ("there is always a list, possibly empty, with no holes").

**The null-propagation rule, because it surprises everyone.** Non-null is a *guarantee enforced at runtime*. If field `User.email: String!` resolves to `null` (or its resolver returns an error), GraphQL cannot honor the non-null contract for `email`, so it nulls out the *entire `User`*. If `User` was itself non-null in its parent (`me: User!`), that null propagates again, up to `me`, and so on until it hits a nullable field — potentially turning a large successful response into `"data": null` with one error. **Design implication:** mark a field non-null only when you can *truly always* produce it. When in doubt, prefer nullable for individual scalar fields, and reserve `!` for ids and for list wrappers (`[T!]!`). This is the single most important nullability lesson.

### 2.5 Enums **[B/I]**

An **enum** is a scalar restricted to a fixed set of named values. It documents intent and lets the server reject invalid values at validation time.

```graphql
enum OrderStatus {
	PENDING
	PAID
	SHIPPED
	CANCELLED
}
```

gqlgen generates a Go `type OrderStatus string` with typed constants and a validating (un)marshaller. **Best practice:** enum *values* are `SCREAMING_SNAKE_CASE` by GraphQL convention. Adding a value is backward-compatible; removing/renaming one is a **breaking change** for clients that match on it — deprecate instead (§12).

### 2.6 Interfaces **[I]**

An **interface** is an abstract type listing fields that implementing types must provide. It lets a field return "any type that *is a* Node" while still being typed. Clients select shared fields directly and type-specific fields via **inline fragments** (`... on Type`).

```graphql
"""Anything with a globally-unique ID (the Relay Node pattern)."""
interface Node {
	id: ID!
}

type User implements Node {
	id: ID!
	email: String!
}

type Product implements Node {
	id: ID!
	sku: String!
}
```

A query against an interface:

```graphql
query {
	node(id: "...") {
		id                    # shared, selectable directly
		... on User { email } # type-specific, via inline fragment
		... on Product { sku }
	}
}
```

The famous **`Node` interface** plus a top-level `node(id: ID!): Node` field is the **Relay Global Object Identification** spec — it lets generic clients refetch any object by id, and underpins caching in Relay/Apollo. Implement it for any serious public schema.

### 2.7 Unions **[I]**

A **union** is like an interface but with **no shared fields** — it just says "this is one of these types." Use it for heterogeneous results (search results, activity feeds, mutation error variants).

```graphql
union SearchResult = User | Product | Article

type Query {
	search(q: String!): [SearchResult!]!
}
```

Clients *must* use inline fragments for every variant (there are no common fields to select except `__typename`). Always let clients select **`__typename`** — it is the meta-field that tells them which concrete type they got, which is how they branch in code.

For both interfaces and unions, gqlgen needs a way to map a Go value back to a concrete GraphQL type. It generates an interface in Go (e.g. `IsNode()` / `IsSearchResult()` marker method) that your concrete structs must satisfy; the resolver returns the concrete struct and gqlgen picks the type.

### 2.8 Input types — the write side **[I]**

Object types are for **output**. For **input** (mutation/field arguments that are structured), GraphQL has a separate kind: the **input object type**, declared with `input`. The split exists because output types can have resolvers, interfaces, and unions — none of which make sense for incoming data — so the language keeps them distinct.

```graphql
input CreateOrderInput {
	customerId: ID!
	# A list of nested input objects — inputs can nest.
	items: [OrderItemInput!]!
	note: String          # nullable: optional field
	idempotencyKey: String!
}

input OrderItemInput {
	productId: ID!
	quantity: Int!
}
```

Rules to internalize: inputs may only contain scalars, enums, and other input types (no object types, no interfaces, no unions, no functions). A **nullable input field is optional**; a **non-null input field is required** and validation rejects the operation if it is missing. gqlgen generates a Go struct (`model.CreateOrderInput`) for each input type.

### 2.9 Custom scalars — `Time`, `UUID`, `JSON` **[I]**

The five built-in scalars are not enough. You will want `Time` (RFC3339 timestamps), `UUID`, `JSON` (arbitrary blob), `Decimal`/`Money` (to dodge `Float` rounding and `Int` overflow). You **declare** a custom scalar in SDL and **implement** its marshal/unmarshal in Go.

```graphql
scalar Time   # declared with no body; behavior lives in Go
scalar UUID
scalar JSON
```

In `gqlgen.yml` you bind each scalar to a Go type that knows how to (un)marshal itself. gqlgen ships ready-made bindings for `Time` and others in `github.com/99designs/gqlgen/graphql`. For a fully custom one you implement two functions:

```go
	// A custom scalar is just a pair of functions: marshal (Go -> JSON) and
	// unmarshal (JSON -> Go). gqlgen calls these wherever the scalar appears.
	package model

	import (
		"fmt"
		"io"

		"github.com/99designs/gqlgen/graphql"
		"github.com/google/uuid"
	)

	// MarshalUUID converts a uuid.UUID into a GraphQL value (a quoted string).
	func MarshalUUID(u uuid.UUID) graphql.Marshaler {
		return graphql.WriterFunc(func(w io.Writer) {
			// JSON-quote the value; uuid.String() is already safe characters.
			_, _ = io.WriteString(w, `"`+u.String()+`"`)
		})
	}

	// UnmarshalUUID parses an incoming value into a uuid.UUID, rejecting junk.
	// Returning an error here causes a clean *validation* error to the client
	// BEFORE any resolver runs -- input is never trusted.
	func UnmarshalUUID(v any) (uuid.UUID, error) {
		s, ok := v.(string)
		if !ok {
			return uuid.UUID{}, fmt.Errorf("UUID must be a string")
		}
		return uuid.Parse(s) // returns an error for malformed input
	}
```

```yaml
# gqlgen.yml -- bind the scalar names to Go marshalers.
models:
  Time:
    model: github.com/99designs/gqlgen/graphql.Time   # built-in
  UUID:
    model: yourmodule/graph/model.UUID                # your funcs above
  JSON:
    model: github.com/99designs/gqlgen/graphql.Map    # arbitrary object
```

**Security note on custom scalars:** the `Unmarshal` function is a **trust boundary**. It is the first code that touches attacker-controlled input for that field. Validate strictly (length caps on strings, parse-or-reject for UUID/Time, reject NaN/Inf for floats). A `JSON` scalar is especially dangerous: it accepts *anything*, bypasses the type system, and can carry huge or deeply-nested payloads — cap its size and avoid it for untrusted input.

### 2.10 The root operation types **[B]**

Three special object types are the entry points. By convention they are named `Query`, `Mutation`, `Subscription`.

```graphql
type Query {
	me: User
	node(id: ID!): Node
	products(first: Int, after: String): ProductConnection!
}

type Mutation {
	createOrder(input: CreateOrderInput!): CreateOrderPayload!
}

type Subscription {
	orderStatusChanged(orderId: ID!): Order!
}

# Tie them together (optional if you use the conventional names).
schema {
	query: Query
	mutation: Mutation
	subscription: Subscription
}
```

### 2.11 Schema design checklist **[I]**

A reference you will reread. Good schemas share these traits:

| Decision | Recommendation |
|---|---|
| **Keys** | Use `ID!`, never `Int`. Keeps representation opaque. |
| **Collections** | `[T!]!` — always a list, never null, no holes. |
| **Scalars (individual)** | Default to **nullable** unless truly always present; non-null cascades on failure. |
| **Naming** | Types `PascalCase`, fields/args `camelCase`, enums `SCREAMING_SNAKE_CASE`. |
| **Booleans** | Name them as positive assertions: `isActive`, `hasShipped` — not `disabled`. |
| **Mutations** | One `input` argument named `input`; return a `Payload` object (§6), not the bare entity. |
| **Pagination** | Use Relay connections (`first`/`after`, edges, `pageInfo`) for anything that grows (§7). |
| **Time** | Custom `Time` scalar (RFC3339 UTC), never raw `Int` epoch unless documented. |
| **Money** | Integer minor units (cents) as a custom `Money`/`Int64` scalar — never `Float`. |
| **Versioning** | Do **not** version the endpoint (`/v2`). Evolve additively; `@deprecated` old fields (§12). |
| **Nullability of relations** | A non-null relation (`order.customer: User!`) ties two objects' fates together via null-propagation — be sure it can never be missing. |

---

## 3. gqlgen Setup & the Codegen Workflow

### 3.1 The schema-first, codegen philosophy **[B]**

gqlgen's central idea: **the SDL schema is the source of truth, and Go code is generated from it.** You write `.graphql` files; you run `go generate`; gqlgen produces (a) Go **models** for your types/inputs/enums, (b) a typed **resolver interface** with one method per field that needs custom logic, and (c) all the **execution machinery** (parsing, validation, planning, JSON serialization). You implement the resolver methods. When you change the schema, you regenerate and the Go *compiler* tells you exactly which resolvers are now missing or mistyped. That compile-time safety is why gqlgen wins over runtime/reflection libraries.

Two consequences to absorb up front:

1. **Never hand-edit generated files** (`generated.go`, `models_gen.go`). They are overwritten on every `go generate`. Your code lives only in the resolver files gqlgen scaffolds once and then *leaves alone*.
2. **The schema drives everything.** The workflow loop is: edit SDL → `go generate ./...` → implement the new/changed resolver stubs → repeat.

### 3.2 Project layout & bootstrapping **[B/I]**

Start a module and pull gqlgen as a **tool dependency** (Go 1.24+ supports `go get -tool`, which records the tool in `go.mod` so the whole team uses the same version — pin it).

```bash
# Create the module.
mkdir graphapi && cd graphapi
go mod init github.com/you/graphapi

# Pin gqlgen as a tool dependency (records it in go.mod's `tool` directive).
go get -tool github.com/99designs/gqlgen@latest

# Scaffold a starter project: gqlgen.yml, a sample schema, server.go, resolvers.
go run github.com/99designs/gqlgen init

# Whenever you change the schema, regenerate:
go run github.com/99designs/gqlgen generate
# ...or, if you add a //go:generate directive (recommended), just:
go generate ./...
```

⚡ **Version note:** older docs use a `tools.go` file with a blank import to pin gqlgen; the modern `go get -tool` / `tool` directive (Go 1.24+) is the current idiom in 2026. Either works; the `tool` directive is cleaner. On **Windows**, run these in PowerShell or Git Bash — the commands are identical; only path separators differ in any file globs.

A typical mature layout:

```
graphapi/
  go.mod
  gqlgen.yml                # codegen config
  graph/
    schema/                 # split your SDL across files here
      schema.graphql
      user.graphql
      order.graphql
    generated.go            # GENERATED — do not edit
    model/
      models_gen.go         # GENERATED structs for types without custom Go models
      models.go             # YOUR hand-written models (e.g. domain structs)
    resolver.go             # the Resolver root struct (holds dependencies) — YOURS
    schema.resolvers.go     # GENERATED stubs, then implemented BY YOU
    user.resolvers.go
    order.resolvers.go
    loaders/                # DataLoaders (§5)
    directives/             # @auth etc. (§9)
  internal/
    store/                  # DB layer (pgx) — pure Go, no GraphQL imports
  server/
    main.go                 # wiring: router, middleware, handler
```

The discipline that pays off: **keep `internal/store` (your data layer) free of any GraphQL types.** Resolvers translate between GraphQL models and your domain/DB types. This keeps the data layer reusable (REST, gRPC, jobs) and testable in isolation.

### 3.3 `gqlgen.yml` — the config that controls codegen **[I]**

This file is the knob-board for generation. The keys you actually touch:

```yaml
# gqlgen.yml — controls what gets generated and where.

# Where your SDL lives. Globs are supported; split schema across many files.
schema:
  - graph/schema/*.graphql

# The big generated execution file.
exec:
  filename: graph/generated.go
  package: graph

# Generated model structs (only for types you don't bind to your own Go types).
model:
  filename: graph/model/models_gen.go
  package: model

# Resolver generation: "follow-schema" puts one <typename>.resolvers.go per
# schema file. This is the modern, recommended layout.
resolver:
  layout: follow-schema
  dir: graph
  package: graph

# Bind scalars and types to your own Go types here.
models:
  ID:
    model:
      - github.com/99designs/gqlgen/graphql.ID
      - github.com/99designs/gqlgen/graphql.IntID
  Int:
    model:
      - github.com/99designs/gqlgen/graphql.Int64   # 64-bit by default
  Time:
    model:
      - github.com/99designs/gqlgen/graphql.Time
  # Bind a GraphQL type to YOUR domain struct so no model is generated for it:
  User:
    model: github.com/you/graphapi/internal/store.User

# Autobind: let gqlgen find matching Go types in these packages automatically.
autobind:
  - github.com/you/graphapi/internal/store
```

The single most useful concept here is **model binding**: under `models:`, point a GraphQL type at an existing Go struct (`User: model: .../store.User`). gqlgen then *does not generate a model* for it and instead uses your struct. Fields it can match by name (case-insensitive) become **plain field access** (zero resolver code); fields it cannot match become **resolver methods you must implement**. This is how you avoid writing a trivial resolver for every scalar: bind to a struct whose fields line up with the schema, and only the *computed*/*related* fields need resolvers.

### 3.4 What gets generated — reading the output **[I]**

After `go generate`, three artifacts matter:

- **`models_gen.go`** — Go structs for every GraphQL type/input/enum *not* bound to your own type. e.g. `type CreateOrderInput struct { CustomerID string; Items []*OrderItemInput; ... }`.
- **`generated.go`** — the execution engine: `NewExecutableSchema`, field-collection, the `ComplexityRoot` struct (for complexity limits — §10), serialization. You never read this except to find the `Complexity` hook.
- **`*.resolvers.go`** — for each field needing logic, a method stub like:

```go
	// schema.resolvers.go — gqlgen scaffolds this stub ONCE; you fill the body.
	// It panics until you implement it, so you can't forget.
	func (r *queryResolver) Me(ctx context.Context) (*model.User, error) {
		panic(fmt.Errorf("not implemented: Me - me"))
	}
```

The resolver **receiver types** (`queryResolver`, `mutationResolver`, `userResolver`, …) all embed your root `*Resolver`, which is where you stash dependencies (DB pool, Redis, loaders) — see §3.6.

### 3.5 Wiring into `net/http` **[B/I]**

gqlgen gives you a `handler.Server` you mount on any `http.Handler`-compatible router. The minimal, production-shaped server:

```go
	// server/main.go
	package main

	import (
		"context"
		"errors"
		"net/http"
		"os"
		"os/signal"
		"time"

		"github.com/99designs/gqlgen/graphql/handler"
		"github.com/99designs/gqlgen/graphql/handler/extension"
		"github.com/99designs/gqlgen/graphql/handler/lru"
		"github.com/99designs/gqlgen/graphql/handler/transport"
		"github.com/99designs/gqlgen/graphql/playground"
		"github.com/jackc/pgx/v5/pgxpool"

		"github.com/you/graphapi/graph"
	)

	func main() {
		ctx := context.Background()

		// 1) Dependencies: a pgx connection pool (see POSTGRESQL_GUIDE.md).
		pool, err := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
		if err != nil {
			panic(err)
		}
		defer pool.Close()

		// 2) Build the executable schema, injecting deps into the Resolver root.
		resolver := &graph.Resolver{DB: pool}
		es := graph.NewExecutableSchema(graph.Config{Resolvers: resolver})

		// 3) Create the server. handler.New gives explicit control over which
		//    transports/extensions are enabled — important for security.
		srv := handler.New(es)
		srv.AddTransport(transport.Options{})            // CORS preflight/OPTIONS
		srv.AddTransport(transport.GET{})                // GET queries (cacheable)
		srv.AddTransport(transport.POST{})               // the normal path
		// Bound the multipart upload size to blunt giant-payload DoS (§10).
		srv.AddTransport(transport.MultipartForm{MaxUploadSize: 32 << 20})

		// 4) Use the automatic persisted-query cache (APQ) — see §11.
		srv.Use(extension.AutomaticPersistedQuery{Cache: lru.New[string](100)})
		// Introspection: enable in dev; DISABLE in prod (§10).
		if os.Getenv("ENV") != "production" {
			srv.Use(extension.Introspection{})
		}

		mux := http.NewServeMux()
		// The Playground is a browser IDE; serve it only in non-prod.
		if os.Getenv("ENV") != "production" {
			mux.Handle("/", playground.Handler("GraphQL", "/query"))
		}
		mux.Handle("/query", srv)

		httpSrv := &http.Server{
			Addr:              ":8080",
			Handler:           mux,
			ReadHeaderTimeout: 5 * time.Second, // slowloris protection
		}

		// 5) Graceful shutdown (see §14 for the full pattern).
		go func() {
			if err := httpSrv.ListenAndServe(); err != nil &&
				!errors.Is(err, http.ErrServerClosed) {
				panic(err)
			}
		}()
		stop := make(chan os.Signal, 1)
		signal.Notify(stop, os.Interrupt)
		<-stop
		shCtx, cancel := context.WithTimeout(ctx, 20*time.Second)
		defer cancel()
		_ = httpSrv.Shutdown(shCtx)
	}
```

⚡ **Version note:** older examples call `handler.NewDefaultServer(es)`, a convenience wrapper that turns on a fixed set of transports/extensions. Newer gqlgen favors `handler.New(es)` + explicit `AddTransport`/`Use` so you control exactly what is exposed — important for security (you do *not* want introspection or unbounded uploads on by accident in prod). The `lru` cache constructor is generic (`lru.New[string](n)`) in current versions.

### 3.6 The Resolver root & dependency injection **[I]**

The `Resolver` struct is *your* file (gqlgen scaffolds it empty). It is where every dependency lives, and every concrete resolver receiver embeds it, so every resolver can reach the DB, cache, loaders, config, and logger. This is plain Go DI — no framework.

```go
	// graph/resolver.go — YOUR file; survives regeneration.
	package graph

	import (
		"github.com/jackc/pgx/v5/pgxpool"
		"github.com/redis/go-redis/v9"

		"github.com/you/graphapi/internal/store"
	)

	//go:generate go run github.com/99designs/gqlgen generate

	// Resolver is the root. Put ALL dependencies here; resolvers reach them
	// via their embedded *Resolver. Keep it small and explicit.
	type Resolver struct {
		DB    *pgxpool.Pool
		RDB   *redis.Client
		Store *store.Store // your data-access layer (no GraphQL imports)
	}
```

**Best practice — never use package-level globals for the DB or config.** Inject them through `Resolver`. This makes tests trivial (construct a `Resolver` with a test store), keeps concurrency safe, and avoids the hidden-state bugs that plague global-pool designs.

### 3.7 Mounting on Gin or chi **[I]**

gqlgen's `handler.Server` is a standard `http.Handler`, so it drops into any router. For **chi** it is `r.Handle("/query", srv)`. For **Gin** (see **[Go Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md)**) wrap it:

```go
	// Gin adapter: gin.WrapH turns an http.Handler into a gin.HandlerFunc.
	r := gin.New()
	r.POST("/query", gin.WrapH(srv))
	r.GET("/query", gin.WrapH(srv))          // for GET queries / APQ
	r.GET("/", gin.WrapH(playground.Handler("GraphQL", "/query")))
```

The only subtlety: middleware that injects values into the request **context** (auth, request-id, loaders) must run *before* `srv` sees the request, because resolvers read those from `ctx`. With Gin, register that middleware on the route group; with chi/net-http, wrap the handler. We use this heavily in §5 (loaders) and §9 (auth).

---

## 4. Resolvers In Depth

### 4.1 What a resolver is and the resolver chain **[I]**

A **resolver** is a function that produces the value for *one field* in the response. GraphQL execution is recursive: to resolve `query { me { orders { total } } }`, the engine calls the `Query.me` resolver, takes its result, then for each requested sub-field calls *that* type's resolver (`User.orders`), and so on down the tree until it reaches scalars (leaves). This recursive walk is the **resolver chain**, and understanding it is the key to both performance (§5) and correctness.

Every resolver in gqlgen has this conceptual signature:

```
func (r *<typeResolver>) <Field>(ctx context.Context, obj *<ParentType>, args...) (result, error)
```

- **`ctx`** — the request context. Carries deadlines, the authenticated user (§9), the request-scoped DataLoaders (§5), tracing spans. Read everything request-scoped from here; never from globals.
- **`obj`** — the **parent object** already resolved one level up. For `User.orders`, `obj` is the `*User` produced by whatever resolved that user. This is how a child field gets the id it needs (`obj.ID`) to fetch its data.
- **`args`** — the field's typed arguments (e.g. `first int, after *string`). gqlgen has already parsed and validated them against the schema.
- **returns** — `(value, error)`. Return the data, or a non-nil error to fail *this field* (and trigger null-propagation per §2.4).

**The crucial mental shift from REST:** there is no single "handler" that builds the whole response. Each field is independent. A single client query can fan out into dozens of resolver calls. That fan-out is GraphQL's power *and* the source of the N+1 problem (§5).

### 4.2 Trivial resolvers vs. real resolvers **[I]**

When you bind a GraphQL type to a Go struct (§3.3), gqlgen reads matching fields **directly off the struct** — no resolver method is generated or called. `User.email` just reads `user.Email`. Only fields that *cannot* be a plain struct field need a resolver:

- **Computed fields** — `User.fullName` derived from first+last.
- **Related/foreign data** — `User.orders` (a DB query), `Order.customer` (another lookup).
- **Fields requiring `ctx`** — anything needing the DB, auth, or loaders.

So a well-bound schema generates *few* resolvers. The ones it generates are exactly the interesting ones: the boundaries where you hit the database or compute something. Keep this in mind — it tells you where the cost is.

### 4.3 Query resolvers — reading data **[I]**

A query resolver fetches and returns. Keep the GraphQL layer thin: it should call your **store** (data layer) and translate, not contain SQL inline if you can help it (inline is fine for small apps; a store is better at scale).

```graphql
# graph/schema/user.graphql
type Query {
	me: User
	user(id: ID!): User
}
```

```go
	// graph/user.resolvers.go
	package graph

	import (
		"context"
		"errors"

		"github.com/jackc/pgx/v5"
		"github.com/you/graphapi/graph/model"
		"github.com/you/graphapi/internal/auth"
	)

	// Query.me — returns the currently-authenticated user, or nil if anonymous.
	// `me` is NULLABLE in the schema (User, not User!) precisely so that an
	// anonymous request returns data:{me:null} rather than erroring.
	func (r *queryResolver) Me(ctx context.Context) (*model.User, error) {
		// auth.UserFromContext was placed there by HTTP middleware (see §9).
		claims, ok := auth.UserFromContext(ctx)
		if !ok {
			return nil, nil // anonymous: no user, no error
		}
		u, err := r.Store.UserByID(ctx, claims.UserID)
		if err != nil {
			if errors.Is(err, pgx.ErrNoRows) {
				return nil, nil // token valid but user deleted: treat as null
			}
			// A real DB failure: return a SANITIZED error (see §4.7).
			return nil, errInternal(ctx, err)
		}
		return u, nil
	}

	// Query.user(id) — a public lookup by id.
	func (r *queryResolver) User(ctx context.Context, id string) (*model.User, error) {
		u, err := r.Store.UserByID(ctx, id)
		if errors.Is(err, pgx.ErrNoRows) {
			return nil, nil
		}
		if err != nil {
			return nil, errInternal(ctx, err)
		}
		return u, nil
	}
```

### 4.4 Field resolvers — walking the graph **[I]**

A **field resolver** on a non-root type resolves a relation using the parent object. This is where the graph is traversed — and where N+1 lurks (§5). Here `Order.customer` looks up the user that owns the order:

```graphql
# graph/schema/order.graphql
type Order {
	id: ID!
	status: OrderStatus!
	total: Int!            # cents; plain field off the struct
	customer: User!        # RELATION — needs a resolver
}
```

```go
	// graph/order.resolvers.go
	// Order.customer — resolves the related User from order.CustomerID.
	// obj is the *model.Order already resolved one level up.
	func (r *orderResolver) Customer(ctx context.Context, obj *model.Order) (*model.User, error) {
		// NAIVE version — one DB round trip PER order. If a query returns 100
		// orders, this runs 100 times: the N+1 problem. §5 fixes it with a
		// DataLoader. Shown here naive ON PURPOSE so the problem is visible.
		u, err := r.Store.UserByID(ctx, obj.CustomerID)
		if err != nil {
			return nil, errInternal(ctx, err)
		}
		return u, nil // §5 replaces this body with: loaders.For(ctx).User.Load(obj.CustomerID)
	}
```

### 4.5 Mapping DB rows ↔ GraphQL types **[I]**

The store returns domain structs; resolvers hand them to gqlgen. The cleanest design makes your **store struct == your bound GraphQL model**, so there is *no mapping code at all* for the common fields. Use `pgx`'s `RowToStructByName` (v5) to scan straight into the struct:

```go
	// internal/store/user.go — the data layer. NO graphql imports here.
	package store

	import (
		"context"
		"time"

		"github.com/jackc/pgx/v5"
		"github.com/jackc/pgx/v5/pgxpool"
	)

	type Store struct{ pool *pgxpool.Pool }

	func New(pool *pgxpool.Pool) *Store { return &Store{pool: pool} }

	// User maps 1:1 onto the GraphQL `User` type (bound via gqlgen.yml).
	// `db` tags drive pgx scanning; gqlgen matches fields by name.
	type User struct {
		ID          string    `db:"id"`
		Email       string    `db:"email"`
		DisplayName string    `db:"display_name"`
		CreatedAt   time.Time `db:"created_at"`
	}

	// UserByID — parameterized query ($1) => NO SQL injection (see §10.10).
	func (s *Store) UserByID(ctx context.Context, id string) (*User, error) {
		rows, err := s.pool.Query(ctx,
			`SELECT id, email, display_name, created_at FROM users WHERE id = $1`, id)
		if err != nil {
			return nil, err
		}
		// CollectExactlyOneRow returns pgx.ErrNoRows if zero rows.
		u, err := pgx.CollectExactlyOneRow(rows, pgx.RowToStructByName[User])
		if err != nil {
			return nil, err
		}
		return &u, nil
	}

	// UsersByIDs — the BATCH query the DataLoader will call (§5).
	// One round trip for many ids via WHERE id = ANY($1).
	func (s *Store) UsersByIDs(ctx context.Context, ids []string) ([]*User, error) {
		rows, err := s.pool.Query(ctx,
			`SELECT id, email, display_name, created_at FROM users WHERE id = ANY($1)`, ids)
		if err != nil {
			return nil, err
		}
		list, err := pgx.CollectRows(rows, pgx.RowToAddrOfStructByName[User])
		if err != nil {
			return nil, err
		}
		return list, nil
	}
```

### 4.6 Context — the request-scoped backpack **[I]**

`ctx` is how *everything request-scoped* travels into resolvers without globals. By the time a resolver runs, the context carries (at minimum): the **deadline/cancellation** (so a client disconnect or timeout aborts DB work), the **authenticated principal** (§9), the **DataLoaders** (§5), and your **logger/trace span**. Always pass `ctx` straight through to every DB and network call — that is what makes cancellation and tracing work end to end. Never start a goroutine in a resolver that outlives the request using this `ctx`; if you must do background work, derive a fresh `context.Background()`-rooted context.

### 4.7 Error handling — GraphQL errors vs Go errors **[I/A]**

This is where GraphQL differs sharply from REST and trips people up. A GraphQL response is **always HTTP 200** (for a well-formed request) and carries a top-level `errors` array *alongside* partial `data`. A single query can **partially succeed**: some fields resolve, others error, and the client gets both. Returning an `error` from a resolver:

1. Adds an entry to the response's `errors` array (with the field's `path` and `locations`).
2. Sets that field to `null` and triggers null-propagation (§2.4) up to the nearest nullable ancestor.

The danger: **a naive resolver returns `err` straight from the DB, leaking your schema, table names, and internals to the client.** That is an information-disclosure vulnerability (§10.7). Always *sanitize*.

gqlgen gives you two tools: `graphql.AddError`/`AddErrorf` to attach an error to the current field *without* aborting siblings, and an **`ErrorPresenter`** hook on the server to rewrite every outgoing error centrally (the right place to strip internals and add an error code).

```go
	// graph/errors.go — central error helpers.
	package graph

	import (
		"context"
		"errors"
		"log/slog"

		"github.com/99designs/gqlgen/graphql"
		"github.com/vektah/gqlparser/v2/gqlerror"
	)

	// errInternal logs the REAL error server-side (with the field path) and
	// returns a generic, code-tagged error to the client. The client learns
	// "something failed here" but NOT what or where in your DB.
	func errInternal(ctx context.Context, err error) error {
		slog.ErrorContext(ctx, "resolver error",
			"path", graphql.GetPath(ctx), "err", err)
		return &gqlerror.Error{
			Message: "internal server error",
			Path:    graphql.GetPath(ctx),
			Extensions: map[string]any{
				"code": "INTERNAL_SERVER_ERROR", // machine-readable for clients
			},
		}
	}

	// A user-facing, SAFE business error (e.g. validation) — message is shown.
	func errUser(ctx context.Context, code, msg string) error {
		return &gqlerror.Error{
			Message:    msg, // safe: you wrote it, no internals
			Path:       graphql.GetPath(ctx),
			Extensions: map[string]any{"code": code},
		}
	}
```

```go
	// Wire a global ErrorPresenter so NOTHING unsanitized ever escapes.
	srv.SetErrorPresenter(func(ctx context.Context, e error) *gqlerror.Error {
		err := graphql.DefaultErrorPresenter(ctx, e)
		var ge *gqlerror.Error
		if errors.As(e, &ge) && ge.Extensions["code"] != nil {
			return err // already a sanitized, code-tagged error: pass through
		}
		// Anything else is unexpected: log it, return a generic message.
		slog.ErrorContext(ctx, "unpresented error", "err", e)
		return &gqlerror.Error{
			Message:    "internal server error",
			Path:       err.Path,
			Extensions: map[string]any{"code": "INTERNAL_SERVER_ERROR"},
		}
	})

	// A RecoverFunc converts resolver panics into safe errors (never leak a
	// stack trace to the client; log it instead).
	srv.SetRecoverFunc(func(ctx context.Context, p any) error {
		slog.ErrorContext(ctx, "panic in resolver", "panic", p)
		return gqlerror.Errorf("internal server error")
	})
```

**The error-handling rules to live by:**

| Rule | Why |
|---|---|
| **Never return raw DB/internal errors to clients.** | Leaks schema, table names, query structure (§10.7). |
| **Always attach a machine-readable `code` in `extensions`.** | Clients branch on `code`, not on the human message string. |
| **Log the real error server-side with the field `path`.** | You need it for debugging; the client must not see it. |
| **Use partial data deliberately.** | Make non-critical fields nullable so one failing field doesn't null the whole response. |
| **Set a `RecoverFunc`.** | A panic must never crash the process or leak a stack trace. |

### 4.8 Resolver dependencies and the store pattern **[A]**

At scale, resolvers should be *thin orchestrators*: read `ctx`, call the store / a loader, translate errors, return. All SQL, transactions, and caching live in `internal/store`. This separation buys you: (1) the data layer is unit-testable without a GraphQL server; (2) the same store powers REST/gRPC/cron jobs; (3) resolvers stay short and reviewable; (4) you can swap pgx for ent (**[Go ent ORM](GO_ENT_ORM_GUIDE.md)**) without touching resolver signatures. Resist the temptation to put business logic in resolvers — they are an *adapter*, not your domain.

---

## 5. The N+1 Problem & DataLoader

> **This section is make-or-break.** A GraphQL server without DataLoaders is, in nearly every real schema, a database-melting machine. Read this twice.

### 5.1 Why GraphQL *invites* N+1 **[A]**

Recall the resolver chain (§4.1): each field resolves independently, and field resolvers run **once per parent object**. Now consider:

```graphql
query {
	orders(first: 100) {   # 1 query: fetch 100 orders
		id
		customer { name }  # Order.customer resolver runs 100 TIMES
	}
}
```

The `orders` resolver does **1** SQL query and returns 100 orders. Then gqlgen calls the `Order.customer` field resolver **once for each of the 100 orders** — and the naive version (§4.4) does a `SELECT ... WHERE id = $1` each time. That is **1 + 100 = 101 queries** for one client request. Add `customer { orders { ... } }` nesting and it multiplies. This is the **N+1 problem**, and it is *intrinsic* to GraphQL's per-field execution model, not a bug in your code.

It is worse than REST's N+1 because the *client* chooses the shape: an attacker (or a careless front-end dev) can craft a query whose nesting explodes your query count, with no code change on your side. So N+1 is both a **performance** problem and a **security** problem (§10.6).

### 5.2 The fix: batch + cache per request **[A]**

The **DataLoader** pattern (from Facebook's reference implementation) solves N+1 with two mechanisms working together:

1. **Batching.** Instead of each `customer` resolver immediately hitting the DB, it calls `loader.Load(customerID)`, which *registers a request* and returns a future. The loader **waits a tick** (collects all `Load` calls made in the same event-loop turn), then fires **one** batch function `BatchUsers(ids []string)` that does a single `WHERE id = ANY($1)` query. 100 `Load` calls collapse into **1** SQL query. The 101 becomes **2** (1 for orders + 1 batched for customers).
2. **Caching (per request).** The loader memoizes by key *within a single request*. If two orders share a customer, `Load("u_1")` runs the DB fetch once and serves both. The cache is **request-scoped** — created fresh per HTTP request and discarded after — so it never serves stale data across requests and never leaks one user's data into another's request.

**Why per-request, not global?** A global cache would serve stale data and, far worse, could serve **one user's authorized data to another user** (a catastrophic authz bug). DataLoader caches are deliberately short-lived: born at request start, dead at request end. For cross-request caching use Redis with explicit invalidation (§11), never the DataLoader cache.

### 5.3 Implementing loaders with `graph-gophers/dataloader` v7 **[A]**

There are two common approaches: the standalone `github.com/graph-gophers/dataloader/v7` library, or gqlgen's own generated dataloaders. We show the graph-gophers library first because it makes the mechanism explicit, then note the gqlgen-generated alternative.

The design: a `Loaders` struct holds one `*dataloader.Loader` per batchable entity; you build it fresh per request in middleware, stash it in `ctx`, and resolvers pull it out.

```go
	// graph/loaders/loaders.go
	package loaders

	import (
		"context"
		"net/http"
		"time"

		"github.com/graph-gophers/dataloader/v7"

		"github.com/you/graphapi/internal/store"
	)

	// ctxKey is a private type so nobody else can collide with our context key.
	type ctxKey struct{}

	// Loaders bundles every per-request loader. Add one field per batchable
	// relation you need (users, products, orders-by-customer, ...).
	type Loaders struct {
		UserByID *dataloader.Loader[string, *store.User]
	}

	// newLoaders builds the loaders for ONE request, closing over the store.
	func newLoaders(s *store.Store) *Loaders {
		return &Loaders{
			UserByID: dataloader.NewBatchedLoader[string, *store.User](
				batchUsers(s),
				// Wait up to 2ms (or until the batch caps) before firing — this
				// window is what lets many Load() calls coalesce into one query.
				dataloader.WithWait[string, *store.User](2*time.Millisecond),
				dataloader.WithBatchCapacity[string, *store.User](100),
			),
		}
	}

	// batchUsers is THE batch function. dataloader hands it ALL the keys
	// collected in the window; it must return results IN THE SAME ORDER as keys.
	func batchUsers(s *store.Store) dataloader.BatchFunc[string, *store.User] {
		return func(ctx context.Context, ids []string) []*dataloader.Result[*store.User] {
			// ONE query for all ids.
			users, err := s.UsersByIDs(ctx, ids)
			results := make([]*dataloader.Result[*store.User], len(ids))
			if err != nil {
				// Propagate the error to every waiting key.
				for i := range ids {
					results[i] = &dataloader.Result[*store.User]{Error: err}
				}
				return results
			}
			// Index results by id so we can return them in KEY ORDER.
			byID := make(map[string]*store.User, len(users))
			for _, u := range users {
				byID[u.ID] = u
			}
			for i, id := range ids {
				if u, ok := byID[id]; ok {
					results[i] = &dataloader.Result[*store.User]{Data: u}
				} else {
					// Missing key: return nil data, NOT an error, so a deleted
					// customer yields null rather than failing the whole field.
					results[i] = &dataloader.Result[*store.User]{Data: nil}
				}
			}
			return results
		}
	}

	// Middleware injects a FRESH Loaders into each request's context.
	func Middleware(s *store.Store, next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			ctx := context.WithValue(r.Context(), ctxKey{}, newLoaders(s))
			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}

	// For pulls the loaders out of ctx inside a resolver.
	func For(ctx context.Context) *Loaders {
		return ctx.Value(ctxKey{}).(*Loaders)
	}
```

Wire the middleware *around* the gqlgen handler (so every request gets loaders before any resolver runs):

```go
	// In main.go, wrap the GraphQL handler:
	mux.Handle("/query", loaders.Middleware(store, srv))
```

Now the **`Order.customer` resolver becomes one line**, and N+1 evaporates:

```go
	// graph/order.resolvers.go — the FIXED version.
	func (r *orderResolver) Customer(ctx context.Context, obj *model.Order) (*model.User, error) {
		// Load registers this key; dataloader batches all sibling Loads into
		// ONE UsersByIDs() call. Thunk() blocks until the batch returns.
		return loaders.For(ctx).UserByID.Load(ctx, obj.CustomerID)()
	}
```

The `Load(...)` returns a *thunk* (a `func() (T, error)`); calling it `()` blocks until the batch resolves. Because all 100 `Customer` resolvers run concurrently (query fields are parallel), they all call `Load` within the wait window, the loader fires **one** batched query, and every thunk unblocks together.

### 5.4 The gqlgen-generated dataloader alternative **[A]**

gqlgen can also *generate* loaders for you (via the `dataloaden` tool or its built-in support), giving typed `loaders.GetUser(ctx, id)` helpers. The mechanics are identical (batch + per-request cache + ctx injection); the difference is codegen vs. hand-written. For most teams the hand-written graph-gophers approach above is clearest and most flexible (you control the batch SQL). Pick one and be consistent.

⚡ **Version note:** the dataloader ecosystem has churned — `graph-gophers/dataloader` v6 was non-generic; **v7 is generic** (the `[K, V]` type params shown above). The old `dataloaden` generator still works but is largely superseded by generics. Confirm the v7 API at pkg.go.dev before copying older blog snippets.

### 5.5 Measuring N+1 — prove it's fixed **[A]**

Do not trust that loaders work — *measure*. Three techniques:

1. **Log every SQL query with a request-id.** Count queries per request. The `orders(first:100){customer{name}}` query should show **2** queries, not 101. pgx's tracing hook (`pgx.QueryTracer`) or a `slog` line per query makes this trivial.
2. **Postgres `pg_stat_statements`.** Watch `calls` for your `WHERE id = $1` user query under load — if it spikes proportional to result-set size, a loader is missing.
3. **gqlgen's `OperationContext` / Apollo tracing** (§14) reports per-field timing; a field with huge `count` and tiny per-call time is a batch candidate you missed.

A good CI test asserts the query count (intercept the DB driver, count calls) for a known nested query — see §13.

### 5.6 DataLoader gotchas **[A]**

| Gotcha | Fix |
|---|---|
| **Loader created once globally** (not per request) | Serves stale data and cross-user data leaks. Build in middleware, per request. |
| **Batch func returns wrong order/count** | Results MUST align 1:1 with input keys by index. Build a map, emit in key order. |
| **Missing key returns an error** | Return `Data: nil` for not-found so it becomes `null`, not a whole-field failure. |
| **Forgetting to call the thunk `()`** | `Load` returns a thunk; you must call it to get the value. |
| **Wait window too long** | Adds latency to every request. 1–5ms is typical; tune under load. |
| **Loading inside the batch func** | Causes deadlock/recursion. Batch funcs hit the store directly, never other loaders for the same key space carelessly. |
| **Not clearing on writes** | After a mutation in the same request, the loader cache may be stale; `loader.Clear(key)` or just don't read-after-write through the loader. |

---

## 6. Mutations & Input Validation

### 6.1 Mutations are writes — and run serially **[I]**

A **mutation** is any operation with side effects: create, update, delete, or trigger. The one execution-model difference from queries you must know: **top-level mutation fields execute serially, in the order written**, not in parallel. This guarantees that `mutation { a b }` runs `a` fully before `b` — so clients can chain dependent writes in one request. (Nested fields *under* each top-level mutation still resolve in parallel, like a query.) Queries, by contrast, run their top-level fields in parallel.

### 6.2 The payload pattern — never return the bare entity **[I]**

A weak mutation returns the entity directly: `createOrder(input): Order!`. Three problems: (1) you cannot return *recoverable* business errors as data (only as the top-level `errors` array, which clients handle clumsily); (2) you cannot return *extra* metadata (the new edge for a connection, a client mutation id); (3) it is hard to evolve. The community standard is the **payload pattern**: every mutation returns a dedicated `XxxPayload` object that *wraps* the result and carries errors as data.

```graphql
type Mutation {
	createOrder(input: CreateOrderInput!): CreateOrderPayload!
}

input CreateOrderInput {
	customerId: ID!
	items: [OrderItemInput!]!
	idempotencyKey: String!
	note: String
}

# The payload wraps the result AND structured, recoverable errors.
type CreateOrderPayload {
	order: Order              # nullable: null when validation failed
	userErrors: [UserError!]! # always present (possibly empty)
}

"""A recoverable, user-facing error tied to an input field."""
type UserError {
	field: [String!]!  # path into the input, e.g. ["items", "0", "quantity"]
	message: String!   # safe, human-readable
	code: String!      # machine-readable: OUT_OF_STOCK, INVALID_QUANTITY, ...
}
```

The distinction that makes this pattern powerful: **`userErrors` are *expected* business outcomes** (out of stock, duplicate email, insufficient funds) returned as *data* — the request *succeeded*, the business rule said no. The top-level `errors` array is reserved for *unexpected* failures (DB down, bug, auth failure). Clients render `userErrors` inline next to form fields; they show a generic "something went wrong" for top-level errors. This separation is banking-grade UX *and* security (you control exactly what business detail leaks).

### 6.3 Validating untrusted input **[I/A]**

**Every byte of input is hostile until validated.** The GraphQL type system gives you *type* safety (a `String` is a string, an `Int!` is present) but **not** *value* safety — it will not stop `quantity: -5`, a 10MB `note`, a malformed email, or 10,000 items in one order. You must validate values yourself, in the resolver or store, *before* touching the database.

```go
	// graph/order.resolvers.go
	func (r *mutationResolver) CreateOrder(
		ctx context.Context, input model.CreateOrderInput,
	) (*model.CreateOrderPayload, error) {
		// 0) AuthZ first (see §9): who is allowed to create this order?
		actor, err := auth.Require(ctx)
		if err != nil {
			return nil, err // top-level auth error
		}

		// 1) VALIDATE values. Collect business errors into userErrors (data),
		//    don't blow up the whole request for a bad quantity.
		var ue []*model.UserError
		if len(input.Items) == 0 {
			ue = append(ue, &model.UserError{
				Field: []string{"items"}, Code: "EMPTY_ORDER",
				Message: "An order must contain at least one item.",
			})
		}
		if len(input.Items) > 200 { // cap to prevent oversized writes/DoS
			ue = append(ue, &model.UserError{
				Field: []string{"items"}, Code: "TOO_MANY_ITEMS",
				Message: "An order may contain at most 200 items.",
			})
		}
		for i, it := range input.Items {
			if it.Quantity < 1 || it.Quantity > 1000 {
				ue = append(ue, &model.UserError{
					Field:   []string{"items", strconv.Itoa(i), "quantity"},
					Code:    "INVALID_QUANTITY",
					Message: "Quantity must be between 1 and 1000.",
				})
			}
		}
		if input.Note != nil && len(*input.Note) > 2000 { // cap string length
			ue = append(ue, &model.UserError{
				Field: []string{"note"}, Code: "NOTE_TOO_LONG",
				Message: "Note must be 2000 characters or fewer.",
			})
		}
		if len(ue) > 0 {
			// Validation failed: return data (null order + the errors), NO error.
			return &model.CreateOrderPayload{UserErrors: ue}, nil
		}

		// 2) AuthZ object-level: can THIS actor order for THIS customer?
		if input.CustomerID != actor.UserID && !actor.IsAdmin {
			return nil, errUser(ctx, "FORBIDDEN", "not allowed")
		}

		// 3) Do the write transactionally + idempotently (next sections).
		order, err := r.Store.CreateOrder(ctx, store.NewOrder{
			CustomerID:     input.CustomerID,
			Items:          toStoreItems(input.Items),
			IdempotencyKey: input.IdempotencyKey,
		})
		if err != nil {
			if errors.Is(err, store.ErrOutOfStock) {
				return &model.CreateOrderPayload{UserErrors: []*model.UserError{{
					Code: "OUT_OF_STOCK", Message: "An item is out of stock.",
					Field: []string{"items"},
				}}}, nil
			}
			return nil, errInternal(ctx, err) // unexpected => top-level error
		}
		return &model.CreateOrderPayload{Order: order}, nil
	}
```

**Validation best practices:** cap every list length and string length (DoS defense), validate ranges and formats (email, phone, enum-like strings), normalize before storing (trim, lowercase emails), and reject — never silently coerce — out-of-range values. For complex rules, a validation library (e.g. `go-playground/validator`) on the input struct keeps resolvers clean.

### 6.4 Transactions **[I/A]**

A mutation that writes multiple rows (an order + its line items + a stock decrement) must be **atomic**: all-or-nothing. Wrap the work in a DB transaction *in the store layer*, so the resolver never sees half-committed state.

```go
	// internal/store/order.go
	func (s *Store) CreateOrder(ctx context.Context, in NewOrder) (*Order, error) {
		// Begin a transaction; pgx.BeginFunc commits on nil error, rolls back
		// on any error or panic — no leaked transactions.
		var created *Order
		err := pgx.BeginFunc(ctx, s.pool, func(tx pgx.Tx) error {
			// Idempotency check INSIDE the tx (see §6.5).
			if existing, ok, err := orderByIdemKey(ctx, tx, in.IdempotencyKey); err != nil {
				return err
			} else if ok {
				created = existing
				return nil // already created: return the prior result
			}
			// 1) Decrement stock with row locks; fails if insufficient.
			for _, it := range in.Items {
				ct, err := tx.Exec(ctx,
					`UPDATE products SET stock = stock - $1
					 WHERE id = $2 AND stock >= $1`, it.Quantity, it.ProductID)
				if err != nil {
					return err
				}
				if ct.RowsAffected() == 0 {
					return ErrOutOfStock // rolls back everything above
				}
			}
			// 2) Insert the order + items.
			created, err = insertOrder(ctx, tx, in)
			return err
		})
		return created, err
	}
```

### 6.5 Idempotency — exactly-once writes over an at-least-once network **[I/A]**

Networks retry. A mobile client whose request times out will resend it; without protection you create the order **twice** (and charge twice — a banking-grade defect). The fix is an **idempotency key**: the client generates a unique key per logical operation and sends it with the mutation; the server records it and, on a repeat, returns the *original* result instead of re-executing.

Implementation (shown inline in §6.4): inside the transaction, check a `orders.idempotency_key` unique column (or a dedicated `idempotency_keys` table). If present, return the stored result; if not, do the work and store the key in the *same* transaction so the check-and-write is atomic. A `UNIQUE` constraint on the key is your last line of defense against a race (two concurrent retries) — one insert wins, the other gets a unique-violation you map back to "return the existing order."

```graphql
input CreateOrderInput {
	# ... the client MUST supply a fresh UUID per logical attempt; retries reuse it.
	idempotencyKey: String!
}
```

**Best practices:** require the key for all non-trivial mutations (payments, orders, transfers); scope the key to the user (`(user_id, key)` unique) so one user's key cannot collide with another's; expire old keys (a TTL job). This is non-negotiable for financial mutations.

### 6.6 Mutation design checklist **[I/A]**

| Decision | Recommendation |
|---|---|
| **Argument** | A single `input: XxxInput!`. Easier to evolve than positional args. |
| **Return** | A `XxxPayload!` with the entity (nullable) + `userErrors: [UserError!]!`. |
| **Recoverable errors** | Return as `userErrors` data, not top-level errors. |
| **Unexpected errors** | Sanitized top-level error with a `code` (§4.7). |
| **Atomicity** | One DB transaction per logical write, in the store. |
| **Idempotency** | Required key + unique constraint for payments/orders/transfers. |
| **Authorization** | Check *before* validation cost; object-level ownership *before* write. |
| **Naming** | Verb-first: `createOrder`, `cancelOrder`, `addItemToCart` — domain actions, not CRUD. |

---

## 7. Pagination — Offset & Relay Cursor Connections

### 7.1 Why pagination is mandatory, not optional **[I]**

A field that returns `[Order!]!` with no limit is a **denial-of-service vulnerability**: a client asks for it and you try to serialize a million rows into one response, blowing memory and the DB. *Every* list that can grow must be paginated, and the limit must be **server-enforced** (a `first` argument the server caps), never client-trusted. Pagination is both a UX feature and a security control (§10.6).

There are two designs: **offset** (simple, flawed at scale) and **cursor / Relay connections** (the production standard).

### 7.2 Offset pagination — simple, and why it breaks **[I]**

`OFFSET 100 LIMIT 20` is the SQL everyone reaches for first. It is fine for small, static admin tables. It has two fatal flaws at scale:

1. **It gets slower the deeper you go.** `OFFSET 1000000` makes Postgres *scan and discard* a million rows before returning 20. Page 50,000 is a full-table scan.
2. **It skips and duplicates rows under concurrent writes.** If a row is inserted while a user pages, every subsequent page shifts — they see a row twice or miss one. Unacceptable for feeds, ledgers, transaction lists.

Use offset only for bounded, rarely-changing data where deep pages are impossible.

### 7.3 Cursor pagination — the production standard **[I/A]**

A **cursor** is an opaque token that says "I left off *here*." Instead of "skip 1,000,000 rows" you say "give me 20 rows *after this cursor*." The cursor encodes the sort key of the last row (e.g. `(created_at, id)`); the SQL becomes a **keyset / seek** query: `WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC LIMIT 21`. This is **O(log n)** via the index (no scanning discarded rows) *and* stable under concurrent inserts (you anchor to a value, not a position). It is the correct default for anything that grows.

**The cursor must be opaque.** Clients must not parse or construct cursors. Base64-encode the keyset so it is clearly not for client consumption (and so you can change the encoding later). Never put unsigned, attacker-mutable data in a cursor that you then trust for a query — validate/decode it server-side.

### 7.4 The Relay Connection spec **[I/A]**

To make cursor pagination *uniform* across the schema (so generic clients like Relay/Apollo understand it), the community standardized the **Connection** shape. Learn this once; reuse it everywhere.

```graphql
# The canonical Relay connection shape for a paginated list of Orders.
type OrderConnection {
	edges: [OrderEdge!]!    # the page of items, each with its cursor
	pageInfo: PageInfo!     # cursors + flags for the page boundaries
	totalCount: Int         # OPTIONAL; expensive — make nullable & lazy
}

type OrderEdge {
	node: Order!            # the actual item
	cursor: String!         # opaque cursor for THIS item (to paginate after it)
}

type PageInfo {
	hasNextPage: Boolean!     # is there a page after this one?
	hasPreviousPage: Boolean! # is there a page before this one?
	startCursor: String       # cursor of the first edge (null if empty)
	endCursor: String         # cursor of the last edge (null if empty)
}

type Query {
	# first/after = forward pagination; last/before = backward.
	orders(first: Int, after: String, last: Int, before: String): OrderConnection!
}
```

The arguments: **`first: N, after: cursor`** = "the first N items after this cursor" (forward). **`last: N, before: cursor`** = backward. A client pages forward by feeding `pageInfo.endCursor` back as the next `after`, until `hasNextPage` is false.

### 7.5 Implementing a connection efficiently over SQL **[A]**

The implementation has four jobs: (1) decode the incoming cursor, (2) run a keyset query fetching `first + 1` rows (the extra row tells you `hasNextPage` without a second query), (3) trim to `first` and build edges with per-row cursors, (4) build `pageInfo`.

```go
	// internal/store/orders_page.go
	package store

	import (
		"context"
		"encoding/base64"
		"fmt"
		"strings"
		"time"

		"github.com/jackc/pgx/v5"
	)

	// cursor encodes the keyset (createdAt, id) of a row.
	type cursor struct {
		CreatedAt time.Time
		ID        string
	}

	func encodeCursor(c cursor) string {
		raw := fmt.Sprintf("%d|%s", c.CreatedAt.UnixNano(), c.ID)
		return base64.StdEncoding.EncodeToString([]byte(raw)) // opaque to client
	}

	// decodeCursor parses & VALIDATES an opaque cursor. Untrusted input => must
	// reject malformed values rather than trusting them in the WHERE clause.
	func decodeCursor(s string) (cursor, error) {
		b, err := base64.StdEncoding.DecodeString(s)
		if err != nil {
			return cursor{}, fmt.Errorf("invalid cursor")
		}
		parts := strings.SplitN(string(b), "|", 2)
		if len(parts) != 2 {
			return cursor{}, fmt.Errorf("invalid cursor")
		}
		var nanos int64
		if _, err := fmt.Sscanf(parts[0], "%d", &nanos); err != nil {
			return cursor{}, fmt.Errorf("invalid cursor")
		}
		return cursor{CreatedAt: time.Unix(0, nanos).UTC(), ID: parts[1]}, nil
	}

	type OrderPage struct {
		Orders      []*Order
		HasNext     bool
		HasPrev     bool
	}

	// OrdersPage runs a keyset query. `first` is ALREADY clamped by the resolver
	// to a sane max (e.g. 100) — never trust the client's number.
	func (s *Store) OrdersPage(
		ctx context.Context, customerID string, first int, after *string,
	) (*OrderPage, error) {
		// Build the keyset predicate from the cursor, using PARAMETERS only.
		where := "customer_id = $1"
		args := []any{customerID}
		if after != nil {
			cur, err := decodeCursor(*after)
			if err != nil {
				return nil, err // surfaces as a clean user error
			}
			// Seek: rows strictly "after" the cursor in (created_at, id) DESC order.
			where += " AND (created_at, id) < ($2, $3)"
			args = append(args, cur.CreatedAt, cur.ID)
		}
		// Fetch first+1 to detect hasNextPage with no extra round trip.
		sql := fmt.Sprintf(`
			SELECT id, customer_id, status, total, created_at
			FROM orders WHERE %s
			ORDER BY created_at DESC, id DESC
			LIMIT %d`, where, first+1)

		rows, err := s.pool.Query(ctx, sql, args...)
		if err != nil {
			return nil, err
		}
		list, err := pgx.CollectRows(rows, pgx.RowToAddrOfStructByName[Order])
		if err != nil {
			return nil, err
		}
		page := &OrderPage{HasPrev: after != nil}
		if len(list) > first { // the +1 row exists => there's a next page
			page.HasNext = true
			list = list[:first] // trim the sentinel
		}
		page.Orders = list
		return page, nil
	}
```

The resolver clamps `first` and assembles the connection:

```go
	// graph/order.resolvers.go
	const maxPageSize = 100

	func (r *queryResolver) Orders(
		ctx context.Context, first *int, after *string, last *int, before *string,
	) (*model.OrderConnection, error) {
		// SERVER-ENFORCED limit: default 20, hard cap 100. Never trust client.
		n := 20
		if first != nil {
			n = *first
		}
		if n < 1 {
			n = 1
		}
		if n > maxPageSize {
			n = maxPageSize // clamp; do NOT error — just cap
		}
		actor, err := auth.Require(ctx)
		if err != nil {
			return nil, err
		}
		page, err := r.Store.OrdersPage(ctx, actor.UserID, n, after)
		if err != nil {
			return nil, errInternal(ctx, err)
		}
		// Build edges with per-row opaque cursors.
		edges := make([]*model.OrderEdge, len(page.Orders))
		for i, o := range page.Orders {
			edges[i] = &model.OrderEdge{
				Node:   o,
				Cursor: encodeCursor(cursor{CreatedAt: o.CreatedAt, ID: o.ID}),
			}
		}
		pi := &model.PageInfo{
			HasNextPage:     page.HasNext,
			HasPreviousPage: page.HasPrev,
		}
		if len(edges) > 0 {
			pi.StartCursor = &edges[0].Cursor
			pi.EndCursor = &edges[len(edges)-1].Cursor
		}
		return &model.OrderConnection{Edges: edges, PageInfo: pi}, nil
	}
```

### 7.6 `totalCount` and other expensive niceties **[A]**

`totalCount` requires a `COUNT(*)` over the whole filtered set — potentially as expensive as the page query itself, and it defeats the keyset speedup. Make it **nullable** and **resolve it lazily** (only run the `COUNT` if the client actually selects `totalCount`). gqlgen lets you know which fields were requested via `graphql.CollectFieldsCtx`/`CollectAllFields`, so a connection resolver can skip the count entirely when unselected. For huge tables, return an **estimate** (`reltuples` from `pg_class`) instead of an exact count, or omit `totalCount` from the schema.

### 7.7 Pagination decision table **[I/A]**

| Need | Use |
|---|---|
| Small, bounded admin list | Offset (`limit`/`offset`) is fine |
| Anything that grows (feeds, transactions, search) | **Cursor / Relay connection** |
| Stable results under concurrent writes | **Cursor** (offset duplicates/skips) |
| Deep pages must stay fast | **Cursor** (keyset is O(log n)) |
| Client needs exact total | `totalCount` lazily, or estimate; consider dropping it |
| Public schema, generic clients | **Relay connection shape** (edges/pageInfo) |

---

## 8. Subscriptions — Realtime over WebSocket

### 8.1 What a subscription is and when to use one **[A]**

A **subscription** is a long-lived operation: the client opens it once and the server **pushes** a stream of results whenever an event happens (an order ships, a message arrives, a price ticks). Unlike a query (one request → one response) a subscription is one request → *many* responses over time, delivered over a **persistent transport** — almost always **WebSocket**.

Use subscriptions for genuinely realtime, server-initiated updates: chat, live dashboards, notifications, collaborative editing, order-status changes. Do **not** use them as a substitute for polling cheap data, and do not use them for request/response work (that is a query). They are the most operationally demanding part of GraphQL — they hold open connections, consume memory per client, and complicate horizontal scaling (§8.5). Add them only when the realtime value is real.

For deep WebSocket mechanics see **[Go Gorilla WebSockets](GO_GORILLA_WEBSOCKETS_GUIDE.md)** and **[Node WebSockets](NODE_WEBSOCKETS_GUIDE.md)**; this section covers the GraphQL-specific layer on top.

### 8.2 The `graphql-ws` protocol **[A]**

Subscriptions ride a sub-protocol over the WebSocket. The modern standard is **`graphql-ws`** (the `graphql-transport-ws` protocol), which replaced the deprecated `subscriptions-transport-ws` (`graphql-ws` legacy). gqlgen's `transport.Websocket` speaks `graphql-transport-ws`. The handshake: client connects with `Sec-WebSocket-Protocol: graphql-transport-ws`, sends `connection_init` (carrying auth — §8.6), server replies `connection_ack`, then the client sends `subscribe` messages and the server streams `next` messages, ending with `complete`.

⚡ **Version note:** ensure your client library uses **`graphql-transport-ws`** (graphql-ws package), not the legacy `subscriptions-transport-ws`. They are wire-incompatible. gqlgen supports both via `InitFunc`/protocol negotiation, but standardize on the modern one.

### 8.3 A subscription resolver returns a channel **[A]**

In gqlgen, a subscription resolver is special: instead of returning a value, it returns a **receive channel**. gqlgen reads from that channel and sends each value to the client until the channel closes or the client disconnects (which cancels `ctx`). Your job: create a channel, register it with a pub-sub broker, and spawn a goroutine that fans events into it and cleans up on `ctx.Done()`.

```graphql
type Subscription {
	# Stream status changes for one order the caller is allowed to see.
	orderStatusChanged(orderId: ID!): Order!
}
```

```go
	// graph/subscription.resolvers.go
	func (r *subscriptionResolver) OrderStatusChanged(
		ctx context.Context, orderID string,
	) (<-chan *model.Order, error) {
		// 1) AUTHORIZE the subscription up front (see §8.6/§9): can this caller
		//    watch this order? Reject here, before any stream is set up.
		actor, err := auth.Require(ctx)
		if err != nil {
			return nil, err
		}
		if ok, err := r.Store.UserCanViewOrder(ctx, actor.UserID, orderID); err != nil {
			return nil, errInternal(ctx, err)
		} else if !ok {
			return nil, errUser(ctx, "FORBIDDEN", "not allowed")
		}

		// 2) Make the output channel gqlgen will drain.
		out := make(chan *model.Order, 1)

		// 3) Subscribe to the broker for this order's topic.
		topic := "order:" + orderID
		events, unsubscribe := r.Broker.Subscribe(ctx, topic)

		// 4) Pump events -> out until the client goes away (ctx cancelled).
		go func() {
			defer close(out)       // signals gqlgen the stream ended
			defer unsubscribe()    // release broker resources (avoid leaks!)
			for {
				select {
				case <-ctx.Done():
					return // client disconnected or server shutting down
				case ev, ok := <-events:
					if !ok {
						return
					}
					order, err := decodeOrderEvent(ev)
					if err != nil {
						continue // skip bad event; don't kill the stream
					}
					select {
					case out <- order:
					case <-ctx.Done():
						return
					}
				}
			}
		}()
		return out, nil
	}
```

Enable the WebSocket transport on the server:

```go
	srv.AddTransport(transport.Websocket{
		KeepAlivePingInterval: 10 * time.Second, // detect dead peers
		// InitFunc authenticates the connection_init payload (see §8.6).
		InitFunc: wsAuthInitFunc,
		// Bound concurrent subscriptions per connection if needed.
	})
```

### 8.4 An in-process pub-sub broker **[A]**

For a single instance, an in-memory broker (a map of topic → set of subscriber channels, guarded by a mutex) is enough. Publishers (mutation resolvers) call `Broker.Publish(topic, event)`; subscribers register a channel. **This does not survive across instances** — see §8.5 for the Redis backplane.

```go
	// internal/pubsub/memory.go
	package pubsub

	import (
		"context"
		"sync"
	)

	type MemoryBroker struct {
		mu   sync.RWMutex
		subs map[string]map[chan []byte]struct{} // topic -> set of subscribers
	}

	func NewMemory() *MemoryBroker {
		return &MemoryBroker{subs: make(map[string]map[chan []byte]struct{})}
	}

	func (b *MemoryBroker) Subscribe(ctx context.Context, topic string) (<-chan []byte, func()) {
		ch := make(chan []byte, 8) // buffered so a slow publisher doesn't block
		b.mu.Lock()
		if b.subs[topic] == nil {
			b.subs[topic] = make(map[chan []byte]struct{})
		}
		b.subs[topic][ch] = struct{}{}
		b.mu.Unlock()

		unsub := func() {
			b.mu.Lock()
			delete(b.subs[topic], ch)
			if len(b.subs[topic]) == 0 {
				delete(b.subs, topic)
			}
			b.mu.Unlock()
			close(ch)
		}
		return ch, unsub
	}

	func (b *MemoryBroker) Publish(topic string, payload []byte) {
		b.mu.RLock()
		defer b.mu.RUnlock()
		for ch := range b.subs[topic] {
			select {
			case ch <- payload: // deliver
			default:            // subscriber slow/full: DROP, never block publisher
			}
		}
	}
```

A mutation publishes when state changes:

```go
	// After committing an order status change:
	payload, _ := json.Marshal(updatedOrder)
	r.Broker.Publish("order:"+updatedOrder.ID, payload)
```

### 8.5 Scaling subscriptions across instances with Redis **[A]**

The moment you run **more than one** server instance behind a load balancer, the in-memory broker breaks: a client's WebSocket lands on instance A, but the mutation that should notify them runs on instance B — and B's in-memory publish never reaches A's subscribers. The fix is a **backplane**: a shared message bus every instance subscribes to. **Redis Pub/Sub** is the standard choice (see **[Redis](REDIS_GUIDE.md)**).

Architecture: each instance keeps its *local* in-memory fan-out (the WebSocket connections live there), **plus** subscribes to the relevant Redis channels. When a mutation publishes, it publishes to **Redis**; every instance receives it via Redis Pub/Sub and fans it out to *its own* local subscribers. So a write on B reaches a WebSocket on A through Redis.

```go
	// internal/pubsub/redis.go — a backplane that bridges Redis <-> local fan-out.
	package pubsub

	import (
		"context"

		"github.com/redis/go-redis/v9"
	)

	type RedisBroker struct {
		rdb   *redis.Client
		local *MemoryBroker // local fan-out to this instance's WS connections
	}

	func NewRedis(rdb *redis.Client) *RedisBroker {
		return &RedisBroker{rdb: rdb, local: NewMemory()}
	}

	// Publish goes to Redis so EVERY instance (incl. this one) receives it.
	func (b *RedisBroker) Publish(ctx context.Context, topic string, payload []byte) error {
		return b.rdb.Publish(ctx, topic, payload).Err()
	}

	// Subscribe registers a LOCAL subscriber, and lazily ensures this instance
	// is listening to the Redis channel for that topic.
	func (b *RedisBroker) Subscribe(ctx context.Context, topic string) (<-chan []byte, func()) {
		b.ensureRedisListener(ctx, topic) // idempotent; one Redis sub per topic
		return b.local.Subscribe(ctx, topic)
	}

	// ensureRedisListener starts a goroutine (once per topic) that reads from
	// Redis and republishes into the LOCAL broker, fanning out to WS clients.
	func (b *RedisBroker) ensureRedisListener(ctx context.Context, topic string) {
		// (Track started topics in a map+mutex to avoid duplicate listeners.)
		sub := b.rdb.Subscribe(ctx, topic)
		go func() {
			ch := sub.Channel()
			for msg := range ch {
				b.local.Publish(msg.Channel, []byte(msg.Payload))
			}
		}()
	}
```

**Scaling considerations and caveats:**

- **Redis Pub/Sub is fire-and-forget.** If an instance is momentarily disconnected from Redis, it misses messages — there is no replay. For at-least-once delivery use **Redis Streams** (consumer groups) instead, at the cost of complexity. For most UIs, fire-and-forget is acceptable (clients refetch on reconnect).
- **Channel cardinality.** One Redis channel per entity (`order:123`) is fine until you have millions of live topics. Consider per-type channels (`orders`) with server-side filtering, trading Redis fan-out for app-side filtering.
- **Connection limits.** Each WebSocket is an open FD and a goroutine; an instance handles tens of thousands, not millions. Scale horizontally and tune `ulimit`, kernel `somaxconn`, and your LB's idle timeout (it must exceed your keepalive ping interval, or the LB kills idle WS — see **[Nginx](NGINX_GUIDE.md)** for `proxy_read_timeout` and `Upgrade` headers).
- **Graceful shutdown.** On SIGTERM, stop accepting new WS, send `complete` to active subscriptions, drain, then exit (§14).

### 8.6 Authenticating the subscription transport **[A]**

A subscription's HTTP upgrade often **cannot carry an `Authorization` header** the way `fetch` does (browsers don't let you set headers on the WebSocket handshake). The `graphql-ws` protocol solves this: the client sends credentials in the **`connection_init`** payload, and the server authenticates them in gqlgen's **`InitFunc`** *before* any subscription runs. Put the resulting principal into the connection's context so subscription resolvers can authorize (as in §8.3).

```go
	// wsAuthInitFunc authenticates the connection_init payload and stuffs the
	// principal into ctx for the lifetime of the WebSocket connection.
	func wsAuthInitFunc(
		ctx context.Context, initPayload transport.InitPayload,
	) (context.Context, *transport.InitPayload, error) {
		// Client sends: { "type":"connection_init", "payload":{"authToken":"Bearer ..."} }
		token := initPayload.Authorization() // helper reads "Authorization" key
		if token == "" {
			// Reject anonymous WS connections (or allow, per your policy).
			return ctx, nil, fmt.Errorf("missing auth")
		}
		claims, err := auth.VerifyBearer(token) // JWT verify (see GO_JWT_ARGON2_GUIDE.md)
		if err != nil {
			return ctx, nil, fmt.Errorf("unauthorized")
		}
		ctx = auth.WithUser(ctx, claims) // available to all subscription resolvers
		return ctx, &initPayload, nil
	}
```

**Subscription auth rules:** authenticate at `connection_init` (transport level) **and** authorize each `subscribe` operation (object-level — "can this user watch *this* order?", as in §8.3). Re-check authorization is still valid for long-lived streams if your tokens expire — a stream opened an hour ago may now be unauthorized; consider closing streams when the token's TTL passes. Never assume "they authenticated at connect" is sufficient forever.

---

## 9. Authentication & Authorization

> **Authentication** = *who are you?* **Authorization** = *what are you allowed to do?* GraphQL changes nothing about authn (do it in HTTP middleware, same as REST) but makes authz *harder* because there is no single endpoint to guard — every field is a potential leak. This section is where most GraphQL security incidents originate.

### 9.1 Authentication: at the HTTP edge, into the context **[A]**

Authenticate **before** GraphQL execution, in standard HTTP middleware, exactly as you would for REST. Verify the credential (a JWT bearer token — see **[Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md)** for the full token/Argon2 story), and on success place the **principal** (user id, roles, scopes) into the request `context`. Every resolver then reads identity from `ctx`. Crucially: **a bad/expired token does not 401 the whole request** by default — you let it through as *anonymous* and let resolvers/authz decide, so public fields still work for logged-out users. (You *can* hard-401 if your API has no public surface.)

```go
	// internal/auth/middleware.go
	package auth

	import (
		"context"
		"net/http"
		"strings"
	)

	type ctxKey struct{}

	type Principal struct {
		UserID  string
		Roles   []string
		Scopes  []string
		IsAdmin bool
	}

	// Middleware verifies a bearer token if present and attaches the Principal.
	// No token / bad token => request continues ANONYMOUSLY (no principal).
	func Middleware(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			ctx := r.Context()
			if h := r.Header.Get("Authorization"); strings.HasPrefix(h, "Bearer ") {
				if claims, err := VerifyBearer(h); err == nil {
					ctx = WithUser(ctx, claims) // attach principal
				}
				// On verify error we DON'T reject here; authz decides per field.
			}
			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}

	func WithUser(ctx context.Context, p *Principal) context.Context {
		return context.WithValue(ctx, ctxKey{}, p)
	}

	// UserFromContext returns the principal and whether one exists.
	func UserFromContext(ctx context.Context) (*Principal, bool) {
		p, ok := ctx.Value(ctxKey{}).(*Principal)
		return p, ok
	}

	// Require returns the principal or a FORBIDDEN error — use in resolvers that
	// demand authentication.
	func Require(ctx context.Context) (*Principal, error) {
		if p, ok := UserFromContext(ctx); ok {
			return p, nil
		}
		return nil, ErrUnauthenticated // mapped to a sanitized top-level error
	}
```

Wire it outside the GraphQL handler (and outside the loaders middleware, so order is: auth → loaders → gqlgen):

```go
	handlerChain := auth.Middleware(loaders.Middleware(store, srv))
	mux.Handle("/query", handlerChain)
```

### 9.2 Authorization is per-field, and that is the hard part **[A]**

In REST, you guard a *route*. In GraphQL there is one route, and a single query can touch dozens of fields across many types. So authorization must be enforced **at the field/object level**, not at the endpoint. There are two complementary mechanisms:

1. **Schema directives** (`@auth`, `@hasRole`) — declarative guards attached to fields in SDL, enforced by gqlgen *before* the resolver runs. Best for coarse "must be logged in / must have role X" checks.
2. **Resolver guards** — imperative checks inside resolvers, with access to the actual data. Required for **object-level / ownership** checks ("is this *your* order?") that depend on runtime values a directive cannot see.

You will use both. Directives handle the broad strokes declaratively; resolver guards handle the fine-grained, data-dependent decisions.

### 9.3 Field-level authorization with directives **[A]**

A **directive** is schema metadata (`@auth`) that gqlgen turns into a Go function wrapping the field's resolution. Declare it in SDL, implement it in Go, and register it in the config.

```graphql
# graph/schema/directives.graphql
directive @auth on FIELD_DEFINITION
directive @hasRole(role: Role!) on FIELD_DEFINITION

enum Role { USER ADMIN AUDITOR }

type Query {
	me: User @auth                       # must be authenticated
	allUsers: [User!]! @hasRole(role: ADMIN)  # must be an admin
}

type Mutation {
	deleteUser(id: ID!): Boolean! @hasRole(role: ADMIN)
}
```

```go
	// graph/directives/auth.go
	package directives

	import (
		"context"

		"github.com/99designs/gqlgen/graphql"

		"github.com/you/graphapi/internal/auth"
		"github.com/you/graphapi/graph/model"
	)

	// Auth runs BEFORE the wrapped resolver. If unauthenticated, it short-
	// circuits with an error and the resolver never runs.
	func Auth(ctx context.Context, _ any, next graphql.Resolver) (any, error) {
		if _, ok := auth.UserFromContext(ctx); !ok {
			return nil, gqlErr(ctx, "UNAUTHENTICATED", "authentication required")
		}
		return next(ctx) // authorized: proceed to the real resolver
	}

	// HasRole takes the directive's typed argument (role) and checks RBAC.
	func HasRole(ctx context.Context, _ any, next graphql.Resolver, role model.Role) (any, error) {
		p, ok := auth.UserFromContext(ctx)
		if !ok {
			return nil, gqlErr(ctx, "UNAUTHENTICATED", "authentication required")
		}
		if !hasRole(p, role) {
			// FORBIDDEN, not UNAUTHENTICATED: they're known but not allowed.
			return nil, gqlErr(ctx, "FORBIDDEN", "insufficient permissions")
		}
		return next(ctx)
	}
```

Register the directives in the schema config:

```go
	// In main.go, when building the executable schema:
	cfg := graph.Config{Resolvers: resolver}
	cfg.Directives.Auth = directives.Auth
	cfg.Directives.HasRole = directives.HasRole
	es := graph.NewExecutableSchema(cfg)
```

⚡ **Version note:** the generated signature of a directive function depends on its declared arguments — change the SDL directive args and you must regenerate and update the Go function signature to match. The compiler enforces this, which is the point.

### 9.4 Object-level / ownership checks in resolvers **[A]**

Directives cannot answer "is this *the caller's* order?" because the answer depends on the *data*, which only exists after a fetch. These checks live in resolvers (or, better, in the store, which is the real authority on the data). The rule: **fetch, then authorize against the fetched object's owner.**

```go
	func (r *queryResolver) Order(ctx context.Context, id string) (*model.Order, error) {
		actor, err := auth.Require(ctx)
		if err != nil {
			return nil, err
		}
		o, err := r.Store.OrderByID(ctx, id)
		if errors.Is(err, pgx.ErrNoRows) {
			// IMPORTANT: return null, not "forbidden" — see §10.8. Distinguishing
			// "doesn't exist" from "exists but not yours" leaks existence.
			return nil, nil
		}
		if err != nil {
			return nil, errInternal(ctx, err)
		}
		// OWNERSHIP check: not your order, and not an admin/auditor => null.
		if o.CustomerID != actor.UserID && !actor.IsAdmin {
			return nil, nil // treat as not-found to avoid existence leak
		}
		return o, nil
	}
```

**Best practice — push authz into the query.** Even cleaner and safer than fetch-then-check is to *scope the SQL itself*: `SELECT ... FROM orders WHERE id = $1 AND customer_id = $2`. The database returns nothing for someone else's order, so there is no object to leak and no separate check to forget. Defense-in-depth: do both — scope the query *and* assert ownership in code.

### 9.5 RBAC, scopes, and centralizing policy **[A]**

For anything beyond a handful of roles, scattering `if actor.IsAdmin` across resolvers becomes unmaintainable and error-prone (one forgotten check = a breach). Centralize authorization into a **policy** object the resolvers call:

```go
	// internal/authz/policy.go — ONE place that knows the rules.
	package authz

	type Policy struct{ /* deps: role store, etc. */ }

	// CanViewOrder is the single source of truth for this decision.
	func (p *Policy) CanViewOrder(actor *auth.Principal, o *store.Order) bool {
		switch {
		case actor == nil:
			return false
		case actor.IsAdmin:
			return true
		case has(actor.Scopes, "orders:read:all"):
			return true
		default:
			return o.CustomerID == actor.UserID
		}
	}
```

Resolvers become `if !r.Policy.CanViewOrder(actor, o) { return nil, nil }`. Now authorization logic is testable in isolation (§13), auditable in one file, and impossible to get subtly inconsistent across fields. For very large systems, externalize to a policy engine (OPA/Rego, Cedar) — but a plain Go policy package covers most needs and is easier to reason about.

### 9.6 The authorization checklist **[A]**

| Check | Where | Note |
|---|---|---|
| Authenticated? | `@auth` directive or `auth.Require` | Coarse gate. |
| Has role/scope? | `@hasRole` directive or policy | RBAC; declarative for simple cases. |
| Owns this object? | Resolver/store, data-dependent | Scope the SQL; assert in code (defense-in-depth). |
| Field visible to this role? | Resolver returns null, or field directive | e.g. `User.ssn` only for the owner/admin. |
| Mutation allowed? | Resolver, *before* validation/write | Fail fast, before doing work. |
| Subscription allowed? | InitFunc (authn) + resolver (authz) | §8.6. |
| Existence not leaked? | Return null for "not yours", same as "not found" | §10.8. |
| Policy centralized? | `internal/authz` package | One source of truth; unit-tested. |

---

## 10. Banking-Grade GraphQL Security

> **The defining GraphQL security fact:** the *client* writes the query. That flexibility, unguarded, is a denial-of-service and data-exfiltration weapon. A REST endpoint does a fixed amount of work; a GraphQL endpoint does *whatever the attacker asks*. Every control below exists to put bounds back on "whatever the attacker asks." Treat this section as the OWASP-for-GraphQL checklist with Go code. **None of these are optional for a public, money-handling API.**

### 10.1 The threat model in one table **[A]**

| Attack | What it does | Primary defense |
|---|---|---|
| **Deep nesting** | `a{b{a{b{...}}}}` — exponential resolver fan-out | Depth limit (§10.2) |
| **Expensive query** | Wide selection / huge `first` — melts DB | Complexity/cost analysis (§10.3) |
| **Query flooding** | Many requests/sec | Rate limiting (§10.4) |
| **Schema recon** | Introspection reveals the whole schema | Disable introspection in prod (§10.5) |
| **Alias/batch amplification** | One request, thousands of aliased fields | Complexity + alias/batch caps (§10.6) |
| **Error/field-suggestion leakage** | "Did you mean `password`?" / stack traces | Hide suggestions, sanitize errors (§10.7) |
| **Existence/IDOR leakage** | "Forbidden" vs "Not found" reveals data exists | Uniform null responses (§10.8) |
| **Injection** | SQL/NoSQL via field args | Parameterized queries (§10.9) |
| **CSRF** | Browser sends authed mutation cross-site | CSRF token / non-simple content-type (§10.10) |
| **AuthZ bypass** | Forgotten field guard | Centralized policy, deny-by-default (§9, §10.11) |

### 10.2 Query depth limiting **[A]**

GraphQL schemas are cyclic (`User.orders → Order.customer → User.orders → …`), so a malicious query can nest arbitrarily deep, and because each level fans out across resolvers (and possibly DataLoader batches), deep nesting is a cheap way to make the server do enormous work. **Cap the maximum query depth** — reject any operation nested deeper than, say, 10–15 levels, *before execution*. gqlgen exposes this via an `OperationContext` validation hook / the `complexity` machinery; a depth-specific limiter walks the parsed operation's selection set.

```go
	// internal/gqlsec/depth.go — reject operations nested deeper than maxDepth.
	package gqlsec

	import (
		"context"

		"github.com/99designs/gqlgen/graphql"
		"github.com/vektah/gqlparser/v2/ast"
		"github.com/vektah/gqlparser/v2/gqlerror"
	)

	// DepthLimit returns an OperationMiddleware-style guard. Register it with
	// srv.AroundOperations(...). It runs AFTER parse/validate, BEFORE execution.
	func DepthLimit(maxDepth int) graphql.OperationMiddleware {
		return func(ctx context.Context, next graphql.OperationHandler) graphql.ResponseHandler {
			oc := graphql.GetOperationContext(ctx)
			if d := operationDepth(oc.Operation.SelectionSet); d > maxDepth {
				return graphql.OneShot(graphql.ErrorResponse(ctx,
					"query exceeds maximum depth of %d (got %d)", maxDepth, d))
			}
			return next(ctx)
		}
	}

	// operationDepth recursively measures the deepest selection nesting.
	func operationDepth(set ast.SelectionSet) int {
		max := 0
		for _, sel := range set {
			var childDepth int
			switch s := sel.(type) {
			case *ast.Field:
				childDepth = 1 + operationDepth(s.SelectionSet)
			case *ast.InlineFragment:
				childDepth = operationDepth(s.SelectionSet) // fragments don't add depth
			case *ast.FragmentSpread:
				if s.Definition != nil {
					childDepth = operationDepth(s.Definition.SelectionSet)
				}
			}
			if childDepth > max {
				max = childDepth
			}
		}
		return max
	}
```

```go
	srv.AroundOperations(gqlsec.DepthLimit(12)) // tune per schema
```

⚡ **Version note:** community packages (e.g. validation-rule helpers) exist for depth limiting, but they track gqlparser's AST, which can change between gqlgen versions. The hand-rolled walker above is robust and dependency-light. Always set a depth limit *and* a complexity limit — depth alone does not stop a *wide* query.

### 10.3 Query complexity / cost analysis **[A]**

Depth limits a *narrow-but-deep* query; they do nothing about a *shallow-but-enormous* one (`users(first: 10000) { orders(first: 10000) { items(first: 10000) { ... } } }` — 10000³ potential nodes at depth 3). **Complexity analysis** assigns each field a *cost* and rejects operations whose total estimated cost exceeds a budget *before execution*. gqlgen has first-class support: it generates a `ComplexityRoot` struct; you assign a cost function per field (especially list fields, where cost ∝ the `first`/limit argument).

```go
	// internal/gqlsec/complexity.go
	package gqlsec

	import "github.com/you/graphapi/graph"

	// Configure costs. The killer move: a list field costs childComplexity *
	// the page size, so asking for more items costs proportionally more.
	func ConfigureComplexity(cfg *graph.Config) {
		// orders(first): cost = first * (cost of each Order subtree)
		cfg.Complexity.Query.Orders = func(childComplexity int, first *int, after *string, last *int, before *string) int {
			n := 20
			if first != nil {
				n = *first
			}
			return n * childComplexity
		}
		// A nested connection multiplies too — this is what stops the 10000^3 query.
		cfg.Complexity.User.Orders = func(childComplexity int, first *int, after *string) int {
			n := 20
			if first != nil {
				n = *first
			}
			return n * childComplexity
		}
		// A plain scalar field has the default cost of 1 (no function needed).
	}
```

```go
	// In main.go — set the per-operation complexity budget.
	cfg := graph.Config{Resolvers: resolver}
	gqlsec.ConfigureComplexity(&cfg)
	es := graph.NewExecutableSchema(cfg)
	srv := handler.New(es)
	// Reject any operation whose computed complexity exceeds the budget.
	srv.Use(extension.FixedComplexityLimit(300)) // tune; start conservative
```

**How to choose the budget:** instrument real client queries, take the 99th-percentile complexity, set the limit modestly above it. Log rejected operations so you catch legitimate queries you accidentally blocked and tune. The combination *depth limit + complexity limit + max page size (§7)* is the core anti-DoS triad — all three, always.

### 10.4 Rate limiting **[A]**

Complexity bounds a *single* operation; rate limiting bounds the *frequency* of operations from one client. Limit per-IP and (better) per-authenticated-principal. A token-bucket in **Redis** (so the limit is shared across instances) is the standard. Apply it in HTTP middleware *before* GraphQL, and optionally a stricter per-operation budget (mutations more limited than queries; subscriptions capped per connection).

```go
	// internal/gqlsec/ratelimit.go — Redis token bucket, shared across instances.
	package gqlsec

	import (
		"context"
		"net/http"
		"time"

		"github.com/redis/go-redis/v9"
	)

	type Limiter struct {
		rdb   *redis.Client
		limit int           // requests
		win   time.Duration // per window
	}

	// Allow uses an atomic INCR+EXPIRE: first hit in the window sets the TTL.
	func (l *Limiter) Allow(ctx context.Context, key string) (bool, error) {
		pipe := l.rdb.TxPipeline()
		incr := pipe.Incr(ctx, key)
		pipe.Expire(ctx, key, l.win) // idempotent enough for a fixed window
		if _, err := pipe.Exec(ctx); err != nil {
			return false, err
		}
		return incr.Val() <= int64(l.limit), nil
	}

	func (l *Limiter) Middleware(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// Prefer the authenticated principal; fall back to IP.
			key := "rl:" + clientKey(r) // e.g. "rl:user:42" or "rl:ip:1.2.3.4"
			ok, err := l.Allow(r.Context(), key)
			if err == nil && !ok {
				w.Header().Set("Retry-After", "1")
				http.Error(w, `{"errors":[{"message":"rate limited","extensions":{"code":"RATE_LIMITED"}}]}`,
					http.StatusTooManyRequests)
				return
			}
			next.ServeHTTP(w, r)
		})
	}
```

⚡ A sliding-window or GCRA algorithm is more accurate than the fixed-window shown; libraries like `go-redis/redis_rate` implement GCRA in one call. For banking-grade APIs, also rate-limit *expensive* operations by *cost* (deduct an operation's complexity from a budget), not just by count.

### 10.5 Disabling introspection in production **[A]**

**Introspection** lets anyone query your full schema (`__schema`, `__type`) — every type, field, argument, and deprecation. In development it powers Playground and codegen; **in production it hands attackers a complete map of your attack surface.** Disable it in prod. With gqlgen you simply *do not* register the `Introspection` extension (it is off unless you add it). Also disable the Playground and any "did you mean" suggestions (§10.7).

```go
	if os.Getenv("ENV") != "production" {
		srv.Use(extension.Introspection{}) // dev only
		mux.Handle("/", playground.Handler("GraphQL", "/query"))
	}
	// In production: neither line runs. __schema/__type queries are rejected.
```

**Nuance:** disabling introspection is *defense in depth*, not a real authz boundary — a determined attacker can still probe field names. The real protections are authz (§9) and the limits above. But removing introspection raises the cost of reconnaissance and is expected in a banking-grade posture. Note: if you use **persisted queries / allow-listing (§10.12)**, introspection is moot anyway because only pre-registered queries run.

### 10.6 Alias and batch amplification **[A]**

Two amplification tricks bypass naive per-field thinking:

- **Aliases.** A client can request the *same* expensive field many times under different aliases in *one* operation: `a1: expensive b1: expensive c1: expensive ...` thousands of times. Each alias is a separate execution. **Defense:** complexity analysis (§10.3) counts aliased fields separately, so the budget catches it. You can also cap the number of aliases/root fields per operation.
- **Query batching.** Some transports accept an *array* of operations in one HTTP request (`[{query:...},{query:...},...]`). An attacker sends thousands. **Defense:** cap the batch size (or disable array-batching if you do not need it). gqlgen's POST transport can be configured; if you do not require batching, do not enable it.

```go
	// Cap root-field count per operation (cheap alias-amplification guard).
	func RootFieldLimit(max int) graphql.OperationMiddleware {
		return func(ctx context.Context, next graphql.OperationHandler) graphql.ResponseHandler {
			oc := graphql.GetOperationContext(ctx)
			if len(oc.Operation.SelectionSet) > max {
				return graphql.OneShot(graphql.ErrorResponse(ctx,
					"too many root fields (max %d)", max))
			}
			return next(ctx)
		}
	}
```

### 10.7 Error message hygiene & field-suggestion leakage **[A]**

GraphQL servers are *helpfully* chatty by default, which leaks information:

- **Raw errors** leak DB structure, table names, and internals (§4.7) — always sanitize via the `ErrorPresenter` and `RecoverFunc`.
- **Field suggestions.** When a query names a non-existent field, gqlparser may reply `Cannot query field "passwrod" on type "User". Did you mean "password"?` — confirming `password` exists. For untrusted clients, **strip "Did you mean" suggestions** from validation errors via the error presenter (rewrite or drop the suggestion text). With introspection disabled, suggestions are the *other* recon channel — close both.
- **Stack traces** must never reach the client (the `RecoverFunc` ensures this).

```go
	// In the ErrorPresenter, scrub suggestion text from validation errors.
	srv.SetErrorPresenter(func(ctx context.Context, e error) *gqlerror.Error {
		err := graphql.DefaultErrorPresenter(ctx, e)
		if strings.Contains(err.Message, "Did you mean") {
			err.Message = "Cannot query the requested field." // generic
		}
		// ...plus the internal-error sanitizing from §4.7...
		return err
	})
```

### 10.8 Existence leakage / IDOR **[A]**

A subtle but serious leak: returning **`FORBIDDEN`** for an object that exists-but-isn't-yours, while returning **`NOT_FOUND`/null** for one that doesn't exist, lets an attacker enumerate which ids exist (an Insecure Direct Object Reference oracle). For sensitive resources, **respond identically** whether the object is missing or merely not authorized — return `null` (treat-as-not-found) in both cases, as shown in §9.4. Reserve explicit `FORBIDDEN` for cases where revealing existence is acceptable (e.g. "this team exists but you must request access").

### 10.9 Injection — parameterized everything **[A]**

GraphQL does not add SQL injection risk *by itself* (args are typed values, not concatenated SQL) — but the moment a resolver builds SQL from a field argument, the classic rules apply. **Always use parameterized queries** (`$1`, `$2` with pgx — see §4.5/§7.5), never string concatenation. The same applies to building `LIKE` patterns, `ORDER BY` columns (whitelist allowed sort fields — never interpolate a client-supplied column name), and any custom-scalar `JSON` passed to a query. A `JSON` scalar is a frequent injection/abuse vector — validate its shape and never feed it raw into a query.

```go
	// SAFE: parameterized. The value is data, never code.
	rows, _ := pool.Query(ctx, `SELECT * FROM users WHERE email = $1`, arg)

	// For client-chosen sort: WHITELIST, never interpolate the raw string.
	col, ok := map[string]string{"created": "created_at", "name": "display_name"}[sortArg]
	if !ok {
		return nil, errUser(ctx, "BAD_SORT", "invalid sort field")
	}
	sql := `SELECT ... ORDER BY ` + col // col is from a fixed whitelist => safe
```

### 10.10 CSRF for mutations **[A]**

If your browser clients authenticate with **cookies**, a malicious site can forge an authenticated GraphQL mutation (CSRF). Three layers of defense: (1) prefer **bearer tokens in the `Authorization` header** over cookies — headers are not auto-sent cross-site, which eliminates classic CSRF; (2) if you must use cookies, require a **non-"simple" `Content-Type`** like `application/json` (a cross-site form cannot set it, and GraphQL needs it) and reject GraphQL over `GET` for mutations; (3) add an explicit **CSRF token** (double-submit cookie) for cookie auth. gqlgen's POST transport requires `application/json` by default, which already blocks the trivial form-based CSRF; do not relax it. Also set `SameSite=Lax`/`Strict` on auth cookies.

### 10.11 Authorization-bypass pitfalls **[A]**

The most common *breach* (not DoS) in GraphQL is a **forgotten authorization check on a field**. Because authz is per-field (§9), a single new field shipped without a guard exposes data. Defenses:

- **Deny-by-default policy.** Centralize authz (§9.5); make the *default* deny, so a field with no explicit allow returns nothing.
- **Authz tests per field.** A test matrix asserting "role X cannot read field Y" (§13). New fields without a test fail review.
- **Scope the query, not just the resolver.** `WHERE owner_id = $actor` means the DB enforces authz even if a code check is missed (§9.4).
- **Beware nested authz.** `Query.publicThing.privateChild` — the parent may be public but a child field private. Each field guards itself; do not assume the parent's guard covers children.
- **DataLoader + authz.** Never let a DataLoader bypass authz: load by id, *then* authorize the loaded object — the loader fetches, the policy decides.

### 10.12 Persisted queries & allow-listing (APQ) **[A]**

The strongest anti-abuse control for a **first-party** API (your own web/mobile apps): **do not accept arbitrary queries at all.** With **persisted queries / allow-listing**, the client sends a *hash* of a pre-registered query (and variables), and the server runs *only* queries on the allow-list. Benefits: (1) attackers cannot craft novel expensive/malicious queries — only your own queries run; (2) smaller payloads (a hash, not the full query) — see §11; (3) introspection/depth/complexity worries shrink because the query set is finite and known.

gqlgen's **Automatic Persisted Queries (APQ)** is the lighter cousin: the client first sends a hash; on a cache miss the server asks for the full query once, caches it by hash, and serves the hash thereafter. APQ is a *performance* feature by default (anyone can register a query), but you can make it a *security* control by switching to a **closed allow-list**: only queries you registered at build time are accepted; unknown hashes are rejected.

```go
	// Performance-mode APQ (anyone can register): smaller payloads, no security.
	srv.Use(extension.AutomaticPersistedQuery{Cache: lru.New[string](1000)})

	// Security-mode (allow-list): pre-load the cache with ONLY your app's
	// queries at startup and REJECT unknown hashes (custom cache that never
	// stores new entries). Then no arbitrary query ever executes.
```

**Recommendation:** for public third-party APIs you must accept arbitrary queries — rely on depth+complexity+rate limits (§10.2–10.4). For first-party apps, **allow-listing is the gold standard** and sidesteps most of this section. Banking-grade APIs that serve only their own front-ends should allow-list.

### 10.13 The production security checklist **[A]**

| Control | Status it must be in prod |
|---|---|
| Introspection | **OFF** |
| Playground | **OFF** (or auth-gated) |
| Depth limit | **ON** (≈10–15) |
| Complexity limit | **ON** (budgeted from real traffic) |
| Max page size | **ON** (server-clamped `first`) |
| Rate limiting | **ON** (per principal + per IP, Redis) |
| Query batching | **OFF** or capped |
| Alias/root-field cap | **ON** |
| Field suggestions | **STRIPPED** from errors |
| Error sanitization | **ON** (ErrorPresenter + RecoverFunc) |
| Existence leakage | **CLOSED** (uniform null) |
| Parameterized SQL | **ALWAYS** (+ sort whitelist) |
| CSRF | **HANDLED** (bearer tokens / JSON content-type / token) |
| AuthZ | **Per-field, deny-by-default, query-scoped, tested** |
| Persisted/allow-listed queries | **ON for first-party APIs** |
| Upload/body size cap | **ON** |
| Subscription auth | **ON** (InitFunc + per-op) |
| TLS everywhere | **ON** (see NETWORKING_GUIDE / NGINX_GUIDE) |

---

## 11. Performance & Scaling to Production

### 11.1 The performance hierarchy **[A]**

GraphQL performance work has a clear priority order — fix them top-down, because a lower item rarely matters if a higher one is broken:

1. **Eliminate N+1 with DataLoaders (§5).** This is 80% of GraphQL performance. Nothing else helps if every relation is a separate query.
2. **Bound the work per request (§10).** Depth/complexity/page limits stop pathological queries from dominating your capacity.
3. **Cache (§11.2).** Response cache, field cache (Redis), and `@cacheControl` cut repeated work.
4. **Tune the data layer (§11.4).** Connection pooling, indexes, query shape.
5. **Cut payload size (§11.3).** APQ/persisted GETs reduce bytes and enable CDN caching.
6. **Scale horizontally (§11.5).** Stateless instances + a subscription backplane.
7. **Profile to find the real bottleneck (§11.6).** Measure; don't guess.

### 11.2 Caching: response, field, and `@cacheControl` **[A]**

GraphQL's `POST`-by-default defeats HTTP caches, so caching happens *inside* the server. Three layers:

- **Per-request DataLoader cache** (§5) — already covered; dedupes within one request.
- **Cross-request field/object cache in Redis** — cache expensive resolver results by a stable key (`product:{id}`), with explicit invalidation on writes. This is the workhorse for read-heavy data. The danger is **stale or cross-tenant data**: key by tenant, set sane TTLs, and invalidate on the mutation that changes the data.
- **Whole-response cache** — gqlgen can cache full query results keyed by query+variables (+ the authenticated principal, or you leak data across users!). Only safe for *public, non-personalized* fields. **Never cache a response across users without including identity in the key** — this is a classic GraphQL data-leak.

The **`@cacheControl`** directive (Apollo convention) annotates fields with `maxAge` and `scope: PUBLIC|PRIVATE`, letting tooling/CDNs compute a cacheability hint for the whole response (the minimum maxAge across selected fields; `PRIVATE` if any field is private). It is most useful with persisted GETs behind a CDN (§11.3).

```go
	// Field/object cache pattern in the store: read-through Redis with TTL.
	func (s *Store) ProductByID(ctx context.Context, id string) (*Product, error) {
		key := "product:" + id
		if b, err := s.rdb.Get(ctx, key).Bytes(); err == nil {
			var p Product
			if json.Unmarshal(b, &p) == nil {
				return &p, nil // cache hit
			}
		}
		p, err := s.productFromDB(ctx, id) // miss: hit Postgres
		if err != nil {
			return nil, err
		}
		if b, err := json.Marshal(p); err == nil {
			s.rdb.Set(ctx, key, b, 5*time.Minute) // TTL bounds staleness
		}
		return p, nil
	}
	// On any mutation that changes the product: s.rdb.Del(ctx, "product:"+id).
```

### 11.3 Cutting payloads: APQ and persisted GETs + CDN **[A]**

Two wins from persisted queries (§10.12) beyond security:

- **Smaller request payloads.** Sending a 64-char hash instead of a multi-KB query shrinks every request — meaningful on mobile.
- **CDN-cacheable GETs.** A *persisted* query can be issued as `GET /query?extensions=...&variables=...` with a stable URL. Because it is a `GET` with a deterministic URL, a **CDN can cache the response** (for public data with `@cacheControl`), serving it without ever hitting your origin. This is how you get REST-grade edge caching back for the public, read-only slice of a GraphQL API. Mark such operations `PUBLIC` and route them through the GET transport.

### 11.4 Data-layer tuning **[A]**

- **Connection pooling.** Size the pgx pool deliberately: too small throttles concurrency, too large overwhelms Postgres (each connection is a backend process). A common heuristic is `pool_max ≈ (core_count * 2) + effective_spindle_count` per instance, capped so *total* connections across all instances stay within Postgres `max_connections` (use **PgBouncer** in transaction mode if instance count is high — see **[PostgreSQL](POSTGRESQL_GUIDE.md)**). Set `MaxConnLifetime`/`MaxConnIdleTime` to recycle connections.
- **Indexes for keyset pagination.** The keyset query (§7.5) needs a composite index matching the `ORDER BY`: `CREATE INDEX ON orders (customer_id, created_at DESC, id DESC)`. Without it, the "fast" pagination is a table scan.
- **Select only needed columns** when you can tell which fields were requested (gqlgen's `CollectFields`), avoiding fetching huge unrequested columns (a TEXT blob you didn't ask for).
- **Pass `ctx` to every query** so a client disconnect cancels in-flight DB work and frees the connection.

### 11.5 Horizontal scaling **[A]**

The GraphQL HTTP server is **stateless** — scale it horizontally behind a load balancer with no special handling. The two pieces of shared state that need coordination:

1. **Subscriptions** need a backplane (Redis Pub/Sub or Streams — §8.5) so events reach clients regardless of which instance holds their WebSocket. WebSocket connections are *sticky to an instance*; configure the LB for long-lived upgraded connections (idle timeout > keepalive interval) and ideally session affinity for the WS path.
2. **Rate-limit and cache state** live in Redis (§10.4, §11.2) so limits and caches are shared, not per-instance.

Everything else (resolvers, DataLoaders) is per-request and per-instance — no coordination needed. This statelessness is why GraphQL-over-HTTP scales as easily as a REST service.

### 11.6 Profiling **[A]**

Measure before optimizing. Tools:

- **gqlgen field timing / Apollo tracing extension** — per-field latency and call counts in the response extensions; find the slow/hot field.
- **`net/http/pprof`** — CPU and heap profiles of the Go process; find allocation hotspots (JSON marshaling, large result sets).
- **Postgres `EXPLAIN (ANALYZE, BUFFERS)`** and `pg_stat_statements` — find the slow SQL behind a slow field.
- **OpenTelemetry traces (§14)** — a span per resolver and per DB call, so one trace shows exactly where a request's time went across the whole graph.

A practical loop: load-test a representative query (k6, vegeta), read the trace to find the dominant span, fix it (usually a missing loader or index), repeat. Do not micro-optimize Go before you have killed N+1 and added the right indexes.

---

## 12. Schema Architecture at Scale — Modules & Federation

### 12.1 Modular schemas in one service **[A]**

Within a single service, **split your SDL across many files** by domain (`user.graphql`, `order.graphql`, `billing.graphql`) — gqlgen's `schema: [graph/schema/*.graphql]` glob stitches them at codegen. You can **extend** a type defined in another file with `extend type Query { ... }`, so each domain file owns its slice of the root types. Mirror this in Go: one `*.resolvers.go` per domain, a `store` sub-package per domain. This keeps a large schema navigable and lets teams own modules without merge conflicts on a monolithic file.

```graphql
# user.graphql
type Query { me: User }
type User { id: ID! email: String! }

# order.graphql — extends the SAME Query type, owns the order slice.
extend type Query { orders(first: Int, after: String): OrderConnection! }
type Order { id: ID! customer: User! }
```

### 12.2 Stitching vs. federation — the two ways to combine services **[A]**

When the graph spans *multiple services*, you compose them into one client-facing schema. Two approaches:

- **Schema stitching** (older) — a gateway merges remote schemas and you manually wire cross-service links with resolvers on the gateway. Flexible but the gateway holds glue logic, which becomes a maintenance burden. Largely superseded.
- **Federation** (Apollo Federation, the modern standard) — each service (a **subgraph**) owns part of the graph and *declares* how its types extend others via directives (`@key`, `@external`, `@requires`). A **gateway/router** composes the subgraphs automatically and plans cross-subgraph queries. The glue lives *in the subgraphs as declarations*, not as hand-written gateway code. This is how large orgs let many teams own slices of one supergraph.

### 12.3 Building a Go federated subgraph with gqlgen **[A]**

gqlgen has **first-class federation support**: enable it in `gqlgen.yml`, annotate your entities with `@key`, and gqlgen generates the `_entities`/`_service` resolvers the router needs. Your service becomes a subgraph the Apollo Router (or any federation-compatible gateway) can compose.

```yaml
# gqlgen.yml — turn on federation v2 codegen.
federation:
  filename: graph/federation.go
  package: graph
  version: 2
```

```graphql
# This subgraph OWNS User. @key says "User is identified by id" so other
# subgraphs can reference a User by id and the router stitches them.
type User @key(fields: "id") {
	id: ID!
	email: String!
}

# Another subgraph (e.g. "reviews") EXTENDS User it doesn't own:
extend type User @key(fields: "id") {
	id: ID! @external          # provided by the owning subgraph
	reviews: [Review!]!        # this subgraph contributes this field
}
```

gqlgen generates a resolver where the router asks this subgraph to *resolve an entity by its key* — you implement "given an id, return the `User`":

```go
	// graph/entity.resolvers.go (generated stub you implement) — the router
	// calls this to fetch entities of a type this subgraph owns, by @key.
	func (r *entityResolver) FindUserByID(ctx context.Context, id string) (*model.User, error) {
		return r.Store.UserByID(ctx, id) // batch this with a DataLoader at scale!
	}
```

**Federation gotcha:** the `FindXByID` entity resolver is a *prime N+1 source* — the router may call it once per referenced id. **Back it with a DataLoader** (§5) exactly like any other relation, or your federated graph will hammer the DB.

⚡ **Version note:** prefer **Federation v2** (`version: 2`) for new work; v1 is legacy. The directives (`@key`, `@shareable`, `@override`, `@requires`, `@external`) and their semantics differ between v1 and v2 — match your router's federation version. Confirm against the current Apollo Federation spec and gqlgen federation docs.

### 12.4 Schema evolution — deprecate, never version **[A]**

GraphQL's superpower for evolution: because clients request *specific fields*, you almost never need a `/v2`. Instead you evolve **additively**:

- **Add** fields, types, optional args, enum values freely — existing clients ignore what they don't select. (Non-breaking.)
- **Deprecate** a field/enum value with `@deprecated(reason: "use X")` — it keeps working, tooling flags it, and you watch usage drop to zero before removal.
- **Remove** only after telemetry shows no client selects it. (Removal is the only common breaking change.)

```graphql
type User {
	displayName: String!
	name: String! @deprecated(reason: "Renamed to displayName; removed 2026-12-01.")
}
```

**Breaking changes to avoid:** removing a field, making a nullable field non-null *as input had been optional*, changing a field's type, removing an enum value, adding a *required* argument. Run **schema-diff in CI** (e.g. a breaking-change checker over the SDL) to block accidental breaks — the same discipline as `buf breaking` for protobuf (see **[Go gRPC & RPC](GO_GRPC_RPC_GUIDE.md)**). Track per-field usage (operation/field metrics — §14) so you know when a deprecated field is safe to delete.

### 12.5 Code organization at scale **[A]**

| Layer | Owns | Rule |
|---|---|---|
| `graph/schema/*.graphql` | SDL per domain | One file per bounded context; `extend type Query`. |
| `graph/*.resolvers.go` | Thin adapters | No business logic; call store/policy/loaders. |
| `graph/loaders` | DataLoaders | One loader per batchable relation. |
| `graph/directives` | `@auth` etc. | Declarative cross-cutting guards. |
| `internal/store` | Data access (pgx) | **No GraphQL imports.** Reusable. |
| `internal/authz` | Policy | Single source of authorization truth. |
| `internal/pubsub` | Broker | In-mem + Redis backplane. |

The invariant that keeps a large GraphQL codebase sane: **resolvers are a translation layer, never a home for logic.** Domain logic lives in `internal/*`; resolvers just wire `ctx` → store/policy/loader → GraphQL model. This makes the whole thing testable, reusable across API styles, and resistant to the spaghetti that kills big GraphQL servers.

---

## 13. Testing

### 13.1 What to test, and at which level **[A]**

A GraphQL server has three testable layers; test each at the cheapest level that gives confidence:

| Layer | Test type | What it proves |
|---|---|---|
| Store (`internal/store`) | Unit / integration (real DB via Testcontainers) | SQL is correct, pagination/transactions work. |
| Policy (`internal/authz`) | Pure unit | Authorization rules are right. |
| Resolvers | Unit (mock store) | Translation, error mapping, null-on-not-found. |
| Whole server | Integration (gqlgen `client`) | End-to-end query → JSON, directives, limits. |

The two highest-value GraphQL-specific tests are: (1) **the integration test using gqlgen's test `client`**, which runs real queries through the real schema; and (2) **authz and complexity-limit tests**, because those are the controls that, if broken, become breaches or outages.

### 13.2 Integration tests with the gqlgen `client` **[A]**

gqlgen ships `client.New` + `handler.NewDefaultServer` (or your configured server) so you can run operations against the *actual* executable schema in-process — no network, fast. It binds the JSON result straight into a Go struct.

```go
	// graph/integration_test.go
	package graph_test

	import (
		"testing"

		"github.com/99designs/gqlgen/client"
		"github.com/99designs/gqlgen/graphql/handler"
		"github.com/stretchr/testify/require"

		"github.com/you/graphapi/graph"
	)

	func newTestClient(t *testing.T, store *fakeStore) *client.Client {
		es := graph.NewExecutableSchema(graph.Config{
			Resolvers: &graph.Resolver{Store: store},
		})
		srv := handler.New(es)
		// add the same transports/extensions as prod so tests exercise them
		return client.New(srv)
	}

	func TestQuery_User(t *testing.T) {
		c := newTestClient(t, seededStore())

		var resp struct {
			User struct {
				ID    string
				Email string
			}
		}
		// client.Post runs the operation and unmarshals data into resp.
		c.MustPost(`query($id: ID!){ user(id:$id){ id email } }`, &resp,
			client.Var("id", "u_1"))

		require.Equal(t, "u_1", resp.User.ID)
		require.Equal(t, "ada@example.com", resp.User.Email)
	}
```

### 13.3 Testing authorization **[A]**

The single most important security test matrix: for each protected field, assert that the wrong role/owner is denied and the right one allowed. Drive identity by injecting a principal into the context the test client uses.

```go
	func TestAuthz_OrderOwnership(t *testing.T) {
		c := newTestClient(t, storeWithOrder("o_1", "owner_42"))

		// As a DIFFERENT user: must get null (not the order, not a 'forbidden'
		// that leaks existence — see §10.8).
		var resp struct{ Order *struct{ ID string } }
		c.MustPost(`{ order(id:"o_1"){ id } }`, &resp,
			withPrincipal("intruder_99")) // sets ctx principal via AddContext
		require.Nil(t, resp.Order)

		// As the owner: must get the order.
		c.MustPost(`{ order(id:"o_1"){ id } }`, &resp, withPrincipal("owner_42"))
		require.NotNil(t, resp.Order)
	}
```

`withPrincipal` is a `client.Option` that wraps the request context (gqlgen's `client.AddContext`/an option that calls `auth.WithUser`). Build this helper once and reuse it across every authz test.

### 13.4 Testing the security limits **[A]**

Your depth/complexity limits are security controls — test that they actually reject. A test that a deeply-nested or high-complexity query returns the limit error guards against a future change silently removing the protection.

```go
	func TestDepthLimit_Rejects(t *testing.T) {
		c := newTestClientWithLimits(t) // server with DepthLimit(5) installed
		var resp any
		err := c.Post(`{ a{ b{ c{ d{ e{ f{ id } } } } } } }`, &resp) // depth 6
		require.Error(t, err)
		require.Contains(t, err.Error(), "maximum depth")
	}
```

Add a similar test asserting an over-budget complexity query is rejected, and a test that a missing DataLoader would blow the query count (intercept the store, count `UserByID` calls; assert it's 1 batched call, not N — this is your N+1 regression guard from §5.5).

### 13.5 Real-DB integration with Testcontainers **[A]**

For store-layer tests that must run against *real* Postgres (so SQL, keyset pagination, and transactions are genuinely exercised), use **Testcontainers** to spin up a throwaway Postgres in Docker per test run. This gives production-fidelity SQL testing without a shared test DB.

```go
	// internal/store/store_integration_test.go
	func TestStore_OrdersPage(t *testing.T) {
		ctx := context.Background()
		// Start a disposable Postgres container.
		pg, err := postgres.Run(ctx, "postgres:17-alpine",
			postgres.WithDatabase("test"),
			postgres.WithUsername("test"), postgres.WithPassword("test"),
			testcontainers.WithWaitStrategy(
				wait.ForListeningPort("5432/tcp")),
		)
		require.NoError(t, err)
		defer pg.Terminate(ctx)

		dsn, _ := pg.ConnectionString(ctx, "sslmode=disable")
		pool, _ := pgxpool.New(ctx, dsn)
		migrate(t, pool)          // apply schema
		s := store.New(pool)
		seedOrders(t, s, 50)      // insert known data

		// Now assert keyset pagination returns stable, ordered pages.
		page1, _ := s.OrdersPage(ctx, "owner", 20, nil)
		require.Len(t, page1.Orders, 20)
		require.True(t, page1.HasNext)
		// page 2 uses page1's last cursor; assert no overlap with page1.
	}
```

⚡ **Version note:** the Testcontainers Go API (`postgres.Run`, wait strategies) has evolved across versions — confirm the current `testcontainers-go` and `modules/postgres` signatures. On **Windows** you need Docker Desktop running; in CI, a Docker-in-Docker or service container.

### 13.6 Testing checklist **[A]**

| Test | Guards against |
|---|---|
| gqlgen `client` integration per query/mutation | Broken schema↔resolver wiring. |
| Authz matrix (wrong role/owner denied) | Authorization-bypass breaches (§10.11). |
| Existence-leak (not-yours == not-found) | IDOR oracle (§10.8). |
| Depth + complexity rejection | DoS protection silently removed (§10.2–3). |
| N+1 query-count assertion | DataLoader regressions (§5). |
| Store integration on real Postgres | Wrong SQL, broken pagination/transactions. |
| Policy unit tests | Subtle authorization rule errors. |
| Error-sanitization (no internals leak) | Information disclosure (§4.7, §10.7). |

---

## 14. Deployment & Observability

### 14.1 Dockerizing — small, non-root, distroless **[A]**

Ship a tiny, hardened image: a multi-stage build that compiles a static binary then copies it into a minimal, non-root base. (See **[Docker](DOCKER_GUIDE.md)** for the full treatment.)

```dockerfile
# ---- build stage ----
FROM golang:1.26-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
# CGO off => fully static binary; trims size & avoids libc surprises.
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app ./server

# ---- runtime stage ----
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /app /app
USER nonroot:nonroot        # never run as root
EXPOSE 8080
ENTRYPOINT ["/app"]
```

**Build-time codegen note:** run `go generate ./...` *before* the image build (in CI), or commit generated files. Do not run gqlgen inside the runtime image.

### 14.2 Behind Nginx **[A]**

Front the service with Nginx (or your cloud LB) for TLS termination, compression, and — critically for subscriptions — **WebSocket upgrade** support and a long read timeout. (Full config in **[Nginx](NGINX_GUIDE.md)**.)

```nginx
location /query {
    proxy_pass http://graphql_upstream;
    proxy_http_version 1.1;
    # WebSocket upgrade for subscriptions:
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade; # map to "upgrade"/"" 
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    # Idle timeout MUST exceed the WS keepalive ping interval (§8.5),
    # or the proxy kills live subscriptions.
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
    # Cap request body to blunt giant-payload DoS (matches §10).
    client_max_body_size 1m;
}
```

### 14.3 Health & readiness endpoints **[A]**

Expose **liveness** (process is up) and **readiness** (dependencies — DB, Redis — are reachable) as plain HTTP endpoints, *outside* GraphQL, so orchestrators (Kubernetes) and LBs can probe without running a query.

```go
	mux.HandleFunc("/healthz", func(w http.ResponseWriter, _ *http.Request) {
		w.WriteHeader(http.StatusOK) // liveness: am I running?
	})
	mux.HandleFunc("/readyz", func(w http.ResponseWriter, r *http.Request) {
		ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
		defer cancel()
		if err := pool.Ping(ctx); err != nil { // readiness: can I serve?
			http.Error(w, "db down", http.StatusServiceUnavailable)
			return
		}
		w.WriteHeader(http.StatusOK)
	})
```

### 14.4 Observability — tracing, metrics, slow-query logs **[A]**

You cannot operate a GraphQL server blind — a slow "query" could be any of its dozens of fields. Three pillars:

- **Distributed tracing (OpenTelemetry).** Instrument with a span per operation, per resolver, and per DB call. gqlgen has an OTel/Apollo-tracing extension; pgx has a tracer. One trace then shows *exactly* which field and which SQL dominated a slow request — indispensable for a graph where time is spread across many resolvers.
- **Metrics (Prometheus).** Per-operation latency histograms, error rates by `code`, complexity scores, rejected-by-limit counts, active subscription count, DataLoader batch sizes. Alert on p99 latency, error-rate spikes, and limit-rejection spikes (a rejection spike is either an attack or a client you broke).
- **Structured slow-query logging.** Log operations exceeding a latency threshold with their (sanitized) query, complexity, and field timings — your forensic trail for both performance and abuse.

```go
	// An AroundOperations middleware: time the whole op, log slow ones, count.
	srv.AroundOperations(func(ctx context.Context, next graphql.OperationHandler) graphql.ResponseHandler {
		oc := graphql.GetOperationContext(ctx)
		start := time.Now()
		resp := next(ctx)
		return func(ctx context.Context) *graphql.Response {
			r := resp(ctx)
			dur := time.Since(start)
			opMetrics.Observe(oc.OperationName, dur) // Prometheus histogram
			if dur > 500*time.Millisecond {
				slog.WarnContext(ctx, "slow operation",
					"op", oc.OperationName, "ms", dur.Milliseconds())
			}
			return r
		}
	})
```

⚡ **Field-level tracing has overhead.** A span per field is costly under high QPS — sample it (trace 1–5% of operations) in production rather than 100%.

### 14.5 Graceful shutdown **[A]**

On SIGTERM (a deploy/scale-down), stop accepting new work, let in-flight operations finish, close subscriptions cleanly, and release the pool — so no request is dropped mid-flight and no connection leaks. The skeleton from §3.5, completed:

```go
	stop := make(chan os.Signal, 1)
	signal.Notify(stop, os.Interrupt, syscall.SIGTERM)
	<-stop

	// 1) Flip readiness to NOT READY so the LB drains us (see /readyz).
	atomic.StoreInt32(&ready, 0)

	// 2) Stop accepting new HTTP; wait up to 20s for in-flight to finish.
	shCtx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
	defer cancel()
	if err := httpSrv.Shutdown(shCtx); err != nil {
		slog.Error("forced shutdown", "err", err)
	}

	// 3) Close subscriptions (broker should signal ctx.Done to each stream),
	//    then close the DB pool and Redis.
	broker.CloseAll()
	pool.Close()
	_ = rdb.Close()
```

### 14.6 Deployment checklist **[A]**

| Item | Requirement |
|---|---|
| Image | Multi-stage, distroless, non-root, static binary. |
| Codegen | Run in CI, not at runtime; committed or built before image. |
| TLS | Terminated at LB/Nginx; HSTS on. |
| WebSockets | LB idle timeout > keepalive; upgrade headers set. |
| Body size | Capped at LB and in gqlgen transport. |
| Health/Ready | `/healthz` + `/readyz` (DB+Redis check). |
| Tracing | OTel, sampled in prod. |
| Metrics | Latency, error-by-code, complexity, limit-rejections, subs. |
| Slow logs | Sanitized, above a threshold. |
| Shutdown | Graceful drain on SIGTERM. |
| Secrets | DB/JWT keys from env/secret store, never baked into the image. |

---

## 15. Gotchas & Best Practices

A consolidated field guide to the mistakes that bite real GraphQL-in-Go teams. Each is a hard-won lesson; the fix column is the one-line takeaway.

### 15.1 Correctness & performance gotchas **[I/A]**

| Gotcha | Why it hurts | Fix |
|---|---|---|
| **No DataLoader** | Every relation = N+1 queries; the DB melts under nested queries. | Batch+cache per request with a DataLoader (§5). The #1 mistake. |
| **DataLoader created globally** | Stale data; cross-user data leaks across requests. | Build loaders per-request in middleware (§5.2). |
| **Nullable everything** | Clients can't trust any field; null-checks everywhere. | Use `!` deliberately; `[T!]!` for lists; nullable only when truly absent (§2.4). |
| **Non-null everything** | One failing field nulls a huge subtree via propagation. | Mark non-null only when *always* producible (§2.4). |
| **`Int` for 64-bit ids/money** | Silent overflow at 2³¹. | Custom scalar / `Int64` / `String` for big numbers (§2.2). |
| **`Float` for money** | Rounding errors; lost cents. | Integer minor units (cents) as a `Money` scalar (§2.11). |
| **No pagination on a list** | Unbounded result = OOM/DoS. | Relay cursor connection + server-clamped `first` (§7). |
| **Offset pagination on big/changing tables** | Slow deep pages; skips/dupes under writes. | Keyset/cursor pagination (§7.3). |
| **Business logic in resolvers** | Untestable, unreusable, spaghetti. | Resolvers are thin adapters; logic in `internal/*` (§4.8, §12.5). |
| **`totalCount` eagerly** | A `COUNT(*)` on every page; defeats keyset speed. | Resolve lazily or estimate; consider dropping it (§7.6). |
| **Goroutine using request `ctx` for background work** | Cancelled when the request ends. | Derive a fresh context for background jobs (§4.6). |
| **Missing index for keyset** | "Fast" pagination is a table scan. | Composite index matching `ORDER BY` (§11.4). |

### 15.2 Security gotchas **[A]**

| Gotcha | Why it hurts | Fix |
|---|---|---|
| **Introspection on in prod** | Hands attackers the full schema map. | Disable; don't register the Introspection extension (§10.5). |
| **No depth limit** | Cyclic schema → arbitrarily deep, expensive queries. | `AroundOperations` depth guard (§10.2). |
| **No complexity limit** | Shallow-but-wide queries melt the DB. | gqlgen complexity budget + per-list cost (§10.3). |
| **Leaking raw errors** | DB/table names, internals exposed to clients. | `ErrorPresenter` + `RecoverFunc` sanitize (§4.7, §10.7). |
| **Field suggestions** | "Did you mean `password`?" confirms fields exist. | Strip suggestions from validation errors (§10.7). |
| **Existence leak (FORBIDDEN vs NOT FOUND)** | IDOR oracle: enumerate which ids exist. | Uniform null for not-yours and not-found (§10.8). |
| **Forgotten field authz** | One unguarded field = data breach. | Deny-by-default policy; per-field authz tests; scope the SQL (§9, §10.11). |
| **Caching responses across users** | One user's data served to another. | Include identity in cache keys; only cache public fields (§11.2). |
| **No rate limit** | Query flooding / amplification. | Redis token bucket per principal+IP (§10.4). |
| **Alias/batch amplification** | One request → thousands of executions. | Complexity counts aliases; cap batch & root fields (§10.6). |
| **String-built SQL from args** | SQL injection. | Parameterized queries; whitelist sort columns (§10.9). |
| **Cookie auth without CSRF defense** | Forged cross-site mutations. | Bearer tokens / JSON content-type / CSRF token (§10.10). |
| **Federation entity resolver without loader** | Router calls it per id → N+1 across the supergraph. | DataLoader-back `FindXByID` (§12.3). |
| **Subscription auth only at connect** | Token expires but stream stays open. | Re-check authz over the stream's lifetime (§8.6). |

### 15.3 Operational gotchas **[A]**

| Gotcha | Why it hurts | Fix |
|---|---|---|
| **In-memory pub-sub with >1 instance** | Events miss clients on other instances. | Redis backplane (§8.5). |
| **LB idle timeout < WS keepalive** | Subscriptions silently dropped. | Set proxy read timeout above keepalive (§14.2). |
| **Hand-editing generated files** | Overwritten on next `go generate`. | Only edit `*.resolvers.go` and your own files (§3.1). |
| **Versioning the endpoint (`/v2`)** | Loses GraphQL's evolution model. | Evolve additively; `@deprecated` (§12.4). |
| **100% field-level tracing in prod** | Tracing overhead dwarfs the work. | Sample 1–5% (§14.4). |
| **No graceful shutdown** | Dropped requests / leaked connections on deploy. | Drain on SIGTERM (§14.5). |
| **Pool too large/small** | DB overload or throttled concurrency. | Size deliberately; PgBouncer at scale (§11.4). |

### 15.4 The ten commandments (memorize) **[A]**

1. **Thou shalt DataLoader** every relation.
2. **Thou shalt limit** depth, complexity, and page size — all three.
3. **Thou shalt authorize** per field, deny-by-default, and scope the SQL.
4. **Thou shalt never leak** raw errors, suggestions, or existence.
5. **Thou shalt rate-limit** by principal and by cost.
6. **Thou shalt disable introspection** (and allow-list queries for first-party apps).
7. **Thou shalt paginate** with cursors and clamp `first` server-side.
8. **Thou shalt keep resolvers thin** — logic lives in the store/policy.
9. **Thou shalt make mutations** idempotent, transactional, and payload-wrapped.
10. **Thou shalt measure** — trace, count queries, prove N+1 is dead.

---

## 16. Study Path & Build-to-Learn Projects

The fastest way to internalize GraphQL-in-Go is to build **one** server and harden it stage by stage. Each stage maps to sections above; do them in order — later stages assume earlier ones.

### Stage 0 — Prerequisites (½ day)

Confirm you can read Go: `context`, goroutines/channels, interfaces, errors, `database/sql`/pgx. If shaky, read **[Go (Golang)](GO_GUIDE.md)** and **[Go — Language & Patterns](GO_LANG_AND_PATTERNS_GUIDE.md)**. Stand up Postgres and Redis locally (Docker — **[Docker](DOCKER_GUIDE.md)**).

### Stage 1 — Schema & first resolvers (Week 1)

1. **Design the schema** for a small e-commerce domain: `User`, `Product`, `Order`, `OrderItem`, with the `Node` interface and custom `Time`/`UUID` scalars (§2). Get nullability right.
2. **Bootstrap gqlgen** (`init`, `gqlgen.yml`, `go generate`); wire it into `net/http` with a Playground (§3).
3. **Implement query resolvers** backed by a `pgx` store, with the store/resolver separation (§4). Map rows with `RowToStructByName`.
4. **Verify** in Playground: fetch a user, their orders, each order's items.

### Stage 2 — Kill N+1 (Week 1–2)

5. **Reproduce N+1**: query 100 orders + each `customer`; log SQL and watch 101 queries.
6. **Add DataLoaders** (graph-gophers v7) for `User`, `Product`; wire per-request middleware (§5). Re-run: 2 queries.
7. **Write the N+1 regression test** (assert query count) (§5.5, §13.4).

### Stage 3 — Mutations, validation, pagination (Week 2)

8. **`createOrder`** with the payload pattern, input validation, a transaction, and an idempotency key (§6).
9. **Relay cursor pagination** for `orders` and `products`: keyset SQL, opaque cursors, `pageInfo`, server-clamped `first` (§7).
10. **Test** mutations (success + `userErrors`) and pagination stability with the gqlgen `client` (§13).

### Stage 4 — Auth & authorization (Week 3)

11. **HTTP auth middleware** verifying a JWT into the context (reuse **[Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md)**) (§9.1).
12. **`@auth`/`@hasRole` directives** plus **object-level ownership** checks, with a centralized `internal/authz` policy (§9).
13. **Authz test matrix**: wrong owner → null (no existence leak), right owner → data; admin override (§13.3).

### Stage 5 — Realtime subscriptions (Week 3–4)

14. **`orderStatusChanged` subscription** over `graphql-transport-ws`; in-memory broker; publish from the status-change mutation (§8).
15. **Authenticate the WS** via `connection_init`/`InitFunc`; authorize per subscription (§8.6).
16. **Redis backplane**: run two instances; prove a status change on one reaches a subscriber on the other (§8.5).

### Stage 6 — Banking-grade hardening (Week 4–5)

17. **Depth limit + complexity limit + max page size** — the anti-DoS triad; tests that they reject (§10.2–10.3, §13.4).
18. **Rate limiting** (Redis token bucket), **error sanitization**, **strip field suggestions**, **disable introspection/Playground in prod**, **uniform existence responses** (§10).
19. **Persisted/allow-listed queries (APQ)** for the first-party client; measure payload shrinkage (§10.12, §11.3).
20. **Security review** against the §10.13 checklist; run a load test (k6) of a nasty nested query and confirm it's rejected, not served.

### Stage 7 — Scale, federate, ship (Week 5–6)

21. **Caching**: Redis read-through for `Product`; `@cacheControl`; persisted GET behind a CDN for public reads (§11).
22. **Federation**: split `reviews` into a second subgraph; compose with a router; back the entity resolver with a DataLoader (§12.3).
23. **Observability**: OTel tracing (sampled), Prometheus metrics (latency, error-by-code, complexity, limit-rejections, subs), slow-query logs (§14.4).
24. **Deploy**: distroless non-root image, behind Nginx (TLS + WS), `/healthz` + `/readyz`, graceful shutdown; schema-diff in CI to block breaking changes (§12.4, §14).

### Capstone

Combine everything into **one production GraphQL API**: schema → resolvers + DataLoaders → mutations (idempotent, validated, transactional) → cursor pagination → JWT auth + per-field authz → realtime subscriptions on a Redis backplane → the full security triad + rate limiting + allow-listed queries → tracing/metrics → load-tested and Dockerized behind Nginx. If your load test can't DoS it and your authz matrix is green, you have built a banking-grade GraphQL server.

### Reference bookmarks

| Resource | URL |
|---|---|
| gqlgen docs | https://gqlgen.com |
| gqlgen GitHub | https://github.com/99designs/gqlgen |
| GraphQL spec & learn | https://graphql.org |
| graph-gophers/dataloader | https://github.com/graph-gophers/dataloader |
| graphql-ws (protocol) | https://github.com/enisdenjo/graphql-ws |
| Apollo Federation | https://www.apollographql.com/docs/federation/ |
| OWASP GraphQL Cheat Sheet | https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html |
| pgx (Postgres driver) | https://github.com/jackc/pgx |
| Relay connections spec | https://relay.dev/graphql/connections.htm |

### Related guides in this library

| Guide | Why read it alongside this one |
|---|---|
| [Go (Golang)](GO_GUIDE.md) | The language itself — context, goroutines, channels, interfaces, errors. |
| [Go — Language & Patterns](GO_LANG_AND_PATTERNS_GUIDE.md) | Idioms behind the store/resolver/policy structure. |
| [Go net/http REST API](GO_NET_HTTP_REST_API_GUIDE.md) | The REST alternative; REST-vs-GraphQL trade-offs (§1.4). |
| [Go Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) | Mounting gqlgen on Gin; file uploads. |
| [Go gRPC & RPC](GO_GRPC_RPC_GUIDE.md) | The third API style — compare GraphQL vs gRPC vs REST (§1.4). |
| [Go ent ORM](GO_ENT_ORM_GUIDE.md) | An ORM behind resolvers (has a gqlgen integration). |
| [PostgreSQL](POSTGRESQL_GUIDE.md) | The DB behind the store; indexes, keyset pagination, pooling. |
| [Redis](REDIS_GUIDE.md) | Caching, rate limiting, the subscription backplane. |
| [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md) | Auth primitives behind §9–§10. |
| [NestJS GraphQL](NESTJS_GRAPHQL_GUIDE.md) | The same ideas in TypeScript — compare approaches. |
| [Docker](DOCKER_GUIDE.md) | Containerizing the server (§14). |
| [Nginx](NGINX_GUIDE.md) | TLS + WebSocket reverse proxy (§14.2). |
| [Networking](NETWORKING_GUIDE.md) | HTTP/WS/TLS fundamentals under the transport. |

---

*Guide accurate as of Go 1.25/1.26, gqlgen v0.17.x, pgx v5, graph-gophers/dataloader v7, and Apollo Federation v2 (June 2026). The gqlgen API (`gqlgen.yml` keys, generated-code layout, handler/transport/extension packages), the dataloader generics API, and the federation directives evolve between releases — always cross-reference gqlgen.com, pkg.go.dev, graphql.org, and the Apollo Federation docs for the latest before copying older snippets.*
