# NestJS — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I've never built a backend" to "I architect and ship production-grade, testable, secure NestJS services" — without an internet connection. Every concept is explained in prose first: *what it is*, *the logic / why NestJS works this way*, *what it's for and when you reach for it*, *how to use it*, the *key decorators / options / parameters*, *best practices*, and *security recommendations* — and only then the heavily-commented, runnable TypeScript. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **NestJS 11** (current in 2026), running on **Node.js 22 LTS / 24** and **TypeScript 5.x**. NestJS 11 ships first-class **Fastify v5** support, a faster module resolver, and updated `@nestjs/*` packages. Where an API changed recently it is flagged with **⚡ Version note**. The author is on **Windows 11**, so OS-specific notes (`npm` scripts, paths) are called out where they matter.
>
> **Cross-references:** NestJS sits *on top of* an HTTP adapter — usually Express, optionally **Fastify** (see `FASTIFY_GUIDE.md` for the underlying adapter's internals: hooks, schema validation, plugins). For the database layer this guide leans on **Prisma** (see `PRISMA_ORM_GUIDE.md`) and **PostgreSQL** (see `POSTGRESQL_GUIDE.md`). For an alternative, framework-agnostic auth layer see `BETTERAUTH_GUIDE.md`. Cross-links appear inline throughout.

---

## Table of Contents

1. [What NestJS Is, the Philosophy & When to Use It](#1-what-nestjs-is-the-philosophy--when-to-use-it) **[B]**
2. [Setup & the Nest CLI](#2-setup--the-nest-cli) **[B]**
3. [Project & Folder Structure](#3-project--folder-structure) **[B]**
4. [Modules — the Encapsulation System](#4-modules--the-encapsulation-system) **[B/I]**
5. [Controllers & Routing](#5-controllers--routing) **[B]**
6. [Providers & Dependency Injection (in depth)](#6-providers--dependency-injection-in-depth) **[I/A]**
7. [DTOs & Validation](#7-dtos--validation) **[I]**
8. [The Request Lifecycle: Pipes, Guards, Interceptors, Middleware & Filters](#8-the-request-lifecycle-pipes-guards-interceptors-middleware--filters) **[I/A]**
9. [Exception Handling](#9-exception-handling) **[I]**
10. [Configuration (@nestjs/config)](#10-configuration-nestjsconfig) **[I]**
11. [Database Integration — Prisma, TypeORM & Mongoose](#11-database-integration--prisma-typeorm--mongoose) **[I/A]**
12. [Authentication & Authorization (Passport, JWT, RBAC)](#12-authentication--authorization-passport-jwt-rbac) **[I/A]**
13. [OpenAPI / Swagger Docs](#13-openapi--swagger-docs) **[I]**
14. [WebSockets & Gateways](#14-websockets--gateways) **[I/A]**
15. [File Uploads](#15-file-uploads) **[I]**
16. [Caching, Task Scheduling & Queues](#16-caching-task-scheduling--queues) **[I/A]**
17. [Microservices Overview](#17-microservices-overview) **[A]**
18. [Testing — Unit & E2E](#18-testing--unit--e2e) **[I/A]**
19. [Security, Best Practices & Production Hardening](#19-security-best-practices--production-hardening) **[I/A]**
20. [Tips, Tricks & Gotchas](#20-tips-tricks--gotchas) **[I/A]**
21. [Study Path & Build-to-Learn Projects](#21-study-path--build-to-learn-projects)

---

## 1. What NestJS Is, the Philosophy & When to Use It

### What it is

NestJS is a **progressive Node.js framework** for building server-side applications — REST APIs, GraphQL APIs, WebSocket servers, microservices, and CLI/worker apps. The word "progressive" means it grows with you: a tiny app uses three concepts (modules, controllers, services), and the same architecture scales to a hundred-module enterprise system without rewrites.

The single most important thing to understand is that **NestJS is not an HTTP server.** It does not implement routing, request parsing, or response writing itself. Instead it is an **architectural layer that sits on top of an HTTP "adapter"** — by default **Express**, optionally **Fastify** (covered in `FASTIFY_GUIDE.md`). NestJS owns the *structure and wiring* of your application; the adapter owns the *raw HTTP plumbing*. This separation is why you can swap Express for Fastify by changing one line in `main.ts` and your controllers, services, guards, and pipes keep working unchanged.

What NestJS layers on top of the adapter:

- **TypeScript-first design.** Decorators (`@Controller`, `@Injectable`, `@Get`), interfaces, generics, and types are everywhere. NestJS leans hard into TypeScript's metadata reflection (`reflect-metadata`) to wire your app together at runtime.
- **An opinionated, modular architecture** inspired by **Angular**: the building blocks are **Modules** (organize and encapsulate), **Controllers** (handle requests), and **Providers** (do the work — services, repositories, factories). If you know Angular's `@NgModule`/`@Injectable`/DI model, NestJS will feel like home on the backend.
- **A built-in Inversion-of-Control (IoC) / Dependency-Injection (DI) container.** You declare *what* a class needs in its constructor; NestJS figures out *how* to build and supply it. This is the heart of the framework (covered deeply in §6).
- **Lifecycle hooks** — `OnModuleInit`, `OnApplicationBootstrap`, `OnModuleDestroy`, `OnApplicationShutdown` — so you can open DB connections on startup and close them gracefully on shutdown.
- **Official, batteries-included packages** for almost everything: `@nestjs/config`, `@nestjs/jwt`, `@nestjs/passport`, `@nestjs/swagger`, `@nestjs/cache-manager`, `@nestjs/schedule`, `@nestjs/bullmq`, `@nestjs/websockets`, `@nestjs/microservices`, `@nestjs/typeorm`, `@nestjs/mongoose`. These are maintained alongside the core, so they don't rot.

### The philosophy — why NestJS works the way it does

The logic behind NestJS is a reaction to a real pain: plain Express gives you total freedom and zero structure. On a solo toy project that freedom feels great. On a team of ten, six months in, it becomes chaos — every developer invents their own folder layout, their own way to validate input, their own error format, their own way to share a database connection. NestJS makes a bet: **conventions and structure are worth more than freedom on any project that lives longer than a weekend.** It encodes a set of architectural decisions (separation of concerns, dependency injection, declarative cross-cutting concerns) so that any NestJS developer can drop into any NestJS codebase and immediately know where things live and how they connect.

Three ideas underpin everything:

1. **Separation of concerns.** Controllers *only* deal with HTTP (extract data, return a result). Services *only* contain business logic and know nothing about HTTP. This makes business logic reusable across HTTP, WebSockets, queues, and microservices — and trivially unit-testable.
2. **Dependency Injection.** Classes never construct their own dependencies with `new`. They declare them and receive them. This is what makes testing easy (swap a real DB for a mock with one line), and what lets NestJS manage object lifetimes (singletons, request scope) for you.
3. **Declarative cross-cutting concerns.** Validation, authentication, logging, caching, and error formatting are not scattered through your handlers — they are *layers* (pipes, guards, interceptors, filters) you attach declaratively with decorators. Write the auth check once as a guard; apply it to a thousand routes with one `@UseGuards()`.

### When to choose NestJS — and when not to

| Situation | NestJS? | Why |
|-----------|---------|-----|
| REST or GraphQL API for a SaaS / product | **Yes — ideal** | Structure, DI, validation, Swagger pay off immediately |
| Microservices with brokers (Redis, RabbitMQ, Kafka, NATS, gRPC) | **Yes — first-class** | `@nestjs/microservices` abstracts transports uniformly |
| Real-time (chat, presence, notifications) over WebSockets | **Yes** | Gateways reuse the same DI/guards/pipes as HTTP |
| Large or growing team needing consistent conventions | **Strongly yes** | The whole point: shared mental model |
| Enterprise app needing testability & long-term maintenance | **Strongly yes** | DI + thin controllers = easy mocking and refactors |
| A single-route proxy, glue script, or tiny webhook receiver | **Overkill** | Use plain Express/Fastify; NestJS's structure is wasted |
| Maximum raw throughput, minimal abstraction | **Use Fastify directly** | NestJS adds a thin overhead; see `FASTIFY_GUIDE.md` |
| A CLI tool with no HTTP | **Usually overkill** | Though `nest-commander` exists if you want NestJS DI in a CLI |

**NestJS vs Fastify vs Express, decided simply:** Reach for **plain Express/Fastify** when the app is tiny and you value zero ceremony. Reach for **Fastify directly** when raw performance and a small dependency surface dominate (see `FASTIFY_GUIDE.md` — and remember NestJS can *use* Fastify as its adapter, giving you most of that performance *with* the structure). Reach for **NestJS** the moment the app has more than a handful of routes, more than one developer, or a lifespan measured in months — the structure stops being overhead and starts being a force multiplier.

### Architecture & request-lifecycle overview

Every HTTP request flows through a fixed pipeline of layers. Knowing this order cold is the single most useful piece of NestJS knowledge — it tells you *where* to put each piece of logic (covered fully in §8):

```
                          HTTP Request
                               │
                               ▼
   ┌──────────────────────────────────────────────────────┐
   │  1. MIDDLEWARE   — runs first, raw req/res, like       │
   │     Express MW (logging, body parsing, cors, helmet)   │
   └───────────────────────────┬──────────────────────────┘
                               ▼
   ┌──────────────────────────────────────────────────────┐
   │  2. GUARDS       — "may this request proceed?"         │
   │     authentication & authorization → true / throw 403  │
   └───────────────────────────┬──────────────────────────┘
                               ▼
   ┌──────────────────────────────────────────────────────┐
   │  3. INTERCEPTORS (pre)  — wrap the handler; run code    │
   │     BEFORE it (start timer, set up context)            │
   └───────────────────────────┬──────────────────────────┘
                               ▼
   ┌──────────────────────────────────────────────────────┐
   │  4. PIPES        — validate & transform the inputs     │
   │     bound to handler params (ValidationPipe, ParseInt) │
   └───────────────────────────┬──────────────────────────┘
                               ▼
   ┌──────────────────────────────────────────────────────┐
   │  5. CONTROLLER HANDLER  — YOUR route method            │
   │     (extract data → call a service → return result)    │
   └───────────────────────────┬──────────────────────────┘
                               ▼
   ┌──────────────────────────────────────────────────────┐
   │  6. INTERCEPTORS (post) — run code AFTER the handler   │
   │     (transform response, log duration, cache result)   │
   └───────────────────────────┬──────────────────────────┘
                               ▼
                         HTTP Response
                               │
   ┌──────────────────────────────────────────────────────┐
   │  EXCEPTION FILTERS — catch anything thrown ANYWHERE    │
   │  above and turn it into a formatted error response     │
   └──────────────────────────────────────────────────────┘
```

The **Module** is the unit of organization that ties controllers and providers together and controls what is visible to the rest of the app. Every feature lives in a module; the root `AppModule` imports them all. We start there after setup.

---

## 2. Setup & the Nest CLI

### The logic of the CLI

NestJS ships an official **command-line tool**, `@nestjs/cli`, and you will use it constantly. Its value is two-fold. First, it **scaffolds** projects and files with the correct structure and boilerplate so you never hand-write a module shell. Second — and this is the part beginners miss — its `generate` command **wires new files into their module automatically** (adds a controller to the module's `controllers` array, a service to `providers`). It uses "schematics" (templated generators) under the hood. Learning the CLI early saves enormous time and prevents the classic "I created a service but forgot to register it as a provider" error.

### Prerequisites

```bash
node -v   # must be >= 20; this guide assumes Node 22 LTS or 24
npm -v    # npm 10+ recommended (ships with Node 22+)
```

> **⚡ Version note:** NestJS 11 requires Node 20+. Node 18 reached end-of-life; use 22 LTS for production. On Windows, install Node from nodejs.org or `winget install OpenJS.NodeJS.LTS`.

### Install the Nest CLI globally

```bash
npm install -g @nestjs/cli
nest --version   # should print 11.x.x
```

You can also avoid a global install and invoke it on demand with `npx @nestjs/cli@latest ...`, which guarantees the latest version per run.

### Create a new project

```bash
nest new my-api
# The CLI asks which package manager you want: npm / yarn / pnpm
#   - npm     → simplest, ships with Node
#   - pnpm    → fast, disk-efficient, great for monorepos
cd my-api
npm run start:dev   # starts in watch mode: recompiles & restarts on every save
```

The app boots on `http://localhost:3000`. `start:dev` uses the TypeScript compiler in watch mode (SWC-based in recent CLIs for speed) so saving a file restarts the server in well under a second.

### `nest generate` — the schematics you'll use daily

`nest generate` (alias `nest g`) creates files *and* registers them. Here is the full reference table:

| Command | Creates | Auto-registered? |
|---------|---------|------------------|
| `nest g module users` | `src/users/users.module.ts` | imported into `AppModule` |
| `nest g controller users` | `users.controller.ts` (+ spec) | added to module `controllers` |
| `nest g service users` | `users.service.ts` (+ spec) | added to module `providers` |
| `nest g resource users` | module + controller + service + DTOs + entity + CRUD stub | fully wired — **fastest start** |
| `nest g guard auth` | `auth/auth.guard.ts` | no (apply with `@UseGuards`) |
| `nest g pipe validation` | `validation.pipe.ts` | no |
| `nest g interceptor logging` | `logging.interceptor.ts` | no |
| `nest g middleware logger` | `logger.middleware.ts` | no (wire in module `configure()`) |
| `nest g filter http-exception` | `http-exception.filter.ts` | no |
| `nest g gateway events` | `events.gateway.ts` | added to module `providers` |
| `nest g decorator current-user` | a custom decorator stub | no |

```bash
# The single most useful command when starting a feature — full CRUD scaffold:
nest g resource users
#   It asks: REST API? GraphQL? Microservice? WebSockets?  → pick REST API
#   It asks: generate CRUD entry points? → yes
#   Result: a working controller + service + create/update DTOs + entity, all wired.
```

> **Tip:** Add `--no-spec` to skip generating the `.spec.ts` test file when scaffolding quickly (you can write tests later). Use `--dry-run` to preview what would be created without writing anything — great for confirming paths.

```bash
nest g service users --no-spec
nest g resource products --dry-run   # preview only
```

### `main.ts` — the bootstrap file

`main.ts` is the entry point. It creates the Nest application instance from the root module, applies app-wide configuration (global pipes, CORS, prefixes, Swagger), and starts listening. **This file is where global, DI-free setup happens.** Read it carefully on any NestJS project.

```typescript
// src/main.ts — the canonical bootstrap, annotated
import { NestFactory } from '@nestjs/core';
import { ValidationPipe, Logger } from '@nestjs/common';
import { AppModule } from './app.module';

async function bootstrap() {
  // NestFactory.create builds the whole app from the root module:
  // it resolves the entire dependency graph and instantiates providers.
  const app = await NestFactory.create(AppModule);

  // A global validation pipe — applies DTO validation to EVERY route. (See §7.)
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,            // strip properties not declared on the DTO
      forbidNonWhitelisted: true, // 400 if the client sends unknown properties
      transform: true,            // turn plain payloads into typed DTO instances
    }),
  );

  // A versioned, prefixed API surface: routes become /api/v1/...
  app.setGlobalPrefix('api/v1');

  // Enable graceful shutdown so OnModuleDestroy hooks fire on SIGTERM/SIGINT.
  app.enableShutdownHooks();

  const port = process.env.PORT ?? 3000;
  await app.listen(port);
  Logger.log(`Application running on ${await app.getUrl()}`, 'Bootstrap');
}
bootstrap();
```

### Switching to the Fastify adapter (optional, for performance)

By default NestJS uses Express. **Fastify** is a faster adapter (2–3× more requests/second in benchmarks, with schema-based serialization). Because NestJS abstracts the adapter, switching is a `main.ts` change — your controllers and providers are untouched. For the deep details of what Fastify itself does (lifecycle hooks, JSON-schema validation, plugin encapsulation), read `FASTIFY_GUIDE.md`; here we only flip the adapter.

```bash
npm install @nestjs/platform-fastify fastify
```

```typescript
// main.ts — swap the underlying HTTP adapter to Fastify
import { NestFactory } from '@nestjs/core';
import {
  FastifyAdapter,
  NestFastifyApplication,
} from '@nestjs/platform-fastify';
import { AppModule } from './app.module';

async function bootstrap() {
  // The generic <NestFastifyApplication> gives you Fastify-specific typings.
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter({ logger: true }), // Fastify's own pino logger
  );
  // '0.0.0.0' makes it reachable from outside the container/host loopback.
  await app.listen({ port: 3000, host: '0.0.0.0' });
}
bootstrap();
```

> **⚡ Version note:** NestJS 11 supports **Fastify v5**. Watch out: some community middleware is Express-only (it expects Express's `req`/`res` shape). Check compatibility before switching. Fastify-specific NestJS packages exist where needed (e.g. `@fastify/static`, `@fastify/multipart`). See `FASTIFY_GUIDE.md` for what changes at the adapter level.

---

## 3. Project & Folder Structure

### The default scaffold

`nest new` produces a minimal, conventional layout. Understanding each file removes the "where does anything go?" anxiety:

```
my-api/
├── src/
│   ├── main.ts                  # bootstrap: create the app, configure globals, listen
│   ├── app.module.ts            # ROOT module — imports every feature module
│   ├── app.controller.ts        # demo root controller (delete in real apps)
│   ├── app.controller.spec.ts   # demo unit test
│   └── app.service.ts           # demo root service (delete in real apps)
├── test/
│   ├── app.e2e-spec.ts          # end-to-end test (boots the whole app)
│   └── jest-e2e.json            # Jest config specifically for e2e tests
├── dist/                        # compiled JavaScript output (git-ignored)
├── .env                         # environment variables (NEVER commit — git-ignore it)
├── nest-cli.json                # CLI config: source root, assets, compiler plugins
├── tsconfig.json                # TypeScript compiler options
├── tsconfig.build.json          # build-time TS config (excludes test files)
└── package.json
```

### Recommended structure for real apps — organize by feature, not by layer

The most important structural rule in NestJS: **organize by domain/feature, not by technical layer.** A `UsersModule` folder containing the users controller, service, DTOs, and entity is right. A top-level `services/` folder containing every service in the app is wrong — it scatters one feature across the codebase and couples everything. The logic: features change together, so keeping a feature's files together minimizes the blast radius of any change and makes modules genuinely self-contained and movable.

```
src/
├── main.ts
├── app.module.ts
│
├── config/                      # typed config factory + validation
│   └── configuration.ts
│
├── common/                      # SHARED, cross-cutting building blocks
│   ├── decorators/              # @CurrentUser(), @Roles(), @Public()
│   ├── filters/                 # global exception filters
│   │   └── all-exceptions.filter.ts
│   ├── guards/                  # reusable guards
│   ├── interceptors/            # transform / logging / timeout interceptors
│   │   └── transform.interceptor.ts
│   └── pipes/                   # custom pipes
│
├── users/                       # a FEATURE module — fully self-contained
│   ├── users.module.ts
│   ├── users.controller.ts
│   ├── users.service.ts
│   ├── users.repository.ts      # optional explicit data-access layer
│   ├── dto/
│   │   ├── create-user.dto.ts
│   │   └── update-user.dto.ts
│   └── entities/
│       └── user.entity.ts
│
├── auth/                        # another feature module
│   ├── auth.module.ts
│   ├── auth.controller.ts
│   ├── auth.service.ts
│   ├── strategies/
│   │   └── jwt.strategy.ts
│   └── guards/
│       └── jwt-auth.guard.ts
│
└── database/                    # shared DB module (Prisma or TypeOrm connection)
    └── database.module.ts
```

> **Best practice:** Put genuinely shared, app-wide concerns (filters, interceptors, decorators) in `common/`. Put a piece of logic in a feature module if it belongs to that feature. When two features need the same provider, move it into a small dedicated module and `export` it rather than duplicating it (see §4 on `exports`).

---

## 4. Modules — the Encapsulation System

### What a module is and the logic behind it

A **module** is a class annotated with `@Module()` that groups together a cohesive set of capabilities. It is the **unit of organization and — crucially — of encapsulation** in NestJS. Every controller and provider belongs to exactly one module. The reason modules matter so much is the *encapsulation boundary* they create: **a provider declared in module A is invisible to module B unless A explicitly `exports` it and B `imports` A.** This is not a minor detail; it is the mechanism that keeps a large app from collapsing into a tangle of global singletons. It forces you to think about the public surface of each feature.

The `@Module()` decorator takes four arrays, and understanding the precise meaning of each is the key to the whole system:

- **`imports`** — *other modules* whose **exported** providers this module needs. Importing a module pulls its exported providers into this module's injector.
- **`controllers`** — the controllers that belong to **this** module (their routes get registered).
- **`providers`** — the injectables (services, factories, custom providers) instantiated and owned by **this** module. They are private to the module by default.
- **`exports`** — the subset of this module's `providers` (or re-exported imported modules) that this module makes **visible** to modules that import it. This is the module's public API.

### A feature module

```typescript
// users/users.module.ts
import { Module } from '@nestjs/common';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  imports: [],                       // this module needs no other modules' exports yet
  controllers: [UsersController],    // routes owned by this module
  providers: [UsersService],         // instantiated & owned here; private by default
  exports: [UsersService],           // make UsersService available to importers (e.g. AuthModule)
})
export class UsersModule {}
```

The logic of `exports: [UsersService]` above: the `AuthModule` will need to look users up during login. By exporting `UsersService`, `UsersModule` declares "this is part of my public contract"; `AuthModule` then does `imports: [UsersModule]` and can inject `UsersService`. Without the export, injecting it from `AuthModule` throws a "Nest can't resolve dependencies" error — the classic beginner trap, and now you know exactly why it happens.

### The root (App) module

```typescript
// app.module.ts — the composition root that wires the whole application
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { UsersModule } from './users/users.module';
import { AuthModule } from './auth/auth.module';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }), // available everywhere; no re-import needed
    UsersModule,
    AuthModule,
  ],
})
export class AppModule {}
```

### Shared modules — the standard way to reuse a provider

There is nothing special about a "shared" module: it is simply any module that **exports** providers so other modules can import and use them. Because NestJS providers are singletons by default (§6), importing a shared module everywhere gives every importer the **same instance** — exactly what you want for a database connection or a config service.

```typescript
// A shared module: export the service so any importer reuses the same instance.
@Module({
  providers: [LoggerService],
  exports: [LoggerService], // ← without this, importers can't inject it
})
export class LoggerModule {}
```

### Global modules — convenience, used sparingly

A module marked `@Global()` registers its **exports** into the global injector scope, so other modules can inject those providers **without importing the module**. The logic: a handful of providers (config, the database connection, an app-wide logger) are needed almost everywhere, and importing them into 40 modules is noise. `@Global()` removes that noise.

**Best practice / caution:** Use `@Global()` sparingly — for genuinely app-wide infrastructure only (config, DB, logging). Overusing it defeats the encapsulation that makes modules valuable, and it hides dependencies (a reader can no longer tell what a module needs by looking at its `imports`). When in doubt, prefer an explicit `imports`.

```typescript
import { Module, Global } from '@nestjs/common';
import { DatabaseService } from './database.service';

@Global() // exported providers are now injectable everywhere without re-importing
@Module({
  providers: [DatabaseService],
  exports: [DatabaseService],
})
export class DatabaseModule {}
```

### Dynamic modules — configure a module at import time

Most third-party modules you use (`ConfigModule.forRoot()`, `TypeOrmModule.forRootAsync()`, `JwtModule.register()`) are **dynamic modules**. A dynamic module is a module whose configuration is supplied **when it is imported**, by calling a static method that returns a `DynamicModule` object. This is the pattern that lets one reusable module behave differently per app (different DB URL, different JWT secret) — the configuration is passed in rather than hard-coded.

The naming convention is meaningful: **`forRoot()`** configures a module once for the whole app (a connection, a global setting); **`forFeature()`** scopes a piece of configuration to a feature (registering specific entities/repositories with the already-established connection); the **`*Async`** variants (`forRootAsync`) let the configuration itself depend on other providers (e.g. read the DB URL from `ConfigService`).

```typescript
// database/database.module.ts — a hand-written dynamic module to see the mechanics
import { Module, DynamicModule } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({}) // empty static metadata — everything is decided at import time
export class DatabaseModule {
  // Caller supplies options; we return a fully-formed module description.
  static forRoot(options: { url: string }): DynamicModule {
    return {
      module: DatabaseModule,
      imports: [
        TypeOrmModule.forRoot({
          type: 'postgres',
          url: options.url,
          autoLoadEntities: true,
          synchronize: false, // NEVER true outside local dev — it can drop columns/data
        }),
      ],
      exports: [TypeOrmModule], // re-export so feature modules can use repositories
    };
  }
}

// Usage in app.module.ts:
//   imports: [ DatabaseModule.forRoot({ url: process.env.DATABASE_URL! }) ]
```

> **Gotcha:** A common confusion is whether to use `forRoot` or `forRootAsync`. Use `forRootAsync` whenever the config must come from another provider (almost always `ConfigService`), because the synchronous `forRoot` runs before DI is ready and can't inject anything. See §10 and §11 for the `useFactory` + `inject` pattern.

### Module lifecycle hooks

Modules and providers can implement lifecycle interfaces so NestJS calls them at well-defined moments. Use these to open/close resources cleanly. The order is: `OnModuleInit` → `OnApplicationBootstrap` → (app runs) → `OnModuleDestroy` → `BeforeApplicationShutdown` → `OnApplicationShutdown`.

```typescript
import { Injectable, OnModuleInit, OnApplicationShutdown } from '@nestjs/common';

@Injectable()
export class ConnectionService implements OnModuleInit, OnApplicationShutdown {
  async onModuleInit() {
    // Runs once the host module's providers are resolved — open connections here.
    console.log('Opening connection pool...');
  }
  async onApplicationShutdown(signal?: string) {
    // Runs on SIGTERM/SIGINT IF app.enableShutdownHooks() was called in main.ts.
    console.log(`Shutting down (${signal}) — closing connection pool...`);
  }
}
```

---

## 5. Controllers & Routing

### What a controller is and why it must stay thin

A **controller** is a class annotated with `@Controller()` whose methods handle incoming requests for a set of routes. Its single responsibility is the **HTTP boundary**: pull data out of the request (params, query, body, headers), hand it to a service, and shape the result back into a response. The cardinal rule — repeated throughout this guide because it is the most-violated one — is **keep controllers thin: no business logic, no database queries, no branching on domain rules.** All of that belongs in services (§6).

The reason is reuse and testability. Business logic in a service can be called from an HTTP controller today, a WebSocket gateway tomorrow, and a queue consumer next week, and it can be unit-tested without any HTTP machinery. Business logic stuck in a controller can only ever run over HTTP and is painful to test.

### Routing — how a path becomes a handler

The `@Controller('users')` argument is the **route prefix**: every method route is appended to it. The HTTP-method decorators (`@Get`, `@Post`, `@Put`, `@Patch`, `@Delete`, plus `@Options`, `@Head`, `@All`) bind a method to a verb + sub-path. So `@Get(':id')` inside `@Controller('users')` handles `GET /users/:id`. NestJS reads these decorators at startup via reflection and registers them with the underlying adapter's router.

```typescript
// users/users.controller.ts — a full CRUD controller, annotated
import {
  Controller, Get, Post, Put, Patch, Delete,
  Param, Body, Query, Headers, Ip,
  HttpCode, HttpStatus, ParseIntPipe, DefaultValuePipe,
} from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

@Controller('users') // base path: /users
export class UsersController {
  // Dependency injection: NestJS sees the typed constructor param and supplies
  // the singleton UsersService. `private readonly` declares + assigns the field
  // in one go (a TypeScript shorthand). The controller NEVER does `new UsersService()`.
  constructor(private readonly usersService: UsersService) {}

  // GET /users?page=2&limit=20  — pipes give defaults and convert types.
  @Get()
  findAll(
    @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
    @Query('limit', new DefaultValuePipe(20), ParseIntPipe) limit: number,
  ) {
    return this.usersService.findAll({ page, limit });
  }

  // GET /users/search?name=Alice&role=admin
  // IMPORTANT: a literal path ('search') must be declared BEFORE the param route
  // (':id'), or '/users/search' would be captured by ':id'. Route order matters.
  @Get('search')
  search(@Query('name') name?: string, @Query('role') role?: string) {
    return this.usersService.search({ name, role });
  }

  // GET /users/:id  — ParseIntPipe converts the string param to a number and
  // throws 400 Bad Request if it isn't numeric, so the service gets a clean number.
  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.findOne(id);
  }

  // POST /users  — Nest returns 201 Created by default for POST.
  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  // PATCH /users/:id  — partial update (the canonical REST verb for partials).
  @Patch(':id')
  update(
    @Param('id', ParseIntPipe) id: number,
    @Body() updateUserDto: UpdateUserDto,
  ) {
    return this.usersService.update(id, updateUserDto);
  }

  // PUT /users/:id  — full replacement of the resource.
  @Put(':id')
  replace(
    @Param('id', ParseIntPipe) id: number,
    @Body() updateUserDto: UpdateUserDto,
  ) {
    return this.usersService.replace(id, updateUserDto);
  }

  // DELETE /users/:id  — 204 No Content is the correct status for a delete
  // that returns nothing. @HttpCode overrides the default (200).
  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  remove(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.remove(id);
  }
}
```

### Parameter decorators — pulling data out of the request

NestJS gives you **declarative parameter decorators** so you never touch the raw `req` object for the common cases. This keeps handlers clean and adapter-agnostic (the same `@Body()` works on Express and Fastify).

| Decorator | What it extracts | Example |
|-----------|------------------|---------|
| `@Param('id')` | A route param | `/users/:id` → `id` |
| `@Param()` | All route params as an object | `{ id, category }` |
| `@Query('page')` | One query-string field | `?page=2` → `'2'` |
| `@Query()` | The whole query object | `{ page, limit }` |
| `@Body()` | The full parsed JSON body | `{ name, email }` |
| `@Body('email')` | One field from the body | `'a@b.com'` |
| `@Headers('authorization')` | One header value | `'Bearer ...'` |
| `@Req()` / `@Request()` | The raw adapter request object | use rarely |
| `@Res()` / `@Response()` | The raw response object | avoid; see gotcha |
| `@Ip()` | The client IP address | for logging/rate limits |
| `@HostParam()` | A param from host-based routing | multi-tenant subdomains |
| `@Session()` | The session object | when using sessions |

> **Security note:** Never trust raw input. Every `@Body()` and `@Query()` value is attacker-controlled until validated. Pair body input with a DTO + `ValidationPipe` (§7), and route/query params with parse pipes (`ParseIntPipe`, `ParseUUIDPipe`). This is your first and most important line of defense.

### Response status codes and headers

```typescript
import { Get, Header, HttpCode, HttpStatus, Res, StreamableFile } from '@nestjs/common';
import { Response } from 'express';
import { createReadStream } from 'fs';

@Get('report')
@HttpCode(HttpStatus.OK)                       // override the default status
@Header('Cache-Control', 'no-store')           // set a static response header
downloadReport(@Res({ passthrough: true }) res: Response) {
  // `passthrough: true` is CRITICAL: it lets you set headers on the raw response
  // BUT keeps NestJS in control of actually sending the body — so interceptors,
  // serialization, and the return-value mechanism still work.
  res.setHeader('Content-Disposition', 'attachment; filename="report.pdf"');
  res.setHeader('Content-Type', 'application/pdf');
  return new StreamableFile(createReadStream('./report.pdf')); // efficient streaming
}
```

> **Gotcha (important):** Injecting `@Res()` **without** `{ passthrough: true }` switches NestJS into "library-specific mode" — you take full manual control of the response, and **interceptors, the automatic JSON serialization, and returning a value all stop working**. Always add `{ passthrough: true }` unless you specifically need raw, manual control (e.g. piping a stream yourself or using SSE). This trips up nearly everyone once.

### Route patterns, wildcards, and sub-routes

```typescript
@Get('profile/me')      // a specific literal sub-path — declare BEFORE ':id'
@Get(':id')             // single dynamic param
@Get(':category/:id')   // multiple params: /electronics/42
@Get('files/*path')     // ⚡ NestJS 11 named wildcard: captures the rest as `path`
```

> **⚡ Version note:** NestJS 11 (running on Express 5 / path-to-regexp 8) changed wildcard syntax. The old bare `*` and `ab*cd` forms are gone; use **named wildcards** like `*path` or `*splat`, and access the captured value via `@Param('path')`. If you upgrade an older app and routes break, this is usually why.

### Sub-domain / host-based routing

```typescript
// Route only requests whose Host header matches admin.<anything>.
@Controller({ host: 'admin.:tenant.example.com' })
export class AdminController {
  @Get()
  index(@HostParam('tenant') tenant: string) {
    return `Admin panel for tenant: ${tenant}`;
  }
}
```

---

## 6. Providers & Dependency Injection (in depth)

This is the conceptual heart of NestJS. Spend time here — once DI clicks, the rest of the framework falls into place.

### What a provider is

A **provider** is any class (or value, or factory) that NestJS's IoC container can create and **inject** into other classes. The most common provider is a **service** — a class decorated with `@Injectable()` that holds business logic. But repositories, factories, configuration objects, helpers, and even plain constant values can all be providers. The defining trait of a provider is that it is managed by the container: you declare it once in a module's `providers` array, and from then on NestJS constructs it and hands it to whoever needs it.

### Dependency Injection and the IoC container — the *why* and the *how*

**Inversion of Control (IoC)** is the principle; **Dependency Injection (DI)** is the technique. Normally a class controls its own dependencies: `this.db = new Database()`. With IoC you *invert* that control — the class no longer creates its dependencies; an external authority (the **container**) creates them and supplies ("injects") them. The class merely *declares what it needs*, typically as constructor parameters.

Why bother? Three concrete payoffs:

1. **Testability.** If `UsersService` creates `new Database()` internally, you cannot test it without a real database. If it *receives* a database via its constructor, your test passes a fake one. This single property is the biggest reason DI exists.
2. **Loose coupling.** Classes depend on *abstractions* (a type/token), not concrete implementations. You can swap the implementation (real ↔ mock, Postgres ↔ in-memory) without touching the dependent class.
3. **Lifetime management.** The container decides whether a dependency is a shared singleton or created per-request, and it wires up the entire dependency graph in the correct order for you — no manual `new` chains.

**How NestJS does it mechanically:** When NestJS instantiates a class, it reads the **types** of the constructor parameters using TypeScript's metadata reflection (enabled by `emitDecoratorMetadata` in `tsconfig.json` and the `reflect-metadata` import). For each parameter type, it looks up a matching provider — the **token** (usually the class itself) — in the module's injector. It recursively builds those dependencies first, then constructs your class with them. This recursive resolution is why you can have deep dependency graphs and never write a single `new`.

```typescript
// The whole DI story in three lines:
@Injectable()
export class UsersService {
  // Nest sees the type `PrismaService`, finds the provider registered under that
  // token, builds it (and ITS dependencies) first, then injects it here.
  constructor(private readonly prisma: PrismaService) {}
}
```

### A canonical service

```typescript
// users/users.service.ts — business logic lives here, NOT in the controller
import { Injectable, NotFoundException, ConflictException } from '@nestjs/common';
import { CreateUserDto } from './dto/create-user.dto';

// A simple in-memory model so this example runs without a database.
interface User { id: number; name: string; email: string; }

@Injectable() // marks the class as a provider the container can manage
export class UsersService {
  private users: User[] = [];
  private nextId = 1;

  findAll(): User[] {
    return this.users;
  }

  findOne(id: number): User {
    const user = this.users.find((u) => u.id === id);
    if (!user) {
      // Throwing a built-in HTTP exception sets the right status automatically.
      // The controller doesn't need to know about 404 — the service owns the rule.
      throw new NotFoundException(`User #${id} not found`);
    }
    return user;
  }

  create(dto: CreateUserDto): User {
    if (this.users.some((u) => u.email === dto.email)) {
      throw new ConflictException('Email already in use'); // → 409
    }
    const user: User = { id: this.nextId++, name: dto.name, email: dto.email };
    this.users.push(user);
    return user;
  }

  update(id: number, dto: Partial<CreateUserDto>): User {
    const user = this.findOne(id); // reuses the 404 logic — DRY
    Object.assign(user, dto);
    return user;
  }

  remove(id: number): void {
    const idx = this.users.findIndex((u) => u.id === id);
    if (idx === -1) throw new NotFoundException(`User #${id} not found`);
    this.users.splice(idx, 1);
  }
}
```

### Provider tokens and the four custom-provider recipes

Most of the time the **token** for a provider is the class itself, and `providers: [UsersService]` is shorthand for `{ provide: UsersService, useClass: UsersService }`. But sometimes you need to inject something that isn't a class you can name as a type — a string config, a third-party object, a computed connection, or a swappable implementation. For those, NestJS gives you four explicit recipes:

```typescript
// 1) useValue — inject a ready-made constant or object. Token is a string here,
//    so consumers must use @Inject('API_KEY') (you can't type-resolve a string).
{ provide: 'API_KEY', useValue: process.env.API_KEY }

// 2) useClass — bind a token to a concrete class. Powerful for swapping impls,
//    e.g. a real mailer in prod, a fake in tests, chosen at module-config time.
{ provide: MailerService, useClass: process.env.NODE_ENV === 'test'
    ? FakeMailerService
    : SmtpMailerService }

// 3) useFactory — compute the provider, optionally depending on OTHER providers.
//    `inject` lists the dependencies passed (in order) to the factory function.
//    Factories may be async — Nest awaits them before the app starts.
{
  provide: 'DB_CONNECTION',
  useFactory: async (config: ConfigService) => {
    const url = config.getOrThrow<string>('DATABASE_URL');
    return createConnection(url); // returns a Promise — Nest awaits it
  },
  inject: [ConfigService],
}

// 4) useExisting — create an alias to an already-registered provider (same instance).
{ provide: 'LOGGER', useExisting: WinstonLoggerService }
```

Injecting a non-class (string/symbol) token requires the **`@Inject()`** decorator, because TypeScript can't reflect a string type:

```typescript
import { Inject, Injectable } from '@nestjs/common';

@Injectable()
export class AppService {
  constructor(
    @Inject('API_KEY') private readonly apiKey: string,
    @Inject('DB_CONNECTION') private readonly db: unknown,
  ) {}
}
```

> **Best practice:** Prefer **`InjectionToken` constants** (exported symbols/strings) over magic string literals to avoid typos and get go-to-definition. For implementation-swapping, code against an **interface** + an abstract class or token, and bind the concrete class with `useClass`. This is how you write code that's trivial to mock and easy to evolve.

### Provider scopes — singleton, request, transient

By default every provider is a **singleton**: NestJS creates one instance and shares it across the entire app for its whole lifetime. This is the right default — it's memory-efficient and fast, and stateless services don't need more. But two other scopes exist for specific needs:

| Scope | Lifetime | When to use |
|-------|----------|-------------|
| `Scope.DEFAULT` (singleton) | One instance for the whole app | Almost always — stateless services |
| `Scope.REQUEST` | A fresh instance per incoming request | Per-request state: tenant id, request-scoped tracing, the current user baked into the instance |
| `Scope.TRANSIENT` | A fresh instance every time it's injected | Each consumer needs its own private, non-shared instance |

```typescript
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.DEFAULT })   // singleton (the default; shown for clarity)
export class ConfigCache {}

@Injectable({ scope: Scope.REQUEST })   // new per HTTP request
export class TenantContext {}

@Injectable({ scope: Scope.TRANSIENT }) // new for each injection site
export class ScratchBuffer {}
```

> **Gotcha — the request-scope performance trap:** Scope **propagates upward**. If a singleton injects a `REQUEST`-scoped provider, the singleton must *also* become request-scoped (it can't hold a reference to something that changes per request), and so on up the chain. This can silently turn much of your graph request-scoped, creating a new object tree on every request and hurting throughput. **Prefer `DEFAULT`**; only use `REQUEST` when you truly need per-request mutable state, and keep it at the leaves of your dependency graph. To read request-scoped data without making a service request-scoped, inject the request via `@Inject(REQUEST)` only where needed, or use `AsyncLocalStorage` (see `@nestjs/core`'s `ClsModule`-style patterns).

### Optional, property, and circular injection

```typescript
import { Injectable, Optional, Inject } from '@nestjs/common';

@Injectable()
export class ReportService {
  // @Optional(): don't throw if no provider is registered — `cache` will be undefined.
  constructor(@Optional() @Inject('CACHE') private readonly cache?: CacheLike) {}
}
```

For **circular dependencies** (Service A needs B, B needs A), use `forwardRef` on both sides (covered in §20). Circular deps are usually a design smell — prefer extracting the shared logic into a third module — but `forwardRef` is the escape hatch when you can't.

---

## 7. DTOs & Validation

### What a DTO is and why validation is non-negotiable

A **DTO (Data Transfer Object)** is a class that defines the **shape and rules of data crossing a boundary** — most often the request body of a POST/PATCH. On its own a DTO is just a typed contract. Its power comes from pairing it with two libraries — **`class-validator`** (declarative validation decorators) and **`class-transformer`** (turning plain JSON into typed class instances) — wired through NestJS's **`ValidationPipe`**.

The *why* is blunt: **all input is hostile until proven otherwise.** TypeScript types vanish at runtime; the compiler cannot stop a client from POSTing `{ "age": "drop table" }`. Validation is the runtime gate that rejects malformed and malicious input *before* it reaches your business logic — it is both a correctness mechanism and a primary **security** control (it's how you enforce length limits, allowed values, and — with `whitelist` — strip unexpected fields that could otherwise reach your ORM).

### Install

```bash
npm install class-validator class-transformer
```

### Defining a DTO with validation rules

```typescript
// users/dto/create-user.dto.ts
import {
  IsEmail, IsString, IsNotEmpty, MinLength, MaxLength,
  IsOptional, IsEnum, IsInt, Min, Max, IsArray, ValidateNested,
} from 'class-validator';
import { Type } from 'class-transformer';

export enum UserRole {
  ADMIN = 'admin',
  USER = 'user',
  MODERATOR = 'moderator',
}

export class CreateUserDto {
  // Each decorator is one rule. They stack; ALL must pass. The pipe collects
  // every failure and returns them together (a 400 with a messages array).
  @IsString()
  @IsNotEmpty()
  @MinLength(2)
  @MaxLength(50)
  name: string;

  @IsEmail() // format check — rejects "not-an-email"
  email: string;

  // Each decorator accepts a `message` override for user-facing errors.
  @IsString()
  @MinLength(8, { message: 'Password must be at least 8 characters' })
  password: string;

  // @IsOptional() means "skip all other validators if the field is absent".
  // Without it, an absent role would fail @IsEnum.
  @IsEnum(UserRole)
  @IsOptional()
  role?: UserRole = UserRole.USER;

  @IsInt()
  @Min(13)
  @Max(120)
  @IsOptional()
  age?: number;
}
```

### Derived DTOs with mapped types — DRY for updates

PATCH endpoints accept *partial* data: every field optional. Rather than hand-writing a second DTO, derive it. `@nestjs/mapped-types` (or `@nestjs/swagger`'s versions, which also carry Swagger metadata) provide transformers:

```bash
npm install @nestjs/mapped-types
```

```typescript
// users/dto/update-user.dto.ts
import { PartialType } from '@nestjs/mapped-types';
import { CreateUserDto } from './create-user.dto';

// Every field of CreateUserDto becomes optional, validators preserved.
export class UpdateUserDto extends PartialType(CreateUserDto) {}
```

| Helper | Effect |
|--------|--------|
| `PartialType(T)` | All fields of `T` become optional (perfect for PATCH) |
| `PickType(T, ['a','b'])` | Keep only the listed fields |
| `OmitType(T, ['password'])` | Drop the listed fields |
| `IntersectionType(A, B)` | Combine fields of two DTOs |

> **⚡ Note:** If you use Swagger (§13), import these from `@nestjs/swagger` instead of `@nestjs/mapped-types` so the generated OpenAPI docs stay correct.

### Enabling `ValidationPipe` globally — the canonical config

Apply the pipe once in `main.ts` so every route with a DTO is validated automatically. The options below are the security-conscious baseline you should treat as the default:

```typescript
// main.ts
import { ValidationPipe } from '@nestjs/common';

app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,            // STRIP any property not declared on the DTO.
                                //   Security-critical: stops mass-assignment of
                                //   fields the client shouldn't control (e.g. isAdmin).
    forbidNonWhitelisted: true, // Go further: 400 if unknown props are present,
                                //   so clients can't silently send junk.
    transform: true,            // Convert the plain JSON object into an actual
                                //   DTO instance, and coerce primitive types.
    transformOptions: {
      enableImplicitConversion: true, // "5" in a query string → number 5 when the
                                      //   DTO/handler types it as number. Convenient,
                                      //   but be deliberate — it can mask type bugs.
    },
    // disableErrorMessages: process.env.NODE_ENV === 'production', // optional: hide
    //   detailed validation messages in prod to avoid leaking schema details.
  }),
);
```

> **Best practice:** Always set `whitelist: true`. The `whitelist`/`forbidNonWhitelisted` pair is one of the cheapest, highest-value security controls in the framework — it neutralizes **mass-assignment** attacks (a client sneaking `"role":"admin"` into a profile update) by default.

### Nested objects and arrays

`class-validator` cannot, by itself, validate nested objects — it doesn't know the inner class at runtime. You must tell `class-transformer` the type with `@Type()` and tell the validator to recurse with `@ValidateNested()`:

```typescript
import { IsString, IsArray, ValidateNested } from 'class-validator';
import { Type } from 'class-transformer';

class AddressDto {
  @IsString() street: string;
  @IsString() city: string;
}

export class CreateProfileDto {
  @IsString() name: string;

  @ValidateNested()        // recurse INTO the nested object
  @Type(() => AddressDto)  // tell class-transformer which class to build
  address: AddressDto;

  @IsArray()
  @ValidateNested({ each: true }) // validate EACH element of the array
  @Type(() => AddressDto)
  previousAddresses: AddressDto[];
}
```

### Common `class-validator` decorators (reference)

| Decorator | Validates |
|-----------|-----------|
| `@IsString()` `@IsNumber()` `@IsInt()` `@IsBoolean()` | Primitive type |
| `@IsEmail()` `@IsUrl()` `@IsUUID()` `@IsDate()` `@IsJSON()` | Format |
| `@IsEnum(MyEnum)` | Membership in an enum |
| `@IsNotEmpty()` `@IsOptional()` | Presence rules |
| `@IsArray()` `@ArrayMinSize(n)` `@ArrayMaxSize(n)` | Array shape |
| `@MinLength(n)` `@MaxLength(n)` `@Length(min,max)` | String length |
| `@Min(n)` `@Max(n)` | Numeric range |
| `@Matches(/regex/)` | Regex pattern |
| `@IsStrongPassword()` | Built-in password strength (configurable) |
| `@ValidateNested()` | Recurse into nested objects |
| `@ValidateIf((o) => ...)` | Conditional validation |

### Custom validation — three levels

When the built-ins aren't enough, NestJS/`class-validator` give you three escalating tools.

#### 1) A reusable custom constraint (`@ValidatorConstraint`)

Use this for a rule you'll apply across many DTOs (e.g. a project-specific strong-password policy). Implement `ValidatorConstraintInterface`, then wrap it in a decorator with `registerDecorator` so DTOs use a clean `@IsStrongPassword()` syntax.

```typescript
import {
  registerDecorator, ValidationOptions, ValidatorConstraint,
  ValidatorConstraintInterface, ValidationArguments,
} from 'class-validator';

@ValidatorConstraint({ name: 'isStrongPassword', async: false })
export class IsStrongPasswordConstraint implements ValidatorConstraintInterface {
  // Return true when valid. Here: 8+ chars with upper, lower, digit, symbol.
  validate(value: string): boolean {
    if (typeof value !== 'string') return false;
    return /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^A-Za-z0-9]).{8,}$/.test(value);
  }
  // The message used when validation fails. args.property = the field name.
  defaultMessage(args: ValidationArguments): string {
    return `${args.property} must be 8+ chars with upper, lower, number and symbol`;
  }
}

// Clean decorator wrapper so DTOs just write @IsStrongPassword().
export function IsStrongPassword(options?: ValidationOptions) {
  return (object: object, propertyName: string) =>
    registerDecorator({
      target: object.constructor,
      propertyName,
      options,
      constraints: [],
      validator: IsStrongPasswordConstraint,
    });
}
```

#### 2) Cross-field validation (compare two properties)

A frequent need: `confirmPassword` must equal `password`. The validator receives the whole object via `args.object`, so you can compare sibling fields; pass the other field's name through `constraints`.

```typescript
import { registerDecorator, ValidationOptions, ValidationArguments } from 'class-validator';

export function Match(property: string, options?: ValidationOptions) {
  return (object: object, propertyName: string) =>
    registerDecorator({
      target: object.constructor,
      propertyName,
      options,
      constraints: [property], // remembered, read back in validate()
      validator: {
        validate(value: unknown, args: ValidationArguments) {
          const [related] = args.constraints as string[];
          return value === (args.object as Record<string, unknown>)[related];
        },
        defaultMessage(args: ValidationArguments) {
          return `${args.property} must match ${args.constraints[0]}`;
        },
      },
    });
}

// Usage:
export class ChangePasswordDto {
  @IsStrongPassword() password: string;
  @Match('password', { message: 'Passwords do not match' }) confirmPassword: string;
}
```

#### 3) Async validation with DI (e.g. "email already taken")

Set `async: true`, return a `Promise<boolean>`, and inject a service to hit the database. The catch: `class-validator` instantiates constraints itself by default, so injected services come back `undefined`. The fix is `useContainer` in `main.ts`, which tells `class-validator` to resolve constraints through Nest's DI container.

```typescript
import { Injectable } from '@nestjs/common';
import {
  ValidatorConstraint, ValidatorConstraintInterface, ValidationArguments,
} from 'class-validator';
import { UsersService } from './users.service';

@ValidatorConstraint({ name: 'isEmailUnique', async: true })
@Injectable() // makes it injectable AND lets it receive injected deps
export class IsEmailUniqueConstraint implements ValidatorConstraintInterface {
  constructor(private readonly usersService: UsersService) {}
  async validate(email: string): Promise<boolean> {
    return !(await this.usersService.findByEmail(email)); // valid only if no match
  }
  defaultMessage(args: ValidationArguments) {
    return `${args.property} is already registered`;
  }
}
```

```typescript
// main.ts — REQUIRED for DI inside custom validators:
import { useContainer } from 'class-validator';
import { AppModule } from './app.module';

const app = await NestFactory.create(AppModule);
useContainer(app.select(AppModule), { fallbackOnErrors: true });
```

> **Security note:** Async DB-backed validators are convenient but run a query on every request — they can become a denial-of-service vector and a race condition (two requests pass the "unique" check simultaneously). Treat them as a UX nicety, and enforce true uniqueness with a **database unique constraint** as the real guarantee (catch the resulting error and translate it to a 409). See `POSTGRESQL_GUIDE.md` on unique constraints.

#### A fully custom validation **pipe** (non-DTO cases)

For one-off validation outside the DTO model (e.g. a single route param), write a class implementing `PipeTransform`. Prefer the built-in `ValidationPipe` for DTOs; reach for this only for special cases.

```typescript
import { PipeTransform, Injectable, BadRequestException } from '@nestjs/common';

@Injectable()
export class PositiveIntPipe implements PipeTransform<string, number> {
  transform(value: string): number {
    const n = Number(value);
    if (!Number.isInteger(n) || n <= 0) {
      throw new BadRequestException('Parameter must be a positive integer');
    }
    return n; // controller receives a clean, validated number
  }
}
// Usage: @Get(':id') findOne(@Param('id', PositiveIntPipe) id: number) {}
```

---

## 8. The Request Lifecycle: Pipes, Guards, Interceptors, Middleware & Filters

These five layers wrap your controller handlers. The single most valuable thing to internalize is **which one runs when** (revisit the diagram in §1) — because that determines where each kind of logic belongs. Mnemonic: **M**iddleware → **G**uards → **I**nterceptors(pre) → **P**ipes → **Handler** → **I**nterceptors(post) → **Filters** (on error). Below, each layer is explained — what it is, when it runs, what it's for — then shown with runnable code.

### Middleware — runs first, closest to the metal

**What/when:** Middleware runs **before** routing resolves to a handler, operating on the raw adapter `req`/`res`. It is the Express/Fastify middleware concept, integrated into NestJS. **What for:** logging, request id assignment, body/cookie parsing, CORS, helmet, raw rate limiting — concerns that are about the *transport*, not your domain. Because it runs before guards, it does **not** have access to the resolved handler/class metadata (use a guard or interceptor if you need that).

```typescript
// common/middleware/logger.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable() // class middleware can use DI (functional middleware cannot)
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const start = Date.now();
    const { method, originalUrl } = req;
    res.on('finish', () => {
      // 'finish' fires after the response is sent — accurate duration & status.
      console.log(`${method} ${originalUrl} ${res.statusCode} +${Date.now() - start}ms`);
    });
    next(); // MUST call next(), or the request hangs forever
  }
}
```

```typescript
// app.module.ts — middleware is wired in the module's configure(), not via a decorator
import { Module, NestModule, MiddlewareConsumer, RequestMethod } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';

@Module({ /* ... */ })
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .exclude({ path: 'health', method: RequestMethod.GET }) // skip noisy routes
      .forRoutes('*'); // or a specific controller / { path, method }
  }
}
```

### Guards — the authorization gate

**What/when:** A guard runs **after middleware, before interceptors and pipes**. It implements `CanActivate` and answers exactly one yes/no question: **"is this request allowed to proceed to the handler?"** Return `true` to allow; return `false` or throw (e.g. `ForbiddenException`) to deny. **What for:** authentication (is there a valid user?) and authorization (does the user have the right role/permission?). Guards run before pipes, so they protect a route *before* the (potentially expensive) body validation runs.

The key tool a guard uses is the **`Reflector`**, which reads metadata that decorators (`@Roles('admin')`, `@Public()`) attached to the handler or class. This metadata-driven pattern is how you write one guard and configure it per-route with decorators.

```typescript
// common/guards/roles.guard.ts — role-based access control (RBAC)
import {
  Injectable, CanActivate, ExecutionContext, ForbiddenException,
} from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from '../decorators/roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    // getAllAndOverride checks method-level metadata first, then class-level —
    // so a method @Roles() overrides a controller-wide @Roles().
    const required = this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [
      context.getHandler(), // the route method
      context.getClass(),   // the controller class
    ]);
    if (!required || required.length === 0) return true; // no restriction → allow

    // ExecutionContext abstracts HTTP/WS/RPC; switchToHttp() gets the HTTP request.
    const { user } = context.switchToHttp().getRequest();
    const ok = required.some((role) => user?.roles?.includes(role));
    if (!ok) throw new ForbiddenException('Insufficient permissions'); // → 403
    return true;
  }
}
```

```typescript
// common/decorators/roles.decorator.ts — attach role metadata declaratively
import { SetMetadata } from '@nestjs/common';
export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);

// Usage on a route:
//   @Roles('admin')
//   @UseGuards(JwtAuthGuard, RolesGuard) // JwtAuthGuard sets req.user; RolesGuard checks it
//   @Delete(':id') remove(...) {}
```

How to apply guards, from narrowest to widest scope:

```typescript
@UseGuards(AuthGuard)                 // one route or one controller
app.useGlobalGuards(new AuthGuard()); // globally, but CANNOT use DI (built before container)
// Preferred global registration — allows DI (Reflector, services):
//   providers: [{ provide: APP_GUARD, useClass: JwtAuthGuard }]
```

### Interceptors — wrap the handler (before *and* after)

**What/when:** An interceptor implements `NestInterceptor` and **wraps** handler execution: code in it runs **before** the handler, and — because it works with an RxJS `Observable` of the handler's result — also **after**, letting you transform the response or react to errors. **What for:** uniform response shaping, logging/timing, caching, response serialization, timeouts, and mapping exceptions. The `next.handle()` call returns the handler's result stream; you `.pipe()` RxJS operators onto it.

```typescript
// common/interceptors/transform.interceptor.ts — wrap every response in an envelope
import {
  Injectable, NestInterceptor, ExecutionContext, CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

interface Envelope<T> { data: T; statusCode: number; timestamp: string; }

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Envelope<T>> {
  intercept(ctx: ExecutionContext, next: CallHandler): Observable<Envelope<T>> {
    // (anything before next.handle() runs BEFORE the handler)
    return next.handle().pipe(  // next.handle() runs the handler, returns its result
      map((data) => ({          // transform the result AFTER the handler returns
        data,
        statusCode: ctx.switchToHttp().getResponse().statusCode,
        timestamp: new Date().toISOString(),
      })),
    );
  }
}
```

```typescript
// common/interceptors/timeout.interceptor.ts — fail slow requests fast
import {
  Injectable, NestInterceptor, ExecutionContext, CallHandler, RequestTimeoutException,
} from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(_ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    return next.handle().pipe(
      timeout(5000), // after 5s, the stream errors with TimeoutError
      catchError((err) =>
        err instanceof TimeoutError
          ? throwError(() => new RequestTimeoutException()) // → 408
          : throwError(() => err),
      ),
    );
  }
}
```

```typescript
// Apply: @UseInterceptors(TransformInterceptor) per route/controller, or globally:
app.useGlobalInterceptors(new TransformInterceptor());
// DI-enabled global: providers: [{ provide: APP_INTERCEPTOR, useClass: TransformInterceptor }]
```

### Pipes — validate & transform handler inputs

**What/when:** A pipe runs **just before** the handler, on the **arguments bound to handler parameters** (a `@Body()`, a `@Param()`, a `@Query()`). It implements `PipeTransform` and either **transforms** the value (string → number) or **validates** it (throw on bad input). **What for:** DTO validation (`ValidationPipe`, §7) and primitive parsing/coercion. Pipes run *after* guards, which is deliberate — you don't waste validation effort on a request that auth will reject anyway.

| Built-in pipe | Purpose |
|---------------|---------|
| `ValidationPipe` | Validate DTOs via class-validator |
| `ParseIntPipe` / `ParseFloatPipe` | String → number, 400 on failure |
| `ParseBoolPipe` | `"true"/"false"` → boolean |
| `ParseArrayPipe` | CSV/JSON → array (with item validation) |
| `ParseUUIDPipe` | Validate a UUID param |
| `ParseEnumPipe` | Validate enum membership |
| `DefaultValuePipe` | Substitute a default when the value is undefined |
| `ParseFilePipe` | Validate uploaded files (size/type) |

```typescript
// Pipes compose left-to-right: default first, then parse.
@Get()
list(
  @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
  @Query('limit', new DefaultValuePipe(20), ParseIntPipe) limit: number,
) {
  return this.service.list({ page, limit });
}
```

```typescript
// A custom transform pipe — trim whitespace off strings before they hit the handler.
import { Injectable, PipeTransform } from '@nestjs/common';

@Injectable()
export class TrimPipe implements PipeTransform {
  transform(value: unknown) {
    if (typeof value === 'string') return value.trim();
    if (value && typeof value === 'object') {
      for (const k of Object.keys(value)) {
        const v = (value as Record<string, unknown>)[k];
        if (typeof v === 'string') (value as Record<string, unknown>)[k] = v.trim();
      }
    }
    return value;
  }
}
```

### Exception filters — the last line (see §9 for depth)

**What/when:** A filter implements `ExceptionFilter` and catches exceptions thrown **anywhere** in the pipeline above (guards, pipes, handler, interceptors). **What for:** turning errors into a consistent JSON error response and logging them. Because they run on the error path regardless of where the error originated, filters are how you guarantee every error your API emits has the same shape. Covered fully in §9.

### Execution order summary & where to put what

| Layer | Runs | Put here… |
|-------|------|-----------|
| Middleware | First, raw req/res | logging, cors, helmet, body parsing, request id |
| Guard | Before interceptors/pipes | authn/authz (is this allowed?) |
| Interceptor (pre) | Before handler | start timers, set up tracing context |
| Pipe | Just before handler | validate & transform inputs |
| Handler | — | call a service, return a result |
| Interceptor (post) | After handler | shape response, cache, log duration |
| Filter | On any thrown error | format & log the error response |

---

## 9. Exception Handling

### How NestJS handles errors out of the box

NestJS ships a **global exception layer**. When code anywhere in the request pipeline throws, NestJS catches it. If the thrown object is an **`HttpException`** (or a subclass), Nest reads its status and response body and sends a correct HTTP error. If it's any other error, Nest sends a generic **500 Internal Server Error** (and logs the stack server-side, never leaking it to the client). This is why you can simply `throw new NotFoundException()` from deep in a service and get a clean 404 — no manual `res.status(404)` plumbing.

### Built-in HTTP exceptions

Throw these from anywhere (services especially) and the right status flows out automatically. This keeps HTTP status decisions next to the business rule that triggers them, while keeping controllers thin.

```typescript
import {
  BadRequestException,          // 400
  UnauthorizedException,        // 401 — not authenticated
  ForbiddenException,           // 403 — authenticated but not allowed
  NotFoundException,            // 404
  ConflictException,            // 409 — e.g. duplicate email
  GoneException,                // 410
  PayloadTooLargeException,     // 413
  UnprocessableEntityException, // 422
  TooManyRequestsException,     // 429
  InternalServerErrorException, // 500
  ServiceUnavailableException,  // 503
} from '@nestjs/common';

throw new NotFoundException(`User #${id} not found`);
// You can pass an object to enrich the body:
throw new ConflictException({ message: 'Email already exists', field: 'email' });
// Custom status + cause:
// throw new HttpException('Teapot', 418, { cause: originalError });
```

### A custom HTTP-exception filter

Use a filter to give every error a **uniform body** (status, timestamp, path, message) and to log it. `@Catch(HttpException)` narrows this filter to HTTP exceptions; a `@Catch()` with no argument catches everything (next snippet).

```typescript
// common/filters/http-exception.filter.ts
import {
  ExceptionFilter, Catch, ArgumentsHost, HttpException, Logger,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException) // only catches HttpException and its subclasses
export class HttpExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(HttpExceptionFilter.name);

  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse<Response>();
    const req = ctx.getRequest<Request>();
    const status = exception.getStatus();
    const payload = exception.getResponse(); // string OR object

    const body = {
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: req.url,
      method: req.method,
      message: typeof payload === 'object' ? (payload as any).message : payload,
    };

    this.logger.error(`${req.method} ${req.url} ${status}`, JSON.stringify(body));
    res.status(status).json(body);
  }
}
```

### A catch-all filter (last-resort safety net)

```typescript
import {
  ExceptionFilter, Catch, ArgumentsHost, HttpException, HttpStatus, Logger,
} from '@nestjs/common';
import { Response } from 'express';

@Catch() // no argument = catch EVERYTHING (including non-HTTP errors)
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);

  catch(exception: unknown, host: ArgumentsHost) {
    const res = host.switchToHttp().getResponse<Response>();
    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    // Log the FULL error server-side for debugging...
    this.logger.error(exception instanceof Error ? exception.stack : String(exception));

    // ...but NEVER leak internals to the client on a 500 (security).
    res.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      message:
        exception instanceof HttpException
          ? exception.message
          : 'Internal server error', // generic — no stack, no DB error text
    });
  }
}
```

```typescript
// Register globally. ORDER matters: more specific filters before catch-all.
app.useGlobalFilters(new AllExceptionsFilter());
// DI-enabled: providers: [{ provide: APP_FILTER, useClass: AllExceptionsFilter }]
```

> **Security note:** Exception messages are an information-disclosure surface. A raw database error ("duplicate key value violates unique constraint users_email_key") tells an attacker your table and column names. Always translate low-level errors into safe, generic messages for clients, while logging the detail server-side. In production, consider `disableErrorMessages: true` on `ValidationPipe` and never include stack traces in responses.

---

## 10. Configuration (@nestjs/config)

### Why a config module instead of `process.env`

Reading `process.env` directly everywhere has three problems: values are always `string | undefined` (no typing), there's no validation (a missing `DATABASE_URL` fails at the worst possible moment, mid-request, instead of at startup), and config becomes scattered and untestable. **`@nestjs/config`** solves all three: it loads `.env` files, lets you define a **typed config factory**, **validates** required variables at boot (fail fast), and exposes everything through an injectable **`ConfigService`**.

```bash
npm install @nestjs/config
npm install joi   # schema validation of environment variables
```

### A typed config factory

```typescript
// config/configuration.ts — one place that maps env vars to a typed structure
export default () => ({
  port: parseInt(process.env.PORT ?? '3000', 10),
  database: {
    url: process.env.DATABASE_URL,
    ssl: process.env.DATABASE_SSL === 'true',
  },
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRES_IN ?? '15m',
  },
  redis: {
    host: process.env.REDIS_HOST ?? 'localhost',
    port: parseInt(process.env.REDIS_PORT ?? '6379', 10),
  },
});
```

### Registering ConfigModule with startup validation

```typescript
// app.module.ts
import { ConfigModule } from '@nestjs/config';
import * as Joi from 'joi';
import configuration from './config/configuration';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,           // inject ConfigService anywhere without re-importing
      load: [configuration],    // load the typed factory above
      envFilePath: ['.env.local', '.env'], // multiple files; earlier wins
      cache: true,              // cache parsed env for faster lookups
      validationSchema: Joi.object({
        NODE_ENV: Joi.string().valid('development', 'production', 'test').default('development'),
        PORT: Joi.number().default(3000),
        DATABASE_URL: Joi.string().required(),       // app refuses to start if missing
        JWT_SECRET: Joi.string().required().min(32), // enforce a strong secret length
      }),
      validationOptions: { abortEarly: false }, // report ALL bad vars at once, not just the first
    }),
  ],
})
export class AppModule {}
```

The logic of `abortEarly: false`: when several env vars are wrong, you want to see *all* of them in one startup error and fix them together, rather than play whack-a-mole one restart at a time.

### Using ConfigService

```typescript
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class SomeService {
  constructor(private readonly config: ConfigService) {}

  example() {
    const port = this.config.get<number>('port');             // typed read
    const dbUrl = this.config.get<string>('database.url');    // dotted path into the factory
    const secret = this.config.getOrThrow<string>('jwt.secret'); // throws if absent — use for required values
    return { port, dbUrl, secret };
  }
}
```

> **⚡ Tip:** `ConfigService` is generic-friendly. With `infer: true` you get strict typing: `config.get('jwt.secret', { infer: true })`. For required values prefer `getOrThrow` so a misconfiguration surfaces immediately rather than as a confusing downstream `undefined`.

### `.env` example

```bash
# .env  — local development values; NEVER commit real secrets
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
JWT_SECRET=super-long-random-secret-at-least-32-characters-long
JWT_EXPIRES_IN=15m
REDIS_HOST=localhost
REDIS_PORT=6379
```

> **Security recommendations (critical):**
> - **Never commit `.env`.** Add it to `.gitignore`. Commit a `.env.example` with placeholder (non-secret) values so teammates know what's required.
> - **Generate secrets with a CSPRNG**, e.g. `node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"` — never a human-typed string.
> - **In production, inject secrets via the platform** (Docker secrets, Kubernetes Secrets, AWS Secrets Manager, Railway/Render env vars) rather than a file on disk.
> - **Validate at startup** (the Joi schema above) so a missing or weak secret crashes the deploy instead of silently weakening security.

---

## 11. Database Integration — Prisma, TypeORM & Mongoose

NestJS is database-agnostic. The three mainstream choices are **Prisma** (schema-first, best-in-class DX and type-safety — the modern default), **TypeORM** (decorator/Active-Record-or-DataMapper, deeply integrated via `@nestjs/typeorm`), and **Mongoose** (for MongoDB). This section covers all three; **Prisma is presented first and most fully** because it is the recommended choice for new projects — see `PRISMA_ORM_GUIDE.md` for the ORM in depth and `POSTGRESQL_GUIDE.md` for the database itself.

### Prisma (recommended for new projects)

**What it is and why:** Prisma is a **schema-first** ORM. You describe your data model in a single `schema.prisma` file; Prisma **generates a fully type-safe client** from it. The payoff is end-to-end type safety: if you query a column that doesn't exist, TypeScript errors at compile time, and query results are typed precisely (including selected fields and included relations). Its migration tooling is first-class. The integration pattern with NestJS is simple: wrap `PrismaClient` in an injectable service.

```bash
npm install prisma @prisma/client
npx prisma init   # creates prisma/schema.prisma and a .env entry
```

```prisma
// prisma/schema.prisma — the single source of truth for your data model
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL") // read from .env
}

model User {
  id        String   @id @default(uuid())
  name      String
  email     String   @unique          // DB-level uniqueness = the real guarantee
  password  String
  role      Role     @default(USER)
  posts     Post[]                     // one-to-many relation
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Post {
  id        String   @id @default(uuid())
  title     String
  content   String?
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id])
  authorId  String
  createdAt DateTime @default(now())
}

enum Role { USER ADMIN MODERATOR }
```

```bash
npx prisma migrate dev --name init  # create + apply a migration in dev
npx prisma generate                 # regenerate the typed client after schema edits
npx prisma studio                   # GUI to browse/edit data
```

```typescript
// database/prisma.service.ts — wrap PrismaClient as an injectable, lifecycle-aware service
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  async onModuleInit() {
    await this.$connect();    // open the connection pool when the module starts
  }
  async onModuleDestroy() {
    await this.$disconnect(); // close it cleanly on shutdown
  }
}
```

```typescript
// database/database.module.ts — expose Prisma app-wide
import { Module, Global } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global() // DB access is needed almost everywhere — a justified use of @Global
@Module({ providers: [PrismaService], exports: [PrismaService] })
export class DatabaseModule {}
```

```typescript
// users/users.service.ts (Prisma) — note the explicit `select` to exclude password
import { Injectable, NotFoundException } from '@nestjs/common';
import { PrismaService } from '../database/prisma.service';
import { CreateUserDto } from './dto/create-user.dto';
import * as bcrypt from 'bcrypt';

@Injectable()
export class UsersService {
  constructor(private readonly prisma: PrismaService) {}

  findAll() {
    return this.prisma.user.findMany({
      orderBy: { createdAt: 'desc' },
      // SECURITY: never return the password hash. Whitelist the safe fields.
      select: { id: true, name: true, email: true, role: true, createdAt: true },
    });
  }

  async findOne(id: string) {
    const user = await this.prisma.user.findUnique({ where: { id } });
    if (!user) throw new NotFoundException(`User ${id} not found`);
    return user;
  }

  findByEmail(email: string) {
    // Used by auth — this one DOES include password for bcrypt comparison.
    return this.prisma.user.findUnique({ where: { email } });
  }

  async create(dto: CreateUserDto) {
    const password = await bcrypt.hash(dto.password, 12); // hash BEFORE storing
    return this.prisma.user.create({
      data: { name: dto.name, email: dto.email, password },
      select: { id: true, name: true, email: true, role: true }, // never echo the hash
    });
  }

  update(id: string, data: Partial<CreateUserDto>) {
    return this.prisma.user.update({ where: { id }, data });
  }

  remove(id: string) {
    return this.prisma.user.delete({ where: { id } });
  }

  // Eager-load a relation, filtered.
  findWithPosts(id: string) {
    return this.prisma.user.findUnique({
      where: { id },
      include: { posts: { where: { published: true } } },
    });
  }

  // A transaction: both writes succeed or both roll back atomically.
  async createUserWithPost(dto: CreateUserDto, title: string) {
    return this.prisma.$transaction(async (tx) => {
      const user = await tx.user.create({ data: { ...dto, password: '...' } });
      const post = await tx.post.create({ data: { title, authorId: user.id } });
      return { user, post };
    });
  }
}
```

> **Cross-reference:** Prisma's query API, relations, migrations, and advanced patterns (raw queries, middleware/extensions, connection pooling with PgBouncer) are covered in `PRISMA_ORM_GUIDE.md`. The PostgreSQL features Prisma maps to (indexes, constraints, JSONB, transactions isolation levels) are in `POSTGRESQL_GUIDE.md`.

### TypeORM

**What it is and why:** TypeORM is a mature ORM that models tables as **decorated entity classes** and gives you a `Repository` per entity. It's tightly integrated through `@nestjs/typeorm`. Choose it if you prefer entity-class decorators, need its query builder, or are maintaining an existing TypeORM codebase. Its `forRootAsync` setup is the textbook example of a dynamic module reading config from `ConfigService`.

```bash
npm install @nestjs/typeorm typeorm pg   # pg = the PostgreSQL driver
```

```typescript
// app.module.ts — async config so the DB URL comes from ConfigService
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        type: 'postgres',
        url: config.getOrThrow<string>('DATABASE_URL'),
        autoLoadEntities: true, // pick up entities registered via forFeature()
        // synchronize auto-creates/alters tables to match entities. CONVENIENT in
        // local dev, CATASTROPHIC in prod (it can drop columns and data). Gate it.
        synchronize: config.get('NODE_ENV') === 'development',
        logging: config.get('NODE_ENV') === 'development',
      }),
    }),
  ],
})
export class AppModule {}
```

```typescript
// users/entities/user.entity.ts
import {
  Entity, Column, PrimaryGeneratedColumn,
  CreateDateColumn, UpdateDateColumn, Index, BeforeInsert,
} from 'typeorm';
import * as bcrypt from 'bcrypt';

@Entity('users') // maps to the "users" table
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ length: 50 })
  name: string;

  @Index({ unique: true })
  @Column({ unique: true })
  email: string;

  // select: false → this column is NOT returned by default queries (security).
  @Column({ select: false })
  password: string;

  @Column({ type: 'enum', enum: ['admin', 'user', 'moderator'], default: 'user' })
  role: string;

  @Column({ default: true })
  isActive: boolean;

  @CreateDateColumn() createdAt: Date;
  @UpdateDateColumn() updatedAt: Date;

  // Entity lifecycle hook: hash the password right before the row is inserted.
  @BeforeInsert()
  async hashPassword() {
    this.password = await bcrypt.hash(this.password, 12);
  }
}
```

```typescript
// users/users.module.ts — forFeature registers the repository for THIS module
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './entities/user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])], // provides Repository<User>
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

```typescript
// users/users.service.ts (TypeORM) — inject the repository, use it
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './entities/user.entity';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User) private readonly userRepo: Repository<User>,
  ) {}

  findAll() {
    return this.userRepo.find({ where: { isActive: true }, order: { createdAt: 'DESC' } });
  }

  async findOne(id: string) {
    const user = await this.userRepo.findOne({ where: { id } });
    if (!user) throw new NotFoundException(`User ${id} not found`);
    return user;
  }

  findByEmail(email: string) {
    // Query builder lets us re-include the select:false password column for auth.
    return this.userRepo
      .createQueryBuilder('user')
      .addSelect('user.password')
      .where('user.email = :email', { email })
      .getOne();
  }

  create(dto: CreateUserDto) {
    const user = this.userRepo.create(dto); // builds instance, runs @BeforeInsert on save
    return this.userRepo.save(user);
  }

  // A transaction via the entity manager — all-or-nothing.
  transferCredits(fromId: string, toId: string, amount: number) {
    return this.userRepo.manager.transaction(async (mgr) => {
      // ...load both, mutate, save, within one atomic transaction
    });
  }
}
```

> **Gotcha:** **Never** run `synchronize: true` against staging or production — it can silently drop columns and destroy data. Use migrations:
> ```bash
> npx typeorm migration:generate src/migrations/Init -d src/data-source.ts
> npx typeorm migration:run -d src/data-source.ts
> ```

### Mongoose (MongoDB)

```bash
npm install @nestjs/mongoose mongoose
```

```typescript
// app.module.ts
import { MongooseModule } from '@nestjs/mongoose';

@Module({ imports: [MongooseModule.forRoot(process.env.MONGODB_URI!)] })
export class AppModule {}
```

```typescript
// users/schemas/user.schema.ts
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { HydratedDocument } from 'mongoose';

export type UserDocument = HydratedDocument<User>;

@Schema({ timestamps: true })
export class User {
  @Prop({ required: true }) name: string;
  @Prop({ required: true, unique: true, lowercase: true }) email: string;
  @Prop({ required: true, select: false }) password: string; // hidden by default
}
export const UserSchema = SchemaFactory.createForClass(User);
```

```typescript
// users.module.ts: imports: [MongooseModule.forFeature([{ name: User.name, schema: UserSchema }])]
// users.service.ts:
//   constructor(@InjectModel(User.name) private userModel: Model<UserDocument>) {}
```

> See `MONGODB_GUIDE.md` for MongoDB modeling, indexing, and aggregation depth.

---

## 12. Authentication & Authorization (Passport, JWT, RBAC)

### The mental model: authentication vs authorization

Two distinct questions, two distinct mechanisms — keep them separate in your head:

- **Authentication ("who are you?")** — verifying identity. Implemented as a login flow (verify a password) that issues a **JWT**, plus a **guard** that verifies the JWT on subsequent requests and attaches the user to `req.user`.
- **Authorization ("what may you do?")** — checking permissions for an *already-authenticated* user. Implemented as a second guard (`RolesGuard`, §8) that reads role/permission metadata.

NestJS implements authentication via **Passport** (the standard Node auth library) wrapped by `@nestjs/passport`, plus `@nestjs/jwt` for signing/verifying tokens. The **JWT** itself is a signed, stateless token: the server signs a small payload (user id, role) with a secret; the client sends it back on each request in `Authorization: Bearer <token>`; the server verifies the signature without a database lookup. For a deeper, framework-agnostic auth system (sessions, social login, account linking) see `BETTERAUTH_GUIDE.md`.

```bash
npm install @nestjs/passport passport passport-jwt @nestjs/jwt bcrypt
npm install --save-dev @types/passport-jwt @types/bcrypt
```

### The JWT strategy — how tokens get verified

A Passport **strategy** is a pluggable verification method. `JwtStrategy` extracts the bearer token, verifies its signature/expiry with your secret, and — in `validate()` — turns the decoded payload into the `req.user` object the rest of the app sees.

```typescript
// auth/strategies/jwt.strategy.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { ConfigService } from '@nestjs/config';
import { UsersService } from '../../users/users.service';

export interface JwtPayload { sub: string; email: string; role: string } // sub = subject (user id)

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(
    config: ConfigService,
    private readonly usersService: UsersService,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(), // "Authorization: Bearer x"
      ignoreExpiration: false,                 // reject expired tokens (do NOT set true)
      secretOrKey: config.getOrThrow<string>('jwt.secret'),
    });
  }

  // Runs only AFTER the signature & expiry are verified. The return value
  // becomes req.user. Re-fetching the user lets you reject deactivated accounts.
  async validate(payload: JwtPayload) {
    const user = await this.usersService.findOne(payload.sub);
    if (!user) throw new UnauthorizedException();
    return user; // → req.user
  }
}
```

### The auth service — login & password verification

```typescript
// auth/auth.service.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { UsersService } from '../users/users.service';
import * as bcrypt from 'bcrypt';

@Injectable()
export class AuthService {
  constructor(
    private readonly usersService: UsersService,
    private readonly jwtService: JwtService,
  ) {}

  private async validateUser(email: string, password: string) {
    const user = await this.usersService.findByEmail(email); // includes password hash
    // SECURITY: identical error + always run bcrypt.compare to avoid leaking which
    // part failed and to resist timing attacks (user-enumeration / timing oracle).
    const ok = user && (await bcrypt.compare(password, user.password));
    if (!ok) throw new UnauthorizedException('Invalid credentials');
    return user;
  }

  async login(email: string, password: string) {
    const user = await this.validateUser(email, password);
    const payload: JwtPayload = { sub: user.id, email: user.email, role: user.role };
    return {
      accessToken: this.jwtService.sign(payload), // signing options come from JwtModule config
      user: { id: user.id, email: user.email, name: user.name, role: user.role }, // no hash
    };
  }
}
```

### Wiring the auth module

```typescript
// auth/auth.module.ts
import { Module } from '@nestjs/common';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { JwtStrategy } from './strategies/jwt.strategy';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [
    UsersModule,                                       // for UsersService (must be exported there)
    PassportModule.register({ defaultStrategy: 'jwt' }),
    JwtModule.registerAsync({                          // async: secret comes from config
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        secret: config.getOrThrow<string>('jwt.secret'),
        signOptions: { expiresIn: config.get('jwt.expiresIn', '15m') }, // short-lived access token
      }),
    }),
  ],
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy],
  exports: [AuthService, JwtModule],
})
export class AuthModule {}
```

### The auth controller

```typescript
// auth/auth.controller.ts
import { Controller, Post, Body, HttpCode, HttpStatus } from '@nestjs/common';
import { IsEmail, IsString } from 'class-validator';
import { AuthService } from './auth.service';
import { Public } from '../common/decorators/public.decorator';

class LoginDto {
  @IsEmail() email: string;
  @IsString() password: string;
}

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Public()                 // login itself must be reachable without a token (see global guard)
  @Post('login')
  @HttpCode(HttpStatus.OK)  // POST defaults to 201; login should be 200
  login(@Body() dto: LoginDto) {
    return this.authService.login(dto.email, dto.password);
  }
}
```

### The JWT auth guard + a `@Public()` escape hatch

A common pattern is **secure by default**: register `JwtAuthGuard` globally so *every* route requires a valid token, then opt specific routes out with a `@Public()` decorator (login, health checks, public reads). The guard reads the `@Public()` metadata and skips verification for those.

```typescript
// common/decorators/public.decorator.ts
import { SetMetadata } from '@nestjs/common';
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

```typescript
// auth/guards/jwt-auth.guard.ts
import { Injectable, ExecutionContext } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { Reflector } from '@nestjs/core';
import { IS_PUBLIC_KEY } from '../../common/decorators/public.decorator';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') { // extends Passport's jwt guard
  constructor(private readonly reflector: Reflector) { super(); }

  canActivate(context: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;        // @Public() routes bypass auth
    return super.canActivate(context); // otherwise run the JWT strategy
  }
}
```

```typescript
// app.module.ts providers — secure-by-default registration:
//   { provide: APP_GUARD, useClass: JwtAuthGuard }   // every route requires a token...
//   { provide: APP_GUARD, useClass: RolesGuard }     // ...and roles are enforced where declared
```

### Reading the current user — a custom param decorator

```typescript
// common/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

// createParamDecorator builds a reusable @CurrentUser() that reads req.user
// (set by JwtStrategy.validate). Cleaner than @Req() in every handler.
export const CurrentUser = createParamDecorator(
  (data: keyof Express.User | undefined, ctx: ExecutionContext) => {
    const user = ctx.switchToHttp().getRequest().user;
    return data ? user?.[data] : user; // @CurrentUser('id') or @CurrentUser()
  },
);

// Usage:
//   @Get('me') getProfile(@CurrentUser() user: User) { return user; }
```

> **Security recommendations (auth):**
> - **Hash passwords with bcrypt (cost ≥ 12) or argon2.** Never store or log plaintext. Never return the hash in any response.
> - **Use short-lived access tokens (5–15 min) + a separate refresh token.** Store refresh tokens server-side (hashed) so they can be revoked; rotate them on use.
> - **Keep `JWT_SECRET` long (≥ 32 bytes), random, and out of source control.** Rotate it if leaked. Consider RS256 (asymmetric) so verifiers don't hold the signing key.
> - **Never set `ignoreExpiration: true`.** Always verify expiry.
> - **Make errors uniform** ("Invalid credentials") to prevent user-enumeration; always run `bcrypt.compare` even when the user doesn't exist to flatten timing.
> - **Rate-limit the login endpoint** (`@nestjs/throttler`, §19) to blunt brute-force attacks.
> - For a richer, audited auth stack (sessions, OAuth/social, MFA, account linking) consider **Better Auth** — see `BETTERAUTH_GUIDE.md`.

---

## 13. OpenAPI / Swagger Docs

### What it is and why

**OpenAPI** (formerly Swagger) is a standard for describing REST APIs. `@nestjs/swagger` **generates that description automatically from your decorators** — your controllers, DTOs, and validation metadata become interactive documentation (a "Swagger UI" page) where developers can read every endpoint and even execute requests. The logic: your code is already the source of truth, so derive the docs from it rather than maintaining a separate, always-stale document. This is one of NestJS's highest-leverage features for any API consumed by a frontend or third party.

```bash
npm install @nestjs/swagger
```

```typescript
// main.ts — build the document and mount the UI
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';

const config = new DocumentBuilder()
  .setTitle('My API')
  .setDescription('REST API for the my-api product')
  .setVersion('1.0')
  .addBearerAuth()                  // adds the "Authorize" button for JWT
  .addTag('users', 'User management')
  .addTag('auth', 'Authentication')
  .build();

const document = SwaggerModule.createDocument(app, config);
SwaggerModule.setup('api/docs', app, document, {
  swaggerOptions: { persistAuthorization: true }, // remember the JWT across page reloads
});
// → interactive docs served at http://localhost:3000/api/docs
```

### Decorating DTOs and controllers

```typescript
// dto/create-user.dto.ts
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export class CreateUserDto {
  @ApiProperty({ example: 'Alice Johnson', description: 'Full name' })
  @IsString() name: string;

  @ApiProperty({ example: 'alice@example.com' })
  @IsEmail() email: string;

  @ApiProperty({ example: 'Str0ng!Pass', minLength: 8, writeOnly: true })
  @IsString() @MinLength(8) password: string;

  @ApiPropertyOptional({ enum: UserRole, default: UserRole.USER })
  @IsEnum(UserRole) @IsOptional() role?: UserRole;
}
```

```typescript
// users.controller.ts — describe operations & responses for the UI
import {
  ApiTags, ApiOperation, ApiResponse, ApiBearerAuth, ApiParam, ApiQuery,
} from '@nestjs/swagger';

@ApiTags('users')      // groups these endpoints under "users" in the UI
@ApiBearerAuth()       // marks the whole controller as requiring a JWT
@Controller('users')
export class UsersController {
  @ApiOperation({ summary: 'List users', description: 'Paginated list' })
  @ApiQuery({ name: 'page', required: false, type: Number })
  @ApiResponse({ status: 200, description: 'List of users' })
  @Get()
  findAll() { /* ... */ }

  @ApiOperation({ summary: 'Get user by ID' })
  @ApiParam({ name: 'id', type: String, description: 'User UUID' })
  @ApiResponse({ status: 200, description: 'Found' })
  @ApiResponse({ status: 404, description: 'Not found' })
  @Get(':id')
  findOne(@Param('id') id: string) { /* ... */ }
}
```

> **⚡ Version note:** With `@nestjs/swagger` v8+ (NestJS 11) the **CLI plugin** can infer most `@ApiProperty()` data (types, required-ness, descriptions from JSDoc) automatically, so you write far fewer decorators. Enable it in `nest-cli.json`:
> ```json
> { "compilerOptions": { "plugins": ["@nestjs/swagger"] } }
> ```

> **Security note:** Swagger UI exposes your entire API surface. **Disable or password-protect it in production**, or at least gate it behind auth, so you don't hand attackers a map of every endpoint and payload shape. A common pattern: only call `SwaggerModule.setup()` when `NODE_ENV !== 'production'`.

---

## 14. WebSockets & Gateways

### What a gateway is and when to use WebSockets

HTTP is request/response: the client asks, the server answers, the connection closes. For **real-time, bidirectional** features — chat, live notifications, presence, collaborative editing, dashboards — you want a persistent connection the server can push to at any time. That's **WebSockets**. In NestJS, a **gateway** is the WebSocket equivalent of a controller: a class decorated with `@WebSocketGateway()` whose methods handle incoming socket *events* (rather than HTTP routes). Crucially, **gateways reuse the same DI, guards, pipes, and interceptors as HTTP** — your services and auth logic work unchanged over sockets.

```bash
npm install @nestjs/websockets @nestjs/platform-socket.io socket.io
```

### A chat gateway

```typescript
// events/events.gateway.ts
import {
  WebSocketGateway, WebSocketServer, SubscribeMessage,
  MessageBody, ConnectedSocket,
  OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';
import { Logger } from '@nestjs/common';

@WebSocketGateway({
  namespace: '/chat',          // logical channel; default is '/'
  cors: { origin: process.env.FRONTEND_URL ?? 'http://localhost:3000' }, // lock down in prod!
})
export class EventsGateway
  implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect
{
  @WebSocketServer() server: Server; // the socket.io Server, for server→client pushes
  private readonly logger = new Logger(EventsGateway.name);

  afterInit() { this.logger.log('WebSocket gateway initialized'); }

  handleConnection(client: Socket) {
    // Authenticate here: read client.handshake.auth.token, verify the JWT, or
    // disconnect. Connections are otherwise UNAUTHENTICATED by default.
    this.logger.log(`Client connected: ${client.id}`);
  }

  handleDisconnect(client: Socket) {
    this.logger.log(`Client disconnected: ${client.id}`);
  }

  // Handle an event named 'message' sent by a client.
  @SubscribeMessage('message')
  handleMessage(
    @MessageBody() data: { room: string; text: string },
    @ConnectedSocket() client: Socket,
  ) {
    // Broadcast to everyone in the room EXCEPT the sender.
    client.to(data.room).emit('message', {
      text: data.text,
      sender: client.id,
      timestamp: new Date().toISOString(),
    });
    return { event: 'message', data: 'delivered' }; // returned value = ack to sender
  }

  @SubscribeMessage('joinRoom')
  handleJoin(@MessageBody() room: string, @ConnectedSocket() client: Socket) {
    client.join(room);
    client.to(room).emit('userJoined', { userId: client.id });
    return { joined: room };
  }

  // Call these from any service to push from server → clients.
  broadcastToAll(event: string, data: unknown) { this.server.emit(event, data); }
  emitToRoom(room: string, event: string, data: unknown) {
    this.server.to(room).emit(event, data);
  }
}
```

```typescript
// events/events.module.ts — a gateway is a provider
@Module({ providers: [EventsGateway], exports: [EventsGateway] })
export class EventsModule {}
```

```javascript
// client (browser or Node) — socket.io-client
import { io } from 'socket.io-client';
const socket = io('http://localhost:3000/chat', { auth: { token: jwt } });
socket.on('connect', () => socket.emit('joinRoom', 'general'));
socket.emit('message', { room: 'general', text: 'Hello!' });
socket.on('message', (msg) => console.log('received', msg));
```

> **Security recommendations (WebSockets):**
> - **Authenticate the handshake.** Verify a JWT in `handleConnection` (from `client.handshake.auth.token`) and disconnect unauthenticated sockets — sockets are *not* protected by your HTTP global guard automatically. NestJS supports **WS guards** (`@UseGuards` on a gateway) that work like HTTP guards.
> - **Lock down CORS `origin`** to your real frontend domain(s); never ship `origin: '*'` to production.
> - **Validate `@MessageBody()`** with DTOs + a `ValidationPipe` just like HTTP bodies — socket payloads are equally untrusted.
> - **Rate-limit messages** per socket to prevent flooding.
> - For raw WebSocket internals (the protocol, ping/pong, scaling with a Redis adapter), the Go equivalent `GO_GORILLA_WEBSOCKETS_GUIDE.md` explains the underlying mechanics well.

---

## 15. File Uploads

### How uploads work in NestJS

File uploads arrive as `multipart/form-data`. On Express, NestJS uses **Multer** (bundled with `@nestjs/platform-express`) via **file interceptors** (`FileInterceptor`, `FilesInterceptor`) that parse the multipart body and expose the file(s) through `@UploadedFile()` / `@UploadedFiles()`. You choose **disk storage** (write straight to a folder — for keeping files) or **memory storage** (a `Buffer` in RAM — for processing then discarding/forwarding). Uploads are a notorious security surface, so validation (type, size) is mandatory.

```bash
# Multer ships with @nestjs/platform-express; you only need its types:
npm install --save-dev @types/multer
```

```typescript
// upload/upload.controller.ts
import {
  Controller, Post, Get, Param, Res,
  UploadedFile, UploadedFiles, UseInterceptors, BadRequestException,
} from '@nestjs/common';
import { FileInterceptor, FilesInterceptor } from '@nestjs/platform-express';
import { diskStorage } from 'multer';
import { extname, join } from 'path';
import { Response } from 'express';

// Generate a collision-proof filename (never trust the client's originalname for the path).
const editFileName = (_req: any, file: Express.Multer.File, cb: Function) => {
  const unique = `${Date.now()}-${Math.round(Math.random() * 1e9)}`;
  cb(null, `${file.fieldname}-${unique}${extname(file.originalname)}`);
};

// Reject anything that isn't an allowed image type (defense against malicious uploads).
const imageFilter = (_req: any, file: Express.Multer.File, cb: Function) => {
  if (!/^image\/(jpe?g|png|gif|webp)$/.test(file.mimetype)) {
    return cb(new BadRequestException('Only image files are allowed'), false);
  }
  cb(null, true);
};

@Controller('upload')
export class UploadController {
  @Post('avatar')
  @UseInterceptors(
    FileInterceptor('avatar', {                    // field name = 'avatar'
      storage: diskStorage({ destination: './uploads/avatars', filename: editFileName }),
      fileFilter: imageFilter,
      limits: { fileSize: 5 * 1024 * 1024 },       // 5 MB cap (DoS protection)
    }),
  )
  uploadAvatar(@UploadedFile() file: Express.Multer.File) {
    if (!file) throw new BadRequestException('File is required');
    return { filename: file.filename, size: file.size, url: `/upload/files/${file.filename}` };
  }

  @Post('photos')
  @UseInterceptors(FilesInterceptor('photos', 10, { limits: { fileSize: 5 * 1024 * 1024 } }))
  uploadMany(@UploadedFiles() files: Express.Multer.File[]) {
    return files.map((f) => ({ filename: f.filename, size: f.size }));
  }

  @Get('files/:filename')
  serve(@Param('filename') filename: string, @Res() res: Response) {
    // Serve from a fixed directory; the filename was sanitized on upload.
    res.sendFile(join(process.cwd(), 'uploads/avatars', filename));
  }
}
```

```typescript
// In-memory storage — for processing (parse CSV, resize image) without saving to disk.
import { memoryStorage } from 'multer';

@Post('import')
@UseInterceptors(FileInterceptor('file', { storage: memoryStorage() }))
import(@UploadedFile() file: Express.Multer.File) {
  const text = file.buffer.toString('utf-8'); // raw bytes are in file.buffer
  // ...parse and process, then forward to S3 etc.
}
```

```typescript
// Serve a whole folder as static assets (e.g. public uploads):
import { ServeStaticModule } from '@nestjs/serve-static';
import { join } from 'path';
// app.module.ts imports:
ServeStaticModule.forRoot({
  rootPath: join(__dirname, '..', 'uploads'),
  serveRoot: '/uploads', // → http://localhost:3000/uploads/<file>
});
```

> **Security recommendations (uploads):**
> - **Always set a `fileSize` limit** — unbounded uploads are a trivial DoS.
> - **Validate type by MIME *and* extension**, and ideally sniff magic bytes; never trust the client-provided `originalname` or content-type alone.
> - **Generate your own filenames** (as above) to prevent path traversal (`../../etc/passwd`) and overwrites.
> - **Store uploads outside the web root** or on object storage (S3); never execute uploaded files; serve them with `Content-Disposition: attachment` and a restrictive `Content-Type` so a malicious HTML/SVG can't run in your origin.
> - For production, prefer streaming straight to **object storage** rather than local disk.
> - ⚡ On **Fastify**, use `@fastify/multipart` instead of Multer; see `FASTIFY_GUIDE.md`.

---

## 16. Caching, Task Scheduling & Queues

### Caching — trade freshness for speed

**What/why:** Caching stores the result of an expensive operation (a DB query, an external API call) so subsequent identical requests return instantly. The trade-off is **freshness vs latency/cost**: cached data can be stale until it expires (TTL) or is invalidated. Cache **read-heavy, slow-changing** data; invalidate on writes. NestJS's `@nestjs/cache-manager` gives both an automatic interceptor (cache whole GET responses) and a manual `Cache` API.

```bash
npm install @nestjs/cache-manager cache-manager
# For a shared/distributed cache across instances, add a Redis store (see REDIS_GUIDE.md):
npm install cache-manager-redis-store ioredis
```

```typescript
// app.module.ts
import { CacheModule } from '@nestjs/cache-manager';

@Module({
  imports: [
    CacheModule.register({
      isGlobal: true,
      ttl: 60_000,  // ⚡ NestJS 11 / cache-manager v5: TTL is in MILLISECONDS
      max: 100,     // max items in the in-memory store
    }),
    // Distributed (multi-instance) cache → use Redis so all instances share it:
    // CacheModule.registerAsync({ useFactory: () => ({ store: redisStore, host, port, ttl }) }),
  ],
})
export class AppModule {}
```

```typescript
// Auto-cache a GET response with the interceptor:
import { CacheInterceptor, CacheKey, CacheTTL } from '@nestjs/cache-manager';
import { UseInterceptors } from '@nestjs/common';

@UseInterceptors(CacheInterceptor)
@CacheKey('all-users')
@CacheTTL(30_000) // override TTL for this route (ms)
@Get()
findAll() { return this.usersService.findAll(); }
```

```typescript
// Manual cache control in a service (cache-aside pattern):
import { Inject, Injectable } from '@nestjs/common';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

@Injectable()
export class ProductsService {
  constructor(@Inject(CACHE_MANAGER) private readonly cache: Cache) {}

  async findAll() {
    const cached = await this.cache.get<Product[]>('products');
    if (cached) return cached;                        // cache hit
    const products = await this.db.product.findMany(); // cache miss → load
    await this.cache.set('products', products, 60_000); // populate (ms TTL)
    return products;
  }

  async invalidate() { await this.cache.del('products'); } // call on writes
}
```

> **⚡ Version note:** In cache-manager v5 (used by NestJS 11) **TTL is milliseconds**, not seconds — a frequent upgrade footgun. See `REDIS_GUIDE.md` for using Redis as the backing store and for cache-invalidation strategies.

### Task scheduling — run code on a timer

`@nestjs/schedule` lets you run methods on a **cron schedule**, at a fixed **interval**, or once after a **timeout** — declaratively, with decorators. Use it for cleanups, periodic syncs, report generation. **Caution in multi-instance deploys:** every instance runs the same cron, so a "daily cleanup" runs N times. Guard such jobs with a distributed lock (e.g. a Redis lock) or run the scheduler in a single dedicated worker.

```bash
npm install @nestjs/schedule
```

```typescript
import { ScheduleModule } from '@nestjs/schedule';
// app.module.ts imports: [ ScheduleModule.forRoot() ]

import { Injectable, Logger } from '@nestjs/common';
import { Cron, Interval, Timeout, CronExpression } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  @Cron(CronExpression.EVERY_DAY_AT_MIDNIGHT)
  cleanupSessions() { this.logger.log('Cleaning expired sessions...'); }

  @Cron('0 9 * * 1') // standard cron: every Monday at 09:00
  weeklyReport() { this.logger.log('Sending weekly report...'); }

  @Interval(30_000)  // every 30 seconds
  poll() { this.logger.log('Polling...'); }

  @Timeout(5_000)    // once, 5s after startup
  warmup() { this.logger.log('Warmup task ran'); }
}
```

### Queues with BullMQ — offload heavy/async work

**What/why:** Some work shouldn't block the HTTP response — sending email, processing images, generating reports, calling slow third parties. A **queue** lets you enqueue a *job* quickly (returning to the client immediately) while a separate **worker** processes it in the background, with **retries, backoff, and persistence** (jobs survive restarts because they live in Redis). `@nestjs/bullmq` wraps BullMQ.

```bash
npm install @nestjs/bullmq bullmq ioredis
```

```typescript
// app.module.ts
import { BullModule } from '@nestjs/bullmq';

@Module({
  imports: [
    BullModule.forRoot({ connection: { host: 'localhost', port: 6379 } }),
    BullModule.registerQueue({ name: 'emails' }),
  ],
})
export class AppModule {}
```

```typescript
// email/email.producer.ts — enqueue a job (returns instantly)
import { Injectable } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';

@Injectable()
export class EmailProducer {
  constructor(@InjectQueue('emails') private readonly queue: Queue) {}

  async sendWelcome(userId: string, email: string) {
    await this.queue.add('welcome', { userId, email }, {
      attempts: 3,                                  // retry up to 3 times on failure
      backoff: { type: 'exponential', delay: 1000 }, // 1s, 2s, 4s between retries
      removeOnComplete: true,                       // tidy succeeded jobs
      removeOnFail: 100,                            // keep last 100 failures for inspection
    });
  }
}
```

```typescript
// email/email.consumer.ts — process jobs in the background
import { Processor, WorkerHost } from '@nestjs/bullmq';
import { Job } from 'bullmq';

@Processor('emails')
export class EmailConsumer extends WorkerHost {
  async process(job: Job) {
    switch (job.name) {
      case 'welcome': {
        const { email } = job.data;
        // ...send via nodemailer / SendGrid; throwing here triggers a retry
        return { sent: email };
      }
    }
  }
}
```

> See `REDIS_GUIDE.md` for the Redis configuration BullMQ depends on (persistence, eviction policy — note BullMQ requires `maxmemory-policy noeviction`).

---

## 17. Microservices Overview

### What "microservices" means in NestJS

NestJS's **microservices** support is a uniform abstraction over many **transport** layers (TCP, Redis pub/sub, NATS, RabbitMQ, Kafka, gRPC, MQTT). Instead of HTTP routes, a microservice handles **message patterns** (request/response) and **event patterns** (fire-and-forget). The beauty is that **the same controllers/providers/DI/pipes/guards** you know from HTTP apply — only the entry-point and the decorators change. This lets you split a monolith into services, or build event-driven systems, without learning a new programming model.

Two interaction styles, both first-class:

- **Message pattern (`@MessagePattern`)** — request/response. A client `send()`s and awaits a reply (returns an RxJS `Observable`). Like an RPC.
- **Event pattern (`@EventPattern`)** — fire-and-forget. A client `emit()`s; no reply. For broadcasting "something happened" to any interested service.

```bash
npm install @nestjs/microservices
# transport drivers as needed:
npm install ioredis            # Redis
npm install amqplib amqp-connection-manager   # RabbitMQ
npm install kafkajs            # Kafka
```

```typescript
// A microservice app's bootstrap (no HTTP server — it listens on a transport):
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
    transport: Transport.REDIS,
    options: { host: 'localhost', port: 6379 },
  });
  await app.listen();
}
bootstrap();
```

```typescript
// A microservice "controller" — handlers respond to patterns, not routes.
import { Controller } from '@nestjs/common';
import { MessagePattern, EventPattern, Payload, Ctx, RedisContext } from '@nestjs/microservices';

@Controller()
export class OrdersController {
  @MessagePattern({ cmd: 'get_orders' })           // request/response
  getOrders(@Payload() data: { userId: string }) {
    return this.ordersService.findByUser(data.userId); // value is sent back to caller
  }

  @EventPattern('order_created')                    // fire-and-forget
  onOrderCreated(@Payload() data: unknown, @Ctx() ctx: RedisContext) {
    // react to the event; no return value is sent back
  }
}
```

```typescript
// A client (often an HTTP gateway service) talking to the microservice:
import { ClientsModule, Transport, ClientProxy } from '@nestjs/microservices';
import { Inject } from '@nestjs/common';

// app.module.ts imports:
ClientsModule.register([
  { name: 'ORDERS', transport: Transport.REDIS, options: { host: 'localhost', port: 6379 } },
]);

// in a service:
//   constructor(@Inject('ORDERS') private readonly client: ClientProxy) {}
//   getOrders(userId: string) { return this.client.send({ cmd: 'get_orders' }, { userId }); }
//   notify(order: any)        { this.client.emit('order_created', order); }
```

| Transport | Best for |
|-----------|----------|
| `TCP` | Simple internal RPC on a trusted network |
| `Redis` | Lightweight pub/sub & simple queuing |
| `NATS` | High-throughput, low-latency messaging |
| `RabbitMQ` | Reliable delivery, acks, enterprise messaging |
| `Kafka` | High-volume event streaming, replayable logs |
| `gRPC` | Strongly-typed, high-performance RPC across languages |
| `MQTT` | IoT / lightweight devices |

> **Hybrid apps:** A single Nest process can serve **both** HTTP and a microservice transport (`app.connectMicroservice(...)` + `app.startAllMicroservices()`), common for an API that also consumes events. For raw gRPC/RPC mechanics across languages, `GO_GRPC_RPC_GUIDE.md` is a useful companion.

---

## 18. Testing — Unit & E2E

### The testing philosophy DI enables

This is where the dependency-injection investment pays off. Because services receive their dependencies rather than constructing them, you can build a **`TestingModule`** that swaps real dependencies for **mocks** with one line. Two complementary levels:

- **Unit tests** — test one provider/controller in isolation, with all its dependencies mocked. Fast, deterministic, no DB/network. This is where most of your tests should live.
- **End-to-end (e2e) tests** — boot the *real* application (or most of it) and fire real HTTP requests with **Supertest**, asserting on responses. Fewer, slower, but they verify the wiring (guards, pipes, routing) actually works together.

NestJS pre-configures **Jest**; `nest g resource` generates `.spec.ts` stubs.

### Unit testing a service (mocking the repository)

```typescript
// users/users.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { NotFoundException } from '@nestjs/common';
import { UsersService } from './users.service';
import { User } from './entities/user.entity';

// A factory that produces a fully-mocked repository (all methods are jest.fn()).
const mockRepository = () => ({
  find: jest.fn(), findOne: jest.fn(), create: jest.fn(),
  save: jest.fn(), update: jest.fn(), delete: jest.fn(),
});
type MockRepo<T = any> = Partial<Record<keyof Repository<T>, jest.Mock>>;

describe('UsersService', () => {
  let service: UsersService;
  let repo: MockRepo<User>;

  beforeEach(async () => {
    // Build a tiny DI container with the real service but a FAKE repository.
    const moduleRef: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        { provide: getRepositoryToken(User), useFactory: mockRepository },
      ],
    }).compile();

    service = moduleRef.get(UsersService);
    repo = moduleRef.get(getRepositoryToken(User));
  });

  it('returns a user when found', async () => {
    const user = { id: '1', name: 'Alice' };
    repo.findOne!.mockResolvedValue(user);
    await expect(service.findOne('1')).resolves.toEqual(user);
    expect(repo.findOne).toHaveBeenCalledWith({ where: { id: '1' } });
  });

  it('throws NotFoundException when missing', async () => {
    repo.findOne!.mockResolvedValue(null);
    await expect(service.findOne('999')).rejects.toThrow(NotFoundException);
  });
});
```

### Unit testing a controller (mocking the service)

```typescript
// users/users.controller.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

const mockUsersService = { findAll: jest.fn(), findOne: jest.fn(), create: jest.fn() };

describe('UsersController', () => {
  let controller: UsersController;

  beforeEach(async () => {
    const moduleRef: TestingModule = await Test.createTestingModule({
      controllers: [UsersController],
      providers: [{ provide: UsersService, useValue: mockUsersService }],
    }).compile();
    controller = moduleRef.get(UsersController);
  });

  it('delegates findAll to the service', async () => {
    const users = [{ id: '1', name: 'Alice' }];
    mockUsersService.findAll.mockResolvedValue(users);
    await expect(controller.findAll()).resolves.toEqual(users);
    expect(mockUsersService.findAll).toHaveBeenCalled();
  });
});
```

### E2E testing with Supertest

```typescript
// test/users.e2e-spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('Users (e2e)', () => {
  let app: INestApplication;
  let token: string;

  beforeAll(async () => {
    const moduleRef: TestingModule = await Test.createTestingModule({
      imports: [AppModule], // boots the REAL app — point it at a TEST database!
    }).compile();

    app = moduleRef.createNestApplication();
    // Replicate global config from main.ts so behavior matches production.
    app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
    await app.init();

    const res = await request(app.getHttpServer())
      .post('/auth/login')
      .send({ email: 'admin@test.com', password: 'Test1234!' });
    token = res.body.accessToken;
  });

  afterAll(async () => { await app.close(); });

  it('GET /users → 401 without a token', () =>
    request(app.getHttpServer()).get('/users').expect(401));

  it('GET /users → 200 with a token', () =>
    request(app.getHttpServer())
      .get('/users')
      .set('Authorization', `Bearer ${token}`)
      .expect(200));

  it('POST /users → 201 and never echoes the password', () =>
    request(app.getHttpServer())
      .post('/users')
      .set('Authorization', `Bearer ${token}`)
      .send({ name: 'Test', email: 'test@test.com', password: 'Pass1234!' })
      .expect(201)
      .expect((r) => {
        expect(r.body).toHaveProperty('id');
        expect(r.body).not.toHaveProperty('password'); // security regression guard
      }));

  it('POST /users → 400 for an invalid email', () =>
    request(app.getHttpServer())
      .post('/users')
      .set('Authorization', `Bearer ${token}`)
      .send({ name: 'X', email: 'nope', password: 'Pass1234!' })
      .expect(400));
});
```

```json
// package.json scripts
{ "test": "jest", "test:watch": "jest --watch",
  "test:cov": "jest --coverage", "test:e2e": "jest --config ./test/jest-e2e.json" }
```

> **Best practices:** Override only the providers you must (use `.overrideProvider(X).useValue(mock)` for surgical swaps in e2e). Run e2e against a **disposable test database** (a fresh schema or a throwaway Docker Postgres), and reset state between tests. Add a test that asserts the password is never returned — security regressions should fail the build.

---

## 19. Security, Best Practices & Production Hardening

Security is not a single feature; it's a set of defaults you apply across the stack. Below is a consolidated, prioritized checklist with the *why* for each — most of these have appeared inline, gathered here for production review.

### Input & data security
- **Validate everything** with DTOs + a global `ValidationPipe` set to `whitelist: true, forbidNonWhitelisted: true` (§7). This neutralizes mass-assignment and malformed input — your single highest-value control.
- **Use parse pipes** (`ParseUUIDPipe`, `ParseIntPipe`) on path/query params so handlers never see unvalidated raw strings.
- **Never expose secrets in responses or logs** — exclude password hashes via `select`/`@Exclude()` + `ClassSerializerInterceptor`; sanitize error messages so DB internals don't leak (§9).
- **Parameterize all queries.** Prisma/TypeORM do this for you; if you drop to raw SQL, never string-concatenate user input (SQL injection). See `POSTGRESQL_GUIDE.md`.

### AuthN / AuthZ
- **bcrypt (cost ≥ 12) or argon2** for passwords; short-lived JWTs + revocable refresh tokens; long random `JWT_SECRET` out of source control; secure-by-default global `JwtAuthGuard` with a `@Public()` opt-out; uniform credential errors; **rate-limit login** (§12).

### Transport & headers
- **Helmet** sets safe security headers (CSP, HSTS, X-Frame-Options, etc.).
- **CORS** must be explicit — never `origin: '*'` with credentials in production.
- **Rate limiting** with `@nestjs/throttler` to blunt brute-force and DoS.

```bash
npm install helmet @nestjs/throttler
```

```typescript
// main.ts
import helmet from 'helmet';
app.use(helmet());                 // secure HTTP headers
app.enableCors({
  origin: process.env.FRONTEND_URL ?? 'http://localhost:3000', // explicit allow-list
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  credentials: true,
});
```

```typescript
// app.module.ts — global rate limiting
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler';
import { APP_GUARD } from '@nestjs/core';

@Module({
  imports: [ThrottlerModule.forRoot([{ ttl: 60_000, limit: 100 }])], // 100 req/min/IP
  providers: [{ provide: APP_GUARD, useClass: ThrottlerGuard }],
})
export class AppModule {}
```

> **⚡ Fastify note:** With the Fastify adapter use `@fastify/helmet` / `@fastify/rate-limit` (registered as Fastify plugins) instead of the Express middleware. See `FASTIFY_GUIDE.md`.

### Serialization — never leak fields
```typescript
// On the entity/DTO:
import { Exclude } from 'class-transformer';
class UserEntity { @Exclude() password: string; /* ...other fields... */ }

// Globally enable, so @Exclude() is honored on every response:
//   providers: [{ provide: APP_INTERCEPTOR, useClass: ClassSerializerInterceptor }]
```

### Production config
- **Disable `synchronize`** (TypeORM); run **migrations** instead.
- **Disable / protect Swagger** in production (§13).
- **`disableErrorMessages: true`** on `ValidationPipe` in prod to avoid schema leakage.
- **`app.enableShutdownHooks()`** for graceful shutdown (close DB pools, drain queues).
- **Run behind a reverse proxy / TLS**; set `app.set('trust proxy', 1)` (Express) so client IPs and rate limiting work correctly behind a load balancer.

### Performance (production)
- **Prefer Fastify** for raw throughput when no Express-only middleware blocks you (§2, `FASTIFY_GUIDE.md`).
- **Paginate** every list endpoint — never return an unbounded result set.
- **`select` only needed columns** (Prisma/TypeORM) — avoid `SELECT *` on wide tables.
- **Cache** read-heavy endpoints; **enable compression** (Express: `compression()`).
- **Avoid request-scoped providers** unless necessary (§6).

```typescript
import * as compression from 'compression';
app.use(compression()); // gzip responses (Express)
```

---

## 20. Tips, Tricks & Gotchas

### Architecture & design
- **Keep controllers thin, services fat.** Controllers do HTTP; services do logic. This is the rule everything else depends on.
- **Never import `Request`/`Response` into a service.** Services must be transport-agnostic so they're reusable from gateways, queues, and microservices — and unit-testable.
- **One module per feature, not per layer.** `UsersModule` ✅; a global `ServicesModule` ❌.
- **Be conservative with `exports`.** A module's exports are its public API; the smaller, the better.

### Circular dependencies
When module A imports B and B imports A (or service A injects B and vice-versa), use `forwardRef` on both sides. It's a smell — prefer extracting shared logic into a third module — but here's the escape hatch:
```typescript
// users.module.ts:  imports: [forwardRef(() => AuthModule)]
// auth.module.ts:   imports: [forwardRef(() => UsersModule)]
// in a service:
constructor(@Inject(forwardRef(() => AuthService)) private readonly auth: AuthService) {}
```

### The classic footguns
- **`@Res()` without `{ passthrough: true }`** disables interceptors & auto-serialization (§5). Almost always add `passthrough: true`.
- **`ValidationPipe` without `transform: true`** leaves DTOs as plain objects — class methods and `@Type()` conversions won't run.
- **`whitelist: true` silently strips unknown fields.** Add `forbidNonWhitelisted: true` if you want a 400 instead of silent stripping.
- **Globals registered in `main.ts` can't use DI** — they're built before the container. To inject into a global pipe/guard/interceptor/filter, register it as a provider with the `APP_PIPE` / `APP_GUARD` / `APP_INTERCEPTOR` / `APP_FILTER` token instead:
  ```typescript
  // app.module.ts providers:
  { provide: APP_PIPE, useClass: ValidationPipe }
  ```
- **`synchronize: true` (TypeORM) destroys data** outside dev — use migrations (§11).
- **Request scope is a performance trap** — it propagates upward (§6).
- **cache-manager v5 TTLs are milliseconds**, not seconds (§16).
- **NestJS 11 wildcard routes** use named params (`*path`), not bare `*` (§5).
- **Mixing Promises and Observables carelessly** causes subtle bugs. If a service method is async, make the controller method async too.

### Don't leak the password
```typescript
import { Exclude } from 'class-transformer';
class User { @Exclude() password: string; } // honored by ClassSerializerInterceptor
// providers: [{ provide: APP_INTERCEPTOR, useClass: ClassSerializerInterceptor }]
```

### Development workflow
- **Hot reload:** `npm run start:dev` (watch mode).
- **Debug (VS Code)** — `npm run start:debug`, then attach:
  ```json
  { "type": "node", "request": "attach", "name": "Attach NestJS",
    "port": 9229, "restart": true, "sourceMaps": true }
  ```
- **Preview generators:** `nest g resource products --dry-run`.
- **Skip tests while scaffolding:** add `--no-spec`.

### Quick reference: install map
```bash
# Core
npm install -g @nestjs/cli && nest new my-api

# Validation
npm install class-validator class-transformer @nestjs/mapped-types

# Config
npm install @nestjs/config joi

# Prisma (recommended) / TypeORM / Mongoose
npm install prisma @prisma/client && npx prisma init
npm install @nestjs/typeorm typeorm pg
npm install @nestjs/mongoose mongoose

# Auth
npm install @nestjs/passport passport passport-jwt @nestjs/jwt bcrypt
npm install --save-dev @types/passport-jwt @types/bcrypt

# Swagger / WebSockets / Uploads
npm install @nestjs/swagger
npm install @nestjs/websockets @nestjs/platform-socket.io socket.io
npm install --save-dev @types/multer

# Caching / Scheduling / Queues
npm install @nestjs/cache-manager cache-manager
npm install @nestjs/schedule
npm install @nestjs/bullmq bullmq ioredis

# Microservices
npm install @nestjs/microservices

# Security & performance
npm install helmet @nestjs/throttler compression

# Fastify adapter (optional)
npm install @nestjs/platform-fastify fastify
```

---

## 21. Study Path & Build-to-Learn Projects

Learning happens by building. Follow this order; each phase ends in a concrete project.

### Phase 1 — Foundations (Week 1) **[B]**
1. Install Node 22+ and the Nest CLI. Run `nest new`, read `main.ts` and `app.module.ts` until the bootstrap flow is clear.
2. **Build:** a CRUD REST API for a `Tasks` resource using an in-memory array (no DB yet). Practice every verb, `@Param`, `@Query`, `@Body`.
3. Add a global `ValidationPipe` + DTOs with `class-validator`. Test with curl/Postman; deliberately send bad input and watch the 400s.

### Phase 2 — Database (Week 2) **[I]**
4. Run PostgreSQL locally (`docker run -e POSTGRES_PASSWORD=pass -p 5432:5432 postgres`).
5. Add **Prisma** to the Tasks app; replace the array with a real table. Practice relations (a `User` has many `Tasks`), `include`, and a transaction. (See `PRISMA_ORM_GUIDE.md`, `POSTGRESQL_GUIDE.md`.)
6. **Optional:** redo with TypeORM to compare DX; pick one going forward.

### Phase 3 — Auth (Week 3) **[I/A]**
7. Add a `Users` module with registration + bcrypt-hashed passwords.
8. Implement JWT login, the `JwtStrategy`, and a `JwtAuthGuard`.
9. Add `@Public()`; register the guard globally so everything else is protected by default.
10. Add RBAC: `RolesGuard` + `@Roles()`. **Build:** a complete auth system; test register → login → token → protected route. (Compare with `BETTERAUTH_GUIDE.md`.)

### Phase 4 — The lifecycle stack (Week 4) **[I/A]**
11. Write a request-logging middleware (global).
12. Write a `TransformInterceptor` (envelope every response) and a global `AllExceptionsFilter`.
13. Write a `TimeoutInterceptor`. Configure Swagger for the whole API.

### Phase 5 — Advanced features (Weeks 5–6) **[A]**
14. Add `@nestjs/config` with Joi validation; move all secrets to `.env`.
15. Add caching to read-heavy endpoints (Redis store — `REDIS_GUIDE.md`).
16. Add a BullMQ queue ("send welcome email after registration") and task scheduling ("nightly cleanup of unverified accounts").
17. **Build:** a complete blog API — users, posts, comments, tags, auth, pagination, search, Swagger.

### Phase 6 — Real-time & files (Week 7) **[I/A]**
18. Add a WebSocket gateway for real-time notifications (authenticate the handshake).
19. Add avatar upload with type/size validation; serve uploads as static assets.

### Phase 7 — Testing & production (Week 8) **[A]**
20. Unit-test every service with Jest mocks; e2e-test the critical flows.
21. Dockerize: `Dockerfile` + `docker-compose.yml` (app + Postgres + Redis) — see `DOCKER_GUIDE.md`.
22. Harden for production: disable `synchronize`, run migrations, add Helmet + rate limiting, protect Swagger, enable shutdown hooks.
23. **Final project:** deploy a production-ready API (Railway, Render, Fly.io, or a VPS).

### Build-to-learn projects (ordered by complexity)

| # | Project | Concepts practiced |
|---|---------|--------------------|
| 1 | Task Manager API | Modules, controllers, services, DTOs, validation, Prisma/TypeORM |
| 2 | Auth System | Passport, JWT, guards, RBAC, decorators, bcrypt |
| 3 | Blog Platform | Relations (user→posts→comments), pagination, search, Swagger |
| 4 | File Storage Service | Upload, static serving, streaming responses, validation |
| 5 | Real-Time Chat API | WebSocket gateway, rooms, auth over WS |
| 6 | Job Board + Email Queue | BullMQ, nodemailer, scheduling, background jobs |
| 7 | E-Commerce Backend | Complex relations, Stripe webhooks, inventory transactions, caching |
| 8 | Microservices System | Split the blog into Auth / Posts / Notifications services over Redis transport |

---

*Official docs: https://docs.nestjs.com — comprehensive and well-written; bookmark it. All code here targets **NestJS 11 on Node 22/24 (2026)**. Cross-reference the underlying adapter (`FASTIFY_GUIDE.md`), the data layer (`PRISMA_ORM_GUIDE.md`, `POSTGRESQL_GUIDE.md`, `MONGODB_GUIDE.md`, `REDIS_GUIDE.md`), and alternative auth (`BETTERAUTH_GUIDE.md`). For fast-moving packages (BullMQ, Prisma, Passport) always confirm specifics against the package's own docs.*
