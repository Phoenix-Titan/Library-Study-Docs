# Fastify v5 — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Node.js developers (complete beginner through advanced) who want to build fast, secure, production-grade HTTP APIs with **Fastify** — the schema-first, plugin-based framework that consistently tops Node.js performance benchmarks. You should know JavaScript and the basics of Node first (see the **JavaScript** and **Node.js** guides in this library). This guide teaches Fastify *itself*: not just the syntax, but the **logic behind each design decision**, when to reach for each feature, how to use it in practice, the key options, the best practices, and the **security implications** of every concept. Every idea is explained in prose first, then shown in heavily-commented runnable code — **TypeScript + ESM first**, with notes for plain JavaScript. You should not need the internet to learn Fastify from this document. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Fastify v5** (released late 2024, current throughout 2026) running on **Node.js 22 / 24 LTS**. Fastify 5 requires **Node.js 20 or newer**, ships first-class **ESM**, uses **Ajv 8** for JSON Schema validation, **fast-json-stringify** for serialization, and **Pino 9** for logging. Several v5 defaults changed from v4: `useSemicolonDelimiter` for querystrings is now `false`, `exposeHeadRoutes` defaults to `true`, the `request.routeOptions` shape was finalized, decorating request/reply with reference-type values is now blocked, and `@fastify/*` plugins are versioned to track core v5. Where a detail moves fast or changed between majors, look for the **⚡ Version note** flag, and always confirm against the version printed by `app.version`.
>
> **How this fits the rest of the library:** Fastify runs on Node, so the **Node.js guide** covers the event loop, streams, and graceful shutdown primitives this guide builds on. If you want a heavier, batteries-included, decorator/DI architecture, compare with the **NestJS guide** (NestJS can actually run *on top of* Fastify). For the database layer used in the examples, see the **PostgreSQL** and **Prisma** guides.

---

## Table of Contents

1. [What Fastify Is & Why (vs Express / NestJS)](#1-what-fastify-is--why-vs-express--nestjs) **[B]**
2. [Setup & Hello World](#2-setup--hello-world) **[B]**
3. [Routing](#3-routing) **[B/I]**
4. [The Request & Reply Lifecycle and Hooks](#4-the-request--reply-lifecycle-and-hooks) **[I/A]**
5. [Validation & Serialization with JSON Schema](#5-validation--serialization-with-json-schema) **[I]**
6. [TypeScript Deeply — Type Providers & Generics](#6-typescript-deeply--type-providers--generics) **[I/A]**
7. [The Plugin System — Encapsulation & Decorators](#7-the-plugin-system--encapsulation--decorators) **[I/A]**
8. [Core Plugin Ecosystem](#8-core-plugin-ecosystem) **[I]**
9. [Authentication & Authorization](#9-authentication--authorization) **[I/A]**
10. [Error Handling](#10-error-handling) **[I]**
11. [Logging with Pino](#11-logging-with-pino) **[I]**
12. [Database Integration Patterns](#12-database-integration-patterns) **[I]**
13. [Testing with node:test and app.inject()](#13-testing-with-nodetest-and-appinject) **[I]**
14. [Performance — Why Fastify Is Fast](#14-performance--why-fastify-is-fast) **[A]**
15. [Configuration, Graceful Shutdown, Deployment & Clustering](#15-configuration-graceful-shutdown-deployment--clustering) **[A]**
16. [Project Structure & @fastify/autoload](#16-project-structure--fastifyautoload) **[I/A]**
17. [Security Checklist](#17-security-checklist) **[I/A]**
18. [Gotchas & Best Practices](#18-gotchas--best-practices)
19. [Study Path & Build-to-Learn Projects](#19-study-path--build-to-learn-projects)

---

## 1. What Fastify Is & Why (vs Express / NestJS)

### What it is

**Fastify** is a low-overhead web framework for Node.js. Where a framework like Express is essentially a router with a middleware chain, Fastify is a router *plus* a schema compiler *plus* an encapsulated plugin system *plus* a structured logger — all engineered around a single obsession: doing as little work as possible per request while still being safe and ergonomic. Its four defining characteristics:

- **Speed.** Fastify is one of the fastest general-purpose Node frameworks. It earns this not with clever tricks at request time but by *moving work to boot time*: your route schemas are compiled once into hyper-optimized JavaScript functions (validation via Ajv, serialization via `fast-json-stringify`). At request time it just calls those pre-built functions.
- **Schema-first.** You attach a **JSON Schema** to each route describing the `body`, `querystring`, `params`, `headers`, and — crucially — the **response**. Fastify uses these schemas both to *validate* incoming data and to *serialize* outgoing data up to ~2x faster than `JSON.stringify`. The schema is the single source of truth that drives validation, serialization, TypeScript types, and your OpenAPI docs.
- **Plugin architecture with encapsulation.** *Everything* in Fastify is a plugin. Plugins create isolated ("encapsulated") contexts: the decorators, hooks, and routes you register inside a plugin do not leak out to siblings or the parent unless you explicitly opt out of encapsulation with `fastify-plugin`. This gives you a lightweight, predictable form of dependency injection without a heavy DI container.
- **Developer experience.** Built-in structured logging (Pino), first-class TypeScript via *type providers*, a complete lifecycle-hook system, and a large official `@fastify/*` plugin ecosystem that is version-matched to the core.

### Why it works the way it does (the logic)

The central insight is that a JSON API does the *same shape* of work on every request: parse a body, check it matches an expected structure, run some logic, then turn an object back into a JSON string. Most frameworks do all of this dynamically, inspecting types at request time. Fastify asks you to *declare the shape up front* so it can generate specialized code once. A validator that knows "this body must be an object with a string `email` and an integer `age`" can be compiled into branch-light JavaScript; a serializer that knows the exact response shape can emit a string by concatenation instead of walking an unknown object. Declaring shape also makes your API safer (unknown fields are rejected, secret fields are stripped) and self-documenting (the same schema generates OpenAPI). Speed, safety, and docs all fall out of one decision.

### Why use it / when to choose it

| You need... | Fastify fit |
|---|---|
| A high-throughput JSON REST API | Ideal — its sweet spot |
| Built-in validation + fast, leak-proof serialization | Ideal — schema-driven by design |
| Structured production logging out of the box | Ideal — Pino is built in |
| A lightweight framework you compose yourself | Ideal — explicit plugins, not magic |
| Microservices where p99 latency matters | Strong fit |
| Heavy enterprise structure with DI, modules, decorators | Consider **NestJS** (which can run *on* Fastify) |
| A 5-line throwaway script | Plain `node:http` is fine |

### Fastify vs Express vs NestJS

| Aspect | Express | Fastify | NestJS |
|---|---|---|---|
| Philosophy | Minimal, unopinionated | Performance + schema-first | Opinionated, structured (Angular-like) |
| Performance | Baseline | ~2-3x Express throughput | Depends on adapter (can use Fastify) |
| Validation | DIY (manual or middleware) | Built-in via JSON Schema | `class-validator` DTOs |
| Serialization | `res.json` (`JSON.stringify`) | `fast-json-stringify` (schema-compiled) | Depends on adapter |
| Logging | DIY (morgan, etc.) | Pino built in | DIY / `nestjs-pino` |
| TypeScript | Types via `@types/express` | First-class, type providers | First-class, decorators |
| Async handlers | Manual `next(err)` | Native async/await + auto error handling | Native async/await |
| Plugins/middleware | Connect middleware | Plugins + hooks (+ middleware via `@fastify/middie`) | Modules + providers |
| Learning curve | Lowest | Low-medium | Medium-high |
| Security defaults | Few | Good (body limits, schema rejection); add helmet | Good; add helmet |

**Mental model:** Express is a router with a middleware chain. Fastify is that router *plus* a schema compiler *plus* an encapsulated DI-lite plugin system *plus* a logger. NestJS is an *architecture layer* (modules, providers, decorators, dependency injection) that can sit on top of either Express or Fastify. If you already know NestJS, think of Fastify as the fast, unopinionated layer NestJS can delegate the HTTP work to.

### A note on security philosophy

Fastify is *safer by default* than Express but is **not** "secure out of the box" with zero effort. It gives you body-size limits, schema-based input rejection, and output stripping for free, but you still must add security headers (`@fastify/helmet`), rate limiting (`@fastify/rate-limit`), CORS rules (`@fastify/cors`), and careful auth. Every section below ends with the specific security considerations for that feature; §17 collects them into one checklist. Treat that checklist as mandatory before shipping.

---

## 2. Setup & Hello World

**What you are doing and why.** Before any code, you create a project, install Fastify and (for the TypeScript path) the TypeScript toolchain, and decide between ESM (`import`/`export`) and CommonJS (`require`). Fastify v5 is ESM-first, so we use ESM throughout — it is the modern default, gives you top-level `await`, and matches what the Node.js guide recommends. `tsx` runs TypeScript directly in development (no compile step), while `tsc` compiles to JavaScript for production.

```bash
# Create a project
mkdir fastify-app && cd fastify-app
npm init -y

# Core runtime dependency
npm install fastify

# TypeScript tooling (dev only)
npm install -D typescript @types/node tsx
```

Make it an ESM project by adding `"type": "module"` to `package.json`:

```jsonc
// package.json (relevant parts)
{
  "type": "module",                       // ⚡ enables native ESM (import/export)
  "scripts": {
    "dev": "tsx watch src/server.ts",     // hot reload in dev, no compile step
    "build": "tsc -p tsconfig.json",      // emit JS to dist/ for production
    "start": "node dist/server.js"        // run the compiled output
  }
}
```

A modern `tsconfig.json` tuned for Fastify v5 + ESM. The comments explain *why* each option matters — Fastify's types reward strictness, and `NodeNext` resolution is what makes ESM `import` paths (with `.js` extensions) work correctly.

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",          // ESM resolution that respects package.json "type"
    "moduleResolution": "NodeNext",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,                 // always on — Fastify's generics rely on it for inference
    "esModuleInterop": true,
    "skipLibCheck": true,           // skip type-checking deps for faster builds
    "declaration": false,
    "sourceMap": true               // map runtime errors back to .ts lines
  },
  "include": ["src/**/*.ts"]
}
```

### The classic Hello World

Every Fastify app is: create the instance (with options), register routes/plugins, then `listen`. Note that handlers are **async** and you simply **return** the payload — Fastify serializes the returned value into the response body. You almost never touch the raw Node `req`/`res`.

```typescript
// src/server.ts
import Fastify from 'fastify' // default import; the ESM-friendly entry point

// Create the instance. The options object configures the WHOLE app:
// logging, body limits, timeouts, the JSON Schema validator, etc.
const app = Fastify({
  logger: true, // turn on the built-in Pino logger (pretty in dev, raw JSON in prod)
})

// Register a route. Because the handler is async, you just RETURN the payload —
// Fastify serializes the return value and sends it as the response body.
app.get('/', async (request, reply) => {
  // `request` and `reply` are Fastify's wrappers (NOT the raw Node req/res).
  request.log.info('handling root request') // per-request child logger (tagged with reqId)
  return { hello: 'world' }                  // returned object is JSON-serialized automatically
})

// Start listening. In v5 `listen` returns a Promise.
const start = async () => {
  try {
    // host '0.0.0.0' is important inside Docker so the port is reachable from outside.
    await app.listen({ port: 3000, host: '0.0.0.0' })
    // No need to log here — logger:true already prints "Server listening at ...".
  } catch (err) {
    app.log.error(err)
    process.exit(1) // fail loudly; a crashed boot must not look "running"
  }
}

start()
```

Run it:

```bash
npm run dev
# In another terminal:
curl http://localhost:3000/
# → {"hello":"world"}
```

**⚡ Version note:** In Fastify v5 the recommended import is the default export `import Fastify from 'fastify'`. Helpers like `FastifyInstance`, `FastifyRequest`, `FastifyReply`, `FastifyPluginAsync` are imported as **types**: `import type { FastifyInstance } from 'fastify'`.

#### Plain JavaScript (ESM) version

```javascript
// src/server.js  (with "type": "module" in package.json)
import Fastify from 'fastify'

const app = Fastify({ logger: true })

app.get('/', async () => ({ hello: 'world' }))

await app.listen({ port: 3000 }) // top-level await works in ESM modules
```

#### CommonJS version (only if you cannot use ESM)

```javascript
// src/server.cjs  (no "type": "module")
const Fastify = require('fastify')
const app = Fastify({ logger: true })
app.get('/', async () => ({ hello: 'world' }))
app.listen({ port: 3000 }).catch((e) => { app.log.error(e); process.exit(1) })
```

### Key constructor options you should know early

The options object passed to `Fastify({...})` is where many security- and performance-relevant defaults live. A few worth knowing from day one:

| Option | Default | Why it matters |
|---|---|---|
| `logger` | `false` | Enables Pino. Turn it on (object form for config). |
| `bodyLimit` | `1048576` (1 MB) | Max accepted request body. Caps memory abuse; raise/lower per need. |
| `connectionTimeout` | `0` (off) | Socket inactivity timeout in ms. |
| `requestTimeout` | `0` (off) | Max time to receive a full request; set it to resist slow-loris attacks. |
| `trustProxy` | `false` | Whether to trust `X-Forwarded-*` headers (only behind a known proxy/LB). |
| `disableRequestLogging` | `false` | Turn off automatic per-request log lines if you log manually. |
| `ajv` | sensible | Customize the JSON Schema validator (see §5). |
| `genReqId` | random | Function to generate the request id used in logs. |

**Security note (setup).** Keep `bodyLimit` as low as your largest legitimate payload requires. Set `requestTimeout` (e.g. `30000`) in production so a client that opens a connection and trickles bytes cannot tie up resources indefinitely. Only enable `trustProxy` when you actually run behind a trusted load balancer — trusting `X-Forwarded-For` blindly lets clients spoof their IP and defeat rate limiting.

---

## 3. Routing

A *route* maps an HTTP method + URL pattern to a handler. Fastify's router is `find-my-way`, a radix-tree router with O(path-length) lookups — fast and predictable regardless of how many routes you register. You declare routes either with shorthand method functions or with the full route-options object (which unlocks per-route schemas, hooks, and constraints).

### Registering routes — two styles

```typescript
import Fastify from 'fastify'
const app = Fastify({ logger: true })

// 1) Shorthand methods (most common, most readable for simple routes)
app.get('/users', async () => [{ id: 1 }])
app.post('/users', async (req, reply) => { reply.code(201); return { created: true } })
app.put('/users/:id', async () => ({ updated: true }))
app.patch('/users/:id', async () => ({ patched: true }))
app.delete('/users/:id', async () => ({ deleted: true }))
app.head('/users', async () => undefined)
app.options('/users', async () => undefined)

// 2) Full route-option object (the most powerful form — schema/hooks live here)
app.route({
  method: 'GET',              // a string, or an array like ['GET', 'HEAD']
  url: '/health',
  handler: async () => ({ status: 'ok' }),
})
```

**Why two styles?** The shorthand is for quick routes. The object form is the canonical place to attach a `schema`, per-route hooks (`preHandler`, etc.), `config`, and `constraints` — everything that makes a route production-ready. Most real apps use the object form as soon as a route needs validation.

### The route-options object — full reference

| Option | Purpose |
|---|---|
| `method` | HTTP method string or array of strings |
| `url` | Route path (supports `:param`, `*` wildcard) |
| `handler` | The route handler `(request, reply) => ...` |
| `schema` | JSON Schema for `body`, `querystring`, `params`, `headers`, `response` |
| `onRequest` / `preParsing` / `preValidation` / `preHandler` / `preSerialization` / `onSend` / `onResponse` | Route-level lifecycle hooks (single fn or array) |
| `config` | Arbitrary config object, readable as `request.routeOptions.config` |
| `constraints` | Match on `version` / `host` / custom constraints |
| `bodyLimit` | Override max body size for this route (bytes) |
| `logLevel` | Per-route log level |
| `exposeHeadRoutes` | Auto-create a HEAD route for a GET (default `true` in v5) |
| `errorHandler` | Route-scoped error handler |
| `schemaErrorFormatter` | Customize validation-error formatting for this route |

### Path & query parameters

```typescript
// Path params: prefix a segment with ":"
app.get('/users/:id/posts/:postId', async (request, reply) => {
  // By default params are typed as Record<string, string>.
  // EVERYTHING from the URL is a string until a schema validates/coerces it.
  const { id, postId } = request.params as { id: string; postId: string }
  return { id, postId }
})

// Querystring: ?page=2&limit=10
app.get('/search', async (request) => {
  const q = request.query as { page?: string; limit?: string }
  return { page: q.page ?? '1', limit: q.limit ?? '20' }
})
```

**⚡ Version note:** Coercion of `"2"` → `2` (number) only happens when a JSON Schema with `type: 'integer'`/`'number'` is attached to that route (see §5). Without a schema, params and query values stay strings. Relying on JavaScript's loose equality to compare an un-coerced string param is a classic bug — always validate.

**Security note (params).** Treat every path/query value as hostile until validated. Un-coerced, unvalidated params flow straight into database queries, file paths, and shell commands. Attach a schema to constrain type, length, and format; never interpolate a raw param into SQL (use parameterized queries — see §12) or into a filesystem path (it enables path traversal like `../../etc/passwd`).

### Wildcards and optional segments

```typescript
// Wildcard: matches the rest of the path. Captured as params['*'].
app.get('/files/*', async (request) => {
  const rest = (request.params as Record<string, string>)['*']
  return { path: rest } // GET /files/a/b/c → { path: "a/b/c" }
})

// Optional parameter: the same handler serves /posts and /posts/123
app.get('/posts/:id?', async (request) => {
  const { id } = request.params as { id?: string }
  return id ? { one: id } : { list: true }
})
```

**Security note (wildcards).** A `*` wildcard used to build a filesystem path is the single most common path-traversal vector in Node apps. If you serve files, prefer `@fastify/static` (which sanitizes paths) over hand-rolling `fs.readFile(join(root, params['*']))`. If you must build the path yourself, resolve it and verify the result still starts with your intended root directory before reading.

### Route prefixes via plugins

The idiomatic way to namespace a group of routes is to register them as a plugin with a `prefix`. Inside the plugin, `/` means the prefix root. Prefixes compose, so you can nest groups.

```typescript
// routes/users.ts
import type { FastifyInstance } from 'fastify'

export default async function userRoutes(app: FastifyInstance) {
  // Inside this plugin, "/" actually means the prefix root.
  app.get('/', async () => ['all users'])     // → GET /api/users/
  app.get('/:id', async () => ({ one: true })) // → GET /api/users/:id
}
```

```typescript
// server.ts
import userRoutes from './routes/users.js' // ⚡ note the .js extension in ESM imports
app.register(userRoutes, { prefix: '/api/users' })
```

A plugin registered with `/api` that itself registers `userRoutes` with `/users` yields `/api/users` — prefixes stack.

### API versioning

Two common strategies, with different trade-offs:

```typescript
// 1) URL-prefix versioning (simplest, most explicit, cache-friendly)
app.register(v1Routes, { prefix: '/v1' })
app.register(v2Routes, { prefix: '/v2' })

// 2) Header/version constraints (same URL, different versions via Accept-Version)
app.route({
  method: 'GET',
  url: '/users',
  constraints: { version: '1.2.0' }, // matches header Accept-Version: 1.2.0 (semver)
  handler: async () => ({ v: '1.2.0' }),
})
app.route({
  method: 'GET',
  url: '/users',
  constraints: { version: '2.0.0' },
  handler: async () => ({ v: '2.0.0' }),
})
// Client sends header:  Accept-Version: 2.x  → routed to the 2.0.0 handler.
```

URL-prefix versioning is the pragmatic default: it is visible in logs, cacheable by CDNs, and easy to reason about. Header versioning keeps URLs stable but hides the version from caches and humans.

### Constraints (version, host, custom)

Constraints let two routes share a URL but match on additional request properties.

```typescript
// Host constraint: route only matches a specific Host header
app.route({
  method: 'GET',
  url: '/',
  constraints: { host: 'admin.example.com' },
  handler: async () => ({ area: 'admin' }),
})

// Custom constraint strategy (advanced) — match on ANY request property.
const app2 = Fastify({
  constraints: {
    // A strategy keyed by name; here we constrain on a "tenant" header.
    tenant: {
      name: 'tenant',
      storage() {
        const tenants: Record<string, unknown> = {}
        return {
          get: (k: string) => tenants[k] ?? null,
          set: (k: string, v: unknown) => { tenants[k] = v },
        }
      },
      deriveConstraint: (req) => req.headers['x-tenant'] as string,
      validate() { return true },
    },
  },
})
app2.route({
  method: 'GET', url: '/', constraints: { tenant: 'acme' },
  handler: async () => ({ tenant: 'acme' }),
})
```

### Inspecting the route tree

```typescript
// After all routes are registered (e.g. once the app is ready):
await app.ready()
console.log(app.printRoutes())     // pretty tree of the radix router — great for debugging
// console.log(app.printPlugins())  // plugin registration tree (debug load order)
```

**Best practices (routing).** Keep URLs noun-based and plural (`/users/:id`), use the right HTTP method (POST creates, PUT replaces, PATCH partially updates, DELETE removes), and let `exposeHeadRoutes` auto-create HEAD for your GETs. Group related routes into prefixed plugins so they stay encapsulated and discoverable.

---

## 4. The Request & Reply Lifecycle and Hooks

Understanding the lifecycle is the single most important thing for using Fastify well. Every request flows through an **ordered pipeline of hooks**. Hooks are where you put cross-cutting concerns — authentication, authorization, input normalization, response enveloping, metrics, auditing — so your handlers stay focused on business logic. Knowing *which* hook fires *when* is what lets you put each concern in exactly the right place.

### The big picture

```
                            Incoming HTTP request
                                     │
                                     ▼
                          ┌────────────────────┐
                          │     onRequest      │  auth checks, rate limiting, request id
                          └─────────┬──────────┘  (body NOT parsed yet — cheap to reject here)
                                    │
                                    ▼
                          ┌────────────────────┐
                          │    preParsing      │  transform the raw body STREAM
                          └─────────┬──────────┘  (decompress, decrypt)
                                    │
                                    ▼   (body is parsed into an object here)
                          ┌────────────────────┐
                          │   preValidation    │  mutate parsed payload BEFORE validation
                          └─────────┬──────────┘  (inject defaults, normalize)
                                    │
                                    ▼   (JSON Schema validation runs here — Ajv)
                          ┌────────────────────┐
                          │    preHandler      │  authorization guards, load related data
                          └─────────┬──────────┘  (input is now valid)
                                    │
                                    ▼
                          ┌────────────────────┐
                          │   ROUTE HANDLER    │  your business logic
                          └─────────┬──────────┘  (handler returns / reply.send called)
                                    │
                                    ▼
                          ┌────────────────────┐
                          │  preSerialization  │  mutate the payload object before serialize
                          └─────────┬──────────┘  (wrap in an envelope)
                                    │
                                    ▼   (fast-json-stringify serializes here)
                          ┌────────────────────┐
                          │      onSend        │  modify serialized bytes / headers
                          └─────────┬──────────┘  (compress, add headers, mask)
                                    │
                                    ▼   (bytes written to the socket)
                          ┌────────────────────┐
                          │    onResponse      │  metrics, audit (response is done)
                          └────────────────────┘

   Error at any point ─────────────────────────►  onError  ──►  errorHandler
   Request exceeds requestTimeout ──────────────►  onTimeout
   Client disconnects mid-request ──────────────►  onRequestAbort
```

### Why this order exists (the logic)

The pipeline runs cheapest-and-most-decisive checks first. `onRequest` fires before the body is even read, so a rate-limit or auth rejection there costs almost nothing — you never pay to parse a body you are going to throw away. Parsing and validation happen in the middle, so by the time `preHandler` runs you can trust the input shape and focus on *authorization* (does this valid user have permission?) and *data loading*. After the handler, `preSerialization` works with the still-live JavaScript object (easy to wrap or transform), then `onSend` works with the final bytes (right place to compress or set headers). `onResponse` runs after everything is on the wire — perfect for metrics that should not slow the response.

### Hook reference table

| Hook | When it fires | What it can change | Typical use |
|---|---|---|---|
| `onRequest` | First, before body parsing | request only | Auth, rate-limit, request id, IP allowlist |
| `preParsing` | Before body is parsed | raw body stream | Decompress, decrypt, transform raw stream |
| `preValidation` | After parse, before validation | parsed body/query | Inject defaults, normalize input |
| `preHandler` | After validation, before handler | request/reply | Authorization (RBAC), load related data |
| `preSerialization` | After handler, before serialize | the payload object | Wrap/envelope responses |
| `onSend` | After serialize, before send | serialized payload + headers | Add headers, compress, mask data |
| `onResponse` | After response fully sent | nothing (read-only) | Metrics, access logging, cleanup |
| `onError` | When an error is thrown/sent | read-only | Custom error logging/alerting |
| `onTimeout` | When `requestTimeout` fires | read-only | Log slow requests |
| `onRequestAbort` | Client disconnects mid-request | read-only | Cancel work, release resources |

There are also **application lifecycle hooks** (not per-request): `onReady` (all plugins loaded), `onListen`, `onClose` / `preClose` (shutdown), `onRoute` (a route was registered), and `onRegister` (a plugin was registered).

### Application-level vs route-level hooks

Application-level hooks run for **every route in the current encapsulation context** (see §7). Route-level hooks run only for that one route. Use app-level for broad concerns (logging) and route-level for targeted ones (this route needs admin).

```typescript
// Application-level: runs for every route registered in this context.
app.addHook('onRequest', async (request, reply) => {
  request.log.info({ url: request.url }, 'incoming')
})

// Route-level: only this route.
app.route({
  method: 'GET',
  url: '/secure',
  preHandler: async (request, reply) => {
    // Throw to abort; Fastify routes the thrown error to the error handler.
    if (!request.headers.authorization) {
      reply.code(401)
      throw new Error('missing auth')
    }
  },
  handler: async () => ({ ok: true }),
})
```

### Two ways to signal "done" in a hook

Fastify supports both async/await and the legacy callback (`done`) style. **Pick one per hook and never mix them** — that is the most common lifecycle bug.

```typescript
// 1) Async style (RECOMMENDED): return to continue, throw to abort.
app.addHook('preHandler', async (request, reply) => {
  await doSomething()
  // return nothing → continue
  // throw err      → jump to the error handler
  // return reply   → short-circuit (e.g. after you called reply.send())
})

// 2) Callback style (legacy): call done(). Do NOT also make this async.
app.addHook('preHandler', (request, reply, done) => {
  doSomething(() => done())     // done() to continue
  // done(new Error('nope'))    // pass an error to abort
})
```

**⚡ Gotcha:** If your hook is `async`, never also call `done` — returning resolves it. Mixing them causes "callback already called" errors and double-sent responses.

### Short-circuiting a request from a hook

```typescript
app.addHook('onRequest', async (request, reply) => {
  if (isBlocked(request.ip)) {
    // Send a response AND tell Fastify to stop the pipeline by returning reply.
    reply.code(403).send({ error: 'forbidden' })
    return reply // returning the reply object halts further hooks/handler
  }
})
```

### preParsing — transform the raw stream

```typescript
import { Readable } from 'node:stream'

app.addHook('preParsing', async (request, reply, payload) => {
  // `payload` is a stream of the raw request body (bytes, not yet an object).
  // You MUST return a stream (the same one or a new one). Example: pass-through.
  return payload // real uses: gunzip, decrypt, line-count limiting, etc.
})
```

### preSerialization vs onSend — a frequently confused pair

The difference is the *type* of the payload: `preSerialization` sees the JavaScript **object** before it becomes a string; `onSend` sees the **serialized string/Buffer** and can also tweak headers.

```typescript
// preSerialization sees the JS object BEFORE serialization.
app.addHook('preSerialization', async (request, reply, payload) => {
  // Wrap every successful payload in a consistent envelope.
  if (reply.statusCode < 400) {
    return { data: payload, meta: { ts: Date.now() } }
  }
  return payload
})

// onSend sees the SERIALIZED payload (string/Buffer) and can tweak headers.
app.addHook('onSend', async (request, reply, payload) => {
  reply.header('x-content-type-options', 'nosniff')
  return payload // must return the (possibly modified) payload
})
```

### onResponse — for metrics and access logs

```typescript
app.addHook('onResponse', async (request, reply) => {
  // reply.elapsedTime is the ms spent handling the request.
  request.log.info(
    { method: request.method, url: request.url, status: reply.statusCode, ms: reply.elapsedTime },
    'request completed',
  )
})
```

### The Reply object — key methods

| Method | Effect |
|---|---|
| `reply.send(payload)` | Send a response (objects are serialized) |
| `reply.code(n)` / `reply.status(n)` | Set the HTTP status code |
| `reply.header(k, v)` / `reply.headers({})` | Set response header(s) |
| `reply.type('application/json')` | Shortcut for Content-Type |
| `reply.redirect(url, code?)` | Redirect (default 302) |
| `reply.serialize(obj)` | Manually run the route's serializer |
| `reply.raw` | The underlying Node `ServerResponse` (escape hatch) |
| `reply.sent` | Boolean: has the reply been sent? |
| `reply.hijack()` | Tell Fastify you'll manage the raw socket yourself |
| `reply.elapsedTime` | ms since the request started (use in `onResponse`) |

**⚡ Best practice:** In `async` handlers, prefer **returning** the payload over calling `reply.send()`. Return the value and let Fastify serialize it; use `reply.send()` only when you need imperative control (e.g. inside a callback). Never both `return` and `reply.send()` the same response — the second one is ignored and the code is confusing.

```typescript
// GOOD (async): set status, then return the payload
app.get('/a', async (req, reply) => {
  reply.code(201)          // setting status is fine
  return { created: true } // returning sends it
})

// AVOID: returning AND sending double-handles the response
app.get('/b', async (req, reply) => {
  reply.send({ x: 1 })
  return { x: 1 } // ⚡ ignored once send() was called — confusing, drop it
})
```

**Security note (hooks).** `onRequest` is the cheapest, earliest place to reject unauthenticated or rate-limited traffic — do gatekeeping there to avoid wasting work. Do *authorization* (role/ownership checks) in `preHandler`, after validation, so the user and input are both known. Use `onSend` to enforce response security headers globally. Be careful in `preSerialization`/`onSend` not to *add back* sensitive fields that your response schema (§5) deliberately stripped.

---

## 5. Validation & Serialization with JSON Schema

This is **the** core Fastify feature and the reason to choose it. You declare JSON Schemas for input and output; Fastify compiles them at boot into fast validators (Ajv) and serializers (`fast-json-stringify`). The same schema gives you input validation, output serialization, type inference (§6), and OpenAPI docs (§8) — one declaration, four payoffs.

### What JSON Schema is

JSON Schema is a vocabulary for describing the shape of JSON data: types (`string`, `integer`, `object`, `array`), constraints (`minimum`, `maxLength`, `pattern`, `enum`, `required`), and structure (`properties`, `items`). Fastify uses **Ajv 8** to compile a schema into a validation function and `fast-json-stringify` to compile a *response* schema into a serialization function.

### Request validation: body, querystring, params, headers

```typescript
app.post('/users', {
  schema: {
    // Validate the JSON body
    body: {
      type: 'object',
      required: ['email', 'age'],
      additionalProperties: false, // ⚡ reject unknown fields (STRONGLY recommended — see security note)
      properties: {
        email: { type: 'string', format: 'email', maxLength: 254 },
        age: { type: 'integer', minimum: 0, maximum: 130 },
        name: { type: 'string', minLength: 1, maxLength: 100 },
      },
    },
    // Validate AND COERCE the querystring (strings → numbers/booleans)
    querystring: {
      type: 'object',
      properties: {
        verbose: { type: 'boolean', default: false }, // "true" → true
        limit: { type: 'integer', default: 20, maximum: 100 }, // "20" → 20, capped
      },
    },
    // Validate path params
    params: {
      type: 'object',
      properties: { tenant: { type: 'string', pattern: '^[a-z0-9-]+$' } },
    },
    // Validate specific headers
    headers: {
      type: 'object',
      required: ['x-api-key'],
      properties: { 'x-api-key': { type: 'string' } },
    },
  },
}, async (request) => {
  // request.body is now validated AND coerced according to the schema.
  return { received: request.body }
})
```

If validation fails, Fastify automatically responds `400` with a descriptive error — you write **zero** validation code in the handler. That is the point: validation is declarative and lives next to the route.

### Response serialization — the speed *and* security benefit

Adding a `response` schema does three things, and the third is a security feature people often miss:

1. It serializes with `fast-json-stringify`, significantly faster than `JSON.stringify`.
2. It documents the response shape for OpenAPI.
3. It **strips any property not in the schema** — so even if your handler accidentally returns a `passwordHash` or an internal flag, it never reaches the client.

```typescript
const userResponseSchema = {
  type: 'object',
  properties: {
    id: { type: 'integer' },
    email: { type: 'string' },
    // NOTE: `passwordHash` is intentionally NOT here. Even if the handler returns
    // it, fast-json-stringify strips it from the output. This is data-leak defense.
  },
} as const

app.get('/users/:id', {
  schema: {
    response: {
      200: userResponseSchema,                 // schema per status code
      404: { type: 'object', properties: { message: { type: 'string' } } },
      '2xx': userResponseSchema,               // wildcard ranges also allowed
    },
  },
}, async (request, reply) => {
  // Even though we return extra fields, only id + email reach the client.
  return { id: 1, email: 'a@b.com', passwordHash: 'SECRET-WOULD-LEAK?' } // stripped!
})
```

**⚡ Why it is faster:** `JSON.stringify` must inspect every value's type at runtime on every call. `fast-json-stringify` generates a bespoke function *from your schema at boot* — it already knows the field order and types, so it emits the JSON string with minimal branching. For high-throughput APIs this is a meaningful, measurable win (see §14).

### Shared schemas with `$ref` and `addSchema`

Avoid duplicating schemas by registering them once with a stable `$id`, then referencing them by `$ref`. This keeps your schemas DRY and makes OpenAPI output cleaner (named components).

```typescript
// Register a reusable schema with a stable $id.
app.addSchema({
  $id: 'user',
  type: 'object',
  properties: {
    id: { type: 'integer' },
    email: { type: 'string', format: 'email' },
  },
})

app.addSchema({
  $id: 'errorResponse',
  type: 'object',
  properties: { message: { type: 'string' }, code: { type: 'string' } },
})

// Reference shared schemas via $ref: "<$id>#"
app.get('/users/:id', {
  schema: {
    response: {
      200: { $ref: 'user#' },
      404: { $ref: 'errorResponse#' },
    },
  },
}, async () => ({ id: 1, email: 'a@b.com' }))

// You can also reference a sub-property: { $ref: 'user#/properties/email' }
```

**⚡ Encapsulation note:** Schemas added with `addSchema` are scoped to the encapsulation context (§7). A schema added at the root is visible to child plugins, but a schema added *inside* a plugin is **not** visible to siblings or the parent. Add shared schemas at the top level (or via an `fp`-wrapped plugin) so every route can `$ref` them.

### Customizing validation error output

You can tune Ajv, format errors globally in an error handler, or format per route.

```typescript
const app = Fastify({
  // Option A: tune Ajv directly
  ajv: {
    customOptions: {
      allErrors: true,         // collect ALL errors, not just the first (nicer 400s)
      removeAdditional: 'all', // strip extra props instead of erroring (use carefully — see note)
      coerceTypes: true,       // default true — string→number coercion
      useDefaults: true,       // apply `default` values from the schema
    },
  },
})

// Option B: format the validation error response yourself (global)
app.setErrorHandler((error, request, reply) => {
  if (error.validation) {
    // error.validation is the raw Ajv error array
    return reply.code(400).send({
      error: 'ValidationError',
      details: error.validation.map((e) => ({
        field: e.instancePath || e.params?.missingProperty,
        message: e.message,
      })),
    })
  }
  request.log.error(error)
  return reply.code(500).send({ error: 'Internal Server Error' })
})

// Option C: per-route schemaErrorFormatter
app.route({
  method: 'POST', url: '/strict',
  schema: { body: { type: 'object', required: ['x'], properties: { x: { type: 'string' } } } },
  schemaErrorFormatter: (errors, dataVar) => new Error(`Bad ${dataVar}: ${errors[0]?.message}`),
  handler: async () => ({ ok: true }),
})
```

**⚡ `removeAdditional` vs `additionalProperties: false`:** these are opposite policies. `additionalProperties: false` *rejects* a request that contains unknown fields (a 400). `removeAdditional: 'all'` *silently strips* them and proceeds. Rejecting is the safer, more explicit default for APIs — silent stripping can mask client bugs and (combined with mass-assignment) hide attacks. Prefer `additionalProperties: false` unless you have a specific reason.

### Adding formats (e.g. `uuid`)

Ajv 8 ships common formats via `ajv-formats`, which Fastify v5 bundles. Use `format: 'email'`, `'uuid'`, `'date-time'`, `'uri'`, etc. directly.

```typescript
app.get('/items/:id', {
  schema: {
    params: {
      type: 'object',
      required: ['id'],
      properties: { id: { type: 'string', format: 'uuid' } }, // uuid format from ajv-formats
    },
  },
}, async (req) => ({ id: (req.params as any).id }))
```

**Security note (validation — the most important in this guide).** Schema validation is your primary input-trust boundary. Always:
- Set **`additionalProperties: false`** on every body schema to block **mass-assignment** (a client sneaking `isAdmin: true` into a user-create body).
- Add **length/size limits** (`maxLength`, `maxItems`, `maximum`) to bound memory and prevent abuse; an unbounded `string` or `array` is a denial-of-service vector.
- Add a **`response` schema** to every route that returns data, so internal/secret fields cannot leak even by accident.
- Validate **before** you touch the database or filesystem; never trust coerced-but-unchecked values.
- Be aware of **Ajv regex (`pattern`) ReDoS**: avoid catastrophic backtracking patterns on attacker-controlled input.

---

## 6. TypeScript Deeply — Type Providers & Generics

Fastify is written in TypeScript and supports two levels of typing. The goal is to make your **JSON Schema the single source of truth** so types and runtime validation never drift apart.

1. **Manual generics** — you tell Fastify the shape of `Body`, `Querystring`, `Params`, `Headers`, `Reply`. Simple but duplicates the shape (once in TS, once in the schema).
2. **Type providers** — your schema is *inferred into* TypeScript types automatically, no duplication. Two popular providers: **TypeBox** and **json-schema-to-ts**.

### Manual generics

```typescript
import Fastify, { type FastifyRequest, type FastifyReply } from 'fastify'

const app = Fastify()

// Describe the request/response shapes as a generic argument.
interface CreateUserRoute {
  Body: { email: string; age: number }
  Querystring: { dryRun?: boolean }
  Params: { tenant: string }
  Headers: { 'x-api-key': string }
  Reply: { id: number; email: string }
}

app.post<CreateUserRoute>('/:tenant/users', async (request, reply) => {
  request.body.email            // typed as string
  request.query.dryRun          // typed as boolean | undefined
  request.params.tenant         // typed as string
  request.headers['x-api-key']  // typed as string
  return { id: 1, email: request.body.email } // Reply type enforced
})
```

This works, but the TS interface and the JSON Schema are separate — they can drift. Type providers eliminate that risk.

### Type Provider: TypeBox (recommended)

TypeBox lets you write a schema that is *simultaneously* a valid JSON Schema *and* a TypeScript type. One object, both worlds.

```bash
npm install @sinclair/typebox @fastify/type-provider-typebox
```

```typescript
// src/server.ts
import Fastify from 'fastify'
import { Type } from '@sinclair/typebox'
import { TypeBoxTypeProvider } from '@fastify/type-provider-typebox'

// withTypeProvider rebinds the instance so schemas drive types.
const app = Fastify().withTypeProvider<TypeBoxTypeProvider>()

// Define schema ONCE — it is runtime JSON Schema AND a static TS type.
const CreateUser = Type.Object({
  email: Type.String({ format: 'email' }),
  age: Type.Integer({ minimum: 0 }),
}, { additionalProperties: false })

const UserReply = Type.Object({
  id: Type.Integer(),
  email: Type.String(),
})

app.post('/users', {
  schema: {
    body: CreateUser,
    response: { 200: UserReply },
  },
}, async (request, reply) => {
  // request.body is fully typed as { email: string; age: number } — INFERRED!
  const { email } = request.body
  // The return value is type-checked against UserReply.
  return { id: 1, email }
})
```

Useful TypeBox building blocks:

```typescript
import { Static, Type } from '@sinclair/typebox'

const Paging = Type.Object({
  page: Type.Optional(Type.Integer({ minimum: 1, default: 1 })),
  limit: Type.Integer({ minimum: 1, maximum: 100, default: 20 }),
})
type Paging = Static<typeof Paging>     // derive a plain TS type to use elsewhere

const Role = Type.Union([Type.Literal('admin'), Type.Literal('user')]) // string enum
const Nullable = Type.Union([Type.String(), Type.Null()])              // string | null
```

### Type Provider: json-schema-to-ts

If you prefer hand-written JSON Schema (no DSL), `json-schema-to-ts` infers types from a plain schema object marked `as const`.

```bash
npm install -D json-schema-to-ts
npm install @fastify/type-provider-json-schema-to-ts
```

```typescript
import Fastify from 'fastify'
import { JsonSchemaToTsProvider } from '@fastify/type-provider-json-schema-to-ts'

const app = Fastify().withTypeProvider<JsonSchemaToTsProvider>()

app.post('/users', {
  schema: {
    body: {
      type: 'object',
      required: ['email'],
      additionalProperties: false,
      properties: {
        email: { type: 'string' },
        age: { type: 'integer' },
      },
    } as const, // ⚡ `as const` is REQUIRED for inference to work
    response: {
      200: {
        type: 'object',
        properties: { id: { type: 'integer' }, email: { type: 'string' } },
      } as const,
    },
  },
}, async (request) => {
  request.body.email // inferred as string
  return { id: 1, email: request.body.email }
})
```

| Provider | Pros | Cons |
|---|---|---|
| **TypeBox** | Concise DSL, single source, great inference, supports most JSON Schema features | Adds a small runtime dependency |
| **json-schema-to-ts** | Plain JSON Schema, no DSL, dev-only dependency | Needs `as const`; more verbose; slower TS compile on large schemas |

### Typing the FastifyInstance and module augmentation

When you decorate the instance/request/reply (see §7), use **declaration merging** to teach TypeScript about those additions so they are known everywhere, fully typed.

```typescript
// src/types/fastify.d.ts
import type { FastifyInstance, FastifyRequest, FastifyReply } from 'fastify'

declare module 'fastify' {
  interface FastifyInstance {
    // a decorated db client, a config object, and an auth guard
    db: import('pg').Pool
    config: { JWT_SECRET: string; PORT: number }
    authenticate: (request: FastifyRequest, reply: FastifyReply) => Promise<void>
  }
  interface FastifyRequest {
    // user attached by an auth hook
    user?: { id: number; role: 'admin' | 'user' }
  }
  interface FastifyReply {
    // custom reply decorator
    sendError: (code: number, message: string) => void
  }
}
```

Now `app.db`, `request.user`, and `reply.sendError(...)` are fully typed across the whole codebase.

### A typed `FastifyPluginAsync`

```typescript
import type { FastifyPluginAsync } from 'fastify'
import type { FastifyPluginAsyncTypebox } from '@fastify/type-provider-typebox'

// Generic typed plugin with its own options type:
const plugin: FastifyPluginAsync<{ verbose?: boolean }> = async (app, opts) => {
  app.get('/ping', async () => ({ pong: true, verbose: opts.verbose ?? false }))
}

// TypeBox-aware plugin (inherits the type provider automatically):
const tbPlugin: FastifyPluginAsyncTypebox = async (app) => {
  // app already carries the TypeBox provider — schemas infer types here too
}
```

**Security note (types).** Types are a *developer-time* guard, not a *runtime* one. TypeScript types vanish at runtime; a malicious client can send anything. Never rely on a TS type alone to keep bad data out — the **runtime JSON Schema** is the real boundary. The value of type providers is that they keep your types and your schema in lock-step, so the thing you validate is exactly the thing you typed.

---

## 7. The Plugin System — Encapsulation & Decorators

Everything you build in Fastify is a **plugin**. Plugins give you **encapsulation**: hooks, decorators, and schemas registered inside a plugin are isolated to that plugin and its children. This is Fastify's answer to dependency injection — lighter than a DI container, but with the same payoff: shared services declared once, used everywhere they should be, and nowhere they shouldn't.

### Anatomy of a plugin

```typescript
import type { FastifyPluginAsync } from 'fastify'

// A plugin is just an (async) function receiving the instance and its options.
const myPlugin: FastifyPluginAsync<{ greeting: string }> = async (app, opts) => {
  app.get('/hello', async () => ({ msg: opts.greeting }))
}

// Register it; options are passed as the 2nd argument.
app.register(myPlugin, { greeting: 'hi' })
```

### Encapsulation — the core concept

When you `register` a plugin, Fastify creates a **child context**. Decorators, hooks, and schemas added in that child are visible to the child and its descendants — but **not** to the parent or to sibling plugins. This means a hook you add inside a route plugin cannot accidentally affect unrelated routes.

```typescript
app.register(async (child) => {
  // This decorator + hook only exist INSIDE this child context.
  child.decorate('secret', 42)
  child.addHook('onRequest', async () => { /* runs only for THIS plugin's routes */ })
  child.get('/inside', async () => ({ secret: (child as any).secret })) // works
})

// Out here, app.secret is UNDEFINED — it did not leak out.
// app.get('/outside', async () => ({ secret: (app as any).secret })) // would be undefined!
```

```
        app (root context)
          │  decorate('a')      ← visible to EVERYTHING below
          ├── plugin X (child)
          │      decorate('b')  ← visible only inside X and X's children
          │      └── plugin X1  ← sees 'a' and 'b'
          └── plugin Y (child)
                 decorate('c')  ← visible only inside Y
                                  (Y does NOT see 'b'; X does NOT see 'c')
```

### Breaking encapsulation with `fastify-plugin` (fp)

Sometimes you *want* a decorator/hook to be visible to the parent — for example a shared database connection or an `authenticate` guard that every feature needs. Wrap the plugin in `fastify-plugin` (`fp`) so it executes in the *parent's* context instead of creating a new child.

```bash
npm install fastify-plugin
```

```typescript
import fp from 'fastify-plugin'
import type { FastifyInstance } from 'fastify'

async function dbConnector(app: FastifyInstance) {
  const pool = createPool()
  app.decorate('db', pool)                                  // attach a shared singleton
  app.addHook('onClose', async () => { await pool.end() })  // clean up on shutdown
}

// Wrapping in fp() makes `app.db` available to the PARENT and all siblings.
export default fp(dbConnector, {
  name: 'db',                  // unique name (used by dependency resolution)
  fastify: '5.x',              // declare the supported Fastify version range
  // dependencies: ['config']  // ensure other named plugins load first
})
```

**Rule of thumb:** use `fp()` for **infrastructure** (db, auth, config, metrics) that the whole app shares. Do **not** wrap **route plugins** in `fp()` — you *want* their hooks and schemas encapsulated so they cannot leak into other features.

### Decorators — decorate / decorateRequest / decorateReply

Decorators attach properties/methods to the instance, the request, or the reply. They are how you share services and per-request data without globals.

```typescript
// 1) Decorate the INSTANCE (shared singletons: config, db, services)
app.decorate('config', { JWT_SECRET: 'x', PORT: 3000 })
app.config.JWT_SECRET // accessible anywhere in this encapsulation context

// 2) Decorate the REQUEST (per-request data, e.g. the authenticated user)
// ⚡ Initialize with null/primitive/getter — v5 FORBIDS decorating with a
//    reference-type value directly (it would be SHARED across all requests).
app.decorateRequest('user', null)
app.addHook('preHandler', async (request) => {
  ;(request as any).user = { id: 1, role: 'admin' } // set per request
})

// 3) Decorate the REPLY (custom response helpers)
app.decorateReply('sendError', function (this: any, code: number, message: string) {
  this.code(code).send({ error: message })
})
app.get('/x', async (req, reply) => {
  if (!req.headers.authorization) return (reply as any).sendError(401, 'no auth')
  return { ok: true }
})
```

**⚡ Version note (decorateRequest/decorateReply):** In Fastify v5, decorating a request/reply with an **object or array value** is blocked, because the same reference would be shared across every request — a serious data-leak/concurrency bug (one user's data bleeding into another's). Decorate with a **primitive, `null`, or a getter function**, then assign the real per-request value inside a hook. For objects, the pattern is `app.decorateRequest('user', null)` then `request.user = ...` in `preHandler`.

### Checking for and depending on decorators

```typescript
// Guard against missing dependencies (fail fast at boot, not at request time):
if (!app.hasDecorator('db')) {
  throw new Error('db plugin must be registered before this one')
}

// fastify-plugin `dependencies` enforces load order automatically and clearly:
export default fp(authPlugin, { name: 'auth', dependencies: ['db', 'config'] })
```

### Register options & overrides

```typescript
app.register(somePlugin, {
  prefix: '/v1',        // route prefix for routes inside the plugin
  logLevel: 'warn',     // override the log level for this context
  // ...any custom options your plugin reads from `opts`
  featureFlag: true,
})
```

**Security note (plugins/decorators).** The decorateRequest-with-objects ban exists *because of* a real cross-request data-leak class. Respect it. Use `fp` only for things that are genuinely shared and safe to share — never use it to hoist a route's auth bypass into the parent. Encapsulation is also a security feature: keeping each feature's hooks/schemas isolated means a permissive change in one plugin cannot silently weaken another.

---

## 8. Core Plugin Ecosystem

Fastify's official plugins live under the `@fastify/*` scope and are version-matched to core v5. Install only what you need. Each plugin below is registered with `await app.register(...)` because registration is asynchronous.

```bash
npm install @fastify/cors @fastify/helmet @fastify/jwt @fastify/cookie \
  @fastify/session @fastify/rate-limit @fastify/static @fastify/multipart \
  @fastify/swagger @fastify/swagger-ui @fastify/websocket @fastify/under-pressure \
  @fastify/compress
```

### @fastify/cors — Cross-Origin Resource Sharing

**What/why.** Browsers block JavaScript from reading responses from a different origin unless the server opts in with CORS headers. This plugin sets those headers and answers preflight `OPTIONS` requests.

```typescript
import cors from '@fastify/cors'

await app.register(cors, {
  origin: ['https://app.example.com'], // string | string[] | RegExp | function | true(all)
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  credentials: true,                   // allow cookies/Authorization across origins
  maxAge: 86400,                       // cache the preflight for 24h
})
```

**Security note.** Never use `origin: true` (reflect any origin) together with `credentials: true` in production — that combination lets *any* website make authenticated requests on a logged-in user's behalf. Pin an explicit allowlist of origins. Treat CORS as a browser convenience, not a server access control: it does not protect non-browser clients.

### @fastify/helmet — security headers

**What/why.** Sets a bundle of HTTP security headers (CSP, HSTS, X-Frame-Options, X-Content-Type-Options, etc.) that mitigate XSS, clickjacking, and MIME-sniffing. This is the single highest-value security plugin to add.

```typescript
import helmet from '@fastify/helmet'

await app.register(helmet, {
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      imgSrc: ["'self'", 'data:'],
    },
  },
  // For pure JSON APIs (no HTML), you often disable CSP and keep the rest:
  // contentSecurityPolicy: false,
})
```

**Security note.** Enable helmet in every production app. For HTML-serving apps, invest in a tight Content-Security-Policy — it is the strongest single defense against XSS. Ensure HSTS is on when serving over HTTPS so browsers refuse to downgrade to HTTP.

### @fastify/jwt — JSON Web Tokens

**What/why.** Signs and verifies JWTs and adds `request.jwtVerify()`. JWTs are a stateless way to carry an authenticated identity in a signed token; see §9 for the full auth flow.

```typescript
import jwt from '@fastify/jwt'

await app.register(jwt, {
  secret: process.env.JWT_SECRET!, // HS256 symmetric secret; or { private, public } for RS256
  sign: { expiresIn: '15m' },      // default sign options — keep access tokens SHORT
})

// Sign a token (e.g. on login):
app.post('/login', async (request, reply) => {
  // ...verify credentials first...
  const token = app.jwt.sign({ id: 1, role: 'admin' })
  return { token }
})

// Decorate a reusable auth guard:
app.decorate('authenticate', async (request: any, reply: any) => {
  try {
    await request.jwtVerify() // reads Authorization: Bearer <token>, sets request.user
  } catch {
    reply.code(401).send({ error: 'unauthorized' })
  }
})

app.get('/me', { preHandler: [app.authenticate] }, async (request: any) => request.user)
```

**Security note.** Use a long, random `JWT_SECRET` (≥32 bytes) kept out of source control. Keep access tokens short-lived (5–15 min) and pair them with refresh tokens (§9). Put **no secrets** in the payload — JWTs are signed, not encrypted; anyone can base64-decode them. For multi-service setups prefer RS256 (asymmetric) so services can verify without holding the signing key.

### @fastify/cookie

```typescript
import cookie from '@fastify/cookie'

await app.register(cookie, {
  secret: process.env.COOKIE_SECRET, // enables signed cookies (tamper detection)
})

app.get('/set', async (request, reply) => {
  reply.setCookie('sid', 'abc123', {
    path: '/',
    httpOnly: true,    // not readable by JS → mitigates XSS token theft
    secure: true,      // sent only over HTTPS
    sameSite: 'lax',   // CSRF mitigation
    signed: true,      // sign with the secret above
    maxAge: 3600,
  })
  return { ok: true }
})

app.get('/get', async (request) => {
  const raw = request.cookies.sid
  const { valid, value } = request.unsignCookie(raw ?? '')
  return { valid, value }
})
```

**Security note.** For any cookie carrying identity, set `httpOnly`, `secure`, and `sameSite` (`lax` or `strict`). `httpOnly` keeps XSS from stealing the cookie; `secure` keeps it off plaintext HTTP; `sameSite` is your first line of CSRF defense.

### @fastify/session — server-side sessions

```typescript
import cookie from '@fastify/cookie'
import session from '@fastify/session'

await app.register(cookie)
await app.register(session, {
  secret: process.env.SESSION_SECRET!, // must be at least 32 chars
  cookie: { secure: true, httpOnly: true, sameSite: 'lax', maxAge: 86_400_000 },
  // store: new RedisStore(...) // use a real store in production (default is in-memory!)
})

app.post('/login', async (request: any, reply) => {
  request.session.user = { id: 1 } // persisted server-side; only an id rides in the cookie
  return { ok: true }
})
```

**Security note.** The default in-memory store loses sessions on restart and does not work across multiple instances — use Redis or a database store in production. Regenerate the session id on login/privilege change to prevent session fixation.

### @fastify/rate-limit

```typescript
import rateLimit from '@fastify/rate-limit'

await app.register(rateLimit, {
  max: 100,                 // max requests per window per key (default key: IP)
  timeWindow: '1 minute',
  // redis: new Redis(...)  // share limits across instances in production
})

// Per-route override (tighter limit on an expensive/sensitive endpoint):
app.post('/login', {
  config: { rateLimit: { max: 5, timeWindow: '1 minute' } },
}, async () => ({ ok: true }))
```

**Security note.** Rate limiting is essential against brute-force (login), credential-stuffing, and scraping. Apply a strict limit to auth endpoints specifically. Use a Redis store so the limit is shared across instances — an in-memory limit is trivially bypassed when you run multiple replicas. If behind a proxy, configure `trustProxy` so the real client IP is used as the key (otherwise everyone shares the proxy's IP).

### @fastify/static — serve files

```typescript
import fastifyStatic from '@fastify/static'
import { fileURLToPath } from 'node:url'
import { dirname, join } from 'node:path'

// __dirname does not exist in ESM — derive it:
const __dirname = dirname(fileURLToPath(import.meta.url))

await app.register(fastifyStatic, {
  root: join(__dirname, '..', 'public'), // absolute path
  prefix: '/public/',                    // URL prefix → GET /public/logo.png
})

app.get('/home', async (request, reply) => reply.sendFile('index.html')) // relative to `root`
```

**Security note.** `@fastify/static` sanitizes paths to prevent traversal — prefer it over hand-rolled file serving. Keep `root` pointed at a dedicated public directory that contains nothing sensitive (no `.env`, source, or config files).

### @fastify/multipart — file uploads

```typescript
import multipart from '@fastify/multipart'
import { pipeline } from 'node:stream/promises'
import { createWriteStream } from 'node:fs'

await app.register(multipart, {
  limits: { fileSize: 10 * 1024 * 1024, files: 5 }, // 10 MB/file, 5 files max
})

// Stream a single file to disk (memory-efficient — never buffer the whole file):
app.post('/upload', async (request, reply) => {
  const data = await request.file() // get the first file part
  if (!data) return reply.code(400).send({ error: 'no file' })
  await pipeline(data.file, createWriteStream(`./uploads/${data.filename}`))
  return { filename: data.filename, mimetype: data.mimetype }
})

// Multiple files + fields:
app.post('/upload-many', async (request) => {
  const saved: string[] = []
  for await (const part of request.parts()) {
    if (part.type === 'file') {
      await pipeline(part.file, createWriteStream(`./uploads/${part.filename}`))
      saved.push(part.filename)
    } else {
      request.log.info({ field: part.fieldname, value: part.value })
    }
  }
  return { saved }
})
```

**⚡ Gotcha:** If you do not consume a file's stream, the request hangs/leaks. Always pipe it or call `part.toBuffer()`. Setting `attachFieldsToBody: true` lets you read files via `request.body`.

**Security note.** Always set `limits` (`fileSize`, `files`) — unbounded uploads are a denial-of-service vector. **Never** trust the client-supplied `filename`: sanitize it (strip path separators) or generate your own name, or an attacker writes outside your upload dir via traversal. Validate the real content type, not just the declared `mimetype`. Store uploads outside the web root and consider scanning them.

### @fastify/swagger + @fastify/swagger-ui — OpenAPI docs for free

Because routes already carry JSON Schema, OpenAPI generation is nearly automatic.

```typescript
import swagger from '@fastify/swagger'
import swaggerUi from '@fastify/swagger-ui'

await app.register(swagger, {
  openapi: {
    info: { title: 'My API', version: '1.0.0' },
    components: {
      securitySchemes: { bearerAuth: { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' } },
    },
  },
})
await app.register(swaggerUi, { routePrefix: '/docs' }) // interactive docs at /docs

// Routes auto-document from their schema. Add tags/summary for nicer output:
app.get('/users/:id', {
  schema: {
    tags: ['users'],
    summary: 'Get a user by id',
    security: [{ bearerAuth: [] }],
    params: { type: 'object', properties: { id: { type: 'integer' } } },
    response: { 200: { type: 'object', properties: { id: { type: 'integer' } } } },
  },
}, async (req) => ({ id: Number((req.params as any).id) }))
```

**Security note.** Do not expose the Swagger UI publicly in production unless intended — it advertises your entire API surface to attackers. Gate `/docs` behind auth or disable it in production.

### @fastify/websocket

```typescript
import websocket from '@fastify/websocket'

await app.register(websocket)

app.get('/ws', { websocket: true }, (socket /* ws WebSocket */, req) => {
  socket.on('message', (message) => socket.send(`echo: ${message.toString()}`))
  socket.on('close', () => req.log.info('ws closed'))
})
```

**⚡ Version note:** In Fastify v5 the handler signature is `(socket, request)` — the first arg is the raw `ws` WebSocket (v4 passed a `{ socket }` wrapper). Update old handlers.

**Security note.** Authenticate the connection during the upgrade (check a token in `onRequest`/the query), validate every incoming message (they bypass your route schemas), and apply your own message-rate limits — a socket can flood you outside the HTTP rate limiter.

### @fastify/under-pressure — overload protection

```typescript
import underPressure from '@fastify/under-pressure'

await app.register(underPressure, {
  maxEventLoopDelay: 1000,          // ms; reject if the event loop lags too much
  maxHeapUsedBytes: 1_000_000_000,
  maxRssBytes: 1_500_000_000,
  maxEventLoopUtilization: 0.98,
  message: 'Server under pressure, try again later',
  retryAfter: 50,
  exposeStatusRoute: '/status',     // GET /status → { status: 'ok' } (health check)
})
```

This is *load shedding*: when the process is overwhelmed, it returns `503` quickly instead of slowly degrading for everyone — a resilience and availability safeguard.

### @fastify/compress

```typescript
import compress from '@fastify/compress'
await app.register(compress, { global: true }) // gzip/brotli responses based on Accept-Encoding
```

**Security note.** Be cautious compressing responses that mix secret and attacker-influenced content over a confidential channel (BREACH attack class). For most JSON APIs this is fine; for sensitive HTML with reflected input, consider disabling compression on those responses.

### Plugin quick reference

| Plugin | Purpose |
|---|---|
| `@fastify/cors` | CORS headers / preflight |
| `@fastify/helmet` | Security headers (add this!) |
| `@fastify/jwt` | JWT sign/verify + `request.jwtVerify()` |
| `@fastify/cookie` | Read/write (signed) cookies |
| `@fastify/session` | Server-side sessions (needs a store in prod) |
| `@fastify/rate-limit` | Request throttling |
| `@fastify/static` | Serve static assets, `reply.sendFile` |
| `@fastify/multipart` | `multipart/form-data` file uploads |
| `@fastify/swagger` (+ ui) | OpenAPI spec + Swagger UI from schemas |
| `@fastify/websocket` | WebSocket routes |
| `@fastify/under-pressure` | Load shedding / health |
| `@fastify/compress` | gzip/brotli responses |
| `@fastify/autoload` | Auto-register plugins/routes from folders (§16) |
| `@fastify/env` | Validate & type environment config (§15) |
| `@fastify/formbody` | Parse `application/x-www-form-urlencoded` |
| `@fastify/middie` | Run Express-style middleware |
| `@fastify/sensible` | `httpErrors`, sensible defaults & helpers |

---

## 9. Authentication & Authorization

**Authentication** answers "who are you?"; **authorization** answers "are you allowed to do this?". Keep them distinct: authenticate early (cheap, request-wide), authorize per-action (role/ownership checks). The idiomatic Fastify pattern packages both as an `fp`-wrapped plugin that decorates the instance with reusable guards, so every feature can opt in with a one-line `preHandler`.

### JWT auth flow end-to-end

```typescript
// src/plugins/auth.ts
import fp from 'fastify-plugin'
import jwt from '@fastify/jwt'
import type { FastifyInstance, FastifyReply, FastifyRequest } from 'fastify'

// Augment types so request.user / app.authenticate are known everywhere.
declare module 'fastify' {
  interface FastifyInstance {
    authenticate: (req: FastifyRequest, reply: FastifyReply) => Promise<void>
    authorize: (roles: string[]) => (req: FastifyRequest, reply: FastifyReply) => Promise<void>
  }
}
declare module '@fastify/jwt' {
  interface FastifyJWT {
    payload: { id: number; role: 'admin' | 'user' } // what we sign
    user: { id: number; role: 'admin' | 'user' }    // what request.user becomes
  }
}

export default fp(async (app: FastifyInstance) => {
  await app.register(jwt, {
    secret: process.env.JWT_SECRET!,
    sign: { expiresIn: '15m' }, // short-lived access tokens
  })

  // Guard: verify the bearer token, populate request.user, else 401.
  app.decorate('authenticate', async (request, reply) => {
    try {
      await request.jwtVerify()
    } catch {
      reply.code(401).send({ error: 'Unauthorized' })
    }
  })

  // RBAC guard FACTORY: returns a preHandler that checks role membership.
  app.decorate('authorize', (roles: string[]) => {
    return async (request, reply) => {
      // assumes authenticate ran first, so request.user is set
      if (!request.user || !roles.includes(request.user.role)) {
        reply.code(403).send({ error: 'Forbidden' })
      }
    }
  })
}, { name: 'auth' })
```

Use it in routes:

```typescript
// Login issues a token after verifying credentials:
app.post('/login', {
  schema: {
    body: { type: 'object', required: ['email', 'password'], additionalProperties: false,
      properties: { email: { type: 'string' }, password: { type: 'string' } } },
  },
}, async (request, reply) => {
  const { email, password } = request.body as { email: string; password: string }
  const user = await verifyCredentials(email, password) // your logic (bcrypt/argon2 compare)
  if (!user) return reply.code(401).send({ error: 'bad credentials' })
  const token = app.jwt.sign({ id: user.id, role: user.role })
  return { token }
})

// Any authenticated user:
app.get('/me', { preHandler: [app.authenticate] }, async (request) => request.user)

// Admin only — run authenticate FIRST, then authorize (order matters):
app.delete('/users/:id', {
  preHandler: [app.authenticate, app.authorize(['admin'])],
}, async () => ({ deleted: true }))
```

### Where to put auth in the lifecycle

- Cheap, request-wide checks (IP allowlists, API-key presence, token verification): `onRequest` or `preHandler`.
- Role/ownership checks (RBAC, "can this user edit *this* resource?"): `preHandler`, after validation, so the user *and* the validated input are both available.

### Password storage

Never store plaintext or reversibly-encrypted passwords. Hash with a slow, salted algorithm.

```typescript
import argon2 from 'argon2' // or bcrypt
// On register:
const hash = await argon2.hash(plainPassword)
// On login:
const ok = await argon2.verify(hash, suppliedPassword) // constant-time comparison
```

### Refresh-token pattern (sketch)

Access tokens are short-lived (so a leak expires fast); a longer-lived refresh token (stored in an httpOnly cookie and tracked server-side so it can be revoked) mints new access tokens.

```typescript
app.post('/refresh', async (request: any, reply) => {
  const refresh = request.cookies.refresh
  const { valid } = request.unsignCookie(refresh ?? '')
  if (!valid) return reply.code(401).send({ error: 'invalid refresh' })
  // ...look up the refresh token in your store, confirm it is not revoked, ROTATE it...
  const access = app.jwt.sign({ id: 1, role: 'user' })
  return { token: access }
})
```

**Security note (auth — read twice).**
- Hash passwords with **argon2** or **bcrypt**; never roll your own.
- Compare credentials and tokens in **constant time** (the hashing libs do this); a plain `===` on secrets can leak via timing.
- Keep access tokens **short-lived**; use **rotating, revocable refresh tokens** for longevity.
- Put **no secrets** in JWT payloads (they are readable). Sign with a strong secret and validate `exp`.
- Enforce **least privilege** in `authorize`; default to deny.
- Authorization must check **ownership**, not just role — a `user` editing `/posts/:id` must own that post (an *insecure direct object reference* / IDOR bug otherwise).
- Rate-limit `/login`, `/refresh`, and `/register` to blunt brute-force (see §8).

---

## 10. Error Handling

### Default behavior (and why it is convenient)

If a handler or hook throws (or rejects), Fastify catches it, logs it, and sends an error response automatically. Validation errors → `400`; uncaught errors → `500`. Because of this you rarely need `try/catch` in handlers — you just `throw`, and the framework turns it into a response. This keeps handlers clean and ensures no error silently escapes.

### Custom global error handler

```typescript
app.setErrorHandler((error, request, reply) => {
  // Validation errors carry `.validation` (the raw Ajv array).
  if (error.validation) {
    return reply.code(400).send({
      error: 'ValidationError',
      message: error.message,
      details: error.validation,
    })
  }

  // Errors with an explicit statusCode are passed through.
  const status = error.statusCode ?? 500
  if (status >= 500) {
    request.log.error({ err: error }, 'unhandled error') // log server errors with full stack
  }

  reply.code(status).send({
    error: error.name,
    // ⚡ Hide internals for 5xx — never leak stack traces / messages to clients.
    message: status >= 500 ? 'Internal Server Error' : error.message,
  })
})
```

### Custom error classes with `@fastify/error`

```typescript
import createError from '@fastify/error'

// Define a reusable typed error: code, message template, default status.
const NotFoundError = createError('ERR_NOT_FOUND', 'Resource %s not found', 404)

app.get('/widgets/:id', async (request) => {
  const widget = await findWidget((request.params as any).id)
  if (!widget) throw new NotFoundError((request.params as any).id) // → 404 with code
  return widget
})
```

Or throw a plain `Error` with a `statusCode`:

```typescript
const err = new Error('Payment required') as Error & { statusCode: number }
err.statusCode = 402
throw err
```

### Not-found handler

```typescript
app.setNotFoundHandler((request, reply) => {
  reply.code(404).send({ error: 'NotFound', path: request.url })
})
```

### Error-handling scoping

`setErrorHandler` and `setNotFoundHandler` respect encapsulation (§7): set them inside a plugin to handle errors only for that plugin's routes. The most specific (innermost) handler wins, so an `/api` plugin can shape its errors differently from the rest of the app.

```typescript
app.register(async (api) => {
  api.setErrorHandler((err, req, reply) => {
    reply.code(err.statusCode ?? 500).send({ scope: 'api', message: err.message })
  })
  api.get('/boom', async () => { throw new Error('kaboom') })
}, { prefix: '/api' })
```

**Security note (errors).** Never return stack traces, raw exception messages, SQL errors, or file paths to clients on a 5xx — they leak internal structure that aids attackers. Log the full error server-side (with the request id) and return a generic message. *Client* errors (4xx) can be descriptive since they reflect the client's own bad input. Make sure errors thrown deep in async code are not swallowed — Fastify catches thrown/rejected errors in handlers and hooks, but a detached `setTimeout` or unawaited promise can crash the process (see the Node.js guide on `unhandledRejection`).

---

## 11. Logging with Pino

Fastify ships **Pino** as its logger — extremely fast, structured, JSON-by-default. Structured (JSON) logs are machine-parseable, so your log aggregator can index and search fields like `reqId`, `status`, and `userId`. This is why you log with `request.log.info({ ...fields }, 'message')` rather than `console.log` string concatenation.

### Enabling and configuring

```typescript
const app = Fastify({
  logger: {
    level: 'info', // trace | debug | info | warn | error | fatal | silent
    // Pretty-print ONLY in development (requires `pino-pretty` installed):
    transport: process.env.NODE_ENV === 'development'
      ? { target: 'pino-pretty', options: { colorize: true, translateTime: 'HH:MM:ss' } }
      : undefined,
  },
})
```

In production leave `transport` off so logs are raw JSON (fast and aggregator-friendly). Install `pino-pretty` as a dev dependency only.

### The per-request child logger

Every request gets `request.log`, a Pino **child logger** automatically tagged with a unique `reqId`. Always use it (not `console.log`) so all log lines for one request can be correlated.

```typescript
app.get('/order/:id', async (request) => {
  request.log.info({ orderId: (request.params as any).id }, 'fetching order')
  // Every line includes reqId, so you can trace one request end to end.
  return { ok: true }
})
```

### Customizing request/response log fields (serializers)

```typescript
const app = Fastify({
  logger: {
    serializers: {
      req(request) { return { method: request.method, url: request.url, id: request.id } },
      res(reply) { return { statusCode: reply.statusCode } },
    },
  },
})
```

### Redaction — hide secrets in logs (do not skip this)

```typescript
const app = Fastify({
  logger: {
    redact: {
      paths: ['req.headers.authorization', 'req.headers.cookie', '*.password', '*.token'],
      censor: '[REDACTED]',
    },
  },
})
// Authorization headers, cookies, and any `password`/`token` field are masked in every line.
```

### Disabling or tuning auto request logging

```typescript
const app = Fastify({ logger: true, disableRequestLogging: true }) // log manually instead

app.addHook('onResponse', async (request, reply) => {
  request.log.info({ status: reply.statusCode, ms: reply.elapsedTime }, 'done')
})
```

### Custom request IDs (trace across services)

```typescript
import { randomUUID } from 'node:crypto'
const app = Fastify({
  logger: true,
  genReqId: (req) => (req.headers['x-request-id'] as string) ?? randomUUID(),
})
```

**Security note (logging).** Logs are a serious leak surface. **Always configure `redact`** for `authorization`, `cookie`, and any `password`/`token`/`secret`/`creditCard` fields — otherwise secrets land in plaintext in your log store. Do not log full request bodies on auth or payment endpoints. Be mindful of privacy regulations: avoid logging unnecessary personal data, and have a retention policy. Honoring an inbound `x-request-id` aids tracing but treat it as untrusted (do not let it be used to forge audit trails).

---

## 12. Database Integration Patterns

The idiomatic pattern is: connect once at boot, **decorate the instance** with the client via an `fp` plugin (so every route can use it), verify connectivity so the app fails fast on misconfiguration, and close the connection on shutdown via an `onClose` hook. See the **PostgreSQL** and **Prisma** guides for the database side; this section is about wiring.

### Postgres with `pg` (raw, parameterized)

```typescript
// src/plugins/db.ts
import fp from 'fastify-plugin'
import pg from 'pg'
import type { FastifyInstance } from 'fastify'

declare module 'fastify' {
  interface FastifyInstance { pg: pg.Pool }
}

export default fp(async (app: FastifyInstance) => {
  const pool = new pg.Pool({ connectionString: process.env.DATABASE_URL })
  await pool.query('SELECT 1')                              // fail fast if DB is unreachable
  app.decorate('pg', pool)
  app.addHook('onClose', async () => { await pool.end() })  // graceful cleanup on shutdown
}, { name: 'db' })
```

```typescript
// usage in a route — ALWAYS parameterize
app.get('/users/:id', async (request) => {
  const { rows } = await app.pg.query(
    'SELECT id, email FROM users WHERE id = $1',
    [(request.params as any).id], // parameterized — NEVER string-concat user input into SQL
  )
  return rows[0] ?? null
})
```

There is also an official `@fastify/postgres` plugin that does the decorate + pool management for you and exposes `app.pg.transact(...)`.

### Prisma

```typescript
// src/plugins/prisma.ts
import fp from 'fastify-plugin'
import { PrismaClient } from '@prisma/client'
import type { FastifyInstance } from 'fastify'

declare module 'fastify' {
  interface FastifyInstance { prisma: PrismaClient }
}

export default fp(async (app: FastifyInstance) => {
  const prisma = new PrismaClient()
  await prisma.$connect()
  app.decorate('prisma', prisma)
  app.addHook('onClose', async () => { await prisma.$disconnect() })
}, { name: 'prisma' })
```

```typescript
// usage
app.get('/users', async () => app.prisma.user.findMany({ select: { id: true, email: true } }))

app.post('/users', {
  schema: { body: { type: 'object', required: ['email'], additionalProperties: false,
    properties: { email: { type: 'string', format: 'email' } } } },
}, async (request, reply) => {
  const user = await app.prisma.user.create({ data: request.body as any })
  reply.code(201)
  return user
})
```

### Drizzle / Mongo / Redis

The same `fp` + `decorate` + `onClose` pattern applies to any client (Drizzle, `mongodb`, `ioredis`). Official plugins exist too: `@fastify/mongodb`, `@fastify/redis`. Always: (1) create the client in an `fp` plugin, (2) `app.decorate('<name>', client)`, (3) close it in `onClose`, (4) list its name in `dependencies` for plugins that need it so load order is enforced.

**Security note (databases).**
- **Always use parameterized queries / an ORM** — string-concatenating user input into SQL is the #1 web vulnerability (SQL injection). The schema validation in §5 is defense in depth, not a substitute.
- Use a DB account with **least privilege** (no DDL/superuser for the app's runtime user).
- Use a **`select` allowlist** (as above) so queries return only the columns you intend, and pair it with a response schema so secrets cannot leak.
- Keep `DATABASE_URL` in env/secret storage, never in source. Use **connection pooling** with sane limits to resist connection-exhaustion abuse.
- For NoSQL (Mongo), beware **operator injection** — validate that query inputs are the expected primitive types, not objects like `{ $ne: null }`.

---

## 13. Testing with node:test and app.inject()

Fastify's standout testing feature is **`app.inject()`**: it dispatches a *simulated* HTTP request through the **full lifecycle** (hooks, validation, serialization, error handling) **without opening a socket**. Tests are fast, deterministic, need no free port, and exercise exactly the code path a real request would. Combined with Node's built-in `node:test`, you need no extra test framework.

### Make the app factory testable

Export a *builder* that returns a configured but not-yet-listening instance. The entrypoint (`server.ts`) only calls `listen`. This separation is what lets tests build an app, inject requests, and close it cleanly.

```typescript
// src/app.ts
import Fastify, { type FastifyServerOptions } from 'fastify'
import userRoutes from './routes/users.js'

export function buildApp(opts: FastifyServerOptions = {}) {
  const app = Fastify(opts)
  app.register(userRoutes, { prefix: '/users' })
  return app
}
```

```typescript
// src/server.ts (entrypoint just starts it)
import { buildApp } from './app.js'
const app = buildApp({ logger: true })
await app.listen({ port: 3000, host: '0.0.0.0' })
```

### Tests with the built-in `node:test`

```typescript
// test/users.test.ts
import { test, after, before } from 'node:test'
import assert from 'node:assert/strict'
import { buildApp } from '../src/app.js'

const app = buildApp() // logger off → quiet tests

before(async () => { await app.ready() }) // wait for all plugins to finish loading
after(async () => { await app.close() })  // clean shutdown → onClose hooks (db disconnect) run

test('GET /users returns 200', async () => {
  const res = await app.inject({ method: 'GET', url: '/users' })
  assert.equal(res.statusCode, 200)
  assert.equal(res.headers['content-type'], 'application/json; charset=utf-8')
  assert.ok(Array.isArray(res.json())) // res.json() parses the body
})

test('POST /users validates body', async () => {
  const res = await app.inject({
    method: 'POST',
    url: '/users',
    payload: { age: 5 }, // missing required "email"
  })
  assert.equal(res.statusCode, 400)
  assert.match(res.json().message, /email/)
})

test('POST /users with auth header', async () => {
  const res = await app.inject({
    method: 'POST',
    url: '/users',
    headers: { authorization: 'Bearer test-token' },
    payload: { email: 'a@b.com', age: 30 },
  })
  assert.equal(res.statusCode, 201)
})
```

Run:

```bash
node --import tsx --test test/**/*.test.ts   # run TS tests directly
# Or compile first, then: node --test
```

### The inject response object

| Property | What it is |
|---|---|
| `res.statusCode` | Numeric status |
| `res.headers` | Response headers object |
| `res.body` | Raw string body |
| `res.json()` | Parsed JSON body |
| `res.payload` | Alias for `res.body` |
| `res.cookies` | Parsed Set-Cookie array |

**⚡ Best practice:** Always `await app.ready()` before injecting (or use the `before` hook) and `await app.close()` after, so async plugin loading completes and `onClose` hooks run. Test the security-relevant status codes explicitly: 400 (bad input), 401 (unauthenticated), 403 (unauthorized), 404 (missing) — not just the happy 200 path.

**Security note (testing).** Write tests that *assert* your authorization boundaries: that a non-admin gets 403 on admin routes, that user A cannot read user B's resource (IDOR), that oversized bodies are rejected, and that unknown fields are refused. Tests are how these guarantees survive future refactors.

---

## 14. Performance — Why Fastify Is Fast

### The sources of speed (the logic)

1. **Schema-compiled serialization.** `fast-json-stringify` turns a response schema into a purpose-built function at boot — no per-field runtime type inspection.
2. **Schema-compiled validation.** Ajv compiles each schema into optimized JS once at boot, not per request.
3. **A fast radix-tree router** (`find-my-way`) with lookups proportional to path length, independent of route count.
4. **Minimal abstraction over Node's `http`.** The request/reply wrappers are deliberately thin.
5. **Hot-path discipline** internally — reusing objects and avoiding allocations where it counts.

### Practical tips to stay fast

```typescript
// 1) ALWAYS add a response schema to hot endpoints — the single biggest win.
app.get('/fast', {
  schema: { response: { 200: { type: 'object', properties: { ok: { type: 'boolean' } } } } },
}, async () => ({ ok: true }))

// 2) Prefer async/await hooks; avoid unnecessary preHandlers in hot paths.

// 3) Reuse shared schemas via $ref instead of inlining big duplicate schemas.

// 4) Tune keep-alive / limits for high-throughput services:
const app2 = Fastify({
  keepAliveTimeout: 72_000,  // ms; should exceed your load balancer's idle timeout
  bodyLimit: 1_048_576,      // 1 MB default body cap; raise/lower per need
  requestTimeout: 30_000,    // bound how long a request may take (also a DoS guard)
})
```

### Benchmarking

```bash
# autocannon: a Node load-testing tool from the Fastify authors
npx autocannon -c 100 -d 10 http://localhost:3000/fast
# -c 100 connections, -d 10 seconds. Reports req/s and latency percentiles.
```

Compare a route **with** vs **without** a response schema to see the serialization win firsthand. Profile with `node --prof` or `clinic.js` (`npx clinic doctor -- node dist/server.js`) when latency spikes.

### Avoid these performance traps

| Trap | Fix |
|---|---|
| No response schema on hot routes | Add one — uses fast-json-stringify |
| Logging at `debug`/`trace` in prod | Use `info`+ in prod |
| `pino-pretty` transport in prod | Only use it in dev |
| Huge bodies parsed needlessly | Set `bodyLimit`, stream large uploads |
| Blocking the event loop (sync crypto, big `JSON.parse`) | Offload to worker threads / streams (see Node.js guide) |
| Per-request object allocation in hooks | Reuse, minimize work |

**Security note (performance).** Performance and security overlap: `bodyLimit`, `requestTimeout`, `@fastify/rate-limit`, and `@fastify/under-pressure` are all both performance *and* denial-of-service defenses. An endpoint that allocates unbounded memory or blocks the event loop is a DoS waiting to happen. Bound everything attacker-controllable.

---

## 15. Configuration, Graceful Shutdown, Deployment & Clustering

### Typed environment config with `@fastify/env`

Validate environment variables at boot with a JSON Schema so the app **refuses to start** if config is missing/invalid — far better than crashing later on a missing secret.

```bash
npm install @fastify/env
```

```typescript
import fp from 'fastify-plugin'
import fastifyEnv from '@fastify/env'
import type { FastifyInstance } from 'fastify'

const schema = {
  type: 'object',
  required: ['PORT', 'JWT_SECRET', 'DATABASE_URL'],
  properties: {
    PORT: { type: 'number', default: 3000 },
    JWT_SECRET: { type: 'string', minLength: 32 }, // enforce a strong secret
    DATABASE_URL: { type: 'string' },
    NODE_ENV: { type: 'string', default: 'development' },
  },
} as const

declare module 'fastify' {
  interface FastifyInstance {
    config: { PORT: number; JWT_SECRET: string; DATABASE_URL: string; NODE_ENV: string }
  }
}

export default fp(async (app: FastifyInstance) => {
  await app.register(fastifyEnv, { schema, dotenv: true }) // dotenv loads .env automatically
  // app.config is now validated & typed; the app fails to boot on invalid env.
}, { name: 'config' })
```

### Graceful shutdown

On `SIGTERM`/`SIGINT`, stop accepting new connections, let in-flight requests finish, run `onClose` hooks (db disconnect, flush), then exit. This prevents dropped requests and corrupted state during deploys. (See the Node.js guide for signal-handling background.)

```typescript
import { buildApp } from './app.js'

const app = buildApp({ logger: true })
await app.listen({ port: app.config.PORT, host: '0.0.0.0' })

const shutdown = async (signal: string) => {
  app.log.info({ signal }, 'shutting down')
  await app.close() // waits for in-flight requests + runs onClose hooks
  process.exit(0)
}
process.on('SIGTERM', () => shutdown('SIGTERM'))
process.on('SIGINT', () => shutdown('SIGINT'))
```

For automatic handling, `close-with-grace` adds a forced-exit timeout safety net:

```typescript
import closeWithGrace from 'close-with-grace'
closeWithGrace({ delay: 10_000 }, async ({ err }) => {
  if (err) app.log.error(err)
  await app.close()
})
```

### Deployment with Docker

```dockerfile
# Dockerfile — multi-stage for a small, secure production image
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci                 # reproducible install from lockfile
COPY . .
RUN npm run build          # tsc → dist/

FROM node:22-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev      # only production deps in the final image
COPY --from=build /app/dist ./dist
EXPOSE 3000
USER node                  # run as non-root for security
CMD ["node", "dist/server.js"]
```

```
# .dockerignore
node_modules
dist
.git
*.log
.env
```

**⚡ Remember:** Inside containers, always `listen({ host: '0.0.0.0' })` — the default `localhost` is unreachable from outside the container.

### Clustering / scaling

Node runs one thread per process. To use all CPU cores, run multiple processes:

```typescript
// src/cluster.ts — fork one worker per CPU
import cluster from 'node:cluster'
import { availableParallelism } from 'node:os'

if (cluster.isPrimary) {
  const cpus = availableParallelism()
  for (let i = 0; i < cpus; i++) cluster.fork()
  cluster.on('exit', (worker) => {
    console.log(`worker ${worker.process.pid} died; restarting`)
    cluster.fork()
  })
} else {
  await import('./server.js') // each worker runs the full app
}
```

In practice, prefer an external process manager (PM2) or orchestrator (Kubernetes running N replicas) — simpler, with restarts and rolling deploys handled for you. When you run multiple instances, move shared state (rate limits, sessions) to **Redis** so it is consistent across replicas.

**Security note (deployment).**
- Run as **non-root** in containers; use **multi-stage builds** so source and dev deps stay out of the final image.
- Add **`.env` to `.dockerignore`** and `.gitignore`; inject secrets via the orchestrator's secret store, never bake them into images.
- Enable `requestTimeout` and keep `bodyLimit` tight in production.
- Run `npm audit` / Dependabot to catch vulnerable dependencies.
- Validate env at boot (`@fastify/env`) so a misconfigured deploy fails fast instead of running insecurely.
- Terminate TLS at a trusted proxy/LB and set `trustProxy` accordingly so client IPs (and rate limiting) are accurate.

---

## 16. Project Structure & @fastify/autoload

`@fastify/autoload` registers every plugin/route file in a directory automatically, enforcing a clean, convention-based layout and removing manual `register` boilerplate. The structure encodes the §7 rule: `plugins/` holds shared infrastructure (`fp`-wrapped), `routes/` holds encapsulated features.

```bash
npm install @fastify/autoload
```

### Recommended layout

```
src/
├── app.ts               # builds the Fastify instance, wires autoload
├── server.ts            # entrypoint: build app + listen + graceful shutdown
├── plugins/             # CROSS-CUTTING plugins (loaded first, usually fp-wrapped)
│   ├── env.ts           # @fastify/env config
│   ├── db.ts            # database decorator
│   ├── auth.ts          # jwt + authenticate/authorize decorators
│   └── sensible.ts      # @fastify/sensible helpers
├── routes/              # FEATURE routes (auto-prefixed by folder name)
│   ├── users/
│   │   ├── index.ts     # → /api/users
│   │   └── schemas.ts
│   └── health/
│       └── index.ts     # → /api/health
└── types/
    └── fastify.d.ts     # module augmentation for decorators
```

### Wiring autoload

```typescript
// src/app.ts
import Fastify from 'fastify'
import autoload from '@fastify/autoload'
import { fileURLToPath } from 'node:url'
import { dirname, join } from 'node:path'

const __dirname = dirname(fileURLToPath(import.meta.url))

export function buildApp(opts = {}) {
  const app = Fastify(opts)

  // 1) Load shared plugins FIRST. These are typically fp-wrapped so their
  //    decorators (db, config, auth) are visible to the route plugins below.
  app.register(autoload, { dir: join(__dirname, 'plugins') })

  // 2) Load routes. autoload uses the folder name as the route prefix.
  app.register(autoload, {
    dir: join(__dirname, 'routes'),
    options: { prefix: '/api' }, // everything under /api → routes/users = /api/users
  })

  return app
}
```

### A route module

```typescript
// src/routes/users/index.ts
import type { FastifyInstance } from 'fastify'

export default async function (app: FastifyInstance) {
  // app.prisma / app.authenticate exist here because plugins loaded first.
  app.get('/', { preHandler: [app.authenticate] }, async () => app.prisma.user.findMany())
  app.get('/:id', async (request) =>
    app.prisma.user.findUnique({ where: { id: Number((request.params as any).id) } }),
  )
}
```

**⚡ Why this layout works:** `plugins/` holds infrastructure the whole app needs — so it is `fp`-wrapped, load order matters (use `dependencies`), and its decorators are shared. `routes/` holds feature plugins that are **not** `fp`-wrapped, so each feature's hooks and schemas stay encapsulated and cannot leak into other features. Autoload removes manual wiring and keeps the structure discoverable.

---

## 17. Security Checklist

A consolidated, ship-blocking checklist. Each item links back to the section that explains it.

**Input & data**
- [ ] Every body schema has `additionalProperties: false` (blocks mass-assignment) — §5.
- [ ] Every string/array/number has length/size/range limits (DoS defense) — §5.
- [ ] Every data route has a `response` schema (prevents secret-field leaks) — §5.
- [ ] All DB access is parameterized / via ORM (SQL/NoSQL injection) — §12.
- [ ] File uploads have `limits` and never trust client `filename` (traversal) — §8.
- [ ] Wildcard/param-built file paths are validated against a root (traversal) — §3.

**Auth**
- [ ] Passwords hashed with argon2/bcrypt; constant-time compare — §9.
- [ ] Access tokens short-lived; refresh tokens rotating + revocable — §9.
- [ ] `authorize` checks role **and ownership** (no IDOR) — §9.
- [ ] `/login`, `/refresh`, `/register` are rate-limited — §8, §9.
- [ ] `JWT_SECRET` ≥32 bytes, from env, never committed — §9, §15.

**Transport & headers**
- [ ] `@fastify/helmet` registered; CSP for HTML apps; HSTS on HTTPS — §8.
- [ ] CORS uses an explicit origin allowlist (no `true` + `credentials`) — §8.
- [ ] Identity cookies are `httpOnly` + `secure` + `sameSite` — §8.

**Operational**
- [ ] Logger `redact` covers `authorization`, `cookie`, `password`, `token` — §11.
- [ ] 5xx responses hide stack traces / internals — §10.
- [ ] `requestTimeout` set; `bodyLimit` tight — §2, §14.
- [ ] `@fastify/rate-limit` (Redis-backed in multi-instance) — §8.
- [ ] Runs as non-root; `.env` git/docker-ignored; env validated at boot — §15.
- [ ] `trustProxy` set correctly behind a load balancer — §2, §15.
- [ ] Swagger UI not publicly exposed in production — §8.
- [ ] `npm audit` / dependency scanning in CI — §15.

---

## 18. Gotchas & Best Practices

### Lifecycle & async
- **Don't mix `async` and `done`** in hooks/handlers. Pick one style per function.
- **Don't `return` AND `reply.send()`** the same response. In async handlers, prefer `return`.
- **`await app.ready()`** before inspecting routes/decorators or injecting in tests.
- **Returning `reply`** from a hook short-circuits the pipeline — use it to stop early after `reply.send()`.

### Schemas
- **Always add a `response` schema** on data routes (speed + prevents leaks).
- **Set `additionalProperties: false`** on body schemas to reject unexpected fields.
- **Shared schemas (`addSchema`) are encapsulated** — register them at the root (or via `fp`) so children can `$ref` them.
- Querystring/params are **strings** unless a schema with a numeric type coerces them.
- Prefer `additionalProperties: false` over Ajv `removeAdditional` (reject, don't silently strip).

### Plugins & encapsulation
- **`fp()` for infrastructure** (db, auth, config); **plain plugins for routes**.
- **Don't decorate request/reply with object values** in v5 — use `null`/getter, then set in a hook.
- **Use `dependencies`/`name`** on `fp` plugins to enforce load order; the app errors clearly if a dependency is missing.

### Production
- **`listen({ host: '0.0.0.0' })`** in containers.
- **No `pino-pretty` in production** — emit raw JSON logs.
- **Redact secrets** in the logger config.
- **Run as non-root** in Docker; use multi-stage builds.
- **Handle SIGTERM** with `app.close()` for graceful shutdown.
- **Share state (rate limit, sessions) via Redis** across instances.

### Quick "do / don't" table

| Don't | Do |
|---|---|
| `console.log` in handlers | `request.log.info(...)` (correlated by reqId) |
| Concatenate SQL strings | Parameterized queries (`$1`, prepared statements) |
| Validate manually in handlers | Attach a JSON Schema to the route |
| Return secrets accidentally | Define a `response` schema to strip extra fields |
| `fp`-wrap route plugins | Keep route plugins encapsulated |
| Block the event loop | Stream / use worker threads for heavy CPU work |
| Trust client `Content-Length` | Set `bodyLimit`; stream uploads with multipart |
| Leak stack traces on 5xx | Generic message to client, full error in logs |
| `origin: true` + `credentials` | Explicit CORS origin allowlist |

---

## 19. Study Path & Build-to-Learn Projects

### Suggested learning order

1. **Hello world + routing (§2–3).** Build a few routes by hand; print the route tree with `app.printRoutes()`. **[B]**
2. **Schemas (§5).** Add body/query/response schemas. Intentionally leak a field with no response schema, then add one and watch it get stripped. **[I]**
3. **Lifecycle & hooks (§4).** Add an `onRequest` logger and a `preHandler` guard; trace the order with log lines. **[I]**
4. **TypeScript + TypeBox (§6).** Convert routes to a type provider so schema = types. **[I]**
5. **Plugins & decorators (§7).** Extract a db plugin with `fp`; observe encapsulation by trying to read a non-`fp` decorator from outside. **[I/A]**
6. **Auth (§9) + error handling (§10).** Add JWT login, an `authenticate` guard, RBAC, ownership checks, and a custom error handler. **[I/A]**
7. **Ecosystem (§8).** Layer in cors, helmet, rate-limit, swagger. **[I]**
8. **Testing (§13).** Cover the whole API with `app.inject()` — assert 200/400/401/403/404. **[I]**
9. **Structure & deploy (§15–16).** Refactor to the autoload layout; containerize; add graceful shutdown. **[A]**
10. **Performance (§14).** Benchmark with autocannon; compare with/without response schemas. **[A]**
11. **Security pass (§17).** Walk the checklist against your app; fix every box. **[A]**

### Project 1 — Mini URL shortener (beginner)

- `POST /links` (validate the URL via schema) → store, return a short code.
- `GET /:code` → 302 redirect.
- Response schemas everywhere; Pino logging; tests via `inject()`.
- **Learn:** routing, schemas, serialization, testing.

### Project 2 — Notes REST API with auth (intermediate)

- JWT auth (`/register`, `/login`), `authenticate` + `authorize` guards, ownership checks.
- CRUD for notes owned by the user; RBAC (`admin` sees all).
- Prisma or `pg` via an `fp` db plugin; `@fastify/env` config; rate-limited auth routes.
- Swagger UI at `/docs` (gated) generated from schemas.
- Full test suite covering 200/400/401/403/404.
- **Learn:** plugins, decorators, auth, error handling, OpenAPI, security.

### Project 3 — Realtime task board (advanced)

- WebSocket channel (`@fastify/websocket`) broadcasting task updates, with authenticated upgrades.
- File attachments via `@fastify/multipart` streamed to disk/S3, with sanitized filenames and limits.
- `@fastify/rate-limit` + `@fastify/under-pressure` for resilience.
- TypeBox type provider end-to-end; `@fastify/autoload` structure.
- Dockerized, graceful shutdown, Redis-backed rate-limit/sessions, autocannon benchmarks.
- **Learn:** websockets, uploads, load shedding, production deployment, scaling.

### Where to go next

- Read the source of a small `@fastify/*` plugin (e.g. `@fastify/cookie`) to internalize the plugin/decorator/hook patterns.
- Try running **NestJS** on the Fastify adapter (see the NestJS guide) to see how a DI framework layers on top.
- Explore `@fastify/sensible` (`httpErrors`, useful defaults) and `@fastify/compress`.
- Benchmark your API against an Express equivalent — the difference is the lesson.
- Revisit the **Node.js guide** for the event-loop, streams, and worker-thread internals that underpin Fastify's performance.

---

*End of reference. Build something, break it, read the logs, then run the security checklist — that is the fastest path to mastering Fastify.*
