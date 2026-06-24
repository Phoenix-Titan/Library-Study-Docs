# Supabase (with Next.js & Go) — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Full-stack developers going from "I have never touched Supabase" to "I can architect, secure, and ship a production app on it" — entirely offline. The unusual angle of this guide is that it covers **two clients against the same project at once**: a Next.js front-end (App Router) using the JavaScript client, *and* a Go back-end talking directly to Postgres and verifying Supabase's auth tokens. Every concept is explained in **prose first** — what it is, the logic/why it was designed that way, what it's for and when to reach for it, how to use it, the key options and parameters, best practices, and the **security implications** — and only then shown as heavily-commented, runnable code. Read top-to-bottom the first time; afterward use the Table of Contents as a lookup. Sections carry **[B]** beginner, **[I]** intermediate, **[A]** advanced tags.
>
> **Version note:** This guide targets **Supabase as of 2026**. Key facts baked in:
> - The backend is **PostgreSQL** (Postgres 15/16/17 depending on project age). Everything is "just Postgres" underneath — the deep relational material lives in **`POSTGRESQL_GUIDE.md`** and **`RELATIONAL_DB_DESIGN_GUIDE.md`**; this guide assumes you'll lean on those for SQL/modeling depth and focuses on what is *Supabase-specific*.
> - The Next.js integration uses the **`@supabase/ssr`** package. The older **`@supabase/auth-helpers-nextjs` is deprecated** — do **not** use it for new work. (See **`NEXTJS_16_GUIDE.md`** for the App Router model these patterns build on.)
> - The JavaScript client is **supabase-js v2** (`@supabase/supabase-js`).
> - Local dev, migrations, and type generation use the **Supabase CLI** over a Docker-based local stack.
> - JWTs: legacy projects sign with a shared **HS256** secret; newer projects (2025+) support **asymmetric signing keys (RS256/ES256) exposed via JWKS**. Both are covered.
>
> Supabase moves fast. Where a detail is especially likely to drift it is flagged with **⚡ Version note**. Confirm exact API shapes against the official docs, but the *concepts* here are stable. (Go code blocks use **tabs** for indentation, per Go convention.)

---

## Table of Contents

1. [What Supabase Is & When to Use It](#1-what-supabase-is--when-to-use-it) **[B]**
2. [Getting Started: Project, Keys, CLI, Local Dev](#2-getting-started-project-keys-cli-local-dev) **[B]**
3. [The Database & Row Level Security (RLS)](#3-the-database--row-level-security-rls) **[B/I/A]**
4. [supabase-js Client Deep Dive](#4-supabase-js-client-deep-dive) **[B/I]**
5. [Auth (GoTrue)](#5-auth-gotrue) **[B/I]**
6. [Next.js Integration (App Router) with @supabase/ssr](#6-nextjs-integration-app-router-with-supabasessr) **[I/A]**
7. [Realtime](#7-realtime) **[I]**
8. [Storage](#8-storage) **[I]**
9. [Edge Functions (Deno)](#9-edge-functions-deno) **[I]**
10. [Database Migrations & Generated Types](#10-database-migrations--generated-types) **[I]**
11. [Golang Integration (Direct DB, JWT, REST, RLS)](#11-golang-integration-direct-db-jwt-rest-rls) **[A]**
12. [pgvector / AI: Embeddings & Semantic Search](#12-pgvector--ai-embeddings--semantic-search) **[A]**
13. [Security Best Practices](#13-security-best-practices) **[I/A]**
14. [Self-Hosting & Pricing/Limits Awareness](#14-self-hosting--pricinglimits-awareness) **[I]**
15. [Gotchas & Best Practices](#15-gotchas--best-practices) **[A]**
16. [Study Path & Build-to-Learn Projects](#16-study-path--build-to-learn-projects)

---

## 1. What Supabase Is & When to Use It

### 1.1 The one-sentence definition **[B]**

Supabase is an **open-source Firebase alternative built on PostgreSQL**: a managed Postgres database wrapped in a suite of services — Auth, an auto-generated REST API, Realtime, Storage, and Edge Functions — that together turn the database into a complete backend you can talk to directly from a browser.

The reason that sentence matters is the contrast it draws. Firebase made backend-less apps popular, but its datastore is a proprietary NoSQL document store: no joins, no transactions across documents, no SQL, weak constraints, and vendor lock-in. Supabase keeps the developer experience Firebase pioneered — *the frontend talks to the backend directly, no API tier to hand-write* — but puts a **real relational SQL database** underneath it. You get joins, foreign keys, transactions, check constraints, views, triggers, and the entire Postgres extension ecosystem, and your data is a standard Postgres dump you can take anywhere.

### 1.2 The core mental model: the database is the security boundary **[B]**

This is the single most important idea in the whole product, so internalize it before anything else. In a traditional app, a hand-written backend sits between the client and the database and enforces "who can do what." Supabase mostly **removes that middle tier**: the browser holds a public API key and talks to Postgres (through a thin REST layer). That sounds insane — how is it not wide open? — and the answer is the defining design decision of Supabase:

**Authorization is enforced *inside the database itself*, per-row, by PostgreSQL's Row Level Security (RLS).** Auth issues a signed JWT carrying the user's id; every access path (the REST API, Realtime, Storage) forwards that JWT into Postgres; and RLS policies — rules you write once in SQL — decide which rows that user may select, insert, update, or delete. Write your security rules *once*, and **every** client (JS, Go, mobile, a raw HTTP call) is governed by them identically. This is why understanding RLS (§3) is non-negotiable: it *is* your backend's security.

### 1.3 The pieces **[B]**

| Service | What it is | Underlying tech |
|---|---|---|
| **Database** | A full PostgreSQL instance you own — tables, SQL, extensions, functions, triggers. | PostgreSQL |
| **Auth** | User sign-up/sign-in, sessions, OAuth, magic links, MFA. Issues JWTs. | **GoTrue** (a Go auth server, fittingly) |
| **Auto REST API** | Every table/view is instantly a RESTful endpoint with filtering, joins, pagination. | **PostgREST** |
| **Realtime** | Subscribe to DB changes (insert/update/delete), broadcast messages, presence. | Elixir/Phoenix server + Postgres logical replication |
| **Storage** | S3-style object storage for files, access controlled by the *same* Postgres RLS. | Storage API + a `storage` schema in Postgres |
| **Edge Functions** | Serverless TypeScript/JavaScript functions at the edge. | **Deno** runtime |
| **Vector** | Store and similarity-search embeddings for AI/semantic search. | **pgvector** Postgres extension |

Notice the pattern: each service is a thin layer over Postgres, and each one routes authorization back through RLS. There is one source of truth (the database) and one security model (RLS). That coherence is the product.

### 1.4 When to use it — and when not to **[B]**

**Reach for Supabase when:**
- You want a real SQL database with relations, transactions, and constraints (not document soup), but don't want to hand-build auth, storage, and realtime as three separate services.
- You're building a SaaS, dashboard, social app, marketplace, internal tool, or any CRUD-heavy product — the bread-and-butter case.
- You value **not being locked in** — it's open source and self-hostable, and your data is a standard Postgres dump.
- You want to ship a frontend-only MVP fast (the JS client + RLS can replace an entire backend tier) yet keep the door open to add a custom backend (e.g. in Go) later, against the *same* database.

**Be cautious / consider alternatives when:**
- You need a globally-distributed multi-master *write* database. Supabase is single-primary Postgres (plus read replicas). Writes go to one region.
- Your workload is extreme write throughput or append-only telemetry at massive scale — a purpose-built store (ClickHouse, Cassandra, a time-series DB) may fit better.
- You refuse to learn SQL and RLS. You will fight the security model the entire way; the product's value *is* in embracing Postgres.

> **The sweet spot:** Supabase as your Postgres + Auth + Storage backbone, accessed directly from Next.js via the JS client *and* from a Go service via `pgx`/HTTP, all sharing one authorization model. That hybrid is the architecture this guide builds toward.

---

## 2. Getting Started: Project, Keys, CLI, Local Dev

### 2.1 Create a project (hosted) **[B]**

A "project" in Supabase is one isolated Postgres database plus its bundled services, with its own URL and keys. To create one:

1. Sign in at the Supabase dashboard and create an **organization**, then a **project** inside it.
2. Choose a **region** close to your users. Latency to the database is dominated by physical distance, and the region is **fixed after creation** — picking wrong means recreating the project. Choose deliberately.
3. Set a **database password**. This is the password for the `postgres` role used in direct connection strings (the ones your Go backend or migrations use). Store it in a password manager immediately — it is shown once and you need it for `pgx` access.
4. Wait ~2 minutes for provisioning.

### 2.2 The dashboard — what each section is for **[B]**

The dashboard is a Supabase-built UI (called **Studio**) over your project. Knowing where things live saves hours:

| Section | Use |
|---|---|
| **Table Editor** | Spreadsheet-like view to create/edit tables and rows. Fine for exploring; *don't* make production schema changes here untracked (use migrations — §10). |
| **SQL Editor** | Run arbitrary SQL, save snippets, create policies/functions. Your power tool. |
| **Authentication** | Manage users, providers (Google, GitHub…), email templates, MFA, URL config. |
| **Storage** | Create buckets, browse files, set policies. |
| **Database** | Schemas, roles, extensions, replication, **connection pooling** settings, backups. |
| **Edge Functions** | Deploy/inspect Deno functions and their secrets. |
| **Realtime** | Inspect channels and publication settings. |
| **Project Settings → API** | Your **Project URL**, **anon key**, **service_role key**, JWT settings. |
| **Project Settings → Database** | Connection strings (direct + pooler), pooler mode. |
| **Advisors** | Security & Performance linters — flags RLS-disabled tables, missing indexes, exposed functions. Check these regularly. |

### 2.3 Keys: `anon` vs `service_role` (the CRITICAL difference) **[B]**

This subsection is where most security incidents are born, so read it slowly. Every project ships two primary API keys. **Both are JWTs signed by your project**, and the *only* thing that distinguishes them is which Postgres role they map to — and therefore whether RLS applies.

| Key | Role it maps to | Where it may live | RLS |
|---|---|---|---|
| **`anon`** (public/publishable) | Postgres role `anon` (or `authenticated` after the user logs in) | **Safe in the browser / client apps.** It is *meant* to be shipped in your Next.js bundle. | **Subject to RLS.** This is the whole point — RLS protects you even though the key is public. |
| **`service_role`** (secret) | Postgres role `service_role` | **SERVER ONLY. Never in a browser, never in a mobile app, never in a public repo, never in a `NEXT_PUBLIC_*` variable.** | **BYPASSES RLS entirely.** Effectively god-mode over all your data. |

Why is it safe to put the `anon` key in public JavaScript that anyone can read? Because the key only identifies *which project* and *which baseline role* — it does **not** grant access by itself. The moment a request hits Postgres, RLS decides per-row what is allowed. A logged-out visitor is the `anon` role; after login the JWT upgrades them to `authenticated` with their user id embedded. The key is a doorbell, not a master key.

The `service_role` key is the opposite. It maps to a role that is configured to **skip RLS checks completely**, so any query it runs sees and can modify every row in every table.

> **🚨 The single most important security rule in this guide:** the **`service_role` key bypasses Row Level Security**. If it leaks to a client, anyone can read and modify *all* data in *all* tables. Treat it exactly like a database root password. It belongs only in trusted server environments (your Go backend, Edge Functions, server-only Next.js modules) — and even there, prefer the anon key + the user's JWT whenever you can, reaching for `service_role` only for genuinely privileged operations.

> **⚡ Version note:** Supabase has been migrating naming toward **publishable keys** (`sb_publishable_...`, replaces `anon`) and **secret keys** (`sb_secret_...`, replaces `service_role`) alongside the asymmetric-JWT-signing-keys rollout. The *semantics* are identical: one public/RLS-respecting key, one secret/RLS-bypassing key. This guide uses the classic `anon`/`service_role` names since they remain valid and are clearer to reason about.

### 2.4 Connection strings & the pooler **[I]**

Beyond the REST API, you can connect to the Postgres database directly over the wire — that's how a Go server, migrations, or any standard Postgres tool talks to it. From **Project Settings → Database** you get connection strings in three flavors, and the difference is about **connection pooling**, which matters enormously for correctness and scale.

The problem pooling solves: Postgres dedicates real resources to every open connection, so the number of *direct* connections is limited (often a few dozen). A serverless or high-traffic app can blow past that in seconds. **Supavisor** (Supabase's pooler, which replaced PgBouncer) sits in front and multiplexes many client connections onto a smaller set of real server connections.

```
# 1) DIRECT connection (port 5432) — a real Postgres TCP connection to the primary.
#    Best for: long-lived servers, migrations, anything needing session features
#    (LISTEN/NOTIFY, prepared statements, advisory locks). Limited connection count.
postgresql://postgres:[PASSWORD]@db.[PROJECT-REF].supabase.co:5432/postgres

# 2) SUPAVISOR — SESSION mode (port 5432 via the pooler host).
#    Keeps one server connection per client session. Behaves like a direct
#    connection but lets more clients connect. Prepared statements OK.
postgresql://postgres.[PROJECT-REF]:[PASSWORD]@aws-0-[REGION].pooler.supabase.com:5432/postgres

# 3) SUPAVISOR — TRANSACTION mode (port 6543).
#    Pools per-transaction: a server connection is borrowed only for the duration of a
#    transaction, then returned. Ideal for serverless / many short-lived clients.
#    ⚠️ Prepared statements & session state DON'T survive across transactions here.
postgresql://postgres.[PROJECT-REF]:[PASSWORD]@aws-0-[REGION].pooler.supabase.com:6543/postgres
```

| Mode | Port | Best for | Caveat |
|---|---|---|---|
| **Direct** | 5432 | Single long-lived server, migrations | Few connections available (IPv6 by default on hosted) |
| **Session** | 5432 (pooler host) | Persistent servers needing more connections + session features | Holds a connection per client |
| **Transaction** | 6543 | Serverless, edge, high concurrency, short queries | **No prepared-statement caching, no session-level state across queries** |

**How to choose:** a single persistent Go server → **session** (or direct) mode. A serverless function that spins up per-request → **transaction** mode. Migrations → **direct** (they need session features).

> **⚡ Go + pooler gotcha (preview — full detail in §11):** `pgx` and `lib/pq` cache prepared statements by default. Against **transaction-mode** pooling this breaks (`prepared statement "stmtcache_..." already exists`). Fix: disable statement caching (`pgx` simple protocol / `default_query_exec_mode=simple_protocol`) or use **session/direct** mode for a persistent server. This is the #1 surprise for Go developers.

### 2.5 The Supabase CLI & local development **[B]**

Here is a feature that genuinely sets Supabase apart for offline and team work: the CLI runs the **entire Supabase stack locally in Docker** — Postgres, GoTrue, PostgREST, Realtime, Storage, and Studio — so you develop against a real, disposable copy of your backend with no internet and no risk to production. The same client code (only the URL/keys change) runs against local or hosted.

**Why this matters:** you get version-controlled schema (migrations), reproducible environments for every teammate, and a safe place to test destructive changes. It transforms Supabase from "a website you click around in" into a proper engineering workflow.

```bash
# Install (pick one)
npm install -g supabase               # via npm
brew install supabase/tap/supabase    # macOS
scoop install supabase                # Windows (scoop bucket)

supabase --version                    # verify

# Initialize Supabase in your repo (creates ./supabase/ with config.toml, migrations/, etc.)
supabase init

# Start the local stack (requires Docker running). First run pulls images.
supabase start
# → prints local URLs and keys:
#   API URL:        http://127.0.0.1:54321
#   GraphQL URL:    http://127.0.0.1:54321/graphql/v1
#   DB URL:         postgresql://postgres:postgres@127.0.0.1:54322/postgres
#   Studio URL:     http://127.0.0.1:54323
#   anon key:       eyJ...
#   service_role:   eyJ...

supabase status                       # show running services + keys again
supabase stop                         # stop the stack (add --no-backup to discard local data)
```

Local stack key facts to memorize:
- Local **DB** is on port **54322** with user/password `postgres`/`postgres`.
- Local **API** (PostgREST + GoTrue + Storage gateway) is on **54321**.
- Local **Studio** (the dashboard) is on **54323**.
- The local anon/service_role keys are **fixed dev keys** printed by `supabase start`. They only work against your local stack, so they are *not* secret — committing them in `.env.local` for the team is fine.

#### Linking local to a hosted project & pushing schema **[I]**

```bash
supabase login                                  # authenticate the CLI to your account
supabase link --project-ref [PROJECT-REF]       # connect this repo to a remote project

supabase db pull                                # pull remote schema down into a migration
supabase migration new create_profiles          # create a new empty migration to write SQL into
supabase db reset                               # wipe LOCAL db & replay ALL migrations + seed.sql
supabase db push                                # push local migrations UP to the linked remote

# Generate TypeScript types from the (local or remote) schema:
supabase gen types typescript --local > src/types/database.types.ts
supabase gen types typescript --project-id [PROJECT-REF] > src/types/database.types.ts
```

A typical `supabase/` directory:

```
supabase/
├── config.toml            # local stack config (ports, auth settings, etc.)
├── seed.sql               # data inserted on `supabase db reset` (dev seed data)
├── migrations/            # timestamped .sql files = your schema history (commit these!)
│   ├── 20260101120000_create_profiles.sql
│   └── 20260102090000_add_posts.sql
└── functions/             # Edge Functions (Deno)
    └── hello/index.ts
```

> **Best practice:** treat `supabase/migrations/` as the canonical schema. Never click-create tables in production and forget them — make schema changes as migrations so they're reproducible, reviewable, and rollback-friendly. (Full migration workflow in §10.)

---

## 3. The Database & Row Level Security (RLS)

This is **the** section. RLS is the core security model of Supabase, and almost every "Supabase data leak" headline traces back to misunderstanding it. If you internalize one thing from this entire guide, make it this. (For the SQL fundamentals behind these examples — data types, constraints, indexes, joins — see `POSTGRESQL_GUIDE.md` and `RELATIONAL_DB_DESIGN_GUIDE.md`.)

### 3.1 What RLS is and why it exists here **[B]**

**Row Level Security** is a PostgreSQL feature (not a Supabase invention) that lets you attach SQL rules — *policies* — to a table that decide, *per row* and *per current user*, whether a SELECT/INSERT/UPDATE/DELETE is allowed. When RLS is enabled on a table, every query against it is silently rewritten to "…AND (the policy expression is true for this row)." Rows the user isn't allowed to see simply don't exist as far as their query is concerned.

**Why it is the linchpin of Supabase specifically:** because the `anon` key is *public* and the JS client talks to your database directly through PostgREST, you cannot rely on "the client won't send bad requests." Anyone can open dev tools, copy the anon key, and call your API with arbitrary filters — `select * from users` included. The only thing standing between that and a breach is RLS: Postgres itself, not your frontend, decides what each request may touch. In a hand-written backend, *you* are the gate. In Supabase, *the database* is the gate, and RLS is how you configure it.

### 3.2 Enabling RLS — and the deny-all default **[B]**

There are two states that trip people up, so be precise:

- **RLS disabled** = the table is **wide open** to whatever the role's table privileges allow. Via the public API that means the anon key can read/write everything. This is a breach.
- **RLS enabled with no policies** = **deny all**. No rows go in or out for the `anon`/`authenticated` roles. This is *safe* — it just doesn't do anything yet. You then open access deliberately, one policy at a time.

When you create a table through the dashboard, RLS is **enabled by default** (good). When you create one via raw SQL, you must enable it yourself — and forgetting to is the classic mistake.

```sql
-- A table of user profiles, keyed to Supabase's built-in auth.users table.
create table profiles (
  id uuid primary key references auth.users (id) on delete cascade,
  username text unique,
  bio text,
  created_at timestamptz not null default now()
);

-- TURN ON RLS. Until you add policies below, NO ONE (anon/authenticated) can read or write.
alter table profiles enable row level security;
```

> **🚨 Danger:** A table with RLS **disabled** is fully exposed through the public API. **Never disable RLS on a table reachable via the API.** If you genuinely need a table to be server-only, keep RLS *on* with no policies (deny-all) and access it solely through the `service_role` key or `security definer` functions (§3.7).

### 3.3 Anatomy of a policy **[I]**

A policy is a named rule scoped to a table, an operation, and a role, with up to two boolean expressions. Understanding the two expressions — `using` and `with check` — is the whole game.

```sql
create policy "policy name"
  on <table>
  as { permissive | restrictive }       -- default permissive (OR-combined). restrictive is AND-combined.
  for { all | select | insert | update | delete }
  to <role>                             -- e.g. authenticated, anon, public
  using ( <expression> )                -- row visibility: which EXISTING rows the op may touch
  with check ( <expression> );          -- validation for NEW/updated row VALUES (INSERT/UPDATE)
```

The two clauses do two distinct jobs, and conflating them causes subtle bugs:
- **`using`** is checked against rows that *already exist*. For SELECT/UPDATE/DELETE it filters which rows are visible or affected. ("Which rows are you even allowed to look at / target?")
- **`with check`** is checked against the *new values* you're trying to write on INSERT and UPDATE. ("Is the row you're about to create/leave-behind a legal one?") It prevents, for example, inserting a row owned by someone else, or *changing* a row's owner to someone else.

| Operation | Uses `using`? | Uses `with check`? |
|---|---|---|
| SELECT | ✅ | — |
| INSERT | — | ✅ |
| UPDATE | ✅ (which rows) | ✅ (resulting values) |
| DELETE | ✅ | — |

**Permissive vs restrictive:** by default policies are **permissive**, meaning multiple policies for the same operation are combined with **OR** — a row is allowed if *any* policy passes. This lets you layer policies ("owners can read their rows" OR "admins can read all rows"). **Restrictive** policies are combined with **AND** and act as mandatory gates that *every* access must pass (e.g. "and the user's email must be verified"). Use restrictive sparingly, for cross-cutting constraints.

### 3.4 The auth helper functions **[I]**

Inside a policy you need to know *who is asking*. Supabase injects the current request's identity — extracted from the JWT that PostgREST forwarded — and exposes it through three SQL functions:

```sql
auth.uid()    -- the user's UUID (the JWT 'sub' claim), or NULL if not logged in
auth.jwt()    -- the full decoded JWT as jsonb (claims, app_metadata, user_metadata, role)
auth.role()   -- 'anon' or 'authenticated' (the Postgres role for the request)
```

`auth.uid()` is the workhorse — it's how a policy says "this row belongs to the person asking." Reading custom claims out of the JWT lets you do role- and attribute-based rules:

```sql
auth.uid()                                   -- current user id
(auth.jwt() ->> 'email')                     -- email claim (text)
(auth.jwt() -> 'app_metadata' ->> 'role')    -- a custom role stored in app_metadata
(auth.jwt() #>> '{user_metadata,full_name}') -- nested user_metadata value
```

> **🚨 Trust boundary:** `app_metadata` is **set by the server only** (via the Admin API / service_role) and is therefore trustworthy — use it for roles and permissions. `user_metadata` is **user-editable** (the user can set it during sign-up or via `updateUser`). **Never** make an authorization decision based on `user_metadata`; a malicious user can put `"role":"admin"` in it.

### 3.5 Policy recipes — the ones you'll actually write **[I]**

These five patterns cover the vast majority of real apps. Learn them as templates.

**Owner-only (private rows): each user sees and edits only their own.** This is the default for anything personal — todos, notes, settings.

```sql
create table todos (
  id bigint generated always as identity primary key,
  user_id uuid not null default auth.uid() references auth.users(id) on delete cascade,
  task text not null,
  is_done boolean not null default false,
  inserted_at timestamptz not null default now()
);
alter table todos enable row level security;

-- READ: only your own todos
create policy "select own todos"
  on todos for select to authenticated
  using ( (select auth.uid()) = user_id );

-- INSERT: you may only create rows owned by you
create policy "insert own todos"
  on todos for insert to authenticated
  with check ( (select auth.uid()) = user_id );

-- UPDATE: you may change your rows, and may NOT reassign ownership to someone else
create policy "update own todos"
  on todos for update to authenticated
  using      ( (select auth.uid()) = user_id )   -- which rows you can target
  with check ( (select auth.uid()) = user_id );  -- the row must STILL be yours afterward

-- DELETE: only your own
create policy "delete own todos"
  on todos for delete to authenticated
  using ( (select auth.uid()) = user_id );
```

> **⚡ Performance note:** wrap `auth.uid()` in a scalar subquery — `(select auth.uid())` — in policies. Postgres can then evaluate it **once per query** (as an InitPlan) instead of **once per row**. On large tables this is a dramatic speedup and is the officially recommended pattern. The naked `auth.uid() = user_id` form is correct but slow at scale.

**Public read, authenticated write.** Anyone can read; only logged-in users create, and only as themselves.

```sql
create policy "public can read posts"
  on posts for select to anon, authenticated
  using ( true );

create policy "authenticated can insert posts"
  on posts for insert to authenticated
  with check ( (select auth.uid()) = author_id );
```

**Role-based (admin can do anything).** Store the role in `app_metadata` (server-set, trusted) and check it. Because policies are permissive (OR), this *adds* to the owner policies — a normal user matches the owner policy, an admin matches this one.

```sql
create policy "admins manage all posts"
  on posts for all to authenticated
  using      ( (auth.jwt() -> 'app_metadata' ->> 'role') = 'admin' )
  with check ( (auth.jwt() -> 'app_metadata' ->> 'role') = 'admin' );
```

**Team / membership-based (multi-tenant).** Visibility flows through a join table — a user can read a document if they belong to its team.

```sql
create policy "team members read docs"
  on documents for select to authenticated
  using (
    exists (
      select 1 from team_members tm
      where tm.team_id = documents.team_id
        and tm.user_id = (select auth.uid())
    )
  );
```

> For complex membership checks, factor them into a `security definer` helper function (§3.7) and call it from the policy — both for readability and to avoid the policy itself being blocked by RLS on the join table.

**Time/attribute-based (restrictive gate).** A restrictive policy that applies on top of everything — e.g. soft-deleted rows are invisible to all:

```sql
create policy "hide soft-deleted"
  on documents as restrictive for select to authenticated
  using ( deleted_at is null );
```

### 3.6 Testing policies — the discipline that prevents leaks **[I]**

A policy you haven't tested from the *attacker's* seat is a policy you don't trust. Always verify three perspectives:

1. **As anonymous** (logged out) — should sensitive tables return nothing?
2. **As a logged-in non-owner** — can user B read/edit user A's rows? (This is the breach you're hunting for.)
3. **As the owner** — does the legitimate path still work?

You can simulate roles directly in the SQL editor:

```sql
-- Pretend to be the 'authenticated' role with a specific user id, then run a query.
begin;
  set local role authenticated;
  select set_config('request.jwt.claims', '{"sub":"<some-user-uuid>","role":"authenticated"}', true);
  select * from todos;   -- you should only see that user's rows
rollback;
```

### 3.7 `security definer` functions **[A]**

By default a SQL function runs as the **caller** (`security invoker`), so it's still subject to RLS — calling it doesn't grant any extra access. A **`security definer`** function instead runs with the privileges of its **owner/creator**, letting it bypass RLS for a *specific, tightly-controlled* operation. This is the *safe, intentional* way to do something RLS would otherwise block: checking membership without exposing the entire `team_members` table to clients, or performing a privileged write inside an otherwise-locked-down API.

The logic: rather than poking holes in your RLS policies (which weakens them everywhere), you encapsulate the privileged bit behind a function with a narrow, audited interface. The client can call `is_team_member(team_id)` but can't read `team_members` directly.

```sql
-- Returns true if the current user belongs to the given team.
-- SECURITY DEFINER so it can read team_members regardless of that table's RLS,
-- without exposing team_members directly to the client.
create or replace function public.is_team_member(p_team_id uuid)
returns boolean
language sql
security definer
set search_path = ''        -- 🔒 ALWAYS pin search_path on definer functions (prevents hijack)
as $$
  select exists (
    select 1
    from public.team_members tm
    where tm.team_id = p_team_id
      and tm.user_id = (select auth.uid())
  );
$$;

-- Use it inside a policy:
create policy "team members read docs (via fn)"
  on documents for select to authenticated
  using ( public.is_team_member(team_id) );
```

> **🚨 `security definer` rules (each one is a real CVE class):**
> - **Always `set search_path = ''`** (or an explicit, trusted schema). Otherwise a malicious user can create a function/table in their own schema that *shadows* one your function references, and your privileged function will call *their* code. This is the #1 definer vulnerability. With `search_path = ''` you must schema-qualify everything (`public.team_members`).
> - Keep them **small** and validate inputs — they bypass RLS, so any bug is a hole that bypasses your entire security model.
> - **Revoke `execute` from `anon`/`public`** if a function shouldn't be callable by everyone: `revoke execute on function public.some_fn from public, anon;`.

### 3.8 Recommended RLS workflow **[I]**

1. Create table → **enable RLS immediately** (same migration).
2. Add the *minimum* policies needed — start from deny-all and open up deliberately, never the reverse.
3. Test as anonymous, as a logged-in non-owner, and as the owner (§3.6).
4. Use the dashboard's **policy templates** and the **Security Advisor** ("RLS not enabled" / "policy allows public access") as guardrails.
5. **Never** paper over RLS pain by using `service_role` from the client — that's a breach waiting to happen. If RLS is fighting you, the fix is a `security definer` function or an RPC, not the secret key in the browser.

---

## 4. supabase-js Client Deep Dive

`@supabase/supabase-js` (v2) is the universal JS/TS client and the primary way a frontend talks to Supabase. It is a single object that wraps five sub-APIs: PostgREST (the database **query builder**), GoTrue (`.auth`), Realtime (`.channel`), Storage (`.storage`), and Functions (`.functions`). This section covers the query builder in depth; auth is §5, realtime §7, storage §8.

```bash
npm install @supabase/supabase-js
```

```ts
import { createClient } from "@supabase/supabase-js";

// In a plain (non-SSR) context. For Next.js use the @supabase/ssr helpers in §6.
const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,   // https://<ref>.supabase.co
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
);
```

**The cardinal rule of the query builder:** every operation returns a `{ data, error }` object and **does not throw by default**. You must check `error` on every call — ignoring it is the most common source of silent bugs (a failed insert that you never noticed). The builder is also *lazy and chainable*: methods like `.eq()` and `.order()` build up a query, which executes when you `await` it.

### 4.1 Select & the query builder **[B]**

`.select()` is how you read. The string you pass is a PostgREST column projection — far more powerful than it looks, because it can also embed related tables (joins).

```ts
const { data, error } = await supabase.from("posts").select("*");       // all columns
await supabase.from("posts").select("id, title, created_at");           // specific columns
await supabase.from("posts").select("id, headline:title");             // alias a column in the result
await supabase.from("posts").select("id, title::text");                // cast a column
```

#### Nested / foreign-table joins (embedding) **[I]**

This is one of supabase-js's best features: when foreign keys exist between tables, PostgREST can **embed** related rows in a single round-trip — no manual joins, no N+1. The relationship is inferred from the FK; you just name the related table.

```ts
// One-to-one / many-to-one: posts with their author (FK posts.author_id -> profiles.id)
const { data } = await supabase
  .from("posts")
  .select(`
    id,
    title,
    author:profiles ( id, username )      -- embed the related profile, aliased as 'author'
  `);

// One-to-many: a profile and all its posts (returns an array)
await supabase
  .from("profiles")
  .select(`
    username,
    posts ( id, title )                   -- array of related posts
  `);

// Disambiguating MULTIPLE FKs to the same table: name the specific relationship/constraint.
await supabase
  .from("messages")
  .select(`
    body,
    sender:profiles!messages_sender_id_fkey ( username ),
    recipient:profiles!messages_recipient_id_fkey ( username )
  `);

// Inner-join semantics + filtering on the embedded table:
await supabase
  .from("posts")
  .select("title, comments!inner(body)")  // !inner drops posts with no comments
  .eq("comments.is_approved", true);      // filter on the embedded table's column
```

> **RLS still applies to embeds.** An embedded `comments` array only contains comments the current user is allowed to SELECT. The join doesn't bypass policies — each table is filtered independently.

#### Filters & operators **[B]**

Filters narrow the rows. Each maps to a Postgres/PostgREST operator. They chain, and they're combined with AND by default.

```ts
const q = supabase.from("posts").select("*");

q.eq("status", "published");        // =
q.neq("status", "draft");           // !=
q.gt("views", 100);                 // >
q.gte("views", 100);                // >=
q.lt("views", 100);                 // <
q.lte("views", 100);                // <=
q.like("title", "%sql%");           // LIKE (case-sensitive)
q.ilike("title", "%sql%");          // ILIKE (case-insensitive)
q.is("deleted_at", null);           // IS NULL / IS true / IS false (use for null, not .eq)
q.in("category", ["go", "ts"]);     // IN (...)
q.contains("tags", ["postgres"]);   // array/jsonb contains @>
q.containedBy("tags", ["a", "b"]);  // <@
q.overlaps("tags", ["go", "rust"]); // && (arrays overlap)
q.match({ status: "published", featured: true });  // multiple eq at once (AND)

// OR conditions (one string, dot-separated):
q.or("status.eq.published,featured.eq.true");

// Negation:
q.not("status", "eq", "archived");
```

> **Gotcha:** use `.is("col", null)` for null checks, not `.eq("col", null)` — SQL `= NULL` is never true. And `.range(column, lo, hi)` here is a *Postgres range-type* operator, **not** pagination — pagination is `.range(from, to)` on the builder (below). Same name, different methods.

#### Ordering, limiting, pagination **[B]**

```ts
// Order (default ascending)
await supabase.from("posts").select("*").order("created_at", { ascending: false });

// Order by an EMBEDDED table's column
await supabase.from("posts").select("*, profiles(username)")
  .order("username", { referencedTable: "profiles", ascending: true });

// Limit
await supabase.from("posts").select("*").limit(10);

// PAGINATION with .range(from, to) — zero-based, INCLUSIVE on both ends.
//   page size 20, page index `p` (0-based):
const p = 2, size = 20;
await supabase
  .from("posts")
  .select("*")
  .order("created_at", { ascending: false })   // ALWAYS order before paginating, or pages overlap
  .range(p * size, p * size + size - 1);        // rows 40..59
```

> **Always pair pagination with a deterministic `.order()`.** Without an order, the database may return rows in any order, so "page 2" can contain rows you already saw on page 1. For very large offsets, *keyset* pagination (filter on `created_at < lastSeen`) outperforms `.range()`.

#### Single-row helpers **[B]**

```ts
// .single() — expects EXACTLY one row; errors if 0 or >1. data is the object (not an array).
const { data, error } = await supabase
  .from("profiles").select("*").eq("id", id).single();

// .maybeSingle() — 0 or 1 row; returns null data if none (NO error). Use when the row may not exist.
const { data: maybe } = await supabase
  .from("profiles").select("*").eq("id", id).maybeSingle();
```

> Reach for `.maybeSingle()` for "fetch the user's profile if it exists" and `.single()` for "this row must exist (e.g. fetch by primary key after insert)." Using `.single()` where zero rows is legitimate produces spurious errors — a common beginner trap.

### 4.2 Insert / Update / Upsert / Delete **[B]**

Writes mirror the read API. By default a write returns no rows; chain `.select()` to get the affected rows back (useful for getting a generated id).

```ts
// INSERT one or many. .select().single() returns the inserted row.
const { data, error } = await supabase
  .from("posts")
  .insert({ title: "Hello", author_id: userId })
  .select()
  .single();

await supabase.from("posts").insert([{ title: "A" }, { title: "B" }]); // bulk insert

// UPDATE — ALWAYS scope with a filter, or you update every row your RLS allows!
await supabase
  .from("posts")
  .update({ title: "Edited" })
  .eq("id", postId)
  .select();

// UPSERT — insert, or update on conflict of the PK / a unique constraint.
await supabase
  .from("profiles")
  .upsert(
    { id: userId, username: "neo" },
    { onConflict: "id" }            // conflict target; defaults to the primary key
  );

// DELETE — also scope it!
await supabase.from("posts").delete().eq("id", postId);
```

> **🚨 Footgun:** `update(...)` / `delete(...)` **without a filter** affects *every row visible to your RLS policy*. A missing `.eq()` on an admin client (service_role) can wipe a table. Always add a filter, and treat an unfiltered mutation as a bug in code review. RLS limits the blast radius for normal users, but is no help when you're using `service_role`.

### 4.3 RPC — calling Postgres functions **[I]**

`.rpc()` calls a database function you defined in SQL. This is the escape hatch for anything the query builder can't express cleanly: **atomic multi-statement operations** (the function is one transaction), complex queries, aggregations, and `security definer` privileged logic that you don't want to expose as a raw table.

```sql
create or replace function increment_views(p_post_id bigint)
returns void language sql as $$
  update posts set views = views + 1 where id = p_post_id;
$$;

create or replace function search_posts(term text)
returns setof posts language sql stable as $$
  select * from posts where title ilike '%' || term || '%';
$$;
```

```ts
// Call a void function with named args
await supabase.rpc("increment_views", { p_post_id: 42 });

// A set-returning function — you can CHAIN filters/order on the result, like a table!
const { data } = await supabase
  .rpc("search_posts", { term: "postgres" })
  .order("created_at", { ascending: false })
  .limit(5);
```

> **When to prefer RPC over the builder:** anything that must be atomic (e.g. "decrement stock and create order" — do it in one function so a crash can't leave half-done state), anything needing privileged access (`security definer`), and read-heavy logic you want to keep in the database close to the data. Mark read-only functions `stable` and pure ones `immutable` so Postgres can optimize them.

### 4.4 Count **[I]**

```ts
// Get rows AND an exact count in one request
const { data, count } = await supabase
  .from("posts")
  .select("*", { count: "exact" });

// Count ONLY (transfer no rows) — head: true. Great for "total results" on a paginated list.
const { count: total } = await supabase
  .from("posts")
  .select("*", { count: "exact", head: true });

// count options: 'exact' (accurate, scans), 'planned' (estimate from planner), 'estimated'
```

> Use `'exact'` for small/medium tables where an accurate total matters; switch to `'planned'`/`'estimated'` on huge tables where an exact count is too slow and an approximate "about 2M results" is fine.

### 4.5 `.throwOnError()` **[I]**

If you prefer `try/catch` ergonomics (and want errors to bubble up so you can't forget them), chain `.throwOnError()` to convert the `{ error }` return into a thrown exception.

```ts
try {
  const { data } = await supabase.from("posts").select("*").throwOnError();
  return data;
} catch (e) {
  // handle / log — errors can no longer be silently ignored
}
```

### 4.6 Full-text search **[I]**

Postgres has excellent built-in full-text search. The pattern: a generated `tsvector` column plus a GIN index, queried with `.textSearch()`.

```sql
-- A generated tsvector column + GIN index for fast search
alter table posts add column fts tsvector
  generated always as (to_tsvector('english', coalesce(title,'') || ' ' || coalesce(body,''))) stored;
create index posts_fts_idx on posts using gin (fts);
```

```ts
await supabase.from("posts").select("*")
  .textSearch("fts", "postgres & realtime");           // AND of terms

await supabase.from("posts").select("*")
  .textSearch("fts", "'web app'", { type: "phrase" }); // phrase search

await supabase.from("posts").select("*")
  .textSearch("fts", "go | golang", { type: "websearch", config: "english" }); // Google-style query
```

> Use `type: "websearch"` for user-entered search boxes — it tolerates the free-form syntax people actually type. For *semantic* (meaning-based) search rather than keyword matching, see pgvector (§12).

### 4.7 Typed client (preview) **[I]**

After generating types from your schema (§10), parameterize the client with the `Database` type so every query, filter, and result is type-checked end-to-end.

```ts
import { createClient } from "@supabase/supabase-js";
import type { Database } from "@/types/database.types";

const supabase = createClient<Database>(url, anonKey);
// Now .from("posts") knows the columns of posts, and `data` is fully typed.
```

---

## 5. Auth (GoTrue)

Supabase Auth is powered by **GoTrue**, a dedicated Go auth server. It owns the `auth.users` table, issues tokens, and supports a large menu of sign-in methods. Crucially, the token it issues is the *exact same JWT* that RLS reads via `auth.uid()` and that your Go backend will verify (§11) — auth is the thread connecting every part of this guide.

### 5.1 The session / JWT model — understand this first **[B]**

When a user signs in, GoTrue returns a **session** made of two tokens with very different lifetimes and purposes:

- **`access_token`** — a short-lived **JWT** (default ~1 hour). It is *self-contained* and *signed*: it carries the user's identity (`sub` = their UUID) plus `role`, `email`, `app_metadata`, `user_metadata`, `aud`, `exp`. Every request to Supabase sends it as `Authorization: Bearer <token>`, and PostgREST/Realtime/Storage decode it to apply RLS. Because it's signed, anyone (including your Go service) can verify it *without a database lookup* — that's the entire point of JWTs.
- **`refresh_token`** — long-lived and single-use. It is **not** a JWT; it's an opaque token whose only job is "exchange me for a fresh access_token when the old one expires." The client does this automatically just before expiry.

This two-token design balances security and performance: the access token is short-lived (so a stolen one expires fast) and stateless (so verifying it is free), while the refresh token allows long sessions without re-login. The whole machinery is invisible in the browser; the complexity surfaces only in SSR (§6), where you must shuttle these tokens through cookies.

### 5.2 Email / password **[B]**

```ts
// SIGN UP — creates a user. By default sends a confirmation email (email confirm ON).
const { data, error } = await supabase.auth.signUp({
  email: "user@example.com",
  password: "S3curePassw0rd!",
  options: {
    emailRedirectTo: "https://myapp.com/auth/callback", // where the confirm link lands
    data: { full_name: "Ada Lovelace" },                // -> stored in user_metadata
  },
});

// SIGN IN
const { data: signIn, error: e2 } = await supabase.auth.signInWithPassword({
  email: "user@example.com",
  password: "S3curePassw0rd!",
});

// SIGN OUT (clears local session + revokes the refresh token)
await supabase.auth.signOut();
```

> **⚡ Email confirmation gotcha:** if "Confirm email" is enabled (the default), `signUp` creates the user but returns **no session** until they click the email link. Your UI must handle the "check your inbox" state rather than assuming the user is logged in. You can disable confirmation for local dev in `config.toml`. Also: Supabase enforces a configurable minimum password length and (optionally) strength — surface validation errors to users.

### 5.3 Magic links & OTP (passwordless) **[I]**

Passwordless flows remove the password attack surface entirely. A magic link emails a one-click login URL; an OTP emails/SMSes a short code the user types back.

```ts
// Magic link: emails a one-click login link
await supabase.auth.signInWithOtp({
  email: "user@example.com",
  options: { emailRedirectTo: "https://myapp.com/auth/callback" },
});

// Email OTP code (6-digit) instead of a link — then verify it:
await supabase.auth.signInWithOtp({ email, options: { shouldCreateUser: true } });
await supabase.auth.verifyOtp({ email, token: "123456", type: "email" });

// Phone/SMS OTP (needs an SMS provider configured in the dashboard)
await supabase.auth.signInWithOtp({ phone: "+15551234567" });
await supabase.auth.verifyOtp({ phone: "+15551234567", token: "123456", type: "sms" });
```

### 5.4 OAuth providers **[I]**

OAuth lets users sign in with an existing account (Google, GitHub, Apple, Discord, …). You enable and configure the provider's client id/secret in the dashboard; the client just kicks off the redirect.

```ts
await supabase.auth.signInWithOAuth({
  provider: "github",
  options: {
    redirectTo: "https://myapp.com/auth/callback",
    scopes: "read:user",
  },
});
// The user is bounced to the provider, then back to your callback route,
// which exchanges the returned ?code for a session (see §6.6 for the SSR route handler).
```

### 5.5 `getUser()` vs `getSession()` — the safety distinction **[I]**

These two look interchangeable and are *not*. Choosing wrong on the server is a real vulnerability.

```ts
// getSession() — reads the session from local storage / cookies. FAST but UNVERIFIED:
//   it takes the access token at face value without checking it with the auth server.
const { data: { session } } = await supabase.auth.getSession();

// getUser() — sends the access token to GoTrue to VALIDATE it (signature, expiry, revocation)
//   and returns the authenticated user. Slower (a network call) but TRUSTWORTHY.
const { data: { user } } = await supabase.auth.getUser();
```

> **🚨 Server-side rule:** **always use `getUser()` on the server** (Server Components, Route Handlers, middleware) for any authorization decision. `getSession()` merely reads cookies that a malicious client could have tampered with; it does **not** verify the JWT. In the browser, `getSession()` is fine for reading "am I logged in" because the browser is the user's own trusted environment, but the moment a *decision about access* is made server-side, use `getUser()`.

### 5.6 Password reset & email change **[I]**

```ts
// 1) Send the reset email
await supabase.auth.resetPasswordForEmail("user@example.com", {
  redirectTo: "https://myapp.com/account/update-password",
});
// 2) On that page the user is in a temporary recovery session; set the new password:
await supabase.auth.updateUser({ password: "newSecret123!" });

// Change email (sends a confirmation to the NEW address before it takes effect)
await supabase.auth.updateUser({ email: "new@example.com" });
```

### 5.7 User metadata — the trust boundary, again **[I]**

```ts
// user_metadata: user-editable profile-ish data. NEVER for authorization.
await supabase.auth.updateUser({ data: { full_name: "Ada", avatar_url: "..." } });

// app_metadata: server-set, TRUSTED (roles/permissions). Set it only from a trusted context
// (Edge Function / Go backend with service_role via the Admin API) — NOT from the client.
```

```ts
// React to auth changes anywhere in the app (browser). Useful for syncing UI state.
supabase.auth.onAuthStateChange((event, session) => {
  // event: 'SIGNED_IN' | 'SIGNED_OUT' | 'TOKEN_REFRESHED' | 'USER_UPDATED' | 'PASSWORD_RECOVERY'
});
```

### 5.8 Anonymous sign-ins **[I]**

```ts
// Create a throwaway anonymous user — a real auth.users row + JWT, role 'authenticated'.
// Great for "try before you sign up": the user can save data, then convert later.
const { data, error } = await supabase.auth.signInAnonymously();

// Convert later by adding real credentials to the SAME user (data is preserved):
await supabase.auth.updateUser({ email: "real@example.com" });
```

> Anonymous users hit RLS as `authenticated`, so they pass your normal policies. If you need to restrict what un-converted anon users can do, branch on the `is_anonymous` JWT claim in a policy: `using ( (auth.jwt() ->> 'is_anonymous')::boolean is not true )`.

### 5.9 MFA (multi-factor auth) **[A]**

MFA adds a second factor (TOTP authenticator app) on top of the password, raising the *assurance level* of a session from `aal1` to `aal2`.

```ts
// 1) Enroll a TOTP factor → returns a QR code / secret to show the user
const { data: factor } = await supabase.auth.mfa.enroll({ factorType: "totp" });
// factor.totp.qr_code (SVG), factor.id

// 2) Verify enrollment with a code from their authenticator app
const { data: challenge } = await supabase.auth.mfa.challenge({ factorId: factor!.id });
await supabase.auth.mfa.verify({
  factorId: factor!.id,
  challengeId: challenge!.id,
  code: "123456",
});

// 3) On future logins, after the password step, check assurance level & step up:
const { data: aal } = await supabase.auth.mfa.getAuthenticatorAssuranceLevel();
// aal.currentLevel 'aal1' (password only) vs nextLevel 'aal2' (MFA satisfied)
```

> **Enforce MFA in RLS**, not just in the UI, for truly sensitive tables — the UI can be bypassed but the database can't: `using ( (auth.jwt() ->> 'aal') = 'aal2' )`.

### 5.10 Admin API (server-only) **[A]**

With the `service_role` key you can manage users programmatically — create, delete, list, and (importantly) set the trusted `app_metadata`. This is exactly how your Go backend or an Edge Function assigns roles. It is **server-only** because it uses the secret key. (Go equivalent: the GoTrue Admin REST endpoints — §11.3.)

```ts
const admin = createClient(url, SERVICE_ROLE_KEY); // SERVER ONLY — never ship this client to a browser
await admin.auth.admin.updateUserById(userId, { app_metadata: { role: "admin" } });
await admin.auth.admin.createUser({ email, password, email_confirm: true });
await admin.auth.admin.deleteUser(userId);
```

---

## 6. Next.js Integration (App Router) with @supabase/ssr

This is the canonical 2026 setup and the most error-prone part of Supabase + Next.js, so it gets careful treatment. (For the App Router concepts — Server vs Client Components, Server Actions, middleware — see `NEXTJS_16_GUIDE.md`.)

### 6.1 The core SSR challenge **[I]**

In a pure browser app, supabase-js stores the session in `localStorage` and everything just works. Server-side rendering breaks that model: the **server** also needs to know who the user is (to render their dashboard, enforce auth on a protected page), but the server has no `localStorage` — it only sees the HTTP request's **cookies**. And the access token expires every hour, so *something* must refresh it and write the new cookies back, in a way that works across the four Next.js execution contexts: Server Components (read-only cookies), Server Actions, Route Handlers (read/write cookies), and **middleware** (the one place that can reliably refresh on every request).

The `@supabase/ssr` package exists to wire this cookie plumbing correctly. You build three small helper modules (a browser client, a server client, a middleware client) plus a `middleware.ts`. Get these right once and forget about them.

```bash
npm install @supabase/supabase-js @supabase/ssr
```

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://<ref>.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
# NEVER put SERVICE_ROLE here as NEXT_PUBLIC_. If you need it server-side:
SUPABASE_SERVICE_ROLE_KEY=eyJ...   # used only in server-only modules, never imported into client code
```

> **⚡ `@supabase/auth-helpers-nextjs` is deprecated.** Everything below uses `@supabase/ssr`. If you see `createClientComponentClient`/`createServerComponentClient` from auth-helpers in old tutorials, ignore them and use these patterns.

### 6.2 `utils/supabase/client.ts` — the browser client **[I]**

Used inside Client Components (`"use client"`). It reads/writes the session via browser cookies (not localStorage, so the server can see it too).

```ts
// src/utils/supabase/client.ts
import { createBrowserClient } from "@supabase/ssr";

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  );
}
```

### 6.3 `utils/supabase/server.ts` — the server client **[I]**

Used in Server Components, Server Actions, and Route Handlers. It bridges Supabase's auth to Next.js's cookie store via `getAll`/`setAll`.

```ts
// src/utils/supabase/server.ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createClient() {
  // In Next.js 15/16, cookies() is async — await it.
  const cookieStore = await cookies();

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options),
            );
          } catch {
            // Setting cookies from a Server Component is not allowed and throws here.
            // This is SAFE TO IGNORE *because* the middleware (§6.4) refreshes the
            // session and persists cookies on every request.
          }
        },
      },
    },
  );
}
```

### 6.4 Middleware — session refresh (REQUIRED, not optional) **[A]**

The access token expires hourly; **middleware refreshes it on every request** and writes the new cookies so Server Components see a valid session. This is the load-bearing piece — skip it and users get logged out at random and Server Components see stale auth. The ordering rules in the comments are *not* stylistic; violating them causes logout loops that are miserable to debug.

```ts
// src/utils/supabase/middleware.ts
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

export async function updateSession(request: NextRequest) {
  // Start with a pass-through response we can attach refreshed cookies to.
  let supabaseResponse = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet) {
          // Write to BOTH the request (so THIS pass sees them) and the response
          // (so the browser stores them).
          cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value));
          supabaseResponse = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options),
          );
        },
      },
    },
  );

  // 🚨 Do NOT run any code between createServerClient and getUser(). getUser() triggers
  // the token refresh + cookie write; anything in between can corrupt the refresh.
  const { data: { user } } = await supabase.auth.getUser();

  // Optional: redirect unauthenticated users away from protected paths.
  if (
    !user &&
    !request.nextUrl.pathname.startsWith("/login") &&
    !request.nextUrl.pathname.startsWith("/auth")
  ) {
    const url = request.nextUrl.clone();
    url.pathname = "/login";
    return NextResponse.redirect(url);
  }

  // 🚨 Return supabaseResponse as-is (with its cookies). If you build a NEW response,
  // you MUST copy supabaseResponse.cookies onto it or auth silently breaks.
  return supabaseResponse;
}
```

```ts
// src/middleware.ts
import { type NextRequest } from "next/server";
import { updateSession } from "@/utils/supabase/middleware";

export async function middleware(request: NextRequest) {
  return await updateSession(request);
}

export const config = {
  // Run on everything except static assets & images (performance).
  matcher: [
    "/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)",
  ],
};
```

### 6.5 Reading the user in a Server Component **[I]**

```tsx
// src/app/dashboard/page.tsx — a protected Server Component
import { redirect } from "next/navigation";
import { createClient } from "@/utils/supabase/server";

export default async function DashboardPage() {
  const supabase = await createClient();

  // Always getUser() (verified) on the server — never trust getSession() for gating.
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect("/login");

  // RLS automatically scopes this to the logged-in user via the forwarded JWT — no WHERE clause needed.
  const { data: todos } = await supabase
    .from("todos")
    .select("*")
    .order("inserted_at", { ascending: false });

  return (
    <main>
      <h1>Welcome {user.email}</h1>
      <ul>{todos?.map((t) => <li key={t.id}>{t.task}</li>)}</ul>
    </main>
  );
}
```

### 6.6 Server Actions (mutations + auth) **[I]**

Server Actions are the idiomatic way to mutate data from forms in the App Router — no API route needed.

```tsx
// src/app/login/actions.ts
"use server";
import { redirect } from "next/navigation";
import { revalidatePath } from "next/cache";
import { createClient } from "@/utils/supabase/server";

export async function login(formData: FormData) {
  const supabase = await createClient();
  const { error } = await supabase.auth.signInWithPassword({
    email: String(formData.get("email")),
    password: String(formData.get("password")),
  });
  if (error) return redirect("/login?error=" + encodeURIComponent(error.message));
  revalidatePath("/", "layout");
  redirect("/dashboard");
}

export async function signup(formData: FormData) {
  const supabase = await createClient();
  const { error } = await supabase.auth.signUp({
    email: String(formData.get("email")),
    password: String(formData.get("password")),
    options: { emailRedirectTo: `${process.env.NEXT_PUBLIC_SITE_URL}/auth/callback` },
  });
  if (error) return redirect("/login?error=" + encodeURIComponent(error.message));
  redirect("/login?message=Check your email to confirm");
}

export async function addTodo(formData: FormData) {
  const supabase = await createClient();
  // user_id defaults to auth.uid() in the table; RLS enforces ownership.
  const { error } = await supabase.from("todos").insert({ task: String(formData.get("task")) });
  if (error) throw error;
  revalidatePath("/dashboard");
}
```

```tsx
// src/app/login/page.tsx — the form wired to the actions
import { login, signup } from "./actions";

export default function LoginPage() {
  return (
    <form>
      <input name="email" type="email" required />
      <input name="password" type="password" required />
      <button formAction={login}>Log in</button>
      <button formAction={signup}>Sign up</button>
    </form>
  );
}
```

### 6.7 Route Handler: the OAuth / email-confirm callback **[I]**

OAuth and magic links redirect back to your app with a `?code`. This route exchanges that code for a session and writes the auth cookies.

```ts
// src/app/auth/callback/route.ts
import { NextResponse } from "next/server";
import { createClient } from "@/utils/supabase/server";

export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url);
  const code = searchParams.get("code");
  const next = searchParams.get("next") ?? "/dashboard";

  if (code) {
    const supabase = await createClient();
    const { error } = await supabase.auth.exchangeCodeForSession(code);
    if (!error) return NextResponse.redirect(`${origin}${next}`);
  }
  return NextResponse.redirect(`${origin}/login?error=auth`);
}
```

### 6.8 Calling supabase-js from a Client Component **[I]**

```tsx
"use client";
import { useEffect, useState } from "react";
import { createClient } from "@/utils/supabase/client";

export function LiveTodos() {
  const supabase = createClient();
  const [todos, setTodos] = useState<any[]>([]);
  useEffect(() => {
    supabase.from("todos").select("*").then(({ data }) => setTodos(data ?? []));
  }, []);
  return <ul>{todos.map((t) => <li key={t.id}>{t.task}</li>)}</ul>;
}
```

### 6.9 SSR auth pitfalls (read this twice) **[A]**

| Pitfall | Fix |
|---|---|
| Using `getSession()` server-side for gating | Use **`getUser()`** — it verifies with the auth server. |
| No middleware / not calling `getUser()` in it | Tokens never refresh → random logouts & stale auth. The §6.4 middleware is **required**. |
| Code between `createServerClient` and `getUser()` in middleware | Causes hard-to-debug logout loops. Call `getUser()` immediately. |
| Returning a new `NextResponse` from middleware without copying cookies | Refreshed cookies are dropped → user logged out. Return `supabaseResponse`. |
| Trying to `cookies().set()` in a Server Component | Not allowed; the `try/catch` in `server.ts` swallows it, and middleware persists cookies instead. |
| Putting `service_role` in `NEXT_PUBLIC_*` | Catastrophic leak. Service role is server-only and never `NEXT_PUBLIC_`. |
| Forgetting `emailRedirectTo` / Site URL config | Confirmation/OAuth redirects fail. Set Site URL + a redirect allow-list in the dashboard. |

---

## 7. Realtime

Realtime lets clients **subscribe to database changes**, send ephemeral **broadcast** messages, and track **presence** (who's online) — all over WebSockets, organized into named **channels**. It's what powers live feeds, chat, collaborative cursors, and dashboards that update without polling.

There are three distinct features under the "Realtime" umbrella, each for a different job:

| Feature | What | Use for | Stored? |
|---|---|---|---|
| **Postgres Changes** | Stream INSERT/UPDATE/DELETE on a table | Live data that's also persisted (chat messages, orders) | ✅ in the table |
| **Broadcast** | Low-latency pub/sub of arbitrary messages | Cursors, typing indicators, game state, reactions | ❌ ephemeral |
| **Presence** | Track who is currently in a channel | "online now" lists, collaborative editing avatars | ❌ in-memory |

### 7.1 Postgres Changes (listen to inserts/updates/deletes) **[I]**

First the table must be in the realtime **publication** so its changes are streamed via logical replication:

```sql
-- Add a table to the realtime publication
alter publication supabase_realtime add table messages;
-- (Optional) include full OLD-row data on update/delete for richer payloads
alter table messages replica identity full;
```

```ts
"use client";
import { createClient } from "@/utils/supabase/client";

const supabase = createClient();

const channel = supabase
  .channel("room:lobby")                        // any channel name you choose
  .on(
    "postgres_changes",
    { event: "INSERT", schema: "public", table: "messages" },
    (payload) => {
      console.log("new message:", payload.new); // payload.new = the inserted row
    },
  )
  .on(
    "postgres_changes",
    { event: "*", schema: "public", table: "messages", filter: "room_id=eq.42" }, // server-side filter
    (payload) => {
      // payload.eventType: 'INSERT' | 'UPDATE' | 'DELETE'
      // payload.new / payload.old
    },
  )
  .subscribe((status) => {
    if (status === "SUBSCRIBED") console.log("listening");
  });

// Cleanup (e.g. in a useEffect return) — leaking channels leaks WebSocket connections.
supabase.removeChannel(channel);
```

> **🚨 RLS + Realtime:** Postgres Changes **respect RLS** for the `authenticated`/`anon` roles — a client only receives change events for rows it is allowed to **SELECT**. So you need (1) RLS enabled with a SELECT policy *and* (2) the table added to the publication. Miss the policy and clients receive nothing; the security model is the same one as queries, applied to the live stream.

### 7.2 Broadcast (ephemeral pub/sub) **[I]**

Broadcast doesn't touch the database — it's a fast message bus between connected clients. Ideal for things you don't need to store.

```ts
const channel = supabase.channel("game:1", {
  config: { broadcast: { self: true } },   // also receive your OWN broadcasts
});

channel
  .on("broadcast", { event: "cursor" }, ({ payload }) => {
    // payload = whatever a sender sent
  })
  .subscribe();

// Send to everyone on the channel (low-latency, not persisted)
channel.send({ type: "broadcast", event: "cursor", payload: { x: 12, y: 80 } });
```

> You can gate broadcast/presence with **Realtime Authorization** policies on the `realtime.messages` table — the same RLS approach, applied to channel access.

### 7.3 Presence (who's online) **[I]**

```ts
const channel = supabase.channel("room:lobby", {
  config: { presence: { key: userId } },   // a unique key per client
});

channel
  .on("presence", { event: "sync" }, () => {
    const state = channel.presenceState();  // { userId: [{...meta}], ... }
    console.log("online users:", Object.keys(state));
  })
  .on("presence", { event: "join" }, ({ newPresences }) => {/* someone joined */})
  .on("presence", { event: "leave" }, ({ leftPresences }) => {/* someone left */})
  .subscribe(async (status) => {
    if (status === "SUBSCRIBED") {
      await channel.track({ online_at: new Date().toISOString(), name: "Ada" });
    }
  });
```

### 7.4 A live chat (Postgres Changes + insert) **[I]**

Putting it together: load history once, then append new rows as they arrive live.

```tsx
"use client";
import { useEffect, useState } from "react";
import { createClient } from "@/utils/supabase/client";

type Msg = { id: number; body: string; user_id: string; created_at: string };

export function Chat({ roomId }: { roomId: number }) {
  const supabase = createClient();
  const [messages, setMessages] = useState<Msg[]>([]);
  const [text, setText] = useState("");

  useEffect(() => {
    // Initial load (RLS scopes it)
    supabase.from("messages").select("*").eq("room_id", roomId)
      .order("created_at").then(({ data }) => setMessages(data ?? []));

    // Live new messages
    const ch = supabase
      .channel(`room:${roomId}`)
      .on("postgres_changes",
        { event: "INSERT", schema: "public", table: "messages", filter: `room_id=eq.${roomId}` },
        (p) => setMessages((m) => [...m, p.new as Msg]))
      .subscribe();

    return () => { supabase.removeChannel(ch); };
  }, [roomId]);

  async function send() {
    if (!text.trim()) return;
    // user_id defaults to auth.uid(); RLS verifies ownership/room access.
    await supabase.from("messages").insert({ room_id: roomId, body: text });
    setText("");
  }

  return (
    <div>
      <ul>{messages.map((m) => <li key={m.id}>{m.body}</li>)}</ul>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <button onClick={send}>Send</button>
    </div>
  );
}
```

---

## 8. Storage

Storage is S3-style object storage organized into **buckets**, with access governed by the *same* **Postgres RLS** you already learned — files are rows in the `storage.objects` table, and you write policies against it. That unification means you don't learn a second permission system; per-user file isolation is just another RLS policy.

### 8.1 Buckets: public vs private **[I]**

A bucket is a top-level container. Its `public` flag decides the default read behavior:

```ts
// Create buckets (usually once, via dashboard or service_role)
await admin.storage.createBucket("avatars", { public: true });   // public read by URL
await admin.storage.createBucket("invoices", {
  public: false,                                                  // requires auth / signed URLs
  fileSizeLimit: "5MB",
  allowedMimeTypes: ["application/pdf"],
});
```

| Bucket type | Read access | Use for |
|---|---|---|
| **Public** | Anyone with the URL can GET (no auth). RLS still governs writes. | Avatars, public images, marketing assets |
| **Private** | No public URL; need auth + RLS, or a time-limited **signed URL**. | Invoices, user documents, anything sensitive |

> **Security default:** make buckets **private** unless the content is genuinely public. A "public" bucket means a leaked or guessed URL is readable forever by anyone — fine for avatars, a breach for medical records or invoices.

### 8.2 Upload / download / list / remove **[I]**

```ts
const supabase = createClient();

// UPLOAD (path within the bucket). upsert overwrites if it exists.
const { data, error } = await supabase.storage
  .from("avatars")
  .upload(`${userId}/profile.png`, file, {
    cacheControl: "3600",
    upsert: true,
    contentType: "image/png",
  });

// PUBLIC URL (only meaningful for public buckets) — synchronous, no auth check
const { data: pub } = supabase.storage.from("avatars").getPublicUrl(`${userId}/profile.png`);

// SIGNED URL (private buckets) — a time-limited link to a private object
const { data: signed } = await supabase.storage
  .from("invoices")
  .createSignedUrl(`${userId}/2026-06.pdf`, 60 * 60); // valid 1 hour

// DOWNLOAD (returns a Blob)
const { data: blob } = await supabase.storage.from("invoices").download(`${userId}/2026-06.pdf`);

// LIST files under a prefix
const { data: files } = await supabase.storage.from("avatars").list(userId, {
  limit: 100, offset: 0, sortBy: { column: "name", order: "asc" },
});

// REMOVE
await supabase.storage.from("avatars").remove([`${userId}/profile.png`]);
```

> **Signed URLs are your private-file workhorse.** A signed URL embeds a short-lived token, so you can hand a browser a direct link to a private object without exposing credentials. Keep expiry short (minutes to an hour) and generate them on demand — never store long-lived signed URLs.

### 8.3 RLS policies on storage **[I]**

The canonical pattern: store each user's files under a folder named after their user id, then a policy checks that the first path segment matches `auth.uid()`. This gives per-user isolation in one line.

```sql
-- Allow authenticated users to UPLOAD into their own folder in 'avatars'
create policy "users upload own avatar"
  on storage.objects for insert to authenticated
  with check (
    bucket_id = 'avatars'
    and (storage.foldername(name))[1] = (select auth.uid())::text  -- first path segment = user id
  );

-- Allow reading own files (for a private bucket)
create policy "users read own files"
  on storage.objects for select to authenticated
  using (
    bucket_id = 'invoices'
    and (storage.foldername(name))[1] = (select auth.uid())::text
  );

-- Allow updating/deleting own files
create policy "users manage own avatar"
  on storage.objects for update to authenticated
  using ( bucket_id = 'avatars' and (storage.foldername(name))[1] = (select auth.uid())::text );
```

> `storage.foldername(name)` splits the object path into an array of segments; `[1]` is the first folder. Naming uploads `"<uid>/file.ext"` makes per-user isolation trivial — so **always prefix uploads with the user id** or these policies can't work.

### 8.4 Image transformations **[I]**

For images you can request resized/optimized variants on the fly (a Pro feature on hosted), avoiding the need to pre-generate thumbnails.

```ts
const { data } = supabase.storage.from("avatars").getPublicUrl(`${userId}/profile.png`, {
  transform: { width: 200, height: 200, resize: "cover", quality: 80 },
});
// The same transform option works on createSignedUrl({ transform }) and download({ transform }).
```

### 8.5 Resumable uploads (large files) **[I]**

For large files use **resumable uploads** (the TUS protocol) so an interrupted upload can continue rather than restart. The Storage TUS endpoint is `/storage/v1/upload/resumable`, and `tus-js-client` integrates with it. Use this for video / multi-GB files; the plain `upload()` is fine up to the bucket's size limit for ordinary assets.

---

## 9. Edge Functions (Deno)

Edge Functions are serverless TypeScript/JavaScript functions running on the **Deno** runtime, deployed to the edge. Use them whenever you need server-side code that the browser shouldn't run: webhooks (Stripe, GitHub), logic that needs the `service_role` key safely, third-party API calls that hide secrets, scheduled jobs, and custom auth flows.

The mental model: an Edge Function is your "trusted server snippet" inside the Supabase platform. It receives the caller's JWT, can act *as that user* (forwarding the JWT so RLS applies) or *as an admin* (using `service_role` from its secure environment), and has your secrets injected.

### 9.1 Write & run locally **[I]**

```bash
supabase functions new hello       # scaffolds supabase/functions/hello/index.ts
supabase functions serve hello     # run locally (hot reload) at /functions/v1/hello
```

```ts
// supabase/functions/hello/index.ts
// Deno runtime. Imports are by URL or JSR specifier (or via an import map / deno.json).
import { createClient } from "jsr:@supabase/supabase-js@2";

Deno.serve(async (req) => {
  // CORS preflight (functions are often called from browsers)
  if (req.method === "OPTIONS") {
    return new Response("ok", {
      headers: {
        "Access-Control-Allow-Origin": "*",
        "Access-Control-Allow-Headers": "authorization, content-type",
      },
    });
  }

  const { name } = await req.json().catch(() => ({ name: "world" }));

  // Build a client that ACTS AS THE CALLER (forwards their JWT → RLS applies).
  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_ANON_KEY")!,
    { global: { headers: { Authorization: req.headers.get("Authorization")! } } },
  );
  const { data: { user } } = await supabase.auth.getUser();

  // For privileged work, use the service_role (bypasses RLS) — server-only env, safe here:
  // const admin = createClient(Deno.env.get("SUPABASE_URL")!, Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!);

  return new Response(JSON.stringify({ message: `Hello ${name}`, user: user?.id ?? null }), {
    headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" },
  });
});
```

### 9.2 Secrets **[I]**

```bash
# SUPABASE_URL, SUPABASE_ANON_KEY, SUPABASE_SERVICE_ROLE_KEY are injected automatically.
# Add your own secrets:
supabase secrets set STRIPE_SECRET_KEY=sk_live_...
supabase secrets list
# Local: put them in supabase/functions/.env (loaded by `functions serve`)
```

```ts
const stripeKey = Deno.env.get("STRIPE_SECRET_KEY");
```

### 9.3 Deploy & invoke **[I]**

```bash
supabase functions deploy hello                 # deploy to the linked project
supabase functions deploy hello --no-verify-jwt # allow unauthenticated calls (e.g. public webhooks)
```

```ts
// Invoke from the client (auto-attaches the user's JWT)
const { data, error } = await supabase.functions.invoke("hello", {
  body: { name: "Ada" },
});
```

> **⚡ JWT verification:** by default Edge Functions **require a valid Supabase JWT** — the platform checks it before your code runs. For public webhooks (Stripe, GitHub) deploy with `--no-verify-jwt` so unauthenticated providers can call them, and then **verify the provider's own signature yourself** inside the function. Never leave a `--no-verify-jwt` function unprotected.

---

## 10. Database Migrations & Generated Types

### 10.1 Why migrations, and the workflow **[I]**

A migration is a timestamped SQL file in `supabase/migrations/`. Together they are your schema's complete version history — every table, column, policy, function, and index, recorded as code you commit to git. The logic is the same as code migrations anywhere: schema must be **reproducible** (any teammate or CI can build an identical DB from zero), **reviewable** (a colleague can see the exact DDL in a PR), and **ordered** (applied in sequence, so production matches dev). Clicking tables together in the dashboard gives you none of that — it's an untracked snowflake you can't recreate.

```bash
# Create an empty migration and write SQL into it
supabase migration new add_posts_table
# → supabase/migrations/20260621101500_add_posts_table.sql  (edit this file)

# Apply ALL migrations to the LOCAL db from scratch (+ seed.sql). Use during dev.
supabase db reset

# OR capture changes you made in local Studio into a new migration automatically:
supabase db diff -f add_posts_table   # diffs local schema vs migrations → writes a migration

# Push pending migrations to the linked REMOTE project (production)
supabase db push

# Inspect remote migration state
supabase migration list
```

A migration file is just SQL — and notice it includes the RLS setup, because security is part of the schema:

```sql
-- supabase/migrations/20260621101500_add_posts_table.sql
create table public.posts (
  id bigint generated always as identity primary key,
  author_id uuid not null default auth.uid() references auth.users(id) on delete cascade,
  title text not null,
  body text,
  views int not null default 0,
  created_at timestamptz not null default now()
);
alter table public.posts enable row level security;

create policy "public read posts" on public.posts
  for select to anon, authenticated using (true);
create policy "authors write own posts" on public.posts
  for insert to authenticated with check ((select auth.uid()) = author_id);
```

```sql
-- supabase/seed.sql  (runs on `supabase db reset` — dev seed data only)
insert into public.posts (title, body) values ('Hello', 'First post');
```

### 10.2 Generating TypeScript types **[I]**

The CLI can introspect your schema and emit a `Database` type describing every table's Row/Insert/Update shapes. Wire it into the client and you get **end-to-end type safety**: a typo in a column name, a wrong-typed value, or a missing required field becomes a compile error instead of a runtime surprise.

```bash
supabase gen types typescript --local > src/types/database.types.ts                       # from local
supabase gen types typescript --project-id <ref> --schema public > src/types/database.types.ts  # from remote
```

```ts
import type { Database } from "@/types/database.types";

// Handy row/insert/update helpers derived from the generated types:
type Post = Database["public"]["Tables"]["posts"]["Row"];
type PostInsert = Database["public"]["Tables"]["posts"]["Insert"];
type PostUpdate = Database["public"]["Tables"]["posts"]["Update"];
```

```ts
// Wire it into the SSR clients so EVERY query is typed.
import { createBrowserClient } from "@supabase/ssr";
import type { Database } from "@/types/database.types";

export const createClient = () =>
  createBrowserClient<Database>(url, anonKey); // .from('posts') is now fully typed
```

> **Best practice:** regenerate types whenever the schema changes (add a `package.json` script and/or a CI check). Stale types silently lie to you — they'll claim a dropped column still exists and let you ship code that fails at runtime.

---

## 11. Golang Integration (Direct DB, JWT, REST, RLS)

Supabase's Go story is under-documented because the marketing pushes the JS client, but a Go service is a first-class citizen against the same Postgres + Auth. This section is the guide's signature material: how to make a Go backend coexist with a Supabase project under **one shared authorization model**. (For Go language depth see `GO_GUIDE.md`; for the JWT/crypto background see `GO_JWT_ARGON2_GUIDE.md`.)

There are **four** realistic approaches; production apps usually combine (a) + (b):

| Approach | What | When |
|---|---|---|
| **(a) Direct Postgres** via `pgx` | Connect straight to the DB, write SQL, bypass PostgREST | Custom Go API, heavy/complex queries, background jobs, performance |
| **(b) Verify Supabase JWTs** | Validate the access token Next.js/clients send | Any Go endpoint that must know *who* is calling |
| **(c) Call PostgREST / Storage over HTTP** | Use the REST API with service_role or the user's JWT | Reuse RLS/PostgREST logic; simple CRUD without writing SQL |
| **(d) Respect RLS from Go** | Pass the user's JWT so Postgres enforces policies | One authorization model across JS + Go |

### 11.1 (a) Connecting directly with `pgx` **[A]**

`pgx` is the de-facto Postgres driver for Go. Use a connection *pool* (`pgxpool`) for a server — it manages a set of reusable connections so you don't pay connection setup cost per request.

```bash
go get github.com/jackc/pgx/v5
go get github.com/jackc/pgx/v5/pgxpool
```

```go
package db

import (
	"context"
	"fmt"
	"os"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
)

// NewPool creates a connection pool to Supabase Postgres.
//
// Use the SESSION-mode pooler (port 5432 pooler host) or DIRECT (5432) for a
// long-lived Go server. If you must use TRANSACTION mode (6543), you MUST disable
// prepared-statement caching (see the note below) or pgx errors with
// "prepared statement already exists".
func NewPool(ctx context.Context) (*pgxpool.Pool, error) {
	// Example (session/direct). Keep the password in an env/secret, never in code.
	//   postgresql://postgres.<ref>:<pwd>@aws-0-<region>.pooler.supabase.com:5432/postgres
	connStr := os.Getenv("SUPABASE_DB_URL")

	cfg, err := pgxpool.ParseConfig(connStr)
	if err != nil {
		return nil, fmt.Errorf("parse config: %w", err)
	}

	cfg.MaxConns = 10
	cfg.MaxConnIdleTime = 5 * time.Minute
	cfg.MaxConnLifetime = 30 * time.Minute

	// ── If using the TRANSACTION pooler (port 6543), uncomment to avoid the
	//    prepared-statement cache incompatibility:
	// cfg.ConnConfig.DefaultQueryExecMode = pgx.QueryExecModeSimpleProtocol
	//    (import "github.com/jackc/pgx/v5"). Or append
	//    "?default_query_exec_mode=simple_protocol" to the connection string.

	pool, err := pgxpool.NewWithConfig(ctx, cfg)
	if err != nil {
		return nil, fmt.Errorf("create pool: %w", err)
	}
	if err := pool.Ping(ctx); err != nil {
		return nil, fmt.Errorf("ping: %w", err)
	}
	return pool, nil
}
```

```go
// Example queries with pgx. IMPORTANT: a direct connection uses the `postgres` role,
// which BYPASSES RLS (like service_role). Either enforce authorization in your Go code,
// or set the role/JWT per request to honor RLS (see 11.4).
package store

import (
	"context"

	"github.com/jackc/pgx/v5/pgxpool"
)

type Post struct {
	ID    int64
	Title string
	Body  string
}

func ListPosts(ctx context.Context, pool *pgxpool.Pool) ([]Post, error) {
	rows, err := pool.Query(ctx, `select id, title, coalesce(body,'') from posts order by created_at desc`)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var out []Post
	for rows.Next() {
		var p Post
		if err := rows.Scan(&p.ID, &p.Title, &p.Body); err != nil {
			return nil, err
		}
		out = append(out, p)
	}
	return out, rows.Err()
}

func CreatePost(ctx context.Context, pool *pgxpool.Pool, authorID, title, body string) (int64, error) {
	var id int64
	// Always use parameterized queries ($1, $2…) — never string-concat (SQL injection).
	err := pool.QueryRow(ctx,
		`insert into posts (author_id, title, body) values ($1, $2, $3) returning id`,
		authorID, title, body,
	).Scan(&id)
	return id, err
}
```

> **🚨 Direct DB role bypasses RLS.** When you connect as `postgres` (the default for the connection string), **RLS does not apply** — you are a superuser-ish role, like `service_role`. That's powerful and dangerous: your Go code is now *solely* responsible for authorization, unless you explicitly run as the user (11.4). Treat every direct-connection handler as if it holds the master key.

### 11.2 (b) Verifying Supabase Auth JWTs in Go **[A]**

When your Next.js app (or any client) calls your Go API, it sends `Authorization: Bearer <access_token>`. Your Go code **verifies** that token to learn the user id (`sub`) and claims, *without* a database round-trip — that's the beauty of a signed JWT. There are two signing schemes in 2026:

| Scheme | Algorithm | How to verify | When |
|---|---|---|---|
| **Legacy / shared secret** | **HS256** | Use the project's **JWT Secret** (Settings → API) as the HMAC key | Older projects, or those still on symmetric signing |
| **Asymmetric signing keys** | **RS256 / ES256** | Fetch the project's **JWKS** (`<url>/auth/v1/.well-known/jwks.json`) and verify with the public key | Newer projects (2025+); verify without sharing a secret |

The asymmetric path is strictly better when available: the *public* key verifies tokens, so your Go service never holds a secret that could leak, and key rotation is automatic via JWKS.

```bash
go get github.com/golang-jwt/jwt/v5
```

#### Option 1 — HS256 (shared JWT secret)

```go
package auth

import (
	"errors"
	"fmt"
	"os"

	"github.com/golang-jwt/jwt/v5"
)

// SupabaseClaims models the claims Supabase puts in the access token.
type SupabaseClaims struct {
	Email                string                 `json:"email"`
	Role                 string                 `json:"role"` // 'authenticated' / 'anon'
	AppMetadata          map[string]interface{} `json:"app_metadata"`
	jwt.RegisteredClaims                        // includes Subject (sub), ExpiresAt, Audience…
}

// VerifyHS256 validates a Supabase access token signed with the project's JWT secret.
func VerifyHS256(tokenString string) (*SupabaseClaims, error) {
	secret := os.Getenv("SUPABASE_JWT_SECRET") // from Settings → API → JWT Secret

	claims := &SupabaseClaims{}
	token, err := jwt.ParseWithClaims(tokenString, claims, func(t *jwt.Token) (interface{}, error) {
		// CRITICAL: pin the algorithm. Reject anything that isn't HMAC to prevent
		// the classic "alg=none" / algorithm-confusion attack.
		if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
		}
		return []byte(secret), nil
	},
		// Supabase access tokens have aud = "authenticated"
		jwt.WithAudience("authenticated"),
		jwt.WithValidMethods([]string{"HS256"}),
	)
	if err != nil {
		return nil, err
	}
	if !token.Valid {
		return nil, errors.New("invalid token")
	}
	return claims, nil
}
```

#### Option 2 — RS256/ES256 via JWKS (asymmetric, 2026)

```bash
go get github.com/MicahParks/keyfunc/v3   # fetches & caches the JWKS, gives jwt a Keyfunc
```

```go
package auth

import (
	"context"
	"errors"

	"github.com/MicahParks/keyfunc/v3"
	"github.com/golang-jwt/jwt/v5"
)

// NewJWKS is created once at startup; it auto-refreshes the public keys in the background.
//   url = https://<ref>.supabase.co/auth/v1/.well-known/jwks.json
func NewJWKS(ctx context.Context, jwksURL string) (keyfunc.Keyfunc, error) {
	return keyfunc.NewDefaultCtx(ctx, []string{jwksURL})
}

// VerifyJWKS validates an asymmetrically-signed Supabase token using the JWKS public keys.
func VerifyJWKS(tokenString string, jwks keyfunc.Keyfunc) (*SupabaseClaims, error) {
	claims := &SupabaseClaims{}
	token, err := jwt.ParseWithClaims(
		tokenString, claims, jwks.Keyfunc,
		jwt.WithAudience("authenticated"),
		// Accept only the asymmetric algorithms Supabase may use.
		jwt.WithValidMethods([]string{"RS256", "ES256"}),
	)
	if err != nil {
		return nil, err
	}
	if !token.Valid {
		return nil, errors.New("invalid token")
	}
	return claims, nil
}
```

> **⚡ Which one?** Check **Settings → API / JWT Keys** in your dashboard. A single "JWT Secret" → HS256. Asymmetric **signing keys** with a JWKS endpoint → prefer JWKS verification (no secret to leak; rotation is automatic). A robust backend can try JWKS first and fall back to HS256 during a migration window.

#### Auth middleware (net/http) extracting `sub` & claims

```go
package auth

import (
	"context"
	"net/http"
	"strings"
)

type ctxKey string

const userIDKey ctxKey = "userID"
const claimsKey ctxKey = "claims"

// Middleware verifies the bearer token and injects the user id + claims into context.
func Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// 1) Pull "Authorization: Bearer <token>"
		authHeader := r.Header.Get("Authorization")
		if authHeader == "" || !strings.HasPrefix(authHeader, "Bearer ") {
			http.Error(w, "missing bearer token", http.StatusUnauthorized)
			return
		}
		tokenStr := strings.TrimPrefix(authHeader, "Bearer ")

		// 2) Verify (pick HS256 or JWKS per your project)
		claims, err := VerifyHS256(tokenStr)
		if err != nil {
			http.Error(w, "invalid token: "+err.Error(), http.StatusUnauthorized)
			return
		}

		// 3) sub = the user's UUID. This is the value RLS's auth.uid() would return.
		userID := claims.Subject
		if userID == "" {
			http.Error(w, "token has no subject", http.StatusUnauthorized)
			return
		}

		// 4) Stash for handlers
		ctx := context.WithValue(r.Context(), userIDKey, userID)
		ctx = context.WithValue(ctx, claimsKey, claims)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

// Helpers for handlers
func UserID(ctx context.Context) (string, bool) {
	v, ok := ctx.Value(userIDKey).(string)
	return v, ok
}
func Claims(ctx context.Context) (*SupabaseClaims, bool) {
	v, ok := ctx.Value(claimsKey).(*SupabaseClaims)
	return v, ok
}

// Example role check using app_metadata.role (server-set, trusted)
func RequireAdmin(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		c, ok := Claims(r.Context())
		if !ok || c.AppMetadata["role"] != "admin" {
			http.Error(w, "forbidden", http.StatusForbidden)
			return
		}
		next.ServeHTTP(w, r)
	})
}
```

#### Wiring it together (handler that uses the verified user id with pgx)

```go
package main

import (
	"context"
	"encoding/json"
	"log"
	"net/http"

	"yourapp/auth"
	"yourapp/db"
	"yourapp/store"
)

func main() {
	ctx := context.Background()
	pool, err := db.NewPool(ctx)
	if err != nil {
		log.Fatal(err)
	}
	defer pool.Close()

	mux := http.NewServeMux()

	// GET /posts — public list (no auth required here)
	mux.HandleFunc("GET /posts", func(w http.ResponseWriter, r *http.Request) {
		posts, err := store.ListPosts(r.Context(), pool)
		if err != nil {
			http.Error(w, err.Error(), 500)
			return
		}
		json.NewEncoder(w).Encode(posts)
	})

	// POST /posts — requires auth; author is the verified user id from the JWT.
	createHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		userID, _ := auth.UserID(r.Context()) // guaranteed by middleware
		var body struct{ Title, Body string }
		json.NewDecoder(r.Body).Decode(&body)

		id, err := store.CreatePost(r.Context(), pool, userID, body.Title, body.Body)
		if err != nil {
			http.Error(w, err.Error(), 500)
			return
		}
		w.WriteHeader(http.StatusCreated)
		json.NewEncoder(w).Encode(map[string]int64{"id": id})
	})
	mux.Handle("POST /posts", auth.Middleware(createHandler))

	log.Println("listening on :8080")
	http.ListenAndServe(":8080", mux)
}
```

### 11.3 (c) Calling PostgREST / Storage / Auth over HTTP from Go **[A]**

Sometimes you'd rather reuse PostgREST's filtering and RLS than hand-write SQL. Hit the REST API directly with `net/http`.

```go
package rest

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
)

// GetPosts calls PostgREST. Passing the USER's JWT means RLS applies as that user.
// Passing the service_role key instead bypasses RLS (server-only, use sparingly).
func GetPosts(ctx context.Context, userJWT string) ([]map[string]any, error) {
	base := os.Getenv("SUPABASE_URL") // https://<ref>.supabase.co
	anon := os.Getenv("SUPABASE_ANON_KEY")

	// PostgREST endpoint: /rest/v1/<table>?<filters>
	url := base + "/rest/v1/posts?select=id,title&order=created_at.desc&limit=20"
	req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)

	// apikey = the anon (or service_role) key. Authorization = the per-user JWT for RLS.
	req.Header.Set("apikey", anon)
	req.Header.Set("Authorization", "Bearer "+userJWT) // use the anon key here too if no user
	req.Header.Set("Accept", "application/json")

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()
	if resp.StatusCode >= 300 {
		b, _ := io.ReadAll(resp.Body)
		return nil, fmt.Errorf("postgrest %d: %s", resp.StatusCode, b)
	}
	var out []map[string]any
	return out, json.NewDecoder(resp.Body).Decode(&out)
}
```

```go
// Storage over HTTP (download a private object with service_role):
// GET /storage/v1/object/<bucket>/<path>          (authenticated/service_role)
// GET /storage/v1/object/public/<bucket>/<path>   (public buckets, no auth)
func DownloadObject(ctx context.Context, bucket, path string) ([]byte, error) {
	base := os.Getenv("SUPABASE_URL")
	key := os.Getenv("SUPABASE_SERVICE_ROLE_KEY") // server-only
	url := fmt.Sprintf("%s/storage/v1/object/%s/%s", base, bucket, path)

	req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	req.Header.Set("apikey", key)
	req.Header.Set("Authorization", "Bearer "+key)

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()
	return io.ReadAll(resp.Body)
}
```

```go
// Auth Admin API over HTTP (assign a role via app_metadata, server-only):
// PUT /auth/v1/admin/users/<id>  with service_role — sets trusted app_metadata.
func SetUserRole(ctx context.Context, userID, role string) error {
	base := os.Getenv("SUPABASE_URL")
	key := os.Getenv("SUPABASE_SERVICE_ROLE_KEY")
	payload, _ := json.Marshal(map[string]any{"app_metadata": map[string]any{"role": role}})

	url := fmt.Sprintf("%s/auth/v1/admin/users/%s", base, userID)
	req, _ := http.NewRequestWithContext(ctx, http.MethodPut, url, bytes.NewReader(payload)) // import "bytes"
	req.Header.Set("apikey", key)
	req.Header.Set("Authorization", "Bearer "+key)
	req.Header.Set("Content-Type", "application/json")

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	if resp.StatusCode >= 300 {
		b, _ := io.ReadAll(resp.Body)
		return fmt.Errorf("admin api %d: %s", resp.StatusCode, b)
	}
	return nil
}
```

### 11.4 (d) Respecting RLS from Go (running queries AS the user) **[A]**

This is the elegant move that unifies the architecture. If you want Postgres — not your Go code — to enforce authorization even over a direct `pgx` connection, run each request inside a transaction that **assumes the `authenticated` role and sets the JWT claims**, exactly mimicking what PostgREST does internally. Then `auth.uid()` resolves and your existing RLS policies apply automatically. You write security *once* (in RLS) and both JS and Go honor it.

```go
package db

import (
	"context"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"
)

// WithUserRLS runs fn inside a tx where Postgres sees the request as the given user,
// so RLS policies (auth.uid(), auth.jwt()) are enforced — just like via PostgREST.
func WithUserRLS(ctx context.Context, pool *pgxpool.Pool, jwtClaimsJSON string, fn func(pgx.Tx) error) error {
	tx, err := pool.Begin(ctx)
	if err != nil {
		return err
	}
	defer tx.Rollback(ctx)

	// 1) Become the limited 'authenticated' role (RLS now applies to this tx).
	if _, err := tx.Exec(ctx, `set local role authenticated`); err != nil {
		return err
	}
	// 2) Provide the JWT claims so auth.uid()/auth.jwt() resolve. request.jwt.claims is
	//    the GUC PostgREST sets; auth.uid() reads 'sub' out of it.
	if _, err := tx.Exec(ctx, `select set_config('request.jwt.claims', $1, true)`, jwtClaimsJSON); err != nil {
		return err
	}

	if err := fn(tx); err != nil {
		return err
	}
	return tx.Commit(ctx)
}
```

```go
// Usage: the claims JSON must contain at least {"sub":"<user-uuid>","role":"authenticated"}.
claimsJSON := fmt.Sprintf(`{"sub":%q,"role":"authenticated"}`, userID)
err := WithUserRLS(ctx, pool, claimsJSON, func(tx pgx.Tx) error {
	// This SELECT is now filtered by the table's RLS policies for that user.
	rows, err := tx.Query(ctx, `select id, task from todos order by inserted_at desc`)
	// ... scan ...
	return err
})
```

> **Why `set local`?** It scopes the role/config to the current transaction only, so when the connection returns to the pool it reverts cleanly — no leakage of one user's identity onto the next request. This is the safe way to honor RLS over a shared pool.

### 11.5 When to use the JS client vs direct DB from Go **[A]**

| Situation | Recommendation |
|---|---|
| Next.js frontend, simple CRUD, want RLS for free | **supabase-js** (browser/SSR). No backend needed. |
| Complex queries, transactions, joins, batch jobs, performance | **Go + `pgx` direct** (write SQL). |
| Go API that must authenticate users | **Verify the JWT** (11.2) and trust `sub`. |
| Want one authorization model (RLS) enforced even in Go | **Direct `pgx` + run-as-user tx** (11.4) or **call PostgREST with the user JWT** (11.3c). |
| Privileged admin operations from Go | **service_role** via the Admin API / direct DB (server-only, audited). |
| Need to call from Go but don't want to write SQL | **PostgREST over HTTP** (11.3c). |

> **Common production shape:** Next.js uses supabase-js for the bulk of UI CRUD (RLS does the security), while a Go service handles heavy lifting — background processing, integrations, complex transactional logic — by connecting with `pgx`, verifying user JWTs for any user-facing endpoint, and using `service_role`/direct DB only for trusted operations. The shared RLS model keeps both paths consistent.

---

## 12. pgvector / AI: Embeddings & Semantic Search

Supabase ships **pgvector**, the Postgres extension that turns your database into a vector store for **semantic search** and **RAG** (retrieval-augmented generation). The idea: an embedding model converts text (or images) into a high-dimensional vector where *semantically similar* content lands close together; you store those vectors in a column and query "find the nearest vectors to this one" to retrieve by *meaning* rather than keywords. In 2026 this is how you build "search that understands intent" and how you feed relevant context to an LLM.

```sql
-- Enable the extension
create extension if not exists vector;

-- A documents table with an embedding column.
-- The dimension MUST match your embedding model exactly (e.g. 1536, or 768/1024 for others).
create table documents (
  id bigint generated always as identity primary key,
  content text not null,
  embedding vector(1536),
  owner_id uuid not null default auth.uid() references auth.users(id) on delete cascade,
  created_at timestamptz not null default now()
);
alter table documents enable row level security;
create policy "owner rw documents" on documents
  for all to authenticated
  using ((select auth.uid()) = owner_id)
  with check ((select auth.uid()) = owner_id);

-- Index for fast approximate nearest-neighbor (ANN) search.
-- HNSW (recommended) with cosine distance:
create index on documents using hnsw (embedding vector_cosine_ops);
-- Alternative: IVFFlat — using ivfflat (embedding vector_cosine_ops) with (lists = 100);
```

The distance operators (pick the one matching how your model was trained): `<->` (L2/Euclidean), `<#>` (negative inner product), `<=>` (cosine distance). Most text-embedding models use **cosine**.

A function to find the most similar documents — note RLS still applies, so users only ever match their own documents:

```sql
create or replace function match_documents(
  query_embedding vector(1536),
  match_threshold float,   -- e.g. 0.78 cosine similarity
  match_count int          -- top N
)
returns table (id bigint, content text, similarity float)
language sql stable
as $$
  select
    d.id,
    d.content,
    1 - (d.embedding <=> query_embedding) as similarity  -- cosine similarity = 1 - distance
  from documents d
  where 1 - (d.embedding <=> query_embedding) > match_threshold
  order by d.embedding <=> query_embedding   -- nearest first (smallest distance)
  limit match_count;
$$;
```

```ts
// 1) Embed the user's query with your embedding model (server-side) → number[]
// 2) Call the RPC. RLS still applies, so users only match their own documents.
const { data: matches } = await supabase.rpc("match_documents", {
  query_embedding: queryEmbedding,   // number[1536]
  match_threshold: 0.78,
  match_count: 5,
});
```

```ts
// Inserting an embedding (from a server context where you computed it):
await supabase.from("documents").insert({
  content: text,
  embedding: embedding, // number[] — supabase-js serializes to a pgvector literal
});
```

> **Tips:** match the `vector(N)` dimension to your model exactly (a mismatch errors); normalize vectors if your model expects it; prefer **HNSW** for most workloads (better recall/speed, no training step); and remember you can combine vector search with ordinary SQL filters *and* RLS in the same query — that's "hybrid search," and it's a key advantage of doing vectors *in* your relational database rather than a separate vector DB.

---

## 13. Security Best Practices

A consolidated checklist — most Supabase breaches come from violating one of these. Re-read it before every launch.

1. **Never expose `service_role` to a client.** Not in the browser, not in mobile apps, not in `NEXT_PUBLIC_*`, not in a public repo. It bypasses RLS = full data access. Server-only.
2. **Enable RLS on every table reachable via the API**, and write explicit policies. RLS off = wide open. RLS on with no policies = safe deny-all.
3. **Use `getUser()` (not `getSession()`) for server-side authorization.** `getSession()` is unverified cookie data.
4. **Verify JWTs in your Go backend** — pin the algorithm (`WithValidMethods`), check `aud="authenticated"`, never accept `alg=none`. Prefer JWKS (asymmetric) where available.
5. **`security definer` functions: always `set search_path = ''`** and keep them minimal; revoke `execute` from `public`/`anon` if not needed.
6. **Put authorization data in `app_metadata` (server-set), never `user_metadata`** (user-editable). Don't trust `user_metadata` in policies.
7. **Scope every `update`/`delete`** with a filter; treat an unfiltered mutation as a bug.
8. **Lock down Auth redirect URLs** (Site URL + allow-list) to prevent open-redirect / token theft.
9. **Manage secrets properly** — Edge Function secrets via `supabase secrets`, Go via env/secret manager. Rotate the DB password and keys if leaked.
10. **Storage: private buckets + per-folder RLS + signed URLs** for anything sensitive; never dump private files in public buckets.
11. **Rate-limit and validate** in Edge Functions / Go for public endpoints; verify provider signatures for webhooks deployed with `--no-verify-jwt`.
12. **Run the dashboard's Security Advisor** — it flags tables with RLS disabled, exposed definer functions, and missing policies. Make it part of your release checklist.

---

## 14. Self-Hosting & Pricing/Limits Awareness

### 14.1 Self-hosting **[I]**

Because Supabase is **open source**, you can run the entire stack yourself — useful for data-residency/compliance, cost at very large scale, or air-gapped environments. The standard path is **Docker Compose** (see `DOCKER_GUIDE.md` for the container fundamentals).

```bash
git clone --depth 1 https://github.com/supabase/supabase
cd supabase/docker
cp .env.example .env        # set POSTGRES_PASSWORD, JWT_SECRET, ANON_KEY, SERVICE_ROLE_KEY, SITE_URL...
docker compose up -d        # starts Postgres, GoTrue, PostgREST, Realtime, Storage, Studio, Kong...
```

What you run yourself: **Kong** (API gateway), **GoTrue**, **PostgREST**, **Realtime**, **Storage API**, **Studio**, **Postgres** (+ `pg_meta`), and a logging/analytics stack. You generate your own `JWT_SECRET` and derive matching `ANON_KEY`/`SERVICE_ROLE_KEY` JWTs from it. The trade-off is total control and no usage fees, but you now own upgrades, backups, scaling, HA, and security patching.

> For most teams, hosted is far less operational burden, and the same client code works against either — only the URL/keys change. Self-host when a hard requirement (compliance, air-gap, scale economics) forces it.

### 14.2 Pricing & limits awareness (concepts, not exact figures) **[B]**

Plans roughly: **Free** (great for dev/small projects; idle projects may be paused), **Pro** (production baseline + higher limits + daily backups + add-ons), **Team** (collaboration/compliance needs), **Enterprise** (custom). What consumes quota / drives cost:

| Dimension | Notes |
|---|---|
| **Database size / disk** | Grows with data; disk is provisioned/auto-scaled on paid plans. |
| **Egress / bandwidth** | API + Storage + Realtime data transfer — watch this on media-heavy apps. |
| **Storage** | GB stored + image transformations. |
| **Monthly active users (MAU)** | The Auth pricing dimension. |
| **Edge Function invocations** | Per-invocation / compute. |
| **Realtime** | Concurrent connections + messages. |
| **Compute add-ons** | Bigger instances, read replicas, dedicated pooler. |

> **⚡ Always check the current pricing page for exact numbers** — tiers and limits change. Actionable takeaways: enable backups on production, watch egress on media-heavy apps, and remember Free-tier projects can be paused when idle.

---

## 15. Gotchas & Best Practices

| Gotcha | What happens / Fix |
|---|---|
| **Forgot to enable RLS** | Table fully exposed via the anon key. Enable RLS on every API-reachable table. |
| **RLS on but no policies** | Nothing works (deny-all) — *this is correct/safe*; add policies deliberately. |
| **`auth.uid()` per-row in policies** | Slow on big tables. Wrap as `(select auth.uid())` so it evaluates once. |
| **`getSession()` on the server** | Unverified — use `getUser()` for any auth decision server-side. |
| **No middleware in Next.js** | Sessions don't refresh → random logouts. The §6.4 middleware is mandatory. |
| **New `NextResponse` in middleware** | Drops refreshed cookies → logout. Return the `supabaseResponse` (or copy cookies). |
| **`service_role` in `NEXT_PUBLIC_`** | Full data breach. Service role is server-only, never public. |
| **Transaction-pooler (6543) + prepared statements** | `pgx`/`lib/pq` error. Use simple protocol / disable statement cache, or use session/direct mode. |
| **Direct `pgx` connection ignores RLS** | The `postgres` role bypasses RLS. Authorize in Go or run as user (11.4). |
| **Unfiltered `update`/`delete`** | Mutates all RLS-visible rows. Always add a filter. |
| **`.single()` on 0 or many rows** | Errors. Use `.maybeSingle()` when the row may be absent. |
| **Realtime table not in publication** | No events. `alter publication supabase_realtime add table ...`. |
| **Realtime without a SELECT policy** | Clients receive no change events (RLS blocks them). Add a SELECT policy. |
| **`security definer` without `set search_path`** | Function-hijack vulnerability. Always pin `search_path = ''`. |
| **`alg=none` / algorithm confusion in Go JWT** | Always pass `jwt.WithValidMethods([...])` and check the method type. |
| **Trusting `user_metadata` for roles** | User can edit it. Use server-set `app_metadata`. |
| **Wrong `vector(N)` dimension** | Inserts/queries fail or mismatch. Match the embedding model exactly. |
| **Stale generated types** | Types lie after schema changes. Regenerate on every migration. |
| **Email confirm enabled but unhandled** | `signUp` returns no session; show "check your email". |
| **Storage path not user-prefixed** | Per-user RLS via `storage.foldername(name)[1]` won't work. Name files `"<uid>/..."`. |
| **Missing Auth redirect allow-list** | OAuth/magic-link redirects fail or are unsafe. Configure Site URL + allowed redirects. |
| **Pagination without `.order()`** | Pages overlap/skip rows. Always order before `.range()`. |
| **Ignoring `{ error }`** | Silent failures. Check `error` on every call or use `.throwOnError()`. |

**General best practices:**
- Keep schema in **migrations**; never hand-edit production schema untracked.
- Generate and use **typed clients** end to end.
- One **authorization model (RLS)** shared across JS and Go for consistency.
- Use **Edge Functions / Go** for anything needing secrets or `service_role`.
- Test policies as anon, as a non-owner, and as the owner.
- Monitor with the dashboard's **Logs, Reports, and Security/Performance advisors**.

---

## 16. Study Path & Build-to-Learn Projects

### Suggested order

1. **Database + RLS first.** Create a table in the SQL editor, enable RLS, write owner-only policies, test with the anon key. RLS is the foundation everything else rests on. (§3)
2. **supabase-js query builder.** Practice select/insert/update/delete, joins, filters, pagination, RPC against your table. (§4)
3. **Auth.** Wire email/password + OAuth; understand the JWT/session model and `getUser` vs `getSession`. (§5)
4. **Next.js SSR integration.** Build the three `utils/supabase/*` files, the middleware, and a login → protected dashboard flow. This is the most error-prone part — get it solid. (§6)
5. **CLI + migrations + types.** Move your schema into migrations, generate types, make the client typed. (§2, §10)
6. **Realtime & Storage.** Add a live feed and file uploads with storage RLS. (§7, §8)
7. **Go backend.** Connect with `pgx`, verify the JWTs your Next.js app issues, and decide your RLS strategy. (§11)
8. **Edge Functions & pgvector.** Add a webhook function and a semantic-search feature. (§9, §12)
9. **Harden.** Walk the security checklist; run the advisors. (§13, §15)

### Build-to-learn projects

**Project A — "Notes" (Next.js only).** Auth (email + GitHub OAuth), a `notes` table with owner-only RLS, full CRUD via supabase-js in Server Components + Server Actions, the SSR middleware, storage for note attachments (private bucket, per-user RLS), and Realtime so notes update live across tabs. Goal: master §3–§8.

**Project B — "Realtime chat with a Go moderation API" (Next.js + Go, one Supabase project).** Next.js handles auth and the chat UI (Realtime Postgres Changes for messages). A **Go service** connects via `pgx` for heavy queries, **verifies the Supabase JWT** on its endpoints (extract `sub`), exposes an admin-only `/moderate` endpoint (role from `app_metadata`), and uses the **run-as-user transaction** pattern so RLS is honored even from Go. Goal: master §11 end to end — the shared-authorization-model architecture.

**Project C — "Semantic doc search" (pgvector + Edge Function).** Upload documents to Storage, an Edge Function (or Go service) computes embeddings and stores them in a `documents` table with an HNSW index, and a search page calls `match_documents` for RAG-style results — all scoped by RLS. Goal: master §9 and §12.

> Build all three against a single Supabase project and you will have touched every service, both client paths (JS + Go), and the security model that ties them together. That's a complete, production-grade mental model of Supabase as of 2026 — and a portfolio that demonstrates the full-stack, multi-runtime architecture most teams actually run.
