# Better Auth — Complete Offline Reference (Standalone, Next.js & Go)

A full, study-friendly reference for **Better Auth** — the framework-agnostic, fully type-safe authentication framework for the TypeScript ecosystem. Covers using it standalone (plain Node/Hono), with **Next.js** (App Router), and integrating a **Golang** backend that trusts Better Auth as the auth source of truth.

> **Who this is for:** Developers who want to *own their auth* — the data, the flows, the database tables — instead of renting it from a SaaS. If you've used NextAuth/Auth.js and hit its walls, evaluated Clerk but balked at vendor lock-in and per-MAU pricing, or were eyeing Lucia (now deprecated, see §1), Better Auth is the modern landing spot. This guide assumes you know TypeScript, basic SQL/ORMs, and HTTP/cookies. The Go sections assume basic Go.

> **Version note:** This guide targets **Better Auth v1.x** (the stable line throughout 2025–2026). Better Auth is, as of 2026, the most popular *self-hosted, comprehensive, type-safe* TS auth library: framework-agnostic, you own your database, no per-user pricing. The core API (`betterAuth()`, `createAuthClient()`, the plugin system, the four core tables) is stable. Plugins and adapter details move faster — anything fast-moving is flagged with **⚡ Version note**. For any exact symbol you're unsure about, confirm against the official docs at better-auth.com. Where I'm not 100% certain of an exact export name, I say so rather than invent it.

---

## Table of Contents
1. [What Better Auth Is & How It Compares](#1-what-better-auth-is--how-it-compares)
2. [Core Concepts (Server, Client, Schema, Adapters)](#2-core-concepts)
3. [Standalone Setup (Node / Hono)](#3-standalone-setup)
4. [Email & Password Authentication](#4-email--password-authentication)
5. [Social / OAuth Providers](#5-social--oauth-providers)
6. [Sessions & Cookies](#6-sessions--cookies)
7. [Plugins Ecosystem](#7-plugins-ecosystem)
8. [Next.js Integration (App Router)](#8-nextjs-integration)
9. [Golang Integration (The Realistic Architectures)](#9-golang-integration)
10. [Authorization — Roles, RBAC, Organizations & Admin](#10-authorization)
11. [Security Best Practices](#11-security-best-practices)
12. [Database Schema Reference & Extending the User Model](#12-database-schema-reference)
13. [Testing & Local Development](#13-testing--local-development)
14. [Gotchas & Best Practices](#14-gotchas--best-practices)
15. [Study Path & Build-to-Learn Projects](#15-study-path--build-to-learn-projects)

---

## 1. What Better Auth Is & How It Compares

### What it is

**Better Auth** is a TypeScript authentication *framework* (not a SaaS, not a hosted service). You install it as a dependency, point it at *your* database, and it manages the full lifecycle of authentication: sign-up, sign-in, sessions, OAuth, email verification, password reset, two-factor, passkeys, organizations, and more — through a plugin system.

The three pillars of its philosophy:

1. **Own your data.** Better Auth stores users, sessions, and accounts in **your** database (Postgres, MySQL, SQLite, etc.) via an adapter (Prisma, Drizzle, Kysely, or a direct driver). There's no external "user pool" you're locked into. You can `SELECT * FROM user` any time.
2. **Framework-agnostic.** The core is a request handler that speaks the Web `Request`/`Response` standard. That means it runs on Next.js, Nuxt, SvelteKit, Remix, Astro, Hono, Express, Bun, Node, Cloudflare Workers — anywhere. The *same* `auth` instance powers all of them.
3. **Fully type-safe.** The client is *inferred* from the server config. Add a plugin on the server, and the matching client method appears with correct types — no codegen step. Custom user fields flow through to `session.user` types automatically.

The mental model: **one server `auth` object** (created with `betterAuth({...})`) exposes:
- An HTTP **handler** (`auth.handler`) you mount under `/api/auth/*`.
- A server-side **API** (`auth.api.*`) for calling the same operations directly in your backend code (no HTTP round-trip).

…and **one client `authClient`** (created with `createAuthClient({...})`) that your frontend calls (`authClient.signIn.email(...)`, `authClient.useSession()`, etc.).

### When to choose Better Auth

| You want… | Better Auth fit |
|---|---|
| To own user data in your own DB, no vendor lock-in | Ideal |
| Type-safe end-to-end auth without codegen | Ideal |
| Self-host, no per-MAU billing | Ideal |
| One auth layer across multiple TS frameworks | Ideal |
| Rich features (2FA, passkeys, orgs, magic link) without stitching libs | Ideal — plugins |
| A drop-in hosted UI with zero backend (you don't want a DB) | Use Clerk/Auth0 instead |
| You're not in the JS/TS ecosystem at all | Better Auth is TS-first; for a pure-Go shop, use a Go-native solution. (But see §9 — Go *resource servers* pair with it well.) |

### How it compares

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

**Lucia is deprecated.** Lucia was a popular low-level auth *toolkit* — you assembled sessions yourself. Its maintainer announced its sunset and pointed the community toward learning resources and higher-level options; Better Auth absorbed much of that audience by offering Lucia's "own your data" ethos *plus* batteries-included features. If you see a tutorial recommending Lucia for a new 2026 project, prefer Better Auth.

**vs NextAuth / Auth.js:** Auth.js is excellent and battle-tested, but historically Next-centric and lighter on built-in advanced features (2FA, orgs, admin). Better Auth is more "framework, not just adapters," with a richer plugin ecosystem and stronger inferred types. Both let you own your DB.

**vs Clerk:** Clerk gives you gorgeous drop-in UI and zero backend maintenance — at the cost of vendor lock-in, per-MAU pricing, and your users living in Clerk's cloud. Choose Clerk for speed-to-market when you don't want to own auth; choose Better Auth when ownership, cost at scale, and customization matter.

**vs Supabase Auth:** Great if you're already all-in on Supabase (it's tied to Supabase's Postgres + GoTrue). Better Auth is database-agnostic and not coupled to a single BaaS.

---

## 2. Core Concepts

### 2.1 The server `auth` instance

Everything starts with a single object created by `betterAuth()`. This is the **source of truth** — it knows your DB, your providers, your plugins, your secrets.

```ts
// lib/auth.ts  (this file is imported by both your route handler and your server code)
import { betterAuth } from "better-auth";

export const auth = betterAuth({
  // The base URL of the app where Better Auth's HTTP routes are mounted.
  // Used to build callback/redirect URLs. Often from env.
  baseURL: process.env.BETTER_AUTH_URL, // e.g. "http://localhost:3000"

  // A long random secret used to sign cookies/tokens. KEEP SECRET. Rotate carefully.
  secret: process.env.BETTER_AUTH_SECRET,

  // database: ...        (see §3 — Prisma / Drizzle / direct)
  // emailAndPassword: {} (see §4)
  // socialProviders: {}  (see §5)
  // plugins: []          (see §7)
});

// The type of a fully-resolved auth instance, handy for typing helpers.
export type Auth = typeof auth;
```

From `auth` you get two things you'll use constantly:

- **`auth.handler`** — a function `(req: Request) => Promise<Response>`. Mount it under `/api/auth/*` and Better Auth handles every auth route (sign-in, sign-up, callbacks, session, etc.).
- **`auth.api.*`** — server-side methods that perform the same operations *in-process* (no HTTP). E.g. `auth.api.getSession({ headers })`, `auth.api.signUpEmail({ body })`. Use these inside Server Components, Server Actions, middleware, or any backend code.

> **⚡ Version note:** Exact `auth.api.*` method names follow the route names (e.g. `getSession`, `signInEmail`, `signUpEmail`, `signOut`, `listSessions`). The set expands as you add plugins. Confirm the precise method name against the docs / your editor's autocomplete — the *pattern* (`auth.api.<operation>({ body?, headers?, query? })`) is stable.

### 2.2 The client

The client mirrors the server, with inferred types. You create it once and import it across your frontend.

```ts
// lib/auth-client.ts
import { createAuthClient } from "better-auth/client";
// For React apps, import from the react entry to get hooks like useSession:
// import { createAuthClient } from "better-auth/react";

export const authClient = createAuthClient({
  // Where the auth HTTP routes live. Can be omitted if same-origin and
  // mounted at the default /api/auth, but being explicit is safer.
  baseURL: process.env.NEXT_PUBLIC_BETTER_AUTH_URL, // e.g. "http://localhost:3000"
  // plugins: [...]  // client plugins must MIRROR server plugins (see §7)
});

// Common destructured helpers (names are stable):
export const { signIn, signUp, signOut, useSession, getSession } = authClient;
```

The key magic: if you add, say, the `twoFactor()` plugin on the server, then add its client counterpart, `authClient.twoFactor.*` methods appear *typed*. The client is a typed RPC layer over the server's HTTP routes.

### 2.3 Database & the core schema

Better Auth manages **four core tables**. Plugins add more (e.g. `twoFactor`, `passkey`, `organization`, `member`, `invitation`, `jwks`). The four core tables:

| Table | Purpose | Key columns (representative) |
|---|---|---|
| `user` | The person | `id`, `name`, `email`, `emailVerified`, `image`, `createdAt`, `updatedAt` |
| `session` | An active login | `id`, `userId`, `token`, `expiresAt`, `ipAddress`, `userAgent`, `createdAt`, `updatedAt` |
| `account` | A credential or linked provider | `id`, `userId`, `providerId`, `accountId`, `accessToken`, `refreshToken`, `idToken`, `password` (hashed, for email+password), `expiresAt` |
| `verification` | Short-lived tokens (email verify, password reset, OTP) | `id`, `identifier`, `value`, `expiresAt`, `createdAt` |

Notes that trip people up:
- **`account` holds the password.** For email+password, the hashed password lives in `account` (with `providerId = "credential"`), *not* in `user`. One user can have multiple `account` rows (one per OAuth provider + one credential row) — that's how **account linking** works.
- **`session.token`** is the opaque session identifier stored in the cookie (signed). The DB row is the server-side session.
- **`verification`** is a generic key/value-with-expiry table reused by many flows.

See §12 for the full reference and how to add your own columns (`additionalFields`).

### 2.4 Adapters

An **adapter** teaches Better Auth how to talk to your database. You pick one:

- **Prisma adapter** — `prismaAdapter(prisma, { provider })`. You add Better Auth's models to `schema.prisma`.
- **Drizzle adapter** — `drizzleAdapter(db, { provider, schema })`. You define the tables in Drizzle schema.
- **Kysely adapter** — for Kysely query builder.
- **Direct database** — pass a connection (e.g. a `pg` `Pool`, a MySQL pool, or a SQLite/`better-sqlite3` handle) and Better Auth uses its built-in Kysely-based adapter. Simplest for "no ORM."

The **CLI** (`@better-auth/cli`) reads your `auth` config and either **generates** the schema for your ORM (`generate`) or **migrates** the DB directly (`migrate`, supported for the built-in/Kysely adapter). See §3.

---

## 3. Standalone Setup

This section gets a *framework-free* Better Auth running: install, configure a database three ways (Prisma, Drizzle, direct Postgres), turn on email+password, run migrations, and mount the handler on plain Node and on Hono.

### 3.1 Install

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

Generate a secret and set env vars (`.env`):

```bash
# A 32+ byte random string. One quick way:
#   node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
BETTER_AUTH_SECRET=replace-with-long-random-hex
BETTER_AUTH_URL=http://localhost:3000
DATABASE_URL=postgres://user:pass@localhost:5432/myapp
```

### 3.2 Configure the database — three ways

#### (a) Direct Postgres (no ORM)

The simplest path. Pass a `pg` Pool; Better Auth uses its internal adapter and can run migrations for you.

```ts
// lib/auth.ts
import { betterAuth } from "better-auth";
import { Pool } from "pg";

export const auth = betterAuth({
  baseURL: process.env.BETTER_AUTH_URL,
  secret: process.env.BETTER_AUTH_SECRET,

  // Pass a node-postgres Pool directly. Better Auth detects the dialect.
  database: new Pool({ connectionString: process.env.DATABASE_URL }),

  emailAndPassword: { enabled: true },
});
```

Then create the tables:

```bash
# Generates AND applies the SQL migration against your DATABASE_URL.
# (migrate is available for the built-in/Kysely adapter — i.e. direct DB.)
npx @better-auth/cli migrate
```

#### (b) Prisma adapter

```prisma
// prisma/schema.prisma  (excerpt — the CLI can generate these models for you)
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
  password              String?   // hashed; only for the "credential" provider
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

const prisma = new PrismaClient();

export const auth = betterAuth({
  baseURL: process.env.BETTER_AUTH_URL,
  secret: process.env.BETTER_AUTH_SECRET,

  // The second arg's `provider` must match your Prisma datasource provider.
  database: prismaAdapter(prisma, { provider: "postgresql" }),

  emailAndPassword: { enabled: true },
});
```

Generate the Prisma models from your config (writes/updates `schema.prisma`), then run Prisma's own migrate:

```bash
npx @better-auth/cli generate   # writes Better Auth models into schema.prisma
npx prisma migrate dev          # Prisma applies the migration
```

> **⚡ Version note:** For Prisma/Drizzle, the CLI **generates schema** (`generate`); you then run *your ORM's* migration tool to apply it. `migrate` (direct apply) is for the built-in adapter. Don't expect `@better-auth/cli migrate` to drive Prisma.

#### (c) Drizzle adapter

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
  // Pass the schema so the adapter can map model names to your tables.
  database: drizzleAdapter(db, { provider: "pg", schema }),
  emailAndPassword: { enabled: true },
});
```

```bash
npx @better-auth/cli generate     # emit/refresh Drizzle schema
npx drizzle-kit generate          # create SQL migration
npx drizzle-kit migrate           # apply it
```

### 3.3 The CLI cheat sheet

| Command | What it does |
|---|---|
| `npx @better-auth/cli generate` | Reads your `auth` config and **generates the schema** for your ORM (Prisma models / Drizzle schema), or the SQL for direct DB. |
| `npx @better-auth/cli migrate` | **Applies** schema changes directly (built-in/Kysely adapter — i.e. direct DB connections). |
| `npx @better-auth/cli@latest ...` | Pin/refresh the CLI version. Re-run `generate` after adding plugins (they add tables). |

> **Always re-run `generate` (and migrate) after adding a plugin.** Plugins like `twoFactor`, `passkey`, `organization`, and `jwt` add their own tables/columns. Forgetting this is the #1 "it compiles but fails at runtime" mistake.

### 3.4 Mount the handler on plain Node

`auth.handler` takes a Web-standard `Request` and returns a `Response`. On a bare Node `http` server you must bridge Node's req/res to/from the Web API. The easiest correct approach is to use a small adapter, but here's the concept on raw Node:

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
      : await readBody(nodeReq); // collect the stream into a Buffer/string

  const webReq = new Request(url, {
    method: nodeReq.method,
    headers: nodeReq.headers as any,
    body,
  });

  // 2) Let Better Auth handle it.
  const webRes = await auth.handler(webReq);

  // 3) Convert Web Response -> Node response.
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

In practice you rarely hand-roll this. Use a framework that already speaks the Web API (Hono, or Next.js — see §8), or a community Node adapter. The point: **Better Auth's handler is just `Request → Response`.**

### 3.5 Mount the handler on Hono

Hono is Web-standard natively, so this is clean and is a great way to learn the standalone model:

```ts
// server.hono.ts
import { Hono } from "hono";
import { auth } from "./lib/auth";

const app = new Hono();

// Mount EVERY method on the catch-all so sign-in/up/callbacks all route through.
// Hono gives you the raw Web Request via c.req.raw — exactly what auth.handler wants.
app.on(["GET", "POST"], "/api/auth/*", (c) => auth.handler(c.req.raw));

// A protected example route: read the session server-side via auth.api.
app.get("/api/me", async (c) => {
  const session = await auth.api.getSession({ headers: c.req.raw.headers });
  if (!session) return c.json({ error: "Unauthorized" }, 401);
  return c.json({ user: session.user });
});

export default app; // run with Bun/Node per Hono's docs
```

That's a fully working standalone auth server: `/api/auth/*` handles the flows, `auth.api.getSession` protects your own routes.

---

## 4. Email & Password Authentication

Enable it in the config, optionally wiring email sending for verification and reset:

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
    requireEmailVerification: true,

    // Optional: tune password length bounds.
    minPasswordLength: 8,
    maxPasswordLength: 128,

    // Called when a user requests a password reset. YOU send the email.
    sendResetPassword: async ({ user, url, token }) => {
      // `url` is the full reset link the user clicks; `token` is the raw token
      // if you want to build your own link. Send via your email provider.
      await sendEmail({
        to: user.email,
        subject: "Reset your password",
        html: `Click to reset: <a href="${url}">${url}</a>`,
      });
    },
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

> **⚡ Version note:** The exact callback signatures (`sendResetPassword`, `sendVerificationEmail`) and whether they live under `emailAndPassword` vs `emailVerification` have shifted slightly across minor versions. The destructured args you can rely on are `user`, `url`, and `token`. Verify the exact nesting against the docs for your installed version.

### 4.1 The flows from the client

```ts
import { authClient } from "./lib/auth-client";

// --- SIGN UP ---
const { data, error } = await authClient.signUp.email({
  email: "ada@example.com",
  password: "correct-horse-battery-staple",
  name: "Ada Lovelace",
  // image: "https://...", // optional
  // Any additionalFields you defined on the user model can go here too.
  callbackURL: "/dashboard", // where to land after verification (optional)
});
if (error) console.error(error.message);

// --- SIGN IN ---
await authClient.signIn.email({
  email: "ada@example.com",
  password: "correct-horse-battery-staple",
  rememberMe: true, // persist the session longer (optional)
  callbackURL: "/dashboard",
});

// --- SIGN OUT ---
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

### 4.2 The same flows server-side (`auth.api`)

When you need to do these in backend code (e.g. an admin creating a user, or a server-rendered form):

```ts
import { auth } from "./lib/auth";

// Sign up a user from the server. Returns the created user/session.
await auth.api.signUpEmail({
  body: { email: "ada@example.com", password: "…", name: "Ada" },
});

// Sign in server-side. Pass headers so cookies/IP/UA are captured;
// the returned response sets the session cookie.
const res = await auth.api.signInEmail({
  body: { email: "ada@example.com", password: "…" },
  // asResponse: true returns a Response whose Set-Cookie you can forward.
  asResponse: true,
});
```

> **⚡ Version note:** Whether email verification is *required* to sign in depends on `requireEmailVerification`. With it on, `signIn.email` for an unverified user returns an error and (depending on version/config) may auto-resend the verification email.

---

## 5. Social / OAuth Providers

Add providers under `socialProviders`. Each needs a client ID/secret from that provider's console, plus the redirect/callback URL registered there.

```ts
// lib/auth.ts
export const auth = betterAuth({
  baseURL: process.env.BETTER_AUTH_URL, // critical: used to build callback URLs
  secret: process.env.BETTER_AUTH_SECRET,
  database: /* adapter */ undefined as any,

  socialProviders: {
    github: {
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    },
    google: {
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
      // Optional: request a refresh token, restrict prompts, etc.
      // prompt: "select_account",
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

### 5.1 The client-side flow

```ts
import { authClient } from "./lib/auth-client";

// Kicks off the OAuth dance. The browser is redirected to the provider,
// then back to callbackURL after Better Auth creates the session.
await authClient.signIn.social({
  provider: "github",
  callbackURL: "/dashboard",          // where to land on success
  errorCallbackURL: "/login?error=1", // optional, on failure
});
```

### 5.2 The callback flow (what happens)

1. User clicks "Sign in with GitHub" → `signIn.social({ provider: "github" })`.
2. Browser redirects to GitHub's authorize URL (with state/PKCE).
3. User approves; GitHub redirects to `{baseURL}/api/auth/callback/github?code=...&state=...`.
4. Better Auth's handler exchanges the code for tokens, fetches the profile, **finds or creates** a `user`, creates an `account` row (`providerId = "github"`) storing tokens, creates a `session`, sets the session cookie, and redirects to `callbackURL`.

### 5.3 Account linking

One `user` can have multiple `account` rows. Linking lets a signed-in user connect additional providers (or a credential password) to the *same* user.

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

**Automatic linking by email:** Better Auth can link a new OAuth login to an existing user if the verified email matches. This is convenient but a security trade-off — only enable trusted-provider auto-linking, because an attacker controlling an unverified email at a sloppy provider could otherwise hijack an account.

```ts
account: {
  accountLinking: {
    enabled: true,
    // Only auto-link when these providers assert a VERIFIED email.
    trustedProviders: ["google", "github"],
  },
},
```

> **⚡ Version note:** The exact shape of `account.accountLinking` and method names (`linkSocial`, `unlinkAccount`) are stable in concept; confirm casing against your version's docs.

---

## 6. Sessions & Cookies

### 6.1 How sessions work

On successful auth, Better Auth:
1. Inserts a `session` row (`token`, `userId`, `expiresAt`, `ipAddress`, `userAgent`).
2. Sets a **signed, httpOnly cookie** holding the session token.
3. On each request, reads the cookie, validates the signature, looks up the `session` row, and (if valid and unexpired) loads the user.

This is **database-backed (stateful) session** auth by default — the server can revoke any session instantly by deleting its row. (Contrast with stateless JWTs; see the JWT plugin in §7 and §9 for when you *do* want JWTs — namely for a Go resource server.)

### 6.2 Reading the session

**Client (React):**

```ts
import { authClient } from "./lib/auth-client";

function Profile() {
  // Reactive hook: re-renders when auth state changes.
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

**Server (`auth.api.getSession`)** — pass the incoming request headers so Better Auth can read the cookie:

```ts
import { auth } from "./lib/auth";
// `headers` is a Web Headers object (e.g. from a Request, or Next's headers()).
const session = await auth.api.getSession({ headers });
if (session) {
  console.log(session.user.id, session.session.expiresAt);
}
```

The returned object is roughly `{ user, session }` — `user` is the profile, `session` is the session row (token, expiry, etc.).

### 6.3 Session expiry & refresh

```ts
export const auth = betterAuth({
  // ...
  session: {
    expiresIn: 60 * 60 * 24 * 7,   // 7 days (seconds)
    updateAge: 60 * 60 * 24,       // refresh expiry if older than 1 day (sliding window)

    // Cache the session in a short-lived signed cookie to avoid a DB hit
    // on every request. Great for read-heavy apps; revocation lags by maxAge.
    cookieCache: {
      enabled: true,
      maxAge: 5 * 60, // 5 minutes
    },
  },
});
```

- **`expiresIn`** — absolute lifetime.
- **`updateAge`** — sliding refresh: each request past this age bumps the expiry, keeping active users logged in.
- **`cookieCache`** — stores a signed snapshot of the session in a cookie so `getSession` can skip the DB for `maxAge`. Trade-off: a revoked session may still appear valid until the cache expires.

### 6.4 Cookie configuration & security

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

    // Add a prefix to all auth cookie names (avoid collisions / multi-app domains).
    cookiePrefix: "myapp",

    // For sharing cookies across subdomains (e.g. app + api on same root domain):
    crossSubDomainCookies: {
      enabled: true,
      domain: ".example.com", // leading dot → all subdomains
    },
  },
});
```

Defaults you should know:
- **httpOnly: true** — JS can't read the session cookie (XSS-resistant).
- **sameSite: "lax"** — sent on top-level navigations; good CSRF default. Use `"none"` + `secure` only for true cross-site needs.
- **secure: true in production** — only sent over HTTPS.

> **⚡ Version note:** Exact paths (`advanced.useSecureCookies`, `advanced.crossSubDomainCookies`, `advanced.cookiePrefix`) are stable but occasionally reorganized; check the "Cookies" / "Advanced options" docs for your version. The *security defaults* (httpOnly + lax + secure-in-prod) are reliable.

### 6.5 Listing & revoking sessions (multi-device)

```ts
// All sessions for the current user (each device/login).
const sessions = await authClient.listSessions();

// Revoke one session by its token.
await authClient.revokeSession({ token: someSessionToken });

// Revoke every OTHER session (e.g. "sign out everywhere else").
await authClient.revokeOtherSessions();

// Revoke ALL sessions including the current one.
await authClient.revokeSessions();
```

Multi-session support (being signed into several accounts at once) is its own plugin — see §7.

---

## 7. Plugins Ecosystem

Plugins are how Better Auth grows. The pattern is always the same:
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
    twoFactor({
      issuer: "MyApp", // shown in authenticator apps
    }),

    // --- Magic link (passwordless email login) ---
    magicLink({
      sendMagicLink: async ({ email, url, token }) => {
        await sendEmail({ to: email, subject: "Your login link", html: `<a href="${url}">Sign in</a>` });
      },
    }),

    // --- Passkeys / WebAuthn ---
    passkey({
      rpID: "example.com",            // relying party ID (your domain)
      rpName: "MyApp",
      origin: "https://example.com",  // must match the browser origin
    }),

    // --- Organizations / teams ---
    organization({
      // allowUserToCreateOrganization: true,
      // membership roles default to owner/admin/member; customizable.
    }),

    // --- Admin (user management, impersonation, ban) ---
    admin(),

    // --- Multi-session (be logged into several accounts at once) ---
    multiSession(),

    // --- Bearer token: accept Authorization: Bearer <token> instead of cookies.
    //     ESSENTIAL for non-browser clients (mobile, CLI, a Go service). ---
    bearer(),

    // --- JWT plugin: exposes signed JWTs + a JWKS endpoint so OTHER services
    //     (e.g. your Go API) can verify tokens without calling back. See §9. ---
    jwt(),
  ],
});

async function sendEmail(_: { to: string; subject: string; html: string }) {}
```

```ts
// CLIENT — lib/auth-client.ts (mirror the server plugins)
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
    // bearer & jwt are server-side concerns — no client plugin usually needed.
  ],
});
```

> **⚡ Version note:** Import subpaths matter. Most plugins are exported from `better-auth/plugins` (server) and `better-auth/client/plugins` (client); `passkey` historically lives under `better-auth/plugins/passkey`. If an import fails, check the plugin's doc page for the exact subpath — these are the most version-sensitive strings in the whole library.

### 7.1 Two-factor (2FA / TOTP) usage

```ts
// 1) User enables 2FA — server returns a TOTP secret + otpauth URI for a QR code.
const { data } = await authClient.twoFactor.enable({ password: "current-password" });
// Render data.totpURI as a QR code; user scans it in Google Authenticator/Authy.

// 2) User confirms by entering a code from their app.
await authClient.twoFactor.verifyTotp({ code: "123456" });

// 3) On a future sign-in, if 2FA is on, signIn.email returns a "two factor
//    required" state. Then prompt for the code:
await authClient.twoFactor.verifyTotp({ code: "654321" });

// Backup codes for recovery:
await authClient.twoFactor.generateBackupCodes();
await authClient.twoFactor.verifyBackupCode({ code: "xxxx-xxxx" });
```

### 7.2 Magic link & email OTP

```ts
// Magic link: passwordless. User clicks the emailed link to sign in.
await authClient.signIn.magicLink({ email: "ada@example.com", callbackURL: "/dashboard" });

// Email OTP (a separate plugin, emailOTP()): server emails a 6-digit code.
await authClient.emailOtp.sendVerificationOtp({ email: "ada@example.com", type: "sign-in" });
await authClient.signIn.emailOtp({ email: "ada@example.com", otp: "123456" });
```

### 7.3 Passkeys / WebAuthn

```ts
// Register a passkey for the signed-in user (uses the platform authenticator).
await authClient.passkey.addPasskey();

// Sign in with a passkey (no password).
await authClient.signIn.passkey();
```

### 7.4 Bearer & JWT (the keys to non-cookie clients)

These two unlock mobile apps and **cross-language backends like Go**:

- **`bearer()`** — lets Better Auth accept `Authorization: Bearer <sessionToken>` *and* return the session token in a response header on sign-in (so a non-cookie client can store it and send it back). The token is still a DB-backed session.
- **`jwt()`** — issues **signed JWTs** and exposes a **JWKS** (public keys) endpoint, plus an endpoint to fetch a fresh JWT for the current session. A separate service (your Go API) can verify these JWTs *offline* against the JWKS — no callback to Better Auth per request.

```ts
// Get a JWT for the current session (to send to your Go API).
// Exact method/route may be `authClient.token()` or fetching the JWT endpoint;
// verify the exact call against the JWT plugin docs for your version.
const { data } = await authClient.getSession(); // session must exist first
// Then call the token/JWT endpoint exposed by the jwt() plugin (see §9.1).
```

JWKS is published at a well-known path under the auth routes (commonly `/api/auth/jwks`). See §9 for the full Go-side verification.

### 7.5 Rate limiting

Better Auth has **built-in rate limiting** on its endpoints (configurable), and you can add stricter custom rules. It's on by default in production.

```ts
export const auth = betterAuth({
  // ...
  rateLimit: {
    enabled: true,
    window: 60,        // seconds
    max: 100,          // requests per window per IP (global default)
    // Per-path overrides for sensitive endpoints:
    customRules: {
      "/sign-in/email": { window: 60, max: 5 },
      "/sign-up/email": { window: 60, max: 3 },
    },
    // storage: "database" | "memory" (use database/redis in multi-instance deploys)
  },
});
```

> **⚡ Version note:** In-memory rate limiting doesn't work across multiple server instances. For horizontally-scaled deploys, back rate limiting (and ideally secondary storage) with a shared store like Redis. Check the `rateLimit` and `secondaryStorage` docs.

### 7.6 Plugin quick reference

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

This is the most common deployment: Next.js App Router serves the frontend *and* hosts the Better Auth routes in the same app.

### 8.1 Install & env

```bash
npm install better-auth
# plus your DB adapter (Prisma/Drizzle/pg) as in §3
```

```bash
# .env.local
BETTER_AUTH_SECRET=long-random-hex
BETTER_AUTH_URL=http://localhost:3000
NEXT_PUBLIC_BETTER_AUTH_URL=http://localhost:3000
DATABASE_URL=postgres://...
```

### 8.2 Server instance (same as §2, lives in `lib/auth.ts`)

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
  emailAndPassword: { enabled: true },
  socialProviders: {
    github: {
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    },
  },
});
```

### 8.3 The catch-all route handler

This is the one line that mounts *all* of Better Auth into Next.js:

```ts
// src/app/api/auth/[...all]/route.ts
import { auth } from "@/lib/auth";
import { toNextJsHandler } from "better-auth/next-js";

// toNextJsHandler adapts auth.handler to Next's Route Handler signature
// and returns the correct GET and POST exports.
export const { GET, POST } = toNextJsHandler(auth);
```

That's it — `/api/auth/sign-in/email`, `/api/auth/callback/github`, `/api/auth/get-session`, etc. all work now.

### 8.4 The client (React hooks)

```ts
// src/lib/auth-client.ts
import { createAuthClient } from "better-auth/react";

export const authClient = createAuthClient({
  baseURL: process.env.NEXT_PUBLIC_BETTER_AUTH_URL,
});

export const { signIn, signUp, signOut, useSession } = authClient;
```

### 8.5 Login & Sign-up pages (Client Components)

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
      setError(error.message ?? "Sign in failed");
    } else {
      router.push("/dashboard");
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <input value={email} onChange={(e) => setEmail(e.target.value)} placeholder="Email" type="email" />
      <input value={password} onChange={(e) => setPassword(e.target.value)} placeholder="Password" type="password" />
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
    else router.push("/dashboard");
  }

  return (
    <form onSubmit={handleSubmit}>
      <input placeholder="Name" onChange={(e) => setForm({ ...form, name: e.target.value })} />
      <input placeholder="Email" type="email" onChange={(e) => setForm({ ...form, email: e.target.value })} />
      <input placeholder="Password" type="password" onChange={(e) => setForm({ ...form, password: e.target.value })} />
      {error && <p style={{ color: "red" }}>{error}</p>}
      <button>Create account</button>
    </form>
  );
}
```

### 8.6 Server-side session in a Server Component

Server Components can read the session directly via `auth.api.getSession`, passing Next's request headers. **Do not** make a Client Component fetch just to gate a server page.

```tsx
// src/app/dashboard/page.tsx  (Server Component — protected)
import { auth } from "@/lib/auth";
import { headers } from "next/headers";
import { redirect } from "next/navigation";

export default async function DashboardPage() {
  // headers() must be awaited in Next 15+/16.
  const session = await auth.api.getSession({ headers: await headers() });

  if (!session) {
    redirect("/login"); // not authenticated → bounce to login
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

### 8.7 Server Actions

```tsx
// src/app/dashboard/actions.ts
"use server";
import { auth } from "@/lib/auth";
import { headers } from "next/headers";

export async function updateProfile(formData: FormData) {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session) throw new Error("Unauthorized");

  const name = String(formData.get("name"));
  // Update the user via auth.api.updateUser (verify exact method name) or your ORM.
  await auth.api.updateUser({
    body: { name },
    headers: await headers(),
  });
}
```

### 8.8 Middleware — the important caveat

Next.js middleware runs on the **Edge runtime** and *cannot* reliably do a database lookup. So the recommended pattern is: in middleware, do a **lightweight cookie check** (does an auth cookie exist?) for a fast redirect, and do the **real** session validation in the Server Component / Server Action / Route Handler (Node runtime).

```ts
// src/middleware.ts
import { NextResponse, type NextRequest } from "next/server";
import { getSessionCookie } from "better-auth/cookies";

export function middleware(request: NextRequest) {
  // OPTIMISTIC check only: is a session cookie present?
  // This does NOT validate the session against the DB. It's a fast gate.
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

> **⚡ Critical caveat:** The cookie's *presence* does not mean it's *valid* — it could be expired or forged. **Never** treat the middleware check as real authorization. Always re-validate with `auth.api.getSession` in the actual page/handler (Node runtime, can hit the DB). If you enabled `cookieCache`, `getSessionCookie` reading the cached cookie is still optimistic. The exact helper name (`getSessionCookie` from `better-auth/cookies`) is stable but verify the import path.

> Alternative: if you must validate in middleware, call the `get-session` HTTP endpoint with `fetch` (forwarding cookies) from middleware — but that's an extra network hop on every navigation. The cookie-presence pattern is idiomatic.

### 8.9 Cross-origin Next.js + separate API note

If your Next.js frontend and Better Auth live on **different origins** (e.g. `app.example.com` calling `auth.example.com`), you must set `trustedOrigins` on the server (§11) and configure cross-subdomain cookies (§6.4) so the cookie is shared. For *totally* different domains, you'll lean on the **bearer/JWT** approach instead of cookies.

---

## 9. Golang Integration

This is the section most guides skip. Better Auth is TypeScript — you don't run it *in* Go. Instead, Better Auth is the **identity provider / auth service**, and your **Go service is a resource server** that trusts tokens Better Auth issued. There are three realistic architectures.

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

### 9.1 Architecture A (recommended): Go verifies Better Auth JWTs via JWKS

This is the cleanest, most scalable design and the most "idiomatic" for a polyglot stack. It's standard OAuth2/OIDC resource-server thinking:

1. Enable the **`jwt()` plugin** on Better Auth. It signs JWTs (asymmetric, e.g. EdDSA/ES256) and publishes the **public keys** at a **JWKS** endpoint (commonly `GET {baseURL}/api/auth/jwks`).
2. The frontend obtains a **JWT** for the current session from Better Auth's token endpoint, then sends it to the Go API as `Authorization: Bearer <jwt>`.
3. The **Go API fetches the JWKS once (and caches it), then verifies every incoming JWT offline** — signature, issuer, audience, expiry. **No per-request callback to Better Auth.** Fast and horizontally scalable.

**Getting a JWT on the frontend** (the JWT plugin exposes a token endpoint). The session itself stays a cookie; you exchange it for a short-lived JWT to call Go:

```ts
// Frontend: fetch a JWT to call the Go API.
// The jwt() plugin exposes a token endpoint (commonly /api/auth/token).
// Verify the exact route/method name against the JWT plugin docs for your version.
const res = await fetch(`${BETTER_AUTH_URL}/api/auth/token`, {
  credentials: "include", // send the session cookie so the server knows who you are
});
const { token } = await res.json(); // the JWT to forward to Go

await fetch("https://api.example.com/orders", {
  headers: { Authorization: `Bearer ${token}` },
});
```

**Go side — verify the JWT against Better Auth's JWKS.** Using `github.com/golang-jwt/jwt/v5` plus a JWKS fetcher/cache:

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

// Expected issuer/audience — set these to your Better Auth baseURL / configured aud.
const (
	expectedIssuer   = "https://auth.example.com"
	expectedAudience = "https://api.example.com" // if you configure an audience
)

// Keyfunc holds the cached JWKS and auto-refreshes on key rotation.
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

// VerifyToken parses and fully validates the JWT (signature + standard claims).
func VerifyToken(tokenString string) (*Claims, error) {
	claims := &Claims{}

	token, err := jwt.ParseWithClaims(
		tokenString,
		claims,
		keyfn.Keyfunc, // resolves the right public key by the token's `kid` header
		jwt.WithValidMethods([]string{"EdDSA", "ES256", "RS256"}), // restrict algs
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

// ctxKey avoids collisions in context values.
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

**Why this is idiomatic:** the Go service behaves like any OAuth2 resource server validating a JWT against a JWKS — the same pattern you'd use with Auth0, Keycloak, or Cognito. Better Auth just happens to be the issuer. Asymmetric keys mean Go never needs Better Auth's secret; it only needs the *public* keys from JWKS.

> **⚡ Version note:** Verify (1) the exact JWKS path — commonly `/api/auth/jwks`; (2) the token-fetch route — commonly `/api/auth/token`; (3) the signing algorithm the `jwt()` plugin uses (often EdDSA) so your `WithValidMethods` list matches; (4) whether you've configured an `audience` (if not, drop `WithAudience`). These specifics come from the JWT plugin's config and docs.

### 9.2 Architecture B: Go calls Better Auth's session-validation endpoint

Simpler to reason about, but adds a network hop per request (mitigate with caching). Here Go does **not** verify anything itself — it forwards the caller's cookie/bearer token to Better Auth's `get-session` endpoint and trusts the answer.

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

var httpClient = &http.Client{Timeout: 5 * time.Second}

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

Use it the same way in middleware (swap `VerifyToken` for `ValidateViaBetterAuth`). **Add a short-lived cache** (e.g. keyed by the token, TTL 30–60s) so you're not hammering Better Auth on every request. This requires the **`bearer()`** plugin if the Go client sends a token instead of a cookie.

**Trade-offs vs Architecture A:**

| | A: JWT + JWKS | B: introspect get-session |
|---|---|---|
| Per-request network hop | No (offline verify) | Yes (mitigate w/ cache) |
| Instant revocation | No (until JWT exp) | Yes (DB-backed) |
| Scales horizontally | Excellent | Good w/ caching |
| Complexity in Go | Moderate (JWT libs) | Low (one HTTP call) |
| Best for | High-throughput APIs | Internal/low-traffic, needs instant revoke |

Pick **A** for performance/scale, **B** when instant revocation matters more than latency.

### 9.3 Architecture C: shared cookie (same root domain)

If the Next.js frontend (`app.example.com`) and the Go API (`api.example.com`) share a **root domain**, you can share Better Auth's session cookie directly — the browser sends it to both.

- On Better Auth, enable cross-subdomain cookies with `domain: ".example.com"` (§6.4) and add `api.example.com` to `trustedOrigins`.
- The Go API reads the session cookie and validates it. **But Go can't verify Better Auth's signed cookie or look up the DB session as cleanly** — the cookie is a *signed session token*, and re-implementing the signing scheme in Go is fragile. So in practice, even with a shared cookie, Go still **forwards the cookie to `get-session`** (Architecture B) to resolve it. The "shared cookie" part just means the browser sends it automatically; the validation is still B.

```go
// With a shared cookie, the browser already sends Cookie to api.example.com.
// Go just forwards it (Architecture B) — no JWT exchange needed on the frontend.
session, err := auth.ValidateViaBetterAuth(r)
```

**Honest take:** Don't try to re-implement Better Auth's cookie signing or DB session reads in Go by hand — that couples your Go service to Better Auth internals that can change. Use **JWKS verification (A)** for offline checks, or **forward to get-session (B)** when you want the DB to be the authority. Shared cookies (C) are a convenience for *transport*, not a separate validation method.

### 9.4 CORS for cross-origin frontend → Go

If the browser calls the Go API on another origin with credentials, configure CORS on Go and `trustedOrigins` on Better Auth:

```go
func withCORS(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
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

Authentication ("who are you?") is solved by sessions. **Authorization** ("what can you do?") is layered on top with roles and the org/admin plugins.

### 10.1 Simple roles via additionalFields

For app-wide roles (user/admin), add a `role` field to the user model:

```ts
export const auth = betterAuth({
  // ...
  user: {
    additionalFields: {
      role: {
        type: "string",
        defaultValue: "user",
        input: false, // users can't set their own role at sign-up
      },
    },
  },
});
```

Then gate server-side:

```ts
const session = await auth.api.getSession({ headers });
if (session?.user.role !== "admin") throw new Error("Forbidden");
```

### 10.2 The admin plugin

`admin()` formalizes app-level roles plus user-management superpowers:

```ts
import { admin } from "better-auth/plugins";

plugins: [
  admin({
    // defaultRole: "user",
    // adminRoles: ["admin"],          // which roles are "admins"
  }),
];
```

Client methods (require the caller to be an admin):

```ts
await authClient.admin.listUsers({ query: { limit: 50 } });
await authClient.admin.setRole({ userId, role: "admin" });
await authClient.admin.banUser({ userId, banReason: "spam", banExpiresIn: 60 * 60 * 24 });
await authClient.admin.unbanUser({ userId });
await authClient.admin.impersonateUser({ userId }); // act as another user
await authClient.admin.stopImpersonating();
```

### 10.3 Organizations & teams (multi-tenant RBAC)

The `organization()` plugin adds orgs, members, roles (owner/admin/member by default), and invitations — the backbone of B2B/multi-tenant apps.

```ts
import { organization } from "better-auth/plugins";

plugins: [organization()];
```

```ts
// Create an org (the creator becomes owner).
await authClient.organization.create({ name: "Acme Inc", slug: "acme" });

// Invite a member with a role.
await authClient.organization.inviteMember({
  organizationId,
  email: "bob@acme.com",
  role: "member",
});

// Accept an invitation (invitee, from the emailed link).
await authClient.organization.acceptInvitation({ invitationId });

// Set the "active" org for the current session (scopes subsequent requests).
await authClient.organization.setActive({ organizationId });

// List members, update roles, remove members.
await authClient.organization.updateMemberRole({ organizationId, memberId, role: "admin" });
await authClient.organization.removeMember({ organizationId, memberIdOrEmail: "bob@acme.com" });
```

### 10.4 Access control (fine-grained permissions)

Better Auth ships an **access-control** helper to define permissions per resource and check them. You define a statement of resources→actions, create roles from it, and check `hasPermission`.

```ts
// access.ts
import { createAccessControl } from "better-auth/plugins/access";

// Define what actions exist per resource.
const statement = {
  project: ["create", "read", "update", "delete"],
  billing: ["read", "manage"],
} as const;

export const ac = createAccessControl(statement);

// Build roles from the statement.
export const member = ac.newRole({ project: ["read"] });
export const admin  = ac.newRole({ project: ["create", "read", "update", "delete"], billing: ["read", "manage"] });
```

Wire these roles into the `organization()` (or `admin()`) plugin's `roles`/`ac` options, then check:

```ts
// Check whether the current user can delete a project in the active org.
const { data } = await authClient.organization.hasPermission({
  permissions: { project: ["delete"] },
});
if (!data?.success) throw new Error("Forbidden");
```

> **⚡ Version note:** The access-control API (`createAccessControl`, `newRole`, `hasPermission`, and how `ac`/`roles` plug into `organization()`/`admin()`) is powerful but one of the more version-sensitive areas. Confirm the exact import path (`better-auth/plugins/access`) and option names against the "Access Control" docs for your version.

### 10.5 Authorization in Go

The Go resource server enforces authorization from the JWT/session claims. Put `role` (and org/permission info) into the JWT (configure the `jwt()` plugin's payload) so Go can decide without extra calls:

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

---

## 11. Security Best Practices

| Area | Do this |
|---|---|
| **Secret** | `BETTER_AUTH_SECRET` = 32+ random bytes. Never commit it. Use a secrets manager in prod. Rotating it invalidates signed cookies/tokens (plan for it). |
| **HTTPS** | Always serve auth over HTTPS in production so `secure` cookies work. |
| **Cookies** | Keep defaults: `httpOnly` (no JS access), `sameSite: "lax"`, `secure` in prod. Only loosen for genuine cross-site needs. |
| **CSRF** | Better Auth uses `sameSite` cookies + origin checks. Keep `trustedOrigins` tight (see below). For cross-site cookie auth (`sameSite: "none"`), ensure CSRF protections are understood. |
| **trustedOrigins** | Explicitly list every origin allowed to make auth requests / receive redirects. Don't use wildcards in prod. |
| **Rate limiting** | Keep it enabled; tighten sign-in/sign-up/reset endpoints. Use shared storage (Redis) across instances (§7.5). |
| **Password hashing** | Default is **scrypt** (memory-hard). Don't downgrade. You can plug a custom hasher (e.g. argon2) if you have a reason. |
| **Email verification** | Turn on `requireEmailVerification` for credential sign-ups to stop fake/abused accounts. |
| **Account linking** | Only auto-link via `trustedProviders` that assert verified emails (§5.3). |
| **JWT (Go)** | Use **asymmetric** signing so resource servers verify with public keys only. Always check `iss`, `aud` (if set), `exp`. Keep JWT lifetimes short. |
| **Session revocation** | Remember JWTs (Arch A) can't be revoked before expiry — keep them short-lived; use introspection (Arch B) when instant revoke matters. |

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
  // Custom password hasher (optional — default scrypt is fine):
  // emailAndPassword: { password: { hash: async (p) => ..., verify: async ({ hash, password }) => ... } }
});
```

> **CSRF in plain terms:** Because session cookies default to `sameSite: "lax"`, a malicious cross-site form can't silently send the cookie on a state-changing POST. Combined with origin validation against `trustedOrigins`, this covers the common CSRF vectors. If you switch to `sameSite: "none"` (cross-site), you must add explicit anti-CSRF measures (e.g. tokens) — prefer bearer/JWT for cross-site instead.

---

## 12. Database Schema Reference

### 12.1 Core tables (full reference)

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
| `ipAddress` | string? | captured at creation |
| `userAgent` | string? | captured at creation |
| `createdAt`/`updatedAt` | datetime | |

**`account`** (credentials + linked OAuth providers)

| Column | Type | Notes |
|---|---|---|
| `id` | string (PK) | |
| `userId` | string (FK→user) | |
| `accountId` | string | provider's user id (or the user id for credentials) |
| `providerId` | string | `"credential"`, `"github"`, `"google"`, … |
| `accessToken` | string? | OAuth |
| `refreshToken` | string? | OAuth |
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

Plugin tables you'll commonly also see: `twoFactor`, `passkey`, `jwks` (public/private keys for the JWT plugin), `organization`, `member`, `invitation`, `apiKey`.

### 12.2 Extending the user model (`additionalFields`)

Add custom columns; they flow into types and into sign-up input (unless `input: false`).

```ts
export const auth = betterAuth({
  // ...
  user: {
    additionalFields: {
      role:        { type: "string",  defaultValue: "user", input: false },
      bio:         { type: "string",  required: false },
      isPro:       { type: "boolean", defaultValue: false, input: false },
      age:         { type: "number",  required: false },
      // references: { ... } // relate to other models if supported by your adapter
    },
  },
});
```

After adding fields, **re-run the CLI** to update the schema, then migrate. The new fields are now typed on `session.user`:

```ts
const session = await auth.api.getSession({ headers });
session?.user.bio;   // typed!
session?.user.isPro; // typed!
```

You can also customize table/column names and model mapping via the adapter options or `user.modelName` / field mapping — useful when retrofitting Better Auth onto an existing schema. (Verify the exact mapping option names for your version.)

---

## 13. Testing & Local Development

### 13.1 Local dev tips

- **Email in dev:** Don't wire a real SMTP provider. Log the verification/reset URL to the console in `sendVerificationEmail`/`sendResetPassword`, or use a local catcher (Mailpit/Mailhog/Inbucket via Docker). Clicking the logged link still works.

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

- **OAuth in dev:** Register a separate dev OAuth app per provider with `http://localhost:3000/api/auth/callback/<provider>` as the redirect. Keep dev/prod client secrets separate.
- **DB:** Run Postgres in Docker; point `DATABASE_URL` at it. Re-run `generate`/`migrate` whenever you add a plugin or `additionalFields`.
- **Disable rate limiting in tests** or set a huge `max`, or it'll flake your test suite on repeated sign-ins.
- **Seeded users:** Create test users via `auth.api.signUpEmail` in a seed script so the password hashing matches exactly.

### 13.2 Integration testing

Test against the real `auth` instance using an in-memory/SQLite DB:

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
  // against a temp DB, or use the adapter's schema-push. (Verify approach
  // for your adapter — for direct sqlite, the CLI migrate path applies.)
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
    const session = await auth.api.getSession({
      headers: new Headers({ cookie }),
    });
    expect(session?.user.email).toBe("t@example.com");
  });
});
```

### 13.3 Testing the Go side

- Unit-test `VerifyToken` by signing test JWTs with a throwaway key pair and pointing the verifier at a local JWKS (or inject a `keyfunc` that returns your test public key).
- Integration-test `ValidateViaBetterAuth` against a running Better Auth dev instance, or stub the `get-session` endpoint with an `httptest.Server`.

---

## 14. Gotchas & Best Practices

- **Re-run the CLI after every plugin or `additionalFields` change.** Plugins add tables/columns; forgetting causes runtime "no such column/table" errors. This is the most common pitfall.
- **Client plugins must mirror server plugins.** A server-side `twoFactor()` without `twoFactorClient()` means no typed `authClient.twoFactor.*`. Mismatches = missing methods.
- **Import subpaths are exact and version-sensitive.** `better-auth/plugins`, `better-auth/client/plugins`, `better-auth/plugins/passkey`, `better-auth/adapters/prisma`, `better-auth/next-js`, `better-auth/cookies`. A wrong subpath is the most common "module not found."
- **`baseURL` must be correct.** It builds OAuth callbacks and redirects. A wrong `baseURL` → broken OAuth and "untrusted origin" errors. Match it (and provider redirect URLs) to where the app actually runs.
- **Next.js middleware can't validate sessions (Edge).** Use the cookie-presence optimistic check for redirects; do real validation in the page/handler (§8.8).
- **`headers()` is async in Next 15+/16.** `await headers()` before passing to `auth.api.getSession`.
- **Cookie cache lags revocation.** With `session.cookieCache` on, a revoked/expired session may appear valid until the cache TTL passes. Keep TTL short; for sensitive ops, validate against the DB.
- **JWTs (Arch A) can't be revoked early.** Keep them short-lived; use introspection (Arch B) when you need instant kill-switch.
- **scrypt is the default password hash.** Good and memory-hard. Don't replace it without reason; if you migrate from bcrypt, plan a re-hash-on-login strategy.
- **Account linking by email is a foot-gun if mis-scoped.** Only `trustedProviders` with verified emails.
- **Rate limit storage in multi-instance deploys.** In-memory won't share across pods; use Redis/DB-backed storage.
- **Don't reimplement Better Auth's cookie/session in Go.** Use JWKS (A) or get-session introspection (B). Re-implementing signing couples you to internals.
- **`trustedOrigins` must include every front-end/back-end origin** that talks to auth — otherwise CSRF/origin checks reject requests.
- **Verify exact method/route names against your installed version.** This guide is precise where the API is stable and explicitly flags the spots (token endpoint, JWKS path, some plugin options, reset-password naming) that drift. Trust your editor's autocomplete and the docs over memory.

---

## 15. Study Path & Build-to-Learn Projects

### Suggested learning order

1. **Standalone email+password (Hono or direct pg).** Get `betterAuth()` + adapter + CLI migrate working. Sign up, sign in, read a session with `auth.api.getSession`. *Goal: understand the four tables and the handler.*
2. **Add OAuth (GitHub).** Register a dev app, wire `socialProviders`, complete the callback flow. Inspect the `account` rows. *Goal: understand provider linking.*
3. **Next.js App Router.** Mount `toNextJsHandler`, build login/signup pages, protect a dashboard Server Component, add the optimistic middleware. *Goal: server vs client session access.*
4. **Sessions deep-dive.** Configure `expiresIn`/`updateAge`/`cookieCache`; build a "sessions/devices" page with `listSessions` + `revokeSession`. *Goal: stateful sessions & security trade-offs.*
5. **Plugins.** Add `twoFactor` (with a QR code), `magicLink`, and `organization`. *Goal: the plugin mirror pattern + schema regen.*
6. **JWT + JWKS for Go.** Enable `jwt()` + `bearer()`; fetch a token on the frontend; build the Go resource server that verifies it (Architecture A). *Goal: polyglot, offline verification.*
7. **Authorization.** Add roles via `admin()` and orgs via `organization()`; enforce RBAC in both Next.js and Go. *Goal: real multi-tenant authz.*
8. **Harden.** `trustedOrigins`, rate limits, secure cookies, email verification required, short-lived JWTs. *Goal: production posture.*

### Build-to-learn projects

| # | Project | What it teaches |
|---|---|---|
| 1 | **Auth playground (standalone Hono)** | Core instance, adapter, CLI, `auth.api.getSession`, the four tables |
| 2 | **Next.js SaaS starter** | `toNextJsHandler`, `useSession`, protected Server Components, middleware caveat, OAuth |
| 3 | **Devices & security center** | `listSessions`/`revokeSession`, 2FA enable/verify, passkeys, password reset & email verification |
| 4 | **B2B multi-tenant app** | `organization()` (orgs/members/invites), access-control roles, admin plugin, RBAC |
| 5 | **★ Full-stack polyglot app: Next.js front + Go API, both authed by Better Auth** | The capstone — `jwt()`/`bearer()`, JWKS verification in Go (Arch A) *and* a fallback get-session introspection path (Arch B), CORS, shared-domain cookies (Arch C), role enforcement on both sides |

The capstone (#5) is the real test of this guide: a Next.js app where users sign up / log in via Better Auth, the frontend fetches a JWT and calls a separate Go microservice, and the Go service verifies that JWT against Better Auth's JWKS and enforces roles — no auth logic duplicated, one source of truth, two languages.

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

# Next.js: nothing extra — better-auth ships better-auth/next-js & better-auth/react

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

*Better Auth documentation: https://www.better-auth.com — bookmark it. All code in this guide is written to be accurate to Better Auth **v1.x** as of 2026. Better Auth's core (`betterAuth()`, `createAuthClient()`, the four tables, the plugin-mirror pattern) is stable; plugin option shapes, some exact route names (token/JWKS), and reset-password method naming move fastest — those are flagged with **⚡ Version note** throughout. When an exact symbol matters, trust your editor's inferred types and the official docs over any guide, including this one.*
