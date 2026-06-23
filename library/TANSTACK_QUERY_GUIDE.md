# TanStack Query v5 — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I fetch data with `useEffect` and `useState`" to "I manage server state correctly, with caching, optimistic UI, infinite scroll, and server-side hydration." Every concept is explained in prose first — *what it is, the underlying logic, what it's for, when you reach for it, how to use it, the key options, best practices, and the gotchas* — and only then followed by heavily commented, runnable code. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **TanStack Query v5** (the React adapter, `@tanstack/react-query`), running on **React 19** (current in 2026). TanStack Query v5 was a deliberate breaking release: a single object-only API, `isPending` instead of `isLoading`, `gcTime` instead of `cacheTime`, and first-class Suspense support. Where behaviour is version-sensitive it is flagged with **⚡ Version note**, and v4→v5 migration differences are collected in §20. The code uses **TypeScript** throughout because TanStack Query's types are excellent and catch real mistakes. The author is on **Windows 11**; nothing here is OS-specific. For exact APIs confirm against tanstack.com/query.
>
> **Companion guides in this library:** the [React 19 guide](./REACT_19_GUIDE.md) (hooks, Suspense, Error Boundaries, `use`), the [Next.js 16 guide](./NEXTJS_16_GUIDE.md) (Server Components, the App Router, where server-side prefetch fits), and the [Zustand guide](./ZUSTAND_GUIDE.md) (for *client* state — the thing TanStack Query is deliberately **not**).

---

## Table of Contents

1. [What Server State Is & Why This Library Exists](#1-what-server-state-is--why-this-library-exists) **[B]**
2. [Installation & Setup (QueryClient + Provider)](#2-installation--setup-queryclient--provider) **[B]**
3. [Core Concepts & Vocabulary](#3-core-concepts--vocabulary) **[B]**
4. [useQuery — Reading Data in Depth](#4-usequery--reading-data-in-depth) **[B/I]**
5. [staleTime vs gcTime — The Two Clocks](#5-staletime-vs-gctime--the-two-clocks) **[I]**
6. [Refetch Behaviours & Polling](#6-refetch-behaviours--polling) **[I]**
7. [enabled, select, placeholderData & initialData](#7-enabled-select-placeholderdata--initialdata) **[I]**
8. [Retry & Error Handling in queryFn](#8-retry--error-handling-in-queryfn) **[I]**
9. [Query Keys & Key Factories](#9-query-keys--key-factories) **[B/I]**
10. [The Cache Lifecycle (Fresh → Stale → Inactive → GC)](#10-the-cache-lifecycle-fresh--stale--inactive--gc) **[I]**
11. [useMutation — Writing Data & the Mutation Lifecycle](#11-usemutation--writing-data--the-mutation-lifecycle) **[B/I]**
12. [Query Invalidation & Refetching](#12-query-invalidation--refetching) **[I]**
13. [Optimistic Updates (Both Patterns)](#13-optimistic-updates-both-patterns) **[A]**
14. [Dependent & Parallel Queries](#14-dependent--parallel-queries) **[I]**
15. [useInfiniteQuery & Pagination](#15-useinfinitequery--pagination) **[A]**
16. [Prefetching](#16-prefetching) **[I/A]**
17. [Suspense: useSuspenseQuery & Error Boundaries](#17-suspense-usesuspensequery--error-boundaries) **[A]**
18. [Next.js Integration (Server Prefetch + Hydration)](#18-nextjs-integration-server-prefetch--hydration) **[A]**
19. [Devtools, Testing & Performance](#19-devtools-testing--performance) **[I/A]**
20. [v5 Migration, Gotchas & Quick Reference](#20-v5-migration-gotchas--quick-reference) **[I]**
21. [Study Path & Build-to-Learn Projects](#21-study-path--build-to-learn-projects)

---

## 1. What Server State Is & Why This Library Exists

Before you write a single `useQuery`, you need the mental model, because TanStack Query solves a *specific* problem and using it for the wrong thing leads to confusion. The single most important idea in this whole guide is the distinction between **client state** and **server state**.

**Client state** is data your application *owns* and fully controls: whether a modal is open, the current value of a form input, which tab is selected, the chosen theme. It is synchronous, it never goes stale on its own, and only your app can change it. The right tools for client state are `useState`, `useReducer`, the React Context API, or a dedicated client-state library like [Zustand](./ZUSTAND_GUIDE.md) or Redux.

**Server state** is fundamentally different. It is data that lives somewhere else — a database behind an API — and your app only ever holds a *cached copy* of it. This single fact has enormous consequences:

- **You don't own it.** Another user, a background job, or another browser tab can change it at any moment. Your copy can therefore be *out of date* the instant after you fetch it.
- **It's asynchronous.** Fetching takes time and can fail, so every piece of server state has implicit loading and error states attached.
- **It needs to be shared.** Two components asking for "the current user" should not fire two requests and hold two copies; they should share one cached result.
- **It needs to be kept fresh.** Because it goes stale, you want to refetch it at sensible moments (when the window regains focus, when the network reconnects, after a mutation) — but not so aggressively that you hammer the server.

Redux, Zustand, and Context were all designed for *client* state. When people use them for server state, they end up hand-writing all of the above — caching, deduplication, background refresh, retry, invalidation — and they get it subtly wrong. **TanStack Query is a library purpose-built for the server-state problem.** It is not a general state manager; it is an *async server-state cache* with deep React integration.

### The boilerplate it replaces

Here is the canonical way people fetch data in React without a library. Study what is *missing* from it, because every missing thing is a feature TanStack Query gives you for free.

```tsx
// The hand-rolled approach — works, but it is a trap.
function Todos() {
  const [data, setData] = useState<Todo[]>()
  const [isLoading, setIsLoading] = useState(true)
  const [error, setError] = useState<Error>()

  useEffect(() => {
    let cancelled = false                       // manual cleanup to avoid setting state after unmount
    setIsLoading(true)
    fetch("/api/todos")
      .then((r) => r.json())
      .then((d) => { if (!cancelled) setData(d) })
      .catch((e) => { if (!cancelled) setError(e) })
      .finally(() => { if (!cancelled) setIsLoading(false) })
    return () => { cancelled = true }
  }, [])                                          // [] = run once on mount

  if (isLoading) return <Spinner />
  if (error) return <p>{error.message}</p>
  return <ul>{data!.map((t) => <li key={t.id}>{t.title}</li>)}</ul>
}
```

What this code does *not* do, and never will without a pile more effort:

- **No caching.** Navigate away and back, and it refetches from scratch every time, showing a spinner each time.
- **No deduplication.** Render this component twice on a page and you fire two identical requests.
- **No background refresh.** Once loaded, the data is frozen until a full remount — it silently rots.
- **No retry on transient failure.** One flaky network blip and the user sees an error.
- **No shared state.** Another component that needs todos has to repeat all of this and holds its own separate copy.
- **No request cancellation, no window-focus refetch, no "refetch after I add a todo," no pagination helpers.**

The equivalent in TanStack Query is one hook, and it does *all* of the above:

```tsx
function Todos() {
  // queryKey identifies this data in the cache; queryFn fetches it.
  const { data, isPending, error } = useQuery({
    queryKey: ["todos"],
    queryFn: () => fetch("/api/todos").then((r) => r.json()),
  })

  if (isPending) return <Spinner />              // no data cached yet (true first load)
  if (error) return <p>{error.message}</p>
  return <ul>{data.map((t) => <li key={t.id}>{t.title}</li>)}</ul>
}
```

That is the entire pitch. The rest of this guide is about understanding *exactly* what that hook is doing under the hood, so you can tune it, trust it, and reach for the more advanced features when you need them.

> **The one-sentence summary:** TanStack Query turns "fetch data and store it in component state" into "subscribe to a shared, automatically-managed cache of server data." Once that clicks, everything else follows.

---

## 2. Installation & Setup (QueryClient + Provider)

TanStack Query has two pieces you set up once per application: a **`QueryClient`** (the brain — it holds the cache and all the default configuration) and a **`QueryClientProvider`** (a React context provider that hands that client down to every component so hooks can find it). Every `useQuery`/`useMutation` in your tree talks to the nearest `QueryClient` via context.

```bash
# The core library (required)
npm install @tanstack/react-query

# The devtools — a dev-only floating panel that lets you watch the cache (highly recommended)
npm install -D @tanstack/react-query-devtools
```

### Creating the QueryClient correctly

There is **one** important rule when creating the `QueryClient`: create it **once** and keep the same instance for the life of the app. The `QueryClient` *is* your cache — if you recreate it on every render, you throw the cache away every render and the library does nothing useful. In a plain client React app you can create it as a module-level singleton. In Next.js (or any SSR setup) you must create it inside the component with `useState(() => new QueryClient())` so that each *request* gets a fresh client on the server but the *browser* keeps a single stable one — sharing one client across server requests would leak one user's data into another's. §18 covers the Next.js nuance in full; here is the standard client-app provider.

```tsx
// providers.tsx — in Next.js App Router this file needs the "use client" directive,
// because QueryClientProvider uses React Context, which only works in Client Components.
"use client"
import { QueryClient, QueryClientProvider } from "@tanstack/react-query"
import { ReactQueryDevtools } from "@tanstack/react-query-devtools"
import { useState } from "react"

export function Providers({ children }: { children: React.ReactNode }) {
  // useState with an initializer FUNCTION runs `new QueryClient(...)` exactly once,
  // on first render, and returns the same instance forever after. Writing
  // `useState(new QueryClient())` would construct a new client every render and
  // throw it away — a subtle but cache-destroying mistake. Always pass a function.
  const [queryClient] = useState(
    () =>
      new QueryClient({
        // defaultOptions sets the baseline for EVERY query/mutation. Any individual
        // hook can override these. Setting them once here is the cleanest way to
        // establish app-wide behaviour.
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000,        // treat data as "fresh" for 60s (see §5) — cuts refetches
            gcTime: 5 * 60 * 1000,       // keep unused data cached for 5 min before deleting (default)
            refetchOnWindowFocus: true,  // refetch stale data when the tab regains focus (default true)
            retry: 1,                    // retry a failed request once before surfacing the error
          },
          mutations: {
            retry: 0,                    // usually you do NOT want to auto-retry writes
          },
        },
      }),
  )

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      {/* Devtools render nothing in production builds and are tree-shaken away. */}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  )
}
```

```tsx
// app/layout.tsx (Next.js App Router) — mount the provider near the root so every
// component below it can use the query hooks.
import { Providers } from "./providers"

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  )
}
```

> **Gotcha — "No QueryClient set, use QueryClientProvider":** this runtime error means a hook ran in a component that is *not* inside the provider. Either you forgot to wrap your app, or (in Next.js) the provider file is missing `"use client"`. The provider must be an ancestor of every component that calls `useQuery`/`useMutation`.

> **Best practice — one provider, near the root.** Don't sprinkle multiple `QueryClientProvider`s through your tree (you'd get isolated caches that can't share data). One client, one provider, as high as practical.

---

## 3. Core Concepts & Vocabulary

These terms recur constantly. Skim them now; they'll make full sense as you go, and you can return here as a glossary. The two pairs people most often confuse — `isPending` vs `isFetching`, and `staleTime` vs `gcTime` — each get their own deep section later (§4 and §5), but get the gist here first.

| Term | Meaning |
|---|---|
| **Query** | A *read* of server data, identified by a unique **query key**. Declarative — it runs automatically. |
| **Mutation** | A *write* (create / update / delete). Imperative — it runs only when you call `mutate()`. |
| **Query key** | A serializable array (`["todos", id]`) that uniquely addresses a cache entry. The cache is a `Map` keyed by these. |
| **Query function (`queryFn`)** | An async function that returns the data, or **throws** on failure. The throw is how the library learns a request failed. |
| **Fresh** | Data younger than `staleTime`. Served instantly from cache with **no** background refetch. |
| **Stale** | Data older than `staleTime`. Still served from cache instantly, but a background refetch is triggered on the next opportunity (mount, focus, reconnect). |
| **`staleTime`** | How long data is considered fresh. Controls *refetching*. Default `0` (immediately stale). |
| **`gcTime`** | "Garbage-collection time": how long *unused* (inactive) data stays in the cache before deletion. Controls *memory*. Default 5 min. (Was `cacheTime` in v4.) |
| **Active / inactive query** | A query is *active* while at least one mounted component subscribes to it; *inactive* once no component uses it. The `gcTime` clock only runs while inactive. |
| **Invalidation** | Marking matching queries stale (and refetching the active ones). The primary "things changed, refresh" mechanism, used after mutations. |
| **`isPending`** | No cached data exists yet — the true first load. (Renamed from `isLoading` in v5.) |
| **`isFetching`** | A request is in flight *right now*, whether it's the first load or a silent background refetch. |
| **`isLoading`** | A v5 convenience: `isPending && isFetching` — first load that is actively fetching. |
| **Optimistic update** | Updating the UI *before* the server confirms, then rolling back if it fails — makes the app feel instant. |
| **Dehydrate / hydrate** | Serialize the server's cache to JSON (`dehydrate`) and rebuild it in the browser (`HydrationBoundary`) — the foundation of SSR. |

### The stale-while-revalidate model — the heartbeat of the library

Almost everything TanStack Query does flows from one strategy borrowed from HTTP caching: **stale-while-revalidate**. When a component mounts a query:

1. If there is cached data for that key, it is shown **immediately** — no spinner, no flash.
2. If that cached data is **stale**, the library *also* fires a silent background refetch.
3. When the fresh data arrives, the UI updates seamlessly.

This is why apps built with TanStack Query feel fast: you almost never stare at a spinner for data you've already seen. You see the old data instantly and it quietly updates. Internalize this loop — "show cached now, revalidate in the background" — and the behaviour of `staleTime`, focus refetching, and invalidation all become obvious.

---

## 4. useQuery — Reading Data in Depth

`useQuery` is the hook you'll use most. Its job: *given a key and a fetcher, subscribe this component to that piece of server state and return its current status.* It is **declarative** — you don't "call" it to fetch; you *describe* the data you want, and the library decides when to actually run the `queryFn` (on mount if needed, on focus if stale, on invalidation, etc.). This declarative nature is a mindset shift from imperative `fetch()` calls, and it's the source of most of the convenience.

The two required options are `queryKey` and `queryFn`. The `queryKey` is the cache address (§9). The `queryFn` is an async function that **returns the data on success and throws on failure**. That last part is non-negotiable: `fetch()` does *not* throw on HTTP 404/500 (it only rejects on network failure), so you must check `res.ok` and throw yourself, or the library will cache an error page as if it were valid data.

```tsx
"use client"
import { useQuery } from "@tanstack/react-query"

interface Todo { id: number; title: string; done: boolean }

// The query function. It MUST return data or throw. Note the manual `res.ok` check —
// without it, a 500 response would resolve successfully and you'd cache the error JSON.
async function fetchTodos(): Promise<Todo[]> {
  const res = await fetch("/api/todos")
  if (!res.ok) throw new Error(`Failed to load todos: ${res.status}`)
  return res.json()
}

function Todos() {
  const {
    data,         // the resolved data; `undefined` until the first successful fetch
    isPending,    // true when there is NO cached data yet (the real first load)
    isError,      // true if the queryFn threw and retries are exhausted
    error,        // the thrown error object (typed `Error | null`)
    isFetching,   // true whenever a request is in flight — including silent background refetches
    isSuccess,    // true once data has loaded successfully at least once
    refetch,      // imperatively force a refetch (returns a promise)
    status,       // 'pending' | 'error' | 'success' — the data status
    fetchStatus,  // 'fetching' | 'paused' | 'idle' — the network status
  } = useQuery({
    queryKey: ["todos"],
    queryFn: fetchTodos,
  })

  // Handle the first-load case: no data yet.
  if (isPending) return <Skeleton />

  // Handle errors: queryFn threw and retries ran out.
  if (isError) return <p role="alert">Error: {error.message}</p>

  // From here `data` is guaranteed defined (TypeScript narrows it for you).
  return (
    <>
      {/* isFetching is true during background refreshes too — show a subtle indicator,
          NOT a full skeleton, so the user keeps seeing the old data. */}
      {isFetching && <span className="text-xs">Refreshing…</span>}
      <ul>{data.map((t) => <li key={t.id}>{t.title}</li>)}</ul>
      <button onClick={() => refetch()}>Refresh now</button>
    </>
  )
}
```

### `status` vs `fetchStatus` — two orthogonal axes

This is the concept beginners trip over most, so here it is explicitly. TanStack Query tracks **two independent things**:

- **`status`** answers *"do I have data?"* — `'pending'` (no data), `'success'` (have data), or `'error'` (have an error).
- **`fetchStatus`** answers *"is the queryFn running right now?"* — `'fetching'`, `'paused'` (offline, waiting for network), or `'idle'`.

They are independent because you can have data (`status: 'success'`) *and* be refetching it in the background (`fetchStatus: 'fetching'`). That combination is exactly the stale-while-revalidate case. The boolean flags are derived from these:

| Boolean | Equivalent | Use it for |
|---|---|---|
| `isPending` | `status === 'pending'` | "No data yet" — show a skeleton/spinner on first load. |
| `isError` | `status === 'error'` | Show the error UI. |
| `isSuccess` | `status === 'success'` | Render the data. |
| `isFetching` | `fetchStatus === 'fetching'` | "A request is happening" — subtle refresh indicator. |
| `isLoading` | `isPending && isFetching` | First load that is *actively* fetching (vs paused offline). |
| `isRefetching` | `isFetching && !isPending` | A *background* refetch (data already exists). |

> **Gotcha — `isPending` is not "is a request happening."** A query that is `enabled: false` and has no data is `isPending: true` but `fetchStatus: 'idle'` (nothing is fetching — it's just waiting). If you want "is the spinner-worthy first load happening," use `isLoading` (`isPending && isFetching`). Using `isPending` alone for a spinner can leave a spinner showing forever for a disabled query.

### A query with parameters

To fetch a *specific* item, you put the parameter in the key (so each item gets its own cache slot) and close over it in the `queryFn`. Wrap the whole thing in a custom hook — this is the single best habit in the library (§19), keeping keys and fetchers in one place and out of your components.

```tsx
// A reusable custom hook. Components just call `useTodo(id)` and never see the
// queryKey or queryFn — that detail lives here, in one place.
function useTodo(id: number) {
  return useQuery({
    queryKey: ["todo", id],          // id IS part of the key → separate cache entry per todo
    queryFn: () => fetchTodo(id),    // close over id; the queryFn takes no args here
    enabled: id > 0,                 // don't run until we have a valid id (see §7)
  })
}
```

> **Best practice — never read server data into `useState`.** A tempting anti-pattern is `const [todos, setTodos] = useState(); useEffect(() => setTodos(data), [data])`. Don't. The query *is* your source of truth; read `data` directly. Copying it into local state creates a second, stale copy that you now have to keep in sync — exactly the problem the library exists to eliminate.

---

## 5. staleTime vs gcTime — The Two Clocks

These two options confuse almost everyone, and getting them wrong means either over-fetching (slow, expensive) or showing stale data forever. The trick is to realize they control **completely different things** and are measured by **two separate clocks**.

**`staleTime` controls *freshness* — i.e. refetching.** It answers: *"how long after fetching should I trust this data without checking the server again?"* While data is **fresh** (younger than `staleTime`), mounting a component that uses it shows the cached data and triggers **no** network request. Once it crosses into **stale**, the *next trigger* (a component mounting, the window regaining focus, the network reconnecting) fires a background refetch. `staleTime` does **not** delete anything — stale data is still shown instantly; it just becomes eligible for revalidation. Default is `0`, meaning data is stale the moment it arrives, so the library refetches at every opportunity. That's safe (always fresh) but chatty.

**`gcTime` controls *memory* — i.e. deletion.** It answers: *"once nothing is using this data anymore, how long do I keep it around before throwing it away?"* The `gcTime` clock **only starts when a query becomes inactive** — that is, when the last component using it unmounts. If the user navigates back before `gcTime` elapses, the cached data is still there (instant display). If `gcTime` passes with the query still unused, it is garbage-collected and the next mount starts from `isPending`. Default is 5 minutes. `gcTime` has nothing to do with whether data is fresh or stale — an *active* query is never garbage-collected no matter how old.

Here is the relationship as a table, then a timeline you can trace through:

| | `staleTime` | `gcTime` |
|---|---|---|
| Controls | When to **refetch** (freshness) | When to **delete** (memory) |
| Clock starts | When data is fetched | When the query goes **inactive** |
| Effect when it elapses | Data becomes *stale* → eligible for background refetch | Data is *garbage-collected* → next mount is a cold `isPending` load |
| Default | `0` (immediately stale) | `5 * 60 * 1000` (5 minutes) |
| Old v4 name | (same) | `cacheTime` |

```tsx
useQuery({
  queryKey: ["profile", userId],
  queryFn: () => fetchProfile(userId),
  staleTime: 5 * 60 * 1000,   // fresh for 5 min: no refetch on remount/focus during that window
  gcTime: 10 * 60 * 1000,     // if no component uses it, keep it 10 min before deleting from cache
})
```

A concrete timeline to make it click. Suppose `staleTime: 30s`, `gcTime: 5min`:

```
t=0s    Component A mounts → cache empty → FETCH. Data arrives. (active, FRESH)
t=10s   Component B mounts same key → data is FRESH → instant from cache, NO fetch. (2 subscribers)
t=30s   staleTime elapses → data is now STALE (still shown; no fetch yet — nothing triggered one).
t=40s   You tab away and back → window-focus trigger + data is stale → background REFETCH.
        New data replaces old; UI updates. Fresh clock resets.
... later ...
        Both A and B unmount → query becomes INACTIVE → gcTime clock (5 min) starts.
t+5min  Still unused → garbage-collected → entry removed. Next mount is a cold isPending load.
```

### How to choose values

- **`staleTime: 0` (default):** maximum freshness, maximum requests. Fine for highly dynamic data (live prices, chat) where you want every trigger to revalidate.
- **`staleTime: 30s–5min`:** the sweet spot for most app data (lists, profiles, settings). Cuts redundant requests dramatically while staying reasonably current. Set this as a global default in your `QueryClient` (§2).
- **`staleTime: Infinity`:** never auto-refetch. Use for genuinely static data (a country list, enum options) — you'll refresh it only via explicit invalidation.
- **`gcTime`:** mostly leave at the 5-minute default. Lower it if memory matters and your data is large; raise it (or `Infinity`) if you want instant back-navigation for data the user revisits a lot.

> **Gotcha — `staleTime` and `gcTime` are not the same thing, and `gcTime < staleTime` is usually a mistake.** If `gcTime` is shorter than `staleTime`, data gets deleted while still "fresh," defeating the purpose. As a rule keep `gcTime ≥ staleTime`.

> **⚡ Version note:** `cacheTime` was renamed to `gcTime` in v5 specifically because so many people thought `cacheTime` was "how long until refetch" (that's `staleTime`). The new name — *garbage-collection time* — makes its real job clear.

---

## 6. Refetch Behaviours & Polling

Beyond the initial fetch, TanStack Query refetches data automatically at moments that usually mean "the data might have changed." Each trigger is individually configurable, and **all of them only fire for data that is currently stale** — fresh data is left alone. Understanding these saves you from both surprise requests ("why did it fetch again?!") and missing refreshes.

| Option | Default | What it does |
|---|---|---|
| `refetchOnMount` | `true` | When a component using the query mounts and the data is stale, refetch. (`'always'` ignores staleness; `false` disables.) |
| `refetchOnWindowFocus` | `true` | When the browser tab/window regains focus and data is stale, refetch. The "switch back to the tab and it's up to date" magic. |
| `refetchOnReconnect` | `true` | When the network reconnects after being offline and data is stale, refetch. |
| `refetchInterval` | `false` | Poll: refetch every N milliseconds regardless of focus. For live dashboards. Can be a function returning the next interval. |
| `refetchIntervalInBackground` | `false` | Whether `refetchInterval` keeps polling while the tab is in the background (off by default to save resources). |

```tsx
// A live dashboard that polls every 10 seconds, even while the tab is backgrounded.
useQuery({
  queryKey: ["metrics"],
  queryFn: fetchMetrics,
  refetchInterval: 10_000,                 // poll every 10s
  refetchIntervalInBackground: true,       // keep polling even when the tab isn't focused
})

// Dynamic polling: poll fast until a job finishes, then stop. The function receives
// the query and can read its latest data to decide the next interval (or false = stop).
useQuery({
  queryKey: ["job", jobId],
  queryFn: () => fetchJob(jobId),
  refetchInterval: (query) =>
    query.state.data?.status === "complete" ? false : 2000,  // 2s while running, stop when done
})
```

For most apps, the defaults (`refetchOnWindowFocus: true` especially) are great and give that "always up to date" feel for free. A very common preference, though, is to turn off window-focus refetching during development because it fires every time you alt-tab back from your editor, which is noisy in the Network tab:

```tsx
// In QueryClient defaultOptions — a common, sane production-and-dev choice:
queries: { refetchOnWindowFocus: false, staleTime: 60_000 }
```

> **Manual refetch.** `refetch()` from the hook forces a refetch of *that* query regardless of staleness. Use it for an explicit "Refresh" button. For "refresh because data changed," prefer **invalidation** (§12), which is smarter — it marks data stale and only refetches the queries actually on screen.

---

## 7. enabled, select, placeholderData & initialData

These four options are how you go from "basic fetch" to "polished, fast UX." Each solves a distinct problem.

### `enabled` — conditional and dependent queries

`enabled` is a boolean that controls whether a query is allowed to run at all. When `false`, the query does nothing: it won't fetch on mount, focus, or reconnect, and it sits in the `pending`/`idle` state holding whatever cached data exists (or none). The instant `enabled` flips to `true`, the query springs to life and fetches if needed. This is the mechanism for **dependent queries** (don't fetch B until A's result is available) and **conditional queries** (don't search until the user types something).

```tsx
// Don't fetch the user's projects until we actually have a userId.
const { data: user } = useQuery({ queryKey: ["user"], queryFn: fetchUser })

const { data: projects } = useQuery({
  queryKey: ["projects", user?.id],
  queryFn: () => fetchProjects(user!.id),
  enabled: !!user?.id,   // only runs once user.id exists; before that it's "pending, idle"
})
```

> **Gotcha — disabled queries show `isPending: true`.** Because they have no data and aren't fetching, their `status` is `'pending'` but `fetchStatus` is `'idle'`. Don't render a spinner on `isPending` alone for a query that might be disabled — you'll show an eternal spinner. Check `isLoading` (`isPending && isFetching`) or handle the "waiting for the dependency" state explicitly.

### `select` — transform data & minimize re-renders

`select` runs your data through a transform function *after* fetching, and — critically — the component **only re-renders when the `select` output changes**, not when the underlying data changes in ways you didn't select. This does two jobs: it lets you derive/reshape data (filter, map, pick a field) without polluting the cache, and it's a performance optimization, because a component that only needs the *count* of todos won't re-render when an unrelated todo's title changes.

```tsx
// This component re-renders only when the NUMBER of incomplete todos changes,
// not when any todo's text changes. The cache still holds the full list.
function IncompleteCount() {
  const { data: count } = useQuery({
    queryKey: ["todos"],
    queryFn: fetchTodos,
    select: (todos) => todos.filter((t) => !t.done).length,  // derive a number
  })
  return <span>{count} left</span>
}
```

> **Best practice — keep `select` referentially stable for objects/arrays.** If `select` returns a new array/object every call, the comparison sees a "change" each render and the optimization is lost. For primitives (numbers, strings) it just works. For derived arrays, either memoize the select function or accept that it recomputes. The cache always stores the *raw* `queryFn` result; `select` only changes what *this hook* returns.

### `placeholderData` — show something while loading (and keep previous pages)

`placeholderData` provides temporary data to display *while* the real fetch is in flight. Unlike `initialData`, placeholder data is **not** written to the cache and is **not** treated as real — the query still shows `isPlaceholderData: true` and fetches immediately. Its killer use is **smooth pagination**: passing the special `keepPreviousData` helper keeps the *previous page's* data on screen while the next page loads, so the table doesn't flash empty between pages.

```tsx
import { keepPreviousData } from "@tanstack/react-query"

function TodoTable() {
  const [page, setPage] = useState(1)
  const { data, isPlaceholderData } = useQuery({
    queryKey: ["todos", page],                  // page is in the key → each page is its own entry
    queryFn: () => fetchTodoPage(page),
    placeholderData: keepPreviousData,          // keep showing page N while page N+1 loads
  })

  return (
    <>
      {/* Dim the table while showing stale previous-page data, for a polished feel. */}
      <ul style={{ opacity: isPlaceholderData ? 0.5 : 1 }}>
        {data?.items.map((t) => <li key={t.id}>{t.title}</li>)}
      </ul>
      <button onClick={() => setPage((p) => p + 1)} disabled={isPlaceholderData}>
        Next page
      </button>
    </>
  )
}
```

> **⚡ Version note:** v4's `keepPreviousData: true` option was removed. In v5 you achieve the same with `placeholderData: keepPreviousData` (importing the `keepPreviousData` function). This unified the two "show something while loading" concepts under one option.

### `initialData` — seed the cache with data you already have

`initialData` is data you *already possess* (e.g. you fetched a list and now navigate to a detail page) and want to seed the cache with so the detail view skips the loading state entirely. Unlike `placeholderData`, `initialData` **is written to the cache and treated as real, fully-fetched data** — meaning its `staleTime` clock starts immediately. If your seed is up to date, the query may not refetch at all; if it's possibly stale, set a `staleTime` or use `initialDataUpdatedAt` to tell the library how old it is.

```tsx
function TodoDetail({ id }: { id: number }) {
  const queryClient = useQueryClient()
  const { data } = useQuery({
    queryKey: ["todo", id],
    queryFn: () => fetchTodo(id),
    // Seed from the list query's cached data if it's there — instant render, no spinner.
    initialData: () =>
      queryClient.getQueryData<Todo[]>(["todos"])?.find((t) => t.id === id),
    // Tell the library how fresh the seed is, so it can decide whether to refetch.
    initialDataUpdatedAt: () =>
      queryClient.getQueryState(["todos"])?.dataUpdatedAt,
  })
  return <h1>{data?.title}</h1>
}
```

**`placeholderData` vs `initialData` in one line:** `placeholderData` is *fake, temporary, not cached* (always refetches); `initialData` is *real, persistent, cached* (subject to `staleTime`). Use placeholder for "something to look at"; use initial for "I genuinely already have this data."

---

## 8. Retry & Error Handling in queryFn

When a `queryFn` throws, TanStack Query doesn't immediately give up — by default it **retries 3 times** with exponential backoff before settling into the `error` state. This automatic resilience handles transient network blips gracefully, but you'll want to tune it: retrying a 404 is pointless (the resource genuinely isn't there), while retrying a 503 makes sense.

```tsx
useQuery({
  queryKey: ["todo", id],
  queryFn: () => fetchTodo(id),

  // retry can be a number, a boolean, or a FUNCTION for fine control.
  retry: (failureCount, error) => {
    // Don't retry client errors (4xx) — they won't succeed on retry.
    if (error instanceof HttpError && error.status >= 400 && error.status < 500) return false
    // Retry server/network errors up to 3 times.
    return failureCount < 3
  },

  // retryDelay: how long to wait between attempts. Default is exponential backoff
  // capped at 30s: min(1000 * 2^attempt, 30000). Override for custom timing.
  retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 30_000),
})
```

The clean way to enable status-aware retries is a custom error class so you have the HTTP status in the `error` object:

```tsx
class HttpError extends Error {
  constructor(public status: number, message: string) {
    super(message)
    this.name = "HttpError"
  }
}

async function fetchTodo(id: number): Promise<Todo> {
  const res = await fetch(`/api/todos/${id}`)
  if (!res.ok) throw new HttpError(res.status, `HTTP ${res.status}`)  // carry the status through
  return res.json()
}
```

### Typing errors

In v5 the `error` is typed `Error | null` by default (v4 typed it `unknown`), so `error.message` works without casting as long as you always throw `Error` instances. If you throw your own subclass and want it typed, supply the error type as a generic.

```tsx
const { error } = useQuery<Todo[], HttpError>({ queryKey: ["todos"], queryFn: fetchTodos })
// error is now HttpError | null — error?.status is available.
```

> **Best practice — global error handling via the QueryCache.** Instead of an `onError` on every query (v5 removed per-query `onError`/`onSuccess` callbacks — see §20), wire one handler on the `QueryClient`'s cache. This is the right place for toast notifications and logging.

```tsx
import { QueryClient, QueryCache } from "@tanstack/react-query"

const queryClient = new QueryClient({
  queryCache: new QueryCache({
    // Fires for EVERY query that errors out (after retries). One place for all error toasts.
    onError: (error, query) => {
      toast.error(`Something went wrong: ${error.message}`)
      console.error("Query failed:", query.queryKey, error)
    },
  }),
})
```

For showing error *UI* (not toasts), the modern approach is `throwOnError` / Error Boundaries — covered with Suspense in §17.

---

## 9. Query Keys & Key Factories

The **query key** is the single most important concept to get right, because it *is* the cache. Internally the cache is a map from a serialized key to a cache entry. The key has three jobs at once: it **uniquely identifies** a piece of data, it **deduplicates** (two components with the same key share one fetch and one cache entry), and it **drives refetching** (change any value in the key and you get a brand-new query that fetches fresh).

### The rules

- **Keys must be arrays.** Even a single string is wrapped: `["todos"]`, not `"todos"`.
- **Keys must be serializable** — strings, numbers, booleans, and plain objects/arrays. No functions, no class instances, no `Date` objects (they don't serialize stably). The library hashes the key *deterministically*, and — usefully — **object key order doesn't matter**: `["todos", { page: 1, status: "done" }]` and `["todos", { status: "done", page: 1 }]` are the same key.
- **Include every input the query depends on.** If your `queryFn` reads a variable (an id, a filter, a page number), that variable belongs in the key. This is the rule beginners break most, and it causes the classic bug: stale data that doesn't update when the input changes.

```tsx
["todos"]                                  // a whole list
["todos", { status: "done", page: 2 }]     // a filtered/paginated list — filters in the key
["todo", 5]                                // a single item by id
["projects", projectId, "tasks"]           // hierarchical / nested resources
```

> **Why "include every variable" matters — the #1 query-key bug.** If you write `useQuery({ queryKey: ["todo"], queryFn: () => fetchTodo(id) })` and `id` changes from 5 to 6, the key is *still* `["todo"]`, so the library serves the cached todo #5 and never refetches #6. Putting `id` in the key (`["todo", id]`) makes id=6 a different cache slot, which triggers a fetch automatically. **In TanStack Query, "refetch when X changes" is spelled "put X in the query key."**

### Hierarchical keys enable powerful invalidation

Invalidation does **prefix matching** by default (§12). A structured, hierarchical key scheme means you can invalidate broad or narrow with a single call: `["todos"]` invalidates *every* todo-related query (`["todos", {...}]`, `["todo", 5]`... well, only those starting with `["todos"]`), while `["todos", { page: 2 }]` invalidates just that page. Designing keys hierarchically — broadest segment first, narrowing rightward — is what makes the cache controllable.

### Query key factories — the best-practice pattern

In any non-trivial app, scattering raw key arrays everywhere leads to typos and inconsistency (`["todos"]` here, `["todo-list"]` there → they don't share a cache and invalidation misses them). The fix is a **key factory**: a single object per feature that builds all the keys, so keys are defined once and used by reference. This eliminates typos, gives you autocomplete, and makes invalidation trivial and exact.

```ts
// features/todos/keys.ts — the single source of truth for todo query keys.
export const todoKeys = {
  all: ["todos"] as const,                                   // root for prefix-invalidating everything
  lists: () => [...todoKeys.all, "list"] as const,           // ["todos", "list"]
  list: (filters: TodoFilters) => [...todoKeys.lists(), filters] as const, // ["todos","list",{...}]
  details: () => [...todoKeys.all, "detail"] as const,       // ["todos", "detail"]
  detail: (id: number) => [...todoKeys.details(), id] as const,            // ["todos","detail",5]
}
```

```tsx
// Using the factory — keys are now consistent, typo-proof, and refactor-safe.
useQuery({ queryKey: todoKeys.list({ status: "done" }), queryFn: () => fetchTodos({ status: "done" }) })
useQuery({ queryKey: todoKeys.detail(5), queryFn: () => fetchTodo(5) })

// Invalidate ALL todo queries (lists AND details) — one call, thanks to the shared root:
queryClient.invalidateQueries({ queryKey: todoKeys.all })
// Invalidate only the lists (e.g. after adding a todo — details didn't change):
queryClient.invalidateQueries({ queryKey: todoKeys.lists() })
```

> **Best practice:** co-locate the key factory, the fetchers, and the custom query hooks per feature (`features/todos/{keys,api,hooks}.ts`). Components import the hooks and never touch keys directly. This is the structure that scales.

---

## 10. The Cache Lifecycle (Fresh → Stale → Inactive → GC)

Now we tie §5 and §9 together into the complete picture of what happens to a piece of data over its lifetime. Understanding this lifecycle is the difference between "I cargo-cult `staleTime` values" and "I know exactly why my data did or didn't refetch." Open the Devtools (§19) and you can literally watch entries move through these states with color codes.

A cache entry moves through these states:

1. **Fetching (cold):** the key was just used, no data exists → `queryFn` runs → `isPending` / `isFetching`. (Green/blue in Devtools.)
2. **Fresh:** data has arrived and is younger than `staleTime`. Mounting more components with this key shows it **instantly with no refetch**. This is the "free" state — pure cache hits.
3. **Stale:** `staleTime` has elapsed. Data is *still shown instantly*, but it's now eligible for background refetch. The next trigger (mount / focus / reconnect / invalidation) fires a silent refetch. The vast majority of well-tuned queries spend most of their life here, serving cached data and occasionally revalidating.
4. **Inactive:** every component using the key has unmounted. The data stays in the cache (so back-navigation is instant), but the **`gcTime` countdown begins**. An inactive query is never refetched (nothing's subscribed) but isn't deleted yet.
5. **Garbage-collected:** `gcTime` elapsed while inactive → the entry is removed entirely. The next mount of this key is a cold fetch from state #1.

```
   mount (no data)        staleTime          last unmount         gcTime
        │                     │                    │                 │
        ▼                     ▼                    ▼                 ▼
   ┌─────────┐  data   ┌────────┐  time    ┌────────┐  no users ┌──────────┐  time  ┌─────┐
   │FETCHING │────────▶│ FRESH  │─────────▶│ STALE  │──────────▶│ INACTIVE │───────▶│ GC'd│
   └─────────┘         └────────┘          └────────┘           └──────────┘        └─────┘
                            ▲                   │                     │
                            └───────────────────┘                     │
                          refetch (trigger while stale)               │
                            ▲                                         │
                            └─────────── remount before GC ───────────┘
                                       (instant: data still cached)
```

The two clocks again, mapped onto the diagram: **`staleTime`** governs the FRESH→STALE transition (and thus *whether triggers cause refetches*). **`gcTime`** governs the INACTIVE→GC'd transition (and thus *how long unused data survives*). They never interact directly — one is about freshness, the other about memory — which is the whole reason §5 insists they're different.

> **Mental model:** *Fresh* = "trust the cache, do nothing." *Stale* = "show the cache, but check the server on the next nudge." *Inactive* = "nobody's looking; keep it warm for a while in case they come back." *GC'd* = "gone; start over." Every option you tune is really just shifting the boundaries between these four states.

---

## 11. useMutation — Writing Data & the Mutation Lifecycle

Queries *read*; **mutations** *write* (create, update, delete). The differences from queries are deliberate and important:

- **Mutations don't run automatically.** A query runs when its component mounts; a mutation runs only when you imperatively call `mutate(variables)`. This matches reality — you fetch data on mount, but you create a todo only when the user clicks "Add."
- **Mutations aren't cached by key.** There's no `queryKey`; you don't cache the *result* of "create a todo." Instead, after a successful write you tell the *query* cache to refresh (invalidation, §12) or update it yourself (optimistic updates, §13).
- **Mutations have a rich lifecycle** of callbacks — `onMutate`, `onSuccess`, `onError`, `onSettled` — that fire at precise moments and are where you put your cache-sync logic.

```tsx
"use client"
import { useMutation, useQueryClient } from "@tanstack/react-query"

function AddTodo() {
  const queryClient = useQueryClient()

  const mutation = useMutation({
    // mutationFn: the async write. Receives the `variables` you pass to mutate().
    // Like queryFn, it must return data on success or throw on failure.
    mutationFn: async (newTodo: { title: string }) => {
      const res = await fetch("/api/todos", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(newTodo),
      })
      if (!res.ok) throw new Error("Failed to create todo")
      return res.json() as Promise<Todo>
    },

    // onSuccess: runs only if the mutation succeeded. The classic place to invalidate
    // the affected queries so the UI reflects the new server state.
    onSuccess: (createdTodo) => {
      // Mark the todo list stale so it refetches and shows the new item.
      queryClient.invalidateQueries({ queryKey: ["todos"] })
    },

    // onError: runs only if the mutationFn threw. Show feedback, roll back, etc.
    onError: (error) => {
      toast.error(error.message)
    },

    // onSettled: runs after success OR error — for cleanup that must always happen.
    onSettled: () => {
      // e.g. close a dialog, re-enable a form, ensure a final refetch.
    },
  })

  return (
    <button
      disabled={mutation.isPending}                       // disable while the write is in flight
      onClick={() => mutation.mutate({ title: "New task" })}
    >
      {mutation.isPending ? "Adding…" : "Add Todo"}
    </button>
  )
}
```

### The mutation lifecycle, in order

When you call `mutate(variables)`, the callbacks fire in this exact sequence:

```
mutate(variables)
   │
   ▼
onMutate(variables)         ← runs FIRST, BEFORE the request. Synchronous setup point.
   │                          Used for optimistic updates (§13). Its return value becomes `context`.
   ▼
[ mutationFn runs ]         ← the actual network request
   │
   ├── success ─▶ onSuccess(data, variables, context)
   │
   └── failure ─▶ onError(error, variables, context)
                      │
                      ▼
              onSettled(data | undefined, error | null, variables, context)  ← always runs last
```

Note that `onMutate`'s return value is passed as the `context` argument to `onError`, `onSuccess`, and `onSettled` — this is the mechanism that makes rollback possible (you snapshot the old data in `onMutate` and restore it in `onError`). That's exactly what §13 builds on.

### Mutation return values

| Property | Meaning |
|---|---|
| `mutate(vars, options?)` | Fire the mutation (fire-and-forget). Optional per-call `onSuccess`/`onError`/`onSettled`. |
| `mutateAsync(vars)` | Same, but returns a Promise you can `await` / `try-catch`. Use when you need the result inline. |
| `isPending` | The mutation is in flight. (Was `isLoading` in v4.) |
| `isSuccess` / `isError` | Terminal result flags. |
| `data` / `error` | The mutation's result or thrown error. |
| `variables` | The variables from the most recent `mutate` call (handy for optimistic UI without cache writes — §13). |
| `reset()` | Clear the mutation's state (e.g. dismiss an error message). |

> **`mutate` vs `mutateAsync`.** Prefer `mutate` — it never throws (errors go to `onError`), so you can't forget a `try/catch`. Use `mutateAsync` only when you genuinely need to `await` the result in sequence (e.g. create a parent, then create children with its id) — and then you *must* handle rejection or you'll get an unhandled promise rejection.

> **Per-call callbacks have different lifetimes.** Callbacks on `useMutation({...})` always run. Callbacks passed to `mutate(vars, { onSuccess })` run *in addition*, but are skipped if the component unmounts before the mutation settles. Put cache invalidation in the hook-level callbacks (must always run); put UI-only reactions like "navigate away" in the call-level ones.

---

## 12. Query Invalidation & Refetching

Invalidation is the primary way you keep the UI in sync after data changes. The idea: after a mutation (or any event that means "server data changed"), you tell the cache *"these queries are now out of date."* TanStack Query then marks the matching entries **stale** and **immediately refetches the ones that are currently active** (on screen), while leaving inactive ones to refetch lazily the next time they're used. You get correctness (everything refreshes) without waste (off-screen queries don't fetch until needed).

The matching is **prefix-based by default**, which is why hierarchical keys (§9) are so powerful: one invalidation call can refresh a whole subtree of related queries.

```tsx
const queryClient = useQueryClient()

// Invalidate by prefix (DEFAULT). This matches ["todos"] AND ["todos", 5] AND
// ["todos", { page: 2 }] — every query whose key STARTS WITH ["todos"].
queryClient.invalidateQueries({ queryKey: ["todos"] })

// Exact match only — invalidates ONLY ["todos"], not its children.
queryClient.invalidateQueries({ queryKey: ["todos"], exact: true })

// Invalidate everything in the cache (use sparingly — refetches all active queries).
queryClient.invalidateQueries()

// Predicate matching for complex rules the prefix can't express:
queryClient.invalidateQueries({
  predicate: (query) =>
    query.queryKey[0] === "todos" && (query.queryKey[1] as any)?.page > 5,
})
```

### `refetchType` — how aggressive the refetch is

Invalidation marks queries stale *and* refetches them, but you can control *which* matching queries actually refetch right now via `refetchType`:

- `'active'` (default): refetch only the matching queries that are currently rendered. Off-screen ones become stale and refetch when next mounted.
- `'all'`: refetch *every* matching query immediately, even inactive ones.
- `'none'`: just mark them stale, don't refetch any now (they'll refetch on their next trigger).

```tsx
// Mark stale but don't refetch immediately — let them refetch lazily on next use.
queryClient.invalidateQueries({ queryKey: ["todos"], refetchType: "none" })
```

> **Invalidate vs `refetch()` vs `setQueryData`.** Three tools, three jobs: **`invalidateQueries`** = "this data is stale, refresh the right things" — the default choice after a mutation. **`refetch()`** (from a hook) = "refetch *this specific* query now regardless of staleness" — for an explicit refresh button. **`setQueryData`** (§13) = "I already know the new value, write it straight into the cache" — for optimistic updates or when the mutation returns the updated record.

> **Tip — return the invalidation promise.** If you `return queryClient.invalidateQueries(...)` from `onSuccess`, the mutation's `isPending` stays true until the refetch completes, so a single spinner covers both the write *and* the refresh. Without the `return`, the button re-enables before the list updates, which can look janky.

---

## 13. Optimistic Updates (Both Patterns)

An **optimistic update** shows the result of a mutation in the UI *immediately*, before the server confirms it, on the bet that it will succeed. If it does (the common case), the user enjoyed instant feedback. If it fails (rare), you **roll back** to the previous state. This is what makes likes, toggles, reordering, and inline edits feel native rather than laggy. The cost is complexity, so reserve it for interactions where instant feedback genuinely matters and failures are uncommon.

There are **two patterns**, and choosing the right one matters.

### Pattern A — optimistically write the cache (for shared / list data)

Use this when the change must be reflected in data that **multiple components read** — e.g. toggling a todo's `done` status, which should update the list everywhere. You write the new value into the query cache in `onMutate`, snapshot the old value so you can restore it, and reconcile with the server in `onSettled`. This is the more powerful pattern and the one you reach for most.

```tsx
const queryClient = useQueryClient()

const toggleTodo = useMutation({
  mutationFn: (todo: Todo) => updateTodo({ ...todo, done: !todo.done }),

  // onMutate runs BEFORE the request — the optimistic moment.
  onMutate: async (todo) => {
    // 1. Cancel any in-flight refetch for this key, so a slow background fetch
    //    doesn't land AFTER our optimistic write and overwrite it. MUST be awaited.
    await queryClient.cancelQueries({ queryKey: ["todos"] })

    // 2. Snapshot the current cache value so we can roll back on error.
    const previousTodos = queryClient.getQueryData<Todo[]>(["todos"])

    // 3. Optimistically write the new value into the cache. setQueryData must return
    //    a NEW array/object reference (immutable update) so React sees the change.
    queryClient.setQueryData<Todo[]>(["todos"], (old) =>
      old?.map((t) => (t.id === todo.id ? { ...t, done: !t.done } : t)),
    )

    // 4. Return a context object — it becomes the `context` arg in onError/onSettled.
    return { previousTodos }
  },

  // If the mutation fails, restore the snapshot we took in onMutate.
  onError: (_err, _todo, context) => {
    if (context?.previousTodos) {
      queryClient.setQueryData(["todos"], context.previousTodos)
    }
  },

  // Always re-sync with the server afterwards, so the cache matches the source of truth
  // (the server may have applied other changes, normalized the record, etc.).
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ["todos"] })
  },
})

// Usage: the UI flips instantly; if the request fails it flips back.
;<input type="checkbox" checked={todo.done} onChange={() => toggleTodo.mutate(todo)} />
```

The four steps in `onMutate` are a *fixed recipe* — cancel, snapshot, write, return — and getting any one wrong breaks rollback or causes flicker:

1. **Cancel** in-flight queries (`await` it!) so a background refetch can't clobber your optimistic write.
2. **Snapshot** the old data for rollback.
3. **Write** the optimistic value with `setQueryData`, returning a new reference.
4. **Return** the snapshot as context for `onError`.

### Pattern B — optimistically render via the mutation's `variables` (for single-component UI)

When the optimistic state only matters in **one component** — e.g. an "Add comment" form where you just want the new comment to appear in *this* list while it saves — you don't need to touch the cache at all. The mutation exposes `variables` (and `isPending`), so you can render the pending item directly. This is simpler, has no rollback code (a failure just removes the pending item), and is the right call for localized UI.

```tsx
function AddComment({ postId }: { postId: number }) {
  const queryClient = useQueryClient()
  const { data: comments } = useQuery({
    queryKey: ["comments", postId],
    queryFn: () => fetchComments(postId),
  })

  const addComment = useMutation({
    mutationFn: (text: string) => postComment(postId, text),
    onSettled: () => queryClient.invalidateQueries({ queryKey: ["comments", postId] }),
  })

  return (
    <>
      <ul>
        {comments?.map((c) => <li key={c.id}>{c.text}</li>)}
        {/* While pending, render the in-flight comment from `variables` — no cache write.
            On success the invalidation replaces it with the real one; on error it vanishes. */}
        {addComment.isPending && (
          <li style={{ opacity: 0.5 }}>{addComment.variables}</li>
        )}
        {/* Optionally show a retry affordance on failure. */}
        {addComment.isError && (
          <li style={{ color: "red" }}>
            Failed to post.{" "}
            <button onClick={() => addComment.mutate(addComment.variables!)}>Retry</button>
          </li>
        )}
      </ul>
      <CommentForm onSubmit={(text) => addComment.mutate(text)} />
    </>
  )
}
```

> **Choosing between them:** if the optimistic change needs to appear in *more than one place* or in data other components read → **Pattern A (cache write)**. If it's a *single list/component* showing its own pending item → **Pattern B (variables)** — less code, no rollback. Many apps use both, for different interactions.

> **Gotcha — forgetting `await cancelQueries`.** This is the most common optimistic-update bug. If you don't await the cancel, a refetch already in flight can resolve *after* your `setQueryData` and silently overwrite your optimistic value with the old server data, making the UI "flicker back." Always `await queryClient.cancelQueries(...)` as step one of `onMutate`.

---

## 14. Dependent & Parallel Queries

Real screens need multiple pieces of data, sometimes independent and sometimes chained. TanStack Query handles both naturally.

### Parallel queries — just declare several

If you call `useQuery` multiple times in one component, they run **in parallel** automatically — there's no special API needed. Each is independent with its own status, so you can render each part as it arrives or wait for all.

```tsx
function Dashboard() {
  // These three fetches kick off simultaneously on mount.
  const user = useQuery({ queryKey: ["user"], queryFn: fetchUser })
  const stats = useQuery({ queryKey: ["stats"], queryFn: fetchStats })
  const feed = useQuery({ queryKey: ["feed"], queryFn: fetchFeed })

  // Render progressively, or gate on all three:
  if (user.isPending || stats.isPending || feed.isPending) return <Skeleton />
  return <Layout user={user.data} stats={stats.data} feed={feed.data} />
}
```

### Dynamic parallel queries — `useQueries`

When the *number* of queries isn't known at write-time (you have an array of ids and want one query each), you can't call `useQuery` in a loop (that violates the Rules of Hooks). `useQueries` solves this: you pass an array of query options and get back an array of results, all fetched in parallel.

```tsx
import { useQueries } from "@tanstack/react-query"

function TodoList({ ids }: { ids: number[] }) {
  const results = useQueries({
    queries: ids.map((id) => ({
      queryKey: ["todo", id],
      queryFn: () => fetchTodo(id),
    })),
    // Optional `combine` merges the array of results into one value, re-running only when needed.
    combine: (results) => ({
      data: results.map((r) => r.data),
      isPending: results.some((r) => r.isPending),
    }),
  })

  if (results.isPending) return <Skeleton />
  return <ul>{results.data.map((t) => t && <li key={t.id}>{t.title}</li>)}</ul>
}
```

### Dependent (serial) queries — `enabled`

When query B needs a value from query A's result, you chain them with `enabled` (§7): B stays disabled until A's data exists, then automatically runs. This necessarily makes them *sequential* (a waterfall), which is sometimes unavoidable but worth being aware of — if you can fetch in parallel instead, prefer that.

```tsx
// Get the user, then their projects, then that project's tasks — a dependent chain.
const { data: user } = useQuery({ queryKey: ["user", email], queryFn: () => fetchUserByEmail(email) })

const { data: projects } = useQuery({
  queryKey: ["projects", user?.id],
  queryFn: () => fetchProjects(user!.id),
  enabled: !!user?.id,          // waits for user.id
})
```

> **Gotcha — accidental waterfalls.** Dependent queries are sometimes necessary, but each link adds a round-trip's latency. If query B *could* run without A's result (it just needs a value you already have), don't gate it on A. Reserve `enabled`-chaining for genuine data dependencies, and consider a single backend endpoint that returns everything at once for hot paths.

---

## 15. useInfiniteQuery & Pagination

There are two distinct UI patterns for large datasets, and they map to two different tools:

- **Discrete pagination** (page 1, 2, 3 with prev/next buttons): use a regular `useQuery` with the page number in the key, plus `placeholderData: keepPreviousData` so the table doesn't flash empty between pages (covered in §7). Each page is its own cache entry.
- **Infinite scroll / "Load more"** (accumulate pages into one growing list): use **`useInfiniteQuery`**, which is purpose-built for accumulating pages under a single cache entry.

`useInfiniteQuery` differs from `useQuery` in shape: instead of `data` being your array, `data.pages` is an **array of pages** (each the result of one fetch) and `data.pageParams` is the array of params used. You provide `initialPageParam` (the param for the first page) and `getNextPageParam` (given the last page, compute the next param — or return `undefined`/`null` to signal "no more pages," which sets `hasNextPage` to false).

```tsx
import { useInfiniteQuery } from "@tanstack/react-query"

interface Page { items: Post[]; nextCursor: number | null }

function Feed() {
  const {
    data,
    fetchNextPage,        // call this to load the next page
    hasNextPage,          // false once getNextPageParam returns undefined/null
    isFetchingNextPage,   // true while a "load more" fetch is in flight (vs initial load)
    isPending,
    isError,
    error,
  } = useInfiniteQuery({
    queryKey: ["posts"],
    // pageParam is the cursor/offset; its type matches initialPageParam.
    queryFn: ({ pageParam }) => fetchPosts(pageParam),
    initialPageParam: 0,                       // the param for the FIRST page
    // Given the last fetched page (and all pages), return the next param, or undefined to stop.
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
    // For bidirectional (e.g. chat scrolling up), also provide getPreviousPageParam.
  })

  if (isPending) return <Skeleton />
  if (isError) return <p>{error.message}</p>

  return (
    <>
      {/* Flatten the pages for rendering. data.pages is an array of Page objects. */}
      {data.pages.flatMap((page) => page.items).map((post) => (
        <Post key={post.id} {...post} />
      ))}

      {hasNextPage && (
        <button onClick={() => fetchNextPage()} disabled={isFetchingNextPage}>
          {isFetchingNextPage ? "Loading…" : "Load more"}
        </button>
      )}
    </>
  )
}
```

### Auto-loading on scroll with IntersectionObserver

A "Load more" button is the simplest UX, but true infinite scroll loads the next page when a sentinel element scrolls into view. The native `IntersectionObserver` is the right tool — no scroll-event listeners needed.

```tsx
function InfiniteFeed() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteQuery({
    queryKey: ["posts"],
    queryFn: ({ pageParam }) => fetchPosts(pageParam),
    initialPageParam: 0,
    getNextPageParam: (last) => last.nextCursor ?? undefined,
  })

  const loadMoreRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    const el = loadMoreRef.current
    if (!el || !hasNextPage) return
    // Fire fetchNextPage when the sentinel becomes visible.
    const observer = new IntersectionObserver((entries) => {
      if (entries[0].isIntersecting && hasNextPage && !isFetchingNextPage) {
        fetchNextPage()
      }
    })
    observer.observe(el)
    return () => observer.disconnect()   // cleanup on unmount / deps change
  }, [hasNextPage, isFetchingNextPage, fetchNextPage])

  return (
    <>
      {data?.pages.flatMap((p) => p.items).map((post) => <Post key={post.id} {...post} />)}
      <div ref={loadMoreRef} style={{ height: 1 }} />   {/* the invisible sentinel */}
      {isFetchingNextPage && <Spinner />}
    </>
  )
}
```

> **Gotcha — refetching an infinite query refetches *every* loaded page.** When an infinite query is invalidated, by default it refetches all currently-loaded pages sequentially (to keep them consistent) — which can be a lot of requests if the user scrolled far. There's a `maxPages` option (cap how many pages are kept) to bound this. Be deliberate about invalidating infinite queries.

> **Cursor vs offset pagination.** Prefer **cursor-based** (`nextCursor`) over offset (`page * size`) when the underlying data changes, because offsets shift when items are inserted/deleted (you get duplicates or gaps). `getNextPageParam` works with either; just return whatever your API needs next.

---

## 16. Prefetching

Prefetching means loading data into the cache *before* a component that needs it renders, so when it does render the data is already there and the user sees no spinner. It's one of the cheapest ways to make an app feel instant. The most common triggers are **hovering a link** (the user is likely about to click) and **route transitions**. Prefetching uses the imperative `QueryClient` methods rather than the hooks.

```tsx
const queryClient = useQueryClient()

// Prefetch on hover — by the time the user clicks, the detail page's data is cached.
<Link
  href={`/posts/${id}`}
  onMouseEnter={() =>
    queryClient.prefetchQuery({
      queryKey: ["post", id],
      queryFn: () => fetchPost(id),
      staleTime: 60_000,        // don't re-prefetch if we already fetched it recently
    })
  }
>
  {title}
</Link>
```

The relevant `QueryClient` methods:

| Method | Behaviour | Use for |
|---|---|---|
| `prefetchQuery({ queryKey, queryFn })` | Fetch into the cache if not already fresh. Returns `Promise<void>` (never throws — errors are swallowed). | Warming the cache (hover, route prefetch) where you don't need the data inline. |
| `fetchQuery({ queryKey, queryFn })` | Fetch and **return the data** (or throw). | When you need the data right now in imperative code (e.g. server-side, or before navigating). |
| `ensureQueryData({ queryKey, queryFn })` | Return cached data if present, otherwise fetch it. | "Give me this data, fetching only if necessary" — great in route loaders. |
| `prefetchInfiniteQuery(...)` | The infinite-query equivalent. | Pre-warming the first page of a feed. |

```tsx
// In a framework route loader (or any imperative flow), ensure data is ready before render:
async function loader({ params }) {
  await queryClient.ensureQueryData({
    queryKey: ["post", params.id],
    queryFn: () => fetchPost(params.id),
  })
  return null   // the component's useQuery(["post", id]) will now hit a warm cache
}
```

> **Best practice — set a `staleTime` on prefetches.** Without it (default `staleTime: 0`), the prefetched data is instantly stale, so the real `useQuery` will refetch on mount anyway, partly wasting the prefetch. A small `staleTime` makes the prefetch "stick" long enough to actually skip the load.

---

## 17. Suspense: useSuspenseQuery & Error Boundaries

React's **Suspense** lets a component "suspend" rendering while data loads, delegating the loading UI to the nearest `<Suspense fallback={...}>` boundary above it — and errors to the nearest **Error Boundary**. TanStack Query supports this fully via **`useSuspenseQuery`**. The payoff is dramatically cleaner components: because the hook *suspends* instead of returning a pending state, `data` is **guaranteed defined** — no `isPending` checks, no `if (!data) return null`, no optional chaining everywhere. Loading and error handling move *out* of the component and *up* to declarative boundaries. (See the [React 19 guide](./REACT_19_GUIDE.md) for Suspense and Error Boundary fundamentals.)

```tsx
"use client"
import { useSuspenseQuery } from "@tanstack/react-query"

// Notice: no isPending, no isError, no `data?` — data is typed as T, never undefined.
function Todos() {
  const { data } = useSuspenseQuery({ queryKey: ["todos"], queryFn: fetchTodos })
  return <ul>{data.map((t) => <li key={t.id}>{t.title}</li>)}</ul>  // data is guaranteed here
}
```

```tsx
// The parent provides the loading fallback (Suspense) and the error UI (Error Boundary).
import { ErrorBoundary } from "react-error-boundary"
import { Suspense } from "react"

function TodosPage() {
  return (
    <ErrorBoundary fallback={<p>Failed to load todos.</p>}>
      <Suspense fallback={<Skeleton />}>
        <Todos />               {/* suspends while loading; throws to ErrorBoundary on error */}
      </Suspense>
    </ErrorBoundary>
  )
}
```

### Resetting the Error Boundary after a retry

When an error boundary catches a query error, you want a "Try again" button that both resets the boundary *and* tells TanStack Query to retry. The `QueryErrorResetBoundary` wires these together.

```tsx
import { QueryErrorResetBoundary } from "@tanstack/react-query"
import { ErrorBoundary } from "react-error-boundary"

function TodosPage() {
  return (
    <QueryErrorResetBoundary>
      {({ reset }) => (
        <ErrorBoundary
          onReset={reset}                       // reset the query error state when the boundary resets
          fallbackRender={({ resetErrorBoundary }) => (
            <div>
              <p>Something went wrong.</p>
              <button onClick={resetErrorBoundary}>Try again</button>
            </div>
          )}
        >
          <Suspense fallback={<Skeleton />}>
            <Todos />
          </Suspense>
        </ErrorBoundary>
      )}
    </QueryErrorResetBoundary>
  )
}
```

### When to use Suspense mode vs the standard hook

- **`useSuspenseQuery`** shines when you have several queries on a page and want one coordinated loading/error boundary, and when you want maximum component cleanliness. It pairs beautifully with React 19 Server Components streaming.
- **Standard `useQuery`** is better when you want *granular*, inline loading/error states per widget (a dashboard where each card loads independently), or when a query is conditionally disabled (`enabled: false` doesn't combine cleanly with Suspense, since there's nothing to suspend on).

> **Gotcha — `useSuspenseQuery` has no `enabled` option.** Suspense mode always fetches; there's no "pending and idle" state because the component would suspend forever. For conditional fetching, use the standard `useQuery`. Also, multiple `useSuspenseQuery` calls in one component run *serially* (each suspends in turn) — use `useSuspenseQueries` to parallelize them.

> **⚡ Version note:** dedicated Suspense hooks (`useSuspenseQuery`, `useSuspenseInfiniteQuery`, `useSuspenseQueries`) are a v5 feature. In v4 you enabled Suspense via a `suspense: true` option on `useQuery`, which is gone in v5 — the explicit hooks give correct TypeScript (data is non-nullable) and clearer intent.

---

## 18. Next.js Integration (Server Prefetch + Hydration)

In a Next.js App Router app you have two complementary tools: **Server Components**, which fetch on the server for a fast first paint and zero client JS, and **TanStack Query**, which gives the *client* caching, background refresh, mutations, and interactivity. The best architecture combines them: **fetch the initial data on the server, then hand it to TanStack Query on the client so it can take over** — keeping it fresh, refetching after mutations, paginating, etc. This handoff is called **hydration**, and it's done with `dehydrate` (serialize the server cache) + `HydrationBoundary` (rebuild it in the browser). See the [Next.js 16 guide](./NEXTJS_16_GUIDE.md) for Server Component fundamentals.

### When to reach for TanStack Query in Next.js at all

This is worth being deliberate about, because adding it where it isn't needed is overengineering:

- ✅ **Interactive client data:** infinite scroll, polling dashboards, search-as-you-type, optimistic UI, anything that refetches in response to user actions or mutations.
- ✅ **Client mutations** where you want cache sync without a full page reload.
- ❌ **Static or rarely-changing page data** that's read once → just fetch it directly in a Server Component (or use Next.js's own `fetch` caching). No TanStack Query needed.

The rule of thumb: **Server Components for the initial read; TanStack Query for ongoing client-side reads and writes.**

### The setup: a request-scoped QueryClient on the server

On the server you must create a **fresh `QueryClient` per request** (never a module singleton — that would leak one user's data into another's response). A small helper using React's `cache()` gives you one client per request.

```tsx
// lib/get-query-client.ts
import { QueryClient, defaultShouldDehydrateQuery } from "@tanstack/react-query"
import { cache } from "react"

// `cache()` memoizes per-request on the server, so all server code in one request
// shares ONE QueryClient. On the client this just creates one (the provider handles reuse).
export const getQueryClient = cache(
  () =>
    new QueryClient({
      defaultOptions: {
        queries: { staleTime: 60 * 1000 },   // a non-zero staleTime avoids an instant client refetch
        dehydrate: {
          // Include pending queries so streaming SSR can dehydrate in-flight fetches too.
          shouldDehydrateQuery: (query) =>
            defaultShouldDehydrateQuery(query) || query.state.status === "pending",
        },
      },
    }),
)
```

### Prefetch on the server, hydrate on the client

```tsx
// app/posts/page.tsx — a Server Component (no "use client"). Runs on the server.
import { dehydrate, HydrationBoundary } from "@tanstack/react-query"
import { getQueryClient } from "@/lib/get-query-client"
import { Posts } from "./posts"
import { getPosts } from "@/lib/api"

export default async function PostsPage() {
  const queryClient = getQueryClient()

  // Prefetch on the server. Use prefetchQuery (not fetchQuery) here — it doesn't throw,
  // so a fetch failure won't crash the page; the client query can retry instead.
  await queryClient.prefetchQuery({ queryKey: ["posts"], queryFn: getPosts })

  return (
    // dehydrate() serializes the server cache to JSON; HydrationBoundary rebuilds it
    // into the client's QueryClient so the client query finds data already present.
    <HydrationBoundary state={dehydrate(queryClient)}>
      <Posts />
    </HydrationBoundary>
  )
}
```

```tsx
// app/posts/posts.tsx — a Client Component. Uses the SAME queryKey as the server prefetch.
"use client"
import { useQuery } from "@tanstack/react-query"
import { getPosts } from "@/lib/api"

export function Posts() {
  // Because the server prefetched ["posts"], this data is hydrated and present on first
  // render — NO loading flash. TanStack Query then keeps it fresh on the client
  // (background refetch when stale, refetch after mutations, etc.).
  const { data } = useQuery({ queryKey: ["posts"], queryFn: getPosts })
  return <ul>{data?.map((p) => <li key={p.id}>{p.title}</li>)}</ul>
}
```

The mechanism in one breath: the server fills its per-request cache, `dehydrate()` turns that cache into a JSON blob shipped with the HTML, `HydrationBoundary` reads that blob into the *browser's* QueryClient, and the client `useQuery` — using the matching `queryKey` — finds the data already cached, so it renders instantly and then manages it from there.

> **Gotcha — keys must match exactly.** The server prefetch and the client `useQuery` must use the *same* `queryKey` (and ideally the same fetcher). A mismatched key means the hydrated data sits in a different cache slot and the client refetches from scratch, defeating the prefetch. Key factories (§9) make this reliable.

> **Gotcha — set a non-zero `staleTime`.** With the default `staleTime: 0`, the hydrated data is instantly stale and the client refetches immediately on mount — you did the server work *and* a client fetch. A small `staleTime` (e.g. 60s) lets the hydrated data satisfy the first render without an immediate refetch.

> **⚡ Version note:** for Server Actions and mutations in the App Router, you can still use `useMutation` on the client and call your Server Action as the `mutationFn`, then invalidate — combining Next.js's server mutations with TanStack Query's client cache sync.

---

## 19. Devtools, Testing & Performance

### Devtools — your single best learning tool

The React Query Devtools render a floating panel that shows **every query in the cache**: its key, current state (fresh/stale/fetching/inactive, color-coded), the cached data, when it was last updated, and buttons to manually refetch, invalidate, or reset each one. While learning, keep it open constantly — watching entries go *fresh → stale → fetching → fresh* makes the abstract lifecycle (§10) concrete in a way no amount of reading can. It's excluded from production builds automatically.

```tsx
import { ReactQueryDevtools } from "@tanstack/react-query-devtools"
// Render once, inside the provider (see §2). initialIsOpen controls whether it starts expanded.
<ReactQueryDevtools initialIsOpen={false} buttonPosition="bottom-right" />
```

### Testing

The key principle for testing components that use TanStack Query: wrap them in a `QueryClientProvider` with a **test-specific client** that disables retries (so failed-request tests fail fast instead of retrying for seconds) and turns off background noise. Mock the network at the boundary (with MSW — Mock Service Worker — ideally, so you test real fetch behaviour) rather than mocking the hooks themselves.

```tsx
// test-utils.tsx — a render helper that provides a fresh, retry-free client per test.
import { QueryClient, QueryClientProvider } from "@tanstack/react-query"
import { render } from "@testing-library/react"

function createTestClient() {
  return new QueryClient({
    defaultOptions: {
      queries: { retry: false, gcTime: Infinity },   // no retries; keep cache stable during the test
      mutations: { retry: false },
    },
  })
}

export function renderWithClient(ui: React.ReactNode) {
  const client = createTestClient()             // a FRESH client per test = no cross-test cache leakage
  return render(<QueryClientProvider client={client}>{ui}</QueryClientProvider>)
}
```

```tsx
// A test, using the helper. waitFor lets the async query resolve before asserting.
import { screen, waitFor } from "@testing-library/react"

test("renders todos from the API", async () => {
  renderWithClient(<Todos />)
  // First the skeleton shows; then the data arrives.
  await waitFor(() => expect(screen.getByText("Buy milk")).toBeInTheDocument())
})
```

> **Best practice — a fresh `QueryClient` per test.** Sharing one client across tests leaks cached data between them, causing flaky, order-dependent failures. Create a new client in each test (or in `beforeEach`).

### Performance & best practices (the consolidated list)

1. **Wrap query/mutation logic in custom hooks** (`useTodos`, `useAddTodo`). Components stay clean; keys and fetchers live in one place.
2. **Use query key factories** (§9) for consistency and exact, refactor-safe invalidation.
3. **Set a sensible global `staleTime`** (30–60s) in `defaultOptions` to cut redundant requests across the whole app.
4. **Always throw in `queryFn`/`mutationFn` on failure** — including manual `res.ok` checks for `fetch`.
5. **Keep server state out of `useState`.** Read `data` directly; never mirror it into local state.
6. **Use `select`** to derive data and minimize re-renders (a component subscribing to a slice only re-renders when that slice changes).
7. **Use `placeholderData: keepPreviousData`** for paginated tables to avoid empty flashes.
8. **Invalidate after mutations** (or use optimistic updates) to keep the UI in sync.
9. **Prefetch likely-next data** on hover or route change for instant transitions.
10. **Co-locate per feature:** `features/<feature>/{keys,api,hooks}.ts`.
11. **One `QueryClient`** — module singleton in client apps, `useState`-created (and request-scoped on the server) in SSR.
12. **Reach for Suspense** (`useSuspenseQuery`) when you want coordinated loading/error boundaries and cleaner components.
13. **Keep the Devtools open while developing** — it's the fastest path to a correct mental model.

---

## 20. v5 Migration, Gotchas & Quick Reference

### v5 changes (if you're reading older v4 tutorials)

TanStack Query v5 was an intentional simplification. The big renames and removals:

| v4 | v5 | Why |
|---|---|---|
| `isLoading` | `isPending` (no data yet) | "Loading" was ambiguous; `isPending` clearly means "no data." `isLoading` still exists as `isPending && isFetching`. |
| `cacheTime` | `gcTime` | The old name implied "time until refetch" (that's `staleTime`). "Garbage-collection time" is accurate. |
| `keepPreviousData: true` | `placeholderData: keepPreviousData` | Unified the two "show something while loading" concepts. |
| `useQuery(key, fn, opts)` | `useQuery({ queryKey, queryFn, ...opts })` | **Single object signature only** — the positional overloads are gone. |
| `suspense: true` option | `useSuspenseQuery` hook | Dedicated hooks give correct (non-nullable) types and clearer intent. |
| `onSuccess` / `onError` / `onSettled` on `useQuery` | **Removed** from queries | Per-query callbacks caused bugs (they fired per-observer, not per-fetch). Use the global `QueryCache`/`MutationCache` callbacks, or `useEffect` on `data`. (Mutations *keep* these callbacks.) |
| error type `unknown` | error type `Error` | Sensible default; you can still override with a generic. |

> **⚡ The callback removal is the one that trips up migrators most.** In v5, `useQuery` has **no** `onSuccess`/`onError`/`onSettled`. For side effects on success, either react to `data` in a `useEffect`, or — for cross-cutting concerns like error toasts — use the `QueryCache`'s `onError` (§8). **Mutations still have these callbacks**, which is where most "do something after this succeeds" logic belongs anyway.

### Common gotchas (the greatest hits)

- ❌ **`queryFn` that doesn't throw on HTTP errors.** `fetch` resolves on 404/500; check `res.ok` and throw, or you cache an error response as valid data. (§4, §8)
- ❌ **Forgetting a variable in the `queryKey`.** The query serves stale cached data and never refetches when the input changes. (§9)
- ❌ **Recreating `QueryClient` on every render.** Wipes the cache constantly. Use a module singleton or `useState(() => new QueryClient())`. (§2)
- ❌ **Using TanStack Query for client/UI state** (modal open, form inputs). That's `useState`/Zustand. (§1)
- ❌ **Over-fetching from `staleTime: 0` + `refetchOnWindowFocus: true`.** Tune `staleTime` up. (§5, §6)
- ❌ **Mutating cached data in place** instead of returning a new reference from `setQueryData`. React won't see the change. (§13)
- ❌ **Forgetting `"use client"`** on components using the hooks in the Next.js App Router. (§2, §18)
- ❌ **Not awaiting `cancelQueries` in `onMutate`** — an in-flight refetch clobbers your optimistic update. (§13)
- ❌ **`isPending` used as a loading flag for a possibly-disabled query** — shows an eternal spinner. Use `isLoading`. (§4, §7)
- ❌ **Mismatched keys between server prefetch and client `useQuery`** in Next.js — defeats hydration. (§18)

### Quick reference card

```tsx
// READ
const { data, isPending, isError, error, refetch } = useQuery({
  queryKey: ["resource", id],
  queryFn: () => fetchResource(id),
  enabled: !!id,
  staleTime: 60_000,
  select: (d) => d.items,          // optional transform / re-render minimizer
})

// READ (Suspense — data is guaranteed defined; loading/error via boundaries)
const { data } = useSuspenseQuery({ queryKey: ["resource", id], queryFn: () => fetchResource(id) })

// WRITE
const { mutate, isPending } = useMutation({
  mutationFn: createResource,
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ["resource"] }),
})
mutate(payload)

// PAGINATE (discrete pages)
useQuery({ queryKey: ["feed", page], queryFn: () => fetchPage(page), placeholderData: keepPreviousData })

// INFINITE SCROLL
const { data, fetchNextPage, hasNextPage } = useInfiniteQuery({
  queryKey: ["feed"],
  queryFn: ({ pageParam }) => fetchPage(pageParam),
  initialPageParam: 0,
  getNextPageParam: (last) => last.nextCursor ?? undefined,
})

// CACHE CONTROL (imperative, via useQueryClient())
const queryClient = useQueryClient()
queryClient.invalidateQueries({ queryKey: ["resource"] })   // mark stale + refetch active
queryClient.setQueryData(["resource", id], next)            // write the cache directly
queryClient.getQueryData(["resource", id])                  // read the cache
queryClient.prefetchQuery({ queryKey, queryFn })            // warm the cache (hover/route)
queryClient.cancelQueries({ queryKey: ["resource"] })       // cancel in-flight (optimistic updates)
```

### Decision guide

- **Reading server data on the client?** → `useQuery` (or `useSuspenseQuery` with boundaries).
- **Changing server data?** → `useMutation` + invalidate (or optimistic update).
- **Discrete pages?** → `useQuery` + `placeholderData: keepPreviousData`. **Infinite scroll?** → `useInfiniteQuery`.
- **Many parallel queries from an array?** → `useQueries`.
- **Query B depends on query A?** → `enabled: !!a.data`.
- **Want instant UX?** → optimistic updates (§13) + prefetch on hover (§16).
- **First-load data in Next.js?** → server `prefetchQuery` + `HydrationBoundary` (§18).
- **Need client state, not server state?** → not this library — use `useState` / [Zustand](./ZUSTAND_GUIDE.md).

---

## 21. Study Path & Build-to-Learn Projects

Knowledge sticks when you *build*. Read for understanding, then immediately apply each cluster of concepts. The order below front-loads the everyday essentials (queries, keys, the two clocks, mutations) and saves the advanced material (optimistic updates, infinite scroll, Suspense, SSR) for once the fundamentals are automatic. Keep the **Devtools open** the entire time — it turns the invisible cache into something you can watch.

**Suggested order:**
1. §1–4 — what server state is, setup, and `useQuery` basics. Internalize `status` vs `fetchStatus`.
2. §5, §10 — `staleTime` vs `gcTime` and the cache lifecycle. This is the conceptual core; don't rush it.
3. §9 — query keys & factories. "Refetch when X changes = put X in the key."
4. §6–8 — refetch behaviours, `enabled`/`select`/`placeholderData`, retry & errors.
5. §11–12 — mutations and invalidation. The read-write loop.
6. §13 — optimistic updates (both patterns).
7. §14–16 — dependent/parallel queries, infinite scroll, prefetching.
8. §17–18 — Suspense and Next.js hydration.

**Build these to cement it (each targets specific sections):**
1. **Todo app (CRUD)** — `useQuery` for the list, `useMutation` for add/toggle/delete, invalidation after each write. Exercises §4, §9, §11, §12. *Stretch:* a custom-hook + key-factory structure (§9, §19).
2. **Paginated data table** — page number in the key, `placeholderData: keepPreviousData` for flicker-free paging, a "rows per page" control. Exercises §5, §7.
3. **Infinite social feed** — `useInfiniteQuery` with cursor pagination and `IntersectionObserver` auto-load. Exercises §15.
4. **Optimistic "like" button** — Pattern A (cache write) with cancel/snapshot/rollback, plus an "Add comment" form using Pattern B (variables). Exercises §13.
5. **Dashboard with live data** — parallel queries, `refetchInterval` polling, a global `useIsFetching` top-bar loader, Suspense boundaries per widget. Exercises §6, §14, §17.
6. **Next.js app with SSR hydration** — Server Component prefetch + `HydrationBoundary`, client mutations calling Server Actions, matching key factories. Exercises §18, and ties into the [Next.js 16 guide](./NEXTJS_16_GUIDE.md).

**Next steps after this guide:** explore persisting the cache to `localStorage` (`@tanstack/query-persist-client`), the framework-agnostic core for Vue/Svelte/Solid adapters, and pairing TanStack Query (server state) with [Zustand](./ZUSTAND_GUIDE.md) (client state) — the combination covers essentially all state-management needs in a modern React app. For the React primitives this library builds on (Suspense, Error Boundaries, `use`, transitions), see the [React 19 guide](./REACT_19_GUIDE.md).

---

*Part of the offline developer study library. Written for TanStack Query v5 on React 19 as of 2026. The concepts (server-state caching, stale-while-revalidate, the cache lifecycle) are stable; confirm fast-moving API details against tanstack.com/query.*
