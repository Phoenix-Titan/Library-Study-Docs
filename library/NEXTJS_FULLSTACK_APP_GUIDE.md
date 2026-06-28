# Standalone Full-Stack Next.js for Production (Banking-Grade) — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Developers who can write some React and some Next.js and now want to build **one cohesive, production-grade, banking-grade-secure full-stack application** — the kind where the same Next.js codebase *is* the frontend *and* the backend, backed by Prisma + PostgreSQL/Supabase, protected by Better Auth with OTP, MFA, OAuth and RBAC, and pushing live updates over WebSockets. This is an **explain-first** guide: every concept is taught in prose (what it is, *why* the architecture pushes you this way, when and how to use it, the key options, best practices, and — because money and identity are on the line — explicit **security implications**) before any heavily-commented code. This guide is the **capstone** of the library: it does not re-teach each tool in full. Instead it teaches you to **compose** them. Wherever a topic has a dedicated guide, this one cross-references it and shows only the integration glue. It remains self-sufficient enough to build a secure app end-to-end. Read it top-to-bottom once; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Next.js 16 (App Router)** on **React 19**, **Prisma 6**, **Better Auth v1.x**, **Supabase (2026)**, **Node.js 24 LTS**, and **TypeScript 5.x** — all current as of **June 2026**. Specific fast-moving APIs are flagged with **⚡ Version note**. Next.js, Prisma and Better Auth all evolve quickly; confirm exact symbol names against the dedicated guides and official docs, and trust your editor's inferred types over memory.
>
> **This guide's place in the library — read these alongside it:** [Next.js 16](NEXTJS_16_GUIDE.md) (rendering, caching, the App Router), [React 19](REACT_19_GUIDE.md) (Server Components, `use`, Actions), [Prisma ORM](PRISMA_ORM_GUIDE.md) (schema, migrations, queries), [Supabase](SUPABASE_GUIDE.md) (managed Postgres/Auth/Storage/Realtime + RLS), [Better Auth](BETTERAUTH_GUIDE.md) (the identity layer), [Go JWT + Argon2 (Banking-Grade Auth)](GO_JWT_ARGON2_GUIDE.md) (the *concepts & threat models* for OTP/MFA/OAuth/passkeys/rate-limiting/encryption — the language is Go but the security reasoning transfers directly to TypeScript), [PostgreSQL](POSTGRESQL_GUIDE.md), [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md), [Redis](REDIS_GUIDE.md), [TanStack Query](TANSTACK_QUERY_GUIDE.md), [Zustand](ZUSTAND_GUIDE.md), [React Hook Form](REACT_HOOK_FORM_GUIDE.md), [Node.js](NODEJS_GUIDE.md), [Docker](DOCKER_GUIDE.md), [Nginx](NGINX_GUIDE.md), and [Networking](NETWORKING_GUIDE.md).

---

## Table of Contents

1. [What "Standalone Full-Stack Next.js" Means & the Architecture Decision](#1-what-standalone-full-stack-nextjs-means--the-architecture-decision) **[B]**
2. [Project Setup & Structure](#2-project-setup--structure) **[B]**
3. [The Data Layer — Prisma + PostgreSQL](#3-the-data-layer--prisma--postgresql) **[B/I]**
4. [Supabase — Managed Postgres, Storage, Realtime & RLS in Depth](#4-supabase--managed-postgres-storage-realtime--rls-in-depth) **[I/A]**
5. [Data Fetching & Mutations — Server Components, Server Actions, Route Handlers](#5-data-fetching--mutations--server-components-server-actions-route-handlers) **[I/A]**
6. [Authentication with Better Auth](#6-authentication-with-better-auth) **[I/A]**
7. [OTP & Email/Phone Verification](#7-otp--emailphone-verification) **[A]**
8. [MFA / Two-Factor Authentication](#8-mfa--two-factor-authentication) **[A]**
9. [OAuth / Social Login](#9-oauth--social-login) **[A]**
10. [RBAC & Authorization](#10-rbac--authorization) **[I/A]**
11. [Realtime with WebSockets](#11-realtime-with-websockets) **[A]**
12. [Banking-Grade Security Hardening](#12-banking-grade-security-hardening) **[A]**
13. [Performance & Scaling](#13-performance--scaling) **[A]**
14. [Observability](#14-observability) **[I/A]**
15. [Testing](#15-testing) **[A]**
16. [Deployment & Production](#16-deployment--production) **[A]**
17. [Gotchas & Best Practices](#17-gotchas--best-practices) **[I/A]**
18. [Study Path & Build-to-Learn Projects](#18-study-path--build-to-learn-projects)

---

## 1. What "Standalone Full-Stack Next.js" Means & the Architecture Decision

### 1.1 The core idea **[B]**

A "standalone full-stack Next.js application" is one where **a single Next.js codebase provides both the user interface and the backend** — there is no separate Express/Nest/Go API service sitting beside it for the core app. The same project that renders your pages also reads and writes the database, authenticates users, enforces authorization, talks to third-party APIs, and emits realtime events. You deploy *one* artifact (or one image), version *one* repository, and share *one* set of TypeScript types across the whole stack.

This is possible because Next.js 16's App Router gives you three distinct *server-side* execution surfaces, all running on the server, never shipped to the browser:

1. **Server Components (RSC)** — components that render on the server and can directly `await` database queries, read secrets, and call internal services. Their output is serialized to the client as HTML plus a compact React payload; *their source code and the data they touch never reach the browser*. This is where most of your *reads* live.
2. **Server Actions** — `async` functions marked `"use server"` that the client can invoke like an RPC. They run on the server, validate input, mutate the database, and revalidate caches. This is where most of your *writes* live.
3. **Route Handlers** — `app/**/route.ts` files exporting `GET`/`POST`/etc. that build a Web-standard `Request → Response`. This is your *public HTTP API*: webhooks, OAuth callbacks, mobile-client endpoints, file streaming, anything that needs a real URL.

Understanding *why* this exists matters more than the mechanics. Historically, "full stack" meant two codebases: a React SPA and a separate API. That split forced you to design, document, version and secure an HTTP contract between them, duplicate validation, duplicate types, and pay a network hop for every read. Next.js collapses the boundary: a Server Component calling `prisma.account.findMany()` is *just a function call*, type-checked end to end, with no serialization contract to maintain. The framework is, in effect, your backend.

### 1.2 When you still need a separate API or service **[B/I]**

The single-codebase model is the right default, but it is not universal. Reach for a **separate service** when:

- **You need a long-lived process.** Serverless/edge deployments tear down between requests, so a persistent **WebSocket server**, a **background job worker**, a **cron scheduler**, or a **message-queue consumer** does not belong inside the Next.js request path. These run as separate Node services (see §11 and §14). *This is the single most common reason banking-style apps add a sidecar service.*
- **Multiple frontends share one backend.** If a native mobile app, a partner integration, and the web app all consume the same domain logic, a dedicated API (Go, Nest, Fastify) with a versioned, documented contract is cleaner than every client calling Next.js internals. You can still keep Next.js as one of its consumers.
- **Heavy CPU or specialized runtimes.** Cryptographic batch processing, ML inference, or PDF generation can be offloaded to a service tuned for it, keeping your web tier responsive.
- **Regulatory isolation.** A card-data or PII vault may need to live in a separate, hardened, separately-audited deployment boundary (PCI-DSS-style segmentation).

A pragmatic banking-grade architecture is therefore often: **Next.js as the full-stack web tier**, plus **a small Node WebSocket service**, plus **Postgres (Supabase or self-hosted)**, plus **Redis** (sessions/rate-limits/cache/pub-sub), behind **Nginx** as a TLS-terminating reverse proxy. Every other piece is optional until a concrete need appears. See [Networking](NETWORKING_GUIDE.md) and [Nginx](NGINX_GUIDE.md) for the edge.

It is worth being precise about what lives where, because the naming overlaps confuse newcomers. **Server Components** are your *read* path: they render on the server and stream HTML. **Server Actions** are your *write* path: RPC-style functions invoked from the client but executed on the server. **Route Handlers** are your *public HTTP* path: addressable URLs for webhooks, OAuth callbacks, and non-browser clients. All three are "the backend" — there is no fourth thing called "the API" sitting elsewhere. When someone asks "where's the backend?", the honest answer for a standalone Next.js app is "it's the server-side half of this same codebase, split across these three surfaces."

A simple way to reason about the decision is to ask three questions of any piece of functionality. *Does it need to outlive a single request?* If yes (sockets, queues, schedulers), it is a separate process. *Is it consumed by clients other than this web app?* If yes (mobile, partners), a versioned API is cleaner. *Does it require a runtime or isolation Next.js can't provide?* If yes (heavy CPU, a hardened PCI boundary), split it out. If all three answers are "no", it belongs inside the Next.js codebase, and putting it there is the *simpler, safer* choice — fewer moving parts, fewer network hops to secure, fewer contracts to keep in sync. Resist the instinct, carried over from the SPA-plus-API era, to build a separate API "because that's how backends are done". In the App Router world, the framework *is* the backend until a concrete constraint forces a split.

### 1.2.1 How this compares to the alternatives **[B/I]**

It helps to place the standalone model against the architectures you may already know:

| Architecture | Frontend | Backend | Type sharing | Network hop per read | Best when |
|---|---|---|---|---|---|
| SPA + REST API | React SPA | Separate (Express/Nest/Go) | Manual (OpenAPI/codegen) | Always | Many clients, polyglot teams |
| SPA + GraphQL | React SPA | Separate GraphQL server | Schema + codegen | Always | Complex client-driven querying |
| **Standalone Next.js** | RSC + Client Components | Same codebase (RSC/Actions/Handlers) | **Automatic (one tsconfig)** | **None for RSC** | One primary web client, one team |
| Next.js + thin API | RSC + Client Components | Next.js + a sidecar service | Mostly automatic | Only for sidecar calls | Web app that *also* needs realtime/jobs |

The standout advantage of the standalone model is the **eliminated contract**. In a SPA-plus-API setup, a change to a field name means: update the API type, regenerate the client types, update the client, deploy both in a compatible order. In standalone Next.js, it is one rename that the compiler propagates everywhere instantly. For a small-to-medium team shipping one main web product — which describes a great many banking, fintech, and SaaS apps — this is a large, sustained productivity and *correctness* win, because the most common source of bugs (drift between two independently-deployed sides of a contract) simply cannot occur.

### 1.3 The request lifecycle **[B/I]**

Trace one navigation to internalize where code runs:

1. A request hits the **edge / reverse proxy** ([Nginx](NGINX_GUIDE.md)), which terminates TLS and forwards to the Next.js server.
2. **`middleware.ts`** runs first (Node runtime in Next.js 16). It is the right place for *coarse* gating — redirect unauthenticated users, attach security headers, set request IDs — but **not** for authoritative authorization (§10.6 explains why).
3. The matched **layout(s) and page** render as **Server Components**, `await`-ing data directly. Streaming + Suspense let the static shell flush immediately while dynamic holes resolve.
4. The browser receives HTML + the RSC payload and **hydrates** interactive **Client Components**.
5. A user action triggers either a **Server Action** (preferred for mutations) or a `fetch` to a **Route Handler**. Either runs on the server, re-validates everything, mutates, and revalidates affected caches.
6. Realtime updates arrive over a **WebSocket** from the separate WS service (§11), not from the Next.js request path.

The practical takeaway for security: there are **multiple independent entry points** to your server (page render, Server Action, Route Handler, and the WS service), and *each one* is independently reachable by an attacker. A user does not have to go through your UI to call a Server Action or hit a Route Handler — they can craft the request directly. This is why the authorization checklist (§5.2, §10) must be applied *at every entry point*, not once at the "front door". There is no front door; there are many doors, and each needs its own lock.

### 1.4 The security boundary: server vs client **[B/I]** ⚡ Version note

This is the most important mental model in the entire guide, so internalize it before writing a line of code: **everything in a Client Component, and every value prefixed `NEXT_PUBLIC_`, is public.** It is shipped in the JavaScript bundle, visible in DevTools, and readable by anyone. Everything in a Server Component, Server Action, or Route Handler stays on the server *as long as you do not pass it across the boundary*.

The boundary is crossed in exactly two directions, and each is a security checkpoint:

- **Server → Client:** props passed from a Server Component to a Client Component, and the return value of a Server Action, are **serialized and sent to the browser**. If you pass a full Prisma `user` row to a client component "just to show the name", you have leaked the password hash, the TOTP secret, the email, and every other column. *Always project to a minimal, explicit DTO at the boundary.*
- **Client → Server:** every argument a client sends to a Server Action or Route Handler is **attacker-controlled**. The user can call your Server Action from the console with any payload. *Never trust it; re-authenticate, re-authorize, and re-validate every single time* (§5.3, §12).

Next.js 16's React Compiler and `"use cache"` change *how* code is optimized but do not change this boundary. The rule is invariant: **the network is the trust boundary, and the client is hostile.**

A concrete illustration of the most common leak, so you can recognize it in review:

```tsx
// LEAK — passing the full Prisma row to a client component ships EVERY column.
import { ProfileCard } from "./ProfileCard"; // "use client"
export default async function Page() {
  const user = await db.user.findUnique({ where: { id } }); // includes passwordHash, twoFactorSecret...
  return <ProfileCard user={user} />;  // ❌ all of it is now in the browser's RSC payload
}

// SAFE — project to an explicit DTO at the boundary; only these fields cross.
export default async function Page() {
  const user = await db.user.findUnique({
    where: { id },
    select: { id: true, name: true, email: true }, // ✅ minimal, intentional
  });
  return <ProfileCard user={user} />;
}
```

The fix is mechanical but the discipline is everything: **never pass a raw database row across the boundary.** Make `select` (or a mapping function) the habit, and your reviewers can verify safety by reading the `select` clause. This single habit prevents a large fraction of real-world PII leaks.

> **Reference — where code runs and what it may touch**

| Surface | Runs on | May read secrets? | May query DB directly? | Shipped to browser? | Use for |
|---|---|---|---|---|---|
| Server Component | Server | Yes | Yes | No (output only) | Reads, page rendering |
| Server Action | Server | Yes | Yes | No (callable handle only) | Mutations, form submits |
| Route Handler | Server | Yes | Yes | No | Public API, webhooks, OAuth callbacks |
| `middleware.ts` | Server (Node rt) | Yes (be careful) | Avoid (keep it fast) | No | Coarse gating, headers, redirects |
| Client Component | Browser | **No** | **No** | **Yes** | Interactivity, local state |

---

## 2. Project Setup & Structure

### 2.1 Scaffolding **[B]**

Start from the official scaffold and choose TypeScript, the App Router, and Turbopack. Use a current LTS Node (24) so that crypto, `fetch`, and the Web Streams APIs behave consistently between dev and production.

```bash
# Node 24 LTS recommended; verify first.
node -v   # v24.x

# Scaffold: App Router, TypeScript, ESLint, Tailwind, src/ dir, import alias.
npx create-next-app@latest banking-app \
  --typescript --eslint --app --src-dir --import-alias "@/*" --turbopack

cd banking-app

# Core dependencies for this guide's stack.
npm i @prisma/client zod
npm i -D prisma
npm i better-auth                      # identity layer (§6)
npm i @supabase/supabase-js @supabase/ssr   # Supabase client + SSR cookie bridge (§4)
npm i @tanstack/react-query            # client server-state cache (§5.5)
npm i zustand                          # client UI state (§5.6)
npm i react-hook-form @hookform/resolvers  # forms (§5.7)
npm i ioredis                          # Redis for rate-limit/cache/pub-sub (§12.10, §13)
```

See [Next.js 16 §2](NEXTJS_16_GUIDE.md) for scaffold details and [Node.js](NODEJS_GUIDE.md) for runtime specifics.

A note on the runtime version: pin Node to the same major version (24 LTS) across local dev, CI, and production. Subtle differences in the `crypto`, `fetch`, and Web Streams implementations between Node majors can cause "works on my machine" bugs in exactly the security-sensitive code (AES-GCM, signature verification) where you least want surprises. A `.nvmrc` (or the `engines` field in `package.json`) plus a CI check that the running version matches keeps everyone aligned. ⚡ **Version note:** Turbopack is the default bundler in Next.js 16 for both dev and build; the legacy Webpack path still exists but is not where the project is investing — prefer Turbopack unless a specific plugin forces otherwise.

### 2.2 Folder structure — feature-first **[B]**

For a small app the default `app/` tree is fine. For a banking-grade app that will grow, organize by **feature** (a vertical slice owns its UI, actions, data access, and validation) rather than by technical layer. The key discipline: a clear, *enforced* boundary between server-only code and code that may be imported by client components.

```
src/
  app/                      # routing only — thin pages/layouts/route handlers
    (marketing)/            # public route group
    (app)/                  # authenticated route group, shares an auth layout
      dashboard/page.tsx
      accounts/[id]/page.tsx
    api/
      auth/[...all]/route.ts      # Better Auth handler (§6)
      webhooks/stripe/route.ts    # signed webhook (§5.4)
    layout.tsx
    middleware.ts                  # (lives at src/ root in src-dir setups)
  features/
    accounts/
      actions.ts            # "use server" mutations for this feature
      data.ts               # server-only DB reads (import "server-only")
      schema.ts             # Zod schemas (shared client+server)
      components/           # feature UI (mix of server/client components)
    transfers/
      ...
  lib/
    db.ts                   # Prisma singleton (§3.3)
    auth.ts                 # betterAuth() server instance (§6.2)
    auth-client.ts          # createAuthClient() (client)
    supabase/
      server.ts             # @supabase/ssr server client (§4.3)
      client.ts             # browser client (anon key only)
    rbac.ts                 # permission checks (§10)
    rate-limit.ts           # Redis limiter (§12.10)
    env.ts                  # validated, typed env (§2.4)
    redis.ts
  components/ui/            # shared, app-agnostic UI primitives
prisma/
  schema.prisma
  migrations/
```

The reason to organize by feature rather than by technical layer (a `controllers/` folder, a `services/` folder, a `models/` folder) is **cohesion and change locality**. When you build or modify the "transfers" capability, everything you touch — its UI, its server actions, its data access, its validation — lives in `features/transfers/`, so a change is a focused, reviewable diff rather than a scatter across five top-level folders. It also keeps the security-critical files (`actions.ts`, `data.ts`) next to the feature they protect, so a reviewer evaluating "is the transfer flow secure?" reads one directory. As the app grows past a handful of features this structure scales far better than layer-folders, which grow into giant grab-bags where the relationship between files is invisible.

Two conventions make the boundary safe:

- Put `import "server-only"` at the top of any module that must never reach the browser (`lib/db.ts`, `features/*/data.ts`, anything touching secrets). If a client component imports it, the build **fails loudly** instead of silently leaking. Conversely, `import "client-only"` guards browser-only modules.
- Keep `actions.ts` (mutations) and `data.ts` (reads) separate from components. It makes the auth/validation checkpoints easy to audit — a reviewer can read every server entry point in one place.

### 2.2.1 Route groups and the authenticated layout **[B/I]**

The parentheses folders — `(marketing)` and `(app)` — are **route groups**: they organize routes and let you apply different layouts *without* adding a URL segment (`(app)/dashboard` serves at `/dashboard`, not `/app/dashboard`). This is the idiomatic way to separate your public surface from your authenticated surface. The `(app)` group gets a shared layout that performs the coarse auth gate and renders the app chrome (nav, user menu), so every page underneath inherits it.

```tsx
// src/app/(app)/layout.tsx — shared layout for all authenticated routes
import { requireUser } from "@/lib/auth";

export default async function AppLayout({ children }: { children: React.ReactNode }) {
  const user = await requireUser();   // gate: unauthenticated users are redirected here
  return (
    <div className="app-shell">
      <AppNav user={{ name: user.name, role: user.role }} />  {/* minimal DTO, not the full row */}
      <main>{children}</main>
    </div>
  );
}
```

Even with this layout-level gate, **every page and action still re-checks** authorization (§10) — the layout is a convenience and a redirect, not the security boundary. Two reasons: a layout runs once and may be cached/streamed differently than you expect, and per-object ownership (§10.5) can only be checked where the object id is known, which is in the page/action, not the layout.

### 2.2.2 `next.config` essentials **[B/I]** ⚡ Version note

A few config choices matter for a production self-hosted app: `output: "standalone"` (a minimal server bundle for a small Docker image — §16.1), `poweredByHeader: false` (don't advertise the framework), `reactStrictMode: true`, and your security `headers()` (§12.9). Keep image domains tight (only allow remote image hosts you trust) to avoid turning your image optimizer into an open proxy (an SSRF vector, §12.8).

```ts
// next.config.ts (essentials; merge with the headers() block from §12.9)
import type { NextConfig } from "next";
const config: NextConfig = {
  output: "standalone",
  poweredByHeader: false,
  reactStrictMode: true,
  images: { remotePatterns: [{ protocol: "https", hostname: "cdn.example.com" }] }, // allowlist only
};
export default config;
```

### 2.3 The `NEXT_PUBLIC_` rule **[B]** ⚡ Version note

Next.js inlines any environment variable named `NEXT_PUBLIC_*` into the **client bundle at build time**. That is the *entire* mechanism for exposing config to the browser — and it is a one-way door. Anything you must keep secret (database URLs, the Supabase `service_role` key, Better Auth secret, OAuth client secrets, encryption keys, webhook signing secrets) must **never** carry that prefix. A leaked `service_role` key bypasses every Row Level Security policy in your database — it is a full database compromise.

| Variable | Prefix | Where readable | Example |
|---|---|---|---|
| `DATABASE_URL` | none | server only | Postgres connection |
| `BETTER_AUTH_SECRET` | none | server only | session/cookie signing |
| `SUPABASE_SERVICE_ROLE_KEY` | none | **server only — RLS bypass** | privileged Supabase |
| `ENCRYPTION_KEY` | none | server only | encrypt TOTP secrets at rest |
| `NEXT_PUBLIC_SUPABASE_URL` | public | browser + server | project URL (not secret) |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | public | browser + server | anon key (RLS-gated, safe to ship) |
| `NEXT_PUBLIC_APP_URL` | public | browser + server | canonical origin |

A useful habit: grep your built bundle (`.next/`) for a known secret value before every release; CI can fail the build if it finds one (§12.7).

### 2.4 Typed, validated environment **[B/I]**

Do not read `process.env.X` scattered through the codebase. A missing or malformed secret should crash the app **at boot**, not page a developer at 3 a.m. when a code path first hits it. Validate the whole environment once with Zod and export a typed object.

```ts
// src/lib/env.ts
import "server-only";          // guarantees this never reaches the client
import { z } from "zod";

const schema = z.object({
  DATABASE_URL: z.string().url(),
  DIRECT_URL: z.string().url().optional(),       // unpooled, for migrations (§13.4)
  BETTER_AUTH_SECRET: z.string().min(32),        // 256-bit+ signing secret
  BETTER_AUTH_URL: z.string().url(),
  SUPABASE_SERVICE_ROLE_KEY: z.string().min(20),
  ENCRYPTION_KEY: z.string().length(64),         // 32 bytes hex for AES-256-GCM (§8.3)
  REDIS_URL: z.string().url(),
  GITHUB_CLIENT_ID: z.string().optional(),
  GITHUB_CLIENT_SECRET: z.string().optional(),
  NEXT_PUBLIC_SUPABASE_URL: z.string().url(),
  NEXT_PUBLIC_SUPABASE_ANON_KEY: z.string().min(20),
  NEXT_PUBLIC_APP_URL: z.string().url(),
});

// Throws at startup if anything is missing/invalid — fail fast, fail loud.
export const env = schema.parse(process.env);
```

See [Next.js 16 §18](NEXTJS_16_GUIDE.md) for the full environment-and-config treatment.

**Which `.env` file, and what's committed.** Next.js loads several files in a defined precedence: `.env` (defaults, may be committed if it holds no secrets), `.env.local` (machine-specific overrides, **gitignored** — your real dev secrets live here), and `.env.production`/`.env.development` (per-environment). The rules that keep you safe: **never commit a file containing real secrets** (add `.env*.local` to `.gitignore`, which `create-next-app` does by default), commit a **`.env.example`** listing the *names* of required variables with placeholder values (so a teammate knows what to provide), and in production inject secrets through the platform's secret store rather than any file (§12.7). The Zod validation in `env.ts` then runs against whatever was loaded, failing the boot if anything is missing — turning "forgot to set a secret" from a runtime mystery into a clear startup error.

```bash
# .env.example — committed; documents required vars WITHOUT real values.
DATABASE_URL=
DIRECT_URL=
BETTER_AUTH_SECRET=          # generate: openssl rand -base64 48
ENCRYPTION_KEY=              # generate: openssl rand -hex 32
REDIS_URL=
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

---

## 3. The Data Layer — Prisma + PostgreSQL

This section is the integration view; for the full ORM treatment read [Prisma ORM](PRISMA_ORM_GUIDE.md), and for schema design principles (normalization, keys, indexing, constraints) read [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md) and [PostgreSQL](POSTGRESQL_GUIDE.md).

### 3.1 Why Prisma, and where it runs **[B]**

Prisma is a type-safe ORM: you describe your data model in `schema.prisma`, Prisma generates a fully-typed client, and your queries are checked at compile time. For a banking-grade app this buys you three things that matter: **parameterized queries by construction** (Prisma never string-concatenates your inputs into SQL, so SQL injection is off the table for ordinary queries — §12.4), **migrations as reviewable, versioned artifacts** (every schema change is a file in `prisma/migrations/`, auditable like code), and **end-to-end types** (a column rename surfaces as a compile error across the whole app).

Critically, **Prisma runs only on the server**. The Prisma client opens a TCP connection to Postgres and must hold your `DATABASE_URL`; it can never be imported into a Client Component. Enforce this with `import "server-only"` in the module that constructs it (§3.3). In Next.js, Prisma lives inside Server Components, Server Actions, and Route Handlers — never anywhere a browser can reach.

### 3.2 A representative schema **[B/I]**

A fintech-shaped schema needs identity tables (we let Better Auth own most of these — §6.3), domain tables (accounts, transactions), and security/audit tables. Use `BigInt`/`Decimal` for money (**never floating point** — `0.1 + 0.2 ≠ 0.3` is unacceptable for a ledger), enforce invariants with database constraints, and index every foreign key and every column you filter on.

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")   // pooled connection for the app (§13.4)
  directUrl = env("DIRECT_URL")     // unpooled, used by `prisma migrate`
}

// --- Identity (extends Better Auth's user table; §6.3) ---
model User {
  id            String   @id @default(cuid())
  email         String   @unique
  emailVerified Boolean  @default(false)
  name          String?
  role          Role     @default(CUSTOMER)   // RBAC anchor (§10)
  twoFactor     TwoFactor?
  accounts      Account[]
  auditLogs     AuditLog[]
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
}

enum Role {
  CUSTOMER
  SUPPORT
  ADMIN
}

// --- Domain: money lives here ---
model Account {
  id           String        @id @default(cuid())
  userId       String                              // ownership anchor (IDOR defense, §10.5)
  user         User          @relation(fields: [userId], references: [id], onDelete: Cascade)
  currency     String        @default("USD")
  // Store money as the smallest unit (cents) in a BigInt, OR use Decimal. Never Float.
  balanceCents BigInt        @default(0)
  status       AccountStatus @default(ACTIVE)
  transactions Transaction[]
  createdAt    DateTime      @default(now())

  @@index([userId])           // fast "accounts for this user" + ownership checks
}

enum AccountStatus { ACTIVE FROZEN CLOSED }

model Transaction {
  id          String   @id @default(cuid())
  accountId   String
  account     Account  @relation(fields: [accountId], references: [id])
  amountCents BigInt                              // signed: negative = debit
  kind        String                              // "transfer_in" | "transfer_out" | ...
  // Idempotency key prevents a retried request from double-charging (§5.4).
  idempotencyKey String? @unique
  createdAt   DateTime @default(now())

  @@index([accountId, createdAt])  // statement queries: latest first per account
}

// --- Security / verification (hashed, single-use, short-lived; §7, §8) ---
model OneTimeCode {
  id        String   @id @default(cuid())
  userId    String
  purpose   String                                 // "email_verify" | "login_otp" | ...
  codeHash  String                                 // store the HASH, never the code (§7.2)
  expiresAt DateTime
  consumedAt DateTime?                             // single-use enforcement
  attempts  Int      @default(0)                   // throttle brute force
  createdAt DateTime @default(now())

  @@index([userId, purpose])
}

model TwoFactor {
  id            String   @id @default(cuid())
  userId        String   @unique
  user          User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  secretEnc     String                             // TOTP secret, AES-256-GCM encrypted (§8.3)
  verifiedAt    DateTime?
  recoveryCodes RecoveryCode[]
}

model RecoveryCode {
  id          String   @id @default(cuid())
  twoFactorId String
  twoFactor   TwoFactor @relation(fields: [twoFactorId], references: [id], onDelete: Cascade)
  codeHash    String                              // hashed, single-use (§8.4)
  usedAt      DateTime?
}

// --- Audit trail: append-only record of security-relevant events (§12.11) ---
model AuditLog {
  id        String   @id @default(cuid())
  userId    String?
  user      User?    @relation(fields: [userId], references: [id])
  action    String                                // "login.success", "transfer.create", ...
  ip        String?
  userAgent String?
  metadata  Json?
  createdAt DateTime @default(now())

  @@index([userId, createdAt])
  @@index([action, createdAt])
}
```

```bash
# Create and apply a migration (dev). Generates SQL in prisma/migrations/.
npx prisma migrate dev --name init
# Regenerate the typed client after schema changes.
npx prisma generate
```

**A word on representing money.** Notice `balanceCents` is a `BigInt` holding the smallest currency unit (cents), not a `Float` holding dollars. This is non-negotiable for a ledger. Binary floating-point cannot exactly represent most decimal fractions, so `0.1 + 0.2` is `0.30000000000000004` — and a bank that loses a fraction of a cent per transaction across millions of transactions has a real accounting problem and an audit failure. Two correct approaches: store integer **minor units** (`BigInt` cents) and divide by 100 only for *display* (as the UI code does), or use Postgres `NUMERIC`/Prisma `Decimal` for exact decimal arithmetic. Pick one and be consistent. Also: store the **currency** alongside every amount (you can't add USD to EUR), and never do currency math by mixing the two — convert explicitly through a rate at a known time, and record the rate used in the audit trail. These habits look pedantic until the first reconciliation, when they are the difference between books that balance and books that don't.

### 3.3 The Next.js Prisma singleton **[B/I]**

In development, Next.js hot-reloads modules on every save. A naive `new PrismaClient()` at module scope would create a **new client (and a new connection pool) on every reload**, quickly exhausting Postgres connections (`too many clients already`). The fix is the canonical singleton: cache the client on `globalThis` so reloads reuse it.

```ts
// src/lib/db.ts
import "server-only";              // CRITICAL: never let this reach the browser
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as { prisma?: PrismaClient };

export const db =
  globalForPrisma.prisma ??
  new PrismaClient({
    // Log slow/erroring queries in dev; keep production logs structured (§14).
    log: process.env.NODE_ENV === "development" ? ["warn", "error"] : ["error"],
  });

// Only cache in dev — production starts fresh each boot.
if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = db;
```

### 3.4 Relations, the N+1 trap, and transactions **[I]**

**Avoid N+1 queries.** The N+1 trap is fetching a list and then querying related data per item in a loop — 1 query becomes 1 + N. With Prisma, fetch relations in one round-trip using `include`/`select`, or batch with a single `findMany({ where: { id: { in: ids } } })`. Always `select` only the columns you need; this both improves performance and reduces the risk of over-fetching sensitive columns to the boundary.

```ts
// BAD — N+1: one query for accounts, then one per account for transactions.
const accounts = await db.account.findMany({ where: { userId } });
for (const a of accounts) {
  a.tx = await db.transaction.findMany({ where: { accountId: a.id } }); // N queries
}

// GOOD — single query, and select only what the UI needs (boundary hygiene).
const accounts = await db.account.findMany({
  where: { userId },
  select: {
    id: true,
    currency: true,
    balanceCents: true,
    transactions: {
      select: { id: true, amountCents: true, kind: true, createdAt: true },
      orderBy: { createdAt: "desc" },
      take: 20,
    },
  },
});
```

**Transactions** make multi-step money operations atomic — either every step commits or none does. A transfer must debit one account and credit another *together*; a partial apply would create or destroy money. Use an **interactive transaction** so you can read, branch, and write inside one atomic unit, and let the database enforce invariants.

```ts
// A money transfer: atomic, with an ownership check and an idempotency guard.
export async function transfer(opts: {
  userId: string; fromId: string; toId: string; amountCents: bigint; idempotencyKey: string;
}) {
  return db.$transaction(async (tx) => {
    // Idempotency: if we've already processed this key, do nothing (safe retries).
    const dup = await tx.transaction.findUnique({ where: { idempotencyKey: opts.idempotencyKey } });
    if (dup) return { ok: true, idempotent: true };

    // Ownership + state check inside the transaction (defense vs IDOR + race; §10.5).
    const from = await tx.account.findFirst({
      where: { id: opts.fromId, userId: opts.userId, status: "ACTIVE" },
    });
    if (!from) throw new Error("Source account not found or not yours");
    if (from.balanceCents < opts.amountCents) throw new Error("Insufficient funds");

    await tx.account.update({ where: { id: opts.fromId }, data: { balanceCents: { decrement: opts.amountCents } } });
    await tx.account.update({ where: { id: opts.toId },   data: { balanceCents: { increment: opts.amountCents } } });
    await tx.transaction.create({ data: {
      accountId: opts.fromId, amountCents: -opts.amountCents, kind: "transfer_out", idempotencyKey: opts.idempotencyKey,
    }});
    await tx.transaction.create({ data: {
      accountId: opts.toId, amountCents: opts.amountCents, kind: "transfer_in",
    }});
    return { ok: true };
  });
}
```

> **Note on concurrency:** for a real ledger, also guard against concurrent transfers draining a balance with either a `SELECT ... FOR UPDATE` (raw SQL inside the transaction), an optimistic `version` column, or a serializable isolation level. Prisma exposes isolation via `db.$transaction(fn, { isolationLevel: "Serializable" })`. See [PostgreSQL](POSTGRESQL_GUIDE.md) on isolation and locking.

### 3.5 The race condition every money app must handle **[I/A]**

Consider what happens if the same user fires two transfers of \$80 each from an account holding \$100, at the same instant, on two app instances. Without protection, both transactions read `balance = 100`, both see "sufficient funds", both decrement, and the account ends at `-60` — money created from nothing. This is a **time-of-check-to-time-of-use (TOCTOU)** race, and it is the canonical concurrency bug in financial software. There are three standard defenses, in increasing strictness:

1. **Pessimistic row lock (`SELECT ... FOR UPDATE`).** The first transaction to read the row locks it; the second blocks until the first commits, then reads the *updated* balance and correctly rejects. This is the most explicit and predictable approach for a ledger.
2. **Optimistic concurrency (a `version` column).** Read the row with its version; on update, require `WHERE version = <read value>` and bump the version. If a concurrent write got there first, zero rows update and you retry. Good when conflicts are rare.
3. **Serializable isolation.** Ask Postgres to make concurrent transactions behave as if they ran one at a time; conflicting transactions fail with a serialization error that you catch and retry. Strongest correctness, with a retry cost under contention.

```ts
// Pessimistic lock inside the transaction — the second concurrent transfer waits, then sees the truth.
export async function transferLocked(opts: {
  userId: string; fromId: string; toId: string; amountCents: bigint;
}) {
  return db.$transaction(async (tx) => {
    // FOR UPDATE locks the row until this transaction commits/rolls back.
    const rows = await tx.$queryRaw<{ balanceCents: bigint }[]>`
      SELECT "balanceCents" FROM "Account"
      WHERE id = ${opts.fromId} AND "userId" = ${opts.userId} AND status = 'ACTIVE'
      FOR UPDATE`;
    const from = rows[0];
    if (!from) throw new Error("Source account not found or not yours");
    if (from.balanceCents < opts.amountCents) throw new Error("Insufficient funds");

    await tx.account.update({ where: { id: opts.fromId }, data: { balanceCents: { decrement: opts.amountCents } } });
    await tx.account.update({ where: { id: opts.toId },   data: { balanceCents: { increment: opts.amountCents } } });
  }, { isolationLevel: "Serializable" }); // belt and suspenders for a ledger
}
```

The lesson generalizes beyond money: any "read a value, decide based on it, then write" sequence on shared data needs one of these defenses. Stock/inventory counts, seat reservations, rate-limit counters, and unique-slug generation all share this shape. See [PostgreSQL](POSTGRESQL_GUIDE.md) and [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md) for locking and isolation depth.

### 3.6 The least-privilege database user **[I/A]**

A defense-in-depth measure often skipped: the database user your app connects as should hold the **minimum privileges** it needs — typically `SELECT`/`INSERT`/`UPDATE`/`DELETE` on application tables, and *not* `DROP`, schema-altering rights, or superuser. If an injection or logic flaw ever does reach raw SQL, a least-privilege user limits the blast radius (it can't drop tables or read other schemas). Run migrations as a separate, more-privileged user via `DIRECT_URL`, and keep the runtime user constrained. This mirrors the OS principle of not running services as root (§16.1).

### 3.7 Seeding development data **[B/I]**

A reproducible seed script lets every developer (and CI) start from a known, realistic dataset — and lets you exercise the security paths (multiple users, multiple roles) locally. Keep seeds idempotent so re-running is safe, and never seed production with test credentials.

```ts
// prisma/seed.ts — idempotent dev seed. Wire it via package.json "prisma": { "seed": "tsx prisma/seed.ts" }.
import { PrismaClient } from "@prisma/client";
const db = new PrismaClient();

async function main() {
  // upsert keeps the script idempotent (safe to run repeatedly).
  const alice = await db.user.upsert({
    where: { email: "alice@example.com" },
    update: {},
    create: { email: "alice@example.com", name: "Alice", emailVerified: true, role: "CUSTOMER" },
  });
  const admin = await db.user.upsert({
    where: { email: "admin@example.com" },
    update: {},
    create: { email: "admin@example.com", name: "Admin", emailVerified: true, role: "ADMIN" },
  });
  // Give Alice an account with a starting balance for transfer testing.
  await db.account.upsert({
    where: { id: "seed-alice-acct" },
    update: {},
    create: { id: "seed-alice-acct", userId: alice.id, currency: "USD", balanceCents: 100_00n },
  });
  console.log("Seeded:", { alice: alice.id, admin: admin.id });
}

main().finally(() => db.$disconnect());
```

```bash
npx prisma db seed   # runs the seed script after a fresh migrate/reset
```

Seeding distinct roles (a customer and an admin) is what makes your local environment able to *demonstrate* RBAC — you can log in as each and confirm the role gates behave. This pays off again in E2E tests (§15.3), which need known users to drive the negative authorization cases.

---

## 4. Supabase — Managed Postgres, Storage, Realtime & RLS in Depth

Read [Supabase](SUPABASE_GUIDE.md) for the full product. Here we cover the integration decisions and the **one mistake that compromises most Next.js + Supabase apps**.

### 4.1 When to use Supabase vs self-host **[I]**

Supabase is a managed platform built on Postgres: it gives you a hosted database, an auth service, object storage, realtime subscriptions, edge functions, and — its defining feature — **Row Level Security (RLS)** wired into a public, browser-callable API. The decision:

- **Use managed Supabase** when you want to move fast without operating Postgres, backups, connection pooling, and storage yourself; when you want built-in realtime and storage with signed URLs; or when RLS-enforced, client-direct data access is appealing. Most teams should start here.
- **Self-host Postgres** (or use a plain managed Postgres like RDS) when you need full control over the instance, custom extensions/tuning, strict data-residency, or when you are *not* using Supabase's auth/storage/realtime and only want a database — in which case Supabase adds little over plain Postgres. See [PostgreSQL](POSTGRESQL_GUIDE.md) and [Database Server Admin].

You can also **use Supabase purely as your Postgres host and run Prisma on top** (§4.6) while ignoring its other features — a very common and clean setup.

### 4.2 The two keys — and the #1 security mistake **[I/A]** ⚡ Version note

Supabase issues two API keys, and confusing them is the single most damaging Next.js + Supabase mistake:

- **`anon` key** (`NEXT_PUBLIC_SUPABASE_ANON_KEY`) — safe to ship to the browser. Every request made with it is **subject to RLS policies**. It is "public" precisely because RLS is what protects the data.
- **`service_role` key** (`SUPABASE_SERVICE_ROLE_KEY`) — **bypasses RLS entirely**. It is a god key. It must live *only* on the server, never in `NEXT_PUBLIC_*`, never in a Client Component, never in the bundle.

The catastrophic mistake is using the `service_role` key in code that ends up client-reachable (or, equivalently, *disabling RLS* and relying on the app to filter rows). Either way an attacker reads or writes every row in your database. The rule: **client-side Supabase calls use the `anon` key and lean on RLS; server-side privileged operations use `service_role` only inside server-only modules, and you still verify authorization in your own code (defense in depth).**

### 4.3 `@supabase/ssr` — cookies across the server/client boundary **[I/A]**

Because Next.js renders on both server and client, the Supabase session (a JWT) must be readable in both places, which means it lives in **cookies** that the server can read and refresh. The `@supabase/ssr` package bridges this: it creates a Supabase client wired to Next.js's cookie store so the session survives SSR, Server Actions, and Route Handlers.

```ts
// src/lib/supabase/server.ts — server client (RLS-gated via the user's session)
import "server-only";
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";
import { env } from "@/lib/env";

export async function getSupabaseServer() {
  const cookieStore = await cookies();         // ⚡ async in Next.js 16
  return createServerClient(
    env.NEXT_PUBLIC_SUPABASE_URL,
    env.NEXT_PUBLIC_SUPABASE_ANON_KEY,         // anon key: RLS still applies
    {
      cookies: {
        getAll: () => cookieStore.getAll(),
        setAll: (toSet) => toSet.forEach(({ name, value, options }) =>
          cookieStore.set(name, value, options)),
      },
    },
  );
}
```

```ts
// src/lib/supabase/client.ts — browser client (anon key only, never service_role)
"use client";
import { createBrowserClient } from "@supabase/ssr";
import { env } from "@/lib/env"; // only NEXT_PUBLIC_* values are present client-side

export const supabaseBrowser = createBrowserClient(
  env.NEXT_PUBLIC_SUPABASE_URL,
  env.NEXT_PUBLIC_SUPABASE_ANON_KEY,
);
```

```ts
// Privileged server client — service_role, RLS-bypassing. Use sparingly, server-only.
import "server-only";
import { createClient } from "@supabase/supabase-js";
import { env } from "@/lib/env";

export const supabaseAdmin = createClient(
  env.NEXT_PUBLIC_SUPABASE_URL,
  env.SUPABASE_SERVICE_ROLE_KEY,               // bypasses RLS — handle like a root password
  { auth: { persistSession: false } },
);
```

### 4.4 Row Level Security in depth **[A]**

RLS is Postgres enforcing access control **inside the database**, per row, no matter which query arrives. Once you `ENABLE ROW LEVEL SECURITY` on a table, the default is *deny all*; you then grant access with `POLICY` statements that are SQL boolean expressions evaluated per row. The genius is that the rule travels with the data: even a buggy query or a compromised client cannot read rows the policy forbids, because the *database* refuses.

The policy expressions use `auth.uid()` — the authenticated user's id from the request JWT. The mental model: write policies as if every client is hostile, because with the anon key, they are.

```sql
-- Enable RLS (default-deny) on the table.
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;

-- A user may SELECT only their own accounts.
CREATE POLICY "owner can read own accounts"
  ON accounts FOR SELECT
  USING ( user_id = auth.uid() );

-- A user may INSERT an account only for themselves (prevents forging user_id).
CREATE POLICY "owner can insert own accounts"
  ON accounts FOR INSERT
  WITH CHECK ( user_id = auth.uid() );

-- Updates: must own the row both before (USING) and after (WITH CHECK) the change.
CREATE POLICY "owner can update own accounts"
  ON accounts FOR UPDATE
  USING ( user_id = auth.uid() )
  WITH CHECK ( user_id = auth.uid() );
```

Key distinctions to remember: `USING` filters which existing rows a statement can *see/affect*; `WITH CHECK` validates the *new* row values on INSERT/UPDATE (this is what stops a user from setting `user_id` to someone else — a mass-assignment/IDOR defense at the database layer). RLS does **not** apply to the `service_role` key, so any server code using `service_role` must do its own authorization (§10) — RLS is defense in depth *under* your app logic, not a replacement for it.

> **Banking-grade posture:** turn RLS on for every table holding user data, write default-deny policies, and treat the `service_role` path as a privileged escape hatch used only where genuinely necessary (admin tooling, system jobs) and still authorization-checked in app code.

**Role-aware policies and helper functions.** Real apps need more than "owner can read own". A support agent may read any account; an admin may freeze them. Encode this in policy with a helper that reads the caller's role from a claim or a lookup, keeping the SQL readable:

```sql
-- A SECURITY DEFINER helper that returns the current user's role.
CREATE OR REPLACE FUNCTION current_role_name() RETURNS text
LANGUAGE sql STABLE SECURITY DEFINER AS $$
  SELECT role FROM "User" WHERE id = auth.uid()
$$;

-- Owners read their own; support/admin read any.
CREATE POLICY "read accounts by role"
  ON accounts FOR SELECT
  USING ( user_id = auth.uid() OR current_role_name() IN ('SUPPORT','ADMIN') );

-- Only admins may change account status (freeze/close).
CREATE POLICY "admins manage account status"
  ON accounts FOR UPDATE
  USING ( current_role_name() = 'ADMIN' )
  WITH CHECK ( current_role_name() = 'ADMIN' );
```

**When you use Prisma for writes (not the anon client), RLS does not apply** to that connection — so your app-layer RBAC (§10) is the *primary* control on the Prisma path, and RLS is the backstop for any anon-key, client-direct access. Keep both in sync conceptually: a permission you grant in `rbac.ts` should have a matching RLS policy, so the two layers agree about who can do what. Drift between them is itself a bug (one layer allows what the other forbids), so review them together when permissions change.

### 4.5 Storage & signed URLs **[I/A]**

Supabase Storage holds files in buckets that are **private by default** (and should stay private for anything sensitive — statements, KYC documents). To let a user download a private file, the **server** generates a **time-limited signed URL**: a URL containing a token that grants access to one object for a short window. The browser never holds a key; it holds a URL that expires.

```ts
// Generate a short-lived signed URL on the server, after authorizing the user.
import { getSupabaseServer } from "@/lib/supabase/server";

export async function getStatementUrl(userId: string, path: string) {
  // 1) Authorize: confirm this user owns this object path (your app logic, §10.5).
  assertOwnsPath(userId, path);
  // 2) Sign with a short TTL — minimize the window if the URL leaks.
  const supabase = await getSupabaseServer();
  const { data, error } = await supabase.storage
    .from("statements")
    .createSignedUrl(path, 60); // valid 60 seconds
  if (error) throw error;
  return data.signedUrl;
}
```

The full upload-then-serve flow: the client sends the file to a Server Action; the server validates it by *bytes* (§12.12), uploads it under a generated key to a private bucket, and records the key in the DB. Later, downloads go through a server-generated signed URL.

```ts
// src/features/documents/actions.ts — secure upload to a private Supabase bucket.
"use server";
import { requireFullAuth } from "@/lib/auth";
import { validateUpload } from "@/lib/upload";
import { supabaseAdmin } from "@/lib/supabase/admin"; // service_role, server-only
import { db } from "@/lib/db";

export async function uploadStatement(formData: FormData) {
  const user = await requireFullAuth();
  const file = formData.get("file");
  if (!(file instanceof File)) throw new Error("no file");

  const { mime, storageKey } = await validateUpload(file);      // magic-byte validation
  const path = `${user.id}/${storageKey}`;                       // namespace by user (ownership)
  const { error } = await supabaseAdmin.storage
    .from("statements")                                          // private bucket
    .upload(path, file, { contentType: mime, upsert: false });
  if (error) throw error;

  await db.document.create({ data: { userId: user.id, path, mime } }); // record ownership in DB
  return { ok: true };
}
```

Validate uploads rigorously (size, MIME, magic-byte sniffing, never trust the extension) — see §12.12. Note the path is namespaced by `user.id`, which means the later signed-URL step (§4.5) can verify ownership simply by checking the path prefix against the requesting user.

### 4.6 Prisma + Supabase together **[A]**

A common, clean pattern: **Prisma owns the schema and the server-side writes; Supabase provides the hosted Postgres plus optional client-direct reads under RLS plus Storage/Realtime.** Point Prisma's `DATABASE_URL` at Supabase's **pooled** connection string (transaction pooler, for serverless) and `DIRECT_URL` at the unpooled string (for migrations). Run `prisma migrate` to manage the schema, and write RLS policies as raw SQL migrations alongside. The Prisma client (server, `service_role`-equivalent power via direct DB access) handles trusted writes; the Supabase anon client (client/server) handles RLS-gated reads when you want client-direct data. Keep one source of truth for the schema — Prisma — and never let the two drift.

```env
# .env — Supabase connection strings for the Prisma+Supabase pattern
# Pooled (transaction mode, port 6543) — the app uses this. Note pgbouncer flag.
DATABASE_URL="postgresql://postgres.PROJECT:PASSWORD@aws-region.pooler.supabase.com:6543/postgres?pgbouncer=true&connection_limit=1"
# Direct (port 5432) — migrations use this; pooler can't run DDL/advisory locks well.
DIRECT_URL="postgresql://postgres.PROJECT:PASSWORD@aws-region.pooler.supabase.com:5432/postgres"
```

Decide *per read* whether it goes through Prisma (server-side, trusted, full power) or the Supabase anon client (RLS-gated, can be client-direct). A pragmatic default for banking-grade: **route essentially all reads and writes through Prisma in Server Components/Actions** (so your app-layer RBAC is the consistent control), and use the Supabase client mainly for Storage signed URLs (§4.5) and Realtime (§11) where it shines. Reserve client-direct anon reads for genuinely public or simple owner-scoped data where RLS is a clean fit — and remember that *every* such table needs correct RLS, because the anon key is in the browser.

---

## 5. Data Fetching & Mutations — Server Components, Server Actions, Route Handlers

This is the heart of "Next.js as your backend". See [Next.js 16 §7, §10, §13](NEXTJS_16_GUIDE.md) for depth; here we focus on the secure composition.

### 5.1 Reading data in Server Components **[I]**

A Server Component can `await` a query directly. There is no API endpoint, no client fetch, no loading flag for the initial render — the data is resolved before the HTML is sent. The discipline that makes this banking-grade: **authorize before you query, and project to a minimal DTO before you return JSX that crosses to the client.**

```tsx
// src/app/(app)/accounts/page.tsx  — a Server Component (no "use client")
import { requireUser } from "@/lib/auth";      // throws/redirects if not signed in (§6.5)
import { db } from "@/lib/db";

export default async function AccountsPage() {
  const user = await requireUser();            // 1) authenticate

  // 2) query scoped to this user (ownership baked into WHERE — never "fetch all, filter in JS")
  const accounts = await db.account.findMany({
    where: { userId: user.id },
    select: { id: true, currency: true, balanceCents: true, status: true }, // 3) minimal DTO
  });

  return (
    <ul>
      {accounts.map((a) => (
        <li key={a.id}>
          {a.currency} {(Number(a.balanceCents) / 100).toFixed(2)} — {a.status}
        </li>
      ))}
    </ul>
  );
}
```

Notice three things that distinguish this from a typical SPA component. There is **no `useEffect`, no fetch, no loading spinner, no `useState`** for the data — the component is `async` and the data is simply awaited; the user receives fully-rendered HTML. There is **no API endpoint** in between — the query is a direct, type-checked function call, so a typo in a column name is a compile error, not a runtime 500. And the **authorization and the data fetch are co-located** — you cannot accidentally render the page without the `requireUser()` gate, because they live in the same function the framework calls to produce the page.

### 5.1.1 Streaming and Suspense for slow data **[I/A]**

When part of a page is slow (an external API, an expensive aggregate), you don't have to block the whole page on it. Wrap the slow part in `<Suspense>` and Next.js streams the fast shell immediately, then streams the slow content into its slot when ready. This keeps perceived performance high without sacrificing the server-rendered, secure model. Each suspended subtree is its own async Server Component that does its own authorization — streaming changes *when* HTML arrives, never *whether* the security checks run.

```tsx
import { Suspense } from "react";

export default async function DashboardPage() {
  const user = await requireUser();
  return (
    <>
      <Header user={user} />                         {/* fast: renders immediately */}
      <Suspense fallback={<BalanceSkeleton />}>
        <Balances userId={user.id} />                {/* slow: streams in when ready */}
      </Suspense>
      <Suspense fallback={<ActivitySkeleton />}>
        <RecentActivity userId={user.id} />          {/* independent slow subtree */}
      </Suspense>
    </>
  );
}
// Each child is its own async Server Component and re-derives/re-checks authorization.
```

### 5.2 Mutations with Server Actions — the security model **[I/A]**

A **Server Action** is an `async` function with `"use server"`. The client gets a *handle* to call it (an opaque action id), and Next.js routes the call to the server. This is wonderfully ergonomic — you call a server function straight from a form — but it creates a public, attacker-reachable RPC endpoint. **Treat every Server Action like an unauthenticated public API route, because that is exactly what it is.** Anyone can invoke it with any arguments.

The non-negotiable checklist for every Server Action (in this order):

1. **Authenticate** — get the session; reject if absent.
2. **Authorize** — check the user's role *and* their ownership of the specific object (§10).
3. **Validate** — parse every argument with Zod; reject malformed/extra fields (mass-assignment defense).
4. **Act** — perform the mutation, ideally in a transaction.
5. **Audit** — record security-relevant events (§12.11).
6. **Revalidate** — invalidate affected caches/paths so the UI reflects the change.

```ts
// src/features/transfers/actions.ts
"use server";
import { z } from "zod";
import { requireUser } from "@/lib/auth";
import { transfer } from "@/features/transfers/data";
import { rateLimit } from "@/lib/rate-limit";
import { audit } from "@/lib/audit";
import { revalidatePath } from "next/cache";

// Validate EVERYTHING. Note: amount as integer cents, positive, bounded.
const TransferInput = z.object({
  fromId: z.string().cuid(),
  toId: z.string().cuid(),
  amountCents: z.coerce.bigint().positive().max(100_000_00n), // server-side ceiling
  idempotencyKey: z.string().uuid(),
});

export async function transferAction(raw: unknown) {
  const user = await requireUser();                              // 1) authenticate
  await rateLimit(`transfer:${user.id}`, { max: 5, windowSec: 60 }); // throttle (§12.10)

  const input = TransferInput.parse(raw);                        // 3) validate (throws on bad input)

  // 2) authorize: ownership of the source account is checked atomically inside transfer()
  try {
    const result = await transfer({ userId: user.id, ...input });
    await audit(user.id, "transfer.create", { ...input, amountCents: input.amountCents.toString() }); // 5)
    revalidatePath("/accounts");                                 // 6) refresh the UI
    return { ok: true, result };
  } catch (e) {
    await audit(user.id, "transfer.failed", { reason: String(e) });
    // Never leak internal error detail to the client (§12).
    return { ok: false, error: "Transfer failed" };
  }
}
```

> **Why "never trust the client" is literal here:** the client also sends hidden form fields, and an attacker can replay the action id with a forged `fromId` belonging to *your* account. The only thing that stops them is step 2 — the server re-deriving the user from the session and checking ownership. The client cannot be the source of identity or authorization. Ever.

**Error handling in actions — don't leak internals.** Notice the `catch` returns a generic `"Transfer failed"` rather than the raw error. This is deliberate: a database error string, a stack trace, or "account xyz not found" can hand an attacker information (schema details, whether an id exists). Log the full error server-side (for your own debugging), audit the failure, and return a **generic, user-safe message** to the client. The same applies to Route Handlers — never `return new Response(String(err))`. Map known errors to safe messages and treat everything else as a generic 500. Pair this with a global `error.tsx` boundary (see [Next.js 16 §12](NEXTJS_16_GUIDE.md)) so an unexpected throw renders a friendly page, not a stack trace, to the user.

The return type of an action is also part of the contract: prefer a discriminated result (`{ ok: true, ... } | { ok: false, error: string }`) over throwing for *expected* failures (validation, insufficient funds), and reserve throws for *unexpected* ones. This makes the client code that consumes the action straightforward and keeps you from accidentally surfacing an internal exception message in the UI.

### 5.3 Validation as a shared contract **[I]**

Define Zod schemas in `features/*/schema.ts` and use the *same* schema on the client (for instant UX feedback via React Hook Form, §5.7) and on the server (for security). Client validation is a convenience; **server validation is the security control**. Never skip the server side because "the form already validated it" — the form is in the hostile zone.

### 5.4 Route Handlers — public APIs and webhooks **[I/A]**

Use Route Handlers when you need a *real URL*: third-party webhooks (Stripe, payment processors), OAuth callbacks, mobile/partner APIs, file streaming, health checks. A webhook is a public endpoint receiving money-relevant events, so it must **verify the signature** (proving the sender is who they claim) and be **idempotent** (the provider *will* retry, and you must not double-process).

```ts
// src/app/api/webhooks/payments/route.ts
import { headers } from "next/headers";
import { verifyWebhookSignature } from "@/lib/payments";
import { db } from "@/lib/db";

export async function POST(req: Request) {
  const body = await req.text();                       // raw body needed for signature check
  const sig = (await headers()).get("x-signature") ?? "";

  // 1) Authenticity: reject anything not signed by the provider's secret.
  if (!verifyWebhookSignature(body, sig)) {
    return new Response("invalid signature", { status: 400 });
  }
  const event = JSON.parse(body);

  // 2) Idempotency: provider event id is the dedupe key (DB unique constraint).
  try {
    await db.transaction.create({
      data: { /* ... */ idempotencyKey: event.id },
    });
  } catch {
    return new Response("already processed", { status: 200 }); // unique violation = dup, OK
  }
  return new Response("ok", { status: 200 });
}
```

### 5.5 Client cache with TanStack Query **[I/A]**

Server Components handle the *initial* render. For client-driven interactions — polling balances, infinite transaction lists, optimistic UI, background refetch — use [TanStack Query](TANSTACK_QUERY_GUIDE.md). It caches server state on the client, dedupes requests, and handles stale/refetch. Hydrate it from the server so the first paint has data and the client cache is pre-seeded (no double fetch). The data still comes from your Server Actions / Route Handlers, so all the authorization rules above still apply.

The hydration pattern: a Server Component prefetches into a `QueryClient`, serializes its state with `dehydrate`, and a Client Component rehydrates it. The user sees server-rendered data instantly *and* gets the client cache primed, so the first client interaction doesn't refetch.

```tsx
// Server Component: prefetch on the server, then hand the cache to the client.
import { QueryClient, dehydrate, HydrationBoundary } from "@tanstack/react-query";
import { requireUser } from "@/lib/auth";
import { getRecentTransactions } from "@/features/transactions/data";
import { TransactionList } from "./TransactionList"; // client component using useQuery

export default async function ActivityPage() {
  const user = await requireUser();
  const qc = new QueryClient();
  // Prefetch with the SAME authorization the client path would enforce.
  await qc.prefetchQuery({
    queryKey: ["transactions", user.id],
    queryFn: () => getRecentTransactions(user.id),
  });
  return (
    <HydrationBoundary state={dehydrate(qc)}>
      <TransactionList userId={user.id} /> {/* useQuery finds the data already cached */}
    </HydrationBoundary>
  );
}
```

> **Security reminder:** `dehydrate` serializes whatever is in the cache to the client. Only prefetch queries whose results are safe to send to *this* user, and ensure those query functions returned minimal DTOs. The hydration boundary is the server→client boundary (§1.4) wearing a different hat.

### 5.6 Client UI state with Zustand **[I]**

For purely *client* state — a sidebar toggle, a multi-step form's wizard step, the currently-selected account in the UI — use [Zustand](ZUSTAND_GUIDE.md). The rule of thumb: **TanStack Query for server state, Zustand for UI state.** Never put auth tokens, balances, or any authoritative data in Zustand as if it were a source of truth — it is client memory and therefore untrusted.

### 5.7 Forms with React Hook Form + Zod **[I]**

[React Hook Form](REACT_HOOK_FORM_GUIDE.md) gives performant, accessible forms; pair it with the *same* Zod schema (via `@hookform/resolvers/zod`) for client-side feedback, then submit to a Server Action that re-validates with the identical schema.

The deeper reason to share the schema is **single source of truth**. The shape of valid input is defined exactly once, in `features/*/schema.ts`. The client imports it to give the user instant, friendly feedback (red field, helpful message) *before* a network round-trip — that is the UX layer. The server imports the same object to enforce the rule as a *security control* — that is the trust layer. They can never disagree, because they are literally the same constant. If you defined validation twice (once for the form, once for the action), they would inevitably drift, and the drift is where bugs and vulnerabilities live. Keep the schema pure (no server-only imports) so both sides can import it; put server-only logic in the action, not the schema.

```tsx
"use client";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { TransferInput } from "@/features/transfers/schema"; // shared schema
import { transferAction } from "@/features/transfers/actions";
import { z } from "zod";

type FormValues = z.infer<typeof TransferInput>;

export function TransferForm() {
  const form = useForm<FormValues>({ resolver: zodResolver(TransferInput) });
  return (
    <form onSubmit={form.handleSubmit(async (values) => {
      // Server Action re-validates — client validation is UX only.
      const res = await transferAction({ ...values, idempotencyKey: crypto.randomUUID() });
      if (!res.ok) form.setError("root", { message: res.error });
    })}>
      {/* inputs ... */}
    </form>
  );
}
```

---

## 6. Authentication with Better Auth

Read [Better Auth](BETTERAUTH_GUIDE.md) for the full library and [Go JWT + Argon2 (Banking-Grade Auth)](GO_JWT_ARGON2_GUIDE.md) for the cryptographic reasoning (hashing, token design, threat models) — those concepts apply identically here even though that guide is in Go.

### 6.1 Sessions vs JWTs — choose deliberately **[I/A]**

The first authentication decision is how you remember a logged-in user across requests. Two models:

- **Server sessions (stateful):** on login you create a session record server-side (DB/Redis) and hand the browser an opaque **session id in an HttpOnly cookie**. Each request, the server looks up the session. *Pros:* instant revocation (delete the row → logged out everywhere), no sensitive data in the token, easy to rotate. *Cons:* a lookup per request (cheap with Redis). **This is the right default for a banking app** because *immediate, reliable revocation* is a hard requirement (you must be able to kill a session the moment fraud is detected).
- **JWTs (stateless):** a signed token the server can verify without a lookup. *Pros:* no per-request DB hit; great for distributing identity to *other* services (a separate Go API verifying the same token — see Better Auth §9 / the Go JWT guide). *Cons:* hard to revoke before expiry (you need a denylist, which reintroduces state). Use short-lived access tokens + refresh rotation if you go this way.

Better Auth defaults to **secure server sessions with HttpOnly cookies** and can additionally issue JWTs (via its JWT plugin) when you need to authenticate a *separate* backend. For the standalone web app: sessions. For a web app + separate Go/mobile API: sessions for the web, JWTs for the API.

| Property | Server sessions | JWTs |
|---|---|---|
| Revocation | Instant (delete the row) | Hard (needs a denylist) |
| Per-request cost | A lookup (cheap with Redis) | None (verify signature) |
| Sensitive data exposure | None (opaque id) | Claims are readable (base64, not encrypted) |
| Cross-service auth | Awkward (shared session store) | Natural (verify the signature anywhere) |
| Best for | The web app itself (banking default) | A *separate* API/microservice/mobile client |

The banking-grade reasoning that tips this to sessions for the web tier: **you must be able to log a user out instantly** — the moment fraud is detected, a device is reported stolen, or a password is changed. With server sessions that is a single `DELETE`; with JWTs you're stuck until the token expires unless you maintain a denylist (which is just sessions with extra steps). Use JWTs where their statelessness is the actual benefit: handing identity to a *different* service that shouldn't share your session store.

### 6.2 Configuring `betterAuth()` **[I/A]**

The server `auth` instance is the source of truth. It wires the database adapter (Prisma), password hashing (Argon2id — memory-hard, the current best practice for password storage), cookie security, and plugins (2FA, OTP, social — added in later sections).

```ts
// src/lib/auth.ts
import "server-only";
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { db } from "@/lib/db";
import { env } from "@/lib/env";

export const auth = betterAuth({
  database: prismaAdapter(db, { provider: "postgresql" }), // Better Auth owns its tables in YOUR db
  secret: env.BETTER_AUTH_SECRET,                          // signs cookies/tokens — keep secret
  baseURL: env.BETTER_AUTH_URL,

  emailAndPassword: {
    enabled: true,
    requireEmailVerification: true,                        // no unverified logins (§7)
    // Better Auth uses Argon2id by default — memory-hard, GPU-resistant. Don't downgrade.
    // password: { hash, verify }  // override only if you have a strong reason
  },

  session: {
    expiresIn: 60 * 60 * 24 * 7,        // 7 days
    updateAge: 60 * 60 * 24,            // sliding refresh once per day
    cookieCache: { enabled: true, maxAge: 60 }, // short cache to cut DB hits, still revocable
  },

  advanced: {
    cookiePrefix: "bank",
    useSecureCookies: true,             // Secure flag (HTTPS only) — required in prod
    // HttpOnly + SameSite are applied to session cookies by default (§6.4)
  },

  // plugins: [ twoFactor(), emailOTP(), ... ]  // added in §7–§9
});

export type Session = typeof auth.$Infer.Session; // end-to-end typed session
```

### 6.3 The Prisma adapter & the schema **[I]**

Better Auth needs its core tables (`user`, `session`, `account`, `verification`, plus plugin tables for 2FA, etc.). With the Prisma adapter you generate these into *your* `schema.prisma` (so they live in the same database as your domain tables and can be joined). Better Auth provides a CLI to emit the required models; merge them with your domain models (the `User` in §3.2 is the same `user` table, extended with `role` and relations).

| Table | Holds | Notes |
|---|---|---|
| `user` | Identity, email, verification flag | Extend with `role`, profile fields (§3.2) |
| `session` | Active sessions (id, expiry, user) | Deleting a row = instant revocation (§6.1) |
| `account` | Credentials + linked OAuth identities | One per provider; stores `providerAccountId`, tokens |
| `verification` | Pending verification/OTP tokens | Short-lived, single-use |
| `twoFactor` (plugin) | TOTP secret, status | Encrypt the secret (§8.3) |
| `passkey` (plugin) | WebAuthn credentials | Phishing-resistant factor |

The big advantage of the Prisma adapter living in *your* database: you can write a single query joining `user` to your `account`/`transaction` domain tables (e.g. "all admins who moved money this week"), which is impossible when auth lives in a separate SaaS. Keep Better Auth's tables under its CLI's control — extend `user` with your columns, but let the CLI own the auth-managed shape so upgrades stay smooth.

```bash
# Generate/print the Better Auth schema, then merge into prisma/schema.prisma.
npx @better-auth/cli generate     # ⚡ confirm exact command in the Better Auth guide/docs
npx prisma migrate dev --name better-auth
```

### 6.4 The Next.js App Router handler **[I/A]**

Mount Better Auth's request handler under a catch-all Route Handler. This single endpoint serves every auth route (sign-up, sign-in, callbacks, verification, 2FA).

```ts
// src/app/api/auth/[...all]/route.ts
import { auth } from "@/lib/auth";
import { toNextJsHandler } from "better-auth/next-js";

export const { GET, POST } = toNextJsHandler(auth.handler);
```

```ts
// src/lib/auth-client.ts — the typed client for components
"use client";
import { createAuthClient } from "better-auth/react";
export const authClient = createAuthClient({ baseURL: process.env.NEXT_PUBLIC_APP_URL });
```

**Cookie security (banking-grade defaults):** session cookies must be `HttpOnly` (JavaScript cannot read them → XSS cannot steal the session), `Secure` (HTTPS only → never sent over plaintext), and `SameSite=Lax` or `Strict` (the browser won't attach them to cross-site requests → blunts CSRF). Better Auth applies these; do not weaken them. See §12.3 for CSRF and §12.6 for session rotation.

| Attribute | Value (banking-grade) | What it defends against |
|---|---|---|
| `HttpOnly` | `true` | XSS reading the cookie (token theft) |
| `Secure` | `true` | Transmission over plaintext HTTP |
| `SameSite` | `Lax` (or `Strict`) | CSRF (cross-site cookie attachment) |
| `Path` | `/` | Scope control |
| `Max-Age`/`Expires` | short, sliding | Limits the window of a stolen cookie |
| `__Host-`/`__Secure-` prefix | use where possible | Binds the cookie to the host + Secure |

`SameSite=Strict` is the safest but can log users out when they follow a link from another site (the cookie isn't sent on the cross-site top-level navigation); `Lax` is the usual pragmatic choice and still blocks the dangerous cross-site *POST*. For a banking app, consider `Strict` for the most sensitive cookies and accept the slightly stricter UX. The `__Host-` cookie name prefix is a browser-enforced guarantee that the cookie was set with `Secure`, `Path=/`, and no `Domain` — a cheap extra integrity guard.

### 6.5 Reading the session on the server **[I/A]**

Provide one helper that every server entry point calls. Read the session from `auth.api.getSession` using the request headers — and **always read it on the server**, never trust a client-provided "I am user X" value.

```ts
// src/lib/auth.ts (continued)
import { headers } from "next/headers";
import { redirect } from "next/navigation";

export async function getSession() {
  return auth.api.getSession({ headers: await headers() });
}

// Use in Server Components / Actions that require a logged-in user.
export async function requireUser() {
  const session = await getSession();
  if (!session?.user) redirect("/login");
  return session.user;
}
```

### 6.6 The complete login flow, end to end **[A]**

It is worth tracing the full authentication journey once, because it ties together password auth (§6), OTP verification (§7), and MFA step-up (§8) into a single state machine. Each transition is a server-side decision the client cannot skip:

1. **Sign-up.** User submits email + password. Server validates (Zod), Better Auth hashes the password with Argon2id and creates the user as `emailVerified: false`. Server issues an email OTP (§7) and shows a "check your email" screen. *No session is granted yet* — an unverified account cannot log in.
2. **Email verification.** User submits the code. Server verifies it (hashed, single-use, TTL, attempt-capped), flips `emailVerified: true`, and audits `email.verified`. The account is now usable.
3. **Login — first factor.** User submits email + password. Better Auth verifies the hash in constant-ish time. On failure, increment a rate-limit counter and return a *generic* error ("invalid credentials" — never reveal whether the email exists). On success, check whether the user has verified 2FA.
4. **Login — step-up (if 2FA enrolled).** Instead of a full session, the server marks the session **MFA-pending** (authenticated as to password, *not* yet authorized for sensitive routes). The UI prompts for a TOTP or recovery code. Only a valid second factor promotes the session to fully authenticated — and at that moment the server **rotates the session id** (§12.6) to defeat fixation.
5. **Session established.** A fresh, HttpOnly/Secure/SameSite session cookie is set; `login.success` is audited with IP and user-agent. Subsequent requests are gated by `requireUser()` plus per-route role/ownership checks.
6. **Logout.** Clear the cookie *and* delete the server-side session row, so the session is dead even if the cookie were somehow retained.

The crucial property: at **every** transition the authority lives on the server. A client that POSTs "I verified my email" or "I passed MFA" without the server having recorded it gets nowhere, because each step checks server state, not a client claim. This is the same "never trust the client" principle from §1.4, applied to the most sensitive flow in the app.

---

## 7. OTP & Email/Phone Verification

The concepts here mirror [Go JWT + Argon2 §18](GO_JWT_ARGON2_GUIDE.md) (one-time codes); we apply them in TypeScript. Better Auth provides an **email OTP plugin** that implements most of this securely — prefer it over hand-rolling. The principles below explain *why* it is built the way it is, and how to apply the same rigor to any code you write yourself (e.g. phone/WhatsApp OTP through an SMS provider).

### 7.1 What OTP is for and the threat model **[A]**

A one-time password is a short, randomly-generated code, delivered out-of-band (email, SMS, WhatsApp, an authenticator app), that proves the user controls that channel. It is used to **verify ownership** of an email/phone and as a **second factor** or **step-up** challenge. The threats it must resist: **brute force** (guessing a 6-digit code), **interception/replay** (reusing a captured code), **enumeration** (learning which emails exist), and **flooding** (spamming a victim with codes / running up your SMS bill).

It helps to distinguish OTP from the adjacent mechanisms it's often confused with. A **magic link** is conceptually an OTP embedded in a clickable URL — same single-use, short-TTL, hashed-at-rest rules apply, but the "code" is a long random token in the link rather than 6 digits the user types; magic links trade typing for a click and are convenient on desktop but awkward when the email opens on a different device than the app. **TOTP** (§8) is *not* delivered at all — it's computed locally from a shared secret, so there's nothing to intercept per login, which is why it's a stronger *second factor* than an emailed/SMS'd OTP. The rule of thumb: use **emailed OTP** to *verify ownership* of an email and as a step-up where convenience matters; use **TOTP** as the *strong second factor*; treat **SMS OTP** as a last-resort fallback because of SIM-swap risk.

### 7.2 The five rules of a secure OTP **[A]**

Every property below maps directly to a threat:

1. **Cryptographically random.** Generate codes with a CSPRNG (`crypto.randomInt`), not `Math.random`. A predictable code is no code.
2. **Hashed at rest.** Store the **hash** of the code, never the plaintext — a database leak must not hand attackers live codes. (Codes are short and short-lived, so a fast hash like SHA-256 is acceptable here; passwords still need Argon2.)
3. **Single-use.** Mark consumed on first successful verification; a second attempt with the same code fails. This kills replay.
4. **Short TTL.** Expire in minutes (5–10). A small valid window shrinks brute-force and interception value.
5. **Rate-limited & attempt-capped.** Limit *requests* per user/IP (anti-flood) and *verification attempts* per code (anti-brute-force). After N failed attempts, invalidate the code.

Additionally: **avoid user enumeration** — respond identically whether or not the email exists ("If an account exists, a code has been sent").

```ts
// src/features/auth/otp.ts — illustrative hand-rolled OTP (prefer Better Auth's emailOTP plugin)
import "server-only";
import { createHash, randomInt } from "node:crypto";
import { db } from "@/lib/db";
import { rateLimit } from "@/lib/rate-limit";

const hashCode = (code: string) => createHash("sha256").update(code).digest("hex");

export async function issueOtp(userId: string, purpose: string) {
  await rateLimit(`otp:issue:${userId}`, { max: 3, windowSec: 600 }); // anti-flood
  const code = String(randomInt(0, 1_000_000)).padStart(6, "0");      // CSPRNG, 6 digits
  await db.oneTimeCode.create({
    data: { userId, purpose, codeHash: hashCode(code), expiresAt: new Date(Date.now() + 5 * 60_000) },
  });
  await sendEmail(userId, code); // deliver out-of-band; NEVER return the code to the caller
  // Return nothing identifying — caller shows a generic "code sent" message.
}

export async function verifyOtp(userId: string, purpose: string, code: string) {
  await rateLimit(`otp:verify:${userId}`, { max: 5, windowSec: 600 }); // anti-brute-force
  const rec = await db.oneTimeCode.findFirst({
    where: { userId, purpose, consumedAt: null, expiresAt: { gt: new Date() } },
    orderBy: { createdAt: "desc" },
  });
  if (!rec) return false;
  if (rec.attempts >= 5) return false;                                  // attempt cap
  if (rec.codeHash !== hashCode(code)) {
    await db.oneTimeCode.update({ where: { id: rec.id }, data: { attempts: { increment: 1 } } });
    return false;
  }
  await db.oneTimeCode.update({ where: { id: rec.id }, data: { consumedAt: new Date() } }); // single-use
  return true;
}
```

> **Phone / SMS / WhatsApp:** the logic is identical; only the delivery channel changes (a provider API). Note that SMS is the *weakest* OTP channel (SIM-swap, SS7 interception) — for banking-grade, prefer email + an authenticator-app TOTP (§8) over SMS, and treat SMS OTP as a fallback, not the primary second factor.

### 7.3 Wiring OTP into Server Actions **[A]**

The OTP primitives above become user-facing through two Server Actions: one to request a code, one to verify it. Both must be rate-limited and enumeration-safe. Note how the "request" action returns the *same* response whether or not the email exists — that is what prevents an attacker from using the endpoint to discover which emails are registered.

```ts
// src/features/auth/actions.ts
"use server";
import { z } from "zod";
import { db } from "@/lib/db";
import { issueOtp, verifyOtp } from "@/features/auth/otp";
import { audit } from "@/lib/audit";

const RequestSchema = z.object({ email: z.string().email() });

export async function requestVerificationAction(raw: unknown) {
  const { email } = RequestSchema.parse(raw);
  const user = await db.user.findUnique({ where: { email } });
  // Enumeration-safe: issue only if the user exists, but ALWAYS return the same message.
  if (user) await issueOtp(user.id, "email_verify");
  return { ok: true, message: "If an account exists, a code has been sent." };
}

const VerifySchema = z.object({ email: z.string().email(), code: z.string().length(6) });

export async function verifyEmailAction(raw: unknown) {
  const { email, code } = VerifySchema.parse(raw);
  const user = await db.user.findUnique({ where: { email } });
  if (!user) return { ok: false, error: "Invalid or expired code" }; // generic, no enumeration
  const valid = await verifyOtp(user.id, "email_verify", code);
  if (!valid) {
    await audit(user.id, "email.verify.failed");
    return { ok: false, error: "Invalid or expired code" };
  }
  await db.user.update({ where: { id: user.id }, data: { emailVerified: true } });
  await audit(user.id, "email.verified");
  return { ok: true };
}
```

### 7.4 OTP design choices and their trade-offs **[A]**

A few decisions you will face, and how to reason about them:

- **Code length.** Six digits (1,000,000 possibilities) combined with a 5-attempt cap and rate limiting gives roughly a 5-in-a-million guess probability per code lifetime — adequate. If you need higher assurance, use 8 digits or alphanumeric. Never go below 6.
- **TTL.** Shorter is safer but more annoying. 5–10 minutes balances usability and exposure. Couple a short TTL with resend (itself rate-limited) rather than a long TTL.
- **One active code per purpose.** Issuing a new code should invalidate the previous one, so an attacker can't accumulate a window of several valid codes by repeatedly requesting them.
- **Constant-time comparison.** When comparing the submitted code's hash to the stored hash, the comparison of fixed-length hashes is effectively constant-time; if you ever compare raw codes, use `crypto.timingSafeEqual` to avoid leaking information through timing.
- **Separate purposes.** Tag codes with a `purpose` ("email_verify", "login_otp", "password_reset") and verify the purpose, so a code minted for one flow can't be replayed in another.

---

## 8. MFA / Two-Factor Authentication

MFA requires a second proof of identity beyond the password, so a stolen password alone is not enough. For banking-grade, **TOTP (authenticator app)** and **passkeys/WebAuthn** are the strong options; SMS is weak (see §7). Better Auth's **`twoFactor` plugin** implements TOTP enrollment, verification, and recovery codes; its **`passkey` plugin** implements WebAuthn. Read [Go JWT + Argon2 §19](GO_JWT_ARGON2_GUIDE.md) for the TOTP/recovery-code threat model.

### 8.1 TOTP — how it works and why it's secure **[A]**

TOTP (Time-based One-Time Password, RFC 6238) derives a 6-digit code from a **shared secret** plus the **current 30-second time window**, using HMAC. The server and the authenticator app hold the same secret, so both compute the same code without ever transmitting it — there is nothing to intercept on each login. Enrollment shares the secret once (via a QR code encoding an `otpauth://` URI); thereafter the phone generates codes offline.

The security hinges on protecting that shared secret: if an attacker steals it, they can generate valid codes forever. Therefore the secret must be **encrypted at rest** (§8.3) and only ever decrypted server-side to verify a code.

### 8.2 Enrollment & verification flow **[A]**

1. **Enroll:** user (already authenticated, ideally re-prompted for password) requests 2FA setup. Server generates a TOTP secret, stores it **encrypted, marked unverified**, and returns the QR/otpauth URI.
2. **Confirm:** user scans the QR and submits a code from their app. Server verifies it; only on success mark `verifiedAt`. This proves the secret transferred correctly before you rely on it.
3. **Issue recovery codes:** generate 8–10 single-use recovery codes, show them **once**, store only their hashes (§8.4).
4. **Login (step-up):** after a valid password, if the user has verified 2FA, mark the session **MFA-pending** and require a TOTP (or recovery) code before granting full access.

```ts
// Conceptual flow using Better Auth's twoFactor plugin (client side).
import { authClient } from "@/lib/auth-client";

// Enroll — server returns a QR/otpauth URI; render it for the user to scan.
await authClient.twoFactor.enable({ password });           // re-auth with password
// Confirm with a code from the authenticator app:
await authClient.twoFactor.verifyTotp({ code });           // marks 2FA verified

// At login, after password, if 2FA pending:
await authClient.twoFactor.verifyTotp({ code });           // completes step-up
// Or use a recovery code if the device is lost:
await authClient.twoFactor.verifyBackupCode({ code });
```

If you ever verify TOTP yourself (rather than via Better Auth), the server decrypts the secret, computes the expected code for the current time window (allowing a ±1 window skew for clock drift), and compares in constant time:

```ts
// Illustrative server-side TOTP verify (prefer Better Auth's plugin in practice).
import { decrypt } from "@/lib/crypto";
import { authenticator } from "otplib"; // a vetted TOTP library

export async function verifyTotp(encryptedSecret: string, code: string) {
  const secret = decrypt(encryptedSecret);       // decrypt ONLY server-side, in memory
  authenticator.options = { window: 1 };          // tolerate ±1 step (~30s) of clock drift
  return authenticator.verify({ token: code, secret }); // constant-time compare inside
}
```

Two defenses to layer on top: **rate-limit and attempt-cap** TOTP verification just like OTP (§7.2), so an attacker can't brute-force the 6-digit code; and **mark a code as used** within its window if you want to prevent the (small) replay surface inside a 30-second step. The decrypted secret should never be logged, returned to the client, or held longer than the verification call.

### 8.3 Encrypting the TOTP secret at rest **[A]**

The secret is the crown jewel of 2FA, so store it encrypted with **AES-256-GCM** (authenticated encryption — provides confidentiality *and* tamper detection) using a key from your environment/KMS, **never** the same value as a password and **never** in `NEXT_PUBLIC_*`.

```ts
// src/lib/crypto.ts
import "server-only";
import { createCipheriv, createDecipheriv, randomBytes } from "node:crypto";
import { env } from "@/lib/env";

const KEY = Buffer.from(env.ENCRYPTION_KEY, "hex"); // 32 bytes = AES-256

export function encrypt(plain: string) {
  const iv = randomBytes(12);                          // unique per encryption (GCM nonce)
  const cipher = createCipheriv("aes-256-gcm", KEY, iv);
  const enc = Buffer.concat([cipher.update(plain, "utf8"), cipher.final()]);
  const tag = cipher.getAuthTag();                     // integrity tag
  return [iv.toString("hex"), tag.toString("hex"), enc.toString("hex")].join(":");
}

export function decrypt(payload: string) {
  const [ivHex, tagHex, dataHex] = payload.split(":");
  const decipher = createDecipheriv("aes-256-gcm", KEY, Buffer.from(ivHex, "hex"));
  decipher.setAuthTag(Buffer.from(tagHex, "hex"));     // throws if tampered
  return Buffer.concat([decipher.update(Buffer.from(dataHex, "hex")), decipher.final()]).toString("utf8");
}
```

> If you use Better Auth's `twoFactor` plugin, it manages secret storage — confirm in the Better Auth guide how it encrypts the secret and configure a strong encryption key. The pattern above is what you'd use for any secret you store yourself.

**Why GCM specifically, and a few non-negotiables.** AES-256-GCM is *authenticated* encryption: alongside confidentiality it produces an authentication tag that lets `decrypt` detect any tampering (a flipped bit, a swapped ciphertext) and throw rather than return corrupted plaintext. Three rules you must never violate with GCM: (1) the **IV/nonce must be unique per encryption** under a given key — reusing a nonce with GCM is catastrophic and can leak the key stream, which is why we `randomBytes(12)` every time; (2) never reuse a password or a low-entropy value as the key — use 32 cryptographically-random bytes from a KMS/secret store; (3) store the IV and tag alongside the ciphertext (we join them with `:`), since you need both to decrypt. If you ever need to **rotate the encryption key**, keep the old key available, decrypt-with-old then re-encrypt-with-new in a migration, and prefix stored values with a key id so you know which key produced them — this "envelope" pattern is how production systems rotate without downtime.

### 8.3.1 Enforcing the MFA-pending state **[A]**

The step-up flow only works if a session that has passed the password but *not* the second factor is treated as **not fully authorized**. The cleanest way is to store an `mfaPending` flag (or an `aal` "authenticator assurance level") on the session and have your `requireUser` family check it. A user mid-step-up may reach the MFA-entry page and the verify action, and nothing else.

```ts
// src/lib/auth.ts (continued) — require a fully-authenticated (MFA-complete) session.
export async function requireFullAuth() {
  const session = await getSession();
  if (!session?.user) redirect("/login");
  // If the user enrolled 2FA but hasn't completed it this session, force step-up.
  if (session.mfaPending) redirect("/login/2fa");
  return session.user;
}

// Sensitive routes/actions use requireFullAuth(); the 2FA page itself uses a lighter check.
```

The subtle point: an attacker who steals a *password* still cannot act, because the session they'd create is `mfaPending` and `requireFullAuth()` blocks every sensitive operation until the second factor is presented. MFA only delivers this guarantee if the pending state is enforced *server-side on every protected entry point* — not merely by hiding the UI.

### 8.4 Recovery codes **[A]**

If the user loses their authenticator, recovery codes are the escape hatch — and therefore a high-value target. Treat them like passwords: generate with a CSPRNG, **show once**, store only **hashes**, enforce **single-use** (mark `usedAt`), and let the user **regenerate** (invalidating the old set). Account lockout/recovery is the most-attacked auth surface, so log every recovery-code use to the audit trail (§12.11) and consider alerting the user.

---

## 9. OAuth / Social Login

OAuth lets users log in with an existing identity provider (Google, GitHub, Apple) without you ever handling their password. Better Auth's **social providers** (or Supabase Auth) implement the flow; you configure provider client id/secret and redirect URIs. Read [Better Auth §5](BETTERAUTH_GUIDE.md) and [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md) for the protocol-level reasoning.

### 9.1 The Authorization Code flow with PKCE **[A]**

The modern, secure OAuth flow for web apps is **Authorization Code + PKCE**:

1. Your app redirects the user to the provider with a `state` parameter (random, anti-CSRF) and a PKCE `code_challenge` (the hash of a secret `code_verifier` you keep).
2. The user authenticates *at the provider* and consents.
3. The provider redirects back to your **callback Route Handler** with an authorization `code` and the `state`.
4. Your **server** exchanges the `code` + `code_verifier` for tokens — over a back-channel HTTPS call, so the tokens never touch the browser.
5. You fetch the user profile, then create/link a local account and issue *your own* session.

**Why PKCE and `state` matter:** `state` must be verified on return to stop **CSRF/login-CSRF** (an attacker injecting their own code). PKCE binds the code to the original requester so an intercepted code cannot be redeemed by someone else. The token exchange must happen **server-side** because it uses your OAuth **client secret**, which must never reach the browser. Better Auth handles `state`, PKCE, and the server-side exchange for you; your job is to configure it correctly and never leak the client secret.

Walk the flow once more as a sequence, because each arrow is a security checkpoint:

| Step | Who | What | Security control |
|---|---|---|---|
| 1 | Your app → provider | Redirect with `state` + `code_challenge` | `state` is random, stored server-side |
| 2 | User ↔ provider | Authenticate + consent at the provider | You never see the password |
| 3 | Provider → your callback | Returns `code` + `state` | Verify `state` matches (anti-CSRF) |
| 4 | Your server → provider | Exchange `code` + `code_verifier` for tokens | Uses client secret; server-only; PKCE binds the code |
| 5 | Your server | Fetch profile, check `email_verified`, link/create | Only link on verified email (§9.2) |
| 6 | Your server → browser | Issue *your own* session, redirect to allowlisted path | No open redirect |

The tokens from step 4 never touch the browser, and the session the user ends up with is *yours* (§6), not the provider's — the provider only proved identity once. This is why a compromised provider token doesn't directly grant access to your app: your session is independent and independently revocable.

```ts
// src/lib/auth.ts — add social providers
export const auth = betterAuth({
  // ...prior config...
  socialProviders: {
    github: {
      clientId: env.GITHUB_CLIENT_ID!,
      clientSecret: env.GITHUB_CLIENT_SECRET!,  // SERVER ONLY — never NEXT_PUBLIC_
    },
    // google: { clientId, clientSecret }, apple: { ... }
  },
});
```

```tsx
"use client";
import { authClient } from "@/lib/auth-client";
export function GitHubButton() {
  return <button onClick={() => authClient.signIn.social({ provider: "github" })}>
    Continue with GitHub
  </button>;
}
```

### 9.2 Account linking & security pitfalls **[A]**

When a user logs in socially with an email that already exists locally, you may want to **link** the accounts. The cardinal rule: **only auto-link on a *verified* email.** If you link on an unverified email, an attacker can register a social account with a victim's email and hijack the victim's account. Better Auth exposes account-linking settings; configure it to require verified emails (or require an explicit, authenticated linking step).

Store the provider identity (`provider`, `providerAccountId`, tokens if you need offline access) in the `account` table Better Auth manages. Other pitfalls to defend:

- **Open redirect:** validate any post-login `redirect` parameter against an allowlist of internal paths — never redirect to an arbitrary URL from the query string.
- **Email not verified by provider:** some providers return unverified emails; check the `email_verified` claim before trusting it.
- **Scope creep:** request the minimum scopes; more scopes = more damage if tokens leak.
- **Token storage:** if you store provider access/refresh tokens, encrypt them at rest (§8.3) — they are bearer credentials.

### 9.3 What "linking on verified email" actually defends against **[A]**

It is worth spelling out the attack the verified-email rule prevents, because it is subtle and has bitten major products. Suppose your app auto-links any social login to a local account with the same email address:

1. The attacker signs up locally with the victim's email, `victim@example.com`, but cannot verify it (they don't control the inbox) — so the account sits unverified.
2. Later, the *victim* signs in with "Continue with Google" using their real Google account, whose email is `victim@example.com`.
3. Your app sees a matching email and **links** the Google identity to the pre-existing local account — the one the *attacker* created and may have set a password on.
4. The attacker now logs in with their password and is inside the victim's account.

The fix is to **never link to an unverified local account**, and to only trust a provider's email if the provider asserts it is verified (`email_verified: true`). Better Auth's account-linking configuration lets you require this; for a banking app, prefer requiring an *explicit, authenticated* linking step (the user is already signed in and clicks "link Google") over any automatic linking. The general principle — *an email address is not proof of identity until someone has proven control of it* — recurs throughout auth design.

### 9.4 The callback as a Route Handler **[A]**

Better Auth's catch-all handler (§6.4) already serves the OAuth callback, so in the common case you write no callback code. If you ever implement a provider by hand, the callback is a Route Handler that: verifies `state` against what you stored, exchanges the `code` (with the PKCE `code_verifier`) for tokens server-side using the client secret, fetches and validates the profile (including `email_verified`), creates-or-links the local account, issues your own session, and finally redirects to an **allowlisted** internal path. Every one of those steps is a security checkpoint; skipping any of them is a known CVE class. Lean on Better Auth rather than hand-rolling unless you have a strong reason.

---

## 10. RBAC & Authorization

Authentication answers *who are you*; **authorization** answers *what may you do*. For banking-grade you need both **role-based** checks (coarse: is this an admin?) and **object-level ownership** checks (fine: is this *your* account?). The most common real-world breach is not a broken login — it is **broken object-level authorization (IDOR)**: a logged-in user accessing another user's resource by changing an id.

### 10.1 The model **[I/A]**

Anchor roles in the database. A simple, effective model: a `role` enum on the user for coarse buckets, plus, when you need finer control, a permissions mapping (role → set of permissions) checked in code. For multi-tenant or org scenarios, model memberships (user ↔ org ↔ role). Better Auth's admin/organization plugins provide ready-made role/permission structures — see [Better Auth §10](BETTERAUTH_GUIDE.md).

```ts
// src/lib/rbac.ts
import "server-only";
export type Role = "CUSTOMER" | "SUPPORT" | "ADMIN";

// Map roles to permissions — the single source of truth for "may do X".
const PERMISSIONS: Record<Role, Set<string>> = {
  CUSTOMER: new Set(["account:read:own", "transfer:create:own"]),
  SUPPORT:  new Set(["account:read:any", "ticket:manage"]),
  ADMIN:    new Set(["account:read:any", "account:freeze", "user:manage", "audit:read"]),
};

export function can(role: Role, permission: string): boolean {
  return PERMISSIONS[role]?.has(permission) ?? false;
}

export function assert(role: Role, permission: string) {
  if (!can(role, permission)) throw new Error("Forbidden");
}
```

A permission *matrix* makes the model auditable at a glance — keep one in the codebase (or generated from `PERMISSIONS`) so anyone can see who can do what:

| Permission | CUSTOMER | SUPPORT | ADMIN |
|---|:---:|:---:|:---:|
| `account:read:own` | ✓ | ✓ | ✓ |
| `account:read:any` | | ✓ | ✓ |
| `transfer:create:own` | ✓ | | ✓ |
| `account:freeze` | | | ✓ |
| `user:manage` | | | ✓ |
| `audit:read` | | | ✓ |

The distinction between `:own` and `:any` permissions is the bridge between role checks (§10.2–10.4) and ownership checks (§10.5). A `:own` permission means "may do this *to their own* resources" — the role check passes, and the ownership check in the query enforces the "own" part. An `:any` permission (support/admin) means "may do this to *any* resource" — the role check passes and the ownership filter is widened accordingly. Encoding the scope in the permission name keeps the two layers explicit and prevents the common bug of granting a broad capability where a narrow one was intended.

### 10.2 Enforcing in Server Components **[I/A]**

A Server Component that renders privileged data must check the role *before* querying. Hiding a button in the UI is not authorization — the data fetch is the control point.

```tsx
import { requireUser } from "@/lib/auth";
import { assert } from "@/lib/rbac";

export default async function AdminAuditPage() {
  const user = await requireUser();
  assert(user.role, "audit:read");        // throws → caught by error boundary / 403
  // ... query audit logs ...
}
```

### 10.3 Enforcing in Server Actions **[I/A]**

Repeat the check in *every* mutation. A common, dangerous assumption is that "the page already checked the role, so the action is safe" — false, because the action is independently callable. Authorize in the action itself.

```ts
"use server";
import { requireUser } from "@/lib/auth";
import { assert } from "@/lib/rbac";

export async function freezeAccountAction(accountId: string) {
  const user = await requireUser();
  assert(user.role, "account:freeze");    // role gate
  // ... freeze ...
}
```

### 10.4 Enforcing in Route Handlers **[I]**

Same pattern: read the session from headers, check role/ownership, reject with `403` otherwise. Public API endpoints are the most-probed surface, so make the check the first thing the handler does.

```ts
// src/app/api/admin/accounts/[id]/freeze/route.ts — authorize before anything else.
import { getSession } from "@/lib/auth";
import { can } from "@/lib/rbac";
import { freezeAccount } from "@/features/accounts/data";

export async function POST(_req: Request, { params }: { params: Promise<{ id: string }> }) {
  const session = await getSession();
  if (!session?.user) return new Response("Unauthorized", { status: 401 });
  if (!can(session.user.role, "account:freeze")) return new Response("Forbidden", { status: 403 });

  const { id } = await params;                          // ⚡ async params in Next.js 16
  await freezeAccount(session.user.id, id);             // data layer still checks state/ownership
  return Response.json({ ok: true });
}
```

Note the response codes: **401** means "not authenticated" (you don't know who this is), **403** means "authenticated but not allowed" (you know who they are; they may not do this). Returning the right code matters for clients and for not leaking information — a 404 instead of 403 for "exists but not yours" can be deliberately used to hide existence (§10.5), which is the better choice for object-level checks.

### 10.5 Row-level ownership checks — the IDOR defense **[A]**

Role checks are not enough. A `CUSTOMER` legitimately has `account:read:own` — but *which* accounts are "own"? You must verify the **specific object** belongs to the user, and the safest way is to **bake ownership into the query** rather than fetch-then-compare:

```ts
// WRONG — fetches anyone's account, then checks (and may leak via timing/errors).
const account = await db.account.findUnique({ where: { id } });
if (account.userId !== user.id) throw new Error("Forbidden");

// RIGHT — the WHERE clause makes "not yours" indistinguishable from "doesn't exist".
const account = await db.account.findFirst({ where: { id, userId: user.id } });
if (!account) throw new Error("Not found");   // no information leak about existence
```

This single pattern — *every query scoped by the owner id derived from the server session* — prevents the most common class of real breaches. Combine it with Supabase RLS (§4.4) for defense in depth: even if app code forgets a check, the database refuses the row.

### 10.6 `middleware.ts` — the auth gate and its limits **[I/A]** ⚡ Version note

`middleware.ts` runs before every matched request and is great for **coarse** gating: redirect unauthenticated users away from `(app)` routes, attach security headers, set request ids. But it has real limits, and treating it as your authorization layer is a classic mistake:

- It runs on **every matched request**, so it must be **fast** — do *not* do heavy DB work or full session loads there. A common pattern is an optimistic cookie-presence check in middleware, with the **authoritative** session check done in the Server Component/Action (where you can hit the DB).
- It does **not** protect Server Actions or Route Handlers that aren't matched, and matcher misconfigurations silently leave routes open.
- It cannot see per-object ownership.

**Conclusion: middleware is a first line of defense and a redirect convenience, never the only authorization.** Always re-check in the data layer.

### 10.7 A reusable authorization helper **[A]**

To make the "authenticate → authorize → own" sequence consistent and hard to forget, wrap it in one helper that every action and handler funnels through. Centralizing it means a reviewer can audit *the* authorization logic in one place, and a new action gets it right by construction.

```ts
// src/lib/guard.ts — one funnel for auth + role + ownership.
import "server-only";
import { requireFullAuth } from "@/lib/auth";
import { assert, type Role } from "@/lib/rbac";
import { rateLimit } from "@/lib/rate-limit";
import { audit } from "@/lib/audit";

type GuardOpts<T> = {
  permission: string;                       // required role permission (§10.1)
  rateKey?: (userId: string) => string;     // optional per-action rate limit
  rateLimit?: { max: number; windowSec: number };
  parse: (raw: unknown) => T;               // Zod schema parse (validation)
};

export async function guard<T>(raw: unknown, opts: GuardOpts<T>) {
  const user = await requireFullAuth();                       // 1) authenticate (MFA-complete)
  if (opts.rateKey && opts.rateLimit)                         //    throttle abuse
    await rateLimit(opts.rateKey(user.id), opts.rateLimit);
  assert(user.role as Role, opts.permission);                 // 2) authorize (role)
  const input = opts.parse(raw);                              // 3) validate (Zod allowlist)
  return { user, input };
  // Ownership (4) is still checked in the data layer's WHERE clause (§10.5),
  // because only the query knows which object is being touched.
}
```

```ts
// Using the guard keeps each action terse and uniformly secure.
"use server";
import { guard } from "@/lib/guard";
import { FreezeSchema } from "@/features/accounts/schema";
import { freezeAccount } from "@/features/accounts/data";

export async function freezeAccountAction(raw: unknown) {
  const { user, input } = await guard(raw, {
    permission: "account:freeze",
    rateKey: (id) => `freeze:${id}`,
    rateLimit: { max: 10, windowSec: 60 },
    parse: FreezeSchema.parse,
  });
  return freezeAccount(user.id, input.accountId); // data layer enforces ownership/state
}
```

The pattern scales: as you add actions, the only per-action decisions are *which permission*, *which schema*, and *which rate*. The cross-cutting security plumbing — authentication, MFA-completeness, role check, validation, throttling — is written once and reused. That uniformity is itself a security property: it removes the opportunity to forget a step.

```ts
// src/middleware.ts — coarse gate + security headers (authoritative checks happen deeper)
import { NextResponse, type NextRequest } from "next/server";

export function middleware(req: NextRequest) {
  const res = NextResponse.next();

  // Security headers on every response (§12.9).
  res.headers.set("X-Content-Type-Options", "nosniff");
  res.headers.set("X-Frame-Options", "DENY");
  res.headers.set("Referrer-Policy", "strict-origin-when-cross-origin");

  // Optimistic gate: if no session cookie on an app route, bounce to login.
  const isAppRoute = req.nextUrl.pathname.startsWith("/dashboard")
    || req.nextUrl.pathname.startsWith("/accounts");
  const hasSession = req.cookies.has("bank.session_token"); // presence only — not validation
  if (isAppRoute && !hasSession) {
    return NextResponse.redirect(new URL("/login", req.url));
  }
  return res;
}

export const config = { matcher: ["/dashboard/:path*", "/accounts/:path*"] };
```

---

## 11. Realtime with WebSockets

Banking apps need live updates: balance changes, transaction notifications, fraud alerts, support chat. The architectural reality drives the whole design.

### 11.1 Why Next.js can't host the WebSocket server **[A]**

A WebSocket is a **long-lived, stateful connection**. Serverless and edge deployments (Vercel and most autoscaling platforms) run your code as **short-lived, stateless function invocations** that terminate after each request — there is nowhere for a persistent socket to live. Even on a long-running Node deployment, mixing a WS server into the Next.js request handler is fragile and doesn't scale with the app tier. So for production you choose one of three options:

1. **A separate Node WebSocket service** (the most flexible, banking-grade choice). A small standalone Node process owns the sockets; Next.js (via Server Actions/Route Handlers) publishes events to it, typically through Redis pub/sub. Read [Node.js](NODEJS_GUIDE.md) for the service, and note the [Go Gorilla WebSockets] guide for the same patterns in Go if you prefer a Go socket service. Scale it horizontally with Redis as the backplane.
2. **Supabase Realtime.** If you're on Supabase, you can subscribe to database changes or broadcast on channels from the client — no socket server to operate. Authorization is enforced by **RLS on the underlying tables** (§4.4), so user-scoped events come for free if your policies are correct.
3. **A custom Node server hosting Next.js** (`next start` behind your own `http`/`ws` server). Possible for self-hosted single-node deployments, but you lose serverless scaling and take on the operational burden; only worth it for simple cases.

For banking-grade scale and isolation, **option 1** (separate service + Redis) is the standard; **option 2** is excellent when you're already all-in on Supabase.

| Option | Operates a server? | Auth model | Scaling | Best when |
|---|---|---|---|---|
| Separate Node WS service | Yes (you run it) | Your session/token + per-user channels | Redis pub/sub backplane | Banking-grade; full control |
| Supabase Realtime | No (managed) | RLS on the underlying tables | Managed | Already on Supabase |
| Custom Node server hosting Next.js | Yes (one process) | Your session | Single-node only | Simple self-hosted apps |

The crucial security difference between the options is *where authorization lives*. With the separate service you enforce it explicitly in code (validate the token, derive the user, subscribe only to their channel). With Supabase Realtime you enforce it via **RLS** on the tables being broadcast — which means a missing or wrong policy silently broadcasts data to the wrong users. Whichever you pick, the test is the same: *can client A ever receive an event meant only for client B?* If your design can't make that impossible by construction, fix the design before shipping.

### 11.2 Authenticating the socket **[A]**

A WebSocket connection must be authenticated and **scoped to the user**, or you'll broadcast one customer's balance to another. The connection is just as hostile as an HTTP request. Authenticate it with the **same session** as the web app:

- The browser opens the socket to the WS service; the service reads the **session cookie** (same-site) or a **short-lived signed token** the web app minted for exactly this purpose, validates it, and maps the connection to a user id.
- The service then only ever delivers events on channels the user is authorized for (e.g. `user:<id>`), never letting a client subscribe to arbitrary channels.

```ts
// ws-service/server.ts — standalone Node WebSocket service (runs separately from Next.js)
import { WebSocketServer } from "ws";
import { createClient } from "redis";
import { verifySessionToken } from "./auth"; // validates the token your Next.js app minted

const wss = new WebSocketServer({ port: 8080 });
const sub = createClient({ url: process.env.REDIS_URL }); await sub.connect();

wss.on("connection", async (socket, req) => {
  // 1) Authenticate the socket using a short-lived token from the query/cookie.
  const token = new URL(req.url!, "http://x").searchParams.get("token") ?? "";
  const user = await verifySessionToken(token);     // reject if invalid/expired
  if (!user) return socket.close(1008, "unauthorized");

  // 2) Subscribe ONLY to this user's channel — never trust a client-supplied channel name.
  const channel = `user:${user.id}`;
  await sub.subscribe(channel, (message) => socket.send(message));

  socket.on("close", () => sub.unsubscribe(channel));
});
```

```ts
// In a Next.js Server Action: publish a user-scoped event after a mutation.
import { createClient } from "redis";
const pub = createClient({ url: process.env.REDIS_URL }); await pub.connect();

export async function notifyBalanceChange(userId: string, payload: unknown) {
  // Redis is the backplane: any WS service instance subscribed to this user delivers it.
  await pub.publish(`user:${userId}`, JSON.stringify({ type: "balance", payload }));
}
```

### 11.3 Scaling with Redis & the client hook **[A]**

With multiple WS service instances behind a load balancer, a user's socket might be on instance A while the event is published from instance B. **Redis pub/sub is the backplane**: every instance subscribes to the channels for its connected users, so any instance can publish and the right instance delivers. This is the standard horizontal-scaling pattern for realtime. See [Redis](REDIS_GUIDE.md) for pub/sub details and [Networking](NETWORKING_GUIDE.md) for the WS upgrade handshake (and [Nginx](NGINX_GUIDE.md) for proxying `Upgrade`/`Connection` headers).

```tsx
// src/features/realtime/useUserEvents.ts — client hook with reconnect + backoff
"use client";
import { useEffect, useRef } from "react";

export function useUserEvents(getToken: () => Promise<string>, onEvent: (e: unknown) => void) {
  const closedByUs = useRef(false);
  useEffect(() => {
    closedByUs.current = false;
    let ws: WebSocket;
    let attempt = 0;

    const connect = async () => {
      const token = await getToken();                       // mint a fresh short-lived token
      ws = new WebSocket(`wss://realtime.example.com?token=${token}`);
      ws.onopen = () => { attempt = 0; };                   // reset backoff on success
      ws.onmessage = (m) => onEvent(JSON.parse(m.data));
      ws.onclose = () => {
        if (closedByUs.current) return;                     // don't reconnect on unmount
        // Exponential backoff with a cap and jitter — avoids thundering-herd reconnects.
        const delay = Math.min(30_000, 2 ** attempt++ * 1000) + Math.random() * 1000;
        setTimeout(connect, delay);
      };
    };
    connect();
    return () => { closedByUs.current = true; ws?.close(); };
  }, [getToken, onEvent]);
}
```

A few realtime design notes worth internalizing. **Mint a fresh, short-lived token per connection** (not a long-lived one), so a leaked token expires quickly; the hook refetches it on every reconnect. **Use exponential backoff with jitter** so that when the WS service restarts, thousands of clients don't reconnect in the same millisecond and knock it over again (the "thundering herd"). **Never trust realtime payloads as authoritative** — a pushed balance is a *hint to refetch* or an optimistic display; the source of truth is still the authorized server query. And **scope strictly server-side**: the client asks for "my events" implicitly (the service derives the channel from the authenticated identity), never by sending a channel name the service trusts. See [Redis](REDIS_GUIDE.md) for the pub/sub backplane and [Networking](NETWORKING_GUIDE.md)/[Nginx](NGINX_GUIDE.md) for proxying the `Upgrade` handshake.

---

## 12. Banking-Grade Security Hardening

Security is woven through every prior section; this section consolidates the **threat model and the defense for each attack class**, in the spirit of OWASP. Read [Go JWT + Argon2 (Banking-Grade Auth)](GO_JWT_ARGON2_GUIDE.md) for the cryptographic depth — the threat models transfer directly. The governing principle: **defense in depth** (multiple independent layers, so one mistake isn't fatal) and **never trust the client** (the network is the trust boundary).

### 12.1 The threat model **[A]**

Before defenses, name what you're defending against: an attacker who can send arbitrary HTTP requests and call any Server Action with any payload; a malicious or compromised logged-in user; a leaked dependency or secret; a network eavesdropper; and an insider. For each, you want layered controls so that a single failure is contained. The table at the end of this section maps attacks to defenses; the subsections explain each.

Two principles govern everything in this section, and if you remember nothing else, remember these:

1. **Never trust the client.** Every byte that arrives from a browser — arguments, headers, cookies, file contents, hidden fields — is attacker-controlled until your server has validated it. Identity comes from the server-side session, never from a client claim. Authorization is re-checked on the server at every entry point. Validation happens on the server with Zod, every time. The client is a convenience and a UX layer; it is never a security control.
2. **Defense in depth.** Assume each individual control will sometimes fail — a check forgotten, a config slipped, a library bug — and arrange independent layers so a single failure isn't a breach. RLS sits under app-layer RBAC; `SameSite` cookies sit under explicit origin checks; HttpOnly cookies sit under CSP; the least-privilege DB user sits under parameterized queries. The point is not any one wall but that there are several, and an attacker must defeat all of them.

Everything below is an application of these two ideas to a specific attack class.

### 12.2 XSS (Cross-Site Scripting) **[A]**

XSS is injecting attacker JavaScript that runs in your users' browsers — letting it read the DOM, exfiltrate data, and act as the user. React's defense is **automatic escaping**: `{userInput}` in JSX is rendered as text, not HTML, so markup is neutralized. You break this protection only with **`dangerouslySetInnerHTML`** — avoid it; if you must render user HTML, **sanitize** it server-side with a library like DOMPurify and an allowlist. Layer a **Content Security Policy** (CSP) on top: a CSP restricts which scripts may run, so even injected script is refused. Use **nonces** for inline scripts (Next.js can generate a per-request nonce) and avoid `unsafe-inline`.

Because session cookies are **HttpOnly** (§6.4), even successful XSS cannot read the session token directly — that's defense in depth working.

A strong CSP for a banking app uses a **per-request nonce**: middleware generates a random nonce, sets it in the CSP header, and you attach it to any inline script you genuinely need. The browser then refuses *any* inline script lacking the matching nonce — so even if an attacker injects a `<script>`, it won't run. This is far stronger than `unsafe-inline`, which permits all inline scripts and effectively disables CSP's XSS protection.

```ts
// src/middleware.ts (excerpt) — per-request CSP nonce
import { NextResponse, type NextRequest } from "next/server";

export function middleware(req: NextRequest) {
  const nonce = Buffer.from(crypto.randomUUID()).toString("base64");
  const csp = [
    `default-src 'self'`,
    `script-src 'self' 'nonce-${nonce}' 'strict-dynamic'`, // only nonced scripts run
    `style-src 'self' 'nonce-${nonce}'`,
    `img-src 'self' data:`,
    `connect-src 'self' wss://realtime.example.com`,
    `frame-ancestors 'none'`,
    `base-uri 'self'`,
    `form-action 'self'`,
  ].join("; ");

  // Pass the nonce downstream so Server Components can read it via headers().
  const requestHeaders = new Headers(req.headers);
  requestHeaders.set("x-nonce", nonce);
  const res = NextResponse.next({ request: { headers: requestHeaders } });
  res.headers.set("Content-Security-Policy", csp);
  return res;
}
```

**Stored vs reflected vs DOM-based XSS** — know the three forms: *reflected* (payload in the request is echoed into the response — validate/escape output), *stored* (payload saved to the DB and rendered later to other users — the most dangerous; sanitize on render, never trust stored data), and *DOM-based* (client JS writes attacker input into the DOM via `innerHTML`/`document.write` — avoid those sinks entirely). React's escaping handles the common cases for all three *as long as you don't reach for `dangerouslySetInnerHTML`*; the moment you do, you own the sanitization.

### 12.3 CSRF (Cross-Site Request Forgery) **[A]**

CSRF tricks a logged-in user's browser into making a state-changing request to your app from a malicious site, riding their cookies. Defenses, layered: **`SameSite=Lax/Strict` cookies** (the browser won't attach your session cookie to cross-site requests — the primary modern defense, applied by Better Auth), plus **origin/referer checks** on mutations, plus **CSRF tokens** for sensitive actions. Next.js Server Actions include built-in CSRF protections, but for a banking app verify the `Origin` header on money-moving operations as an explicit extra check. See §6.4 and [Better Auth §11](BETTERAUTH_GUIDE.md).

```ts
// src/lib/origin.ts — explicit origin check for sensitive mutations (defense in depth).
import "server-only";
import { headers } from "next/headers";
import { env } from "@/lib/env";

export async function assertSameOrigin() {
  const h = await headers();
  const origin = h.get("origin");
  // A cross-site forged request carries a foreign Origin (or none for some attacks).
  if (origin && origin !== env.NEXT_PUBLIC_APP_URL) {
    throw new Error("cross-origin request rejected");
  }
}
// Call assertSameOrigin() at the top of transfer/withdraw/role-change actions.
```

The layering is deliberate: `SameSite` cookies are the browser-enforced primary defense, the explicit `Origin` check is a server-enforced second line (in case of a `SameSite` gap or a misconfigured proxy), and a per-form CSRF token is a third for the most sensitive operations. Defense in depth means no single failure — a browser quirk, a config slip — opens the door.

### 12.4 SQL Injection **[A]**

SQL injection is smuggling SQL through user input. **Prisma parameterizes all ordinary queries** — your inputs are bound parameters, never concatenated into the SQL string — so the standard query API is safe by construction. The risk reappears only when you use **raw SQL**: use Prisma's **tagged-template** `db.$queryRaw` (which parameterizes) and **never** `db.$queryRawUnsafe` with interpolated user input. See [Prisma ORM](PRISMA_ORM_GUIDE.md) and [PostgreSQL](POSTGRESQL_GUIDE.md).

```ts
// SAFE — tagged template binds `email` as a parameter.
await db.$queryRaw`SELECT id FROM "User" WHERE email = ${email}`;
// DANGEROUS — string built from user input. Never do this.
// await db.$queryRawUnsafe(`SELECT id FROM "User" WHERE email = '${email}'`);
```

### 12.5 Authentication & session security **[A]**

Covered in §6–§9; the banking-grade checklist: Argon2id password hashing; HttpOnly + Secure + SameSite cookies; **session rotation on privilege change** (regenerate the session id on login and on step-up to defeat **session fixation**); short session lifetimes with sliding refresh; **immediate revocation** capability (server sessions); MFA for sensitive accounts; lockout/backoff on repeated failures; and audit logging of every auth event.

**Why Argon2id specifically.** Password hashing must be *deliberately slow and memory-hard* so that an attacker who steals the hash database cannot feasibly brute-force the originals. Argon2id (the winner of the Password Hashing Competition) is memory-hard, which neutralizes the GPU/ASIC parallelism that makes older hashes (MD5, SHA-1, even fast SHA-256) catastrophic for passwords. Better Auth uses it by default; do not downgrade to a fast hash, and never invent your own. The general rule: *passwords get a slow, memory-hard hash (Argon2id); short-lived tokens and OTP codes can use a fast hash (SHA-256) because their value expires in minutes*. Matching the hash to the secret's lifetime and value is the skill — see [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md) for the cryptographic depth.

### 12.6 Session rotation & fixation **[A]**

Session fixation is an attacker fixing a known session id on a victim, then using it after the victim logs in. The defense is to **issue a fresh session id at every authentication boundary** (login, step-up MFA, password change) and invalidate the old one. Better Auth handles session creation; ensure you call sign-out/invalidate on password change and propagate revocation everywhere (delete server-side session rows). On logout, clear the cookie *and* delete the server record — a cookie-only logout leaves the session valid server-side.

```ts
// On a password change, revoke ALL other sessions for the user — a stolen session dies immediately.
"use server";
import { requireFullAuth, auth } from "@/lib/auth";
import { audit } from "@/lib/audit";
import { headers } from "next/headers";

export async function changePasswordAction(raw: unknown) {
  const user = await requireFullAuth();
  // ...validate + verify current password + set new (Argon2id) via Better Auth...

  // Revoke every other session so any attacker-held session is invalidated now.
  await auth.api.revokeOtherSessions({ headers: await headers() }); // ⚡ confirm method in Better Auth docs
  await audit(user.id, "password.changed");
  return { ok: true };
}
```

The principle is **"a change in the security state invalidates the old security context."** Logging in, completing MFA, or changing a password should all produce a *new* session and retire the old one, so neither a fixed session id nor a previously-stolen session survives the transition. This is also why "log out everywhere" is a feature users expect from a bank — it's `revokeOtherSessions` exposed in the UI.

### 12.7 Secrets management **[A]**

Secrets (DB URLs, `service_role`, signing/encryption keys, OAuth secrets, webhook secrets) live **only on the server**, **never** with `NEXT_PUBLIC_`, **never** in the repo. In production, inject them via the platform's secret store (Vercel encrypted env vars, Docker/K8s secrets, a vault). Rotate them on a schedule and on any suspected leak. Add a CI gate that greps the built bundle for known secret patterns and a secret-scanner (e.g. gitleaks) on the repo. See §2.3–§2.4.

### 12.8 SSRF, IDOR & mass assignment **[A]**

These three are among the most common real-world authorization/injection failures, so treat each as first-class.

**SSRF (Server-Side Request Forgery):** if your server fetches a user-supplied URL (webhooks, image proxies, link previews), an attacker can point it at internal services or cloud metadata endpoints. Defend with an **allowlist** of hosts/schemes, block private/link-local IP ranges, disable redirects to internal addresses, and never reflect internal responses verbatim.

```ts
// A guarded fetch for user-supplied URLs — the core of SSRF defense.
import { lookup } from "node:dns/promises";
import net from "node:net";

const ALLOWED_HOSTS = new Set(["api.partner.com", "hooks.provider.com"]);

export async function safeFetch(rawUrl: string) {
  const url = new URL(rawUrl);
  if (url.protocol !== "https:") throw new Error("only https allowed"); // no file://, http://, gopher://
  if (!ALLOWED_HOSTS.has(url.hostname)) throw new Error("host not allowlisted");

  // Resolve and reject private/link-local/loopback targets (defeats DNS rebinding to internal IPs).
  const { address } = await lookup(url.hostname);
  if (isPrivateIp(address)) throw new Error("resolved to a private address");

  // Disable redirects so a 302 can't bounce us to an internal address after the checks.
  return fetch(url, { redirect: "error" });
}

function isPrivateIp(ip: string) {
  if (net.isIPv4(ip)) {
    const [a, b] = ip.split(".").map(Number);
    return a === 10 || a === 127 || (a === 192 && b === 168) ||
           (a === 172 && b >= 16 && b <= 31) || (a === 169 && b === 254); // incl. link-local metadata
  }
  return ip === "::1" || ip.startsWith("fc") || ip.startsWith("fd"); // IPv6 loopback/ULA
}
```

The cloud-metadata endpoint (`169.254.169.254`) is the classic SSRF target — it can hand an attacker your instance's IAM credentials — which is exactly why the link-local `169.254.x.x` range is blocked above.

**IDOR / broken object-level authz:** the §10.5 pattern — scope every query by the session-derived owner id; never trust an id from the client to imply ownership. This is the single most common serious vulnerability in real apps, precisely because the code "works" in testing (you test as the owner) and only fails when an attacker swaps in someone else's id.

**Mass assignment:** never spread client input straight into a DB write (`data: { ...input }`). Parse with a Zod schema that lists **only** the allowed fields, so the client can't set `role`, `userId`, `balanceCents`, or `isVerified`. RLS `WITH CHECK` (§4.4) is the database-layer backstop.

```ts
// Mass-assignment defense: explicit allowlist via Zod; ignore everything else.
const UpdateProfile = z.object({ name: z.string().min(1).max(100) }); // ONLY name
const data = UpdateProfile.parse(raw);   // role/userId/etc. are dropped, not written
await db.user.update({ where: { id: user.id }, data });
```

### 12.9 Security headers (CSP, HSTS, etc.) **[A]**

Set a strong header baseline on every response, via `middleware.ts` (§10.6) and/or `next.config` headers. The key headers and why:

- **`Content-Security-Policy`** — restricts script/style/connect sources; the strongest XSS mitigation.
- **`Strict-Transport-Security` (HSTS)** — forces HTTPS, defeating downgrade/stripping attacks.
- **`X-Content-Type-Options: nosniff`** — stops MIME sniffing (drive-by execution of uploads).
- **`X-Frame-Options: DENY` / `frame-ancestors`** — blocks clickjacking.
- **`Referrer-Policy: strict-origin-when-cross-origin`** — limits referrer leakage.
- **`Permissions-Policy`** — disables unneeded browser features (camera, geolocation).

```ts
// next.config.ts — global security headers (combine with middleware for per-request nonces)
import type { NextConfig } from "next";
const config: NextConfig = {
  async headers() {
    return [{
      source: "/:path*",
      headers: [
        { key: "Strict-Transport-Security", value: "max-age=63072000; includeSubDomains; preload" },
        { key: "X-Content-Type-Options", value: "nosniff" },
        { key: "X-Frame-Options", value: "DENY" },
        { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
        { key: "Permissions-Policy", value: "camera=(), microphone=(), geolocation=()" },
        // CSP: tighten to your real sources; prefer nonces over 'unsafe-inline'.
        { key: "Content-Security-Policy",
          value: "default-src 'self'; img-src 'self' data:; connect-src 'self' wss://realtime.example.com; frame-ancestors 'none';" },
      ],
    }];
  },
};
export default config;
```

### 12.10 Rate limiting with Redis **[A]**

Rate limiting throttles abuse: credential stuffing, OTP brute force, scraping, money-moving spam. Implement it in **Redis** (atomic counters with TTL, shared across all app instances — an in-memory limiter fails the moment you scale past one instance). Limit by user id *and* by IP, with stricter limits on sensitive endpoints (login, OTP, transfers). See [Redis](REDIS_GUIDE.md) and the rate-limit reasoning in [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md).

```ts
// src/lib/rate-limit.ts — fixed-window limiter (simple; has a burst edge at window boundaries)
import "server-only";
import { redis } from "@/lib/redis";

export async function rateLimit(key: string, opts: { max: number; windowSec: number }) {
  const k = `rl:${key}`;
  const count = await redis.incr(k);                  // atomic increment
  if (count === 1) await redis.expire(k, opts.windowSec); // set TTL on first hit
  if (count > opts.max) throw new Error("Too many requests"); // 429 at the boundary
}
```

The fixed-window limiter above is simple but has a known weakness: a caller can send `max` requests at the very end of one window and `max` more at the start of the next, briefly allowing `2×max` in a short span. For sensitive endpoints (login, OTP, transfers) prefer a **sliding-window** (or token-bucket) limiter, which smooths that boundary by counting requests over a rolling window using a sorted set:

```ts
// Sliding-window limiter — counts requests in the trailing `windowSec` using a Redis sorted set.
export async function slidingRateLimit(key: string, opts: { max: number; windowSec: number }) {
  const k = `rl:sw:${key}`;
  const now = Date.now();
  const windowStart = now - opts.windowSec * 1000;
  const member = `${now}-${Math.random()}`; // unique per request

  const pipe = redis.multi();
  pipe.zremrangebyscore(k, 0, windowStart);  // drop entries older than the window
  pipe.zadd(k, now, member);                 // record this request
  pipe.zcard(k);                             // how many in the window now?
  pipe.expire(k, opts.windowSec);            // self-cleaning key
  const res = await pipe.exec();
  const count = res?.[2]?.[1] as number;
  if (count > opts.max) throw new Error("Too many requests");
}
```

Either way, the limiter must live in **Redis (shared state)**, not process memory — an in-memory limiter resets per instance and is trivially defeated by load-balancing across instances. Apply *different* limits per endpoint sensitivity: generous on read pages, strict on login/OTP (e.g. 5/min), strictest on money movement, and always combine a **per-user** key with a **per-IP** key so neither a single account nor a single source can be abused. See [Redis](REDIS_GUIDE.md).

### 12.11 Audit logging **[A]**

A banking-grade app keeps an **append-only audit trail** of security-relevant events: logins (success/failure), MFA enroll/use, recovery-code use, role changes, transfers, admin actions, data exports. Each entry records who, what, when, from where (IP/user-agent). Audit logs are for forensics and compliance, so make them **append-only** (no updates/deletes from app code), store them outside the primary mutable tables if possible, and never log secrets/PII you don't need. The `AuditLog` model (§3.2) and an `audit()` helper called from every Server Action (§5.2) implement this.

```ts
// src/lib/audit.ts — capture who/what/when/where for security-relevant events.
import "server-only";
import { db } from "@/lib/db";
import { headers } from "next/headers";

export async function audit(userId: string | null, action: string, metadata?: Record<string, unknown>) {
  const h = await headers();
  await db.auditLog.create({
    data: {
      userId,
      action,                                          // "login.success", "transfer.create", ...
      ip: h.get("x-forwarded-for")?.split(",")[0] ?? h.get("x-real-ip"),
      userAgent: h.get("user-agent"),
      metadata: metadata as any,                       // NEVER put secrets/full PII here
    },
  });
}
```

Enforce append-only at the **database** level (not just in app code) by granting the app's DB user `INSERT` and `SELECT` on `AuditLog` but *not* `UPDATE`/`DELETE` (§3.6). Then even a compromised app cannot tamper with the trail — which is exactly the property auditors and incident responders need. For the highest assurance, ship audit events to a separate, write-once store (a logging pipeline or a dedicated append-only table in another schema).

### 12.12 Input validation & file-upload validation **[A]**

**Validate everything, everywhere, with Zod** — at every server entry point, treating client validation as UX only. For **file uploads**, the filename and the client-supplied MIME type are attacker-controlled, so: enforce a **size limit**; check the **magic bytes** (real file signature), not the extension; restrict to an allowlist of types; store with a generated name (never the user's filename → path-traversal defense); serve from a separate origin/bucket with `Content-Disposition: attachment` and `nosniff`; and scan for malware where feasible. A "PNG" that is actually an HTML/JS payload is a classic stored-XSS vector — magic-byte sniffing plus serving from a non-app origin neutralizes it.

```ts
// src/lib/upload.ts — validate an upload by its real bytes, not its claimed type/name.
import "server-only";
import { randomUUID } from "node:crypto";

const MAX_BYTES = 5 * 1024 * 1024; // 5 MB
// First bytes ("magic numbers") of allowed types.
const SIGNATURES: Record<string, number[]> = {
  "image/png":  [0x89, 0x50, 0x4e, 0x47],
  "image/jpeg": [0xff, 0xd8, 0xff],
  "application/pdf": [0x25, 0x50, 0x44, 0x46], // "%PDF"
};

export async function validateUpload(file: File) {
  if (file.size === 0 || file.size > MAX_BYTES) throw new Error("invalid file size");
  const head = new Uint8Array(await file.slice(0, 8).arrayBuffer());

  // Find a signature that matches the actual bytes — ignore file.type and file.name entirely.
  const matched = Object.entries(SIGNATURES).find(([, sig]) =>
    sig.every((byte, i) => head[i] === byte));
  if (!matched) throw new Error("file content does not match an allowed type");

  const [mime] = matched;
  const ext = mime === "image/png" ? "png" : mime === "image/jpeg" ? "jpg" : "pdf";
  // Generated name: never the user's filename (path-traversal / overwrite defense).
  return { mime, storageKey: `${randomUUID()}.${ext}` };
}
```

The reasoning behind each control: the **size cap** stops a memory/disk DoS; the **magic-byte check** stops a polyglot file (e.g. a file that is valid HTML *and* a valid image) from being served and executed; the **generated name** stops `../../etc/passwd`-style path traversal and stops one user overwriting another's file by guessing names; serving from a **separate origin** with `attachment`+`nosniff` means even if a malicious file slips through, the browser downloads rather than renders it, and any script in it runs on a throwaway origin, not your app's. Layer them — no single check is sufficient.

### 12.13 Encryption in transit & at rest **[A]**

- **In transit:** TLS everywhere — HTTPS for the app (HSTS, §12.9), `wss://` for sockets, TLS to the database and Redis. Terminate TLS at [Nginx](NGINX_GUIDE.md) or the platform edge; never run plaintext between tiers in production.
- **At rest:** enable database encryption (Supabase/managed Postgres provide it); **application-layer encrypt** the most sensitive fields (TOTP secrets, stored provider tokens) with AES-256-GCM (§8.3) so even a DB dump doesn't expose them; encrypt backups; manage keys in a KMS/secret store and rotate them.

### 12.14 The server/client trust boundary (recap) **[A]**

The thread running through this entire guide: **never leak secrets or PII into the client bundle.** Concretely: keep secrets out of `NEXT_PUBLIC_*`; mark server-only modules with `import "server-only"`; project DB rows to minimal DTOs before passing them to Client Components or returning from Server Actions; never return a full `user`/`account` row to the client; and remember that *anything* a Client Component receives is public. When in doubt, assume the client is the attacker — because sometimes it is.

### 12.15 Dependency security **[A]**

Your dependencies are part of your attack surface. Run `npm audit` and a Software Composition Analysis tool in CI, pin versions with a lockfile, enable automated dependency-update PRs, and review new dependencies for maintenance health before adding them. A compromised transitive package can read your secrets at runtime, so minimize the dependency count on security-critical paths.

> **Reference — attack → defense**

| Attack | Primary defense | Backstop / defense-in-depth |
|---|---|---|
| XSS | React escaping; sanitize HTML | CSP + nonces; HttpOnly cookies |
| CSRF | SameSite cookies | Origin check + CSRF tokens on mutations |
| SQL injection | Prisma parameterization | Avoid `$queryRawUnsafe`; least-priv DB user |
| Broken auth/session | Argon2id; Secure/HttpOnly cookies | Rotation; revocation; MFA; lockout |
| Session fixation | New session id at each auth boundary | Server-side session deletion on logout |
| IDOR / object authz | Scope queries by session owner id | Supabase RLS `USING` |
| Mass assignment | Zod allowlist (explicit fields) | RLS `WITH CHECK`; never spread input |
| SSRF | Host allowlist; block private IPs | No redirects to internal; no reflected responses |
| Rate abuse / brute force | Redis rate limiting (user + IP) | Attempt caps; exponential backoff |
| Secret leakage | Server-only; no `NEXT_PUBLIC_` | Bundle/secret scanners in CI; rotation |
| Malicious upload | Magic-byte + size + type checks | Separate origin; `nosniff`; generated names |
| Eavesdropping | TLS in transit (HSTS, wss, DB TLS) | AES-256-GCM at rest for sensitive fields |
| Missing forensics | Append-only audit log | Alerting on high-risk events |
| Vulnerable deps | `npm audit` + SCA in CI | Lockfile pinning; minimal deps |

---

## 13. Performance & Scaling

A secure app must also stay fast under load. See [Next.js 16 §8, §9, §20](NEXTJS_16_GUIDE.md) for the rendering/caching depth.

### 13.1 The Next.js caches **[A]** ⚡ Version note

Next.js 16 has several caching layers and the defaults changed from the 14 era — internalize them or you'll either serve stale data or miss easy wins:

- **`fetch()` is not cached by default** (since 15); opt in explicitly per call (`{ cache: "force-cache" }`, `next: { revalidate } }`).
- **`"use cache"`** (Cache Components / the productionized PPR) lets you cache a component or function's output with explicit lifetimes and tags.
- **Full Route Cache / Router Cache** govern static route output and client-side navigation.
- **`revalidatePath` / `revalidateTag`** invalidate caches after a mutation — call them in Server Actions so the UI reflects writes.

**Security caution:** never cache **user-specific** or **authorized** responses at a shared layer. A misconfigured cache that serves one user's dashboard to another is a data breach. Keep per-user data dynamic (or cache per-user with a key that includes the user id).

```ts
// Cache a SHARED, non-user-specific computation with a tag for targeted invalidation.
import { unstable_cache } from "next/cache"; // ⚡ API name/shape evolves — confirm in the Next.js guide

export const getExchangeRates = unstable_cache(
  async () => fetchRatesFromProvider(),  // shared across all users — safe to cache
  ["exchange-rates"],                     // cache key parts
  { revalidate: 300, tags: ["rates"] },   // 5 min TTL + a tag you can bust on demand
);

// After an admin updates rates, bust just that tag — no full purge.
import { revalidateTag } from "next/cache";
export async function updateRatesAction(/* ... */) {
  // ...write...
  revalidateTag("rates");
}
```

```ts
// NEVER do this — caching per-user data at a shared key leaks one user's data to another.
// const balance = unstable_cache(() => getBalance(userId), ["balance"]); // BUG: key ignores userId
// If you must cache per-user, include the user id IN the key and keep TTLs short:
const getUserBalance = (userId: string) =>
  unstable_cache(() => getBalance(userId), ["balance", userId], { revalidate: 10 })();
```

The decision rule is simple and worth memorizing: **cache by *who can see it*.** Public, identical-for-everyone data (marketing copy, exchange rates, a public product list) caches at a shared layer. Anything that varies by user, role, or tenant either stays dynamic or is cached under a key that includes the identity — and even then with a short TTL, because a stale balance is a support ticket and a leaked balance is an incident.

### 13.2 RSC payload size & streaming **[A]**

Server Components serialize their result to the client; large payloads slow hydration. Keep the boundary lean (minimal DTOs — also a security win), push interactivity to small Client Components (lower the "use client" leaves), and use **Suspense + streaming** so the static shell flushes immediately while slow data streams in. PPR/Cache Components give you a static shell with dynamic holes in one response.

The "push `use client` to the leaves" principle is worth dwelling on. Every component you mark `"use client"` — *and everything it imports* — ships to the browser as JavaScript. If you mark a big page-level component as client just to make one button interactive, you've sent the whole subtree's code to the browser and turned what could have been server-rendered HTML into client work. Instead, keep the page a Server Component and extract *only* the interactive bit into a small client leaf:

```tsx
// GOOD: the page stays a Server Component (no JS shipped for it);
// only the small interactive piece is a client leaf.
import { LikeButton } from "./LikeButton"; // "use client" — tiny

export default async function PostPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;                 // ⚡ async params in Next.js 16
  const post = await getPost(id);              // server-rendered, no client JS for the body
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
      <LikeButton postId={post.id} initialLikes={post.likes} /> {/* only this hydrates */}
    </article>
  );
}
```

⚡ **Version note:** Next.js 16's **Cache Components** (`use cache`) productionize PPR — a route can declare cached, static portions and dynamic, per-request holes, delivered in one streamed response. The exact directive syntax and configuration evolve; confirm against [Next.js 16 §8–§9](NEXTJS_16_GUIDE.md). The security rule is unchanged: never let a per-user value land in a *statically cached* portion.

### 13.3 DB connection pooling **[A]**

Serverless functions each open DB connections; without pooling you exhaust Postgres. Use a **pooler** — **Supabase's connection pooler (transaction mode)** or **PgBouncer** — for the app's `DATABASE_URL`, and a **direct (unpooled)** connection only for migrations (`DIRECT_URL`). Prisma's `directUrl` (§3.2) exists for exactly this split. See [PostgreSQL](POSTGRESQL_GUIDE.md) and [Database Server Admin].

Why the split exists, concretely: a **transaction-mode pooler** multiplexes many short-lived function invocations onto a small set of real Postgres connections, which is exactly what serverless needs (hundreds of concurrent invocations, each wanting a connection, would otherwise blow past Postgres's `max_connections`). But transaction-mode pooling can't support session-level features that span statements — including some DDL and advisory locks that `prisma migrate` relies on — which is why **migrations must use the direct, unpooled connection**. Set `connection_limit=1` on the pooled URL in serverless so each invocation holds at most one connection. On a long-running (non-serverless) deployment the pressure is lower, but a pooler is still good practice under load. Getting this wrong manifests as intermittent `too many connections` or `prepared statement already exists` errors under traffic — both classic symptoms of a pooling misconfiguration.

### 13.4 Redis caching & edge vs node **[A]**

Cache expensive, shareable computations in [Redis](REDIS_GUIDE.md) (with sensible TTLs and explicit invalidation on writes) — but, again, never cache authorized per-user data at a shared key. Choose the **runtime** per route: the **Node runtime** for anything needing Node APIs (Prisma, crypto, the full Node stdlib — most of a banking app); the **edge runtime** only for ultra-light, globally-distributed work (simple redirects, geolocation) where its constraints are acceptable. Optimize images with `next/image` and serve static assets via a CDN.

The edge-vs-node choice has real constraints worth knowing: the **edge runtime** is a restricted, Web-APIs-only environment — no Node `crypto` module access to the full API, no direct TCP (so no Prisma over a raw socket), smaller bundle limits — but it runs geographically close to users with near-zero cold start. A banking app's authenticated routes need Prisma, Node `crypto` (for AES-GCM, §8.3), and other Node APIs, so they run on the **node runtime**. Reserve the edge for the few things that benefit from it and fit its constraints: lightweight redirects, A/B bucketing, geolocation-based routing, and simple header manipulation in middleware. Don't reach for edge by default; reach for it when a specific route is latency-critical *and* edge-compatible.

A quick caching decision aid:

| Data | Where to cache | TTL | Why |
|---|---|---|---|
| Marketing copy, public lists | Full Route Cache / CDN | long | Identical for everyone |
| Exchange rates, public reference | `unstable_cache` + tag | minutes | Shared; bust on update |
| A user's dashboard/balance | Don't cache (or per-user key) | seconds if at all | Per-user; leak risk |
| Auth/session state | Never cache at a shared layer | — | Security-critical |

---

## 14. Observability

You cannot operate (or secure) what you cannot see. Observability is the difference between "a customer reported money missing" and "our alerting paged us the instant the anomaly started."

### 14.1 Structured logging **[I/A]**

Log as **structured JSON** (one object per event with consistent fields: timestamp, level, request id, user id, route, latency), not free-form strings — structured logs are queryable and aggregatable. Attach a **request/correlation id** (set in middleware) so you can trace one request across the app, the DB, and the WS service. **Never log secrets, passwords, tokens, full card numbers, or unneeded PII** — logs are a frequent leak vector. Separate the security **audit log** (§12.11, durable, append-only) from operational logs (higher volume, shorter retention).

```ts
// src/lib/logger.ts — minimal structured logger with automatic PII/secret redaction.
import "server-only";

const REDACT = ["password", "token", "secret", "authorization", "cookie", "code", "ssn", "card"];

function scrub(obj: Record<string, unknown>): Record<string, unknown> {
  const out: Record<string, unknown> = {};
  for (const [k, v] of Object.entries(obj)) {
    out[k] = REDACT.some((r) => k.toLowerCase().includes(r)) ? "[redacted]" : v;
  }
  return out;
}

export function log(level: "info" | "warn" | "error", msg: string, ctx: Record<string, unknown> = {}) {
  // One JSON object per line — ingestible by any log pipeline (Loki, Datadog, CloudWatch).
  console[level === "error" ? "error" : "log"](JSON.stringify({
    ts: new Date().toISOString(), level, msg, ...scrub(ctx),
  }));
}
```

The redaction step matters more than it looks: the most common way secrets end up in logs is a developer logging a whole request object or error context "to debug something" and forgetting it. Building redaction into the logger so it happens *by default* turns a recurring footgun into a non-issue. Set a correlation id in middleware and pass it into `ctx` so every line for one request shares an id you can grep on.

### 14.2 Error tracking, tracing, health checks, metrics **[I/A]**

- **Error tracking:** capture server and client errors with stack traces and context (Sentry-style), but **scrub PII/secrets** before sending to any third party.
- **Tracing:** OpenTelemetry traces across Next.js → DB → WS service show *where* latency lives.
- **Health checks:** a `/api/health` Route Handler that verifies DB and Redis connectivity, for the load balancer and uptime monitors.
- **Metrics:** request rate, error rate, p50/p95/p99 latency, DB pool saturation, rate-limit rejections, auth failure rate (a spike is an attack signal). Alert on the security-relevant ones.

### 14.3 Security signals to watch **[A]**

Observability is not just for outages; for a banking-grade app it is an active part of your security posture. The following are early-warning signals — wire alerts to them so a human is paged before an incident becomes a breach:

| Signal | What a spike often means |
|---|---|
| Auth failures per IP/user | Credential stuffing or brute force |
| OTP/MFA verification failures | Code brute-forcing or an SMS-flood attack |
| Rate-limit rejections | Automated abuse / scraping in progress |
| 403 (forbidden) rate | An attacker probing for IDOR / authz gaps |
| New-device / new-geo logins | Account takeover attempts |
| Recovery-code use | High-value recovery flow being exercised |
| `service_role` / admin DB calls | Privileged-path usage that must be rare and audited |
| 5xx after a deploy | A migration or rollout regression |

The audit log (§12.11) is the *durable, forensic* record; metrics and alerts are the *real-time* layer over the same events. Together they let you both detect an attack as it happens and reconstruct exactly what occurred afterward — both of which regulators and incident responders will ask for.

```ts
// src/app/api/health/route.ts — liveness/readiness for the load balancer
import { db } from "@/lib/db";
import { redis } from "@/lib/redis";

export async function GET() {
  try {
    await db.$queryRaw`SELECT 1`;        // DB reachable?
    await redis.ping();                   // Redis reachable?
    return Response.json({ status: "ok" });
  } catch {
    return new Response("unhealthy", { status: 503 });
  }
}
```

---

## 15. Testing

Tests are part of banking-grade discipline: an authorization bug must be caught by a failing test, not a customer. Build a pyramid — many fast unit tests, fewer component tests, a focused set of end-to-end tests over the critical security flows.

### 15.1 Unit tests with Vitest **[A]**

Unit-test pure logic with **Vitest**: Zod schemas, the RBAC `can`/`assert` functions, the money/transfer math, the crypto round-trip (encrypt → decrypt), the OTP hashing/expiry logic. These are fast and should run on every save. Keep these functions *pure* (no DB, no network) precisely so they're trivially testable — the transfer math takes numbers and returns numbers; the encrypt/decrypt round-trip takes a string and returns it unchanged. Testability is a design pressure that pushes you toward exactly the small, focused, side-effect-free functions that are also easiest to reason about and to secure.

```ts
// crypto.test.ts — the round-trip property and the tamper-detection guarantee.
import { describe, it, expect } from "vitest";
import { encrypt, decrypt } from "@/lib/crypto";

describe("AES-256-GCM", () => {
  it("round-trips plaintext exactly", () => {
    const secret = "JBSWY3DPEHPK3PXP";          // a sample TOTP secret
    expect(decrypt(encrypt(secret))).toBe(secret);
  });
  it("produces a different ciphertext each time (unique IV)", () => {
    expect(encrypt("same")).not.toBe(encrypt("same")); // nonce reuse would break this
  });
  it("rejects tampered ciphertext", () => {
    const enc = encrypt("data");
    const tampered = enc.slice(0, -2) + (enc.endsWith("00") ? "11" : "00");
    expect(() => decrypt(tampered)).toThrow();   // GCM auth tag catches it
  });
});
```

```ts
// rbac.test.ts
import { describe, it, expect } from "vitest";
import { can } from "@/lib/rbac";

describe("rbac", () => {
  it("denies customers admin permissions", () => {
    expect(can("CUSTOMER", "account:freeze")).toBe(false);
  });
  it("grants admins user management", () => {
    expect(can("ADMIN", "user:manage")).toBe(true);
  });
});
```

### 15.2 Component tests with Testing Library **[A]**

Test Client Components with **React Testing Library** from the user's perspective (roles, labels, interactions) rather than implementation details — does the transfer form show the validation error, disable submit while pending, surface the server error?

### 15.3 E2E with Playwright, and testing Server Actions & auth flows **[A]**

**Playwright** drives a real browser through whole flows: sign up → verify OTP → enroll MFA → log in with TOTP → make a transfer → see the realtime update. For security, write **negative** tests as first-class citizens — they're what prove your defenses:

```ts
// e2e/idor.spec.ts — prove a user cannot read another user's account by URL tampering.
import { test, expect } from "@playwright/test";

test("a customer cannot view another customer's account (IDOR)", async ({ page }) => {
  await loginAs(page, "alice@example.com");           // helper logs in via the real UI
  // Alice tries to open Bob's account id directly.
  await page.goto("/accounts/bob-account-id");
  // The ownership-scoped query returns nothing → 404, not Bob's data.
  await expect(page.getByText("Not found")).toBeVisible();
  await expect(page.getByText(/balance/i)).toHaveCount(0);
});

test("login is rate-limited after repeated failures", async ({ page }) => {
  await page.goto("/login");
  for (let i = 0; i < 6; i++) {
    await page.getByLabel("Email").fill("alice@example.com");
    await page.getByLabel("Password").fill("wrong-password");
    await page.getByRole("button", { name: "Sign in" }).click();
  }
  await expect(page.getByText(/too many/i)).toBeVisible();   // limiter engaged
});
```

The negative tests every banking-grade app should have:

- A customer **cannot** call an admin Server Action (expect 403/forbidden).
- A user **cannot** read another user's account by changing the id in the URL (IDOR).
- Rate limiting kicks in after N login attempts.
- An OTP is single-use and expires.
- A Server Action rejects malformed/extra fields.

Test Server Actions both via the UI (Playwright) and directly (invoke the action with crafted input in an integration test, asserting auth/validation rejects bad calls) — because the action is independently callable, your tests should be too.

```ts
// transfer.security.test.ts — negative tests are the proof your defenses work.
import { describe, it, expect, vi } from "vitest";
import { transferAction } from "@/features/transfers/actions";

describe("transferAction security", () => {
  it("rejects an unauthenticated caller", async () => {
    mockSession(null);
    await expect(transferAction(validInput())).rejects.toThrow(); // redirect/throw, never proceeds
  });

  it("rejects a transfer from an account the user does not own (IDOR)", async () => {
    mockSession({ id: "user-A" });
    const res = await transferAction({ ...validInput(), fromId: "account-owned-by-B" });
    expect(res.ok).toBe(false);                     // ownership check in the WHERE clause stops it
  });

  it("rejects malformed/extra fields (mass assignment)", async () => {
    mockSession({ id: "user-A" });
    // @ts-expect-error — deliberately send a field the schema does not allow
    await expect(transferAction({ ...validInput(), role: "ADMIN" })).resolves.toMatchObject({ ok: false });
  });

  it("enforces the rate limit after the cap", async () => {
    mockSession({ id: "user-A" });
    for (let i = 0; i < 5; i++) await transferAction(validInput());
    await expect(transferAction(validInput())).rejects.toThrow(/Too many/);
  });
});
```

### 15.4 What to test, prioritized **[A]**

You cannot test everything, so prioritize by *blast radius*. In descending order of importance for a banking-grade app: (1) **authorization** — every role gate and every ownership check, including negative cases; (2) **money correctness** — transfer math, idempotency, the concurrency race (§3.5) under simulated parallel calls; (3) **auth flows** — sign-up, OTP, MFA step-up, session rotation, logout-revokes-server-side; (4) **validation** — that every Server Action rejects malformed and extra input; (5) **the happy paths**, last, because a feature that works but is insecure is worse than a feature that doesn't. A useful CI gate: fail the build if coverage on `lib/rbac.ts`, the actions, and the crypto/OTP modules drops below a high threshold — those are the files where a bug becomes a breach.

---

## 16. Deployment & Production

### 16.1 Vercel vs self-host **[A]**

- **Vercel** (the managed Next.js platform): least operational effort — automatic builds, serverless/edge scaling, env-var secret storage, preview deployments. Trade-offs for banking-grade: serverless means no long-lived WS server (run that separately, §11), watch connection pooling (§13.3), and confirm data-residency/compliance requirements are met.
- **Self-host with Docker + Node** ([Docker](DOCKER_GUIDE.md), [Node.js](NODEJS_GUIDE.md)): full control over runtime, network isolation, and compliance boundaries, behind [Nginx](NGINX_GUIDE.md). More to operate (scaling, TLS, patching), but often required for strict regulatory environments. Use Next.js's `output: "standalone"` to produce a minimal server bundle for a small image.

| Concern | Vercel (managed) | Self-host (Docker + Node) |
|---|---|---|
| Operational effort | Minimal | High (scaling, TLS, patching, monitoring) |
| Scaling | Automatic (serverless/edge) | You build it (orchestrator, autoscaler) |
| Long-lived WS server | Run separately regardless | Can co-locate the WS service |
| Connection pooling | Must use a pooler (serverless) | Pool as you like |
| Data residency / compliance | Confirm region & certifications | Full control of the boundary |
| Secrets | Encrypted env vars | Orchestrator/vault secrets |
| Cost model | Per-usage | Fixed infra + ops time |

For most teams Vercel is the pragmatic start; strict regulatory or data-residency requirements, or a desire to co-locate everything behind your own Nginx, push toward self-hosting. Either way the *application* code is identical — the architecture in this guide is deployment-agnostic, which is itself a benefit (you can move later without a rewrite).

```dockerfile
# Multi-stage build for a lean standalone Next.js image (self-host path).
FROM node:24-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:24-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npx prisma generate && npm run build   # next.config: output: "standalone"

FROM node:24-alpine AS run
WORKDIR /app
ENV NODE_ENV=production
# Run as a non-root user — never run the app as root.
RUN addgroup -S app && adduser -S app -G app
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static
COPY --from=build /app/public ./public
USER app
EXPOSE 3000
CMD ["node", "server.js"]
```

### 16.2 Migrations on deploy, behind Nginx, CDN, zero-downtime **[A]**

- **Migrations on deploy:** run `prisma migrate deploy` (applies committed migrations, never generates them) as a **separate, gated step before** the new app version goes live — not at app boot (concurrent boots would race). Make migrations **backward-compatible** (expand-then-contract: add columns first, deploy code that tolerates both shapes, remove old columns in a later release) to enable zero-downtime rolling deploys.

The **expand-contract** pattern deserves a concrete walkthrough because getting it wrong causes downtime or data loss. Say you're renaming `balance` to `balanceCents`. The unsafe move is a single migration that renames the column while the old code is still running — old instances query a column that no longer exists and 500. The safe sequence is three deploys: (1) **expand** — add `balanceCents`, backfill it, and write to *both* columns from the app; old and new code both work. (2) **migrate reads** — deploy code that reads `balanceCents`; both columns still exist. (3) **contract** — once no running code references `balance`, a final migration drops it. At every step the database and the running code are mutually compatible, so rolling deploys never break. This discipline is non-negotiable for an app that can't take a maintenance window.

```yaml
# .github/workflows/deploy.yml — gated pipeline: test → build → migrate → release
name: deploy
on: { push: { branches: [main] } }
jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 24 }
      - run: npm ci
      - run: npx prisma generate
      - run: npm run lint
      - run: npm run typecheck            # tsc --noEmit — types are part of the contract
      - run: npm run test                 # Vitest unit + integration (incl. negative auth tests)
      - run: npm audit --audit-level=high # dependency security gate
      - run: npx gitleaks detect          # secret scanner — fail if a secret leaked into git
      - run: npm run build
  migrate-and-release:
    needs: ci
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # Apply committed migrations BEFORE the new version serves traffic.
      - run: npx prisma migrate deploy
        env: { DATABASE_URL: ${{ secrets.MIGRATION_DATABASE_URL }} } # privileged user, DIRECT_URL
      - run: ./scripts/release.sh         # roll out the new image (blue/green or rolling)
```

The remaining production concerns:

- **Behind Nginx:** terminate TLS, set security headers, proxy `Upgrade`/`Connection` for WebSockets, enforce request size limits and timeouts. See [Nginx](NGINX_GUIDE.md) and [Networking](NETWORKING_GUIDE.md).
- **CDN:** serve static assets and images from a CDN; never let it cache authenticated responses.
- **Secrets:** inject via the platform/orchestrator secret store, never baked into the image.
- **Rollback plan:** every deploy needs a tested way back. Keep the previous image tagged and ready; because migrations are expand-contract (backward-compatible), rolling the *code* back is safe without touching the database. Practice the rollback before you need it — a rollback you've never run is not a plan.
- **Health-gated rollout:** the orchestrator should route traffic to a new instance only after its `/api/health` (§14.2) passes, and roll back automatically if error rates spike post-deploy.

```nginx
# /etc/nginx/conf.d/app.conf — TLS termination + reverse proxy for self-hosted Next.js
server {
  listen 443 ssl http2;
  server_name app.example.com;

  ssl_certificate     /etc/letsencrypt/live/app.example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;
  ssl_protocols TLSv1.2 TLSv1.3;                  # no legacy protocols

  add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
  client_max_body_size 10m;                       # cap upload size (DoS + upload abuse)

  location / {
    proxy_pass http://127.0.0.1:3000;             # the Next.js standalone server
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;   # so the app knows it's HTTPS
  }
}

# Separate vhost for the WebSocket service — note the Upgrade headers.
server {
  listen 443 ssl http2;
  server_name realtime.example.com;
  # ...ssl...
  location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;        # required for the WS handshake
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 3600s;                      # long-lived sockets
  }
}

# Redirect all plaintext to HTTPS.
server { listen 80; server_name app.example.com realtime.example.com; return 301 https://$host$request_uri; }
```

> **Production checklist (banking-grade):**

| Area | Must-have |
|---|---|
| Secrets | All server-only; none in `NEXT_PUBLIC_`; in a secret store; rotated |
| Database | Pooled `DATABASE_URL`; unpooled `DIRECT_URL`; RLS on; backups + restore tested |
| Auth | Argon2id; HttpOnly/Secure/SameSite cookies; MFA available; revocation works |
| Authorization | Role + ownership checks in every action/handler; middleware is gate-only |
| Transport | HTTPS + HSTS; wss; TLS to DB/Redis |
| Headers | CSP (no `unsafe-inline`), HSTS, nosniff, frame-ancestors, Permissions-Policy |
| Rate limiting | Redis-backed, on login/OTP/transfers, by user + IP |
| Validation | Zod at every server entry point; upload magic-byte checks |
| Migrations | `migrate deploy` gated; backward-compatible; zero-downtime |
| Observability | Structured logs (no secrets), error tracking, health check, alerts |
| Audit | Append-only trail for auth + money + admin events |
| Realtime | Separate WS service; socket authenticated + user-scoped; Redis backplane |
| Deps | `npm audit`/SCA clean; lockfile committed; image runs as non-root |

---

## 17. Gotchas & Best Practices

- **Server/client boundary leaks.** Passing a full Prisma row to a Client Component or returning it from a Server Action ships every column (password hash, TOTP secret) to the browser. *Always project to a minimal DTO.* Guard server modules with `import "server-only"`.
- **`NEXT_PUBLIC_` leaks.** Anything with that prefix is in the bundle forever. Audit every one; never put `service_role`, signing keys, or DB URLs there. Grep the build for secrets in CI.
- **RLS + `service_role` misuse.** The `service_role` key **bypasses RLS** — using it client-reachable (or disabling RLS) is a full database compromise. Keep it server-only and still authorize in app code. Turn RLS *on* for every user-data table.
- **Middleware auth limits.** `middleware.ts` is a coarse gate, runs on every request (keep it fast — no heavy DB), and a matcher typo silently opens routes. **Always re-check authorization in the data layer**, never rely on middleware alone.
- **Server Action security.** Every Server Action is a public, attacker-callable RPC. Authenticate, authorize (role **and** ownership), and validate (Zod) **inside every action** — "the page already checked" is not a defense.
- **IDOR.** Don't fetch-then-compare ownership; **scope queries by the session-derived owner id** so "not yours" is indistinguishable from "doesn't exist".
- **Mass assignment.** Never `data: { ...input }`. Parse with a Zod schema listing only allowed fields, so the client can't set `role`/`balanceCents`/`userId`.
- **Caching surprises.** `fetch` is uncached by default in 16; opt in deliberately. **Never** cache per-user/authorized responses at a shared layer — it's a data-breach vector. Call `revalidatePath`/`revalidateTag` after mutations.
- **N+1 queries.** Use `include`/`select` or batch `findMany({ where: { id: { in } } })`; never query in a loop. `select` only what the UI needs (perf *and* boundary hygiene).
- **Money as float.** Never store money in a float; use `BigInt` cents or `Decimal`. Use transactions for multi-step money moves and idempotency keys for retries.
- **OTP/MFA done wrong.** Hash codes at rest, single-use, short TTL, rate-limited, attempt-capped; encrypt TOTP secrets (AES-256-GCM); hash recovery codes; avoid user enumeration.
- **OAuth auto-link on unverified email.** Only link accounts on a **verified** email, or an attacker hijacks accounts. Validate post-login redirects against an allowlist (open-redirect).
- **WebSocket in serverless.** You can't host a long-lived socket in serverless Next.js. Run a separate Node WS service, authenticate the socket with the session, scope channels per user, and scale with Redis pub/sub.
- **Connection exhaustion.** Use the Prisma singleton in dev; use a pooler (`DATABASE_URL`) + direct URL (`DIRECT_URL`) in production.
- **Cookie-only logout.** Always delete the **server-side** session record on logout, not just the cookie.
- **Migrations at boot.** Run `prisma migrate deploy` as a gated pre-deploy step, not on app start (boot races). Keep migrations backward-compatible.
- **Leaking errors to the client.** Don't return raw exception/DB error strings from actions or handlers — log them server-side, return a generic message. Stack traces and "row not found" detail help attackers (§5.2).
- **Trusting realtime payloads.** A pushed WebSocket event is a hint to refetch, not authoritative data. Re-fetch through an authorized path before acting on it (§11.3).
- **Forgetting the second factor's pending state.** MFA is only effective if a password-only session is blocked from sensitive routes server-side (`requireFullAuth`), not merely hidden in the UI (§8.3.1).

---

## 18. Study Path & Build-to-Learn Projects

The fastest way to internalize this guide is to build the capstone app it describes, one secure layer at a time. Follow the path; at each step, read the cross-referenced guide for depth, then integrate it here.

**Phase 1 — Foundation [B].** Scaffold the app (§2). Set up the typed env (§2.4) and the Prisma singleton (§3.3). Model a minimal schema (users, accounts) and run your first migration. Render an accounts list in a Server Component scoped to a hardcoded user. *Read alongside:* [Next.js 16](NEXTJS_16_GUIDE.md), [React 19](REACT_19_GUIDE.md), [Prisma ORM](PRISMA_ORM_GUIDE.md), [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md).

**Phase 2 — Auth core [I].** Add Better Auth with email/password (Argon2id), secure cookies, and the App Router handler (§6). Add `requireUser`/`getSession`. Gate routes with middleware (§10.6) and re-check in the data layer. *Read:* [Better Auth](BETTERAUTH_GUIDE.md), [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md).

**Phase 3 — Mutations & validation [I].** Implement transfers as a Server Action with the full checklist — authenticate, authorize, Zod-validate, transaction, audit, revalidate (§5.2). Build the form with React Hook Form + shared Zod (§5.7). Add a money transfer with idempotency. *Read:* [React Hook Form](REACT_HOOK_FORM_GUIDE.md), [TanStack Query](TANSTACK_QUERY_GUIDE.md), [Zustand](ZUSTAND_GUIDE.md).

**Phase 4 — Verification & MFA [A].** Add email OTP (hashed, single-use, TTL, rate-limited — §7), then TOTP MFA with encrypted secrets and hashed recovery codes (§8). Implement the step-up flow. *Read:* [Go JWT + Argon2 §18–§19](GO_JWT_ARGON2_GUIDE.md), [Redis](REDIS_GUIDE.md).

**Phase 5 — OAuth & RBAC [A].** Add a social provider with PKCE and verified-email linking (§9). Build the RBAC model and enforce role + ownership in every server entry point; add Supabase RLS as defense in depth (§4.4, §10). *Read:* [Supabase](SUPABASE_GUIDE.md), [PostgreSQL](POSTGRESQL_GUIDE.md).

**Phase 6 — Realtime [A].** Stand up a separate Node WS service, authenticate sockets with the session, publish user-scoped balance events via Redis pub/sub, and consume them with the client hook (§11). *Read:* [Node.js](NODEJS_GUIDE.md), [Redis](REDIS_GUIDE.md), [Networking](NETWORKING_GUIDE.md).

**Phase 7 — Harden [A].** Walk the entire §12 checklist: CSP/nonces, HSTS and headers, rate limits everywhere, file-upload validation, secret scanning in CI, audit logging, encryption at rest/in transit, dependency audit. Write the negative security tests (§15.3). *Read:* [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md), [Nginx](NGINX_GUIDE.md).

**Phase 8 — Ship [A].** Add observability (structured logs, health check, error tracking, alerts — §14). Containerize and deploy behind Nginx with gated migrations and zero-downtime; complete the production checklist (§16). *Read:* [Docker](DOCKER_GUIDE.md), [Nginx](NGINX_GUIDE.md).

**Capstone project — a fintech-style dashboard.** Build a full app a hiring manager would respect: customers sign up (email OTP verification), enroll TOTP MFA, log in with step-up, and land on a dashboard showing accounts and live balances. They can transfer money (validated, atomic, idempotent, audited) and watch the balance update over WebSocket in real time. Support staff get a role-gated view (read-only access, ticket handling); admins can freeze accounts and read the audit log. Every action is authorized by role **and** ownership, every input is Zod-validated, every secret is server-only, every money-relevant event is audited, and the whole thing runs behind Nginx with security headers, rate limiting, and RLS as a database-layer backstop. Add payments-shaped flows (a signed, idempotent webhook crediting an account) to exercise Route Handlers. When it's done, write the threat model document for it — listing each attack from §12 and pointing at the code that defends it. That document, plus the working app, is the proof you've mastered building a banking-grade standalone full-stack Next.js application.

**Further directions:** add passkeys/WebAuthn (Better Auth passkey plugin) as a phishing-resistant factor; add a separate Go API verifying Better Auth JWTs ([Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md), [Better Auth §9](BETTERAUTH_GUIDE.md)); add a background worker for scheduled statements; and run a self-directed security review against the §12 table.

**How to study this guide effectively.** Don't read it once and move on. Read it top-to-bottom the first time to build the mental model, then *build* the capstone in the phased order above, returning to the relevant section as each phase begins — the guide is designed to be re-entered. For each tool you integrate, open its dedicated guide in parallel: this capstone shows the *composition and the security glue*, while the per-tool guides go deep on the API surface. Keep the §17 gotchas list and the §12 attack→defense table next to you while you build; treat them as a checklist, not as reading. The fastest path to mastery is the loop: read a section, build the slice it describes, write the negative security test that proves it's safe, then move to the next. By the end you won't have just read about a banking-grade full-stack Next.js app — you'll have built one, and you'll be able to explain, for every attack in the OWASP top ten, exactly which line of your code defends against it and why.

Mastery here is not memorizing APIs (those move; your editor and the per-tool guides cover them). It's internalizing the *invariants* — never trust the client, defense in depth, the server is the only authority, scope every query by the owner, validate at every entry point, keep secrets server-side, and make money operations atomic and idempotent. Those principles outlast any framework version, and they are what separates a developer who can wire features together from one who can build a system people trust with their money.
