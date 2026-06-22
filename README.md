# 📚 Developer Study Library & Workspace

A self-contained, **offline-first** collection of in-depth technical guides plus a real demo project. Everything here is written to be learned and built from **without an internet connection** — each guide has concepts, runnable commented examples, reference tables, gotchas, and a study path, accurate **as of 2026**.

> **How to use this:** All guides live in [`library/`](library/). Pick a track below, open the guide, and work top-to-bottom. Each guide ends with a "Study Path" and build-to-learn projects.

---

## 🗂️ The Guides (`library/`)

### Frontend — React Ecosystem
| Guide | What it covers | Lines |
|---|---|---|
| [React 19](library/REACT_19_GUIDE.md) | Core + all React 19 APIs (`use`, Actions, `useActionState`, `useOptimistic`, ref-as-prop, document metadata, Compiler), migration from 18 | 2686 |
| [Next.js 16](library/NEXTJS_16_GUIDE.md) | App Router, Server/Client Components, rendering, caching, Server Actions, SEO, performance | 803 |
| [TanStack Query v5](library/TANSTACK_QUERY_GUIDE.md) | Server-state: `useQuery`/`useMutation`, caching, invalidation, optimistic updates, Next.js hydration | 612 |
| [Zustand](library/ZUSTAND_GUIDE.md) | Client state: stores, selectors, middleware (persist/immer/devtools), slices, Next.js SSR pattern | 1452 |
| [React Hook Form](library/REACT_HOOK_FORM_GUIDE.md) | `useForm`/`register`, validation, `Controller`, zod resolvers, `useFieldArray`, shadcn + Next.js integration | 1889 |
| [Motion (animation)](library/MOTION_ANIMATION_GUIDE.md) | `motion` components, transitions, gestures, variants, `AnimatePresence`, layout & scroll animations, Next.js usage | 2384 |
| [Tailwind CSS (v4)](library/TAILWIND_CHEATSHEET.md) | Full utility cheatsheet, variants, arbitrary values, dark mode, common combos | 487 |
| [shadcn/ui](library/SHADCN_UI_CHEATSHEET.md) | Component-by-component reference, props, the Form pattern, theming | 677 |
| [Material UI (MUI)](library/MATERIAL_UI_GUIDE.md) | Theming, `sx`/`styled`, v6 Grid, component reference, DataGrid, Next.js setup, dark mode | 2365 |

### Mobile
| Guide | What it covers | Lines |
|---|---|---|
| [React Native + Expo](library/REACT_NATIVE_EXPO_GUIDE.md) | Expo Router, core components, SDK device APIs, EAS Build/Submit/Update, animations, shipping to stores | 2791 |

### Backend — Node.js
| Guide | What it covers | Lines |
|---|---|---|
| [NestJS](library/NESTJS_GUIDE.md) | Modules, DI, controllers, pipes/guards/interceptors/filters, validation + **custom DTO validators**, Prisma/TypeORM, JWT auth, Swagger, WebSockets, testing | 3071 |
| [Fastify (v5)](library/FASTIFY_GUIDE.md) | Routing, full hook lifecycle, JSON-Schema validation/serialization, plugins & decorators, TypeScript type providers, core plugins (cors/jwt/multipart/swagger/websocket), Pino logging, testing with `inject()` | 2152 |
| [Prisma ORM](library/PRISMA_ORM_GUIDE.md) | Schema, relations, migrations, full CRUD/query API, transactions, raw queries, seeding, Next.js singleton | 1875 |

### Databases & Caching
| Guide | What it covers | Lines |
|---|---|---|
| [PostgreSQL (beginner→advanced)](library/POSTGRESQL_GUIDE.md) | PG 17: data types, DDL/DML, joins, CTEs & window functions, JSONB, indexing, `EXPLAIN ANALYZE`, transactions/MVCC, PL/pgSQL & triggers, partitioning, RLS, extensions (pgvector), `pg`/`pgx` clients | 2640 |
| [Redis](library/REDIS_GUIDE.md) | Redis 7/8: all data types, Streams, pub/sub, transactions & Lua, persistence, caching patterns, eviction, distributed locks & rate limiting, Sentinel/Cluster, ACLs/TLS, vector search, `redis-cli`/`ioredis`/`go-redis` | 2314 |

### Backend — Go
| Guide | What it covers | Lines |
|---|---|---|
| [Go `net/http` REST API](library/GO_NET_HTTP_REST_API_GUIDE.md) | Go 1.22+ routing, handlers, middleware, full CRUD API, context, graceful shutdown, testing | 1964 |
| [Go Gin — REST API & File Upload](library/GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) | Routing, binding/validation, middleware, CRUD, deep image-upload section, JWT, CORS, testing | 2130 |
| [Go Gorilla WebSockets](library/GO_GORILLA_WEBSOCKETS_GUIDE.md) | Upgrader, two-goroutine model, ping/pong, full Hub chat server, JS + Go clients, scaling | 1565 |
| [Go ent ORM](library/GO_ENT_ORM_GUIDE.md) | Schema-as-code, fields/edges, codegen, CRUD, predicates, eager loading, transactions, hooks | 1897 |
| [Go JWT + Argon2](library/GO_JWT_ARGON2_GUIDE.md) | Argon2id hashing (PHC, constant-time verify), golang-jwt v5, access/refresh tokens, auth middleware, RBAC | 1979 |
| [Go — File System, OS & CLIs](library/GO_FILESYSTEM_OS_CLI_GUIDE.md) | Reading/writing files, dirs & paths (`filepath`), `io/fs`/`embed`/`os.Root`, **running external commands** (`os/exec`: npm/go get/pip/git…), env vars, system info, signals, **building CLIs** (`flag`, Cobra, Viper), archives, cross-platform | 2980 |
| [gRPC & RPC with Go](library/GO_GRPC_RPC_GUIDE.md) | `net/rpc` basics, Protocol Buffers (proto3), buf toolchain, all 4 streaming types, metadata, status errors, interceptors (auth/logging), TLS/mTLS, deadlines/retries, gRPC-Gateway & ConnectRPC, reflection, bufconn testing | 2469 |

### Auth & Backend Platforms
| Guide | What it covers | Lines |
|---|---|---|
| [Better Auth (Standalone, Next.js & Go)](library/BETTERAUTH_GUIDE.md) | `betterAuth()` server & client, DB adapters, email/password & OAuth, sessions/cookies, plugins (2FA/passkeys/org/JWT/bearer), Next.js App Router integration, **verifying Better Auth JWTs in a Go backend** | 2039 |
| [Supabase (with Next.js & Go)](library/SUPABASE_GUIDE.md) | Postgres + Auth/Realtime/Storage/Edge Functions, **Row Level Security**, supabase-js query builder, `@supabase/ssr` for Next.js, CLI/migrations/types, **Go: direct `pgx` + verifying Supabase Auth JWTs**, pgvector | 2222 |

### Infrastructure & Protocols
| Guide | What it covers | Lines |
|---|---|---|
| [Docker & Docker Compose](library/DOCKER_GUIDE.md) | Images/containers, Dockerfiles, multi-stage builds, volumes, networking, Compose, real stacks, deployment | 802 |
| [FTP Server — Go & Node](library/FTP_SERVER_GO_AND_NODE_GUIDE.md) | FTP/FTPS/SFTP, Go (`ftpserverlib`) + Node (`ftp-srv`) servers, SFTP skeletons, firewall/Docker, security | 1709 |

### Shells & Scripting
| Guide | What it covers | Lines |
|---|---|---|
| [Bash Scripting (beginner→advanced)](library/BASH_SCRIPTING_GUIDE.md) | Bash 5.2+: quoting, variables & parameter expansion, conditionals/loops, functions & arrays, redirection & process substitution, `set -euo pipefail`/`trap`, grep/sed/awk/find/xargs, `getopts`, regex, robust-scripting recipes, portability/CRLF gotchas | 2440 |
| [Windows CMD & Batch (beginner→advanced)](library/WINDOWS_CMD_BATCH_GUIDE.md) | `cmd.exe` & `.bat`/`.cmd`: built-in commands, `set`/arithmetic/prompts, arg modifiers (`%~dp0`), `if`/`for` (all forms incl. `for /F`), **delayed expansion**, `call`/`goto` subroutines, redirection, errorlevels, string ops, escaping reference, system one-liners | 1957 |
| [PowerShell (beginner→advanced)](library/POWERSHELL_GUIDE.md) | 5.1 vs 7: the object pipeline, cmdlets & `Get-Member`, `Select`/`Where`/`ForEach-Object`, operators & regex, collections/hashtables, functions & `[CmdletBinding()]`, error handling, filesystem/registry providers, `PSCustomObject`/JSON/CSV, modules, `Invoke-RestMethod`, execution policy, 5.1↔7 encoding gotchas | 2091 |

---

## 🧭 Suggested Learning Tracks

Pick the track that matches your goal. Within a track, follow the order listed.

**① Frontend (React ecosystem)**
1. [React 19](library/REACT_19_GUIDE.md) → 2. [Tailwind](library/TAILWIND_CHEATSHEET.md) → 3. [Next.js 16](library/NEXTJS_16_GUIDE.md) → 4. [shadcn/ui](library/SHADCN_UI_CHEATSHEET.md) → 5. [React Hook Form](library/REACT_HOOK_FORM_GUIDE.md) → 6. [Motion](library/MOTION_ANIMATION_GUIDE.md) → 7. [TanStack Query](library/TANSTACK_QUERY_GUIDE.md) → 8. [Zustand](library/ZUSTAND_GUIDE.md) → 9. [Material UI](library/MATERIAL_UI_GUIDE.md)

**② Full-stack with Node**
[Next.js 16](library/NEXTJS_16_GUIDE.md) → [Fastify](library/FASTIFY_GUIDE.md) / [NestJS](library/NESTJS_GUIDE.md) → [PostgreSQL](library/POSTGRESQL_GUIDE.md) → [Prisma](library/PRISMA_ORM_GUIDE.md) → [Redis](library/REDIS_GUIDE.md) → [Docker](library/DOCKER_GUIDE.md)

**③ Backend with Go**
[Go net/http](library/GO_NET_HTTP_REST_API_GUIDE.md) → [Gin + uploads](library/GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) → [JWT + Argon2](library/GO_JWT_ARGON2_GUIDE.md) → [PostgreSQL](library/POSTGRESQL_GUIDE.md) → [ent ORM](library/GO_ENT_ORM_GUIDE.md) → [Redis](library/REDIS_GUIDE.md) → [Gorilla WebSockets](library/GO_GORILLA_WEBSOCKETS_GUIDE.md) → [gRPC & RPC](library/GO_GRPC_RPC_GUIDE.md) → [File System / OS / CLIs](library/GO_FILESYSTEM_OS_CLI_GUIDE.md) → [Docker](library/DOCKER_GUIDE.md)

**④ Data layer (databases & caching)**
[PostgreSQL](library/POSTGRESQL_GUIDE.md) → [Prisma](library/PRISMA_ORM_GUIDE.md) (Node) / [ent ORM](library/GO_ENT_ORM_GUIDE.md) (Go) → [Redis](library/REDIS_GUIDE.md)

**⑤ Auth & backend platforms**
[Better Auth](library/BETTERAUTH_GUIDE.md) (own your data: Next.js front + Go API) — or — [Supabase](library/SUPABASE_GUIDE.md) (managed Postgres + auth/realtime/storage, Next.js + Go). Pair either with [PostgreSQL](library/POSTGRESQL_GUIDE.md).

**⑥ Mobile**
[React 19](library/REACT_19_GUIDE.md) → [React Native + Expo](library/REACT_NATIVE_EXPO_GUIDE.md)

**⑦ Shells & scripting (automation)**
[Bash](library/BASH_SCRIPTING_GUIDE.md) (Linux/macOS/CI/Git Bash) · [PowerShell](library/POWERSHELL_GUIDE.md) (modern Windows + cross-platform automation) · [Windows CMD & Batch](library/WINDOWS_CMD_BATCH_GUIDE.md) (legacy/`.bat`, still everywhere). On Windows, learn PowerShell for new work and CMD for reading existing scripts.

**⑧ Specialized**
[Go gRPC & RPC](library/GO_GRPC_RPC_GUIDE.md) — service-to-service APIs · [Go File System / OS / CLIs](library/GO_FILESYSTEM_OS_CLI_GUIDE.md) — tooling & scripts · [FTP Server (Go & Node)](library/FTP_SERVER_GO_AND_NODE_GUIDE.md) — file-transfer services.

---

## ✅ Accuracy & Verification Notes

- **Written for 2026.** Fast-moving APIs are flagged inside each guide with **⚡ Version notes** (and **🆕 React 19** markers in the React guide). Always confirm exact package versions against official docs when you build.
- **Go code was syntax-verified (original six Go guides).** All ~230 Go code blocks across the original six Go guides were parsed with the Go toolchain (`gofmt`); one real bug (an invalid variadic parameter in the JWT guide) was found and fixed. Note: this verifies *syntax*, not full type-checking of examples that depend on third-party packages.
- **Newly added guides** — Redis, Fastify, PostgreSQL, Go File System/OS/CLI, Better Auth, Supabase, and Go gRPC/RPC — are written for 2026 accuracy with runnable, commented examples. Their **Go code was syntax-verified**: all 194 ` ```go ` blocks across these guides were parsed with the Go toolchain (`go/parser`, Go 1.26) — **0 syntax errors**. (Six blocks intentionally interleave top-level declarations with example usage statements in one snippet; each part is valid Go but they don't compile as a single unit — that's a teaching style, not a bug.) The **TypeScript/SQL/Deno** examples and exact third-party **API/symbol names** were *not* compiler-checked — confirm against official docs before production use. For fast-moving libraries (e.g. Better Auth plugins), inline notes flag where to double-check.
- **Shell/scripting guides were linted.** The **Bash** guide's 96 ` ```bash ` blocks were run through **ShellCheck 0.11** (`--severity=error`): 92 pass clean; the 4 flagged are deliberate anti-pattern demos (each marked `# WRONG`/`# BUG` — ShellCheck agreeing they're bad confirms the lesson). The **PowerShell** guide's 94 ` ```powershell ` blocks were parsed with the **PowerShell 7.6** parser and **PSScriptAnalyzer 1.25** (Error severity): **0 errors** (7-only syntax such as `?:`, `??`, `&&`, `?.` is valid under 7 and clearly flagged as 7-only in the guide). **Windows CMD/Batch** has no standard linter, so it was not auto-checked — its logic was hand-reviewed.
- **Security-critical code** (Argon2 params, JWT algorithm validation, file-upload validation, Supabase RLS, command-injection avoidance in `os/exec`) follows current best practices but should be reviewed against official guidance before production use.

---

*This library is a personal study + reference workspace. Each guide is standalone — start anywhere.*
