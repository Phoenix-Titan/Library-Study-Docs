# Ã°Å¸â€œÅ¡ Developer Study Library & Workspace

A self-contained, **offline-first** collection of in-depth technical guides plus a real demo project. Everything here is written to be learned and built from **without an internet connection** Ã¢â‚¬â€ each guide has concepts, runnable commented examples, reference tables, gotchas, and a study path, accurate **as of 2026**.

> **How to use this:** All guides live in [`library/`](library/). Pick a track below, open the guide, and work top-to-bottom. Each guide ends with a "Study Path" and build-to-learn projects.

---

## Ã°Å¸â€”â€šÃ¯Â¸Â The Guides (`library/`)

### Programming Languages (beginner Ã¢â€ â€™ advanced)
| Guide | What it covers | Lines |
|---|---|---|
| [Python](library/PYTHON_GUIDE.md) | 3.13/3.14: syntax, data structures, control flow & `match`, functions/closures, OOP & dataclasses, modules, exceptions, generators/context managers, decorators, type hints, **File System/OS/`subprocess`/CLIs**, concurrency (threads/multiprocessing/asyncio + GIL), stdlib, pytest, packaging | 1457 |
| [JavaScript](library/JAVASCRIPT_GUIDE.md) | ES2024/25: types & coercion, modern syntax, functions, scope/closures, `this`, objects, prototypes & classes, arrays, async (event loop/promises/await), modules, **a detailed DOM & browser-APIs section**, FS/OS via a runtime, metaprogramming | 1103 |
| [Node.js](library/NODEJS_GUIDE.md) | 22/24 LTS: event loop, CJS vs ESM, npm, **File System / OS / `child_process`**, streams, events, core modules, building CLIs (`parseArgs`), workers/cluster, error handling & graceful shutdown, `node:test`, `node:sqlite` | 621 |
| [Java](library/JAVA_GUIDE.md) | 21 & 25 LTS: JVM, syntax, OOP, records/sealed/pattern matching, generics, collections, streams & lambdas, exceptions, **`java.nio.file`/`ProcessBuilder`/OS info**, concurrency (incl. virtual threads), build tools, JUnit 5 | 2461 |
| [Kotlin](library/KOTLIN_GUIDE.md) | 2.x/K2: null safety, functions & extensions, scope functions, OOP, data/sealed/value classes, generics & variance, collections, coroutines & Flow, **File System/OS/`ProcessBuilder`**, delegation/DSLs, Java interop | 2597 |
| [Go (Golang) Ã¢â‚¬â€ Complete Language Reference](library/GO_GUIDE.md) | **Pure-language** beginnerÃ¢â€ â€™mastery: philosophy & toolchain, packages/modules, types/zero values, strings/runes/UTF-8, control flow & `defer`, slices (header/aliasing)/maps/structs, pointers, methods & **interfaces** (implicit), errors (`%w`/`Is`/`As`), generics, **concurrency** (goroutines/channels/`select`/`sync`/`context`/patterns), stdlib tour, testing/fuzzing, modules/tooling, perf/GC/pprof, security, gotchas | 2781 |
| [Go Ã¢â‚¬â€ Language & Production Patterns](library/GO_LANG_AND_PATTERNS_GUIDE.md) | Go language beginnerÃ¢â€ â€™advanced (slices, interfaces, errors, generics, concurrency) **plus** production design patterns Ã¢â‚¬â€ clean/hexagonal architecture, **repository, factory, adapter, DI, strategy/functional options, middleware** Ã¢â‚¬â€ with a complete Gin example + testing | 3054 |
| [Rust](library/RUST_GUIDE.md) | 2024 edition: **ownership/borrowing/lifetimes** explained patiently, structs/enums/`Option`/`Result`, traits & generics, collections & iterators, smart pointers, modules, concurrency & async/tokio, **`std::fs`/`std::process`/OS**, macros, testing | 2529 |

### Web Fundamentals
| Guide | What it covers | Lines |
|---|---|---|
| [HTML & HTML5](library/HTML_GUIDE.md) | Document structure, semantics, text/links/media, responsive images, tables, **forms (deep)**, metadata/SEO, accessibility/ARIA, `dialog`/`popover`/`details`, web-components basics, HTML5 API pointers | 2255 |
| [CSS & CSS3](library/CSS_GUIDE.md) | Selectors, **cascade/specificity/inheritance**, box model, units & custom properties, color (`oklch`), typography, **flexbox & grid (deep)**, positioning, responsive design & **container queries**, transitions/animations, nesting/`:has()`/`@layer`, BEM, a11y | 2225 |

### Frontend Ã¢â‚¬â€ React Ecosystem
| Guide | What it covers | Lines |
|---|---|---|
| [React 19](library/REACT_19_GUIDE.md) | The mental model (declarative UI, reconciliation, the render cycle, immutability), JSX, props/state, **all hooks explained**, the **when-NOT-to-use-`useEffect`** trap, context, refs, performance & the Compiler, and all React 19 APIs (`use`, Actions, `useActionState`/`useFormStatus`/`useOptimistic`, ref-as-prop, metadata), migration from 18 | 2272 |
| [Next.js 16](library/NEXTJS_16_GUIDE.md) | App Router (file conventions), **Server vs Client Components** & the boundary, routing (dynamic/parallel/intercepting), data fetching, **the four caches**, **Server Actions**, rendering strategies (static/dynamic/streaming/PPR), metadata/SEO, middleware, env-var security, auth patterns, deployment | 1527 |
| [TanStack Query v5](library/TANSTACK_QUERY_GUIDE.md) | Server-state model (vs useEffect/Redux), `useQuery` (staleTime vs gcTime, select, enabled), the cache lifecycle, `useMutation` & its lifecycle, invalidation, **optimistic updates**, dependent/parallel & **infinite** queries, prefetching, error boundaries, **Next.js hydration**, devtools | 1467 |
| [Zustand](library/ZUSTAND_GUIDE.md) | Client state: stores, **selectors & why they matter for re-renders**, `useShallow`, set/get & immer updates, middleware (persist/immer/devtools/subscribeWithSelector), slices, state outside React, async actions, the **Next.js per-request SSR pattern**, testing | 1546 |
| [React Hook Form](library/REACT_HOOK_FORM_GUIDE.md) | Why uncontrolled-first (the re-render story), `useForm` options, `register`/`handleSubmit`/`formState`, **zod validation**, `Controller` (when you need it), `watch`/`useWatch`, **`useFieldArray`**, `useFormContext`, async/server validation, the **shadcn Form pattern**, **Next.js Server Actions**, a11y | 1841 |
| [Motion (animation)](library/MOTION_ANIMATION_GUIDE.md) | Why an animation lib, the **`motion` component**, transitions (**tween vs spring**), **variants & orchestration** (stagger), **gestures** (hover/tap/drag/inView), **`AnimatePresence`** exit animations, **layout & shared-element** transitions, **scroll animations**, motion values, **performance & reduced-motion**, Next.js | 2342 |
| [Tailwind CSS (v4)](library/TAILWIND_CHEATSHEET.md) | Beginner-to-advanced guide: utility-first philosophy, v4 `@theme` setup, every core category, responsive/state variants (group/peer/has), dark mode, arbitrary values, `cn()`/CVA, plugins & framework integration | 1516 |
| [shadcn/ui](library/SHADCN_UI_CHEATSHEET.md) | The **own-your-code philosophy** (Radix + Tailwind + CVA), CLI (`init`/`add`), component anatomy (CVA/`cn()`/forwardRef), theming & dark mode, component-by-component reference with key props, the **Form pattern (RHF + zod)**, building your own variants, a11y, Next.js integration | 1578 |
| [Material UI (MUI)](library/MATERIAL_UI_GUIDE.md) | When to choose MUI, **the theme in depth** (palette/typography/overrides/CSS vars), **`sx` vs `styled()`**, the layout system (Box/Stack/v6 Grid), component reference with key props, **forms with RHF**, the **DataGrid** (sorting/filtering/server-side), dark mode, **Next.js App Router setup**, performance | 1970 |

### Mobile
| Guide | What it covers | Lines |
|---|---|---|
| [React Native + Expo](library/REACT_NATIVE_EXPO_GUIDE.md) | How RN works (Metro/Hermes/JSI/**New Architecture**), RN vs native vs Flutter, **core components** & native mapping, styling/flexbox, **Expo Router**, device/SDK APIs, **FlatList performance**, Reanimated & gestures, platform-specific code, **EAS Build/Submit/Update (OTA)**, testing, shipping | 2728 |
| [Android (Android Studio, Kotlin)](library/ANDROID_STUDIO_GUIDE.md) | **Project/folder structure** & the **Layout Editor**, Gradle/version catalogs, activity lifecycle & intents, **Jetpack Compose** + state, MVVM/ViewModel/StateFlow/Hilt, RecyclerView vs LazyColumn, Room/DataStore, Retrofit, coroutines/Flow, permissions, navigation, debugging/profiling, signing/publishing | 2320 |

### Backend Ã¢â‚¬â€ Node.js
| Guide | What it covers | Lines |
|---|---|---|
| [NestJS](library/NESTJS_GUIDE.md) | The DI-driven architecture & **IoC container**, modules/controllers/providers, **the request lifecycle** (pipes/guards/interceptors/filters, each explained), DTO validation, DB (Prisma/TypeORM), Passport **JWT/RBAC**, Swagger, WebSockets, microservices, caching, testing, security hardening | 2999 |
| [Fastify (v5)](library/FASTIFY_GUIDE.md) | Why Fastify, routing, **the full hook lifecycle**, **JSON-Schema validation & serialization**, **plugins & encapsulation**, decorators, TS type providers, core plugins (cors/jwt/multipart/swagger/websocket/rate-limit/helmet), error handling, Pino logging, **`app.inject()` testing**, production hardening | 2297 |
| [Prisma ORM](library/PRISMA_ORM_GUIDE.md) | What/when an ORM, schema & **relations in depth**, migrations (shadow DB), the full Client query API (filters/select/include/pagination/aggregates), **transactions**, **injection-safe raw queries**, the N+1 problem, the **Next.js singleton**, pooling, error codes, security | 1922 |

### Databases & Caching
| Guide | What it covers | Lines |
|---|---|---|
| [PostgreSQL (beginnerÃ¢â€ â€™advanced)](library/POSTGRESQL_GUIDE.md) | PG 17: data types, DDL/DML, joins, CTEs & window functions, JSONB, indexing, `EXPLAIN ANALYZE`, transactions/MVCC, PL/pgSQL & triggers, partitioning, RLS, extensions (pgvector), `pg`/`pgx` clients | 2702 |
| [Redis](library/REDIS_GUIDE.md) | Redis 7/8: all data types, Streams, pub/sub, transactions & Lua, persistence, caching patterns, eviction, distributed locks & rate limiting, Sentinel/Cluster, ACLs/TLS, vector search, `redis-cli`/`ioredis`/`go-redis` | 2531 |
| [Relational DB Design](library/RELATIONAL_DB_DESIGN_GUIDE.md) | Relational model, keys, **ER modeling**, relationships, **normalization (1NFÃ¢â‚¬â€œBCNF) & denormalization**, indexing for design, **advanced schema patterns** (trees, polymorphic, multi-tenancy, versioning, soft deletes), a worked e-commerce schema consumed from **NestJS+Prisma and Gin+pgx**, migrations & scaling | 2110 |
| [MongoDB & Document Design](library/MONGODB_GUIDE.md) | MongoDB 8: document model, CRUD, **embedding vs referencing**, **schema design patterns** (bucket/outlier/computed/subset/tree), indexing (ESR), the **aggregation pipeline**, validation, transactions, replication/sharding, **Node (native + Mongoose) & Go (mongo-go-driver)**, vector search | 2201 |
| [SQLite3](library/SQLITE3_GUIDE.md) | Embedded/serverless architecture, **type affinity** & STRICT tables, CRUD/CTEs/window functions, indexing, FK enforcement, transactions & **WAL**, key PRAGMAs, JSON1, FTS5, **Node (better-sqlite3, `node:sqlite`) & Go (modernc/mattn)**, tuning & ops | 2203 |

### Backend Ã¢â‚¬â€ Go
| Guide | What it covers | Lines |
|---|---|---|
| [Go `net/http` REST API](library/GO_NET_HTTP_REST_API_GUIDE.md) | The `net/http` Handler model, **1.22+ enhanced routing** (method+pattern, wildcards), Request/ResponseWriter in depth, a full CRUD API, **middleware as composable decorators**, context & cancellation, graceful shutdown, JSON errors, **security hardening** (timeouts/body limits/TLS), `httptest` | 2427 |
| [Go Gin Ã¢â‚¬â€ Production-Grade RESTful Backend](library/GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) | **Gin in depth** (engine/router/`*gin.Context`/middleware chain), RESTful design, binding/validation, **both folder structures (layer-based & feature-based)**, architecture & **design patterns** (handlerÃ¢â€ â€™serviceÃ¢â€ â€™repository, DI, factory, options, adapter), middleware, JWT/RBAC auth, DB layer (pgx/GORM, tx, migrations), config/secrets, observability, error handling, **file uploads**, performance/scaling, **security hardening**, testing, Docker deploy, a complete worked Users+Orders API, live reload with Air | 3336 |
| [Go Gorilla WebSockets](library/GO_GORILLA_WEBSOCKETS_GUIDE.md) | What WebSockets are (vs polling/SSE), the **Upgrader** & `CheckOrigin` (CSWSH), the **one-reader/one-writer** rule, **ping/pong** keepalive, the full **Hub** chat pattern, Go & browser JS clients, backpressure, **scaling with a Redis backplane**, auth & security | 1910 |
| [Go ent ORM](library/GO_ENT_ORM_GUIDE.md) | Schema-as-Go-code + **codegen** for fully type-safe queries, fields/**edges (relations)**, the generated builder, CRUD/query API with predicates, **eager loading (avoiding N+1)**, transactions, **hooks/interceptors**, versioned migrations (Atlas), privacy layer, security | 2220 |
| [Go JWT + Argon2](library/GO_JWT_ARGON2_GUIDE.md) | Password-hashing threat model, **Argon2id** (PHC format, parameter tuning, constant-time verify), JWT structure & golang-jwt v5 with **rigorous algorithm validation**, **access/refresh rotation & revocation**, auth middleware (net/http & Gin), **RBAC**, HMAC vs RSA/ECDSA, key management, token-storage security | 2164 |
| [Go Ã¢â‚¬â€ File System, OS & CLIs](library/GO_FILESYSTEM_OS_CLI_GUIDE.md) | Reading/writing files, dirs & paths (`filepath`), `io/fs`/`embed`/`os.Root`, **running external commands** (`os/exec`: npm/go get/pip/gitÃ¢â‚¬Â¦), env vars, system info, signals, **building CLIs** (`flag`, Cobra, Viper), archives, cross-platform | 2980 |
| [gRPC & RPC with Go](library/GO_GRPC_RPC_GUIDE.md) | `net/rpc` basics, Protocol Buffers (proto3), buf toolchain, all 4 streaming types, metadata, status errors, interceptors (auth/logging), TLS/mTLS, deadlines/retries, gRPC-Gateway & ConnectRPC, reflection, bufconn testing | 2602 |

### Auth & Backend Platforms
| Guide | What it covers | Lines |
|---|---|---|
| [Better Auth (Standalone, Next.js & Go)](library/BETTERAUTH_GUIDE.md) | Own-your-data auth: **sessions vs JWTs**, `betterAuth()` config, DB adapters, email/password & **OAuth**, secure cookies/CSRF, plugins (**2FA/passkeys**/org/JWT/bearer), **Next.js App Router** integration, and **verifying Better Auth JWTs in a Go backend** (JWKS) — security throughout | 2212 |
| [Supabase (with Next.js & Go)](library/SUPABASE_GUIDE.md) | Managed Postgres + Auth/Realtime/Storage/Edge Functions, the supabase-js query builder, Auth, **Row Level Security in depth** (anon vs service_role), Realtime subscriptions, Storage & signed URLs, the CLI/migrations/types, **`@supabase/ssr` for Next.js**, the **Go path** (pgx + JWT verification), pgvector | 2320 |

### Infrastructure & Protocols
| Guide | What it covers | Lines |
|---|---|---|
| [Docker & Docker Compose](library/DOCKER_GUIDE.md) | Containers vs VMs (namespaces/cgroups), the CLI, **Dockerfile instructions in depth** (CMD vs ENTRYPOINT, layer caching), **multi-stage builds** for tiny images, volumes & networking, env/secrets, **Compose** multi-service stacks, **security hardening** (non-root/distroless/scanning), image optimization, deployment | 1552 |
| [FTP Server Ã¢â‚¬â€ Go & Node](library/FTP_SERVER_GO_AND_NODE_GUIDE.md) | How FTP works (control/data, **active vs passive**), **FTP vs FTPS vs SFTP** (the security distinction), Go (`ftpserverlib`) + Node (`ftp-srv`) servers, FTPS/TLS & SFTP skeletons, user isolation, passive-port/firewall/Docker config, and a thorough **security** section | 1820 |

### Shells & Scripting
| Guide | What it covers | Lines |
|---|---|---|
| [Bash Scripting (beginnerÃ¢â€ â€™advanced)](library/BASH_SCRIPTING_GUIDE.md) | Bash 5.2+: quoting, variables & parameter expansion, conditionals/loops, functions & arrays, redirection & process substitution, `set -euo pipefail`/`trap`, grep/sed/awk/find/xargs, `getopts`, regex, robust-scripting recipes, portability/CRLF gotchas | 2440 |
| [Windows CMD & Batch (beginnerÃ¢â€ â€™advanced)](library/WINDOWS_CMD_BATCH_GUIDE.md) | `cmd.exe` & `.bat`/`.cmd`: built-in commands, `set`/arithmetic/prompts, arg modifiers (`%~dp0`), `if`/`for` (all forms incl. `for /F`), **delayed expansion**, `call`/`goto` subroutines, redirection, errorlevels, string ops, escaping reference, system one-liners | 1957 |
| [PowerShell (beginnerÃ¢â€ â€™advanced)](library/POWERSHELL_GUIDE.md) | 5.1 vs 7: the object pipeline, cmdlets & `Get-Member`, `Select`/`Where`/`ForEach-Object`, operators & regex, collections/hashtables, functions & `[CmdletBinding()]`, error handling, filesystem/registry providers, `PSCustomObject`/JSON/CSV, modules, `Invoke-RestMethod`, execution policy, 5.1Ã¢â€ â€7 encoding gotchas | 2091 |

---

## Ã°Å¸Â§Â­ Suggested Learning Tracks

Pick the track that matches your goal. Within a track, follow the order listed.

**Ã¢â€˜Â  Foundations (a language + the web)**
Learn a language top-to-bottom: [Python](library/PYTHON_GUIDE.md) Ã‚Â· [JavaScript](library/JAVASCRIPT_GUIDE.md) + [Node.js](library/NODEJS_GUIDE.md) Ã‚Â· [Java](library/JAVA_GUIDE.md) Ã‚Â· [Kotlin](library/KOTLIN_GUIDE.md) Ã‚Â· [Go](library/GO_GUIDE.md) Ã‚Â· [Rust](library/RUST_GUIDE.md). For the web, do [HTML](library/HTML_GUIDE.md) Ã¢â€ â€™ [CSS](library/CSS_GUIDE.md) Ã¢â€ â€™ [JavaScript](library/JAVASCRIPT_GUIDE.md) (incl. its DOM section).

**Ã¢â€˜Â¡ Frontend (React ecosystem)**
[HTML](library/HTML_GUIDE.md) Ã¢â€ â€™ [CSS](library/CSS_GUIDE.md) Ã¢â€ â€™ [JavaScript](library/JAVASCRIPT_GUIDE.md) Ã¢â€ â€™ 1. [React 19](library/REACT_19_GUIDE.md) Ã¢â€ â€™ 2. [Tailwind](library/TAILWIND_CHEATSHEET.md) Ã¢â€ â€™ 3. [Next.js 16](library/NEXTJS_16_GUIDE.md) Ã¢â€ â€™ 4. [shadcn/ui](library/SHADCN_UI_CHEATSHEET.md) Ã¢â€ â€™ 5. [React Hook Form](library/REACT_HOOK_FORM_GUIDE.md) Ã¢â€ â€™ 6. [Motion](library/MOTION_ANIMATION_GUIDE.md) Ã¢â€ â€™ 7. [TanStack Query](library/TANSTACK_QUERY_GUIDE.md) Ã¢â€ â€™ 8. [Zustand](library/ZUSTAND_GUIDE.md) Ã¢â€ â€™ 9. [Material UI](library/MATERIAL_UI_GUIDE.md)

**Ã¢â€˜Â¢ Full-stack with Node**
[JavaScript](library/JAVASCRIPT_GUIDE.md) Ã¢â€ â€™ [Node.js](library/NODEJS_GUIDE.md) Ã¢â€ â€™ [Next.js 16](library/NEXTJS_16_GUIDE.md) Ã¢â€ â€™ [Fastify](library/FASTIFY_GUIDE.md) / [NestJS](library/NESTJS_GUIDE.md) Ã¢â€ â€™ [PostgreSQL](library/POSTGRESQL_GUIDE.md) Ã¢â€ â€™ [Prisma](library/PRISMA_ORM_GUIDE.md) Ã¢â€ â€™ [Redis](library/REDIS_GUIDE.md) Ã¢â€ â€™ [Docker](library/DOCKER_GUIDE.md)

**Ã¢â€˜Â£ Backend with Go**
[Go Ã¢â‚¬â€ Language & Patterns](library/GO_LANG_AND_PATTERNS_GUIDE.md) Ã¢â€ â€™ [Go net/http](library/GO_NET_HTTP_REST_API_GUIDE.md) Ã¢â€ â€™ [Gin + uploads](library/GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) Ã¢â€ â€™ [JWT + Argon2](library/GO_JWT_ARGON2_GUIDE.md) Ã¢â€ â€™ [PostgreSQL](library/POSTGRESQL_GUIDE.md) Ã¢â€ â€™ [ent ORM](library/GO_ENT_ORM_GUIDE.md) Ã¢â€ â€™ [Redis](library/REDIS_GUIDE.md) Ã¢â€ â€™ [Gorilla WebSockets](library/GO_GORILLA_WEBSOCKETS_GUIDE.md) Ã¢â€ â€™ [gRPC & RPC](library/GO_GRPC_RPC_GUIDE.md) Ã¢â€ â€™ [File System / OS / CLIs](library/GO_FILESYSTEM_OS_CLI_GUIDE.md) Ã¢â€ â€™ [Docker](library/DOCKER_GUIDE.md)

**Ã¢â€˜Â¤ Data layer & database design**
[Relational DB Design](library/RELATIONAL_DB_DESIGN_GUIDE.md) Ã¢â€ â€™ [PostgreSQL](library/POSTGRESQL_GUIDE.md) Ã¢â€ â€™ [Prisma](library/PRISMA_ORM_GUIDE.md) (Node) / [ent ORM](library/GO_ENT_ORM_GUIDE.md) (Go) Ã¢â€ â€™ [SQLite3](library/SQLITE3_GUIDE.md) (embedded/local) Ã¢â€ â€™ [MongoDB](library/MONGODB_GUIDE.md) (document) Ã¢â€ â€™ [Redis](library/REDIS_GUIDE.md) (cache/KV)

**Ã¢â€˜Â¥ Auth & backend platforms**
[Better Auth](library/BETTERAUTH_GUIDE.md) (own your data: Next.js front + Go API) Ã¢â‚¬â€ or Ã¢â‚¬â€ [Supabase](library/SUPABASE_GUIDE.md) (managed Postgres + auth/realtime/storage, Next.js + Go). Pair either with [PostgreSQL](library/POSTGRESQL_GUIDE.md).

**Ã¢â€˜Â¦ Mobile**
Cross-platform: [React 19](library/REACT_19_GUIDE.md) Ã¢â€ â€™ [React Native + Expo](library/REACT_NATIVE_EXPO_GUIDE.md). Native Android: [Kotlin](library/KOTLIN_GUIDE.md) Ã¢â€ â€™ [Android (Android Studio)](library/ANDROID_STUDIO_GUIDE.md).

**Ã¢â€˜Â§ Shells & scripting (automation)**
[Bash](library/BASH_SCRIPTING_GUIDE.md) (Linux/macOS/CI/Git Bash) Ã‚Â· [PowerShell](library/POWERSHELL_GUIDE.md) (modern Windows + cross-platform automation) Ã‚Â· [Windows CMD & Batch](library/WINDOWS_CMD_BATCH_GUIDE.md) (legacy/`.bat`, still everywhere). On Windows, learn PowerShell for new work and CMD for reading existing scripts.

**Ã¢â€˜Â¨ Specialized**
[Go gRPC & RPC](library/GO_GRPC_RPC_GUIDE.md) Ã¢â‚¬â€ service-to-service APIs Ã‚Â· [Go File System / OS / CLIs](library/GO_FILESYSTEM_OS_CLI_GUIDE.md) Ã¢â‚¬â€ tooling & scripts Ã‚Â· [Rust](library/RUST_GUIDE.md) Ã¢â‚¬â€ systems/performance Ã‚Â· [FTP Server (Go & Node)](library/FTP_SERVER_GO_AND_NODE_GUIDE.md) Ã¢â‚¬â€ file-transfer services.

---

## Ã¢Å“â€¦ Accuracy & Verification Notes

- **Written for 2026.** Fast-moving APIs are flagged inside each guide with **Ã¢Å¡Â¡ Version notes** (and **Ã°Å¸â€ â€¢ React 19** markers in the React guide). Always confirm exact package versions against official docs when you build.
- **Go code was syntax-verified (original six Go guides).** All ~230 Go code blocks across the original six Go guides were parsed with the Go toolchain (`gofmt`); one real bug (an invalid variadic parameter in the JWT guide) was found and fixed. Note: this verifies *syntax*, not full type-checking of examples that depend on third-party packages.
- **Newly added guides** Ã¢â‚¬â€ Redis, Fastify, PostgreSQL, Go File System/OS/CLI, Better Auth, Supabase, and Go gRPC/RPC Ã¢â‚¬â€ are written for 2026 accuracy with runnable, commented examples. Their **Go code was syntax-verified**: all 194 ` ```go ` blocks across these guides were parsed with the Go toolchain (`go/parser`, Go 1.26) Ã¢â‚¬â€ **0 syntax errors**. (Six blocks intentionally interleave top-level declarations with example usage statements in one snippet; each part is valid Go but they don't compile as a single unit Ã¢â‚¬â€ that's a teaching style, not a bug.) The **TypeScript/SQL/Deno** examples and exact third-party **API/symbol names** were *not* compiler-checked Ã¢â‚¬â€ confirm against official docs before production use. For fast-moving libraries (e.g. Better Auth plugins), inline notes flag where to double-check.
- **Shell/scripting guides were linted.** The **Bash** guide's 96 ` ```bash ` blocks were run through **ShellCheck 0.11** (`--severity=error`): 92 pass clean; the 4 flagged are deliberate anti-pattern demos (each marked `# WRONG`/`# BUG` Ã¢â‚¬â€ ShellCheck agreeing they're bad confirms the lesson). The **PowerShell** guide's 94 ` ```powershell ` blocks were parsed with the **PowerShell 7.6** parser and **PSScriptAnalyzer 1.25** (Error severity): **0 errors** (7-only syntax such as `?:`, `??`, `&&`, `?.` is valid under 7 and clearly flagged as 7-only in the guide). **Windows CMD/Batch** has no standard linter, so it was not auto-checked Ã¢â‚¬â€ its logic was hand-reviewed.
- **Security-critical code** (Argon2 params, JWT algorithm validation, file-upload validation, Supabase RLS, command-injection avoidance in `os/exec`) follows current best practices but should be reviewed against official guidance before production use.

---

*This library is a personal study + reference workspace. Each guide is standalone Ã¢â‚¬â€ start anywhere.*
