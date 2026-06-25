# 📚 Developer Study Library & Workspace

A self-contained, **offline-first** collection of in-depth technical guides plus a real demo project. Everything here is written to be learned and built from **without an internet connection** — each guide has concepts, runnable commented examples, reference tables, gotchas, and a study path, accurate **as of June 2026** (see the **Versions at a Glance** table below for exactly what each targets).

> **How to use this:** All guides live in [`library/`](library/). Pick a track below, open the guide, and work top-to-bottom. Each guide ends with a "Study Path" and build-to-learn projects.

---

## 📌 Versions at a Glance — what each guide targets (as of **June 2026**)

So you always know *exactly* what you're learning against. Each guide's own **Version note** has the full detail and flags fast-moving APIs with **⚡**; this is the quick lookup. Verified current **June 2026**.

| Guide | Targets (June 2026) |
|---|---|
| **Languages** | |
| [Python](library/PYTHON_GUIDE.md) | Python **3.13 / 3.14** |
| [JavaScript](library/JAVASCRIPT_GUIDE.md) | **ECMAScript 2024 / 2025** (Node 22+, Deno, Bun) |
| [Node.js](library/NODEJS_GUIDE.md) | Node.js **22 / 24 LTS** |
| [Java](library/JAVA_GUIDE.md) | JDK **21 & 25 LTS** |
| [Kotlin](library/KOTLIN_GUIDE.md) | Kotlin **2.x** (K2) |
| [Go](library/GO_GUIDE.md) · [Patterns](library/GO_LANG_AND_PATTERNS_GUIDE.md) | **Go 1.25 / 1.26** |
| [Rust](library/RUST_GUIDE.md) | Rust **2024 edition**, stable toolchain |
| [PHP](library/PHP_GUIDE.md) | **PHP 8.4** (8.5 from Nov 2025 flagged) |
| [C](library/C_GUIDE.md) | **C23** (+ C17 baseline), GCC 14/15 · Clang 18/19 |
| [C++](library/CPP_GUIDE.md) | **C++23** (C++26 flagged), GCC 14/15 · Clang 18/19 · MSVC |
| **Web fundamentals** | |
| [HTML](library/HTML_GUIDE.md) | **HTML Living Standard** (WHATWG, 2026) |
| [CSS](library/CSS_GUIDE.md) | **Modern/baseline CSS 2026** (`:has()`, container queries, nesting, `oklch`) |
| **Frontend — React** | |
| [React 19](library/REACT_19_GUIDE.md) | **React 19** (stable) |
| [Next.js 16](library/NEXTJS_16_GUIDE.md) | **Next.js 16** (App Router) + React 19 |
| [TanStack Query](library/TANSTACK_QUERY_GUIDE.md) | **v5** (`@tanstack/react-query`) + React 19 |
| [Zustand](library/ZUSTAND_GUIDE.md) | **v5** + React 19 / Next 15 |
| [React Hook Form](library/REACT_HOOK_FORM_GUIDE.md) | **v7** + Zod **v3/v4** + React 19 |
| [Motion](library/MOTION_ANIMATION_GUIDE.md) | **Motion** (ex-Framer Motion) + React 19 / Next 15-16 |
| [Tailwind CSS](library/TAILWIND_CHEATSHEET.md) | **v4** (Oxide engine, CSS-first `@theme`) |
| [shadcn/ui](library/SHADCN_UI_CHEATSHEET.md) | current `shadcn` CLI + React 19 + Tailwind v4 |
| [Material UI](library/MATERIAL_UI_GUIDE.md) | MUI **v6 & v7** + React 19 |
| [GSAP + React/Next.js](library/GSAP_GUIDE.md) | **GSAP 3.13+** (all plugins now free) + `@gsap/react` `useGSAP()` + React 19 / Next 16 |
| **Mobile** | |
| [React Native + Expo](library/REACT_NATIVE_EXPO_GUIDE.md) | **Expo SDK 54**, **React Native 0.81+** (New Architecture), React 19 |
| [Android (Android Studio)](library/ANDROID_STUDIO_GUIDE.md) | 2026 Android toolchain + **Jetpack Compose** |
| **Backend — Python** | |
| [FastAPI](library/FASTAPI_GUIDE.md) | **FastAPI 0.115+** + **Pydantic v2** + SQLAlchemy 2.0 + Python 3.13/3.14 |
| [Django](library/DJANGO_GUIDE.md) | **Django 5.2 LTS & 6.0** + **DRF** + Python 3.12–3.14 |
| **Backend — Java** | |
| [Spring Boot](library/SPRING_BOOT_GUIDE.md) | **Spring Boot 3.4/3.5** (SB 4 flagged) + Spring Framework 6.2 + **Java 21 LTS** |
| **Backend — PHP** | |
| [Laravel](library/LARAVEL_GUIDE.md) | **Laravel 12** (PHP 8.2+; **L13** ~Q1 2026 flagged) |
| **Backend — Node.js** | |
| [NestJS](library/NESTJS_GUIDE.md) | **NestJS 11** + Node 22/24 + TS 5.x |
| [Fastify](library/FASTIFY_GUIDE.md) | **Fastify v5** + Node 22/24 |
| [Prisma ORM](library/PRISMA_ORM_GUIDE.md) | **Prisma 6** + Node 20/22 + TS 5.x |
| **Databases & caching** | |
| [PostgreSQL](library/POSTGRESQL_GUIDE.md) | **PostgreSQL 17 & 18** (PG 18 current, GA Sept 2025) |
| [Redis](library/REDIS_GUIDE.md) | **Redis 7.x & 8.x** |
| [Relational DB Design](library/RELATIONAL_DB_DESIGN_GUIDE.md) | **PostgreSQL 18** dialect (principles are RDBMS-agnostic) |
| [MongoDB](library/MONGODB_GUIDE.md) | **MongoDB 8.x** |
| [SQLite3](library/SQLITE3_GUIDE.md) | **SQLite 3.45–3.50** |
| **Backend — Go** (all **Go 1.25 / 1.26**) | |
| [Go net/http](library/GO_NET_HTTP_REST_API_GUIDE.md) | stdlib `net/http` (1.22+ routing) |
| [Go Gin](library/GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) | **Gin v1.10+** |
| [Go Gorilla WebSockets](library/GO_GORILLA_WEBSOCKETS_GUIDE.md) | **gorilla/websocket v1.5.x** |
| [Go ent ORM](library/GO_ENT_ORM_GUIDE.md) | **ent v0.14+** |
| [Go JWT + Argon2](library/GO_JWT_ARGON2_GUIDE.md) | **golang-jwt/jwt v5** + `x/crypto/argon2` |
| [Go FS/OS/CLIs](library/GO_FILESYSTEM_OS_CLI_GUIDE.md) | stdlib + Cobra/Viper |
| [Go gRPC & RPC](library/GO_GRPC_RPC_GUIDE.md) | **proto3**, `grpc` v1.6x+, `protobuf` v1.3x+ |
| **Auth & platforms** | |
| [Better Auth](library/BETTERAUTH_GUIDE.md) | **Better Auth v1.x** + Next.js / Go |
| [Supabase](library/SUPABASE_GUIDE.md) | Supabase 2026 (`@supabase/ssr`) + Next.js / Go |
| **Infrastructure & protocols** | |
| [Networking](library/NETWORKING_GUIDE.md) | 2026 protocols: **HTTP/3 + QUIC**, **TLS 1.3**, IPv6 (fundamentals are timeless) |
| [Docker](library/DOCKER_GUIDE.md) | **Docker Engine 28.x** + **Compose v2** |
| [Nginx](library/NGINX_GUIDE.md) | **Nginx 1.29+** (1.28 stable / 1.29 mainline), HTTP/2 & /3 |
| [FTP Server (Go & Node)](library/FTP_SERVER_GO_AND_NODE_GUIDE.md) | Go 1.25/1.26 + ftpserverlib 0.21–0.22; Node 20/22 + ftp-srv v5 |
| **Server & systems admin** | |
| [Linux Server Admin](library/LINUX_SERVER_ADMIN_GUIDE.md) | **Ubuntu 24.04 / Debian 12 / RHEL 9–10** (systemd, nftables) |
| [Windows Server Admin](library/WINDOWS_SERVER_ADMIN_GUIDE.md) | **Windows Server 2025** (2022/2019 noted) |
| [Database Server Admin](library/DATABASE_SERVER_ADMIN_GUIDE.md) | PG 17/18, MySQL 8.4 LTS, MariaDB 11.x, MongoDB 8, Redis 7/8, SQL Server 2022/2025 |
| **Version control & DevOps** | |
| [Git](library/GIT_GUIDE.md) | **Git 2.45+** |
| [CI/CD with GitHub Actions](library/GITHUB_ACTIONS_CICD_GUIDE.md) | GitHub Actions 2026 (current `actions/*`, OIDC) |
| **Shells & scripting** | |
| [Bash](library/BASH_SCRIPTING_GUIDE.md) | **Bash 5.2+** |
| [Windows CMD & Batch](library/WINDOWS_CMD_BATCH_GUIDE.md) | **Windows 11 `cmd.exe`** (legacy/frozen) |
| [PowerShell](library/POWERSHELL_GUIDE.md) | **Windows PowerShell 5.1 & PowerShell 7.x** |

---

## 🗂️ The Guides (`library/`)

### Programming Languages (beginner → advanced)
| Guide | What it covers | Lines |
|---|---|---|
| [Python](library/PYTHON_GUIDE.md) | 3.13/3.14: syntax, data structures, control flow & `match`, functions/closures, OOP & dataclasses, modules, exceptions, generators/context managers, decorators, type hints, **File System/OS/`subprocess`/CLIs**, concurrency (threads/multiprocessing/asyncio + GIL), stdlib, pytest, packaging | 1457 |
| [JavaScript](library/JAVASCRIPT_GUIDE.md) | ES2024/25: types & coercion, modern syntax, functions, scope/closures, `this`, objects, prototypes & classes, arrays, async (event loop/promises/await), modules, **a detailed DOM & browser-APIs section**, FS/OS via a runtime, metaprogramming | 1103 |
| [Node.js](library/NODEJS_GUIDE.md) | 22/24 LTS: event loop, CJS vs ESM, npm, **File System / OS / `child_process`**, streams, events, core modules, building CLIs (`parseArgs`), workers/cluster, error handling & graceful shutdown, `node:test`, `node:sqlite` | 621 |
| [Java](library/JAVA_GUIDE.md) | 21 & 25 LTS: JVM, syntax, OOP, records/sealed/pattern matching, generics, collections, streams & lambdas, exceptions, **`java.nio.file`/`ProcessBuilder`/OS info**, concurrency (incl. virtual threads), build tools, JUnit 5 | 2461 |
| [Kotlin](library/KOTLIN_GUIDE.md) | 2.x/K2: null safety, functions & extensions, scope functions, OOP, data/sealed/value classes, generics & variance, collections, coroutines & Flow, **File System/OS/`ProcessBuilder`**, delegation/DSLs, Java interop | 2597 |
| [Go (Golang) — Complete Language Reference](library/GO_GUIDE.md) | **Pure-language** beginner→mastery: philosophy & toolchain, packages/modules, types/zero values, strings/runes/UTF-8, control flow & `defer`, slices (header/aliasing)/maps/structs, pointers, methods & **interfaces** (implicit), errors (`%w`/`Is`/`As`), generics, **concurrency** (goroutines/channels/`select`/`sync`/`context`/patterns), stdlib tour, testing/fuzzing, modules/tooling, perf/GC/pprof, security, gotchas | 2781 |
| [Go — Language & Production Patterns](library/GO_LANG_AND_PATTERNS_GUIDE.md) | Go language beginner→advanced (slices, interfaces, errors, generics, concurrency) **plus** production design patterns — clean/hexagonal architecture, **repository, factory, adapter, DI, strategy/functional options, middleware** — with a complete Gin example + testing | 3054 |
| [Rust](library/RUST_GUIDE.md) | 2024 edition: **ownership/borrowing/lifetimes** explained patiently, structs/enums/`Option`/`Result`, traits & generics, collections & iterators, smart pointers, modules, concurrency & async/tokio, **`std::fs`/`std::process`/OS**, macros, testing | 2529 |
| [PHP](library/PHP_GUIDE.md) | **Modern PHP 8.4** (+ 8.5 flagged), June-2026 current: setup/CLI/FPM/Composer, the typed syntax & `match`, functions/closures, **arrays in depth**, OOP (typed/readonly props, constructor promotion, interfaces/traits, **enums**), the modern type system (**property hooks & asymmetric visibility**, union/intersection), namespaces/PSR-4, errors, **File System/OS/`proc_open`/exec**, JSON/dates/PCRE, **PDO & injection-safe prepared statements**, web (superglobals/sessions/**XSS/CSRF/`password_hash`**), Composer/PSRs, attributes/generators/**Fibers**, **PHPUnit/Pest + PHPStan**, OPcache/JIT, security | 2351 |
| [C](library/C_GUIDE.md) | **C23** (+ C17 baseline), GCC/Clang: the compile pipeline, types & the **integer/overflow** minefield, **pointers in depth**, arrays/strings, **memory management** (malloc/free + the classic bugs), structs/unions/enums/alignment, a rigorous **Undefined Behavior** section, the preprocessor (`#embed`), **File System/OS/`fork`/`exec`/`pipe`**, the stdlib, multi-file builds (**Make/CMake**), concurrency (pthreads/C11 threads/atomics), **production-safe C** (ASan/UBSan/valgrind, CERT-C, fuzzing), debugging/testing, performance | 2059 |
| [C++](library/CPP_GUIDE.md) | **C++23** (C++26 flagged), modern C++ not C-with-classes: build/CMake, fundamentals & `auto`, **classes & RAII** (Rule of 0/3/5), **smart pointers & ownership**, **move semantics & value categories**, the **STL** (containers/iterators/**ranges**), **templates & concepts**, error handling (**`std::expected`**/exceptions/`optional`), polymorphism & `variant`, **`<filesystem>`/OS/processes**, concurrency (`jthread`/atomics/coroutines), **CMake/vcpkg/Conan**, **production C++** (sanitizers/Core Guidelines), GoogleTest/Catch2, performance | 2037 |

### Web Fundamentals
| Guide | What it covers | Lines |
|---|---|---|
| [HTML & HTML5](library/HTML_GUIDE.md) | Document structure, semantics, text/links/media, responsive images, tables, **forms (deep)**, metadata/SEO, accessibility/ARIA, `dialog`/`popover`/`details`, web-components basics, HTML5 API pointers | 2255 |
| [CSS & CSS3](library/CSS_GUIDE.md) | Selectors, **cascade/specificity/inheritance**, box model, units & custom properties, color (`oklch`), typography, **flexbox & grid (deep)**, positioning, responsive design & **container queries**, transitions/animations, nesting/`:has()`/`@layer`, BEM, a11y | 2225 |

### Frontend — React Ecosystem
| Guide | What it covers | Lines |
|---|---|---|
| [React 19](library/REACT_19_GUIDE.md) | The mental model (declarative UI, reconciliation, the render cycle, immutability), JSX, props/state, **all hooks explained**, the **when-NOT-to-use-`useEffect`** trap, context, refs, performance & the Compiler, and all React 19 APIs (`use`, Actions, `useActionState`/`useFormStatus`/`useOptimistic`, ref-as-prop, metadata), migration from 18 | 2272 |
| [Next.js 16](library/NEXTJS_16_GUIDE.md) | App Router (file conventions), **Server vs Client Components** & the boundary, routing (dynamic/parallel/intercepting), data fetching, **the four caches**, **Server Actions**, rendering strategies (static/dynamic/streaming/PPR), metadata/SEO, middleware, env-var security, auth patterns, deployment | 1527 |
| [TanStack Query v5](library/TANSTACK_QUERY_GUIDE.md) | Server-state model (vs useEffect/Redux), `useQuery` (staleTime vs gcTime, select, enabled), the cache lifecycle, `useMutation` & its lifecycle, invalidation, **optimistic updates**, dependent/parallel & **infinite** queries, prefetching, error boundaries, **Next.js hydration**, devtools | 1467 |
| [Zustand](library/ZUSTAND_GUIDE.md) | Client state: stores, **selectors & why they matter for re-renders**, `useShallow`, set/get & immer updates, middleware (persist/immer/devtools/subscribeWithSelector), slices, state outside React, async actions, the **Next.js per-request SSR pattern**, testing | 1546 |
| [React Hook Form](library/REACT_HOOK_FORM_GUIDE.md) | Why uncontrolled-first (the re-render story), `useForm` options, `register`/`handleSubmit`/`formState`, **zod validation**, `Controller` (when you need it), `watch`/`useWatch`, **`useFieldArray`**, `useFormContext`, async/server validation, the **shadcn Form pattern**, **Next.js Server Actions**, a11y | 1841 |
| [Motion (animation)](library/MOTION_ANIMATION_GUIDE.md) | Why an animation lib, the **`motion` component**, transitions (**tween vs spring**), **variants & orchestration** (stagger), **gestures** (hover/tap/drag/inView), **`AnimatePresence`** exit animations, **layout & shared-element** transitions, **scroll animations**, motion values, **performance & reduced-motion**, Next.js | 2342 |
| [GSAP (GreenSock) + React/Next.js](library/GSAP_GUIDE.md) | **GSAP 3.13+** (now **100% free incl. all plugins** post-Webflow): tweens (`to`/`from`/`fromTo`/`set`), **easing**, **timelines** & the position parameter, staggers, **the React integration done right** (`@gsap/react` + **`useGSAP()`** + cleanup), **ScrollTrigger** (scrub/pin/snap/`matchMedia`), **Next.js specifics** (`"use client"`/SSR/hydration), the free plugin suite (**SplitText/ScrollSmoother/MorphSVG/Draw SVG/MotionPath/Flip**), the **Flip** plugin with React state, responsive & reduced-motion a11y, **performance**, GSAP vs Motion | 1761 |
| [Tailwind CSS (v4)](library/TAILWIND_CHEATSHEET.md) | Beginner-to-advanced guide: utility-first philosophy, v4 `@theme` setup, every core category, responsive/state variants (group/peer/has), dark mode, arbitrary values, `cn()`/CVA, plugins & framework integration | 1516 |
| [shadcn/ui](library/SHADCN_UI_CHEATSHEET.md) | The **own-your-code philosophy** (Radix + Tailwind + CVA), CLI (`init`/`add`), component anatomy (CVA/`cn()`/forwardRef), theming & dark mode, component-by-component reference with key props, the **Form pattern (RHF + zod)**, building your own variants, a11y, Next.js integration | 1578 |
| [Material UI (MUI)](library/MATERIAL_UI_GUIDE.md) | When to choose MUI, **the theme in depth** (palette/typography/overrides/CSS vars), **`sx` vs `styled()`**, the layout system (Box/Stack/v6 Grid), component reference with key props, **forms with RHF**, the **DataGrid** (sorting/filtering/server-side), dark mode, **Next.js App Router setup**, performance | 1970 |

### Mobile
| Guide | What it covers | Lines |
|---|---|---|
| [React Native + Expo](library/REACT_NATIVE_EXPO_GUIDE.md) | How RN works (Metro/Hermes/JSI/**New Architecture**), RN vs native vs Flutter, **core components** & native mapping, styling/flexbox, **Expo Router**, device/SDK APIs, **FlatList performance**, Reanimated & gestures, platform-specific code, **EAS Build/Submit/Update (OTA)**, testing, shipping | 2728 |
| [Android (Android Studio, Kotlin)](library/ANDROID_STUDIO_GUIDE.md) | **Project/folder structure** & the **Layout Editor**, Gradle/version catalogs, activity lifecycle & intents, **Jetpack Compose** + state, MVVM/ViewModel/StateFlow/Hilt, RecyclerView vs LazyColumn, Room/DataStore, Retrofit, coroutines/Flow, permissions, navigation, debugging/profiling, signing/publishing | 2320 |

### Backend — Python (FastAPI & Django)
| Guide | What it covers | Lines |
|---|---|---|
| [FastAPI](library/FASTAPI_GUIDE.md) | **FastAPI 0.115+** on **Python 3.13/3.14**, **Pydantic v2**, **SQLAlchemy 2.0** (June 2026): the async ASGI model & request lifecycle, path operations & routing, **Pydantic v2 models/validation**, requests/responses & error handling, **Dependency Injection** (`Depends`/yield), **async & not-blocking the loop**, **databases** (SQLAlchemy/SQLModel async + Alembic, N+1), **auth** (OAuth2/JWT/hashing/scopes), settings (`pydantic-settings`), middleware/CORS, **background tasks & queues** (Celery/ARQ), WebSockets/streaming, **testing**, **scale-ready project structure**, **production deploy** (Gunicorn/Uvicorn behind Nginx, Docker, observability), performance/caching, security | 1584 |
| [Django](library/DJANGO_GUIDE.md) | **Django 5.2 LTS** (+ 6.0) & **DRF**, Python 3.12–3.14 (June 2026): the batteries-included MTV model, URLs/views, **the ORM in depth** (QuerySets, relationships, **N+1 killers** `select_related`/`prefetch_related`, transactions), **migrations**, the **Admin**, forms, templates, **auth** (custom user model from day one, permissions/groups), 12-factor settings/secrets, middleware, **Django REST Framework** (serializers/viewsets/auth/permissions/throttling), **async Django**, **caching** (Redis), **Celery** background tasks, **testing**, scale-ready structure, **production deploy** (Gunicorn/ASGI behind Nginx, WhiteNoise, Docker, the deploy checklist), **security** (CSRF/XSS/SQLi, `check --deploy`), ORM performance | 2254 |

### Backend — Java (Spring Boot)
| Guide | What it covers | Lines |
|---|---|---|
| [Spring Boot](library/SPRING_BOOT_GUIDE.md) | **Spring Boot 3.4/3.5** (Spring Framework 6.2, **Java 21 LTS**; SB 4 flagged), June 2026: the **IoC container & DI** (constructor injection), Initializr/Maven & `@SpringBootApplication`, **config & profiles** (`@ConfigurationProperties`), **REST APIs** (`@RestController`, DTOs-vs-entities), validation & `@ControllerAdvice`, **Spring Data JPA** (entities/repositories/`@Query`, **N+1**/fetch, **Flyway/Liquibase**), **Spring Security** (the lambda DSL, **JWT/OAuth2**, method security), **`@Transactional`** & the service layer, AOP, **testing** (slices/MockMvc/**Testcontainers**), async/scheduling/**virtual threads**, caching (Redis), **Actuator/Micrometer observability**, production (jar/**Docker**/**GraalVM native**/behind Nginx), clean architecture at scale, performance (HikariCP, WebFlux note), security | 1718 |

### Backend — PHP (Laravel)
| Guide | What it covers | Lines |
|---|---|---|
| [Laravel](library/LARAVEL_GUIDE.md) | **Laravel 12** (PHP 8.2+, June-2026 current; L13 flagged): the **request lifecycle** & the **slimmed L11+ skeleton** (`bootstrap/app.php`, no Kernel.php), setup (installer/**Herd**/**Sail**), routing & **route model binding**, controllers/requests/responses, **Blade** + the new **React/Vue/Livewire starter kits** (Inertia/Volt), **the service container & DI**, service providers, **Eloquent in depth** (migrations, relationships, **N+1/eager loading**, casts, factories), query builder & transactions, **form-request validation**, middleware, **auth** (Gates/Policies, **Sanctum** SPA+API, Fortify/Socialite), **REST APIs** (API Resources), **queues/jobs/Horizon**, **events & broadcasting (Reverb)**, Artisan & scheduling, cache/Redis, mail/notifications, storage, **Pest/PHPUnit testing**, production (**Octane**/Forge/Vapor, Nginx+FPM), security | 2705 |

### Backend — Node.js
| Guide | What it covers | Lines |
|---|---|---|
| [NestJS](library/NESTJS_GUIDE.md) | The DI-driven architecture & **IoC container**, modules/controllers/providers, **the request lifecycle** (pipes/guards/interceptors/filters, each explained), DTO validation, DB (Prisma/TypeORM), Passport **JWT/RBAC**, Swagger, WebSockets, microservices, caching, testing, security hardening | 2999 |
| [Fastify (v5)](library/FASTIFY_GUIDE.md) | Why Fastify, routing, **the full hook lifecycle**, **JSON-Schema validation & serialization**, **plugins & encapsulation**, decorators, TS type providers, core plugins (cors/jwt/multipart/swagger/websocket/rate-limit/helmet), error handling, Pino logging, **`app.inject()` testing**, production hardening | 2297 |
| [Prisma ORM](library/PRISMA_ORM_GUIDE.md) | What/when an ORM, schema & **relations in depth**, migrations (shadow DB), the full Client query API (filters/select/include/pagination/aggregates), **transactions**, **injection-safe raw queries**, the N+1 problem, the **Next.js singleton**, pooling, error codes, security | 1922 |

### Databases & Caching
| Guide | What it covers | Lines |
|---|---|---|
| [PostgreSQL (beginner→advanced)](library/POSTGRESQL_GUIDE.md) | PG 17: data types, DDL/DML, joins, CTEs & window functions, JSONB, indexing, `EXPLAIN ANALYZE`, transactions/MVCC, PL/pgSQL & triggers, partitioning, RLS, extensions (pgvector), `pg`/`pgx` clients | 2702 |
| [Redis](library/REDIS_GUIDE.md) | Redis 7/8: all data types, Streams, pub/sub, transactions & Lua, persistence, caching patterns, eviction, distributed locks & rate limiting, Sentinel/Cluster, ACLs/TLS, vector search, `redis-cli`/`ioredis`/`go-redis` | 2531 |
| [Relational DB Design](library/RELATIONAL_DB_DESIGN_GUIDE.md) | Relational model, keys, **ER modeling**, relationships, **normalization (1NF–BCNF) & denormalization**, indexing for design, **advanced schema patterns** (trees, polymorphic, multi-tenancy, versioning, soft deletes), a worked e-commerce schema consumed from **NestJS+Prisma and Gin+pgx**, migrations & scaling | 2110 |
| [MongoDB & Document Design](library/MONGODB_GUIDE.md) | MongoDB 8: document model, CRUD, **embedding vs referencing**, **schema design patterns** (bucket/outlier/computed/subset/tree), indexing (ESR), the **aggregation pipeline**, validation, transactions, replication/sharding, **Node (native + Mongoose) & Go (mongo-go-driver)**, vector search | 2201 |
| [SQLite3](library/SQLITE3_GUIDE.md) | Embedded/serverless architecture, **type affinity** & STRICT tables, CRUD/CTEs/window functions, indexing, FK enforcement, transactions & **WAL**, key PRAGMAs, JSON1, FTS5, **Node (better-sqlite3, `node:sqlite`) & Go (modernc/mattn)**, tuning & ops | 2203 |

### Backend — Go
| Guide | What it covers | Lines |
|---|---|---|
| [Go `net/http` REST API](library/GO_NET_HTTP_REST_API_GUIDE.md) | The `net/http` Handler model, **1.22+ enhanced routing** (method+pattern, wildcards), Request/ResponseWriter in depth, a full CRUD API, **middleware as composable decorators**, context & cancellation, graceful shutdown, JSON errors, **security hardening** (timeouts/body limits/TLS), `httptest` | 2427 |
| [Go Gin — Production-Grade RESTful Backend](library/GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) | **Gin in depth** (engine/router/`*gin.Context`/middleware chain), RESTful design, binding/validation, **both folder structures (layer-based & feature-based)**, architecture & **design patterns** (handler→service→repository, DI, factory, options, adapter), middleware, JWT/RBAC auth, DB layer (pgx/GORM, tx, migrations), config/secrets, observability, error handling, **file uploads**, performance/scaling, **security hardening**, testing, Docker deploy, a complete worked Users+Orders API, live reload with Air | 3336 |
| [Go Gorilla WebSockets](library/GO_GORILLA_WEBSOCKETS_GUIDE.md) | What WebSockets are (vs polling/SSE), the **Upgrader** & `CheckOrigin` (CSWSH), the **one-reader/one-writer** rule, **ping/pong** keepalive, the full **Hub** chat pattern, Go & browser JS clients, backpressure, **scaling with a Redis backplane**, auth & security | 1910 |
| [Go ent ORM](library/GO_ENT_ORM_GUIDE.md) | Schema-as-Go-code + **codegen** for fully type-safe queries, fields/**edges (relations)**, the generated builder, CRUD/query API with predicates, **eager loading (avoiding N+1)**, transactions, **hooks/interceptors**, versioned migrations (Atlas), privacy layer, security | 2220 |
| [Go JWT + Argon2](library/GO_JWT_ARGON2_GUIDE.md) | Password-hashing threat model, **Argon2id** (PHC format, parameter tuning, constant-time verify), JWT structure & golang-jwt v5 with **rigorous algorithm validation**, **access/refresh rotation & revocation**, auth middleware (net/http & Gin), **RBAC**, HMAC vs RSA/ECDSA, key management, token-storage security | 2164 |
| [Go — File System, OS & CLIs](library/GO_FILESYSTEM_OS_CLI_GUIDE.md) | Reading/writing files, dirs & paths (`filepath`), `io/fs`/`embed`/`os.Root`, **running external commands** (`os/exec`: npm/go get/pip/git…), env vars, system info, signals, **building CLIs** (`flag`, Cobra, Viper), archives, cross-platform | 2980 |
| [gRPC & RPC with Go](library/GO_GRPC_RPC_GUIDE.md) | `net/rpc` basics, Protocol Buffers (proto3), buf toolchain, all 4 streaming types, metadata, status errors, interceptors (auth/logging), TLS/mTLS, deadlines/retries, gRPC-Gateway & ConnectRPC, reflection, bufconn testing | 2602 |

### Auth & Backend Platforms
| Guide | What it covers | Lines |
|---|---|---|
| [Better Auth (Standalone, Next.js & Go)](library/BETTERAUTH_GUIDE.md) | Own-your-data auth: **sessions vs JWTs**, `betterAuth()` config, DB adapters, email/password & **OAuth**, secure cookies/CSRF, plugins (**2FA/passkeys**/org/JWT/bearer), **Next.js App Router** integration, and **verifying Better Auth JWTs in a Go backend** (JWKS) — security throughout | 2212 |
| [Supabase (with Next.js & Go)](library/SUPABASE_GUIDE.md) | Managed Postgres + Auth/Realtime/Storage/Edge Functions, the supabase-js query builder, Auth, **Row Level Security in depth** (anon vs service_role), Realtime subscriptions, Storage & signed URLs, the CLI/migrations/types, **`@supabase/ssr` for Next.js**, the **Go path** (pgx + JWT verification), pgvector | 2320 |

### Infrastructure & Protocols
| Guide | What it covers | Lines |
|---|---|---|
| [Networking (beginner→advanced)](library/NETWORKING_GUIDE.md) | A full networking course for software engineers: the **layered model & encapsulation** (OSI/TCP-IP), **Ethernet/MAC/ARP/switches/VLANs**, **IP addressing & subnetting/CIDR** + IPv6, **routing/NAT/BGP**, **TCP vs UDP** (handshake, flow & **congestion control**), **sockets** (Go/Node/Python), **DNS**, **HTTP/1.1→2→3 & QUIC**, **TLS/PKI/certificates**, **WebSockets/SSE**, **security** (firewalls, VPNs, MITM/DDoS/spoofing & defenses), the **troubleshooting toolkit** (ping/traceroute/dig/ss/curl/tcpdump/Wireshark/nmap), **robust network programming** (timeouts/retries/pooling), **cloud/container networking** (Docker/k8s/VPC/CDN/service mesh), and **performance** (latency vs bandwidth, RTT, the BDP) | 1432 |
| [Docker & Docker Compose](library/DOCKER_GUIDE.md) | Containers vs VMs (namespaces/cgroups), the CLI, **Dockerfile instructions in depth** (CMD vs ENTRYPOINT, layer caching), **multi-stage builds** for tiny images, volumes & networking, env/secrets, **Compose** multi-service stacks, **security hardening** (non-root/distroless/scanning), image optimization, deployment | 1552 |
| [Nginx](library/NGINX_GUIDE.md) | The event-driven architecture (C10k), the config model & **`location` matching**, static serving, **reverse proxy** (headers/WebSockets), **load balancing** (algorithms/health checks), **API gateway** (routing, **rate limiting**, `auth_request`, CORS), **TLS/HTTP-2/3**, **caching** & compression, **security hardening**, performance tuning, real-world configs | 1929 |
| [FTP Server — Go & Node](library/FTP_SERVER_GO_AND_NODE_GUIDE.md) | How FTP works (control/data, **active vs passive**), **FTP vs FTPS vs SFTP** (the security distinction), Go (`ftpserverlib`) + Node (`ftp-srv`) servers, FTPS/TLS & SFTP skeletons, user isolation, passive-port/firewall/Docker config, and a thorough **security** section | 1820 |

### Server & Systems Administration (setup → production, company-level)
| Guide | What it covers | Lines |
|---|---|---|
| [Linux Server Administration](library/LINUX_SERVER_ADMIN_GUIDE.md) | Beginner→pro Linux ops on 2026 distros (Ubuntu 24.04/Debian 12/RHEL 9-10): **SSH in depth** & hardening, the FHS, **users/groups/permissions/sudo**, package management (apt/dnf), processes & resources, **systemd in depth** (units/journald/timers/cgroups), storage (**LVM/RAID/filesystems**/fstab/swap), server networking, **firewalls** (nftables/ufw/firewalld/fail2ban), **hardening** (SELinux/AppArmor/sysctl/auditd), automation & cron, **logging & monitoring** (Prometheus/Grafana), **production web serving** (systemd + TLS/certbot + reverse proxy), containers, **backups & DR** (3-2-1/restic/borg), performance tuning, **IaC with Ansible**, and company-level ops (runbooks/on-call/change mgmt) | 2050 |
| [Windows Server Administration](library/WINDOWS_SERVER_ADMIN_GUIDE.md) | Beginner→pro on **Windows Server 2025**: editions/licensing, **Server Core vs Desktop Experience**, install & `sconfig`, **PowerShell for admins** (remoting/JEA), accounts & **NTFS vs share** permissions/UAC, **Active Directory** (forest/domain/OU/DC/FSMO/sites/replication/trusts), **Group Policy in depth** (LSDOU), **DNS & DHCP** roles, **file services** (SMB/DFS/FSRM/Storage Spaces/VSS), **IIS** hosting, **Hyper-V**, storage (ReFS/iSCSI), Defender Firewall, **hardening** (BitLocker/Credential Guard/LAPS/WSUS/baselines), monitoring (Event Viewer/PerfMon/WAC), **backup & AD recovery**, automation (**DSC**/scheduled tasks/Azure Arc), and company-level ops | 1599 |
| [Database Server Administration (DBA)](library/DATABASE_SERVER_ADMIN_GUIDE.md) | The **ops/DBA discipline** (not SQL — the engine guides teach that): install/setup & cluster init, **config & tuning** (shared_buffers/buffer pool/work_mem) + **connection pooling** (PgBouncer/ProxySQL), auth/**roles & GRANT**, networking/TLS/access control, **backups & PITR** (logical vs physical, WAL/binlog archiving — "untested backup isn't a backup"), **HA & replication** (streaming/logical, sync/async, failover with Patroni/Orchestrator, split-brain), **scaling/sharding**, **monitoring** (replication lag/cache hit/locks/slow queries), maintenance (**VACUUM**/bloat/stats), **security & hardening** (encryption/pgAudit/compliance), **upgrades & low-downtime migrations**, operating **MongoDB/Redis/SQL Server**, containers/cloud, and company-level practices — Postgres + MySQL/MariaDB primary, Mongo/Redis/MSSQL covered | 1614 |

### Version Control & DevOps
| Guide | What it covers | Lines |
|---|---|---|
| [Git](library/GIT_GUIDE.md) | **The mental model** (working dir/staging/repo, snapshots & the object model), commits & `.gitignore`, branching/merging & **conflict resolution**, remotes (`fetch` vs `pull`), **rebasing** (& the golden rule), **undoing things** (`reset`/`revert`/`reflog`), stashing, `bisect`/`blame`, tags, **collaboration workflows** & PRs, hooks/submodules/worktrees/LFS, `--force-with-lease`, removing secrets from history | 1895 |
| [CI/CD with GitHub Actions](library/GITHUB_ACTIONS_CICD_GUIDE.md) | CI vs Delivery vs Deployment, the workflow/job/step/runner model, **triggers** (incl. the `pull_request_target` landmine), contexts/expressions/`if:`, **matrix builds** & `needs` DAG, **caching & artifacts v4**, **secrets & security** (least-privilege `GITHUB_TOKEN`, **SHA-pinning**, **OIDC** keyless auth), reusable/composite actions, **environments & approvals**, worked **production pipelines** (Node/Go/Docker→GHCR), deployment strategies, hardening | 1988 |

### Shells & Scripting
| Guide | What it covers | Lines |
|---|---|---|
| [Bash Scripting (beginner→advanced)](library/BASH_SCRIPTING_GUIDE.md) | Bash 5.2+: quoting, variables & parameter expansion, conditionals/loops, functions & arrays, redirection & process substitution, `set -euo pipefail`/`trap`, grep/sed/awk/find/xargs, `getopts`, regex, robust-scripting recipes, portability/CRLF gotchas | 2440 |
| [Windows CMD & Batch (beginner→advanced)](library/WINDOWS_CMD_BATCH_GUIDE.md) | `cmd.exe` & `.bat`/`.cmd`: built-in commands, `set`/arithmetic/prompts, arg modifiers (`%~dp0`), `if`/`for` (all forms incl. `for /F`), **delayed expansion**, `call`/`goto` subroutines, redirection, errorlevels, string ops, escaping reference, system one-liners | 1957 |
| [PowerShell (beginner→advanced)](library/POWERSHELL_GUIDE.md) | 5.1 vs 7: the object pipeline, cmdlets & `Get-Member`, `Select`/`Where`/`ForEach-Object`, operators & regex, collections/hashtables, functions & `[CmdletBinding()]`, error handling, filesystem/registry providers, `PSCustomObject`/JSON/CSV, modules, `Invoke-RestMethod`, execution policy, 5.1↔7 encoding gotchas | 2091 |

---

## 🧭 Suggested Learning Tracks

Pick the track that matches your goal. Within a track, follow the order listed.

**① Foundations (a language + the web)**
Learn a language top-to-bottom: [Python](library/PYTHON_GUIDE.md) · [JavaScript](library/JAVASCRIPT_GUIDE.md) + [Node.js](library/NODEJS_GUIDE.md) · [Java](library/JAVA_GUIDE.md) · [Kotlin](library/KOTLIN_GUIDE.md) · [Go](library/GO_GUIDE.md) · [Rust](library/RUST_GUIDE.md) · [PHP](library/PHP_GUIDE.md). For systems/low-level, do [C](library/C_GUIDE.md) → [C++](library/CPP_GUIDE.md) (then [Rust](library/RUST_GUIDE.md) for memory safety). For the web, do [HTML](library/HTML_GUIDE.md) → [CSS](library/CSS_GUIDE.md) → [JavaScript](library/JAVASCRIPT_GUIDE.md) (incl. its DOM section).

**② Frontend (React ecosystem)**
[HTML](library/HTML_GUIDE.md) → [CSS](library/CSS_GUIDE.md) → [JavaScript](library/JAVASCRIPT_GUIDE.md) → 1. [React 19](library/REACT_19_GUIDE.md) → 2. [Tailwind](library/TAILWIND_CHEATSHEET.md) → 3. [Next.js 16](library/NEXTJS_16_GUIDE.md) → 4. [shadcn/ui](library/SHADCN_UI_CHEATSHEET.md) → 5. [React Hook Form](library/REACT_HOOK_FORM_GUIDE.md) → 6. [Motion](library/MOTION_ANIMATION_GUIDE.md) → 7. [TanStack Query](library/TANSTACK_QUERY_GUIDE.md) → 8. [Zustand](library/ZUSTAND_GUIDE.md) → 9. [Material UI](library/MATERIAL_UI_GUIDE.md). For motion, pick [Motion](library/MOTION_ANIMATION_GUIDE.md) (declarative, React-native) or [GSAP + React/Next.js](library/GSAP_GUIDE.md) (timeline/scroll-driven, `useGSAP()`).

**③ Full-stack with Node**
[JavaScript](library/JAVASCRIPT_GUIDE.md) → [Node.js](library/NODEJS_GUIDE.md) → [Next.js 16](library/NEXTJS_16_GUIDE.md) → [Fastify](library/FASTIFY_GUIDE.md) / [NestJS](library/NESTJS_GUIDE.md) → [PostgreSQL](library/POSTGRESQL_GUIDE.md) → [Prisma](library/PRISMA_ORM_GUIDE.md) → [Redis](library/REDIS_GUIDE.md) → [Docker](library/DOCKER_GUIDE.md)

**④ Backend with Go**
[Go — Language & Patterns](library/GO_LANG_AND_PATTERNS_GUIDE.md) → [Go net/http](library/GO_NET_HTTP_REST_API_GUIDE.md) → [Gin + uploads](library/GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) → [JWT + Argon2](library/GO_JWT_ARGON2_GUIDE.md) → [PostgreSQL](library/POSTGRESQL_GUIDE.md) → [ent ORM](library/GO_ENT_ORM_GUIDE.md) → [Redis](library/REDIS_GUIDE.md) → [Gorilla WebSockets](library/GO_GORILLA_WEBSOCKETS_GUIDE.md) → [gRPC & RPC](library/GO_GRPC_RPC_GUIDE.md) → [File System / OS / CLIs](library/GO_FILESYSTEM_OS_CLI_GUIDE.md) → [Docker](library/DOCKER_GUIDE.md)

**④a Backend with Python (FastAPI or Django)**
[Python](library/PYTHON_GUIDE.md) → choose: [FastAPI](library/FASTAPI_GUIDE.md) (async APIs, microservices) **or** [Django](library/DJANGO_GUIDE.md) + DRF (batteries-included apps + admin) → [Relational DB Design](library/RELATIONAL_DB_DESIGN_GUIDE.md) → [PostgreSQL](library/POSTGRESQL_GUIDE.md) → [Redis](library/REDIS_GUIDE.md) (cache/queues) → ship behind [Nginx](library/NGINX_GUIDE.md) in [Docker](library/DOCKER_GUIDE.md).

**④b Backend with Java (Spring Boot)**
[Java](library/JAVA_GUIDE.md) (or [Kotlin](library/KOTLIN_GUIDE.md)) → [Spring Boot](library/SPRING_BOOT_GUIDE.md) (REST → Spring Data JPA → Spring Security/JWT → testing with Testcontainers → Actuator) → [Relational DB Design](library/RELATIONAL_DB_DESIGN_GUIDE.md) → [PostgreSQL](library/POSTGRESQL_GUIDE.md) → [Redis](library/REDIS_GUIDE.md) → ship behind [Nginx](library/NGINX_GUIDE.md) in [Docker](library/DOCKER_GUIDE.md).

**⑤ Data layer & database design**
[Relational DB Design](library/RELATIONAL_DB_DESIGN_GUIDE.md) → [PostgreSQL](library/POSTGRESQL_GUIDE.md) → [Prisma](library/PRISMA_ORM_GUIDE.md) (Node) / [ent ORM](library/GO_ENT_ORM_GUIDE.md) (Go) → [SQLite3](library/SQLITE3_GUIDE.md) (embedded/local) → [MongoDB](library/MONGODB_GUIDE.md) (document) → [Redis](library/REDIS_GUIDE.md) (cache/KV)

**⑥ Auth & backend platforms**
[Better Auth](library/BETTERAUTH_GUIDE.md) (own your data: Next.js front + Go API) — or — [Supabase](library/SUPABASE_GUIDE.md) (managed Postgres + auth/realtime/storage, Next.js + Go). Pair either with [PostgreSQL](library/POSTGRESQL_GUIDE.md).

**⑦ Mobile**
Cross-platform: [React 19](library/REACT_19_GUIDE.md) → [React Native + Expo](library/REACT_NATIVE_EXPO_GUIDE.md). Native Android: [Kotlin](library/KOTLIN_GUIDE.md) → [Android (Android Studio)](library/ANDROID_STUDIO_GUIDE.md).

**⑧ Shells & scripting (automation)**
[Bash](library/BASH_SCRIPTING_GUIDE.md) (Linux/macOS/CI/Git Bash) · [PowerShell](library/POWERSHELL_GUIDE.md) (modern Windows + cross-platform automation) · [Windows CMD & Batch](library/WINDOWS_CMD_BATCH_GUIDE.md) (legacy/`.bat`, still everywhere). On Windows, learn PowerShell for new work and CMD for reading existing scripts.

**⑨ Version control, CI/CD & deployment (DevOps)**
[Networking](library/NETWORKING_GUIDE.md) (understand the wire first — IP/TCP/DNS/HTTP/TLS, the toolkit) → [Git](library/GIT_GUIDE.md) (master the model first) → [CI/CD with GitHub Actions](library/GITHUB_ACTIONS_CICD_GUIDE.md) (automate test/build/deploy) → [Docker](library/DOCKER_GUIDE.md) (package it) → [Nginx](library/NGINX_GUIDE.md) (reverse proxy / load balancer / API gateway in front of your app). Pair with [Bash](library/BASH_SCRIPTING_GUIDE.md) for pipeline scripting.

**⑩ Specialized**
[Go gRPC & RPC](library/GO_GRPC_RPC_GUIDE.md) — service-to-service APIs · [Go File System / OS / CLIs](library/GO_FILESYSTEM_OS_CLI_GUIDE.md) — tooling & scripts · [Rust](library/RUST_GUIDE.md) — systems/performance · [FTP Server (Go & Node)](library/FTP_SERVER_GO_AND_NODE_GUIDE.md) — file-transfer services.

**⑪ Server & systems administration (SysAdmin / DBA)**
Start with [Networking](library/NETWORKING_GUIDE.md) (the wire under everything), then pick your OS: [Linux Server Administration](library/LINUX_SERVER_ADMIN_GUIDE.md) (pair with [Bash](library/BASH_SCRIPTING_GUIDE.md)) and/or [Windows Server Administration](library/WINDOWS_SERVER_ADMIN_GUIDE.md) (pair with [PowerShell](library/POWERSHELL_GUIDE.md)). Then run the data tier with [Database Server Administration](library/DATABASE_SERVER_ADMIN_GUIDE.md) (pair with the engine guides: [PostgreSQL](library/POSTGRESQL_GUIDE.md) / [MongoDB](library/MONGODB_GUIDE.md) / [Redis](library/REDIS_GUIDE.md)). Put it in production behind [Nginx](library/NGINX_GUIDE.md), package with [Docker](library/DOCKER_GUIDE.md), and automate via [CI/CD](library/GITHUB_ACTIONS_CICD_GUIDE.md). This track takes you from first login to company-level operations.

**⑫ Full-stack with PHP & Laravel**
[PHP](library/PHP_GUIDE.md) (learn the modern, typed language first) → [Laravel](library/LARAVEL_GUIDE.md) (routing → Blade → the container/DI → **Eloquent** → validation → auth/Sanctum → APIs → queues/events → testing) → a database ([PostgreSQL](library/POSTGRESQL_GUIDE.md) / [SQLite3](library/SQLITE3_GUIDE.md), design with [Relational DB Design](library/RELATIONAL_DB_DESIGN_GUIDE.md)) → [Redis](library/REDIS_GUIDE.md) (cache/queues) → ship behind [Nginx + PHP-FPM](library/NGINX_GUIDE.md) in [Docker](library/DOCKER_GUIDE.md). For a JS frontend on Laravel's starter kits, see [React 19](library/REACT_19_GUIDE.md).

---

## ✅ Accuracy & Verification Notes

- **Written for 2026.** Fast-moving APIs are flagged inside each guide with **⚡ Version notes** (and **🆕 React 19** markers in the React guide). Always confirm exact package versions against official docs when you build.
- **Go code was syntax-verified (original six Go guides).** All ~230 Go code blocks across the original six Go guides were parsed with the Go toolchain (`gofmt`); one real bug (an invalid variadic parameter in the JWT guide) was found and fixed. Note: this verifies *syntax*, not full type-checking of examples that depend on third-party packages.
- **Newly added guides** — Redis, Fastify, PostgreSQL, Go File System/OS/CLI, Better Auth, Supabase, and Go gRPC/RPC — are written for 2026 accuracy with runnable, commented examples. Their **Go code was syntax-verified**: all 194 ` ```go ` blocks across these guides were parsed with the Go toolchain (`go/parser`, Go 1.26) — **0 syntax errors**. (Six blocks intentionally interleave top-level declarations with example usage statements in one snippet; each part is valid Go but they don't compile as a single unit — that's a teaching style, not a bug.) The **TypeScript/SQL/Deno** examples and exact third-party **API/symbol names** were *not* compiler-checked — confirm against official docs before production use. For fast-moving libraries (e.g. Better Auth plugins), inline notes flag where to double-check.
- **Shell/scripting guides were linted.** The **Bash** guide's 96 ` ```bash ` blocks were run through **ShellCheck 0.11** (`--severity=error`): 92 pass clean; the 4 flagged are deliberate anti-pattern demos (each marked `# WRONG`/`# BUG` — ShellCheck agreeing they're bad confirms the lesson). The **PowerShell** guide's 94 ` ```powershell ` blocks were parsed with the **PowerShell 7.6** parser and **PSScriptAnalyzer 1.25** (Error severity): **0 errors** (7-only syntax such as `?:`, `??`, `&&`, `?.` is valid under 7 and clearly flagged as 7-only in the guide). **Windows CMD/Batch** has no standard linter, so it was not auto-checked — its logic was hand-reviewed.
- **Security-critical code** (Argon2 params, JWT algorithm validation, file-upload validation, Supabase RLS, command-injection avoidance in `os/exec`) follows current best practices but should be reviewed against official guidance before production use.

---

*This library is a personal study + reference workspace. Each guide is standalone — start anywhere.*
