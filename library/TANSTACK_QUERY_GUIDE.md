# TanStack Query (React Query) v5 Complete Guide

A full, study-friendly reference for **TanStack Query v5** — the standard library for fetching, caching, syncing, and updating server state in React apps. Includes setup, every core hook, options, mutations, optimistic updates, Next.js integration, patterns, and gotchas.

> **What it is:** TanStack Query (formerly React Query) manages **server state** — data that lives on a server and you fetch/cache in the UI. It is *not* a state manager for client/UI state (use `useState`/Zustand for that). It handles caching, background refetching, deduping, pagination, and loading/error states so you stop writing `useEffect` + `useState` fetch boilerplate.

---

## Table of Contents
1. [Why It Exists](#1-why-it-exists)
2. [Installation & Setup](#2-installation--setup)
3. [Core Concepts & Vocabulary](#3-core-concepts--vocabulary)
4. [useQuery — Reading Data](#4-usequery--reading-data)
5. [Query Keys](#5-query-keys)
6. [Query Options Reference](#6-query-options-reference)
7. [useMutation — Writing Data](#7-usemutation--writing-data)
8. [Query Invalidation](#8-query-invalidation)
9. [Optimistic Updates](#9-optimistic-updates)
10. [useInfiniteQuery — Pagination & Infinite Scroll](#10-useinfinitequery)
11. [Other Hooks](#11-other-hooks)
12. [The QueryClient API](#12-the-queryclient-api)
13. [Next.js Integration (Server Components + Hydration)](#13-nextjs-integration)
14. [Devtools](#14-devtools)
15. [Patterns & Best Practices](#15-patterns--best-practices)
16. [Gotchas & v5 Changes](#16-gotchas--v5-changes)
17. [Quick Reference Card](#17-quick-reference-card)

---

## 1. Why It Exists

Without it, client data fetching looks like:
```tsx
const [data, setData] = useState()
const [loading, setLoading] = useState(true)
const [error, setError] = useState()
useEffect(() => {
  fetch("/api/x").then(r => r.json()).then(setData).catch(setError).finally(() => setLoading(false))
}, [])
```
This has no caching, no dedup, no background refresh, no retry, no shared state across components. TanStack Query gives you **all of that in one hook**:
```tsx
const { data, isPending, error } = useQuery({ queryKey: ["x"], queryFn: fetchX })
```

**It solves:** caching, deduplication, background refetching, stale-while-revalidate, retries, pagination, infinite scroll, optimistic updates, request cancellation, and window-focus refetching.

---

## 2. Installation & Setup

```bash
npm install @tanstack/react-query
npm install -D @tanstack/react-query-devtools
```

Wrap your app once with a provider:
```tsx
// providers.tsx ("use client" in Next.js App Router)
"use client"
import { QueryClient, QueryClientProvider } from "@tanstack/react-query"
import { ReactQueryDevtools } from "@tanstack/react-query-devtools"
import { useState } from "react"

export function Providers({ children }: { children: React.ReactNode }) {
  // useState so the client is stable across re-renders (one per app)
  const [queryClient] = useState(() => new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,        // data fresh for 60s
        refetchOnWindowFocus: false, // common preference
        retry: 1,
      },
    },
  }))
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  )
}
```
```tsx
// app/layout.tsx — wrap children
import { Providers } from "./providers"
export default function RootLayout({ children }) {
  return <html><body><Providers>{children}</Providers></body></html>
}
```

---

## 3. Core Concepts & Vocabulary

| Term | Meaning |
|---|---|
| **Query** | A request for data tied to a unique **key**. Read-only fetch. |
| **Mutation** | A request that **changes** server data (create/update/delete). |
| **Query Key** | A serializable array uniquely identifying a query (the cache address). |
| **Query Function** | An async function that returns the data (or throws). |
| **Stale** | Data older than `staleTime` — will refetch in the background on triggers. |
| **Fresh** | Data within `staleTime` — served from cache, no refetch. |
| **`gcTime`** | Garbage-collection time: how long unused/inactive cache is kept (was `cacheTime` in v4). Default 5 min. |
| **Invalidation** | Marking queries stale so they refetch (usually after a mutation). |
| **`isPending`** | No data yet (first load). (Renamed from `isLoading` in v5.) |
| **`isFetching`** | A request is in flight (including background refetches). |
| **Optimistic update** | Updating the UI *before* the server confirms, for instant feel. |

**Stale-while-revalidate model:** when you mount a query, it shows cached data instantly (if any), then silently refetches in the background if the data is stale, and updates the UI when fresh data arrives. This is why apps feel fast.

---

## 4. useQuery — Reading Data

```tsx
"use client"
import { useQuery } from "@tanstack/react-query"

async function fetchTodos() {
  const res = await fetch("/api/todos")
  if (!res.ok) throw new Error("Failed to fetch")   // MUST throw on error
  return res.json()
}

function Todos() {
  const { data, isPending, isError, error, isFetching, refetch } = useQuery({
    queryKey: ["todos"],
    queryFn: fetchTodos,
  })

  if (isPending) return <Skeleton />
  if (isError) return <p>Error: {error.message}</p>

  return (
    <>
      {isFetching && <span>Refreshing…</span>}
      <ul>{data.map(t => <li key={t.id}>{t.title}</li>)}</ul>
      <button onClick={() => refetch()}>Refresh</button>
    </>
  )
}
```

### Return values you'll use most
| Property | Meaning |
|---|---|
| `data` | The resolved data (undefined until first success). |
| `isPending` | True while there's no cached data yet (initial load). |
| `isError` / `error` | Error state and the thrown error. |
| `isSuccess` | Data loaded successfully. |
| `isFetching` | A fetch is happening (initial OR background). |
| `isRefetching` | A background refetch is happening (not the first load). |
| `refetch()` | Manually trigger a refetch. |
| `status` | `'pending' | 'error' | 'success'`. |
| `fetchStatus` | `'fetching' | 'paused' | 'idle'`. |

### Query with parameters (dependent on input)
```tsx
function useTodo(id: number) {
  return useQuery({
    queryKey: ["todo", id],            // id in the key → separate cache per id
    queryFn: () => fetchTodo(id),
    enabled: !!id,                     // don't run until id exists
  })
}
```

---

## 5. Query Keys

The **query key** is how the cache is addressed. Rules:
- Must be an **array**.
- Must be **serializable** (strings, numbers, objects — no functions).
- **Include every variable** the query depends on. Different keys = different cache entries.

```tsx
["todos"]                              // a list
["todo", 5]                            // a single item
["todos", { status: "done", page: 2 }] // a filtered/paginated list
["projects", projectId, "tasks"]       // nested/hierarchical
```

> **Why it matters:** changing any value in the key triggers a new fetch and a new cache slot. This is how you get automatic refetching when filters/pagination/IDs change — just put them in the key.

### Query key factories (best practice for big apps)
```ts
export const todoKeys = {
  all: ["todos"] as const,
  lists: () => [...todoKeys.all, "list"] as const,
  list: (filters: object) => [...todoKeys.lists(), filters] as const,
  details: () => [...todoKeys.all, "detail"] as const,
  detail: (id: number) => [...todoKeys.details(), id] as const,
}
// Invalidate all todo lists: queryClient.invalidateQueries({ queryKey: todoKeys.lists() })
```

---

## 6. Query Options Reference

The options you'll actually tune:

| Option | Default | What it does |
|---|---|---|
| `queryKey` | — | Unique cache key (required). |
| `queryFn` | — | Async function returning data (required). |
| `enabled` | `true` | If false, query won't run (for dependent queries). |
| `staleTime` | `0` | How long data stays "fresh" (ms). Higher = fewer refetches. |
| `gcTime` | `5 min` | How long inactive cache is kept before garbage collection. |
| `refetchOnWindowFocus` | `true` | Refetch when the tab regains focus. |
| `refetchOnMount` | `true` | Refetch when a component using the query mounts (if stale). |
| `refetchOnReconnect` | `true` | Refetch when network reconnects. |
| `refetchInterval` | `false` | Poll every N ms (e.g. live dashboards). |
| `retry` | `3` | Retry count on failure (or a function). |
| `retryDelay` | exp backoff | Delay between retries. |
| `select` | — | Transform/derive data without re-rendering on unrelated changes. |
| `placeholderData` | — | Temporary data while fetching (e.g. `keepPreviousData`). |
| `initialData` | — | Seed the cache with known data (treated as real). |
| `meta` | — | Arbitrary metadata for logging/error handling. |

### `select` — derive/transform
```tsx
useQuery({
  queryKey: ["todos"],
  queryFn: fetchTodos,
  select: (data) => data.filter(t => !t.done),   // component only re-renders when this output changes
})
```

### `placeholderData` for smooth pagination (keep old page visible)
```tsx
import { keepPreviousData } from "@tanstack/react-query"
useQuery({
  queryKey: ["todos", page],
  queryFn: () => fetchPage(page),
  placeholderData: keepPreviousData,   // shows previous page while next loads (no flash)
})
```

### `staleTime` strategy
- `staleTime: 0` (default) → refetches aggressively (always fresh, more requests).
- `staleTime: 60_000` → good balance for most data.
- `staleTime: Infinity` → never auto-refetches (for static-ish data; refetch manually/on invalidation).

---

## 7. useMutation — Writing Data

For creating/updating/deleting. Unlike queries, mutations don't run automatically — you call `mutate()`.

```tsx
"use client"
import { useMutation, useQueryClient } from "@tanstack/react-query"

function AddTodo() {
  const queryClient = useQueryClient()

  const mutation = useMutation({
    mutationFn: (newTodo: { title: string }) =>
      fetch("/api/todos", { method: "POST", body: JSON.stringify(newTodo) }).then(r => r.json()),

    onSuccess: () => {
      // refresh the list after a successful create
      queryClient.invalidateQueries({ queryKey: ["todos"] })
    },
    onError: (error) => {
      toast.error(error.message)
    },
    onSettled: () => {
      // runs after success OR error (cleanup)
    },
  })

  return (
    <button
      disabled={mutation.isPending}
      onClick={() => mutation.mutate({ title: "New task" })}
    >
      {mutation.isPending ? "Adding…" : "Add Todo"}
    </button>
  )
}
```

### Mutation return values
| Property | Meaning |
|---|---|
| `mutate(vars)` | Fire the mutation (fire-and-forget). |
| `mutateAsync(vars)` | Same but returns a Promise (await it / try-catch). |
| `isPending` | Mutation in flight. |
| `isSuccess` / `isError` | Result state. |
| `data` / `error` | Result/error. |
| `reset()` | Reset mutation state. |
| `variables` | The variables passed to the last `mutate`. |

### Lifecycle callbacks order
`onMutate` → (request) → `onSuccess` **or** `onError` → `onSettled`.

---

## 8. Query Invalidation

The main way to keep the UI in sync after a change: mark queries stale so they refetch.

```tsx
const queryClient = useQueryClient()

// Invalidate one query
queryClient.invalidateQueries({ queryKey: ["todos"] })

// Invalidate ALL queries whose key STARTS WITH ["todos"] (prefix match)
queryClient.invalidateQueries({ queryKey: ["todos"] })   // also matches ["todos", 5], ["todos", {...}]

// Exact match only
queryClient.invalidateQueries({ queryKey: ["todos"], exact: true })

// Invalidate everything
queryClient.invalidateQueries()
```
> **Key insight:** invalidation does prefix matching by default. `["todos"]` invalidates `["todos"]`, `["todos", 5]`, `["todos", {page:2}]` — all of them. This is why hierarchical keys are powerful.

---

## 9. Optimistic Updates

Update the UI **before** the server responds, then roll back if it fails. Makes apps feel instant.

```tsx
const queryClient = useQueryClient()

const mutation = useMutation({
  mutationFn: updateTodo,

  onMutate: async (newTodo) => {
    // 1. Cancel outgoing refetches so they don't overwrite our optimistic update
    await queryClient.cancelQueries({ queryKey: ["todos"] })
    // 2. Snapshot the previous value
    const previous = queryClient.getQueryData(["todos"])
    // 3. Optimistically update the cache
    queryClient.setQueryData(["todos"], (old: any) =>
      old.map((t: any) => (t.id === newTodo.id ? newTodo : t))
    )
    // 4. Return context with the snapshot
    return { previous }
  },

  onError: (err, newTodo, context) => {
    // Roll back on failure
    queryClient.setQueryData(["todos"], context?.previous)
  },

  onSettled: () => {
    // Always refetch to ensure server truth
    queryClient.invalidateQueries({ queryKey: ["todos"] })
  },
})
```

Use optimistic updates for likes, toggles, reordering, quick edits — anything where instant feedback matters and failures are rare.

---

## 10. useInfiniteQuery

For "Load more" buttons and infinite scroll.

```tsx
import { useInfiniteQuery } from "@tanstack/react-query"

function Feed() {
  const {
    data, fetchNextPage, hasNextPage, isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ["posts"],
    queryFn: ({ pageParam }) => fetchPosts(pageParam),
    initialPageParam: 0,
    getNextPageParam: (lastPage, allPages) => lastPage.nextCursor ?? undefined,
  })

  return (
    <>
      {data?.pages.map((page, i) => (
        <Fragment key={i}>
          {page.items.map(p => <Post key={p.id} {...p} />)}
        </Fragment>
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
- `data.pages` → array of fetched pages.
- `getNextPageParam` → returns the next cursor/page, or `undefined` to stop.
- Pair with an `IntersectionObserver` to auto-load on scroll.

---

## 11. Other Hooks

### `useQueries` — dynamic parallel queries
```tsx
const results = useQueries({
  queries: ids.map(id => ({ queryKey: ["todo", id], queryFn: () => fetchTodo(id) })),
})
```

### `useSuspenseQuery` — Suspense-mode (no `isPending` checks)
```tsx
const { data } = useSuspenseQuery({ queryKey: ["todos"], queryFn: fetchTodos })
// data is guaranteed defined; loading handled by a parent <Suspense>, errors by an Error Boundary
```

### `useIsFetching` — global loading indicator
```tsx
const fetching = useIsFetching()   // number of queries currently fetching
return fetching ? <TopBarLoader /> : null
```

### `useMutationState` — observe mutations globally.

---

## 12. The QueryClient API

Imperative cache control (call via `useQueryClient()`):

| Method | Use |
|---|---|
| `invalidateQueries({ queryKey })` | Mark stale → refetch. |
| `setQueryData(key, updater)` | Write directly to the cache (optimistic updates). |
| `getQueryData(key)` | Read current cached value. |
| `prefetchQuery({ queryKey, queryFn })` | Load data before it's needed (hover, route prefetch). |
| `fetchQuery(...)` | Fetch and return a promise (await the data). |
| `ensureQueryData(...)` | Return cached data or fetch if missing. |
| `cancelQueries({ queryKey })` | Cancel in-flight requests (used in optimistic updates). |
| `removeQueries({ queryKey })` | Delete from cache. |
| `resetQueries({ queryKey })` | Reset to initial state. |
| `setQueryDefaults(key, opts)` | Per-key default options. |

### Prefetching on hover (snappy UX)
```tsx
<Link
  href={`/posts/${id}`}
  onMouseEnter={() =>
    queryClient.prefetchQuery({ queryKey: ["post", id], queryFn: () => fetchPost(id) })
  }
>
```

---

## 13. Next.js Integration

Combine **Server Components** (fetch on server) with TanStack Query (client caching/refetching) via **hydration**. Best of both: fast first paint + client interactivity.

```tsx
// app/posts/page.tsx — Server Component prefetches, then hydrates the client
import { dehydrate, HydrationBoundary, QueryClient } from "@tanstack/react-query"
import { Posts } from "./posts"

export default async function PostsPage() {
  const queryClient = new QueryClient()

  // Prefetch on the server
  await queryClient.prefetchQuery({
    queryKey: ["posts"],
    queryFn: getPosts,
  })

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <Posts />   {/* client component using useQuery(["posts"]) — gets data instantly */}
    </HydrationBoundary>
  )
}
```
```tsx
// posts.tsx
"use client"
import { useQuery } from "@tanstack/react-query"
export function Posts() {
  const { data } = useQuery({ queryKey: ["posts"], queryFn: getPosts })
  // data is already there from server prefetch — no loading flash, then keeps it fresh client-side
  return <ul>{data?.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}
```

**When to use TanStack Query in a Next.js app:**
- ✅ Client-side interactive data: infinite scroll, polling dashboards, search-as-you-type, optimistic UI, anything that refetches after user actions.
- ❌ Simple static page data → just fetch in a Server Component (no TanStack Query needed).
- ✅ Mutations from the client where you want cache sync without full page reloads.

> Rule of thumb: **Server Components for the initial read; TanStack Query for ongoing client-side reads/writes.** Don't add it to a site that only needs server rendering.

---

## 14. Devtools

```tsx
import { ReactQueryDevtools } from "@tanstack/react-query-devtools"
<ReactQueryDevtools initialIsOpen={false} />
```
Shows every query, its key, status (fresh/stale/fetching/inactive), cached data, and lets you refetch/invalidate manually. **Indispensable while learning** — you can literally watch the cache work. Excluded from production builds automatically.

---

## 15. Patterns & Best Practices

1. **Wrap query logic in custom hooks.** Don't call `useQuery` directly in components everywhere.
   ```tsx
   export function useTodos() {
     return useQuery({ queryKey: ["todos"], queryFn: fetchTodos })
   }
   ```
2. **Use query key factories** (see §5) for consistency and easy invalidation.
3. **Set a sane global `staleTime`** (e.g. 30–60s) to cut redundant requests.
4. **Throw in `queryFn` on errors** — TanStack only knows a request failed if it throws.
5. **Invalidate after mutations** (or use optimistic updates) to keep UI in sync.
6. **Use `enabled`** for dependent/conditional queries.
7. **Use `select`** to derive data and minimize re-renders.
8. **Use `placeholderData: keepPreviousData`** for paginated UIs.
9. **Co-locate keys + fetchers** per feature (`features/todos/api.ts`).
10. **Don't duplicate server state in `useState`** — read it straight from the query.
11. **One `QueryClient` per app** (created in `useState` so it's stable / not recreated on render).
12. **Prefetch** likely-next data on hover or route change for instant transitions.

---

## 16. Gotchas & v5 Changes

**v5 renames / changes (if you read old v4 tutorials):**
| v4 | v5 |
|---|---|
| `isLoading` | `isPending` (for queries with no data yet) |
| `cacheTime` | `gcTime` |
| `keepPreviousData: true` | `placeholderData: keepPreviousData` |
| Multiple signatures | **Single object signature only** — `useQuery({ queryKey, queryFn })` |
| `useQuery(key, fn)` | ❌ removed — always pass one options object |

**Common mistakes:**
- ❌ `queryFn` that doesn't throw on HTTP errors → query thinks it succeeded. Check `res.ok` and throw.
- ❌ Forgetting variables in the `queryKey` → stale data when inputs change.
- ❌ Recreating `QueryClient` on every render → cache wiped constantly (use `useState`/module singleton).
- ❌ Using it for pure UI state (modals open, form inputs) → that's `useState`, not server state.
- ❌ Over-fetching because `staleTime: 0` and `refetchOnWindowFocus: true` → tune these.
- ❌ Mutating cached data directly instead of via `setQueryData` (must return new references).
- ❌ Forgetting `"use client"` on components using the hooks in Next.js App Router.
- ❌ Not awaiting `cancelQueries` in `onMutate` → optimistic update gets clobbered by an in-flight refetch.

---

## 17. Quick Reference Card

```tsx
// READ
const { data, isPending, isError, error, refetch } = useQuery({
  queryKey: ["resource", id],
  queryFn: () => fetchResource(id),
  enabled: !!id,
  staleTime: 60_000,
})

// WRITE
const { mutate, isPending } = useMutation({
  mutationFn: createResource,
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ["resource"] }),
})
mutate(payload)

// PAGINATE / INFINITE
const { data, fetchNextPage, hasNextPage } = useInfiniteQuery({
  queryKey: ["feed"],
  queryFn: ({ pageParam }) => fetchPage(pageParam),
  initialPageParam: 0,
  getNextPageParam: (last) => last.nextCursor,
})

// CACHE CONTROL
const queryClient = useQueryClient()
queryClient.invalidateQueries({ queryKey: ["resource"] })   // refetch
queryClient.setQueryData(["resource", id], newData)         // write cache
queryClient.getQueryData(["resource", id])                  // read cache
queryClient.prefetchQuery({ queryKey, queryFn })            // preload
```

### Decision guide
- **Reading server data on the client?** → `useQuery`.
- **Changing server data?** → `useMutation` + invalidate (or optimistic update).
- **Pages / infinite scroll?** → `useInfiniteQuery`.
- **Many parallel queries?** → `useQueries`.
- **Want instant UX?** → optimistic updates + prefetch on hover.
- **First-load data in Next.js?** → prefetch in a Server Component + `HydrationBoundary`.

---

### Study path (offline mastery order)
1. Setup + `useQuery` basics (§2, §4).
2. Query keys — internalize how the cache is addressed (§5).
3. Options: `staleTime`, `enabled`, `select`, `placeholderData` (§6).
4. Mutations + invalidation (§7, §8).
5. Optimistic updates (§9).
6. Infinite queries (§10).
7. Next.js hydration (§13).
8. Build: a todo app (CRUD + invalidation), a paginated table (`placeholderData`), an infinite feed (`useInfiniteQuery`), and an optimistic "like" button. That covers the real-world 90%.

> Keep the **Devtools open** while building — watching queries go fresh → stale → fetching is the fastest way to truly understand the library.
