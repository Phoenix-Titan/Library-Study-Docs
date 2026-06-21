# Fastify — Complete Offline Reference

> **Who this is for:** Node.js developers (beginner to advanced) who want to build fast, production-grade HTTP APIs with **Fastify** — the schema-first, plugin-based framework that consistently tops Node.js performance benchmarks. Every concept below is explained with runnable, heavily commented code, primarily in **TypeScript with ESM**, with notes for plain JavaScript. You should not need the internet to learn Fastify from this document.
>
> **Version note:** This guide targets **Fastify v5** (released late 2024, current throughout 2026). Fastify 5 requires **Node.js 20 or newer**, ships first-class **ESM** support, uses **Ajv 8** for JSON Schema, **Pino 9** for logging, and changed several defaults (e.g. `useSemicolonDelimiter` for querystrings is now `false`, `exposeHeadRoutes` defaults to `true`, the `request.routeOptions` shape was finalized, and `@fastify/*` plugins are versioned to match core v5). Where a detail moves fast or changed between major versions, look for the **⚡ Version note** flag. Confirm exact APIs against the version printed by `fastify.version`.

---

## Table of Contents

1. [What Fastify Is & Why (vs Express / NestJS)](#1-what-fastify-is--why-vs-express--nestjs)
2. [Routing](#2-routing)
3. [Request & Reply Lifecycle and Hooks](#3-request--reply-lifecycle-and-hooks)
4. [Validation & Serialization with JSON Schema](#4-validation--serialization-with-json-schema)
5. [TypeScript Deeply — Type Providers & Generics](#5-typescript-deeply--type-providers--generics)
6. [The Plugin System — Encapsulation & Decorators](#6-the-plugin-system--encapsulation--decorators)
7. [Core Plugin Ecosystem](#7-core-plugin-ecosystem)
8. [Authentication & Authorization](#8-authentication--authorization)
9. [Error Handling](#9-error-handling)
10. [Logging with Pino](#10-logging-with-pino)
11. [Database Integration Patterns](#11-database-integration-patterns)
12. [Testing with node:test / tap and app.inject()](#12-testing-with-nodetest--tap-and-appinject)
13. [Performance — Why Fastify Is Fast](#13-performance--why-fastify-is-fast)
14. [Configuration, Graceful Shutdown, Deployment & Clustering](#14-configuration-graceful-shutdown-deployment--clustering)
15. [Project Structure & @fastify/autoload](#15-project-structure--fastifyautoload)
16. [Gotchas & Best Practices](#16-gotchas--best-practices)
17. [Study Path & Build-to-Learn Projects](#17-study-path--build-to-learn-projects)

---

## 1. What Fastify Is & Why (vs Express / NestJS)

### What it is

**Fastify** is a low-overhead web framework for Node.js. Its defining characteristics:

- **Speed.** Fastify is one of the fastest Node.js frameworks. It achieves this by compiling your route schemas into hyper-optimized functions ahead of time (validation via Ajv, serialization via `fast-json-stringify`).
- **Schema-first.** You attach a **JSON Schema** to each route describing the body, querystring, params, headers, and — crucially — the **response**. Fastify uses these schemas to validate input *and* to serialize output up to 2x faster than `JSON.stringify`.
- **Plugin architecture with encapsulation.** Everything in Fastify is a plugin. Plugins create encapsulated contexts so decorators, hooks, and routes registered inside a plugin do not leak out unless you explicitly break encapsulation with `fastify-plugin`.
- **Developer experience.** Built-in structured logging (Pino), first-class TypeScript via type providers, lifecycle hooks, and a huge official `@fastify/*` plugin ecosystem.

### Why use it

| You need... | Fastify fit |
|---|---|
| A high-throughput JSON REST API | Ideal — its sweet spot |
| Built-in validation + fast serialization | Ideal — schema-driven by design |
| Structured production logging out of the box | Ideal — Pino is built in |
| A lightweight framework you can compose yourself | Ideal — plugins, not magic |
| Microservices where p99 latency matters | Strong fit |
| Heavy enterprise structure with DI, decorators, modules | Consider NestJS (which can run *on* Fastify) |
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

**Mental model:** Express is a router with middleware. Fastify is a router with middleware **plus** a schema compiler **plus** an encapsulated DI-lite plugin system **plus** a logger. NestJS is an architecture layer that can sit on top of Express *or* Fastify.

### Install & Hello World (TypeScript + ESM)

```bash
# Create a project
mkdir fastify-app && cd fastify-app
npm init -y

# Core + TypeScript tooling
npm install fastify
npm install -D typescript @types/node tsx

# tsx runs TS directly in dev; for prod you compile with tsc.
```

Make it an ESM project by adding `"type": "module"` to `package.json`:

```jsonc
// package.json (relevant parts)
{
  "type": "module",                 // ⚡ enables native ESM (import/export)
  "scripts": {
    "dev": "tsx watch src/server.ts",   // hot reload in dev
    "build": "tsc -p tsconfig.json",    // emit JS to dist/
    "start": "node dist/server.js"      // run compiled output
  }
}
```

A modern `tsconfig.json` for Fastify v5 + ESM:

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",          // ESM resolution that respects package.json "type"
    "moduleResolution": "NodeNext",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,                 // always on — Fastify's types reward strictness
    "esModuleInterop": true,
    "skipLibCheck": true,
    "declaration": false,
    "sourceMap": true
  },
  "include": ["src/**/*.ts"]
}
```

The classic hello world:

```typescript
// src/server.ts
import Fastify from 'fastify' // default import; ESM-friendly

// Create the instance. The options object configures the whole app.
const app = Fastify({
  logger: true, // turn on the built-in Pino logger (pretty in dev, JSON in prod)
})

// Register a route. The handler is async, so you just return the payload —
// Fastify serializes the return value and sends it as the response body.
app.get('/', async (request, reply) => {
  // `request` and `reply` are Fastify's wrappers (NOT raw Node req/res).
  request.log.info('handling root request') // per-request child logger
  return { hello: 'world' } // returned object is JSON-serialized automatically
})

// Start listening. `listen` returns a promise in v5.
const start = async () => {
  try {
    // host '0.0.0.0' is important inside Docker so the port is reachable.
    await app.listen({ port: 3000, host: '0.0.0.0' })
    // No need to log here — logger:true prints "Server listening at ..."
  } catch (err) {
    app.log.error(err)
    process.exit(1)
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

**⚡ Version note:** In Fastify v5 the recommended import is the default export `import Fastify from 'fastify'`. Named/typed helpers like `FastifyInstance`, `FastifyRequest`, `FastifyReply` are imported as types: `import type { FastifyInstance } from 'fastify'`.

#### Plain JavaScript (ESM) version

```javascript
// src/server.js  (with "type": "module" in package.json)
import Fastify from 'fastify'

const app = Fastify({ logger: true })

app.get('/', async () => ({ hello: 'world' }))

await app.listen({ port: 3000 }) // top-level await works in ESM
```

#### CommonJS version (if you cannot use ESM)

```javascript
// src/server.cjs  (no "type": "module")
const Fastify = require('fastify')
const app = Fastify({ logger: true })
app.get('/', async () => ({ hello: 'world' }))
app.listen({ port: 3000 }).catch((e) => { app.log.error(e); process.exit(1) })
```

---

## 2. Routing

### Registering routes — two styles

```typescript
import Fastify from 'fastify'
const app = Fastify({ logger: true })

// 1) Shorthand methods (most common)
app.get('/users', async () => [{ id: 1 }])
app.post('/users', async (req, reply) => { reply.code(201); return { created: true } })
app.put('/users/:id', async () => ({ updated: true }))
app.patch('/users/:id', async () => ({ patched: true }))
app.delete('/users/:id', async () => ({ deleted: true }))
app.head('/users', async () => undefined)
app.options('/users', async () => undefined)

// 2) Full route option object (the most powerful form)
app.route({
  method: 'GET',              // or an array: ['GET', 'HEAD']
  url: '/health',
  // schema, handler, hooks, constraints, etc. all live here
  handler: async () => ({ status: 'ok' }),
})
```

### The route options object — full reference

| Option | Purpose |
|---|---|
| `method` | HTTP method string or array of strings |
| `url` | Route path (supports `:param`, wildcards) |
| `handler` | The route handler `(request, reply) => ...` |
| `schema` | JSON Schema for `body`, `querystring`, `params`, `headers`, `response` |
| `preValidation`, `preHandler`, etc. | Route-level lifecycle hooks (arrays allowed) |
| `onRequest` | Route-level onRequest hook |
| `config` | Arbitrary config object available as `request.routeOptions.config` |
| `constraints` | Match on version / host / custom constraints |
| `bodyLimit` | Override max body size for this route (bytes) |
| `logLevel` | Per-route log level |
| `exposeHeadRoutes` | Auto-create a HEAD route for a GET (default `true` in v5) |
| `errorHandler` | Route-scoped error handler |
| `schemaErrorFormatter` | Customize validation error formatting for this route |

### Path & query parameters

```typescript
// Path params: prefix a segment with ":"
app.get('/users/:id/posts/:postId', async (request, reply) => {
  // By default params are typed as Record<string, string>.
  // Everything from the URL is a string until validated/coerced by a schema.
  const { id, postId } = request.params as { id: string; postId: string }
  return { id, postId }
})

// Querystring: ?page=2&limit=10
app.get('/search', async (request) => {
  const q = request.query as { page?: string; limit?: string }
  return { page: q.page ?? '1', limit: q.limit ?? '20' }
})
```

**⚡ Version note:** Fastify uses the `find-my-way` router. Coercion of `"2"` → `2` (number) only happens when a JSON Schema with `type: 'integer'`/`'number'` is attached. Without a schema, params and query values stay strings. See §4 for schema-based coercion.

### Wildcards and optional segments

```typescript
// Wildcard: matches the rest of the path. Captured as params['*'].
app.get('/files/*', async (request) => {
  const rest = (request.params as Record<string, string>)['*']
  return { path: rest } // GET /files/a/b/c → { path: "a/b/c" }
})

// Optional parameter syntax via two routes (find-my-way style):
app.get('/posts/:id?', async (request) => {
  const { id } = request.params as { id?: string }
  return id ? { one: id } : { list: true }
})
```

### Route prefixes via plugins

The idiomatic way to namespace routes is to register a plugin with a `prefix`:

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
import userRoutes from './routes/users.js' // note .js extension in ESM imports
app.register(userRoutes, { prefix: '/api/users' })
```

Prefixes compose: a plugin registered with `/api` that itself registers `userRoutes` with `/users` yields `/api/users`.

### API versioning

Two common strategies:

```typescript
// 1) URL-prefix versioning (simplest, most explicit)
app.register(v1Routes, { prefix: '/v1' })
app.register(v2Routes, { prefix: '/v2' })

// 2) Header/version constraints (same URL, different versions)
app.route({
  method: 'GET',
  url: '/users',
  constraints: { version: '1.2.0' }, // matches Accept-Version: 1.2.0 (semver)
  handler: async () => ({ v: '1.2.0' }),
})
app.route({
  method: 'GET',
  url: '/users',
  constraints: { version: '2.0.0' },
  handler: async () => ({ v: '2.0.0' }),
})
// Client sends header:  Accept-Version: 2.x  → routed to the 2.0.0 handler
```

### Constraints (version, host, custom)

```typescript
// Host constraint: route only matches a specific Host header
app.route({
  method: 'GET',
  url: '/',
  constraints: { host: 'admin.example.com' },
  handler: async () => ({ area: 'admin' }),
})

// Custom constraint strategy (advanced) — match on any request property.
const app2 = Fastify({
  constraints: {
    // Strategy keyed by name; here we constrain on a "tenant" header.
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
// After all routes are registered (e.g. inside app.ready()):
await app.ready()
console.log(app.printRoutes())       // pretty tree of the radix router
// console.log(app.printPlugins())    // plugin registration tree (debugging)
```

---

## 3. Request & Reply Lifecycle and Hooks

### The big picture

Fastify processes every request through an ordered pipeline of **hooks**. Understanding this order is the single most important thing for using Fastify well.

```
                            Incoming HTTP request
                                     │
                                     ▼
                          ┌────────────────────┐
                          │     onRequest      │  auth checks, very early work
                          └─────────┬──────────┘  (body NOT parsed yet)
                                    │
                                    ▼
                          ┌────────────────────┐
                          │    preParsing      │  transform the raw body stream
                          └─────────┬──────────┘
                                    │
                                    ▼   (body is parsed here)
                          ┌────────────────────┐
                          │   preValidation    │  mutate payload before validation
                          └─────────┬──────────┘
                                    │
                                    ▼   (JSON Schema validation runs here)
                          ┌────────────────────┐
                          │    preHandler      │  guards, load related data
                          └─────────┬──────────┘
                                    │
                                    ▼
                          ┌────────────────────┐
                          │   ROUTE HANDLER    │  your business logic
                          └─────────┬──────────┘
                                    │ (handler returns / reply.send called)
                                    ▼
                          ┌────────────────────┐
                          │  preSerialization  │  mutate payload before it's serialized
                          └─────────┬──────────┘
                                    │
                                    ▼   (fast-json-stringify serializes here)
                          ┌────────────────────┐
                          │      onSend        │  modify the serialized payload/headers
                          └─────────┬──────────┘
                                    │
                                    ▼   (bytes written to socket)
                          ┌────────────────────┐
                          │    onResponse      │  metrics, audit (response is done)
                          └────────────────────┘

   Error at any point ─────────────────────────►  onError  ──►  errorHandler
   Request exceeds connectionTimeout ──────────►  onTimeout
```

### Hook reference table

| Hook | When | Can change | Typical use |
|---|---|---|---|
| `onRequest` | First, before body parsing | request only | Auth, rate-limit, request id |
| `preParsing` | Before body is parsed | raw body stream | Decompress, transform raw stream |
| `preValidation` | After parse, before validation | parsed body/query | Inject defaults, normalize input |
| `preHandler` | After validation, before handler | request/reply | Authorization guards, load data |
| `preSerialization` | After handler, before serialize | the payload | Wrap/envelope responses |
| `onSend` | After serialize, before send | serialized payload + headers | Add headers, compress, mask |
| `onResponse` | After response fully sent | nothing (read-only) | Metrics, logging, cleanup |
| `onError` | When an error is thrown/sent | read-only | Custom error logging |
| `onTimeout` | When `connectionTimeout` fires | read-only | Log slow requests |
| `onRequestAbort` | Client disconnects mid-request | read-only | Cancel work, cleanup |

There are also **application lifecycle hooks** (not per-request): `onReady`, `onListen`, `onClose`, `preClose`, `onRoute`, `onRegister`.

### Application-level vs route-level hooks

```typescript
// Application-level (runs for every route in this encapsulation context):
app.addHook('onRequest', async (request, reply) => {
  request.log.info({ url: request.url }, 'incoming')
})

// Route-level (only this route):
app.route({
  method: 'GET',
  url: '/secure',
  preHandler: async (request, reply) => {
    // throw to abort; Fastify routes the error to the error handler
    if (!request.headers.authorization) {
      reply.code(401)
      throw new Error('missing auth')
    }
  },
  handler: async () => ({ ok: true }),
})
```

### Two ways to signal "done" in a hook

```typescript
// 1) Async style (RECOMMENDED): just await; return to continue, throw to abort.
app.addHook('preHandler', async (request, reply) => {
  await doSomething()
  // return nothing → continue
  // throw err     → jump to error handler
  // return reply  → short-circuit (e.g. after reply.send())
})

// 2) Callback style (legacy): call done(). Do NOT mix with async.
app.addHook('preHandler', (request, reply, done) => {
  doSomething(() => done())     // done() to continue
  // done(new Error('nope'))    // error to abort
})
```

**⚡ Gotcha:** Never use both `async` and the `done` callback in the same hook. If your function is `async`, never call `done` — returning resolves it. Mixing them causes "callback already called" bugs.

### Short-circuiting a request from a hook

```typescript
app.addHook('onRequest', async (request, reply) => {
  if (isBlocked(request.ip)) {
    // Send a response and tell Fastify to stop the pipeline by returning reply.
    reply.code(403).send({ error: 'forbidden' })
    return reply // returning the reply object halts further hooks/handler
  }
})
```

### preParsing — transform the raw stream

```typescript
import { Readable } from 'node:stream'

app.addHook('preParsing', async (request, reply, payload) => {
  // `payload` is a stream of the raw request body.
  // You must return a new stream (or the same one). Example: pass-through.
  return payload // here we don't modify; real use: gunzip, decrypt, etc.
})
```

### preSerialization vs onSend

```typescript
// preSerialization sees the JS object BEFORE it becomes a string.
app.addHook('preSerialization', async (request, reply, payload) => {
  // Wrap every successful payload in an envelope.
  if (reply.statusCode < 400) {
    return { data: payload, meta: { ts: Date.now() } }
  }
  return payload
})

// onSend sees the SERIALIZED payload (string/Buffer) and can tweak headers.
app.addHook('onSend', async (request, reply, payload) => {
  reply.header('x-powered-by', 'fastify')
  return payload // must return the (possibly modified) payload
})
```

### onResponse — for metrics

```typescript
app.addHook('onResponse', async (request, reply) => {
  // reply.elapsedTime gives the ms spent handling the request.
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
| `reply.code(n)` / `reply.status(n)` | Set HTTP status code |
| `reply.header(k, v)` / `reply.headers({})` | Set response header(s) |
| `reply.type('application/json')` | Shortcut for Content-Type |
| `reply.redirect(url, code?)` | Redirect (default 302) |
| `reply.serialize(obj)` | Manually run the route's serializer |
| `reply.raw` | The underlying Node `ServerResponse` (escape hatch) |
| `reply.sent` | Boolean: has the reply been sent? |
| `reply.hijack()` | Tell Fastify you'll manage the raw socket yourself |

**⚡ Best practice:** In `async` handlers, prefer **returning** the payload over calling `reply.send()`. Return the value and let Fastify serialize it. Use `reply.send()` only when you need imperative control (e.g. inside a callback). Never both `return` and `reply.send()` the same response.

```typescript
// GOOD (async): return the payload
app.get('/a', async (req, reply) => {
  reply.code(201)         // setting status is fine
  return { created: true } // returning sends it
})

// AVOID: returning AND sending double-sends
app.get('/b', async (req, reply) => {
  reply.send({ x: 1 })
  return { x: 1 } // ⚡ this return value is ignored once send() was called; confusing
})
```

---

## 4. Validation & Serialization with JSON Schema

This is **the** core Fastify feature. You declare JSON Schemas for input and output; Fastify compiles them into fast validators (Ajv) and serializers (`fast-json-stringify`).

### Request validation: body, querystring, params, headers

```typescript
app.post('/users', {
  schema: {
    // Validate the JSON body
    body: {
      type: 'object',
      required: ['email', 'age'],
      additionalProperties: false, // ⚡ reject unknown fields (strongly recommended)
      properties: {
        email: { type: 'string', format: 'email' },
        age: { type: 'integer', minimum: 0, maximum: 130 },
        name: { type: 'string', minLength: 1, maxLength: 100 },
      },
    },
    // Validate & COERCE the querystring (strings → numbers/booleans)
    querystring: {
      type: 'object',
      properties: {
        verbose: { type: 'boolean', default: false }, // "true" → true
        limit: { type: 'integer', default: 20 },       // "20" → 20
      },
    },
    // Validate path params
    params: {
      type: 'object',
      properties: { tenant: { type: 'string' } },
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

If validation fails, Fastify automatically responds `400` with a descriptive error — you write zero validation code.

### Response serialization (the speed benefit)

Adding a `response` schema does two things: (1) it **strips** any properties not in the schema (preventing accidental data leaks like password hashes), and (2) it serializes with `fast-json-stringify`, which is significantly faster than `JSON.stringify` because the shape is known ahead of time.

```typescript
const userResponseSchema = {
  type: 'object',
  properties: {
    id: { type: 'integer' },
    email: { type: 'string' },
    // NOTE: `passwordHash` is intentionally NOT here, so even if your handler
    // returns it, fast-json-stringify will strip it from the output.
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
  // Even if we return extra fields, only id+email reach the client.
  return { id: 1, email: 'a@b.com', passwordHash: 'SECRET-LEAKED?' } // stripped!
})
```

**⚡ Why faster:** `JSON.stringify` must inspect every value's type at runtime. `fast-json-stringify` generates a bespoke function from your schema — it already knows field order and types, so it emits a string with minimal branching. For high-throughput APIs this is a meaningful win.

### Shared schemas with `$ref` and `addSchema`

Avoid duplicating schemas by registering them once and referencing by `$id`.

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

**⚡ Encapsulation note:** Schemas added with `addSchema` are scoped to the encapsulation context. A schema added at the root is visible to child plugins, but a schema added inside a plugin is **not** visible to siblings/parent. Add shared schemas at the top level (or via an `fp`-wrapped plugin — see §6).

### Customizing validation error output

```typescript
const app = Fastify({
  // Option A: tweak Ajv directly
  ajv: {
    customOptions: {
      allErrors: true,        // collect all errors, not just the first
      removeAdditional: 'all',// strip extra props instead of erroring (use carefully)
      coerceTypes: true,      // default true — string→number coercion
      useDefaults: true,      // apply `default` values from schema
    },
  },
})

// Option B: format the validation error response yourself
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
  schemaErrorFormatter: (errors, dataVar) => {
    const e = new Error(`Bad ${dataVar}: ${errors[0]?.message}`)
    return e
  },
  handler: async () => ({ ok: true }),
})
```

### Adding formats (e.g. `uuid`, custom)

Ajv 8 includes common formats only via `ajv-formats`. Fastify v5 bundles it; enable extra formats like so:

```typescript
const app = Fastify({
  ajv: { customOptions: { formats: { /* register custom formats here */ } } },
})

// Or add a format programmatically after creation:
app.addSchema({
  $id: 'uuidParam',
  type: 'object',
  properties: { id: { type: 'string', format: 'uuid' } }, // uuid format from ajv-formats
  required: ['id'],
})
```

---

## 5. TypeScript Deeply — Type Providers & Generics

Fastify is written in TypeScript and supports two levels of typing:

1. **Manual generics** — you tell Fastify the shape of `Body`, `Querystring`, `Params`, `Headers`, `Reply`.
2. **Type providers** — your **JSON Schema becomes the source of truth** and types are inferred from it (no duplication). Two popular providers: **TypeBox** and **json-schema-to-ts**.

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
  request.body.email       // typed as string
  request.query.dryRun     // typed as boolean | undefined
  request.params.tenant    // typed as string
  request.headers['x-api-key'] // typed as string
  return { id: 1, email: request.body.email } // Reply type enforced
})
```

This works but **duplicates** the shape (once in TS, once in JSON Schema). Type providers fix that.

### Type Provider: TypeBox (recommended)

TypeBox lets you write a schema that is *both* a valid JSON Schema *and* a TypeScript type.

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

// Define schema once — it's runtime JSON Schema AND a static TS type.
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
  // request.body is fully typed as { email: string; age: number } — inferred!
  const { email } = request.body
  // The return is type-checked against UserReply.
  return { id: 1, email }
})
```

Extra TypeBox tricks:

```typescript
import { Static, Type } from '@sinclair/typebox'

const Paging = Type.Object({
  page: Type.Optional(Type.Integer({ minimum: 1, default: 1 })),
  limit: Type.Integer({ minimum: 1, maximum: 100, default: 20 }),
})
type Paging = Static<typeof Paging> // derive a plain TS type if you need it elsewhere

const Role = Type.Union([Type.Literal('admin'), Type.Literal('user')]) // enums
const Nullable = Type.Union([Type.String(), Type.Null()])
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
| **TypeBox** | Concise DSL, single source, great inference, supports JSON Schema features | Adds a runtime dependency (small) |
| **json-schema-to-ts** | Plain JSON Schema, no DSL, dev-only dependency | Needs `as const`, more verbose, slower TS compile on big schemas |

### Typing the FastifyInstance and module augmentation

When you decorate the instance/request (see §6), augment Fastify's types so they're known everywhere:

```typescript
// src/types/fastify.d.ts
import type { FastifyInstance } from 'fastify'

declare module 'fastify' {
  interface FastifyInstance {
    // e.g. a decorated db client and a config object
    db: import('pg').Pool
    config: { JWT_SECRET: string; PORT: number }
    authenticate: (request: FastifyRequest, reply: FastifyReply) => Promise<void>
  }
  interface FastifyRequest {
    // e.g. user attached by an auth hook
    user?: { id: number; role: 'admin' | 'user' }
  }
  interface FastifyReply {
    // custom reply decorator
    sendError: (code: number, message: string) => void
  }
}
```

Now `app.db`, `request.user`, and `reply.sendError(...)` are fully typed across the codebase.

### A typed `FastifyPluginAsync`

```typescript
import type { FastifyPluginAsync, FastifyPluginAsyncTypebox } from 'fastify'

// Generic typed plugin
const plugin: FastifyPluginAsync<{ verbose?: boolean }> = async (app, opts) => {
  app.get('/ping', async () => ({ pong: true, verbose: opts.verbose ?? false }))
}

// TypeBox-aware plugin (inherits the type provider automatically):
const tbPlugin: FastifyPluginAsyncTypebox = async (app) => {
  // app already has the TypeBox provider — schemas infer types here too
}
```

---

## 6. The Plugin System — Encapsulation & Decorators

Everything you build in Fastify is a **plugin**. Plugins give you **encapsulation**: hooks, decorators, and schemas registered inside a plugin are isolated to that plugin and its children.

### Anatomy of a plugin

```typescript
import type { FastifyInstance, FastifyPluginAsync } from 'fastify'

// A plugin is just an (async) function receiving the instance and options.
const myPlugin: FastifyPluginAsync<{ greeting: string }> = async (app, opts) => {
  app.get('/hello', async () => ({ msg: opts.greeting }))
}

// Register it (options are passed as the 2nd arg)
app.register(myPlugin, { greeting: 'hi' })
```

### Encapsulation — the core concept

```typescript
app.register(async (child) => {
  // This decorator + hook only exist INSIDE this child context.
  child.decorate('secret', 42)
  child.addHook('onRequest', async () => { /* only runs for child's routes */ })
  child.get('/inside', async () => ({ secret: (child as any).secret })) // works
})

// Out here, app.secret is UNDEFINED — it did not leak out.
// app.get('/outside', async () => ({ secret: (app as any).secret })) // undefined!
```

```
        app (root context)
          │  decorate('a')      ← visible to everything below
          ├── plugin X (child)
          │      decorate('b')  ← visible only inside X and X's children
          │      └── plugin X1  ← sees 'a' and 'b'
          └── plugin Y (child)
                 decorate('c')  ← visible only inside Y
                                  (Y does NOT see 'b'; X does NOT see 'c')
```

### Breaking encapsulation with `fastify-plugin` (fp)

Sometimes you *want* a decorator/hook to be visible to the parent (e.g. a shared db connection, an auth decorator). Wrap the plugin in `fastify-plugin` so it executes in the *parent's* context instead of creating a new one.

```bash
npm install fastify-plugin
```

```typescript
import fp from 'fastify-plugin'
import type { FastifyInstance } from 'fastify'

async function dbConnector(app: FastifyInstance) {
  const pool = createPool()
  // decorate the instance with `db`
  app.decorate('db', pool)
  // close it when the app shuts down
  app.addHook('onClose', async () => { await pool.end() })
}

// Wrapping in fp() makes `app.db` available to the PARENT and all siblings.
export default fp(dbConnector, {
  name: 'db',                 // unique name (used by dependency resolution)
  fastify: '5.x',             // declare the supported Fastify version range
  // dependencies: ['config'] // ensure other plugins load first
})
```

**Rule of thumb:** Use `fp()` for **infrastructure** (db, auth, config) that the whole app shares. Don't wrap **route plugins** in `fp()` — you want their hooks/schemas encapsulated.

### Decorators — decorate / decorateRequest / decorateReply

```typescript
// 1) Decorate the instance (shared singletons: config, db, services)
app.decorate('config', { JWT_SECRET: 'x', PORT: 3000 })
app.config.JWT_SECRET // accessible anywhere in this context

// 2) Decorate the request (per-request data, e.g. authenticated user)
// ⚡ Initialize with null/getter — Fastify v5 forbids decorating with a
//    reference type value directly (would be shared across requests).
app.decorateRequest('user', null)
app.addHook('preHandler', async (request) => {
  ;(request as any).user = { id: 1, role: 'admin' } // set per request
})

// 3) Decorate the reply (custom response helpers)
app.decorateReply('sendError', function (this: any, code: number, message: string) {
  this.code(code).send({ error: message })
})
app.get('/x', async (req, reply) => {
  if (!req.headers.authorization) return (reply as any).sendError(401, 'no auth')
  return { ok: true }
})
```

**⚡ Version note (decorateRequest):** In Fastify v5, decorating a request/reply with an **object/array value** is discouraged because the same reference would be shared across all requests. Decorate with a **primitive, `null`, or a getter function**, then assign the real value in a hook. For objects, prefer `app.decorateRequest('user', null)` then set `request.user = ...` in `preHandler`.

### Checking for and depending on decorators

```typescript
// Guard against double-registration / missing dependencies:
if (!app.hasDecorator('db')) {
  throw new Error('db plugin must be registered before this one')
}

// fastify-plugin `dependencies` enforces load order automatically.
export default fp(authPlugin, { name: 'auth', dependencies: ['db', 'config'] })
```

### Register options & overrides

```typescript
app.register(somePlugin, {
  prefix: '/v1',        // route prefix for routes inside the plugin
  logLevel: 'warn',     // override log level for this context
  // any custom options your plugin reads from `opts`
  featureFlag: true,
})
```

---

## 7. Core Plugin Ecosystem

Fastify's official plugins live under the `@fastify/*` scope and are version-matched to core v5. Install only what you need.

```bash
npm install @fastify/cors @fastify/helmet @fastify/jwt @fastify/cookie \
  @fastify/session @fastify/rate-limit @fastify/static @fastify/multipart \
  @fastify/swagger @fastify/swagger-ui @fastify/websocket @fastify/under-pressure
```

### @fastify/cors

```typescript
import cors from '@fastify/cors'

await app.register(cors, {
  origin: ['https://app.example.com'], // string | string[] | RegExp | function | true(all)
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  credentials: true,                   // allow cookies/Authorization across origins
  maxAge: 86400,                       // cache preflight for 24h
})
```

### @fastify/helmet (security headers)

```typescript
import helmet from '@fastify/helmet'

await app.register(helmet, {
  // Sets sensible security headers (CSP, HSTS, X-Frame-Options, etc.)
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      imgSrc: ["'self'", 'data:'],
    },
  },
  // For APIs you often disable CSP and keep the rest:
  // contentSecurityPolicy: false,
})
```

### @fastify/jwt

```typescript
import jwt from '@fastify/jwt'

await app.register(jwt, {
  secret: process.env.JWT_SECRET!, // or { private, public } for RS256
  sign: { expiresIn: '15m' },      // default sign options
})

// Sign a token (e.g. on login):
app.post('/login', async (request, reply) => {
  // ...verify credentials...
  const token = app.jwt.sign({ id: 1, role: 'admin' })
  return { token }
})

// Decorate a reusable auth guard:
app.decorate('authenticate', async (request: any, reply: any) => {
  try {
    await request.jwtVerify() // reads Authorization: Bearer <token>, sets request.user
  } catch (err) {
    reply.code(401).send({ error: 'unauthorized' })
  }
})

app.get('/me', { preHandler: [app.authenticate] }, async (request: any) => {
  return request.user // the decoded JWT payload
})
```

### @fastify/cookie

```typescript
import cookie from '@fastify/cookie'

await app.register(cookie, {
  secret: process.env.COOKIE_SECRET, // enables signed cookies
})

app.get('/set', async (request, reply) => {
  reply.setCookie('sid', 'abc123', {
    path: '/',
    httpOnly: true,    // not readable by JS (XSS protection)
    secure: true,      // HTTPS only
    sameSite: 'lax',
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

### @fastify/session

```typescript
import cookie from '@fastify/cookie'
import session from '@fastify/session'

await app.register(cookie)
await app.register(session, {
  secret: process.env.SESSION_SECRET!, // must be at least 32 chars
  cookie: { secure: true, httpOnly: true, maxAge: 86_400_000 },
  // store: new RedisStore(...) // use a real store in production (default is in-memory!)
})

app.post('/login', async (request: any, reply) => {
  request.session.user = { id: 1 } // persisted server-side, id in the cookie
  return { ok: true }
})

app.get('/profile', async (request: any, reply) => {
  if (!request.session.user) return reply.code(401).send({ error: 'no session' })
  return request.session.user
})
```

### @fastify/rate-limit

```typescript
import rateLimit from '@fastify/rate-limit'

await app.register(rateLimit, {
  max: 100,            // max requests
  timeWindow: '1 minute',
  // redis: new Redis(...)  // share limits across instances in production
})

// Per-route override:
app.get('/expensive', {
  config: { rateLimit: { max: 5, timeWindow: '1 minute' } },
}, async () => ({ ok: true }))
```

### @fastify/static (serve files)

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

// reply.sendFile is added by the plugin:
app.get('/home', async (request, reply) => {
  return reply.sendFile('index.html') // relative to `root`
})
```

### @fastify/multipart (file uploads)

```typescript
import multipart from '@fastify/multipart'
import { pipeline } from 'node:stream/promises'
import { createWriteStream } from 'node:fs'

await app.register(multipart, {
  limits: { fileSize: 10 * 1024 * 1024 }, // 10 MB max
})

// Streaming a single file to disk (memory-efficient):
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

**⚡ Gotcha:** With `@fastify/multipart`, if you don't consume the file stream, the request hangs/leaks. Always pipe or call `part.toBuffer()`. Setting `attachFieldsToBody: true` lets you access files via `request.body` instead.

### @fastify/swagger + @fastify/swagger-ui

Because routes already carry JSON Schema, OpenAPI generation is nearly free.

```typescript
import swagger from '@fastify/swagger'
import swaggerUi from '@fastify/swagger-ui'

await app.register(swagger, {
  openapi: {
    info: { title: 'My API', version: '1.0.0' },
    components: {
      securitySchemes: {
        bearerAuth: { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' },
      },
    },
  },
})
await app.register(swaggerUi, {
  routePrefix: '/docs', // interactive docs at /docs
})

// Routes auto-document from their schema. Add tags/summary for nicer docs:
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

### @fastify/websocket

```typescript
import websocket from '@fastify/websocket'

await app.register(websocket)

app.get('/ws', { websocket: true }, (socket /* WebSocket */, req) => {
  socket.on('message', (message) => {
    socket.send(`echo: ${message.toString()}`)
  })
  socket.on('close', () => req.log.info('ws closed'))
})
```

**⚡ Version note:** In Fastify v5 `@fastify/websocket`'s handler signature is `(socket, request)` — the first arg is the raw `ws` WebSocket (earlier versions passed a `{ socket }` connection wrapper). Update old handlers accordingly.

### @fastify/under-pressure (overload protection)

```typescript
import underPressure from '@fastify/under-pressure'

await app.register(underPressure, {
  maxEventLoopDelay: 1000,        // ms; reject if event loop is too lagged
  maxHeapUsedBytes: 1_000_000_000,
  maxRssBytes: 1_500_000_000,
  maxEventLoopUtilization: 0.98,
  message: 'Server under pressure, try again later',
  retryAfter: 50,
  // Exposes app.isUnderPressure() and an optional health route:
  exposeStatusRoute: '/status',   // GET /status → { status: 'ok' }
})
```

### Plugin quick reference

| Plugin | Purpose |
|---|---|
| `@fastify/cors` | CORS headers / preflight |
| `@fastify/helmet` | Security headers |
| `@fastify/jwt` | JWT sign/verify + `request.jwtVerify()` |
| `@fastify/cookie` | Read/write (signed) cookies |
| `@fastify/session` | Server-side sessions (needs a store in prod) |
| `@fastify/rate-limit` | Request throttling |
| `@fastify/static` | Serve static assets, `reply.sendFile` |
| `@fastify/multipart` | `multipart/form-data` file uploads |
| `@fastify/swagger` (+ ui) | OpenAPI spec + Swagger UI from schemas |
| `@fastify/websocket` | WebSocket routes |
| `@fastify/under-pressure` | Load shedding / health |
| `@fastify/autoload` | Auto-register plugins/routes from folders (§15) |
| `@fastify/compress` | gzip/brotli responses |
| `@fastify/formbody` | Parse `application/x-www-form-urlencoded` |
| `@fastify/middie` | Run Express-style middleware |

---

## 8. Authentication & Authorization

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
    sign: { expiresIn: '15m' },
  })

  // Guard: verifies the bearer token, populates request.user, else 401.
  app.decorate('authenticate', async (request, reply) => {
    try {
      await request.jwtVerify()
    } catch {
      reply.code(401).send({ error: 'Unauthorized' })
    }
  })

  // RBAC guard factory: returns a preHandler that checks role membership.
  app.decorate('authorize', (roles: string[]) => {
    return async (request, reply) => {
      // assumes authenticate ran first (request.user is set)
      if (!request.user || !roles.includes(request.user.role)) {
        reply.code(403).send({ error: 'Forbidden' })
      }
    }
  })
}, { name: 'auth' })
```

Use it in routes:

```typescript
// Login issues a token:
app.post('/login', {
  schema: {
    body: { type: 'object', required: ['email', 'password'],
      properties: { email: { type: 'string' }, password: { type: 'string' } } },
  },
}, async (request, reply) => {
  const { email, password } = request.body as { email: string; password: string }
  const user = await verifyCredentials(email, password) // your logic
  if (!user) return reply.code(401).send({ error: 'bad credentials' })
  const token = app.jwt.sign({ id: user.id, role: user.role })
  return { token }
})

// Any authenticated user:
app.get('/me', { preHandler: [app.authenticate] }, async (request) => request.user)

// Admin only (run authenticate first, then authorize):
app.delete('/users/:id', {
  preHandler: [app.authenticate, app.authorize(['admin'])],
}, async () => ({ deleted: true }))
```

### Where to put auth in the lifecycle

- Cheap, request-wide checks (IP allowlists, API key presence): `onRequest`.
- Token verification + role checks (RBAC): `preHandler` (after validation so you don't waste work on malformed input — though for pure auth either is fine).

### Refresh-token pattern (sketch)

```typescript
// Issue a short-lived access token + a long-lived refresh token (stored httpOnly cookie).
app.post('/refresh', async (request: any, reply) => {
  const refresh = request.cookies.refresh
  const { valid } = request.unsignCookie(refresh ?? '')
  if (!valid) return reply.code(401).send({ error: 'invalid refresh' })
  // ...look up the refresh token in your store, rotate it...
  const access = app.jwt.sign({ id: 1, role: 'user' })
  return { token: access }
})
```

---

## 9. Error Handling

### Default behavior

If a handler/hook throws (or rejects), Fastify catches it, logs it, and sends an error response. Validation errors → `400`. Unhandled errors → `500`. You rarely need try/catch in handlers — just throw.

### Custom global error handler

```typescript
app.setErrorHandler((error, request, reply) => {
  // Validation errors carry `.validation`
  if (error.validation) {
    return reply.code(400).send({
      error: 'ValidationError',
      message: error.message,
      details: error.validation,
    })
  }

  // Errors with an explicit statusCode (or `code`) are passed through.
  const status = error.statusCode ?? 500
  if (status >= 500) {
    request.log.error({ err: error }, 'unhandled error') // log server errors fully
  }

  reply.code(status).send({
    error: error.name,
    message: status >= 500 ? 'Internal Server Error' : error.message, // hide 500 internals
  })
})
```

### Custom error classes with `@fastify/error` or `http-errors`

```typescript
import createError from '@fastify/error'

// Define a reusable typed error: name, message template, default status.
const NotFoundError = createError('ERR_NOT_FOUND', 'Resource %s not found', 404)

app.get('/widgets/:id', async (request) => {
  const widget = await findWidget((request.params as any).id)
  if (!widget) throw new NotFoundError((request.params as any).id) // → 404 with code
  return widget
})
```

You can also just throw an `Error` with a `statusCode` property:

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

// Scoped (per-plugin) not-found handlers are also supported — register inside the plugin.
```

### Error handling scoping

`setErrorHandler` / `setNotFoundHandler` respect encapsulation: set them inside a plugin to handle errors only for that plugin's routes. The most specific (innermost) handler wins.

```typescript
app.register(async (api) => {
  api.setErrorHandler((err, req, reply) => {
    reply.code(err.statusCode ?? 500).send({ scope: 'api', message: err.message })
  })
  api.get('/boom', async () => { throw new Error('kaboom') })
}, { prefix: '/api' })
```

---

## 10. Logging with Pino

Fastify ships **Pino** as its logger — fast, structured, JSON-by-default.

### Enabling and configuring

```typescript
const app = Fastify({
  logger: {
    level: 'info', // trace | debug | info | warn | error | fatal | silent
    // Pretty-print only in development (requires `pino-pretty` installed):
    transport: process.env.NODE_ENV === 'development'
      ? { target: 'pino-pretty', options: { colorize: true, translateTime: 'HH:MM:ss' } }
      : undefined,
  },
})
```

In production, leave `transport` off so logs are raw JSON (machine-parseable for log aggregators). `npm install -D pino-pretty` for dev.

### The per-request child logger

Every request gets `request.log`, a Pino **child logger** automatically tagged with a `reqId`. Use it instead of `console.log`.

```typescript
app.get('/order/:id', async (request) => {
  request.log.info({ orderId: (request.params as any).id }, 'fetching order')
  // Logs include reqId so you can correlate all logs for one request.
  return { ok: true }
})
```

### Customizing request/response log fields (serializers)

```typescript
const app = Fastify({
  logger: {
    serializers: {
      req(request) {
        return { method: request.method, url: request.url, id: request.id }
      },
      res(reply) {
        return { statusCode: reply.statusCode }
      },
    },
  },
})
```

**⚡ Version note (v5):** Pino 9 changed some default serialization, and Fastify v5 refined the default request log to be leaner. If you relied on the old verbose `req`/`res` shape, define explicit serializers as above.

### Redaction (hide secrets in logs)

```typescript
const app = Fastify({
  logger: {
    redact: {
      paths: ['req.headers.authorization', 'req.headers.cookie', '*.password'],
      censor: '[REDACTED]',
    },
  },
})
// Now Authorization headers and any `password` field are masked in every log line.
```

### Disabling auto request logging or tuning it

```typescript
const app = Fastify({
  logger: true,
  disableRequestLogging: true, // turn off automatic incoming/completed logs
})

// ...and log manually where you want:
app.addHook('onResponse', async (request, reply) => {
  request.log.info({ status: reply.statusCode, ms: reply.elapsedTime }, 'done')
})
```

### Custom request IDs

```typescript
import { randomUUID } from 'node:crypto'
const app = Fastify({
  logger: true,
  genReqId: (req) => (req.headers['x-request-id'] as string) ?? randomUUID(),
})
```

---

## 11. Database Integration Patterns

The idiomatic pattern is: connect once, **decorate the instance** with the client via an `fp` plugin, and close it on shutdown.

### Postgres with `pg` (raw)

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
  // Verify connectivity at boot so the app fails fast on misconfig.
  await pool.query('SELECT 1')
  app.decorate('pg', pool)
  app.addHook('onClose', async () => { await pool.end() }) // graceful cleanup
}, { name: 'db' })
```

```typescript
// usage in a route
app.get('/users/:id', async (request) => {
  const { rows } = await app.pg.query(
    'SELECT id, email FROM users WHERE id = $1',
    [(request.params as any).id], // parameterized — never string-concat SQL
  )
  return rows[0] ?? null
})
```

**⚡ Tip:** There is also an official `@fastify/postgres` plugin that does the decorate + pool management for you, exposing `app.pg` and `app.pg.transact(...)`.

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
app.get('/users', async () => {
  return app.prisma.user.findMany({ select: { id: true, email: true } })
})

app.post('/users', {
  schema: { body: { type: 'object', required: ['email'],
    properties: { email: { type: 'string', format: 'email' } } } },
}, async (request, reply) => {
  const user = await app.prisma.user.create({ data: request.body as any })
  reply.code(201)
  return user
})
```

### Drizzle / Mongo / Redis

The same `fp` + `decorate` + `onClose` pattern applies to any client (Drizzle, `mongodb`, `ioredis`). There are also official plugins: `@fastify/mongodb`, `@fastify/redis`. Always:

1. Create the client in an `fp` plugin.
2. `app.decorate('<name>', client)`.
3. Close it in an `onClose` hook.
4. Declare its name in `dependencies` for plugins that need it.

---

## 12. Testing with node:test / tap and app.inject()

Fastify's killer testing feature is **`app.inject()`**: it dispatches a simulated HTTP request through the full lifecycle **without opening a socket**. Tests are fast and need no port.

### Make the app factory testable

Export a builder that returns a configured (but not-yet-listening) instance.

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

const app = buildApp() // logger off for quiet tests

before(async () => { await app.ready() }) // wait for all plugins to load
after(async () => { await app.close() })  // clean shutdown after the suite

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
node --test               # discovers test/**/*.test.js (compile first) 
# Or run TS directly:
node --import tsx --test test/**/*.test.ts
```

### With `tap` (Fastify's traditional choice)

```typescript
import { test } from 'tap'
import { buildApp } from '../src/app.js'

test('health', async (t) => {
  const app = buildApp()
  t.teardown(() => app.close()) // tap auto-awaits teardown
  const res = await app.inject({ method: 'GET', url: '/health' })
  t.equal(res.statusCode, 200)
})
```

### The inject response object

| Property | What |
|---|---|
| `res.statusCode` | Numeric status |
| `res.headers` | Response headers object |
| `res.body` | Raw string body |
| `res.json()` | Parsed JSON body |
| `res.payload` | Alias for `res.body` |
| `res.cookies` | Parsed Set-Cookie array |

**⚡ Best practice:** Always `await app.ready()` before injecting if you're not using the `before` hook, and `await app.close()` after, so `onClose` hooks (db disconnect, etc.) run.

---

## 13. Performance — Why Fastify Is Fast

### The sources of speed

1. **Schema-compiled serialization.** `fast-json-stringify` turns a response schema into a purpose-built function. No runtime type inspection per field.
2. **Schema-compiled validation.** Ajv compiles each schema into optimized JS once at boot, not per request.
3. **A fast radix-tree router** (`find-my-way`) with O(k) lookups by path length.
4. **Minimal abstraction over Node's http.** The request/reply wrappers are thin.
5. **Avoiding closures and reusing objects** in hot paths internally.

### Practical tips to stay fast

```typescript
// 1) ALWAYS add a response schema to hot endpoints — biggest single win.
app.get('/fast', {
  schema: { response: { 200: { type: 'object', properties: { ok: { type: 'boolean' } } } } },
}, async () => ({ ok: true }))

// 2) Prefer async/await over callback hooks; avoid unnecessary preHandlers.

// 3) Reuse shared schemas via $ref instead of inlining big duplicate schemas.

// 4) Tune keep-alive for high-throughput services:
const app2 = Fastify({
  keepAliveTimeout: 72_000,  // ms; should exceed your LB's idle timeout
  connectionTimeout: 0,      // 0 = no socket timeout (let LB manage)
  bodyLimit: 1_048_576,      // 1 MB default body cap; raise/lower per need
})
```

### Async vs callback hooks

Async hooks are idiomatic in v5 and just as fast; callbacks are legacy. The real cost is **doing unnecessary work** in `preHandler` for every request. Keep hot paths lean.

### Benchmarking

```bash
# autocannon: a Node load-testing tool from the Fastify authors
npx autocannon -c 100 -d 10 http://localhost:3000/fast
# -c 100 connections, -d 10 seconds. Reports req/s, latency percentiles.
```

Compare a route **with** vs **without** a response schema to see the serialization win firsthand. Profile with `node --prof` or `clinic.js` (`npx clinic doctor -- node dist/server.js`) when latency spikes.

### Avoid these performance traps

| Trap | Fix |
|---|---|
| No response schema on hot routes | Add one — uses fast-json-stringify |
| Logging at `debug`/`trace` in prod | Use `info`+ in prod |
| `pino-pretty` transport in prod | Only use it in dev |
| Huge bodies parsed needlessly | Set `bodyLimit`, stream large uploads |
| Blocking the event loop (sync crypto, big JSON.parse) | Offload to worker threads / streams |
| Per-request object allocation in hooks | Reuse, minimize work |

---

## 14. Configuration, Graceful Shutdown, Deployment & Clustering

### Typed environment config with `@fastify/env`

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
    JWT_SECRET: { type: 'string' },
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
  await app.register(fastifyEnv, {
    schema,
    dotenv: true, // load a .env file automatically
  })
  // Now app.config is validated & typed; the app refuses to boot if env is invalid.
}, { name: 'config' })
```

### Graceful shutdown

```typescript
import { buildApp } from './app.js'

const app = buildApp({ logger: true })

await app.listen({ port: app.config.PORT, host: '0.0.0.0' })

// On SIGTERM/SIGINT, stop accepting new connections and run onClose hooks
// (db disconnect, flush, etc.), then exit.
const shutdown = async (signal: string) => {
  app.log.info({ signal }, 'shutting down')
  await app.close() // waits for in-flight requests + runs onClose hooks
  process.exit(0)
}
process.on('SIGTERM', () => shutdown('SIGTERM'))
process.on('SIGINT', () => shutdown('SIGINT'))
```

For automatic handling there's also `@fastify/graceful-shutdown` and the `closeWithGrace` package:

```typescript
import closeWithGrace from 'close-with-grace'
closeWithGrace({ delay: 10_000 }, async ({ err }) => {
  if (err) app.log.error(err)
  await app.close()
})
```

### Deployment with Docker

```dockerfile
# Dockerfile — multi-stage for a small production image
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci                 # reproducible install
COPY . .
RUN npm run build          # tsc → dist/

FROM node:20-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev      # only production deps
COPY --from=build /app/dist ./dist
EXPOSE 3000
# Run as non-root for security:
USER node
CMD ["node", "dist/server.js"]
```

```dockerignore
# .dockerignore
node_modules
dist
.git
*.log
```

**⚡ Remember:** Inside containers, always `listen({ host: '0.0.0.0' })`. The default `localhost` is not reachable from outside the container.

### Clustering / scaling

Node is single-threaded per process. To use all CPU cores, run multiple processes:

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

In practice, prefer an external process manager (PM2) or a container orchestrator (Kubernetes) that runs N replicas — it's simpler and handles restarts/rollouts. Use a shared store (Redis) for rate-limit/session state across instances.

---

## 15. Project Structure & @fastify/autoload

`@fastify/autoload` registers every plugin/route file in a directory automatically, enforcing a clean, convention-based layout.

```bash
npm install @fastify/autoload
```

### Recommended layout

```
src/
├── app.ts               # builds the Fastify instance, wires autoload
├── server.ts            # entrypoint: builds app + listen + shutdown
├── plugins/             # CROSS-CUTTING plugins (loaded first, usually fp-wrapped)
│   ├── env.ts           # @fastify/env config
│   ├── db.ts            # database decorator
│   ├── auth.ts          # jwt + authenticate/authorize decorators
│   └── sensible.ts      # @fastify/sensible helpers
├── routes/              # FEATURE routes (auto-prefixed by folder name)
│   ├── users/
│   │   ├── index.ts     # → /users
│   │   └── schemas.ts
│   └── health/
│       └── index.ts     # → /health
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

  // 1) Load shared plugins first. These are typically fp-wrapped so their
  //    decorators (db, config, auth) are visible to the route plugins below.
  app.register(autoload, {
    dir: join(__dirname, 'plugins'),
  })

  // 2) Load routes. autoload uses the folder name as the route prefix.
  app.register(autoload, {
    dir: join(__dirname, 'routes'),
    options: { prefix: '/api' }, // everything under /api
    // dirNameRoutePrefix: true (default) → routes/users → /api/users
  })

  return app
}
```

### A route module

```typescript
// src/routes/users/index.ts
import type { FastifyInstance } from 'fastify'

export default async function (app: FastifyInstance) {
  // app.db / app.authenticate exist here because plugins loaded first.
  app.get('/', { preHandler: [app.authenticate] }, async () => {
    return app.prisma.user.findMany()
  })
  app.get('/:id', async (request) => {
    return app.prisma.user.findUnique({ where: { id: Number((request.params as any).id) } })
  })
}
```

**⚡ Why this layout works:** `plugins/` holds infrastructure that the whole app needs (so it's `fp`-wrapped and load order matters — use `dependencies`). `routes/` holds encapsulated feature plugins (NOT `fp`-wrapped) so each feature's hooks/schemas stay isolated. Autoload removes manual `register` boilerplate and keeps the wiring discoverable.

---

## 16. Gotchas & Best Practices

### Lifecycle & async

- **Don't mix `async` and `done`** in hooks/handlers. Pick one style.
- **Don't `return` AND `reply.send()`** the same response. In async handlers, prefer `return`.
- **`await app.ready()`** before inspecting routes/decorators or injecting in tests.
- **Returning `reply`** from a hook short-circuits the pipeline — use it to stop early after `reply.send()`.

### Schemas

- **Always add a `response` schema** on hot routes (speed + prevents data leaks).
- **Set `additionalProperties: false`** on body schemas to reject unexpected fields.
- **Shared schemas (`addSchema`) are encapsulated** — register them at the root (or via `fp`) so children can `$ref` them.
- Querystring/params are **strings** unless a schema with a numeric type coerces them.

### Plugins & encapsulation

- **`fp()` for infrastructure** (db, auth, config) that must be shared; **plain plugins for routes**.
- **Don't decorate request/reply with object values** in v5 — use `null`/getter then set in a hook.
- **Use `dependencies`/`name`** on `fp` plugins to enforce load order; the app errors clearly if a dependency is missing.

### Production

- **`listen({ host: '0.0.0.0' })`** in containers.
- **No `pino-pretty` in production** — emit raw JSON logs.
- **Redact secrets** (`authorization`, `cookie`, `password`) in the logger config.
- **Run as non-root** in Docker and use multi-stage builds.
- **Handle SIGTERM** with `app.close()` for graceful shutdown.
- **Share state (rate limit, sessions) via Redis** when running multiple instances.

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

---

## 17. Study Path & Build-to-Learn Projects

### Suggested learning order

1. **Hello world + routing (§1–2).** Build a few routes by hand; print the route tree with `app.printRoutes()`.
2. **Schemas (§4).** Add body/query/response schemas. Intentionally leak a field with no response schema, then add one and watch it get stripped.
3. **Lifecycle & hooks (§3).** Add an `onRequest` logger and a `preHandler` guard. Trace the order with log lines.
4. **TypeScript + TypeBox (§5).** Convert your routes to a type provider so schema = types.
5. **Plugins & decorators (§6).** Extract a db plugin with `fp`; observe encapsulation by trying to access a non-`fp` decorator from outside.
6. **Auth (§8) + error handling (§9).** Add JWT login, an `authenticate` guard, RBAC, and a custom error handler.
7. **Ecosystem (§7).** Layer in cors, helmet, rate-limit, swagger.
8. **Testing (§12).** Cover the whole API with `app.inject()` — aim for zero real network in tests.
9. **Structure & deploy (§14–15).** Refactor to the autoload layout; containerize; add graceful shutdown.
10. **Performance (§13).** Benchmark with autocannon; compare with/without response schemas.

### Project 1 — Mini URL shortener (beginner)

- `POST /links` (validate URL via schema) → store, return short code.
- `GET /:code` → 302 redirect.
- Response schemas everywhere; Pino logging; tests via `inject()`.
- **Learn:** routing, schemas, serialization, testing.

### Project 2 — Notes REST API with auth (intermediate)

- JWT auth (`/register`, `/login`), `authenticate` + `authorize` guards.
- CRUD for notes owned by the user; RBAC (`admin` can see all).
- Prisma or `pg` via an `fp` db plugin; `@fastify/env` config.
- Swagger UI at `/docs` generated from schemas.
- Full test suite with `inject()` covering 200/400/401/403/404.
- **Learn:** plugins, decorators, auth, error handling, OpenAPI.

### Project 3 — Realtime task board (advanced)

- WebSocket channel (`@fastify/websocket`) broadcasting task updates.
- File attachments via `@fastify/multipart` streamed to disk/S3.
- `@fastify/rate-limit` + `@fastify/under-pressure` for resilience.
- TypeBox type provider end-to-end; `@fastify/autoload` project structure.
- Dockerized, graceful shutdown, Redis-backed rate-limit/sessions, autocannon benchmarks.
- **Learn:** websockets, uploads, load shedding, production deployment, scaling.

### Where to go next

- Read the source of a small `@fastify/*` plugin (e.g. `@fastify/cookie`) to internalize the plugin/decorator/hook patterns.
- Try running NestJS on the Fastify adapter to see how the two relate.
- Explore `@fastify/sensible` (httpErrors, useful defaults) and `@fastify/compress`.
- Benchmark your own API against an Express equivalent — the difference is the lesson.

---

*End of reference. Build something, break it, read the logs — that is the fastest path to mastering Fastify.*
