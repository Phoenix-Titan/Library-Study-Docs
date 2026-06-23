# Next.js 16 — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I know some React but I've never built a real Next.js app" to "I architect production Next.js apps and can reason about rendering, caching, Server Components, and security from first principles" — entirely offline. This is an **explain-first** guide: every concept is taught in prose (what it is, *why* Next.js was designed this way, when and how to use it, the key options, best practices, and security notes), then demonstrated with heavily-commented, runnable code. Read it top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Next.js 16** with the **App Router**, running on **React 19** (current in 2026). Modern features you should know and that this guide uses throughout:
> - **Turbopack is the default bundler** for both `next dev` and `next build` — the old Webpack pipeline is legacy. Builds and HMR are dramatically faster.
> - **`async` `params` / `searchParams` / `cookies()` / `headers()` / `draftMode()`** — all request-time APIs are Promises now (since 15). You `await` them.
> - **Cache Components** (the `use cache` directive — the productionized evolution of Partial Prerendering, PPR) is the headline caching model of the 16 era: a static shell with streamed dynamic "holes" in a single response, plus component/function-level caching.
> - **`fetch()` is NOT cached by default** (since 15) — a deliberate reversal of the 14 behaviour. You opt into caching explicitly.
> - **Node.js runtime for Middleware** (not Edge-only), the **React Compiler** for automatic memoization, and improved error overlays.
>
> Where behaviour is version-sensitive it's flagged with **⚡ Version note**. The author is on **Windows 11**, so cross-platform notes (paths, shells) are called out. Next.js evolves fast — always confirm exact APIs at nextjs.org.
>
> **Companion guides in this library** (cross-referenced throughout): **`REACT_19_GUIDE.md`** (the React layer this is built on — hooks, Suspense, the `use` hook), **`TANSTACK_QUERY_GUIDE.md`** (client-side server-state), **`ZUSTAND_GUIDE.md`** (client UI state), **`TAILWIND_CHEATSHEET.md`** (styling), and **`PRISMA_ORM_GUIDE.md`** (the database layer for examples here).

---

## Table of Contents

1. [What Next.js Is & Why It Exists (SSR/SSG/ISR/RSC)](#1-what-nextjs-is--why-it-exists) **[B]**
2. [Install & Project Structure](#2-install--project-structure) **[B]**
3. [The App Router — File Conventions](#3-the-app-router--file-conventions) **[B]**
4. [Server vs Client Components — The Mental Model](#4-server-vs-client-components) **[B/I]**
5. [Routing Deep Dive](#5-routing-deep-dive) **[I]**
6. [Layouts, Templates & Navigation](#6-layouts-templates--navigation) **[B/I]**
7. [Data Fetching](#7-data-fetching) **[I]**
8. [Caching — The Four Layers Explained](#8-caching--the-four-layers-explained) **[I/A]**
9. [Rendering Strategies (Static / Dynamic / Streaming / PPR)](#9-rendering-strategies) **[I/A]**
10. [Server Actions & Mutations](#10-server-actions--mutations) **[I/A]**
11. [Loading, Streaming & Suspense](#11-loading-streaming--suspense) **[I]**
12. [Error Handling](#12-error-handling) **[I]**
13. [Route Handlers (the API layer)](#13-route-handlers-the-api-layer) **[I]**
14. [Metadata & SEO](#14-metadata--seo) **[I]**
15. [Images, Fonts & the `<Link>` Component](#15-images-fonts--the-link-component) **[B/I]**
16. [Styling](#16-styling) **[B]**
17. [Middleware](#17-middleware) **[I/A]**
18. [Environment Variables & Config (security)](#18-environment-variables--config) **[I]**
19. [Authentication Patterns](#19-authentication-patterns) **[A]**
20. [Performance Optimization](#20-performance-optimization) **[I/A]**
21. [Deployment](#21-deployment) **[I]**
22. [Design Patterns & Architecture](#22-design-patterns--architecture) **[A]**
23. [Gotchas & Best Practices](#23-gotchas--best-practices) **[I/A]**
24. [Study Path & Build-to-Learn Projects](#24-study-path--build-to-learn-projects)

---

## 1. What Next.js Is & Why It Exists

### 1.1 The problem Next.js solves **[B]**

To understand Next.js you first have to understand the limitation it was built to escape. A plain React app (the kind `create-react-app` or a bare Vite + React setup produces) is a **Single-Page Application (SPA)**. The server sends a near-empty HTML file plus a large JavaScript bundle; the browser downloads and runs that JavaScript, and *only then* does React build the page in the DOM. This is called **client-side rendering (CSR)**.

CSR has two chronic problems. First, **SEO and social sharing suffer**: the initial HTML is essentially blank, so search-engine crawlers and link-preview bots that don't execute JavaScript well see nothing. Second, **perceived performance suffers**: the user stares at a blank screen or spinner while a megabyte of JavaScript downloads, parses, and executes before any content appears. For a marketing site, a blog, or e-commerce — exactly the cases where ranking on Google and loading instantly are the whole point — CSR is a poor fit.

Next.js is a **React framework**: it takes React's component model and wraps it in everything a real web app needs that React itself deliberately leaves out — **server-side rendering, file-based routing, a build system, a backend/API layer, image and font optimization, and a sophisticated caching system**. The core idea is that the server does meaningful work *before* the browser gets involved, so the user receives real HTML immediately and JavaScript only "hydrates" (attaches interactivity to) that HTML afterwards.

**Reach for Next.js when** you need: a site that must rank on Google and load fast (marketing, blogs, docs); a full web app with both UI and backend logic (dashboards, SaaS, e-commerce) without standing up a separate API server; or anything where the React ecosystem is a fit but a bare SPA's SEO and first-load cost are unacceptable. **Don't reach for it when** you're building a purely internal tool where SEO and first-load are irrelevant and a Vite SPA is simpler, or a non-web target (use React Native — see `REACT_NATIVE_EXPO_GUIDE.md`).

### 1.2 The rendering vocabulary — SSR, SSG, ISR, RSC **[B/I]**

These four acronyms confuse everyone at first because they answer slightly different questions. Here is the precise mental model.

- **CSR (Client-Side Rendering)** — HTML is built *in the browser* by JavaScript. This is the plain-React baseline. In Next.js this still happens for the interactive parts (Client Components) after hydration.

- **SSR (Server-Side Rendering)** — HTML is built *on the server, per request*. Every time a user hits the page, the server runs your components, produces full HTML, and sends it. The user sees content immediately; React then hydrates it for interactivity. Use SSR when the content is **personalized or constantly changing** (a logged-in dashboard, a page that reads the current user's cookies). The cost: the server does work on every request, so it's slower and more expensive than serving a pre-built file.

- **SSG (Static Site Generation)** — HTML is built *once, at build time*. When you run `next build`, Next.js renders the page to a static HTML file and ships it. Every visitor gets that same pre-rendered file, served instantly from a CDN with zero per-request server work. This is the **fastest possible** option. Use it for content that's the same for everyone and doesn't change often: marketing pages, documentation, blog posts. The limitation: the content is frozen at build time until you rebuild.

- **ISR (Incremental Static Regeneration)** — SSG's smarter sibling. The page is static (served from cache) but Next.js **regenerates it in the background** on a schedule or on demand. You get SSG's speed *and* fresh content. A news site might serve a cached homepage but rebuild it every 60 seconds; an e-commerce catalog might revalidate when a product changes. This is usually the sweet spot for content that changes occasionally.

- **RSC (React Server Components)** — this one is a *different axis* and the foundation of the modern App Router. SSR/SSG/ISR are about *when* HTML is produced. RSC is about *where a component runs and whether its code ships to the browser at all*. A Server Component runs **only on the server**, can directly `await` a database query, and its JavaScript is **never sent to the client**. This is how Next.js can ship dramatically less JavaScript than a SPA: most of your tree never becomes browser code. RSC is covered in depth in §4 and in `REACT_19_GUIDE.md`.

The beautiful part of the App Router is that **these are per-route, automatic decisions** Next.js makes based on how you write the code, and you can override them. You don't pick "I'm building an SSG app" up front the way you used to; each route becomes static, dynamic, or a hybrid based on what data it touches.

### 1.3 The two routers — App vs Pages **[B]**

Next.js currently ships **two routing systems** and you must not confuse them:

- **App Router** (`app/` directory) — the modern, default system, built on React Server Components. Layouts, streaming, Server Actions, and the new caching model all live here. **This is what you learn and what this guide covers.**
- **Pages Router** (`pages/` directory) — the original system (`getServerSideProps`, `getStaticProps`, `_app.tsx`). Still supported for backward compatibility, but legacy. You'll only touch it maintaining older codebases.

> They can technically coexist in one project during migration, but for anything new in 2026 you write App Router exclusively.

---

## 2. Install & Project Structure

### 2.1 Creating a project **[B]**

```bash
npx create-next-app@latest my-app
```

The CLI wizard asks a series of questions. Recommended answers for modern client work:

- **TypeScript?** → **Yes.** Types catch entire categories of bugs and the Next.js APIs are richly typed.
- **ESLint?** → Yes (linting catches mistakes early).
- **Tailwind CSS?** → **Yes** (the de-facto styling choice; see `TAILWIND_CHEATSHEET.md`).
- **`src/` directory?** → Recommended — keeps app code separate from config files at the root.
- **App Router?** → **Yes** (always, for new projects).
- **Turbopack?** → Yes (it's the default and much faster).
- **Import alias?** → Keep the default `@/*` so you can write `import { db } from "@/lib/db"` from anywhere instead of fragile `../../../` relative paths.

Then:

```bash
cd my-app
npm run dev     # start the dev server (Turbopack) → http://localhost:3000
npm run build   # produce an optimized production build
npm run start   # serve the production build locally
npm run lint    # run ESLint
```

> **⚡ Version note:** In Next.js 16, **Turbopack is the default bundler for both `dev` and `build`**. The legacy Webpack path still exists but you opt into it; assume Turbopack. The **React Compiler** is also available — it auto-memoizes components so you rarely hand-write `useMemo`/`useCallback`/`memo` (see `REACT_19_GUIDE.md`).

### 2.2 Project structure — the whole map **[B]**

A clean, scalable Next.js 16 layout. Read the comments — the *rules* here matter more than the exact folders:

```
my-app/
├── public/                      # static files served as-is at the root URL
│   ├── images/                  #   public/images/x.png  →  /images/x.png
│   └── favicon.ico
├── src/
│   ├── app/                     # ←★ App Router lives here. Folders = routes.
│   │   ├── layout.tsx           # ROOT layout (REQUIRED) — wraps every page, has <html>/<body>
│   │   ├── page.tsx             # home page  →  "/"
│   │   ├── globals.css          # global styles (imported ONCE in the root layout)
│   │   ├── loading.tsx          # global loading UI (Suspense fallback)
│   │   ├── error.tsx            # global error boundary UI
│   │   ├── not-found.tsx        # 404 UI
│   │   │
│   │   ├── about/page.tsx       # →  "/about"
│   │   │
│   │   ├── blog/
│   │   │   ├── page.tsx         # →  "/blog"
│   │   │   └── [slug]/page.tsx  # dynamic segment  →  "/blog/my-post"
│   │   │
│   │   ├── dashboard/
│   │   │   ├── layout.tsx       # nested layout (sidebar) — only wraps dashboard pages
│   │   │   ├── page.tsx
│   │   │   └── settings/page.tsx
│   │   │
│   │   ├── api/
│   │   │   └── users/route.ts   # Route Handler  →  "/api/users"  (GET/POST/…)
│   │   │
│   │   └── (marketing)/         # route GROUP — organizes files WITHOUT affecting the URL
│   │       ├── pricing/page.tsx # →  "/pricing"  (no "marketing" in the URL)
│   │       └── contact/page.tsx
│   │
│   ├── components/              # reusable React components
│   │   ├── ui/                  #   shadcn/ui primitives
│   │   └── layout/             #   navbar, footer, etc.
│   ├── lib/                     # logic: db clients, API wrappers, pure helpers
│   │   ├── db.ts               #   Prisma client (see PRISMA_ORM_GUIDE.md)
│   │   └── utils.ts
│   ├── hooks/                   # custom React hooks (useXyz)
│   ├── actions/                # Server Actions ("use server")
│   ├── types/                  # shared TypeScript types
│   └── config/                # site config, constants, nav data
│
├── .env.local                  # secrets — NEVER commit (it's in .gitignore by default)
├── next.config.ts              # Next.js configuration
├── tsconfig.json
└── package.json
```

**The one rule that governs everything in `app/`:** *folders define routes, and special filenames define behavior*. A folder only becomes a public URL when it contains a `page.tsx`. A folder with only a `layout.tsx` or helper files is invisible to the router.

---

## 3. The App Router — File Conventions

The App Router's power comes from a small set of **reserved filenames**. Drop a file with one of these names into any route folder and Next.js wires up that behavior automatically — no configuration, no registration. This is "convention over configuration" taken to its logical end. Learn this table and the router stops being mysterious.

| File | What it does | Component type |
|---|---|---|
| `page.tsx` | The **page UI** for a route. Its presence is what makes the route publicly reachable. | Server (default) or Client |
| `layout.tsx` | Shared UI that wraps a page **and all nested routes below it**. Persists across navigation (does not re-mount). The **root** `layout.tsx` is required and must render `<html>` and `<body>`. | Server (default) or Client |
| `template.tsx` | Like `layout`, but **re-mounts on every navigation** (fresh state, re-runs effects). Use for enter animations or per-navigation logging. | Server or Client |
| `loading.tsx` | Instant loading UI. Next automatically wraps the segment's `page` in a `<Suspense>` with this as the fallback. | Server or Client |
| `error.tsx` | **Error boundary** for the segment — catches render/runtime errors below it and shows recovery UI. **Must be a Client Component.** | Client (required) |
| `global-error.tsx` | Catches errors thrown in the **root layout itself**. Must render its own `<html>`/`<body>`. | Client (required) |
| `not-found.tsx` | UI shown when `notFound()` is called or a route doesn't match. | Server or Client |
| `route.ts` | A **Route Handler** (API endpoint). **Cannot coexist with `page.tsx`** in the same folder — a segment is either a page or an API endpoint, not both. | Server only |
| `default.tsx` | Fallback UI for an unmatched **parallel route** slot (see §5). | Server or Client |

And the **folder naming conventions** that shape URLs:

| Pattern | Meaning | Example |
|---|---|---|
| `folder/` | A normal route segment → appears in the URL. | `app/about/` → `/about` |
| `[id]/` | **Dynamic** segment — matches any value, exposed as `params.id`. | `app/post/[id]/` → `/post/123` |
| `[...slug]/` | **Catch-all** — matches one *or more* segments into an array. | `app/docs/[...slug]/` → `/docs/a/b/c` → `["a","b","c"]` |
| `[[...slug]]/` | **Optional catch-all** — like catch-all but also matches the parent with no segments. | also matches `/docs` |
| `(group)/` | **Route group** — organizes files / shares a layout, **omitted from the URL**. | `app/(auth)/login/` → `/login` |
| `_folder/` | **Private folder** — opted out of routing entirely (for colocating components). | `app/blog/_components/` |
| `@slot/` | **Parallel route** slot — render multiple pages in one layout simultaneously. | `app/dashboard/@analytics/` |
| `(.)folder` | **Intercepting route** — render a route's content in the current layout (e.g. a modal). | `(.)photo`, `(..)feed`, `(...)root` |

We'll use most of these concretely in §5.

---

## 4. Server vs Client Components

**This is the single most important concept in modern Next.js.** Internalize it and everything else clicks; misunderstand it and you'll fight the framework constantly. It's also explained from the React side in `REACT_19_GUIDE.md` — read both.

### 4.1 The mental model **[B/I]**

In the App Router, **every component is a React Server Component (RSC) by default.** It runs on the server, its code is never sent to the browser, and it can do server-only things like query a database directly. You opt a component *out* of being a Server Component — making it a **Client Component** — by putting the string `"use client"` at the very top of its file.

Why does this distinction exist? Because the two environments have genuinely different superpowers and constraints, and forcing you to choose makes the trade-off explicit:

**Server Components** run only on the server. They:
- **Can be `async`** and directly `await` data — DB queries, `fetch`, reading files. No `useEffect`, no loading states, no API round-trip.
- **Keep secrets safe** — they can read `process.env.DATABASE_URL`, API keys, etc., because none of their code reaches the browser.
- **Ship zero JavaScript** for themselves — they render to HTML on the server, shrinking your client bundle. This is the headline performance win.
- **Cannot** use React hooks (`useState`, `useEffect`, `useReducer`, `useContext`), browser APIs (`window`, `localStorage`), or event handlers (`onClick`, `onChange`). They have no interactivity — they're "render once on the server" components.

**Client Components** (`"use client"`) run on the server for the initial HTML *and* in the browser (where they hydrate and become interactive). They:
- **Can use state and effects** — `useState`, `useEffect`, `useReducer`, refs, context.
- **Can handle events** — `onClick`, `onChange`, form interactions.
- **Can use browser APIs** — `window`, `localStorage`, `navigator`, the DOM.
- **Cannot** be `async` components, and cannot directly access server resources (no DB queries, no secret env vars — that code would leak to the browser).

> **The naming is slightly misleading:** "Client Components" still render on the server for the first paint (so there's no blank flash). The name means "this component's code *also* ships to and runs in the client." Think of `"use client"` as "this and everything it imports must be sent to the browser."

```tsx
// app/products/page.tsx
// Server Component (the DEFAULT — note there is NO "use client" directive).
// It can be async and await data directly. None of this code reaches the browser.
import { db } from "@/lib/db"          // Prisma client — safe, server-only

export default async function ProductsPage() {
  const products = await db.product.findMany()   // direct DB query — runs on the server
  return (
    <ul>
      {products.map((p) => (
        <li key={p.id}>{p.name} — ${p.price}</li>
      ))}
    </ul>
  )
}
```

```tsx
// components/counter.tsx
"use client"                          // ← opts this file (and its imports) into the client bundle
import { useState } from "react"

export default function Counter() {
  const [n, setN] = useState(0)       // hooks require a Client Component
  // onClick is an event handler — also requires a Client Component
  return <button onClick={() => setN(n + 1)}>Clicked {n} times</button>
}
```

### 4.2 The `"use client"` boundary — what it actually means **[I]**

`"use client"` is not "make this one component a client component." It marks a **boundary**: this file *and every module it imports* become part of the client bundle. Everything from this point down the import tree is client code. That's why you put `"use client"` as high as it needs to be and **no higher** — every component you pull below the boundary ships to the browser.

The corollary that trips people up: **a Server Component can import and render a Client Component, but a Client Component cannot import a Server Component.** Once you're in client-land, you can't reach back to server-land by importing. Why? Because the Client Component's code runs in the browser, and a Server Component's code (with its DB queries and secrets) can't run there.

So how do you put server-rendered content *inside* a client component (say, a server-fetched list inside a client-side collapsible panel)? **You pass it as `children` (or any prop).** The Server Component renders its server content, then hands the already-rendered React tree to the Client Component as a prop. The client component just slots it in — it never sees the server code.

```tsx
// components/collapsible.tsx — CLIENT (needs state for open/closed)
"use client"
import { useState } from "react"

export default function Collapsible({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false)
  return (
    <div>
      <button onClick={() => setOpen((o) => !o)}>{open ? "Hide" : "Show"}</button>
      {open && children}        {/* children is already-rendered server content */}
    </div>
  )
}
```

```tsx
// app/page.tsx — SERVER (default). Composes a server child INTO a client wrapper.
import Collapsible from "@/components/collapsible"
import { db } from "@/lib/db"

async function ServerList() {                 // a Server Component
  const items = await db.item.findMany()
  return <ul>{items.map((i) => <li key={i.id}>{i.name}</li>)}</ul>
}

export default function Page() {
  return (
    <Collapsible>
      {/* ServerList runs on the server; its rendered output is passed as children.
          Collapsible (client) never imports ServerList — it just receives children. */}
      <ServerList />
    </Collapsible>
  )
}
```

### 4.3 Serialization — the props rule **[I]**

When a Server Component passes props *down* to a Client Component, those props have to cross the server→client boundary, which means they get **serialized** (turned into data that can be sent over the wire and reconstructed in the browser). Therefore props passed from Server → Client **must be serializable**: strings, numbers, booleans, plain objects/arrays, dates, `null`. You **cannot** pass functions, class instances, Symbols, or Maps/Sets. If you need to give a client component "an action to run on the server," you pass a **Server Action** (§10), which is the framework-blessed way to send a callable across the boundary.

### 4.4 The golden rules **[I]**

1. **Default to Server Components.** Only add `"use client"` to the components that genuinely need interactivity, state, or browser APIs.
2. **Push `"use client"` to the leaves.** Don't make a whole page a Client Component because one button needs an `onClick` — extract the button into its own client component and keep the page on the server.
3. **Server Components for data + structure; Client Components for interactivity.** Fetch and shape data on the server; sprinkle small islands of client interactivity on top.
4. **Compose, don't import across the boundary.** Pass server content into client wrappers via `children`/props.

**Mental model:** think of your app as mostly server-rendered HTML with small **islands of interactivity** (the client components) embedded in it. Less client JS = faster loads and better INP (interactivity) scores.

### 4.5 What actually happens on a request — the lifecycle **[I/A]**

It helps enormously to know the sequence of events, because it explains *why* the rules above exist. When a browser requests a route:

1. **The server renders the Server Components.** Next.js runs your async Server Components, awaiting their data. The output isn't HTML strings — it's a special serialized format called the **RSC Payload** (a compact description of the rendered tree, including "holes" where Client Components go and the serialized props to pass them).
2. **The server produces initial HTML.** From that payload plus the Client Components (rendered once on the server too), Next.js generates the full HTML document and streams it to the browser. The user sees a complete, content-filled page right away — no blank flash, good SEO.
3. **The browser downloads the client JS bundle** — *only* the code for the Client Components (the `"use client"` boundaries and their imports). Server Component code is never in this bundle.
4. **Hydration.** React attaches event listeners and state to the already-present HTML for the Client Components, making them interactive. The static, server-rendered parts just sit there as fast HTML.
5. **On subsequent client navigations** (`<Link>` clicks), the browser fetches just the *RSC Payload* for the new route (not a fresh HTML document) and React reconciles the change, preserving layouts and client state.

This is the whole game: **Server Components contribute structure and data but zero runtime JS; Client Components are the only thing that ships and hydrates.** Now the rules are obvious — a Server Component can't have an `onClick` because it's never in the browser to handle a click; props to a Client Component must be serializable because they travel inside the RSC Payload; and pushing `"use client"` to the leaves matters because everything below a boundary joins that downloaded bundle.

### 4.6 Reading cookies, headers & draft mode **[I]**

Server Components and Server Actions can read the incoming request's cookies and headers via async helpers from `next/headers`. **Reading any of these marks the route as dynamic** (it can't be static if it depends on per-request data) — a key thing to know when you're wondering why a page won't render statically.

```tsx
import { cookies, headers, draftMode } from "next/headers"

export default async function Page() {
  const cookieStore = await cookies()           // async since 15
  const theme = cookieStore.get("theme")?.value // read a cookie

  const headerList = await headers()            // async
  const ua = headerList.get("user-agent")       // read a request header

  const { isEnabled } = await draftMode()       // CMS "preview" toggle
  // ...
}
```

> Note: you can only **set** cookies from a Server Action or Route Handler (a place that produces a response), not from a Server Component during render. `cookieStore.set("theme", "dark")` inside an action works; inside a page render it doesn't.

---

## 5. Routing Deep Dive

The App Router is **file-system based**: the shape of your `app/` folder *is* your URL structure. This section covers the routing features beyond the basic "folder = path."

### 5.1 Dynamic segments **[I]**

A folder named `[param]` matches any value in that position and exposes it via the page's `params` prop. This is how you build pages like `/blog/my-first-post` or `/users/42` without a route per value.

> **⚡ Version note:** Since Next.js 15, `params` and `searchParams` are **Promises** — you must `await` them. The same applies to `cookies()`, `headers()`, and `draftMode()`. This change lets Next.js start rendering the static parts of a page before the dynamic request data resolves.

```tsx
// app/blog/[slug]/page.tsx  →  matches /blog/anything
export default async function BlogPost({
  params,
  searchParams,
}: {
  params: Promise<{ slug: string }>          // dynamic segment values
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>  // ?key=val
}) {
  const { slug } = await params              // MUST await — it's a Promise now
  const { ref } = await searchParams         // e.g. /blog/x?ref=twitter
  return <h1>Post: {slug} (referred by {ref ?? "nobody"})</h1>
}
```

**Catch-all** segments (`[...slug]`) collect *multiple* path segments into an array — useful for docs, file browsers, or any arbitrarily-nested path. **Optional catch-all** (`[[...slug]]`) additionally matches the bare parent route.

```tsx
// app/docs/[...slug]/page.tsx
// /docs/getting-started      → slug = ["getting-started"]
// /docs/api/auth/tokens      → slug = ["api", "auth", "tokens"]
export default async function Docs({ params }: { params: Promise<{ slug: string[] }> }) {
  const { slug } = await params
  return <p>Path depth: {slug.length} → {slug.join(" / ")}</p>
}
```

### 5.2 `generateStaticParams` — pre-rendering dynamic pages (SSG) **[I]**

A dynamic route is normally rendered on demand. But if you know all the possible values at build time (every blog slug, every product ID), you can **pre-render them all into static HTML** by exporting `generateStaticParams`. This turns a dynamic route into an SSG route — the best of both worlds: clean dynamic URLs, but static-file speed.

```tsx
// app/blog/[slug]/page.tsx
import { getAllPostSlugs, getPost } from "@/lib/posts"

// Runs at BUILD time. Returns the list of params to pre-render.
export async function generateStaticParams() {
  const slugs = await getAllPostSlugs()
  return slugs.map((slug) => ({ slug }))   // [{ slug: "post-1" }, { slug: "post-2" }, …]
}

// Optional: control what happens for a slug NOT in the list above.
// "blocking" (default) = render on first request then cache; false = 404 unknown slugs.
export const dynamicParams = true

export default async function Post({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params
  const post = await getPost(slug)
  return <article>{post.body}</article>
}
```

### 5.3 Route groups `(group)` **[I]**

Wrap a folder name in parentheses and it **disappears from the URL** while still letting you organize files and — crucially — **share a layout** among a subset of routes. The classic use: give your marketing pages one layout (public navbar/footer) and your app pages a different one (authenticated shell), without `/marketing` or `/app` polluting the URLs.

```
app/
├── (marketing)/
│   ├── layout.tsx        # public navbar + footer
│   ├── page.tsx          # → "/"   (the home page)
│   └── pricing/page.tsx  # → "/pricing"
└── (app)/
    ├── layout.tsx        # authenticated sidebar shell
    └── dashboard/page.tsx # → "/dashboard"
```

### 5.4 Parallel routes `@slot` **[A]**

Parallel routes let a single layout render **multiple pages at once**, each in its own "slot," each able to load, stream, and error **independently**. Think of a dashboard with an analytics panel and a team panel side by side, where one can show a spinner while the other is ready. Slots are folders named `@name`; they're passed to the layout as props (not part of the URL).

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,                 // the dashboard/page.tsx content
  analytics,                // ← from app/dashboard/@analytics/page.tsx
  team,                     // ← from app/dashboard/@team/page.tsx
}: {
  children: React.ReactNode
  analytics: React.ReactNode
  team: React.ReactNode
}) {
  return (
    <div className="grid grid-cols-2 gap-4">
      <section>{children}</section>
      <section>{analytics}</section>      {/* independent loading/error boundaries */}
      <section>{team}</section>
    </div>
  )
}
```

Each slot should have a `default.tsx` so it renders sensibly when the current URL doesn't match a state for that slot.

### 5.5 Intercepting routes `(.)` **[A]**

Intercepting routes let you **show a route's content inside the current page's layout** — most famously, opening a photo or product in a **modal** while keeping a real, shareable URL. Click a photo in a feed → it opens in a modal (intercepted); refresh or share the URL → the full photo page renders (the real route). The matchers mirror relative imports: `(.)` same level, `(..)` one up, `(...)` from root.

```
app/
├── feed/page.tsx
├── photo/[id]/page.tsx              # the FULL page (direct visit / refresh)
└── feed/
    └── (.)photo/[id]/page.tsx       # INTERCEPTED — renders as a modal over the feed
```

### 5.6 Programmatic navigation **[B/I]**

For navigation from event handlers (after a form submit, etc.) use the `useRouter` hook — **only available in Client Components**:

```tsx
"use client"
import { useRouter, usePathname, useSearchParams, useParams } from "next/navigation"

export default function NavButtons() {
  const router = useRouter()
  const pathname = usePathname()         // current path string, e.g. "/blog/x"
  const search = useSearchParams()       // read-only URLSearchParams of the query
  // router.push("/dashboard")           // navigate, add a history entry
  // router.replace("/login")            // navigate WITHOUT a history entry
  // router.refresh()                    // re-fetch Server Component data for the current route
  // router.back()  /  router.forward()  // history navigation
  return <button onClick={() => router.push("/dashboard")}>Go ({pathname})</button>
}
```

> **`router.refresh()` is a power tool:** it re-runs the Server Components for the current route and merges fresh server data into the page **without losing client state** (form inputs, scroll position). It's how you update server-rendered data after a mutation without a full reload. (Note: import from `next/navigation`, never the legacy `next/router`, which is the Pages Router.)

---

## 6. Layouts, Templates & Navigation

### 6.1 Layouts — shared, persistent UI **[B/I]**

A `layout.tsx` wraps a page **and everything nested below it**. Layouts are how you build shared chrome — navbars, sidebars, footers — without repeating it on every page. The critical property: **layouts persist across navigation**. When you move from `/dashboard` to `/dashboard/settings`, the dashboard layout does *not* re-mount — its state, scroll position, and any client components inside it stay alive. Only the changed `page` swaps out. This makes navigation feel instant and preserves things like an open sidebar or a playing video.

The **root layout** (`app/layout.tsx`) is mandatory, must render `<html>` and `<body>`, and is the one place global providers and `globals.css` belong.

```tsx
// app/layout.tsx — the ROOT layout (required)
import type { Metadata } from "next"
import "./globals.css"

export const metadata: Metadata = {
  title: { default: "Acme", template: "%s | Acme" },
  description: "We build fast, modern websites.",
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {/* Global providers (theme, query client, etc.) wrap children here */}
        {children}
      </body>
    </html>
  )
}
```

```tsx
// app/dashboard/layout.tsx — a NESTED layout (only wraps /dashboard/*)
export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex">
      <aside className="w-64">{/* sidebar — persists across dashboard navigation */}</aside>
      <main className="flex-1">{children}</main>
    </div>
  )
}
```

### 6.2 Templates — when you need a fresh mount **[I]**

A `template.tsx` is almost identical to a layout, but it **re-mounts on every navigation**: a new instance, fresh state, effects re-run. You want this rarely — for enter/exit animations that must replay on each page change, or per-navigation logging/analytics that depend on a `useEffect` firing every time. If you don't need a fresh mount, use a layout (persistence is usually what you want).

### 6.3 `<Link>` and prefetching **[B]**

Client-side navigation in Next.js goes through `<Link>` (from `next/link`), not raw `<a>` tags. `<Link>` does a soft navigation — it swaps the page content without a full document reload, preserving layouts and client state — and it **prefetches** the linked route's code and data when the link enters the viewport, so the destination is already loaded by the time the user clicks. That prefetch-on-hover/in-view behavior is most of why Next.js apps feel snappy.

```tsx
import Link from "next/link"

// Basic — prefetches automatically when visible
<Link href="/about">About</Link>

// Disable prefetch for rarely-clicked or expensive routes
<Link href="/admin" prefetch={false}>Admin</Link>

// Dynamic href + replace (no history entry) + scroll control
<Link href={`/blog/${slug}`} replace scroll={false}>Read</Link>
```

---

## 7. Data Fetching

### 7.1 Fetching in Server Components — just `await` **[I]**

The biggest quality-of-life improvement of the App Router: in a Server Component you fetch data by **awaiting it directly in the component body**. No `useEffect`, no loading-state boilerplate, no separate data-fetching layer. Because the component runs on the server, you can hit a `fetch` URL *or* go straight to your database.

```tsx
// Server Component — fetch from an HTTP API
export default async function Page() {
  const res = await fetch("https://api.example.com/posts")
  if (!res.ok) throw new Error("Failed to load posts")  // → triggers nearest error.tsx
  const posts = await res.json()
  return <PostList posts={posts} />
}
```

```tsx
// Server Component — go straight to the database (no API layer needed at all)
import { db } from "@/lib/db"           // Prisma — see PRISMA_ORM_GUIDE.md
export default async function Page() {
  const users = await db.user.findMany({ orderBy: { createdAt: "desc" } })
  return <UserTable users={users} />
}
```

### 7.2 Parallel vs sequential — killing waterfalls **[I]**

A common performance bug: awaiting requests one after another when they don't depend on each other, creating a "waterfall" where each request waits for the previous to finish. If two fetches are independent, fire them together with `Promise.all`.

```tsx
// ❌ Sequential waterfall — getPosts can't start until getUser finishes
const user = await getUser()
const posts = await getPosts()

// ✅ Parallel — both start immediately, you wait once for the slower of the two
const [user, posts] = await Promise.all([getUser(), getPosts()])
```

When fetches *do* depend on each other (you need the user's ID to fetch their posts), sequential is correct — you can't parallelize a true dependency.

### 7.3 Request memoization — automatic dedupe **[I]**

If two different components in the same render both call `fetch("/api/user")` with identical arguments, Next.js **deduplicates** them automatically — the request fires once and both get the result. This is **request memoization** (one of the four caches in §8) and it means you can fetch the same data wherever you need it without manually hoisting it into a parent and prop-drilling. It applies per render pass and only to `fetch` (for DB calls, use React's `cache()` helper to get the same dedupe).

```tsx
import { cache } from "react"
import { db } from "@/lib/db"

// Wrap a DB function in cache() so repeated calls in one render hit the DB once.
export const getUser = cache(async (id: string) => {
  return db.user.findUnique({ where: { id } })
})
```

### 7.4 Client-side fetching — when and how **[I]**

You fetch on the *client* when the data changes after the initial load and you need to refetch, poll, paginate, or optimistically update from the browser — a live search, an infinite feed, a dashboard that auto-refreshes. **Do not hand-roll this with `useEffect` + `fetch`** for anything non-trivial; you'll reinvent caching, deduping, retries, and stale-while-revalidate badly. Use **TanStack Query** (see `TANSTACK_QUERY_GUIDE.md`), which is the standard client server-state library. For purely client-side UI state (modals, toggles), reach for `useState` or Zustand (`ZUSTAND_GUIDE.md`) — not a data-fetching library.

A powerful hybrid pattern: fetch the initial data on the server (fast first paint, SEO), then hand it to TanStack Query as `initialData` so the client takes over for refetching.

---

## 8. Caching — The Four Layers Explained

Caching is where Next.js newcomers get most confused, because there isn't *one* cache — there are **four distinct layers**, each with its own scope, lifetime, and default. Once you can name them and know their defaults, the behavior stops being surprising. Here they are, ordered from most ephemeral to most persistent.

| Cache | Where it lives | What it caches | Lifetime | Default in 16 |
|---|---|---|---|---|
| **Request Memoization** | Server, in-memory | Duplicate `fetch()` calls within **one render pass** | A single request/render | **On** (automatic) |
| **Data Cache** | Server, persistent | `fetch()` (and `use cache`) **results across requests** | Until revalidated | **Off** for `fetch` by default — opt in |
| **Full Route Cache** | Server, persistent | The rendered **HTML/RSC payload of static routes** | Until rebuild/revalidate | **On** for static routes |
| **Router Cache** | Client, in-memory | Visited **route segments** (for instant back/forward) | Short-lived (session) | **On** |

### 8.1 Request Memoization

Covered in §7.3 — automatic dedupe of identical `fetch` calls in a single render. You don't manage it; you just benefit. It's why fetching the same resource in three components doesn't make three network calls.

### 8.2 The Data Cache — the one you actively control **[I/A]**

The Data Cache persists `fetch` results **across requests and deployments**, so the second visitor (and the second request) doesn't re-fetch. This is the cache that turns a route static or ISR.

> **⚡ Version note:** Since Next.js 15 (and continuing in 16), **`fetch` is NOT cached by default** — every `fetch` is treated as dynamic (`no-store`) unless you opt in. This reversed the Next.js 14 behavior where `fetch` cached by default and surprised everyone with stale data. The rule now: **be explicit about caching.**

```tsx
fetch(url)                                  // NOT cached (dynamic) — fetched every request
fetch(url, { cache: "force-cache" })        // cache indefinitely (static, like SSG)
fetch(url, { cache: "no-store" })           // explicitly never cache (always fresh)
fetch(url, { next: { revalidate: 60 } })    // ISR — cache, then refresh in background after 60s
fetch(url, { next: { tags: ["posts"] } })   // tag this data for targeted on-demand revalidation
```

**A worked ISR walkthrough.** Imagine a product page with `fetch(url, { next: { revalidate: 60 } })`. Here's the exact behavior, which is what "stale-while-revalidate" means in practice:

1. First request after deploy → no cached data, so Next.js fetches, renders, and **stores** the result. The user waits for the fetch.
2. Every request in the next 60 seconds → served **instantly from cache**. No fetch, no waiting. The data is "fresh."
3. The first request *after* 60 seconds → the user **still gets the stale cached version instantly** (no waiting), and Next.js kicks off a background re-fetch + re-render.
4. Once that background regeneration finishes, the cache is updated. The *next* visitor gets the new version.

So nobody ever waits for a regeneration (except the very first cold request) — they always get a fast cached page, and the content trails reality by at most ~60 seconds. That trade-off (slightly stale, always fast) is why ISR is the default choice for content that changes occasionally.

### 8.3 The Full Route Cache

When a route renders to fully static output, Next.js caches the **rendered result** (HTML + the RSC payload) at build time and serves it from the cache — no re-render per request. A route becomes static automatically when it uses no dynamic APIs (`cookies()`, `headers()`, uncached `fetch`, etc.). Using any dynamic API "opts the route out" of the Full Route Cache and makes it render per request.

The Full Route Cache sits *on top of* the Data Cache: if the route is static, its rendered HTML is cached; the Data Cache underneath it caches the raw fetch results. A `revalidate` (time-based or on-demand) busts both the cached data *and* the cached rendered route, so the next request re-renders with fresh data.

### 8.4 The Router Cache (client) **[I]**

On the client, when you navigate between routes, Next.js keeps the rendered segments in an in-memory cache so going back/forward (or returning to a recently visited route) is instant — no refetch, no flash. This is why `<Link>` navigations feel immediate. After a mutation you sometimes need to bust it with `router.refresh()` or `revalidatePath` so the user sees fresh data.

### 8.5 On-demand revalidation — `revalidatePath` & `revalidateTag` **[I/A]**

ISR's time-based `revalidate` is great for "refresh every N seconds." But often you want to refresh **exactly when something changes** — right after a user edits a post, purge the cache for the blog. That's **on-demand revalidation**, used from Server Actions or Route Handlers.

```tsx
import { revalidatePath, revalidateTag } from "next/cache"

revalidatePath("/blog")        // invalidate the cache for this specific path
revalidatePath("/blog/[slug]", "page")  // invalidate all pages of a dynamic route
revalidateTag("posts")         // invalidate every fetch tagged { next: { tags: ["posts"] } }
```

Tags are the more powerful tool: tag related fetches across many routes with the same tag, then one `revalidateTag("posts")` refreshes them all wherever they appear.

### 8.6 `use cache` — Cache Components (the modern model) **[A]**

> **⚡ Version note:** **Cache Components** (the `use cache` directive — the productionized successor to Partial Prerendering / PPR) is the flagship caching feature of the Next.js 16 era. You enable it in config (e.g. `cacheComponents: true` in `next.config.ts` — confirm the exact flag for your version) and then mark functions or components as cacheable with a `"use cache"` directive. This lets Next.js render a **static shell instantly** and **stream in the dynamic, uncached parts** as they resolve — all in a single response — and cache at the component/function granularity rather than the whole route.

```tsx
// A cached data function — its result is stored in the Data Cache.
import { unstable_cacheLife as cacheLife, unstable_cacheTag as cacheTag } from "next/cache"

async function getProducts() {
  "use cache"                    // ← this function's output is cached
  cacheLife("hours")             // how long the cache is fresh ("seconds"/"minutes"/"hours"/…)
  cacheTag("products")           // tag for on-demand revalidateTag("products")
  return await db.product.findMany()
}
```

### 8.7 Practical caching rules of thumb

- **Static marketing/docs content** → cache it (`force-cache` or static) for max speed.
- **Per-user / authenticated content** → don't cache (dynamic); it's personalized.
- **Content that changes occasionally** → ISR (`revalidate: N`) for fresh-but-fast.
- **Right after a write** → `revalidatePath`/`revalidateTag` to purge stale data immediately.
- **When unsure** → assume nothing is cached and opt in deliberately. Explicit beats surprising.

---

## 9. Rendering Strategies

Next.js decides *how* each route renders based on what it touches, and you can override that decision. The four strategies map onto the concepts from §1.2.

| Strategy | What happens | Best for | How to trigger |
|---|---|---|---|
| **Static (SSG)** | Rendered at **build time** to HTML, served from cache/CDN. | Marketing, blogs, docs — same for everyone. | Default when the route uses no dynamic APIs. |
| **Dynamic (SSR)** | Rendered **per request** on the server. | Personalized/auth pages, always-fresh data. | Using `cookies()`, `headers()`, uncached `fetch`, or `export const dynamic = "force-dynamic"`. |
| **ISR** | Static, **regenerated in the background** on a schedule or on demand. | Content that changes occasionally. | `export const revalidate = 60` or `fetch(url, { next: { revalidate: 60 } })`. |
| **Streaming** | HTML sent in chunks as data resolves; fast parts paint first. | Pages with one slow widget. | `loading.tsx` + `<Suspense>` (§11). |
| **PPR / Cache Components** | Static shell + streamed dynamic holes in **one** response. | The hybrid sweet spot. | `use cache` + the config flag (§8.6). |

### 9.1 Route segment config **[I]**

Each `page.tsx` / `layout.tsx` can export config constants that override the automatic decision. These are the main levers:

```tsx
// Force this route to render dynamically (per request), even if it looks static.
export const dynamic = "force-dynamic"   // "auto" | "force-dynamic" | "force-static" | "error"

// ISR: regenerate the static page in the background every 3600s. false = never.
export const revalidate = 3600

// Default cache behavior for fetches in this route.
export const fetchCache = "force-cache"  // among others; rarely needed if you set per-fetch

// Where this route runs.
export const runtime = "nodejs"          // "nodejs" (full Node APIs) | "edge" (low-latency, limited APIs)

// Geographic hint for edge deployments.
export const preferredRegion = "auto"
```

`dynamic = "force-static"` is handy to force-cache a route that would otherwise be dynamic; `dynamic = "error"` makes the build *fail* if anything dynamic sneaks in, which is a good guardrail for routes you intend to be fully static.

---

## 10. Server Actions & Mutations

### 10.1 What they are and why they exist **[I/A]**

A **Server Action** is an `async` function that runs **on the server** but can be **called from your client UI** — invoked directly by a `<form>`'s `action` prop or called from a client event handler — *without you writing an API route, a fetch call, or any wiring*. You mark a function (or a whole file) with `"use server"`, and Next.js handles the rest: it creates a hidden endpoint, serializes the arguments, runs your function on the server, and returns the result.

The "why" is ergonomics and progressive enhancement. Before Server Actions, every mutation meant: write a Route Handler, write client-side `fetch` code, manage loading/error state by hand, and re-validate caches manually. Server Actions collapse that into a single typed function. And because they integrate with native `<form>` submission, **forms work even before JavaScript loads** — the form posts to the action the old-fashioned way, then upgrades to a fetch once hydrated (progressive enhancement).

### 10.2 A complete form + action **[I]**

```tsx
// app/actions/posts.ts
"use server"                                 // marks every export as a Server Action
import { revalidatePath } from "next/cache"
import { redirect } from "next/navigation"
import { z } from "zod"                       // validation — never trust client input
import { db } from "@/lib/db"
import { auth } from "@/lib/auth"             // your session helper

const PostSchema = z.object({
  title: z.string().min(1).max(120),
  body: z.string().min(1),
})

export async function createPost(formData: FormData) {
  // 1) AUTHORIZE — a Server Action is a public endpoint; check the session yourself.
  const session = await auth()
  if (!session) throw new Error("Unauthorized")

  // 2) VALIDATE — parse and reject bad input before touching the DB.
  const parsed = PostSchema.safeParse({
    title: formData.get("title"),
    body: formData.get("body"),
  })
  if (!parsed.success) {
    return { error: "Invalid input", issues: parsed.error.flatten() }
  }

  // 3) MUTATE
  await db.post.create({ data: { ...parsed.data, authorId: session.userId } })

  // 4) REVALIDATE the cache so the list reflects the new post
  revalidatePath("/blog")

  // 5) NAVIGATE away (redirect throws internally — put it last)
  redirect("/blog")
}
```

```tsx
// app/blog/new/page.tsx — wire the action straight into the form's action prop
import { createPost } from "@/app/actions/posts"

export default function NewPost() {
  return (
    <form action={createPost}>          {/* works without JS — progressive enhancement */}
      <input name="title" required />
      <textarea name="body" required />
      <button type="submit">Create</button>
    </form>
  )
}
```

### 10.3 Pending state, validation feedback & optimistic UI **[I/A]**

For a polished form you want a disabled button while submitting and inline error messages. React 19 gives you `useActionState` (action + returned state + pending flag) and `useFormStatus` (pending state of the enclosing form). See `REACT_19_GUIDE.md` for the React-side detail.

```tsx
"use client"
import { useActionState } from "react"
import { useFormStatus } from "react-dom"
import { createPost } from "@/app/actions/posts"

// useFormStatus reads the pending state of the nearest parent <form>.
function SubmitButton() {
  const { pending } = useFormStatus()
  return <button disabled={pending}>{pending ? "Saving…" : "Save"}</button>
}

export default function PostForm() {
  // useActionState wires the action and exposes its returned value + a pending flag.
  const [state, formAction, pending] = useActionState(createPost, null)
  return (
    <form action={formAction}>
      <input name="title" />
      {state?.error && <p className="text-red-600">{state.error}</p>}
      <SubmitButton />
    </form>
  )
}
```

For instant feedback before the server responds, `useOptimistic` lets you render the expected result immediately and reconcile when the action resolves (covered in `REACT_19_GUIDE.md`).

### 10.4 Calling actions outside a form & binding arguments **[I]**

Not every mutation is a form submit. You can call a Server Action straight from a Client Component's event handler — it's just an async function. And when an action needs an argument that *isn't* a form field (a row's `id`, say), you pass it with `.bind()`, which pre-fills the first parameter on the server side. Binding (rather than putting the id in a hidden input) is safer because hidden inputs are trivially editable in DevTools, whereas a bound argument is fixed at render time on the server.

```tsx
// The action takes an id (bound) PLUS the FormData (passed by the form).
// app/actions/posts.ts
"use server"
export async function deletePost(id: string) {     // single-arg action, no form needed
  const session = await auth()
  if (!session) throw new Error("Unauthorized")
  const post = await db.post.findUnique({ where: { id } })
  if (post?.authorId !== session.userId) throw new Error("Forbidden")
  await db.post.delete({ where: { id } })
  revalidatePath("/blog")
}
```

```tsx
// Calling from an event handler in a Client Component
"use client"
import { useTransition } from "react"
import { deletePost } from "@/app/actions/posts"

export default function DeleteButton({ id }: { id: string }) {
  const [pending, startTransition] = useTransition()   // keeps the UI responsive during the call
  return (
    <button
      disabled={pending}
      onClick={() => startTransition(() => deletePost(id))}   // just call the async action
    >
      {pending ? "Deleting…" : "Delete"}
    </button>
  )
}
```

```tsx
// Binding an extra argument to a form action with .bind()
import { updatePost } from "@/app/actions/posts"

export function EditForm({ id }: { id: string }) {
  // updatePost(id, formData) — id is bound on the server, formData comes from the <form>
  const updateWithId = updatePost.bind(null, id)
  return (
    <form action={updateWithId}>
      <input name="title" />
      <button>Save</button>
    </form>
  )
}
```

### 10.5 Server Actions vs Route Handlers — which to use **[I]**

- **Server Actions** → mutations triggered from *your own UI* (form submits, button clicks): simpler, typed end-to-end, no fetch plumbing, progressive enhancement.
- **Route Handlers** (§13) → *public/programmatic* APIs: webhooks, third-party integrations, mobile clients, anything that isn't "my own React form."

### 10.6 Security — treat actions as public endpoints **[A]**

This is the most important and most missed point. A Server Action compiles to a **publicly reachable POST endpoint**. Anyone can call it with any arguments — the form UI is not a gate. Therefore, inside every action you **must**:

1. **Authenticate & authorize** — verify the session and that *this* user may perform *this* action on *this* resource. Never assume the caller is a legitimate logged-in user.
2. **Validate every input** with a schema (Zod). The `FormData` (or arguments) are attacker-controlled.
3. **Never leak secrets** in returned values or error messages.
4. **Avoid passing sensitive IDs you trust blindly** — re-derive ownership from the session, don't take `userId` from the form.

If you skip auth/validation in an action, you've shipped an open, unauthenticated mutation endpoint.

---

## 11. Loading, Streaming & Suspense

### 11.1 Why streaming matters **[I]**

Traditionally a server-rendered page is all-or-nothing: the server waits for *every* data dependency before sending a single byte, so one slow query holds the whole page hostage. **Streaming** breaks that: Next.js sends the fast parts of the page immediately and streams the slow parts in as their data resolves. The user sees the header, nav, and skeletons instantly, then the slow widget pops in — a huge *perceived* performance win even when total time is unchanged.

### 11.2 `loading.tsx` — page-level streaming for free **[I]**

Drop a `loading.tsx` into a route folder and Next.js automatically wraps that segment's `page` in a `<Suspense>` boundary, showing your loading UI while the page's data resolves. Zero extra code at the call site.

```tsx
// app/dashboard/loading.tsx — shown instantly while dashboard/page.tsx loads its data
export default function Loading() {
  return <DashboardSkeleton />     // a skeleton always beats a blank screen
}
```

### 11.3 Granular streaming with `<Suspense>` **[I]**

For finer control, wrap individual slow components in `<Suspense>` yourself. Everything outside the boundary renders immediately; the wrapped component streams in when ready. This is how you keep one slow query from blocking the rest of the page.

```tsx
import { Suspense } from "react"

export default function Page() {
  return (
    <>
      <Header />                              {/* renders instantly */}
      <Suspense fallback={<Skeleton />}>
        <SlowProductList />                    {/* its own data fetch; streams in when done */}
      </Suspense>
      <Suspense fallback={<Skeleton />}>
        <SlowRecommendations />                {/* independent — won't block the list */}
      </Suspense>
    </>
  )
}
```

The component inside Suspense should be a Server Component that fetches its own data (so it can "suspend"). This pattern + parallel fetching (§7.2) is the backbone of fast Next.js pages.

---

## 12. Error Handling

Next.js gives you **declarative error boundaries via files**. An `error.tsx` in a route segment catches any error thrown by the components below it and renders recovery UI instead of crashing the whole app. It must be a **Client Component** (error boundaries need client-side React) and it receives the `error` and a `reset` function to retry the segment.

```tsx
// app/dashboard/error.tsx — MUST be a Client Component
"use client"
import { useEffect } from "react"

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }   // digest is a hashed id for server-log correlation
  reset: () => void                     // re-render the segment to retry
}) {
  useEffect(() => {
    // log to your error-reporting service (Sentry, etc.)
    console.error(error)
  }, [error])

  return (
    <div>
      <h2>Something went wrong</h2>
      <button onClick={() => reset()}>Try again</button>
    </div>
  )
}
```

- **`error.tsx`** catches errors in its segment and children — but *not* in the same-level layout (the layout is "above" the boundary). Put a deeper `error.tsx` if you need to catch layout-level issues, or use `global-error.tsx`.
- **`global-error.tsx`** is the last resort — it catches errors in the **root layout** and must render its own `<html>`/`<body>` because it replaces the whole document.
- **`notFound()`** (from `next/navigation`) throws a special error that renders the nearest `not-found.tsx` — use it for "this record doesn't exist."

```tsx
import { notFound } from "next/navigation"

const post = await getPost(slug)
if (!post) notFound()        // renders the nearest not-found.tsx, sends a 404
```

> **Security note:** in production, errors thrown in Server Components are **sanitized** — the message and stack are *not* sent to the browser (you get a generic message plus the `digest` to find it in your server logs). This prevents leaking internal details like SQL or file paths. Don't rely on `error.message` showing the real cause to users in production.

---

## 13. Route Handlers (the API layer)

When you need a real HTTP endpoint — a webhook receiver, a public JSON API, an endpoint for a mobile app, a file upload handler — you write a **Route Handler**: a `route.ts` file exporting functions named after HTTP methods. A folder can have *either* a `page.tsx` *or* a `route.ts`, not both.

```ts
// app/api/users/route.ts  →  /api/users
import { NextRequest, NextResponse } from "next/server"
import { z } from "zod"
import { db } from "@/lib/db"

// GET /api/users?limit=10
export async function GET(req: NextRequest) {
  const limit = Number(req.nextUrl.searchParams.get("limit") ?? 20)
  const users = await db.user.findMany({ take: limit })
  return NextResponse.json(users)
}

// POST /api/users  with a JSON body
const CreateUser = z.object({ name: z.string().min(1), email: z.string().email() })

export async function POST(req: NextRequest) {
  const body = await req.json()
  const parsed = CreateUser.safeParse(body)        // validate — never trust the body
  if (!parsed.success) {
    return NextResponse.json({ error: parsed.error.flatten() }, { status: 400 })
  }
  const user = await db.user.create({ data: parsed.data })
  return NextResponse.json(user, { status: 201 })
}
```

Dynamic API routes get their params the same way pages do (awaited):

```ts
// app/api/users/[id]/route.ts  →  /api/users/123
export async function GET(req: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const user = await db.user.findUnique({ where: { id } })
  if (!user) return NextResponse.json({ error: "Not found" }, { status: 404 })
  return NextResponse.json(user)
}
```

Supported method exports: `GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS`.

> **⚡ Version note:** Since Next.js 15, **GET Route Handlers are NOT cached by default** (they were in 14). If a GET endpoint returns cacheable data, opt in with `export const dynamic = "force-static"` or a `revalidate` export.

> **Security:** validate every input, authenticate/authorize on each handler (they're public), set CORS deliberately (`OPTIONS` + headers) only if cross-origin access is intended, and for webhooks **verify the signature** the provider sends before trusting the payload.

---

## 14. Metadata & SEO

SEO is one of the top reasons clients pick Next.js, and the **Metadata API** makes it clean and type-safe. You export a `metadata` object (static) or a `generateMetadata` function (dynamic) from a `layout` or `page`, and Next.js renders the right `<head>` tags — title, description, Open Graph, Twitter cards, canonical URLs — into the server HTML where crawlers can see them.

### 14.1 Static metadata **[I]**

```tsx
import type { Metadata } from "next"

export const metadata: Metadata = {
  title: "Acme — Modern Websites",
  description: "We build fast, modern websites.",
  keywords: ["web design", "next.js"],
  openGraph: {                              // controls Facebook/LinkedIn/etc. link previews
    title: "Acme",
    description: "Fast modern websites.",
    url: "https://acme.com",
    images: ["/og.png"],
    type: "website",
  },
  twitter: { card: "summary_large_image", images: ["/og.png"] },
  alternates: { canonical: "https://acme.com" },   // canonical URL to avoid duplicate-content penalties
}
```

### 14.2 Dynamic metadata + title templates **[I]**

```tsx
// Per-page metadata derived from the route's data
export async function generateMetadata({
  params,
}: {
  params: Promise<{ slug: string }>
}): Promise<Metadata> {
  const { slug } = await params
  const post = await getPost(slug)
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: { images: [post.coverImage] },
  }
}
```

```tsx
// In the root layout — child pages fill the %s
export const metadata = {
  title: { default: "Acme", template: "%s | Acme" },  // a page titled "About" → "About | Acme"
}
```

### 14.3 sitemap, robots & OG images **[I]**

Next.js generates `sitemap.xml`, `robots.txt`, and dynamic social images from special files:

```ts
// app/sitemap.ts  →  /sitemap.xml
import type { MetadataRoute } from "next"
export default function sitemap(): MetadataRoute.Sitemap {
  return [
    { url: "https://acme.com", lastModified: new Date(), priority: 1 },
    { url: "https://acme.com/blog", changeFrequency: "weekly", priority: 0.8 },
  ]
}
```

```ts
// app/robots.ts  →  /robots.txt
import type { MetadataRoute } from "next"
export default function robots(): MetadataRoute.Robots {
  return { rules: { userAgent: "*", allow: "/" }, sitemap: "https://acme.com/sitemap.xml" }
}
```

```tsx
// app/opengraph-image.tsx — generate a dynamic OG image with ImageResponse
import { ImageResponse } from "next/og"
export const size = { width: 1200, height: 630 }
export const contentType = "image/png"
export default function OG() {
  return new ImageResponse(
    <div style={{ fontSize: 64, background: "white", width: "100%", height: "100%", display: "flex", alignItems: "center", justifyContent: "center" }}>
      Acme
    </div>,
    size,
  )
}
```

### 14.4 SEO checklist **[I]**

- ✅ Unique `title` + `description` per page (`generateMetadata` for dynamic pages).
- ✅ Open Graph + Twitter cards for shareable previews.
- ✅ `sitemap.ts` and `robots.ts` present.
- ✅ Canonical URLs via `alternates.canonical`.
- ✅ Structured data (JSON-LD) in a `<script type="application/ld+json">` for rich results.
- ✅ Semantic HTML — one `<h1>`, proper landmarks, `alt` text on images.
- ✅ Static/ISR rendering so crawlers get full HTML instantly.

---

## 15. Images, Fonts & the `<Link>` Component

These three built-in components deliver "free" Core Web Vitals improvements — use them everywhere instead of their raw HTML equivalents.

### 15.1 `next/image` **[B/I]**

The native `<img>` tag ships full-size images, doesn't lazy-load, and causes layout shift (the page jumps when the image loads). `next/image` fixes all of that: it automatically resizes images, serves modern formats (AVIF/WebP), lazy-loads off-screen images, and **reserves space** (via `width`/`height` or `fill`) so there's no layout shift. You must give it dimensions — either explicit `width`/`height`, or `fill` with a positioned parent.

```tsx
import Image from "next/image"

// Known dimensions
<Image
  src="/hero.jpg"
  alt="Team at work"                 // alt is required (a11y + SEO)
  width={1200}
  height={600}
  priority                            // preload — use on above-the-fold/LCP images ONLY
  className="rounded-xl object-cover"
/>

// Unknown size → fill a positioned container, give a sizes hint for responsive loading
<div className="relative h-64 w-full">
  <Image src="/cover.jpg" alt="" fill sizes="100vw" className="object-cover" />
</div>
```

For external image hosts, you must whitelist their domains (a security measure preventing your optimizer from being abused as an open proxy):

```ts
// next.config.ts
const nextConfig = {
  images: { remotePatterns: [{ protocol: "https", hostname: "images.unsplash.com" }] },
}
```

### 15.2 `next/font` **[B/I]**

`next/font` **self-hosts fonts at build time** — it downloads Google Fonts (or loads your local files) into your own deployment, so there's no request to Google at runtime (better privacy and speed) and no layout shift (it computes fallback metrics). Apply the font via a CSS variable so Tailwind and your styles can reference it.

```tsx
// app/layout.tsx
import { Inter } from "next/font/google"
const inter = Inter({ subsets: ["latin"], variable: "--font-inter", display: "swap" })

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={inter.variable}>
      <body className="font-sans">{children}</body>   {/* Tailwind reads --font-inter */}
    </html>
  )
}
```

### 15.3 `next/link`

Covered in §6.3 — use `<Link>` for all internal navigation to get soft client-side transitions and automatic prefetching. Use a plain `<a>` only for external links.

---

## 16. Styling

| Option | When to use | Notes |
|---|---|---|
| **Tailwind CSS** | Default & recommended for almost everything. | Utility classes, works perfectly with Server Components (no runtime). See `TAILWIND_CHEATSHEET.md`. |
| **CSS Modules** | Scoped component styles when you prefer real CSS. | `Button.module.css` → locally-scoped class names, no collisions. |
| **Global CSS** | Resets, base styles, CSS variables. | `globals.css`, imported **once** in the root layout. |
| **CSS-in-JS** | Legacy / specific needs. | Runtime cost; many libraries need a `"use client"` boundary and don't play well with RSC. Avoid for new work. |

The winning combo for client work is **Tailwind + shadcn/ui** (see `SHADCN_UI_CHEATSHEET.md`): consistent, fast, easy to theme, and zero styling runtime in the browser.

---

## 17. Middleware

**Middleware** is code that runs **before a request is completed**, for *every* matching request, *before* the route renders. It's the right place for cross-cutting concerns that must happen early: auth gating (redirect anonymous users away from `/dashboard`), redirects/rewrites, geolocation-based routing, A/B test bucketing, and setting request headers/cookies. It lives in a single `middleware.ts` at the project root (next to `app/`).

```ts
// middleware.ts
import { NextRequest, NextResponse } from "next/server"

export function middleware(req: NextRequest) {
  const token = req.cookies.get("session")?.value

  // Gate the dashboard: bounce unauthenticated users to /login
  if (!token && req.nextUrl.pathname.startsWith("/dashboard")) {
    const loginUrl = new URL("/login", req.url)
    loginUrl.searchParams.set("from", req.nextUrl.pathname)   // remember where they were going
    return NextResponse.redirect(loginUrl)
  }

  return NextResponse.next()        // continue to the route normally
}

// matcher limits WHICH paths run the middleware — keep it tight for performance
export const config = {
  matcher: ["/dashboard/:path*", "/account/:path*"],
}
```

> **Security caveat — middleware is a coarse gate, not the only one.** Middleware is great for redirecting unauthenticated users for UX, but **do not treat it as your sole authorization layer.** Always re-check auth in the Server Component / Server Action / Route Handler that actually accesses data — defense in depth. A misconfigured matcher or an edge case can let a request slip past middleware, so the data layer must protect itself too.

> **⚡ Version note:** Next.js 16 supports the **Node.js runtime for middleware** (not just the Edge runtime), so you can use more Node APIs (e.g. some auth libraries) directly in middleware. There's also an evolving `proxy` capability — confirm specifics for your version.

---

## 18. Environment Variables & Config

Environment variables hold configuration that differs between environments (dev/staging/prod) and secrets that must not live in source control. Next.js reads them from `.env.local` (and `.env`, `.env.production`, etc.). The single most important security rule here:

> **`NEXT_PUBLIC_`-prefixed variables are inlined into the browser bundle. Everything else is server-only.**

```bash
# .env.local  (gitignored by default — NEVER commit secrets)
DATABASE_URL="postgres://user:pass@host/db"     # server-only (no prefix) — safe for secrets
STRIPE_SECRET_KEY="sk_live_…"                   # server-only — NEVER expose
NEXT_PUBLIC_API_URL="https://api.acme.com"      # exposed to the browser — public info ONLY
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY="pk_live_…"  # publishable key is meant to be public — OK
```

- Variables **without** `NEXT_PUBLIC_` are available only in Server Components, Server Actions, Route Handlers, and middleware — they never reach the browser, so secrets are safe there.
- Variables **with** `NEXT_PUBLIC_` are **literally substituted into the JavaScript sent to the browser at build time.** Anyone can read them in DevTools. Put *only* non-secret values there (public API base URLs, publishable keys). **Putting a secret behind `NEXT_PUBLIC_` is a leak**, and a depressingly common one.

```ts
// next.config.ts — typed configuration
import type { NextConfig } from "next"

const nextConfig: NextConfig = {
  images: { remotePatterns: [{ protocol: "https", hostname: "cdn.acme.com" }] },
  // experimental: { cacheComponents: true },   // enable Cache Components (§8.6) as needed
}
export default nextConfig
```

> **Best practice:** validate your env vars at startup with a Zod schema so a missing/misnamed variable fails loudly at boot instead of producing a confusing runtime error later.

---

## 19. Authentication Patterns

Auth in the App Router has a few moving parts. The clean mental model is **three layers**: (1) a session/login mechanism, (2) reading the session in your server code, (3) protecting routes and data.

### 19.1 Choosing an approach **[A]**

- **A library** — Auth.js (NextAuth), Clerk, or **Better Auth** (see `BETTERAUTH_GUIDE.md`) handle the hard, security-sensitive parts (OAuth flows, session cookies, CSRF). Strongly preferred over rolling your own.
- **Roll your own** — only if you must. You'll manage password hashing (argon2/bcrypt — never plaintext), secure session cookies (`httpOnly`, `secure`, `sameSite`), and CSRF. Easy to get subtly wrong.

### 19.2 Reading the session on the server **[A]**

Wrap your library's session read in a small helper and call it wherever you need the user. Use React's `cache()` so multiple calls in one render hit the session store once.

```ts
// lib/auth.ts
import { cache } from "react"
import { cookies } from "next/headers"

export const auth = cache(async () => {
  const cookieStore = await cookies()           // cookies() is async now
  const token = cookieStore.get("session")?.value
  if (!token) return null
  return verifyAndLoadSession(token)             // your library / custom verify
})
```

### 19.3 Protecting routes & data **[A]**

Defense in depth — gate at multiple layers:

```tsx
// In a Server Component or layout — redirect if not logged in
import { redirect } from "next/navigation"
import { auth } from "@/lib/auth"

export default async function DashboardPage() {
  const session = await auth()
  if (!session) redirect("/login")     // server-side redirect, before any data leaks
  const data = await getUserData(session.userId)
  return <Dashboard data={data} />
}
```

```ts
// In a Server Action / Route Handler — re-check, never trust the client
export async function deletePost(id: string) {
  "use server"
  const session = await auth()
  if (!session) throw new Error("Unauthorized")
  const post = await db.post.findUnique({ where: { id } })
  if (post?.authorId !== session.userId) throw new Error("Forbidden")  // ownership check
  await db.post.delete({ where: { id } })
}
```

### 19.4 Security checklist **[A]**

- Session cookies: `httpOnly` (JS can't read them — blocks XSS token theft), `secure` (HTTPS only), `sameSite: "lax"` or `"strict"` (CSRF mitigation).
- **Authorize in the data layer**, not just middleware — middleware is for UX redirects, the Server Action/Component is the real gate.
- Hash passwords with argon2id or bcrypt; never store or log plaintext.
- Re-derive the user identity from the session, not from client-supplied IDs.
- Rate-limit login and sensitive actions.

---

## 20. Performance Optimization

Most Next.js performance wins come from *defaults done right* plus a handful of deliberate techniques. The big levers, roughly in order of impact:

| Technique | Why it helps |
|---|---|
| **Server Components by default** | Less JS shipped → faster load and better INP. Keep client components to interactive leaves. |
| **Static / ISR rendering** | Pre-rendered HTML served from CDN — near-instant. |
| **Streaming + `<Suspense>`** | One slow query no longer blocks the whole page; content paints progressively. |
| **`next/image`** | Resizing, lazy-loading, modern formats, no layout shift (CLS). |
| **`next/font`** | Self-hosted fonts, no layout shift, no third-party request. |
| **Parallel fetching (`Promise.all`)** | Eliminates request waterfalls. |
| **`next/dynamic` for heavy client widgets** | Lazy-load charts/maps/editors so they don't bloat the initial bundle. |
| **`<Link>` prefetching** | Destination is loaded before the click. |
| **React Compiler** | Auto-memoization → fewer wasted re-renders without manual `useMemo`. |
| **Bundle analysis** | `@next/bundle-analyzer` to find and trim bloat. |

### 20.1 Lazy-loading client components **[I]**

```tsx
import dynamic from "next/dynamic"

// A heavy chart library: don't put it in the initial bundle; load it on demand,
// and skip SSR for a browser-only library.
const Chart = dynamic(() => import("@/components/Chart"), {
  ssr: false,
  loading: () => <div className="h-64 animate-pulse rounded bg-gray-200" />,
})
```

### 20.2 Core Web Vitals targets **[I]**

These are what Google and clients measure:

- **LCP (Largest Contentful Paint) < 2.5s** — optimize the hero image with `next/image priority`; render static.
- **CLS (Cumulative Layout Shift) < 0.1** — always set image/font dimensions; use `next/image` and `next/font`.
- **INP (Interaction to Next Paint) < 200ms** — ship less client JS; keep logic on the server.

Audit with Lighthouse (in Chrome DevTools) and aim for 90+ across Performance, SEO, Accessibility, and Best Practices before handing off.

---

## 21. Deployment

| Host | Notes |
|---|---|
| **Vercel** | Built by the Next.js team. Zero-config, automatic preview deploys per push, native ISR/edge/Cache Components support. The default for client work. |
| **Netlify / Cloudflare** | Good Next.js support via adapters. |
| **Self-host (Node)** | `next build && next start` behind a reverse proxy (nginx/Caddy). Use `output: "standalone"` for a minimal Docker image (see `DOCKER_GUIDE.md`). |
| **Static export** | `output: "export"` produces a fully static site — but you lose SSR, ISR, Server Actions, and Route Handlers. Only for purely static content. |

**The standard Vercel workflow:** push to GitHub → import the repo into Vercel → set environment variables in the dashboard → every push to a branch gets a unique **preview URL** (great for sharing with clients/reviewers), and merges to `main` deploy to production. Set all `NEXT_PUBLIC_` and secret env vars in the host's dashboard, not in committed files.

```ts
// next.config.ts — standalone output for Docker self-hosting
const nextConfig = { output: "standalone" }
export default nextConfig
```

---

## 22. Design Patterns & Architecture

Patterns that keep real projects maintainable as they grow:

1. **Server Components for data, Client Components for interactivity.** Fetch and shape on the server; embed small interactive islands. (§4)
2. **Push `"use client"` to the leaves.** Keep pages and layouts on the server; extract interactive bits.
3. **Container / Presentational split.** A Server Component fetches data and passes it to a "dumb" presentational component (which may be client). Testable and reusable.
4. **Composition over the boundary.** Pass Server Components into Client wrappers as `children` (§4.2).
5. **`lib/` for logic.** DB clients, API wrappers, pure helpers — never put data access inside components' render code beyond the await.
6. **`actions/` for mutations.** Group Server Actions; validate every input with Zod; authorize every call.
7. **Colocation with private folders.** Route-only components live in `app/route/_components/`; shared ones in `src/components`.
8. **Route groups for layout boundaries.** `(marketing)` with a public shell, `(app)` with an authenticated shell. (§5.3)
9. **Feature folders for big apps.** Once the app grows, group by feature (`features/billing`, `features/auth`) rather than by file type.
10. **One source of truth for config.** `config/site.ts` for nav, metadata defaults, social links.
11. **State boundaries.** Server data → Server Components / TanStack Query (`TANSTACK_QUERY_GUIDE.md`); client UI state → `useState` / Zustand (`ZUSTAND_GUIDE.md`). Don't mix concerns.

```
src/
├── app/
│   └── (app)/dashboard/
│       ├── page.tsx              # Server Component: fetches data
│       ├── _components/          # private, route-only components
│       │   └── revenue-card.tsx
│       ├── loading.tsx
│       └── error.tsx
├── components/ui/                # shadcn primitives
├── lib/db.ts                     # Prisma client
├── actions/billing.ts            # "use server" mutations (validated + authorized)
└── types/index.ts
```

---

## 23. Gotchas & Best Practices

**Do:**
- ✅ Default to Server Components; add `"use client"` only at the interactive leaves.
- ✅ Be **explicit about caching** — assume nothing is cached, opt in deliberately.
- ✅ Use `loading.tsx` + `<Suspense>` so pages feel instant.
- ✅ Use `next/image` and `next/font` everywhere — free Core Web Vitals wins.
- ✅ `generateStaticParams` + ISR for blogs/catalogs → fast and fresh.
- ✅ **Validate AND authorize** in every Server Action and Route Handler (they're public).
- ✅ Use `Promise.all` to fetch in parallel and kill waterfalls.
- ✅ Set unique metadata per page.

**Don't / common bugs:**
- ❌ **Forgetting to `await`** `params` / `searchParams` / `cookies()` / `headers()` — they're Promises now (since 15).
- ❌ Slapping `"use client"` on a whole page just for one interactive widget.
- ❌ Passing **functions or class instances** as props from Server → Client (must be serializable; use a Server Action instead).
- ❌ Using `useState` / `useEffect` / `window` in a Server Component (runtime error).
- ❌ Expecting `fetch` to be cached automatically (it isn't in 15/16).
- ❌ **Leaking secrets via `NEXT_PUBLIC_`** variables — they ship to the browser.
- ❌ Importing a Server Component *into* a Client Component (pass as `children` instead).
- ❌ **Hydration mismatch** — rendering `Date.now()` / `Math.random()` / locale-dependent output differently on server vs client. Guard browser-only code (`useEffect` or `typeof window !== "undefined"`).
- ❌ Treating middleware as your only auth layer (re-check in the data layer).
- ❌ Eagerly importing heavy client libs (charts/maps) — lazy-load with `next/dynamic`.

**Tricks:**
- 🔹 `router.refresh()` re-fetches server data without losing client state — perfect after a mutation.
- 🔹 Intercepting routes for modals that are still shareable, refreshable URLs.
- 🔹 `app/opengraph-image.tsx` generates per-page social images automatically.
- 🔹 Route groups give marketing vs app sections different layouts with clean URLs.
- 🔹 `export const dynamic = "error"` makes the build fail if a route you intend to be static touches anything dynamic — a great static-guarantee guardrail.
- 🔹 Tag fetches with `next: { tags: [...] }` and use one `revalidateTag` to purge related data across many routes at once.

### 23.1 Quick glossary **[B/I]**

| Term | One-line meaning |
|---|---|
| **App Router** | The modern `app/`-directory routing system built on Server Components. |
| **RSC** | React Server Component — runs only on the server, ships no JS to the browser. |
| **Client Component** | A component marked `"use client"` whose code also runs (and hydrates) in the browser. |
| **RSC Payload** | The serialized description of a rendered server tree sent to the client. |
| **Hydration** | React attaching interactivity to server-rendered HTML in the browser. |
| **SSG** | Static Site Generation — render to HTML at build time. |
| **SSR** | Server-Side Rendering — render per request on the server. |
| **ISR** | Incremental Static Regeneration — static pages re-generated on a schedule/on demand. |
| **PPR / Cache Components** | Static shell + streamed dynamic holes in one response (`use cache`). |
| **Server Action** | A `"use server"` function callable from the client that runs on the server. |
| **Route Handler** | A `route.ts` HTTP endpoint (the API layer). |
| **Data Cache** | Persistent server cache of `fetch` results across requests. |
| **Full Route Cache** | Cached rendered output of static routes. |
| **Router Cache** | Client-side in-memory cache of visited segments for instant navigation. |
| **`revalidate`** | Time-based cache refresh interval (ISR). |
| **`revalidatePath`/`revalidateTag`** | On-demand cache invalidation by path or tag. |
| **Streaming** | Sending HTML/UI in chunks as data resolves (via `<Suspense>`/`loading.tsx`). |
| **Middleware** | Code that runs before a request completes, for every matching path. |
| **`NEXT_PUBLIC_`** | Prefix that exposes an env var to the browser bundle (never for secrets). |

---

## 24. Study Path & Build-to-Learn Projects

**Offline mastery order** — learn in this sequence and each step builds on the last:

1. **Server vs Client Components (§4)** — internalize this *first*; everything else depends on it. Pair with `REACT_19_GUIDE.md`.
2. **File conventions + project structure (§2, §3)** — learn the special files until the router feels obvious.
3. **Routing (§5, §6)** — dynamic routes, layouts, navigation, then groups/parallel/intercepting.
4. **Data fetching + caching (§7, §8)** — the four caches and explicit `fetch` options. This is where most people stay fuzzy; get crisp here.
5. **Rendering strategies (§9)** — connect static/dynamic/streaming/PPR to how you wrote the code.
6. **Server Actions (§10), loading/streaming (§11), errors (§12)** — the mutation + UX loop.
7. **Route Handlers (§13), SEO (§14), images/fonts (§15)** — the API layer and your selling points.
8. **Middleware (§17), env/security (§18), auth (§19)** — protect it properly.
9. **Performance (§20) + deployment (§21)** — ship it fast.
10. **Apply the patterns (§22)** across two or three real projects.

**Build to learn** — this trio covers ~90% of real client work:

- **A blog** — SSG + ISR + dynamic routes (`[slug]`) + `generateStaticParams` + dynamic metadata + a sitemap. Teaches static rendering, caching, and SEO.
- **A dashboard** — authenticated route group, Server Components for data, Server Actions for mutations, `<Suspense>` streaming, parallel routes for panels, and Zustand/TanStack Query for client state. Teaches the full interactive app loop.
- **A landing page** — laser-focused on performance and SEO: `next/image priority`, `next/font`, Lighthouse 90+, OG images, perfect metadata. Teaches the optimization craft clients pay for.

> Cross-reference as you build: `REACT_19_GUIDE.md` (the React layer), `TANSTACK_QUERY_GUIDE.md` (client server-state), `ZUSTAND_GUIDE.md` (client UI state), `TAILWIND_CHEATSHEET.md` (styling), `PRISMA_ORM_GUIDE.md` (the database). Next.js is the framework that ties them all together.
