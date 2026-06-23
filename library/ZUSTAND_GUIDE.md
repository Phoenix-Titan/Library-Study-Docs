# Zustand — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I've heard React state is hard" to "I architect, test, and SSR-safely ship a production Zustand store" — without an internet connection. Every concept is explained in prose first: *what it is*, *the logic / why it works this way*, *what it's for and when to reach for it*, *how to use it*, *the key options*, *best practices*, and *the gotchas* — and only then a heavily-commented, runnable code block. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Zustand v5** (the current major in 2026), **React 19**, and **Next.js 15 (App Router)**. Things worth knowing about this stack:
> - **Zustand v5** dropped the deprecated default `import create from "zustand"` (default export) in favour of the named `import { create } from "zustand"`. The object-selector-without-equality footgun now *throws a helpful warning* in dev, pushing you toward `useShallow`. The shallow hook lives at `zustand/react/shallow`.
> - **React 19** ships `useSyncExternalStore` as the stable subscription primitive Zustand builds on, plus the `use()` hook and Actions — none of which replace Zustand, but they change *when* you need it (see §1 and §15).
> - **TanStack Query v5** is the companion for *server* state; the client-state vs server-state boundary is the single most important architectural decision in this guide (see §1 and §16).
>
> Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**; package-manager commands work identically across platforms. Confirm exact APIs at docs.pmnd.rs/zustand.

---

## Table of Contents

1. [What Client State Is & Why Zustand](#1-what-client-state-is--why-zustand) **[B]**
2. [Installation & Mental Model](#2-installation--mental-model) **[B]**
3. [Creating a Store with `create`](#3-creating-a-store-with-create) **[B]**
4. [Reading State: Selectors & Why They Matter](#4-reading-state-selectors--why-they-matter) **[B/I]**
5. [`useShallow` — Selecting Multiple Values](#5-useshallow--selecting-multiple-values) **[I]**
6. [Updating State: `set`, `get`, Merge vs Replace](#6-updating-state-set-get-merge-vs-replace) **[B/I]**
7. [Actions: Putting Logic in the Store](#7-actions-putting-logic-in-the-store) **[I]**
8. [Async Actions](#8-async-actions) **[I]**
9. [Derived & Computed State](#9-derived--computed-state) **[I]**
10. [Middleware](#10-middleware) **[I/A]**
11. [The Slices Pattern](#11-the-slices-pattern) **[A]**
12. [Accessing State Outside React](#12-accessing-state-outside-react) **[I/A]**
13. [Transient Updates](#13-transient-updates) **[A]**
14. [Zustand with Next.js & SSR](#14-zustand-with-nextjs--ssr) **[A]**
15. [Zustand vs Context vs Redux vs TanStack Query](#15-zustand-vs-context-vs-redux-vs-tanstack-query) **[I]**
16. [Testing Stores](#16-testing-stores) **[I]**
17. [Best Practices, Tips & Gotchas](#17-best-practices-tips--gotchas) **[I/A]**
18. [Study Path & Build-to-Learn Projects](#18-study-path--build-to-learn-projects)

---

## 1. What Client State Is & Why Zustand

Before you touch the API, you need a sharp mental model of *what kind of data Zustand is for*, because using it for the wrong kind of data is the single most common architectural mistake. State management in a React app splits into two fundamentally different categories, and they have different lifecycles, different correctness rules, and different tools.

**Server state** is data that lives authoritatively on a server and is *cached* in your client. The list of blog posts, the current user's profile from the database, search results — you don't own this data, you borrow a copy. It can go stale (someone else edited it), it needs refetching, deduplication, retries, background revalidation, and cache invalidation. This is the job of **TanStack Query** (or RTK Query, SWR). Trying to manage server state by hand in a global store means reinventing caching, loading flags, and invalidation — badly.

**Client state** (also called UI state or application state) is data your client *owns* and that has no authoritative server copy: which tab is selected, whether a modal is open, the contents of a not-yet-submitted shopping cart, the chosen theme, a multi-step form's progress, a "currently selected row" id. There's no "stale" concept — it's simply whatever the user last did. **This is what Zustand is for.**

The reason this distinction matters so much: people pull in Zustand, then start putting `fetchPosts` and `posts` and `isLoading` into it, and slowly recreate a worse TanStack Query. Keep the line clean — *Zustand owns what the user decides; Query owns what the server knows* — and your architecture stays simple. They compose beautifully together (Query holds the post list; Zustand holds `selectedPostId` and the filter UI state).

### The problem Zustand solves, concretely

The naive way to share client state across distant components is React Context. Context works, but it has a re-render characteristic that bites at scale: **every component that consumes a context re-renders whenever the context value changes, regardless of which part changed.** If your `AppContext` holds `{ theme, cart, user }` and the cart updates, every component reading the theme re-renders too. The standard workaround is splitting into many tiny contexts and memoizing providers — which is exactly the boilerplate you wanted to avoid.

Redux solves the re-render problem (components subscribe to slices via `useSelector` and only re-render when their slice changes) but charges a heavy ceremony tax: a store config, reducers, action types, action creators, `<Provider>`, `useSelector`, `useDispatch`, and dispatching action objects. For large teams enforcing strict patterns that's a feature; for most client state it's friction.

```tsx
// The Context re-render problem, illustrated.
// Every consumer of CartContext re-renders when ANY field in `value` changes —
// even a component that only reads `value.theme` and never touches the cart.
const CartContext = createContext(null)

function App() {
  const [cart, setCart] = useState([])
  const [theme, setTheme] = useState("light")
  // A NEW object literal each render → all consumers re-render every render
  return (
    <CartContext.Provider value={{ cart, setCart, theme, setTheme }}>
      {/* deeply nested tree of consumers */}
    </CartContext.Provider>
  )
}
```

### What Zustand gives you

Zustand is a tiny (~1 KB gzipped) state library built on React's official `useSyncExternalStore`. Its design choices map directly onto the pain points above:

- **No provider** — the store is a module-level singleton you import anywhere. (The one exception is SSR; see §14.)
- **No reducers, no action types** — actions are plain functions that call `set`. The store is just an object of state and functions.
- **Selector-based subscriptions** — a component subscribes to *exactly* the slice its selector returns, and re-renders only when that slice changes (compared with `Object.is`). This is the Redux re-render benefit without the Redux ceremony.
- **Works outside React** — `getState()`, `setState()`, and `subscribe()` are available on the store with no hook, so non-React code (WebSocket handlers, analytics, utility modules) can read and write.
- **Composable middleware** — persistence, devtools, Immer, and selector-subscriptions are opt-in wrappers.

```tsx
// The entire counter store — no provider, no reducer, no action constants.
import { create } from "zustand"

const useCounterStore = create((set) => ({
  count: 0,
  increment: () => set((s) => ({ count: s.count + 1 })),
  decrement: () => set((s) => ({ count: s.count - 1 })),
  reset: () => set({ count: 0 }),
}))

function Counter() {
  // This component subscribes ONLY to `count`. It re-renders only when count changes.
  const count = useCounterStore((s) => s.count)
  // Actions never change identity, so subscribing to one never causes a re-render.
  const increment = useCounterStore((s) => s.increment)
  return <button onClick={increment}>{count}</button>
}
```

### Choosing the right tool — the decision table

| Scenario | Tool | Why |
|---|---|---|
| Server data: loading, caching, background sync, retries | **TanStack Query** | Purpose-built cache; don't hand-roll it |
| Global client/UI state shared across many components | **Zustand** | Selector subscriptions, no boilerplate |
| State used by exactly one component | **`useState` / `useReducer`** | No need to globalize local state |
| Rarely-changing config passed to a subtree (theme tokens, locale) | **Context** | Low churn → re-render cost is negligible |
| Complex server state AND rich client state in one app | **Query + Zustand** | Each does its own job; they compose |
| Fine-grained, bottom-up atomic state | **Jotai** | Atom model fits some apps better |

> **⚡ React 19 note:** React 19 adds `use()`, Actions, `useActionState`, and `useOptimistic`. These cover *form-and-mutation* flows and reading promises/context — they do not replace a shared client-state store. You still reach for Zustand when many unrelated components across the tree need to read and mutate the same piece of UI state.

---

## 2. Installation & Mental Model

Zustand has zero peer dependencies beyond React and adds almost nothing to your bundle. Install it with your package manager of choice.

```bash
npm install zustand
# or: pnpm add zustand   /   yarn add zustand   /   bun add zustand
```

If you plan to use the Immer middleware (for ergonomic nested updates, §10.3), install Immer too — it's a separate package:

```bash
npm install immer
```

### The mental model: a store is an external object React reads from

Internally a Zustand store is a plain JavaScript object living in module scope, plus a list of subscriber callbacks. When you call `set`, Zustand computes the next state, swaps it in, then notifies every subscriber. The React binding (`useCounterStore(selector)`) registers your component as a subscriber via `useSyncExternalStore`; on each notification it runs your selector, compares the result to the previous result with `Object.is`, and re-renders the component *only if the result changed*.

Three consequences of this model are worth internalising now, because most Zustand "bugs" trace back to misunderstanding one of them:

1. **The selector is your subscription.** What you return from the selector defines what you re-render on. Return the whole store → re-render on everything. Return one primitive → re-render only when that primitive changes.
2. **Equality is `Object.is` by default.** A selector that builds a *new* object or array every call (e.g. `(s) => ({ a: s.a, b: s.b })`) is never `Object.is`-equal to its previous result, so it re-renders on *every* store update. This is what `useShallow` fixes (§5).
3. **Identity stability matters.** Functions defined once in the store (your actions) keep the same reference forever, so selecting them is free. Values you compute fresh each call are not stable.

⚡ **Version note (v5):** In v5 the default export was removed — always use the named import `{ create }`. The `useShallow` hook moved to `zustand/react/shallow`. The vanilla store creator is `createStore` from `zustand/vanilla`, and the generic React binder is `useStore` from `zustand`.

---

## 3. Creating a Store with `create`

`create` is the function you'll use 95% of the time. **What it is:** a factory that takes an *initializer* function and returns a React hook bound to a new store. **The logic:** the initializer receives Zustand's `set` and `get` functions and returns the store's initial shape — both the data *and* the actions, together in one flat object. There's no separation of "state" and "methods"; in Zustand, actions are just fields whose values happen to be functions.

**How to use it (TypeScript):** you pass your state interface as the generic and call `create` in a curried, double-call form: `create<State>()(initializer)`. The empty `()` after the generic is not a typo — it exists so TypeScript can fully infer the types of `set` and `get` inside your initializer. Without the generic (`create(fn)`), it still runs, but `set`/`get` lose their precise typing.

```ts
// store/counterStore.ts
import { create } from "zustand"

// 1. Describe the FULL store shape — state fields AND action signatures together.
interface CounterState {
  // --- state ---
  count: number
  step: number
  // --- actions (functions live alongside data) ---
  increment: () => void
  decrement: () => void
  incrementByStep: () => void
  setStep: (step: number) => void
  reset: () => void
}

// 2. create<State>()(initializer). The empty () enables full type inference of set/get.
export const useCounterStore = create<CounterState>()((set, get) => ({
  // --- initial state ---
  count: 0,
  step: 1,

  // --- actions ---
  // Functional update form: derive the next value from the previous state.
  // ALWAYS use this form when the new value depends on the old value (avoids stale reads).
  increment: () => set((s) => ({ count: s.count + 1 })),
  decrement: () => set((s) => ({ count: s.count - 1 })),

  // `get()` reads the current state synchronously inside an action without subscribing.
  incrementByStep: () => set((s) => ({ count: s.count + get().step })),

  // Object form: pass the keys you want to change; Zustand shallow-merges them in.
  setStep: (step) => set({ step }),

  // Reset just sets the field(s) back to their initial values.
  reset: () => set({ count: 0 }),
}))
```

### Why state and actions share one object

Beginners coming from Redux expect a reducer separate from state. Zustand deliberately fuses them: an action is a closure that already captures `set` and `get`, so it can read and write the store directly. This is what eliminates action types and dispatch. The trade-off is that your interface mixes data and functions — embrace it; it's idiomatic. A common convention is to group actions under a nested `actions` key if you prefer visual separation, but the flat form is the default and the one most examples use.

### File organisation conventions

Keep stores **domain-scoped** — one store per cohesive area of concern — rather than a single god-store holding everything. A god-store couples unrelated features and makes the slices pattern (§11) harder to adopt later.

```
src/
  store/
    counterStore.ts      # one store per domain
    cartStore.ts
    authStore.ts
    useBoundStore.ts     # the combined store, if/when you adopt slices
```

> **Naming convention:** prefix the returned hook with `use` (e.g. `useCounterStore`) because it *is* a React hook and ESLint's rules-of-hooks plugin relies on the `use` prefix to lint it correctly.

---

## 4. Reading State: Selectors & Why They Matter

A **selector** is the function you pass to the store hook: `useCounterStore((s) => s.count)`. **What it is:** a pure function from the full state to the specific value your component needs. **Why it exists / the logic:** as established in §2, the selector *is* your subscription. Zustand runs it after every `set`, compares the new return value to the old with `Object.is`, and re-renders your component only on a change. So the selector is simultaneously "what do I want to read" and "what do I want to re-render on" — one of the most elegant ideas in the library, and the source of its performance.

**Why this matters for performance (the core lesson):** if you call the hook with *no* selector, you subscribe to the entire store object. Every `set` — even one touching a completely unrelated field — produces a new top-level state reference, so your component re-renders on every change in the app. With a narrow selector, you re-render only when *your* data changes. In a large app this is the difference between a snappy UI and one that re-renders hundreds of components on every keystroke.

```tsx
// components/CounterDisplay.tsx
import { useCounterStore } from "../store/counterStore"

export function CounterDisplay() {
  // Selector subscribes ONLY to `count`.
  // Re-renders ONLY when count changes — not when step or anything else changes.
  const count = useCounterStore((s) => s.count)
  return <p>Count: {count}</p>
}
```

```tsx
// BAD — no selector subscribes to the ENTIRE store.
// This re-renders on every single state change anywhere in the store.
function Bad() {
  const store = useCounterStore() // ❌ whole store
  return <p>{store.count}</p>
}

// GOOD — narrow selector subscribes to exactly one primitive.
function Good() {
  const count = useCounterStore((s) => s.count) // ✅ just count
  return <p>{count}</p>
}
```

### Selecting actions is free

Actions are defined once when the store is created and never reassigned, so their function identity is permanently stable. Selecting an action therefore never triggers a re-render and never needs `useShallow`. Select each action with its own narrow selector:

```tsx
export function CounterControls() {
  // Three separate selectors, each returning a stable function reference.
  // None of these cause re-renders when other state changes.
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

### Deriving a value inside the selector

You can compute a derived value right in the selector. It re-runs on every render but only triggers a re-render when its *result* changes (for primitives). This is the simplest form of derived state (more in §9):

```tsx
// Re-renders only when the computed total changes value (a number → Object.is works).
const totalItems = useCartStore((s) =>
  s.cart.reduce((acc, item) => acc + item.quantity, 0)
)
```

> **Gotcha — returning fresh objects/arrays from a selector.** A selector that returns a *new* object or array each call (`(s) => ({ a: s.a, b: s.b })` or `(s) => s.list.filter(...)`) is never `Object.is`-equal to its previous result, so it re-renders on *every* store update — defeating the entire point of selectors. The fix is `useShallow` (§5) for the multi-value case, or `useMemo`/storing derived state for the computed-array case (§9).

> **Best practice:** prefer many small narrow selectors over one big object selector. A component reading three fields with three `useStore` calls is idiomatic, fast, and needs no equality function. Reach for `useShallow` only when you genuinely must return a composite in one call.

---

## 5. `useShallow` — Selecting Multiple Values

Sometimes you genuinely want several values from one selector call — perhaps you're destructuring a group of related fields. The naive approach returns an object literal, which, as §4 warned, re-renders on every update because the new object is never reference-equal to the old one.

**What `useShallow` is:** a wrapper for your selector that swaps the default `Object.is` comparison for a **shallow** comparison — it compares the object's (or array's) top-level entries one by one. **The logic:** if every field of the new object equals the corresponding field of the old object, the result is treated as unchanged and no re-render happens, even though the *container* object is a new reference. **When to use it:** any time a selector returns an object or array assembled from multiple store values. **When NOT to use it:** for a single primitive (default `Object.is` is correct and cheaper), for a single action (stable identity), or for a single object you intend to compare by reference.

```tsx
// ❌ BAD — new object literal each render → re-renders on EVERY store update.
const { count, step } = useCounterStore((s) => ({ count: s.count, step: s.step }))

// ✅ GOOD — useShallow compares { count, step } field-by-field.
// Re-renders only when count OR step actually changes value.
import { useShallow } from "zustand/react/shallow" // v5 import path

const { count, step } = useCounterStore(
  useShallow((s) => ({ count: s.count, step: s.step }))
)
```

You can also select an array — handy for grabbing a value plus its action together:

```tsx
import { useShallow } from "zustand/react/shallow"

// Array selector: shallow comparison walks the array elements.
const [count, increment] = useCounterStore(
  useShallow((s) => [s.count, s.increment])
)
```

A particularly common and useful pattern is an **action-bundle selector** — group all of a store's actions (which are stable) into one object so a component grabs them in a single call. Because actions never change identity, even a plain object selector here is safe, but `useShallow` makes the intent explicit and future-proof:

```tsx
import { useShallow } from "zustand/react/shallow"

const { addItem, removeItem, clearCart } = useCartStore(
  useShallow((s) => ({
    addItem: s.addItem,
    removeItem: s.removeItem,
    clearCart: s.clearCart,
  }))
)
```

⚡ **Version note:** In Zustand v4 the equality function was passed as a second argument (`useStore(selector, shallow)` with `shallow` from `zustand/shallow`). In **v5** the idiomatic API is the `useShallow` hook from `zustand/react/shallow`, which wraps the selector instead. v5 also *warns in development* when it detects a selector returning a new object without a stable equality strategy — your cue to add `useShallow`.

---

## 6. Updating State: `set`, `get`, Merge vs Replace

### `set` — the only way to change state

`set` is handed to every action by the initializer. **What it does:** it computes the next state and notifies subscribers. **Crucial detail — it shallow-merges by default**, exactly like the old React class `this.setState`: the object you pass is merged into the top level of the current state, so keys you *don't* mention are preserved untouched. This is why a `setName` action can pass just `{ name }` without wiping `age` or `address`.

`set` accepts two forms:
- **Object form** — `set({ count: 5 })`: merge these keys.
- **Functional form** — `set((s) => ({ count: s.count + 1 }))`: receive the current state, return a partial to merge. **Always use the functional form when the new value depends on the old one**, because it reads the freshest state at call time and avoids stale-closure bugs (the same reasoning as functional `useState` updates).

```ts
import { create } from "zustand"

interface UserState {
  name: string
  age: number
  address: { city: string; country: string }
  setName: (name: string) => void
  birthday: () => void
  setCity: (city: string) => void
}

const useUserStore = create<UserState>()((set, get) => ({
  name: "Alice",
  age: 30,
  address: { city: "London", country: "UK" },

  // Object form, shallow merge: only `name` changes; age and address survive.
  setName: (name) => set({ name }),

  // Functional form: next value depends on previous → use the updater function.
  birthday: () => set((s) => ({ age: s.age + 1 })),

  // ⚠️ NESTED objects are NOT deep-merged. You must spread the old level yourself,
  // otherwise you replace the whole `address` with just { city }.
  setCity: (city) =>
    set((s) => ({
      address: { ...s.address, city }, // spread old address, then override city
    })),
}))
```

### `get` — reading state inside actions

`get()` returns the current full state synchronously. **Why it's separate from the selector:** inside an action you don't want to *subscribe* to anything — you just want the latest values to compute with. `get()` gives you exactly that, and it's also how one action calls another or reads a value updated moments ago in an async flow.

```ts
const useStore = create<{ count: number; doubled: () => number; addDoubled: () => void }>()(
  (set, get) => ({
    count: 5,
    // Compute from current state without subscribing.
    doubled: () => get().count * 2,
    // Use get() to read freshly, then set().
    addDoubled: () => set({ count: get().count + get().doubled() }),
  })
)
```

### Merge vs Replace

The optional second argument to `set` controls whether to **merge** (default) or **replace** (wipe everything not in the new object). Replace is mainly used to reset a whole store to a known shape.

```ts
// Default — MERGE (top-level shallow merge): unmentioned keys are kept.
set({ name: "Bob" })
// → { name: "Bob", age: 30, address: {...} }   (age & address survive)

// REPLACE — pass true as the second argument: keys not in the new object are removed.
set({ name: "Bob" }, true)
// → { name: "Bob" }   (age & address are GONE — including your ACTIONS!)
```

> **Gotcha — replace deletes your actions too.** Because actions live in the same object as state, `set(partial, true)` wipes them unless you include them. The safe reset pattern is to keep an `initialState` constant of *just the data* and merge it (not replace), or to spread the actions back in:

```ts
const initialState = { count: 0, step: 1 } as const

const useStore = create<typeof initialState & { reset: () => void }>()((set) => ({
  ...initialState,
  // Merge the initial DATA back in — actions are untouched because we don't replace.
  reset: () => set({ ...initialState }),
}))
```

⚡ **Version note (v5):** v5 refined the `set` typing so that when you pass `true` (replace), TypeScript requires the object to match the full state shape (it's a genuine replace, after all). The boolean shorthand still works; some typings also accept `{ replace: true }`. Prefer the boolean for brevity, and prefer *merge-based resets* to sidestep the action-wipe gotcha entirely.

---

## 7. Actions: Putting Logic in the Store

An **action** is just a field whose value is a function that calls `set` (and maybe `get`). **The logic / why colocate them in the store:** business logic that mutates state belongs *next to* that state, not scattered across components. When a component reaches for raw `setState` and writes `set({ count: count + 1 })` inline, the rule for "how the count changes" leaks into the view. Put it in the store as `increment`, and every component that needs to increment calls the same tested function. This keeps components dumb (read + call) and makes the store the single source of truth for *behaviour*, not just data.

```ts
// ✅ Logic lives in the store; components just call named actions.
const useCounterStore = create<CounterState>()((set, get) => ({
  count: 0,
  // The RULE for incrementing is defined once, here.
  increment: () => set((s) => ({ count: s.count + 1 })),
  // Actions can be arbitrarily rich: validate, branch, call other actions via get().
  incrementIfPositive: () => {
    if (get().count >= 0) set((s) => ({ count: s.count + 1 }))
  },
}))
```

```tsx
// Components stay thin: read state, call action. No business logic here.
function Counter() {
  const count = useCounterStore((s) => s.count)
  const increment = useCounterStore((s) => s.increment)
  return <button onClick={increment}>{count}</button>
}
```

> **Best practice — never reach for `setState` from a component for domain logic.** It's available (it's a static method on the hook, §12), but using it in components scatters logic. Reserve direct `setState` for test setup and genuine non-React integration points.

> **Optional structure — group actions.** Some teams nest actions under an `actions` key (`actions: { increment, decrement }`) so selectors clearly separate data from behaviour, and so a single `useStore(s => s.actions)` grabs them all (the `actions` object reference is stable). This is a stylistic choice; the flat layout is the default.

---

## 8. Async Actions

Because actions are plain functions, making one `async` requires no special middleware, thunks, or sagas — you simply write `async () => { ... }` and call `set` whenever you have something to store. **When to use Zustand for async:** fire-and-forget mutations, one-time loads you fully control, or coordinating client state around an operation. **When NOT to:** anything that's truly *server state* with caching/refetch/dedup needs — that's TanStack Query's job (see the note at the end and §15). The pattern below (manual `posts`/`isLoading`/`error`) is shown so you understand it, but in real apps you'd usually let Query own that triad.

```ts
// store/postsStore.ts
import { create } from "zustand"

interface Post { id: number; title: string; body: string }

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
    // Guard against overlapping fetches: read live state via get().
    if (get().isLoading) return

    set({ isLoading: true, error: null }) // enter loading
    try {
      const res = await fetch("https://jsonplaceholder.typicode.com/posts?_limit=10")
      if (!res.ok) throw new Error(`HTTP ${res.status}`)
      const posts: Post[] = await res.json()
      set({ posts, isLoading: false }) // success
    } catch (err) {
      // Always clear isLoading in the failure path too, or the UI hangs on a spinner.
      set({ error: (err as Error).message, isLoading: false })
    }
  },

  addPost: async (newPost) => {
    const res = await fetch("https://jsonplaceholder.typicode.com/posts", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(newPost),
    })
    const created: Post = await res.json()
    // Prepend to the local list once the server confirms.
    set((s) => ({ posts: [created, ...s.posts] }))
    return created
  },
}))
```

```tsx
// components/PostsList.tsx
import { useEffect } from "react"
import { useShallow } from "zustand/react/shallow"
import { usePostsStore } from "../store/postsStore"

export function PostsList() {
  // useShallow because we select multiple values in one object.
  const { posts, isLoading, error } = usePostsStore(
    useShallow((s) => ({ posts: s.posts, isLoading: s.isLoading, error: s.error }))
  )
  // Action selected separately — stable reference, no re-render concern.
  const fetchPosts = usePostsStore((s) => s.fetchPosts)

  useEffect(() => { fetchPosts() }, [fetchPosts]) // fetchPosts is stable; safe in deps

  if (isLoading) return <p>Loading…</p>
  if (error) return <p>Error: {error}</p>
  return <ul>{posts.map((p) => <li key={p.id}>{p.title}</li>)}</ul>
}
```

> **Gotcha — error paths must reset flags.** Every `try` that sets `isLoading: true` needs its `catch` (and any early returns) to set `isLoading: false`, or the UI is stuck. A `finally` block is the bulletproof way to guarantee it.

> **Client-state vs server-state, restated.** The block above is essentially a hand-rolled cache. For real server data, prefer `useQuery({ queryKey: ["posts"], queryFn })` and let Zustand hold only the *UI* state around it (selected id, filters, sort). See §15 for the full comparison.

---

## 9. Derived & Computed State

Derived state is any value computed *from* other state — a cart total from cart items, a "completed count" from a todo list. Zustand has no built-in `computed` field (unlike Vue's `computed` or MobX), so you choose among three patterns based on cost and reuse. Understanding the trade-offs prevents both unnecessary re-renders and stale derived values.

**Pattern A — derive inline in a selector (default, simplest).** Compute the value right in the selector. It recomputes on each render, and (for a primitive result) re-renders only when the result changes value. Best for cheap computations.

```tsx
// Cheap reduction; result is a number, so Object.is gates re-renders correctly.
const totalPrice = useCartStore((s) =>
  s.cart.reduce((sum, i) => sum + i.price * i.quantity, 0)
)
```

**Pattern B — memoize with `useMemo` for expensive work or object results.** When the derivation is costly, or returns an object/array (which would otherwise re-render every time), select the raw input and memoize the computation in the component. The memo recomputes only when its dependency changes.

```tsx
import { useMemo } from "react"

function OrderSummary() {
  const cart = useCartStore((s) => s.cart) // select the raw input
  // Memoized: recomputes only when `cart` reference changes.
  const { total, itemCount } = useMemo(() => ({
    total: cart.reduce((s, i) => s + i.price * i.quantity, 0),
    itemCount: cart.reduce((s, i) => s + i.quantity, 0),
  }), [cart])
  return <div>Items: {itemCount}, Total: ${total.toFixed(2)}</div>
}
```

**Pattern C — store the derived value and keep it in sync inside actions.** If many components read the derived value and recomputing it everywhere is wasteful, compute it once *in the action that changes the inputs* and store it as a real field. The cost is discipline: every action that touches the inputs must update the derived field, or it goes stale.

```ts
const useCartStore = create<CartState>()((set) => ({
  cart: [],
  total: 0, // derived field kept in sync by every mutating action
  addItem: (item) =>
    set((s) => {
      const cart = [...s.cart, item]
      // Recompute total here so `total` is always consistent with `cart`.
      return { cart, total: cart.reduce((a, i) => a + i.price * i.quantity, 0) }
    }),
}))
```

> **Best practice:** start with Pattern A. Move to B when the computation is expensive or returns a composite. Reach for C only when the derived value is read in many places and recomputation is a measured bottleneck — it trades correctness-by-construction for performance, so guard it with tests.

---

## 10. Middleware

Middleware are wrappers around your store initializer that add behaviour — persistence, devtools, mutable-style updates, selector subscriptions. **The logic:** each middleware takes an initializer and returns an enhanced initializer, so you compose them by nesting. **Order matters** because the outermost wraps last and sees the final shape; the conventional stack from outside in is `devtools(persist(immer(...)))` — devtools observes everything, persist serializes, immer is closest to your raw `set` calls. (More on ordering in §10.6.)

### 10.1 `devtools` — Redux DevTools integration

**What it does:** pipes every state change into the Redux DevTools browser extension, giving you a timeline, action names, state diffs, and time-travel — for free. **When:** during development of any non-trivial store. **How:** wrap the initializer and optionally name each action via the third `set` argument so the timeline reads meaningfully.

```ts
import { create } from "zustand"
import { devtools } from "zustand/middleware"

export const useCounterStore = create<CounterState>()(
  devtools(
    (set) => ({
      count: 0,
      // set(updater, replace?, actionName?) — the 3rd arg labels this action in DevTools.
      increment: () => set((s) => ({ count: s.count + 1 }), false, "counter/increment"),
      reset: () => set({ count: 0 }, false, "counter/reset"),
    }),
    { name: "CounterStore", enabled: process.env.NODE_ENV !== "production" }
  )
)
```

> **Best practice:** gate devtools to development with `enabled` so you don't ship the instrumentation (and its action-name strings) to production.

### 10.2 `persist` — localStorage / sessionStorage / custom storage

**What it does:** automatically writes the store (or a chosen subset) to a storage backend on every change, and rehydrates it on load. **When:** theme, cart, "remember me" preferences, draft form data — any client state that should survive a refresh. **The key options** are worth knowing precisely:

- **`name`** *(required)* — the storage key.
- **`storage`** — wrap your backend with `createJSONStorage(() => localStorage)`. Defaults to `localStorage`; swap for `sessionStorage`, cookies, or React Native's AsyncStorage.
- **`partialize`** — a function returning only the fields to persist. Use it to *exclude* transient state (loading flags, open menus) so you don't restore stale UI.
- **`version`** + **`migrate`** — version your persisted shape; when you bump `version`, `migrate` transforms old persisted data into the new shape so returning users don't crash on a schema change.
- **`onRehydrateStorage`** — a callback fired before/after rehydration (useful to flip a "hydrated" flag).
- **`skipHydration`** — defer rehydration so *you* trigger it manually (critical for SSR; see §14).

```ts
import { create } from "zustand"
import { persist, createJSONStorage } from "zustand/middleware"

interface ThemeState {
  theme: "light" | "dark"
  fontSize: number
  _hasHydrated: boolean
  toggleTheme: () => void
  setFontSize: (n: number) => void
}

export const useThemeStore = create<ThemeState>()(
  persist(
    (set) => ({
      theme: "light",
      fontSize: 16,
      _hasHydrated: false,
      toggleTheme: () => set((s) => ({ theme: s.theme === "light" ? "dark" : "light" })),
      setFontSize: (fontSize) => set({ fontSize }),
    }),
    {
      name: "theme-storage",                            // localStorage key
      storage: createJSONStorage(() => localStorage),   // backend (default shown)
      // Persist ONLY theme + fontSize; never persist the hydration flag.
      partialize: (s) => ({ theme: s.theme, fontSize: s.fontSize }),
      onRehydrateStorage: () => (state) => {
        // Runs after rehydration completes; mark the store as ready.
        state?.setFontSize(state.fontSize)
      },
    }
  )
)
```

#### Persist with migrations

```ts
export const useSettingsStore = create<SettingsState>()(
  persist((set) => ({ /* ... */ }), {
    name: "settings",
    version: 2, // bump whenever the persisted SHAPE changes
    migrate: (persisted: unknown, fromVersion: number) => {
      // Transform old persisted data forward to the current shape.
      if (fromVersion === 0) {
        // v0 → v1: field `color` was renamed to `theme`.
        const old = persisted as { color: string }
        return { theme: old.color }
      }
      if (fromVersion === 1) {
        // v1 → v2: a new `fontSize` field gained a default.
        const old = persisted as { theme: string }
        return { ...old, fontSize: 16 }
      }
      return persisted // already current
    },
  })
)
```

#### Custom storage (cookies, AsyncStorage)

```ts
import { StateStorage } from "zustand/middleware"

// Any object with getItem/setItem/removeItem is a valid backend.
const cookieStorage: StateStorage = {
  getItem: (name) => {
    const m = document.cookie.match(new RegExp(`(^| )${name}=([^;]+)`))
    return m ? decodeURIComponent(m[2]) : null
  },
  setItem: (name, value) => {
    document.cookie = `${name}=${encodeURIComponent(value)}; path=/; max-age=31536000`
  },
  removeItem: (name) => { document.cookie = `${name}=; max-age=0` },
}

const useStore = create<MyState>()(
  persist((set) => ({ /* ... */ }), {
    name: "my-cookie-store",
    storage: createJSONStorage(() => cookieStorage),
  })
)
```

> **Gotcha — never persist server state or secrets.** `partialize` away anything that should come fresh from the server, plus tokens you don't want sitting in `localStorage`. And remember `persist` triggers the SSR hydration hazard covered in §14.

### 10.3 `immer` — mutable-style updates for nested state

**The problem it solves:** updating deeply nested state immutably means spreading at every level (`{ ...s, a: { ...s.a, b: { ...s.a.b, c } } }`), which is verbose and error-prone. **What `immer` does:** lets you *mutate* a draft inside `set`, then produces a correct new immutable state behind the scenes. **When:** stores with nested objects/arrays where manual spreading is painful. Requires the `immer` package installed.

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
    // Inside immer's set you MUTATE the draft directly — no spreading.
    addTodo: (text) =>
      set((s) => { s.todos.push({ id: Date.now(), text, done: false }) }),
    toggleTodo: (id) =>
      set((s) => {
        const t = s.todos.find((t) => t.id === id)
        if (t) t.done = !t.done // mutate safely; immer makes it immutable
      }),
    removeTodo: (id) =>
      set((s) => { s.todos = s.todos.filter((t) => t.id !== id) }),
  }))
)
```

> **Gotcha — with immer, return nothing OR a full new state, not a partial.** Immer's `set` either mutates the draft (return `undefined`) or you assign fields on the draft. Don't return a partial object the way you would without immer — mixing the two confuses the draft semantics.

### 10.4 `subscribeWithSelector` — granular subscriptions outside React

By default `store.subscribe(listener)` fires on *every* change and hands you the whole state. **What `subscribeWithSelector` adds:** the ability to subscribe to a *specific slice* with its own equality function and to receive `(newValue, prevValue)`. **When:** non-React reactions to a single field — sync `document.title`, push to analytics on a specific change, trigger side effects at a threshold.

```ts
import { create } from "zustand"
import { subscribeWithSelector } from "zustand/middleware"

const useStore = create<{ count: number; increment: () => void }>()(
  subscribeWithSelector((set) => ({
    count: 0,
    increment: () => set((s) => ({ count: s.count + 1 })),
  }))
)

// Fires ONLY when `count` changes, with both new and previous values.
const unsub = useStore.subscribe(
  (s) => s.count,                       // selector: which slice to watch
  (count, prevCount) => {               // listener: react to the change
    console.log(`count: ${prevCount} → ${count}`)
    if (count >= 10) console.log("Reached 10!")
  },
  { equalityFn: Object.is, fireImmediately: false } // options
)

unsub() // stop listening
```

### 10.5 `combine` — let TypeScript infer the state type

**What it does:** `combine(initialState, actionsCreator)` infers the full store type from the initial state object, so you don't write an interface. **When:** quick stores where you'd rather not maintain a separate type. Trade-off: less explicit than a hand-written interface for large stores.

```ts
import { create } from "zustand"
import { combine } from "zustand/middleware"

export const useStore = create(
  combine(
    { bears: 0, bees: 0 },           // initial state → type inferred from this
    (set, get) => ({
      addBear: () => set((s) => ({ bears: s.bears + 1 })),
      addBee: () => set((s) => ({ bees: s.bees + 1 })),
      total: () => get().bears + get().bees,
    })
  )
)
```

### 10.6 Stacking middleware — order and reasoning

```ts
import { create } from "zustand"
import { devtools, persist } from "zustand/middleware"
import { immer } from "zustand/middleware/immer"

// Convention from OUTSIDE in: devtools → persist → immer.
//  - devtools outermost: it observes the final, post-everything state.
//  - persist next: serializes the (already-immer-managed) state.
//  - immer innermost: closest to your set() calls so drafts work.
export const useCartStore = create<CartState>()(
  devtools(
    persist(
      immer((set) => ({
        items: [],
        addItem: (item) => set((s) => { s.items.push(item) }),
        clearCart: () => set((s) => { s.items = [] }),
      })),
      { name: "cart-storage", partialize: (s) => ({ items: s.items }) }
    ),
    { name: "CartStore" }
  )
)
```

> **Gotcha — `persist(immer(...))`, not `immer(persist(...))`.** If `persist` is *inside* `immer`, persist may try to serialize Immer draft objects, which can break. Keep `immer` innermost.

### Middleware reference

| Middleware | Import | Adds | Reach for it when |
|---|---|---|---|
| `devtools` | `zustand/middleware` | Redux DevTools timeline/time-travel | Debugging any real store |
| `persist` | `zustand/middleware` | Save/restore to storage | State must survive refresh |
| `immer` | `zustand/middleware/immer` | Mutable-style nested updates | Deep nested state |
| `subscribeWithSelector` | `zustand/middleware` | Per-slice `subscribe` with prev value | Non-React reactions to one field |
| `combine` | `zustand/middleware` | Auto-inferred state type | Quick stores, no interface |

---

## 11. The Slices Pattern

As an app grows, one store file holding cart + user + UI + settings becomes a sprawling, hard-to-navigate object. The **slices pattern** splits the single store into multiple *slice creators* — each a self-contained piece defining part of the state and its actions — then merges them into one combined store. **The logic:** you keep Zustand's single-store ergonomics (one hook, cross-slice access) while organising code by domain. It's the recommended way to scale a Zustand store without fragmenting into many independent stores (which would lose cross-domain coordination).

The key type is **`StateCreator`**, which takes (up to) four generics: `StateCreator<FullStore, Mutators, [], ThisSlice>`. The first generic is the *whole* combined store type — that's what lets a slice's `get()` reach into other slices. The second is the middleware-mutators list (leave `[]` unless the slice itself declares middleware). The fourth is what *this* slice contributes.

### Step 1 — define each slice creator

```ts
// store/slices/counterSlice.ts
import { StateCreator } from "zustand"
import type { BoundState } from "../useBoundStore"

export interface CounterSlice {
  count: number
  increment: () => void
  resetCount: () => void
}

// StateCreator<FullStore, [], [], ThisSlice>
//   FullStore (BoundState) as generic #1 → get() can read OTHER slices.
export const createCounterSlice: StateCreator<BoundState, [], [], CounterSlice> =
  (set) => ({
    count: 0,
    increment: () => set((s) => ({ count: s.count + 1 })),
    resetCount: () => set({ count: 0 }),
  })
```

```ts
// store/slices/userSlice.ts
import { StateCreator } from "zustand"
import type { BoundState } from "../useBoundStore"

export interface UserSlice {
  user: { name: string; email: string } | null
  setUser: (user: UserSlice["user"]) => void
  logout: () => void
}

export const createUserSlice: StateCreator<BoundState, [], [], UserSlice> =
  (set) => ({
    user: null,
    setUser: (user) => set({ user }),
    logout: () => set({ user: null }),
  })
```

### Step 2 — combine into one bound store

```ts
// store/useBoundStore.ts
import { create } from "zustand"
import { CounterSlice, createCounterSlice } from "./slices/counterSlice"
import { UserSlice, createUserSlice } from "./slices/userSlice"

// The combined type is the intersection of all slice types.
export type BoundState = CounterSlice & UserSlice

// Spread each slice creator with the same (set, get, store) args (...args).
export const useBoundStore = create<BoundState>()((...args) => ({
  ...createCounterSlice(...args),
  ...createUserSlice(...args),
}))
```

### Step 3 — use it like any store

```tsx
// Every slice is available on the one hook. Narrow selectors as always.
const count = useBoundStore((s) => s.count)
const user = useBoundStore((s) => s.user)
const increment = useBoundStore((s) => s.increment)
const logout = useBoundStore((s) => s.logout)
```

### Cross-slice actions

Because every slice receives the *full* store type, one slice's action can read or write another's via `get()`/`set()` — coordination without coupling files:

```ts
// counterSlice can react to user state because get() returns the full BoundState.
export const createCounterSlice: StateCreator<BoundState, [], [], CounterSlice> =
  (set, get) => ({
    count: 0,
    increment: () => set((s) => ({ count: s.count + 1 })),
    resetCount: () => set({ count: 0 }),
    // Cross-slice read: only increment when a user is logged in.
    incrementIfLoggedIn: () => {
      if (get().user !== null) set((s) => ({ count: s.count + 1 }))
    },
  })
```

### Slices with middleware

Middleware wraps the *combined* creator, so it applies to the whole store. When middleware is present, the `StateCreator` mutator generic must reflect it (e.g. `["zustand/immer", never]`), but for the common `devtools` + `persist` case you usually wrap at the top and keep slice creators plain:

```ts
export const useBoundStore = create<BoundState>()(
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

> **Best practice:** keep slices truly independent in their *data*; use cross-slice actions sparingly and deliberately. If two slices constantly reach into each other, they probably belong together.

---

## 12. Accessing State Outside React

Every store created with `create` is, under the hood, a vanilla store with the React binding bolted on. That means the hook object itself carries **static methods** — `getState`, `setState`, `subscribe`, and (with `persist`) a `persist` API — usable from *any* JavaScript, no component or hook required. **Why this is powerful:** WebSocket handlers, route guards, analytics, service workers, plain utility functions, and tests can all read and write the same store the UI uses, keeping one source of truth across React and non-React code.

```ts
// store/counterStore.ts
export const useCounterStore = create<CounterState>()((set) => ({
  count: 0,
  increment: () => set((s) => ({ count: s.count + 1 })),
}))

// --- Anywhere outside React: utilities, event handlers, sockets, etc. ---

// Read the current state snapshot (no subscription).
const count = useCounterStore.getState().count

// Write directly (merge by default, like set inside an action).
useCounterStore.setState({ count: 99 })
useCounterStore.setState((s) => ({ count: s.count + 10 }))

// Call an action defined in the store.
useCounterStore.getState().increment()

// Subscribe to ALL changes; returns an unsubscribe function — always call it on teardown.
const unsub = useCounterStore.subscribe((state) => {
  console.log("count is now", state.count)
})
unsub()
```

### Vanilla stores — no React dependency at all

If you want a store with *no* React coupling (shared logic in a non-React package, a game loop, a CLI), use `createStore` from `zustand/vanilla`. It returns a store object with the same `getState`/`setState`/`subscribe` methods but no hook. You can bind it to React later with the generic `useStore`.

```ts
import { createStore } from "zustand/vanilla"

// Identical initializer; returns a plain store, not a hook.
const counterStore = createStore<CounterState>()((set) => ({
  count: 0,
  increment: () => set((s) => ({ count: s.count + 1 })),
}))

// Use in vanilla JS:
counterStore.getState().increment()
console.log(counterStore.getState().count) // 1

// Bind to React when needed:
import { useStore } from "zustand"
function Counter() {
  const count = useStore(counterStore, (s) => s.count)
  return <p>{count}</p>
}
```

> **Gotcha — leaked subscriptions.** `subscribe` outside React is *not* auto-cleaned the way a component's hook subscription is. Hold the returned `unsub` and call it (on `beforeunload`, socket close, test teardown) or you leak listeners and memory.

---

## 13. Transient Updates

Some state changes *very* frequently — mouse position, scroll offset, drag coordinates, a live cursor in a collaborative app. Routing each change through React's render cycle (selector → `Object.is` → re-render) would thrash the UI at 60+ updates per second. **Transient updates** are the escape hatch: you `subscribe` to the change imperatively and write the value straight to a DOM node or ref, *bypassing React rendering entirely*. **The logic:** React shouldn't re-render for data that only needs to update a single element's style or text; subscribe outside the render path and mutate that element directly.

```tsx
import { useRef, useEffect } from "react"
import { create } from "zustand"

const useMouseStore = create<{ x: number; y: number; set: (x: number, y: number) => void }>()(
  (set) => ({ x: 0, y: 0, set: (x, y) => set({ x, y }) })
)

function Cursor() {
  const ref = useRef<HTMLDivElement>(null)

  useEffect(() => {
    // Subscribe imperatively. The callback writes to the DOM directly —
    // NO setState, NO re-render of this component, even at 100+ updates/sec.
    const unsub = useMouseStore.subscribe((s) => {
      if (ref.current) {
        ref.current.style.transform = `translate(${s.x}px, ${s.y}px)`
      }
    })
    return unsub // clean up the subscription on unmount
  }, [])

  // The component renders ONCE; movement updates the DOM node, not React state.
  return <div ref={ref} className="cursor" />
}
```

> **When to use it:** only for high-frequency values where re-rendering is a measured problem. For ordinary state, the normal selector flow is simpler and correct. Pair transient updates with `subscribeWithSelector` (§10.4) when you want to watch just one high-frequency field.

---

## 14. Zustand with Next.js & SSR

This is the section that trips up production teams, so understand the hazard before the fix.

### The hydration hazard — why the singleton breaks on the server

A Zustand store created with `create` at module scope is a **single shared object for the lifetime of the module**. In the browser that's perfect — one user, one tab, one store. But a Next.js server runs in a **long-lived Node.js process that handles many users' requests concurrently**, all sharing the same loaded modules. If request A mutates the module-level store during render, request B — running in the same process — can read A's data. That's a **cross-request state leak**: one user seeing another user's data. It's a correctness and security bug, not a performance nit.

A second, related problem is **client hydration mismatch**: the server renders HTML from the store's default state, then the browser hydrates. If the client store has *different* initial values (e.g. `persist` just loaded `localStorage`), React's hydration sees server HTML and client render disagree, and warns/breaks.

### The fix — a per-request store created with a factory + Provider

The solution is to stop using a module singleton for SSR and instead **create a fresh store per request**, held in React state/ref and passed down via Context. Each request (each render tree) gets its own isolated store, so no leakage. You also pass server-fetched initial data into the factory, so server and client agree on initial state, killing the hydration mismatch.

```ts
// store/counterStore.ts
import { createStore } from "zustand/vanilla"

export interface CounterState {
  count: number
  increment: () => void
}

// A FACTORY: call it once per request to get a brand-new, isolated store.
export const createCounterStore = (initialCount = 0) =>
  createStore<CounterState>()((set) => ({
    count: initialCount,
    increment: () => set((s) => ({ count: s.count + 1 })),
  }))

export type CounterStore = ReturnType<typeof createCounterStore>
```

```tsx
// store/CounterStoreProvider.tsx — a Client Component holding the per-tree store.
"use client"
import { createContext, useContext, useRef } from "react"
import { useStore } from "zustand"
import { createCounterStore, type CounterStore, type CounterState } from "./counterStore"

const CounterStoreContext = createContext<CounterStore | null>(null)

export function CounterStoreProvider({
  children,
  initialCount = 0, // server can pass fetched data in as the initial state
}: { children: React.ReactNode; initialCount?: number }) {
  // useRef ensures the store is created exactly ONCE per mounted provider —
  // not on every render, and a fresh instance per request tree.
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

// Consume via this hook instead of importing a singleton.
export function useCounterStore<T>(selector: (s: CounterState) => T): T {
  const store = useContext(CounterStoreContext)
  if (!store) throw new Error("useCounterStore must be used within CounterStoreProvider")
  return useStore(store, selector)
}
```

```tsx
// app/layout.tsx — a Server Component: fetch initial data, hand it to the provider.
import { CounterStoreProvider } from "../store/CounterStoreProvider"

export default async function Layout({ children }: { children: React.ReactNode }) {
  const initialCount = 0 // could be fetched from a DB/session here
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
// components/Counter.tsx — a Client Component reading the per-request store.
"use client"
import { useCounterStore } from "../store/CounterStoreProvider"

export function Counter() {
  const count = useCounterStore((s) => s.count)
  const increment = useCounterStore((s) => s.increment)
  return <button onClick={increment}>Count: {count}</button>
}
```

### When you DON'T need the factory

If the store is **purely client-side** and only ever used inside `"use client"` components — a theme toggle, a UI-only cart, a "is this menu open" flag that's never read during SSR — the module singleton is fine, because nothing reads or mutates it on the server. The factory + provider is required when: the store is read/written during server rendering, you need per-request/per-user initial state, or you pass server data into client state. Server Components themselves can never use Zustand hooks (they don't run on the client) — that's by design.

### `persist` + Next.js — defer hydration to avoid mismatch

`persist` reads `localStorage`, which doesn't exist on the server, so server render uses defaults while the client wants the persisted value — a classic hydration mismatch. Use `skipHydration: true` and rehydrate manually after mount, or gate rendering until hydrated.

```ts
// In the store: skip automatic rehydration.
export const useThemeStore = create<ThemeState>()(
  persist(
    (set) => ({
      theme: "light",
      toggleTheme: () => set((s) => ({ theme: s.theme === "light" ? "dark" : "light" })),
    }),
    { name: "theme", skipHydration: true }
  )
)
```

```tsx
// Rehydrate after mount so server and first client render agree (both use defaults),
// then the persisted value applies on the client only.
"use client"
import { useEffect } from "react"
import { useThemeStore } from "../store/themeStore"

export function ThemeHydration() {
  useEffect(() => { useThemeStore.persist.rehydrate() }, [])
  return null
}
```

```tsx
// Alternative: render nothing until hydrated, avoiding any mismatch in the markup.
function ThemeAware() {
  const [hydrated, setHydrated] = useState(false)
  useEffect(() => {
    // onFinishHydration fires once rehydration completes.
    const unsub = useThemeStore.persist.onFinishHydration(() => setHydrated(true))
    useThemeStore.persist.rehydrate()
    return unsub
  }, [])
  const theme = useThemeStore((s) => s.theme)
  if (!hydrated) return null
  return <div data-theme={theme}>…</div>
}
```

> **Best practice:** for anything read on the server or seeded with server data, use the factory + provider. For persisted client-only state, use `skipHydration` + manual rehydrate. Don't mix the two casually — pick the pattern that matches whether the server touches the data.

---

## 15. Zustand vs Context vs Redux vs TanStack Query

Choosing tools well is most of the battle. The first table contrasts the *client-state* options; the second draws the all-important *client vs server* line.

### Client-state libraries compared

| Dimension | **Zustand** | **Redux Toolkit** | **Jotai** | **React Context** |
|---|---|---|---|---|
| Bundle size | ~1 KB | ~12 KB | ~3 KB | 0 (built-in) |
| Boilerplate | Minimal | Moderate (slices, actions) | Minimal | Low |
| Provider required | No (except SSR) | Yes | No (Provider optional) | Yes |
| Re-render model | Selector subscription | Selector subscription | Atom subscription | All consumers on any change |
| DevTools | Redux DevTools (middleware) | Redux DevTools (built-in) | Jotai DevTools | React DevTools only |
| Async | Plain async functions | `createAsyncThunk` | Async atoms | Manual |
| TypeScript | Excellent | Excellent | Excellent | Good |
| Outside React | Yes (`getState`/`setState`) | Yes | Limited | No |
| SSR safety | Factory pattern (§14) | Per-request store | Per-request Provider | Per-request Provider |
| Best for | Global client/UI state | Large teams, strict patterns | Fine-grained atomic state | Low-churn config to a subtree |

**Reading the table:** Context loses on re-render granularity (every consumer re-renders on any change), so it's best for values that rarely change (theme tokens, locale, an auth-status flag). Redux Toolkit matches Zustand on the re-render model but costs more ceremony — worth it for big teams enforcing uniform patterns or migrating an existing Redux codebase. Jotai's atom model suits apps that compose state bottom-up. Zustand is the sweet spot for "shared client state, minimal fuss."

### Client state vs server state — when to use which

This is the decision that most affects architecture. **Use Zustand for state your client owns; use TanStack Query for state the server owns.**

| | **Zustand (client state)** | **TanStack Query (server state)** |
|---|---|---|
| Owns | UI/app state the user decides | Cached copy of server data |
| Caching | Manual | Automatic, keyed, with stale-time |
| Background refetch | Manual | Automatic (on focus, interval, reconnect) |
| Loading / error states | You write them | Built-in (`isLoading`, `isError`, …) |
| Deduplication | Manual | Automatic by query key |
| Invalidation | Manual | `invalidateQueries`, targeted |
| Example data | Selected tab, modal open, cart draft, theme | Posts list, user profile, search results |

**The composition pattern in practice:** Query fetches and caches the posts; Zustand holds *which* post is selected and the filter/sort UI. They never overlap — and when they're tempted to (e.g. you want to "store" the fetched list in Zustand), that's your signal you're recreating Query and should let Query own it.

```tsx
// Server state via Query; client state via Zustand. Clean boundary.
function Dashboard() {
  // Server data — cached, refetched, deduped by Query.
  const { data: posts, isLoading } = useQuery({ queryKey: ["posts"], queryFn: fetchPosts })
  // Client/UI state — owned by the user, held in Zustand.
  const selectedId = useUIStore((s) => s.selectedPostId)
  const setSelected = useUIStore((s) => s.setSelectedPostId)

  if (isLoading) return <Spinner />
  return <PostList posts={posts} selectedId={selectedId} onSelect={setSelected} />
}
```

---

## 16. Testing Stores

Zustand stores are highly testable because the logic is plain functions over a plain object. **The golden rule:** isolate state between tests so one test can't pollute the next. There are two complementary strategies — test the store directly via a fresh instance, and test components by resetting the singleton in `beforeEach`.

### Testing store logic directly (preferred for unit tests)

Extract the *initializer* (or a creator function) so each test can spin up a brand-new store with `createStore` — no shared state, no cleanup needed.

```ts
// store/__tests__/counterStore.test.ts
import { describe, it, expect, beforeEach } from "vitest"
import { createStore } from "zustand/vanilla"
import type { CounterState } from "../counterStore"

// A creator so every test gets a fresh, isolated store.
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
  beforeEach(() => { store = createCounterStore() }) // fresh per test

  it("starts at 0", () => {
    expect(store.getState().count).toBe(0)
  })
  it("increments", () => {
    store.getState().increment()
    expect(store.getState().count).toBe(1)
  })
  it("resets", () => {
    store.getState().increment()
    store.getState().reset()
    expect(store.getState().count).toBe(0)
  })
})
```

### Testing async actions (mock the network)

```ts
// store/__tests__/postsStore.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest"
import { createStore } from "zustand/vanilla"
import type { PostsState } from "../postsStore"

global.fetch = vi.fn() // stub the network

const createPostsStore = () =>
  createStore<PostsState>()((set, get) => ({
    posts: [], isLoading: false, error: null,
    fetchPosts: async () => {
      if (get().isLoading) return
      set({ isLoading: true, error: null })
      try {
        const res = await fetch("/api/posts")
        set({ posts: await res.json(), isLoading: false })
      } catch (err) {
        set({ error: (err as Error).message, isLoading: false })
      }
    },
  }))

describe("postsStore async", () => {
  let store: ReturnType<typeof createPostsStore>
  beforeEach(() => { store = createPostsStore(); vi.clearAllMocks() })

  it("loads posts on success", async () => {
    const mock = [{ id: 1, title: "Hi" }]
    ;(fetch as ReturnType<typeof vi.fn>).mockResolvedValueOnce({ ok: true, json: async () => mock })
    await store.getState().fetchPosts()
    expect(store.getState().posts).toEqual(mock)
    expect(store.getState().isLoading).toBe(false)
  })

  it("records errors and clears loading", async () => {
    ;(fetch as ReturnType<typeof vi.fn>).mockRejectedValueOnce(new Error("Network error"))
    await store.getState().fetchPosts()
    expect(store.getState().error).toBe("Network error")
    expect(store.getState().isLoading).toBe(false)
  })
})
```

### Testing components that use a singleton store

For components bound to a module-singleton hook, reset that singleton to a known state before each test with `setState`. Use the replace flag carefully (it wipes actions — see §6), so usually merge a known *data* snapshot.

```tsx
// components/__tests__/Counter.test.tsx
import { render, screen, fireEvent } from "@testing-library/react"
import { describe, it, expect, beforeEach } from "vitest"
import { Counter } from "../Counter"
import { useCounterStore } from "../../store/counterStore"

describe("Counter", () => {
  // Reset the singleton's DATA before each test (merge, so actions survive).
  beforeEach(() => { useCounterStore.setState({ count: 0 }) })

  it("renders the count", () => {
    render(<Counter />)
    expect(screen.getByText(/0/)).toBeInTheDocument()
  })
  it("increments on click", () => {
    render(<Counter />)
    fireEvent.click(screen.getByRole("button"))
    expect(screen.getByText(/1/)).toBeInTheDocument()
  })
  it("can be seeded", () => {
    useCounterStore.setState({ count: 42 })
    render(<Counter />)
    expect(screen.getByText(/42/)).toBeInTheDocument()
  })
})
```

> **Gotcha — Jest auto-mock of Zustand.** A common community setup auto-mocks Zustand so that `create` resets every store between tests automatically (via a `__mocks__/zustand.ts` file). If your tests leak state across files, that mock — or the explicit `setState` reset above — is what you're missing.

---

## 17. Best Practices, Tips & Gotchas

A consolidated checklist. Each item is a real mistake people make; the fix follows.

### 1. Never select the whole store without a reason

```ts
const all = useStore()              // ❌ re-renders on every change anywhere
const count = useStore((s) => s.count) // ✅ re-renders only when count changes
```

### 2. Selectors returning fresh objects/arrays re-render every time

```ts
// ❌ new array each call → re-renders on every store update
const open = useStore((s) => s.todos.filter((t) => !t.done))

// ✅ A: useShallow shallow-compares the result
import { useShallow } from "zustand/react/shallow"
const open = useStore(useShallow((s) => s.todos.filter((t) => !t.done)))

// ✅ B: select the raw input, memoize the derivation
const todos = useStore((s) => s.todos)
const open2 = useMemo(() => todos.filter((t) => !t.done), [todos])
```

### 3. Actions are stable — safe to omit from / include in deps arrays

```ts
const fetchPosts = useStore((s) => s.fetchPosts)
useEffect(() => { fetchPosts() }, [fetchPosts]) // stable identity → runs once
```

### 4. Define stores at module scope, never inside a component

```tsx
// ❌ a new store every render → all state lost on each render
function Bad() { const s = create(() => ({ n: 0 })); /* ... */ }

// ✅ module-level singleton (or the SSR factory + ref pattern from §14)
const useStore = create(() => ({ n: 0 }))
function Good() { const n = useStore((s) => s.n) }
```

### 5. Keep logic in actions, not components

```ts
// ✅ the rule lives in the store; components just call it
const useStore = create((set) => ({
  count: 0,
  increment: () => set((s) => ({ count: s.count + 1 })),
}))
```

### 6. Spread every nested level (or use Immer)

```ts
updateBio: (bio: string) =>
  set((s) => ({
    // ✅ spread at EACH level; forgetting one level silently drops siblings
    user: { ...s.user, profile: { ...s.user.profile, bio } },
  }))
// Or add the immer middleware and write s.user.profile.bio = bio
```

### 7. Middleware order: `devtools(persist(immer(...)))`

```ts
// ✅ immer innermost so persist never serializes a draft
create()(devtools(persist(immer(fn), opts), { name: "X" }))
```

### 8. `partialize` away transient and sensitive state from `persist`

```ts
persist(fn, {
  name: "app",
  // ✅ persist only what should survive a refresh; never tokens or loading flags
  partialize: (s) => ({ theme: s.theme, cart: s.cart }),
})
```

### 9. Reset stores between tests

```ts
beforeEach(() => {
  // merge a known data snapshot (replace=true would also wipe your actions, §6)
  useStore.setState(initialData)
})
```

### 10. Use the SSR factory whenever the server touches the store

A module singleton on a Next.js server leaks one request's state into another's. If the store is read/written during SSR or seeded with server data, use the per-request factory + provider from §14. Purely client-only stores can stay singletons.

### 11. Don't put server state in Zustand

```ts
// ❌ reinventing TanStack Query inside a store
const usePosts = create((set) => ({ posts: [], loading: false, fetch: async () => {/*…*/} }))

// ✅ Query owns server data; Zustand owns the UI around it
const { data: posts } = useQuery({ queryKey: ["posts"], queryFn: fetchPosts })
const selectedId = useUIStore((s) => s.selectedPostId)
```

### 12. Always clean up non-React subscriptions

```ts
const unsub = useStore.subscribe(listener)
// …later, on teardown:
unsub() // forgetting this leaks listeners (§12, §13)
```

---

## 18. Study Path & Build-to-Learn Projects

Work through these in order; each phase builds on the last. The goal is not to read but to *build* — type every example, then extend it.

### Phase 1 — Core API (Day 1–2) [B]

1. Build a **counter**: `create`, `set` (object + functional forms), `get`, narrow selectors.
2. Add a `step` field and `incrementByStep` using `get()`.
3. Open the React DevTools Profiler and confirm only the subscribed component re-renders. Deliberately make the "select the whole store" mistake and watch the extra re-renders to feel the difference.

### Phase 2 — Real state shapes (Day 3–4) [B/I]

4. Build a **todo app** (add, toggle, delete, filter) with a typed `create<TodoState>()`.
5. Add a component that selects `{ todos, filter }` together; first without, then with `useShallow`, observing the re-render change.
6. Write unit tests for the store with `createStore` from `zustand/vanilla`.

### Phase 3 — Middleware (Day 5–6) [I]

7. Add `persist` so todos survive a refresh; `partialize` to skip a transient filter.
8. Add `devtools` and explore time-travel.
9. Refactor nested updates with `immer`; compare before/after readability.
10. Add `subscribeWithSelector` to sync `document.title` with the open-todo count.

### Phase 4 — Architecture (Day 7–9) [A]

11. Build a **shopping cart** with three slices: `cartSlice`, `userSlice`, `uiSlice`.
12. Combine them into `useBoundStore` (slices pattern), with a cross-slice action that clears the cart on logout.
13. Read the cart total from a non-React analytics module via `getState`/`subscribe`, cleaning up the subscription.

### Phase 5 — SSR & Next.js (Day 10–11) [A]

14. Scaffold a Next.js App Router app.
15. Implement the per-request store factory + provider (§14); pass server-fetched initial data through the provider's props.
16. Add a persisted theme store and handle the hydration hazard with `skipHydration` + manual rehydrate.

### Phase 6 — Client + server together (Day 12–14) [I/A]

17. Build a **blog dashboard**: TanStack Query fetches posts; Zustand holds `selectedPostId` + filters.
18. Prove the boundary — verify nothing duplicates server data into Zustand.
19. Write integration tests (`@testing-library/react`) for a component using both.

### Build-to-learn projects

| Project | Concepts practiced |
|---|---|
| **Counter / Todo** | `create`, `set`, `get`, selectors, TypeScript |
| **Shopping Cart** | Arrays in state, slices, `persist`, `devtools` |
| **Auth Flow** | Async actions, `user` state, `logout`, `persist`, cross-slice |
| **Theme Switcher** | `persist`, SSR hydration, Next.js integration |
| **Dashboard + Data** | Zustand (UI) + TanStack Query (server) boundary |
| **Kanban Board** | Immer for nested mutations, complex shapes |
| **Live Cursor / Realtime** | `subscribeWithSelector`, transient updates, vanilla store, WebSocket outside React |

### Key resources

- **Docs:** docs.pmnd.rs/zustand
- **GitHub:** github.com/pmndrs/zustand (the `/examples` and `/docs/guides` folders cover most patterns)
- **Companion guides in this library:** `TANSTACK_QUERY_GUIDE.md` (server state), `REACT_GUIDE.md` (React 19), `NEXTJS_GUIDE.md` (App Router & SSR)

---

*Zustand v5 · React 19 · Next.js 15 · Last updated June 2026*
