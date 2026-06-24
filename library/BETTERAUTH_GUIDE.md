# Better Auth (Standalone, Next.js & Go) — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Developers who want to *own their authentication* — the data, the flows, the database tables, the security posture — instead of renting it from a SaaS. This guide takes you from "I have only ever used a hosted login button" all the way to "I run Better Auth as the identity source of truth for a Next.js frontend *and* a separate Go microservice, with 2FA, passkeys, organizations, and offline JWT verification." Every concept is explained in **prose first** — what it is, the underlying logic and *why* it works that way, what it's for and when to reach for it, how to use it, the key options/parameters, best practices, and — because this is authentication — explicit **security recommendations** — and only then shown as heavily-commented, runnable code. Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced. The Go sections assume basic Go (see the cross-referenced Go guides).
>
> **Version note:** This guide targets **Better Auth v1.x** (the stable line throughout 2025–2026). Better Auth is, as of 2026, the most popular *self-hosted, comprehensive, type-safe* TypeScript auth library: framework-agnostic, you own your database, no per-user pricing. The core API (`betterAuth()`, `createAuthClient()`, the plugin system, the four core tables, the handler-is-`Request→Response` model) is stable. Plugins and adapter details move faster — anything fast-moving is flagged with **⚡ Version note**. For any exact symbol you're unsure about, confirm against the official docs at better-auth.com and trust your editor's *inferred* types over memory. Where I am not 100% certain of an exact export name or route, I say so rather than invent it.
>
> **This guide's place in the library:** It cross-references the **Next.js** guide (App Router, Server Components, middleware), the **PostgreSQL/Prisma** guides (the database underneath), and especially **`GO_JWT_ARGON2_GUIDE.md`** (the Go side of JWT validation and password hashing). Better Auth is the *identity layer*; those guides are the surrounding stack. Where this guide shows Go JWT verification, the JWT/Argon2 guide goes deeper on the cryptography.

---

## Table of Contents
1. [What Better Auth Is & How It Compares](#1-what-better-auth-is--how-it-compares) **[B]**
2. [Core Concepts: Sessions vs JWTs, Server, Client, Schema, Adapters](#2-core-concepts) **[B/I]**
3. [Standalone Setup (Node / Hono)](#3-standalone-setup) **[B/I]**
4. [Email & Password Authentication](#4-email--password-authentication) **[B/I]**
5. [Social / OAuth Providers](#5-social--oauth-providers) **[I]**
6. [Sessions & Cookies](#6-sessions--cookies) **[I]**
7. [Plugins Ecosystem (2FA, Passkeys, Orgs, Magic Links, JWT, Bearer)](#7-plugins-ecosystem) **[I/A]**
8. [Next.js Integration (App Router)](#8-nextjs-integration) **[I/A]**
9. [Golang Integration (Verifying Better Auth JWTs)](#9-golang-integration) **[A]**
10. [Authorization — Roles, RBAC, Organizations & Admin](#10-authorization) **[I/A]**
11. [Security Best Practices](#11-security-best-practices) **[I/A]**
12. [Database Schema Reference & Extending the User Model](#12-database-schema-reference) **[I]**
13. [Testing & Local Development](#13-testing--local-development) **[I]**
14. [Gotchas & Best Practices](#14-gotchas--best-practices) **[A]**
15. [Study Path & Build-to-Learn Projects](#15-study-path--build-to-learn-projects)

---

## 1. What Better Auth Is & How It Compares

### 1.1 What it is **[B]**

**Better Auth** is a TypeScript authentication *framework* — not a SaaS, not a hosted service, not a UI kit. You install it as a normal dependency (`npm install better-auth`), point it at *your own* database, and it manages the full lifecycle of authentication: sign-up, sign-in, sessions, OAuth/social login, email verification, password reset, two-factor, passkeys, organizations, API keys, and more — extended through a plugin system. There is no external dashboard you log into, no "user pool" living in someone else's cloud, and no per-monthly-active-user bill.

The reason a "framework" framing matters: most auth libraries are either (a) *toolkits* that hand you primitives (hash this, sign that) and leave you to wire the flows yourself — powerful but error-prone, because auth has a thousand sharp edges — or (b) *hosted services* that hand you a polished login box but own your users. Better Auth sits deliberately in the middle: it implements the *correct, secure flows* for you (CSRF-resistant cookies, PKCE for OAuth, constant-time-ish credential checks, token rotation) while keeping the *data and the deployment* entirely yours. You get the safety of a managed service with the ownership of a self-hosted one.

The three pillars of its philosophy — and the *why* behind each:

1. **Own your data.** Better Auth stores users, sessions, and accounts in **your** database (Postgres, MySQL, SQLite, etc.) via an adapter (Prisma, Drizzle, Kysely, or a direct driver). *Why this matters:* you can `SELECT * FROM "user"` at any time, join auth data to your domain data in a single query, export everything, run your own analytics, and migrate providers without a data-export ticket. There is no lock-in surface.
2. **Framework-agnostic.** The core is a single request handler that speaks the Web-standard `Request`/`Response` interface. *Why this matters:* the *same* `auth` instance runs unchanged on Next.js, Nuxt, SvelteKit, Remix, Astro, Hono, Express, Bun, Node, and Cloudflare Workers. You learn one mental model and it transfers everywhere. It also means a Go or Python *backend* can sit alongside it as a resource server (§9).
3. **Fully type-safe.** The client is *inferred* from the server config — no codegen step. *Why this matters:* add a plugin on the server and the matching client method appears, fully typed, in your editor. Add a custom field to the user model and `session.user.yourField` becomes typed across your whole app. Types are the documentation, and they cannot drift from reality because they *are* reality.

**The mental model — burn this in.** There is **one server `auth` object** (created with `betterAuth({...})`) which is the source of truth. It exposes:
- An HTTP **handler** (`auth.handler`) you mount under `/api/auth/*`. Every auth route lives behind this one function.
- A server-side **API** (`auth.api.*`) for calling the same operations *directly in your backend code* with no HTTP round-trip — used inside Server Components, Server Actions, middleware, route handlers.

…and **one client `authClient`** (created with `createAuthClient({...})`) that your frontend calls: `authClient.signIn.email(...)`, `authClient.useSession()`, `authClient.signOut()`, and so on. The client is a typed RPC layer over the server's HTTP routes.

### 1.2 When to choose Better Auth — and when not to **[B]**

Choosing an auth approach is a long-term commitment (migrating auth later is painful), so reason about it deliberately. The trade-off axis is **ownership/control vs. speed-to-market/maintenance burden**.

| You want… | Better Auth fit |
|---|---|
| To own user data in your own DB, no vendor lock-in | Ideal |
| Type-safe end-to-end auth without a codegen step | Ideal |
| Self-host, no per-MAU billing as you grow | Ideal |
| One auth layer shared across multiple TS frameworks | Ideal |
| Rich features (2FA, passkeys, orgs, magic link) without stitching libraries | Ideal — plugins |
| A drop-in hosted UI with literally zero backend (you don't want a DB) | Use Clerk/Auth0 instead |
| Compliance/certifications handled *for* you (SOC2 of the auth vendor) | A hosted vendor carries that for you |
| Pure non-JS shop (no TypeScript anywhere) | Use a native solution — though Go *resource servers* pair beautifully (§9) |

The honest summary: **choose Better Auth when ownership, cost-at-scale, customization, and a unified TS stack matter; choose a hosted vendor when you want to ship a login box this afternoon and never think about auth infrastructure again.** Most product teams that expect to grow and want control land on Better Auth; very early prototypes and teams who refuse to run a database lean hosted.

### 1.3 How it compares **[B]**

| Feature | Better Auth | NextAuth / Auth.js | Clerk | Lucia | Supabase Auth |
|---|---|---|---|---|---|
| Type | Self-hosted lib | Self-hosted lib | SaaS (hosted) | (Deprecated lib) | SaaS / self-host w/ Supabase |
| Own your data | Yes (your DB) | Yes (your DB) | No (Clerk's cloud) | Yes (your DB) | Yes, but in Supabase's Postgres/schema |
| Framework-agnostic | **Yes** | Mostly Next-centric (Auth.js broadened it) | SDK per framework | Yes (low-level) | Client SDKs |
| Built-in 2FA / passkeys / orgs | **Yes (plugins)** | Limited / DIY | Yes (paid tiers) | DIY (it's a toolkit) | Partial |
| Hosted UI components | No (you build UI) | No | **Yes** | No | Some (Auth UI) |
| Pricing | Free (OSS) | Free (OSS) | Per-MAU | Free (OSS) | Free tier + usage |
| Type safety | **Excellent (inferred)** | Good | Good | Good (manual) | Good |
| 2026 status | **Actively growing, popular** | Mature, stable | Mature, popular | **Deprecated** | Mature |

**Lucia is deprecated.** Lucia was a beloved low-level auth *toolkit* — you assembled sessions yourself from its primitives. Its maintainer announced its sunset and pointed people toward learning resources and higher-level options. Better Auth absorbed much of that audience by offering Lucia's "own your data" ethos *plus* batteries-included features. If a 2026 tutorial recommends Lucia for a *new* project, prefer Better Auth.

**vs NextAuth / Auth.js:** Auth.js is excellent and battle-tested, but historically Next-centric and lighter on built-in *advanced* features (2FA, orgs, admin tooling) — you often reach for extra libraries. Better Auth is more "a framework, not just adapters," with a richer first-party plugin ecosystem and stronger *inferred* types. Both let you own your DB.

**vs Clerk:** Clerk gives you gorgeous drop-in UI components (`<SignIn />`) and zero backend maintenance — at the cost of vendor lock-in, per-MAU pricing that bites at scale, and your users physically living in Clerk's cloud. Choose Clerk for raw speed-to-market when you do not want to own auth; choose Better Auth when ownership, cost, and customization win.

**vs Supabase Auth:** Great if you are already all-in on Supabase (it is tied to Supabase's Postgres + the GoTrue service). Better Auth is database-agnostic and not coupled to a single BaaS.

> **Security framing for the whole guide:** authentication is the part of your app where a single mistake is catastrophic — a leaked session, a forgeable token, a CSRF hole, or a weak password hash can compromise *every* user. Throughout this guide, security is not an appendix; each feature is explained with its security implications inline. The single most important habit: **never hand-roll the parts Better Auth already does for you** (cookie signing, token rotation, OAuth state/PKCE, password hashing). Lean on the framework's defaults, and only deviate with a clear reason.

---

## 2. Core Concepts

Before any setup, internalize the concepts. Two of them — **sessions vs JWTs** and the **server/client/schema/adapter** split — determine almost every architectural decision you will make.

### 2.1 Sessions vs JWTs — the single most important decision **[B/I]**

Almost every confusion in auth comes from blurring these two models. They answer the same question ("is this request from a logged-in user, and who?") in opposite ways.

**Stateful sessions (the database-backed model — Better Auth's default).**
*What it is:* on login, the server creates a row in a `session` table (`token`, `userId`, `expiresAt`) and hands the browser a **signed, httpOnly cookie** holding the opaque session token. On every request, the server reads the cookie, validates the signature, looks up the row, and loads the user.
*The logic / why:* the **server holds the truth**. The cookie is just a pointer ("here is my session id"); authority lives in the DB row. *Why this is the safe default:* you can **revoke any session instantly** by deleting its row — a "log out everywhere," a "this device was stolen," a "ban this user" all take effect on the very next request. The downside is a database lookup per request (mitigated by a short-lived cookie cache, §6.3) and the need for a shared DB across server instances.
*When to use it:* browser-facing apps (the overwhelming majority), anything where instant revocation matters (banking, admin tools, anything sensitive), single-language or monolith deployments. **This is the right default for a Next.js app.**

**Stateless JWTs (the token model).**
*What it is:* on login, the server signs a JSON Web Token containing the user's identity (`sub`, `email`, `role`, `exp`) and gives it to the client. The client sends it back as `Authorization: Bearer <jwt>`. Any server with the *public key* can verify the signature **offline** — no database lookup.
*The logic / why:* the **token is self-contained truth**, cryptographically signed so it cannot be forged or tampered with. *Why you would want this:* zero per-request DB hits and trivial horizontal scaling, and — critically for this guide — a **separate service in another language (your Go API) can verify the token without ever calling back to Better Auth**. The hard downside: a JWT **cannot be revoked before it expires** (the server isn't consulted), so a stolen short-lived JWT is valid until `exp`. You mitigate this by keeping JWT lifetimes short (minutes) and pairing them with a revocable session or refresh mechanism.
*When to use it:* cross-service / cross-language architectures (Next.js + Go, §9), high-throughput APIs where a DB lookup per request hurts, and stateless edge deployments. **Use JWTs at the boundary to other services, not as your primary browser session.**

**The Better Auth way — and why it's the right synthesis.** Better Auth uses **stateful, DB-backed sessions by default** (revocable, secure, cookie-based) for the browser, and offers the **`jwt()` plugin** to *additionally* mint short-lived JWTs when you need to call a separate service. This gives you the best of both: revocable sessions for users, offline-verifiable tokens for service-to-service. You are not forced to choose globally — you choose per boundary.

| | Stateful sessions (default) | Stateless JWTs (`jwt()` plugin) |
|---|---|---|
| Source of truth | DB row | The signed token itself |
| Per-request DB lookup | Yes (cacheable) | No (offline verify) |
| Instant revocation | **Yes** | No (only at `exp`) |
| Horizontal scale | Good (shared DB / cache) | Excellent (no shared state) |
| Cross-language verify | Awkward (forward to get-session) | **Easy (JWKS public key)** |
| Best for | Browser apps, sensitive ops | Service-to-service, Go/other backends |

> **Security recommendation:** default to sessions for anything a human's browser touches; use JWTs only for the machine-to-machine hop. Always keep JWTs short-lived (e.g. 5–15 minutes) and verify `iss`/`aud`/`exp` on the receiving side (§9). Treat a JWT as a *bearer* credential — whoever holds it is the user — so it must travel only over HTTPS and never land in `localStorage` if you can avoid it.

### 2.2 The server `auth` instance **[B]**

Everything starts with one object created by `betterAuth()`. This is the **source of truth** — it knows your DB, your providers, your plugins, your secrets. It is created once, imported everywhere on the server. Treat the file that creates it (`lib/auth.ts`) as the most security-sensitive file in your codebase.

```ts
// lib/auth.ts  — imported by BOTH your route handler and your server-side code.
import { betterAuth } from "better-auth";

export const auth = betterAuth({
  // The canonical base URL where Better Auth's HTTP routes are mounted.
  // Used to BUILD callback/redirect URLs (OAuth, email links). Get it wrong and
  // OAuth breaks and origin checks reject requests. Always from env, never hardcoded.
  baseURL: process.env.BETTER_AUTH_URL, // e.g. "http://localhost:3000"

  // A long random secret used to SIGN cookies/tokens. KEEP SECRET. 32+ bytes.
  // Rotating it invalidates all existing signed cookies/tokens — plan for it.
  secret: process.env.BETTER_AUTH_SECRET,

  // database: ...        (see §3 — Prisma / Drizzle / direct)
  // emailAndPassword: {} (see §4)
  // socialProviders: {}  (see §5)
  // plugins: []          (see §7)
});

// The fully-resolved type of the auth instance — handy for typing helpers.
export type Auth = typeof auth;
```

From `auth` you constantly use two things:

- **`auth.handler`** — a function `(req: Request) => Promise<Response>`. Mount it under `/api/auth/*` and Better Auth handles *every* auth route (sign-in, sign-up, OAuth callbacks, get-session, sign-out, password reset, …). You never write those route bodies yourself.
- **`auth.api.*`** — server-side methods that perform the same operations *in-process* (no HTTP). E.g. `auth.api.getSession({ headers })`, `auth.api.signUpEmail({ body })`, `auth.api.signOut({ headers })`. Use these inside Server Components, Server Actions, middleware, or any backend code where you already have the request context.

> **⚡ Version note:** Exact `auth.api.*` method names follow the route names (`getSession`, `signInEmail`, `signUpEmail`, `signOut`, `listSessions`, …). The set *expands* as you add plugins. Confirm the precise name via your editor's autocomplete — the *pattern* (`auth.api.<operation>({ body?, headers?, query?, asResponse? })`) is stable.

### 2.3 The client **[B]**

The client mirrors the server, with inferred types. You create it once and import it across your frontend (Client Components, browser code).

```ts
// lib/auth-client.ts
import { createAuthClient } from "better-auth/client";
// For React apps, import from the react entry to also get hooks like useSession:
// import { createAuthClient } from "better-auth/react";

export const authClient = createAuthClient({
  // Where the auth HTTP routes live. Can be omitted if same-origin at the default
  // /api/auth, but being explicit is safer and required for cross-origin setups.
  baseURL: process.env.NEXT_PUBLIC_BETTER_AUTH_URL, // e.g. "http://localhost:3000"
  // plugins: [...]  // CLIENT plugins must MIRROR the server plugins (see §7)
});

// Common destructured helpers (names are stable):
export const { signIn, signUp, signOut, useSession, getSession } = authClient;
```

The key magic and *why* it is so valuable: if you add, say, `twoFactor()` on the server and `twoFactorClient()` on the client, then `authClient.twoFactor.*` methods appear *typed* with no codegen. The client is a typed RPC layer over the server's HTTP routes — change the server, the client's types follow automatically.

> **Security note:** the client runs in the browser, so it never sees the `secret` or the database — it only calls HTTP routes. The session cookie is `httpOnly`, meaning the client *cannot read it with JavaScript* (deliberate, to resist XSS). The client knows you are logged in via the `useSession`/`getSession` *responses*, not by reading the cookie.

### 2.4 Database & the core schema **[B/I]**

Better Auth manages **four core tables**. Plugins add more (`twoFactor`, `passkey`, `organization`, `member`, `invitation`, `jwks`, `apiKey`, …). Understanding the four is essential because almost every "it works but behaves weirdly" bug traces back to a misunderstanding here.

| Table | Purpose | Key columns (representative) |
|---|---|---|
| `user` | The person | `id`, `name`, `email`, `emailVerified`, `image`, `createdAt`, `updatedAt` |
| `session` | An active login (one per device/login) | `id`, `userId`, `token`, `expiresAt`, `ipAddress`, `userAgent`, `createdAt`, `updatedAt` |
| `account` | A credential or linked provider | `id`, `userId`, `providerId`, `accountId`, `accessToken`, `refreshToken`, `idToken`, `password` (hashed, for email+password), `expiresAt` |
| `verification` | Short-lived tokens (email verify, password reset, OTP) | `id`, `identifier`, `value`, `expiresAt`, `createdAt` |

Notes that trip people up — and *why* the design is this way:
- **`account` holds the password, not `user`.** For email+password, the hashed password lives in `account` with `providerId = "credential"`. *Why:* one user can have *multiple* `account` rows — one credential row plus one per linked OAuth provider — and that is exactly how **account linking** works (one person, one `user`, many ways to sign in). Putting the password on `user` would break that model.
- **`session.token`** is the opaque session identifier stored (signed) in the cookie. The DB row *is* the server-side session; deleting it revokes the login.
- **`verification`** is a generic key/value-with-expiry table reused by many flows (email verification, password reset, email OTP). It is intentionally generic so plugins can reuse it.

See §12 for the full reference and how to add your own columns (`additionalFields`).

> **Security note:** the `account` table can hold OAuth `accessToken`/`refreshToken`/`idToken` — these are sensitive third-party credentials. Protect this table like you protect passwords: encrypted at rest is ideal, least-privilege DB access, and never log these columns.

### 2.5 Adapters **[I]**

An **adapter** teaches Better Auth how to talk to your specific database/ORM. *Why an adapter layer exists:* it decouples Better Auth's logic from any one database, so the same auth config runs on Postgres via Prisma, MySQL via Drizzle, or SQLite directly. You pick exactly one:

- **Prisma adapter** — `prismaAdapter(prisma, { provider })`. You add Better Auth's models to `schema.prisma`. Best if your app already uses Prisma (see the Prisma guide in this library).
- **Drizzle adapter** — `drizzleAdapter(db, { provider, schema })`. You define the tables in Drizzle schema. Best for a TypeScript-native, SQL-close ORM.
- **Kysely adapter** — for the Kysely query builder.
- **Direct database** — pass a connection (a `pg` `Pool`, a MySQL pool, or a `better-sqlite3` handle) and Better Auth uses its built-in Kysely-based adapter. Simplest path for "no ORM," and the only one where the CLI can *apply* migrations directly.

The **CLI** (`@better-auth/cli`) reads your `auth` config and either **generates** the schema for your ORM (`generate`) or **migrates** the DB directly (`migrate`, supported for the built-in/Kysely adapter). Details in §3.

---

## 3. Standalone Setup

This section gets a *framework-free* Better Auth running: install, configure a database three ways (Prisma, Drizzle, direct Postgres), enable email+password, run migrations, and mount the handler on plain Node and on Hono. Doing it standalone first is the best way to *understand* the model before Next.js hides the wiring.

### 3.1 Install **[B]**

```bash
# Core library
npm install better-auth

# The CLI (used to generate/migrate the schema). Often run via npx.
npm install --save-dev @better-auth/cli

# Pick a DB driver / ORM (one of):
npm install pg                       # direct Postgres
npm install prisma @prisma/client    # Prisma
npm install drizzle-orm postgres     # Drizzle + postgres.js driver
```

Generate a secret and set env vars (`.env`). **The secret is the master key that signs every cookie and token** — generate it with a CSPRNG, never type it by hand, and keep it out of version control:

```bash
# A 32+ byte random string. One quick way (uses Node's crypto = a CSPRNG):
#   node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
BETTER_AUTH_SECRET=replace-with-long-random-hex
BETTER_AUTH_URL=http://localhost:3000
DATABASE_URL=postgres://user:pass@localhost:5432/myapp
```

> **Security recommendation:** in production, load `BETTER_AUTH_SECRET` from a secrets manager (AWS Secrets Manager, Vault, Doppler, your platform's encrypted env), *never* from a committed file. Add `.env` to `.gitignore`. Use a *different* secret per environment. Rotating the secret invalidates all signed cookies/tokens (everyone gets logged out) — that is a feature for incident response, but plan the rotation.

### 3.2 Configure the database — three ways **[B/I]**

#### (a) Direct Postgres (no ORM) **[B]**

The simplest path and the best for learning. Pass a `pg` `Pool`; Better Auth uses its internal adapter and can run migrations for you. *Why start here:* no ORM concepts in the way — you see exactly which tables Better Auth creates.

```ts
// lib/auth.ts
import { betterAuth } from "better-auth";
import { Pool } from "pg";

export const auth = betterAuth({
  baseURL: process.env.BETTER_AUTH_URL,
  secret: process.env.BETTER_AUTH_SECRET,

  // Pass a node-postgres Pool directly. Better Auth detects the dialect and uses
  // its built-in (Kysely-based) adapter. A Pool reuses connections efficiently.
  database: new Pool({ connectionString: process.env.DATABASE_URL }),

  emailAndPassword: { enabled: true },
});
```

Then create the tables — `migrate` both generates *and applies* the SQL against your `DATABASE_URL` (only available for the built-in/direct adapter):

```bash
npx @better-auth/cli migrate
```

#### (b) Prisma adapter **[I]**

Prisma is the most common choice in TS apps and pairs with the Prisma guide in this library. You add Better Auth's models to your schema, then let Prisma migrate.

```prisma
// prisma/schema.prisma  (excerpt — the CLI can GENERATE these models for you)
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
generator client {
  provider = "prisma-client-js"
}

model User {
  id            String    @id @default(cuid())
  name          String
  email         String    @unique
  emailVerified Boolean   @default(false)
  image         String?
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  sessions      Session[]
  accounts      Account[]
}

model Session {
  id        String   @id @default(cuid())
  userId    String
  token     String   @unique
  expiresAt DateTime
  ipAddress String?
  userAgent String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  // onDelete: Cascade => deleting a user removes their sessions (no orphans).
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model Account {
  id                    String    @id @default(cuid())
  userId                String
  accountId             String
  providerId            String
  accessToken           String?
  refreshToken          String?
  idToken               String?
  accessTokenExpiresAt  DateTime?
  refreshTokenExpiresAt DateTime?
  scope                 String?
  password              String?   // hashed; ONLY for the "credential" provider
  createdAt             DateTime  @default(now())
  updatedAt             DateTime  @updatedAt
  user                  User      @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model Verification {
  id         String   @id @default(cuid())
  identifier String
  value      String
  expiresAt  DateTime
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt
}
```

```ts
// lib/auth.ts
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { PrismaClient } from "@prisma/client";

// Reuse one PrismaClient (creating many exhausts DB connections — see Prisma guide).
const prisma = new PrismaClient();

export const auth = betterAuth({
  baseURL: process.env.BETTER_AUTH_URL,
  secret: process.env.BETTER_AUTH_SECRET,

  // The `provider` MUST match your Prisma datasource provider.
  database: prismaAdapter(prisma, { provider: "postgresql" }),

  emailAndPassword: { enabled: true },
});
```

Generate the Prisma models from your config (writes/updates `schema.prisma`), then run Prisma's own migrate:

```bash
npx @better-auth/cli generate   # writes Better Auth models INTO schema.prisma
npx prisma migrate dev          # Prisma applies the migration
```

> **⚡ Version note:** For Prisma/Drizzle the CLI **generates schema** (`generate`); you then run *your ORM's* migration tool to apply it. `migrate` (direct apply) is for the built-in adapter. Don't expect `@better-auth/cli migrate` to drive Prisma.

#### (c) Drizzle adapter **[I]**

```ts
// db/schema.ts  (the CLI can generate this; shown abbreviated)
import { pgTable, text, timestamp, boolean } from "drizzle-orm/pg-core";

export const user = pgTable("user", {
  id: text("id").primaryKey(),
  name: text("name").notNull(),
  email: text("email").notNull().unique(),
  emailVerified: boolean("email_verified").default(false).notNull(),
  image: text("image"),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});

export const session = pgTable("session", {
  id: text("id").primaryKey(),
  userId: text("user_id").notNull().references(() => user.id, { onDelete: "cascade" }),
  token: text("token").notNull().unique(),
  expiresAt: timestamp("expires_at").notNull(),
  ipAddress: text("ip_address"),
  userAgent: text("user_agent"),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});

// ...account and verification tables similarly...
```

```ts
// lib/auth.ts
import { betterAuth } from "better-auth";
import { drizzleAdapter } from "better-auth/adapters/drizzle";
import { drizzle } from "drizzle-orm/postgres-js";
import postgres from "postgres";
import * as schema from "../db/schema";

const client = postgres(process.env.DATABASE_URL!);
const db = drizzle(client, { schema });

export const auth = betterAuth({
  baseURL: process.env.BETTER_AUTH_URL,
  secret: process.env.BETTER_AUTH_SECRET,
  // Pass the schema so the adapter can map Better Auth's model names to your tables.
  database: drizzleAdapter(db, { provider: "pg", schema }),
  emailAndPassword: { enabled: true },
});
```

```bash
npx @better-auth/cli generate     # emit/refresh Drizzle schema
npx drizzle-kit generate          # create SQL migration
npx drizzle-kit migrate           # apply it
```

### 3.3 The CLI cheat sheet **[B]**

| Command | What it does |
|---|---|
| `npx @better-auth/cli generate` | Reads your `auth` config and **generates the schema** for your ORM (Prisma models / Drizzle schema), or the SQL for direct DB. |
| `npx @better-auth/cli migrate` | **Applies** schema changes directly (built-in/Kysely adapter — i.e. direct DB connections). |
| `npx @better-auth/cli@latest ...` | Pin/refresh the CLI version. Re-run `generate` after adding any plugin. |

> **Always re-run `generate` (and migrate) after adding a plugin or an `additionalField`.** Plugins like `twoFactor`, `passkey`, `organization`, and `jwt` add their own tables/columns. Forgetting this is the #1 "it compiles but fails at runtime with 'no such table/column'" mistake in the entire library.

### 3.4 Mount the handler on plain Node **[I]**

`auth.handler` takes a Web-standard `Request` and returns a `Response`. On a bare Node `http` server you must bridge Node's `req`/`res` to/from the Web API. You rarely hand-roll this in production — but seeing it once makes the "handler is just `Request → Response`" idea concrete.

```ts
// server.node.ts — minimal, framework-free example
import { createServer } from "node:http";
import { auth } from "./lib/auth";

const server = createServer(async (nodeReq, nodeRes) => {
  // Only the /api/auth/* paths belong to Better Auth.
  if (!nodeReq.url?.startsWith("/api/auth")) {
    nodeRes.statusCode = 404;
    nodeRes.end("Not found");
    return;
  }

  // 1) Convert Node IncomingMessage -> Web Request.
  const url = `http://${nodeReq.headers.host}${nodeReq.url}`;
  const body =
    nodeReq.method === "GET" || nodeReq.method === "HEAD"
      ? undefined
      : await readBody(nodeReq); // collect the stream into a string

  const webReq = new Request(url, {
    method: nodeReq.method,
    headers: nodeReq.headers as any,
    body,
  });

  // 2) Let Better Auth handle it — this is the entire integration surface.
  const webRes = await auth.handler(webReq);

  // 3) Convert Web Response -> Node response (copy status, headers, body).
  nodeRes.statusCode = webRes.status;
  webRes.headers.forEach((value, key) => nodeRes.setHeader(key, value));
  nodeRes.end(Buffer.from(await webRes.arrayBuffer()));
});

function readBody(req: import("node:http").IncomingMessage): Promise<string> {
  return new Promise((resolve, reject) => {
    let data = "";
    req.on("data", (c) => (data += c));
    req.on("end", () => resolve(data));
    req.on("error", reject);
  });
}

server.listen(3000, () => console.log("http://localhost:3000"));
```

In practice you use a framework that already speaks the Web API (Hono, Next.js), or a community Node adapter, instead of this bridge.

### 3.5 Mount the handler on Hono **[I]**

Hono is Web-standard natively, so this is clean and is a great way to learn the standalone model. Notice how *little* code mounts the entire auth surface.

```ts
// server.hono.ts
import { Hono } from "hono";
import { auth } from "./lib/auth";

const app = new Hono();

// Mount EVERY method on the catch-all so sign-in/up/callbacks all route through.
// Hono exposes the raw Web Request via c.req.raw — exactly what auth.handler wants.
app.on(["GET", "POST"], "/api/auth/*", (c) => auth.handler(c.req.raw));

// A protected example route: read the session server-side via auth.api.
app.get("/api/me", async (c) => {
  // Pass the incoming headers so Better Auth can read & validate the cookie.
  const session = await auth.api.getSession({ headers: c.req.raw.headers });
  if (!session) return c.json({ error: "Unauthorized" }, 401);
  return c.json({ user: session.user });
});

export default app; // run with Bun/Node per Hono's docs
```

That is a fully working standalone auth server: `/api/auth/*` handles the flows, `auth.api.getSession` protects your own routes. Everything else in this guide is variations on this foundation.

---

## 4. Email & Password Authentication

Email + password is the workhorse credential flow. *What it is:* the user picks a password, the server hashes it and stores the hash in `account` (`providerId = "credential"`), and on sign-in the server hashes the submitted password and compares. *Why the details matter for security:* passwords are the most-attacked credential — reused, phished, brute-forced — so every defense (slow hashing, length limits, verification, rate limiting) compounds.

### 4.1 Configuring it, with secure password handling **[B/I]**

Enable it in the config, optionally wiring email sending for verification and reset. Read the comments — they encode the security decisions.

```ts
// lib/auth.ts
import { betterAuth } from "better-auth";

export const auth = betterAuth({
  baseURL: process.env.BETTER_AUTH_URL,
  secret: process.env.BETTER_AUTH_SECRET,
  database: /* ...adapter... */ undefined as any,

  emailAndPassword: {
    enabled: true,

    // Require users to verify their email before they can sign in.
    // SECURITY: stops abuse of unverified/typo/forged emails and confirms the
    // address is reachable before you trust it for password resets.
    requireEmailVerification: true,

    // Tune password length bounds. A MINIMUM matters far more than a maximum.
    // SECURITY: prefer long passphrases over arcane composition rules — length
    // beats complexity. Don't set max too low (it blocks password managers).
    minPasswordLength: 8,    // 12+ is better for sensitive apps
    maxPasswordLength: 128,

    // Called when a user requests a password reset. YOU send the email.
    // `url` is the full reset link; `token` is the raw token if you build your own.
    sendResetPassword: async ({ user, url, token }) => {
      await sendEmail({
        to: user.email,
        subject: "Reset your password",
        html: `Click to reset: <a href="${url}">${url}</a>`,
      });
    },

    // OPTIONAL: plug a custom password hasher. Default is scrypt (memory-hard, good).
    // SECURITY: only override with a reason. If you do, use a memory-hard algorithm
    // (scrypt/argon2id/bcrypt) — NEVER a plain/fast hash (md5/sha256) for passwords.
    // password: {
    //   hash: async (plain) => await argon2id(plain),
    //   verify: async ({ hash, password }) => await argon2Verify(hash, password),
    // },
  },

  // Email verification config lives under emailVerification.
  emailVerification: {
    sendVerificationEmail: async ({ user, url, token }) => {
      await sendEmail({
        to: user.email,
        subject: "Verify your email",
        html: `Verify: <a href="${url}">${url}</a>`,
      });
    },
    // Auto-send the verification email right after sign-up.
    sendOnSignUp: true,
  },
});

// Pretend mailer — wire to Resend, Postmark, SES, Nodemailer, etc.
async function sendEmail(_: { to: string; subject: string; html: string }) {}
```

**Why scrypt (the default) is the right call.** Password hashing must be *deliberately slow and memory-hard* so that an attacker who steals your database cannot brute-force the hashes cheaply on GPUs. scrypt and argon2id both burn memory as well as CPU, defeating GPU/ASIC parallelism. Better Auth defaults to scrypt; the **`GO_JWT_ARGON2_GUIDE.md`** in this library covers argon2id in depth if you want to align your Go side or migrate. The cardinal rule: **never use a fast general-purpose hash (SHA-256, MD5) for passwords** — those are designed to be fast, which is exactly wrong here.

> **⚡ Version note:** The exact callback signatures (`sendResetPassword`, `sendVerificationEmail`) and whether they nest under `emailAndPassword` vs `emailVerification` have shifted slightly across minor versions. The destructured args you can rely on are `user`, `url`, and `token`. Verify the nesting for your installed version.

> **Security recommendations for credentials:**
> - Turn on `requireEmailVerification` to block unverified accounts from signing in.
> - **Reset tokens must be single-use and short-lived** (Better Auth handles this via the `verification` table) — never email the password itself.
> - **Don't leak account existence.** On "forgot password," return the *same* response whether or not the email exists, so an attacker can't enumerate users. On sign-in, prefer a generic "invalid email or password" over "no such user."
> - **Rate-limit sign-in, sign-up, and reset** hard (§7.5) — this is your primary defense against credential stuffing and brute force.
> - Consider blocking known-breached passwords (e.g. a HaveIBeenPwned k-anonymity check) in your sign-up flow for sensitive apps.

### 4.2 The flows from the client **[B]**

```ts
import { authClient } from "./lib/auth-client";

// --- SIGN UP ---
const { data, error } = await authClient.signUp.email({
  email: "ada@example.com",
  password: "correct-horse-battery-staple", // a passphrase: long, memorable, strong
  name: "Ada Lovelace",
  // image: "https://...",        // optional
  // ...any additionalFields you defined on the user model can go here too.
  callbackURL: "/dashboard",       // where to land after verification (optional)
});
if (error) console.error(error.message); // ALWAYS handle the error path

// --- SIGN IN ---
await authClient.signIn.email({
  email: "ada@example.com",
  password: "correct-horse-battery-staple",
  rememberMe: true,            // persist the session longer (optional)
  callbackURL: "/dashboard",
});

// --- SIGN OUT (clears the session + cookie) ---
await authClient.signOut();

// --- REQUEST PASSWORD RESET (triggers sendResetPassword on the server) ---
await authClient.requestPasswordReset({
  email: "ada@example.com",
  redirectTo: "/reset-password", // page that will read the token from the URL
});
// Note: older versions named this `forgetPassword`. Verify against your version.

// --- COMPLETE PASSWORD RESET (on your /reset-password page) ---
const token = new URLSearchParams(window.location.search).get("token")!;
await authClient.resetPassword({
  newPassword: "a-new-strong-password",
  token,
});

// --- RESEND / SEND VERIFICATION EMAIL ---
await authClient.sendVerificationEmail({
  email: "ada@example.com",
  callbackURL: "/dashboard",
});
```

### 4.3 The same flows server-side (`auth.api`) **[I]**

When you need these in backend code (an admin creating a user, a server-rendered form, a seed script):

```ts
import { auth } from "./lib/auth";

// Sign up a user from the server. Returns the created user/session.
await auth.api.signUpEmail({
  body: { email: "ada@example.com", password: "…", name: "Ada" },
});

// Sign in server-side. asResponse:true returns a Response whose Set-Cookie you
// can forward to the browser (e.g. from a custom route or Server Action).
const res = await auth.api.signInEmail({
  body: { email: "ada@example.com", password: "…" },
  asResponse: true,
});
```

> **⚡ Version note:** Whether email verification is *required* to sign in depends on `requireEmailVerification`. With it on, `signIn.email` for an unverified user returns an error and (depending on version/config) may auto-resend the verification email.

---

## 5. Social / OAuth Providers

OAuth ("Sign in with GitHub/Google/…") lets users authenticate via a provider they already trust, so you never see or store their password. *What it is:* a delegated-authorization protocol where the provider proves the user's identity to you via a signed redirect dance. *Why use it:* lower friction (no new password), often higher trust (the provider has done the verification), and access to provider APIs if you request scopes. *The security upside:* you offload password risk to Google/GitHub; *the trade-off:* you depend on the provider and must handle account linking carefully (§5.3).

### 5.1 Configuring providers **[I]**

Add providers under `socialProviders`. Each needs a client ID/secret from that provider's developer console, plus the redirect/callback URL registered there.

```ts
// lib/auth.ts
export const auth = betterAuth({
  baseURL: process.env.BETTER_AUTH_URL, // CRITICAL: used to build callback URLs
  secret: process.env.BETTER_AUTH_SECRET,
  database: /* adapter */ undefined as any,

  socialProviders: {
    github: {
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!, // SECRET — env only
    },
    google: {
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
      // Optional: request a refresh token, restrict prompts, scope down, etc.
      // prompt: "select_account",
      // scope: ["openid", "email", "profile"],  // request the MINIMUM you need
    },
    // discord, apple, microsoft, gitlab, twitch, spotify, etc. are supported.
  },
});
```

**Register the callback URL** in each provider's console. Better Auth's OAuth callback path follows the pattern:

```
{baseURL}/api/auth/callback/{providerId}
# e.g. http://localhost:3000/api/auth/callback/github
```

> **Security recommendations:**
> - **Request the minimum scopes** you actually need. Over-requesting scares users and widens blast radius if tokens leak.
> - **Client secrets are secrets** — env/secrets-manager only, never in client code or git. Use separate OAuth apps (and secrets) for dev vs prod.
> - Better Auth uses **OAuth state + PKCE** under the hood to prevent CSRF and code-interception in the OAuth flow — don't try to bypass or reimplement this.
> - The `account` table stores the provider's `accessToken`/`refreshToken` — treat them as sensitive (§2.4).

### 5.2 The client-side flow & what happens **[I]**

```ts
import { authClient } from "./lib/auth-client";

// Kicks off the OAuth dance. The browser is redirected to the provider, then
// back to callbackURL after Better Auth creates the session.
await authClient.signIn.social({
  provider: "github",
  callbackURL: "/dashboard",          // where to land on success
  errorCallbackURL: "/login?error=1", // optional, on failure
});
```

**The callback flow, step by step (so you can debug it):**
1. User clicks "Sign in with GitHub" → `signIn.social({ provider: "github" })`.
2. Browser redirects to GitHub's authorize URL (carrying a random `state` and a PKCE challenge — anti-CSRF/anti-interception).
3. User approves; GitHub redirects to `{baseURL}/api/auth/callback/github?code=...&state=...`.
4. Better Auth's handler validates `state`/PKCE, exchanges the `code` for tokens (server-to-server, using the client secret), fetches the user's profile, **finds or creates** a `user`, creates/updates an `account` row (`providerId = "github"`) storing tokens, creates a `session`, sets the session cookie, and redirects to `callbackURL`.

The user ends up with the same cookie-backed session as email+password — OAuth is just a different way to *establish* it.

### 5.3 Account linking — convenient and a security foot-gun **[I/A]**

One `user` can have multiple `account` rows. **Linking** lets a signed-in user connect additional providers (or a credential password) to the *same* user — so "Ada via Google" and "Ada via GitHub" are one account, not two.

```ts
// While signed in, link another provider to the current account.
await authClient.linkSocial({
  provider: "google",
  callbackURL: "/settings/connections",
});

// List the current user's linked accounts.
const accounts = await authClient.listAccounts();

// Unlink a provider.
await authClient.unlinkAccount({ providerId: "google" });
```

**Automatic linking by email** — Better Auth can link a new OAuth login to an *existing* user when the verified email matches. This is convenient but a real security trade-off, and you must understand it.

*The risk:* if you auto-link on email and an attacker can sign in to a *sloppy* provider that does **not** verify emails (or lets them claim any email), they could create a token asserting `victim@example.com` and get silently merged into the victim's account — account takeover. The defense is to only auto-link from providers that *guarantee a verified email*.

```ts
account: {
  accountLinking: {
    enabled: true,
    // Only auto-link when these providers assert a VERIFIED email.
    // SECURITY: include only providers you trust to verify emails (Google, GitHub).
    trustedProviders: ["google", "github"],
  },
},
```

> **Security recommendation:** keep `trustedProviders` to providers with strong email verification. For anything else, require the user to be *already signed in* before linking (explicit linking via `linkSocial`), never silent auto-link. When in doubt, prefer explicit linking — it is the safe default.

> **⚡ Version note:** The shape of `account.accountLinking` and method names (`linkSocial`, `unlinkAccount`, `listAccounts`) are stable in concept; confirm casing against your version's docs.

---

## 6. Sessions & Cookies

Sessions are how a stateless HTTP request "remembers" who you are. This section explains the lifecycle, how to read sessions everywhere, expiry/refresh tuning, and — most importantly for security — the cookie flags that make sessions safe.

### 6.1 How sessions work (the lifecycle) **[B/I]**

On successful auth (any method), Better Auth:
1. Inserts a `session` row (`token`, `userId`, `expiresAt`, `ipAddress`, `userAgent`).
2. Sets a **signed, httpOnly cookie** holding the session token. *Signed* means tampering is detectable (the secret signs it); *httpOnly* means JavaScript can't read it.
3. On each request, reads the cookie, validates the signature, looks up the `session` row, and (if valid and unexpired) loads the user.

This is **database-backed (stateful) session auth** — the server holds authority, so it can **revoke any session instantly** by deleting its row (recall §2.1). Contrast with stateless JWTs (the `jwt()` plugin, §7.4, and Go integration §9), which you use *in addition* for cross-service hops.

### 6.2 Reading the session — client and server **[B/I]**

**Client (React) — reactive hook.** Re-renders when auth state changes; ideal for nav bars and conditional UI.

```ts
import { authClient } from "./lib/auth-client";

function Profile() {
  const { data: session, isPending, error } = authClient.useSession();
  if (isPending) return <p>Loading…</p>;
  if (!session) return <p>Not signed in</p>;
  return <p>Hello {session.user.name}</p>;
}
```

**Client (one-shot, no hook):**

```ts
const { data: session } = await authClient.getSession();
```

**Server (`auth.api.getSession`)** — pass the incoming request headers so Better Auth can read the cookie. *This is the authoritative check* (it hits the DB unless cookie-cache is on):

```ts
import { auth } from "./lib/auth";
// `headers` is a Web Headers object (from a Request, or Next's headers()).
const session = await auth.api.getSession({ headers });
if (session) {
  console.log(session.user.id, session.session.expiresAt);
}
```

The returned object is roughly `{ user, session }` — `user` is the profile, `session` is the session row (token, expiry, etc.).

> **Security recommendation:** *gate sensitive server logic with the server-side `getSession`, never with client state alone.* Client-side `useSession` is for UX (showing/hiding UI); a malicious user can fake client state, but they cannot fake a valid server-validated session.

### 6.3 Session expiry & refresh **[I]**

These knobs trade off security (shorter = less stolen-cookie exposure) against UX (longer = fewer re-logins). Understand each.

```ts
export const auth = betterAuth({
  // ...
  session: {
    expiresIn: 60 * 60 * 24 * 7,   // 7 days (seconds) — absolute max lifetime
    updateAge: 60 * 60 * 24,       // sliding refresh: bump expiry if older than 1 day

    // Cache the session in a short-lived SIGNED cookie to avoid a DB hit on every
    // request. Great for read-heavy apps. TRADE-OFF: revocation lags by maxAge.
    cookieCache: {
      enabled: true,
      maxAge: 5 * 60, // 5 minutes
    },
  },
});
```

- **`expiresIn`** — absolute lifetime. *Shorter is safer* but logs users out more often. 7 days is common; minutes-to-hours for high-security apps.
- **`updateAge`** — sliding refresh: each request past this age bumps the expiry, keeping *active* users logged in while *idle* sessions still expire. Balances "don't annoy active users" with "don't keep dead sessions alive forever."
- **`cookieCache`** — stores a signed snapshot in a cookie so `getSession` can skip the DB for `maxAge`. *Security trade-off:* a revoked session may still appear valid until the cache expires. Keep `maxAge` short (a few minutes), and for truly sensitive operations validate against the DB explicitly.

### 6.4 Cookie configuration & security — the part attackers probe **[I/A]**

Cookies are where session security lives or dies. Better Auth's *defaults are already secure*; this section explains them and the rare cases for overrides.

```ts
export const auth = betterAuth({
  // ...
  advanced: {
    // Force secure cookies even behind proxies / in non-prod testing.
    useSecureCookies: true, // typically auto-on in production over HTTPS

    cookies: {
      // Per-cookie overrides (names auto-derived; you can prefix them).
      // sessionToken: { name: "myapp.session", attributes: { sameSite: "lax" } },
    },

    // Prefix all auth cookie names (avoid collisions on multi-app domains).
    cookiePrefix: "myapp",

    // For sharing cookies across subdomains (e.g. app + api on one root domain):
    crossSubDomainCookies: {
      enabled: true,
      domain: ".example.com", // leading dot => all subdomains
    },
  },
});
```

**The defaults you must understand (and why each one defends you):**
- **`httpOnly: true`** — JavaScript cannot read the session cookie. *Why:* this is your primary defense against **XSS stealing sessions** — even if an attacker injects script, they can't exfiltrate the cookie.
- **`sameSite: "lax"`** — the cookie is sent on top-level navigations but not on cross-site sub-requests. *Why:* this is the primary defense against **CSRF** — a malicious site can't silently make the browser send your session cookie on a state-changing POST. Use `"none"` (which forces `secure`) *only* for genuine cross-site needs, and add explicit CSRF tokens if you do.
- **`secure: true` in production** — the cookie is only sent over HTTPS. *Why:* prevents the session cookie from being sniffed over plaintext HTTP.

> **CSRF in plain terms:** Cross-Site Request Forgery is when a malicious page tricks the *user's browser* into making an authenticated request to *your* site using the cookies it already holds. Because Better Auth's cookie is `sameSite: "lax"`, the browser won't attach it to cross-site state-changing requests — closing the common CSRF vector. Combined with **origin validation** against `trustedOrigins` (§11), this covers the standard attacks. If you ever switch to `sameSite: "none"`, you reopen CSRF and must add anti-CSRF tokens — prefer the bearer/JWT model for cross-site instead (§7.4, §9).

> **⚡ Version note:** Exact paths (`advanced.useSecureCookies`, `advanced.crossSubDomainCookies`, `advanced.cookiePrefix`) are stable but occasionally reorganized; check the "Cookies" / "Advanced options" docs for your version. The *security defaults* (httpOnly + lax + secure-in-prod) are reliable.

### 6.5 Listing & revoking sessions (multi-device) **[I]**

A "your devices / sign out everywhere" feature is both good UX and a security control (users can kill a stolen session themselves).

```ts
// All sessions for the current user (each device/login).
const sessions = await authClient.listSessions();

// Revoke one session by its token (e.g. "log out this device").
await authClient.revokeSession({ token: someSessionToken });

// Revoke every OTHER session (e.g. "sign out everywhere else" after a password change).
await authClient.revokeOtherSessions();

// Revoke ALL sessions including the current one.
await authClient.revokeSessions();
```

> **Security recommendation:** after a **password change** or **suspected compromise**, call `revokeOtherSessions()` so a thief holding an old cookie is kicked out. This is exactly the kind of instant revocation that stateful sessions (vs JWTs) make trivial. Multi-session support (being signed into several accounts at once) is its own plugin — see §7.

---

## 7. Plugins Ecosystem

Plugins are how Better Auth grows from "login" to "complete identity platform." *The why:* the core stays small and secure; advanced features (2FA, passkeys, orgs, JWTs) are opt-in modules that add their own routes, tables, and typed client methods. The pattern is **always the same**, and getting it wrong is the most common source of "missing method"/"missing table" bugs:

1. Add the plugin to the **server** `plugins: [...]`.
2. Add the matching plugin to the **client** `plugins: [...]` (so typed methods appear).
3. Re-run the **CLI** if the plugin adds tables (`generate` + migrate).

```ts
// SERVER — lib/auth.ts
import { betterAuth } from "better-auth";
import { twoFactor } from "better-auth/plugins";
import { magicLink } from "better-auth/plugins";
import { organization } from "better-auth/plugins";
import { admin } from "better-auth/plugins";
import { bearer } from "better-auth/plugins";
import { jwt } from "better-auth/plugins";
import { multiSession } from "better-auth/plugins";
// passkey lives in its own subpath:
import { passkey } from "better-auth/plugins/passkey";

export const auth = betterAuth({
  baseURL: process.env.BETTER_AUTH_URL,
  secret: process.env.BETTER_AUTH_SECRET,
  database: /* adapter */ undefined as any,
  emailAndPassword: { enabled: true },

  plugins: [
    // --- Two-factor (TOTP + optionally OTP/backup codes) ---
    twoFactor({ issuer: "MyApp" }),               // issuer shown in authenticator apps

    // --- Magic link (passwordless email login) ---
    magicLink({
      sendMagicLink: async ({ email, url, token }) => {
        await sendEmail({ to: email, subject: "Your login link", html: `<a href="${url}">Sign in</a>` });
      },
    }),

    // --- Passkeys / WebAuthn (phishing-resistant, passwordless) ---
    passkey({
      rpID: "example.com",            // relying party ID (your registrable domain)
      rpName: "MyApp",
      origin: "https://example.com",  // MUST match the browser origin exactly
    }),

    // --- Organizations / teams (multi-tenant) ---
    organization(),

    // --- Admin (user management, impersonation, ban, roles) ---
    admin(),

    // --- Multi-session (be logged into several accounts at once) ---
    multiSession(),

    // --- Bearer token: accept Authorization: Bearer <token> instead of cookies.
    //     ESSENTIAL for non-browser clients (mobile, CLI, a Go service). ---
    bearer(),

    // --- JWT: exposes signed JWTs + a JWKS endpoint so OTHER services
    //     (e.g. your Go API) can verify tokens offline. See §9. ---
    jwt(),
  ],
});

async function sendEmail(_: { to: string; subject: string; html: string }) {}
```

```ts
// CLIENT — lib/auth-client.ts (mirror the server plugins!)
import { createAuthClient } from "better-auth/react";
import { twoFactorClient } from "better-auth/client/plugins";
import { magicLinkClient } from "better-auth/client/plugins";
import { organizationClient } from "better-auth/client/plugins";
import { adminClient } from "better-auth/client/plugins";
import { multiSessionClient } from "better-auth/client/plugins";
import { passkeyClient } from "better-auth/client/plugins";

export const authClient = createAuthClient({
  baseURL: process.env.NEXT_PUBLIC_BETTER_AUTH_URL,
  plugins: [
    twoFactorClient(),
    magicLinkClient(),
    organizationClient(),
    adminClient(),
    multiSessionClient(),
    passkeyClient(),
    // bearer & jwt are server-side concerns — usually no client plugin needed.
  ],
});
```

> **⚡ Version note:** Import subpaths matter and are the most version-sensitive strings in the whole library. Most plugins export from `better-auth/plugins` (server) and `better-auth/client/plugins` (client); `passkey` historically lives under `better-auth/plugins/passkey`. If an import fails, check the plugin's doc page for the exact subpath.

### 7.1 Two-factor (2FA / TOTP) **[I]**

*What it is:* a second proof of identity beyond the password — a time-based one-time code (TOTP) from an authenticator app (Google Authenticator, Authy, 1Password). *Why it matters:* even if a password is phished or leaked, the attacker still can't log in without the rolling code. It is the single highest-leverage security upgrade you can offer users. *How it works:* the server generates a shared secret, the user's app and your server both derive the same 6-digit code from that secret + current time, and you verify the code.

```ts
// 1) User enables 2FA — server returns a TOTP secret + otpauth URI for a QR code.
//    Requiring the current password here prevents a hijacked session from silently
//    adding 2FA the real user doesn't control.
const { data } = await authClient.twoFactor.enable({ password: "current-password" });
// Render data.totpURI as a QR code; the user scans it in their authenticator app.

// 2) User confirms by entering a code from their app (proves the secret synced).
await authClient.twoFactor.verifyTotp({ code: "123456" });

// 3) On a future sign-in, if 2FA is on, signIn.email returns a "two-factor required"
//    state. Then prompt for the code:
await authClient.twoFactor.verifyTotp({ code: "654321" });

// Backup codes for recovery (store hashed; user keeps them offline):
await authClient.twoFactor.generateBackupCodes();
await authClient.twoFactor.verifyBackupCode({ code: "xxxx-xxxx" });
```

> **Security recommendations:** require the password to enable/disable 2FA; **issue backup codes** (and store them hashed) so a lost phone doesn't lock users out forever; rate-limit code verification to stop brute-forcing the 6-digit space; and consider *requiring* 2FA for admin/privileged roles.

### 7.2 Magic link & email OTP **[I]**

*What they are:* passwordless email flows. **Magic link** emails a one-click sign-in URL; **email OTP** emails a short numeric code the user types. *Why use them:* no password to forget, phish, or reuse — the user's control of their inbox *is* the credential. *The trade-off:* security now hinges on email security, and link/code delivery latency affects UX.

```ts
// Magic link: user clicks the emailed link to sign in (no password).
await authClient.signIn.magicLink({ email: "ada@example.com", callbackURL: "/dashboard" });

// Email OTP (a separate emailOTP() plugin): server emails a 6-digit code.
await authClient.emailOtp.sendVerificationOtp({ email: "ada@example.com", type: "sign-in" });
await authClient.signIn.emailOtp({ email: "ada@example.com", otp: "123456" });
```

> **Security recommendations:** make links/codes **single-use and short-lived** (a few minutes); rate-limit issuance to prevent email-bombing a victim; and don't reveal whether the email exists in the response (same anti-enumeration rule as password reset). Magic links are only as strong as the user's email account — for high-security apps, pair with another factor.

### 7.3 Passkeys / WebAuthn — the phishing-resistant gold standard **[I/A]**

*What it is:* WebAuthn/FIDO2 credentials — a public/private key pair bound to your domain, stored in the device's secure hardware (Touch ID, Face ID, Windows Hello, a YubiKey) or synced via the platform (iCloud/Google passkeys). The private key *never leaves the device*; sign-in is a cryptographic challenge-response. *Why it is the gold standard:* passkeys are **phishing-resistant by design** — the credential is cryptographically bound to your exact origin, so a lookalike phishing site simply cannot use it. There is no shared secret to steal, leak, or reuse. *How it works:* registration creates a key pair (public key sent to your server, private key kept on device); sign-in signs a server challenge with the private key, which the server verifies with the stored public key.

```ts
// Register a passkey for the signed-in user (uses the platform authenticator).
await authClient.passkey.addPasskey();

// Sign in with a passkey (no password — a biometric/PIN prompt appears).
await authClient.signIn.passkey();
```

The server config's `rpID`/`origin` (above) must exactly match your domain — that binding is *the* security property. A mismatch breaks passkeys (and is what makes them phishing-proof).

> **Security recommendation:** offer passkeys as a primary or step-up factor for any serious app — they eliminate the entire class of password and phishing attacks. Let users register **multiple** passkeys (phone + laptop + hardware key) so losing one device doesn't lock them out, and keep a recovery path (e.g. email OTP or backup codes).

### 7.4 Bearer & JWT — the keys to non-cookie clients **[I/A]**

These two plugins unlock **mobile apps, CLIs, and cross-language backends like Go** — anything that can't (or shouldn't) rely on browser cookies.

- **`bearer()`** — lets Better Auth accept `Authorization: Bearer <sessionToken>` *and* return the session token in a response header on sign-in, so a non-cookie client can store it and send it back. The token is still a **DB-backed session** (revocable, recall §2.1). Use this when the client is a native app or another service that holds the session token directly.
- **`jwt()`** — issues **signed JWTs** and publishes a **JWKS** (the *public* keys) endpoint, plus an endpoint to fetch a fresh JWT for the current session. *Why this is the cross-service unlock:* a separate service (your Go API) can verify these JWTs **offline** against the JWKS — no callback to Better Auth per request (full Go treatment in §9). Because JWKS exposes only public keys, the Go service never needs Better Auth's secret.

```ts
// Get a JWT for the current session (to send to your Go API).
// The session itself stays a cookie; you exchange it for a short-lived JWT.
// The jwt() plugin exposes a token endpoint (commonly /api/auth/token) — verify
// the exact route/method against the JWT plugin docs for your version.
const res = await fetch(`${BETTER_AUTH_URL}/api/auth/token`, {
  credentials: "include", // send the session cookie so the server knows who you are
});
const { token } = await res.json(); // the JWT to forward to a backend
```

JWKS is published at a well-known path under the auth routes (commonly `/api/auth/jwks`). See §9 for full Go-side verification.

> **Security recommendations:** keep `jwt()` JWTs **short-lived** (minutes) because they can't be revoked early (§2.1); use **asymmetric** signing (EdDSA/ES256) so resource servers verify with the public key only and never hold a shared secret; and always validate `iss`, `aud` (if set), and `exp` on the receiving side. For mobile apps using `bearer()`, store the token in the platform's secure storage (Keychain/Keystore), never in plaintext.

### 7.5 Rate limiting — your front line against brute force **[I/A]**

*What it is:* limiting how many times a given client (usually by IP) can hit an endpoint in a time window. *Why it is non-negotiable for auth:* without it, attackers can brute-force passwords, stuff leaked credentials, brute-force 2FA codes and OTPs, and email-bomb victims. Better Auth has **built-in rate limiting** (on by default in production) and lets you tighten sensitive endpoints.

```ts
export const auth = betterAuth({
  // ...
  rateLimit: {
    enabled: true,
    window: 60,        // seconds
    max: 100,          // requests per window per IP (global default)
    // Per-path overrides for sensitive endpoints — tighten these aggressively:
    customRules: {
      "/sign-in/email":  { window: 60, max: 5 },
      "/sign-up/email":  { window: 60, max: 3 },
      "/two-factor/verify-totp": { window: 60, max: 5 },
    },
    // storage: "database" | "memory"  (use database/redis in multi-instance deploys)
  },
});
```

> **⚡ Version note:** In-memory rate limiting does **not** work across multiple server instances — each pod counts separately, so an attacker hitting different pods evades the limit. For horizontally-scaled deploys, back rate limiting (and ideally `secondaryStorage`) with a shared store like Redis. Check the `rateLimit` and `secondaryStorage` docs.

> **Security recommendation:** layer rate limiting (per-IP at the app, plus a WAF/edge limiter at the platform) and tighten sign-in/sign-up/reset/2FA endpoints far below the global default. Also consider account-level lockouts/backoff after repeated failures, not just IP-level.

### 7.6 Plugin quick reference **[I]**

| Plugin | Purpose | Adds tables? |
|---|---|---|
| `twoFactor()` | TOTP 2FA, backup codes | Yes (`twoFactor`) |
| `magicLink()` | Passwordless email link | Uses `verification` |
| `emailOTP()` | Email one-time codes | Uses `verification` |
| `passkey()` | WebAuthn / passkeys | Yes (`passkey`) |
| `organization()` | Orgs, teams, members, invites | Yes (`organization`,`member`,`invitation`) |
| `admin()` | User mgmt, ban, impersonate, roles | May add columns (e.g. `role`,`banned`) |
| `multiSession()` | Multiple simultaneous logins | Uses `session` |
| `bearer()` | Accept/return bearer tokens (non-cookie clients) | No |
| `jwt()` | Issue JWTs + JWKS endpoint | Yes (`jwks`) |
| `apiKey()` | API keys for programmatic access | Yes (`apiKey`) |
| `username()` | Username login alongside email | Adds `username` column |

---

## 8. Next.js Integration

This is the most common deployment: Next.js App Router serves the frontend *and* hosts the Better Auth routes in the same app. (See the **Next.js guide** in this library for App Router fundamentals — Server vs Client Components, Server Actions, middleware/Edge runtime.) The integration is remarkably small, which is the point: one route file mounts everything.

### 8.1 Install & env **[B]**

```bash
npm install better-auth
# plus your DB adapter (Prisma/Drizzle/pg) as in §3
```

```bash
# .env.local
BETTER_AUTH_SECRET=long-random-hex
BETTER_AUTH_URL=http://localhost:3000
NEXT_PUBLIC_BETTER_AUTH_URL=http://localhost:3000   # NEXT_PUBLIC_ => exposed to browser
DATABASE_URL=postgres://...
```

> **Security note:** only the `NEXT_PUBLIC_*` var is shipped to the browser (and that's fine — it's just the base URL). `BETTER_AUTH_SECRET` and `DATABASE_URL` are server-only; *never* prefix secrets with `NEXT_PUBLIC_` or they leak into the client bundle.

### 8.2 Server instance (lives in `lib/auth.ts`) **[B]**

```ts
// src/lib/auth.ts
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient();

export const auth = betterAuth({
  baseURL: process.env.BETTER_AUTH_URL,
  secret: process.env.BETTER_AUTH_SECRET,
  database: prismaAdapter(prisma, { provider: "postgresql" }),
  emailAndPassword: { enabled: true, requireEmailVerification: true },
  socialProviders: {
    github: {
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    },
  },
});
```

### 8.3 The catch-all route handler — one file mounts everything **[B/I]**

`toNextJsHandler` adapts `auth.handler` to Next's Route Handler signature. The `[...all]` catch-all means *every* auth route lives behind this one file.

```ts
// src/app/api/auth/[...all]/route.ts
import { auth } from "@/lib/auth";
import { toNextJsHandler } from "better-auth/next-js";

// Returns the correctly-typed GET and POST exports Next expects.
export const { GET, POST } = toNextJsHandler(auth);
```

That is it — `/api/auth/sign-in/email`, `/api/auth/callback/github`, `/api/auth/get-session`, etc. all work now. *Why a catch-all:* Better Auth owns dozens of sub-routes; the catch-all forwards them all to the single handler instead of you declaring each one.

### 8.4 The client (React hooks) **[B]**

```ts
// src/lib/auth-client.ts
import { createAuthClient } from "better-auth/react";

export const authClient = createAuthClient({
  baseURL: process.env.NEXT_PUBLIC_BETTER_AUTH_URL,
});

export const { signIn, signUp, signOut, useSession } = authClient;
```

### 8.5 Login & Sign-up pages (Client Components) **[B/I]**

These are `"use client"` because they manage interactive form state. Note the careful error handling — auth UIs must communicate failures clearly without leaking *why* (anti-enumeration).

```tsx
// src/app/login/page.tsx
"use client";
import { useState } from "react";
import { useRouter } from "next/navigation";
import { authClient } from "@/lib/auth-client";

export default function LoginPage() {
  const router = useRouter();
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setLoading(true);
    setError(null);

    const { error } = await authClient.signIn.email({
      email,
      password,
      callbackURL: "/dashboard",
    });

    setLoading(false);
    if (error) {
      // Generic message — don't reveal whether the email exists (anti-enumeration).
      setError("Invalid email or password");
    } else {
      router.push("/dashboard");
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <input value={email} onChange={(e) => setEmail(e.target.value)} placeholder="Email" type="email" autoComplete="email" />
      <input value={password} onChange={(e) => setPassword(e.target.value)} placeholder="Password" type="password" autoComplete="current-password" />
      {error && <p style={{ color: "red" }}>{error}</p>}
      <button disabled={loading}>{loading ? "…" : "Sign in"}</button>

      {/* Social sign-in */}
      <button type="button" onClick={() => authClient.signIn.social({ provider: "github", callbackURL: "/dashboard" })}>
        Continue with GitHub
      </button>
    </form>
  );
}
```

```tsx
// src/app/signup/page.tsx
"use client";
import { useState } from "react";
import { useRouter } from "next/navigation";
import { authClient } from "@/lib/auth-client";

export default function SignupPage() {
  const router = useRouter();
  const [form, setForm] = useState({ name: "", email: "", password: "" });
  const [error, setError] = useState<string | null>(null);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    const { error } = await authClient.signUp.email({
      name: form.name,
      email: form.email,
      password: form.password,
      callbackURL: "/dashboard",
    });
    if (error) setError(error.message ?? "Sign up failed");
    else router.push("/dashboard"); // with requireEmailVerification, send to a "check your email" page
  }

  return (
    <form onSubmit={handleSubmit}>
      <input placeholder="Name" autoComplete="name" onChange={(e) => setForm({ ...form, name: e.target.value })} />
      <input placeholder="Email" type="email" autoComplete="email" onChange={(e) => setForm({ ...form, email: e.target.value })} />
      <input placeholder="Password" type="password" autoComplete="new-password" onChange={(e) => setForm({ ...form, password: e.target.value })} />
      {error && <p style={{ color: "red" }}>{error}</p>}
      <button>Create account</button>
    </form>
  );
}
```

### 8.6 Server-side session in a Server Component — the secure way to protect a page **[I]**

Server Components can read the session directly via `auth.api.getSession`, passing Next's request headers. *Why this is the right pattern:* the check runs on the server (it can hit the DB and is authoritative), the redirect happens before any protected markup is sent, and you never trust the client. **Do not** gate a server page by fetching session state in a Client Component — that leaks the page first and trusts the browser.

```tsx
// src/app/dashboard/page.tsx  (Server Component — protected)
import { auth } from "@/lib/auth";
import { headers } from "next/headers";
import { redirect } from "next/navigation";

export default async function DashboardPage() {
  // headers() is ASYNC in Next 15+/16 — must be awaited.
  const session = await auth.api.getSession({ headers: await headers() });

  if (!session) {
    redirect("/login"); // not authenticated -> bounce before rendering anything
  }

  return (
    <main>
      <h1>Welcome, {session.user.name}</h1>
      <p>Email: {session.user.email}</p>
      <SignOutButton />
    </main>
  );
}
```

```tsx
// src/components/sign-out-button.tsx
"use client";
import { authClient } from "@/lib/auth-client";
import { useRouter } from "next/navigation";

export function SignOutButton() {
  const router = useRouter();
  return (
    <button
      onClick={async () => {
        await authClient.signOut();
        router.push("/login");
      }}
    >
      Sign out
    </button>
  );
}
```

### 8.7 Protecting Server Actions & Route Handlers **[I]**

Server Actions and Route Handlers are *server* code that mutates data — they **must** re-check the session themselves. *Why:* a UI that hides a button does not stop someone from calling the action directly. The check belongs at the action, every time.

```tsx
// src/app/dashboard/actions.ts
"use server";
import { auth } from "@/lib/auth";
import { headers } from "next/headers";

export async function updateProfile(formData: FormData) {
  // ALWAYS re-validate inside the action. Never assume the caller is authed.
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session) throw new Error("Unauthorized");

  const name = String(formData.get("name"));
  await auth.api.updateUser({ body: { name }, headers: await headers() });
}
```

```ts
// src/app/api/orders/route.ts — a custom Route Handler protected by the session.
import { auth } from "@/lib/auth";
import { headers } from "next/headers";
import { NextResponse } from "next/server";

export async function GET() {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  // ...fetch this user's orders using session.user.id...
  return NextResponse.json({ orders: [] });
}
```

### 8.8 Middleware — the important caveat **[I/A]**

Next.js middleware runs on the **Edge runtime** and *cannot* reliably do a database lookup. *Why this matters:* you cannot fully validate a session in middleware. The idiomatic pattern: in middleware do a **lightweight cookie-presence check** for a fast redirect, and do the **real** validation in the Server Component / Action / Route Handler (Node runtime, can hit the DB).

```ts
// src/middleware.ts
import { NextResponse, type NextRequest } from "next/server";
import { getSessionCookie } from "better-auth/cookies";

export function middleware(request: NextRequest) {
  // OPTIMISTIC check only: is a session cookie present? Does NOT validate it.
  const sessionCookie = getSessionCookie(request);

  if (!sessionCookie) {
    return NextResponse.redirect(new URL("/login", request.url));
  }
  return NextResponse.next();
}

// Only run middleware on protected paths.
export const config = {
  matcher: ["/dashboard/:path*", "/settings/:path*"],
};
```

> **⚡ Critical security caveat:** the cookie's *presence* does NOT mean it is *valid* — it could be expired, revoked, or forged. **Never** treat the middleware check as real authorization. Always re-validate with `auth.api.getSession` in the actual page/handler (Node runtime). Middleware is a fast UX gate, not a security boundary. The helper (`getSessionCookie` from `better-auth/cookies`) is stable; verify the import path.

> Alternative: you *can* validate in middleware by `fetch`-ing the `get-session` HTTP endpoint (forwarding cookies) — but that adds a network hop on every navigation. The cookie-presence pattern is idiomatic; keep the real check at the destination.

### 8.9 Cross-origin Next.js + separate API **[A]**

If your Next.js frontend and Better Auth live on **different origins** (e.g. `app.example.com` calling `auth.example.com`), you must set `trustedOrigins` on the server (§11) and configure cross-subdomain cookies (§6.4) so the cookie is shared. For *totally* different domains, lean on the **bearer/JWT** approach (§7.4, §9) instead of cookies — cross-site cookies require `sameSite: "none"`, which reopens CSRF.

---

## 9. Golang Integration

This is the section most guides skip. Better Auth is TypeScript — you do not run it *in* Go. Instead, **Better Auth is the identity provider / auth service**, and your **Go service is a resource server** that trusts tokens Better Auth issued. This is exactly the OAuth2/OIDC resource-server pattern you'd use with Auth0, Keycloak, or Cognito — Better Auth just happens to be the issuer. The **`GO_JWT_ARGON2_GUIDE.md`** in this library goes deeper on the Go JWT mechanics and password hashing; this section focuses on the Better-Auth-specific wiring.

```
┌──────────────┐      cookie/JWT      ┌──────────────────────┐
│  Next.js     │ ───────────────────▶ │  Better Auth (TS)    │  ← owns users, sessions
│  frontend    │ ◀─── session ─────── │  /api/auth/*         │     in your Postgres
└──────┬───────┘                      └──────────┬───────────┘
       │  Authorization: Bearer <JWT>            │ JWKS (public keys)
       ▼                                         ▼
┌──────────────────────────────────────────────────────────┐
│  Go API (resource server) — VERIFIES tokens, no auth UI   │
└──────────────────────────────────────────────────────────┘
```

There are three realistic architectures. Pick by your revocation/latency needs.

### 9.1 Architecture A (recommended): Go verifies Better Auth JWTs via JWKS **[A]**

This is the cleanest, most scalable design, and the most idiomatic for a polyglot stack. The logic, end to end:

1. Enable the **`jwt()` plugin** on Better Auth. It signs JWTs **asymmetrically** (e.g. EdDSA/ES256) and publishes the **public keys** at a **JWKS** endpoint (commonly `GET {baseURL}/api/auth/jwks`).
2. The frontend obtains a **JWT** for the current session from Better Auth's token endpoint, then sends it to the Go API as `Authorization: Bearer <jwt>`.
3. The **Go API fetches the JWKS once (and caches it), then verifies every incoming JWT offline** — signature, issuer, audience, expiry. **No per-request callback to Better Auth.** Fast and horizontally scalable.

*Why asymmetric signing is the security crux:* with asymmetric keys, Go only ever needs the **public** key (from JWKS) to *verify* — it can never *mint* tokens and never holds Better Auth's secret. Even if the Go service is compromised, the attacker cannot forge valid tokens. This is why you must confirm the `jwt()` plugin uses an asymmetric algorithm and restrict your verifier to those algorithms (defeating the classic "alg=none" / algorithm-confusion attacks).

**Getting a JWT on the frontend** — the session stays a cookie; you exchange it for a short-lived JWT to call Go:

```ts
// Frontend: fetch a JWT to call the Go API.
// The jwt() plugin exposes a token endpoint (commonly /api/auth/token).
// Verify the exact route/method against the JWT plugin docs for your version.
const res = await fetch(`${BETTER_AUTH_URL}/api/auth/token`, {
  credentials: "include", // send the session cookie so the server knows who you are
});
const { token } = await res.json(); // the JWT to forward to Go

await fetch("https://api.example.com/orders", {
  headers: { Authorization: `Bearer ${token}` },
});
```

**Go side — verify the JWT against Better Auth's JWKS.** Using `github.com/golang-jwt/jwt/v5` plus a JWKS fetcher/cache. (Tabs in the Go blocks.)

```go
// auth/jwks.go
package auth

import (
	"context"
	"errors"
	"net/http"
	"strings"
	"time"

	"github.com/golang-jwt/jwt/v5"
	// A small, well-maintained JWKS client that fetches + caches keys and
	// refreshes them on rotation. (github.com/MicahParks/keyfunc/v3)
	"github.com/MicahParks/keyfunc/v3"
)

// JWKS endpoint exposed by Better Auth's jwt() plugin.
const jwksURL = "https://auth.example.com/api/auth/jwks"

// Expected issuer/audience — set to your Better Auth baseURL / configured aud.
// SECURITY: validating iss/aud stops a token minted for a DIFFERENT service or
// issuer from being accepted here (audience-confusion attacks).
const (
	expectedIssuer   = "https://auth.example.com"
	expectedAudience = "https://api.example.com" // only if you configure an audience
)

// keyfn holds the cached JWKS and auto-refreshes on key rotation.
var keyfn keyfunc.Keyfunc

// InitJWKS loads the JWKS once at startup and keeps it fresh in the background.
func InitJWKS(ctx context.Context) error {
	k, err := keyfunc.NewDefaultCtx(ctx, []string{jwksURL}) // fetches + auto-refreshes
	if err != nil {
		return err
	}
	keyfn = k
	return nil
}

// Claims is what we expect inside the Better Auth JWT.
// Better Auth puts the user id in `sub`; extra claims depend on your jwt() config.
type Claims struct {
	Email string `json:"email"`
	Name  string `json:"name"`
	Role  string `json:"role"`
	jwt.RegisteredClaims
}

// VerifyToken parses and FULLY validates the JWT (signature + standard claims).
func VerifyToken(tokenString string) (*Claims, error) {
	claims := &Claims{}

	token, err := jwt.ParseWithClaims(
		tokenString,
		claims,
		keyfn.Keyfunc, // resolves the right public key by the token's `kid` header
		// SECURITY: restrict to the asymmetric algs Better Auth actually uses.
		// This is the defense against alg=none and algorithm-confusion attacks.
		jwt.WithValidMethods([]string{"EdDSA", "ES256", "RS256"}),
		jwt.WithIssuer(expectedIssuer),
		jwt.WithAudience(expectedAudience), // omit if you don't set an audience
		jwt.WithExpirationRequired(),       // reject tokens without exp
		jwt.WithLeeway(30*time.Second),     // small clock-skew tolerance
	)
	if err != nil {
		return nil, err
	}
	if !token.Valid {
		return nil, errors.New("invalid token")
	}
	return claims, nil
}

// BearerFromHeader extracts the raw token from an Authorization header.
func BearerFromHeader(r *http.Request) (string, error) {
	h := r.Header.Get("Authorization")
	if h == "" {
		return "", errors.New("missing Authorization header")
	}
	parts := strings.SplitN(h, " ", 2)
	if len(parts) != 2 || !strings.EqualFold(parts[0], "Bearer") {
		return "", errors.New("malformed Authorization header")
	}
	return parts[1], nil
}
```

```go
// auth/middleware.go — net/http middleware that gates protected routes.
package auth

import (
	"context"
	"net/http"
)

// ctxKey avoids collisions in context values (always use an unexported type).
type ctxKey string

const userCtxKey ctxKey = "user"

// AuthUser is the lightweight identity we attach to the request context.
type AuthUser struct {
	ID    string
	Email string
	Name  string
	Role  string
}

// RequireAuth wraps a handler, rejecting requests without a valid Better Auth JWT.
func RequireAuth(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		raw, err := BearerFromHeader(r)
		if err != nil {
			http.Error(w, "unauthorized", http.StatusUnauthorized)
			return
		}

		claims, err := VerifyToken(raw)
		if err != nil {
			// Don't leak WHY (expired vs malformed) to the caller — log it server-side.
			http.Error(w, "unauthorized", http.StatusUnauthorized)
			return
		}

		user := AuthUser{
			ID:    claims.Subject, // Better Auth puts the user id in `sub`
			Email: claims.Email,
			Name:  claims.Name,
			Role:  claims.Role,
		}

		// Attach the user to the context for downstream handlers.
		ctx := context.WithValue(r.Context(), userCtxKey, user)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

// UserFromContext retrieves the authenticated user inside a handler.
func UserFromContext(ctx context.Context) (AuthUser, bool) {
	u, ok := ctx.Value(userCtxKey).(AuthUser)
	return u, ok
}
```

```go
// main.go — wiring it together.
package main

import (
	"context"
	"encoding/json"
	"log"
	"net/http"

	"yourapp/auth"
)

func main() {
	ctx := context.Background()
	if err := auth.InitJWKS(ctx); err != nil { // load Better Auth's public keys
		log.Fatalf("failed to init JWKS: %v", err)
	}

	mux := http.NewServeMux()

	// Public route.
	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("ok"))
	})

	// Protected route — only reachable with a valid Better Auth JWT.
	protected := auth.RequireAuth(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		user, _ := auth.UserFromContext(r.Context())
		json.NewEncoder(w).Encode(map[string]any{
			"message": "hello " + user.Email,
			"userId":  user.ID,
			"role":    user.Role,
		})
	}))
	mux.Handle("/api/me", protected)

	log.Println("Go API on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

**Why this is idiomatic:** the Go service behaves like any OAuth2 resource server validating a JWT against a JWKS. Asymmetric keys mean Go never needs Better Auth's secret — only the *public* keys. See `GO_JWT_ARGON2_GUIDE.md` for deeper coverage of `golang-jwt`, claim validation, and key handling.

> **⚡ Version note:** Verify (1) the JWKS path — commonly `/api/auth/jwks`; (2) the token-fetch route — commonly `/api/auth/token`; (3) the signing algorithm the `jwt()` plugin uses (often EdDSA) so your `WithValidMethods` list matches; (4) whether you configured an `audience` (if not, drop `WithAudience`). These come from the JWT plugin's config/docs.

### 9.2 Architecture B: Go calls Better Auth's session-validation endpoint **[A]**

Simpler to reason about, and it preserves **instant revocation** (the DB is consulted every time) — at the cost of a network hop per request (mitigate with a short cache). Here Go verifies nothing itself: it forwards the caller's cookie/bearer token to Better Auth's `get-session` endpoint and trusts the answer.

```go
// auth/introspect.go — ask Better Auth "who is this?" by forwarding credentials.
package auth

import (
	"encoding/json"
	"errors"
	"net/http"
	"time"
)

const getSessionURL = "https://auth.example.com/api/auth/get-session"

var httpClient = &http.Client{Timeout: 5 * time.Second} // ALWAYS set a timeout

// SessionResponse mirrors Better Auth's get-session shape: { user, session } or null.
type SessionResponse struct {
	User *struct {
		ID    string `json:"id"`
		Email string `json:"email"`
		Name  string `json:"name"`
		Role  string `json:"role"`
	} `json:"user"`
	Session *struct {
		ID        string    `json:"id"`
		ExpiresAt time.Time `json:"expiresAt"`
	} `json:"session"`
}

// ValidateViaBetterAuth forwards the incoming request's Cookie AND Authorization
// headers to Better Auth and returns the resolved user, or an error.
func ValidateViaBetterAuth(incoming *http.Request) (*SessionResponse, error) {
	req, err := http.NewRequest(http.MethodGet, getSessionURL, nil)
	if err != nil {
		return nil, err
	}

	// Forward whatever the client sent so Better Auth can resolve the session.
	if c := incoming.Header.Get("Cookie"); c != "" {
		req.Header.Set("Cookie", c) // cookie-based sessions (browser/Next.js)
	}
	if a := incoming.Header.Get("Authorization"); a != "" {
		req.Header.Set("Authorization", a) // bearer() plugin token (mobile/api)
	}

	resp, err := httpClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return nil, errors.New("session validation failed")
	}

	var out SessionResponse
	if err := json.NewDecoder(resp.Body).Decode(&out); err != nil {
		return nil, err
	}
	if out.User == nil {
		return nil, errors.New("no active session")
	}
	return &out, nil
}
```

Use it in middleware exactly like §9.1 (swap `VerifyToken` for `ValidateViaBetterAuth`). **Add a short-lived cache** (keyed by the token, TTL 30–60s) so you're not hammering Better Auth on every request. This requires the **`bearer()`** plugin if the Go client sends a token instead of a cookie.

**Trade-offs vs Architecture A:**

| | A: JWT + JWKS | B: introspect get-session |
|---|---|---|
| Per-request network hop | No (offline verify) | Yes (mitigate w/ cache) |
| Instant revocation | No (until JWT exp) | **Yes (DB-backed)** |
| Scales horizontally | **Excellent** | Good w/ caching |
| Complexity in Go | Moderate (JWT libs) | Low (one HTTP call) |
| Best for | High-throughput APIs | Internal/low-traffic, needs instant revoke |

Pick **A** for performance/scale; pick **B** when instant revocation matters more than latency.

### 9.3 Architecture C: shared cookie (same root domain) **[A]**

If the Next.js frontend (`app.example.com`) and the Go API (`api.example.com`) share a **root domain**, the browser can send Better Auth's session cookie to both.

- On Better Auth, enable cross-subdomain cookies with `domain: ".example.com"` (§6.4) and add `api.example.com` to `trustedOrigins`.
- The Go API reads the session cookie and validates it — **but Go can't cleanly verify Better Auth's signed cookie or look up the DB session by hand.** The cookie is a *signed session token*, and re-implementing the signing scheme in Go is fragile and couples you to internals. So in practice, even with a shared cookie, Go still **forwards the cookie to `get-session`** (Architecture B) to resolve it. "Shared cookie" just means the browser sends it automatically; the *validation* is still B.

```go
// With a shared cookie, the browser already sends Cookie to api.example.com.
// Go just forwards it (Architecture B) — no JWT exchange needed on the frontend.
session, err := auth.ValidateViaBetterAuth(r)
```

> **Honest, security-minded take:** do not re-implement Better Auth's cookie signing or DB session reads in Go. Use **JWKS verification (A)** for offline checks, or **forward to get-session (B)** when you want the DB to be the authority. Shared cookies (C) are a convenience for *transport*, not a separate validation method.

### 9.4 CORS for cross-origin frontend → Go **[A]**

If the browser calls the Go API on another origin *with credentials*, configure CORS on Go and `trustedOrigins` on Better Auth. *Security note:* with credentials, you **cannot** use a wildcard `*` origin — you must echo an explicit, allow-listed origin, or the browser blocks the request.

```go
func withCORS(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// SECURITY: explicit origin (never "*" together with credentials).
		w.Header().Set("Access-Control-Allow-Origin", "https://app.example.com")
		w.Header().Set("Access-Control-Allow-Credentials", "true") // needed for cookies
		w.Header().Set("Access-Control-Allow-Headers", "Authorization, Content-Type")
		w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
		if r.Method == http.MethodOptions {
			w.WriteHeader(http.StatusNoContent)
			return
		}
		next.ServeHTTP(w, r)
	})
}
```

---

## 10. Authorization

Authentication ("who are you?") is solved by sessions/JWTs. **Authorization** ("what are you allowed to do?") is a separate layer built on roles and the org/admin plugins. *Why keep them separate:* identity rarely changes; permissions change constantly (promotions, role grants, per-resource rules). Conflating them produces brittle, hard-to-audit code.

### 10.1 Simple roles via additionalFields **[I]**

For app-wide roles (user/admin), add a `role` field to the user model. *The key security detail* is `input: false` — it stops a user from setting their own role at sign-up.

```ts
export const auth = betterAuth({
  // ...
  user: {
    additionalFields: {
      role: {
        type: "string",
        defaultValue: "user",
        input: false, // SECURITY: users CANNOT set their own role at sign-up
      },
    },
  },
});
```

Then gate server-side (never client-side alone):

```ts
const session = await auth.api.getSession({ headers });
if (session?.user.role !== "admin") throw new Error("Forbidden");
```

### 10.2 The admin plugin **[I]**

`admin()` formalizes app-level roles plus user-management superpowers (ban, impersonate, list).

```ts
import { admin } from "better-auth/plugins";

plugins: [
  admin({
    // defaultRole: "user",
    // adminRoles: ["admin"],   // which roles count as "admins"
  }),
];
```

Client methods (the caller must already be an admin — enforced server-side):

```ts
await authClient.admin.listUsers({ query: { limit: 50 } });
await authClient.admin.setRole({ userId, role: "admin" });
await authClient.admin.banUser({ userId, banReason: "spam", banExpiresIn: 60 * 60 * 24 });
await authClient.admin.unbanUser({ userId });
await authClient.admin.impersonateUser({ userId }); // act as another user
await authClient.admin.stopImpersonating();
```

> **Security recommendations:** impersonation is powerful and dangerous — **audit-log every impersonation** (who, whom, when, why), time-box it, and consider requiring 2FA to start it. Banning should revoke the target's sessions (§6.5) so a banned user with a live cookie is kicked immediately.

### 10.3 Organizations & teams (multi-tenant RBAC) **[I/A]**

The `organization()` plugin adds orgs, members, roles (owner/admin/member by default), and invitations — the backbone of B2B/multi-tenant apps where permissions are scoped *per organization*.

```ts
import { organization } from "better-auth/plugins";
plugins: [organization()];
```

```ts
// Create an org (the creator becomes owner).
await authClient.organization.create({ name: "Acme Inc", slug: "acme" });

// Invite a member with a role.
await authClient.organization.inviteMember({ organizationId, email: "bob@acme.com", role: "member" });

// Accept an invitation (invitee, from the emailed link).
await authClient.organization.acceptInvitation({ invitationId });

// Set the "active" org for the current session (scopes subsequent requests).
await authClient.organization.setActive({ organizationId });

// Manage membership.
await authClient.organization.updateMemberRole({ organizationId, memberId, role: "admin" });
await authClient.organization.removeMember({ organizationId, memberIdOrEmail: "bob@acme.com" });
```

> **Security recommendation:** **always scope queries by the active org and verify membership** on the server. The classic multi-tenant bug is an IDOR — a member of Org A reading Org B's data by guessing IDs. Every data fetch must confirm the user belongs to (and has rights in) the org owning that resource.

### 10.4 Access control (fine-grained permissions) **[A]**

Beyond coarse roles, Better Auth ships an **access-control** helper to define permissions per resource and check them. You declare a statement of resources→actions, build roles from it, and check `hasPermission`.

```ts
// access.ts
import { createAccessControl } from "better-auth/plugins/access";

// Define what actions exist per resource.
const statement = {
  project: ["create", "read", "update", "delete"],
  billing: ["read", "manage"],
} as const;

export const ac = createAccessControl(statement);

// Build roles from the statement (least privilege: grant only what's needed).
export const member = ac.newRole({ project: ["read"] });
export const admin  = ac.newRole({ project: ["create", "read", "update", "delete"], billing: ["read", "manage"] });
```

Wire these roles into the `organization()` (or `admin()`) plugin's `roles`/`ac` options, then check:

```ts
const { data } = await authClient.organization.hasPermission({
  permissions: { project: ["delete"] },
});
if (!data?.success) throw new Error("Forbidden");
```

> **⚡ Version note:** The access-control API (`createAccessControl`, `newRole`, `hasPermission`, and how `ac`/`roles` plug into `organization()`/`admin()`) is powerful but version-sensitive. Confirm the import path (`better-auth/plugins/access`) and option names against the "Access Control" docs for your version.

### 10.5 Authorization in Go **[A]**

The Go resource server enforces authorization from the JWT/session claims. Put `role` (and org/permission info) into the JWT (configure the `jwt()` plugin's payload) so Go can decide *without* extra calls. (Tabs in Go.)

```go
func RequireRole(role string, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		user, ok := auth.UserFromContext(r.Context())
		if !ok || user.Role != role {
			http.Error(w, "forbidden", http.StatusForbidden)
			return
		}
		next.ServeHTTP(w, r)
	})
}
// Usage: mux.Handle("/api/admin", auth.RequireAuth(RequireRole("admin", handler)))
```

> **Security recommendation:** put only the *minimum* identity/role data in the JWT, and treat its claims as authoritative *only after* full signature/`iss`/`aud`/`exp` validation (§9.1). For permissions that change second-to-second (e.g. a just-revoked role), remember JWTs lag until expiry — use short lifetimes or Architecture B for those flows.

---

## 11. Security Best Practices

A consolidated checklist. Because this is auth, treat every row as a requirement, not a suggestion. Most of these are explained in depth in the relevant section above; here they live in one place for audit.

| Area | Do this |
|---|---|
| **Secret** | `BETTER_AUTH_SECRET` = 32+ random bytes from a CSPRNG. Never commit. Use a secrets manager in prod, different per environment. Rotating it invalidates all signed cookies/tokens (plan for it / use it during incidents). |
| **HTTPS** | Always serve auth over HTTPS in production so `secure` cookies work and tokens aren't sniffable. |
| **Cookies** | Keep defaults: `httpOnly` (XSS-resistant), `sameSite: "lax"` (CSRF-resistant), `secure` in prod. Only loosen for genuine cross-site needs (and then add CSRF tokens). |
| **CSRF** | Better Auth uses `sameSite` cookies + origin checks. Keep `trustedOrigins` tight. For cross-site cookie auth, understand and add explicit CSRF protections — or prefer bearer/JWT. |
| **trustedOrigins** | Explicitly list every origin allowed to make auth requests / receive redirects. No wildcards in prod. |
| **Rate limiting** | Keep enabled; tighten sign-in/sign-up/reset/2FA endpoints far below the default. Use shared storage (Redis) across instances (§7.5). |
| **Password hashing** | Default scrypt (memory-hard) — don't downgrade. If overriding, use argon2id/scrypt/bcrypt, never a fast hash. (See `GO_JWT_ARGON2_GUIDE.md`.) |
| **Email verification** | Turn on `requireEmailVerification` for credential sign-ups to stop fake/abused accounts. |
| **Account linking** | Only auto-link via `trustedProviders` that assert verified emails (§5.3); otherwise require explicit linking. |
| **2FA / passkeys** | Offer 2FA (require it for admins); offer passkeys (phishing-resistant). Issue + hash backup codes. |
| **Anti-enumeration** | Use generic errors on sign-in and identical responses on "forgot password," so attackers can't discover which emails exist. |
| **JWT (Go)** | Use **asymmetric** signing so resource servers verify with public keys only. Always check `iss`, `aud` (if set), `exp`; restrict `alg`. Keep lifetimes short. |
| **Session revocation** | JWTs (Arch A) can't be revoked before expiry — keep them short; use introspection (Arch B) when instant revoke matters. Revoke other sessions after password change. |
| **Token storage (clients)** | Bearer tokens on mobile → secure storage (Keychain/Keystore). Never put session tokens in `localStorage` if a cookie works. |
| **Logging** | Never log secrets, passwords, raw tokens, or OAuth access/refresh tokens. Audit-log sensitive actions (impersonation, role changes, bans). |

```ts
export const auth = betterAuth({
  baseURL: "https://app.example.com",
  secret: process.env.BETTER_AUTH_SECRET, // from a secrets manager
  // Explicit allow-list of origins that may initiate auth flows / be redirected to.
  trustedOrigins: [
    "https://app.example.com",
    "https://api.example.com", // your Go API origin, if it makes auth calls
  ],
  rateLimit: { enabled: true },
  emailAndPassword: { enabled: true, requireEmailVerification: true },
  advanced: { useSecureCookies: true },
  // Custom password hasher (optional — default scrypt is fine):
  // emailAndPassword: { password: { hash: async (p) => ..., verify: async ({ hash, password }) => ... } }
});
```

> **Defense in depth:** no single control is sufficient. Secure cookies *and* `trustedOrigins` *and* rate limiting *and* email verification *and* 2FA *and* short-lived JWTs together produce a posture where one failure doesn't mean game over. Re-read this table before every production launch.

---

## 12. Database Schema Reference

### 12.1 Core tables (full reference) **[I]**

**`user`**

| Column | Type | Notes |
|---|---|---|
| `id` | string (PK) | cuid/uuid |
| `name` | string | display name |
| `email` | string (unique) | |
| `emailVerified` | boolean | set true after verification |
| `image` | string? | avatar URL |
| `createdAt` | datetime | |
| `updatedAt` | datetime | |

**`session`**

| Column | Type | Notes |
|---|---|---|
| `id` | string (PK) | |
| `userId` | string (FK→user) | cascade on delete |
| `token` | string (unique) | the value behind the cookie |
| `expiresAt` | datetime | |
| `ipAddress` | string? | captured at creation (useful for the devices page) |
| `userAgent` | string? | captured at creation |
| `createdAt`/`updatedAt` | datetime | |

**`account`** (credentials + linked OAuth providers)

| Column | Type | Notes |
|---|---|---|
| `id` | string (PK) | |
| `userId` | string (FK→user) | |
| `accountId` | string | provider's user id (or the user id for credentials) |
| `providerId` | string | `"credential"`, `"github"`, `"google"`, … |
| `accessToken` | string? | OAuth (sensitive) |
| `refreshToken` | string? | OAuth (sensitive) |
| `idToken` | string? | OIDC |
| `accessTokenExpiresAt` | datetime? | |
| `refreshTokenExpiresAt` | datetime? | |
| `scope` | string? | granted OAuth scopes |
| `password` | string? | **hashed** password (credential provider only) |
| `createdAt`/`updatedAt` | datetime | |

**`verification`** (generic short-lived tokens)

| Column | Type | Notes |
|---|---|---|
| `id` | string (PK) | |
| `identifier` | string | what the value is for (e.g. email + purpose) |
| `value` | string | the token/code (often hashed) |
| `expiresAt` | datetime | |
| `createdAt`/`updatedAt` | datetime | |

Plugin tables you'll commonly also see: `twoFactor`, `passkey`, `jwks` (the JWT plugin's signing keys), `organization`, `member`, `invitation`, `apiKey`.

> **Security note:** the `jwks` table holds the *private* signing keys for the `jwt()` plugin — protect it like the secret. The `account` table holds password hashes and OAuth tokens. Apply least-privilege DB access and consider encryption at rest for these columns.

### 12.2 Extending the user model (`additionalFields`) **[I]**

Add custom columns; they flow into types and into sign-up input (unless `input: false`). *Why `input: false` matters for security:* it prevents the field from being set by the client at sign-up — essential for `role`, `isPro`, or anything privilege-related.

```ts
export const auth = betterAuth({
  // ...
  user: {
    additionalFields: {
      role:  { type: "string",  defaultValue: "user", input: false }, // SECURITY: not client-settable
      bio:   { type: "string",  required: false },
      isPro: { type: "boolean", defaultValue: false, input: false },  // SECURITY: not client-settable
      age:   { type: "number",  required: false },
    },
  },
});
```

After adding fields, **re-run the CLI** (`generate` + migrate). The new fields are now typed on `session.user`:

```ts
const session = await auth.api.getSession({ headers });
session?.user.bio;   // typed!
session?.user.isPro; // typed!
```

You can also customize table/column names and model mapping via adapter options or `user.modelName` / field mapping — useful when retrofitting Better Auth onto an existing schema. (Verify the exact mapping option names for your version.)

---

## 13. Testing & Local Development

### 13.1 Local dev tips **[I]**

- **Email in dev:** don't wire a real SMTP provider. Log the verification/reset URL to the console, or use a local catcher (Mailpit/Mailhog/Inbucket via Docker). Clicking the logged link still works.

```ts
emailVerification: {
  sendVerificationEmail: async ({ user, url }) => {
    if (process.env.NODE_ENV !== "production") {
      console.log(`[dev] Verify ${user.email}: ${url}`); // click it from the terminal
      return;
    }
    await realMailer.send(/* ... */);
  },
};
```

- **OAuth in dev:** register a *separate* dev OAuth app per provider with `http://localhost:3000/api/auth/callback/<provider>` as the redirect. Keep dev/prod client secrets separate.
- **DB:** run Postgres in Docker; point `DATABASE_URL` at it. Re-run `generate`/`migrate` whenever you add a plugin or `additionalFields`.
- **Disable rate limiting in tests** (or set a huge `max`), or it'll flake your suite on repeated sign-ins.
- **Seeded users:** create test users via `auth.api.signUpEmail` in a seed script so password hashing matches exactly.

### 13.2 Integration testing **[I]**

Test against the real `auth` instance using an in-memory SQLite DB — fast and isolated.

```ts
// auth.test.ts (Vitest)
import { describe, it, expect, beforeAll } from "vitest";
import { betterAuth } from "better-auth";
import Database from "better-sqlite3";

const auth = betterAuth({
  secret: "test-secret-test-secret-test-secret",
  baseURL: "http://localhost:3000",
  database: new Database(":memory:"), // fast, isolated per run
  emailAndPassword: { enabled: true },
});

beforeAll(async () => {
  // Apply schema to the in-memory DB. In a real setup, run the CLI migrate
  // against a temp DB, or use the adapter's schema-push.
});

describe("email/password", () => {
  it("signs a user up and reads the session", async () => {
    const res = await auth.api.signUpEmail({
      body: { email: "t@example.com", password: "password123", name: "T" },
      asResponse: true,
    });
    expect(res.status).toBe(200);

    // Forward the Set-Cookie to getSession to confirm the session resolves.
    const cookie = res.headers.get("set-cookie")!;
    const session = await auth.api.getSession({ headers: new Headers({ cookie }) });
    expect(session?.user.email).toBe("t@example.com");
  });
});
```

### 13.3 Testing the Go side **[I]**

- Unit-test `VerifyToken` by signing test JWTs with a throwaway key pair and pointing the verifier at a local JWKS (or inject a `keyfunc` that returns your test public key). **Test the negative paths too:** expired token, wrong issuer, wrong audience, `alg=none`, tampered signature — these are your security regression tests.
- Integration-test `ValidateViaBetterAuth` against a running Better Auth dev instance, or stub the `get-session` endpoint with an `httptest.Server`.

---

## 14. Gotchas & Best Practices

- **Re-run the CLI after every plugin or `additionalFields` change.** Plugins add tables/columns; forgetting causes runtime "no such column/table" errors. The single most common pitfall.
- **Client plugins must mirror server plugins.** A server-side `twoFactor()` without `twoFactorClient()` means no typed `authClient.twoFactor.*`. Mismatches = missing methods.
- **Import subpaths are exact and version-sensitive.** `better-auth/plugins`, `better-auth/client/plugins`, `better-auth/plugins/passkey`, `better-auth/adapters/prisma`, `better-auth/next-js`, `better-auth/cookies`, `better-auth/plugins/access`. A wrong subpath is the most common "module not found."
- **`baseURL` must be correct.** It builds OAuth callbacks and redirects. A wrong `baseURL` → broken OAuth and "untrusted origin" errors. Match it (and provider redirect URLs) to where the app actually runs.
- **Next.js middleware can't validate sessions (Edge).** Use the cookie-presence optimistic check for redirects; do real validation in the page/handler (§8.8). Middleware is *not* a security boundary.
- **`headers()` is async in Next 15+/16.** `await headers()` before passing to `auth.api.getSession`.
- **Cookie cache lags revocation.** With `session.cookieCache` on, a revoked/expired session may appear valid until the cache TTL passes. Keep TTL short; for sensitive ops, validate against the DB.
- **JWTs (Arch A) can't be revoked early.** Keep them short-lived; use introspection (Arch B) for instant kill-switch.
- **scrypt is the default password hash.** Good and memory-hard. Don't replace without reason; if migrating from bcrypt, plan a re-hash-on-login strategy.
- **Account linking by email is a foot-gun if mis-scoped.** Only `trustedProviders` with verified emails; otherwise explicit linking only.
- **Rate-limit storage in multi-instance deploys.** In-memory won't share across pods; use Redis/DB-backed storage, or attackers evade limits by hitting different pods.
- **Don't reimplement Better Auth's cookie/session in Go.** Use JWKS (A) or get-session introspection (B). Re-implementing signing couples you to internals that change.
- **`trustedOrigins` must include every front-end/back-end origin** that talks to auth — otherwise CSRF/origin checks reject legitimate requests (and wildcards reopen CSRF).
- **Never put secrets in `NEXT_PUBLIC_*`.** Those variables ship to the browser bundle.
- **Verify exact method/route names against your installed version.** This guide is precise where the API is stable and flags the spots that drift (token endpoint, JWKS path, some plugin options, reset-password naming). Trust your editor's autocomplete and the docs over memory.

| Gotcha | Symptom | Fix |
|---|---|---|
| Forgot CLI after adding plugin | runtime "no such table/column" | `generate` + migrate |
| Server plugin, no client plugin | `authClient.x` undefined | mirror it on the client |
| Wrong import subpath | "module not found" | check plugin doc page |
| Wrong `baseURL` | OAuth fails / untrusted origin | match it to the real URL |
| Trusting middleware cookie check | unauthorized access to data | re-validate in page/handler |
| `headers()` not awaited | type/runtime error (Next 15+) | `await headers()` |
| Long-lived JWT | can't revoke a stolen token | short `exp` + Arch B fallback |
| `*` CORS with credentials | browser blocks the request | echo an explicit origin |
| Secret in client bundle | full compromise | server-only env, never `NEXT_PUBLIC_` |

---

## 15. Study Path & Build-to-Learn Projects

### Suggested learning order

Auth is best learned by building each layer once, in order — each step's mental model unlocks the next. Write code at every step; reading alone won't stick.

1. **Standalone email+password (Hono or direct pg).** Get `betterAuth()` + adapter + CLI migrate working. Sign up, sign in, read a session with `auth.api.getSession`. *Goal: understand the four tables and the handler-is-`Request→Response` model.* (§2–§4)
2. **Add OAuth (GitHub).** Register a dev app, wire `socialProviders`, complete the callback flow, inspect the `account` rows. *Goal: understand provider linking and the OAuth dance.* (§5)
3. **Next.js App Router.** Mount `toNextJsHandler`, build login/signup pages, protect a dashboard Server Component, add the optimistic middleware. *Goal: server vs client session access and the middleware caveat.* (§8)
4. **Sessions deep-dive.** Configure `expiresIn`/`updateAge`/`cookieCache`; build a "sessions/devices" page with `listSessions` + `revokeSession`. *Goal: stateful sessions, revocation, and security trade-offs.* (§6)
5. **Plugins.** Add `twoFactor` (with a QR code), `magicLink`, passkeys, and `organization`. *Goal: the plugin mirror pattern + schema regen + phishing-resistant auth.* (§7)
6. **JWT + JWKS for Go.** Enable `jwt()` + `bearer()`; fetch a token on the frontend; build the Go resource server that verifies it offline (Architecture A). *Goal: polyglot, offline verification — the sessions-vs-JWTs distinction made real.* (§9, and `GO_JWT_ARGON2_GUIDE.md`)
7. **Authorization.** Add roles via `admin()` and orgs via `organization()`; enforce RBAC in both Next.js and Go. *Goal: real multi-tenant authz without IDOR holes.* (§10)
8. **Harden.** `trustedOrigins`, rate limits, secure cookies, required email verification, short-lived JWTs, anti-enumeration. *Goal: production posture (§11).*

### Build-to-learn projects

| # | Project | What it teaches |
|---|---|---|
| 1 | **Auth playground (standalone Hono)** | Core instance, adapter, CLI, `auth.api.getSession`, the four tables |
| 2 | **Next.js SaaS starter** | `toNextJsHandler`, `useSession`, protected Server Components, the middleware caveat, OAuth |
| 3 | **Devices & security center** | `listSessions`/`revokeSession`, 2FA enable/verify, passkeys, password reset & email verification |
| 4 | **B2B multi-tenant app** | `organization()` (orgs/members/invites), access-control roles, admin plugin, RBAC, anti-IDOR |
| 5 | **★ Full-stack polyglot app: Next.js front + Go API, both authed by Better Auth** | The capstone — `jwt()`/`bearer()`, JWKS verification in Go (Arch A) *and* a get-session introspection fallback (Arch B), CORS, shared-domain cookies (Arch C), role enforcement on both sides |

The capstone (#5) is the real test of this guide: a Next.js app where users sign up / log in via Better Auth, the frontend fetches a short-lived JWT and calls a separate Go microservice, and the Go service verifies that JWT against Better Auth's JWKS and enforces roles — no auth logic duplicated, one source of truth, two languages. It exercises every core idea: sessions vs JWTs, secure cookies, plugins, offline verification, and authorization across a polyglot stack.

### Cross-references in this library

- **Next.js guide** — App Router, Server/Client Components, Server Actions, middleware/Edge runtime (the framework Better Auth mounts into in §8).
- **PostgreSQL / Prisma guides** — the database and ORM underneath the adapters (§3); relational design, migrations, connection pooling.
- **`GO_JWT_ARGON2_GUIDE.md`** — the Go side of JWT validation and argon2/password hashing in depth (pairs directly with §9 and §4's hashing notes).
- **`GO_NET_HTTP_REST_API_GUIDE.md` / `GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`** — the Go resource-server HTTP layer the §9 middleware plugs into.

---

### Quick reference: install map

```bash
# Core
npm install better-auth
npm install --save-dev @better-auth/cli

# Adapters / drivers (pick what you use)
npm install pg                          # direct Postgres
npm install prisma @prisma/client       # Prisma
npm install drizzle-orm postgres        # Drizzle + postgres.js
npm install better-sqlite3              # SQLite (great for tests)
# Next.js needs nothing extra — better-auth ships better-auth/next-js & better-auth/react

# Go resource server (Architecture A)
go get github.com/golang-jwt/jwt/v5
go get github.com/MicahParks/keyfunc/v3   # JWKS fetch + cache + rotation
```

### Quick reference: the import map (verify subpaths per version)

```ts
import { betterAuth }            from "better-auth";
import { createAuthClient }      from "better-auth/react";          // or "better-auth/client"
import { prismaAdapter }         from "better-auth/adapters/prisma";
import { drizzleAdapter }        from "better-auth/adapters/drizzle";
import { toNextJsHandler }       from "better-auth/next-js";
import { getSessionCookie }      from "better-auth/cookies";
import { twoFactor, magicLink, organization, admin, bearer, jwt, multiSession } from "better-auth/plugins";
import { passkey }               from "better-auth/plugins/passkey";
import { createAccessControl }   from "better-auth/plugins/access";
import { twoFactorClient, organizationClient, adminClient, passkeyClient } from "better-auth/client/plugins";
```

---

*Better Auth documentation: https://www.better-auth.com — bookmark it. All code in this guide is written to be accurate to Better Auth **v1.x** as of 2026. Better Auth's core (`betterAuth()`, `createAuthClient()`, the four tables, the plugin-mirror pattern, sessions-by-default + JWTs-for-services) is stable; plugin option shapes, some exact route names (token/JWKS), and reset-password method naming move fastest — those are flagged with **⚡ Version note** throughout. When an exact symbol matters, trust your editor's inferred types and the official docs over any guide, including this one. And remember the through-line: in authentication, lean on the framework's secure defaults, never hand-roll the cryptographic parts, and treat security as a property of every feature — not a final chapter.*
