# React 19 Complete Reference Guide

A full, offline-study reference for **React 19** — the stable library release that landed in late 2024 and matured through 2025–2026. Covers every core concept, every new API, performance patterns, migration from React 18, and a structured learning path you can follow without internet access.

> **Version note:** This guide targets **React 19 (stable)**. All React 19 APIs described here are stable and shipped as of 2026. Where an API is brand-new in React 19, it is marked **🆕 React 19**. Where something is still evolving or framework-dependent, it is flagged **⚡ Version note**. The guide uses **TypeScript** (TSX) throughout — the recommended default for all new projects.

---

## Table of Contents

1. [What React Is & React 19 Overview](#1-what-react-is--react-19-overview)
2. [Fundamentals Refresher](#2-fundamentals-refresher)
3. [State & Core Hooks](#3-state--core-hooks)
4. [🆕 The `use()` API](#4--the-use-api)
5. [🆕 Actions & Form Actions](#5--actions--form-actions)
6. [🆕 useActionState](#6--useactionstate)
7. [🆕 useFormStatus](#7--useformstatus)
8. [🆕 useOptimistic](#8--useoptimistic)
9. [🆕 ref as a Prop — No More forwardRef](#9--ref-as-a-prop--no-more-forwardref)
10. [🆕 Context as a Provider — No More .Provider](#10--context-as-a-provider--no-more-provider)
11. [🆕 Document Metadata & Resource Preloading](#11--document-metadata--resource-preloading)
12. [Concurrent Features](#12-concurrent-features)
13. [Server Components & Server Actions](#13-server-components--server-actions)
14. [Error Handling](#14-error-handling)
15. [The React Compiler](#15-the-react-compiler)
16. [Refs, Portals & Lazy Loading](#16-refs-portals--lazy-loading)
17. [Performance Best Practices & Common Pitfalls](#17-performance-best-practices--common-pitfalls)
18. [Migrating from React 18 to 19](#18-migrating-from-react-18-to-19)
19. [Tips, Tricks & Gotchas](#19-tips-tricks--gotchas)
20. [Study Path](#20-study-path)

---

## 1. What React Is & React 19 Overview

React is a **JavaScript library for building user interfaces**. It is maintained by Meta and the open-source community. React's core idea is simple: **describe what your UI should look like for a given state, and React figures out how to update the DOM efficiently.**

React does NOT include:
- A router (use React Router, TanStack Router, or Next.js App Router)
- A server (use Next.js, Remix, or a custom Node setup)
- A global state manager (use Zustand, Jotai, Redux Toolkit, or built-in context/useReducer)
- A build system (use Vite, Next.js, or Parcel)

### React 19 — What Changed at a Glance

React 19 is the **largest API surface expansion since React 16 (Hooks)**. It shipped as stable in December 2024 and represents the culmination of the "Concurrent React" era.

| Area | React 18 | React 19 |
|---|---|---|
| Form handling | Manual state + fetch | Native `<form action={fn}>` with async actions |
| Loading state | Manual `isLoading` flag | `useActionState`, `useFormStatus` |
| Optimistic UI | Roll your own | `useOptimistic` built-in |
| Reading promises | Suspense only at boundary | `use(promise)` anywhere in render |
| Reading context | `useContext(Ctx)` only | `use(Ctx)` — can be called conditionally |
| ref forwarding | `forwardRef()` wrapper required | `ref` is a plain prop on function components |
| Context provider | `<MyCtx.Provider value={v}>` | `<MyCtx value={v}>` (shorter) |
| Document head | Third-party (react-helmet) | `<title>`, `<meta>`, `<link>` work natively anywhere |
| Resource hints | Framework-only | `preload`, `preinit`, `prefetchDNS`, `preconnect` built-in |
| Error reporting | `onRecoverableError` option only | `onCaughtError`, `onUncaughtError`, `onRecoverableError` |
| Auto memoization | Manual `useMemo`/`useCallback` | React Compiler handles it (opt-in, 2025+) |
| `forwardRef` | Required for ref in function components | Deprecated — just use `ref` prop |
| `propTypes` | Runtime prop checking | Removed from React core (use TypeScript) |
| String refs | Deprecated long ago | Removed entirely |
| Legacy context (`contextTypes`) | Deprecated | Removed entirely |
| ReactDOM.render | Deprecated | Removed — use `createRoot` |
| `act` in tests | `react-dom/test-utils` | Moved to `react` package directly |

### Installing React 19

```bash
# New project with Vite (recommended for SPAs)
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install

# Or upgrade an existing project
npm install react@19 react-dom@19

# Check your version
npm list react
```

```tsx
// main.tsx — React 19 entry point
import { StrictMode } from "react";
import { createRoot } from "react-dom/client"; // createRoot is the ONLY way in React 19
import App from "./App";
import "./index.css";

// createRoot replaced ReactDOM.render (which was removed in React 19)
createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

> **⚡ Version note:** `ReactDOM.render()` was deprecated in React 18 and is **fully removed in React 19**. If you upgrade an older project, every render call must be migrated to `createRoot`.

---

## 2. Fundamentals Refresher

### Components

A React component is a **function that returns JSX** (or `null`). Components must start with a capital letter.

```tsx
// Functional component — the only style you should write in 2026
// Props are typed with an interface or type alias
interface GreetingProps {
  name: string;
  age?: number; // optional prop
}

export function Greeting({ name, age = 0 }: GreetingProps) {
  return (
    <div className="greeting">
      <h1>Hello, {name}!</h1>
      {age > 0 && <p>You are {age} years old.</p>}
    </div>
  );
}

// Usage
<Greeting name="Alice" age={30} />
<Greeting name="Bob" />          // age defaults to 0, paragraph omitted
```

### JSX Rules

```tsx
// 1. Return a single root element — or use a Fragment
function MultiReturn() {
  return (
    <>                      {/* Fragment — compiles to nothing in the DOM */}
      <h1>Title</h1>
      <p>Paragraph</p>
    </>
  );
}

// 2. All tags must be closed (including void elements)
// WRONG:  <input>  <br>  <img src="x">
// RIGHT:  <input /> <br /> <img src="x" />

// 3. Attributes use camelCase (HTML → JSX mapping)
// class → className
// for → htmlFor
// onclick → onClick
// tabindex → tabIndex
// stroke-width → strokeWidth (SVG)

// 4. JavaScript expressions go inside { }
const items = ["apples", "bananas", "cherries"];
function ExpressionDemo() {
  const now = new Date().getFullYear();
  return (
    <p>
      Year: {now} | Items: {items.length} | Greeting: {"hello".toUpperCase()}
    </p>
  );
}

// 5. Inline styles take an OBJECT, not a string
<div style={{ color: "red", fontSize: "16px" }}>Styled</div>
```

### Props

```tsx
// Props flow ONE way: parent → child
// They are READ-ONLY inside the child component

// Spreading props (forward all to a DOM element)
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: "primary" | "secondary";
}

function Button({ variant = "primary", children, ...rest }: ButtonProps) {
  return (
    <button
      className={`btn btn-${variant}`}
      {...rest} // forwards onClick, disabled, type, etc.
    >
      {children}
    </button>
  );
}

// Children is a special prop — anything between opening/closing tags
<Button onClick={() => alert("hi")} disabled={false}>
  Click Me
</Button>
```

### Conditional Rendering

```tsx
interface UserProps {
  isLoggedIn: boolean;
  role?: "admin" | "user";
}

function UserPanel({ isLoggedIn, role }: UserProps) {
  // 1. if/else (most readable for complex logic)
  if (!isLoggedIn) {
    return <p>Please log in.</p>;
  }

  return (
    <div>
      {/* 2. Ternary — good for simple inline if/else */}
      <p>{role === "admin" ? "Admin Dashboard" : "User Dashboard"}</p>

      {/* 3. && short-circuit — renders right side only when left is truthy */}
      {role === "admin" && <button>Delete All Users</button>}

      {/* ⚠️  GOTCHA: 0 is falsy but still renders as "0" in JSX!
           Use explicit boolean: {items.length > 0 && <List />}  */}
      {/* WRONG:  {items.length && <List />}  -- renders "0" when empty */}
    </div>
  );
}
```

### List Rendering & Keys

```tsx
interface Product {
  id: number;
  name: string;
  price: number;
}

const products: Product[] = [
  { id: 1, name: "Widget", price: 9.99 },
  { id: 2, name: "Gadget", price: 24.99 },
  { id: 3, name: "Doohickey", price: 4.99 },
];

function ProductList() {
  return (
    <ul>
      {products.map((product) => (
        // KEY is required — must be stable, unique among siblings
        // Keys help React identify which items changed/moved/were added/removed
        // ⚠️ NEVER use the array index as a key when the list can reorder or filter
        <li key={product.id}>
          {product.name} — ${product.price.toFixed(2)}
        </li>
      ))}
    </ul>
  );
}

// When you MUST render a list of fragments with keys:
function FragmentList({ items }: { items: { id: number; name: string; desc: string }[] }) {
  return (
    <dl>
      {items.map((item) => (
        // Use explicit <Fragment> (not the <> shorthand) when you need a key
        <Fragment key={item.id}>
          <dt>{item.name}</dt>
          <dd>{item.desc}</dd>
        </Fragment>
      ))}
    </dl>
  );
}
```

### Event Handling

```tsx
import { type MouseEvent, type FormEvent, type ChangeEvent } from "react";

function EventDemo() {
  // React events are SyntheticEvents — cross-browser wrappers around native events
  function handleClick(e: MouseEvent<HTMLButtonElement>) {
    e.preventDefault(); // works just like native
    console.log("Button clicked at", e.clientX, e.clientY);
  }

  function handleSubmit(e: FormEvent<HTMLFormElement>) {
    e.preventDefault(); // stop full-page reload
    const form = e.currentTarget;
    const data = new FormData(form);
    console.log(data.get("username"));
  }

  function handleChange(e: ChangeEvent<HTMLInputElement>) {
    console.log("Input value:", e.target.value);
  }

  return (
    <>
      <button onClick={handleClick}>Click me</button>

      {/* Inline arrow function — OK for simple cases, but creates a new function each render */}
      <button onClick={() => console.log("inline")}>Inline</button>

      <form onSubmit={handleSubmit}>
        <input name="username" onChange={handleChange} />
        <button type="submit">Submit</button>
      </form>
    </>
  );
}
```

---

## 3. State & Core Hooks

Hooks are functions that start with `use`. They can only be called at the **top level** of a function component or custom hook — never inside loops, conditions, or nested functions.

### useState

```tsx
import { useState } from "react";

// Basic counter
function Counter() {
  // useState returns [currentValue, setter]
  // TypeScript infers the type from the initial value
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Count: {count}</p>
      {/* Always use the updater function form when next state depends on previous */}
      <button onClick={() => setCount((prev) => prev + 1)}>+</button>
      <button onClick={() => setCount((prev) => prev - 1)}>-</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}

// Object state — always spread to avoid losing other fields
interface UserForm {
  name: string;
  email: string;
  age: number;
}

function UserFormDemo() {
  const [form, setForm] = useState<UserForm>({ name: "", email: "", age: 0 });

  function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    setForm((prev) => ({
      ...prev,                        // keep existing fields
      [e.target.name]: e.target.value // update the changed field
    }));
  }

  return (
    <form>
      <input name="name"  value={form.name}  onChange={handleChange} />
      <input name="email" value={form.email} onChange={handleChange} />
    </form>
  );
}

// Lazy initialization — pass a FUNCTION to avoid expensive computation on every render
function ExpensiveInit() {
  // computeInitialState() runs only ONCE, not on every re-render
  const [data, setData] = useState(() => computeInitialState());
  return <div>{JSON.stringify(data)}</div>;
}

function computeInitialState() {
  // Imagine this is reading from localStorage or doing a complex calculation
  return JSON.parse(localStorage.getItem("savedData") ?? "null") ?? {};
}
```

### useEffect

```tsx
import { useEffect, useState } from "react";

// useEffect runs AFTER the browser paints
// Dependency array controls WHEN it runs:
//   [] → once after first render (mount)
//   [a, b] → after first render AND whenever a or b changes
//   omitted → after EVERY render (rare, almost always a mistake)

function DataFetcher({ userId }: { userId: number }) {
  const [user, setUser] = useState<{ name: string } | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    let cancelled = false; // prevent stale state updates after unmount/re-run

    async function fetchUser() {
      setLoading(true);
      setError(null);
      try {
        const res = await fetch(`/api/users/${userId}`);
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const data = await res.json();
        if (!cancelled) setUser(data); // only update if still relevant
      } catch (err) {
        if (!cancelled) setError(err as Error);
      } finally {
        if (!cancelled) setLoading(false);
      }
    }

    fetchUser();

    // Cleanup function — runs before the next effect OR on unmount
    return () => {
      cancelled = true;
    };
  }, [userId]); // Re-run when userId changes

  if (loading) return <p>Loading...</p>;
  if (error)   return <p>Error: {error.message}</p>;
  return <p>User: {user?.name}</p>;
}

// --- COMMON EFFECT PATTERNS ---

// 1. Subscribing to external store / events
function WindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    function handleResize() { setWidth(window.innerWidth); }
    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize); // CLEANUP!
  }, []); // no deps — add listener once, remove on unmount

  return <p>Width: {width}px</p>;
}

// 2. Synchronizing with a third-party library
function MapWidget({ center }: { center: [number, number] }) {
  const mapRef = useRef<HTMLDivElement>(null);
  const mapInstance = useRef<SomeMapLib | null>(null);

  useEffect(() => {
    if (!mapRef.current) return;
    // Create the map on mount
    mapInstance.current = new SomeMapLib(mapRef.current, { center });
    return () => {
      // Destroy on unmount
      mapInstance.current?.destroy();
    };
  }, []); // Only on mount

  useEffect(() => {
    // Update center when prop changes
    mapInstance.current?.setCenter(center);
  }, [center]);

  return <div ref={mapRef} style={{ width: "100%", height: "400px" }} />;
}

// ⚠️  DEPENDENCY PITFALLS:
// 1. Missing a dep → stale closure bug (lint rule exhaustive-deps catches this)
// 2. Object/array in deps → always "changed" (use useMemo or primitives)
// 3. Functions in deps → always "changed" (use useCallback)
```

### useContext

```tsx
import { createContext, useContext, useState, type ReactNode } from "react";

// 1. Define the shape
interface ThemeContextType {
  theme: "light" | "dark";
  toggleTheme: () => void;
}

// 2. Create context with a sensible default (useful for tests without a provider)
const ThemeContext = createContext<ThemeContextType>({
  theme: "light",
  toggleTheme: () => {},
});

// 3. Provider component
export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<"light" | "dark">("light");

  function toggleTheme() {
    setTheme((t) => (t === "light" ? "dark" : "light"));
  }

  return (
    // 🆕 React 19: you can write <ThemeContext value={...}> instead of <ThemeContext.Provider value={...}>
    // See section 10 for details. Both syntaxes work.
    <ThemeContext value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext>
  );
}

// 4. Custom hook — encapsulates useContext + error boundary check
export function useTheme() {
  const ctx = useContext(ThemeContext);
  // Optional safety check (only needed if your default context doesn't make sense)
  return ctx;
}

// 5. Consumer
function ThemedButton() {
  const { theme, toggleTheme } = useTheme();
  return (
    <button
      className={theme === "dark" ? "btn-dark" : "btn-light"}
      onClick={toggleTheme}
    >
      Toggle Theme (currently {theme})
    </button>
  );
}
```

### useRef

```tsx
import { useRef, useEffect } from "react";

// useRef is a "box" that holds a mutable value WITHOUT triggering a re-render
// Two main uses: (1) accessing DOM elements, (2) storing mutable values

// --- USE CASE 1: DOM access ---
function AutoFocusInput() {
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    // After mount, focus the input
    inputRef.current?.focus();
  }, []);

  return <input ref={inputRef} placeholder="I'm auto-focused" />;
}

// --- USE CASE 2: Storing a mutable value across renders ---
function IntervalCounter() {
  const [count, setCount] = useState(0);
  // Store the interval ID so we can clear it on unmount
  const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null);

  useEffect(() => {
    intervalRef.current = setInterval(() => {
      setCount((c) => c + 1);
    }, 1000);

    return () => {
      if (intervalRef.current) clearInterval(intervalRef.current);
    };
  }, []);

  return <p>Seconds: {count}</p>;
}

// --- USE CASE 3: Previous value ---
function usePrevious<T>(value: T): T | undefined {
  const prevRef = useRef<T | undefined>(undefined);

  useEffect(() => {
    prevRef.current = value; // updates AFTER render
  });

  return prevRef.current; // returns value from PREVIOUS render
}
```

### useReducer

```tsx
import { useReducer } from "react";

// useReducer is best when:
// - State transitions are complex / have multiple sub-values
// - Next state depends on previous state in multiple ways
// - You want to move state logic out of the component

type Action =
  | { type: "increment" }
  | { type: "decrement" }
  | { type: "reset" }
  | { type: "set"; payload: number };

interface CounterState {
  count: number;
  history: number[];
}

// Reducer is a PURE function — no side effects, no async
function counterReducer(state: CounterState, action: Action): CounterState {
  switch (action.type) {
    case "increment":
      return { count: state.count + 1, history: [...state.history, state.count + 1] };
    case "decrement":
      return { count: state.count - 1, history: [...state.history, state.count - 1] };
    case "reset":
      return { count: 0, history: [] };
    case "set":
      return { count: action.payload, history: [...state.history, action.payload] };
    default:
      return state; // always return state for unknown actions
  }
}

function ReducerCounter() {
  const [state, dispatch] = useReducer(counterReducer, { count: 0, history: [] });

  return (
    <div>
      <p>Count: {state.count}</p>
      <p>History: {state.history.join(", ")}</p>
      <button onClick={() => dispatch({ type: "increment" })}>+</button>
      <button onClick={() => dispatch({ type: "decrement" })}>-</button>
      <button onClick={() => dispatch({ type: "set", payload: 100 })}>Set 100</button>
      <button onClick={() => dispatch({ type: "reset" })}>Reset</button>
    </div>
  );
}
```

### useMemo

```tsx
import { useMemo, useState } from "react";

// useMemo caches the result of an expensive calculation
// It recalculates ONLY when its dependencies change

function ExpensiveComponent({ numbers, filter }: { numbers: number[]; filter: number }) {
  const [unrelated, setUnrelated] = useState(0);

  // WITHOUT useMemo: runs on every render, even when 'unrelated' changes
  // WITH useMemo: only runs when numbers or filter changes
  const filtered = useMemo(() => {
    console.log("Computing filtered list..."); // expensive operation
    return numbers.filter((n) => n > filter).sort((a, b) => a - b);
  }, [numbers, filter]); // deps: recalculate when these change

  return (
    <div>
      <p>Unrelated state: {unrelated}</p>
      <button onClick={() => setUnrelated((n) => n + 1)}>Re-render</button>
      <ul>{filtered.map((n) => <li key={n}>{n}</li>)}</ul>
    </div>
  );
}

// ⚡ Version note: With the React Compiler (section 15), you often DON'T need
// to write useMemo manually. The compiler inserts it for you. But it's still
// important to understand it for React 19 projects not using the compiler.
```

### useCallback

```tsx
import { useCallback, useState, memo } from "react";

// useCallback caches a FUNCTION reference
// Without it, a new function is created on every parent render,
// causing memoized children to re-render unnecessarily

// Memoized child — only re-renders when its props change
const ExpensiveChild = memo(function ExpensiveChild({
  label,
  onClick,
}: {
  label: string;
  onClick: () => void;
}) {
  console.log(`Rendering ${label}`);
  return <button onClick={onClick}>{label}</button>;
});

function Parent() {
  const [countA, setCountA] = useState(0);
  const [countB, setCountB] = useState(0);

  // Without useCallback: new function on every render → ExpensiveChild always re-renders
  // With useCallback: same function reference → ExpensiveChild only re-renders when countA changes
  const handleA = useCallback(() => {
    console.log("A clicked, countA =", countA);
    setCountA((c) => c + 1);
  }, [countA]); // re-created only when countA changes

  const handleB = useCallback(() => {
    setCountB((c) => c + 1);
  }, []); // never re-created (no deps)

  return (
    <div>
      <ExpensiveChild label={`A: ${countA}`} onClick={handleA} />
      <ExpensiveChild label={`B: ${countB}`} onClick={handleB} />
    </div>
  );
}
```

---

## 4. 🆕 The `use()` API

`use()` is a new React 19 API that can read the value of a **Promise** or a **Context** inside a component's render function. Unlike other hooks, `use()` **can be called inside loops, conditions, and early returns** — making it more flexible.

### Reading a Promise with `use()`

```tsx
// 🆕 React 19
import { use, Suspense } from "react";

// The component that reads the promise MUST be wrapped in <Suspense>
// While the promise is pending, the nearest Suspense boundary shows its fallback

async function fetchUser(id: number): Promise<{ name: string; email: string }> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error("Failed to fetch user");
  return res.json();
}

// Create the promise OUTSIDE the component (or pass it as a prop)
// ⚠️ Do NOT create the promise inside the component body — it creates a new promise every render
const userPromise = fetchUser(1);

function UserProfile() {
  // use() suspends this component until userPromise resolves
  // If the promise rejects, the nearest error boundary catches it
  const user = use(userPromise);

  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}

// Parent wraps it in Suspense
function App() {
  return (
    <Suspense fallback={<p>Loading user...</p>}>
      <UserProfile />
    </Suspense>
  );
}
```

### Passing Promises as Props (the real pattern)

```tsx
// 🆕 React 19 — the idiomatic "async RSC → client" pattern
// (especially used with React Server Components + frameworks like Next.js)

// Server Component (or a parent that creates the promise):
async function UserPage({ userId }: { userId: number }) {
  // Start the fetch here — don't await, pass the promise down
  const userPromise = fetchUser(userId);

  return (
    <Suspense fallback={<Skeleton />}>
      <UserCard promise={userPromise} />
    </Suspense>
  );
}

// Client Component that reads it:
function UserCard({ promise }: { promise: Promise<{ name: string; email: string }> }) {
  const user = use(promise); // suspends until resolved

  return <div className="card">{user.name}</div>;
}
```

### Reading Context Conditionally with `use()`

```tsx
// 🆕 React 19 — use() can read context inside conditions
// This is NOT possible with useContext (which must always be at top level)

import { use, createContext } from "react";

const ThemeContext = createContext<"light" | "dark">("light");

function ConditionalComponent({ showTheme }: { showTheme: boolean }) {
  // ✅ Valid in React 19 — use() inside a condition
  if (showTheme) {
    const theme = use(ThemeContext);
    return <p>Current theme: {theme}</p>;
  }

  return <p>Theme hidden</p>;
}

// useContext still works and is preferred for unconditional reads
// use(Context) is for when you NEED conditional reading
```

---

## 5. 🆕 Actions & Form Actions

React 19 introduces **Actions** — a first-class pattern for handling mutations (form submissions, data changes). Instead of manually tracking loading states, you pass an `async` function directly to a form or transition.

### `<form action={asyncFunction}>` — the New Native Pattern

```tsx
// 🆕 React 19 — async function passed to <form action>
// React automatically handles pending state, error boundaries, and optimistic updates

import { startTransition } from "react";

// The "action" function receives a FormData object
async function createPost(formData: FormData) {
  "use server"; // only needed in RSC/Next.js — marks this as a Server Action
  const title = formData.get("title") as string;
  const body = formData.get("body") as string;

  await fetch("/api/posts", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ title, body }),
  });

  // Can throw to trigger error handling
}

function CreatePostForm() {
  return (
    // 🆕 React 19: pass an async function directly to action
    // The form will automatically reset after successful submission
    <form action={createPost}>
      <input name="title" placeholder="Post title" required />
      <textarea name="body" placeholder="Post body" />
      <button type="submit">Create Post</button>
    </form>
  );
}
```

### Client-side Async Actions (without "use server")

```tsx
// 🆕 React 19 — purely client-side async action
import { useActionState } from "react"; // covered in section 6

async function submitContactForm(formData: FormData) {
  const name = formData.get("name") as string;
  const email = formData.get("email") as string;
  const message = formData.get("message") as string;

  const res = await fetch("/api/contact", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ name, email, message }),
  });

  if (!res.ok) {
    throw new Error("Failed to send message");
  }
}

function ContactForm() {
  return (
    <form action={submitContactForm}>
      <input name="name" placeholder="Your name" required />
      <input name="email" type="email" placeholder="Your email" required />
      <textarea name="message" placeholder="Your message" required />
      <button type="submit">Send</button>
    </form>
  );
}
```

### Async Transitions

```tsx
// 🆕 React 19 — startTransition now accepts async functions
import { startTransition, useState } from "react";

function AsyncTransitionDemo() {
  const [result, setResult] = useState<string | null>(null);
  const [isPending, startTransitionFn] = useTransition();

  function handleSearch(query: string) {
    // startTransition now accepts async — new in React 19
    startTransitionFn(async () => {
      const data = await searchAPI(query);
      setResult(data.summary); // state updates inside are batched as a transition
    });
  }

  return (
    <div>
      <input onChange={(e) => handleSearch(e.target.value)} />
      {isPending && <p>Searching...</p>}
      {result && <p>{result}</p>}
    </div>
  );
}

async function searchAPI(query: string): Promise<{ summary: string }> {
  const res = await fetch(`/api/search?q=${encodeURIComponent(query)}`);
  return res.json();
}
```

---

## 6. 🆕 useActionState

`useActionState` is the hook for managing the full lifecycle of an **action**: its pending state, the result, and any error. It pairs perfectly with form actions.

```tsx
// 🆕 React 19 — useActionState
import { useActionState } from "react";

// Shape: useActionState(actionFn, initialState, permalink?)
// Returns: [state, dispatch, isPending]

// --- SIMPLE EXAMPLE: like button ---

type LikeState = { likes: number; error: string | null };

async function likePost(
  prevState: LikeState,   // ← previous state is passed as FIRST argument
  formData: FormData       // ← FormData is second
): Promise<LikeState> {
  try {
    const res = await fetch("/api/like", { method: "POST" });
    if (!res.ok) throw new Error("Failed to like");
    const data = await res.json();
    return { likes: data.likes, error: null };
  } catch (err) {
    return { likes: prevState.likes, error: (err as Error).message };
  }
}

function LikeButton({ initialLikes }: { initialLikes: number }) {
  const [state, dispatch, isPending] = useActionState(
    likePost,
    { likes: initialLikes, error: null } // initial state
  );

  return (
    <form action={dispatch}>
      {state.error && <p className="error">{state.error}</p>}
      <button type="submit" disabled={isPending}>
        {isPending ? "Liking..." : `❤️ ${state.likes}`}
      </button>
    </form>
  );
}
```

```tsx
// --- FULL FORM EXAMPLE: registration ---

type RegisterState =
  | { status: "idle" }
  | { status: "success"; username: string }
  | { status: "error"; message: string };

async function registerUser(
  prevState: RegisterState,
  formData: FormData
): Promise<RegisterState> {
  const username = formData.get("username") as string;
  const password = formData.get("password") as string;

  if (username.length < 3) {
    return { status: "error", message: "Username must be at least 3 characters" };
  }

  try {
    const res = await fetch("/api/register", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ username, password }),
    });

    if (!res.ok) {
      const err = await res.json();
      return { status: "error", message: err.message };
    }

    return { status: "success", username };
  } catch {
    return { status: "error", message: "Network error. Please try again." };
  }
}

function RegisterForm() {
  const [state, formAction, isPending] = useActionState(
    registerUser,
    { status: "idle" }
  );

  if (state.status === "success") {
    return <p>Welcome, {state.username}! Registration successful.</p>;
  }

  return (
    <form action={formAction}>
      {state.status === "error" && (
        <div role="alert" className="error-banner">
          {state.message}
        </div>
      )}

      <label>
        Username
        <input name="username" type="text" autoComplete="username" required />
      </label>

      <label>
        Password
        <input name="password" type="password" autoComplete="new-password" required />
      </label>

      <button type="submit" disabled={isPending}>
        {isPending ? "Creating account..." : "Register"}
      </button>
    </form>
  );
}
```

> **Key insight:** The action function receives `(previousState, formData)`. The return value becomes the new state. This is similar to a reducer, but async and tied to form submission.

---

## 7. 🆕 useFormStatus

`useFormStatus` reads the submission state of the **nearest ancestor `<form>`**. It must be used in a component that is a child of a form — it cannot be used in the same component that renders the form.

```tsx
// 🆕 React 19 — useFormStatus
import { useFormStatus } from "react-dom"; // Note: from "react-dom", not "react"

// ✅ This component must be INSIDE the form to work
function SubmitButton() {
  const { pending, data, method, action } = useFormStatus();
  // pending → true while the form action is running
  // data    → FormData that was submitted
  // method  → "get" | "post"
  // action  → the action function or URL

  return (
    <button type="submit" disabled={pending} aria-busy={pending}>
      {pending ? "Submitting..." : "Submit"}
    </button>
  );
}

// Loading indicator that lives inside the form
function FormLoadingBar() {
  const { pending } = useFormStatus();
  return pending ? <div className="progress-bar" role="progressbar" /> : null;
}

// ❌ WRONG — useFormStatus is in the SAME component as the form
function WrongUsage() {
  const { pending } = useFormStatus(); // ← does NOT read this form's status!
  return (
    <form action={someAction}>
      <button disabled={pending}>Submit</button> {/* pending is always false */}
    </form>
  );
}

// ✅ CORRECT — SubmitButton is a separate child component
function CorrectUsage() {
  return (
    <form action={someAction}>
      <input name="email" type="email" />
      <FormLoadingBar />   {/* reads parent form status */}
      <SubmitButton />     {/* reads parent form status */}
    </form>
  );
}

async function someAction(formData: FormData) {
  await new Promise((r) => setTimeout(r, 2000)); // simulate delay
}
```

---

## 8. 🆕 useOptimistic

`useOptimistic` lets you show an **optimistic (assumed-successful) UI update** immediately, while the real async operation happens in the background. If the operation fails, it rolls back automatically.

```tsx
// 🆕 React 19 — useOptimistic
import { useOptimistic, useActionState } from "react";

interface Todo {
  id: number;
  text: string;
  completed: boolean;
}

interface TodoState {
  todos: Todo[];
  error: string | null;
}

// --- EXAMPLE 1: Optimistic todo toggle ---
function TodoList({ initialTodos }: { initialTodos: Todo[] }) {
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    initialTodos,
    // Reducer: (currentState, optimisticValue) → newState
    // This runs SYNCHRONOUSLY when you call addOptimisticTodo()
    (state, toggledId: number) =>
      state.map((todo) =>
        todo.id === toggledId ? { ...todo, completed: !todo.completed } : todo
      )
  );

  async function handleToggle(id: number) {
    // 1. Immediately show the optimistic update (no waiting)
    addOptimisticTodo(id);

    // 2. Actually perform the async operation
    try {
      await fetch(`/api/todos/${id}/toggle`, { method: "PATCH" });
      // On success: the real state replaces the optimistic state
    } catch {
      // On failure: React automatically rolls back to the previous state
      // You may also want to show an error toast here
    }
  }

  return (
    <ul>
      {optimisticTodos.map((todo) => (
        <li
          key={todo.id}
          style={{ opacity: todo.completed ? 0.5 : 1 }}
        >
          <input
            type="checkbox"
            checked={todo.completed}
            onChange={() => handleToggle(todo.id)}
          />
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

```tsx
// --- EXAMPLE 2: Optimistic message sending ---
interface Message {
  id: string;
  text: string;
  sending?: boolean; // extra optimistic flag
}

function ChatRoom({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState<Message[]>([]);

  const [optimisticMessages, addOptimisticMessage] = useOptimistic(
    messages,
    (state, newText: string) => [
      ...state,
      {
        id: `temp-${Date.now()}`,
        text: newText,
        sending: true, // show "sending..." indicator
      },
    ]
  );

  async function sendMessage(formData: FormData) {
    const text = formData.get("text") as string;

    // Show it immediately
    addOptimisticMessage(text);

    // Actually send it
    const res = await fetch(`/api/rooms/${roomId}/messages`, {
      method: "POST",
      body: JSON.stringify({ text }),
      headers: { "Content-Type": "application/json" },
    });
    const saved = await res.json() as Message;

    // Update real state with server response
    setMessages((prev) => [...prev, saved]);
  }

  return (
    <div>
      <ul>
        {optimisticMessages.map((m) => (
          <li key={m.id} style={{ opacity: m.sending ? 0.6 : 1 }}>
            {m.text} {m.sending && <span>(sending...)</span>}
          </li>
        ))}
      </ul>
      <form action={sendMessage}>
        <input name="text" placeholder="Type a message..." required />
        <button type="submit">Send</button>
      </form>
    </div>
  );
}
```

---

## 9. 🆕 ref as a Prop — No More forwardRef

In React 19, function components receive `ref` **as a plain prop** — just like `className` or `onClick`. The `forwardRef()` wrapper is **deprecated** and will be removed in a future version.

### Before (React 18) vs After (React 19)

```tsx
// ❌ React 18 way — now DEPRECATED in React 19
import { forwardRef, type Ref } from "react";

const OldInput = forwardRef(function OldInput(
  props: { placeholder?: string },
  ref: Ref<HTMLInputElement>
) {
  return <input ref={ref} {...props} />;
});

// ✅ React 19 way — ref is just a prop
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  ref?: React.Ref<HTMLInputElement>;
}

function NewInput({ ref, ...props }: InputProps) {
  return <input ref={ref} {...props} />;
}

// Usage is identical for both:
function Parent() {
  const inputRef = useRef<HTMLInputElement>(null);
  return <NewInput ref={inputRef} placeholder="Focus me" />;
}
```

### Ref Cleanup Functions — 🆕 React 19

```tsx
// 🆕 React 19 — callback refs can return a cleanup function
// This is NEW — previously you had to handle cleanup manually

function VideoPlayer({ src }: { src: string }) {
  return (
    <video
      src={src}
      ref={(videoElement) => {
        if (videoElement === null) return; // element unmounted, cleanup ran

        // Setup — runs when element mounts
        videoElement.play().catch(console.error);

        // 🆕 React 19: return a cleanup function!
        // Called when the element unmounts or before the next ref call
        return () => {
          videoElement.pause();
          videoElement.src = ""; // release the resource
        };
      }}
    />
  );
}

// Another example: attaching a third-party library to a DOM element
function Chart({ data }: { data: number[] }) {
  return (
    <canvas
      ref={(canvas) => {
        if (!canvas) return;

        const chart = new SomeChartLib(canvas, { data });

        // 🆕 cleanup function — no more checking for null in the same callback
        return () => chart.destroy();
      }}
    />
  );
}
```

---

## 10. 🆕 Context as a Provider — No More `.Provider`

React 19 lets you render a context object **directly as a JSX element** to provide its value. The `.Provider` sub-component still works but is now unnecessary.

```tsx
// 🆕 React 19 — render <MyContext> directly
import { createContext, useContext, useState, type ReactNode } from "react";

interface AuthContextType {
  user: { id: number; name: string } | null;
  logout: () => void;
}

const AuthContext = createContext<AuthContextType>({ user: null, logout: () => {} });

// ✅ React 19: No need for <AuthContext.Provider value={...}>
function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<{ id: number; name: string } | null>(null);

  function logout() { setUser(null); }

  // 🆕 React 19 syntax:
  return (
    <AuthContext value={{ user, logout }}>
      {children}
    </AuthContext>
  );

  // React 18 syntax (still valid, but verbose):
  // return (
  //   <AuthContext.Provider value={{ user, logout }}>
  //     {children}
  //   </AuthContext.Provider>
  // );
}

function UserMenu() {
  const { user, logout } = useContext(AuthContext);
  if (!user) return <a href="/login">Login</a>;
  return (
    <div>
      <span>Hello, {user.name}</span>
      <button onClick={logout}>Logout</button>
    </div>
  );
}

function App() {
  return (
    <AuthProvider>
      <header>
        <UserMenu />
      </header>
      <main>...</main>
    </AuthProvider>
  );
}
```

---

## 11. 🆕 Document Metadata & Resource Preloading

React 19 supports rendering `<title>`, `<meta>`, and `<link>` tags **anywhere in your component tree**. React hoists them to the `<head>` automatically, with deduplication.

### Native Document Metadata

```tsx
// 🆕 React 19 — no more react-helmet or next/head needed for basic cases
// Works in both Client and Server Components

function BlogPost({ post }: { post: { title: string; description: string; slug: string } }) {
  return (
    <article>
      {/* 🆕 These are hoisted to <head> automatically */}
      <title>{post.title} | My Blog</title>
      <meta name="description" content={post.description} />
      <meta property="og:title" content={post.title} />
      <meta property="og:type" content="article" />
      <link rel="canonical" href={`https://myblog.com/posts/${post.slug}`} />

      {/* The actual content */}
      <h1>{post.title}</h1>
      <p>{post.description}</p>
    </article>
  );
}

// Multiple components can render the same <title> — React deduplicates
// The "last one wins" rule applies for <title>
// For <meta name>, React deduplicates by the `name` attribute
// For <link rel="stylesheet">, React deduplicates by `href`
```

### Stylesheets with Precedence

```tsx
// 🆕 React 19 — <link rel="stylesheet"> with precedence control
// React ensures stylesheets load before the component renders
// and deduplicates if the same href is used multiple times

function ComponentWithStyles() {
  return (
    <>
      {/* precedence controls loading order — "default" < "high" */}
      <link rel="stylesheet" href="/styles/base.css" precedence="default" />
      <link rel="stylesheet" href="/styles/component.css" precedence="high" />
      <div className="my-component">Content</div>
    </>
  );
}
```

### Async Scripts

```tsx
// 🆕 React 19 — <script async> is deduplicated by src
// React only loads the script once, even if multiple components render it

function AnalyticsWidget() {
  return (
    <>
      <script async src="https://analytics.example.com/tracker.js" />
      <div>Analytics enabled</div>
    </>
  );
}
```

### 🆕 Resource Preloading APIs

React 19 ships these imperative preloading functions. Import from `react-dom`.

```tsx
import {
  prefetchDNS,   // resolve DNS early
  preconnect,    // DNS + TCP + TLS handshake
  preload,       // fetch and cache a resource (but don't execute)
  preinit,       // fetch AND execute/apply the resource immediately
} from "react-dom";

// Call these as early as possible (e.g., in a layout component or route handler)
function AppShell() {
  // Warm up DNS for third-party origins
  prefetchDNS("https://fonts.googleapis.com");

  // Full connection warm-up (DNS + TCP + TLS)
  preconnect("https://api.example.com");

  // Preload a resource — doesn't execute it, just caches
  preload("https://fonts.googleapis.com/css2?family=Inter:wght@400;700", {
    as: "style",
  });
  preload("/hero-image.webp", { as: "image", fetchPriority: "high" });

  // Preinit — fetch AND apply immediately (for critical CSS/JS)
  preinit("https://cdn.example.com/critical.css", { as: "style" });
  preinit("https://cdn.example.com/analytics.js", { as: "script" });

  return (
    <main>
      <HeroSection />
      <ContentSection />
    </main>
  );
}
```

| API | What it does | Use for |
|---|---|---|
| `prefetchDNS(href)` | Resolves DNS hostname in background | Any third-party origin you'll need |
| `preconnect(href, options?)` | DNS + TCP + TLS handshake | API servers, CDNs, font providers |
| `preload(href, options)` | Downloads & caches (doesn't execute) | Fonts, images, large scripts used shortly |
| `preinit(href, options)` | Downloads AND executes/applies | Critical CSS, analytics scripts |

---

## 12. Concurrent Features

React 18 introduced concurrent rendering — React can work on multiple tasks simultaneously, pause work, and resume it. React 19 builds on this. These are the main APIs.

### useTransition

```tsx
import { useTransition, useState } from "react";

// useTransition marks a state update as "non-urgent"
// React can interrupt it to handle more urgent updates (like typing in an input)

function SearchPage() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState<string[]>([]);
  const [isPending, startTransition] = useTransition();

  function handleSearch(e: React.ChangeEvent<HTMLInputElement>) {
    const value = e.target.value;
    setQuery(value); // ← URGENT: update the input immediately

    // startTransition: mark the search as low-priority
    // React will keep the UI responsive while this runs
    startTransition(() => {
      const filtered = heavySearch(value); // expensive filter
      setResults(filtered);
    });
  }

  return (
    <div>
      <input value={query} onChange={handleSearch} placeholder="Search..." />
      {/* Show a visual indicator while the transition is pending */}
      <div style={{ opacity: isPending ? 0.5 : 1 }}>
        {results.map((r) => <p key={r}>{r}</p>)}
      </div>
    </div>
  );
}

function heavySearch(query: string): string[] {
  // Simulates an expensive synchronous search
  const items = Array.from({ length: 10_000 }, (_, i) => `Item ${i}`);
  return items.filter((item) => item.toLowerCase().includes(query.toLowerCase()));
}
```

### useDeferredValue

```tsx
import { useDeferredValue, useState, memo } from "react";

// useDeferredValue is like a "lazy copy" of a value
// The deferred value lags behind the real value
// React renders with the stale value first (keeping the UI responsive),
// then re-renders with the fresh value when the browser is idle

const HeavyList = memo(function HeavyList({ filter }: { filter: string }) {
  // Imagine this is expensive to compute
  const items = Array.from({ length: 5000 }, (_, i) => `Item ${i}`)
    .filter((item) => item.includes(filter));

  return (
    <ul>
      {items.slice(0, 50).map((item) => (
        <li key={item}>{item}</li>
      ))}
    </ul>
  );
});

function FilteredList() {
  const [filter, setFilter] = useState("");
  const deferredFilter = useDeferredValue(filter);
  // filter   → updates immediately (what the input shows)
  // deferredFilter → lags behind, used for the expensive list

  return (
    <div>
      <input
        value={filter}
        onChange={(e) => setFilter(e.target.value)}
        placeholder="Filter..."
      />
      {/* Show a visual "stale" indicator */}
      <div style={{ opacity: deferredFilter !== filter ? 0.6 : 1 }}>
        <HeavyList filter={deferredFilter} />
      </div>
    </div>
  );
}
```

### Suspense

```tsx
import { Suspense, lazy } from "react";

// Suspense catches components that "suspend" (are not yet ready to render)
// and shows a fallback UI until they are ready

// Use case 1: Code splitting with React.lazy
const HeavyChart = lazy(() => import("./HeavyChart"));

function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Suspense fallback={<div className="skeleton" />}>
        <HeavyChart />
      </Suspense>
    </div>
  );
}

// Use case 2: Data fetching (with use() or RSC)
function UserSection({ userPromise }: { userPromise: Promise<User> }) {
  return (
    <Suspense fallback={<UserSkeleton />}>
      <UserProfile promise={userPromise} />
    </Suspense>
  );
}

// Use case 3: Nested Suspense boundaries for granular loading
function PageLayout({ user, posts }: {
  user: Promise<User>;
  posts: Promise<Post[]>;
}) {
  return (
    <main>
      {/* User section resolves fast — its own boundary */}
      <Suspense fallback={<Skeleton width={200} height={50} />}>
        <UserHeader promise={user} />
      </Suspense>

      {/* Posts take longer — separate boundary, different fallback */}
      <Suspense fallback={<PostListSkeleton />}>
        <PostList promise={posts} />
      </Suspense>
    </main>
  );
}
```

### startTransition (without the hook)

```tsx
import { startTransition } from "react";

// Use the standalone startTransition when you don't need isPending
// (e.g., in an event handler where you just want to mark something as low-priority)

function NavLink({ href, children }: { href: string; children: React.ReactNode }) {
  function handleClick(e: React.MouseEvent) {
    e.preventDefault();
    startTransition(() => {
      // Navigate — this is a low-priority update
      // React can interrupt it if the user clicks something else
      navigateTo(href);
    });
  }

  return <a href={href} onClick={handleClick}>{children}</a>;
}

function navigateTo(href: string) {
  // Your custom router logic
  window.history.pushState({}, "", href);
}
```

---

## 13. Server Components & Server Actions

> **⚡ Version note:** React 19 ships the **specification** for Server Components and Server Actions. The actual implementation is done by frameworks. **Next.js (App Router), Remix, and others** provide the runtime. If you are using plain Vite + React, you do NOT have Server Components unless you add a framework layer.

### The Mental Model

```
┌─────────────────────────────────────────────────────────────┐
│  SERVER                                                     │
│  ┌─────────────────────┐                                    │
│  │ Server Components   │  Run only on server                │
│  │  - Can await data   │  Never shipped to browser          │
│  │  - No useState/     │  Zero JS bundle impact             │
│  │    useEffect/hooks  │  Direct DB/FS access               │
│  └──────────┬──────────┘                                    │
│             │ renders into                                  │
└─────────────┼───────────────────────────────────────────────┘
              │ (React serialized format over network)
┌─────────────▼───────────────────────────────────────────────┐
│  BROWSER                                                    │
│  ┌─────────────────────┐                                    │
│  │ Client Components   │  "use client" at top of file       │
│  │  - useState, hooks  │  Shipped to browser as JS          │
│  │  - Event handlers   │  Interactive                       │
│  │  - useEffect        │                                    │
│  └─────────────────────┘                                    │
└─────────────────────────────────────────────────────────────┘
```

### Server Components

```tsx
// No "use client" directive → this is a Server Component (in a framework like Next.js)
// File: app/products/page.tsx

interface Product {
  id: number;
  name: string;
  price: number;
}

// Can be async! Direct DB access. No useEffect needed.
async function ProductsPage() {
  // This runs on the server — no API round-trip, no loading state needed
  const products = await db.query("SELECT * FROM products ORDER BY name");

  return (
    <section>
      <h1>Products</h1>
      {products.map((p: Product) => (
        // ProductCard is a client component (has a cart button)
        <ProductCard key={p.id} product={p} />
      ))}
    </section>
  );
}

export default ProductsPage;
```

### Client Components

```tsx
// "use client" — this file and everything it imports runs in the browser
"use client";

import { useState } from "react";

interface Product {
  id: number;
  name: string;
  price: number;
}

function ProductCard({ product }: { product: Product }) {
  const [inCart, setInCart] = useState(false);

  return (
    <div className="card">
      <h2>{product.name}</h2>
      <p>${product.price}</p>
      <button onClick={() => setInCart((prev) => !prev)}>
        {inCart ? "Remove from Cart" : "Add to Cart"}
      </button>
    </div>
  );
}

export default ProductCard;
```

### Server Actions

```tsx
// Server Actions are async functions that run on the server
// Triggered from the client (form submission, button click)
// Defined with "use server" directive

// Option 1: "use server" at the top of a dedicated file (actions.ts)
"use server";

import { revalidatePath } from "next/cache"; // Next.js specific

export async function deleteProduct(formData: FormData) {
  const id = Number(formData.get("id"));

  await db.query("DELETE FROM products WHERE id = ?", [id]);

  // Next.js: revalidate the products page
  revalidatePath("/products");
}

export async function updateProductPrice(
  prevState: { error: string | null },
  formData: FormData
): Promise<{ error: string | null }> {
  const id = Number(formData.get("id"));
  const price = Number(formData.get("price"));

  if (isNaN(price) || price < 0) {
    return { error: "Invalid price" };
  }

  await db.query("UPDATE products SET price = ? WHERE id = ?", [price, id]);
  revalidatePath("/products");

  return { error: null };
}
```

```tsx
// Option 2: "use server" inside a Server Component (inline)
// This is only valid inside a Server Component file
async function ServerPage() {
  // Inline Server Action
  async function handleDelete(formData: FormData) {
    "use server"; // ← marks THIS function as a Server Action
    const id = formData.get("id");
    await db.delete(id);
  }

  return (
    <form action={handleDelete}>
      <input type="hidden" name="id" value="123" />
      <button type="submit">Delete</button>
    </form>
  );
}
```

---

## 14. Error Handling

### Error Boundaries

```tsx
// Error boundaries catch errors in the React component tree during rendering
// They must be CLASS components (no hook equivalent exists)

import { Component, type ErrorInfo, type ReactNode } from "react";

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends Component<
  { children: ReactNode; fallback?: ReactNode },
  ErrorBoundaryState
> {
  constructor(props: { children: ReactNode; fallback?: ReactNode }) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    // Called when a descendant throws during rendering
    // Return new state to show the fallback UI
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    // Log the error to an error reporting service
    console.error("Error boundary caught:", error, info.componentStack);
    // reportError(error, info); // e.g., Sentry
  }

  render() {
    if (this.state.hasError) {
      return (
        this.props.fallback ?? (
          <div role="alert">
            <h2>Something went wrong.</h2>
            <p>{this.state.error?.message}</p>
            <button onClick={() => this.setState({ hasError: false, error: null })}>
              Try again
            </button>
          </div>
        )
      );
    }

    return this.props.children;
  }
}

// Usage
function App() {
  return (
    <ErrorBoundary fallback={<p>Failed to load content</p>}>
      <SomeComponentThatMightThrow />
    </ErrorBoundary>
  );
}

// ⚠️ Error boundaries do NOT catch:
// - Errors in event handlers (use try/catch in the handler)
// - Errors in async code (use try/catch + state)
// - Errors in the error boundary itself
// - Errors during SSR
```

### 🆕 Improved Root Error Reporting in React 19

```tsx
// 🆕 React 19 — new root-level error hooks
import { createRoot } from "react-dom/client";

createRoot(document.getElementById("root")!, {
  // 🆕 Called when an error is caught by an Error Boundary
  onCaughtError(error: Error, errorInfo: React.ErrorInfo) {
    console.error("[Caught by Error Boundary]", error, errorInfo.componentStack);
    reportToSentry(error, { handled: true });
  },

  // 🆕 Called for errors NOT caught by any Error Boundary (will crash the app)
  onUncaughtError(error: Error, errorInfo: React.ErrorInfo) {
    console.error("[Uncaught — app will crash]", error, errorInfo.componentStack);
    reportToSentry(error, { handled: false, fatal: true });
  },

  // Existed in React 18 — called for recoverable errors React auto-recovers from
  onRecoverableError(error: Error, errorInfo: React.ErrorInfo) {
    console.warn("[Recoverable]", error);
  },
}).render(<App />);

function reportToSentry(error: Error, context: Record<string, unknown>) {
  // Sentry.captureException(error, { extra: context });
  console.log("Reporting to Sentry:", error.message, context);
}
```

---

## 15. The React Compiler

The **React Compiler** is an official build-time tool that automatically inserts memoization (`useMemo`, `useCallback`, `React.memo`) where it determines they are beneficial. It analyzes your component code and transforms it at build time.

> **⚡ Version note:** The React Compiler was released as a stable opt-in tool in 2025. As of 2026, it is widely adopted but not yet the default in all frameworks. Next.js 16+ supports it via config. Vite users can add it via the Babel or SWC plugin.

### What it does

```tsx
// What YOU write:
function SearchResults({ query, items }: { query: string; items: string[] }) {
  const filtered = items.filter((item) => item.includes(query));

  return <ul>{filtered.map((item) => <li key={item}>{item}</li>)}</ul>;
}

// What the compiler effectively produces (conceptually):
function SearchResults({ query, items }: { query: string; items: string[] }) {
  const filtered = useMemo(
    () => items.filter((item) => item.includes(query)),
    [items, query]
  );

  return <ul>{filtered.map((item) => <li key={item}>{item}</li>)}</ul>;
}

// You don't need to write useMemo manually — the compiler does it for you
```

### Enabling the React Compiler

```bash
# Install the Babel plugin
npm install --save-dev babel-plugin-react-compiler
```

```js
// babel.config.js
module.exports = {
  plugins: [
    ["babel-plugin-react-compiler", {
      target: "19", // React 19
    }],
  ],
};
```

```ts
// next.config.ts (Next.js 16+)
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  experimental: {
    reactCompiler: true,
  },
};

export default nextConfig;
```

### Rules of React — Required for the Compiler

The compiler only works if your components follow the **Rules of React**:
1. Components must be **pure** (same inputs → same output, no side effects during render)
2. Hooks must follow the **Rules of Hooks** (top-level, no conditions)
3. Props and state must be treated as **immutable** (never mutate them directly)
4. No direct DOM manipulation during render

```tsx
// ❌ BREAKS the compiler — mutation during render
function Bad({ items }: { items: number[] }) {
  items.sort(); // MUTATION — breaks purity, compiler opts this out
  return <ul>{items.map((i) => <li key={i}>{i}</li>)}</ul>;
}

// ✅ Correct — create a new sorted array
function Good({ items }: { items: number[] }) {
  const sorted = [...items].sort();
  return <ul>{sorted.map((i) => <li key={i}>{i}</li>)}</ul>;
}
```

### Opting Out

```tsx
// If a component can't be compiled safely, you can opt it out
function LegacyComponent() {
  "use no memo"; // ← directive to skip compilation for this component
  // ... complex code the compiler can't handle
}
```

### Impact on Development Workflow

| Before Compiler | With Compiler |
|---|---|
| Must manually add `useMemo` for expensive calculations | Compiler adds it automatically |
| Must manually add `useCallback` for stable callbacks | Compiler handles it |
| Must wrap with `React.memo()` to prevent child re-renders | Compiler auto-memoizes components |
| Easy to introduce performance bugs by forgetting memo | Compiler catches most cases |
| `useMemo`/`useCallback` are still useful for clarity | Can mostly be omitted |

---

## 16. Refs, Portals & Lazy Loading

### useRef vs createRef

```tsx
import { useRef, createRef } from "react";

// useRef — persists across re-renders (for function components) ✅
function FunctionComponent() {
  const ref = useRef<HTMLDivElement>(null); // same ref object every render
  return <div ref={ref}>...</div>;
}

// createRef — creates a NEW ref every call (use only in class components or one-offs)
// In function components, createRef creates a new ref on every render ❌
```

### Portals

```tsx
// Portals render children into a DOM node OUTSIDE the parent component's DOM tree
// Common uses: modals, tooltips, toasts, dropdowns (escape overflow:hidden)

import { createPortal } from "react-dom";
import { useState, useEffect } from "react";

interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  children: React.ReactNode;
}

function Modal({ isOpen, onClose, children }: ModalProps) {
  // Close on Escape key
  useEffect(() => {
    if (!isOpen) return;
    function handleKey(e: KeyboardEvent) {
      if (e.key === "Escape") onClose();
    }
    document.addEventListener("keydown", handleKey);
    return () => document.removeEventListener("keydown", handleKey);
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  // createPortal(content, targetDOMNode)
  // Renders into document.body even though <Modal> is nested in a sidebar
  return createPortal(
    <div
      className="modal-overlay"
      role="dialog"
      aria-modal="true"
      onClick={(e) => {
        if (e.target === e.currentTarget) onClose(); // click backdrop to close
      }}
    >
      <div className="modal-content">
        <button
          className="modal-close"
          onClick={onClose}
          aria-label="Close modal"
        >
          ×
        </button>
        {children}
      </div>
    </div>,
    document.body // ← portal target — can be any DOM element
  );
}

function App() {
  const [showModal, setShowModal] = useState(false);

  return (
    <div className="app">
      <button onClick={() => setShowModal(true)}>Open Modal</button>
      <Modal isOpen={showModal} onClose={() => setShowModal(false)}>
        <h2>Modal Title</h2>
        <p>Modal content here.</p>
      </Modal>
    </div>
  );
}
```

### React.lazy & Code Splitting

```tsx
// React.lazy lets you split large components into separate JS chunks
// The chunk is loaded only when the component is first rendered

import { lazy, Suspense, useState } from "react";

// The import() is dynamic — Vite/Webpack create a separate chunk
const HeavyEditor = lazy(() => import("./HeavyEditor"));
const AdminPanel = lazy(() =>
  import("./AdminPanel").then((module) => ({ default: module.AdminPanel }))
  // ↑ If the module doesn't have a default export, map it
);

function App() {
  const [showEditor, setShowEditor] = useState(false);
  const isAdmin = true;

  return (
    <div>
      <button onClick={() => setShowEditor(true)}>Open Editor</button>

      {showEditor && (
        // Suspense fallback shows while the chunk downloads
        <Suspense fallback={<div className="editor-skeleton">Loading editor...</div>}>
          <HeavyEditor />
        </Suspense>
      )}

      {isAdmin && (
        <Suspense fallback={<p>Loading admin panel...</p>}>
          <AdminPanel />
        </Suspense>
      )}
    </div>
  );
}

// Route-level code splitting (most common pattern with a router)
import { BrowserRouter, Routes, Route } from "react-router-dom";

const HomePage = lazy(() => import("./pages/HomePage"));
const AboutPage = lazy(() => import("./pages/AboutPage"));
const ContactPage = lazy(() => import("./pages/ContactPage"));

function Router() {
  return (
    <BrowserRouter>
      <Suspense fallback={<div className="page-loader">Loading...</div>}>
        <Routes>
          <Route path="/"        element={<HomePage />} />
          <Route path="/about"   element={<AboutPage />} />
          <Route path="/contact" element={<ContactPage />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

---

## 17. Performance Best Practices & Common Pitfalls

### Key Performance Rules

```tsx
// ✅ 1. Collocate state — put state as low in the tree as it can be
// ❌ WRONG: lifting all state to root causes whole-tree re-renders
// ✅ RIGHT: keep state in the component that needs it

// ✅ 2. Memoize only what is actually slow
// The React Compiler handles most of this, but if you're not using it:

const ExpensiveList = memo(function ExpensiveList({
  items,
  onSelect,
}: {
  items: string[];
  onSelect: (item: string) => void;
}) {
  console.log("ExpensiveList rendered"); // you'd see this less with memo
  return (
    <ul>
      {items.map((item) => (
        <li key={item} onClick={() => onSelect(item)}>
          {item}
        </li>
      ))}
    </ul>
  );
});

// ✅ 3. Avoid creating objects/arrays inline in JSX props
function Parent() {
  // ❌ Creates a new array EVERY render → child always re-renders
  return <ExpensiveList items={["a", "b", "c"]} onSelect={() => {}} />;
}

// ✅ Better — stable reference
const ITEMS = ["a", "b", "c"]; // outside component = never recreated

function BetterParent() {
  const handleSelect = useCallback((item: string) => {
    console.log("Selected:", item);
  }, []); // stable function reference

  return <ExpensiveList items={ITEMS} onSelect={handleSelect} />;
}
```

### Common Pitfalls

```tsx
// ❌ PITFALL 1: Setting state unconditionally in useEffect without deps
useEffect(() => {
  setCount((c) => c + 1); // ← infinite loop! Missing dep array
}); // No [] — runs after EVERY render → triggers re-render → repeat

// ❌ PITFALL 2: Stale closure in event handlers or effects
function StaleClosureExample() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(count); // ← STALE: always logs 0 (captured at mount)
      setCount(count + 1); // ← STALE: count is always 0, so sets to 1 forever
    }, 1000);
    return () => clearInterval(id);
  }, []); // missing 'count' in deps
}

// ✅ Fix: use the updater function form
useEffect(() => {
  const id = setInterval(() => {
    setCount((c) => c + 1); // ← safe: uses current value, not closure
  }, 1000);
  return () => clearInterval(id);
}, []);

// ❌ PITFALL 3: Passing new objects/functions to memoized components
const styles = { color: "red" }; // ✅ defined outside → stable

function GoodParent() {
  return <Child style={styles} />; // stable ref → Child won't re-render unnecessarily
}

function BadParent() {
  return <Child style={{ color: "red" }} />; // ❌ new object every render
}

// ❌ PITFALL 4: Index as key in dynamic lists
function IndexKeyList({ items }: { items: string[] }) {
  return (
    <ul>
      {items.map((item, index) => (
        // ❌ If list reorders/filters, React gets confused about which item is which
        <li key={index}>{item}</li>
      ))}
    </ul>
  );
}

// ❌ PITFALL 5: Not cleaning up side effects
useEffect(() => {
  const subscription = externalStore.subscribe(handleUpdate);
  // ❌ No cleanup → memory leak and multiple subscriptions
}, []);

// ✅ Always clean up
useEffect(() => {
  const subscription = externalStore.subscribe(handleUpdate);
  return () => subscription.unsubscribe(); // ✅
}, []);

// ❌ PITFALL 6: Deriving state from props → "anti-pattern"
function BadComponent({ userId }: { userId: number }) {
  const [id, setId] = useState(userId); // ❌ id won't update when userId changes!
  // The state is initialized from userId, but is not synced with it
}

// ✅ Just use the prop directly, or compute during render
function GoodComponent({ userId }: { userId: number }) {
  // If you need derived state, compute it during render (or useMemo if expensive)
  const displayId = `User #${userId}`;
  return <p>{displayId}</p>;
}

// ❌ PITFALL 7: Forgetting that 0 is falsy but renders
function ZeroPitfall({ count }: { count: number }) {
  return (
    <div>
      {count && <p>Count: {count}</p>} {/* ❌ Renders "0" when count is 0! */}
      {count > 0 && <p>Count: {count}</p>} {/* ✅ Explicit boolean check */}
    </div>
  );
}
```

---

## 18. Migrating from React 18 to 19

### Summary of Breaking Changes

| What Changed | React 18 | React 19 | Action |
|---|---|---|---|
| `ReactDOM.render` | Deprecated | **REMOVED** | Replace with `createRoot` |
| `ReactDOM.hydrate` | Deprecated | **REMOVED** | Replace with `hydrateRoot` |
| `unmountComponentAtNode` | Deprecated | **REMOVED** | Call `.unmount()` on root |
| `renderToNodeStream` | Deprecated | **REMOVED** | Use `renderToPipeableStream` |
| Legacy Context (`contextTypes`, `getChildContext`) | Deprecated | **REMOVED** | Use `createContext` + `useContext` |
| String refs (`ref="myRef"`) | Deprecated | **REMOVED** | Use `useRef` |
| `propTypes` | Deprecated, in core | **REMOVED from React** | Use TypeScript |
| `defaultProps` on function components | Deprecated | **Deprecated** | Use ES6 default params |
| `forwardRef` | Required for ref passing | **Deprecated** | Use `ref` as plain prop |
| `<Context.Provider>` | Required | Deprecated (still works) | Use `<Context>` directly |
| `act` in tests | `react-dom/test-utils` | Moved to `react` | Update import |
| ReactDOMTestUtils | In `react-dom/test-utils` | **Deprecated** | Use `@testing-library/react` |

### Step-by-Step Migration

```bash
# Step 1: Upgrade packages
npm install react@19 react-dom@19
npm install --save-dev @types/react@19 @types/react-dom@19
```

```tsx
// Step 2: Fix ReactDOM.render → createRoot
// BEFORE
import ReactDOM from "react-dom";
ReactDOM.render(<App />, document.getElementById("root"));

// AFTER
import { createRoot } from "react-dom/client";
createRoot(document.getElementById("root")!).render(<App />);
```

```tsx
// Step 3: Remove forwardRef wrappers
// BEFORE
const Input = forwardRef<HTMLInputElement, { label: string }>(
  ({ label }, ref) => <div><label>{label}</label><input ref={ref} /></div>
);

// AFTER — ref is just a prop
function Input({ label, ref }: { label: string; ref?: React.Ref<HTMLInputElement> }) {
  return <div><label>{label}</label><input ref={ref} /></div>;
}
```

```tsx
// Step 4: Update Context providers (optional but cleaner)
// BEFORE
<MyContext.Provider value={value}>{children}</MyContext.Provider>

// AFTER
<MyContext value={value}>{children}</MyContext>
```

```tsx
// Step 5: Remove propTypes
// BEFORE
import PropTypes from "prop-types";

function Button({ label }: { label: string }) {
  return <button>{label}</button>;
}
Button.propTypes = { label: PropTypes.string.isRequired }; // ← DELETE THIS

// AFTER: just TypeScript types
function Button({ label }: { label: string }) {
  return <button>{label}</button>;
}
```

```tsx
// Step 6: Fix defaultProps on function components
// BEFORE (deprecated)
function Input({ value, placeholder }: { value: string; placeholder?: string }) {
  return <input value={value} placeholder={placeholder} />;
}
Input.defaultProps = { placeholder: "Enter text..." }; // ← DEPRECATED

// AFTER: ES6 default parameters
function Input({ value, placeholder = "Enter text..." }: {
  value: string;
  placeholder?: string;
}) {
  return <input value={value} placeholder={placeholder} />;
}
```

```tsx
// Step 7: Update test imports
// BEFORE
import { act } from "react-dom/test-utils";

// AFTER
import { act } from "react";
```

### Using the Codemod

```bash
# React provides an official codemod for common migrations
npx codemod react/19/replace-reactdom-render
npx codemod react/19/replace-string-ref
npx codemod react/19/replace-act-import
npx codemod react/19/replace-use-form-state  # useFormState → useActionState
```

> **⚡ Version note:** Run `npx codemod --list react` to see all available React 19 codemods. The most critical one is `replace-reactdom-render` — all others are either automated fixes or optional cleanups.

---

## 19. Tips, Tricks & Gotchas

### State & Hooks

```tsx
// TIP 1: Batch multiple state updates
// In React 18+, ALL state updates are batched automatically (even inside setTimeout/fetch)
// You don't need to manually batch in React 19

// TIP 2: If you need to "force" a re-render (rare, usually a code smell):
function useForceRender() {
  const [, forceRender] = useReducer((x) => x + 1, 0);
  return forceRender;
}

// TIP 3: Lifting state up vs context — know when to use each
// Lifting up: 2-3 levels, infrequent updates, few consumers
// Context: deeply nested, many consumers (but causes all consumers to re-render!)
// Zustand/Jotai: when context re-render cost becomes a problem

// TIP 4: useLayoutEffect vs useEffect
// useEffect → runs AFTER the browser paints (async) — use 99% of the time
// useLayoutEffect → runs SYNCHRONOUSLY after DOM mutations, before browser paint
//   Use it when you need to measure DOM and prevent visible flicker (e.g., tooltips)
import { useLayoutEffect, useRef } from "react";

function Tooltip({ text }: { text: string }) {
  const ref = useRef<HTMLDivElement>(null);

  useLayoutEffect(() => {
    // Measure the tooltip to position it correctly BEFORE the paint
    // This prevents a flicker where the tooltip appears in the wrong position
    const rect = ref.current?.getBoundingClientRect();
    if (rect && rect.right > window.innerWidth) {
      ref.current!.style.transform = "translateX(-100%)";
    }
  });

  return <div ref={ref} className="tooltip">{text}</div>;
}
```

### Forms & Actions

```tsx
// TIP 5: useActionState previous state is undefined on the first call
// Always handle both cases:
async function myAction(
  prevState: MyState | undefined,  // can be undefined on first run
  formData: FormData
): Promise<MyState> {
  const prev = prevState ?? DEFAULT_STATE;
  // ...
}

// TIP 6: Form data is only available during the action
// If you need to access it later, extract it in the action and return it in state

// TIP 7: use() and Suspense — the promise must be stable
// ❌ Creates a new promise every render → infinite Suspense loop
function Bad({ id }: { id: number }) {
  const data = use(fetch(`/api/${id}`).then((r) => r.json())); // new Promise every render!
}

// ✅ Stable promise — created outside the component or passed as a prop
const promises = new Map<number, Promise<unknown>>();
function getStablePromise(id: number) {
  if (!promises.has(id)) {
    promises.set(id, fetch(`/api/${id}`).then((r) => r.json()));
  }
  return promises.get(id)!;
}

function Good({ id }: { id: number }) {
  const data = use(getStablePromise(id));
}
```

### Refs & DOM

```tsx
// TIP 8: Don't read ref.current during render
// Refs are NOT reactive — reading them during render can give stale values
// Read refs in effects and event handlers

// ✅
function MyComponent() {
  const ref = useRef<HTMLDivElement>(null);
  const handleClick = () => console.log(ref.current?.offsetHeight); // ✅ in event handler
  useEffect(() => { console.log(ref.current?.offsetHeight); }, []); // ✅ in effect
  return <div ref={ref}>Content</div>;
}

// TIP 9: Avoid null-checking ref.current with TypeScript
// If you KNOW the ref will be set by the time an effect runs, use the ! non-null assertion
// Or check: if (!ref.current) return;
```

### Server Components & Actions

```tsx
// TIP 10: You can't use hooks in Server Components
// Server Components → no useState, useEffect, useRef, etc.
// Move interactive parts to Client Components ("use client")

// TIP 11: Server Actions throw on validation errors — catch in useActionState
// Don't throw for user errors — return them in state
// Throw only for unexpected server errors (caught by error boundary)

// TIP 12: Server Actions automatically revalidate in Next.js with revalidatePath/revalidateTag

// TIP 13: Client Components CAN import Server Components? No!
// Server Components CAN receive Client Components as children via props ✅
// Client Components CANNOT import Server Components ❌
// Pass Server Component output as `children` prop to Client Component wrapper
```

### Performance

```tsx
// TIP 14: Profile before you optimize
// Use React DevTools Profiler to find actual bottlenecks
// Don't add useMemo/useCallback everywhere "just in case"

// TIP 15: Key prop to force-reset component state
// Adding a different key to a component unmounts and remounts it (resets all state)
function ResetDemo({ userId }: { userId: number }) {
  // When userId changes, the form fully remounts — all state is reset
  return <UserForm key={userId} userId={userId} />;
}

// TIP 16: Virtualize long lists
// For 100+ items, use react-window or @tanstack/react-virtual
// Rendering 10,000 DOM nodes is always slow — virtualization fixes it

// TIP 17: Avoid layout thrashing in effects
// Don't read-then-write-then-read the DOM in the same effect
// Batch reads together, batch writes together
```

### TypeScript

```tsx
// TIP 18: Type children correctly
import type { ReactNode, PropsWithChildren } from "react";

// Option A — PropsWithChildren utility type
function Card({ title, children }: PropsWithChildren<{ title: string }>) {
  return <div><h2>{title}</h2>{children}</div>;
}

// Option B — explicit ReactNode
function Layout({ children }: { children: ReactNode }) {
  return <main>{children}</main>;
}

// TIP 19: Type component props with generics
function Select<T extends string | number>({
  options,
  value,
  onChange,
}: {
  options: T[];
  value: T;
  onChange: (value: T) => void;
}) {
  return (
    <select value={String(value)} onChange={(e) => onChange(e.target.value as T)}>
      {options.map((opt) => (
        <option key={opt} value={String(opt)}>{opt}</option>
      ))}
    </select>
  );
}

// TIP 20: as const for string union types from arrays
const ROLES = ["admin", "user", "moderator"] as const;
type Role = (typeof ROLES)[number]; // "admin" | "user" | "moderator"
```

---

## 20. Study Path

### Phase 1 — Core React (Weeks 1–2)

Learn the fundamentals cold before touching anything else.

1. **Understand JSX** — how it compiles to `React.createElement`, what rules apply
2. **Components & Props** — building small, composable UI pieces
3. **useState** — managing local component state, re-render model
4. **Conditional & list rendering** — ternary, `&&`, `.map()`, keys
5. **Event handling** — synthetic events, forms, controlled inputs
6. **useEffect** — data fetching, subscriptions, cleanup — this takes real practice

**Build:** A multi-step form with validation, a filterable list, a dark/light mode toggle.

### Phase 2 — Intermediate Hooks & Patterns (Weeks 3–4)

1. **useContext** — global theme, auth state (without a library)
2. **useReducer** — complex state machines (shopping cart, wizard forms)
3. **useRef** — DOM access, mutable values, previous value pattern
4. **useMemo & useCallback** — understand WHY, not just HOW
5. **Custom hooks** — extract and reuse logic (useFetch, useLocalStorage, useDebounce)
6. **React.memo** — preventing unnecessary re-renders

**Build:** A Kanban board with drag (useReducer), a modal system (portals), a custom form hook.

### Phase 3 — React 19 New APIs (Week 5)

1. **use()** — reading promises inside Suspense, conditional context reads
2. **useActionState** — form actions with pending/error/result state
3. **useFormStatus** — submit button that knows its form's state
4. **useOptimistic** — instant UI feedback with async rollback
5. **ref as a prop** — simplifying forwardRef patterns
6. **Document metadata** — title, meta, link anywhere in the tree
7. **Resource preloading APIs** — preload, preinit, prefetchDNS

**Build:** A full CRUD app using only React 19 form actions (no fetch in useEffect).

### Phase 4 — Concurrent & Performance (Week 6)

1. **Suspense** — data loading boundaries, nested boundaries, lazy components
2. **useTransition** — keeping the UI responsive during slow updates
3. **useDeferredValue** — debouncing expensive derived state
4. **Code splitting** — React.lazy + Suspense for route-level chunks
5. **React DevTools Profiler** — finding and fixing actual bottlenecks
6. **The React Compiler** — how it works, when to opt out

**Build:** A dashboard with multiple independent data-loading sections, each with its own Suspense.

### Phase 5 — Server Components & Full-Stack (Weeks 7–8)

1. **React Server Components** — the mental model, what they can/can't do
2. **"use client" / "use server"** — the boundary model
3. **Server Actions** — mutations without API routes
4. **Pick a framework** — Next.js 16 App Router is the dominant choice in 2026
5. **Error boundaries** — production error handling, onCaughtError, onUncaughtError
6. **Hydration** — SSR → client handoff, avoiding hydration mismatches

**Build:** A blog or e-commerce site with Next.js 16: server-fetched data, Server Actions for comments/cart, error boundaries per section.

### Phase 6 — Production Patterns (Ongoing)

1. **Authentication patterns** — session management, protected routes
2. **State management at scale** — Zustand or Jotai for complex client state
3. **Testing** — Vitest + @testing-library/react, mocking fetch, testing async
4. **Accessibility** — ARIA attributes, focus management, keyboard navigation
5. **Internationalization** — react-i18next or next-intl
6. **Monitoring** — Sentry integration, onCaughtError/onUncaughtError hooks

### Capstone Projects (Build These to Prove Fluency)

| Project | Core Skills Tested |
|---|---|
| **Real-time Todo App** | useState, useReducer, useOptimistic, form actions |
| **E-commerce Product Catalog** | Server Components, Suspense, lazy, React 19 metadata |
| **Multi-step Wizard Form** | useActionState, useFormStatus, validation, transitions |
| **Chat Application** | useOptimistic, WebSocket, useEffect cleanup, Suspense |
| **Admin Dashboard** | Data fetching, error boundaries, concurrent features, portals |
| **Full-Stack Blog** | Next.js + Server Actions + Server Components + RSC caching |

### Key Resources (for when you DO have internet)

- `react.dev` — Official docs, fully rewritten for React 19 with hooks-first approach
- `react.dev/blog` — React team announcements for new APIs
- `github.com/reactwg/react-18` and `reactwg/react-19` — Working group discussions
- `nextjs.org/docs` — For Server Components implementation details
- `github.com/facebook/react/blob/main/CHANGELOG.md` — Every API change documented

---

*Guide accurate as of June 2026. React 19 is stable. The React Compiler is stable and opt-in. Server Components are stable in Next.js 16+ and Remix. All code examples use TypeScript (React 19 types included in `@types/react@19`).*
