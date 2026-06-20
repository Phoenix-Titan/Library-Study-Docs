# Next.js 16+ Complete Guide (App Router Era)

A full, study-friendly reference for **Next.js 16+** with the **App Router**. Covers folder structure, rendering models, design patterns, data fetching & caching, SEO, performance, and the tips/tricks that make clients happy.

> **Version note:** This guide targets **Next.js 16+** running **React 19**. Next.js evolves fast, so for any exact API always confirm against the official docs at nextjs.org. The *concepts* here (Server Components, the App Router model, caching philosophy) are stable across 15 → 16. Where something changed recently, it's flagged with **⚡ Version note**.

---

## Table of Contents
1. [What Next.js Is & When to Use It](#1-what-nextjs-is--when-to-use-it)
2. [Creating a Project](#2-creating-a-project)
3. [Folder Structure (The Whole Map)](#3-folder-structure)
4. [The App Router — Special Files](#4-the-app-router--special-files)
5. [Routing Deep Dive](#5-routing-deep-dive)
6. [Server vs Client Components (THE core concept)](#6-server-vs-client-components)
7. [Rendering Strategies (SSR / SSG / ISR / Streaming)](#7-rendering-strategies)
8. [Data Fetching](#8-data-fetching)
9. [Caching Model (the part everyone gets wrong)](#9-caching-model)
10. [Server Actions & Mutations](#10-server-actions--mutations)
11. [Loading, Streaming & Suspense](#11-loading-streaming--suspense)
12. [Error Handling](#12-error-handling)
13. [Metadata & SEO](#13-metadata--seo)
14. [Performance & Optimization](#14-performance--optimization)
15. [Images, Fonts & Assets](#15-images-fonts--assets)
16. [Styling](#16-styling)
17. [Middleware & Proxy](#17-middleware--proxy)
18. [Environment Variables & Config](#18-environment-variables--config)
19. [Design Patterns & Project Architecture](#19-design-patterns--project-architecture)
20. [Deployment](#20-deployment)
21. [Tips, Tricks & Gotchas](#21-tips-tricks--gotchas)
22. [Client-Pleasing Checklist](#22-client-pleasing-checklist)

---

## 1. What Next.js Is & When to Use It

Next.js is a **React framework** that adds: file-based routing, server-side rendering, a build system, image/font optimization, API/backend capabilities, and great SEO out of the box.

**Use Next.js when the client needs:**
- A marketing site / landing page that must rank on Google (SEO) and load fast.
- A web app with both frontend and some backend logic (forms, auth, dashboards).
- E-commerce, blogs, SaaS, portfolios — anything where speed + SEO matter.

**Two routers exist:**
- **App Router** (`app/` folder) — modern, default, built on React Server Components. **Learn this.**
- **Pages Router** (`pages/` folder) — legacy, still supported. Only touch it for old client projects.

> This guide is App Router only. It's what you sell in 2026.

---

## 2. Creating a Project

```bash
npx create-next-app@latest my-app
```

The wizard asks: TypeScript (**yes**), ESLint, Tailwind (**yes**), `src/` directory (recommended), App Router (**yes**), import alias (`@/*`), Turbopack.

```bash
cd my-app
npm run dev     # start dev server (Turbopack) → http://localhost:3000
npm run build   # production build
npm run start   # run the production build
```

> **⚡ Version note:** In Next.js 16, **Turbopack is the default bundler** for both `dev` and `build` — much faster than the old Webpack pipeline. The React Compiler is also available to auto-memoize components.

---

## 3. Folder Structure

A clean, scalable Next.js 16 project:

```
my-app/
├── public/                      # static files served as-is (/logo.png → public/logo.png)
│   ├── images/
│   └── favicon.ico
├── src/
│   ├── app/                     # ←★ App Router lives here (routes = folders)
│   │   ├── layout.tsx           # ROOT layout (required) — wraps every page
│   │   ├── page.tsx             # home page → "/"
│   │   ├── globals.css          # global styles (import once in root layout)
│   │   ├── loading.tsx          # global loading UI
│   │   ├── error.tsx            # global error UI
│   │   ├── not-found.tsx        # 404 UI
│   │   │
│   │   ├── about/
│   │   │   └── page.tsx         # → "/about"
│   │   │
│   │   ├── blog/
│   │   │   ├── page.tsx         # → "/blog"
│   │   │   └── [slug]/          # dynamic segment
│   │   │       └── page.tsx     # → "/blog/my-post"
│   │   │
│   │   ├── dashboard/
│   │   │   ├── layout.tsx       # nested layout (sidebar, only for dashboard pages)
│   │   │   ├── page.tsx
│   │   │   └── settings/
│   │   │       └── page.tsx
│   │   │
│   │   ├── api/                 # Route Handlers (backend endpoints)
│   │   │   └── users/
│   │   │       └── route.ts     # → "/api/users" (GET/POST/...)
│   │   │
│   │   └── (marketing)/         # route GROUP — organizes without affecting URL
│   │       ├── pricing/page.tsx # → "/pricing" (no "marketing" in URL)
│   │       └── contact/page.tsx
│   │
│   ├── components/              # reusable React components
│   │   ├── ui/                  # shadcn/ui components live here
│   │   └── layout/              # navbar, footer, etc.
│   │
│   ├── lib/                     # utilities, helpers, db clients, API wrappers
│   │   ├── utils.ts
│   │   └── db.ts
│   │
│   ├── hooks/                   # custom React hooks (useXyz)
│   ├── types/                   # shared TypeScript types
│   ├── actions/                 # Server Actions ("use server")
│   └── config/                  # site config, constants, nav data
│
├── .env.local                   # secrets (never commit)
├── next.config.ts               # Next.js config
├── tsconfig.json
├── components.json              # shadcn config
└── package.json
```

**Key rule:** Inside `app/`, **folders define routes** and **special filenames define behavior**. A folder only becomes a public route when it contains a `page.tsx`.

---

## 4. The App Router — Special Files

These reserved filenames in any route folder do specific jobs:

| File | Purpose |
|---|---|
| `page.tsx` | The **page** UI for a route (makes the route public). |
| `layout.tsx` | Shared UI wrapping a page **and all its children**; preserves state on navigation. Root one is required. |
| `template.tsx` | Like layout but **re-mounts** on every navigation (use for enter animations). |
| `loading.tsx` | Instant loading UI (wraps the page in `<Suspense>` automatically). |
| `error.tsx` | Error boundary for the segment (must be a Client Component). |
| `global-error.tsx` | Catches errors in the root layout itself. |
| `not-found.tsx` | UI for `notFound()` calls and unmatched routes. |
| `route.ts` | **API endpoint** (Route Handler) — can't coexist with `page.tsx` in the same folder. |
| `default.tsx` | Fallback UI for parallel routes. |

**Folder naming conventions:**

| Pattern | Meaning |
|---|---|
| `folder/` | A route segment → part of the URL. |
| `[id]/` | **Dynamic** segment (`/post/123` → `params.id`). |
| `[...slug]/` | **Catch-all** (`/docs/a/b/c` → `params.slug = ['a','b','c']`). |
| `[[...slug]]/` | **Optional catch-all** (also matches the parent route). |
| `(group)/` | **Route group** — organizes files, **omitted from URL**. |
| `_folder/` | **Private folder** — opted out of routing (for colocation). |
| `@slot/` | **Parallel route** slot (render multiple pages in one layout). |
| `(.)folder` | **Intercepting route** (e.g. open a post in a modal over the feed). |

---

## 5. Routing Deep Dive

### Linking & navigation
```tsx
import Link from "next/link"
<Link href="/about" prefetch>About</Link>   // prefetches in viewport by default
```
```tsx
"use client"
import { useRouter, usePathname, useSearchParams, useParams } from "next/navigation"
const router = useRouter()
router.push("/dashboard")     // navigate
router.replace("/login")      // navigate without history entry
router.refresh()              // re-fetch server data for current route
router.back()
```

### Reading route data
```tsx
// In a Server Component page — params & searchParams are async (Promises)
export default async function Page({ params, searchParams }: {
  params: Promise<{ slug: string }>,
  searchParams: Promise<{ q?: string }>,
}) {
  const { slug } = await params
  const { q } = await searchParams
}
```
> **⚡ Version note:** Since Next.js 15, `params` and `searchParams` are **Promises** — you must `await` them. Same for `cookies()`, `headers()`, and `draftMode()` (all async now).

### Dynamic params generation (for SSG)
```tsx
// Pre-build these blog pages at build time
export async function generateStaticParams() {
  const posts = await getPosts()
  return posts.map((p) => ({ slug: p.slug }))
}
```

### Route Groups & Parallel Routes — when to use
- **Route group `(group)`**: share a layout among some pages, or organize, without changing URLs. e.g. `(auth)/login`, `(auth)/register` sharing an auth layout.
- **Parallel routes `@slot`**: render multiple independent sections in one layout (e.g. a dashboard with `@team` and `@analytics` panels that load and error independently).
- **Intercepting routes `(.)`**: show a route in a modal while keeping the URL shareable (e.g. Instagram-style photo modals).

---

## 6. Server vs Client Components

**This is the single most important concept in modern Next.js.** Get it and everything else clicks.

### The default: Server Components
Every component in `app/` is a **Server Component** unless you opt out. They:
- Run **only on the server** — never shipped to the browser (smaller JS bundle).
- Can be **async** and directly `await` data (DB, fetch, file system).
- Can access secrets/env safely.
- **Cannot** use hooks (`useState`, `useEffect`), browser APIs, or event handlers (`onClick`).

```tsx
// Server Component (default) — fetch data right inside it
export default async function ProductList() {
  const products = await db.product.findMany()   // runs on server
  return <ul>{products.map(p => <li key={p.id}>{p.name}</li>)}</ul>
}
```

### Opting into Client Components
Add `"use client"` at the top of the file. Use only when you need:
- State / effects (`useState`, `useEffect`, `useReducer`)
- Event handlers (`onClick`, `onChange`)
- Browser APIs (`window`, `localStorage`)
- Hooks from libraries (TanStack Query, framer-motion, etc.)

```tsx
"use client"
import { useState } from "react"
export default function Counter() {
  const [n, setN] = useState(0)
  return <button onClick={() => setN(n + 1)}>{n}</button>
}
```

### The golden patterns
1. **Keep components on the server by default.** Only mark the leaves that truly need interactivity as `"use client"`.
2. **Push `"use client"` to the leaves.** Don't make a whole page client just for one button — extract the button.
3. **Server Components can import Client Components**, but not vice-versa. To put server content inside a client component, **pass it as `children`/props**:
```tsx
// ClientWrapper is "use client", but ServerThing stays a Server Component
<ClientWrapper><ServerThing /></ClientWrapper>
```
4. **Never pass non-serializable data** (functions, class instances) from Server → Client components as props. Data must be serializable.

**Mental model:** Server Components = data + structure. Client Components = interactivity. Compose them like nesting dolls.

---

## 7. Rendering Strategies

Next.js picks per-route. You control it.

| Strategy | What | When | How to trigger |
|---|---|---|---|
| **Static (SSG)** | Rendered at **build time**, served as static HTML. Fastest. | Marketing pages, blogs, docs. | Default when no dynamic data/APIs used. |
| **Dynamic (SSR)** | Rendered **per request** on the server. | Personalized/auth pages, live data. | Using `cookies()`, `headers()`, uncached fetch, or `dynamic = 'force-dynamic'`. |
| **ISR** | Static, but **regenerated** in the background every N seconds. | Content that changes occasionally (product listings, news). | `export const revalidate = 60` or `fetch(url, { next: { revalidate: 60 } })`. |
| **Streaming** | Send HTML in chunks as data resolves. | Slow data that shouldn't block the whole page. | `loading.tsx` + `<Suspense>`. |
| **PPR / Cache Components** | Static shell + streamed dynamic holes in **one** response. | Best of both — fast shell, fresh data. | `use cache` directive (see §9). |

### Route segment config (export from a `page.tsx`/`layout.tsx`)
```tsx
export const dynamic = "force-dynamic"  // 'auto' | 'force-dynamic' | 'force-static' | 'error'
export const revalidate = 3600          // ISR interval in seconds (or false)
export const fetchCache = "force-cache"
export const runtime = "nodejs"         // 'nodejs' | 'edge'
export const preferredRegion = "auto"
```

---

## 8. Data Fetching

In Server Components you just `await`. No `useEffect`, no loading boilerplate.

```tsx
// Direct fetch in a Server Component
export default async function Page() {
  const res = await fetch("https://api.example.com/posts")
  const posts = await res.json()
  return <PostList posts={posts} />
}
```

```tsx
// Direct DB access (server only — safe)
import { db } from "@/lib/db"
export default async function Page() {
  const users = await db.user.findMany()
  return <UserTable users={users} />
}
```

### Parallel vs sequential fetching
```tsx
// ❌ Sequential (waterfall — slow): each await blocks the next
const user = await getUser()
const posts = await getPosts()

// ✅ Parallel: fire together, await together
const [user, posts] = await Promise.all([getUser(), getPosts()])
```

### Client-side fetching
For data that changes after load, is user-specific, or needs caching/refetching on the client → use **TanStack Query** (see the companion guide). Don't fetch in `useEffect` manually for anything non-trivial.

### Route Handlers (your backend/API)
```ts
// app/api/users/route.ts
import { NextRequest, NextResponse } from "next/server"

export async function GET(req: NextRequest) {
  const users = await db.user.findMany()
  return NextResponse.json(users)
}

export async function POST(req: NextRequest) {
  const body = await req.json()
  const user = await db.user.create({ data: body })
  return NextResponse.json(user, { status: 201 })
}
```
Supported exports: `GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS`.
> **⚡ Version note:** Since Next.js 15, **GET Route Handlers are NOT cached by default** (they were in 14). Opt in with `export const dynamic = 'force-static'` or `revalidate`.

---

## 9. Caching Model

Next.js has several cache layers. This trips everyone up — here's the clear version.

### The four caches
| Cache | Where | Caches | Default in 16 |
|---|---|---|---|
| **Request Memoization** | Server, per-request | Duplicate `fetch()` in one render pass | On (dedupes automatically). |
| **Data Cache** | Server, persistent | `fetch()` results across requests | **Off by default** — opt in. |
| **Full Route Cache** | Server, build/revalidate | Rendered static routes | On for static routes. |
| **Router Cache** | Client, in-memory | Visited route segments (for fast back/forward) | On (short-lived). |

### Controlling the Data Cache with fetch
```tsx
fetch(url)                                   // NOT cached by default (dynamic)
fetch(url, { cache: "force-cache" })         // cache indefinitely (static)
fetch(url, { next: { revalidate: 60 } })     // ISR: cache, refresh every 60s
fetch(url, { cache: "no-store" })            // never cache (always fresh)
fetch(url, { next: { tags: ["posts"] } })    // tag for on-demand revalidation
```
> **⚡ Version note:** This "uncached by default" behavior arrived in Next.js 15 and continues in 16 — a big change from 14 where fetch was cached by default. **Always be explicit about caching.**

### On-demand revalidation (e.g. after a content update)
```tsx
import { revalidatePath, revalidateTag } from "next/cache"
revalidatePath("/blog")        // re-render this path
revalidateTag("posts")         // re-fetch everything tagged "posts"
```

### `use cache` — Cache Components (the modern way)
```tsx
// Cache an entire function/component's output
async function getProducts() {
  "use cache"
  return await db.product.findMany()
}
```
> **⚡ Version note:** **Cache Components** (the `use cache` directive, the evolution of Partial Prerendering) is the headline caching feature of the Next.js 16 era. It lets you cache at the component/function level and combine a static shell with streamed dynamic content in a single render. Enable it in `next.config.ts` (e.g. `cacheComponents: true`) — confirm the exact flag in the docs for your version.

### Practical caching rules of thumb
- **Static marketing content** → cache it (`force-cache` / static).
- **Per-user/auth content** → don't cache (dynamic).
- **Content that changes hourly/daily** → ISR (`revalidate`).
- **After a write (form submit)** → `revalidatePath`/`revalidateTag` to refresh.

---

## 10. Server Actions & Mutations

Server Actions are async functions that **run on the server** but can be called from client components / forms — no manual API route needed. Great for forms and mutations.

```tsx
// app/actions/createPost.ts
"use server"
import { revalidatePath } from "next/cache"
import { redirect } from "next/navigation"

export async function createPost(formData: FormData) {
  const title = formData.get("title") as string
  await db.post.create({ data: { title } })
  revalidatePath("/blog")     // refresh the list
  redirect("/blog")           // navigate away
}
```

```tsx
// Use directly in a form (works without JS — progressive enhancement)
import { createPost } from "@/app/actions/createPost"
export default function NewPost() {
  return (
    <form action={createPost}>
      <input name="title" />
      <button type="submit">Create</button>
    </form>
  )
}
```

### With pending state & validation (client side)
```tsx
"use client"
import { useActionState } from "react"
import { useFormStatus } from "react-dom"

function SubmitButton() {
  const { pending } = useFormStatus()
  return <button disabled={pending}>{pending ? "Saving..." : "Save"}</button>
}
```

**When to use Server Actions vs Route Handlers:**
- **Server Actions** → form submissions & mutations from your own UI (simpler, typed, no fetch wiring).
- **Route Handlers** → public APIs, webhooks, third-party integrations, mobile clients.

> **Security:** Always validate & authorize inside the action (use `zod`). Server Actions are public endpoints under the hood — never trust the input.

---

## 11. Loading, Streaming & Suspense

### Instant loading UI
Drop a `loading.tsx` in any route folder; Next wraps the page in Suspense automatically:
```tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return <DashboardSkeleton />
}
```

### Granular streaming with Suspense
Stream slow parts independently so the fast parts render immediately:
```tsx
import { Suspense } from "react"
export default function Page() {
  return (
    <>
      <Header />                                  {/* instant */}
      <Suspense fallback={<Skeleton />}>
        <SlowProductList />                        {/* streams in when ready */}
      </Suspense>
    </>
  )
}
```
This is a huge perceived-performance win — the page feels instant even when one widget is slow.

---

## 12. Error Handling

```tsx
// app/dashboard/error.tsx — must be a Client Component
"use client"
export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div>
      <p>Something went wrong: {error.message}</p>
      <button onClick={() => reset()}>Try again</button>
    </div>
  )
}
```
```tsx
// Trigger a 404 from a Server Component
import { notFound } from "next/navigation"
if (!post) notFound()   // renders the nearest not-found.tsx
```
- `error.tsx` catches errors in its segment's components.
- `global-error.tsx` catches errors in the **root layout** (must include its own `<html>`/`<body>`).
- Errors in Server Components are sanitized in production (don't leak stack traces).

---

## 13. Metadata & SEO

SEO is a top reason clients choose Next.js. The Metadata API makes it clean.

### Static metadata
```tsx
// app/layout.tsx or any page.tsx
import type { Metadata } from "next"
export const metadata: Metadata = {
  title: "Acme — Modern Websites",
  description: "We build fast, modern websites.",
  keywords: ["web design", "next.js"],
  openGraph: {
    title: "Acme",
    description: "Fast modern websites.",
    url: "https://acme.com",
    images: ["/og.png"],
    type: "website",
  },
  twitter: { card: "summary_large_image", images: ["/og.png"] },
  alternates: { canonical: "https://acme.com" },
}
```

### Dynamic metadata (per page, e.g. blog posts)
```tsx
export async function generateMetadata({ params }: { params: Promise<{ slug: string }> }): Promise<Metadata> {
  const { slug } = await params
  const post = await getPost(slug)
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: { images: [post.coverImage] },
  }
}
```

### Title templates
```tsx
export const metadata = {
  title: { default: "Acme", template: "%s | Acme" }, // child pages → "About | Acme"
}
```

### SEO essentials checklist
- ✅ **Unique title + description per page** (use `generateMetadata` for dynamic).
- ✅ **Open Graph + Twitter cards** for social sharing previews.
- ✅ **`app/sitemap.ts`** → auto-generates `sitemap.xml`.
- ✅ **`app/robots.ts`** → auto-generates `robots.txt`.
- ✅ **Structured data (JSON-LD)** — inject a `<script type="application/ld+json">` for rich results.
- ✅ **Canonical URLs** via `alternates.canonical`.
- ✅ **Semantic HTML** (`<h1>` once, `<header>`, `<nav>`, `<main>`, alt text on images).
- ✅ **Static/ISR rendering** so crawlers get full HTML instantly.
- ✅ **`app/opengraph-image.tsx`** → generate dynamic OG images with `ImageResponse`.

```ts
// app/sitemap.ts
import type { MetadataRoute } from "next"
export default function sitemap(): MetadataRoute.Sitemap {
  return [
    { url: "https://acme.com", lastModified: new Date(), priority: 1 },
    { url: "https://acme.com/blog", changeFrequency: "weekly", priority: 0.8 },
  ]
}
```

```ts
// app/robots.ts
import type { MetadataRoute } from "next"
export default function robots(): MetadataRoute.Robots {
  return { rules: { userAgent: "*", allow: "/" }, sitemap: "https://acme.com/sitemap.xml" }
}
```

---

## 14. Performance & Optimization

What separates a "fine" site from a fast one clients rave about:

| Technique | How |
|---|---|
| **Server Components first** | Less JS shipped = faster loads. Default behavior — keep it. |
| **Static / ISR rendering** | Pre-render pages; serve HTML instantly from the edge/CDN. |
| **Streaming + Suspense** | Don't let one slow query block the whole page. |
| **`next/image`** | Auto resize, lazy-load, modern formats (AVIF/WebP), no layout shift. |
| **`next/font`** | Self-host fonts, zero layout shift, no external request. |
| **Code splitting** | Automatic per-route; use `next/dynamic` for heavy client widgets. |
| **`dynamic(() => import(...), { ssr: false })`** | Lazy-load client-only/heavy components (charts, maps, editors). |
| **Prefetching** | `<Link>` prefetches in-viewport routes automatically. |
| **Parallel data fetching** | `Promise.all` to kill waterfalls. |
| **Bundle analysis** | `@next/bundle-analyzer` to find bloat. |
| **React Compiler** | Auto-memoizes — fewer needless re-renders (Next 16). |
| **Edge runtime** | For ultra-low-latency, geo-distributed responses. |

### Core Web Vitals to hit (clients & Google care)
- **LCP** (Largest Contentful Paint) < 2.5s → optimize hero image with `next/image priority`.
- **CLS** (Cumulative Layout Shift) < 0.1 → always set image/font dimensions; use `next/font`.
- **INP** (Interaction to Next Paint) < 200ms → ship less client JS; keep components on the server.

```tsx
import dynamic from "next/dynamic"
const Chart = dynamic(() => import("@/components/Chart"), {
  ssr: false,
  loading: () => <Skeleton className="h-64" />,
})
```

---

## 15. Images, Fonts & Assets

### Images — always use `next/image`
```tsx
import Image from "next/image"
<Image
  src="/hero.jpg"
  alt="Hero"
  width={1200}
  height={600}
  priority           // for above-the-fold/LCP images — preloads
  className="rounded-xl object-cover"
/>
// For unknown-size/fill layouts:
<div className="relative h-64 w-full">
  <Image src="/x.jpg" alt="" fill sizes="100vw" className="object-cover" />
</div>
```
For external images, whitelist domains in `next.config.ts`:
```ts
const nextConfig = {
  images: { remotePatterns: [{ protocol: "https", hostname: "images.unsplash.com" }] },
}
```

### Fonts — `next/font` (self-hosted, no layout shift)
```tsx
import { Inter } from "next/font/google"
const inter = Inter({ subsets: ["latin"], variable: "--font-inter" })
export default function RootLayout({ children }) {
  return <html lang="en" className={inter.variable}><body>{children}</body></html>
}
```

### Static assets
Files in `public/` are served from the root: `public/logo.png` → `/logo.png`.

---

## 16. Styling

| Option | Notes |
|---|---|
| **Tailwind CSS** | Default & recommended. Pairs with shadcn/ui. (See Tailwind cheatsheet.) |
| **CSS Modules** | `styles.module.css` → scoped class names. |
| **Global CSS** | `globals.css`, imported once in root layout. |
| **CSS-in-JS** | Possible but adds runtime cost; Tailwind is preferred for Server Components. |

> Tailwind + shadcn/ui is the winning combo for client work — fast, consistent, and easy to theme.

---

## 17. Middleware & Proxy

Middleware runs **before** a request completes — great for auth gating, redirects, geolocation, A/B tests.

```ts
// middleware.ts (project root, next to app/)
import { NextRequest, NextResponse } from "next/server"
export function middleware(req: NextRequest) {
  const token = req.cookies.get("token")
  if (!token && req.nextUrl.pathname.startsWith("/dashboard")) {
    return NextResponse.redirect(new URL("/login", req.url))
  }
  return NextResponse.next()
}
export const config = { matcher: ["/dashboard/:path*"] }   // only run on these paths
```
> **⚡ Version note:** Next.js 16 supports the **Node.js runtime for middleware** (not only Edge), so you can use more Node APIs there. There's also a `proxy` capability evolving — confirm specifics in the docs for your version.

---

## 18. Environment Variables & Config

```bash
# .env.local (never commit)
DATABASE_URL=postgres://...
NEXT_PUBLIC_API_URL=https://api.acme.com   # NEXT_PUBLIC_ → exposed to the browser
```
- Variables **without** `NEXT_PUBLIC_` are server-only (safe for secrets).
- Variables **with** `NEXT_PUBLIC_` are inlined into client JS — never put secrets there.

```ts
// next.config.ts
import type { NextConfig } from "next"
const nextConfig: NextConfig = {
  images: { remotePatterns: [/* ... */] },
  // experimental: { cacheComponents: true },  // enable as needed
}
export default nextConfig
```

---

## 19. Design Patterns & Project Architecture

Patterns that keep client projects clean and maintainable:

1. **Server Components for data, Client Components for interactivity.** Fetch on the server, hydrate islands of interactivity.
2. **Colocation:** keep route-specific components inside the route folder (use `_components/` private folders). Shared ones go in `src/components`.
3. **`lib/` for logic:** DB clients, API wrappers, and pure helpers live in `lib/` — not in components.
4. **`actions/` for mutations:** group Server Actions; validate every input with `zod`.
5. **Container/Presentational split:** a Server Component fetches and passes data to a "dumb" presentational component.
6. **Composition over prop-drilling:** pass Server Components as `children` into Client wrappers.
7. **Route groups for layout boundaries:** `(marketing)` with a public navbar, `(app)` with an authenticated shell.
8. **Type everything:** share types in `src/types`; infer from `zod` schemas where possible.
9. **Feature folders for big apps:** group by feature (`features/auth`, `features/billing`) rather than by file type once the app grows.
10. **One source of truth for config:** `config/site.ts` for nav, metadata defaults, social links.

```
src/
├── app/
│   └── (app)/dashboard/
│       ├── page.tsx              # Server Component: fetches data
│       ├── _components/          # private, route-only components
│       │   └── revenue-card.tsx
│       └── loading.tsx
├── components/ui/                # shadcn
├── lib/db.ts                     # prisma/drizzle client
├── actions/billing.ts            # "use server" mutations
└── types/index.ts
```

---

## 20. Deployment

| Host | Notes |
|---|---|
| **Vercel** | Made by Next.js team. Zero-config, free tier, automatic preview deploys, ISR/edge support. **Default choice for client work.** |
| **Netlify** | Good Next.js support. |
| **Self-host (Node)** | `next build && next start` behind a reverse proxy; or `output: "standalone"` for Docker. |
| **Static export** | `output: "export"` for fully static sites (no SSR/ISR features). |

**Workflow:** push to GitHub → connect repo to Vercel → set env vars → every push gets a preview URL, `main` deploys to production. Clients love sharing preview links.

---

## 21. Tips, Tricks & Gotchas

**Do:**
- ✅ Default to Server Components; add `"use client"` only at the leaves.
- ✅ Be **explicit about caching** — assume nothing is cached, opt in deliberately.
- ✅ Use `loading.tsx` + `<Suspense>` so pages feel instant.
- ✅ Use `next/image` and `next/font` everywhere — free Core Web Vitals wins.
- ✅ `generateStaticParams` + ISR for blogs/catalogs → fast + fresh.
- ✅ Validate Server Action / Route Handler input with `zod`.
- ✅ Use `Promise.all` to fetch in parallel.
- ✅ Set unique metadata per page.

**Don't / common bugs:**
- ❌ Forgetting to `await params`/`searchParams`/`cookies()` (they're Promises now).
- ❌ Putting `"use client"` at the top of a whole page just for one interactive bit.
- ❌ Passing functions or class instances as props from Server → Client components.
- ❌ Using `useState`/`useEffect`/`window` in a Server Component (you'll get an error).
- ❌ Expecting fetch to be cached automatically (it isn't in 15/16).
- ❌ Leaking secrets via `NEXT_PUBLIC_` vars.
- ❌ Importing a Server Component *into* a Client Component (pass as `children` instead).
- ❌ Hydration mismatch — don't render `Date.now()`/`Math.random()` differently on server vs client; guard browser-only code.
- ❌ Heavy client libs (charts/maps) loaded eagerly — lazy-load with `next/dynamic`.

**Tricks:**
- 🔹 `router.refresh()` re-fetches server data without losing client state — great after mutations.
- 🔹 Intercepting routes for modals that are still shareable URLs.
- 🔹 `app/opengraph-image.tsx` generates dynamic social images per page.
- 🔹 Use route groups to give marketing vs app sections different layouts.
- 🔹 `export const dynamic = 'force-static'` to force-cache a route that would otherwise be dynamic.

---

## 22. Client-Pleasing Checklist

Before you hand off any Next.js site:

- [ ] **Lighthouse 90+** on Performance, SEO, Accessibility, Best Practices.
- [ ] **Fully responsive** (test 360px, 768px, 1440px).
- [ ] **Unique title + meta description** on every page.
- [ ] **OG/Twitter cards** render correctly (test with a preview tool).
- [ ] **sitemap.xml + robots.txt** present.
- [ ] **All images via `next/image`** with `alt` text and `priority` on the hero.
- [ ] **Fonts via `next/font`** (no layout shift).
- [ ] **404 + error pages** styled, not default.
- [ ] **Loading skeletons** on data-heavy pages.
- [ ] **Forms validated** with friendly error messages.
- [ ] **No console errors / hydration warnings.**
- [ ] **Deployed to Vercel** with a shareable URL + env vars set.
- [ ] **Favicon + meta theme color** set.

---

### Study path (offline mastery order)
1. Server vs Client Components (§6) — internalize this first.
2. Folder structure + special files (§3, §4).
3. Routing (§5).
4. Data fetching + caching (§8, §9).
5. Rendering strategies (§7).
6. Server Actions (§10), loading/streaming (§11), errors (§12).
7. SEO (§13) + performance (§14) — your selling points.
8. Build 2–3 real projects applying §19 patterns.

> Build to learn: a blog (SSG + ISR + dynamic routes + metadata), a dashboard (Server Components + Server Actions + streaming), and a landing page (performance + SEO). That trio covers ~90% of client work.
