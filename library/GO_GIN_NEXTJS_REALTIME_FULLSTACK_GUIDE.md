# Go (Gin + Ent) Backend + Next.js Admin Dashboard — Full-Stack RBAC CRUD & Realtime — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Developers who can write *some* Go and *some* React/Next.js and now want to build **one complete decoupled full-stack system**: a **Go API service** (Gin + Ent ORM + Argon2id + JWT + Gorilla WebSockets, on a Supabase-hosted PostgreSQL) that a **separate Next.js 16 admin dashboard** consumes over **REST for CRUD** and a **WebSocket for live realtime updates** — with **role-based access control (RBAC)** enforced end-to-end. We build it around one concrete resource — **Courses** — and take it from an empty folder to a production-shaped app with create/read/update/delete, roles & permissions, a live-updating table, optimistic UI, and deployment. This is an **explain-first** guide: every concept is taught in prose (what it is, *why* the architecture pushes you this way, when and how to use it, the key options, best practices, and the **security implications**) before any heavily-commented, runnable code. Read it top-to-bottom once; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Go 1.25 / 1.26**, **`github.com/gin-gonic/gin` v1.10+**, **`entgo.io/ent` v0.14+**, **`github.com/gorilla/websocket` v1.5.x**, **`golang-jwt/jwt` v5**, **`golang.org/x/crypto/argon2`**, **Next.js 16 (App Router)** on **React 19**, **`@tanstack/react-query` v5**, **TypeScript 5.x**, and **Supabase (2026)** managed PostgreSQL — all current as of **June 2026**. Fast-moving APIs are flagged with **⚡ Version note**. The author is on **Windows 11**, so cross-platform notes (path separators, `.exe`, shells) are called out. Confirm exact symbol names against each library's dedicated guide and `pkg.go.dev` / official docs when online.
>
> **This guide's place in the library — it is the integration capstone for the Go+Next.js stack.** It does *not* re-teach each tool from scratch; it teaches you to **compose** them into one working system and shows the *integration glue* that the single-tool guides leave out (the CORS contract, the WS auth handshake from a browser, broadcasting a DB mutation to connected dashboards, mirroring backend RBAC in the UI). Wherever a topic has a dedicated guide, read it alongside:
> - [Go Gin — Production-Grade RESTful Backend](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) — the framework in depth (router, `*gin.Context`, middleware, binding).
> - [Go ent ORM](GO_ENT_ORM_GUIDE.md) — schema-as-code, codegen, edges, eager loading, hooks.
> - [Go JWT + Argon2 — Banking-Grade Auth](GO_JWT_ARGON2_GUIDE.md) — the password-hashing & token threat models in full.
> - [Go Gorilla WebSockets](GO_GORILLA_WEBSOCKETS_GUIDE.md) — the Upgrader, the one-reader/one-writer rule, the Hub, the Redis backplane.
> - [Next.js 16](NEXTJS_16_GUIDE.md) · [React 19](REACT_19_GUIDE.md) · [TanStack Query](TANSTACK_QUERY_GUIDE.md) · [React Hook Form](REACT_HOOK_FORM_GUIDE.md) · [shadcn/ui](SHADCN_UI_CHEATSHEET.md) — the frontend half.
> - [Supabase](SUPABASE_GUIDE.md) · [PostgreSQL](POSTGRESQL_GUIDE.md) · [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md) — the data tier.
> - [Networking](NETWORKING_GUIDE.md) · [Nginx](NGINX_GUIDE.md) · [Docker](DOCKER_GUIDE.md) — the edge & deployment.

---

## Table of Contents

1. [What We're Building & the Decoupled Architecture](#1-what-were-building--the-decoupled-architecture) **[B]**
2. [Prerequisites & The Monorepo Layout](#2-prerequisites--the-monorepo-layout) **[B]**
3. [Backend Foundation — Gin Engine, Config & Supabase Postgres](#3-backend-foundation--gin-engine-config--supabase-postgres) **[B]**
4. [Modeling the Domain with Ent — Users, Roles & Courses](#4-modeling-the-domain-with-ent--users-roles--courses) **[B/I]**
5. [Password Security with Argon2id & The Signup/Login Flow](#5-password-security-with-argon2id--the-signuplogin-flow) **[I/A]**
6. [JWT Access + Refresh Tokens & Auth Middleware](#6-jwt-access--refresh-tokens--auth-middleware) **[I/A]**
7. [RBAC — Roles, Permissions & Authorization Middleware](#7-rbac--roles-permissions--authorization-middleware) **[I/A]**
8. [Building the CRUD REST API for Courses](#8-building-the-crud-rest-api-for-courses) **[I]**
9. [CORS, the API Contract & Error Envelopes](#9-cors-the-api-contract--error-envelopes) **[I]**
10. [Realtime with Gorilla WebSockets — The Hub & Auth](#10-realtime-with-gorilla-websockets--the-hub--auth) **[A]**
11. [Broadcasting CRUD Events to the Dashboard](#11-broadcasting-crud-events-to-the-dashboard) **[A]**
12. [The Next.js Admin Dashboard — Setup & Architecture](#12-the-nextjs-admin-dashboard--setup--architecture) **[B/I]**
13. [Auth on the Frontend — Login, Token Storage & Route Protection](#13-auth-on-the-frontend--login-token-storage--route-protection) **[I/A]**
14. [Data Fetching & Mutations with TanStack Query](#14-data-fetching--mutations-with-tanstack-query) **[I]**
15. [The Course CRUD UI — Tables, Forms & Optimistic Updates](#15-the-course-crud-ui--tables-forms--optimistic-updates) **[I/A]**
16. [Consuming WebSocket Realtime in React](#16-consuming-websocket-realtime-in-react) **[A]**
17. [RBAC on the Frontend — Conditional UI & Route Guards](#17-rbac-on-the-frontend--conditional-ui--route-guards) **[I/A]**
18. [End-to-End Walkthrough: A Create Flows Through the Whole Stack](#18-end-to-end-walkthrough-a-create-flows-through-the-whole-stack) **[A]**
19. [Security Hardening (Full-Stack)](#19-security-hardening-full-stack) **[A]**
20. [Deployment — Go API + Next.js + Supabase](#20-deployment--go-api--nextjs--supabase) **[A]**
21. [Gotchas & Best Practices](#21-gotchas--best-practices) **[A]**
22. [Study Path & Build-to-Learn Projects](#22-study-path--build-to-learn-projects)

---

## 1. What We're Building & the Decoupled Architecture

### 1.1 The system in one picture **[B]**

We are building two **separately deployed** programs that talk to each other over the network:

```
┌─────────────────────────────┐         ┌──────────────────────────────────┐
│  Next.js 16 Admin Dashboard │         │   Go API Service (Gin)           │
│  (the browser app)          │         │                                  │
│                             │  HTTPS  │   ┌──────────────────────────┐   │
│  • Login page               │ ──REST──▶   │ Handlers (Gin routes)    │   │
│  • Course table (live)      │ ◀──JSON─┤   │ Auth + RBAC middleware   │   │
│  • Create/Edit/Delete forms │         │   │ Service layer            │   │
│  • TanStack Query cache      │         │   │ Ent ORM (type-safe SQL)  │   │
│  • useWebSocket hook         │  WSS    │   │ Argon2id + JWT           │   │
│                             │ ◀═realtime═  │ Gorilla WS Hub           │   │
└─────────────────────────────┘  events │   └────────────┬─────────────┘   │
                                         └────────────────┼─────────────────┘
                                                          │ pgx (TLS)
                                                          ▼
                                              ┌───────────────────────┐
                                              │  Supabase PostgreSQL  │
                                              │  (managed Postgres)   │
                                              └───────────────────────┘
```

Two channels connect the dashboard to the API:

1. **REST over HTTPS** — the request/response channel for everything the *user initiates*: log in, list courses, create a course, edit, delete. The browser asks, the server answers, the exchange ends. This is your CRUD.
2. **A WebSocket over WSS** — a single long-lived, full-duplex connection the server uses to **push** events the user did *not* ask for at that instant: "another admin just created course #42," "course #7 was deleted." This is what makes the table update live across every open dashboard without anyone hitting refresh.

The database (Supabase's managed Postgres) is reached **only by the Go service**. The browser never holds a database connection or a database credential — a non-negotiable security boundary we return to in [§19](#19-security-hardening-full-stack).

### 1.2 Why decoupled (two services) instead of one Next.js app? **[B]**

The library already has a [standalone full-stack Next.js capstone](NEXTJS_FULLSTACK_APP_GUIDE.md) where *one* Next.js codebase is both frontend and backend. That is the right default for many apps. So why does *this* guide split them?

You reach for a **separate Go API** when one or more of these is true — and an admin dashboard backed by a Go service typically hits several at once:

- **The backend already exists, or must be Go.** Your team standardized on Go for services (performance, a single static binary, strong concurrency). The dashboard is *a* client of that API, not the owner of the domain logic.
- **Multiple clients share one backend.** A web dashboard, a mobile app, a partner integration, and internal cron jobs all consume the same Courses API. A dedicated service with a versioned HTTP contract is cleaner than every client reaching into Next.js internals.
- **You need a long-lived process for realtime.** A Gorilla WebSocket **Hub** holding thousands of live connections is a persistent, stateful process. That does not fit the serverless/edge request model Next.js is often deployed under. A long-running Go binary is exactly the right home for it. (This is the single most common reason apps add a Go sidecar.)
- **Clear separation of concerns.** The frontend team ships UI on its own cadence; the backend team owns data integrity, auth, and the realtime fabric. The HTTP+WS contract between them is the only coupling.

The cost you accept in exchange: you now **own an explicit contract** between two programs. You must design REST shapes, handle **CORS** (the browser's cross-origin rules — [§9](#9-cors-the-api-contract--error-envelopes)), authenticate **both** the REST calls and the WebSocket upgrade ([§10](#10-realtime-with-gorilla-websockets--the-hub--auth)), and keep TypeScript types on the frontend in sync with Go structs on the backend. Most of this guide is about doing exactly that, correctly and securely.

> **Mental model to hold throughout:** the Go service is the **single source of truth and the only thing the database trusts**. The Next.js app is a *client* — a very capable, security-conscious client, but a client. Every rule that matters (who may delete a course, what a valid course looks like) is enforced on the **server**. The frontend mirrors those rules only to produce good UX (hide a button the user can't use); it never *is* the enforcement. We will repeat this until it is reflex.

### 1.3 Why this exact stack **[B]**

Each piece earns its place; here is the one-line justification and the guide that goes deep:

| Layer | Choice | Why (short) | Deep-dive guide |
|---|---|---|---|
| HTTP framework | **Gin** | Fast radix-tree router, middleware chain, binding/validation — the Go default for REST | [Gin guide](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) |
| ORM | **Ent** | Schema-as-Go-code + **codegen** → fully type-safe queries and edges; no hand-written SQL strings to get wrong | [Ent guide](GO_ENT_ORM_GUIDE.md) |
| Password hashing | **Argon2id** | Memory-hard, the 2015 Password Hashing Competition winner; current best practice | [JWT+Argon2 guide](GO_JWT_ARGON2_GUIDE.md) |
| Stateless auth | **JWT (access) + rotating refresh** | Access token in memory, refresh in an HttpOnly cookie; scales without server-side session lookups on every call | [JWT+Argon2 guide](GO_JWT_ARGON2_GUIDE.md) |
| Realtime | **Gorilla WebSocket** | Mature, stable v1 API; full-duplex; the Hub pattern fans one event out to many dashboards | [Gorilla guide](GO_GORILLA_WEBSOCKETS_GUIDE.md) |
| Database | **Supabase Postgres** | Managed Postgres with a real connection string; we use it as a plain SQL database from Go (pgx) | [Supabase guide](SUPABASE_GUIDE.md) |
| Frontend | **Next.js 16 + React 19** | App Router, Server/Client components, the React ecosystem for forms/tables | [Next.js 16](NEXTJS_16_GUIDE.md) |
| Server-state | **TanStack Query v5** | The cache between your UI and the REST API: dedupe, refetch, optimistic updates, invalidation | [TanStack Query](TANSTACK_QUERY_GUIDE.md) |

> **⚡ Version note — Supabase from Go:** Supabase's official client library (`supabase-js`) is JavaScript-first. From Go, the clean, production path is to **ignore the JS SDK and talk to Supabase's underlying Postgres directly** with `pgx` (Ent runs on top of `pgx`/`database/sql`). Supabase just *is* Postgres with a connection string; everything in the [Ent](GO_ENT_ORM_GUIDE.md) and [PostgreSQL](POSTGRESQL_GUIDE.md) guides applies unchanged. We use Supabase Auth's GoTrue *not at all* here — auth is owned by our Go service so it works identically for every client. See [§3.4](#34-connecting-to-supabase-postgres).

### 1.4 The data model we'll build, at a glance **[B]**

Three tables drive the whole guide. Keep this diagram in mind; we define it precisely in Ent in [§4](#4-modeling-the-domain-with-ent--users-roles--courses).

```
users                         courses                    (roles are an enum on users)
─────────                     ─────────                  ─────────────────────────────
id        (uuid, pk)          id        (uuid, pk)       role ∈ { admin, editor, viewer }
email     (unique)            title                       ── admin  : full CRUD + manage users
password_hash (argon2id)      description                 ── editor : create/read/update courses
role      (enum)              published (bool)            ── viewer : read-only
created_at                    author_id ──▶ users.id      
                              created_at / updated_at
```

- **`users`** authenticate and carry a **role**.
- **`courses`** are the CRUD resource. Each course has an **author** (a `user`) — an Ent **edge** (foreign key) that lets us enforce *row-level* rules later ("an editor may edit only their own courses").
- **Roles** (`admin` / `editor` / `viewer`) drive **RBAC**: the same Course endpoints behave differently depending on who calls them. This is the heart of [§7](#7-rbac--roles-permissions--authorization-middleware) and [§17](#17-rbac-on-the-frontend--conditional-ui--route-guards).

By the end you will have: signup/login issuing tokens, an authenticated and role-gated `/api/v1/courses` CRUD, every mutation broadcast over a WebSocket to all connected dashboards, and a Next.js admin UI whose table updates live, whose forms validate, and whose buttons appear or vanish based on the logged-in user's role — with the server enforcing every rule regardless of what the UI shows.

---

## 2. Prerequisites & The Monorepo Layout

### 2.1 What to install **[B]**

You need both toolchains because you are building two programs.

| Tool | Version (June 2026) | Check | Used for |
|---|---|---|---|
| **Go** | 1.25 / 1.26 | `go version` | the API service |
| **Node.js** | 22 / 24 LTS | `node -v` | the Next.js dashboard |
| **A Supabase project** | 2026 | dashboard → Project Settings → Database | managed Postgres + a connection string |
| **Git** | 2.45+ | `git --version` | version control |
| (optional) **Air** | latest | `air -v` | Go live-reload in dev (see [Gin guide §2.5](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md)) |

On Windows, run the Go commands in **PowerShell** and the Node commands wherever you like; this guide flags any path/shell differences. A Supabase free-tier project is enough — you only need its **connection string** (Settings → Database → Connection string → "URI", and separately the **pooler** string for production; [§3.4](#34-connecting-to-supabase-postgres)).

### 2.2 Repository layout — one repo, two apps **[B]**

We use a **monorepo**: one Git repository containing both the Go API and the Next.js dashboard in sibling folders. This is not mandatory (you could use two repos), but for a single team building one product it keeps the contract changes (a new field on a Course) reviewable in one pull request.

```
course-admin/                     ← the monorepo root (git init here)
├── api/                          ← the Go service  (module: github.com/you/course-admin/api)
│   ├── cmd/
│   │   └── server/
│   │       └── main.go           ← entrypoint: wire everything, start HTTP server
│   ├── ent/                      ← Ent: schema + GENERATED code (we edit ent/schema/*, ent gen writes the rest)
│   │   └── schema/
│   │       ├── user.go
│   │       └── course.go
│   ├── internal/                 ← all app code (compiler-enforced private — see Gin guide §8)
│   │   ├── config/               ← env loading
│   │   ├── server/               ← router assembly, middleware
│   │   ├── auth/                 ← argon2, jwt, the auth handlers
│   │   ├── rbac/                 ← roles, permissions, the authorize middleware
│   │   ├── course/              ← the Course handlers + service (the CRUD feature)
│   │   └── realtime/             ← the Gorilla WS hub + client pumps
│   ├── .env                      ← secrets (gitignored!)
│   ├── go.mod
│   └── go.sum
│
├── web/                          ← the Next.js dashboard
│   ├── src/
│   │   ├── app/                  ← App Router (login, dashboard, courses pages)
│   │   ├── components/           ← UI (table, forms, dialogs)
│   │   ├── lib/                  ← api client, auth context, ws hook, query client
│   │   └── types/                ← TypeScript types mirroring the Go DTOs
│   ├── .env.local                ← NEXT_PUBLIC_API_URL etc.
│   ├── package.json
│   └── next.config.ts
│
├── .gitignore                    ← ignores api/.env, web/.env.local, node_modules, api/tmp, etc.
└── README.md
```

> **Why `internal/` on the Go side?** Go's compiler enforces that packages under `internal/` can only be imported by code rooted at the parent directory. Putting all application code there means nothing leaks into other modules' public surface. The [Gin guide §8](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) compares the *layer-based* and *feature-based* layouts in depth; here we use a **feature-based** split (`auth/`, `rbac/`, `course/`, `realtime/`) because each feature is cohesive and the system is small enough to read at a glance.

### 2.3 Initialize both projects **[B]**

From the repo root, scaffold the Go module:

```bash
# --- The Go API ---
mkdir -p course-admin/api && cd course-admin/api
go mod init github.com/you/course-admin/api      # use YOUR module path
# Core dependencies we'll add as we go (shown here so you can pre-fetch):
go get github.com/gin-gonic/gin@latest
go get github.com/gin-contrib/cors@latest         # CORS middleware
go get entgo.io/ent/cmd/ent@latest                # Ent codegen CLI + runtime
go get github.com/jackc/pgx/v5/stdlib@latest      # Postgres driver (database/sql adapter for Ent)
go get github.com/golang-jwt/jwt/v5@latest        # JWT
go get golang.org/x/crypto/argon2@latest          # Argon2id (part of x/crypto)
go get github.com/gorilla/websocket@latest        # WebSockets
go get github.com/google/uuid@latest              # UUIDs
go get github.com/joho/godotenv@latest            # load .env in dev
```

Then the Next.js app, from the repo root:

```bash
# --- The Next.js dashboard ---  (run from course-admin/)
npx create-next-app@latest web --typescript --app --tailwind --eslint --src-dir --import-alias "@/*"
cd web
npm install @tanstack/react-query @tanstack/react-query-devtools  # server-state cache
npm install react-hook-form zod @hookform/resolvers               # forms + validation
npm install axios                                                 # HTTP client (interceptors for auth)
# shadcn/ui for the table/dialog/form primitives (optional but used in §15):
npx shadcn@latest init
```

> **Cross-platform note (Windows):** `mkdir -p` is a Bash-ism; in PowerShell use `New-Item -ItemType Directory -Force course-admin/api`. The Go and `npx` commands are identical across shells. Throughout the guide, ` ```bash ` blocks run as-is in Git Bash / WSL; the PowerShell equivalent is given when it differs.

### 2.4 The `.gitignore` (do this before your first commit) **[B]**

The single most common security incident in a project like this is committing the `.env` with your Supabase password and JWT secret. Prevent it on day one:

```gitignore
# --- secrets ---
api/.env
web/.env.local
*.env.local

# --- Go ---
api/tmp/            # Air's build output
api/server.exe
api/server

# --- Node / Next ---
node_modules/
web/.next/
web/out/

# --- misc ---
.DS_Store
```

> **Security recommendation:** treat the `.env` files as *credentials, not config*. They hold the database password, the JWT signing secret, and (in prod) cookie keys. They must never enter Git history — and if one ever does, **rotate every secret in it immediately**, because removing a file from the latest commit does not remove it from history. The [Git guide](GIT_GUIDE.md) covers purging secrets from history with `filter-repo`.

With both apps scaffolded and secrets ignored, we can build the backend foundation.

---

## 3. Backend Foundation — Gin Engine, Config & Supabase Postgres

We start the API service: load configuration, connect to Supabase Postgres through Ent, build a Gin engine with the baseline middleware, and serve a health check. This is the skeleton every later section hangs on.

### 3.1 Typed configuration from the environment **[B]**

**What & why.** Twelve-factor config says: read every environment-specific value (database URL, secrets, port, allowed origins) from the **environment**, never hard-code it. The benefit is that the *same binary* runs in dev, staging, and prod with different env vars — and secrets stay out of the source. We load a `.env` file in development for convenience (via `godotenv`) but in production the platform injects real environment variables and the `.env` file simply isn't present.

**How.** A single `Config` struct, populated once at startup, then passed explicitly to the things that need it. Parsing once and failing fast means a missing `JWT_SECRET` crashes the server *at boot* with a clear message — not at 3 a.m. when the first login arrives.

`api/internal/config/config.go`:

```go
package config

import (
	"fmt"
	"os"
	"strings"
	"time"

	"github.com/joho/godotenv"
)

// Config holds every environment-derived setting. Populated once in Load().
type Config struct {
	Env             string        // "development" | "production"
	Port            string        // e.g. "8080"
	DatabaseURL     string        // Supabase Postgres connection string (pooler in prod)
	JWTSecret       []byte        // HMAC signing key for access tokens
	AccessTokenTTL  time.Duration // short — e.g. 15m
	RefreshTokenTTL time.Duration // long  — e.g. 168h (7 days)
	AllowedOrigins  []string      // CORS allowlist, e.g. the dashboard's URL
}

// Load reads .env (dev only; ignored if absent) then the environment, validates,
// and returns a Config or an error. Fail fast on anything missing.
func Load() (*Config, error) {
	// In dev this populates os.Getenv from api/.env. In prod the file is absent
	// and the platform's real env vars are used — godotenv.Load just no-ops.
	_ = godotenv.Load()

	cfg := &Config{
		Env:             getenv("APP_ENV", "development"),
		Port:            getenv("PORT", "8080"),
		DatabaseURL:     os.Getenv("DATABASE_URL"),
		JWTSecret:       []byte(os.Getenv("JWT_SECRET")),
		AccessTokenTTL:  15 * time.Minute,
		RefreshTokenTTL: 7 * 24 * time.Hour,
		AllowedOrigins:  splitCSV(getenv("ALLOWED_ORIGINS", "http://localhost:3000")),
	}

	// Validate the security-critical values. A short or empty secret is a
	// production incident waiting to happen, so refuse to start.
	if cfg.DatabaseURL == "" {
		return nil, fmt.Errorf("DATABASE_URL is required")
	}
	if len(cfg.JWTSecret) < 32 {
		return nil, fmt.Errorf("JWT_SECRET must be at least 32 bytes (got %d)", len(cfg.JWTSecret))
	}
	return cfg, nil
}

func getenv(key, fallback string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return fallback
}

func splitCSV(s string) []string {
	parts := strings.Split(s, ",")
	out := make([]string, 0, len(parts))
	for _, p := range parts {
		if p = strings.TrimSpace(p); p != "" {
			out = append(out, p)
		}
	}
	return out
}
```

The matching `api/.env` for development (remember: gitignored):

```bash
APP_ENV=development
PORT=8080
# Supabase: Settings → Database → Connection string → URI (use the SESSION pooler for app traffic)
DATABASE_URL=postgresql://postgres.abcdefgh:YOUR-PASSWORD@aws-0-region.pooler.supabase.com:5432/postgres?sslmode=require
# Generate with:  openssl rand -base64 48   (must be ≥32 bytes after decoding)
JWT_SECRET=replace-with-a-long-random-secret-at-least-32-bytes-long
ALLOWED_ORIGINS=http://localhost:3000
```

> **Security recommendation:** generate `JWT_SECRET` with a CSPRNG (`openssl rand -base64 48`), never a memorable phrase. The check above enforces a 32-byte floor because HMAC-SHA256's security rests on the key being long and unguessable. Rotating it invalidates every issued access token (a feature, not a bug — it's your "log everyone out" lever).

### 3.2 What Gin gives us and the baseline engine **[B]**

**What Gin is (recap).** Gin wraps Go's standard `net/http` with a fast radix-tree router, a middleware chain (`Use`/`Next`/`Abort`), struct-tag request binding/validation, and a rich `*gin.Context`. A `*gin.Engine` *is* an `http.Handler`, so we hand it to a standard `http.Server` to control timeouts and graceful shutdown. The [Gin guide](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) covers all of this in depth; here we assemble just what the dashboard needs.

**The middleware baseline.** Every request flows through a chain before reaching your handler. Ours, in order:

1. **Recovery** — converts a panic in any handler into a clean `500` instead of crashing the process.
2. **Structured request logging** — one line per request (method, path, status, latency).
3. **Request ID** — a correlation ID per request, echoed in a header and attached to logs, so you can trace one request across the system.
4. **CORS** — the browser gate that lets the dashboard's origin call this API ([§9](#9-cors-the-api-contract--error-envelopes)).

`api/internal/server/server.go` (the router assembly — we fill in routes as features arrive):

```go
package server

import (
	"net/http"
	"time"

	"github.com/gin-contrib/cors"
	"github.com/gin-gonic/gin"
	"github.com/google/uuid"

	"github.com/you/course-admin/api/ent"
	"github.com/you/course-admin/api/internal/config"
)

// Server bundles everything the HTTP layer needs: the Gin engine, the Ent client
// (DB access), and the config. Handlers are methods on feature structs that close
// over these. We attach the realtime Hub in §10.
type Server struct {
	cfg    *config.Config
	db     *ent.Client
	engine *gin.Engine
}

func New(cfg *config.Config, db *ent.Client) *Server {
	if cfg.Env == "production" {
		gin.SetMode(gin.ReleaseMode) // silence debug logs, disable the debug warning
	}

	// gin.New() gives a bare engine with NO middleware (unlike gin.Default()),
	// so we attach exactly what we want, in the order we want.
	engine := gin.New()
	engine.Use(gin.Recovery())   // panic → 500, process stays alive
	engine.Use(requestID())      // correlation id (defined below)
	engine.Use(structuredLog())  // one log line per request (defined below)
	engine.Use(corsMiddleware(cfg))

	s := &Server{cfg: cfg, db: db, engine: engine}
	s.routes() // register all routes (grows as features are added)
	return s
}

// Handler exposes the engine as a standard http.Handler so main() can wrap it
// in an http.Server with real timeouts and graceful shutdown.
func (s *Server) Handler() http.Handler { return s.engine }

// requestID attaches a per-request UUID, stores it in the context, and echoes it.
func requestID() gin.HandlerFunc {
	return func(c *gin.Context) {
		id := c.GetHeader("X-Request-ID")
		if id == "" {
			id = uuid.NewString()
		}
		c.Set("request_id", id)
		c.Header("X-Request-ID", id)
		c.Next()
	}
}

// structuredLog logs method, path, status, latency and the request id AFTER the
// handler runs (code after c.Next() executes on the way back out of the chain).
func structuredLog() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		c.Next() // run the rest of the chain + handler
		gin.DefaultWriter.Write([]byte(
			time.Now().Format(time.RFC3339) + " " +
				c.Request.Method + " " + c.Request.URL.Path + " " +
				http.StatusText(c.Writer.Status()) + " " +
				time.Since(start).String() + "\n",
		))
	}
}

// corsMiddleware is defined in §9 — it's the cross-origin gate for the dashboard.
func corsMiddleware(cfg *config.Config) gin.HandlerFunc {
	return cors.New(cors.Config{
		AllowOrigins:     cfg.AllowedOrigins,
		AllowMethods:     []string{"GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"},
		AllowHeaders:     []string{"Origin", "Content-Type", "Authorization", "X-Request-ID"},
		ExposeHeaders:    []string{"X-Request-ID"},
		AllowCredentials: true, // required so the browser sends/receives the refresh cookie
		MaxAge:           12 * time.Hour,
	})
}
```

`api/internal/server/routes.go` (just the health check for now):

```go
package server

import "github.com/gin-gonic/gin"

// routes wires every endpoint. We add /auth, /courses, and /ws as we build them.
func (s *Server) routes() {
	// Liveness probe — no auth, no DB. Load balancers and uptime checks hit this.
	s.engine.GET("/healthz", func(c *gin.Context) {
		c.JSON(200, gin.H{"status": "ok"})
	})

	// Versioned API group. Versioning the URL (/api/v1) lets you ship a v2 later
	// without breaking existing clients — a core REST-contract discipline.
	v1 := s.engine.Group("/api/v1")
	_ = v1 // routes attached in later sections
}
```

> **Why `gin.New()` not `gin.Default()`?** `gin.Default()` pre-installs Gin's own logger and recovery. We want explicit control (a structured logger, a request-ID middleware, CORS in a deliberate order), so we start bare and `Use()` exactly our chain. This is the standard production choice.

### 3.3 The entrypoint: wire it together with graceful shutdown **[B/I]**

**Why graceful shutdown matters.** When you deploy a new version or the platform sends `SIGTERM`, you do not want to drop the requests currently in flight. Graceful shutdown stops accepting *new* connections, lets in-flight requests finish (up to a timeout), then exits. For an API serving a dashboard this is the difference between a clean redeploy and a user seeing a failed save.

`api/cmd/server/main.go`:

```go
package main

import (
	"context"
	"errors"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/you/course-admin/api/internal/config"
	"github.com/you/course-admin/api/internal/db"
	"github.com/you/course-admin/api/internal/server"
)

func main() {
	cfg, err := config.Load()
	if err != nil {
		log.Fatalf("config: %v", err) // fail fast — missing secret/URL
	}

	// Open the Ent client against Supabase Postgres (db.Open defined in §3.4).
	client, err := db.Open(cfg.DatabaseURL)
	if err != nil {
		log.Fatalf("database: %v", err)
	}
	defer client.Close()

	srv := server.New(cfg, client)

	// Wrap Gin in a real http.Server so we control timeouts. Timeouts are a
	// security control: they bound slow-loris attacks and stuck connections.
	httpServer := &http.Server{
		Addr:              ":" + cfg.Port,
		Handler:           srv.Handler(),
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       15 * time.Second,
		WriteTimeout:      15 * time.Second,
		IdleTimeout:       60 * time.Second,
	}

	// Start serving in a goroutine so main can wait for a shutdown signal.
	go func() {
		log.Printf("API listening on :%s (env=%s)", cfg.Port, cfg.Env)
		if err := httpServer.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			log.Fatalf("listen: %v", err)
		}
	}()

	// Block until we get SIGINT (Ctrl-C) or SIGTERM (deploy/stop).
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit
	log.Println("shutting down...")

	// Give in-flight requests up to 10s to finish, then force-close.
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	if err := httpServer.Shutdown(ctx); err != nil {
		log.Printf("forced shutdown: %v", err)
	}
	log.Println("bye")
}
```

> **Note on `WriteTimeout` and WebSockets:** a fixed `WriteTimeout` would kill a long-lived WebSocket connection. In [§10](#10-realtime-with-gorilla-websockets--the-hub--auth) we serve the WS endpoint in a way that isn't bounded by this timeout (the upgrade hijacks the connection), and we keep liveness with WebSocket ping/pong instead. Keep the timeouts here for the REST surface; they don't apply once a connection is upgraded.

### 3.4 Connecting to Supabase Postgres **[B/I]**

**What Supabase is to us.** Supabase is a hosted platform built around a real PostgreSQL database. It offers auth, storage, and realtime of its own — but because *our* backend is Go and owns auth itself, we use Supabase as **just a managed Postgres**: we take its connection string and connect with the standard `pgx` driver, exactly as the [PostgreSQL](POSTGRESQL_GUIDE.md) and [Ent](GO_ENT_ORM_GUIDE.md) guides describe. Nothing about Ent, pgx, or SQL changes because the host happens to be Supabase.

**Direct connection vs the pooler — read this carefully.** Supabase gives you two connection strings, and using the wrong one in production will exhaust your connections:

- **Direct connection** (`db.<ref>.supabase.co:5432`) — a real Postgres TCP connection. Fine for migrations and local dev. But Postgres tolerates only a few hundred connections, and serverless/many-instance deployments blow past that.
- **Connection pooler** (`...pooler.supabase.com:6543` for *transaction* mode, `:5432` for *session* mode) — Supavisor sits in front of Postgres and multiplexes many client connections onto a few real ones. **Use the pooler for your application traffic.** Use **session mode** (`:5432`) for a long-lived app like ours (it supports prepared statements, which pgx/Ent use); use transaction mode (`:6543`) for serverless functions.

`api/internal/db/db.go`:

```go
package db

import (
	"context"
	"database/sql"
	"fmt"
	"time"

	"entgo.io/ent/dialect"
	entsql "entgo.io/ent/dialect/sql"
	_ "github.com/jackc/pgx/v5/stdlib" // registers the "pgx" database/sql driver

	"github.com/you/course-admin/api/ent"
)

// Open connects to Supabase Postgres via pgx and returns an Ent client.
// Ent runs on top of database/sql here, so we get a *sql.DB to tune the pool.
func Open(databaseURL string) (*ent.Client, error) {
	sqlDB, err := sql.Open("pgx", databaseURL)
	if err != nil {
		return nil, fmt.Errorf("open: %w", err)
	}

	// Pool tuning. With the Supabase pooler in front, keep the app-side pool
	// modest — you are sharing the pooler's real connections with every instance.
	sqlDB.SetMaxOpenConns(10)               // ceiling on concurrent DB connections
	sqlDB.SetMaxIdleConns(5)                // keep a few warm
	sqlDB.SetConnMaxLifetime(30 * time.Minute) // recycle to avoid stale conns
	sqlDB.SetConnMaxIdleTime(5 * time.Minute)

	// Verify connectivity now, with a timeout, so a bad URL fails at boot.
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := sqlDB.PingContext(ctx); err != nil {
		return nil, fmt.Errorf("ping: %w", err)
	}

	// Hand the *sql.DB to Ent as a Postgres driver.
	drv := entsql.OpenDB(dialect.Postgres, sqlDB)
	client := ent.NewClient(ent.Driver(drv))
	return client, nil
}
```

> **⚡ Version note — SSL is mandatory.** Supabase requires TLS. Keep `?sslmode=require` in the connection string. For the strongest setting, download Supabase's CA certificate and use `sslmode=verify-full` (it prevents man-in-the-middle by validating the server certificate), pointing at the cert with `sslrootcert=`. The [PostgreSQL guide](POSTGRESQL_GUIDE.md) covers the `sslmode` ladder.

At this point `go run ./cmd/server` should boot, ping Supabase, and serve `GET /healthz`. The `ent` package it imports doesn't exist yet — that's next: we define the schema and generate it.

---

## 4. Modeling the Domain with Ent — Users, Roles & Courses

**Why Ent.** Ent is an ORM where **your schema is Go code**, and a code generator turns that schema into a fully **type-safe** client. Instead of writing SQL strings (easy to get wrong, invisible to the compiler) you write `client.Course.Query().Where(course.Published(true)).All(ctx)` and the compiler checks every field and predicate. Relations ("edges") are first-class, eager-loading is explicit (so you avoid N+1 by design), and migrations can be generated from the schema. The [Ent guide](GO_ENT_ORM_GUIDE.md) goes deep; here we define exactly the three-entity model from [§1.4](#14-the-data-model-well-build-at-a-glance).

### 4.1 The mental model: schema → codegen → typed client **[B]**

Ent has an unusual but powerful workflow:

1. You write **schema files** in `ent/schema/` — each describes one entity's **fields** and **edges** (relations).
2. You run **`go generate ./ent`** (which runs Ent's code generator).
3. Ent writes a large amount of generated Go into `ent/` — the client, builders, predicates, and per-entity packages. **You never edit generated files; you edit the schema and regenerate.**
4. You use the generated client in your services with full type safety.

So the loop is always: *edit `ent/schema/*.go` → `go generate ./ent` → use the new API.* Treat the generated code as a compiler artifact (it's committed, but machine-owned).

### 4.2 Bootstrapping Ent's generator **[B]**

Create the generator entrypoint `api/ent/generate.go`:

```go
package ent

//go:generate go run -mod=mod entgo.io/ent/cmd/ent generate ./schema
```

This `//go:generate` directive means "when I run `go generate`, execute Ent's generator against `./schema`." We'll run it after defining the schemas below.

### 4.3 The User schema — fields, the role enum, and edges **[B/I]**

**Design decisions encoded here:**
- The primary key is a **UUID** (`uuid.UUID`), not an auto-increment int. UUIDs don't leak row counts, are safe to expose in URLs, and can be generated client-side. We set a default generator.
- `email` is **unique** and indexed — it's the login identifier.
- `password_hash` stores the **Argon2id** PHC string ([§5](#5-password-security-with-argon2id--the-signuplogin-flow)). It is **`Sensitive()`** so Ent never serializes it (defense against accidentally returning it in JSON).
- `role` is an **enum** with three values — the spine of RBAC.
- A user **has many** courses (the `courses` edge), the inverse of a course belonging to an author.

`api/ent/schema/user.go`:

```go
package schema

import (
	"time"

	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/index"
	"github.com/google/uuid"
)

type User struct {
	ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.UUID("id", uuid.UUID{}).
			Default(uuid.New). // generate a v4 UUID on insert
			Immutable(),       // a primary key never changes

		field.String("email").
			NotEmpty().
			Unique(), // enforced by a unique index in Postgres

		// The Argon2id PHC string. Sensitive() keeps it out of any generated
		// String()/JSON output — it must never leave the server.
		field.String("password_hash").
			Sensitive().
			NotEmpty(),

		// The RBAC role. Values() defines the allowed enum strings; Postgres
		// stores it as text with a CHECK-like constraint enforced by Ent.
		field.Enum("role").
			Values("admin", "editor", "viewer").
			Default("viewer"), // safest default: new users can only read

		field.Time("created_at").
			Default(time.Now).
			Immutable(),
	}
}

// Edges (relations) of the User.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		// One user authors many courses. The matching inverse "author" edge is
		// declared on Course (§4.4); together they create the FK courses.author_id.
		edge.To("courses", Course.Type),
	}
}

// Indexes for performance/uniqueness beyond the field-level Unique().
func (User) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("email").Unique(),
	}
}
```

### 4.4 The Course schema — the CRUD resource and its author edge **[B/I]**

**Design decisions:**
- `title` is required and length-bounded (input hygiene at the schema level — a second line of defense after request validation).
- `published` is a boolean flag defaulting to `false` (drafts by default).
- `created_at` / `updated_at` are timestamps; `updated_at` uses `UpdateDefault` so Ent stamps it on every update automatically.
- The **`author`** edge is the inverse of `User.courses`. `Required()` means a course must always have an author; `Unique()` on this side means *each course has exactly one* author (a many-to-one). This produces the `author_id` foreign key — the thing we use for **row-level RBAC** ("editors edit only their own courses").

`api/ent/schema/course.go`:

```go
package schema

import (
	"time"

	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
	"github.com/google/uuid"
)

type Course struct {
	ent.Schema
}

func (Course) Fields() []ent.Field {
	return []ent.Field{
		field.UUID("id", uuid.UUID{}).
			Default(uuid.New).
			Immutable(),

		field.String("title").
			NotEmpty().
			MaxLen(200), // bound the size — cheap input hygiene

		field.Text("description").
			Optional(), // Text = unbounded; Optional = nullable

		field.Bool("published").
			Default(false),

		field.Time("created_at").
			Default(time.Now).
			Immutable(),

		field.Time("updated_at").
			Default(time.Now).
			UpdateDefault(time.Now), // re-stamped on every Save of an update
	}
}

func (Course) Edges() []ent.Edge {
	return []ent.Edge{
		// The inverse of User.courses. Ref("courses") links them. Unique() makes
		// it many-to-one (a course → one author). Required() forbids orphan courses.
		edge.From("author", User.Type).
			Ref("courses").
			Unique().
			Required(),
	}
}
```

### 4.5 Generate the client and run the schema migration **[B/I]**

Generate the type-safe client:

```bash
cd api
go generate ./ent      # runs Ent's codegen → fills ent/ with the client, builders, predicates
go mod tidy            # tidy up any deps the generated code pulls in
```

You now have `ent.Client`, `ent.User`, `ent.Course`, and packages like `ent/user` and `ent/course` (holding predicates such as `user.Email(...)`). Everything in [§3.4](#34-connecting-to-supabase-postgres) that referenced `ent` now compiles.

**Creating the tables.** Ent can create/alter the database schema to match your Go schema. For development, run **auto-migration** at startup; for production, prefer **versioned migrations** (Atlas) so changes are reviewable and reversible — see the [Ent guide](GO_ENT_ORM_GUIDE.md). The dev auto-migrate, added to `db.Open` or called from `main`:

```go
// Migrate creates/updates tables to match the Ent schema. DEV CONVENIENCE ONLY.
// In production use Atlas versioned migrations (ent/migrate) so every schema
// change is a reviewed, reversible artifact — auto-migrate can't drop columns
// safely and gives you no audit trail.
func Migrate(ctx context.Context, client *ent.Client) error {
	return client.Schema.Create(ctx)
}
```

Call it from `main` after `db.Open` while developing:

```go
if cfg.Env == "development" {
	if err := db.Migrate(context.Background(), client); err != nil {
		log.Fatalf("migrate: %v", err)
	}
}
```

After this runs, your Supabase database has `users` and `courses` tables with the right columns, the unique index on `email`, and the `author_id` foreign key.

> **Security recommendation — Supabase Row Level Security (RLS):** Supabase enables RLS thinking by default, but because *only our Go service* connects (with the Postgres role from the connection string, not anon/auth keys), our authorization is enforced in the **Go layer** (RBAC middleware, [§7](#7-rbac--roles-permissions--authorization-middleware)), not in Postgres RLS. That's the correct model for a trusted backend service. If you ever let the browser talk to Supabase directly (we don't), you would *also* need RLS policies. Keep the database password secret; it is the master key to all data. See the [Supabase guide](SUPABASE_GUIDE.md) for the RLS model in full.

### 4.6 A tiny seed: the first admin **[I]**

RBAC has a chicken-and-egg problem: someone must be the first admin, but only an admin can promote users. Solve it with a one-off seed that reads an email/password from the environment and creates an admin if none exists. We can only write this *after* password hashing exists ([§5](#5-password-security-with-argon2id--the-signuplogin-flow)), so we return to it there — but the shape is: "on boot in dev, if `users` is empty, create the admin from `SEED_ADMIN_EMAIL`/`SEED_ADMIN_PASSWORD`."

With the data layer modeled and generated, we can build authentication on top of it.

---

## 5. Password Security with Argon2id & The Signup/Login Flow

Authentication answers "who is this?" Before any RBAC ("what may they do?") can mean anything, we must store passwords safely and verify them. This section builds the password layer and the signup/login handlers that issue tokens. The [JWT + Argon2 guide](GO_JWT_ARGON2_GUIDE.md) covers the threat models exhaustively; here we implement the practical, banking-grade version for our service.

### 5.1 Why Argon2id and never anything else **[I]**

**The threat.** If your database is ever stolen, the attacker has every `password_hash`. A good hash makes reversing those hashes to plaintext passwords economically infeasible. The properties you need:

- **One-way:** you can verify a guess but never recover the password.
- **Salted:** each password gets a unique random **salt**, so identical passwords produce different hashes and precomputed "rainbow tables" are useless.
- **Slow & memory-hard:** deliberately expensive in CPU *and* memory, so an attacker with GPUs/ASICs can only try a few guesses per second per dollar, not billions.

**Why Argon2id specifically.** It won the 2015 Password Hashing Competition and is the current recommendation (OWASP, IETF RFC 9106). The `id` variant blends Argon2i (resistant to side-channel attacks) and Argon2d (resistant to GPU cracking) — the best general-purpose default. Never use fast hashes (MD5, SHA-256) for passwords — they're built to be fast, which is exactly wrong here. bcrypt is acceptable but caps at 72 bytes and isn't memory-hard; Argon2id is the better 2026 choice.

**The PHC string format.** We store a single self-describing string that encodes the algorithm, its parameters, the salt, and the hash:

```
$argon2id$v=19$m=65536,t=3,p=2$<base64-salt>$<base64-hash>
```

Storing the parameters *inside* the string means you can raise the cost over time and still verify old hashes — the verifier reads the parameters from the stored string, not from your current config. This is essential for long-lived systems.

### 5.2 Implementing hash & verify **[I/A]**

`api/internal/auth/password.go`:

```go
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

// argonParams are the cost parameters. These are sensible 2026 defaults; tune
// memory/time upward on stronger hardware (target ~50–100ms per hash on your box).
type argonParams struct {
	memory      uint32 // KiB of memory  (65536 = 64 MiB)
	iterations  uint32 // time cost (passes over memory)
	parallelism uint8  // lanes (threads)
	saltLength  uint32 // bytes
	keyLength   uint32 // bytes of output hash
}

var defaultParams = argonParams{
	memory:      64 * 1024, // 64 MiB
	iterations:  3,
	parallelism: 2,
	saltLength:  16,
	keyLength:   32,
}

var ErrInvalidHash = errors.New("invalid password hash format")
var ErrMismatch = errors.New("password does not match")

// HashPassword returns a PHC-format Argon2id string safe to store in the DB.
func HashPassword(password string) (string, error) {
	salt := make([]byte, defaultParams.saltLength)
	if _, err := rand.Read(salt); err != nil { // CSPRNG — never math/rand
		return "", fmt.Errorf("salt: %w", err)
	}

	hash := argon2.IDKey(
		[]byte(password), salt,
		defaultParams.iterations,
		defaultParams.memory,
		defaultParams.parallelism,
		defaultParams.keyLength,
	)

	// Encode parameters + salt + hash into the self-describing PHC string.
	b64 := base64.RawStdEncoding.EncodeToString
	return fmt.Sprintf(
		"$argon2id$v=%d$m=%d,t=%d,p=%d$%s$%s",
		argon2.Version,
		defaultParams.memory, defaultParams.iterations, defaultParams.parallelism,
		b64(salt), b64(hash),
	), nil
}

// VerifyPassword parses a stored PHC string, recomputes the hash of the candidate
// using the SAME parameters, and compares in constant time.
func VerifyPassword(password, encoded string) error {
	p, salt, hash, err := decodePHC(encoded)
	if err != nil {
		return err
	}

	// Recompute with the parameters that were used originally (read from the string).
	computed := argon2.IDKey([]byte(password), salt, p.iterations, p.memory, p.parallelism, p.keyLength)

	// subtle.ConstantTimeCompare avoids a timing side-channel: a naive ==
	// would return faster on an early-mismatching byte, leaking information.
	if subtle.ConstantTimeCompare(hash, computed) == 1 {
		return nil
	}
	return ErrMismatch
}

// decodePHC parses "$argon2id$v=19$m=..,t=..,p=..$salt$hash" back into its parts.
func decodePHC(encoded string) (argonParams, []byte, []byte, error) {
	var p argonParams
	parts := strings.Split(encoded, "$")
	if len(parts) != 6 || parts[1] != "argon2id" {
		return p, nil, nil, ErrInvalidHash
	}
	var version int
	if _, err := fmt.Sscanf(parts[2], "v=%d", &version); err != nil || version != argon2.Version {
		return p, nil, nil, ErrInvalidHash
	}
	if _, err := fmt.Sscanf(parts[3], "m=%d,t=%d,p=%d", &p.memory, &p.iterations, &p.parallelism); err != nil {
		return p, nil, nil, ErrInvalidHash
	}
	salt, err := base64.RawStdEncoding.DecodeString(parts[4])
	if err != nil {
		return p, nil, nil, ErrInvalidHash
	}
	hash, err := base64.RawStdEncoding.DecodeString(parts[5])
	if err != nil {
		return p, nil, nil, ErrInvalidHash
	}
	p.saltLength = uint32(len(salt))
	p.keyLength = uint32(len(hash))
	return p, salt, hash, nil
}
```

> **Security recommendations:** (1) Always use `crypto/rand`, never `math/rand`, for salts. (2) Always compare with `subtle.ConstantTimeCompare`. (3) Enforce a **minimum password length** (≥12 chars) at the request-validation layer ([§5.4](#54-the-signup--login-handlers)) — Argon2id protects stored hashes, but a "123456" password is weak regardless. (4) Return the **same** generic error for "no such email" and "wrong password" so you don't reveal which emails are registered (account enumeration). We do this in [§5.4](#54-the-signup--login-handlers).

### 5.3 What gets returned to the client: tokens, not sessions **[I]**

After verifying a password we must give the browser something to prove "I'm logged in" on subsequent requests. We use **JWTs** (built in [§6](#6-jwt-access--refresh-tokens--auth-middleware)), but the login handler's *shape* matters now:

- It issues a **short-lived access token** (15 min) returned in the JSON body. The dashboard holds it **in memory** and sends it as `Authorization: Bearer <token>` on every API call.
- It issues a **long-lived refresh token** (7 days) set as an **HttpOnly, Secure, SameSite cookie**. JavaScript cannot read it (XSS-resistant), and the browser sends it automatically only to our API. The dashboard calls `/auth/refresh` to mint a new access token when the old one expires.

We detail the *why* of this split (and why the access token is *not* a cookie) in [§6](#6-jwt-access--refresh-tokens--auth-middleware) and [§19](#19-security-hardening-full-stack). For now, the login handler returns `{ access_token, user }` and sets the refresh cookie.

### 5.4 The signup & login handlers **[I/A]**

We use **DTOs** (Data Transfer Objects): request structs with `binding` tags for validation, and response structs that expose only safe fields (never the password hash). The [Gin guide §5–6](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) covers binding/validation in depth.

`api/internal/auth/dto.go`:

```go
package auth

import "github.com/google/uuid"

// --- request DTOs (validated by Gin's binding tags) ---

type SignupRequest struct {
	Email    string `json:"email" binding:"required,email"`
	Password string `json:"password" binding:"required,min=12,max=128"`
}

type LoginRequest struct {
	Email    string `json:"email" binding:"required,email"`
	Password string `json:"password" binding:"required"`
}

// --- response DTOs (only safe fields; NEVER the password hash) ---

type UserResponse struct {
	ID    uuid.UUID `json:"id"`
	Email string    `json:"email"`
	Role  string    `json:"role"`
}

type AuthResponse struct {
	AccessToken string       `json:"access_token"`
	User        UserResponse `json:"user"`
}
```

`api/internal/auth/handler.go` (the HTTP handlers; the token-minting `issueTokens` is defined in [§6](#6-jwt-access--refresh-tokens--auth-middleware)):

```go
package auth

import (
	"net/http"

	"github.com/gin-gonic/gin"

	"github.com/you/course-admin/api/ent"
	"github.com/you/course-admin/api/ent/user"
)

// Handler holds the dependencies the auth endpoints need.
type Handler struct {
	db     *ent.Client
	tokens *TokenService // JWT issuing/parsing — built in §6
}

func NewHandler(db *ent.Client, tokens *TokenService) *Handler {
	return &Handler{db: db, tokens: tokens}
}

// Signup creates a viewer-by-default user. (In a closed admin tool you might
// disable open signup and only let admins create users — see §7.6.)
func (h *Handler) Signup(c *gin.Context) {
	var req SignupRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request", "detail": err.Error()})
		return
	}

	hash, err := HashPassword(req.Password)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "could not process password"})
		return
	}

	u, err := h.db.User.Create().
		SetEmail(req.Email).
		SetPasswordHash(hash).
		SetRole(user.RoleViewer). // generated enum constant — type-safe
		Save(c.Request.Context())
	if err != nil {
		// A unique-constraint violation means the email is taken. Return a
		// generic 409 — don't confirm/deny in a way that aids enumeration.
		if ent.IsConstraintError(err) {
			c.JSON(http.StatusConflict, gin.H{"error": "could not create account"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": "could not create account"})
		return
	}

	h.issueTokensAndRespond(c, u)
}

// Login verifies credentials and issues tokens.
func (h *Handler) Login(c *gin.Context) {
	var req LoginRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request"})
		return
	}

	u, err := h.db.User.Query().Where(user.EmailEQ(req.Email)).Only(c.Request.Context())
	if err != nil {
		// User not found. Return the SAME error as a bad password (below) so an
		// attacker can't tell which emails exist (account enumeration defense).
		c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid credentials"})
		return
	}

	if err := VerifyPassword(req.Password, u.PasswordHash); err != nil {
		c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid credentials"})
		return
	}

	h.issueTokensAndRespond(c, u)
}

// issueTokensAndRespond mints the access+refresh pair, sets the refresh cookie,
// and returns the access token + safe user fields.
func (h *Handler) issueTokensAndRespond(c *gin.Context, u *ent.User) {
	access, refresh, err := h.tokens.Issue(u) // defined in §6
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "could not issue tokens"})
		return
	}
	h.tokens.SetRefreshCookie(c, refresh) // HttpOnly/Secure/SameSite cookie — §6

	c.JSON(http.StatusOK, AuthResponse{
		AccessToken: access,
		User: UserResponse{ID: u.ID, Email: u.Email, Role: u.Role.String()},
	})
}
```

> **Note on `ShouldBindJSON` vs `BindJSON`:** the `ShouldBind*` family *returns* the error and leaves the response to you (so you control the status code and envelope); the `Bind*` family auto-writes a `400` and aborts. Always use `ShouldBind*` in production so your error shape is consistent ([§9](#9-cors-the-api-contract--error-envelopes)).

### 5.5 The first-admin seed (revisited) **[I]**

Now that hashing exists, the seed from [§4.6](#46-a-tiny-seed-the-first-admin) is implementable:

```go
// SeedAdmin creates an admin from env vars if the users table is empty. Dev/bootstrap
// convenience — in production, provision the first admin via a one-off migration or
// an out-of-band script, then disable this.
func SeedAdmin(ctx context.Context, db *ent.Client, email, password string) error {
	if email == "" || password == "" {
		return nil // nothing to seed
	}
	n, err := db.User.Query().Count(ctx)
	if err != nil || n > 0 {
		return err // already have users (or a query error) — don't seed
	}
	hash, err := HashPassword(password)
	if err != nil {
		return err
	}
	_, err = db.User.Create().
		SetEmail(email).
		SetPasswordHash(hash).
		SetRole(user.RoleAdmin).
		Save(ctx)
	return err
}
```

With identities established and tokens about to be minted, we build the token layer and the middleware that turns a `Bearer` header into an authenticated request.

---

## 6. JWT Access + Refresh Tokens & Auth Middleware

This section builds the `TokenService` referenced above: issuing a short access token and a long refresh token, validating them rigorously, the refresh-rotation endpoint, and the Gin middleware that protects routes. The [JWT + Argon2 guide](GO_JWT_ARGON2_GUIDE.md) is the deep reference; here is the working, secure implementation for our two-service setup.

### 6.1 What a JWT is and why we use the access/refresh split **[I]**

**A JWT** (JSON Web Token) is a signed, self-contained token: a base64url `header.payload.signature`. The **payload** carries *claims* (who the user is, their role, when it expires). The **signature** — HMAC-SHA256 over header+payload with our `JWT_SECRET` — lets the server verify the token wasn't tampered with **without a database lookup**. That statelessness is the point: the API can authenticate thousands of requests per second by verifying a signature, not by hitting Postgres each time.

**Why two tokens.** A JWT can't be revoked before it expires (it's self-contained), so there's tension between security and convenience:

- A **short-lived access token (15 min)** limits the blast radius if it leaks — it's useless soon. The dashboard sends it on every request.
- A **long-lived refresh token (7 days)**, stored in an HttpOnly cookie, lets the user stay logged in without re-entering a password. When the access token expires, the dashboard silently calls `/auth/refresh` to get a new one.

**Where each lives in the browser (critical):**

| Token | Stored where | Why |
|---|---|---|
| Access | **JavaScript memory** (a variable / React state) | Sent as `Authorization: Bearer`. In memory it dies on tab close and isn't in `localStorage` where any XSS could grab a long-lived credential. Short TTL caps the damage. |
| Refresh | **HttpOnly, Secure, SameSite cookie** | JS cannot read it (XSS can't steal it). The browser attaches it automatically *only* to our API origin. We rotate it on every use. |

> **The cardinal rule: never store the long-lived refresh token where JavaScript can read it (`localStorage`/`sessionStorage`).** That's the classic XSS-to-account-takeover path. HttpOnly cookie for the refresh; memory for the access. We re-justify this in [§13](#13-auth-on-the-frontend--login-token-storage--route-protection) and [§19](#19-security-hardening-full-stack).

### 6.2 The TokenService — issuing tokens **[I/A]**

`api/internal/auth/token.go`:

```go
package auth

import (
	"errors"
	"fmt"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/golang-jwt/jwt/v5"
	"github.com/google/uuid"

	"github.com/you/course-admin/api/ent"
)

// AccessClaims is the payload of the access token. We embed RegisteredClaims for
// the standard exp/iat/sub fields and add our own: the user's role (for RBAC).
type AccessClaims struct {
	Role string `json:"role"`
	jwt.RegisteredClaims
}

// RefreshClaims is intentionally minimal: just identify the user + a unique token
// id (jti) we could use to revoke a specific refresh token (see §6.5).
type RefreshClaims struct {
	jwt.RegisteredClaims
}

type TokenService struct {
	secret     []byte
	accessTTL  time.Duration
	refreshTTL time.Duration
	secureCookie bool   // true in production (HTTPS only)
	cookieDomain string // e.g. "" for localhost
}

func NewTokenService(secret []byte, accessTTL, refreshTTL time.Duration, secure bool, domain string) *TokenService {
	return &TokenService{
		secret: secret, accessTTL: accessTTL, refreshTTL: refreshTTL,
		secureCookie: secure, cookieDomain: domain,
	}
}

// Issue mints a fresh access+refresh pair for a user.
func (t *TokenService) Issue(u *ent.User) (access, refresh string, err error) {
	now := time.Now()

	accessTok := jwt.NewWithClaims(jwt.SigningMethodHS256, AccessClaims{
		Role: u.Role.String(),
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   u.ID.String(),                          // "sub" = the user id
			IssuedAt:  jwt.NewNumericDate(now),
			ExpiresAt: jwt.NewNumericDate(now.Add(t.accessTTL)),
			ID:        uuid.NewString(),                        // unique token id
		},
	})
	access, err = accessTok.SignedString(t.secret)
	if err != nil {
		return "", "", err
	}

	refreshTok := jwt.NewWithClaims(jwt.SigningMethodHS256, RefreshClaims{
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   u.ID.String(),
			IssuedAt:  jwt.NewNumericDate(now),
			ExpiresAt: jwt.NewNumericDate(now.Add(t.refreshTTL)),
			ID:        uuid.NewString(), // jti — the handle for revocation
		},
	})
	refresh, err = refreshTok.SignedString(t.secret)
	if err != nil {
		return "", "", err
	}
	return access, refresh, nil
}
```

### 6.3 Validating tokens — rigorous algorithm checking **[I/A]**

**The single most important JWT security rule:** when verifying, you must **assert the signing algorithm**. The classic attack: an attacker takes a valid token, changes the header to `alg: none` (or swaps HMAC for RSA to abuse your public key as an HMAC secret), and a naive verifier accepts it. golang-jwt v5 lets you pass a parser option that only accepts the algorithm you expect. **Always do this.**

```go
// parse validates a token string against `claims`, enforcing HS256 and expiry.
func (t *TokenService) parse(tokenStr string, claims jwt.Claims) error {
	_, err := jwt.ParseWithClaims(tokenStr, claims,
		func(token *jwt.Token) (interface{}, error) {
			// Reject anything that isn't HMAC. This defeats alg-confusion attacks.
			if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
			}
			return t.secret, nil
		},
		jwt.WithValidMethods([]string{"HS256"}), // belt-and-suspenders allowlist
		jwt.WithExpirationRequired(),            // a token with no exp is invalid
	)
	return err
}

// ParseAccess validates an access token and returns its claims.
func (t *TokenService) ParseAccess(tokenStr string) (*AccessClaims, error) {
	claims := &AccessClaims{}
	if err := t.parse(tokenStr, claims); err != nil {
		return nil, err
	}
	return claims, nil
}

// ParseRefresh validates a refresh token and returns its claims.
func (t *TokenService) ParseRefresh(tokenStr string) (*RefreshClaims, error) {
	claims := &RefreshClaims{}
	if err := t.parse(tokenStr, claims); err != nil {
		return nil, err
	}
	return claims, nil
}
```

### 6.4 The refresh cookie helpers **[I/A]**

```go
const refreshCookieName = "refresh_token"

// SetRefreshCookie writes the refresh token as a hardened cookie. The browser
// will send it automatically — and ONLY — to our API, on the /auth/refresh path.
func (t *TokenService) SetRefreshCookie(c *gin.Context, refresh string) {
	http.SetCookie(c.Writer, &http.Cookie{
		Name:     refreshCookieName,
		Value:    refresh,
		Path:     "/api/v1/auth", // scope: only auth endpoints receive it
		Domain:   t.cookieDomain,
		MaxAge:   int(t.refreshTTL.Seconds()),
		HttpOnly: true,                    // JS cannot read it → XSS can't steal it
		Secure:   t.secureCookie,          // HTTPS-only in production
		SameSite: http.SameSiteLaxMode,    // sent on top-level nav; see §19 for cross-site notes
	})
}

// ClearRefreshCookie expires the cookie on logout.
func (t *TokenService) ClearRefreshCookie(c *gin.Context) {
	http.SetCookie(c.Writer, &http.Cookie{
		Name: refreshCookieName, Value: "", Path: "/api/v1/auth",
		Domain: t.cookieDomain, MaxAge: -1, HttpOnly: true, Secure: t.secureCookie,
		SameSite: http.SameSiteLaxMode,
	})
}
```

> **⚡ Cross-site cookie note:** if your dashboard and API are on **different sites** (e.g. `app.example.com` and `api.example.com` are same-site; but `app.vercel.app` and `api.fly.dev` are cross-site), browsers will **not** send a `SameSite=Lax` cookie on the cross-site `fetch`. For cross-site you need `SameSite=None; Secure` — which then *requires* HTTPS and tightens CSRF considerations ([§19](#19-security-hardening-full-stack)). The cleanest deployment puts both behind one domain via a reverse proxy ([§20](#20-deployment--go-api--nextjs--supabase)) so cookies stay same-site. Decide this early; it shapes your cookie config.

### 6.5 The refresh & logout handlers — with rotation **[A]**

**Refresh rotation** means: every time a refresh token is used, you issue a *new* refresh token and invalidate the old one. This limits how long a stolen refresh token is useful and enables **reuse detection** — if an old (already-rotated) token is presented, something is wrong (likely theft), and you can revoke the whole family. Full rotation with a server-side token store and reuse detection is in the [JWT + Argon2 guide](GO_JWT_ARGON2_GUIDE.md); here we implement the practical version: validate the refresh cookie, issue a brand-new pair, and reset the cookie.

```go
// Refresh reads the refresh cookie, validates it, and issues a new token pair.
func (h *Handler) Refresh(c *gin.Context) {
	cookie, err := c.Cookie(refreshCookieName)
	if err != nil {
		c.JSON(http.StatusUnauthorized, gin.H{"error": "no refresh token"})
		return
	}
	claims, err := h.tokens.ParseRefresh(cookie)
	if err != nil {
		c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid refresh token"})
		return
	}

	uid, err := uuid.Parse(claims.Subject)
	if err != nil {
		c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid token subject"})
		return
	}
	// Re-load the user so the new access token reflects the CURRENT role (an admin
	// may have changed it). This is why refresh hits the DB but normal calls don't.
	u, err := h.db.User.Get(c.Request.Context(), uid)
	if err != nil {
		c.JSON(http.StatusUnauthorized, gin.H{"error": "user no longer exists"})
		return
	}

	h.issueTokensAndRespond(c, u) // mints a NEW pair and resets the cookie (rotation)
}

// Logout clears the refresh cookie. The access token will expire on its own;
// for instant global logout, rotate JWT_SECRET or maintain a denylist (§19).
func (h *Handler) Logout(c *gin.Context) {
	h.tokens.ClearRefreshCookie(c)
	c.JSON(http.StatusOK, gin.H{"status": "logged out"})
}
```

### 6.6 The authentication middleware **[I/A]**

This middleware runs before protected handlers. It extracts the `Bearer` token, validates it, and stores the user id + role in the Gin context for downstream handlers and the RBAC layer.

`api/internal/auth/middleware.go`:

```go
package auth

import (
	"net/http"
	"strings"

	"github.com/gin-gonic/gin"
	"github.com/google/uuid"
)

// Context keys — exported so other packages (rbac, course) can read them.
const (
	CtxUserID = "user_id"
	CtxRole   = "user_role"
)

// RequireAuth validates the access token and populates the context. Any route
// behind this middleware can assume a valid, authenticated user.
func (t *TokenService) RequireAuth() gin.HandlerFunc {
	return func(c *gin.Context) {
		header := c.GetHeader("Authorization")
		// Expect exactly "Bearer <token>".
		parts := strings.SplitN(header, " ", 2)
		if len(parts) != 2 || !strings.EqualFold(parts[0], "Bearer") {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "missing bearer token"})
			return
		}

		claims, err := t.ParseAccess(parts[1])
		if err != nil {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid or expired token"})
			return
		}

		uid, err := uuid.Parse(claims.Subject)
		if err != nil {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid token subject"})
			return
		}

		// Stash identity for handlers and the RBAC middleware.
		c.Set(CtxUserID, uid)
		c.Set(CtxRole, claims.Role)
		c.Next()
	}
}

// Helpers handlers use to read identity without re-parsing.
func UserID(c *gin.Context) uuid.UUID {
	v, _ := c.Get(CtxUserID)
	id, _ := v.(uuid.UUID)
	return id
}

func Role(c *gin.Context) string {
	v, _ := c.Get(CtxRole)
	r, _ := v.(string)
	return r
}
```

> **Why role from the token, not the DB, on every request?** Reading the role from the signed token keeps normal requests DB-free and fast. The cost: a role change only takes effect when the access token next refreshes (≤15 min). That trade-off is standard and acceptable; for instant revocation you add a denylist check ([§19](#19-security-hardening-full-stack)). The refresh endpoint *does* re-read the DB ([§6.5](#65-the-refresh--logout-handlers--with-rotation)), so role changes propagate at the next refresh automatically.

We now know **who** every request is. Next: **what they're allowed to do.**

---

## 7. RBAC — Roles, Permissions & Authorization Middleware

Authentication established identity; **authorization** decides what that identity may do. **RBAC** (Role-Based Access Control) models this as: users have **roles**, roles grant **permissions**, endpoints require permissions. This section builds a small, explicit permission system and the middleware that enforces it — the backend half of the RBAC story (the frontend mirror is [§17](#17-rbac-on-the-frontend--conditional-ui--route-guards)).

### 7.1 Roles, permissions, and why you separate them **[I]**

We have three roles. The naive approach checks the *role* directly in each handler (`if role == "admin"`). That works until requirements shift ("editors can now publish") and you're hunting through handlers. The better model is **permissions**: define granular capabilities, map roles to sets of them, and have endpoints require a *permission*, not a *role*. Adding a capability becomes a one-line map change.

Our permission set for Courses:

| Permission | admin | editor | viewer |
|---|:---:|:---:|:---:|
| `course:read` | ✅ | ✅ | ✅ |
| `course:create` | ✅ | ✅ | ❌ |
| `course:update` | ✅ | ✅ (own only*) | ❌ |
| `course:delete` | ✅ | ❌ | ❌ |
| `user:manage` (change roles, list users) | ✅ | ❌ | ❌ |

\* "own only" is a **row-level** rule that a coarse permission can't express by itself — an editor has `course:update` but only for courses they authored. We enforce the *permission* in middleware and the *ownership* in the service layer ([§7.4](#74-row-level-ownership-the-editor-owns-their-courses)). This two-layer split (coarse gate + fine ownership check) is the standard, robust pattern.

### 7.2 Modeling permissions in code **[I]**

`api/internal/rbac/rbac.go`:

```go
package rbac

// Permission is a granular capability. Strings keep it simple and debuggable.
type Permission string

const (
	CourseRead   Permission = "course:read"
	CourseCreate Permission = "course:create"
	CourseUpdate Permission = "course:update"
	CourseDelete Permission = "course:delete"
	UserManage   Permission = "user:manage"
)

// rolePermissions is the single source of truth mapping each role to its grants.
// Changing the policy = editing this map. Nothing else moves.
var rolePermissions = map[string]map[Permission]bool{
	"admin": {
		CourseRead: true, CourseCreate: true, CourseUpdate: true,
		CourseDelete: true, UserManage: true,
	},
	"editor": {
		CourseRead: true, CourseCreate: true, CourseUpdate: true,
	},
	"viewer": {
		CourseRead: true,
	},
}

// Can reports whether a role holds a permission.
func Can(role string, p Permission) bool {
	perms, ok := rolePermissions[role]
	if !ok {
		return false // unknown role → deny by default (fail closed)
	}
	return perms[p]
}

// PermissionsFor returns the flat list of a role's permissions — handy for the
// frontend (§17): the API can tell the UI exactly what to show.
func PermissionsFor(role string) []Permission {
	out := make([]Permission, 0)
	for p, granted := range rolePermissions[role] {
		if granted {
			out = append(out, p)
		}
	}
	return out
}
```

> **Design principle — fail closed.** `Can` returns `false` for any unknown role or unmapped permission. Authorization must default to *deny*: if you're ever unsure whether someone may do something, the safe answer is no. A system that defaults to *allow* leaks access every time you add a role or forget a mapping.

### 7.3 The authorization middleware **[I/A]**

This middleware runs *after* `RequireAuth` (so the role is already in context) and gates a route on a required permission.

`api/internal/rbac/middleware.go`:

```go
package rbac

import (
	"net/http"

	"github.com/gin-gonic/gin"

	"github.com/you/course-admin/api/internal/auth"
)

// Require returns middleware that allows the request only if the authenticated
// user's role holds the given permission. Use AFTER auth.RequireAuth().
func Require(p Permission) gin.HandlerFunc {
	return func(c *gin.Context) {
		role := auth.Role(c) // read from context (set by RequireAuth)
		if !Can(role, p) {
			// 403 Forbidden = "we know who you are, but you may not do this"
			// (distinct from 401 Unauthorized = "we don't know who you are").
			c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
				"error":    "forbidden",
				"required": string(p),
			})
			return
		}
		c.Next()
	}
}
```

The route wiring then reads almost like the permission table itself (full wiring in [§8.4](#84-wiring-the-routes-with-auth--rbac)):

```go
courses := v1.Group("/courses")
courses.Use(tokens.RequireAuth())                 // must be logged in for ALL course routes
{
	courses.GET("",     rbac.Require(rbac.CourseRead),   h.List)
	courses.GET("/:id", rbac.Require(rbac.CourseRead),   h.Get)
	courses.POST("",    rbac.Require(rbac.CourseCreate), h.Create)
	courses.PUT("/:id", rbac.Require(rbac.CourseUpdate), h.Update)  // + ownership check inside
	courses.DELETE("/:id", rbac.Require(rbac.CourseDelete), h.Delete)
}
```

> **401 vs 403 — get this right.** `401 Unauthorized` means *not authenticated* ("log in first"). `403 Forbidden` means *authenticated but not permitted* ("you're logged in, but this isn't for your role"). The dashboard treats them differently: a 401 triggers a token refresh or redirect to login; a 403 shows a "you don't have permission" message. Returning the wrong one creates confusing redirect loops.

### 7.4 Row-level ownership: the editor owns their courses **[I/A]**

The permission gate says "editors may update courses." It does *not* say "this editor may update *this* course." That finer rule is **row-level** and belongs in the **service layer**, where we have the actual course and can compare its `author_id` to the caller. Admins bypass it.

```go
// canModifyCourse enforces the row-level rule: admins may modify any course;
// editors only their own. Called by the Update/Delete service methods.
func canModifyCourse(role string, callerID, authorID uuid.UUID) bool {
	if role == "admin" {
		return true // admins transcend ownership
	}
	return callerID == authorID // editors: own courses only
}
```

We call this inside `Update`/`Delete` after loading the course ([§8.3](#83-the-course-handlers)). The result: a 403 if an editor tries to touch someone else's course — enforced on the server, regardless of what the UI showed.

> **Security recommendation — this is how you prevent IDOR.** **IDOR** (Insecure Direct Object Reference) is when a user accesses another user's object just by changing the id in the URL (`PUT /courses/<someone-elses-id>`). The coarse permission check won't stop it — only the ownership check does. *Every* endpoint that operates on a specific object owned by someone must verify ownership (or admin) server-side. Never rely on the UI hiding the object; the attacker calls the API directly.

### 7.5 The user-management endpoints (admin only) **[I]**

RBAC needs an admin surface to *change* roles. These sit behind `user:manage`:

```go
// PromoteRequest is the body for changing a user's role.
type PromoteRequest struct {
	Role string `json:"role" binding:"required,oneof=admin editor viewer"`
}

// UpdateUserRole lets an admin change any user's role. Behind rbac.Require(UserManage).
func (h *UserHandler) UpdateUserRole(c *gin.Context) {
	targetID, err := uuid.Parse(c.Param("id"))
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid user id"})
		return
	}
	var req PromoteRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request"})
		return
	}
	u, err := h.db.User.UpdateOneID(targetID).
		SetRole(user.Role(req.Role)). // validated by oneof above
		Save(c.Request.Context())
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "could not update role"})
		return
	}
	c.JSON(http.StatusOK, auth.UserResponse{ID: u.ID, Email: u.Email, Role: u.Role.String()})
}
```

> **Guardrail to add:** prevent an admin from demoting the *last* admin (count admins first; refuse if this would leave zero) — otherwise you can lock everyone out of admin. Also consider forbidding self-demotion. These are small `Count` checks before the update.

### 7.6 Exposing the caller's permissions to the frontend **[I]**

The dashboard needs to know the user's permissions to render the right UI ([§17](#17-rbac-on-the-frontend--conditional-ui--route-guards)). Add a `/auth/me` endpoint that returns the current user plus their permission list — the frontend's source of truth for conditional rendering (while the server stays the source of truth for *enforcement*):

```go
// Me returns the authenticated user and their permissions. The frontend uses
// the permissions to show/hide controls; the server still enforces them.
func (h *Handler) Me(c *gin.Context) {
	uid := auth.UserID(c)
	u, err := h.db.User.Get(c.Request.Context(), uid)
	if err != nil {
		c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
		return
	}
	c.JSON(http.StatusOK, gin.H{
		"user":        auth.UserResponse{ID: u.ID, Email: u.Email, Role: u.Role.String()},
		"permissions": rbac.PermissionsFor(u.Role.String()),
	})
}
```

With identity (who) and authorization (what) in place, we can finally build the CRUD the whole system exists to serve.

---

## 8. Building the CRUD REST API for Courses

Now the core: the create/read/update/delete endpoints for Courses. We organize this as a **handler → service** split — handlers deal with HTTP (parse, validate, status codes), the service deals with business logic (ownership rules, talking to Ent, later: emitting realtime events). This keeps each layer testable and keeps Gin out of the business logic. The [Gin guide §9](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) covers this architecture in depth.

### 8.1 REST design for the Course resource **[I]**

REST models the API as **resources** addressed by URLs, manipulated with HTTP **methods**, with **status codes** conveying outcome. Our contract:

| Method & path | Action | Permission | Success | Body in | Body out |
|---|---|---|---|---|---|
| `GET /api/v1/courses` | list (paginated) | `course:read` | 200 | — | `{ data: [...], pagination }` |
| `GET /api/v1/courses/:id` | read one | `course:read` | 200 | — | `{ data: course }` |
| `POST /api/v1/courses` | create | `course:create` | 201 | `{title, description, published}` | `{ data: course }` |
| `PUT /api/v1/courses/:id` | update | `course:update` (+own) | 200 | `{title, description, published}` | `{ data: course }` |
| `DELETE /api/v1/courses/:id` | delete | `course:delete` | 204 | — | — |

Conventions worth internalizing: **POST returns 201 Created** (a new resource came into being); **DELETE returns 204 No Content** (success, nothing to send back); **GET/PUT return 200**. These aren't pedantry — the dashboard's HTTP client branches on them ([§14](#14-data-fetching--mutations-with-tanstack-query)).

### 8.2 DTOs: separate wire shapes from the entity **[I]**

We never bind directly to the Ent entity or serialize it raw. **Request DTOs** define exactly what a client may set (a client must not set `id`, `author_id`, or timestamps). **Response DTOs** define exactly what we expose. This prevents *over-posting* (a client sneaking `author_id` to forge ownership) and *over-exposure* (leaking internal fields).

`api/internal/course/dto.go`:

```go
package course

import (
	"time"

	"github.com/google/uuid"

	"github.com/you/course-admin/api/ent"
)

// CreateRequest — only what a client may set on create.
type CreateRequest struct {
	Title       string `json:"title" binding:"required,min=1,max=200"`
	Description string `json:"description" binding:"max=5000"`
	Published   bool   `json:"published"`
}

// UpdateRequest — full replacement (PUT). Same shape as create here.
type UpdateRequest struct {
	Title       string `json:"title" binding:"required,min=1,max=200"`
	Description string `json:"description" binding:"max=5000"`
	Published   bool   `json:"published"`
}

// CourseResponse — exactly the fields we expose to clients. Note author is the
// id + email, never the author's password hash or internal fields.
type CourseResponse struct {
	ID          uuid.UUID `json:"id"`
	Title       string    `json:"title"`
	Description string    `json:"description"`
	Published   bool      `json:"published"`
	AuthorID    uuid.UUID `json:"author_id"`
	AuthorEmail string    `json:"author_email,omitempty"`
	CreatedAt   time.Time `json:"created_at"`
	UpdatedAt   time.Time `json:"updated_at"`
}

// toResponse maps an Ent entity to the wire DTO. If the author edge was eager-
// loaded, include the email; otherwise leave it blank (omitempty hides it).
func toResponse(c *ent.Course) CourseResponse {
	resp := CourseResponse{
		ID: c.ID, Title: c.Title, Description: c.Description,
		Published: c.Published, CreatedAt: c.CreatedAt, UpdatedAt: c.UpdatedAt,
	}
	if c.Edges.Author != nil {
		resp.AuthorID = c.Edges.Author.ID
		resp.AuthorEmail = c.Edges.Author.Email
	}
	return resp
}
```

### 8.3 The service layer **[I/A]**

The service owns the data logic and the ownership rule. It's where, in [§11](#11-broadcasting-crud-events-to-the-dashboard), we'll also emit realtime events — one place, so every mutation path is covered. It takes the Ent client (and later the Hub) as dependencies.

`api/internal/course/service.go`:

```go
package course

import (
	"context"
	"errors"

	"github.com/google/uuid"

	"github.com/you/course-admin/api/ent"
	"github.com/you/course-admin/api/ent/course"
	entuser "github.com/you/course-admin/api/ent/user"
)

var (
	ErrNotFound  = errors.New("course not found")
	ErrForbidden = errors.New("not allowed to modify this course")
)

type Service struct {
	db *ent.Client
	// hub *realtime.Hub  // added in §11 to broadcast events
}

func NewService(db *ent.Client) *Service {
	return &Service{db: db}
}

// List returns a page of courses, newest first, with the author eager-loaded
// (WithAuthor) so we avoid an N+1 query when mapping to DTOs.
func (s *Service) List(ctx context.Context, limit, offset int) ([]*ent.Course, int, error) {
	total, err := s.db.Course.Query().Count(ctx)
	if err != nil {
		return nil, 0, err
	}
	rows, err := s.db.Course.Query().
		WithAuthor().                       // eager-load the author edge (no N+1)
		Order(ent.Desc(course.FieldCreatedAt)).
		Limit(limit).
		Offset(offset).
		All(ctx)
	return rows, total, err
}

// Get returns one course (author loaded) or ErrNotFound.
func (s *Service) Get(ctx context.Context, id uuid.UUID) (*ent.Course, error) {
	c, err := s.db.Course.Query().Where(course.IDEQ(id)).WithAuthor().Only(ctx)
	if ent.IsNotFound(err) {
		return nil, ErrNotFound
	}
	return c, err
}

// Create inserts a course authored by the caller. The author is the AUTHENTICATED
// user — never a client-supplied field (that would let anyone forge authorship).
func (s *Service) Create(ctx context.Context, authorID uuid.UUID, req CreateRequest) (*ent.Course, error) {
	c, err := s.db.Course.Create().
		SetTitle(req.Title).
		SetDescription(req.Description).
		SetPublished(req.Published).
		SetAuthorID(authorID). // from the JWT, not the request body
		Save(ctx)
	if err != nil {
		return nil, err
	}
	// Re-load with the author edge so the response/event includes the email.
	return s.Get(ctx, c.ID)
	// §11: s.hub.Broadcast(eventCourseCreated, toResponse(full))
}

// Update replaces a course's fields after checking ownership (admins bypass).
func (s *Service) Update(ctx context.Context, callerID uuid.UUID, role string, id uuid.UUID, req UpdateRequest) (*ent.Course, error) {
	existing, err := s.Get(ctx, id)
	if err != nil {
		return nil, err // ErrNotFound
	}
	// Row-level authorization (the IDOR defense from §7.4).
	if !canModifyCourse(role, callerID, existing.Edges.Author.ID) {
		return nil, ErrForbidden
	}
	updated, err := s.db.Course.UpdateOneID(id).
		SetTitle(req.Title).
		SetDescription(req.Description).
		SetPublished(req.Published).
		Save(ctx)
	if err != nil {
		return nil, err
	}
	return s.Get(ctx, updated.ID)
	// §11: s.hub.Broadcast(eventCourseUpdated, toResponse(full))
}

// Delete removes a course after the same ownership check.
func (s *Service) Delete(ctx context.Context, callerID uuid.UUID, role string, id uuid.UUID) error {
	existing, err := s.Get(ctx, id)
	if err != nil {
		return err
	}
	if !canModifyCourse(role, callerID, existing.Edges.Author.ID) {
		return ErrForbidden
	}
	if err := s.db.Course.DeleteOneID(id).Exec(ctx); err != nil {
		return err
	}
	return nil
	// §11: s.hub.Broadcast(eventCourseDeleted, gin.H{"id": id})
}

// canModifyCourse: admins anything; editors only their own. (From §7.4.)
func canModifyCourse(role string, callerID, authorID uuid.UUID) bool {
	if role == "admin" {
		return true
	}
	return callerID == authorID
}

// ensure the import is used (editor role lookups, etc.)
var _ = entuser.Table
```

> **Why re-load after create/update (`s.Get`)?** Ent's `Create`/`Update` return the row but *not* its edges. We re-query `WithAuthor()` so the response (and the realtime event) carries the author's email. The extra query is cheap and keeps the response shape consistent with `List`/`Get`. For very hot paths you could set the edge manually to skip the round-trip.

### 8.4 The handlers **[I]**

Handlers translate HTTP ⇄ service calls and map service errors to status codes.

`api/internal/course/handler.go`:

```go
package course

import (
	"errors"
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"
	"github.com/google/uuid"

	"github.com/you/course-admin/api/internal/auth"
)

type Handler struct {
	svc *Service
}

func NewHandler(svc *Service) *Handler {
	return &Handler{svc: svc}
}

// List GET /courses?limit=20&offset=0
func (h *Handler) List(c *gin.Context) {
	limit := clampAtoi(c.Query("limit"), 20, 1, 100)
	offset := clampAtoi(c.Query("offset"), 0, 0, 1_000_000)

	rows, total, err := h.svc.List(c.Request.Context(), limit, offset)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "could not list courses"})
		return
	}
	out := make([]CourseResponse, len(rows))
	for i, r := range rows {
		out[i] = toResponse(r)
	}
	c.JSON(http.StatusOK, gin.H{
		"data": out,
		"pagination": gin.H{"total": total, "limit": limit, "offset": offset},
	})
}

// Get GET /courses/:id
func (h *Handler) Get(c *gin.Context) {
	id, err := uuid.Parse(c.Param("id"))
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid id"})
		return
	}
	course, err := h.svc.Get(c.Request.Context(), id)
	if errors.Is(err, ErrNotFound) {
		c.JSON(http.StatusNotFound, gin.H{"error": "course not found"})
		return
	}
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}
	c.JSON(http.StatusOK, gin.H{"data": toResponse(course)})
}

// Create POST /courses
func (h *Handler) Create(c *gin.Context) {
	var req CreateRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusUnprocessableEntity, gin.H{"error": "validation failed", "detail": err.Error()})
		return
	}
	created, err := h.svc.Create(c.Request.Context(), auth.UserID(c), req)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "could not create course"})
		return
	}
	c.JSON(http.StatusCreated, gin.H{"data": toResponse(created)}) // 201
}

// Update PUT /courses/:id
func (h *Handler) Update(c *gin.Context) {
	id, err := uuid.Parse(c.Param("id"))
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid id"})
		return
	}
	var req UpdateRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusUnprocessableEntity, gin.H{"error": "validation failed", "detail": err.Error()})
		return
	}
	updated, err := h.svc.Update(c.Request.Context(), auth.UserID(c), auth.Role(c), id, req)
	switch {
	case errors.Is(err, ErrNotFound):
		c.JSON(http.StatusNotFound, gin.H{"error": "course not found"})
	case errors.Is(err, ErrForbidden):
		c.JSON(http.StatusForbidden, gin.H{"error": "you may only edit your own courses"})
	case err != nil:
		c.JSON(http.StatusInternalServerError, gin.H{"error": "could not update course"})
	default:
		c.JSON(http.StatusOK, gin.H{"data": toResponse(updated)})
	}
}

// Delete DELETE /courses/:id
func (h *Handler) Delete(c *gin.Context) {
	id, err := uuid.Parse(c.Param("id"))
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid id"})
		return
	}
	err = h.svc.Delete(c.Request.Context(), auth.UserID(c), auth.Role(c), id)
	switch {
	case errors.Is(err, ErrNotFound):
		c.JSON(http.StatusNotFound, gin.H{"error": "course not found"})
	case errors.Is(err, ErrForbidden):
		c.JSON(http.StatusForbidden, gin.H{"error": "you may only delete your own courses"})
	case err != nil:
		c.JSON(http.StatusInternalServerError, gin.H{"error": "could not delete course"})
	default:
		c.Status(http.StatusNoContent) // 204, empty body
	}
}

// clampAtoi parses a query int with a default and min/max bounds (defensive paging).
func clampAtoi(s string, def, lo, hi int) int {
	n, err := strconv.Atoi(s)
	if err != nil {
		return def
	}
	if n < lo {
		return lo
	}
	if n > hi {
		return hi
	}
	return n
}
```

### 8.5 Wiring the routes with auth + RBAC **[I]**

Now `routes.go` assembles the protected Course group. The handler/service are constructed in `server.New` and stored on `Server` (omitted for brevity; follow the pattern from `auth`).

```go
func (s *Server) routes() {
	s.engine.GET("/healthz", func(c *gin.Context) { c.JSON(200, gin.H{"status": "ok"}) })

	v1 := s.engine.Group("/api/v1")

	// --- public auth endpoints ---
	authH := s.authHandler // *auth.Handler, built in server.New
	a := v1.Group("/auth")
	{
		a.POST("/signup", authH.Signup)
		a.POST("/login", authH.Login)
		a.POST("/refresh", authH.Refresh) // reads the refresh cookie
		a.POST("/logout", authH.Logout)
		a.GET("/me", s.tokens.RequireAuth(), authH.Me) // requires a valid access token
	}

	// --- protected course CRUD ---
	courseH := s.courseHandler // *course.Handler
	courses := v1.Group("/courses")
	courses.Use(s.tokens.RequireAuth()) // every course route requires auth
	{
		courses.GET("", rbac.Require(rbac.CourseRead), courseH.List)
		courses.GET("/:id", rbac.Require(rbac.CourseRead), courseH.Get)
		courses.POST("", rbac.Require(rbac.CourseCreate), courseH.Create)
		courses.PUT("/:id", rbac.Require(rbac.CourseUpdate), courseH.Update)
		courses.DELETE("/:id", rbac.Require(rbac.CourseDelete), courseH.Delete)
	}

	// --- admin-only user management ---
	userH := s.userHandler
	users := v1.Group("/users")
	users.Use(s.tokens.RequireAuth(), rbac.Require(rbac.UserManage))
	{
		users.GET("", userH.List)
		users.PATCH("/:id/role", userH.UpdateUserRole)
	}

	// WebSocket endpoint added in §10:  s.engine.GET("/ws", ...)
}
```

The backend CRUD is complete and secured. Before the frontend consumes it, we must make the browser *able* to call it — CORS — and standardize the error shape.

---

## 9. CORS, the API Contract & Error Envelopes

Two services on two origins introduce a browser-specific hurdle that doesn't exist in a single Next.js app: **CORS**. Getting it right (and understanding *why* it exists) is essential, and it's where most "it works in Postman but not the browser" confusion lives.

### 9.1 What CORS is and why the browser enforces it **[I]**

The browser's **Same-Origin Policy** blocks a page on `http://localhost:3000` (the dashboard) from reading responses from `http://localhost:8080` (the API) by default. An *origin* is the triple **scheme + host + port** — differ in any one and it's "cross-origin." This protects users: without it, a malicious site you visit could silently script requests to your bank's API using your logged-in cookies and read the responses.

**CORS** (Cross-Origin Resource Sharing) is the *server's* way to say "these specific other origins are allowed to read my responses." The server sends `Access-Control-Allow-Origin` (and friends) headers; the browser checks them and only then lets the page read the response. Crucially:

- CORS is enforced **by the browser**, for **JavaScript `fetch`/XHR**. `curl` and Postman ignore it — which is why your API call works in Postman but the dashboard sees a CORS error. The fix is always **server-side headers**, never a frontend change.
- For "non-simple" requests (anything with `Authorization`, a JSON content-type, or methods like PUT/DELETE), the browser first sends a **preflight** `OPTIONS` request asking "may I?" The server must answer it with the allowed methods/headers before the real request goes out.

### 9.2 Configuring CORS for the dashboard **[I]**

We already wired `gin-contrib/cors` in [§3.2](#32-what-gin-gives-us-and-the-baseline-engine). The configuration choices, explained:

```go
cors.New(cors.Config{
	// EXACT origins only — never "*" when credentials are involved. The browser
	// forbids wildcard origin + credentials together, and "*" would let any site
	// call your API. List your dashboard's real URL(s).
	AllowOrigins:     cfg.AllowedOrigins, // e.g. ["http://localhost:3000", "https://admin.example.com"]

	AllowMethods:     []string{"GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"},

	// Must include every custom header the browser will send. Authorization is
	// required for our Bearer tokens; without it, preflight fails.
	AllowHeaders:     []string{"Origin", "Content-Type", "Authorization", "X-Request-ID"},

	ExposeHeaders:    []string{"X-Request-ID"}, // headers JS is allowed to READ

	// REQUIRED for the refresh cookie to flow cross-origin. With this true, the
	// browser sends cookies AND the frontend must set `credentials: 'include'`.
	AllowCredentials: true,

	MaxAge:           12 * time.Hour, // cache preflight results to cut OPTIONS traffic
})
```

> **The two-sided contract for cookies.** For the refresh cookie to work cross-origin, **both** sides must opt in: the **server** sets `AllowCredentials: true` with an **exact** origin (not `*`), and the **frontend** sends requests with `credentials: 'include'` (axios: `withCredentials: true` — [§13](#13-auth-on-the-frontend--login-token-storage--route-protection)). Miss either and the cookie silently won't be sent or stored. This is the #1 cross-origin auth bug.

> **Security recommendation:** keep `AllowOrigins` an **explicit allowlist** from config — never reflect the request's `Origin` header back unconditionally (a common "fix" that effectively disables the protection). In production it should contain only your real dashboard URL(s) over HTTPS.

### 9.3 A consistent error envelope **[I]**

Ad-hoc error shapes (`{"error": ...}` here, `{"message": ...}` there) make the frontend's error handling brittle. Standardize on one envelope so the dashboard's HTTP layer can parse every error the same way. We've been using `{"error": "...", "detail": "..."}`; let's make it a documented contract:

```jsonc
// Error response — every non-2xx body looks like this:
{
  "error": "validation failed",        // short, stable, machine-readable-ish category
  "detail": "Title is required",       // optional human detail (safe to show or log)
  "request_id": "a1b2c3..."            // correlate with server logs
}
```

A tiny helper makes handlers consistent and lets us attach the request id automatically:

```go
// respondError writes the standard error envelope with the request id attached.
func respondError(c *gin.Context, status int, code, detail string) {
	rid, _ := c.Get("request_id")
	c.AbortWithStatusJSON(status, gin.H{
		"error":      code,
		"detail":     detail,
		"request_id": rid,
	})
}
```

> **Security recommendation — never leak internals in errors.** Return a generic message to the client (`"could not update course"`) and log the *real* error server-side with the request id. Echoing raw database or stack errors to the browser hands attackers a map of your internals (table names, driver versions, file paths). The `detail` field should carry *validation* feedback (safe, user-facing) — not exception text.

### 9.4 The TypeScript ⇄ Go contract **[I]**

Because the two services are separate, the **types must agree by convention** — Go structs on one side, TypeScript interfaces on the other. Keep a `web/src/types/api.ts` that mirrors the Go DTOs exactly:

```typescript
// web/src/types/api.ts — these MUST match the Go response DTOs (§8.2).

export type Role = "admin" | "editor" | "viewer";

export interface Course {
  id: string;            // uuid serializes to string
  title: string;
  description: string;
  published: boolean;
  author_id: string;
  author_email?: string; // omitempty on the Go side → optional here
  created_at: string;    // RFC3339 timestamp string
  updated_at: string;
}

export interface User {
  id: string;
  email: string;
  role: Role;
}

export interface AuthResponse {
  access_token: string;
  user: User;
}

export interface Paginated<T> {
  data: T[];
  pagination: { total: number; limit: number; offset: number };
}

export interface ApiError {
  error: string;
  detail?: string;
  request_id?: string;
}
```

> **Keeping them in sync.** For small projects, hand-maintaining these mirrors in one PR (the monorepo advantage) is fine. To eliminate drift entirely, generate the TypeScript from the Go types (e.g. `tygo`, or emit an OpenAPI spec from Gin and generate a typed client). That's optional; the discipline of updating both sides together is the minimum.

With the contract pinned and the browser able to call the API, only the realtime channel remains before we build the dashboard.

---

## 10. Realtime with Gorilla WebSockets — The Hub & Auth

REST is request/response: the client must ask to learn anything. For a *live* admin dashboard — where one admin's create should appear on every other admin's screen instantly — we need the **server to push**. That's a WebSocket: one long-lived, full-duplex TCP connection over which either side can send a message at any time. This section builds the connection-management core (the **Hub**) and, critically, **authenticates the WebSocket** — the part the single-tool guides under-emphasize and that's easy to get dangerously wrong. The [Gorilla WebSockets guide](GO_GORILLA_WEBSOCKETS_GUIDE.md) is the deep reference.

### 10.1 Why a WebSocket (and not polling) **[A]**

The dashboard could *poll* (`GET /courses` every 5 seconds) to stay fresh, but that's wasteful (most polls return nothing new), laggy (up to 5s stale), and scales badly (N dashboards × constant requests). The alternatives:

| Approach | Direction | Fit for our dashboard |
|---|---|---|
| Short polling | client pulls on a timer | wasteful; laggy; simple |
| **SSE** (Server-Sent Events) | **server → client only** | a great, simpler choice *if* you only push (we mostly do!) |
| **WebSocket** | **full-duplex** | the general answer; needed if the client also pushes over the same channel |

We use a **WebSocket** because it's the canonical tool, generalizes if the dashboard later needs to *send* over the socket (e.g. "who's online" presence), and the Hub pattern is reusable. (If your case is purely server→client notifications, SSE is legitimately simpler and rides plain HTTP — the Gorilla guide compares them. Our broadcast logic in [§11](#11-broadcasting-crud-events-to-the-dashboard) would be nearly identical either way.)

### 10.2 The Hub pattern — one place that owns all connections **[A]**

The core problem: many dashboards connect; when *one* course changes, we must send the event to *all* of them. We need a single owner of the set of live connections that's safe under concurrency (connections come and go on different goroutines while broadcasts fire). The **Hub** pattern solves this with a single goroutine owning the connection set and channels for register/unregister/broadcast — so no mutex-juggling of the map.

```
            register ──▶┐
          unregister ──▶│   ┌─────────── Hub goroutine ───────────┐
                        ├──▶│  clients: map[*Client]bool          │
   service.Broadcast ──▶│   │  select { register | unregister |   │
            (an event)  ┘   │           broadcast }               │
                            └──────────────┬──────────────────────┘
                                           │ fan-out to each client's send channel
                        ┌──────────────────┼──────────────────┐
                        ▼                  ▼                  ▼
                   Client A's          Client B's         Client C's
                   writePump           writePump          writePump  ──▶ browsers
```

Each connected browser is a `Client` with its own buffered `send` channel and two goroutines: a **readPump** (reads from the socket — mostly to detect close and handle pongs) and a **writePump** (the *only* writer to the socket). This honors Gorilla's hard rule: **one concurrent reader and one concurrent writer per connection — never more.** Violating it corrupts the stream.

`api/internal/realtime/hub.go`:

```go
package realtime

import (
	"encoding/json"
	"log"
)

// Event is the message shape pushed to every dashboard. The frontend switches on
// Type to update its cache (§16). Keep it small and JSON-friendly.
type Event struct {
	Type    string      `json:"type"`    // e.g. "course.created"
	Payload interface{} `json:"payload"` // the course DTO, or {id} for deletes
}

// Hub owns the set of live clients and serializes all access via channels, so the
// connection map is touched by exactly one goroutine (run()) — no locks needed.
type Hub struct {
	clients    map[*Client]bool
	register   chan *Client
	unregister chan *Client
	broadcast  chan []byte
}

func NewHub() *Hub {
	return &Hub{
		clients:    make(map[*Client]bool),
		register:   make(chan *Client),
		unregister: make(chan *Client),
		broadcast:  make(chan []byte, 256), // buffered: bursts of events don't block producers
	}
}

// Run is the Hub's single goroutine. Start it once at boot (go hub.Run()).
func (h *Hub) Run() {
	for {
		select {
		case c := <-h.register:
			h.clients[c] = true
			log.Printf("ws: client connected (total=%d)", len(h.clients))

		case c := <-h.unregister:
			if _, ok := h.clients[c]; ok {
				delete(h.clients, c)
				close(c.send) // signals the client's writePump to exit
				log.Printf("ws: client disconnected (total=%d)", len(h.clients))
			}

		case msg := <-h.broadcast:
			// Fan the message out to every client's send buffer. If a client's
			// buffer is full (a slow/stuck consumer), drop and disconnect it
			// rather than blocking the whole hub — backpressure safety (§10.5).
			for c := range h.clients {
				select {
				case c.send <- msg:
				default:
					close(c.send)
					delete(h.clients, c)
				}
			}
		}
	}
}

// Broadcast marshals an event and queues it for all clients. This is the method
// the course service calls after every mutation (§11). Safe to call from any
// goroutine — it just sends on a channel.
func (h *Hub) Broadcast(eventType string, payload interface{}) {
	msg, err := json.Marshal(Event{Type: eventType, Payload: payload})
	if err != nil {
		log.Printf("ws: marshal event: %v", err)
		return
	}
	select {
	case h.broadcast <- msg:
	default:
		log.Printf("ws: broadcast buffer full, dropping event %s", eventType)
	}
}
```

### 10.3 The Client — read/write pumps with heartbeats **[A]**

Each connection gets a `Client`. The **writePump** is the sole writer and also sends periodic **pings**; the **readPump** reads (discarding client messages in our push-only design) and handles **pongs** to keep the connection alive and detect dead peers.

**Why ping/pong?** TCP can stay "open" long after the peer vanished (laptop sleeps, network drops). Without a heartbeat, the server hoards dead connections. The server pings every ~54s; the client's browser auto-replies with a pong; if no pong arrives within the read deadline, the server concludes the client is gone and cleans up.

`api/internal/realtime/client.go`:

```go
package realtime

import (
	"time"

	"github.com/google/uuid"
	"github.com/gorilla/websocket"
)

const (
	writeWait      = 10 * time.Second    // max time to write one message
	pongWait       = 60 * time.Second    // max time to wait for a pong
	pingPeriod     = (pongWait * 9) / 10 // ping a bit before pongWait (54s)
	maxMessageSize = 1024                // cap inbound frames (we expect almost none)
)

// Client is one browser connection. userID/role let us do per-user targeting or
// authorization on the socket later; for now every authed user gets all events.
type Client struct {
	hub    *Hub
	conn   *websocket.Conn
	send   chan []byte // outbound queue; the Hub writes, writePump drains
	userID uuid.UUID
	role   string
}

// readPump is the SOLE reader. In our push-only design it mostly waits for the
// close/pong; it also enforces limits that defend against abusive clients.
func (c *Client) readPump() {
	defer func() {
		c.hub.unregister <- c // tell the hub we're gone
		c.conn.Close()
	}()

	c.conn.SetReadLimit(maxMessageSize)
	_ = c.conn.SetReadDeadline(time.Now().Add(pongWait))
	// Every pong from the browser extends the deadline — proof the peer is alive.
	c.conn.SetPongHandler(func(string) error {
		return c.conn.SetReadDeadline(time.Now().Add(pongWait))
	})

	for {
		// We don't expect inbound messages; ReadMessage blocks until one arrives
		// or the connection errors/closes. Any error breaks the loop → cleanup.
		if _, _, err := c.conn.ReadMessage(); err != nil {
			break
		}
		// (If you later accept client→server messages, validate & handle them here.)
	}
}

// writePump is the SOLE writer: it drains send and emits pings on a ticker.
func (c *Client) writePump() {
	ticker := time.NewTicker(pingPeriod)
	defer func() {
		ticker.Stop()
		c.conn.Close()
	}()

	for {
		select {
		case msg, ok := <-c.send:
			_ = c.conn.SetWriteDeadline(time.Now().Add(writeWait))
			if !ok {
				// Hub closed the channel → send a clean close frame and stop.
				_ = c.conn.WriteMessage(websocket.CloseMessage, []byte{})
				return
			}
			if err := c.conn.WriteMessage(websocket.TextMessage, msg); err != nil {
				return // write failed → peer gone → exit (defer cleans up)
			}

		case <-ticker.C:
			// Heartbeat. If the ping write fails, the peer is gone.
			_ = c.conn.SetWriteDeadline(time.Now().Add(writeWait))
			if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
				return
			}
		}
	}
}
```

### 10.4 Authenticating the WebSocket upgrade — the tricky part **[A]**

**The problem.** A browser's native `WebSocket` constructor **cannot set custom headers** — so you *cannot* send `Authorization: Bearer <token>` on the WS handshake the way you do for REST. Yet an unauthenticated realtime feed would leak every course event to anyone who connects. We must authenticate the upgrade some other way.

**The options, and what we choose:**

1. **Token in the query string** — `ws://host/ws?token=<access-token>`. Simple and works everywhere. Downside: URLs land in server logs and proxy logs, so a long-lived token there is a leak risk. Mitigate by using the **short-lived access token** (15 min) and ensuring TLS (the query string is encrypted on the wire under WSS).
2. **A short-lived "ticket"** — the browser calls an authenticated REST endpoint (`POST /ws-ticket`) to get a single-use, ~30-second token, then connects with `?ticket=...`. The leak window shrinks to seconds. This is the most secure common pattern.
3. **Cookie-based** — rely on the refresh/session cookie being sent on the upgrade (cookies *are* sent on WS handshakes to the same site). Clean, but reintroduces CSRF considerations and only works same-site.

We implement **option 1 with the access token** for clarity (it's the most common and is secure under WSS with a short TTL), and describe the **ticket upgrade** as the hardening step. Either way, **`CheckOrigin` must be enforced** to prevent **Cross-Site WebSocket Hijacking (CSWSH)** — the WS analog of CSRF, where a malicious page opens a socket to your API riding the user's ambient credentials. Gorilla's default `CheckOrigin` *rejects* cross-origin in browsers, but people often disable it with `return true` — **never ship that.**

`api/internal/realtime/handler.go`:

```go
package realtime

import (
	"net/http"

	"github.com/gin-gonic/gin"
	"github.com/google/uuid"
	"github.com/gorilla/websocket"
)

// TokenParser is the minimal interface the WS handler needs from the auth layer
// (decouples realtime from the concrete TokenService).
type TokenParser interface {
	ParseAccessSubjectRole(token string) (userID uuid.UUID, role string, err error)
}

// newUpgrader builds the Upgrader with a STRICT origin allowlist (CSWSH defense).
func newUpgrader(allowedOrigins []string) websocket.Upgrader {
	allow := make(map[string]bool, len(allowedOrigins))
	for _, o := range allowedOrigins {
		allow[o] = true
	}
	return websocket.Upgrader{
		ReadBufferSize:  1024,
		WriteBufferSize: 1024,
		// CheckOrigin gates who may open a socket. Compare against our exact
		// allowlist — NEVER `return true`. Origin is reliable from browsers.
		CheckOrigin: func(r *http.Request) bool {
			return allow[r.Header.Get("Origin")]
		},
	}
}

// ServeWS authenticates the request, upgrades it, and registers the client.
// Mounted at GET /ws. Auth is via ?token=<access-token> because browsers can't
// set Authorization on the WS handshake (§10.4).
func ServeWS(hub *Hub, parser TokenParser, allowedOrigins []string) gin.HandlerFunc {
	upgrader := newUpgrader(allowedOrigins)
	return func(c *gin.Context) {
		// 1) Authenticate BEFORE upgrading. Reject with plain HTTP if invalid —
		//    cheaper and clearer than upgrading then closing.
		token := c.Query("token")
		if token == "" {
			c.JSON(http.StatusUnauthorized, gin.H{"error": "missing token"})
			return
		}
		userID, role, err := parser.ParseAccessSubjectRole(token)
		if err != nil {
			c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid token"})
			return
		}

		// 2) Upgrade the HTTP connection to a WebSocket.
		conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
		if err != nil {
			return // Upgrade already wrote an error response
		}

		// 3) Build the client and start its pumps.
		client := &Client{
			hub: hub, conn: conn, send: make(chan []byte, 256),
			userID: userID, role: role,
		}
		hub.register <- client
		go client.writePump()
		go client.readPump()
	}
}
```

Add the small adapter on `TokenService` so it satisfies `TokenParser`:

```go
// ParseAccessSubjectRole validates an access token and returns the user id + role.
// Lets the realtime package authenticate sockets without importing handler types.
func (t *TokenService) ParseAccessSubjectRole(token string) (uuid.UUID, string, error) {
	claims, err := t.ParseAccess(token)
	if err != nil {
		return uuid.Nil, "", err
	}
	uid, err := uuid.Parse(claims.Subject)
	if err != nil {
		return uuid.Nil, "", err
	}
	return uid, claims.Role, nil
}
```

Wire the route (it sits *outside* the REST timeout concerns from [§3.3](#33-the-entrypoint-wire-it-together-with-graceful-shutdown) because the upgrade hijacks the connection):

```go
// in routes(): the WS endpoint. Auth happens inside ServeWS via the query token,
// NOT via RequireAuth middleware (which expects an Authorization header).
s.engine.GET("/ws", realtime.ServeWS(s.hub, s.tokens, s.cfg.AllowedOrigins))
```

And start the hub once, in `server.New` or `main`:

```go
hub := realtime.NewHub()
go hub.Run() // the single goroutine that owns all connections
```

> **Security checklist for the WS endpoint:** (1) **authenticate before upgrading**; (2) enforce **`CheckOrigin`** against an allowlist (CSWSH); (3) use **WSS** (TLS) in production so the `?token=` query isn't visible on the wire; (4) prefer the **short-lived access token** or a **single-use ticket** over any long-lived credential in the URL; (5) set a **read limit** and **deadlines** (done in the pumps) so a malicious client can't exhaust memory or hold a dead connection. Items 1–2 are non-negotiable.

### 10.5 Backpressure: protecting the server from slow clients **[A]**

A dashboard on a slow network might not drain its messages fast enough. If the Hub blocked waiting for it, *one* slow client would freeze realtime for *everyone*. Our defense (already in [§10.2](#102-the-hub-pattern--one-place-that-owns-all-connections)): each client has a **buffered** `send` channel, and on broadcast we do a **non-blocking send** (`select { case c.send <- msg: default: <drop & disconnect> }`). A client that can't keep up is forcibly disconnected; it'll reconnect ([§16](#16-consuming-websocket-realtime-in-react)) and refetch current state. **Never let a slow consumer apply backpressure to a shared broadcaster.** The Gorilla guide's §10 covers slow-client strategies in depth.

> **⚡ Scaling note — multiple API instances.** The Hub lives *in one process*. If you run **two** API instances behind a load balancer, an event created on instance A won't reach dashboards connected to instance B — each has its own Hub. The fix is a **Redis Pub/Sub backplane**: each instance publishes events to Redis and subscribes to them, so a broadcast fans out across all instances. This is exactly the pattern in the [Gorilla guide](GO_GORILLA_WEBSOCKETS_GUIDE.md) §13 and the [Node WebSockets guide](NODE_WEBSOCKETS_GUIDE.md). For a single-instance admin tool you don't need it yet — but know it's the next step when you scale out, and design `Broadcast` (which we did) as the one chokepoint to route through Redis later.

The realtime fabric exists; now we make CRUD mutations *use* it.

---

## 11. Broadcasting CRUD Events to the Dashboard

The Hub can fan out events; the Course service performs mutations. This short but pivotal section connects them: **every create/update/delete broadcasts an event**, so all connected dashboards update live. This is the payoff that makes the app feel realtime.

### 11.1 Injecting the Hub into the service **[A]**

We give the `course.Service` a reference to the Hub (dependency injection — the service depends on an interface, not the concrete Hub, so it stays testable). Define a tiny interface in the course package so it doesn't import realtime's internals:

```go
// Broadcaster is the slice of the Hub the course service needs. Depending on an
// interface (not *realtime.Hub) keeps the service unit-testable with a fake.
type Broadcaster interface {
	Broadcast(eventType string, payload interface{})
}
```

Update the service constructor and struct ([§8.3](#83-the-service-layer)):

```go
type Service struct {
	db  *ent.Client
	bus Broadcaster
}

func NewService(db *ent.Client, bus Broadcaster) *Service {
	return &Service{db: db, bus: bus}
}
```

`*realtime.Hub` already satisfies `Broadcaster` (it has `Broadcast`), so in `server.New` you pass the hub: `course.NewService(db, hub)`.

### 11.2 The event taxonomy **[A]**

Define event-type constants shared in meaning with the frontend ([§16](#16-consuming-websocket-realtime-in-react)). Mirror the REST resource names so the contract is obvious:

| Event type | Fired when | Payload |
|---|---|---|
| `course.created` | a course is created | the full `CourseResponse` |
| `course.updated` | a course is updated | the full `CourseResponse` |
| `course.deleted` | a course is deleted | `{ "id": "<uuid>" }` |

```go
const (
	EventCourseCreated = "course.created"
	EventCourseUpdated = "course.updated"
	EventCourseDeleted = "course.deleted"
)
```

> **Design choice — send the full object, not just an id.** For created/updated we broadcast the *whole* course DTO. That lets the dashboard update its cache **without a follow-up REST call** — the event carries everything needed to render the new row. For deletes, the id is enough. Sending full objects trades a little bandwidth for far fewer requests and instant UI. (If objects were large or sensitive, you'd send just an id and let clients refetch — a deliberate trade-off.)

### 11.3 Emitting events from the service **[A]**

Add the broadcast calls to the three mutating methods. The pattern is identical: do the DB work, then broadcast the result.

```go
// Create — broadcast the new course to all dashboards.
func (s *Service) Create(ctx context.Context, authorID uuid.UUID, req CreateRequest) (*ent.Course, error) {
	c, err := s.db.Course.Create().
		SetTitle(req.Title).SetDescription(req.Description).
		SetPublished(req.Published).SetAuthorID(authorID).
		Save(ctx)
	if err != nil {
		return nil, err
	}
	full, err := s.Get(ctx, c.ID) // reload with author edge
	if err != nil {
		return nil, err
	}
	s.bus.Broadcast(EventCourseCreated, toResponse(full)) // ← realtime push
	return full, nil
}

// Update — broadcast the updated course.
func (s *Service) Update(ctx context.Context, callerID uuid.UUID, role string, id uuid.UUID, req UpdateRequest) (*ent.Course, error) {
	existing, err := s.Get(ctx, id)
	if err != nil {
		return nil, err
	}
	if !canModifyCourse(role, callerID, existing.Edges.Author.ID) {
		return nil, ErrForbidden
	}
	if _, err := s.db.Course.UpdateOneID(id).
		SetTitle(req.Title).SetDescription(req.Description).
		SetPublished(req.Published).Save(ctx); err != nil {
		return nil, err
	}
	full, err := s.Get(ctx, id)
	if err != nil {
		return nil, err
	}
	s.bus.Broadcast(EventCourseUpdated, toResponse(full)) // ← realtime push
	return full, nil
}

// Delete — broadcast just the id.
func (s *Service) Delete(ctx context.Context, callerID uuid.UUID, role string, id uuid.UUID) error {
	existing, err := s.Get(ctx, id)
	if err != nil {
		return err
	}
	if !canModifyCourse(role, callerID, existing.Edges.Author.ID) {
		return ErrForbidden
	}
	if err := s.db.Course.DeleteOneID(id).Exec(ctx); err != nil {
		return err
	}
	s.bus.Broadcast(EventCourseDeleted, gin.H{"id": id}) // ← realtime push
	return nil
}
```

> **Why broadcast from the service, not the handler?** The service is the one place *all* mutation logic funnels through. If you broadcast from handlers, a future second caller of `Create` (a CLI, a bulk import, a cron) would silently skip the broadcast. Emit from the service and every path that mutates a course automatically notifies dashboards. This is the same reasoning that put the ownership check in the service.

> **Consistency note — broadcast after the DB commit.** We broadcast only *after* the DB write succeeds, so dashboards never hear about a change that didn't actually persist. The tiny window where the DB committed but the broadcast was dropped (buffer full) is self-healing: clients reconnect and refetch, and the REST cache reconciles. For strict guarantees you'd use the *transactional outbox* pattern (write the event to an `events` table in the same transaction, relay it separately) — overkill for an admin dashboard, essential for, say, financial events. Know the name; reach for it only when "might miss a UI update" becomes "must never miss an event."

### 11.4 The backend is done — a mental test **[A]**

Trace a create end-to-end on the server: `POST /api/v1/courses` → CORS preflight passes → `RequireAuth` validates the Bearer token, sets user id + role → `rbac.Require(CourseCreate)` confirms the role may create → `Create` handler binds & validates the DTO → `service.Create` inserts via Ent (author = the JWT's user), reloads with the author edge, returns 201 → **and** calls `hub.Broadcast("course.created", dto)` → the Hub fans the JSON out to every connected dashboard's `send` channel → each `writePump` writes it to its browser. The browser side of that last hop is [§16](#16-consuming-websocket-realtime-in-react).

Everything the dashboard needs now exists on the server: secured CRUD, RBAC, and a live event stream. Time to build the dashboard.

---

## 12. The Next.js Admin Dashboard — Setup & Architecture

We now build the client: a Next.js 16 admin dashboard that logs in against the Go API, lists/creates/edits/deletes courses over REST, updates live over the WebSocket, and shows role-appropriate UI. This section sets up the project structure and the architectural decisions; [§13](#13-auth-on-the-frontend--login-token-storage--route-protection)–[§17](#17-rbac-on-the-frontend--conditional-ui--route-guards) fill in each piece. The [Next.js 16](NEXTJS_16_GUIDE.md) and [React 19](REACT_19_GUIDE.md) guides are the deep references.

### 12.1 The key architectural decision: this dashboard is a client app **[B/I]**

Next.js can render on the server (Server Components, Server Actions). But our backend already *is* the Go service — so what is Next.js's job here? **It renders the UI and acts as a smart HTTP/WS client.** That reframes how we use it:

- The pages that show and edit courses are **Client Components** (`"use client"`) because they need browser-only things: the access token in memory, the WebSocket connection, TanStack Query's cache, interactive forms. A pure Server Component can't hold a live WS or an in-memory token.
- We deliberately **do not** proxy every call through Next.js Server Actions to the Go API. The browser talks to the Go API **directly** (over CORS, [§9](#9-cors-the-api-contract--error-envelopes)). This keeps one source of truth for the contract and avoids a pointless extra hop.
- **When *would* you proxy through Next?** Two good reasons: (a) to keep the API origin private (browser → Next route handler → Go API, server-to-server), or (b) to convert the cookie-based session into the Go API's Bearer scheme server-side. We note this as an option in [§19](#19-security-hardening-full-stack); for clarity the main path is direct browser↔Go.

> **Mental model:** think of this Next.js app the way you'd think of a React SPA that happens to have great routing and build tooling. Most of the interesting code is client-side: an API client, an auth context, a WS hook, and TanStack Query. Server Components mainly provide the shell and route structure.

### 12.2 Folder structure **[B]**

```
web/src/
├── app/                          ← App Router
│   ├── layout.tsx                ← root layout: providers (Query, Auth)
│   ├── page.tsx                  ← redirects to /login or /dashboard
│   ├── login/
│   │   └── page.tsx              ← the login form
│   └── (dashboard)/              ← route group; shares the authed layout
│       ├── layout.tsx            ← guards auth; renders nav + connects the WS
│       └── courses/
│           └── page.tsx          ← the course table (the main screen)
├── components/
│   ├── courses/
│   │   ├── course-table.tsx
│   │   ├── course-form-dialog.tsx
│   │   └── delete-course-dialog.tsx
│   └── ui/                       ← shadcn/ui primitives (button, table, dialog…)
├── lib/
│   ├── api.ts                    ← axios instance + interceptors (auth, refresh)
│   ├── auth-context.tsx          ← React context holding the access token + user
│   ├── query-client.ts           ← the TanStack QueryClient
│   ├── use-courses.ts            ← query/mutation hooks for courses
│   ├── use-websocket.ts          ← the realtime hook
│   └── permissions.ts            ← mirror of the backend permission map (§17)
└── types/
    └── api.ts                    ← the DTO mirror from §9.4
```

### 12.3 Environment & the providers shell **[B]**

`web/.env.local` (gitignored) — note `NEXT_PUBLIC_` makes a var available in the browser; only put **non-secret** values there:

```bash
# Public (browser) — the Go API's base URLs. NOT secrets.
NEXT_PUBLIC_API_URL=http://localhost:8080/api/v1
NEXT_PUBLIC_WS_URL=ws://localhost:8080/ws
```

> **Security note:** anything prefixed `NEXT_PUBLIC_` is **embedded in the JavaScript bundle** and visible to anyone. API *URLs* are fine (they're not secret). Never put the JWT secret, database URL, or any credential in a `NEXT_PUBLIC_` var — those live only in the Go service's environment.

The root layout installs the providers every client feature needs — TanStack Query and our Auth context:

```tsx
// web/src/app/layout.tsx
import type { Metadata } from "next";
import { Providers } from "@/lib/providers";

export const metadata: Metadata = { title: "Course Admin" };

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {/* Providers is a Client Component wrapper (it uses hooks/state). */}
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

```tsx
// web/src/lib/providers.tsx
"use client";

import { QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";
import { getQueryClient } from "@/lib/query-client";
import { AuthProvider } from "@/lib/auth-context";

export function Providers({ children }: { children: React.ReactNode }) {
  const queryClient = getQueryClient();
  return (
    <QueryClientProvider client={queryClient}>
      {/* AuthProvider holds the in-memory access token + current user (§13). */}
      <AuthProvider>{children}</AuthProvider>
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

```ts
// web/src/lib/query-client.ts
import { QueryClient } from "@tanstack/react-query";

// One QueryClient per browser session. We keep data fresh for 30s and retry once.
let client: QueryClient | undefined;
export function getQueryClient(): QueryClient {
  if (!client) {
    client = new QueryClient({
      defaultOptions: {
        queries: { staleTime: 30_000, retry: 1, refetchOnWindowFocus: false },
      },
    });
  }
  return client;
}
```

With the shell in place, the first real feature is authentication.

---

## 13. Auth on the Frontend — Login, Token Storage & Route Protection

This is where the security decisions from [§6](#6-jwt-access--refresh-tokens--auth-middleware) become concrete browser code: store the access token in memory, keep the refresh token in the HttpOnly cookie the server set, attach the token to every request, and transparently refresh on expiry.

### 13.1 The token-storage decision, on the client **[I]**

Restating the rule from [§6.1](#61-what-a-jwt-is-and-why-we-use-the-accessrefresh-split) because it's the most security-critical frontend choice: **the access token lives in JavaScript memory (React state), never in `localStorage`.** The refresh token we never touch from JS at all — it's the HttpOnly cookie the browser sends automatically to `/api/v1/auth/refresh`.

Why memory and not `localStorage`? Because **any** XSS — a vulnerable dependency, a bad `dangerouslySetInnerHTML`, a malicious browser extension — can read `localStorage` and exfiltrate a token sitting there. A token in a closure/React state is far harder to reach and dies on tab close. The cost is that a full page refresh loses the in-memory access token — which is fine, because on mount we **silently call `/auth/refresh`** (using the cookie) to get a new one. That "rehydrate on load" flow is the backbone of this section.

### 13.2 The Auth context **[I/A]**

`web/src/lib/auth-context.tsx` holds the access token and user, and exposes `login`/`logout`/`refresh`. It's a Client Component (it uses state and effects).

```tsx
// web/src/lib/auth-context.tsx
"use client";

import { createContext, useContext, useEffect, useState, useCallback } from "react";
import { api, setAccessToken } from "@/lib/api";
import type { User, AuthResponse } from "@/types/api";

interface AuthState {
  user: User | null;
  loading: boolean; // true during the initial silent refresh
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
}

const AuthContext = createContext<AuthState | null>(null);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  // On mount, try to restore a session: the refresh cookie (if present) lets us
  // mint a fresh access token without re-login. This runs after every full reload.
  useEffect(() => {
    (async () => {
      try {
        const { data } = await api.post<AuthResponse>("/auth/refresh");
        setAccessToken(data.access_token); // store in the api module's memory
        setUser(data.user);
      } catch {
        // No valid refresh cookie → user must log in. Not an error to surface.
      } finally {
        setLoading(false);
      }
    })();
  }, []);

  const login = useCallback(async (email: string, password: string) => {
    const { data } = await api.post<AuthResponse>("/auth/login", { email, password });
    setAccessToken(data.access_token);
    setUser(data.user);
  }, []);

  const logout = useCallback(async () => {
    try {
      await api.post("/auth/logout"); // clears the cookie server-side
    } finally {
      setAccessToken(null);
      setUser(null);
    }
  }, []);

  return (
    <AuthContext.Provider value={{ user, loading, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth must be used within AuthProvider");
  return ctx;
}
```

### 13.3 The API client: interceptors for auth and auto-refresh **[I/A]**

The axios instance is where the auth plumbing concentrates: it attaches the Bearer token to every request and, when a request fails with `401`, transparently refreshes the access token and retries — once. This makes the 15-minute access-token expiry invisible to the user.

`web/src/lib/api.ts`:

```ts
// web/src/lib/api.ts
import axios, { AxiosError, type AxiosRequestConfig } from "axios";
import type { AuthResponse } from "@/types/api";

// --- in-memory access token (NOT localStorage) ---
let accessToken: string | null = null;
export function setAccessToken(t: string | null) { accessToken = t; }
export function getAccessToken() { return accessToken; }

export const api = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  // REQUIRED so the browser sends/receives the refresh cookie cross-origin.
  // Pairs with the server's AllowCredentials:true (§9.2).
  withCredentials: true,
});

// Request interceptor: attach the Bearer token if we have one.
api.interceptors.request.use((config) => {
  if (accessToken) {
    config.headers.Authorization = `Bearer ${accessToken}`;
  }
  return config;
});

// Response interceptor: on a 401, try ONE silent refresh + retry. This hides
// access-token expiry. We guard against loops with a per-request _retry flag and
// never refresh the refresh call itself.
let refreshing: Promise<string> | null = null;

api.interceptors.response.use(
  (res) => res,
  async (error: AxiosError) => {
    const original = error.config as AxiosRequestConfig & { _retry?: boolean };
    const status = error.response?.status;
    const isAuthCall = original.url?.includes("/auth/");

    if (status === 401 && !original._retry && !isAuthCall) {
      original._retry = true;
      try {
        // Coalesce concurrent 401s into a single refresh request.
        refreshing ??= api
          .post<AuthResponse>("/auth/refresh")
          .then((r) => {
            setAccessToken(r.data.access_token);
            return r.data.access_token;
          })
          .finally(() => { refreshing = null; });

        const newToken = await refreshing;
        original.headers = { ...original.headers, Authorization: `Bearer ${newToken}` };
        return api(original); // retry the original request once
      } catch {
        setAccessToken(null);
        // Let the caller/route guard handle the redirect to /login.
      }
    }
    return Promise.reject(error);
  }
);
```

> **Why coalesce refreshes?** If the access token expires and the dashboard fires five requests at once, all five get `401` together. Without coalescing, you'd kick off five parallel refreshes (and four would fail because rotation invalidated the first one's cookie). The shared `refreshing` promise ensures exactly **one** refresh happens and all pending requests await it. This is a subtle but important correctness detail in token-refresh code.

### 13.4 The login page **[I]**

A simple form (React Hook Form + Zod for validation — see the [RHF guide](REACT_HOOK_FORM_GUIDE.md)) calling `login` from the context, then redirecting.

```tsx
// web/src/app/login/page.tsx
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import { useRouter } from "next/navigation";
import { useAuth } from "@/lib/auth-context";
import { useState } from "react";

const schema = z.object({
  email: z.string().email("Enter a valid email"),
  password: z.string().min(1, "Password is required"),
});
type FormValues = z.infer<typeof schema>;

export default function LoginPage() {
  const { login } = useAuth();
  const router = useRouter();
  const [serverError, setServerError] = useState("");
  const { register, handleSubmit, formState: { errors, isSubmitting } } =
    useForm<FormValues>({ resolver: zodResolver(schema) });

  async function onSubmit(values: FormValues) {
    setServerError("");
    try {
      await login(values.email, values.password);
      router.push("/dashboard/courses");
    } catch {
      // Generic message — mirrors the server's non-enumerating error (§5.4).
      setServerError("Invalid email or password");
    }
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="mx-auto max-w-sm space-y-4 p-8">
      <h1 className="text-xl font-semibold">Sign in</h1>
      <div>
        <input {...register("email")} placeholder="Email" className="w-full border p-2" />
        {errors.email && <p className="text-sm text-red-600">{errors.email.message}</p>}
      </div>
      <div>
        <input {...register("password")} type="password" placeholder="Password" className="w-full border p-2" />
        {errors.password && <p className="text-sm text-red-600">{errors.password.message}</p>}
      </div>
      {serverError && <p className="text-sm text-red-600">{serverError}</p>}
      <button disabled={isSubmitting} className="w-full bg-black p-2 text-white disabled:opacity-50">
        {isSubmitting ? "Signing in…" : "Sign in"}
      </button>
    </form>
  );
}
```

### 13.5 Protecting the dashboard routes **[I/A]**

The `(dashboard)` route group's layout guards everything under it: while the initial silent refresh runs, show a loader; if there's no user afterward, redirect to login; otherwise render the app shell (and, in [§16](#16-consuming-websocket-realtime-in-react), open the WebSocket).

```tsx
// web/src/app/(dashboard)/layout.tsx
"use client";

import { useEffect } from "react";
import { useRouter } from "next/navigation";
import { useAuth } from "@/lib/auth-context";

export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  const { user, loading } = useAuth();
  const router = useRouter();

  useEffect(() => {
    if (!loading && !user) router.replace("/login");
  }, [loading, user, router]);

  if (loading) return <div className="p-8">Loading…</div>;
  if (!user) return null; // redirecting

  return (
    <div className="flex min-h-screen flex-col">
      <header className="flex items-center justify-between border-b p-4">
        <span className="font-semibold">Course Admin</span>
        <span className="text-sm text-gray-500">{user.email} · {user.role}</span>
      </header>
      <main className="flex-1 p-6">{children}</main>
    </div>
  );
}
```

> **Client-side guards are UX, not security.** This redirect just stops a logged-out user *seeing* the page chrome. It is **not** a security boundary — anyone can open dev tools and poke around. The real protection is that **every API call requires a valid token** and the **server enforces RBAC** ([§7](#7-rbac--roles-permissions--authorization-middleware)). The data simply won't load without auth. Never put a secret or unauthorized data in the client bundle and rely on a route guard to hide it.

Authentication works end-to-end. Now the actual CRUD data layer.

---

## 14. Data Fetching & Mutations with TanStack Query

The dashboard reads and writes courses through the REST API. **TanStack Query** is the cache and synchronization layer between the UI and that API: it dedupes requests, caches responses, refetches when stale, and gives mutations a clean lifecycle with optimistic updates and rollback. The [TanStack Query guide](TANSTACK_QUERY_GUIDE.md) is the deep reference; here we apply it to Courses.

### 14.1 Why TanStack Query and not `useEffect` + `useState` **[I]**

Fetching with raw `useEffect`/`useState` forces you to hand-roll loading flags, error states, request deduplication, caching, refetching, and cache invalidation after writes — and to keep them consistent across components. TanStack Query provides all of it declaratively. The core concepts:

- **Query** — a cached read keyed by a **query key** (e.g. `["courses", { limit, offset }]`). Many components can `useQuery` the same key and share one request and one cache entry.
- **Mutation** — a write (create/update/delete) with `onMutate`/`onSuccess`/`onError`/`onSettled` lifecycle hooks.
- **Invalidation** — after a successful write, mark related queries **stale** so they refetch. This is how the UI stays consistent with the server after a mutation… *and* it's what our WebSocket events will trigger to make the table live ([§16](#16-consuming-websocket-realtime-in-react)).

### 14.2 The course query hooks **[I]**

`web/src/lib/use-courses.ts`:

```ts
// web/src/lib/use-courses.ts
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { api } from "@/lib/api";
import type { Course, Paginated, CreateCourseInput, UpdateCourseInput } from "@/types/api";

// A stable key factory keeps query keys consistent across the app.
export const courseKeys = {
  all: ["courses"] as const,
  list: (page: { limit: number; offset: number }) => ["courses", page] as const,
  detail: (id: string) => ["courses", "detail", id] as const,
};

// List courses (paginated). The queryFn returns the parsed data; TanStack Query
// handles loading/error/cache around it.
export function useCourses(page = { limit: 20, offset: 0 }) {
  return useQuery({
    queryKey: courseKeys.list(page),
    queryFn: async () => {
      const { data } = await api.get<Paginated<Course>>("/courses", { params: page });
      return data;
    },
  });
}
```

### 14.3 The mutation hooks **[I/A]**

Each mutation calls the API, then invalidates the course list so it refetches. (In [§15](#15-the-course-crud-ui--tables-forms--optimistic-updates) we add *optimistic* updates for instant feedback.)

```ts
// Create
export function useCreateCourse() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: async (input: CreateCourseInput) => {
      const { data } = await api.post<{ data: Course }>("/courses", input);
      return data.data;
    },
    onSuccess: () => {
      // Mark all course lists stale → they refetch with the new row. (The WS
      // event will ALSO arrive; §16 explains why double-updating is harmless.)
      qc.invalidateQueries({ queryKey: courseKeys.all });
    },
  });
}

// Update
export function useUpdateCourse() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: async ({ id, input }: { id: string; input: UpdateCourseInput }) => {
      const { data } = await api.put<{ data: Course }>(`/courses/${id}`, input);
      return data.data;
    },
    onSuccess: () => qc.invalidateQueries({ queryKey: courseKeys.all }),
  });
}

// Delete
export function useDeleteCourse() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: async (id: string) => {
      await api.delete(`/courses/${id}`); // 204, no body
      return id;
    },
    onSuccess: () => qc.invalidateQueries({ queryKey: courseKeys.all }),
  });
}
```

Add the input types to `web/src/types/api.ts` (mirroring the Go request DTOs from [§8.2](#82-dtos-separate-wire-shapes-from-the-entity)):

```ts
export interface CreateCourseInput {
  title: string;
  description: string;
  published: boolean;
}
export type UpdateCourseInput = CreateCourseInput;
```

> **The relationship between invalidation and realtime.** Notice mutations invalidate on success *and* the server broadcasts a WS event for the same change. That looks redundant, but they cover different cases: **invalidation** keeps *the acting user's* UI correct even if the WS is momentarily disconnected; the **WS event** updates *every other* user's dashboard. In [§16](#16-consuming-websocket-realtime-in-react) we make the WS handler write directly into the same TanStack Query cache, so both paths converge on one cache and the UI is always consistent. Designing them to share one cache is what makes "realtime" and "request/response" coexist cleanly.

The data layer is ready. Next we build the UI that uses it — and wire in the live updates.

---

## 15. The Course CRUD UI — Tables, Forms & Optimistic Updates

Now the screens users touch: a table of courses, a create/edit dialog with validation, a delete confirmation, and **optimistic updates** so the UI feels instant. We use [shadcn/ui](SHADCN_UI_CHEATSHEET.md) primitives and [React Hook Form + Zod](REACT_HOOK_FORM_GUIDE.md); the patterns transfer to any component kit.

### 15.1 The course table **[I]**

The table reads from `useCourses` and renders rows. Action buttons (Edit/Delete) are gated by permissions — but we add that in [§17](#17-rbac-on-the-frontend--conditional-ui--route-guards); here we render them all and focus on data.

```tsx
// web/src/components/courses/course-table.tsx
"use client";

import { useState } from "react";
import { useCourses } from "@/lib/use-courses";
import { CourseFormDialog } from "./course-form-dialog";
import { DeleteCourseDialog } from "./delete-course-dialog";
import type { Course } from "@/types/api";

export function CourseTable() {
  const { data, isLoading, isError } = useCourses();
  const [editing, setEditing] = useState<Course | null>(null);
  const [deleting, setDeleting] = useState<Course | null>(null);

  if (isLoading) return <p>Loading courses…</p>;
  if (isError) return <p className="text-red-600">Failed to load courses.</p>;

  return (
    <div className="space-y-4">
      <div className="flex justify-between">
        <h2 className="text-lg font-semibold">Courses</h2>
        <CourseFormDialog mode="create" trigger={<button className="bg-black px-3 py-1.5 text-white">New course</button>} />
      </div>

      <table className="w-full border-collapse text-sm">
        <thead>
          <tr className="border-b text-left">
            <th className="p-2">Title</th>
            <th className="p-2">Author</th>
            <th className="p-2">Published</th>
            <th className="p-2">Updated</th>
            <th className="p-2 text-right">Actions</th>
          </tr>
        </thead>
        <tbody>
          {data?.data.map((course) => (
            <tr key={course.id} className="border-b">
              <td className="p-2 font-medium">{course.title}</td>
              <td className="p-2 text-gray-500">{course.author_email}</td>
              <td className="p-2">{course.published ? "✅" : "—"}</td>
              <td className="p-2 text-gray-500">{new Date(course.updated_at).toLocaleString()}</td>
              <td className="p-2 text-right space-x-2">
                <button onClick={() => setEditing(course)} className="text-blue-600">Edit</button>
                <button onClick={() => setDeleting(course)} className="text-red-600">Delete</button>
              </td>
            </tr>
          ))}
          {data?.data.length === 0 && (
            <tr><td colSpan={5} className="p-4 text-center text-gray-400">No courses yet.</td></tr>
          )}
        </tbody>
      </table>

      {/* Controlled dialogs for the row being edited/deleted */}
      {editing && (
        <CourseFormDialog mode="edit" course={editing} open onOpenChange={(o) => !o && setEditing(null)} />
      )}
      {deleting && (
        <DeleteCourseDialog course={deleting} open onOpenChange={(o) => !o && setDeleting(null)} />
      )}
    </div>
  );
}
```

### 15.2 The create/edit form dialog **[I/A]**

One component handles both create and edit (the only difference is which mutation runs and the initial values). Validation with Zod mirrors the server's binding rules — but remember the server validates regardless ([§8.4](#84-the-handlers)); client validation is for fast feedback.

```tsx
// web/src/components/courses/course-form-dialog.tsx
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import { useCreateCourse, useUpdateCourse } from "@/lib/use-courses";
import type { Course } from "@/types/api";

// Mirror the Go binding tags: title required ≤200, description ≤5000.
const schema = z.object({
  title: z.string().min(1, "Title is required").max(200),
  description: z.string().max(5000).default(""),
  published: z.boolean().default(false),
});
type FormValues = z.infer<typeof schema>;

interface Props {
  mode: "create" | "edit";
  course?: Course;
  trigger?: React.ReactNode;
  open?: boolean;
  onOpenChange?: (open: boolean) => void;
}

export function CourseFormDialog({ mode, course, open, onOpenChange }: Props) {
  const create = useCreateCourse();
  const update = useUpdateCourse();
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm<FormValues>({
    resolver: zodResolver(schema),
    defaultValues: {
      title: course?.title ?? "",
      description: course?.description ?? "",
      published: course?.published ?? false,
    },
  });

  async function onSubmit(values: FormValues) {
    if (mode === "create") {
      await create.mutateAsync(values);
    } else if (course) {
      await update.mutateAsync({ id: course.id, input: values });
    }
    onOpenChange?.(false); // close on success
  }

  // (Dialog chrome omitted; use your shadcn <Dialog>. open/onOpenChange drive it.)
  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-3">
      <input {...register("title")} placeholder="Title" className="w-full border p-2" />
      {errors.title && <p className="text-sm text-red-600">{errors.title.message}</p>}

      <textarea {...register("description")} placeholder="Description" className="w-full border p-2" rows={4} />
      {errors.description && <p className="text-sm text-red-600">{errors.description.message}</p>}

      <label className="flex items-center gap-2">
        <input type="checkbox" {...register("published")} /> Published
      </label>

      <button disabled={isSubmitting} className="bg-black px-3 py-1.5 text-white disabled:opacity-50">
        {isSubmitting ? "Saving…" : mode === "create" ? "Create" : "Save changes"}
      </button>
    </form>
  );
}
```

### 15.3 Optimistic delete with rollback **[A]**

**Optimistic updates** apply the change to the UI *immediately*, before the server confirms — then roll back if the request fails. For delete, the row vanishes instantly; if the API returns an error (e.g. a 403 because an editor tried to delete someone else's course), we restore it. TanStack Query's `onMutate`/`onError`/`onSettled` is built for exactly this.

Upgrade `useDeleteCourse` to be optimistic:

```ts
// web/src/lib/use-courses.ts (optimistic delete)
export function useDeleteCourse() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: async (id: string) => { await api.delete(`/courses/${id}`); return id; },

    // BEFORE the request: snapshot the cache and optimistically remove the row.
    onMutate: async (id: string) => {
      await qc.cancelQueries({ queryKey: courseKeys.all }); // avoid races with in-flight refetches
      const snapshots = qc.getQueriesData<Paginated<Course>>({ queryKey: courseKeys.all });
      for (const [key, page] of snapshots) {
        if (!page) continue;
        qc.setQueryData<Paginated<Course>>(key, {
          ...page,
          data: page.data.filter((c) => c.id !== id),
          pagination: { ...page.pagination, total: page.pagination.total - 1 },
        });
      }
      return { snapshots }; // context passed to onError
    },

    // ON FAILURE: roll back to the snapshot we took.
    onError: (_err, _id, context) => {
      context?.snapshots.forEach(([key, page]) => qc.setQueryData(key, page));
    },

    // ALWAYS: refetch to converge with server truth (also reconciles WS events).
    onSettled: () => qc.invalidateQueries({ queryKey: courseKeys.all }),
  });
}
```

```tsx
// web/src/components/courses/delete-course-dialog.tsx
"use client";

import { useDeleteCourse } from "@/lib/use-courses";
import type { Course } from "@/types/api";

export function DeleteCourseDialog({ course, onOpenChange }: {
  course: Course; open?: boolean; onOpenChange?: (o: boolean) => void;
}) {
  const del = useDeleteCourse();
  return (
    <div className="space-y-4">
      <p>Delete <strong>{course.title}</strong>? This cannot be undone.</p>
      <div className="flex justify-end gap-2">
        <button onClick={() => onOpenChange?.(false)} className="px-3 py-1.5">Cancel</button>
        <button
          onClick={async () => { await del.mutateAsync(course.id); onOpenChange?.(false); }}
          className="bg-red-600 px-3 py-1.5 text-white"
        >Delete</button>
      </div>
    </div>
  );
}
```

> **Optimistic UI and authorization don't always agree — that's fine.** An editor might click Delete on a course they don't own; the row vanishes optimistically, the server returns 403, and `onError` restores it. The user briefly saw the row disappear. To avoid even that flicker, *also* hide the Delete button when the user lacks permission ([§17](#17-rbac-on-the-frontend--conditional-ui--route-guards)) — but the server's 403 remains the real guard. Optimism improves the happy path; it never replaces server authorization.

The CRUD UI works against REST. Now the magic: make it live.

---

## 16. Consuming WebSocket Realtime in React

The server broadcasts `course.created/updated/deleted` events ([§11](#11-broadcasting-crud-events-to-the-dashboard)). Here we connect to the WebSocket from React, and — the key idea — **write incoming events straight into the TanStack Query cache** so the table updates live with no refetch. We also handle the things production WebSocket clients must: authentication, reconnection with backoff, and cleanup.

### 16.1 The design: one WS connection feeding the query cache **[A]**

The cleanest architecture is a **single** WebSocket for the whole dashboard (not one per component), opened in the authed layout, that dispatches each event by mutating the shared TanStack Query cache. Because the cache is the single source of truth the table already reads from, updating the cache *is* updating the UI — every component showing courses re-renders automatically. This is why we built REST and realtime to share one cache ([§14.3](#143-the-mutation-hooks)): an event and a REST refetch both end at `qc.setQueryData`, so they can never disagree.

```
WS event "course.created" ─▶ useWebSocket onMessage ─▶ qc.setQueryData(courses, add row)
                                                              │
REST useCourses() ◀───────── reads the same cache ◀──────────┘ ─▶ table re-renders
```

### 16.2 The `useWebSocket` hook **[A]**

This hook owns the connection lifecycle: connect (with the access token), parse events, update the cache, ping-aware reconnect with exponential backoff, and tear down on unmount. The browser auto-replies to the server's pings, so the client side mainly needs reconnect logic.

```tsx
// web/src/lib/use-websocket.ts
"use client";

import { useEffect, useRef } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { getAccessToken } from "@/lib/api";
import { courseKeys } from "@/lib/use-courses";
import type { Course, Paginated } from "@/types/api";

// Mirror the server's event taxonomy (§11.2).
type WsEvent =
  | { type: "course.created"; payload: Course }
  | { type: "course.updated"; payload: Course }
  | { type: "course.deleted"; payload: { id: string } };

export function useCourseSocket(enabled: boolean) {
  const qc = useQueryClient();
  const wsRef = useRef<WebSocket | null>(null);
  const retryRef = useRef(0);
  const closedByUs = useRef(false);

  useEffect(() => {
    if (!enabled) return;
    closedByUs.current = false;

    function connect() {
      const token = getAccessToken();
      if (!token) return; // not authed yet; the layout re-runs this when it is

      // Auth via query token (§10.4). Use WSS in production.
      const url = `${process.env.NEXT_PUBLIC_WS_URL}?token=${encodeURIComponent(token)}`;
      const ws = new WebSocket(url);
      wsRef.current = ws;

      ws.onopen = () => { retryRef.current = 0; }; // reset backoff on success

      ws.onmessage = (e) => {
        let evt: WsEvent;
        try { evt = JSON.parse(e.data); } catch { return; }
        applyEvent(qc, evt); // update the cache (below)
      };

      ws.onclose = () => {
        if (closedByUs.current) return;
        // Exponential backoff with a cap: 1s, 2s, 4s … max 30s. Jitter avoids
        // a thundering herd when the server restarts and everyone reconnects.
        const delay = Math.min(1000 * 2 ** retryRef.current, 30_000);
        retryRef.current++;
        const jitter = Math.random() * 1000;
        setTimeout(connect, delay + jitter);
      };

      ws.onerror = () => ws.close(); // trigger onclose → reconnect
    }

    connect();

    // Cleanup on unmount / logout: stop reconnecting and close.
    return () => {
      closedByUs.current = true;
      wsRef.current?.close();
    };
  }, [enabled, qc]);
}

// applyEvent maps a server event to a cache mutation across all course-list pages.
function applyEvent(qc: ReturnType<typeof useQueryClient>, evt: WsEvent) {
  const lists = qc.getQueriesData<Paginated<Course>>({ queryKey: courseKeys.all });

  for (const [key, page] of lists) {
    if (!page) continue;

    if (evt.type === "course.created") {
      // Avoid duplicates: the acting user may already have it via REST/invalidation.
      if (page.data.some((c) => c.id === evt.payload.id)) continue;
      qc.setQueryData<Paginated<Course>>(key, {
        ...page,
        data: [evt.payload, ...page.data],
        pagination: { ...page.pagination, total: page.pagination.total + 1 },
      });
    } else if (evt.type === "course.updated") {
      qc.setQueryData<Paginated<Course>>(key, {
        ...page,
        data: page.data.map((c) => (c.id === evt.payload.id ? evt.payload : c)),
      });
    } else if (evt.type === "course.deleted") {
      qc.setQueryData<Paginated<Course>>(key, {
        ...page,
        data: page.data.filter((c) => c.id !== evt.payload.id),
        pagination: { ...page.pagination, total: Math.max(0, page.pagination.total - 1) },
      });
    }
  }
}
```

Activate it in the dashboard layout, enabled only once authenticated:

```tsx
// in web/src/app/(dashboard)/layout.tsx, inside the component:
import { useCourseSocket } from "@/lib/use-websocket";
// ...
useCourseSocket(!!user); // open the socket once we have a user; closes on logout
```

### 16.3 Why idempotency matters (the duplicate-event problem) **[A]**

When *you* create a course, two things update your cache: the mutation's `onSuccess` invalidation/refetch **and** the WS `course.created` event echoing your own action back. Without care you'd insert the row twice. Our `applyEvent` guards every insert with a `some(c => c.id === ...)` existence check, so applying the same event twice is a no-op. **Design realtime handlers to be idempotent** — apply-event-twice must equal apply-once — because networks redeliver, reconnects replay, and your own actions echo back. (Alternatively, the server could skip echoing to the originating client, but idempotent handlers are simpler and more robust.)

### 16.4 Reconnection and the "missed events" gap **[A]**

A WebSocket *will* drop (network blip, server redeploy, laptop sleep). Our hook reconnects with exponential backoff. But events that fired *while disconnected are gone* — WebSockets have no replay. The standard fix: **on reconnect, refetch** to resynchronize with server truth. Add an `onopen` that invalidates after the *first* successful reconnect:

```ts
ws.onopen = () => {
  const wasReconnect = retryRef.current > 0;
  retryRef.current = 0;
  if (wasReconnect) {
    // We may have missed events while down — refetch to catch up.
    qc.invalidateQueries({ queryKey: courseKeys.all });
  }
};
```

This "live updates while connected; refetch on reconnect" pattern gives you realtime *and* correctness. The WS is an **optimization for freshness**, not the system of record — the REST API and its cache remain authoritative, so a flaky socket degrades gracefully to "slightly less live" rather than "wrong."

> **Connection status UX.** Expose the socket state (connecting / live / reconnecting) and show a small indicator ("🟢 Live" / "🟡 Reconnecting…"). Users trust a dashboard more when it's honest about whether it's currently live. Track it with a `useState` set from the WS event handlers and surface it in the header.

The dashboard is now live. The last piece is making it *role-aware*.

---

## 17. RBAC on the Frontend — Conditional UI & Route Guards

The server enforces RBAC on every call ([§7](#7-rbac--roles-permissions--authorization-middleware)). The frontend's job is **UX**: don't show a viewer a "New course" button that would just 403, and don't route an editor to an admin-only users page. This is *mirroring* the backend policy for usability — never *replacing* it.

### 17.1 The golden rule, one more time **[I]**

> **Frontend RBAC is for hiding controls, not for security.** Anything a hidden button would have done, an attacker can do by calling the API directly. The only real enforcement is server-side. So we mirror the permission map purely to produce a clean UI — and we keep that mirror small and clearly labeled as advisory.

This is why [§7.6](#76-exposing-the-callers-permissions-to-the-frontend) had the API return the user's **permissions** list from `/auth/me`: the frontend renders against the *server's* notion of what the user can do, so the two never drift. (We can also hardcode a mirror map; using the server's list is better because it's authoritative.)

### 17.2 A permission mirror and a `Can` helper **[I]**

Fetch the permissions with the user and expose a simple check. Extend the auth context to load `/auth/me` (which returns `{ user, permissions }`) so we have the permission list:

```ts
// web/src/lib/permissions.ts
export type Permission =
  | "course:read" | "course:create" | "course:update"
  | "course:delete" | "user:manage";

// The check: does the current permission set include this permission?
export function can(perms: Permission[] | undefined, p: Permission): boolean {
  return !!perms?.includes(p);
}
```

Store `permissions` in the auth context (populate it from the `/auth/refresh` and `/auth/login` responses by also calling `/auth/me`, or have those endpoints include permissions). Then a tiny hook:

```tsx
// web/src/lib/use-can.ts
"use client";
import { useAuth } from "@/lib/auth-context";
import { can, type Permission } from "@/lib/permissions";

export function useCan(p: Permission): boolean {
  const { permissions } = useAuth(); // permissions: Permission[] from /auth/me
  return can(permissions, p);
}
```

### 17.3 Conditional controls **[I]**

Now gate UI on permissions. The table's create button and row actions:

```tsx
// in course-table.tsx
import { useCan } from "@/lib/use-can";

function CourseTableActions({ course }: { course: Course }) {
  const canUpdate = useCan("course:update");
  const canDelete = useCan("course:delete");
  // Note: this can't express "own only" (that's row-level, server-enforced).
  // We optionally refine below. Viewers see neither button.
  return (
    <td className="p-2 text-right space-x-2">
      {canUpdate && <button className="text-blue-600">Edit</button>}
      {canDelete && <button className="text-red-600">Delete</button>}
    </td>
  );
}
```

And the "New course" button only for creators:

```tsx
const canCreate = useCan("course:create");
// ...
{canCreate && <CourseFormDialog mode="create" trigger={/* ... */} />}
```

### 17.4 Reflecting row-level ownership in the UI **[I/A]**

Permissions are coarse ("editors can update"); the *own-only* rule is row-level ([§7.4](#74-row-level-ownership-the-editor-owns-their-courses)). To avoid showing an editor an Edit button on a course they can't actually edit, combine the permission with an ownership check using the current user's id:

```tsx
function canEditRow(course: Course, userId: string, role: string, hasUpdatePerm: boolean): boolean {
  if (!hasUpdatePerm) return false;       // viewers: no
  if (role === "admin") return true;      // admins: any course
  return course.author_id === userId;     // editors: own courses only
}
```

This mirrors the server's `canModifyCourse` exactly — same logic, two languages. When the policy changes, both must change (the monorepo keeps them in one PR). And again: if the mirror is ever wrong, the worst case is a button that 403s, not a security hole.

### 17.5 Guarding whole routes by role **[I/A]**

Some pages are entirely role-scoped — e.g. `/dashboard/users` is admin-only. Guard at the route with a small wrapper that redirects unauthorized users:

```tsx
// web/src/components/require-permission.tsx
"use client";

import { useEffect } from "react";
import { useRouter } from "next/navigation";
import { useCan } from "@/lib/use-can";
import type { Permission } from "@/lib/permissions";

export function RequirePermission({ permission, children }: {
  permission: Permission; children: React.ReactNode;
}) {
  const allowed = useCan(permission);
  const router = useRouter();
  useEffect(() => {
    if (!allowed) router.replace("/dashboard/courses"); // bounce to a page they can see
  }, [allowed, router]);
  return allowed ? <>{children}</> : null;
}
```

```tsx
// web/src/app/(dashboard)/users/page.tsx
"use client";
import { RequirePermission } from "@/components/require-permission";

export default function UsersPage() {
  return (
    <RequirePermission permission="user:manage">
      {/* the admin-only user management table */}
    </RequirePermission>
  );
}
```

> **Same caveat, scaled up:** this route guard prevents an editor from *navigating* to the users page in the UI. It does not stop them calling `PATCH /api/v1/users/:id/role` directly — that's blocked by `rbac.Require(UserManage)` on the server. The guard is courtesy; the middleware is security.

The dashboard is now complete: authenticated, live, role-aware, and backed by a server that enforces every rule. Let's trace one action through the whole machine to cement how the parts fit.

---

## 18. End-to-End Walkthrough: A Create Flows Through the Whole Stack

To consolidate everything, follow a single action — **an editor creates a course** — through every layer of both programs. Two browsers are open: the editor's (Alice) and another admin's (Bob). When Alice clicks Create, Bob's table must update live without a refresh.

### 18.1 The full trace, step by step **[A]**

```
ALICE'S BROWSER (the actor)                       GO API SERVICE                         BOB'S BROWSER (a watcher)
──────────────────────────                        ─────────────                          ─────────────────────────
1. Clicks "Create" in CourseFormDialog
2. RHF + Zod validate locally (title ≤200)
3. useCreateCourse.mutateAsync(input)
4. axios POST /api/v1/courses
   • request interceptor adds
     Authorization: Bearer <access>
   • withCredentials sends refresh cookie
        │
        │ (browser sends CORS preflight OPTIONS first
        │  for the non-simple request) ───────────────▶ 5. CORS middleware answers preflight (allow
        │                                                   origin, methods, Authorization header)
        │ ──── actual POST ──────────────────────────▶ 6. requestID + structuredLog middleware
        │                                               7. RequireAuth: validate HS256 token,
        │                                                  assert alg, set user_id + role=editor
        │                                               8. rbac.Require(course:create): editor HAS it ✓
        │                                               9. Create handler: ShouldBindJSON → DTO
        │                                              10. service.Create: Ent INSERT, author = Alice
        │                                                  (from JWT — NOT the request body)
        │                                              11. reload WithAuthor → CourseResponse
        │                                              12. hub.Broadcast("course.created", dto) ──┐
        │ ◀──── 201 Created { data: course } ─────────  13. handler returns 201                   │
13a. onSuccess: invalidate ["courses"]                                                            │
14. table refetches (or WS fills it) → row appears                                                │
                                                       Hub.run() fans the JSON to every client's  │
                                                       send channel ──────────────────────────────┤
                                                                                                   ▼
                                                                            15. Bob's writePump writes the frame
                                                       ◀──── WS frame ──────────────────────────  
                                                                            16. useCourseSocket onmessage parses
                                                                                {type:"course.created", payload}
                                                                            17. applyEvent: qc.setQueryData adds row
                                                                                (idempotent existence check)
                                                                            18. Bob's CourseTable re-renders — the
                                                                                new course appears instantly. ✨
```

### 18.2 What each safeguard prevented **[A]**

Walking the same path adversarially shows why each layer exists:

- **If Alice were a viewer** — step 8 (`rbac.Require(course:create)`) returns **403** before any DB work. The frontend also hid the Create button ([§17.3](#173-conditional-controls)), but the server is what actually stopped it.
- **If Alice forged `author_id`** in the JSON to blame Bob — step 10 ignores the body's author entirely and uses the **JWT subject**. Over-posting defeated by DTO design ([§8.2](#82-dtos-separate-wire-shapes-from-the-entity)).
- **If Alice's access token expired** mid-session — the response interceptor caught the **401**, silently hit `/auth/refresh` with the cookie, got a new token, and retried — invisible to Alice ([§13.3](#133-the-api-client-interceptors-for-auth-and-auto-refresh)).
- **If a malicious site tried to open the WebSocket** riding Alice's session — `CheckOrigin` rejected the upgrade (CSWSH defense, [§10.4](#104-authenticating-the-websocket-upgrade--the-tricky-part)).
- **If Bob's network hiccuped** during the broadcast — he missed the event, but on reconnect his hook **refetched** and caught up ([§16.4](#164-reconnection-and-the-missed-events-gap)).
- **If the broadcast and Alice's own refetch both added the row** to her cache — the idempotent `applyEvent` made the duplicate a no-op ([§16.3](#163-why-idempotency-matters-the-duplicate-event-problem)).

This is the whole point of the architecture: **defense in depth**, with the server as the immovable source of truth and the client as a fast, friendly, but untrusted consumer.

### 18.3 Running it locally **[A]**

Three terminals:

```bash
# 1) Go API (with Air for live reload, or plain go run)
cd api && air            # or: go run ./cmd/server

# 2) Next.js dashboard
cd web && npm run dev    # http://localhost:3000

# 3) (optional) watch the WS by hand
npx wscat -c "ws://localhost:8080/ws?token=<paste-an-access-token>"
```

Log in as the seeded admin, create a course, and watch a second browser tab update live. Then create an `editor` and a `viewer` (promote via the admin users page or seed them) and confirm the buttons and the 403s behave per the permission table in [§7.1](#71-roles-permissions-and-why-you-separate-them).

---

## 19. Security Hardening (Full-Stack)

Security has been woven through every section; this consolidates it into a checklist you can audit against, plus the cross-cutting concerns that don't belong to one section. The threat model: an admin tool holds privileged data and privileged actions, so the bar is high.

### 19.1 The trust boundary, stated plainly **[A]**

There is exactly one trust boundary: **the network between the browser and the Go API.** Everything in the browser is **untrusted** — the user can read the JS, forge requests, replay tokens, and skip the UI entirely. Everything the Go service does after authenticating a request is **trusted**. Therefore:

- **Every** rule (validation, RBAC, ownership) is enforced **server-side**. The frontend mirrors rules only for UX.
- The database credential, the JWT secret, and any third-party keys live **only** in the Go service's environment — never in the browser bundle, never in a `NEXT_PUBLIC_` var.
- The browser holds only **short-lived, narrowly-scoped** credentials (a 15-min access token in memory; a path-scoped HttpOnly refresh cookie).

### 19.2 Authentication & token checklist **[A]**

| Control | Where | Status in this guide |
|---|---|---|
| Argon2id password hashing (memory-hard, salted, constant-time verify) | [§5](#5-password-security-with-argon2id--the-signuplogin-flow) | ✅ |
| Minimum password length (≥12) enforced at validation | [§5.4](#54-the-signup--login-handlers) | ✅ |
| Generic auth errors (no account enumeration) | [§5.4](#54-the-signup--login-handlers) | ✅ |
| JWT algorithm assertion (defeat `alg:none`/confusion) | [§6.3](#63-validating-tokens--rigorous-algorithm-checking) | ✅ |
| Short access TTL + rotating refresh | [§6](#6-jwt-access--refresh-tokens--auth-middleware) | ✅ |
| Access token in memory, refresh in HttpOnly cookie | [§6.1](#61-what-a-jwt-is-and-why-we-use-the-accessrefresh-split), [§13.1](#131-the-token-storage-decision-on-the-client) | ✅ |
| `Secure` + `SameSite` cookie flags | [§6.4](#64-the-refresh-cookie-helpers) | ✅ |
| Instant global logout (secret rotation or denylist) | below | ➕ optional |

**Token revocation.** JWTs can't be un-issued before expiry. For an admin tool, two levers suffice: (1) **rotate `JWT_SECRET`** to invalidate *all* access tokens at once (the "break glass" button); (2) maintain a small **Redis denylist** of revoked refresh-token `jti`s checked at refresh time, so disabling one user takes effect at their next refresh (≤15 min). Implement (2) when you add "deactivate user." The [JWT+Argon2 guide](GO_JWT_ARGON2_GUIDE.md) details denylists and reuse-detection.

### 19.3 Input, injection & the web vulnerabilities **[A]**

- **SQL injection** — **not a risk** in our query paths because Ent generates parameterized queries; we never build SQL by string concatenation. (If you ever drop to raw SQL, use parameters, never `fmt.Sprintf`.)
- **XSS (cross-site scripting)** — React escapes interpolated values by default, so rendering `course.title` is safe. The danger is `dangerouslySetInnerHTML` — avoid it, or sanitize with DOMPurify if you must render user HTML. XSS is also *why* the access token is in memory and the refresh cookie is HttpOnly ([§13.1](#131-the-token-storage-decision-on-the-client)).
- **Mass assignment / over-posting** — defeated by request DTOs that list only client-settable fields ([§8.2](#82-dtos-separate-wire-shapes-from-the-entity)). The author is always the JWT subject, never the body.
- **IDOR** — defeated by the server-side ownership check in the service layer ([§7.4](#74-row-level-ownership-the-editor-owns-their-courses)). Never trust the UI to have hidden an object.
- **Request size limits** — cap body size (`http.MaxBytesReader` / Gin's `MaxMultipartMemory`) so a giant payload can't exhaust memory. Bound list `limit` (we clamp to 100, [§8.4](#84-the-handlers)).

### 19.4 CORS, CSRF & the realtime channel **[A]**

- **CORS** — exact-origin allowlist, `AllowCredentials: true`, no wildcard with credentials ([§9.2](#92-configuring-cors-for-the-dashboard)).
- **CSRF** — our primary auth (Bearer header) is **inherently CSRF-resistant**: a cross-site form can't set an `Authorization` header, and a custom header triggers a preflight. The **refresh cookie** is the one cookie-borne credential; `SameSite=Lax` blocks it on cross-site POSTs, and it's scoped to `/api/v1/auth`. If you must run cross-site (`SameSite=None`), add a **double-submit CSRF token** to the refresh endpoint. Decide cookie strategy with [§6.4](#64-the-refresh-cookie-helpers)'s cross-site note in mind.
- **CSWSH** — the WebSocket's `CheckOrigin` allowlist ([§10.4](#104-authenticating-the-websocket-upgrade--the-tricky-part)) is the WS analog of CSRF protection. Never `return true`.
- **WSS + TLS** — in production the WS runs over **WSS** so the `?token=` query and all frames are encrypted. Prefer the short-lived access token or a single-use ticket in that URL.

### 19.5 Rate limiting & abuse **[A]**

Brute-forcing `/auth/login` is the obvious attack. Add a **rate limiter** — per-IP and per-account — on the auth endpoints (a Redis token-bucket; the [Redis guide](REDIS_GUIDE.md) and [JWT+Argon2 guide](GO_JWT_ARGON2_GUIDE.md) show implementations). Also rate-limit the WS endpoint's connection attempts. Lock or exponentially delay an account after repeated failures. Behind a reverse proxy, configure trusted proxies so `c.ClientIP()` reads the real client, not the proxy ([Gin guide §4.6](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md)).

### 19.6 Transport, headers & infra **[A]**

- **HTTPS everywhere** in production (the cookie `Secure` flag *requires* it). Terminate TLS at [Nginx](NGINX_GUIDE.md) or the platform.
- **Security headers** — `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, a **Content-Security-Policy** (the strongest XSS mitigation), `X-Frame-Options: DENY`. Set them at the proxy or with Gin middleware.
- **Secrets management** — `.env` gitignored ([§2.4](#24-the-gitignore-do-this-before-your-first-commit)); in prod inject via the platform's secret store. Rotate on any suspected exposure.
- **Least-privilege DB** — the Postgres role in `DATABASE_URL` should have only the privileges the app needs. Keep the Supabase service-role/admin keys out of the app entirely (we don't use them).
- **Logging hygiene** — never log tokens, passwords, or full cookies. Log the request id and user id for tracing ([§3.2](#32-what-gin-gives-us-and-the-baseline-engine)).

> **Security recommendation — audit against OWASP.** This stack's main risks map to the OWASP Top 10: Broken Access Control (RBAC/IDOR — our biggest surface), Cryptographic Failures (Argon2/TLS), Injection (Ent parameterization), Identification & Auth Failures (token handling), and Security Misconfiguration (CORS/headers/secrets). Re-read [§7](#7-rbac--roles-permissions--authorization-middleware) and this section together before shipping; access control is where admin tools most often fail.

---

## 20. Deployment — Go API + Next.js + Supabase

Two programs deploy independently. The data tier (Supabase) is already managed. This section covers containerizing the Go API, building the Next.js app, the production env differences, and putting them behind one origin so cookies stay simple.

### 20.1 Production configuration differences **[A]**

| Setting | Dev | Production |
|---|---|---|
| `APP_ENV` | `development` (auto-migrate, debug) | `production` (release mode, no auto-migrate) |
| DB connection | direct or session pooler | **session pooler** (`...pooler.supabase.com:5432`) |
| Migrations | `Schema.Create` auto-migrate | **Atlas versioned migrations**, run as a deploy step |
| Cookie `Secure` | `false` (http localhost) | `true` (HTTPS only) |
| `ALLOWED_ORIGINS` | `http://localhost:3000` | `https://admin.example.com` |
| WS URL | `ws://` | `wss://` |
| JWT secret | a dev value | a strong secret from the secret store |

> **Do not auto-migrate in production.** `Schema.Create` can't safely drop/alter columns and gives no audit trail. Generate **Atlas** migrations from your Ent schema (`ent/migrate`), review them in a PR, and apply them as an explicit, ordered deploy step — see the [Ent guide](GO_ENT_ORM_GUIDE.md).

### 20.2 Containerizing the Go API **[A]**

A multi-stage Dockerfile yields a tiny, secure image (build with the full toolchain, ship only the static binary on a minimal base). See the [Docker guide](DOCKER_GUIDE.md) for the reasoning.

```dockerfile
# api/Dockerfile
# --- build stage ---
FROM golang:1.26 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
# CGO off → a fully static binary that runs on a scratch/distroless base.
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /out/server ./cmd/server

# --- run stage ---
FROM gcr.io/distroless/static-debian12:nonroot
WORKDIR /app
COPY --from=build /out/server /app/server
USER nonroot:nonroot           # never run as root
EXPOSE 8080
ENTRYPOINT ["/app/server"]
```

> **WebSockets and the platform.** A WS server is a **long-lived stateful process** — deploy the Go API on a platform that supports persistent connections (a container host like Fly.io / Railway / Render / a VM / Kubernetes), **not** a request-scoped serverless function that freezes between invocations. Configure generous idle/read timeouts at the proxy for the `/ws` path. If you run **more than one instance**, add the Redis Pub/Sub backplane ([§10.5](#105-backpressure-protecting-the-server-from-slow-clients)) so broadcasts reach clients on every instance, and enable **sticky sessions** (or accept that reconnects may land elsewhere — fine, since clients refetch on reconnect).

### 20.3 Deploying the Next.js dashboard **[A]**

The dashboard is a client-heavy Next.js app. Build it (`npm run build`) and deploy to any Next-capable host (Vercel, or a Node container behind Nginx). Set the **public** env vars at build/deploy time:

```bash
NEXT_PUBLIC_API_URL=https://api.example.com/api/v1   # or /api/v1 if same-origin (below)
NEXT_PUBLIC_WS_URL=wss://api.example.com/ws
```

### 20.4 One origin: the reverse-proxy layout that keeps cookies simple **[A]**

The cleanest production topology puts **both apps behind one domain** via a reverse proxy, so the browser sees a single origin and the refresh cookie stays **same-site** ([§6.4](#64-the-refresh-cookie-helpers)):

```
                          ┌──────────────── Nginx / platform proxy ────────────────┐
  https://admin.example.com                                                         │
        │                 │  location /api/  → Go API  (REST)                       │
        ▼                 │  location /ws    → Go API  (WebSocket upgrade headers)  │
   browser ───────────────▶ location /      → Next.js dashboard                    │
                          └────────────────────────────────────────────────────────┘
```

The critical Nginx detail for the WebSocket route is forwarding the upgrade headers:

```nginx
# /ws → Go API, with the Upgrade/Connection headers a WebSocket needs
location /ws {
    proxy_pass http://api_upstream;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;       # REQUIRED for the WS handshake
    proxy_set_header Connection "upgrade";        # REQUIRED
    proxy_set_header Host $host;
    proxy_read_timeout 3600s;                     # don't kill idle live connections
}

location /api/ {
    proxy_pass http://api_upstream;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  # so c.ClientIP() works
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

With this layout, `NEXT_PUBLIC_API_URL` becomes `/api/v1` and `NEXT_PUBLIC_WS_URL` becomes `wss://admin.example.com/ws` — same origin, `SameSite=Lax` works, no cross-site cookie gymnastics, and CORS is barely needed. The [Nginx guide](NGINX_GUIDE.md) covers TLS, load balancing, and WS proxying in full.

### 20.5 Pre-flight production checklist **[A]**

- [ ] `APP_ENV=production`, `gin.ReleaseMode`, auto-migrate **off**.
- [ ] Versioned migrations applied as a deploy step.
- [ ] Strong `JWT_SECRET` from the secret store; rotated from any dev value.
- [ ] Supabase **session pooler** connection string; `sslmode=require` (or `verify-full`).
- [ ] HTTPS/WSS enforced; cookie `Secure=true`; HSTS + CSP headers set.
- [ ] `ALLOWED_ORIGINS` = the real dashboard URL only.
- [ ] Rate limiting on `/auth/*` and `/ws`.
- [ ] Health check (`/healthz`) wired to the platform's probe.
- [ ] Graceful shutdown verified (in-flight requests drain on redeploy).
- [ ] Logs scrubbed of secrets; request-id correlation working.

---

## 21. Gotchas & Best Practices

The mistakes that cost hours, collected. Most live at the *seams* between the two programs — which is exactly where a decoupled architecture is hardest.

### 21.1 Cross-service / CORS / cookies

- **"Works in Postman, fails in the browser."** That's CORS — a browser-only enforcement. Fix it with **server headers** ([§9](#9-cors-the-api-contract--error-envelopes)), never a frontend hack. Check the **preflight** `OPTIONS` response, not just the real request.
- **The refresh cookie isn't sent/stored.** Both sides must opt in: server `AllowCredentials: true` **with an exact origin** (not `*`), frontend `withCredentials: true`. And `SameSite` must fit your topology — `Lax` for same-site, `None; Secure` for cross-site ([§6.4](#64-the-refresh-cookie-helpers), [§9.2](#92-configuring-cors-for-the-dashboard)).
- **Wildcard origin + credentials.** Browsers forbid `Access-Control-Allow-Origin: *` together with credentials. Always echo a specific allowed origin.
- **TS/Go types drift.** A renamed Go field silently breaks the frontend at runtime. Keep `types/api.ts` in lockstep ([§9.4](#94-the-typescript--go-contract)) — ideally generate it.

### 21.2 Auth & tokens

- **Storing the access token in `localStorage`.** The classic XSS-to-takeover bug. Memory for access, HttpOnly cookie for refresh ([§13.1](#131-the-token-storage-decision-on-the-client)).
- **Not asserting the JWT algorithm.** Always pass `WithValidMethods(["HS256"])` and check the method in the keyfunc ([§6.3](#63-validating-tokens--rigorous-algorithm-checking)). Skipping it enables `alg:none`/confusion attacks.
- **Refresh stampede.** Concurrent 401s firing parallel refreshes; coalesce into one shared promise ([§13.3](#133-the-api-client-interceptors-for-auth-and-auto-refresh)).
- **401 vs 403 confusion.** 401 = not authenticated (refresh/redirect); 403 = authenticated but forbidden (show a message). Mixing them creates redirect loops ([§7.3](#73-the-authorization-middleware)).
- **Forgetting role propagation lag.** A role change takes effect at the next access-token refresh (≤15 min) because the role rides in the token. Acceptable; document it, or add a denylist for instant effect ([§6.6](#66-the-authentication-middleware)).

### 21.3 RBAC

- **Enforcing RBAC only on the frontend.** The cardinal sin. Hidden buttons aren't security; the server middleware is ([§17.1](#171-the-golden-rule-one-more-time)).
- **Coarse permission without the ownership check.** `course:update` lets editors update — *which* courses is a row-level check in the service ([§7.4](#74-row-level-ownership-the-editor-owns-their-courses)). Forgetting it is an IDOR hole.
- **Failing open.** Unknown role → deny. `Can` returns false for anything unmapped ([§7.2](#72-modeling-permissions-in-code)).
- **Locking out all admins.** Guard against demoting/deleting the last admin ([§7.5](#75-the-user-management-endpoints-admin-only)).

### 21.4 WebSockets

- **`CheckOrigin: return true`.** Opens you to CSWSH. Use an allowlist ([§10.4](#104-authenticating-the-websocket-upgrade--the-tricky-part)).
- **More than one reader or writer per connection.** Gorilla forbids it; it corrupts the stream. One `readPump`, one `writePump` ([§10.3](#103-the-client--readwrite-pumps-with-heartbeats)).
- **No heartbeat.** Dead connections accumulate. Ping/pong with deadlines evicts them ([§10.3](#103-the-client--readwrite-pumps-with-heartbeats)).
- **Letting a slow client block the hub.** Non-blocking sends + buffered channels + drop-and-disconnect ([§10.5](#105-backpressure-protecting-the-server-from-slow-clients)).
- **Non-idempotent event handlers.** Reconnects replay and your own actions echo; make apply-twice == apply-once ([§16.3](#163-why-idempotency-matters-the-duplicate-event-problem)).
- **Treating the WS as the source of truth.** It's a freshness optimization; refetch on reconnect to fill the gap ([§16.4](#164-reconnection-and-the-missed-events-gap)).
- **Auth header on the WS.** Browsers can't set it on the handshake — use a query token or ticket ([§10.4](#104-authenticating-the-websocket-upgrade--the-tricky-part)).
- **One Hub, multiple instances.** Broadcasts don't cross processes without a Redis backplane ([§10.5](#105-backpressure-protecting-the-server-from-slow-clients)).

### 21.5 Ent / data layer

- **Editing generated Ent code.** Never — edit `ent/schema/*` and `go generate ./ent`.
- **N+1 on the author edge.** Use `WithAuthor()` to eager-load ([§8.3](#83-the-service-layer)).
- **Auto-migrate in production.** Use Atlas versioned migrations ([§20.1](#201-production-configuration-differences)).
- **Direct connection instead of the pooler.** Exhausts Postgres connections under load — use the Supabase pooler ([§3.4](#34-connecting-to-supabase-postgres)).

### 21.6 General best practices

- **Broadcast from the service, not the handler**, so every mutation path notifies ([§11.3](#113-emitting-events-from-the-service)).
- **Keep DTOs separate from entities** in both directions ([§8.2](#82-dtos-separate-wire-shapes-from-the-entity)).
- **One QueryClient; share the cache between REST and WS** so they can't disagree ([§16.1](#161-the-design-one-ws-connection-feeding-the-query-cache)).
- **Fail fast at boot** on missing config ([§3.1](#31-typed-configuration-from-the-environment)).
- **Generic errors to clients, detailed logs server-side** ([§9.3](#93-a-consistent-error-envelope)).

---

## 22. Study Path & Build-to-Learn Projects

### 22.1 How to work through this guide **[B]**

This guide assumes you've met each tool individually. The most efficient path:

1. **Backend language & framework first.** [Go — Language & Patterns](GO_LANG_AND_PATTERNS_GUIDE.md) → [Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md). You must be comfortable with handlers, middleware, and the service/repository split before [§3](#3-backend-foundation--gin-engine-config--supabase-postgres)–[§8](#8-building-the-crud-rest-api-for-courses) land.
2. **The data layer.** [Ent ORM](GO_ENT_ORM_GUIDE.md) (so [§4](#4-modeling-the-domain-with-ent--users-roles--courses) is review) and [Supabase](SUPABASE_GUIDE.md)/[PostgreSQL](POSTGRESQL_GUIDE.md) for the connection model in [§3.4](#34-connecting-to-supabase-postgres).
3. **Auth in depth.** [JWT + Argon2](GO_JWT_ARGON2_GUIDE.md) before [§5](#5-password-security-with-argon2id--the-signuplogin-flow)–[§7](#7-rbac--roles-permissions--authorization-middleware); this guide implements the practical subset, that one explains every threat.
4. **Realtime.** [Gorilla WebSockets](GO_GORILLA_WEBSOCKETS_GUIDE.md) before [§10](#10-realtime-with-gorilla-websockets--the-hub--auth)–[§11](#11-broadcasting-crud-events-to-the-dashboard); the Hub and the one-reader/one-writer rule come from there.
5. **Frontend.** [React 19](REACT_19_GUIDE.md) → [Next.js 16](NEXTJS_16_GUIDE.md) → [TanStack Query](TANSTACK_QUERY_GUIDE.md) → [React Hook Form](REACT_HOOK_FORM_GUIDE.md) → [shadcn/ui](SHADCN_UI_CHEATSHEET.md) before [§12](#12-the-nextjs-admin-dashboard--setup--architecture)–[§17](#17-rbac-on-the-frontend--conditional-ui--route-guards).
6. **Then build this whole system once, top to bottom**, typing every line. Reading it is not the same as wiring CORS, the WS auth handshake, and the optimistic cache yourself.

Then deploy ([§20](#20-deployment--go-api--nextjs--supabase)) and harden ([§19](#19-security-hardening-full-stack)). The integration *is* the lesson — the seams between the two programs are where real systems break.

### 22.2 Build-to-learn projects **[B/I/A]**

Each adds a layer of real-world complexity. Do them in order.

1. **[B] The minimum loop.** Strip to one entity and one role: login, list, create — no RBAC, no realtime. Get the **CORS + Bearer + refresh-cookie** triangle working. This is the hardest *first* hurdle; everything else builds on it.
2. **[I] Full CRUD + RBAC.** Add update/delete, the three roles, the permission middleware, and the row-level ownership rule. Verify every cell of the permission table ([§7.1](#71-roles-permissions-and-why-you-separate-them)) by hitting the API directly with `curl` as each role — prove the server enforces it without the UI.
3. **[I/A] Go live.** Add the WebSocket: the Hub, authenticated upgrade, broadcasts from the service, and the `useCourseSocket` hook writing into the cache. Open two browsers and watch them sync. Then kill the API mid-session and confirm the client reconnects and refetches.
4. **[A] Optimistic everything.** Make create and update optimistic (not just delete), with rollback on the server's 403. Add a connection-status indicator and a toast on rollback.
5. **[A] Add a second resource with a relationship.** Add `Lessons` belonging to a `Course` (a new Ent edge). Reuse the entire pattern — schema → service → handlers → RBAC → broadcast → hooks → UI — and feel how the architecture scales. Add a `course.lessons` eager-load and a nested permission (`lesson:create`).
6. **[A] Scale the realtime tier.** Run two API instances behind Nginx and add a **Redis Pub/Sub backplane** so broadcasts cross instances ([§10.5](#105-backpressure-protecting-the-server-from-slow-clients)). Load-test with many concurrent WS clients.
7. **[A] Harden to production.** Add rate limiting on auth, a denylist for instant user deactivation, CSP headers, Atlas versioned migrations, and deploy behind one origin ([§20.4](#204-one-origin-the-reverse-proxy-layout-that-keeps-cookies-simple)). Run the [§20.5](#205-pre-flight-production-checklist) checklist.

### 22.3 Where to go next **[A]**

- **More auth:** OTP, MFA/TOTP, OAuth/social login, passkeys — all in the [JWT + Argon2 guide](GO_JWT_ARGON2_GUIDE.md). Bolt them onto this auth service.
- **A different API shape:** swap REST for [GraphQL (gqlgen)](GO_GRAPHQL_GUIDE.md) with subscriptions instead of raw WebSockets, or [gRPC](GO_GRPC_RPC_GUIDE.md) for service-to-service.
- **The all-Next.js alternative:** compare this decoupled design against the single-codebase [Next.js Full-Stack capstone](NEXTJS_FULLSTACK_APP_GUIDE.md) to internalize *when* each architecture wins.
- **Operate it:** [Docker](DOCKER_GUIDE.md) → [Nginx](NGINX_GUIDE.md) → [CI/CD with GitHub Actions](GITHUB_ACTIONS_CICD_GUIDE.md) → [Linux Server Admin](LINUX_SERVER_ADMIN_GUIDE.md) to run it like a company would.

You've built a complete, secure, realtime, role-aware full-stack system across two languages and two programs — and, more importantly, you understand the *seams* that hold them together. That understanding is what separates "I followed a tutorial" from "I can architect this." Go build the next one from scratch.

---

*Part of the offline study library. Pair this guide with the single-tool guides it composes — it is deliberately the integration layer on top of them. Built for 2026; confirm fast-moving package APIs against each tool's dedicated guide and official docs before production.*
