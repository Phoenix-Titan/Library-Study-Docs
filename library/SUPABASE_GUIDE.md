# Supabase — Complete Offline Reference (with Next.js & Go)

> **Who this is for:** Full-stack developers who want one complete, internet-free reference for building real apps on **Supabase** — a Next.js front-end (App Router) and a Go back-end both talking to the *same* Supabase project. It covers the database, Row Level Security, Auth, the JavaScript client, Realtime, Storage, Edge Functions, migrations, pgvector, and — in unusual depth — how to wire Supabase into a Go service (direct Postgres via `pgx`, JWT verification, calling the REST/Storage APIs over HTTP, and respecting RLS).

> **Version note:** This guide targets **Supabase as of 2026**. Key facts baked in:
> - Backend is **PostgreSQL** (typically Postgres 15/16/17 depending on project age). Everything is "just Postgres" underneath.
> - The Next.js integration uses the **`@supabase/ssr`** package. The older **`@supabase/auth-helpers-nextjs` is deprecated** — do **not** use it for new work; `@supabase/ssr` is the supported path.
> - JavaScript client is **supabase-js v2** (`@supabase/supabase-js`).
> - Local dev, migrations, and type generation use the **Supabase CLI** (`supabase` binary) over a Docker-based local stack.
> - JWTs: legacy projects sign with a shared **HS256** secret; newer projects (2025+) support **asymmetric signing keys (RS256/ES256) exposed via JWKS** for verification without sharing a secret. Both are covered.
>
> Supabase moves fast. Where a detail is especially likely to drift, it's flagged with **⚡ Version note**. For exact, current API shapes always confirm against the official docs — but the *concepts* here are stable.

---

## Table of Contents

1. [What Supabase Is & When to Use It](#1-what-supabase-is--when-to-use-it)
2. [Getting Started: Project, Keys, CLI, Local Dev](#2-getting-started-project-keys-cli-local-dev)
3. [The Database & Row Level Security (RLS)](#3-the-database--row-level-security-rls)
4. [supabase-js Client Deep Dive](#4-supabase-js-client-deep-dive)
5. [Auth (GoTrue)](#5-auth-gotrue)
6. [Next.js Integration (App Router) with @supabase/ssr](#6-nextjs-integration-app-router-with-supabasessr)
7. [Realtime](#7-realtime)
8. [Storage](#8-storage)
9. [Edge Functions (Deno)](#9-edge-functions-deno)
10. [Database Migrations & Generated Types](#10-database-migrations--generated-types)
11. [Golang Integration (Direct DB, JWT, REST, RLS)](#11-golang-integration-direct-db-jwt-rest-rls)
12. [pgvector / AI: Embeddings & Semantic Search](#12-pgvector--ai-embeddings--semantic-search)
13. [Security Best Practices](#13-security-best-practices)
14. [Self-Hosting & Pricing/Limits Awareness](#14-self-hosting--pricinglimits-awareness)
15. [Gotchas & Best Practices](#15-gotchas--best-practices)
16. [Study Path & Build-to-Learn Projects](#16-study-path--build-to-learn-projects)

---

## 1. What Supabase Is & When to Use It

Supabase is an **open-source Firebase alternative built on PostgreSQL**. Instead of a proprietary NoSQL store, you get a real relational SQL database plus a suite of services bolted around it that turn the database into a complete backend. The defining idea: **everything is Postgres**, and the extra services are thin layers that read/write that Postgres.

### The pieces

| Service | What it is | Underlying tech |
|---|---|---|
| **Database** | A full PostgreSQL instance you own — tables, SQL, extensions, functions, triggers. | PostgreSQL |
| **Auth** | User sign-up/sign-in, sessions, OAuth, magic links, MFA. Issues JWTs. | **GoTrue** (a Go auth server, ironically) |
| **Auto REST API** | Every table/view is instantly a RESTful endpoint with filtering, joins, pagination. | **PostgREST** |
| **Realtime** | Subscribe to DB changes (insert/update/delete), broadcast messages, presence. | Elixir/Phoenix `Realtime` server + Postgres logical replication |
| **Storage** | S3-style object storage for files, with access controlled by Postgres RLS. | Storage API (backed by S3-compatible store + a `storage` schema in Postgres) |
| **Edge Functions** | Serverless TypeScript/JavaScript functions at the edge. | **Deno** runtime |
| **Vector** | Store and similarity-search embeddings for AI/semantic search. | **pgvector** Postgres extension |

The big mental model: **the database is the source of truth and the security boundary.** Auth issues a JWT carrying the user's id; PostgREST, Realtime, and Storage all forward that JWT into Postgres, where **Row Level Security policies** decide what the user can see and do. You write your authorization rules *once*, in SQL, and every access path enforces them.

### When to use it

**Reach for Supabase when:**
- You want a real SQL database with relations, transactions, and constraints (not document soup).
- You want auth, storage, and realtime without standing up three separate services.
- You're building a SaaS, dashboard, social app, marketplace, internal tool, or any CRUD-heavy product.
- You value **not being locked in** — it's open source and self-hostable; your data is a standard Postgres dump.
- You want to ship a frontend-only MVP fast (the JS client + RLS can replace a whole backend tier), but keep the door open to add a custom backend (e.g. in Go) later, against the same database.

**Be cautious / consider alternatives when:**
- You need a globally-distributed multi-master write database (Supabase is single-primary Postgres + read replicas).
- Your workload is extreme write throughput / append-only telemetry at huge scale — a purpose-built store may fit better.
- You're allergic to SQL and don't want to learn RLS (you'll fight the security model otherwise).

> The sweet spot: **Supabase as your Postgres + auth + storage backbone**, accessed directly from a Next.js app via the JS client *and* from a Go service via `pgx`/HTTP, all sharing one authorization model.

---

## 2. Getting Started: Project, Keys, CLI, Local Dev

### 2.1 Create a project (hosted)

1. Sign in at the Supabase dashboard and create an **organization**, then a **project**.
2. Choose a **region** close to your users (latency matters; it's fixed after creation).
3. Set a **database password** — this is the password for the `postgres` superuser-ish role used in connection strings. Store it in a password manager; you'll need it for direct DB access from Go.
4. Wait ~2 minutes for provisioning.

### 2.2 The dashboard (what each section is)

| Section | Use |
|---|---|
| **Table Editor** | Spreadsheet-like view to create/edit tables and rows. |
| **SQL Editor** | Run arbitrary SQL, save snippets, create policies/functions. Your power tool. |
| **Authentication** | Manage users, providers (Google, GitHub…), email templates, MFA, URL config. |
| **Storage** | Create buckets, browse files, set policies. |
| **Database** | Schemas, roles, extensions, replication, **connection pooling** settings, backups. |
| **Edge Functions** | Deploy/inspect Deno functions and their secrets. |
| **Realtime** | Inspect channels and publication settings. |
| **Project Settings → API** | Your **Project URL**, **anon key**, **service_role key**, JWT settings. |
| **Project Settings → Database** | Connection strings (direct + pooler), pooler mode. |

### 2.3 Keys: `anon` vs `service_role` (the CRITICAL difference)

Every project ships two primary API keys, both JWTs signed by your project:

| Key | Role it maps to | Where it may live | RLS |
|---|---|---|---|
| **`anon`** (public/publishable) | Postgres role `anon` (or `authenticated` after login) | **Safe in the browser / client apps.** Exposed in your Next.js bundle. | **Subject to RLS.** This is the whole point — RLS protects you even though the key is public. |
| **`service_role`** (secret) | Postgres role `service_role` | **SERVER ONLY. Never in a browser, never in a mobile app, never in a public repo, never in `NEXT_PUBLIC_*`.** | **BYPASSES RLS entirely.** It is effectively god-mode over your data. |

> **🚨 The single most important security rule in this guide:** the **`service_role` key bypasses Row Level Security**. If it leaks to a client, anyone can read and modify *all* data in *all* tables. Treat it like a database root password. It belongs only in trusted server environments (Go backend, Edge Functions, server-side Next.js secrets — and even there, prefer the anon key + user JWT when you can).

> **⚡ Version note:** Supabase has been migrating naming toward **publishable keys** (`sb_publishable_...`, replaces `anon`) and **secret keys** (`sb_secret_...`, replaces `service_role`) alongside the asymmetric JWT signing keys rollout. The *semantics* are identical: one public/RLS-respecting key, one secret/RLS-bypassing key. This guide uses the classic `anon`/`service_role` names since they remain widely valid and clearer to reason about.

### 2.4 Connection strings

From **Project Settings → Database** you get connection strings. Three flavors matter:

```
# 1) DIRECT connection (port 5432) — a real Postgres TCP connection to the primary.
#    Best for: long-lived servers, migrations, anything needing session features
#    (LISTEN/NOTIFY, prepared statements, advisory locks). Limited connection count.
postgresql://postgres:[PASSWORD]@db.[PROJECT-REF].supabase.co:5432/postgres

# 2) SUPAVISOR — SESSION mode (port 5432 via the pooler host).
#    A connection pooler that keeps one server connection per client session.
#    Behaves like a direct connection but lets more clients connect. Prepared statements OK.
postgresql://postgres.[PROJECT-REF]:[PASSWORD]@aws-0-[REGION].pooler.supabase.com:5432/postgres

# 3) SUPAVISOR — TRANSACTION mode (port 6543).
#    Pooling per-transaction: a server connection is borrowed only for the duration of a
#    transaction, then returned. Ideal for serverless / many short-lived clients.
#    ⚠️ Prepared statements & session state DON'T survive across transactions here.
postgresql://postgres.[PROJECT-REF]:[PASSWORD]@aws-0-[REGION].pooler.supabase.com:6543/postgres
```

**Supavisor** is Supabase's connection pooler (it replaced PgBouncer). Pooler modes summarized:

| Mode | Port | Best for | Caveat |
|---|---|---|---|
| **Direct** | 5432 | Single long-lived server, migrations | Few connections available (IPv6 by default on hosted) |
| **Session** | 5432 (pooler host) | Persistent servers needing more connections + session features | Holds a connection per client |
| **Transaction** | 6543 | Serverless, edge, high concurrency, short queries | **No prepared statement caching, no session-level state across queries** |

> **⚡ Go + pooler gotcha (preview — full detail in §11):** `pgx` and `lib/pq` cache prepared statements by default. Against **transaction-mode** pooling this breaks (`prepared statement "stmtcache_..." already exists` / does not exist). Fix: disable statement caching (`pgx` simple protocol or `default_query_exec_mode = exec`) or append `?default_query_exec_mode=simple_protocol` / `statement_cache_mode=describe`. Use **session/direct** mode for a persistent Go server when you can.

### 2.5 The Supabase CLI & local development

The CLI runs the **entire Supabase stack locally in Docker** — Postgres, GoTrue, PostgREST, Realtime, Storage, Studio — so you can develop offline and version-control your schema.

```bash
# Install (pick one)
npm install -g supabase          # via npm
brew install supabase/tap/supabase   # macOS
scoop install supabase           # Windows (scoop bucket)

supabase --version               # verify

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

supabase status                  # show running services + keys again
supabase stop                    # stop the stack (add --no-backup to discard local data)
```

Local stack key facts:
- Local **DB** is on port **54322** with user/password `postgres`/`postgres`.
- Local **API** (PostgREST + GoTrue + Storage gateway) is on **54321**.
- Local **Studio** (the dashboard) is on **54323**.
- The local anon/service_role keys are **fixed dev keys** printed by `supabase start` — they are not secret (they only work locally), so they're fine to commit in `.env.local` for the team if you want.

#### Linking local to a hosted project & pushing schema

```bash
supabase login                                  # authenticate the CLI to your account
supabase link --project-ref [PROJECT-REF]       # connect this repo to a remote project

# Pull the remote schema down into a migration (when the remote already has tables)
supabase db pull

# Create a new empty migration file to write SQL into
supabase migration new create_profiles

# Apply local migrations to the LOCAL db
supabase db reset       # wipes local db & replays ALL migrations + seed.sql (great for clean state)

# Push local migrations UP to the linked remote project
supabase db push

# Generate TypeScript types from the (local or remote) schema
supabase gen types typescript --local > src/types/database.types.ts
# or from remote:
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

> **Best practice:** treat `supabase/migrations/` as the canonical schema. Never click-create tables in production and forget them — make schema changes as migrations so they're reproducible and reviewable. (Full migration workflow in §10.)

---

## 3. The Database & Row Level Security (RLS)

This is **the** section. RLS is the core security model of Supabase. If you internalize one thing, make it this.

### 3.1 Why RLS exists here

Because the **`anon` key is public** and the JS client talks to your database directly (via PostgREST), you cannot rely on "the client won't send bad requests." Anyone can open dev tools, grab the anon key, and call your API with arbitrary filters. **Row Level Security** is what stands between that and a data breach: Postgres itself decides, per-row, whether the current user may select/insert/update/delete.

### 3.2 Enabling RLS

When you create a table in the dashboard, RLS is **enabled by default but with no policies** — which means **deny all** (no rows in or out via the anon/authenticated roles). For tables you create via SQL, you must enable it explicitly:

```sql
-- A table of user profiles
create table profiles (
  id uuid primary key references auth.users (id) on delete cascade,
  username text unique,
  bio text,
  created_at timestamptz not null default now()
);

-- TURN ON RLS. Until you add policies, NO ONE (anon/authenticated) can read or write.
alter table profiles enable row level security;
```

> **🚨 Danger:** A table with RLS **disabled** is fully exposed through the public API — the anon key can do anything Postgres permissions allow. **Never disable RLS on a table reachable via the API.** If you genuinely need a table to be server-only, keep RLS on with no policies (deny all) and access it solely through the `service_role` key or `security definer` functions.

### 3.3 Anatomy of a policy

```sql
create policy "policy name"
  on <table>
  as { permissive | restrictive }       -- default permissive (OR-combined). restrictive is AND-combined.
  for { all | select | insert | update | delete }
  to <role>                             -- e.g. authenticated, anon, public
  using ( <expression> )                -- row visibility / which existing rows the op may touch
  with check ( <expression> );          -- validation for NEW/updated rows (INSERT/UPDATE)
```

Two clauses, two jobs:
- **`using`** — applied to *existing* rows. For SELECT/UPDATE/DELETE it filters which rows are visible/affected.
- **`with check`** — applied to the *new* row's values on INSERT and UPDATE. It validates what you're trying to write.

| Operation | Uses `using`? | Uses `with check`? |
|---|---|---|
| SELECT | ✅ | — |
| INSERT | — | ✅ |
| UPDATE | ✅ (which rows) | ✅ (resulting values) |
| DELETE | ✅ | — |

### 3.4 The auth helper functions

Inside a policy you have access to the current request's identity, injected from the JWT:

```sql
auth.uid()    -- returns the user's UUID (the JWT 'sub' claim), or NULL if not logged in
auth.jwt()    -- returns the full decoded JWT as jsonb (claims, app_metadata, user_metadata, role)
auth.role()   -- 'anon' or 'authenticated' (the Postgres role for the request)
```

Examples of reading claims:

```sql
auth.uid()                                   -- current user id
(auth.jwt() ->> 'email')                     -- email claim (text)
(auth.jwt() -> 'app_metadata' ->> 'role')    -- a custom role stored in app_metadata
(auth.jwt() #>> '{user_metadata,full_name}') -- nested user_metadata value
```

> `app_metadata` is **set by the server only** (trusted — good for roles/permissions). `user_metadata` is **user-editable** (never trust it for authorization).

### 3.5 Policy recipes (the ones you'll actually write)

**Owner-only (private rows): each user sees and edits only their own.**

```sql
-- A todos table where user_id ties a row to its owner
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

-- UPDATE: you may change your rows, and may not reassign ownership to someone else
create policy "update own todos"
  on todos for update to authenticated
  using      ( (select auth.uid()) = user_id )   -- which rows you can target
  with check ( (select auth.uid()) = user_id );  -- the row must still be yours afterward

-- DELETE: only your own
create policy "delete own todos"
  on todos for delete to authenticated
  using ( (select auth.uid()) = user_id );
```

> **⚡ Performance note:** wrap `auth.uid()` in a scalar subquery — `(select auth.uid())` — in policies. Postgres can then evaluate it **once per query** (an InitPlan) instead of **once per row**. On large tables this is a massive speedup and is the officially recommended pattern.

**Public read, authenticated write.**

```sql
-- Anyone (even logged-out) can read posts; only logged-in users can create them.
create policy "public can read posts"
  on posts for select to anon, authenticated
  using ( true );

create policy "authenticated can insert posts"
  on posts for insert to authenticated
  with check ( (select auth.uid()) = author_id );
```

**Role-based (admin can do anything).** Store the role in `app_metadata` (server-set) and check it via `auth.jwt()`:

```sql
create policy "admins manage all posts"
  on posts for all to authenticated
  using      ( (auth.jwt() -> 'app_metadata' ->> 'role') = 'admin' )
  with check ( (auth.jwt() -> 'app_metadata' ->> 'role') = 'admin' );
```

Because policies are **permissive (OR)** by default, this admin policy *adds* to the owner policies: a normal user matches the owner policy, an admin matches the admin policy.

**Team/membership-based (multi-tenant).** Visibility flows through a join table:

```sql
-- A user can read documents that belong to a team they're a member of.
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

> For complex membership checks, factor them into a `security definer` helper function (next) and call it from the policy — both for readability and to avoid the policy itself being blocked by RLS on the join table.

### 3.6 `security definer` functions

By default a function runs as the **caller** (`security invoker`) — so it's still subject to RLS. A `security definer` function runs as its **creator/owner**, letting it bypass RLS for a *specific, controlled* operation. This is the safe way to do things RLS would otherwise block (e.g. checking membership without exposing the whole `team_members` table, or performing privileged writes).

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

> **🚨 `security definer` rules:**
> - **Always `set search_path = ''`** (or to an explicit, trusted schema) to prevent a malicious user from shadowing functions/tables you call. This is the #1 definer vulnerability.
> - Keep them small and validate inputs — they bypass RLS, so a bug = a hole.
> - Revoke `execute` from `anon`/`public` if a function shouldn't be callable by everyone.

### 3.7 Recommended workflow

1. Create table → **enable RLS immediately**.
2. Add the *minimum* policies needed (start deny-all, open up deliberately).
3. Test as an anonymous user, as a logged-in non-owner, and as the owner.
4. Use the dashboard's **policy templates** and the SQL editor's "RLS not enabled" warnings as guardrails.
5. Never paper over RLS pain by using `service_role` from the client — that's a breach waiting to happen.

---

## 4. supabase-js Client Deep Dive

`@supabase/supabase-js` (v2) is the universal JS/TS client. It wraps PostgREST (the query builder), GoTrue (auth), Realtime, Storage, and Functions.

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

Every query returns a `{ data, error }` object — **you must check `error`**; it does not throw by default.

### 4.1 Select & the query builder

```ts
// Basic select — '*' returns all columns
const { data, error } = await supabase.from("posts").select("*");

// Pick specific columns
await supabase.from("posts").select("id, title, created_at");

// Rename a column in the result with an alias
await supabase.from("posts").select("id, headline:title");

// Aggregate / casting
await supabase.from("posts").select("id, title::text");
```

#### Nested / foreign-table joins (embedding)

PostgREST can embed related tables when foreign keys exist:

```ts
// One-to-many: posts with their author (FK posts.author_id -> profiles.id)
const { data } = await supabase
  .from("posts")
  .select(`
    id,
    title,
    author:profiles ( id, username )      -- embed the related profile, aliased as 'author'
  `);

// Many side: a profile and all its posts
await supabase
  .from("profiles")
  .select(`
    username,
    posts ( id, title )                   -- array of related posts
  `);

// Disambiguating multiple FKs to the same table: specify the FK/relationship name
await supabase
  .from("messages")
  .select(`
    body,
    sender:profiles!messages_sender_id_fkey ( username ),
    recipient:profiles!messages_recipient_id_fkey ( username )
  `);

// Filtering on an embedded table, and inner join semantics:
await supabase
  .from("posts")
  .select("title, comments!inner(body)")  // !inner drops posts with no comments
  .eq("comments.is_approved", true);      // filter on the embedded table's column
```

#### Filters & operators

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
q.is("deleted_at", null);           // IS NULL / IS true/false
q.in("category", ["go", "ts"]);     // IN (...)
q.contains("tags", ["postgres"]);   // array/jsonb contains @>
q.containedBy("tags", ["a", "b"]);  // <@
q.overlaps("tags", ["go", "rust"]); // && (arrays overlap)
q.range("price", 10, 50);           // ⚠️ NOT pagination — this is a range column op
q.match({ status: "published", featured: true });  // multiple eq at once

// OR conditions
q.or("status.eq.published,featured.eq.true");

// Negation
q.not("status", "eq", "archived");

// Text search (see 4.6)
```

#### Ordering, limiting, pagination

```ts
// Order
await supabase.from("posts").select("*").order("created_at", { ascending: false });
// Order by an embedded table's column
await supabase.from("posts").select("*, profiles(username)")
  .order("username", { referencedTable: "profiles", ascending: true });

// Limit
await supabase.from("posts").select("*").limit(10);

// PAGINATION with .range(from, to) — zero-based, INCLUSIVE on both ends
//   page size 20, page index `p` (0-based):
const p = 2, size = 20;
await supabase
  .from("posts")
  .select("*")
  .order("created_at", { ascending: false })
  .range(p * size, p * size + size - 1);   // rows 40..59
```

> **⚡ Note the two `range`s:** `.range(from, to)` (on the builder) is **row pagination**. `.range(column, lo, hi)` (a *filter*) is a Postgres range/`int4range` operator. Different methods despite the shared name.

#### Single-row helpers

```ts
// .single() — expects EXACTLY one row; errors if 0 or >1. data is the object (not array).
const { data, error } = await supabase
  .from("profiles").select("*").eq("id", id).single();

// .maybeSingle() — 0 or 1 row; returns null data if none (no error). Use when row may not exist.
const { data: maybe } = await supabase
  .from("profiles").select("*").eq("id", id).maybeSingle();
```

### 4.2 Insert / Update / Upsert / Delete

```ts
// INSERT one or many. Use .select() to get the inserted rows back.
const { data, error } = await supabase
  .from("posts")
  .insert({ title: "Hello", author_id: userId })
  .select()
  .single();

await supabase.from("posts").insert([{ title: "A" }, { title: "B" }]); // bulk

// UPDATE — ALWAYS scope with a filter, or you'll update every row your RLS allows!
await supabase
  .from("posts")
  .update({ title: "Edited" })
  .eq("id", postId)
  .select();

// UPSERT — insert, or update on conflict of the PK / a unique constraint
await supabase
  .from("profiles")
  .upsert(
    { id: userId, username: "neo" },
    { onConflict: "id" }            // conflict target; default is the primary key
  );

// DELETE — also scope it!
await supabase.from("posts").delete().eq("id", postId);
```

> **🚨 Footgun:** `update(...)` / `delete(...)` **without a filter** affects *all rows visible to your RLS policy*. Always add `.eq(...)` (or similar). For extra safety you can require a `where`-like filter by enabling the PostgREST "max affected rows" / using a filter discipline.

### 4.3 RPC — calling Postgres functions

Define a function in SQL, then call it via `.rpc()`. Great for atomic operations, complex queries, and `security definer` privileged logic.

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

// Call a set-returning function — you can chain filters/order on the result!
const { data } = await supabase
  .rpc("search_posts", { term: "postgres" })
  .order("created_at", { ascending: false })
  .limit(5);
```

### 4.4 Count

```ts
// Get rows AND an exact count (separate round-trip semantics via head/count)
const { data, count } = await supabase
  .from("posts")
  .select("*", { count: "exact" });

// Count ONLY (no rows transferred) — head: true
const { count: total } = await supabase
  .from("posts")
  .select("*", { count: "exact", head: true });

// count options: 'exact' (accurate, slower), 'planned' (estimate), 'estimated'
```

### 4.5 `.throwOnError()`

By default errors live in `error`. If you prefer try/catch (cleaner with async flows, and lets errors bubble), chain `.throwOnError()`:

```ts
try {
  // Throws a typed error instead of returning { error }
  const { data } = await supabase.from("posts").select("*").throwOnError();
  return data;
} catch (e) {
  // handle / log
}

// You can also set it globally on the client:
const supabase = createClient(url, key, { /* ... */ });
```

### 4.6 Full-text search

```sql
-- Add a generated tsvector column + GIN index for fast search
alter table posts add column fts tsvector
  generated always as (to_tsvector('english', coalesce(title,'') || ' ' || coalesce(body,''))) stored;
create index posts_fts_idx on posts using gin (fts);
```

```ts
// textSearch maps to Postgres FTS operators
await supabase.from("posts").select("*")
  .textSearch("fts", "postgres & realtime");           // AND

await supabase.from("posts").select("*")
  .textSearch("fts", "'web app'", { type: "phrase" }); // phrase search

await supabase.from("posts").select("*")
  .textSearch("fts", "go | golang", { type: "websearch", config: "english" });
```

### 4.7 Typed client (preview)

After generating types (§10) you parameterize the client so every query is type-checked:

```ts
import { createClient } from "@supabase/supabase-js";
import type { Database } from "@/types/database.types";

const supabase = createClient<Database>(url, anonKey);
// Now .from("posts") knows the columns of posts, and data is fully typed.
```

---

## 5. Auth (GoTrue)

Supabase Auth is powered by **GoTrue** (a Go service). It manages users in the `auth.users` table, issues a **JWT access token** + a **refresh token**, and supports many sign-in methods. The access token is what RLS reads via `auth.uid()`/`auth.jwt()`.

### 5.1 The session / JWT model

- On sign-in, GoTrue returns a **session**: `access_token` (a short-lived JWT, default 1 hour), `refresh_token` (long-lived, single-use), `expires_at`, and the `user`.
- The **access token** is a JWT whose `sub` claim = the user's UUID. It carries `role` (`authenticated`), `email`, `app_metadata`, `user_metadata`, `aud`, `exp`, etc.
- The client auto-refreshes: before the access token expires it uses the refresh token to get a new pair.
- **Every request to Supabase forwards the access token** in the `Authorization: Bearer` header, so PostgREST/Realtime/Storage know who you are and RLS applies.

### 5.2 Email / password

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
const { data, error } = await supabase.auth.signInWithPassword({
  email: "user@example.com",
  password: "S3curePassw0rd!",
});

// SIGN OUT (clears local session + revokes refresh token)
await supabase.auth.signOut();
```

> **⚡ Email confirmation:** if "Confirm email" is enabled (default), `signUp` creates the user but **no session** until they click the email link. Handle the "check your inbox" state. You can disable confirmation for dev in `config.toml` / dashboard.

### 5.3 Magic links & OTP

```ts
// Magic link: emails a one-click login link (passwordless)
await supabase.auth.signInWithOtp({
  email: "user@example.com",
  options: { emailRedirectTo: "https://myapp.com/auth/callback" },
});

// Email OTP code (6-digit) instead of a link — then verify:
await supabase.auth.signInWithOtp({ email, options: { shouldCreateUser: true } });
await supabase.auth.verifyOtp({ email, token: "123456", type: "email" });

// Phone/SMS OTP (needs an SMS provider configured)
await supabase.auth.signInWithOtp({ phone: "+15551234567" });
await supabase.auth.verifyOtp({ phone: "+15551234567", token: "123456", type: "sms" });
```

### 5.4 OAuth providers

```ts
// Redirect-based OAuth (Google, GitHub, Apple, Discord, etc. — configured in the dashboard)
await supabase.auth.signInWithOAuth({
  provider: "github",
  options: {
    redirectTo: "https://myapp.com/auth/callback",
    scopes: "read:user",
  },
});
// The user is redirected to the provider, then back to your callback route,
// which exchanges the code for a session (see §6 Route Handler for SSR).
```

### 5.5 `getUser()` vs `getSession()` — the safety distinction

```ts
// getSession() — reads the session from local storage / cookies. FAST but UNVERIFIED:
//   the access token is taken at face value. Safe in the browser; NOT safe to trust on the server.
const { data: { session } } = await supabase.auth.getSession();

// getUser() — sends the access token to GoTrue (the Auth server) to VALIDATE it,
//   and returns the authenticated user. Slower (network) but TRUSTWORTHY.
const { data: { user } } = await supabase.auth.getUser();
```

> **🚨 Server-side rule:** **always use `getUser()` on the server** (Server Components, Route Handlers, middleware) to gate access. `getSession()` only reads cookies that a malicious client could tamper with; it does **not** verify the JWT signature/expiry against the auth server. Use `getUser()` for any authorization decision on the server.

### 5.6 Password reset & email change

```ts
// 1) Send the reset email
await supabase.auth.resetPasswordForEmail("user@example.com", {
  redirectTo: "https://myapp.com/account/update-password",
});
// 2) On that page the user is in a recovery session; set the new password:
await supabase.auth.updateUser({ password: "newSecret123!" });

// Change email (sends confirmation to the new address)
await supabase.auth.updateUser({ email: "new@example.com" });
```

### 5.7 User metadata

```ts
// user_metadata: user-editable profile-ish data. NOT for authorization.
await supabase.auth.updateUser({ data: { full_name: "Ada", avatar_url: "..." } });

// app_metadata: server-set, trusted (roles/permissions). Set it from a TRUSTED context only
// (Edge Function / Go backend with service_role via the Admin API), NOT from the client.
```

```ts
// Reacting to auth changes anywhere in the app (browser)
supabase.auth.onAuthStateChange((event, session) => {
  // event: 'SIGNED_IN' | 'SIGNED_OUT' | 'TOKEN_REFRESHED' | 'USER_UPDATED' | 'PASSWORD_RECOVERY'
});
```

### 5.8 Anonymous sign-ins

```ts
// Create a throwaway anonymous user (gets a real auth.users row + JWT, role 'authenticated').
// Great for "try before you sign up" — later link to email/OAuth to convert.
const { data, error } = await supabase.auth.signInAnonymously();

// Convert later by adding credentials to the same user:
await supabase.auth.updateUser({ email: "real@example.com" });
```

> Anonymous users still hit RLS as `authenticated`. Distinguish them in policies via `auth.jwt() ->> 'is_anonymous'` if you need to restrict what anon-converted users can do.

### 5.9 MFA (multi-factor auth)

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

// 3) On future logins, after password, check assurance level & step up:
const { data: aal } = await supabase.auth.mfa.getAuthenticatorAssuranceLevel();
// aal.currentLevel 'aal1' (password) vs nextLevel 'aal2' (MFA required)
```

> Enforce MFA in **RLS** using the JWT's `aal` claim: `using ( (auth.jwt() ->> 'aal') = 'aal2' )` for sensitive tables.

### 5.10 Admin API (server-only)

With the `service_role` key you can manage users programmatically (create, delete, set `app_metadata`, list). This is server-only and is exactly what your Go backend or an Edge Function uses to, e.g., assign roles. (Go equivalent: hit the GoTrue Admin REST endpoints — §11.3.)

```ts
const admin = createClient(url, SERVICE_ROLE_KEY); // SERVER ONLY
await admin.auth.admin.updateUserById(userId, { app_metadata: { role: "admin" } });
await admin.auth.admin.createUser({ email, password, email_confirm: true });
await admin.auth.admin.deleteUser(userId);
```

---

## 6. Next.js Integration (App Router) with @supabase/ssr

This is the canonical 2026 setup. The challenge in SSR is that auth state lives in **cookies** that must be readable/writable across the browser, Server Components, Server Actions, Route Handlers, and **refreshed in middleware**. The `@supabase/ssr` package handles the cookie wiring.

```bash
npm install @supabase/supabase-js @supabase/ssr
```

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://<ref>.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
# NEVER put SERVICE_ROLE here as NEXT_PUBLIC_. If you need it server-side:
SUPABASE_SERVICE_ROLE_KEY=eyJ...   # used only in server-only modules
```

> **⚡ `@supabase/auth-helpers-nextjs` is deprecated.** All code below uses `@supabase/ssr`. If you see `createClientComponentClient` / `createServerComponentClient` from auth-helpers in old tutorials, migrate to the patterns here.

### 6.1 `utils/supabase/client.ts` — the browser client

```ts
// src/utils/supabase/client.ts
// Used in Client Components ("use client"). Reads/writes auth via browser cookies.
import { createBrowserClient } from "@supabase/ssr";

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  );
}
```

### 6.2 `utils/supabase/server.ts` — the server client

```ts
// src/utils/supabase/server.ts
// Used in Server Components, Server Actions, and Route Handlers.
// It bridges Supabase auth to Next.js's cookie store.
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
        // Read all cookies for the request
        getAll() {
          return cookieStore.getAll();
        },
        // Write refreshed auth cookies back.
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options),
            );
          } catch {
            // The `setAll` was called from a Server Component, where setting cookies
            // is not allowed. This is SAFE TO IGNORE *if* you refresh the session in
            // middleware (which we do in §6.3) — middleware will persist the cookies.
          }
        },
      },
    },
  );
}
```

### 6.3 `utils/supabase/middleware.ts` + `middleware.ts` — session refresh (REQUIRED)

The access token expires; **middleware refreshes it on every request** and writes the new cookies so Server Components see a valid session. This is the load-bearing piece — skip it and users get logged out / Server Components see stale auth.

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
          // Write to BOTH the request (so this pass sees them) and the response
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

  // 🚨 IMPORTANT: do NOT run code between createServerClient and getUser().
  // getUser() triggers the token refresh and revalidation. It must come first.
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

  // 🚨 Return supabaseResponse as-is (with its cookies). If you create a NEW
  // response, copy over supabaseResponse.cookies or auth will break.
  return supabaseResponse;
}
```

```ts
// src/middleware.ts (project root of the app dir tree)
import { type NextRequest } from "next/server";
import { updateSession } from "@/utils/supabase/middleware";

export async function middleware(request: NextRequest) {
  return await updateSession(request);
}

export const config = {
  // Run on everything except static assets & images.
  matcher: [
    "/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)",
  ],
};
```

### 6.4 Reading the user in a Server Component

```tsx
// src/app/dashboard/page.tsx — a protected Server Component
import { redirect } from "next/navigation";
import { createClient } from "@/utils/supabase/server";

export default async function DashboardPage() {
  const supabase = await createClient();

  // Always getUser() (verified) on the server — never trust getSession() for gating.
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect("/login");

  // RLS automatically scopes this to the logged-in user via the forwarded JWT.
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

### 6.5 Server Actions (mutations + auth)

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

### 6.6 Route Handler: the OAuth / email-confirm callback

```ts
// src/app/auth/callback/route.ts
// Exchanges the ?code from OAuth / magic link / email confirm for a session,
// writing the auth cookies, then redirects into the app.
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

### 6.7 Calling supabase-js from a Client Component

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

### 6.8 SSR auth pitfalls (read this)

| Pitfall | Fix |
|---|---|
| Using `getSession()` server-side for gating | Use **`getUser()`** — it verifies with the auth server. |
| No middleware / not calling `getUser()` in it | Tokens never refresh → random logouts & stale auth. The middleware in §6.3 is **required**. |
| Code between `createServerClient` and `getUser()` in middleware | Can cause hard-to-debug logout loops. Call `getUser()` immediately. |
| Returning a new `NextResponse` from middleware without copying cookies | Refreshed cookies are dropped → user logged out. Return `supabaseResponse`. |
| Trying to `cookies().set()` in a Server Component | Not allowed; the `try/catch` in `server.ts` swallows it, and middleware persists cookies instead. |
| Putting `service_role` in `NEXT_PUBLIC_*` | Catastrophic leak. Service role is server-only and never `NEXT_PUBLIC_`. |
| Forgetting `emailRedirectTo` / Site URL config | Confirmation/OAuth redirects fail. Set Site URL + redirect allow-list in the dashboard. |

---

## 7. Realtime

Realtime lets clients **subscribe to database changes**, send ephemeral **broadcast** messages, and track **presence** — over WebSockets, organized into **channels**.

### 7.1 Postgres Changes (listen to inserts/updates/deletes)

First, the table must be in the realtime **publication** (so changes are streamed):

```sql
-- Add a table to the realtime publication
alter publication supabase_realtime add table messages;
-- (Optional) include full old-row data on update/delete for richer payloads
alter table messages replica identity full;
```

```ts
"use client";
import { createClient } from "@/utils/supabase/client";

const supabase = createClient();

const channel = supabase
  .channel("room:lobby")                        // any channel name
  .on(
    "postgres_changes",
    { event: "INSERT", schema: "public", table: "messages" }, // filter what to receive
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

// Cleanup (e.g. in useEffect return)
supabase.removeChannel(channel);
```

> **🚨 RLS + Realtime:** Postgres Changes **respect RLS** for the `authenticated`/`anon` roles — a client only receives change events for rows it's allowed to **SELECT**. You must enable RLS and have a SELECT policy, *and* enable the table for realtime, or clients receive nothing (or, if misconfigured without RLS, everything — dangerous). Realtime authorization uses the user's JWT just like queries.

### 7.2 Broadcast (ephemeral pub/sub, not stored)

```ts
const channel = supabase.channel("game:1", {
  config: { broadcast: { self: true } },   // also receive your own broadcasts
});

channel
  .on("broadcast", { event: "cursor" }, ({ payload }) => {
    // payload = whatever a sender sent
  })
  .subscribe();

// Send a message to everyone on the channel (low-latency, not persisted)
channel.send({ type: "broadcast", event: "cursor", payload: { x: 12, y: 80 } });
```

> Broadcast is ideal for cursors, typing indicators, game state, live reactions — anything you don't need to store. You can also gate broadcast/presence with **Realtime Authorization** policies on the `realtime.messages` table.

### 7.3 Presence (who's online)

```ts
const channel = supabase.channel("room:lobby", {
  config: { presence: { key: userId } },   // unique key per client
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

### 7.4 A live chat (Postgres Changes + insert)

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

Storage is S3-style object storage, organized into **buckets**, with access governed by **Postgres RLS** (the `storage.objects` table).

### 8.1 Buckets: public vs private

```ts
// Create buckets (usually done once, via dashboard or service_role)
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

### 8.2 Upload / download / list / remove

```ts
const supabase = createClient();

// UPLOAD (path within bucket). upsert overwrites if it exists.
const { data, error } = await supabase.storage
  .from("avatars")
  .upload(`${userId}/profile.png`, file, {
    cacheControl: "3600",
    upsert: true,
    contentType: "image/png",
  });

// PUBLIC URL (only meaningful for public buckets) — synchronous, no auth check
const { data: pub } = supabase.storage.from("avatars").getPublicUrl(`${userId}/profile.png`);
// pub.publicUrl

// SIGNED URL (private buckets) — time-limited link
const { data: signed } = await supabase.storage
  .from("invoices")
  .createSignedUrl(`${userId}/2026-06.pdf`, 60 * 60); // valid 1 hour
// signed.signedUrl

// DOWNLOAD (returns a Blob)
const { data: blob } = await supabase.storage.from("invoices").download(`${userId}/2026-06.pdf`);

// LIST files under a prefix
const { data: files } = await supabase.storage.from("avatars").list(userId, {
  limit: 100, offset: 0, sortBy: { column: "name", order: "asc" },
});

// REMOVE
await supabase.storage.from("avatars").remove([`${userId}/profile.png`]);
```

### 8.3 RLS policies on storage

Files are rows in `storage.objects`; you write policies against that table. A common pattern: users may only touch files in a folder named after their user id.

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

> `storage.foldername(name)` returns the path segments as an array; `[1]` is the first folder. Naming uploads `"<uid>/file.ext"` makes per-user isolation a one-liner.

### 8.4 Image transformations

For images you can request resized/optimized variants on the fly (Pro feature on hosted):

```ts
const { data } = supabase.storage.from("avatars").getPublicUrl(`${userId}/profile.png`, {
  transform: { width: 200, height: 200, resize: "cover", quality: 80 },
});

// Same options work on createSignedUrl({ transform: {...} }) and download({ transform: {...} })
```

### 8.5 Resumable uploads (large files)

For large files, use **resumable uploads** (TUS protocol) so interrupted uploads can continue. The browser SDK exposes `uploadToSignedUrl`, and the `tus-js-client` integrates with the Storage TUS endpoint (`/storage/v1/upload/resumable`). Use this for video / multi-GB files; standard `upload()` is fine up to the bucket's size limit for normal assets.

---

## 9. Edge Functions (Deno)

Edge Functions are **Deno** serverless functions deployed to the edge. Use them for: webhooks (Stripe, etc.), server-side logic that needs the `service_role` key safely, third-party API calls hiding secrets, scheduled jobs, and custom auth flows.

### 9.1 Write & run locally

```bash
supabase functions new hello       # scaffolds supabase/functions/hello/index.ts
supabase functions serve hello     # run locally (hot reload) at /functions/v1/hello
```

```ts
// supabase/functions/hello/index.ts
// Deno runtime. Imports are by URL (or via an import map / deno.json).
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

  // Build a Supabase client that ACTS AS THE CALLER (forwards their JWT → RLS applies).
  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_ANON_KEY")!,
    { global: { headers: { Authorization: req.headers.get("Authorization")! } } },
  );
  const { data: { user } } = await supabase.auth.getUser();

  // For privileged work, use the service_role (bypasses RLS) — server-only env, safe here.
  // const admin = createClient(Deno.env.get("SUPABASE_URL")!, Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!);

  return new Response(JSON.stringify({ message: `Hello ${name}`, user: user?.id ?? null }), {
    headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" },
  });
});
```

### 9.2 Secrets

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

### 9.3 Deploy & invoke

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

> **⚡ JWT verification:** by default Edge Functions **require a valid Supabase JWT** (the platform checks it before your code runs). For public webhooks (Stripe, GitHub) deploy with `--no-verify-jwt` and verify the provider's signature yourself inside the function.

---

## 10. Database Migrations & Generated Types

### 10.1 Migration workflow

Migrations are timestamped SQL files in `supabase/migrations/`. They are your schema's version history — commit them.

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

A migration file is just SQL:

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

Seed data for local dev:

```sql
-- supabase/seed.sql  (runs on `supabase db reset`)
insert into public.posts (title, body) values ('Hello', 'First post');
```

### 10.2 Generating TypeScript types

```bash
# From the local stack:
supabase gen types typescript --local > src/types/database.types.ts
# From the remote project:
supabase gen types typescript --project-id <ref> --schema public > src/types/database.types.ts
```

```ts
// Use the generated Database type everywhere for end-to-end type safety.
import type { Database } from "@/types/database.types";

// Handy row/insert/update helpers from the generated types:
type Post = Database["public"]["Tables"]["posts"]["Row"];
type PostInsert = Database["public"]["Tables"]["posts"]["Insert"];
type PostUpdate = Database["public"]["Tables"]["posts"]["Update"];
```

Wire it into the SSR clients so all queries are typed:

```ts
import { createBrowserClient } from "@supabase/ssr";
import type { Database } from "@/types/database.types";

export const createClient = () =>
  createBrowserClient<Database>(url, anonKey); // .from('posts') is now fully typed
```

> **Best practice:** regenerate types whenever the schema changes (add it to a `package.json` script or CI). Stale types silently lie to you.

---

## 11. Golang Integration (Direct DB, JWT, REST, RLS)

Supabase's Go story is under-documented because Supabase pushes the JS client. But a Go service is a first-class citizen against the same Postgres + auth. There are **four** realistic approaches; you'll often combine (a) + (b).

| Approach | What | When |
|---|---|---|
| **(a) Direct Postgres** via `pgx` | Connect straight to the DB, write SQL, bypass PostgREST | Custom Go API, heavy/complex queries, background jobs, performance |
| **(b) Verify Supabase JWTs** | Validate the access token Next.js/clients send | Any Go endpoint that must know *who* is calling |
| **(c) Call PostgREST / Storage over HTTP** | Use the REST API with service_role or user JWT | Reuse RLS/PostgREST logic; simple CRUD without writing SQL |
| **(d) Respect RLS from Go** | Pass the user's JWT so Postgres enforces policies | When you want one authorization model across JS + Go |

### 11.1 (a) Connecting directly with `pgx`

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
// prepared-statement caching (see the connStr note below) or pgx will error with
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

	// ── If using TRANSACTION pooler (port 6543), uncomment this to avoid the
	//    prepared-statement cache incompatibility:
	// cfg.ConnConfig.DefaultQueryExecMode = pgx.QueryExecModeSimpleProtocol
	//    (import "github.com/jackc/pgx/v5"). Alternatively append
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
// Example queries with pgx. Note: a direct connection uses the `postgres` role,
// which BYPASSES RLS (like service_role). Enforce authorization in your Go code,
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

> **⚡ Direct DB role bypasses RLS.** When you connect as `postgres` (the default for the connection string), **RLS does not apply** — you're a superuser-ish role. That's powerful and dangerous: your Go code is now solely responsible for authorization, *unless* you explicitly run as the user (11.4).

### 11.2 (b) Verifying Supabase Auth JWTs in Go

Your Next.js app (or any client) sends `Authorization: Bearer <access_token>`. Your Go API verifies it to learn the user id (`sub`) and claims. Two signing schemes exist in 2026:

| Scheme | Algorithm | How to verify | When |
|---|---|---|---|
| **Legacy / shared secret** | **HS256** | Use the project's **JWT Secret** (Settings → API) as the HMAC key | Older projects, or projects still on symmetric signing |
| **Asymmetric signing keys** | **RS256 / ES256** | Fetch the project's **JWKS** (`<url>/auth/v1/.well-known/jwks.json`) and verify with the public key | Newer projects (2025+); lets you verify without sharing a secret |

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
	Email       string                 `json:"email"`
	Role        string                 `json:"role"` // 'authenticated' / 'anon'
	AppMetadata map[string]interface{} `json:"app_metadata"`
	jwt.RegisteredClaims                              // includes Subject (sub), ExpiresAt, Audience…
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

// jwks is created once at startup; it auto-refreshes the public keys in the background.
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
		// Accept the asymmetric algorithms Supabase may use.
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

> **⚡ Which one?** Check **Settings → API / JWT Keys** in your dashboard. If your project shows a single "JWT Secret" you're on HS256. If it shows asymmetric **signing keys** with a JWKS endpoint, prefer JWKS verification (no secret to leak; rotation is automatic). A robust backend can try JWKS first and fall back to HS256 during a migration window.

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

// Example role check using app_metadata.role
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

### 11.3 (c) Calling PostgREST / Storage / Auth over HTTP from Go

Sometimes you want to reuse PostgREST's filtering/RLS rather than write SQL. Hit the REST API directly.

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
// Passing the service_role key bypasses RLS (server-only, use sparingly).
func GetPosts(ctx context.Context, userJWT string) ([]map[string]any, error) {
	base := os.Getenv("SUPABASE_URL") // https://<ref>.supabase.co
	anon := os.Getenv("SUPABASE_ANON_KEY")

	// PostgREST endpoint: /rest/v1/<table>?<filters>
	url := base + "/rest/v1/posts?select=id,title&order=created_at.desc&limit=20"
	req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)

	// apikey = the anon (or service_role) key. Authorization = the per-user JWT for RLS.
	req.Header.Set("apikey", anon)
	req.Header.Set("Authorization", "Bearer "+userJWT) // use anon key here too if no user
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

Storage over HTTP (download a private object with service_role):

```go
// GET /storage/v1/object/<bucket>/<path>   (authenticated/service_role)
// GET /storage/v1/object/public/<bucket>/<path>  (public buckets, no auth)
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

Auth Admin API over HTTP (assign a role via `app_metadata`, server-only):

```go
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
// (strings_NewReader = bytes.NewReader(payload) — shown inline for clarity)
```

### 11.4 (d) Respecting RLS from Go (running queries AS the user)

If you want Postgres — not your Go code — to enforce authorization even on a direct `pgx` connection, run each request inside a transaction that **assumes the `authenticated` role and sets the JWT claims**, mimicking what PostgREST does. Then `auth.uid()` works and policies apply.

```go
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
	// 2) Provide the JWT claims so auth.uid()/auth.jwt() resolve.
	//    request.jwt.claims is the GUC PostgREST sets; auth.uid() reads sub from it.
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
// Usage: claims JSON must at least contain {"sub":"<user-uuid>","role":"authenticated"}.
claimsJSON := fmt.Sprintf(`{"sub":%q,"role":"authenticated"}`, userID)
err := WithUserRLS(ctx, pool, claimsJSON, func(tx pgx.Tx) error {
	// This SELECT is now filtered by the table's RLS policies for that user.
	rows, err := tx.Query(ctx, `select id, task from todos order by inserted_at desc`)
	// ... scan ...
	return err
})
```

> **`set local`** scopes the role/config to the current transaction only, so a pooled connection reverts cleanly afterward. This is the safe way to honor RLS over a shared pool.

### 11.5 When to use the JS client vs direct DB from Go

| Situation | Recommendation |
|---|---|
| Next.js frontend, simple CRUD, want RLS for free | **supabase-js** (browser/SSR). No backend needed. |
| Complex queries, transactions, joins, batch jobs, performance | **Go + `pgx` direct** (write SQL). |
| Go API that must authenticate users | **Verify the JWT** (11.2) and trust `sub`. |
| Want one authorization model (RLS) enforced even in Go | **Direct `pgx` + run-as-user tx** (11.4) or **call PostgREST with the user JWT** (11.3c). |
| Privileged admin operations from Go | **service_role** via Admin API / direct DB (server-only, audited). |
| Need to call from Go but don't want to write SQL | **PostgREST over HTTP** (11.3c). |

> **Common production shape:** Next.js uses supabase-js for the bulk of UI CRUD (RLS does the security), while a Go service handles heavy lifting (background processing, integrations, complex transactional logic) by connecting with `pgx`, verifying user JWTs for any user-facing endpoints, and using `service_role`/direct DB for trusted operations.

---

## 12. pgvector / AI: Embeddings & Semantic Search

Supabase ships **pgvector**, turning Postgres into a vector store for semantic search and RAG. Relevant in 2026 for AI features.

```sql
-- Enable the extension
create extension if not exists vector;

-- A documents table with an embedding column.
-- Dimension must match your embedding model (e.g. 1536 for many models, 768/1024 for others).
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

-- Index for fast approximate nearest-neighbor search.
-- HNSW (recommended) with cosine distance:
create index on documents using hnsw (embedding vector_cosine_ops);
-- (Alternative: IVFFlat — create index ... using ivfflat (embedding vector_cosine_ops) with (lists = 100);)
```

Distance operators: `<->` (L2/Euclidean), `<#>` (negative inner product), `<=>` (cosine distance).

A `security definer`-free RPC to find similar documents:

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

Inserting an embedding (from a server context where you computed it):

```ts
await supabase.from("documents").insert({
  content: text,
  embedding: embedding, // number[] — supabase-js serializes to a pgvector literal
});
```

> **Tips:** match the `vector(N)` dimension to your model exactly; normalize vectors if your model expects it; use **HNSW** for most workloads (better recall/speed, no training); combine vector search with regular SQL filters/RLS in the same query for hybrid search.

---

## 13. Security Best Practices

A consolidated checklist — most breaches with Supabase come from violating these.

1. **Never expose `service_role` to a client.** Not in the browser, not in mobile apps, not in `NEXT_PUBLIC_*`, not in a public repo. It bypasses RLS = full data access. Server-only.
2. **Enable RLS on every table reachable via the API**, and write explicit policies. RLS off = wide open. RLS on with no policies = safe deny-all.
3. **Use `getUser()` (not `getSession()`) for server-side authorization.** `getSession()` is unverified cookie data.
4. **Verify JWTs in your Go backend** — pin the algorithm (`WithValidMethods`), check `aud="authenticated"`, never accept `alg=none`. Prefer JWKS (asymmetric) where available.
5. **`security definer` functions: always `set search_path = ''`** and keep them minimal; revoke `execute` from `public`/`anon` if not needed.
6. **Put authorization data in `app_metadata` (server-set), never `user_metadata`** (user-editable). Don't trust user_metadata in policies.
7. **Scope every `update`/`delete`** with a filter; treat unfiltered mutations as a bug.
8. **Lock down Auth redirect URLs** (Site URL + allow-list) to prevent open-redirect / token theft.
9. **Manage secrets properly** — Edge Function secrets via `supabase secrets`, Go via env/secret manager. Rotate the DB password and keys if leaked.
10. **Storage: private buckets + per-folder RLS + signed URLs** for anything sensitive; don't dump private files in public buckets.
11. **Rate-limit and validate** in Edge Functions / Go for public endpoints; verify provider signatures for webhooks (deployed `--no-verify-jwt`).
12. **Run the dashboard's Security Advisor / linter** — it flags tables with RLS disabled, exposed definer functions, and missing policies.

---

## 14. Self-Hosting & Pricing/Limits Awareness

### 14.1 Self-hosting

Supabase is **open source** and self-hostable (you own the whole stack). The standard path is **Docker Compose**:

```bash
# Clone the repo and use the docker setup
git clone --depth 1 https://github.com/supabase/supabase
cd supabase/docker
cp .env.example .env        # set POSTGRES_PASSWORD, JWT_SECRET, ANON_KEY, SERVICE_ROLE_KEY, SITE_URL...
docker compose up -d        # starts Postgres, GoTrue, PostgREST, Realtime, Storage, Studio, Kong...
```

What you run yourself: **Kong** (API gateway), **GoTrue**, **PostgREST**, **Realtime**, **Storage API**, **Studio**, **Postgres** (+ `pg_meta`), and an analytics/logging stack. You generate your own `JWT_SECRET` and derive matching `ANON_KEY`/`SERVICE_ROLE_KEY` JWTs. Trade-off: full control & no usage fees, but you own upgrades, backups, scaling, HA, and security patching.

> Self-host for data-residency/compliance, cost at scale, or air-gapped environments. For most teams, hosted is far less operational burden. The same client code works against either — only the URL/keys change.

### 14.2 Pricing & limits awareness (concepts, not exact figures)

Plans roughly: **Free** (great for dev/small projects; projects may pause after inactivity), **Pro** (production baseline + higher limits + daily backups + add-ons), **Team** (collaboration/SOC2-style needs), **Enterprise** (custom). Things that consume quota / drive cost:

| Dimension | Notes |
|---|---|
| **Database size / disk** | Grows with data; disk is provisioned/auto-scaled on paid plans. |
| **Egress / bandwidth** | API + Storage + Realtime data transfer. |
| **Storage** | GB stored + transformations. |
| **Monthly active users (MAU)** | Auth pricing tier dimension. |
| **Edge Function invocations** | Per-invocation/compute. |
| **Realtime** | Concurrent connections + messages. |
| **Compute add-ons** | Bigger instance sizes, read replicas, dedicated pooler. |

> **⚡ Always check the current pricing page for exact numbers** — tiers and limits change. The actionable takeaways: enable backups on production, watch egress on media-heavy apps, and remember Free-tier projects can be paused if idle.

---

## 15. Gotchas & Best Practices

| Gotcha | What happens / Fix |
|---|---|
| **Forgot to enable RLS** | Table fully exposed via the anon key. Enable RLS on every API-reachable table. |
| **RLS on but no policies** | Nothing works (deny-all) — *this is correct/safe*; add policies deliberately. |
| **`auth.uid()` per-row in policies** | Slow on big tables. Wrap as `(select auth.uid())` so it evaluates once. |
| **`getSession()` on the server** | Unverified — use `getUser()` for any auth decision server-side. |
| **No middleware in Next.js** | Sessions don't refresh → random logouts. The §6.3 middleware is mandatory. |
| **New `NextResponse` in middleware** | Drops refreshed cookies → logout. Return the `supabaseResponse` (or copy cookies). |
| **`service_role` in `NEXT_PUBLIC_`** | Full data breach. Service role is server-only, never public. |
| **Transaction-pooler (6543) + prepared statements** | `pgx`/`lib/pq` error. Use simple protocol / disable statement cache, or use session/direct mode. |
| **Direct `pgx` connection ignores RLS** | The `postgres` role bypasses RLS. Authorize in Go or run as user (11.4). |
| **Unfiltered `update`/`delete`** | Mutates all RLS-visible rows. Always add a filter. |
| **`.single()` on 0 or many rows** | Errors. Use `.maybeSingle()` when the row may be absent. |
| **Realtime table not in publication** | No events. `alter publication supabase_realtime add table ...`. |
| **Realtime without a SELECT policy** | Clients receive no change events (RLS blocks them). Add SELECT policy. |
| **`security definer` without `set search_path`** | Function hijack vulnerability. Always pin `search_path = ''`. |
| **`alg=none` / algorithm confusion in Go JWT** | Always pass `jwt.WithValidMethods([...])` and check the method type. |
| **Trusting `user_metadata` for roles** | User can edit it. Use server-set `app_metadata`. |
| **Wrong `vector(N)` dimension** | Inserts/queries fail or mismatch. Match the embedding model exactly. |
| **Stale generated types** | Types lie after schema changes. Regenerate on every migration. |
| **Email confirm enabled but unhandled** | `signUp` returns no session; show "check your email". |
| **Storage path not user-prefixed** | Per-user RLS via `storage.foldername(name)[1]` won't work. Name files `"<uid>/..."`. |
| **Missing Auth redirect allow-list** | OAuth/magic-link redirects fail or are unsafe. Configure Site URL + allowed redirects. |

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

**Project C — "Semantic doc search" (pgvector + Edge Function).** Upload documents to Storage, an Edge Function (or Go service) computes embeddings and stores them in a `documents` table with HNSW index, and a search page calls `match_documents` for RAG-style results — all scoped by RLS. Goal: master §9 and §12.

> Build all three against a single Supabase project and you'll have touched every service, both client paths (JS + Go), and the security model that ties them together. That's a complete, production-grade mental model of Supabase as of 2026.
