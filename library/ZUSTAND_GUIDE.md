# Zustand Complete Guide

A full, study-friendly reference for **Zustand v5** — the minimal, fast, and scalable state management library for React. Covers setup, store creation, selectors, middleware, slices, SSR, testing, and production patterns.

> **What it is:** Zustand manages **client/UI state** — data that lives in the browser (selected tab, modal visibility, user preferences, shopping cart, authenticated user). It is *not* designed for server state (use TanStack Query for that). It works without providers, without boilerplate, and without reducers — just a plain function that returns state and actions.

---

## Table of Contents

1. [What Zustand Is & When to Use It](#1-what-zustand-is--when-to-use-it)
2. [Installation](#2-installation)
3. [Creating a Store](#3-creating-a-store)
4. [Using the Store in Components & Selectors](#4-using-the-store-in-components--selectors)
5. [Updating State: set, get, replace vs merge](#5-updating-state-set-get-replace-vs-merge)
6. [Async Actions](#6-async-actions)
7. [useShallow — Selecting Multiple Values](#7-useshallow--selecting-multiple-values)
8. [Middleware](#8-middleware)
9. [The Slices Pattern](#9-the-slices-pattern)
10. [Derived State & store.subscribe](#10-derived-state--storesubscribe)
11. [Accessing the Store Outside React](#11-accessing-the-store-outside-react)
12. [Zustand with Next.js App Router](#12-zustand-with-nextjs-app-router)
13. [Testing Stores](#13-testing-stores)
14. [Zustand vs Redux Toolkit vs Jotai vs Context](#14-zustand-vs-redux-toolkit-vs-jotai-vs-context)
15. [Tips, Tricks & Gotchas](#15-tips-tricks--gotchas)
16. [Study Path](#16-study-path)

---

## 1. What Zustand Is & When to Use It

### The problem without Zustand

Passing state between distant components forces you into **prop drilling** or a Context that re-renders every consumer on every change:

```tsx
// Every component that consumes CartContext re-renders when ANY cart value changes
const CartContext = createContext(null)

function App() {
  const [cart, setCart] = useState([])
  return (
    <CartContext.Provider value={{ cart, setCart }}>
      {/* deeply nested tree */}
    </CartContext.Provider>
  )
}
```

Redux fixes the re-render problem but demands actions, reducers, selectors, a store file, and wiring — a lot of ceremony for simple cases.

### What Zustand gives you

- **No provider** — the store is a module-level singleton
- **No reducers** — actions are plain functions that call `set`
- **Subscription-based** — components only re-render when the slice they subscribed to changes
- **Tiny** — ~1 KB gzipped
- **Works outside React** — `getState()` / `setState()` / `subscribe()` without hooks

```tsx
// The entire counter store — no provider, no reducer, no action types
import { create } from "zustand"

const useCounterStore = create((set) => ({
  count: 0,
  increment: () => set((s) => ({ count: s.count + 1 })),
  decrement: () => set((s) => ({ count: s.count - 1 })),
  reset: () => set({ count: 0 }),
}))

function Counter() {
  const count = useCounterStore((s) => s.count)
  const increment = useCounterStore((s) => s.increment)
  return <button onClick={increment}>{count}</button>
}
```

### Choosing the right tool

| Scenario | Tool |
|---|---|
| Server data: loading, caching, background sync | **TanStack Query** |
| Global client/UI state shared across many components | **Zustand** |
| Simple local state in one component | **useState / useReducer** |
| Prop-drilling fix between a parent and a few children | **Context** (fine here) |
| Complex server state AND client state in one app | **TanStack Query + Zustand** |

### Why not Redux for this?

Redux Toolkit is excellent but requires: a `configureStore` call, `createSlice` with `reducers` and `actions`, `<Provider>`, `useSelector`, `useDispatch`, and dispatching action objects. Zustand skips all of that. For small-to-medium client state, Zustand wins on ergonomics.

---

## 2. Installation

```bash
npm install zustand
```

⚡ **Version note:** This guide covers **Zustand v5** (released late 2024). The v5 `create` API is unchanged from v4, but several internals and TypeScript patterns were refined. The `useShallow` import path moved to `zustand/react/shallow` in v5. Examples below use v5 imports throughout.

No peer dependencies. No context provider needed. Works with React 18+ and React 19.

---

## 3. Creating a Store

### The `create` function

`create` takes a function that receives `set` and `get` and returns the initial state **plus** all actions in one flat object.

```ts
// store/counterStore.ts
import { create } from "zustand"

// 1. Define the shape with a TypeScript interface
interface CounterState {
  count: number
  step: number
  // Actions — functions live alongside state
  increment: () => void
  decrement: () => void
  incrementByStep: () => void
  reset: () => void
  setStep: (step: number) => void
}

// 2. Pass the interface as the generic — Zustand infers the rest
export const useCounterStore = create<CounterState>()((set, get) => ({
  // --- Initial state ---
  count: 0,
  step: 1,

  // --- Actions ---
  // Functional update: always safe when new value depends on old value
  increment: () => set((s) => ({ count: s.count + 1 })),
  decrement: () => set((s) => ({ count: s.count - 1 })),

  // Use `get` to read current state inside an action
  incrementByStep: () => set((s) => ({ count: s.count + get().step })),

  // Partial object: only the keys you provide are merged
  reset: () => set({ count: 0 }),

  setStep: (step) => set({ step }),
}))
```

> **Note the double-call `create<State>()(fn)`:** In v4/v5 TypeScript usage, `create` is curried when a generic is passed. This lets TypeScript infer the function argument fully. Without the generic (`create(fn)`), TS still works but you lose type safety on `set` and `get`.

### File organisation conventions

```
src/
  store/
    counterStore.ts      # one store per domain
    cartStore.ts
    authStore.ts
    index.ts             # optional barrel re-export
```

Keep stores **domain-scoped** — not one massive god-store.

---

## 4. Using the Store in Components & Selectors

### Basic usage

```tsx
// components/CounterDisplay.tsx
import { useCounterStore } from "../store/counterStore"

export function CounterDisplay() {
  // Selector: subscribe only to `count`
  // This component re-renders ONLY when `count` changes
  const count = useCounterStore((s) => s.count)

  return <p>Count: {count}</p>
}
```

```tsx
// components/CounterControls.tsx
import { useCounterStore } from "../store/counterStore"

export function CounterControls() {
  // Selecting actions is safe — actions never change reference, so no re-render
  const increment = useCounterStore((s) => s.increment)
  const decrement = useCounterStore((s) => s.decrement)
  const reset = useCounterStore((s) => s.reset)

  return (
    <div>
      <button onClick={decrement}>-</button>
      <button onClick={increment}>+</button>
      <button onClick={reset}>Reset</button>
    </div>
  )
}
```

### Why selectors matter for performance

```tsx
// BAD — subscribes to the ENTIRE store object
// Re-renders on ANY state change, even unrelated ones
function BadComponent() {
  const store = useCounterStore() // no selector
  return <p>{store.count}</p>
}

// GOOD — subscribes only to `count`
// Re-renders only when count changes
function GoodComponent() {
  const count = useCounterStore((s) => s.count)
  return <p>{count}</p>
}
```

When you omit the selector, you get the **entire store object**. Zustand compares the return value of your selector with `Object.is` after every `set` call. If the selector returns the whole store, any state change (even to an unrelated key) produces a new reference, causing a re-render.

### Computed/derived values inside a selector

```tsx
// Inline derived value — computed on each render, re-renders when `cart` changes
const totalItems = useCartStore((s) =>
  s.cart.reduce((acc, item) => acc + item.quantity, 0)
)
```

---

## 5. Updating State: set, get, replace vs merge

### `set` — the only way to update state

`set` is provided by Zustand to every action. It **merges** by default (shallow merge at the top level, like `setState` in React class components).

```ts
import { create } from "zustand"

interface UserState {
  name: string
  age: number
  address: { city: string; country: string }
  setName: (name: string) => void
  setAddress: (city: string) => void
  birthday: () => void
}

const useUserStore = create<UserState>()((set, get) => ({
  name: "Alice",
  age: 30,
  address: { city: "London", country: "UK" },

  // Merge — only `name` changes, `age` and `address` are untouched
  setName: (name) => set({ name }),

  // Functional update — use when new value depends on old value
  birthday: () => set((s) => ({ age: s.age + 1 })),

  // Nested object: you must spread manually — Zustand does NOT deep merge
  setAddress: (city) =>
    set((s) => ({
      address: { ...s.address, city }, // spread the old address first
    })),
}))
```

### `get` — reading state inside actions

```ts
const useStore = create<{ count: number; doubled: () => number }>()((set, get) => ({
  count: 5,
  // Read state synchronously at call time without subscribing
  doubled: () => get().count * 2,
}))
```

### replace vs merge

```ts
// Default: MERGE (top-level shallow merge)
set({ name: "Bob" })
// Result: { name: "Bob", age: 30, address: {...} }  ← age and address survive

// REPLACE: pass true as second argument — wipes everything not in the new object
set({ name: "Bob" }, true)
// Result: { name: "Bob" }  ← age and address are GONE

// Practical use of replace: resetting the entire store to defaults
const INITIAL = { count: 0, step: 1 }
const useStore = create<typeof INITIAL & { reset: () => void }>()((set) => ({
  ...INITIAL,
  reset: () => set(INITIAL, true), // replace with exact initial shape
}))
```

⚡ **Version note (v5):** In v5, the second argument to `set` changed from a boolean to `{ replace: boolean }` in some internal typing, but the boolean shorthand still works. Prefer the boolean form for brevity in practice.

---

## 6. Async Actions

Actions in Zustand are just functions. Async is trivial — no middleware, no thunks:

```ts
// store/postsStore.ts
import { create } from "zustand"

interface Post {
  id: number
  title: string
  body: string
}

interface PostsState {
  posts: Post[]
  isLoading: boolean
  error: string | null
  fetchPosts: () => Promise<void>
  addPost: (post: Omit<Post, "id">) => Promise<Post>
}

export const usePostsStore = create<PostsState>()((set, get) => ({
  posts: [],
  isLoading: false,
  error: null,

  fetchPosts: async () => {
    // Guard against concurrent fetches
    if (get().isLoading) return

    set({ isLoading: true, error: null })
    try {
      const res = await fetch("https://jsonplaceholder.typicode.com/posts?_limit=10")
      if (!res.ok) throw new Error(`HTTP ${res.status}`)
      const posts: Post[] = await res.json()
      set({ posts, isLoading: false })
    } catch (err) {
      set({ error: (err as Error).message, isLoading: false })
    }
  },

  addPost: async (newPost) => {
    const res = await fetch("https://jsonplaceholder.typicode.com/posts", {
      method: "POST",
      body: JSON.stringify(newPost),
      headers: { "Content-Type": "application/json" },
    })
    const created: Post = await res.json()
    // Optimistic-style: append to local list immediately after server confirms
    set((s) => ({ posts: [created, ...s.posts] }))
    return created
  },
}))
```

```tsx
// components/PostsList.tsx
import { useEffect } from "react"
import { usePostsStore } from "../store/postsStore"

export function PostsList() {
  const { posts, isLoading, error, fetchPosts } = usePostsStore((s) => ({
    posts: s.posts,
    isLoading: s.isLoading,
    error: s.error,
    fetchPosts: s.fetchPosts,
  }))

  useEffect(() => { fetchPosts() }, [fetchPosts])

  if (isLoading) return <p>Loading...</p>
  if (error) return <p>Error: {error}</p>
  return <ul>{posts.map((p) => <li key={p.id}>{p.title}</li>)}</ul>
}
```

> **Tip:** For server data that needs caching, background refetch, and dedup, use **TanStack Query** instead of rolling your own in Zustand. Zustand async actions are best for fire-and-forget mutations or one-time loads that you control.

---

## 7. useShallow — Selecting Multiple Values

When you need multiple values at once, you might be tempted to return an object from a selector:

```tsx
// BAD — returns a new object every render, always triggers re-render
const { count, step } = useCounterStore((s) => ({ count: s.count, step: s.step }))
```

Zustand compares selector return values with `Object.is`. A new object `{}` is never `===` the previous one, so every store update causes a re-render regardless of whether `count` or `step` actually changed.

**Fix: use `useShallow`**, which does a shallow comparison of object/array values:

```tsx
// GOOD — uses shallow equality, re-renders only when count or step changes
import { useShallow } from "zustand/react/shallow" // v5 import path

const { count, step } = useCounterStore(
  useShallow((s) => ({ count: s.count, step: s.step }))
)
```

⚡ **Version note:** In Zustand v4, `shallow` was imported from `"zustand/shallow"` and used as `useStore(selector, shallow)`. In **v5**, `useShallow` is the recommended hook and is imported from `"zustand/react/shallow"`. Both patterns work in v5 but `useShallow` is idiomatic.

### Array selector with useShallow

```tsx
import { useShallow } from "zustand/react/shallow"

// Select as an array instead of an object
const [count, step, increment] = useCounterStore(
  useShallow((s) => [s.count, s.step, s.increment])
)
```

### When you DON'T need useShallow

- Selecting a **single primitive** (`string`, `number`, `boolean`) — `Object.is` comparison works perfectly
- Selecting **actions** — action functions are created once and never change reference
- Selecting a **single object/array** that you're comparing by reference intentionally

---

## 8. Middleware

Middleware wraps `create` to add behaviour. Stack them left-to-right with nested calls.

### 8.1 `devtools` — Redux DevTools integration

```ts
import { create } from "zustand"
import { devtools } from "zustand/middleware"

interface CounterState {
  count: number
  increment: () => void
}

export const useCounterStore = create<CounterState>()(
  devtools(
    (set) => ({
      count: 0,
      increment: () => set((s) => ({ count: s.count + 1 }), false, "counter/increment"),
      //                                                      ^^^^^  ^^^^^^^^^^^^^^^^^^
      //                                                      replace  action name in devtools
    }),
    { name: "CounterStore" } // store name shown in DevTools
  )
)
```

Install the [Redux DevTools browser extension](https://github.com/reduxjs/redux-devtools) and you get time-travel debugging, action history, and state diff for free.

### 8.2 `persist` — localStorage / sessionStorage / custom

```ts
import { create } from "zustand"
import { persist, createJSONStorage } from "zustand/middleware"

interface ThemeState {
  theme: "light" | "dark"
  fontSize: number
  toggleTheme: () => void
  setFontSize: (size: number) => void
}

export const useThemeStore = create<ThemeState>()(
  persist(
    (set) => ({
      theme: "light",
      fontSize: 16,
      toggleTheme: () =>
        set((s) => ({ theme: s.theme === "light" ? "dark" : "light" })),
      setFontSize: (fontSize) => set({ fontSize }),
    }),
    {
      name: "theme-storage",    // localStorage key
      storage: createJSONStorage(() => localStorage), // default; swap for sessionStorage
      // partialize: only persist `theme`, skip `fontSize`
      partialize: (state) => ({ theme: state.theme }),
    }
  )
)
```

#### Persist with migrations

```ts
export const useSettingsStore = create<SettingsState>()(
  persist(
    (set) => ({ /* ... */ }),
    {
      name: "settings-v2",
      version: 2, // increment this when shape changes
      migrate: (persistedState: unknown, version: number) => {
        if (version === 0) {
          // v0 → v1: rename `color` to `theme`
          const old = persistedState as { color: string }
          return { theme: old.color }
        }
        if (version === 1) {
          // v1 → v2: add `fontSize` default
          const old = persistedState as { theme: string }
          return { ...old, fontSize: 16 }
        }
        return persistedState
      },
    }
  )
)
```

#### Custom storage (e.g. AsyncStorage in React Native, cookies)

```ts
import { StateStorage } from "zustand/middleware"

const cookieStorage: StateStorage = {
  getItem: (name) => {
    const match = document.cookie.match(new RegExp(`(^| )${name}=([^;]+)`))
    return match ? decodeURIComponent(match[2]) : null
  },
  setItem: (name, value) => {
    document.cookie = `${name}=${encodeURIComponent(value)}; path=/; max-age=31536000`
  },
  removeItem: (name) => {
    document.cookie = `${name}=; max-age=0`
  },
}

const useStore = create()(
  persist((set) => ({ /* ... */ }), {
    name: "my-cookie-store",
    storage: createJSONStorage(() => cookieStorage),
  })
)
```

### 8.3 `immer` — mutable-style updates

Without Immer, nested updates require manual spreading. With Immer, you write mutations and it produces a new immutable object:

```bash
npm install immer
```

```ts
import { create } from "zustand"
import { immer } from "zustand/middleware/immer"

interface TodoState {
  todos: Array<{ id: number; text: string; done: boolean }>
  addTodo: (text: string) => void
  toggleTodo: (id: number) => void
  removeTodo: (id: number) => void
}

export const useTodoStore = create<TodoState>()(
  immer((set) => ({
    todos: [],

    addTodo: (text) =>
      set((s) => {
        // Direct mutation — Immer handles immutability
        s.todos.push({ id: Date.now(), text, done: false })
      }),

    toggleTodo: (id) =>
      set((s) => {
        const todo = s.todos.find((t) => t.id === id)
        if (todo) todo.done = !todo.done // mutate safely
      }),

    removeTodo: (id) =>
      set((s) => {
        s.todos = s.todos.filter((t) => t.id !== id)
      }),
  }))
)
```

### 8.4 `subscribeWithSelector` — watching specific state slices outside React

```ts
import { create } from "zustand"
import { subscribeWithSelector } from "zustand/middleware"

const useStore = create<{ count: number; name: string }>()(
  subscribeWithSelector((set) => ({
    count: 0,
    name: "Alice",
    increment: () => set((s) => ({ count: s.count + 1 })),
  }))
)

// Subscribe to a specific slice — callback fires only when `count` changes
const unsub = useStore.subscribe(
  (s) => s.count,           // selector
  (count, prevCount) => {   // listener
    console.log(`Count changed: ${prevCount} → ${count}`)
    if (count >= 10) {
      console.log("Reached 10!")
    }
  },
  { equalityFn: Object.is, fireImmediately: false } // options
)

// Stop listening
unsub()
```

### 8.5 `combine` — infer state type automatically

```ts
import { create } from "zustand"
import { combine } from "zustand/middleware"

// combine(initialState, actionsCreator) infers types from initialState
export const useStore = create(
  combine(
    { bears: 0, bees: 0 },  // initial state — type inferred automatically
    (set, get) => ({
      addBear: () => set((s) => ({ bears: s.bears + 1 })),
      addBee: () => set((s) => ({ bees: s.bees + 1 })),
      total: () => get().bears + get().bees,
    })
  )
)
```

### Stacking multiple middleware

```ts
import { create } from "zustand"
import { devtools, persist, immer } from "zustand/middleware"

// Order: outermost middleware wraps first
// Convention: devtools outermost, persist next, immer innermost
export const useCartStore = create<CartState>()(
  devtools(
    persist(
      immer((set, get) => ({
        items: [],
        addItem: (item) => set((s) => { s.items.push(item) }),
        clearCart: () => set((s) => { s.items = [] }),
      })),
      { name: "cart-storage" }
    ),
    { name: "CartStore" }
  )
)
```

---

## 9. The Slices Pattern

When your app grows, one store file becomes unwieldy. Split state into **slices** — separate files that each define a portion of the store — then combine them.

### Step 1: Define individual slice creators

```ts
// store/slices/counterSlice.ts
import { StateCreator } from "zustand"
import type { BoundStoreState } from "../useBoundStore"

export interface CounterSlice {
  count: number
  increment: () => void
  decrement: () => void
  resetCount: () => void
}

// StateCreator takes 3 generics:
//   [1] full store type (for cross-slice access via get())
//   [2] mutators list (leave empty unless using middleware)
//   [3] this slice's type
export const createCounterSlice: StateCreator<
  BoundStoreState,
  [],
  [],
  CounterSlice
> = (set) => ({
  count: 0,
  increment: () => set((s) => ({ count: s.count + 1 })),
  decrement: () => set((s) => ({ count: s.count - 1 })),
  resetCount: () => set({ count: 0 }),
})
```

```ts
// store/slices/userSlice.ts
import { StateCreator } from "zustand"
import type { BoundStoreState } from "../useBoundStore"

export interface UserSlice {
  user: { name: string; email: string } | null
  setUser: (user: UserSlice["user"]) => void
  logout: () => void
}

export const createUserSlice: StateCreator<
  BoundStoreState,
  [],
  [],
  UserSlice
> = (set) => ({
  user: null,
  setUser: (user) => set({ user }),
  logout: () => set({ user: null }),
})
```

### Step 2: Combine into a single bound store

```ts
// store/useBoundStore.ts
import { create } from "zustand"
import { CounterSlice, createCounterSlice } from "./slices/counterSlice"
import { UserSlice, createUserSlice } from "./slices/userSlice"

// Combined type — the full shape of the store
export type BoundStoreState = CounterSlice & UserSlice

export const useBoundStore = create<BoundStoreState>()((...args) => ({
  ...createCounterSlice(...args),
  ...createUserSlice(...args),
}))
```

### Step 3: Use as normal

```tsx
// Both slices are available on the same hook
const count = useBoundStore((s) => s.count)
const user = useBoundStore((s) => s.user)
const increment = useBoundStore((s) => s.increment)
const logout = useBoundStore((s) => s.logout)
```

### Cross-slice actions (a slice calling another slice's state)

```ts
// store/slices/counterSlice.ts — reset count when user logs out
export const createCounterSlice: StateCreator<BoundStoreState, [], [], CounterSlice> =
  (set, get) => ({
    count: 0,
    increment: () => set((s) => ({ count: s.count + 1 })),
    decrement: () => set((s) => ({ count: s.count - 1 })),
    resetCount: () => set({ count: 0 }),
    // Cross-slice: check user state from another slice
    incrementIfLoggedIn: () => {
      if (get().user !== null) {
        set((s) => ({ count: s.count + 1 }))
      }
    },
  })
```

### Slices with middleware

```ts
// Wrap the combined creator — middleware applies to the full store
export const useBoundStore = create<BoundStoreState>()(
  devtools(
    persist(
      (...args) => ({
        ...createCounterSlice(...args),
        ...createUserSlice(...args),
      }),
      { name: "app-storage", partialize: (s) => ({ user: s.user }) }
    ),
    { name: "AppStore" }
  )
)
```

---

## 10. Derived State & store.subscribe

### Derived/computed values

Zustand has no built-in "computed" field. Common patterns:

**Pattern A: derive inline in the selector (simplest)**

```tsx
// Re-computed on every render, but re-render only happens when `cart` changes
const totalPrice = useCartStore((s) =>
  s.cart.reduce((sum, item) => sum + item.price * item.quantity, 0)
)
```

**Pattern B: memoize with useMemo for expensive computations**

```tsx
import { useMemo } from "react"

function OrderSummary() {
  const cart = useCartStore((s) => s.cart)

  // useMemo caches the result — recalculates only when `cart` changes
  const { total, itemCount } = useMemo(() => ({
    total: cart.reduce((s, i) => s + i.price * i.quantity, 0),
    itemCount: cart.reduce((s, i) => s + i.quantity, 0),
  }), [cart])

  return <div>Items: {itemCount}, Total: ${total.toFixed(2)}</div>
}
```

**Pattern C: store derived values in state and update them with subscribeWithSelector**

```ts
// Maintain a `total` field that stays in sync automatically
const useCartStore = create<CartState>()(
  subscribeWithSelector((set) => ({
    cart: [],
    total: 0,
    addItem: (item) =>
      set((s) => {
        const cart = [...s.cart, item]
        return { cart, total: cart.reduce((a, i) => a + i.price * i.quantity, 0) }
      }),
  }))
)
```

### `store.subscribe` for non-React logic

Every Zustand store exposes a `subscribe` method directly on the hook (as a static property). Use it for analytics, logging, syncing to external systems — anything outside the React tree:

```ts
// Log every state change to analytics
const unsub = useCartStore.subscribe((state, prevState) => {
  if (state.cart.length !== prevState.cart.length) {
    analytics.track("cart_updated", { itemCount: state.cart.length })
  }
})

// Subscribe to a specific slice (requires subscribeWithSelector middleware)
const unsubTotal = useCartStore.subscribe(
  (s) => s.total,
  (total) => {
    document.title = `Cart: $${total.toFixed(2)}`
  }
)

// Always clean up subscriptions
window.addEventListener("beforeunload", () => {
  unsub()
  unsubTotal()
})
```

---

## 11. Accessing the Store Outside React

Zustand stores are module-level singletons. The hook (`useCounterStore`) has static methods for vanilla access:

```ts
// store/counterStore.ts
export const useCounterStore = create<CounterState>()((set) => ({
  count: 0,
  increment: () => set((s) => ({ count: s.count + 1 })),
}))

// --- Anywhere outside React (utility functions, event handlers, WebSocket handlers) ---

// Read current state
const count = useCounterStore.getState().count

// Write state
useCounterStore.setState({ count: 99 })
useCounterStore.setState((s) => ({ count: s.count + 10 }))

// Call an action
useCounterStore.getState().increment()

// Subscribe to changes
const unsub = useCounterStore.subscribe((state) => {
  console.log("Count is now:", state.count)
})
unsub() // stop subscribing
```

### Vanilla store (no React dependency at all)

```ts
import { createStore } from "zustand/vanilla"

// createStore works identically to create but returns a store object, not a hook
const counterStore = createStore<CounterState>()((set) => ({
  count: 0,
  increment: () => set((s) => ({ count: s.count + 1 })),
}))

// Use in vanilla JS
counterStore.getState().increment()
console.log(counterStore.getState().count) // 1

// Later, wire it to React if needed
import { useStore } from "zustand"
function Counter() {
  const count = useStore(counterStore, (s) => s.count)
  return <p>{count}</p>
}
```

---

## 12. Zustand with Next.js App Router

### The problem

Zustand stores are **module singletons** — the same object across all imports. In a Node.js SSR environment (Next.js App Router server components), multiple concurrent requests share the same Node.js process and therefore the same module instance. If one user's request mutates the store, the next user might see that stale/wrong state.

### The solution: per-request store factory

Create a **new store instance per request** using `createStore` + React Context to pass it down:

```ts
// store/counterStore.ts
import { createStore } from "zustand/vanilla"

export interface CounterState {
  count: number
  increment: () => void
}

// Factory function — call this once per request/page to get a fresh store
export const createCounterStore = (initialCount = 0) =>
  createStore<CounterState>()((set) => ({
    count: initialCount,
    increment: () => set((s) => ({ count: s.count + 1 })),
  }))

export type CounterStore = ReturnType<typeof createCounterStore>
```

```tsx
// store/CounterStoreProvider.tsx — must be a Client Component
"use client"
import { createContext, useContext, useRef } from "react"
import { useStore } from "zustand"
import { createCounterStore, CounterStore, CounterState } from "./counterStore"

// Context holds the store instance (not state)
const CounterStoreContext = createContext<CounterStore | null>(null)

interface ProviderProps {
  children: React.ReactNode
  initialCount?: number // pass server-fetched data in as initial state
}

export function CounterStoreProvider({ children, initialCount = 0 }: ProviderProps) {
  // useRef ensures the store is created once per component mount (not every render)
  const storeRef = useRef<CounterStore | null>(null)
  if (storeRef.current === null) {
    storeRef.current = createCounterStore(initialCount)
  }

  return (
    <CounterStoreContext.Provider value={storeRef.current}>
      {children}
    </CounterStoreContext.Provider>
  )
}

// Custom hook — use this instead of the raw hook in components
export function useCounterStore<T>(selector: (state: CounterState) => T): T {
  const store = useContext(CounterStoreContext)
  if (!store) throw new Error("useCounterStore must be used inside CounterStoreProvider")
  return useStore(store, selector)
}
```

```tsx
// app/layout.tsx — Server Component: fetch initial data, pass to provider
import { CounterStoreProvider } from "../store/CounterStoreProvider"

export default async function Layout({ children }: { children: React.ReactNode }) {
  // Could fetch initial count from DB here
  const initialCount = 0

  return (
    <html>
      <body>
        <CounterStoreProvider initialCount={initialCount}>
          {children}
        </CounterStoreProvider>
      </body>
    </html>
  )
}
```

```tsx
// app/page.tsx (Server Component) — fine, renders CounterStoreProvider
// components/Counter.tsx (Client Component)
"use client"
import { useCounterStore } from "../store/CounterStoreProvider"

export function Counter() {
  const count = useCounterStore((s) => s.count)
  const increment = useCounterStore((s) => s.increment)
  return <button onClick={increment}>Count: {count}</button>
}
```

### When you DON'T need the factory pattern

If your Zustand store is **purely client-side** (theme, cart, UI state) and you're using it only in `"use client"` components, the singleton pattern is fine. The factory is only needed when:

- The store might be read/written on the server
- You need different initial state per request/user
- You're using Zustand in Server Components (you can't — they're server-only)

### Persist + Next.js: hydration mismatch warning

```tsx
// The persisted store reads localStorage on the client, but SSR renders with defaults.
// Use the `skipHydration` option to avoid mismatches:
"use client"
import { useEffect } from "react"
import { useThemeStore } from "../store/themeStore"

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  useEffect(() => {
    // Manually rehydrate after mount to avoid SSR mismatch
    useThemeStore.persist.rehydrate()
  }, [])
  return <>{children}</>
}

// In the store, set skipHydration: true
export const useThemeStore = create<ThemeState>()(
  persist(
    (set) => ({ theme: "light", toggleTheme: () => set((s) => ({ theme: s.theme === "light" ? "dark" : "light" })) }),
    { name: "theme", skipHydration: true }
  )
)
```

---

## 13. Testing Stores

### Testing the store directly (unit tests)

```ts
// store/__tests__/counterStore.test.ts
import { describe, it, expect, beforeEach } from "vitest"
// Import the store factory, NOT the singleton hook, to get a fresh store per test
import { createStore } from "zustand/vanilla"
import type { CounterState } from "../counterStore"

// Recreate the store logic inline or extract the creator for testing
const createCounterStore = () =>
  createStore<CounterState>()((set) => ({
    count: 0,
    step: 1,
    increment: () => set((s) => ({ count: s.count + 1 })),
    decrement: () => set((s) => ({ count: s.count - 1 })),
    reset: () => set({ count: 0, step: 1 }),
    setStep: (step) => set({ step }),
  }))

describe("counterStore", () => {
  let store: ReturnType<typeof createCounterStore>

  beforeEach(() => {
    // Fresh store for each test — never share store state between tests
    store = createCounterStore()
  })

  it("starts at 0", () => {
    expect(store.getState().count).toBe(0)
  })

  it("increments", () => {
    store.getState().increment()
    expect(store.getState().count).toBe(1)
  })

  it("resets", () => {
    store.getState().increment()
    store.getState().increment()
    store.getState().reset()
    expect(store.getState().count).toBe(0)
  })
})
```

### Testing async actions

```ts
// store/__tests__/postsStore.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest"
import { createStore } from "zustand/vanilla"
import type { PostsState } from "../postsStore"

// Mock fetch
global.fetch = vi.fn()

const createPostsStore = () =>
  createStore<PostsState>()((set, get) => ({
    posts: [],
    isLoading: false,
    error: null,
    fetchPosts: async () => {
      if (get().isLoading) return
      set({ isLoading: true, error: null })
      try {
        const res = await fetch("/api/posts")
        const posts = await res.json()
        set({ posts, isLoading: false })
      } catch (err) {
        set({ error: (err as Error).message, isLoading: false })
      }
    },
  }))

describe("postsStore async", () => {
  let store: ReturnType<typeof createPostsStore>

  beforeEach(() => {
    store = createPostsStore()
    vi.clearAllMocks()
  })

  it("fetches posts successfully", async () => {
    const mockPosts = [{ id: 1, title: "Hello" }]
    ;(fetch as ReturnType<typeof vi.fn>).mockResolvedValueOnce({
      ok: true,
      json: async () => mockPosts,
    })

    await store.getState().fetchPosts()

    expect(store.getState().posts).toEqual(mockPosts)
    expect(store.getState().isLoading).toBe(false)
    expect(store.getState().error).toBeNull()
  })

  it("handles fetch error", async () => {
    ;(fetch as ReturnType<typeof vi.fn>).mockRejectedValueOnce(new Error("Network error"))

    await store.getState().fetchPosts()

    expect(store.getState().error).toBe("Network error")
    expect(store.getState().isLoading).toBe(false)
  })
})
```

### Testing components that use a store

```tsx
// components/__tests__/Counter.test.tsx
import { render, screen, fireEvent } from "@testing-library/react"
import { describe, it, expect, beforeEach } from "vitest"
import { Counter } from "../Counter"
import { useCounterStore } from "../../store/counterStore"

describe("Counter component", () => {
  beforeEach(() => {
    // Reset the store singleton before each test
    useCounterStore.setState({ count: 0 })
  })

  it("displays the count", () => {
    render(<Counter />)
    expect(screen.getByText(/Count: 0/i)).toBeInTheDocument()
  })

  it("increments on click", () => {
    render(<Counter />)
    fireEvent.click(screen.getByRole("button"))
    expect(screen.getByText(/Count: 1/i)).toBeInTheDocument()
  })

  it("starts at a preset value", () => {
    // Seed the store before rendering
    useCounterStore.setState({ count: 42 })
    render(<Counter />)
    expect(screen.getByText(/Count: 42/i)).toBeInTheDocument()
  })
})
```

> **Key pattern:** For singleton stores, use `useStore.setState(...)` in `beforeEach` to reset to a known state. For more isolation, extract store logic into a creator function and call it with `createStore` per test.

---

## 14. Zustand vs Redux Toolkit vs Jotai vs Context

| Dimension | **Zustand** | **Redux Toolkit** | **Jotai** | **React Context** |
|---|---|---|---|---|
| **Bundle size** | ~1 KB | ~12 KB | ~3 KB | 0 (built-in) |
| **Boilerplate** | Minimal | Moderate (slices, actions) | Minimal | Low |
| **Provider required** | No | Yes (`<Provider>`) | No (but `<Provider>` available) | Yes |
| **Re-render model** | Subscription (selector) | Subscription (selector) | Atom subscription | All consumers on any change |
| **DevTools** | Redux DevTools (via middleware) | Redux DevTools (built-in) | Jotai DevTools | React DevTools |
| **Async** | Plain async functions | `createAsyncThunk` | Async atoms | Manual |
| **TypeScript** | Excellent | Excellent | Excellent | Good |
| **Best for** | Global client state, shared UI state | Large teams, strict patterns, existing Redux | Atomic/fine-grained state | Passing config/theme to a subtree |
| **Learning curve** | Low | Medium | Low-Medium | Low |
| **Persistence** | `persist` middleware | `redux-persist` | `atomWithStorage` | Manual |
| **Outside React** | Yes (`getState/setState`) | Yes | Limited | No |
| **SSR safety** | Needs factory pattern | Needs per-request store | Needs per-request Provider | Needs per-request Provider |

### vs TanStack Query (different job)

| | **Zustand** | **TanStack Query** |
|---|---|---|
| **State type** | Client / UI state | Server / async state |
| **Caching** | Manual | Automatic, with stale-time |
| **Background refetch** | Manual | Automatic |
| **Loading/error states** | Manual | Built-in |
| **Deduplication** | Manual | Automatic |
| **Use together?** | Yes — Zustand for UI, TanStack Query for data | Yes |

---

## 15. Tips, Tricks & Gotchas

### 1. Never select the whole store without a reason

```ts
// Gotcha: this component re-renders on every single state change
const everything = useStore() // no selector — subscribes to entire store

// Fix: select what you need
const count = useStore((s) => s.count)
```

### 2. Selectors that return new objects/arrays always re-render

```ts
// Gotcha: new array [] on every call → always re-renders
const items = useStore((s) => s.todos.filter((t) => !t.done))

// Fix option A: useShallow (shallow-compares the array contents)
import { useShallow } from "zustand/react/shallow"
const items = useStore(useShallow((s) => s.todos.filter((t) => !t.done)))

// Fix option B: memoize outside the selector with useMemo
const todos = useStore((s) => s.todos)
const items = useMemo(() => todos.filter((t) => !t.done), [todos])
```

### 3. Actions are stable references — safe to omit from deps arrays

```ts
// Actions never change identity across renders — no need to add to useEffect deps
const fetchPosts = usePostsStore((s) => s.fetchPosts)
useEffect(() => { fetchPosts() }, []) // fetchPosts is stable, safe to omit from deps
// (ESLint will warn — suppress with // eslint-disable-next-line or add it: it won't cause issues)
```

### 4. Don't recreate the store on every render

```tsx
// WRONG — creates a new store on every render, losing all state
function Component() {
  const useLocalStore = create(() => ({ count: 0 })) // inside component!
  const count = useLocalStore((s) => s.count)
}

// RIGHT — define stores at module level (outside components)
const useLocalStore = create(() => ({ count: 0 }))
function Component() {
  const count = useLocalStore((s) => s.count)
}
```

### 5. Persist hydration mismatch in SSR

```tsx
// Gotcha: server renders with default state, client hydrates with localStorage state
// → React hydration mismatch warning

// Fix A: use skipHydration + manual rehydrate after mount (see section 12)

// Fix B: render nothing until hydrated
function ThemeAwareComponent() {
  const [hydrated, setHydrated] = useState(false)
  useEffect(() => setHydrated(true), [])

  const theme = useThemeStore((s) => s.theme)
  if (!hydrated) return null // skip SSR rendering
  return <div data-theme={theme}>...</div>
}

// Fix C: use the store's onFinishHydration callback
useEffect(() => {
  const unsub = useThemeStore.persist.onFinishHydration(() => setHydrated(true))
  return unsub
}, [])
```

### 6. Colocate actions with state

```ts
// BAD: actions live in components, state in store
function Counter() {
  const count = useCounterStore((s) => s.count)
  const set = useCounterStore((s) => s.setState) // reaching for set directly
  const handleClick = () => set({ count: count + 1 }) // business logic in component!
}

// GOOD: all business logic in the store
const useCounterStore = create()((set) => ({
  count: 0,
  increment: () => set((s) => ({ count: s.count + 1 })), // logic lives here
}))
function Counter() {
  const count = useCounterStore((s) => s.count)
  const increment = useCounterStore((s) => s.increment)
  return <button onClick={increment}>{count}</button>
}
```

### 7. Immer + persist: order matters

```ts
// CORRECT: immer inside persist inside devtools
create()(devtools(persist(immer(fn), persistOptions), devtoolsOptions))

// WRONG: persist inside immer — persist can't serialize Immer draft objects
create()(immer(persist(fn, persistOptions)))
```

### 8. Deep-nested state without Immer

```ts
// Gotcha: forgetting to spread nested objects
const useStore = create()((set) => ({
  user: { profile: { name: "Alice", bio: "Dev" } },
  updateBio: (bio: string) =>
    set((s) => ({
      // WRONG — replaces entire user with just { bio }
      user: { bio },

      // CORRECT — spread at each level
      user: { ...s.user, profile: { ...s.user.profile, bio } },
    })),
}))
// Or just use Immer to avoid this
```

### 9. Testing: reset state between tests

```ts
// Use beforeEach to guarantee clean state
beforeEach(() => {
  useMyStore.setState(initialState, true) // true = replace, not merge
})
```

### 10. Don't use Zustand for server state

```ts
// Anti-pattern: manually managing fetch state in Zustand
const usePostsStore = create()((set) => ({
  posts: [],
  loading: false,
  fetchPosts: async () => { /* reinventing TanStack Query */ }
}))

// Better: use TanStack Query for server data
const { data: posts } = useQuery({ queryKey: ["posts"], queryFn: fetchPosts })
// Reserve Zustand for: selected post ID, filter/sort UI state, modal open/closed
const selectedPostId = useUIStore((s) => s.selectedPostId)
```

---

## 16. Study Path

Work through these in order. Each step builds on the last.

### Phase 1 — Core API (Day 1-2)

1. Read the Zustand README at [github.com/pmndrs/zustand](https://github.com/pmndrs/zustand)
2. Build a **counter** with `create`, `set`, functional updates, and selectors
3. Add a **step** field and an `incrementByStep` action using `get()`
4. Verify re-renders: open React DevTools Profiler, confirm only the subscribed component re-renders

### Phase 2 — Real State Shapes (Day 3-4)

5. Build a **Todo app** (add, toggle, delete, filter) — practice with arrays in state
6. Add TypeScript interface typing (`create<TodoState>()`)
7. Add `useShallow` for a component that selects `{ todos, filter }` together
8. Write unit tests for the store logic using `createStore` from `"zustand/vanilla"`

### Phase 3 — Middleware (Day 5-6)

9. Add `persist` middleware to the Todo app — survive page refreshes
10. Add `devtools` middleware and explore time-travel in the Redux DevTools extension
11. Refactor mutation logic using `immer` middleware — compare before/after
12. Add `subscribeWithSelector` and sync `document.title` with todo count

### Phase 4 — Architecture (Day 7-9)

13. Build a **shopping cart** app with at least 3 slices: `cartSlice`, `userSlice`, `uiSlice`
14. Combine them into a `useBoundStore` with the slices pattern
15. Add a cross-slice action (e.g., clear cart on logout)
16. Access the cart total from outside React (in a hypothetical analytics module)

### Phase 5 — SSR & Next.js (Day 10-11)

17. Start a Next.js App Router project
18. Build the per-request store factory + provider pattern (section 12)
19. Pass server-fetched initial data into the store via the Provider's props
20. Handle the `persist` hydration mismatch with `skipHydration`

### Phase 6 — Combined with TanStack Query (Day 12-14)

21. Build a **blog dashboard**: TanStack Query fetches posts, Zustand stores `selectedPostId` and filter state
22. Demonstrate the boundary: Query owns server data, Zustand owns UI/client state
23. Write integration tests with `@testing-library/react` for components that use both

### Build-to-learn Projects

| Project | Concepts practiced |
|---|---|
| **Counter / Todo** | `create`, `set`, `get`, selectors, TypeScript |
| **Shopping Cart** | Arrays in state, slices, `persist`, `devtools` |
| **Auth Flow** | Async actions, `user` state, `logout`, `persist` |
| **Theme Switcher** | `persist`, SSR hydration, Next.js integration |
| **Dashboard + Data** | Zustand (UI) + TanStack Query (server), slices |
| **Kanban Board** | Immer for nested mutations, complex state shapes |
| **Real-time App** | `subscribeWithSelector`, WebSocket outside React, vanilla store |

### Key resources

- **Docs:** [docs.pmnd.rs/zustand](https://docs.pmnd.rs/zustand/getting-started/introduction)
- **GitHub:** [github.com/pmndrs/zustand](https://github.com/pmndrs/zustand)
- **Examples:** The `/examples` folder in the Zustand repo covers most patterns
- **Companion guide:** `TANSTACK_QUERY_GUIDE.md` in this repo — for server state

---

*Zustand v5 · Last updated June 2026*
