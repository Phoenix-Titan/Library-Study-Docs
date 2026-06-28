# GraphQL with NestJS — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Backend developers who already know NestJS fundamentals — modules, controllers, providers, dependency injection, pipes, guards, interceptors, the request lifecycle — and now want to build a **production-scale GraphQL API** with it. If those NestJS basics are shaky, read the [NestJS](NESTJS_GUIDE.md) guide first; this guide assumes them and focuses entirely on the GraphQL layer that sits on top. Every concept is explained in prose first — *what it is*, *why it works this way*, *when you reach for it*, *how to use it*, the *key decorators / options*, *best practices*, and *security implications* — and only then the heavily-commented, runnable TypeScript. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **NestJS 11**, **`@nestjs/graphql` v13** with the **`@nestjs/apollo` driver over Apollo Server v4/v5**, **code-first** as the primary approach (schema-first is covered too), running on **Node.js 24 LTS** and **TypeScript 5.x**. Persistence uses **Prisma 6** (TypeORM noted as an alternative). Batching uses **DataLoader**; subscriptions use the **`graphql-ws`** transport with **Redis** as the cross-instance backplane. Where an API moves fast or changed recently it is flagged with **⚡ Version note**. The author is on **Windows 11**, so OS-specific notes are called out where they matter.
>
> **Cross-references:** This guide builds directly on [NestJS](NESTJS_GUIDE.md). For the database layer it leans on [Prisma ORM](PRISMA_ORM_GUIDE.md) and [PostgreSQL](POSTGRESQL_GUIDE.md); [Redis](REDIS_GUIDE.md) powers caching and the subscription backplane. For authentication and security concepts (JWT lifecycle, password hashing, token threat models) see [Go JWT + Argon2 (Banking-Grade Auth)](GO_JWT_ARGON2_GUIDE.md) and [Better Auth](BETTERAUTH_GUIDE.md). The subscription transport is WebSockets — see [Node WebSockets](NODE_WEBSOCKETS_GUIDE.md). For deployment behind a reverse proxy and networking fundamentals see [Docker](DOCKER_GUIDE.md) and [Networking](NETWORKING_GUIDE.md). To contrast the resolver-and-loader model with a statically-typed, schema-first Go implementation, see [Go GraphQL](GO_GRAPHQL_GUIDE.md). Cross-links appear inline throughout.

---

## Table of Contents

1. [What GraphQL Is, Why It Exists & How NestJS Implements It](#1-what-graphql-is-why-it-exists--how-nestjs-implements-it) **[B]**
2. [Setup: GraphQLModule, the Apollo Driver & Project Structure](#2-setup-graphqlmodule-the-apollo-driver--project-structure) **[B]**
3. [Types with Decorators (Code-First)](#3-types-with-decorators-code-first) **[B/I]**
4. [Resolvers, Queries, Mutations & Field Resolvers](#4-resolvers-queries-mutations--field-resolvers) **[I/A]**
5. [Validation & Error Handling](#5-validation--error-handling) **[I/A]**
6. [The N+1 Problem & DataLoader](#6-the-n1-problem--dataloader) **[A]**
7. [Database Integration with Prisma](#7-database-integration-with-prisma) **[I/A]**
8. [Pagination: Offset & Cursor/Relay Connections](#8-pagination-offset--cursorrelay-connections) **[I/A]**
9. [Subscriptions: Realtime over graphql-ws & Redis](#9-subscriptions-realtime-over-graphql-ws--redis) **[A]**
10. [Authentication & Authorization](#10-authentication--authorization) **[A]**
11. [Banking-Grade GraphQL Security](#11-banking-grade-graphql-security) **[A]**
12. [Performance & Scaling](#12-performance--scaling) **[A]**
13. [Schema Architecture at Scale & Federation](#13-schema-architecture-at-scale--federation) **[A]**
14. [Testing GraphQL APIs](#14-testing-graphql-apis) **[A]**
15. [Deployment & Observability](#15-deployment--observability) **[A]**
16. [Gotchas & Best Practices](#16-gotchas--best-practices) **[I/A]**
17. [Study Path & Build-to-Learn Projects](#17-study-path--build-to-learn-projects)

---

## 1. What GraphQL Is, Why It Exists & How NestJS Implements It

### What GraphQL is

GraphQL is a **query language for APIs and a runtime for executing those queries against your data**. Instead of exposing dozens of fixed REST endpoints — `/users`, `/users/:id`, `/users/:id/posts`, `/posts/:id/comments` — a GraphQL server exposes **one endpoint** (conventionally `POST /graphql`) and a **schema** that describes every type, field, and operation the API supports. The *client* then sends a query describing exactly the shape of the data it wants, and the server returns exactly that shape — no more, no less.

Three operation types exist:

- **Query** — read data (the GraphQL equivalent of HTTP `GET`). Queries are expected to be side-effect-free.
- **Mutation** — write data (create/update/delete; the equivalent of `POST`/`PUT`/`PATCH`/`DELETE`). Mutations run sequentially in the order written, so a client can rely on ordering.
- **Subscription** — a long-lived stream of data pushed from server to client over a persistent connection (WebSocket), for realtime updates.

A representative query and its response:

```graphql
# The client asks for precisely these fields and nothing else.
query GetUserWithPosts {
  user(id: "42") {
    id
    name
    posts(first: 3) {        # nested — one round trip fetches the graph
      edges {
        node { id title }
      }
    }
  }
}
```

```json
{
  "data": {
    "user": {
      "id": "42",
      "name": "Ada Lovelace",
      "posts": { "edges": [ { "node": { "id": "1", "title": "Notes" } } ] }
    }
  }
}
```

### Why GraphQL exists — the problems it solves vs REST

GraphQL is a reaction to concrete pain points that REST APIs accumulate as clients (especially mobile and rich web UIs) grow more demanding:

- **Over-fetching.** A REST `GET /users/42` returns the whole user object even if the screen only needs the name and avatar. Bytes and battery are wasted. GraphQL returns only requested fields.
- **Under-fetching / the N+1 round trips on the client.** To render a profile page you might call `/users/42`, then `/users/42/posts`, then `/posts/:id/comments` for each post — a waterfall of HTTP requests. GraphQL fetches the whole graph in **one request**.
- **Endpoint sprawl & versioning.** REST grows endpoints and `/v2/` versions over time. GraphQL evolves a single schema additively: add fields, deprecate old ones with `@deprecated`, rarely version the whole API.
- **A strongly-typed contract & introspection.** The schema is machine-readable. Tooling (autocomplete, type generation, the Apollo Sandbox) is derived from it automatically. The client and server share one source of truth.

GraphQL is **not** a silver bullet. It moves complexity from the client to the server: now *your* resolvers must avoid the N+1 problem (§6), and you must defend against malicious queries that REST's fixed endpoints made impossible — deeply nested queries, expensive field combinations, alias-based amplification (§11). REST is still the better choice for simple CRUD services, file transfer, public cache-friendly APIs (HTTP caching is trivial with REST, harder with a single POST endpoint), and webhooks.

| Concern | REST | GraphQL |
|---------|------|---------|
| Endpoints | Many, resource-oriented | One (`/graphql`) |
| Response shape | Fixed by server | Chosen by client |
| Over/under-fetching | Common | Eliminated by design |
| Typed contract | Optional (OpenAPI) | Built-in schema + introspection |
| HTTP caching | Trivial (`GET` + ETag) | Hard (POST); needs app-level or APQ |
| Versioning | `/v2/`, new endpoints | Additive schema evolution + `@deprecated` |
| Main new risk | — | Query-cost / depth DoS, N+1 in resolvers |
| Realtime | SSE / WebSockets bolted on | Subscriptions are first-class |

> **Compare the Go approach:** In Go (see [Go GraphQL](GO_GRAPHQL_GUIDE.md)) the dominant library is **gqlgen**, which is **schema-first with code generation**: you write the SDL, run a generator, and implement the generated resolver interfaces. NestJS's *code-first* model inverts that — you write TypeScript classes and the SDL is generated *from* them. Both end at the same place (a typed schema bound to resolver functions); they differ in which artifact is the source of truth. The N+1/DataLoader, depth-limiting, and complexity concerns are identical across both ecosystems.

### How NestJS implements GraphQL

NestJS does **not** reimplement GraphQL. It wraps a mature GraphQL server implementation behind a Nest module and adapts GraphQL's execution model onto Nest's familiar building blocks. The pieces:

- **`@nestjs/graphql`** — the framework-agnostic integration layer: the `GraphQLModule`, the decorators (`@Resolver`, `@Query`, `@Mutation`, `@ObjectType`, `@Field`, `@Args`, `@ResolveField`…), and the **schema-building engine** that turns your decorated classes into a GraphQL schema (the code-first magic).
- **A driver** — the concrete server. The mainstream choice is **`@nestjs/apollo`**, which plugs in **Apollo Server v4/v5**. An alternative driver, **`@nestjs/mercurius`**, runs on the Fastify-native Mercurius server. This guide uses Apollo.
- **Your resolvers** — ordinary Nest providers (classes registered in a module, eligible for dependency injection) decorated with GraphQL decorators. A resolver method is to GraphQL what a controller route handler is to REST.

The crucial mental model: **a resolver class is a provider, a resolver method is a request handler, and the GraphQL field arguments are the route parameters.** Everything you know about Nest DI, guards, pipes, interceptors, and filters still applies — but it applies *per resolver field*, not per HTTP route.

### The request lifecycle through Nest's GraphQL layer

When a GraphQL request arrives at `POST /graphql`, it flows through both the GraphQL execution engine and Nest's enhancer pipeline:

```
        HTTP POST /graphql  (body: { query, variables, operationName })
                               │
                               ▼
   ┌──────────────────────────────────────────────────────────────┐
   │ Apollo Server: parse query → validate against schema →        │
   │   build the GraphQL "context" (Nest calls your context fn)    │
   └───────────────────────────┬──────────────────────────────────┘
                               ▼  for each field to resolve:
   ┌──────────────────────────────────────────────────────────────┐
   │  GUARDS        — authn/authz (GqlAuthGuard, RolesGuard)        │
   │  INTERCEPTORS  — wrap (logging, caching, transform)  [before]  │
   │  PIPES         — validate/transform @Args (ValidationPipe)     │
   │  ── RESOLVER METHOD RUNS (your code, services, Prisma) ──      │
   │  INTERCEPTORS  — [after]                                        │
   │  FILTERS       — catch exceptions → GraphQL error formatting    │
   └───────────────────────────┬──────────────────────────────────┘
                               ▼
                   data assembled into the response shape
                   (field resolvers run lazily as the tree is walked)
```

The single most important difference from REST: **the resolver pipeline runs once per resolved field, not once per request.** A query that returns 100 users, each with a `posts` field resolver, runs that field resolver 100 times. This is exactly why the N+1 problem (§6) is the defining performance concern of GraphQL, and why guards/pipes that do expensive work per field need care.

### Code-first vs schema-first — and why code-first suits Nest

There are two ways to define a GraphQL schema in NestJS:

**Code-first** (the primary approach in this guide). You write **TypeScript classes decorated** with `@ObjectType()`, `@Field()`, etc. NestJS reflects over those decorators at startup and **generates the `.gql` schema file for you**. The TypeScript types are the single source of truth.

- ✅ One source of truth (the classes); no drift between TS types and SDL.
- ✅ Full IDE support, refactoring, and type-safety end to end.
- ✅ Reuse classes as DTOs/entities; integrates with `class-validator`.
- ⚠️ Schema is an *output*; you read generated SDL rather than authoring it directly.

**Schema-first.** You author the **SDL** (`.graphql` files) by hand, and write resolvers that NestJS matches to it (optionally generating TS typings from the SDL with `ts-morph`).

- ✅ The schema is the explicit, reviewed contract — friendly to non-TS consumers and API-design-first teams.
- ⚠️ Two sources of truth to keep in sync (SDL + resolver signatures); more boilerplate.

**Why code-first suits Nest:** Nest is decorator- and class-driven to its core (controllers, providers, DTOs are all decorated classes). Code-first GraphQL is a natural extension of that style — your GraphQL types *are* Nest classes, they participate in DI and validation, and you never leave TypeScript. For the overwhelming majority of Nest projects, code-first is the right default. Reach for schema-first only when an external team owns the schema contract or you are migrating an existing SDL.

```ts
// Code-first: the class IS the schema definition. Nest generates the SDL.
@ObjectType()
export class User {
  @Field(() => ID) id: string;
  @Field() name: string;
}
```

```graphql
# Schema-first: you author this SDL; resolvers are matched to it.
type User {
  id: ID!
  name: String!
}
```

---

## 2. Setup: GraphQLModule, the Apollo Driver & Project Structure

### What you install and why

A code-first NestJS GraphQL API needs four core packages plus `graphql` itself:

- **`@nestjs/graphql`** — the Nest integration and schema builder.
- **`@nestjs/apollo`** — the Apollo driver binding.
- **`@apollo/server`** — Apollo Server v4/v5 itself.
- **`graphql`** — the reference GraphQL.js implementation everything builds on.

> **⚡ Version note:** With `@nestjs/graphql` v13 + `@nestjs/apollo` v13 you use **`@apollo/server`** (v4/v5). The old `apollo-server-express` package (v2/v3) is dead — do not install it. The driver class is `ApolloDriver` imported from `@nestjs/apollo`. Apollo Server v4 **removed the built-in GraphQL Playground** and replaced it with a hosted "Apollo Sandbox" landing page; to get a local, embedded sandbox you add a plugin (shown below).

```bash
# Node 24 LTS, npm. (Windows 11: run in PowerShell or Git Bash — same commands.)
npm i @nestjs/graphql @nestjs/apollo @apollo/server graphql
# We'll also use these across the guide:
npm i graphql-scalars dataloader @prisma/client
npm i -D prisma
```

### Wiring `GraphQLModule.forRoot` (code-first)

`GraphQLModule.forRoot(config)` is registered once, in your root or a dedicated module. The two settings that make it *code-first* are the **driver** and **`autoSchemaFile`**: when `autoSchemaFile` is set, Nest scans your resolvers/types at bootstrap, builds the schema in memory, and writes the generated SDL to that path so you can diff it in code review.

Key options you will set:

| Option | What it does | Production guidance |
|--------|--------------|---------------------|
| `driver` | The server impl (`ApolloDriver`) | Required |
| `autoSchemaFile` | Path (or `true` for in-memory) to generate SDL from classes | Set a path; commit it to detect schema drift |
| `sortSchema` | Alphabetize the generated SDL | `true` — stable diffs |
| `playground` | Legacy GraphQL Playground | `false` — removed in Apollo v4; use the Sandbox plugin |
| `introspection` | Allow clients to query the schema | **`false` in production** (see §11) |
| `context` | Build the per-request context object | Pass `req`/`res`, attach DataLoaders here |
| `formatError` | Sanitize errors before they leave the server | Strip internals in prod (§5, §11) |
| `csrfPrevention` | Apollo's CSRF protection | **`true`** (§11) |
| `includeStacktraceInErrorResponses` | Leak stack traces | **`false` in production** |

```ts
// src/app.module.ts
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';
import { ApolloServerPluginLandingPageLocalDefault } from '@apollo/server/plugin/landingPage/default';
import { join } from 'path';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,

      // ---- code-first: generate SDL from decorated classes ----
      autoSchemaFile: join(process.cwd(), 'src/schema.gql'), // committed file
      sortSchema: true,

      // ---- developer experience ----
      // Apollo v4 dropped Playground. Enable the *embedded local* sandbox in dev only.
      playground: false,
      plugins:
        process.env.NODE_ENV !== 'production'
          ? [ApolloServerPluginLandingPageLocalDefault({ embed: true })]
          : [],

      // ---- security posture (expanded in §11) ----
      introspection: process.env.NODE_ENV !== 'production',
      csrfPrevention: true,
      includeStacktraceInErrorResponses: process.env.NODE_ENV !== 'production',

      // ---- per-request context: req/res available to guards & resolvers ----
      context: ({ req, res }) => ({ req, res }),
    }),
  ],
})
export class AppModule {}
```

Visit `http://localhost:3000/graphql` in development and you get the embedded Apollo Sandbox: a query editor, schema explorer, and docs derived from introspection.

> **⚡ Version note — async config:** If your GraphQL config depends on `ConfigService` or other providers, use `GraphQLModule.forRootAsync<ApolloDriverConfig>({ driver: ApolloDriver, useFactory: (config: ConfigService) => ({...}), inject: [ConfigService] })`. This is the pattern you will use in real apps so secrets and env-dependent flags (introspection, CORS) come from config.

### Recommended project structure

GraphQL fits the standard Nest **feature-module** layout. Each domain owns its types, resolvers, and service. A common, scalable layout:

```
src/
├─ main.ts                     # bootstrap, global pipes, helmet, CORS, shutdown hooks
├─ app.module.ts              # GraphQLModule.forRoot + feature modules
├─ schema.gql                 # GENERATED (code-first) — commit & review diffs
├─ common/
│  ├─ scalars/                # custom scalars (DateTime, JSON, UUID)
│  ├─ decorators/             # @CurrentUser, @Roles
│  ├─ guards/                 # GqlAuthGuard, RolesGuard
│  ├─ filters/                # GraphQL exception filter
│  ├─ plugins/                # complexity, depth, logging Apollo plugins
│  └─ pagination/             # reusable Relay connection types
├─ prisma/
│  ├─ schema.prisma
│  └─ prisma.service.ts       # PrismaClient as an injectable provider
├─ users/
│  ├─ users.module.ts
│  ├─ users.resolver.ts       # @Query/@Mutation/@ResolveField
│  ├─ users.service.ts        # business logic, calls Prisma
│  ├─ users.loader.ts         # per-request DataLoader (§6)
│  ├─ dto/                    # @InputType / @ArgsType classes
│  └─ entities/user.entity.ts # @ObjectType class
└─ posts/  …same shape
```

The **golden rule** mirrors NestJS REST: resolvers stay thin (extract args, call a service, return the shape); **services own business logic and Prisma calls**; entities are `@ObjectType` classes; DTOs are `@InputType`/`@ArgsType` classes. This keeps logic testable and reusable.

---

## 3. Types with Decorators (Code-First)

### The mental model

In code-first GraphQL, **every part of the schema is a decorated TypeScript class or enum**. The schema builder reads these decorators at startup and emits SDL. The decorators you will use constantly:

| Decorator | GraphQL concept | Used for |
|-----------|-----------------|----------|
| `@ObjectType()` | `type` | Output types returned to clients |
| `@Field()` | a field | One property on a type/input |
| `@InputType()` | `input` | Structured arguments (mutation payloads) |
| `@ArgsType()` | inline args | A reusable bag of `@Args` for a resolver |
| `@registerEnumType()` | `enum` | TypeScript enums in the schema |
| `@InterfaceType()` | `interface` | Shared fields across types |
| `createUnionType()` | `union` | "One of these types" results |
| `@Field(() => Scalar)` | scalar | Custom/standard scalar typing |
| `@ID` | `ID` | Opaque identifiers |

### Nullability and types — the single biggest gotcha

GraphQL fields are **non-null by default in SDL** (`String!`), but TypeScript/JS has no way to reflect "non-null" automatically, so **NestJS treats `@Field()` as non-null unless you say otherwise**. You control nullability with the `nullable` option, and you must give an **explicit type function** for anything ambiguous (arrays, `ID`, enums, custom scalars) because TS metadata erases generics.

- `@Field()` → `String!` (non-null) for inferable primitives.
- `@Field({ nullable: true })` → `String` (nullable).
- `@Field(() => [Post])` → `[Post!]!` — you **must** pass the array's element type via the function; reflection cannot see inside `Post[]`.
- `@Field(() => [Post], { nullable: 'items' })` → `[Post]!` (items nullable, list non-null). `nullable: 'itemsAndList'` → `[Post]`.

> **Best practice:** Be deliberate about nullability. A field marked non-null that resolves to `null` makes Apollo **null out the entire parent object** (non-null propagation), turning one missing value into a much larger hole in the response. When in doubt for fields that can legitimately be absent, mark them nullable.

```ts
// src/users/entities/user.entity.ts
import { ObjectType, Field, ID, Int } from '@nestjs/graphql';

@ObjectType({ description: 'A registered application user.' })
export class User {
  @Field(() => ID) // ID! — opaque identifier; always pass the type fn for ID
  id: string;

  @Field() // String! — non-null inferred for a primitive
  email: string;

  @Field({ nullable: true }) // String — legitimately absent for new users
  displayName?: string;

  @Field(() => Int) // Int! — without () => Int, TS `number` maps to Float
  postCount: number;

  @Field(() => [Post]) // [Post!]! — element type REQUIRED via the function
  posts: Post[];

  // No @Field => NOT exposed in the schema. Keep secrets unexposed:
  passwordHash: string; // never decorated => never queryable. (See §11.)
}
```

### Input types and args

Mutations and filtered queries take structured arguments. There are two ways to declare them:

- **`@InputType()`** — a class describing a nested input object (`input CreateUserInput { ... }`). Use for mutation payloads and anything reusable.
- **`@ArgsType()`** — a class whose fields become **inline top-level arguments** on the resolver, spread into the method via `@Args()`. Use for query filters like `skip`/`take`/`search`.

Validation decorators from `class-validator` go directly on input fields and are enforced by the global `ValidationPipe` (§5).

```ts
// src/users/dto/create-user.input.ts
import { InputType, Field } from '@nestjs/graphql';
import { IsEmail, MinLength, IsOptional } from 'class-validator';

@InputType()
export class CreateUserInput {
  @Field()
  @IsEmail() // validated by ValidationPipe before the resolver runs
  email: string;

  @Field()
  @MinLength(12) // banking-grade: enforce strong password length
  password: string;

  @Field({ nullable: true })
  @IsOptional()
  displayName?: string;
}
```

```ts
// src/users/dto/users.args.ts — inline arguments via @ArgsType
import { ArgsType, Field, Int } from '@nestjs/graphql';
import { Min, Max, IsOptional } from 'class-validator';

@ArgsType()
export class UsersArgs {
  @Field(() => Int, { defaultValue: 0 })
  @Min(0)
  skip: number;

  @Field(() => Int, { defaultValue: 20 })
  @Min(1) @Max(100) // cap page size — a cheap, important DoS guard (§11)
  take: number;

  @Field({ nullable: true })
  @IsOptional()
  search?: string;
}
```

### Enums

Register TypeScript enums explicitly so they appear in the schema:

```ts
// src/users/entities/role.enum.ts
import { registerEnumType } from '@nestjs/graphql';

export enum Role {
  USER = 'USER',
  ADMIN = 'ADMIN',
  AUDITOR = 'AUDITOR',
}
registerEnumType(Role, {
  name: 'Role', // the name that appears in SDL
  description: 'Authorization role for RBAC.',
});
```

### Interfaces and unions

**Interfaces** model shared fields. **Unions** model "the result is one of these types" — ideal for search results or typed mutation errors (an underrated pattern: model errors as union members rather than throwing).

```ts
// Interface: shared identity fields
import { InterfaceType, Field, ID, ObjectType } from '@nestjs/graphql';

@InterfaceType({
  resolveType(value) {            // tell GraphQL the concrete type at runtime
    return value.email ? User.name : Organization.name;
  },
})
export abstract class Node {
  @Field(() => ID) id: string;
}

@ObjectType({ implements: () => [Node] })
export class Organization implements Node {
  @Field(() => ID) id: string;
  @Field() legalName: string;
}
```

```ts
// Union: a search returns users OR organizations
import { createUnionType } from '@nestjs/graphql';

export const SearchResult = createUnionType({
  name: 'SearchResult',
  types: () => [User, Organization] as const,
  resolveType(value) {
    return 'email' in value ? User : Organization;
  },
});
```

```ts
// Union for typed mutation results — clients pattern-match instead of try/catch.
export const RegisterResult = createUnionType({
  name: 'RegisterResult',
  types: () => [User, ValidationError, EmailTakenError] as const,
});
```

### Custom scalars (DateTime, JSON, UUID)

GraphQL ships only `Int`, `Float`, `String`, `Boolean`, `ID`. Real apps need dates, JSON, and UUIDs. The **`graphql-scalars`** library provides battle-tested implementations with parsing/validation built in — prefer it over hand-rolling.

```ts
// src/common/scalars/register.ts — make custom scalars available globally
import { GraphQLDateTime, GraphQLJSON, GraphQLUUID } from 'graphql-scalars';

// In a type:
import { Field, ObjectType } from '@nestjs/graphql';
@ObjectType()
export class AuditEvent {
  @Field(() => GraphQLUUID) id: string;          // validates UUID format on input
  @Field(() => GraphQLDateTime) occurredAt: Date; // ISO-8601, parsed to Date
  @Field(() => GraphQLJSON, { nullable: true }) metadata?: Record<string, unknown>;
}
```

> **Security note on `JSON`:** A `JSON` scalar is convenient but **bypasses GraphQL's type checking** — clients can send arbitrary, unbounded structures. Validate and size-limit any `JSON` input with `class-validator`/manual checks; never persist raw client JSON unvalidated. Avoid `JSON` on the response side if you can model the data with real types.

To register a custom scalar as the default mapping for a TS type (so you can write `@Field()` and have `Date` map to `DateTime` automatically), use the module's `resolvers`/`buildSchemaOptions.scalarsMap` — but explicit `@Field(() => GraphQLDateTime)` is clearer and recommended.

---

## 4. Resolvers, Queries, Mutations & Field Resolvers

### What a resolver is

A **resolver** is a Nest provider decorated with `@Resolver()`. Its methods, decorated with `@Query()`, `@Mutation()`, or `@Subscription()`, become root operations in the schema; methods decorated with `@ResolveField()` resolve individual fields of a type. Because a resolver is a provider, **constructor injection works exactly as in any Nest class** — inject your services, and let them talk to Prisma.

Key decorators:

| Decorator | Role |
|-----------|------|
| `@Resolver(() => User)` | Marks the class; the type arg sets the "parent" for `@ResolveField` |
| `@Query(() => [User])` | A root read operation; return type via the function |
| `@Mutation(() => User)` | A root write operation |
| `@Args('id', { type: () => ID })` | Bind a single argument |
| `@Args()` (no name) | Spread an entire `@ArgsType`/`@InputType` |
| `@ResolveField(() => [Post])` | Resolve a field of the parent type lazily |
| `@Parent()` | Inject the parent object inside a `@ResolveField` |
| `@Context()` | Inject the per-request GraphQL context |
| `@Info()` | Inject the raw GraphQL resolve info (rarely needed) |

### Queries and mutations

```ts
// src/users/users.resolver.ts
import { Resolver, Query, Mutation, Args, ID, Int } from '@nestjs/graphql';
import { UsersService } from './users.service';
import { User } from './entities/user.entity';
import { CreateUserInput } from './dto/create-user.input';
import { UsersArgs } from './dto/users.args';

@Resolver(() => User) // () => User makes User the "parent" for @ResolveField below
export class UsersResolver {
  constructor(private readonly usersService: UsersService) {} // DI as usual

  @Query(() => [User], { name: 'users', description: 'Paginated user list.' })
  async users(@Args() args: UsersArgs): Promise<User[]> {
    return this.usersService.findMany(args); // thin resolver → service
  }

  @Query(() => User, { name: 'user', nullable: true })
  async user(
    @Args('id', { type: () => ID }) id: string, // explicit ID type
  ): Promise<User | null> {
    return this.usersService.findById(id);
  }

  @Mutation(() => User)
  async createUser(@Args('input') input: CreateUserInput): Promise<User> {
    // input is already validated by the global ValidationPipe (§5)
    return this.usersService.create(input);
  }

  @Mutation(() => Boolean)
  async deleteUser(@Args('id', { type: () => ID }) id: string): Promise<boolean> {
    await this.usersService.remove(id);
    return true;
  }
}
```

### Field resolvers — the heart of GraphQL composition

A **field resolver** (`@ResolveField`) computes a field *on demand*, only when the client actually requests it. This is what lets `User.posts` exist without every user query joining the posts table. The parent object is injected with `@Parent()`.

The critical insight — and the source of the N+1 problem — is that **a field resolver runs once per parent object**. If a query returns 50 users and asks for each user's `posts`, the `posts` field resolver fires 50 times. Naïvely, that is 50 separate database queries. §6 fixes this with DataLoader; for now, understand *where* the trap is.

```ts
// Inside UsersResolver — resolves User.posts lazily, per user.
import { ResolveField, Parent } from '@nestjs/graphql';
import { Post } from '../posts/entities/post.entity';

@ResolveField(() => [Post], { name: 'posts' })
async posts(@Parent() user: User): Promise<Post[]> {
  // ⚠️ N+1 RISK: this fires once PER user in the list.
  // Fixed with DataLoader in §6. For now, the naive version:
  return this.usersService.findPostsByAuthor(user.id);
}
```

### The field-resolver vs eager-include tradeoff

You have two ways to populate a relation like `User.posts`:

1. **Eager include** — when fetching users, `include: { posts: true }` in Prisma so posts come back in the same query. Zero N+1, but you fetch posts **even when the client did not ask for them** (over-fetching at the DB), and you cannot apply per-field arguments easily.
2. **Field resolver** — resolve `posts` separately, only when requested. No over-fetching, supports field-level args (`posts(first: 3)`), but risks N+1 without DataLoader.

**Rule of thumb:** Use **field resolvers + DataLoader** as the default — it respects the client's selection and scales. Use eager includes only for relations that are *almost always* requested and cheap. §7 explores this against Prisma in depth.

### Returning the right shape — entities vs GraphQL types

A subtle production issue: your Prisma model and your `@ObjectType` are *different classes*. Prisma returns a plain object shaped like the DB row; your resolver's declared return type is the GraphQL entity. Usually the shapes overlap enough that returning the Prisma object directly works (GraphQL only reads the fields it needs). But:

- **Never let a Prisma object leak a sensitive column** (`passwordHash`) just because GraphQL "won't ask for it" — if it's in the object and someone adds a `@Field`, it leaks. Map explicitly or use Prisma `select` to omit secrets (§11).
- Computed fields (`postCount`) belong in a field resolver or a mapper, not the DB row.

---

## 5. Validation & Error Handling

### Validating inputs with class-validator + ValidationPipe

GraphQL validates that a query is *structurally* valid against the schema (right field names, right scalar types), but it does **not** validate business rules — that an email is well-formed, a password is long enough, a page size is capped. That is your job, and in NestJS the idiomatic tool is the same as for REST: **`class-validator` decorators on input classes, enforced by the global `ValidationPipe`.**

Register the pipe globally in `main.ts`. The options matter:

- **`whitelist: true`** — strip properties not declared on the input class. Defends against mass-assignment.
- **`forbidNonWhitelisted: true`** — reject (don't just strip) unknown properties. Stricter; surfaces client mistakes.
- **`transform: true`** — turn plain payloads into class instances so validators and defaults apply. Required for `@ArgsType` defaults and type coercion to behave.

```ts
// src/main.ts
import { ValidationPipe } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,            // strip unknown props (anti mass-assignment)
      forbidNonWhitelisted: true, // reject unknown props
      transform: true,            // enable type coercion & @ArgsType defaults
      transformOptions: { enableImplicitConversion: false }, // be explicit
    }),
  );
  app.enableShutdownHooks(); // graceful shutdown (§15)
  await app.listen(3000);
}
bootstrap();
```

> **⚡ Gotcha:** `transform: true` is essential in GraphQL. Without it, `@ArgsType` `defaultValue`s and nested `@InputType` validation can silently fail to apply, because the args arrive as plain objects that `class-validator` won't fully traverse.

When validation fails, Nest throws a `BadRequestException`. In GraphQL this surfaces as an error in the `errors` array with the field-level messages under `extensions`.

### How GraphQL errors differ from REST errors

GraphQL has **no HTTP status codes for business errors** — a GraphQL response is almost always HTTP `200`, even when something failed. Errors live in a top-level **`errors`** array alongside (possibly partial) `data`. Each error has a `message`, optional `path`/`locations`, and an **`extensions`** object where structured metadata (an error `code`, validation details) lives.

```json
{
  "data": { "createUser": null },
  "errors": [
    {
      "message": "email must be an email",
      "path": ["createUser"],
      "extensions": { "code": "BAD_USER_INPUT" }
    }
  ]
}
```

**Partial data** is a feature: if a query asks for ten fields and one field resolver throws, GraphQL returns the nine that succeeded plus an error for the tenth (unless non-null propagation nulls a parent). Clients must be written to handle `data` and `errors` *together*.

### Throwing errors the right way

You can throw standard Nest `HttpException`s (`NotFoundException`, `ForbiddenException`, `UnauthorizedException`) from resolvers/services — the Apollo driver maps them into GraphQL errors with sensible `extensions.code`. For domain-specific errors, throw Apollo's `GraphQLError` directly so you control the `code`:

```ts
import { GraphQLError } from 'graphql';
import { NotFoundException } from '@nestjs/common';

// Option A: idiomatic Nest exceptions — auto-mapped by the driver.
throw new NotFoundException(`User ${id} not found`);

// Option B: full control over the code & safe extensions.
throw new GraphQLError('Insufficient funds for transfer', {
  extensions: {
    code: 'INSUFFICIENT_FUNDS',  // stable, client-actionable code
    // NEVER put internal details, SQL, stack traces, or PII here.
  },
});
```

> **Best practice — typed errors as union members:** For predictable, client-handled outcomes (validation failure, "email taken"), consider returning a **union** (§3) instead of throwing. Throwing is right for exceptional/unexpected failures; modeled error types are right for expected business outcomes, and they keep your `errors` array clean.

### A GraphQL exception filter

To log, classify, and **sanitize** every error before it leaves the server, use an exception filter bound to the GraphQL host. This is your last line of defense against leaking internals.

```ts
// src/common/filters/gql-exception.filter.ts
import { Catch, ArgumentsHost, HttpException, Logger } from '@nestjs/common';
import { GqlArgumentsHost, GqlExceptionFilter } from '@nestjs/graphql';
import { GraphQLError } from 'graphql';

@Catch()
export class AllGqlExceptionsFilter implements GqlExceptionFilter {
  private readonly logger = new Logger('GraphQL');

  catch(exception: unknown, host: ArgumentsHost) {
    const gqlHost = GqlArgumentsHost.create(host);
    const info = gqlHost.getInfo();

    // Log the FULL error server-side (with stack) for observability...
    this.logger.error(
      `Resolver ${info?.parentType?.name}.${info?.fieldName} failed`,
      exception instanceof Error ? exception.stack : String(exception),
    );

    // ...but return a SANITIZED error to the client.
    if (exception instanceof HttpException) {
      const res = exception.getResponse();
      const message = typeof res === 'string' ? res : (res as any).message;
      return new GraphQLError(Array.isArray(message) ? message.join(', ') : message, {
        extensions: { code: mapStatusToCode(exception.getStatus()) },
      });
    }
    if (exception instanceof GraphQLError) return exception; // already shaped

    // Unknown/unexpected: never leak. Return a generic message.
    return new GraphQLError('Internal server error', {
      extensions: { code: 'INTERNAL_SERVER_ERROR' },
    });
  }
}

function mapStatusToCode(status: number): string {
  if (status === 400) return 'BAD_USER_INPUT';
  if (status === 401) return 'UNAUTHENTICATED';
  if (status === 403) return 'FORBIDDEN';
  if (status === 404) return 'NOT_FOUND';
  return 'INTERNAL_SERVER_ERROR';
}
```

Pair the filter with **`formatError`** on the module for a final global scrub (strip `stacktrace`, drop unexpected `extensions`) — belt and suspenders. The security rationale (no internal leakage, error hygiene) is expanded in §11.

```ts
// In GraphQLModule.forRoot:
formatError: (formattedError) => {
  if (process.env.NODE_ENV === 'production') {
    // remove anything that could leak internals
    const { stacktrace, ...safeExtensions } = (formattedError.extensions ?? {}) as any;
    return { message: formattedError.message, extensions: safeExtensions, path: formattedError.path };
  }
  return formattedError;
},
```

---

## 6. The N+1 Problem & DataLoader

This is the single most important performance topic in GraphQL. Get it wrong and a query that *looks* innocent issues hundreds of database round trips; get it right and the same query issues two.

### Why the N+1 problem happens

Recall from §4 that **field resolvers run once per parent object**. Consider:

```graphql
query {
  users(take: 50) {     # 1 query: SELECT * FROM users LIMIT 50
    id
    posts {             # field resolver runs 50 times...
      title
    }
  }
}
```

The `users` query is **1** database call. But the `posts` field resolver fires **once per user** — **50** more calls (`SELECT * FROM posts WHERE author_id = ?`). Total: **1 + N = 51** queries. Add a `comments` field under each post and you get N+1 nested *again*. This is the **N+1 problem**: the number of database queries grows linearly with the result size, and it is invisible until you watch the query log under load.

### The fix: batching with DataLoader

**DataLoader** (Facebook's library, the de-facto standard) solves this with two mechanisms:

1. **Batching.** Instead of executing each `load(id)` immediately, DataLoader collects all the `load(id)` calls made during a single tick of the event loop and dispatches them together as **one** batch call: your batch function receives `[id1, id2, …, id50]` and runs a single `SELECT * FROM posts WHERE author_id IN (...)`.
2. **Per-request caching (memoization).** Within one request, `load(42)` called twice returns the same promise — the DB is hit once.

So the 51 queries collapse to **2**: one for users, one batched query for all their posts.

The non-negotiable rules:

- **DataLoader instances MUST be per-request, never shared across requests.** The cache is the danger — a shared loader would serve user A's cached data to user B (a serious data-leak/authorization bug) and never see fresh writes. Create new loaders for every request and attach them to the GraphQL **context**.
- **The batch function must return results in the same order and length as the input keys**, mapping each key to its value (or `null`/`Error`). DataLoader matches by index.

### Wiring per-request DataLoaders through context (the scalable pattern)

The cleanest approach in NestJS: a **request-scoped loader provider** (or a factory invoked in the `context` function) that builds a fresh set of loaders per request, exposed on the context so resolvers can reach them.

```ts
// src/posts/posts.loader.ts — a factory that builds per-request loaders.
import DataLoader from 'dataloader';
import { Injectable, Scope } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { Post } from './entities/post.entity';

@Injectable({ scope: Scope.REQUEST }) // a NEW instance per GraphQL request
export class PostsLoader {
  constructor(private readonly prisma: PrismaService) {}

  // Loader keyed by author id, returning that author's posts.
  public readonly byAuthorId = new DataLoader<string, Post[]>(
    async (authorIds: readonly string[]) => {
      // ONE query for ALL authors in this batch:
      const posts = await this.prisma.post.findMany({
        where: { authorId: { in: [...authorIds] } },
      });
      // Group by author, then return in the SAME ORDER as authorIds:
      const byAuthor = new Map<string, Post[]>();
      for (const id of authorIds) byAuthor.set(id, []);
      for (const p of posts) byAuthor.get(p.authorId)!.push(p as Post);
      return authorIds.map((id) => byAuthor.get(id)!); // index-aligned!
    },
  );
}
```

> **⚡ Scoped-provider performance note:** Marking a provider `Scope.REQUEST` makes Nest instantiate it (and its dependency subtree) on **every request**, which has a real cost and can "bubble up" scope to resolvers. For hot paths, the common alternative is to **build the loaders in the `context` function** (plain functions, no DI scope) and read them off the context. Both are shown — pick request-scoped for DI ergonomics, context-built for maximum throughput.

```ts
// Alternative: build loaders in the GraphQLModule context (no request scope).
// context: ({ req }) => ({ req, loaders: createLoaders(prismaSingleton) })
export function createLoaders(prisma: PrismaService) {
  return {
    postsByAuthor: new DataLoader<string, Post[]>(async (ids) => {
      const posts = await prisma.post.findMany({ where: { authorId: { in: [...ids] } } });
      const map = new Map<string, Post[]>(ids.map((id) => [id, []]));
      posts.forEach((p) => map.get(p.authorId)?.push(p as Post));
      return ids.map((id) => map.get(id)!);
    }),
  };
}
```

### Using the loader in a field resolver

```ts
// In UsersResolver — the N+1 from §4 is now ONE batched query.
import { Context } from '@nestjs/graphql';

@ResolveField(() => [Post], { name: 'posts' })
async posts(
  @Parent() user: User,
  @Context() ctx: { loaders: ReturnType<typeof createLoaders> },
): Promise<Post[]> {
  // All 50 .load() calls in this tick collapse into ONE SELECT ... IN (...)
  return ctx.loaders.postsByAuthor.load(user.id);
}
```

Or, with the request-scoped provider, inject `PostsLoader` into the resolver and call `this.postsLoader.byAuthorId.load(user.id)`.

### Single-entity loaders (the classic `byId`)

The most common loader resolves a single entity by id (e.g. `Post.author` → one user). Return an array aligned to the keys, with `null` for misses:

```ts
new DataLoader<string, User | null>(async (ids) => {
  const users = await prisma.user.findMany({ where: { id: { in: [...ids] } } });
  const map = new Map(users.map((u) => [u.id, u as User]));
  return ids.map((id) => map.get(id) ?? null); // index-aligned, null for misses
});
```

### Measuring — prove it worked

Don't trust, verify. Enable Prisma query logging (or PostgreSQL's `log_statement`) and watch the count drop:

```ts
// PrismaService with query logging — count the SELECTs before/after DataLoader.
new PrismaClient({ log: [{ emit: 'event', level: 'query' }] })
  .$on('query', (e) => console.log('SQL:', e.query));
```

Before DataLoader: `1 + 50` SELECTs. After: `2`. In tests, assert the batch function was called exactly once (spy on it) for a 50-result query. This metric — queries-per-request — is the one to alert on in production.

---

## 7. Database Integration with Prisma

This guide uses **Prisma 6** as the data layer (TypeORM is a fine alternative; the GraphQL concerns are identical). See [Prisma ORM](PRISMA_ORM_GUIDE.md) for Prisma depth and [PostgreSQL](POSTGRESQL_GUIDE.md) for the database.

### PrismaService as an injectable provider

Wrap `PrismaClient` in a Nest provider so it participates in DI and lifecycle (connect on init, disconnect on shutdown). A single shared `PrismaClient` instance manages its own connection pool — **never** `new PrismaClient()` per request.

```ts
// src/prisma/prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() {
    await this.$connect(); // open the pool once at startup
  }
  async onModuleDestroy() {
    await this.$disconnect(); // close gracefully on shutdown (§15)
  }
}
```

```prisma
// prisma/schema.prisma
model User {
  id        String   @id @default(uuid())
  email     String   @unique
  password  String   // hashed — NEVER exposed via GraphQL (no @Field on it)
  posts     Post[]
  createdAt DateTime @default(now())
}

model Post {
  id        String   @id @default(uuid())
  title     String
  body      String
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id])
  authorId  String
  createdAt DateTime @default(now())

  @@index([authorId])      // index the FK — critical for the batched IN query (§6)
  @@index([authorId, createdAt]) // composite index for paginated author feeds (§8)
}
```

### Resolvers calling Prisma through a service

Keep Prisma calls in the service, not the resolver. Use `select` to fetch only what you need and to **exclude secrets**:

```ts
// src/users/users.service.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { CreateUserInput } from './dto/create-user.input';
import { UsersArgs } from './dto/users.args';
import * as argon2 from 'argon2';

@Injectable()
export class UsersService {
  constructor(private readonly prisma: PrismaService) {}

  findMany(args: UsersArgs) {
    return this.prisma.user.findMany({
      skip: args.skip,
      take: args.take,
      where: args.search
        ? { email: { contains: args.search, mode: 'insensitive' } } // parameterized → injection-safe
        : undefined,
      // select EXCLUDES `password` by listing only safe fields:
      select: { id: true, email: true, createdAt: true },
      orderBy: { createdAt: 'desc' },
    });
  }

  findById(id: string) {
    return this.prisma.user.findUnique({
      where: { id },
      select: { id: true, email: true, createdAt: true },
    });
  }

  async create(input: CreateUserInput) {
    const password = await argon2.hash(input.password); // hash — see GO_JWT_ARGON2 guide for the why
    return this.prisma.user.create({
      data: { email: input.email, password, /* displayName */ },
      select: { id: true, email: true, createdAt: true },
    });
  }
}
```

> **Injection safety:** Prisma's query methods are **parameterized** — passing user input to `where`, `data`, etc. is injection-safe by construction. The danger zone is **`$queryRawUnsafe`/`$executeRawUnsafe`** and string-interpolated raw SQL. Prefer the tagged `Prisma.sql`/`$queryRaw` template, which parameterizes. (See §11.)

### Relations: transactions

Multi-step writes that must be atomic (create an order, debit an account, write an audit row) use Prisma transactions. For banking-grade work, prefer the **interactive transaction** API so you can run logic between statements with the whole thing rolled back on error:

```ts
async transferFunds(fromId: string, toId: string, amount: number) {
  return this.prisma.$transaction(async (tx) => {
    const from = await tx.account.update({
      where: { id: fromId },
      data: { balance: { decrement: amount } },
    });
    if (from.balance < 0) throw new Error('Insufficient funds'); // rolls back the whole tx
    await tx.account.update({ where: { id: toId }, data: { balance: { increment: amount } } });
    await tx.ledger.create({ data: { fromId, toId, amount } });
    // commit only if no throw
  });
}
```

### The field-resolver-vs-include tradeoff, in Prisma terms

From §4: do you eager-`include` a relation or resolve it as a field?

- **`include`** in the parent query — fetched in one DB round trip, but fetched **always** (even if the client doesn't select it) and you can't easily paginate the relation per-parent. Good for 1:1 or small, always-needed relations.
- **Field resolver + DataLoader** — fetched only when selected, batched across parents, supports relation-level args/pagination. **The default for lists.**

A sophisticated middle ground is **selection-aware fetching**: inspect the GraphQL `@Info()` to see whether the client selected `posts`, and conditionally `include` it. This avoids both over-fetching and N+1 but adds complexity — adopt it only on proven hot paths.

### Pagination over Prisma — efficient SQL

Offset pagination maps to `skip`/`take`, but **`skip` is O(n)** in the database — the DB still scans and discards the skipped rows, so deep pages get slow. Cursor pagination maps to Prisma's `cursor`/`take`/`skip: 1` and uses the indexed column for a `WHERE id > cursor` seek — **O(log n)** and stable under inserts. §8 builds this out as Relay connections.

```ts
// Cursor pagination in Prisma — seeks via the index, no deep-offset scan.
findPostsAfter(cursor: string | null, take: number) {
  return this.prisma.post.findMany({
    take: take + 1,                       // fetch one extra to compute hasNextPage
    ...(cursor ? { cursor: { id: cursor }, skip: 1 } : {}),
    orderBy: { id: 'asc' },               // MUST order by the cursor column
  });
}
```

---

## 8. Pagination: Offset & Cursor/Relay Connections

### Two models, when to use each

**Offset/limit pagination** (`skip`, `take`) is simple and lets clients jump to page N. Its flaws: deep offsets are slow (the DB scans and discards skipped rows), and concurrent inserts/deletes cause **page drift** (an item appears twice or is skipped as you page). Fine for admin tables and small datasets.

**Cursor pagination** encodes "give me items after *this* one." It is **stable** under concurrent writes and **fast** at any depth because it seeks via an indexed column. It cannot jump to an arbitrary page number — only forward/backward from a cursor. This is the right default for public, high-volume, infinite-scroll lists.

The **Relay Connection specification** is the industry-standard *shape* for cursor pagination, and most GraphQL clients (Apollo Client, Relay) understand it natively:

```graphql
type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  totalCount: Int        # optional; can be expensive — make it nullable/lazy
}
type PostEdge { cursor: String!  node: Post! }
type PageInfo { hasNextPage: Boolean!  hasPreviousPage: Boolean!  startCursor: String  endCursor: String }
```

### A reusable, generic connection type (code-first)

Writing `PostConnection`, `UserConnection`, etc. by hand is tedious. A **generic factory function** produces a connection `@ObjectType` for any node type — write it once, use everywhere.

```ts
// src/common/pagination/connection.ts
import { Type } from '@nestjs/common';
import { Field, ObjectType, Int } from '@nestjs/graphql';

@ObjectType()
export class PageInfo {
  @Field() hasNextPage: boolean;
  @Field() hasPreviousPage: boolean;
  @Field({ nullable: true }) startCursor?: string;
  @Field({ nullable: true }) endCursor?: string;
}

// Factory: returns Edge + Connection classes specialized for `classRef`.
export function Paginated<T>(classRef: Type<T>) {
  @ObjectType(`${classRef.name}Edge`)
  abstract class EdgeType {
    @Field() cursor: string;
    @Field(() => classRef) node: T;
  }

  @ObjectType(`${classRef.name}Connection`, { isAbstract: true })
  abstract class ConnectionType {
    @Field(() => [EdgeType]) edges: EdgeType[];
    @Field(() => PageInfo) pageInfo: PageInfo;
    @Field(() => Int, { nullable: true }) totalCount?: number; // nullable: count is costly
  }
  return ConnectionType;
}
```

```ts
// src/posts/entities/post-connection.ts — one line per type.
import { ObjectType } from '@nestjs/graphql';
import { Paginated } from '../../common/pagination/connection';
import { Post } from './post.entity';

@ObjectType()
export class PostConnection extends Paginated(Post) {}
```

### Building cursors and the connection in a resolver

Cursors should be **opaque** to clients — base64-encode them so clients treat them as blobs and you can change the encoding later. Encode the sort key (here the post id; for sort-by-date use `createdAt:id`).

```ts
// src/posts/posts.service.ts (excerpt)
const encodeCursor = (id: string) => Buffer.from(`post:${id}`).toString('base64url');
const decodeCursor = (c: string) => Buffer.from(c, 'base64url').toString().split(':')[1];

async paginate(first: number, after?: string): Promise<PostConnection> {
  const take = Math.min(first, 100);                   // CAP page size (§11)
  const rows = await this.prisma.post.findMany({
    take: take + 1,                                    // +1 sentinel for hasNextPage
    ...(after ? { cursor: { id: decodeCursor(after) }, skip: 1 } : {}),
    orderBy: { id: 'asc' },                            // index-backed seek
  });
  const hasNextPage = rows.length > take;
  const nodes = hasNextPage ? rows.slice(0, take) : rows;
  const edges = nodes.map((node) => ({ node: node as any, cursor: encodeCursor(node.id) }));
  return {
    edges,
    pageInfo: {
      hasNextPage,
      hasPreviousPage: Boolean(after),
      startCursor: edges[0]?.cursor,
      endCursor: edges.at(-1)?.cursor,
    },
  };
}
```

> **Best practice:** Always **cap `first`/`last`** (e.g. ≤ 100). An uncapped page size is a trivial DoS — a client requesting `first: 1000000` forces a huge scan and response. Make `totalCount` nullable and compute it lazily (a separate `COUNT(*)` is expensive); many clients don't need it.

---

## 9. Subscriptions: Realtime over graphql-ws & Redis

### What subscriptions are and how they work

A **subscription** is a long-lived GraphQL operation: the client opens a persistent **WebSocket** connection, sends a subscription query once, and the server **pushes** a stream of results whenever the relevant event occurs (a new message, an order status change, a price tick). Under the hood, NestJS uses an **`AsyncIterator`**: your `@Subscription` resolver returns an iterator, and each `yield`ed value is delivered to subscribed clients.

The transport matters. There are two WebSocket sub-protocols:

- **`graphql-ws`** — the modern, maintained protocol. **Use this.**
- **`subscriptions-transport-ws`** — the legacy protocol, unmaintained and insecure. Avoid.

> **⚡ Version note:** With `@nestjs/graphql` v13 you enable subscriptions in the Apollo driver config via `subscriptions: { 'graphql-ws': { ... } }`. The `'subscriptions-transport-ws'` key still exists for legacy clients but should be left off in new builds. The WebSocket transport itself is covered in [Node WebSockets](NODE_WEBSOCKETS_GUIDE.md).

```ts
// In GraphQLModule.forRoot — enable graphql-ws subscriptions with connection auth.
subscriptions: {
  'graphql-ws': {
    // onConnect runs ONCE per WebSocket handshake — authenticate here (see §10).
    onConnect: (ctx) => {
      const token = (ctx.connectionParams as any)?.authorization;
      const user = verifyJwt(token); // throw to reject the connection
      (ctx.extra as any).user = user; // stash for later use in resolvers
    },
  },
},
```

### Publishing and subscribing with PubSub

The mechanism that connects a mutation ("a message was created") to a subscription ("notify subscribers") is a **PubSub** engine. The in-memory `PubSub` from `graphql-subscriptions` is fine for a single process:

```ts
// src/messages/messages.resolver.ts
import { Resolver, Subscription, Mutation, Args } from '@nestjs/graphql';
import { PubSub } from 'graphql-subscriptions';

const pubSub = new PubSub(); // ⚠️ in-memory: single-instance only (see scaling below)

@Resolver()
export class MessagesResolver {
  @Mutation(() => Message)
  async sendMessage(@Args('input') input: SendMessageInput): Promise<Message> {
    const message = await this.service.create(input);
    pubSub.publish('messageAdded', { messageAdded: message }); // fan-out
    return message;
  }

  @Subscription(() => Message, {
    // filter: only deliver to clients subscribed to THIS room.
    filter: (payload, variables) => payload.messageAdded.roomId === variables.roomId,
  })
  messageAdded(@Args('roomId') roomId: string) {
    return pubSub.asyncIterableIterator('messageAdded'); // the stream
  }
}
```

The `filter` function is essential for both correctness and **security**: without it, every subscriber to `messageAdded` receives every room's messages — a data-leak across tenants. Filter on the server, never trust the client to ignore data it shouldn't see.

### Scaling subscriptions across instances with Redis

The in-memory `PubSub` has a fatal flaw at scale: **it only fans out within one Node process.** If you run three instances behind a load balancer, a mutation handled by instance A cannot notify a subscriber connected to instance B — the event lives only in A's memory. The fix is a **shared backplane**: publish events to **Redis**, and every instance subscribes to Redis, so an event published anywhere reaches subscribers everywhere.

Swap the in-memory `PubSub` for **`graphql-redis-subscriptions`** (`RedisPubSub`). The resolver code is unchanged — only the engine differs. See [Redis](REDIS_GUIDE.md) for Redis operations and clustering.

```bash
npm i graphql-redis-subscriptions ioredis
```

```ts
// src/pubsub/redis-pubsub.provider.ts
import { RedisPubSub } from 'graphql-redis-subscriptions';
import Redis from 'ioredis';

export const PUB_SUB = 'PUB_SUB';

export const redisPubSubProvider = {
  provide: PUB_SUB,
  useFactory: () => {
    const options = {
      host: process.env.REDIS_HOST,
      port: Number(process.env.REDIS_PORT),
      retryStrategy: (times: number) => Math.min(times * 50, 2000), // reconnect backoff
    };
    return new RedisPubSub({
      publisher: new Redis(options),   // separate connections: Redis pub/sub
      subscriber: new Redis(options),  // requires distinct pub & sub clients
    });
  },
};
```

```ts
// Inject the shared engine instead of newing a local PubSub.
constructor(@Inject(PUB_SUB) private readonly pubSub: RedisPubSub) {}
// ...pubSub.publish(...) and pubSub.asyncIterableIterator(...) work identically,
// but now reach subscribers on EVERY instance via Redis.
```

### Operational notes for production subscriptions

- **Sticky sessions / connection draining.** WebSockets are stateful; ensure your load balancer/ingress supports WebSocket upgrade and long-lived connections (see [Networking](NETWORKING_GUIDE.md) and your reverse proxy config).
- **Authenticate on connect, re-check on operation.** A token valid at connect time can expire mid-connection. For sensitive streams, re-validate authorization when each event is about to be delivered (in `filter`), not just at handshake.
- **Backpressure & limits.** Cap subscriptions per connection and total connections; a flood of subscriptions is a DoS vector. Set WebSocket idle timeouts and ping/pong keepalives.
- **Redis as a single point of failure.** Use Redis HA (Sentinel/Cluster) for the backplane; `RedisPubSub` reconnects, but plan for it.

---

## 10. Authentication & Authorization

Authentication ("who are you") and authorization ("what may you do") are enforced in NestJS with **Guards**. The twist for GraphQL is that the request object isn't where guards expect it — you must pull it out of the GraphQL execution context. For the deeper auth concepts (JWT structure, token rotation, refresh tokens, Argon2 password hashing, threat models) see [Go JWT + Argon2 (Banking-Grade Auth)](GO_JWT_ARGON2_GUIDE.md) and [Better Auth](BETTERAUTH_GUIDE.md); the principles are language-agnostic.

### The key difference: extracting the request from GqlExecutionContext

A normal Nest guard reads the request via `context.switchToHttp().getRequest()`. **In GraphQL there is no HTTP request at that layer** — guards must use **`GqlExecutionContext`** to reach the request (which you put on the context in the module's `context` function). This single adaptation is what makes any HTTP guard work for GraphQL:

```ts
// src/common/guards/gql-auth.guard.ts
import { ExecutionContext, Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { GqlExecutionContext } from '@nestjs/graphql';

@Injectable()
export class GqlAuthGuard extends AuthGuard('jwt') {
  // Override getRequest so Passport finds the request inside the GQL context.
  getRequest(context: ExecutionContext) {
    const ctx = GqlExecutionContext.create(context);
    return ctx.getContext().req; // the `req` we attached in GraphQLModule context
  }
}
```

The JWT strategy itself is standard Nest/Passport — verify the token, load the user, return it (attached to `req.user`):

```ts
// src/auth/jwt.strategy.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,                 // reject expired tokens
      secretOrKey: process.env.JWT_SECRET,     // from secrets, never hardcoded
      algorithms: ['HS256'],                   // PIN the algorithm — prevents alg-confusion attacks
    });
  }
  async validate(payload: { sub: string; role: string }) {
    if (!payload?.sub) throw new UnauthorizedException();
    return { id: payload.sub, role: payload.role }; // becomes req.user
  }
}
```

> **Security note — pin the algorithm.** Always set `algorithms: ['HS256']` (or your RS256 list). Omitting it historically enabled the `alg: none` and HS/RS confusion attacks. The [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md) guide details these threat models.

### The @CurrentUser decorator

Resolvers need the authenticated user ergonomically. A custom param decorator extracts `req.user` from the GraphQL context:

```ts
// src/common/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import { GqlExecutionContext } from '@nestjs/graphql';

export const CurrentUser = createParamDecorator(
  (_data: unknown, context: ExecutionContext) => {
    const ctx = GqlExecutionContext.create(context);
    return ctx.getContext().req.user; // the user set by the JWT strategy
  },
);
```

```ts
// Usage in a resolver:
@UseGuards(GqlAuthGuard)
@Query(() => User)
me(@CurrentUser() user: { id: string; role: string }) {
  return this.usersService.findById(user.id);
}
```

### Role-based access control (RBAC)

Coarse-grained authorization by role uses a `@Roles()` metadata decorator + a `RolesGuard` that reads the required roles and compares against `req.user.role`:

```ts
// src/common/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';
import { Role } from '../../users/entities/role.enum';
export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);
```

```ts
// src/common/guards/roles.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { GqlExecutionContext } from '@nestjs/graphql';
import { ROLES_KEY } from '../decorators/roles.decorator';
import { Role } from '../../users/entities/role.enum';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}
  canActivate(context: ExecutionContext): boolean {
    const required = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(), context.getClass(),
    ]);
    if (!required?.length) return true;          // no @Roles → open to any authed user
    const { req } = GqlExecutionContext.create(context).getContext();
    return required.includes(req.user?.role);    // compare against authenticated role
  }
}
```

```ts
// Resolver/field-level RBAC — guards compose; both must pass.
@UseGuards(GqlAuthGuard, RolesGuard)
@Roles(Role.ADMIN)
@Mutation(() => Boolean)
deleteUser(@Args('id', { type: () => ID }) id: string) {
  return this.usersService.remove(id);
}
```

Because guards apply per resolver/field, you can protect a **single field** of a type (e.g. expose `User.email` only to admins or the owner) by putting a guard on its `@ResolveField`.

### Object-level / ownership authorization

RBAC answers "is this user an admin?" but not "does this user own *this* record?" That **object-level** check (can user 42 edit post 99?) cannot live in a role guard — it needs the data. Enforce it in the **service**, after loading the entity:

```ts
async updatePost(user: { id: string; role: Role }, postId: string, input: UpdatePostInput) {
  const post = await this.prisma.post.findUnique({ where: { id: postId } });
  if (!post) throw new NotFoundException();
  const isOwner = post.authorId === user.id;
  const isAdmin = user.role === Role.ADMIN;
  if (!isOwner && !isAdmin) throw new ForbiddenException('Not your post'); // ownership check
  return this.prisma.post.update({ where: { id: postId }, data: input });
}
```

> **Best practice — return 404 vs 403 carefully.** Revealing "403 Forbidden" tells an attacker the resource *exists*. For sensitive resources, returning `NOT_FOUND` for both "doesn't exist" and "exists but not yours" avoids leaking existence. Decide per resource sensitivity.

### Directive-based auth (an alternative)

You can also encode auth as a **schema directive** (`@auth(requires: ADMIN)`) and enforce it with a field-middleware/`SchemaDirectiveVisitor`. This keeps authorization visible in the SDL and is powerful for federation, but guards are more idiomatic and testable in Nest — prefer guards unless you need declarative-in-schema authz.

---

## 11. Banking-Grade GraphQL Security

GraphQL's flexibility is its biggest security liability: the same feature that lets a client fetch exactly what it wants lets an attacker craft a single request that nests fifty levels deep, fans out across millions of rows, or aliases one expensive field a thousand times. REST's fixed endpoints made these attacks impossible; GraphQL re-opens them. This section is the **OWASP-for-GraphQL checklist**, with NestJS code. Treat every item as mandatory for a production, internet-facing API.

### The threat surface, at a glance

| Threat | Mechanism | Defense |
|--------|-----------|---------|
| Deeply-nested query DoS | Recursive relations (`user{posts{author{posts{…}}}}`) | **Depth limit** |
| Expensive-query DoS | Wide selection + large pages | **Complexity/cost analysis**, page caps |
| Alias amplification | `a:expensive b:expensive c:expensive …` | Complexity (counts aliases), max aliases |
| Batch abuse | Array of operations in one HTTP request | Disable/limit batching, rate-limit |
| Schema recon | Introspection enumerates all types/fields | **Disable introspection in prod** |
| Info leakage | Stack traces, SQL in errors | **Error hygiene**, `formatError` |
| Injection | Raw SQL with interpolated input | **Prisma parameterization** |
| CSRF | Mutation via simple cross-site request | **`csrfPrevention`**, CSRF tokens |
| Authz bypass | Missing guard on a field/resolver | Default-deny, audit every field |
| Credential/PII exposure | Sensitive column exposed via `@Field` | Never decorate secrets; `select` |

### 1. Query depth limiting

A recursive schema lets an attacker write a query that nests arbitrarily deep, causing exponential resolver work. Cap the depth with **`graphql-depth-limit`** as an Apollo `validationRules` entry. Depth ~7–10 covers legitimate queries; reject deeper.

```bash
npm i graphql-depth-limit
```

```ts
// In GraphQLModule.forRoot:
import depthLimit from 'graphql-depth-limit';
// ...
validationRules: [depthLimit(8)], // reject any query nested deeper than 8 levels
```

### 2. Query complexity / cost analysis

Depth alone is insufficient — a shallow query can still be ruinously expensive (`users(first: 1000) { posts(first: 1000) { comments(first: 1000) } }`). **Cost analysis** assigns each field a cost (and multiplies by list sizes) and rejects queries above a budget. Use **`graphql-query-complexity`** via an Apollo plugin.

```bash
npm i graphql-query-complexity
```

```ts
// src/common/plugins/complexity.plugin.ts
import { Plugin } from '@nestjs/apollo';
import { GraphQLSchemaHost } from '@nestjs/graphql';
import { ApolloServerPlugin, GraphQLRequestListener } from '@apollo/server';
import {
  fieldExtensionsEstimator, simpleEstimator, getComplexity,
} from 'graphql-query-complexity';
import { GraphQLError } from 'graphql';

@Plugin()
export class ComplexityPlugin implements ApolloServerPlugin {
  constructor(private schemaHost: GraphQLSchemaHost) {}
  async requestDidStart(): Promise<GraphQLRequestListener<any>> {
    const max = 1000; // tune to your DB capacity
    const { schema } = this.schemaHost;
    return {
      async didResolveOperation({ request, document }) {
        const complexity = getComplexity({
          schema, query: document, variables: request.variables,
          estimators: [
            fieldExtensionsEstimator(),     // honor per-field @Field({ complexity }) hints
            simpleEstimator({ defaultComplexity: 1 }),
          ],
        });
        if (complexity > max) {
          throw new GraphQLError(
            `Query is too complex: ${complexity}. Max allowed: ${max}.`,
            { extensions: { code: 'QUERY_TOO_COMPLEX' } },
          );
        }
      },
    };
  }
}
```

Annotate expensive fields with cost multipliers so paginated fields cost proportionally to page size:

```ts
@Field(() => [Post], { complexity: ({ args, childComplexity }) => (args.first ?? 1) * childComplexity })
posts: Post[];
```

### 3. Rate limiting

Per-IP/per-user rate limiting throttles brute force and abuse. Use **`@nestjs/throttler`** with a GraphQL-aware guard (it must read the request via `GqlExecutionContext`).

```bash
npm i @nestjs/throttler
```

```ts
// In AppModule:
ThrottlerModule.forRoot([{ ttl: 60_000, limit: 100 }]), // 100 req/min/IP

// A GraphQL-aware throttler guard:
import { ThrottlerGuard } from '@nestjs/throttler';
import { GqlExecutionContext } from '@nestjs/graphql';
@Injectable()
export class GqlThrottlerGuard extends ThrottlerGuard {
  getRequestResponse(context: ExecutionContext) {
    const gqlCtx = GqlExecutionContext.create(context);
    const ctx = gqlCtx.getContext();
    return { req: ctx.req, res: ctx.req.res };
  }
}
```

Apply tighter, per-operation limits on sensitive mutations (`login`, `register`, `transferFunds`) with `@Throttle({ default: { limit: 5, ttl: 60_000 } })`.

### 4. Disable introspection & the landing page in production

Introspection lets anyone download your entire schema — a gift to attackers mapping your attack surface. The Apollo Sandbox landing page is also an information disclosure. **Disable both in production** (we already wired this in §2):

```ts
introspection: process.env.NODE_ENV !== 'production', // OFF in prod
plugins: process.env.NODE_ENV === 'production'
  ? [ApolloServerPluginLandingPageDisabled()]         // no landing page in prod
  : [ApolloServerPluginLandingPageLocalDefault({ embed: true })],
```

> Disabling introspection is **defense in depth, not security by obscurity** — a determined attacker can still probe field by field. Combine it with the other controls; never rely on it alone.

### 5. Persisted / allow-listed queries (APQ)

The strongest control against arbitrary-query abuse is to **only accept queries you have pre-approved**. With **Automatic Persisted Queries (APQ)**, clients send a hash; the server executes the stored query for that hash. Turn it into an **allow-list** so unknown hashes (i.e. arbitrary queries) are rejected entirely — attackers literally cannot send a novel malicious query. This is the gold standard for first-party, banking-grade APIs where you control all clients.

```ts
// APQ with a shared Redis cache (so all instances agree on the registry).
persistedQueries: {
  cache: myRedisApolloCache, // an ApolloServer KeyValueCache backed by Redis
},
// For a strict allow-list, add a plugin that rejects any operation whose hash
// is not in your pre-registered build-time manifest.
```

### 6. Batching abuse & aliased-query DoS

Two amplification vectors:

- **Query batching** — Apollo can accept an *array* of operations in one HTTP request. An attacker sends hundreds at once. **Disable batching** (`allowBatchedHttpRequests: false`) unless you need it, or cap the batch size.
- **Alias amplification** — `q1: expensiveField q2: expensiveField … q1000: expensiveField` runs the expensive resolver 1000× in one shallow query, bypassing naïve depth limits. **Complexity analysis** (item 2) counts aliases, which is the real defense; you can also limit the number of aliases via a custom validation rule.

```ts
allowBatchedHttpRequests: false, // disable array-batching DoS vector
```

### 7. Input validation & the JSON-scalar trap

Validate every input with `class-validator` (§5): bound string lengths, cap numbers, restrict enums, and **size-limit `JSON` scalars** (which bypass type checking). Set a global **request body size limit** at the HTTP layer so a giant payload can't OOM the process.

### 8. Error hygiene — never leak internals

In production, return generic messages; log details server-side only. Turn off stack traces and scrub `extensions` (§5's filter + `formatError`). A leaked SQL error or stack trace is a roadmap for an attacker.

```ts
includeStacktraceInErrorResponses: false, // never ship stack traces to clients
```

### 9. CSRF for mutations

Because GraphQL accepts `POST` with `Content-Type: application/json`, browsers' simple-request CSRF is largely mitigated — but Apollo's **`csrfPrevention: true`** enforces a preflight (requiring a non-simple header or content type), closing the gap where `GET` queries or form-encoded requests could be forged cross-site. Keep it **on**.

### 10. Injection — Prisma parameterization

Prisma's query builder parameterizes all input — `where`, `data`, filters are injection-safe. The **only** injection risk is `$queryRawUnsafe`/`$executeRawUnsafe` or string-built SQL. Use the tagged-template `$queryRaw\`...\`` which parameterizes, and never interpolate user input into raw SQL strings. (See [PostgreSQL](POSTGRESQL_GUIDE.md).)

```ts
// SAFE — parameterized tagged template:
await prisma.$queryRaw`SELECT * FROM "User" WHERE email = ${userInput}`;
// DANGEROUS — never do this:
await prisma.$queryRawUnsafe(`SELECT * FROM "User" WHERE email = '${userInput}'`);
```

### 11. Authorization-bypass pitfalls

The most common real-world GraphQL breach is a **missing guard on one field**. Because authz is per-field, a single unprotected `@ResolveField` (e.g. `User.ssn`) leaks data even if the parent query is guarded. Defenses: **default-deny** (a global `GqlAuthGuard` with a `@Public()` opt-out for the few open operations), and **audit every field** that touches sensitive data. Also guard **field resolvers reachable from multiple parents** — a protected `User` query doesn't help if `Post.author` exposes the same `User` unguarded.

### 12. helmet, CORS, and transport

Apply **helmet** for security headers and a **strict CORS** allow-list (never `origin: '*'` for an authenticated API). Terminate **TLS** at the proxy ([Nginx](NGINX_GUIDE.md)/[Networking](NETWORKING_GUIDE.md)) and set HSTS. Note helmet's CSP can conflict with the Apollo Sandbox — disable the sandbox in prod and you avoid the conflict.

```ts
// src/main.ts (excerpt)
import helmet from 'helmet';
app.use(helmet()); // security headers
app.enableCors({ origin: ['https://app.example.com'], credentials: true }); // strict allow-list
```

### The checklist (pin this)

- [ ] Depth limit (`depthLimit(8)`)
- [ ] Complexity/cost limit + per-field cost hints + capped page sizes
- [ ] Rate limiting (global + tighter on auth/money mutations)
- [ ] Introspection & landing page **off** in prod
- [ ] APQ / allow-listed queries for first-party clients
- [ ] Batching disabled (or capped); alias amplification covered by complexity
- [ ] All inputs validated; `JSON` scalars size-limited; body size capped
- [ ] Error hygiene: no stack traces, generic messages, server-side logging
- [ ] `csrfPrevention: true`
- [ ] No raw SQL with interpolation (Prisma parameterized)
- [ ] Default-deny authz; every sensitive field guarded; ownership checks in services
- [ ] helmet + strict CORS + TLS/HSTS; secrets from env, never hardcoded

---

## 12. Performance & Scaling

Performance in a GraphQL API is dominated by **how many database round trips a query makes** and **how much work each field does**. The levers, in order of impact:

### DataLoader first

The biggest win, every time, is eliminating N+1 with DataLoader (§6). Before optimizing anything else, confirm every list-bearing field resolver batches. Measure queries-per-request; it should be roughly constant regardless of result size.

### Response & Redis caching with @CacheControl

GraphQL over `POST` defeats HTTP caching, but Apollo supports **field-level cache hints** (`@CacheControl`) that drive a response cache. Public, slow-changing fields (a product catalog, reference data) can be cached for seconds-to-minutes. Back the cache with **Redis** so all instances share it.

```ts
import { CacheControl } from '@nestjs/apollo'; // or set via @Directive/extensions

@Query(() => [Product])
@CacheControl({ maxAge: 60 }) // cache this field's result for 60s
products() { return this.productsService.findAll(); }
```

> **Security caution:** Never cache **authenticated, per-user** data with a shared cache key — you'll serve one user's data to another. Mark private fields `scope: PRIVATE` (or `maxAge: 0`) and key any per-user cache by user id.

### APQ to cut payload & enable GET caching

Automatic Persisted Queries (§11) also help performance: clients send a short hash instead of a multi-KB query string, shrinking request size, and GET-based APQ becomes CDN-cacheable for public queries.

### Avoiding deep cost

Depth and complexity limits (§11) are performance controls as much as security ones — they cap the worst-case work per request, protecting tail latency and the database from a single pathological query.

### Prisma connection pooling

Each `PrismaClient` owns a pool. Under high concurrency, tune the pool (`connection_limit` in the datasource URL) and consider an external pooler (**PgBouncer**) in transaction mode for serverless/high-instance deployments. Too many instances × too-large pools exhausts PostgreSQL's `max_connections`. See [PostgreSQL](POSTGRESQL_GUIDE.md) and [Database/Server Admin](DATABASE_SERVER_ADMIN_GUIDE.md).

### Nest at scale & the subscription backplane

Run multiple stateless instances behind a load balancer; keep resolvers/services stateless so any instance can serve any request. Subscriptions need the **Redis backplane** (§9) to fan out across instances. Avoid request-scoped providers on hot paths (they reinstantiate per request — §6).

### Profiling & tracing

Enable **Apollo tracing / OpenTelemetry** to get per-field resolution timings — this pinpoints the slow resolver or the un-batched field. Log slow queries from Prisma. The metric that matters most: **p95/p99 query latency** and **DB queries per GraphQL request**.

---

## 13. Schema Architecture at Scale & Federation

### Modular GraphQL modules

A large GraphQL API is organized exactly like a large REST Nest app: **one feature module per domain** (`UsersModule`, `PostsModule`, `OrdersModule`), each owning its resolvers, services, entities, and loaders. The root `GraphQLModule.forRoot` stitches the whole schema together from the resolvers Nest discovers across all imported modules. Cross-module relations (e.g. `Post.author: User`) are resolved by a field resolver in the owning module that calls into the other module's service — keep these dependencies explicit via module `imports`/`exports`.

### Apollo Federation with Nest subgraphs

When one schema and one deploy become a bottleneck for many teams, **Apollo Federation** splits the graph into independently-deployed **subgraphs** (each a separate Nest service owning part of the schema) composed by a **gateway/router** into one supergraph clients query. NestJS supports this with a dedicated federation driver.

> **⚡ Version note:** Use **`@apollo/subgraph`** with the **`ApolloFederationDriver`** from `@nestjs/apollo`, and **Federation 2** directives. Each subgraph sets `autoSchemaFile: { path, federation: 2 }`. The gateway uses `ApolloGatewayDriver` (or, increasingly, the standalone **Apollo Router** in Rust for the gateway in production).

```ts
// A subgraph module (e.g. the Users service):
GraphQLModule.forRoot<ApolloFederationDriverConfig>({
  driver: ApolloFederationDriver,
  autoSchemaFile: { path: 'schema.gql', federation: 2 },
});
```

```ts
// Federated entity: @key declares how other subgraphs reference a User.
import { Directive, ObjectType, Field, ID } from '@nestjs/graphql';

@ObjectType()
@Directive('@key(fields: "id")') // this entity is resolvable by id across subgraphs
export class User {
  @Field(() => ID) id: string;
  @Field() email: string;
}
```

```ts
// A reference resolver lets the gateway hydrate a User from just its id.
@ResolveReference()
resolveReference(ref: { __typename: string; id: string }) {
  return this.usersService.findById(ref.id);
}
```

Federation adds significant operational complexity (gateway, composition checks, cross-subgraph N+1). Reach for it only when team/scaling pressure demands it; a well-modularized monolithic schema serves most products well.

### Schema evolution & deprecation

Evolve the schema **additively**: add fields and types freely (non-breaking). To retire a field, mark it `@Field({ deprecationReason: '...' })` (emits `@deprecated` in SDL), monitor usage via Apollo operation metrics, and remove only after clients migrate. **Breaking changes** — removing a field, changing a type, making a nullable field non-null — must be coordinated. Commit the generated `schema.gql` and run **schema-diff checks in CI** to catch accidental breaking changes before they ship.

---

## 14. Testing GraphQL APIs

A production GraphQL API needs three layers of tests: **unit** (resolvers/services in isolation), **integration/e2e** (queries against the real schema), and **authorization/limit** tests (guards, depth, complexity actually reject what they should). See [NestJS](NESTJS_GUIDE.md) for the testing fundamentals; here is the GraphQL-specific layer.

### Unit-testing resolvers and services

Resolvers are plain providers — instantiate them with a mocked service via the Nest testing module and assert behavior. No GraphQL server needed.

```ts
// users.resolver.spec.ts
import { Test } from '@nestjs/testing';
import { UsersResolver } from './users.resolver';
import { UsersService } from './users.service';

describe('UsersResolver', () => {
  let resolver: UsersResolver;
  const service = { findById: jest.fn() };

  beforeEach(async () => {
    const mod = await Test.createTestingModule({
      providers: [UsersResolver, { provide: UsersService, useValue: service }],
    }).compile();
    resolver = mod.get(UsersResolver);
  });

  it('returns the current user', async () => {
    service.findById.mockResolvedValue({ id: '1', email: 'a@b.c' });
    expect(await resolver.me({ id: '1', role: 'USER' } as any)).toEqual({ id: '1', email: 'a@b.c' });
  });
});
```

### E2E GraphQL tests with supertest

E2E tests boot the whole app and fire real GraphQL operations at `/graphql`, asserting on `data` and `errors`. This is where you validate the wiring: schema, resolvers, validation, and guards together.

```ts
// app.e2e-spec.ts
import { Test } from '@nestjs/testing';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

it('creates a user', async () => {
  const moduleRef = await Test.createTestingModule({ imports: [AppModule] }).compile();
  const app = moduleRef.createNestApplication();
  await app.init();

  const res = await request(app.getHttpServer())
    .post('/graphql')
    .send({
      query: `mutation($i: CreateUserInput!){ createUser(input:$i){ id email } }`,
      variables: { i: { email: 'new@example.com', password: 'longenoughpw12' } },
    })
    .expect(200);

  expect(res.body.data.createUser.email).toBe('new@example.com');
  expect(res.body.errors).toBeUndefined();
  await app.close();
});
```

### Testing guards/authz and the limits

These are the tests attackers wish you skipped. Assert that:

- An **unauthenticated** request to a guarded query returns an `UNAUTHENTICATED` error.
- A **non-admin** calling an admin mutation returns `FORBIDDEN`.
- An **over-depth** query is rejected (send a query nested past the limit, expect an error).
- An **over-complexity** query is rejected (`QUERY_TOO_COMPLEX`).
- **Validation** rejects bad input (`BAD_USER_INPUT`).

```ts
it('rejects a query that exceeds the depth limit', async () => {
  const deep = `query{ user(id:"1"){ posts{ author{ posts{ author{ posts{ id }}}}}}}`;
  const res = await request(app.getHttpServer()).post('/graphql').send({ query: deep });
  expect(res.body.errors?.[0].message).toMatch(/exceeds maximum operation depth/i);
});
```

### Testcontainers for a real database

Mocking Prisma is fine for unit tests, but integration tests should hit a **real PostgreSQL** to catch query bugs, constraint violations, and N+1 regressions. **Testcontainers** spins up a throwaway Postgres in Docker per test run, migrates it, and tears it down — true-to-prod without polluting a shared DB. (See [Docker](DOCKER_GUIDE.md).) Assert query counts here to lock in your DataLoader batching.

---

## 15. Deployment & Observability

### Docker

Package the app as a multi-stage Docker image: build with dev deps, ship a slim runtime with only production deps and the generated Prisma client. Run `prisma migrate deploy` on release. See [Docker](DOCKER_GUIDE.md) for the full pattern.

```dockerfile
# --- build stage ---
FROM node:24-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npx prisma generate && npm run build

# --- runtime stage ---
FROM node:24-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/prisma ./prisma
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

### Behind Nginx / a reverse proxy

Run the app on an internal port; terminate TLS and proxy at **Nginx**. Critically for subscriptions, the proxy must **upgrade WebSocket connections** (`Upgrade`/`Connection` headers) and allow long-lived connections. See [Nginx](NGINX_GUIDE.md) and [Networking](NETWORKING_GUIDE.md).

```nginx
location /graphql {
    proxy_pass http://app:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;      # required for graphql-ws
    proxy_set_header Connection "upgrade";       # WebSocket upgrade
    proxy_read_timeout 3600s;                    # long-lived subscriptions
}
```

### Health checks & graceful shutdown

Expose a `/health` endpoint (`@nestjs/terminus`) that checks DB and Redis for the orchestrator's liveness/readiness probes. Enable **graceful shutdown** (`app.enableShutdownHooks()`) so on `SIGTERM` Nest drains in-flight requests, closes subscriptions, and disconnects Prisma/Redis before exit — no dropped requests during a rolling deploy.

### Observability

- **Tracing:** Apollo tracing or **OpenTelemetry** for per-field timings and distributed traces across subgraphs/services.
- **Metrics:** request rate, error rate, p95/p99 latency, **DB queries per request** (your N+1 alarm), active subscription count, complexity-rejection rate.
- **Logging:** structured (JSON) logs with a correlation/request id; log the operation name and (sanitized) variables — **never log tokens, passwords, or PII**.
- **Secrets/env:** all secrets (`JWT_SECRET`, DB URL, Redis URL) from the environment/secret manager, never committed. Validate them at boot with `@nestjs/config` + a schema so the app fails fast on misconfiguration.

---

## 16. Gotchas & Best Practices

A consolidated, scannable list of the traps that bite real NestJS GraphQL teams.

| # | Gotcha | Fix / best practice |
|---|--------|---------------------|
| 1 | **N+1 queries** from un-batched field resolvers | DataLoader, per-request, attached to context (§6); measure queries/request |
| 2 | **Introspection left on in prod** | `introspection: false` in production (§11) |
| 3 | **No depth/complexity limit** → DoS | `depthLimit(8)` + `graphql-query-complexity` + capped page sizes (§11) |
| 4 | Guards can't find the request in GraphQL | Override `getRequest` via `GqlExecutionContext` (§10) |
| 5 | **Sharing a DataLoader across requests** → data leak / stale data | New loaders per request only; never module-singleton (§6) |
| 6 | **Scoped-provider perf** (`Scope.REQUEST` bubbling up) | Build loaders in `context` for hot paths; limit request scope (§6, §12) |
| 7 | **Leaking internals in errors** (stack traces, SQL) | `formatError` + exception filter; `includeStacktraceInErrorResponses: false` (§5, §11) |
| 8 | **Subscriptions don't scale** past one instance | Redis PubSub backplane (`graphql-redis-subscriptions`) (§9) |
| 9 | **`ValidationPipe` without `transform: true`** | Defaults & nested input validation silently skip; always set `transform: true` (§5) |
| 10 | **Non-null field resolves to `null`** → whole parent nulled | Be deliberate about `nullable`; mark legitimately-absent fields nullable (§3) |
| 11 | **Forgetting the type function** `@Field(() => [X])` | Arrays/ID/enums/scalars need an explicit type fn; reflection can't infer them (§3) |
| 12 | **Sensitive column exposed** via `@Field` or leaked Prisma object | Never decorate secrets; use Prisma `select` to omit them (§7, §11) |
| 13 | **Authz bypass** via one unguarded field resolver | Default-deny global guard + audit every sensitive field (§11) |
| 14 | **Uncapped `first`/`take`** page size | Cap at ≤ 100; reject larger (§8, §11) |
| 15 | **Alias amplification & batching DoS** | Complexity analysis counts aliases; `allowBatchedHttpRequests: false` (§11) |
| 16 | **Caching authenticated data** with a shared key | Private scope / per-user keys; never share per-user data (§12) |
| 17 | **`JSON` scalar** accepts unbounded arbitrary input | Validate & size-limit; avoid on responses where real types fit (§3, §11) |
| 18 | **Raw SQL injection** via `$queryRawUnsafe` | Use parameterized `$queryRaw\`...\`` / Prisma query builder (§7, §11) |
| 19 | **Schema drift** between code and committed SDL | Commit generated `schema.gql`; CI schema-diff check (§13) |
| 20 | **Breaking schema changes** shipped unknowingly | `@deprecated` + usage metrics; additive evolution; CI breaking-change check (§13) |

**Top-line best practices:** thin resolvers / fat services; DataLoader everywhere there's a list relation; default-deny authorization; cap everything (depth, complexity, page size, rate); never trust client-sent data shapes; commit the generated schema; and treat the §11 security checklist as a release gate.

---

## 17. Study Path & Build-to-Learn Projects

The fastest way to internalize this material is to build **one production-grade NestJS GraphQL API** end to end, adding each layer in the order this guide presents them. Each milestone below is a working, testable increment.

### Staged study path

1. **Foundations (§1–§2).** Stand up `GraphQLModule.forRoot` with the Apollo driver, code-first, `autoSchemaFile`. Confirm the Sandbox works in dev. Understand code-first vs schema-first and the per-field request lifecycle.
2. **Types & resolvers (§3–§4).** Model `User` and `Post` as `@ObjectType`s with proper nullability and the type functions. Write `@Query`/`@Mutation`/`@ResolveField`. Wire `@InputType`/`@ArgsType`. Reproduce the N+1 problem deliberately and watch the query log.
3. **Data + validation (§5, §7).** Add Prisma, a `PrismaService`, and a global `ValidationPipe` (`transform: true`). Add a GraphQL exception filter and `formatError`. Throw and shape errors; model one error as a union.
4. **Kill N+1 (§6).** Add DataLoaders (per-request, via context). Prove with the query log that `1+N` became `2`. Assert the batch fires once in a test.
5. **Pagination (§8).** Implement a generic Relay connection and cursor pagination over Prisma. Cap page sizes.
6. **Auth & RBAC (§10).** Add JWT auth, `GqlAuthGuard` (via `GqlExecutionContext`), `@CurrentUser`, `@Roles`/`RolesGuard`, and an ownership check in a service. Pin the JWT algorithm.
7. **Subscriptions (§9).** Add a `graphql-ws` subscription with a server-side `filter`. Then swap in-memory PubSub for **Redis PubSub** and run two instances to prove cross-instance fan-out.
8. **Harden (§11).** Add depth limit, complexity analysis with per-field costs, rate limiting, disable introspection/landing page in prod, `csrfPrevention`, helmet/CORS, error hygiene. Add APQ if all clients are first-party.
9. **Test (§14).** Unit-test resolvers/services; e2e the schema with supertest; write the authz/depth/complexity rejection tests; use Testcontainers for a real DB and assert query counts.
10. **Deploy (§15).** Dockerize, run behind Nginx with WebSocket upgrade, add `/health`, graceful shutdown, and OpenTelemetry tracing.

### Build-to-learn projects

- **Project A — "Threaded Forum API."** Users, posts, nested comments. Forces deep field resolvers (N+1 everywhere), Relay pagination on feeds, and depth limiting (comment trees). Add realtime "new reply" subscriptions. *Skills: §3–§9, §11.*
- **Project B — "Multi-tenant Banking Ledger."** Accounts, transfers (interactive Prisma transactions), an immutable audit log. Banking-grade auth: RBAC (`USER`/`AUDITOR`/`ADMIN`), ownership checks, per-operation rate limits on transfers, complexity limits, full error hygiene, APQ allow-list. *Skills: §7, §10, §11, §14 — the security capstone.*
- **Project C — "Realtime Trading Dashboard."** Subscriptions at scale: price-tick streams fanned out via Redis across three instances behind Nginx, authenticated on connect and re-checked per delivery, with backpressure limits. *Skills: §9, §12, §15.*
- **Project D — "Federated Commerce Graph."** Split `Users`, `Products`, and `Orders` into Apollo Federation subgraphs composed by a gateway; resolve cross-subgraph references; run schema-diff checks in CI. *Skills: §13.*

Build A first (it exercises the core), then B (the security capstone — the most important for banking-grade work), then C and D as your needs grow. Cross-reference the [NestJS](NESTJS_GUIDE.md), [Prisma ORM](PRISMA_ORM_GUIDE.md), [PostgreSQL](POSTGRESQL_GUIDE.md), [Redis](REDIS_GUIDE.md), and [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md) guides as you go, and compare your resolver/loader design against the schema-first Go approach in [Go GraphQL](GO_GRAPHQL_GUIDE.md).
