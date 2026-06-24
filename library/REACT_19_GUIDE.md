# React 19 — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I've never written a component" to "I build, optimise, and reason about production React apps" — without an internet connection. Every concept comes first as *prose* that explains **what it is**, **the logic / why React works this way**, **what it's for and when you reach for it**, **how to use it**, **the key props/parameters**, **best practices**, and **the gotchas** — and only then the heavily-commented, runnable code. The goal is the *mental model*, not a list of APIs. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **React 19 (stable)** — the release that landed in December 2024 and matured through 2025–2026. Where an API is brand-new in React 19 it is marked **🆕 React 19**; where behaviour is still evolving or framework-dependent it is flagged **⚡ Version note**. The guide uses **TypeScript (TSX)** throughout — the recommended default for every new project in 2026. React itself is unopinionated about the rest of your stack, so cross-references to its common companions appear where relevant: **Next.js** (the dominant framework), **TanStack Query** (server-state caching), **Zustand** (client state), **React Hook Form** (forms), and **Tailwind CSS** (styling).

---

## Table of Contents

1. [What React Is & Why It Exists](#1-what-react-is--why-it-exists) **[B]**
2. [JSX in Depth](#2-jsx-in-depth) **[B]**
3. [Components & Props](#3-components--props) **[B]**
4. [Rendering & the Render Cycle](#4-rendering--the-render-cycle) **[B/I]**
5. [State with `useState` & Why Immutability Matters](#5-state-with-usestate--why-immutability-matters) **[B]**
6. [Events](#6-events) **[B]**
7. [Conditional Rendering & Lists (Keys)](#7-conditional-rendering--lists-keys) **[B]**
8. [`useEffect` in Depth — and When NOT to Use It](#8-useeffect-in-depth--and-when-not-to-use-it) **[B/I]**
9. [All the Hooks](#9-all-the-hooks) **[I]**
10. [Custom Hooks](#10-custom-hooks) **[I]**
11. [Context for Shared State](#11-context-for-shared-state) **[I]**
12. [Refs & the DOM](#12-refs--the-dom) **[I]**
13. [Performance: memo, useMemo, useCallback & the React Compiler](#13-performance-memo-usememo-usecallback--the-react-compiler) **[I/A]**
14. [The `use()` API](#14-the-use-api) **[I/A]**
15. [Actions & Form Actions](#15-actions--form-actions) **[I/A]**
16. [`useActionState`, `useFormStatus`, `useOptimistic`](#16-useactionstate-useformstatus-useoptimistic) **[I/A]**
17. [`ref` as a Prop, Context as a Provider, Document Metadata](#17-ref-as-a-prop-context-as-a-provider-document-metadata) **[I/A]**
18. [Suspense, Concurrent Features & Data Fetching](#18-suspense-concurrent-features--data-fetching) **[A]**
19. [Error Handling & Error Boundaries](#19-error-handling--error-boundaries) **[I/A]**
20. [Server Components & Server Actions](#20-server-components--server-actions) **[A]**
21. [Migrating from React 18 to 19](#21-migrating-from-react-18-to-19) **[A]**
22. [Tips, Tricks & Gotchas](#22-tips-tricks--gotchas) **[I/A]**
23. [Study Path & Build-to-Learn Projects](#23-study-path--build-to-learn-projects)

---

## 1. What React Is & Why It Exists

React is a **JavaScript library for building user interfaces** out of small, reusable pieces called **components**. It is maintained by Meta and a large open-source community. To use React well you have to internalise *one* central idea, and everything else follows from it:

> **You describe what the UI should look like for a given state, and React figures out how to update the actual page to match.**

That sentence hides a deep shift in how you think about UI, so let us unpack it.

### 1.1 Imperative vs declarative — the core mental model **[B]**

Before React, the dominant way to build web UIs (jQuery, vanilla DOM) was **imperative**: you wrote step-by-step instructions that *mutated* the page. "Find the `<span>` with id `count`, read its text, add one, write it back; now find the button and disable it; now create a new `<li>` and append it to the list." The problem is that as the app grows, the number of these manual transitions explodes. Every piece of state can be reached from many code paths, the DOM and your JavaScript variables drift out of sync, and you spend your life chasing "why is this element showing the wrong thing?" bugs.

React is **declarative**. Instead of describing the *steps to change* the UI, you write a function that, given the current state, *returns a description of the whole UI right now*. When the state changes, React re-runs that function to get a fresh description and works out the minimal set of real-DOM changes needed to make the page match. You never touch the DOM by hand. You think only about "for *this* data, the screen looks like *this*" — a far smaller, more local problem.

The analogy: imperative code is like giving someone turn-by-turn driving directions; declarative code is like handing them the destination address and letting the GPS compute the route. You describe the *what*; React owns the *how*.

```tsx
// IMPERATIVE (vanilla JS) — you manually mutate the DOM on every change
let count = 0;
const btn = document.getElementById("btn")!;
btn.addEventListener("click", () => {
  count++;                                  // update the variable
  document.getElementById("out")!.textContent = `Count: ${count}`; // AND the DOM, by hand
});

// DECLARATIVE (React) — you describe the UI as a function of state; React syncs the DOM
function Counter() {
  const [count, setCount] = useState(0);    // the state
  // This JSX is a DESCRIPTION of the UI for the current count.
  // You never write "find the span and change its text" — React does that for you.
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}
```

### 1.2 Component-based — composing UI from functions **[B]**

A React app is a **tree of components**. A component is just a function that returns a description of some UI. Small components (a `Button`, an `Avatar`) compose into bigger ones (a `Toolbar`, a `Card`), which compose into pages, which compose into the whole app. This is the same "build big things out of small, named, reusable things" principle that functions bring to ordinary programming — applied to UI. The payoff is reuse, isolation (a bug in `Avatar` lives in `Avatar`), and the ability to reason about one piece at a time.

### 1.3 The virtual DOM & reconciliation — *why* React is fast **[I]**

Touching the real DOM is comparatively slow, and doing it carelessly (re-rendering the whole page on every change) would be unusably slow. React's solution is the **virtual DOM**: a lightweight JavaScript object tree that mirrors what the real DOM should look like. When state changes, React builds a *new* virtual tree by re-running your components, then **diffs** it against the previous virtual tree. This diffing process is called **reconciliation**. React computes the *smallest* set of real-DOM operations that turn the old UI into the new one, and applies only those.

The logic: comparing two plain JS objects in memory is cheap; mutating the real DOM is expensive. So React does the cheap thing (diff in memory) to *minimise* the expensive thing (DOM writes). This is why you can re-describe your entire UI on every state change without paying for re-rendering everything — React only changes what actually differs.

A few consequences worth knowing now (we return to each later):
- **Keys** (§7) help reconciliation match list items between renders so it can move rather than recreate them.
- **Immutability** (§5) is what lets React diff quickly: it can tell "did this change?" by comparing references (`oldState !== newState`) instead of deep-walking objects.
- **`React.memo`/`useMemo`** (§13) let you tell React "this subtree's inputs didn't change, skip diffing it entirely."

### 1.4 What React is NOT **[B]**

React is deliberately small — a *library*, not a *framework*. It does **not** include:
- **A router** — use React Router, TanStack Router, or the **Next.js** App Router.
- **A server / SSR runtime** — use **Next.js**, Remix, or a custom Node setup.
- **A global state manager** — use **Zustand**, Jotai, Redux Toolkit, or built-in Context + `useReducer`. (For *server* state — data fetched from an API — reach for **TanStack Query** instead of hand-rolling it.)
- **A build system** — use **Vite** (for SPAs), **Next.js**, or Parcel.
- **A styling solution** — use **Tailwind CSS**, CSS Modules, or any CSS-in-JS library.

This is a feature, not a gap: you assemble the stack that fits your project.

### 1.5 React 19 at a glance **[I]**

React 19 is the **largest API expansion since Hooks arrived in React 16.8**. It is the culmination of the multi-year "Concurrent React" effort and adds a first-class story for forms, mutations, async data, and server rendering. Here is the lay of the land versus React 18; every row is explained in depth in its own section.

| Area | React 18 | React 19 |
|---|---|---|
| Form handling | Manual state + `fetch` in handlers | Native `<form action={fn}>` with async Actions (§15) |
| Pending / loading state | Manual `isLoading` flag | `useActionState`, `useFormStatus` (§16) |
| Optimistic UI | Roll your own + manual rollback | `useOptimistic` built-in (§16) |
| Reading a promise in render | Only via a Suspense data lib | `use(promise)` anywhere (§14) |
| Reading context | `useContext(Ctx)` only, top-level | `use(Ctx)` — can be conditional (§14) |
| Forwarding a ref | `forwardRef()` wrapper required | `ref` is a plain prop (§17) |
| Context provider | `<MyCtx.Provider value={v}>` | `<MyCtx value={v}>` (§17) |
| Document `<head>` | react-helmet / next/head | `<title>`/`<meta>`/`<link>` work anywhere (§17) |
| Resource hints | Framework-only | `preload`/`preinit`/`prefetchDNS`/`preconnect` (§17) |
| Error reporting | `onRecoverableError` only | `onCaughtError` + `onUncaughtError` too (§19) |
| Auto memoization | Manual `useMemo`/`useCallback` | React Compiler does it (opt-in, §13) |
| `ReactDOM.render` | Deprecated | **Removed** — use `createRoot` (§21) |
| `propTypes`, string refs, legacy context | Deprecated | **Removed** — use TypeScript (§21) |

### 1.6 Installing & the entry point **[B]**

```bash
# New SPA with Vite (the standard non-framework choice in 2026)
npm create vite@latest my-app -- --template react-ts
cd my-app && npm install && npm run dev

# Or add/upgrade React in an existing project
npm install react@19 react-dom@19
npm install -D @types/react@19 @types/react-dom@19
npm list react        # confirm the version you actually have
```

Every React app has a single **entry point** that mounts your component tree into one real DOM node (usually `<div id="root">` in `index.html`). The mounting is done by `createRoot`.

```tsx
// main.tsx — the React 19 entry point
import { StrictMode } from "react";
import { createRoot } from "react-dom/client"; // createRoot is the ONLY mount API in React 19
import App from "./App";
import "./index.css";

// createRoot connects React to a real DOM node, then .render() draws your tree into it.
// The "!" is a TypeScript non-null assertion: we promise #root exists in index.html.
createRoot(document.getElementById("root")!).render(
  // <StrictMode> is a dev-only wrapper that double-invokes renders & effects to surface
  // bugs (impure renders, missing cleanup). It renders NOTHING and disappears in production.
  <StrictMode>
    <App />
  </StrictMode>
);
```

> **⚡ Version note:** `ReactDOM.render()` was deprecated in React 18 and is **fully removed in React 19**. When you upgrade an older project, every `ReactDOM.render(...)` call must become `createRoot(...).render(...)`. See §21.

> **StrictMode gotcha (very common beginner confusion):** in development, StrictMode intentionally **mounts every component twice** (mount → unmount → mount) and runs each Effect twice. This is *not* a bug and does *not* happen in production — it is React forcing you to write effects with proper cleanup. If your effect "runs twice," that is the point; fix the effect (§8), don't remove StrictMode.

---

## 2. JSX in Depth

### 2.1 What JSX actually is **[B]**

JSX is the HTML-looking syntax you write inside components. It is **not** HTML, and it is **not** a string — it is **syntactic sugar for function calls**. Your build tool (Vite, Next.js) compiles every JSX element into a call that produces a plain JavaScript object describing the element. That object is a node in the virtual DOM (§1.3).

Understanding that JSX is "just expressions that evaluate to objects" dissolves most beginner confusion. It explains why you can store JSX in a variable, return it from a function, put it in an array, and pass it as a prop — it is ordinary data.

```tsx
// What you write:
const element = <h1 className="title">Hello</h1>;

// What the compiler turns it into (the modern JSX transform, React 17+):
import { jsx as _jsx } from "react/jsx-runtime";
const element = _jsx("h1", { className: "title", children: "Hello" });
// → a plain object roughly like:
// { type: "h1", props: { className: "title", children: "Hello" } }
```

Because JSX is an expression, this all works:

```tsx
const greeting = <p>Hi there</p>;                 // store JSX in a variable
const list = [<li key="a">A</li>, <li key="b">B</li>]; // JSX in an array
function wrap(content: React.ReactNode) {           // JSX passed around as data
  return <div className="box">{content}</div>;
}
```

### 2.2 The rules of JSX, and the logic behind each **[B]**

JSX has a handful of rules. None are arbitrary — each falls out of "this compiles to JavaScript."

**Rule 1 — Return a single root element.** A function can only return one value, and each JSX element is one object. To return several siblings, wrap them. Use a **Fragment** (`<>...</>`) when you do not want an extra wrapper `<div>` polluting the DOM — a Fragment groups children but renders nothing itself.

```tsx
function MultiReturn() {
  return (
    <>                      {/* Fragment — groups siblings, adds NO real DOM node */}
      <h1>Title</h1>
      <p>Paragraph</p>
    </>
  );
}
```

**Rule 2 — Every tag must be closed.** Void HTML elements (`<input>`, `<br>`, `<img>`) are self-closing in JSX: `<input />`, `<br />`, `<img src="x" />`. This is required because JSX must be unambiguous to parse.

**Rule 3 — Attributes are camelCase.** JSX attributes become object keys, and HTML attribute names that clash with JS reserved words or hyphenation are renamed:

| HTML | JSX | Why |
|---|---|---|
| `class` | `className` | `class` is a reserved word in JS |
| `for` | `htmlFor` | `for` is a reserved word |
| `onclick` | `onClick` | camelCase convention for event props |
| `tabindex` | `tabIndex` | camelCase |
| `stroke-width` (SVG) | `strokeWidth` | hyphens are invalid in JS identifiers |

**Rule 4 — JavaScript goes inside `{ }`.** Curly braces drop you out of "JSX mode" and into "JavaScript expression mode." Anything that *evaluates to a value* is allowed — variables, function calls, arithmetic, ternaries. Statements (`if`, `for`) are **not**, because they do not produce a value.

```tsx
const items = ["apples", "bananas", "cherries"];
function ExpressionDemo() {
  const year = new Date().getFullYear();
  return (
    <p>
      Year: {year} | Count: {items.length} | Shout: {"hello".toUpperCase()}
    </p>
  );
}
```

**Rule 5 — `style` takes an object, not a string.** CSS properties are camelCased keys; values are usually strings (or plain numbers, which React treats as pixels).

```tsx
// Note the DOUBLE braces: outer {} = "JS expression", inner {} = "an object literal"
<div style={{ color: "red", fontSize: 16, marginTop: "1rem" }}>Styled</div>
```

> **Gotcha — JSX comments:** you cannot use `//` or `<!-- -->` inside JSX. Use `{/* ... */}` — a JS comment inside an expression slot.

> **Gotcha — what renders and what does not:** inside `{ }`, React renders strings, numbers, and JSX. It renders **nothing** for `null`, `undefined`, `false`, and `true` (handy for conditional rendering). It will *throw* if you try to render a plain object. And critically, it *does* render the number `0` — the source of the most common conditional-rendering bug (see §7).

---

## 3. Components & Props

### 3.1 What a component is **[B]**

A component is a **function whose name starts with a capital letter and which returns JSX (or `null`)**. The capital letter is mandatory and meaningful: in JSX, lowercase names (`<div>`) compile to DOM-tag strings, while capitalised names (`<Greeting>`) compile to references to *your* function. Write `greeting` and React will look for an HTML element called `<greeting>` and silently render nothing useful.

Components must be **pure** with respect to rendering: given the same props and state, a component must return the same JSX and must not cause side effects (no network calls, no DOM mutation, no writing to variables outside itself) *during render*. Purity is what makes React's render-anytime, render-twice (StrictMode), and skip-rendering optimisations safe. Side effects belong in event handlers or Effects (§8), never in the render body.

### 3.2 Props — the inputs to a component **[B]**

**Props** (properties) are how a parent passes data *down* to a child. They are the function arguments of a component. Two rules define them and you must hold both:

1. **Props flow one way: parent → child.** Data goes down the tree. A child cannot reach up and change its parent's data. (To send information *up*, the parent passes down a *callback* prop that the child calls — "data down, events up.")
2. **Props are read-only.** A component must never reassign or mutate its own props. Props are a snapshot of the parent's data for this render; treat them as immutable inputs.

In TypeScript you describe the props with an `interface` or `type`, and destructure them in the parameter list. You can give optional props default values right there with `=`.

```tsx
interface GreetingProps {
  name: string;
  age?: number;            // the "?" makes this prop optional
}

// Destructure props in the signature; give optional ones a default.
export function Greeting({ name, age = 0 }: GreetingProps) {
  return (
    <div className="greeting">
      <h1>Hello, {name}!</h1>
      {age > 0 && <p>You are {age} years old.</p>}
    </div>
  );
}

// Usage — passing props looks like HTML attributes:
<Greeting name="Alice" age={30} />
<Greeting name="Bob" />        {/* age omitted → defaults to 0 → paragraph not shown */}
```

### 3.3 The `children` prop & prop spreading **[B/I]**

`children` is a **special prop**: it is whatever you put *between* a component's opening and closing tags. This is how you build wrapper/layout components that do not need to know what they contain.

The **spread operator** (`{...rest}`) forwards a bag of leftover props onto an element — invaluable for building wrapper components around DOM elements (a custom `Button` that still accepts every native `<button>` attribute).

```tsx
// Extend the native button attributes so our Button accepts onClick, disabled, type, etc.
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: "primary" | "secondary";
}

function Button({ variant = "primary", children, ...rest }: ButtonProps) {
  return (
    // {...rest} forwards every remaining native prop straight to the <button>.
    <button className={`btn btn-${variant}`} {...rest}>
      {children}            {/* whatever was placed between <Button> ... </Button> */}
    </button>
  );
}

<Button variant="secondary" onClick={() => alert("hi")} disabled={false}>
  Click Me                  {/* this text becomes the `children` prop */}
</Button>
```

> **Best practice — keep components small and focused.** A component that does one thing is easy to test, reuse, and reason about. When a component grows past ~150 lines or juggles several unrelated concerns, split it. Extract repeated logic into **custom hooks** (§10), not into ever-larger components.

> **Best practice — type `children` with `ReactNode`.** When a component takes children, type them as `React.ReactNode` (which covers JSX, strings, numbers, arrays, and `null`). The `PropsWithChildren<P>` utility does this for you.

---

## 4. Rendering & the Render Cycle

This is the section that, once it clicks, makes React stop feeling like magic. Read it slowly.

### 4.1 What "render" means **[B/I]**

In React, **"render" means: React calls your component function to get the JSX it returns.** That is it. Rendering is *not* the same as "drawing to the screen" — that later step is called **committing** (or painting). Separating these two ideas is essential.

A render happens in three phases:

1. **Trigger.** Something asks React to render. There are exactly two triggers: (a) the initial mount (your first `createRoot(...).render()`), and (b) a **state update** (calling a `setState` function). Props changing does *not* independently trigger a render — a child re-renders because its *parent* re-rendered and passed it new props.
2. **Render.** React calls the component function (and its children), producing a new virtual-DOM tree, and **reconciles** it against the previous one (§1.3) to find what changed. *No DOM is touched yet.*
3. **Commit.** React applies the minimal computed DOM changes. *Then* the browser paints. **`useEffect`** callbacks run *after* this paint; **`useLayoutEffect`** runs *during* commit, before paint (§8, §9).

### 4.2 Re-renders cascade down, and state is a snapshot **[I]**

When a component re-renders, React **also re-renders all of its descendants** by default (it must call them to know what they now return). This sounds expensive but usually is not, because reconciliation only *commits* the differences — and you can short-circuit re-rendering a subtree with `React.memo` when it is genuinely needed (§13). The takeaway: a state update in a parent re-runs the parent and everything below it.

The single most important nuance: **state is a snapshot for each render.** When React renders, it captures the values of state and props *for that render*. Inside that render's functions (event handlers, effects), those values are **constants** — they never change mid-render. Calling `setState` does not mutate the current `count` variable; it asks React to render *again*, and the *next* render gets the new value.

```tsx
function SnapshotDemo() {
  const [count, setCount] = useState(0);

  function handleClick() {
    // Each setCount queues an update based on the SAME snapshot value of `count` (say, 0).
    setCount(count + 1);   // queues: set to 0 + 1 = 1
    setCount(count + 1);   // queues: set to 0 + 1 = 1  (count is STILL 0 in this snapshot!)
    setCount(count + 1);   // queues: set to 0 + 1 = 1
    // Result after this click: count is 1, NOT 3. This surprises everyone once.
  }

  function handleClickFixed() {
    // The UPDATER form receives the latest pending value, so updates compose correctly.
    setCount((c) => c + 1);  // 0 → 1
    setCount((c) => c + 1);  // 1 → 2
    setCount((c) => c + 1);  // 2 → 3   ✅ result: 3
  }

  return (
    <>
      <p>{count}</p>
      <button onClick={handleClick}>+3 (broken: ends at 1)</button>
      <button onClick={handleClickFixed}>+3 (correct, updater form)</button>
    </>
  );
}
```

### 4.3 Batching **[I]**

React **batches** multiple state updates that happen in the same event into a *single* re-render, for performance — you will not see three flickers from three `setState` calls. In React 18+ this batching applies *everywhere*, including inside `setTimeout`, promises, and native event handlers (React 17 only batched inside React events). You almost never need to think about batching except to remember: *all the `setState` calls in one handler produce one render.*

> **Mental model summary:** State update → React re-runs the component (and children) → diffs the result → commits only the differences → browser paints → effects run. Each render sees a frozen snapshot of its state and props.

### 4.4 A worked render trace **[I]**

Trace what happens when a user clicks a button that increments a counter, to cement the model end-to-end:

```text
Initial mount
  1. createRoot(...).render(<Counter/>)         (TRIGGER: initial render)
  2. React calls Counter()  → count is 0        (RENDER)
  3. React builds virtual tree: <button>Count: 0</button>
  4. No previous tree → React creates real DOM nodes  (COMMIT)
  5. Browser paints "Count: 0"
  6. useEffect callbacks (if any) run

User clicks the +1 button
  7. onClick fires → setCount(c => c + 1)        (TRIGGER: state update)
  8. React schedules a re-render of Counter
  9. React calls Counter() again → count is now 1 (RENDER)
 10. New virtual tree: <button>Count: 1</button>
 11. React DIFFS new vs old → only the text "0"→"1" changed  (RECONCILE)
 12. React updates ONLY that text node in the real DOM  (COMMIT — not the whole button)
 13. Browser paints "Count: 1"
 14. Effects whose dependencies changed re-run (after cleanup of the prior run)
```

The lesson: re-rendering re-runs your function and rebuilds the *virtual* tree, but commits only the *real* difference (one text node). This is why "React re-renders the whole component" is cheap — the expensive DOM work is minimal and targeted.

---

## 5. State with `useState` & Why Immutability Matters

### 5.1 What state is and why props are not enough **[B]**

**State** is data that a component *owns* and that *changes over time in response to interaction*, causing the UI to update. Props come from the parent and are read-only; state is local, mutable-over-time, and private to the component. A text input's current value, whether a dropdown is open, the items in a cart — all state.

You declare state with the **`useState` hook**. It returns a pair: the **current value** for this render, and a **setter function** that, when called, tells React "the value changed — please re-render with the new one." You will almost always name them `[thing, setThing]`.

The reason state has to live in `useState` rather than a plain variable: a plain `let count = 0` inside a component is re-created from scratch on every render, so it can never *persist* a change, and reassigning it would not tell React to re-render. `useState` solves both — React *remembers* the value across renders and *re-renders* when you set it.

```tsx
import { useState } from "react";

function Counter() {
  // useState(initialValue) → [currentValue, setterFunction]
  // TypeScript infers the type (number) from the initial value 0.
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Count: {count}</p>
      {/* Use the UPDATER form whenever next state depends on previous state (see §4.2). */}
      <button onClick={() => setCount((prev) => prev + 1)}>+</button>
      <button onClick={() => setCount((prev) => prev - 1)}>-</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}
```

**Key parameters / forms of the setter:**
- `setCount(5)` — set to an exact new value.
- `setCount(prev => prev + 1)` — the **updater function** form; receives the latest pending value. Use this whenever the new value depends on the old one, to avoid the snapshot trap from §4.2.

### 5.2 Why immutability matters — the deepest "why" in React **[B/I]**

Here is the rule, then the reason: **never mutate state directly. Always create a new value and pass it to the setter.**

Why? Two intertwined reasons:

1. **React decides whether to re-render by comparing references, not contents.** When you call a setter, React asks "is the new value the *same object* as the old one?" (a cheap `Object.is` reference check). If you *mutate* the existing array/object and pass the *same reference* back, React sees "same reference → nothing changed" and **skips the re-render** — your UI goes stale even though the data changed. Creating a *new* object/array gives a new reference, so React knows to update. This same reference-equality check powers `memo`, `useMemo`, and effect dependency arrays (§13, §8) — immutability is the foundation the whole optimisation story rests on.
2. **Renders must be reproducible.** React may render a component multiple times, in the background, or discard a render. If your render or update mutates shared data, those extra/discarded renders corrupt your state. Treating state as immutable keeps every render a pure function of its inputs.

So: to "change" an object or array in state, you build a **new** one with the change applied — typically with the spread operator (`...`) or array methods that *return* new arrays (`map`, `filter`, `concat`), never the ones that *mutate* (`push`, `splice`, `sort`, direct index assignment).

```tsx
// ── OBJECT STATE: spread the old object, override the changed field ──
interface UserForm { name: string; email: string; age: number; }

function UserFormDemo() {
  const [form, setForm] = useState<UserForm>({ name: "", email: "", age: 0 });

  function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    setForm((prev) => ({
      ...prev,                          // copy ALL existing fields into a NEW object
      [e.target.name]: e.target.value,  // then overwrite just the one that changed
    }));
    // ❌ NEVER:  prev.name = e.target.value;  setForm(prev)  — same reference, no re-render
  }

  return (
    <form>
      <input name="name"  value={form.name}  onChange={handleChange} />
      <input name="email" value={form.email} onChange={handleChange} />
    </form>
  );
}

// ── ARRAY STATE: use methods that RETURN new arrays ──
function TodoCrud() {
  const [todos, setTodos] = useState<{ id: number; text: string; done: boolean }[]>([]);

  const add = (text: string) =>
    setTodos((t) => [...t, { id: Date.now(), text, done: false }]); // spread + new item

  const remove = (id: number) =>
    setTodos((t) => t.filter((todo) => todo.id !== id));            // filter returns a new array

  const toggle = (id: number) =>
    setTodos((t) =>
      t.map((todo) =>                                                // map returns a new array
        todo.id === id ? { ...todo, done: !todo.done } : todo        // new object for changed item
      )
    );

  return null; // (UI omitted for brevity)
}
```

> **Best practice for deeply nested state:** spreading several levels deep gets ugly fast (`{...s, a: {...s.a, b: {...s.a.b, c: 1}}}`). When you hit that, either (a) flatten/normalise your state shape, (b) switch to **`useReducer`** (§9) to centralise update logic, or (c) use the **Immer** library, which lets you "mutate" a draft and produces the immutable copy for you. Immutability is the requirement; Immer is just a convenient way to satisfy it.

### 5.3 Lazy initialisation & choosing initial state **[I]**

`useState`'s argument is the initial value **used only on the first render** and ignored thereafter. If computing that initial value is expensive (parsing localStorage, a heavy calculation), pass a **function** instead of a value — React calls it once on mount rather than re-running the expensive expression on every render and throwing the result away.

```tsx
// ❌ readFromStorage() RUNS ON EVERY RENDER (result discarded after the first):
const [data, setData] = useState(readFromStorage());

// ✅ LAZY INITIALISER: the function runs ONCE, on mount only.
const [data2, setData2] = useState(() => readFromStorage());

function readFromStorage() {
  return JSON.parse(localStorage.getItem("savedData") ?? "null") ?? {};
}
```

> **Gotcha — don't derive state you can compute.** If a value can be *calculated* from existing props/state, do **not** put it in its own `useState`; just compute it during render. Storing derived data invites the two copies to drift out of sync (the classic "filtered list stored in state that forgets to update" bug). Reserve state for genuinely independent, changing data. See pitfall §22.

### 5.4 Where state lives: lifting state up & controlled inputs **[B/I]**

A question you face constantly: *which component should own a given piece of state?* The principle is **single source of truth** — each piece of state lives in exactly one component, and that component should be the **closest common ancestor** of all the components that need it. When two sibling components must share or sync state, you cannot pass it sideways (data only flows down); instead you **lift the state up** to their shared parent, which holds the state and passes both the *value* (down as a prop) and a *setter callback* (down as a prop) to the children. The children become "controlled" by the parent: they display the value they are given and report changes back up via the callback. This is the "data down, events up" pattern in action.

The same idea explains **controlled inputs** — the standard way to handle form fields in React. A controlled `<input>` has its `value` driven by state and an `onChange` that updates that state. React is the single source of truth for what the input shows; the DOM input never holds its own independent value. This makes the value available to validate, transform, or submit at any time.

```tsx
import { useState } from "react";

// ── Lifting state up: two children share one temperature, owned by the parent ──
function Thermostat() {
  // The PARENT owns the shared state — the single source of truth.
  const [celsius, setCelsius] = useState(20);

  return (
    <div>
      {/* Pass the value DOWN and a setter callback DOWN to each child. */}
      <TempInput
        label="°C"
        value={celsius}
        onChange={setCelsius}
      />
      <TempInput
        label="°F"
        value={Math.round(celsius * 9 / 5 + 32)}            // derived, not stored
        onChange={(f) => setCelsius(Math.round((f - 32) * 5 / 9))} // convert back up
      />
      <p>{celsius >= 100 ? "Boiling!" : "Not boiling"}</p>
    </div>
  );
}

// A controlled, "dumb" child: it owns NO state — it shows what it's told and reports changes up.
function TempInput({ label, value, onChange }: {
  label: string;
  value: number;
  onChange: (n: number) => void;
}) {
  return (
    <label>
      {label}
      <input
        type="number"
        value={value}                                 // value is DRIVEN by the parent's state
        onChange={(e) => onChange(Number(e.target.value))} // report change UP
      />
    </label>
  );
}
```

> **Controlled vs uncontrolled inputs:** the input above is *controlled* (React owns the value). An *uncontrolled* input lets the DOM keep the value and you read it via a ref or `FormData` only when needed (§6, §15). Controlled gives you live access and validation; uncontrolled is simpler and is what React 19 form **Actions** lean on (they read `FormData` at submit time). For complex forms, **React Hook Form** uses a mostly-uncontrolled approach for performance — fewer re-renders per keystroke.

> **Best practice — collocate state, then lift only as needed.** Start with state as *low* (local) as possible; lift it up only when a sibling genuinely needs it. State lifted higher than necessary causes larger re-render subtrees (§13) and more prop-passing. If lifting leads to deep prop-drilling, that is the signal to reach for Context (§11) or a store like Zustand.

---

## 6. Events

### 6.1 How React events work **[B]**

You respond to user interaction by passing **event handler** functions to event props like `onClick`, `onChange`, `onSubmit`. The convention: the prop is camelCase and you pass it a *function reference* (not a function *call*). `onClick={handleClick}` is right; `onClick={handleClick()}` is wrong — the latter calls the function immediately during render and passes its return value.

React's event objects are **SyntheticEvents** — thin, cross-browser wrappers around the native DOM event. They expose the same API (`e.preventDefault()`, `e.stopPropagation()`, `e.target`, `e.currentTarget`) and behave consistently across browsers, so you do not have to worry about browser quirks. Under the hood React uses a single delegated listener at the root for efficiency, but you do not need to think about that — it behaves as if attached to the element.

```tsx
import { type MouseEvent, type FormEvent, type ChangeEvent } from "react";

function EventDemo() {
  // Type the event by the element it fires on — you get full autocomplete on e.
  function handleClick(e: MouseEvent<HTMLButtonElement>) {
    e.preventDefault();                                   // works exactly like native
    console.log("Clicked at", e.clientX, e.clientY);
  }

  function handleSubmit(e: FormEvent<HTMLFormElement>) {
    e.preventDefault();                                   // stop the full-page reload!
    const data = new FormData(e.currentTarget);           // read all named form fields
    console.log(data.get("username"));
  }

  function handleChange(e: ChangeEvent<HTMLInputElement>) {
    console.log("New value:", e.target.value);            // e.target = the input that fired
  }

  return (
    <>
      <button onClick={handleClick}>Click me</button>

      {/* Inline arrow: fine for tiny handlers or to pass arguments. Note it creates a
          NEW function each render — usually harmless, occasionally a perf concern (§13). */}
      <button onClick={() => console.log("inline")}>Inline</button>

      <form onSubmit={handleSubmit}>
        <input name="username" onChange={handleChange} />
        <button type="submit">Submit</button>
      </form>
    </>
  );
}
```

### 6.2 Passing arguments & the two common patterns **[B]**

To pass an argument to a handler, wrap it in an inline arrow so the call happens *when the event fires*, not during render:

```tsx
// ✅ arrow defers the call until click time, capturing the id
<button onClick={() => deleteItem(item.id)}>Delete</button>

// ❌ this CALLS deleteItem during render, every render, immediately
<button onClick={deleteItem(item.id)}>Delete</button>
```

> **Gotcha — `e.preventDefault()` on forms.** A native `<form>` reloads the page on submit. In a classic controlled form you must call `e.preventDefault()` in `onSubmit` to stop that. React 19's new form **Actions** (`<form action={fn}>`, §15) handle this for you — another reason to prefer them for new code.

> **Best practice — `currentTarget` vs `target`.** `e.target` is the element that *originated* the event (could be a child); `e.currentTarget` is the element the handler is *attached to*. For form data, read from `e.currentTarget`.

---

## 7. Conditional Rendering & Lists (Keys)

### 7.1 Conditional rendering **[B]**

Because JSX is just expressions, "showing different UI for different state" uses ordinary JavaScript. There are three idioms, each suited to a situation:

- **`if`/early return** — clearest for whole-component branches (e.g. loading vs error vs data).
- **Ternary `cond ? a : b`** — inline either/or inside JSX.
- **`&&` short-circuit** — render something *or nothing*. `cond && <X/>` renders `<X/>` when `cond` is truthy and `false` (which React ignores) otherwise.

```tsx
interface UserProps { isLoggedIn: boolean; role?: "admin" | "user"; }

function UserPanel({ isLoggedIn, role }: UserProps) {
  // 1. Early return — best for top-level branching.
  if (!isLoggedIn) return <p>Please log in.</p>;

  return (
    <div>
      {/* 2. Ternary — inline either/or. */}
      <p>{role === "admin" ? "Admin Dashboard" : "User Dashboard"}</p>

      {/* 3. && — render or nothing. */}
      {role === "admin" && <button>Delete All Users</button>}
    </div>
  );
}
```

> **Gotcha — the `0` trap (the most common conditional bug).** `&&` returns its *left* operand when that operand is falsy, and React **renders the number `0`** (unlike `false`/`null`/`undefined`, which render nothing). So `{items.length && <List/>}` renders a stray `0` on the screen when the array is empty, because `0 && ...` evaluates to `0`. Always make the left side a real boolean:

```tsx
{items.length > 0 && <List items={items} />}   // ✅ boolean → renders List or nothing
{items.length && <List items={items} />}        // ❌ renders "0" when empty
{!!items.length && <List items={items} />}       // ✅ alternative: coerce to boolean with !!
```

### 7.2 Rendering lists with `.map()` **[B]**

To render a collection, map each data item to a JSX element. `.map()` returns an array of elements, and React renders an array of elements in order.

### 7.3 Keys — what they are and *why* they are required **[B/I]**

Every element in a rendered list **must have a `key` prop** — a string or number that is **stable** (same item → same key across renders) and **unique among its siblings**. This is not bureaucracy; keys are how reconciliation works for lists.

The logic: when a list re-renders, React needs to match each *new* element to the *old* element it corresponds to, so it can decide what to reuse, move, add, or remove. Without a hint, React can only match by position — so if you insert an item at the top, React thinks *every* item changed (each shifted one slot) and may rebuild the whole list, losing input focus, scroll position, and component state along the way. A `key` gives React an *identity* for each item independent of position, so it can move existing DOM/state to the right place and only touch what truly changed.

```tsx
interface Product { id: number; name: string; price: number; }
const products: Product[] = [
  { id: 1, name: "Widget",   price: 9.99 },
  { id: 2, name: "Gadget",   price: 24.99 },
  { id: 3, name: "Doohickey", price: 4.99 },
];

function ProductList() {
  return (
    <ul>
      {products.map((p) => (
        // key = a STABLE, UNIQUE id. Use the data's own id, not the array index.
        <li key={p.id}>{p.name} — ${p.price.toFixed(2)}</li>
      ))}
    </ul>
  );
}

// When a list item needs to be a Fragment WITH a key, use the explicit <Fragment> form
// (the <>...</> shorthand cannot take a key):
import { Fragment } from "react";
function DescriptionList({ items }: { items: { id: number; term: string; def: string }[] }) {
  return (
    <dl>
      {items.map((item) => (
        <Fragment key={item.id}>
          <dt>{item.term}</dt>
          <dd>{item.def}</dd>
        </Fragment>
      ))}
    </dl>
  );
}
```

> **Gotcha — never use the array index as a key for dynamic lists.** If the list can reorder, filter, sort, or have items inserted/removed anywhere but the end, `key={index}` reattaches the wrong state/DOM to the wrong item (because index 2 means a different item after a reorder). Index keys are only acceptable for a list that is static and never reordered. Always prefer a stable id from your data.

---

## 8. `useEffect` in Depth — and When NOT to Use It

`useEffect` is the most powerful and the most *misused* hook. Beginners reach for it constantly when they should not. Read this section carefully; getting the mental model right will save you a hundred bugs.

### 8.1 The mental model: synchronising with the outside world **[B/I]**

An **Effect** lets your component **synchronise with a system *outside* of React** — the network, the DOM directly, a browser API, a timer, a third-party widget, a subscription. The right mental frame is *not* "run this code when X happens" (that is what event handlers are for). It is: **"keep this external thing in sync with my component's current props/state."**

The logic: rendering must be pure (§3.1), so side effects cannot live in the render body. But you genuinely need to *do* things — fetch data, set up a subscription, focus an input. `useEffect` is React's escape hatch for that: code that runs *after* React has rendered and committed to the DOM, at a moment when touching the outside world is safe. React then *re-runs* or *cleans up* that effect as your dependencies change, so the external thing tracks your state.

```tsx
useEffect(() => {
  // 1. SETUP: connect to / start the external thing.
  // ...
  return () => {
    // 2. CLEANUP (optional): disconnect / stop it. Runs before the next setup AND on unmount.
  };
}, [/* 3. DEPENDENCIES: re-run when any of these change */]);
```

### 8.2 The dependency array — the control knob **[B/I]**

The second argument controls *when* the effect re-runs, and it is where most bugs live:

- **`[]` (empty)** — run the setup once after the first render (mount); run cleanup once on unmount. For "set up once" things (a subscription, an initial fetch with no inputs).
- **`[a, b]`** — run after the first render, then again whenever `a` or `b` changes (by `Object.is` reference comparison). This is the common case: "re-sync when these inputs change."
- **omitted** — run after *every* render. Almost always a mistake (and an easy way to make an infinite loop if the effect sets state).

The golden rule, enforced by the `react-hooks/exhaustive-deps` ESLint rule: **the dependency array must list every reactive value (prop, state, or value derived from them) that the effect reads.** If you read it, declare it. Omitting a dependency causes a **stale closure** — the effect captured an old value and keeps using it (see §22 pitfall 2). Do not "trick" the linter by leaving deps out; fix the root cause (use the updater form, move the value, or memoise it).

### 8.3 Cleanup — the half everyone forgets **[B/I]**

The function you *return* from an effect is the **cleanup**. React runs it (a) right before re-running the effect with new dependencies, and (b) when the component unmounts. Cleanup is how you undo setup: remove the event listener you added, clear the interval you started, unsubscribe from the store, abort the in-flight request. Forgetting cleanup leaks memory and stacks up duplicate listeners/subscriptions. (StrictMode's double-mount in dev exists precisely to catch missing cleanup — if your effect misbehaves on the second mount, your cleanup is incomplete.)

```tsx
import { useEffect, useState, useRef } from "react";

// ── DATA FETCHING with cleanup (the cancellation flag pattern) ──
function DataFetcher({ userId }: { userId: number }) {
  const [user, setUser] = useState<{ name: string } | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    let cancelled = false;                 // guards against setting state after unmount/re-run

    async function fetchUser() {
      setLoading(true);
      setError(null);
      try {
        const res = await fetch(`/api/users/${userId}`);
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const data = await res.json();
        if (!cancelled) setUser(data);     // only apply if this effect run is still relevant
      } catch (err) {
        if (!cancelled) setError(err as Error);
      } finally {
        if (!cancelled) setLoading(false);
      }
    }
    fetchUser();

    // CLEANUP: if userId changes (or we unmount) before the fetch resolves, ignore the result.
    return () => { cancelled = true; };
  }, [userId]);                            // re-fetch whenever userId changes

  if (loading) return <p>Loading…</p>;
  if (error)   return <p>Error: {error.message}</p>;
  return <p>User: {user?.name}</p>;
}

// ── SUBSCRIBING to a browser event (cleanup removes the listener) ──
function WindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);
  useEffect(() => {
    const onResize = () => setWidth(window.innerWidth);
    window.addEventListener("resize", onResize);
    return () => window.removeEventListener("resize", onResize); // ← cleanup is mandatory
  }, []);                                  // set up once, tear down on unmount
  return <p>Width: {width}px</p>;
}

// ── SYNCHRONISING with a third-party library (create on mount, update on prop change) ──
function MapWidget({ center }: { center: [number, number] }) {
  const mapRef = useRef<HTMLDivElement>(null);
  const instance = useRef<{ setCenter(c: [number, number]): void; destroy(): void } | null>(null);

  useEffect(() => {
    if (!mapRef.current) return;
    instance.current = new (window as any).SomeMapLib(mapRef.current, { center });
    return () => instance.current?.destroy();   // tear the map down on unmount
  }, []);

  useEffect(() => {
    instance.current?.setCenter(center);         // keep it in sync when `center` changes
  }, [center]);

  return <div ref={mapRef} style={{ width: "100%", height: 400 }} />;
}
```

### 8.4 When NOT to use `useEffect` — the big beginner trap **[I]**

This deserves its own heading because over-using effects is the #1 source of buggy, slow React code. Before reaching for `useEffect`, ask: *"Am I synchronising with an external system?"* If the answer is no, you almost certainly do **not** need an effect.

Common cases where beginners wrongly use an effect:

1. **Transforming data for rendering.** If you can compute a value from existing props/state, compute it *during render* — do **not** mirror it into state via an effect. An effect here causes an extra render and a chance to drift out of sync.
   ```tsx
   // ❌ effect + state to derive a value
   const [fullName, setFullName] = useState("");
   useEffect(() => { setFullName(first + " " + last); }, [first, last]);
   // ✅ just compute it
   const fullName = first + " " + last;
   ```
2. **Responding to a user event.** Logic that should run *because the user did something* belongs in the **event handler**, not an effect watching the resulting state. (E.g. send analytics or a POST in the click handler, not in an effect.)
3. **Resetting state when a prop changes.** Instead of an effect that clears state, give the component a different **`key`** so React remounts it fresh (§22 tip 15).
4. **Fetching data in a real app.** You *can* fetch in an effect (§8.3), and it is fine for learning, but in production it has rough edges (race conditions, no caching, waterfalls, refetch-on-focus). Prefer a data library — **TanStack Query** for client fetching, or **Server Components / the `use()` API** (§14, §20) — which solve caching, dedup, and cancellation for you.

> **Best practice:** treat `useEffect` as a last resort for talking to the world outside React, not as a general "run code after state changes" tool. The official docs literally have a page titled *"You Might Not Need an Effect."* Internalise that.

---

## 9. All the Hooks

Hooks are functions whose names start with `use`. They let function components "hook into" React features (state, lifecycle, context). Two **Rules of Hooks** are non-negotiable, and the reason is the same for both:

1. **Only call hooks at the top level** — never inside loops, conditions, or nested functions.
2. **Only call hooks from React function components or other custom hooks** — not from regular functions.

The logic behind the rules: React identifies which hook is which purely by **call order**, not by name. The first `useState` in a component is "slot 0," the second is "slot 1," and so on, every render. If a hook call is conditional, the order shifts between renders and React hands back the wrong slot's value — corrupting state. Keeping calls unconditional and top-level guarantees a stable order. (The one exception is the new `use()` API in §14, which *is* allowed in conditions.)

Below is every core hook with what/why/when. `useState` (§5) and `useEffect` (§8) were covered in depth; here are the rest.

### 9.1 `useContext` — read shared data without prop-drilling **[I]**

Reads the current value of a Context (§11). Use it to consume data many components need (theme, current user, locale) without threading props through every intermediate component. Returns the value provided by the nearest matching Provider above it.

```tsx
import { createContext, useContext, useState, type ReactNode } from "react";

interface ThemeCtx { theme: "light" | "dark"; toggle: () => void; }
const ThemeContext = createContext<ThemeCtx>({ theme: "light", toggle: () => {} });

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<"light" | "dark">("light");
  const toggle = () => setTheme((t) => (t === "light" ? "dark" : "light"));
  // 🆕 React 19: render the context object directly as the provider (no .Provider). See §17.
  return <ThemeContext value={{ theme, toggle }}>{children}</ThemeContext>;
}

function ThemedButton() {
  const { theme, toggle } = useContext(ThemeContext);  // read the nearest provider's value
  return <button onClick={toggle}>Theme: {theme}</button>;
}
```

### 9.2 `useReducer` — centralised state transitions **[I]**

An alternative to `useState` for **complex state** where the next state depends on the previous in several ways, or where several values change together. You write a **reducer** — a pure function `(state, action) => newState` — and dispatch **actions** (objects describing *what happened*). The component just dispatches; *all* the update logic lives in one tested, predictable function. This is the same idea Redux popularised, built into React.

**When to choose it over `useState`:** multiple sub-values that change in concert, state machines (idle/loading/success/error), or when update logic is sprawling enough that you want it out of the component and unit-testable.

```tsx
import { useReducer } from "react";

type Action =
  | { type: "increment" } | { type: "decrement" } | { type: "reset" }
  | { type: "set"; payload: number };
interface State { count: number; history: number[]; }

// Reducer = PURE: no fetch, no mutation, no side effects. Same (state, action) → same result.
function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "increment": return { count: state.count + 1, history: [...state.history, state.count + 1] };
    case "decrement": return { count: state.count - 1, history: [...state.history, state.count - 1] };
    case "set":       return { count: action.payload, history: [...state.history, action.payload] };
    case "reset":     return { count: 0, history: [] };
    default:          return state;        // unknown action → return state unchanged
  }
}

function ReducerCounter() {
  const [state, dispatch] = useReducer(reducer, { count: 0, history: [] });
  return (
    <div>
      <p>Count: {state.count} | History: {state.history.join(", ")}</p>
      <button onClick={() => dispatch({ type: "increment" })}>+</button>
      <button onClick={() => dispatch({ type: "set", payload: 100 })}>Set 100</button>
      <button onClick={() => dispatch({ type: "reset" })}>Reset</button>
    </div>
  );
}
```

### 9.3 `useRef` — a mutable box that does not trigger renders **[I]**

`useRef` returns an object `{ current: ... }` that **persists across renders** but whose mutation does **not** cause a re-render. Two distinct uses: (1) holding a reference to a **DOM element** (pass it to the `ref` attribute), and (2) storing a **mutable value** you want to remember between renders without it being "reactive" (a timer id, a previous value, a flag). Covered in depth with the DOM in §12.

```tsx
import { useRef, useEffect, useState } from "react";

// Use 1: DOM access — focus an input after mount.
function AutoFocusInput() {
  const inputRef = useRef<HTMLInputElement>(null);
  useEffect(() => { inputRef.current?.focus(); }, []);
  return <input ref={inputRef} placeholder="auto-focused" />;
}

// Use 2: store a mutable value (interval id) without re-rendering when it changes.
function Timer() {
  const [count, setCount] = useState(0);
  const idRef = useRef<ReturnType<typeof setInterval> | null>(null);
  useEffect(() => {
    idRef.current = setInterval(() => setCount((c) => c + 1), 1000);
    return () => { if (idRef.current) clearInterval(idRef.current); };
  }, []);
  return <p>Seconds: {count}</p>;
}
```

### 9.4 `useMemo` — cache an expensive *value* **[I]**

`useMemo(factory, deps)` runs `factory` and **caches its return value**, recomputing only when a dependency changes. Two reasons to use it: (1) skip an *genuinely expensive* calculation on renders where its inputs did not change, and (2) preserve a *stable reference* for an object/array you pass to a memoised child or an effect dependency (so it does not look "changed" every render). Do not sprinkle it everywhere — it has its own cost, and the React Compiler (§13) increasingly makes it unnecessary.

```tsx
import { useMemo, useState } from "react";
function FilteredNumbers({ numbers, threshold }: { numbers: number[]; threshold: number }) {
  const [, force] = useState(0);
  // Recomputes ONLY when numbers or threshold changes — not when unrelated state forces a render.
  const filtered = useMemo(
    () => numbers.filter((n) => n > threshold).sort((a, b) => a - b),
    [numbers, threshold]
  );
  return (
    <>
      <button onClick={() => force((n) => n + 1)}>Re-render</button>
      <ul>{filtered.map((n) => <li key={n}>{n}</li>)}</ul>
    </>
  );
}
```

### 9.5 `useCallback` — cache a *function* reference **[I]**

`useCallback(fn, deps)` is `useMemo` specialised for functions: it returns the *same function reference* across renders until a dependency changes. Its purpose is almost always to keep a callback stable so that a **memoised child** (`React.memo`) does not re-render just because the parent created a new function. Without it, every parent render makes a new function → new prop reference → memo'd child re-renders anyway. Like `useMemo`, the React Compiler can do this for you.

```tsx
import { useCallback, useState, memo } from "react";

const Child = memo(function Child({ label, onClick }: { label: string; onClick: () => void }) {
  console.log("render", label);           // logged only when label or onClick actually changes
  return <button onClick={onClick}>{label}</button>;
});

function Parent() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);
  // Same reference across renders → Child B doesn't re-render when only `a` changes.
  const onB = useCallback(() => setB((x) => x + 1), []);
  const onA = useCallback(() => setA((x) => x + 1), []);
  return (
    <>
      <Child label={`A:${a}`} onClick={onA} />
      <Child label={`B:${b}`} onClick={onB} />
    </>
  );
}
```

### 9.6 `useLayoutEffect` — measure the DOM before paint **[A]**

Identical API to `useEffect`, but it runs **synchronously after the DOM is mutated and *before* the browser paints**. Use it (rarely) when you must *read* layout (measure an element) and *change* it *before* the user sees a flash — e.g. positioning a tooltip so it does not flicker into the wrong spot. Because it blocks painting, prefer `useEffect` for 99% of effects; reach for `useLayoutEffect` only to prevent a visible layout flicker.

```tsx
import { useLayoutEffect, useRef } from "react";
function Tooltip({ text }: { text: string }) {
  const ref = useRef<HTMLDivElement>(null);
  useLayoutEffect(() => {
    const rect = ref.current?.getBoundingClientRect();
    // Reposition BEFORE paint so the user never sees it off-screen.
    if (rect && rect.right > window.innerWidth) ref.current!.style.transform = "translateX(-100%)";
  });
  return <div ref={ref} className="tooltip">{text}</div>;
}
```

### 9.7 `useId` — stable unique ids for accessibility **[I]**

Generates a unique, **SSR-safe** id string that is identical on server and client (avoiding hydration mismatches). Use it to wire `<label htmlFor>` to an input's `id`, or for `aria-describedby`. Do **not** use it for list keys.

```tsx
import { useId } from "react";
function Field({ label }: { label: string }) {
  const id = useId();                       // e.g. ":r3:" — stable across SSR/CSR
  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </>
  );
}
```

### 9.8 `useTransition` — mark updates as non-urgent **[A]**

Lets you flag a state update as a low-priority **transition** so React can keep the UI responsive (e.g. keep typing snappy) while the expensive update happens in the background, and gives you an `isPending` flag to show progress. Covered with the concurrent features in §18.

### 9.9 `useDeferredValue` — a lagging copy of a value **[A]**

Returns a "deferred" copy of a value that updates *after* the urgent render, so an expensive subtree can render with the stale value first and catch up when idle. Also in §18.

> **Quick reference — which hook?**

| Need | Hook |
|---|---|
| Local state that triggers re-render | `useState` |
| Complex / multi-field state transitions | `useReducer` |
| Sync with an external system (network, DOM, subscription) | `useEffect` |
| Read shared data from a Provider | `useContext` (or `use(Ctx)`, §14) |
| Reference a DOM node / store a non-rendering value | `useRef` |
| Cache an expensive computed value | `useMemo` |
| Cache a function reference for a memo'd child | `useCallback` |
| Measure DOM before paint (avoid flicker) | `useLayoutEffect` |
| Stable accessible id (SSR-safe) | `useId` |
| Keep UI responsive during a slow update | `useTransition` / `useDeferredValue` |
| Read a promise or context in render | `use()` (§14) |
| Form/mutation lifecycle (pending, result, error) | `useActionState` (§16) |
| Optimistic UI | `useOptimistic` (§16) |

---

## 10. Custom Hooks

### 10.1 What they are and why they matter **[I]**

A **custom hook** is a function whose name starts with `use` and that *calls other hooks*. That is the entire definition. Custom hooks are React's mechanism for **reusing stateful logic** — not UI (that is components), but *behaviour*: a fetch-with-loading pattern, a debounced value, a localStorage-backed state, a media-query listener.

The logic: before hooks, sharing stateful logic required awkward patterns (HOCs, render props) that wrapped your tree in extra layers. Custom hooks let you extract the logic into a plain function and call it from any component. Crucially, **each component that calls a custom hook gets its own independent state** — the hook is a recipe for behaviour, not a shared store. Two components calling `useToggle()` have separate toggles.

Best practices: name it `useSomething` (the prefix is what lets the linter apply the Rules of Hooks to it); have it return whatever the caller needs (a value, a tuple, an object); and keep each hook focused on one concern. Custom hooks compose — a `useFetch` can use `useState` and `useEffect`, and a bigger hook can use `useFetch`.

```tsx
import { useState, useEffect, useCallback } from "react";

// ── useToggle: the simplest useful custom hook ──
function useToggle(initial = false): [boolean, () => void] {
  const [on, setOn] = useState(initial);
  const toggle = useCallback(() => setOn((v) => !v), []);
  return [on, toggle];                       // caller destructures like useState
}

// ── useLocalStorage: state that persists to localStorage ──
function useLocalStorage<T>(key: string, initial: T) {
  const [value, setValue] = useState<T>(() => {
    const raw = localStorage.getItem(key);    // lazy init — read storage once
    return raw ? (JSON.parse(raw) as T) : initial;
  });
  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value)); // sync to storage when value changes
  }, [key, value]);
  return [value, setValue] as const;
}

// ── useDebounce: a value that updates only after the user stops changing it ──
function useDebounce<T>(value: T, delayMs = 300): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delayMs);
    return () => clearTimeout(id);            // cancel the pending update if value changes again
  }, [value, delayMs]);
  return debounced;
}

// ── useFetch: encapsulate the loading/error/data + cancellation pattern from §8 ──
function useFetch<T>(url: string) {
  const [state, setState] = useState<{ data: T | null; loading: boolean; error: Error | null }>({
    data: null, loading: true, error: null,
  });
  useEffect(() => {
    let cancelled = false;
    setState((s) => ({ ...s, loading: true, error: null }));
    fetch(url)
      .then((r) => { if (!r.ok) throw new Error(`HTTP ${r.status}`); return r.json(); })
      .then((data) => { if (!cancelled) setState({ data, loading: false, error: null }); })
      .catch((error) => { if (!cancelled) setState({ data: null, loading: false, error }); });
    return () => { cancelled = true; };
  }, [url]);
  return state;
}

// Usage — clean component, logic reused:
function SearchBox() {
  const [query, setQuery] = useState("");
  const debouncedQuery = useDebounce(query, 400);                 // only fires fetch after typing stops
  const { data, loading } = useFetch<string[]>(`/api/search?q=${debouncedQuery}`);
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      {loading ? <p>Searching…</p> : <ul>{data?.map((r) => <li key={r}>{r}</li>)}</ul>}
    </>
  );
}
```

> **Cross-reference:** for real apps, `useFetch` above is a teaching tool — **TanStack Query** (`useQuery`) is the production-grade version with caching, deduplication, background refetch, and retries. And for forms, **React Hook Form**'s `useForm` is a battle-tested custom hook you would use instead of hand-rolling form state.

---

## 11. Context for Shared State

### 11.1 The problem it solves: prop drilling **[I]**

When data is needed by many components at different depths (the current theme, the logged-in user, the language), passing it down as props through every intermediate component is tedious and brittle — this is **prop drilling**. Intermediate components have to accept and forward props they do not even use. **Context** lets a component provide a value that any descendant can read directly, no matter how deep, skipping the intermediaries.

### 11.2 The three pieces, and how to use them **[I]**

1. **`createContext(defaultValue)`** — creates the Context object. The default is used only when a consumer has *no* Provider above it (useful for tests and safety).
2. **A Provider** — wraps a subtree and supplies the value. In React 19 you render the context object itself: `<MyContext value={...}>` (the old `<MyContext.Provider value={...}>` still works; see §17).
3. **A consumer** — reads the value with `useContext(MyContext)` (or `use(MyContext)`, §14).

A strong pattern is to wrap the Provider and the consumer hook in one module, exposing a `Provider` component and a `useX()` hook (with a guard that throws if used outside the Provider). Consumers never touch `createContext` directly.

```tsx
import { createContext, useContext, useState, type ReactNode } from "react";

interface AuthValue {
  user: { id: number; name: string } | null;
  login: (name: string) => void;
  logout: () => void;
}
// undefined default lets us detect "used outside provider" and throw a clear error.
const AuthContext = createContext<AuthValue | undefined>(undefined);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<AuthValue["user"]>(null);
  const login = (name: string) => setUser({ id: Date.now(), name });
  const logout = () => setUser(null);
  return <AuthContext value={{ user, login, logout }}>{children}</AuthContext>; // 🆕 React 19
}

// Custom consumer hook with a safety guard.
export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth must be used inside <AuthProvider>");
  return ctx;
}

function UserMenu() {
  const { user, logout } = useAuth();
  return user
    ? <button onClick={logout}>Logout {user.name}</button>
    : <a href="/login">Login</a>;
}
```

### 11.3 Gotchas & when *not* to use Context **[I/A]**

> **Gotcha — every consumer re-renders when the value changes.** When the Provider's `value` changes, *all* components reading that context re-render, even ones that only use part of it. If the value is an object created inline (`value={{ user, login }}`), it is a *new object every render*, so consumers re-render on every Provider render. Mitigate by memoising the value (`useMemo`) and/or splitting one big context into several focused ones (a rarely-changing config context vs a frequently-changing data context).

> **When not to use Context:** Context is a *transport* mechanism, not a state manager. It is great for low-frequency, widely-read values (theme, auth, locale). For high-frequency updates or large/complex client state where re-render cost matters, reach for **Zustand** or Jotai — they let components subscribe to *slices* of state and skip re-renders when unrelated parts change. And for *server* state (API data), use **TanStack Query** rather than stuffing fetched data into Context.

---

## 12. Refs & the DOM

### 12.1 What a ref is and the two uses **[I]**

A **ref** is an object `{ current: ... }` from `useRef` that React keeps stable across renders and that does **not** trigger a re-render when you change `.current`. Contrast with state: change state → React re-renders; change a ref → nothing visible happens, you just stored a value. That difference defines when to use each: **use state for anything that should appear in the UI; use a ref for things React should not react to.**

Two uses:
1. **Reach a DOM element.** Pass a ref to an element's `ref` attribute and React sets `ref.current` to that DOM node after commit. Now you can imperatively focus it, measure it, scroll it, or play a video — things the declarative model does not express.
2. **Store a mutable value across renders** without causing re-renders — a timer id, the previous value of a prop, whether something has already run.

```tsx
import { useRef, useEffect } from "react";

// DOM element ref:
function SearchField() {
  const ref = useRef<HTMLInputElement>(null);     // typed to the element kind
  return (
    <>
      <input ref={ref} placeholder="search" />
      {/* Imperatively focus the DOM node — an escape hatch from declarative UI. */}
      <button onClick={() => ref.current?.focus()}>Focus</button>
    </>
  );
}

// "Previous value" pattern — a classic ref use:
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);
  useEffect(() => { ref.current = value; });       // runs AFTER render, so it lags one render
  return ref.current;                               // returns the value from the PREVIOUS render
}
```

> **Gotcha — don't read or write `ref.current` *during* render.** Refs are not reactive; reading one during render can give a stale value, and writing one during render breaks purity. Touch refs in event handlers and effects only.

> **`useRef` vs `createRef`:** in function components always use `useRef` — it returns the *same* object every render. `createRef` creates a *new* ref each call, so in a function component it would be reset every render; it exists for class components.

### 12.2 Forwarding refs — and the React 19 simplification **[I/A]**

Sometimes a parent needs a ref to a DOM element *inside* a child component (to focus a custom `Input`). In React 18 this required wrapping the child in `forwardRef`. **React 19 makes `ref` an ordinary prop** of function components — no wrapper needed (full detail in §17). Plus, callback refs can now return a **cleanup function**, mirroring effects.

```tsx
// 🆕 React 19 — ref is just a prop; no forwardRef wrapper.
function FancyInput({ ref, ...props }: React.InputHTMLAttributes<HTMLInputElement> & {
  ref?: React.Ref<HTMLInputElement>;
}) {
  return <input ref={ref} {...props} />;
}
function Parent() {
  const ref = useRef<HTMLInputElement>(null);
  return <FancyInput ref={ref} placeholder="focus me from the parent" />;
}
```

### 12.3 Portals — rendering outside the parent DOM tree **[I/A]**

`createPortal(children, domNode)` renders `children` into a *different* part of the real DOM while keeping them in your React tree logically (state, context, and events still flow as if they were nested normally). Use it for **modals, tooltips, toasts, and dropdowns** that must visually escape a parent's `overflow: hidden` or `z-index` stacking context.

```tsx
import { createPortal } from "react-dom";
import { useEffect } from "react";

function Modal({ open, onClose, children }: {
  open: boolean; onClose: () => void; children: React.ReactNode;
}) {
  useEffect(() => {
    if (!open) return;
    const onKey = (e: KeyboardEvent) => e.key === "Escape" && onClose();
    document.addEventListener("keydown", onKey);
    return () => document.removeEventListener("keydown", onKey);
  }, [open, onClose]);

  if (!open) return null;
  // Render the overlay into <body>, escaping any clipped/overflow parent — but events
  // (onClose) still work because logically this is still inside <Modal>.
  return createPortal(
    <div className="overlay" role="dialog" aria-modal="true"
         onClick={(e) => e.target === e.currentTarget && onClose()}>
      <div className="modal">{children}</div>
    </div>,
    document.body            // ← the portal target: any real DOM node
  );
}
```

---

## 13. Performance: memo, useMemo, useCallback & the React Compiler

### 13.1 First principle: measure, don't guess **[I/A]**

The golden rule of React performance: **profile before you optimise.** Most apps are fast enough without any manual memoisation, and premature `useMemo`/`useCallback` everywhere adds noise and a (small) cost while fixing nothing. Use the **React DevTools Profiler** to find components that *actually* render too often or too slowly, then optimise *those*. Everything below is for the cases the profiler flags.

Recall the render model (§4.2): a component re-renders when its state changes *or when its parent re-renders*. Most "performance problems" are a parent re-rendering and dragging an expensive subtree along for the ride. The three tools below address exactly that.

### 13.2 `React.memo` — skip re-rendering a component when its props are unchanged **[I/A]**

`memo(Component)` returns a memoised version that **skips re-rendering when its props are shallowly equal to the previous render's props.** It is the lever for "this parent re-renders a lot, but this child's props rarely change — don't re-render the child needlessly."

The catch — and why `useCallback`/`useMemo` exist — is that the shallow comparison is by **reference**. If the parent passes a *new* object, array, or function as a prop every render (which inline literals and arrow functions are), `memo` sees "props changed" and re-renders anyway. So `memo` on the child must be paired with *stable references* on the props the parent passes.

```tsx
import { memo, useCallback, useState } from "react";

const ExpensiveList = memo(function ExpensiveList({
  items, onSelect,
}: { items: string[]; onSelect: (item: string) => void }) {
  console.log("ExpensiveList rendered");      // with memo + stable props, this stays quiet
  return <ul>{items.map((i) => <li key={i} onClick={() => onSelect(i)}>{i}</li>)}</ul>;
});

const ITEMS = ["a", "b", "c"];                // defined OUTSIDE the component → stable reference

function Parent() {
  const [count, setCount] = useState(0);
  const onSelect = useCallback((item: string) => console.log(item), []); // stable function
  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>Re-render parent ({count})</button>
      {/* Because ITEMS and onSelect are stable, memo lets ExpensiveList skip re-rendering
          when only `count` changes. */}
      <ExpensiveList items={ITEMS} onSelect={onSelect} />
    </>
  );
}
```

### 13.3 `useMemo` & `useCallback` recap in the performance context **[I/A]**

- **`useMemo`** caches a computed *value* and a *stable reference* for objects/arrays. Use it for (a) genuinely expensive computations, and (b) keeping an object/array prop reference stable for a `memo`'d child or an effect dependency.
- **`useCallback`** caches a *function reference* so a `memo`'d child does not re-render from a fresh function each parent render.

Both take a dependency array with the same semantics as `useEffect`. Both are *optimisations*, never correctness tools — your app must behave identically if React threw the cache away.

```tsx
// Stable object reference so a memo'd child doesn't see "new props" every render:
const config = useMemo(() => ({ pageSize: 20, sort: "name" }), []); // [] → never recomputed
<DataGrid config={config} />;                                        // DataGrid can be memo'd
```

### 13.4 The React Compiler — automatic memoisation **🆕 React 19** **[A]**

Manually scattering `memo`/`useMemo`/`useCallback` is error-prone and clutters code. The **React Compiler** is an official build-time tool that *analyses* your components and *automatically inserts* the right memoisation — so you write plain, clean code and still get the performance of hand-tuned memoisation. It is the long-term answer to "do I need `useMemo` here?": *let the compiler decide.*

> **⚡ Version note:** the React Compiler reached stable, opt-in status in 2025 and is widely adopted by 2026, though not yet the default everywhere. **Next.js 16+** enables it via config; Vite users add a Babel/SWC plugin.

```tsx
// What you write — no memo, no useMemo, no useCallback:
function SearchResults({ query, items }: { query: string; items: string[] }) {
  const filtered = items.filter((i) => i.includes(query));
  return <ul>{filtered.map((i) => <li key={i}>{i}</li>)}</ul>;
}
// What the compiler effectively produces — memoised filtering, automatically:
//   const filtered = useMemo(() => items.filter(i => i.includes(query)), [items, query]);
```

```bash
npm install -D babel-plugin-react-compiler        # Vite/Babel path
```
```js
// babel.config.js
module.exports = { plugins: [["babel-plugin-react-compiler", { target: "19" }]] };
```
```ts
// next.config.ts (Next.js 16+)
import type { NextConfig } from "next";
const nextConfig: NextConfig = { experimental: { reactCompiler: true } };
export default nextConfig;
```

**The compiler only works if your code follows the Rules of React**, because it *assumes* your components are pure to safely cache:
1. Components and hooks are **pure** — same inputs → same output, no side effects during render.
2. **Rules of Hooks** are obeyed (top-level, unconditional).
3. Props and state are treated as **immutable** — never mutated (this is exactly why §5 hammered immutability).

```tsx
// ❌ Mutating during render breaks purity — the compiler bails out (or you get bugs):
function Bad({ items }: { items: number[] }) {
  items.sort();                                  // mutates a prop! illegal
  return <ul>{items.map((i) => <li key={i}>{i}</li>)}</ul>;
}
// ✅ Copy first, then sort the copy:
function Good({ items }: { items: number[] }) {
  const sorted = [...items].sort((a, b) => a - b);
  return <ul>{sorted.map((i) => <li key={i}>{i}</li>)}</ul>;
}

// Escape hatch — opt a single component out of compilation if it can't be compiled safely:
function Legacy() { "use no memo"; /* ... */ return null; }
```

| Without the compiler | With the compiler |
|---|---|
| Manually add `useMemo` for expensive calcs | Inserted automatically |
| Manually add `useCallback` for stable callbacks | Inserted automatically |
| Wrap children in `React.memo` | Auto-memoised |
| Easy to introduce perf bugs by forgetting memo | Handled for most cases |

---

## 14. The `use()` API

### 14.1 What it is and why it is special **🆕 React 19** **[I/A]**

`use()` is a new API that **reads the value of a Promise or a Context during render.** It is unique among React's APIs in two ways:

1. **It can be called conditionally** — inside `if`, loops, and after early returns — unlike every hook (recall the call-order rule, §9). React designed `use()` to be exempt because it does not rely on a fixed call slot the way stateful hooks do.
2. **When given a Promise, it *suspends* the component** until the promise resolves, integrating directly with `<Suspense>` (for the pending UI) and error boundaries (for rejection). This lets you write data-reading code that *looks synchronous* — `const data = use(promise)` — while React handles the waiting.

### 14.2 Reading a Promise (with Suspense) **[I/A]**

When you `use(somePromise)`, the component "pauses" while pending; the nearest `<Suspense>` boundary above shows its `fallback`; when the promise resolves, React re-renders with the value; if it rejects, the nearest error boundary catches it.

The **critical rule**: the promise must be **stable across renders** — created *outside* the component or passed in as a prop. If you create a new promise *inside* the render body, every render makes a fresh promise that is never resolved-from-cache, causing an infinite suspend/re-render loop.

```tsx
import { use, Suspense } from "react";

async function fetchUser(id: number): Promise<{ name: string; email: string }> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error("Failed to fetch user"); // rejection → error boundary
  return res.json();
}

// The component reads the promise; it must sit under a <Suspense> boundary.
function UserCard({ promise }: { promise: Promise<{ name: string; email: string }> }) {
  const user = use(promise);              // suspends until resolved; reads value when ready
  return <div className="card"><h2>{user.name}</h2><p>{user.email}</p></div>;
}

// The PARENT creates the (stable) promise and provides the Suspense fallback.
function UserPage({ userId }: { userId: number }) {
  const userPromise = fetchUser(userId); // started here; do NOT await — pass it down
  return (
    <Suspense fallback={<p>Loading user…</p>}>
      <UserCard promise={userPromise} />
    </Suspense>
  );
}
```

> **Gotcha — the stable-promise rule, made concrete:**
> ```tsx
> // ❌ new promise EVERY render → infinite Suspense loop
> function Bad({ id }: { id: number }) { const d = use(fetch(`/api/${id}`).then(r => r.json())); }
> // ✅ cache the promise so the same one is reused for the same input
> const cache = new Map<number, Promise<unknown>>();
> function stable(id: number) {
>   if (!cache.has(id)) cache.set(id, fetch(`/api/${id}`).then(r => r.json()));
>   return cache.get(id)!;
> }
> function Good({ id }: { id: number }) { const d = use(stable(id)); }
> ```
> In real apps you rarely manage this cache yourself — a framework (Next.js) or **TanStack Query** provides the stable, cached promises.

### 14.3 Reading Context conditionally **[I/A]**

`use(Context)` reads context like `useContext`, but because it can be called conditionally, you can read context inside a branch — impossible with `useContext`. For unconditional reads, `useContext` remains the conventional choice; reach for `use(Context)` only when you genuinely need the conditional read.

```tsx
import { use, createContext } from "react";
const ThemeContext = createContext<"light" | "dark">("light");

function MaybeThemed({ show }: { show: boolean }) {
  if (show) {
    const theme = use(ThemeContext);   // ✅ allowed inside a condition (would be illegal with useContext)
    return <p>Theme: {theme}</p>;
  }
  return <p>Hidden</p>;
}
```

---

## 15. Actions & Form Actions

### 15.1 The problem Actions solve **🆕 React 19** **[I/A]**

Before React 19, handling a form submission or any data mutation meant a tedious manual dance: track an `isPending` flag, `try/catch` for errors, store the result, reset the form, prevent the default reload, maybe show an optimistic update and roll it back on failure. You wrote that boilerplate for *every* mutation. **Actions** make async mutations a first-class concept: you pass an `async` function to a form (or a transition), and React automatically manages pending state, error propagation, and form reset for you.

### 15.2 `<form action={asyncFn}>` — the new native pattern **[I/A]**

In React 19 you can pass an **async function** to a form's `action` prop. When the form submits, React calls your function with the form's **`FormData`**, marks the surrounding transition as pending (so `useFormStatus`/`useActionState` can show progress, §16), and resets the form on success. No `onSubmit`, no `e.preventDefault()`.

```tsx
// A purely client-side async action (no server framework needed).
async function submitContact(formData: FormData) {
  const name = formData.get("name") as string;
  const message = formData.get("message") as string;
  const res = await fetch("/api/contact", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ name, message }),
  });
  if (!res.ok) throw new Error("Failed to send");  // throwing surfaces to error handling
}

function ContactForm() {
  // Pass the async function straight to `action`. React handles pending state + reset.
  return (
    <form action={submitContact}>
      <input name="name" placeholder="Your name" required />
      <textarea name="message" placeholder="Your message" required />
      <button type="submit">Send</button>
    </form>
  );
}
```

> The same `<form action={fn}>` is also how **Server Actions** are wired in Next.js — there the function carries a `"use server"` directive and runs on the server (§20). The component code is identical; only where the function executes differs.

### 15.3 Async transitions **[A]**

`startTransition` now accepts **async functions** in React 19, so you can run a mutation outside a form and still get the pending/transition behaviour. Pair it with `useTransition` for an `isPending` flag.

```tsx
import { useTransition, useState } from "react";
function SaveButton({ id }: { id: number }) {
  const [isPending, startTransition] = useTransition();
  const [saved, setSaved] = useState(false);
  function handleSave() {
    startTransition(async () => {               // 🆕 async allowed in React 19
      await fetch(`/api/items/${id}`, { method: "PUT" });
      setSaved(true);                            // state set inside is part of the transition
    });
  }
  return <button onClick={handleSave} disabled={isPending}>{isPending ? "Saving…" : saved ? "Saved" : "Save"}</button>;
}
```

---

## 16. `useActionState`, `useFormStatus`, `useOptimistic`

These three hooks are the everyday companions to Actions (§15). Learn them together.

### 16.1 `useActionState` — the full lifecycle of an action **🆕 React 19** **[I/A]**

`useActionState(actionFn, initialState)` manages an action's **state, pending flag, and result** in one place. It wraps your action so that:
- the **previous state** is passed to your action as its *first* argument (the `FormData` is the second),
- your action's **return value becomes the new state**,
- you get back `[state, dispatch, isPending]` — wire `dispatch` to `<form action={dispatch}>`.

Think of it as a **reducer for async form submissions**: instead of `(state, action) => state`, it is `async (state, formData) => state`. This is the idiomatic way to show validation errors and results inline.

```tsx
import { useActionState } from "react";

type RegisterState =
  | { status: "idle" }
  | { status: "success"; username: string }
  | { status: "error"; message: string };

// First arg = previous state; second = FormData. Return value = next state.
async function registerUser(prev: RegisterState, formData: FormData): Promise<RegisterState> {
  const username = formData.get("username") as string;
  if (username.length < 3) return { status: "error", message: "Username too short" };
  try {
    const res = await fetch("/api/register", {
      method: "POST", headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ username }),
    });
    if (!res.ok) return { status: "error", message: (await res.json()).message };
    return { status: "success", username };     // ← becomes the new state
  } catch {
    return { status: "error", message: "Network error" };
  }
}

function RegisterForm() {
  const [state, formAction, isPending] = useActionState(registerUser, { status: "idle" });

  if (state.status === "success") return <p>Welcome, {state.username}!</p>;
  return (
    <form action={formAction}>
      {state.status === "error" && <p role="alert" className="error">{state.message}</p>}
      <input name="username" required />
      <button type="submit" disabled={isPending}>
        {isPending ? "Creating…" : "Register"}
      </button>
    </form>
  );
}
```

> **Gotcha — return validation errors, don't throw them.** For *user* errors (bad input), *return* an error state so the form shows it and stays interactive. *Throw* only for unexpected failures you want an error boundary (§19) to catch.

> **Migration note:** this hook replaces React 18's experimental `useFormState`. Import `useActionState` from `react` (the codemod `replace-use-form-state` automates the rename, §21).

### 16.2 `useFormStatus` — read the parent form's submission state **🆕 React 19** **[I/A]**

`useFormStatus()` reads the pending status of the **nearest ancestor `<form>`**. Its purpose is to let a *reusable child component* (a submit button, a spinner) know whether the form it lives in is currently submitting — without the form having to pass that down as a prop. It returns `{ pending, data, method, action }`.

The one rule that trips everyone: it reads the *ancestor* form, so it **must be a child component rendered inside the form** — it cannot be called in the same component that renders the `<form>`.

```tsx
import { useFormStatus } from "react-dom";        // NOTE: from "react-dom", not "react"

// ✅ A reusable submit button that knows its form's status — used INSIDE any form.
function SubmitButton({ label }: { label: string }) {
  const { pending } = useFormStatus();             // reads the nearest ancestor <form>
  return <button type="submit" disabled={pending} aria-busy={pending}>{pending ? "Submitting…" : label}</button>;
}

function MyForm({ action }: { action: (fd: FormData) => Promise<void> }) {
  return (
    <form action={action}>
      <input name="email" type="email" required />
      <SubmitButton label="Subscribe" />          {/* child → can read this form's status */}
    </form>
  );
}

// ❌ WRONG: calling useFormStatus in the SAME component as the form — `pending` is always false,
// because there is no ANCESTOR form from this component's perspective.
```

### 16.3 `useOptimistic` — instant UI with automatic rollback **🆕 React 19** **[I/A]**

`useOptimistic(realState, updateFn)` lets you show an **assumed-successful result immediately**, before the async operation finishes, then automatically **reverts** to the real state if the operation fails or when the real state arrives. This is what makes UIs feel instant (a "like" turns red the moment you click, even though the server round-trip is still in flight).

How it works: you give it your real state and a reducer `(currentState, optimisticInput) => newState`. You call the returned `addOptimistic(input)` to *layer on* an optimistic change synchronously; React shows that until the surrounding action settles, then snaps back to the real state.

```tsx
import { useOptimistic, useState } from "react";

interface Message { id: string; text: string; sending?: boolean; }

function ChatRoom({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState<Message[]>([]);

  // optimisticMessages = real messages + any pending optimistic ones.
  const [optimisticMessages, addOptimistic] = useOptimistic(
    messages,
    // reducer: how to fold the optimistic input into the current state
    (current, text: string) => [...current, { id: `temp-${Date.now()}`, text, sending: true }]
  );

  async function send(formData: FormData) {
    const text = formData.get("text") as string;
    addOptimistic(text);                        // 1. show it instantly (with sending: true)
    const res = await fetch(`/api/rooms/${roomId}/messages`, {
      method: "POST", body: JSON.stringify({ text }), headers: { "Content-Type": "application/json" },
    });
    const saved = (await res.json()) as Message;
    setMessages((prev) => [...prev, saved]);    // 2. real state replaces the optimistic one
  }

  return (
    <>
      <ul>
        {optimisticMessages.map((m) => (
          <li key={m.id} style={{ opacity: m.sending ? 0.6 : 1 }}>
            {m.text} {m.sending && <em>(sending…)</em>}
          </li>
        ))}
      </ul>
      <form action={send}>
        <input name="text" required />
        <button type="submit">Send</button>
      </form>
    </>
  );
}
```

> **Cross-reference:** for complex forms (multi-field validation, dynamic fields, schema validation with Zod) you will likely combine React 19 Actions with **React Hook Form**, which manages field-level state and validation efficiently; for server-state mutations with optimistic updates and cache invalidation, **TanStack Query**'s `useMutation` is the heavier-duty counterpart to `useOptimistic`.

---

## 17. `ref` as a Prop, Context as a Provider, Document Metadata

React 19 includes several ergonomic wins that remove old boilerplate. Group them in your mind as "things that got simpler."

### 17.1 `ref` as a plain prop — goodbye `forwardRef` **🆕 React 19** **[I/A]**

In React 18, a function component could not receive `ref` like a normal prop — React intercepted it, and you had to wrap the component in `forwardRef` to opt in. **React 19 makes `ref` an ordinary prop**: just declare it in your props type and use it. `forwardRef` is **deprecated** and will be removed eventually. Migrating is mechanical: unwrap the `forwardRef`, move `ref` into the props.

```tsx
// ❌ React 18 — forwardRef wrapper required (now deprecated)
import { forwardRef } from "react";
const OldInput = forwardRef<HTMLInputElement, { placeholder?: string }>(
  (props, ref) => <input ref={ref} {...props} />
);

// ✅ React 19 — ref is just a prop
function NewInput({ ref, ...props }: React.InputHTMLAttributes<HTMLInputElement> & {
  ref?: React.Ref<HTMLInputElement>;
}) {
  return <input ref={ref} {...props} />;
}
// Usage is identical for both: <NewInput ref={myRef} placeholder="…" />
```

**Bonus — callback refs can return cleanup.** A callback ref (`ref={(node) => {...}}`) may now return a cleanup function that React runs on unmount, just like an effect — no more null-checking inside the same callback to detect teardown.

```tsx
function VideoPlayer({ src }: { src: string }) {
  return (
    <video src={src} ref={(el) => {
      if (!el) return;
      el.play().catch(() => {});
      return () => { el.pause(); el.src = ""; };   // 🆕 cleanup function on the callback ref
    }} />
  );
}
```

### 17.2 Context as a Provider — goodbye `.Provider` **🆕 React 19** **[I]**

You can now render the context object **directly** as a provider: `<MyContext value={...}>` instead of `<MyContext.Provider value={...}>`. Pure ergonomics — less to type, less to read. The `.Provider` form still works for back-compat.

```tsx
const ThemeContext = createContext("light");
// 🆕 React 19:
<ThemeContext value="dark">{children}</ThemeContext>
// Old (still valid):
// <ThemeContext.Provider value="dark">{children}</ThemeContext.Provider>
```

### 17.3 Document metadata — `<title>`, `<meta>`, `<link>` anywhere **🆕 React 19** **[I/A]**

React 19 lets you render `<title>`, `<meta>`, and `<link>` tags **anywhere in your component tree**, and React automatically **hoists** them into the document `<head>` with sensible deduplication. Before, you needed `react-helmet` (SPA) or `next/head` (Next.js) for this. Now a page or component can declare its own SEO metadata inline, right where the content lives.

Dedup rules: `<title>` — last one wins; `<meta name>` — deduped by `name`; `<link rel="stylesheet">` — deduped by `href`.

```tsx
function BlogPost({ post }: { post: { title: string; description: string; slug: string } }) {
  return (
    <article>
      {/* 🆕 hoisted into <head> automatically — no helmet/next-head needed */}
      <title>{post.title} | My Blog</title>
      <meta name="description" content={post.description} />
      <meta property="og:title" content={post.title} />
      <link rel="canonical" href={`https://example.com/posts/${post.slug}`} />
      <h1>{post.title}</h1>
      <p>{post.description}</p>
    </article>
  );
}
```

> **⚡ Version note:** in **Next.js**, prefer Next's own Metadata API for route-level SEO (it handles streaming and ordering); React's native metadata is ideal for component-local tags and non-framework apps.

### 17.4 Resource preloading APIs **🆕 React 19** **[A]**

React 19 ships imperative functions (from `react-dom`) to hint the browser to fetch resources early, shaving latency off critical paths. Call them as early as possible (a layout component, a route handler).

```tsx
import { prefetchDNS, preconnect, preload, preinit } from "react-dom";
function AppShell() {
  prefetchDNS("https://fonts.googleapis.com");                 // resolve DNS early
  preconnect("https://api.example.com");                       // DNS + TCP + TLS handshake
  preload("/hero.webp", { as: "image", fetchPriority: "high" }); // fetch & cache (don't execute)
  preinit("https://cdn.example.com/critical.css", { as: "style" }); // fetch AND apply now
  return <main>…</main>;
}
```

| API | What it does | Use for |
|---|---|---|
| `prefetchDNS(href)` | Resolve DNS in the background | Any third-party origin you'll hit |
| `preconnect(href)` | DNS + TCP + TLS handshake | API servers, CDNs, font hosts |
| `preload(href, { as })` | Download & cache (no execute) | Fonts, hero images, soon-used scripts |
| `preinit(href, { as })` | Download AND execute/apply | Critical CSS, analytics scripts |

---

## 18. Suspense, Concurrent Features & Data Fetching

### 18.1 What "concurrent" means **[A]**

React 18 introduced **concurrent rendering**: React can prepare a render *in the background*, pause it, resume it, or throw it away — without blocking the main thread. The point is *responsiveness*: a slow re-render (filtering 10,000 rows) no longer freezes typing or clicking, because React can keep urgent updates (your keystrokes) ahead of non-urgent ones (the expensive list). React 19 builds on this with Actions and the `use()` API. The user-facing tools are `<Suspense>`, `useTransition`, and `useDeferredValue`.

### 18.2 `<Suspense>` — declarative loading states **[A]**

`<Suspense fallback={...}>` catches any descendant that **suspends** (is "not ready" — code still loading, or a `use(promise)` still pending) and shows the `fallback` until it is ready. This turns loading states from scattered `if (loading)` checks into a *declarative boundary*: "while anything in here is loading, show this." You can nest boundaries for granular loading (a fast header resolves independently of a slow feed), and stream content in as it becomes ready.

```tsx
import { Suspense, lazy } from "react";

// Use 1: code-splitting with lazy — the chunk loads on first render, fallback shows meanwhile.
const HeavyChart = lazy(() => import("./HeavyChart"));

function Dashboard({ user, posts }: { user: Promise<User>; posts: Promise<Post[]> }) {
  return (
    <main>
      <Suspense fallback={<div className="skeleton" />}><HeavyChart /></Suspense>

      {/* Use 2: nested boundaries — each section shows its own fallback and resolves on its own. */}
      <Suspense fallback={<HeaderSkeleton />}><UserHeader promise={user} /></Suspense>
      <Suspense fallback={<FeedSkeleton />}><PostFeed promise={posts} /></Suspense>
    </main>
  );
}
```

### 18.3 `useTransition` — keep the UI responsive **[A]**

`useTransition()` returns `[isPending, startTransition]`. Wrap a *non-urgent* state update in `startTransition(() => ...)` to tell React "this can be interrupted by anything more urgent." The classic case: an input that filters a huge list — keep the *input* update urgent (instant typing) but mark the *list* update as a transition (so typing never stutters). `isPending` lets you dim the stale results.

```tsx
import { useTransition, useState } from "react";
function SearchPage() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState<string[]>([]);
  const [isPending, startTransition] = useTransition();

  function onChange(e: React.ChangeEvent<HTMLInputElement>) {
    const value = e.target.value;
    setQuery(value);                            // URGENT: the input updates immediately
    startTransition(() => {                      // NON-URGENT: the expensive list can be interrupted
      setResults(heavyFilter(value));
    });
  }
  return (
    <>
      <input value={query} onChange={onChange} />
      <div style={{ opacity: isPending ? 0.5 : 1 }}>{results.map((r) => <p key={r}>{r}</p>)}</div>
    </>
  );
}
function heavyFilter(q: string) {
  return Array.from({ length: 10_000 }, (_, i) => `Item ${i}`)
    .filter((x) => x.toLowerCase().includes(q.toLowerCase()));
}
```

### 18.4 `useDeferredValue` — a lagging copy **[A]**

`useDeferredValue(value)` returns a copy of `value` that **lags behind** during urgent updates. React renders with the *old* deferred value first (keeping the UI responsive), then re-renders with the fresh value when idle. It is like `useTransition` but for a *value* you receive rather than a *setter* you call — handy when the slow part consumes a value (e.g. a prop) you cannot wrap in `startTransition`.

```tsx
import { useDeferredValue, useState, memo } from "react";
const HeavyList = memo(function HeavyList({ filter }: { filter: string }) {
  const items = Array.from({ length: 5000 }, (_, i) => `Item ${i}`).filter((x) => x.includes(filter));
  return <ul>{items.slice(0, 50).map((x) => <li key={x}>{x}</li>)}</ul>;
});
function FilteredList() {
  const [filter, setFilter] = useState("");
  const deferred = useDeferredValue(filter);     // lags behind `filter` during fast typing
  return (
    <>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      <div style={{ opacity: deferred !== filter ? 0.6 : 1 }}>
        <HeavyList filter={deferred} />            {/* expensive render uses the lagging value */}
      </div>
    </>
  );
}
```

### 18.5 Data-fetching strategies, summarised **[A]**

| Approach | When | Notes |
|---|---|---|
| `fetch` in `useEffect` | Learning, tiny apps | Manual loading/error/cancel; race-prone (§8) |
| `use(promise)` + `<Suspense>` | React 19, promise created by parent/RSC | Declarative loading; needs stable promise (§14) |
| **TanStack Query** | Most client-side data fetching | Caching, dedup, refetch, retries — the production default |
| **Server Components** | Next.js / RSC frameworks | Fetch on the server, zero client JS for the data (§20) |

> **Best practice:** in 2026, default to **Server Components** (in a framework) or **TanStack Query** (in an SPA) for data fetching. Reserve `fetch`-in-`useEffect` for throwaway demos.

---

## 19. Error Handling & Error Boundaries

### 19.1 Error boundaries — catching render-time errors **[I/A]**

If a component throws *during rendering*, React by default unmounts the whole tree (a blank screen). An **error boundary** is a component that **catches errors thrown by its descendants during render**, logs them, and shows a fallback UI instead — containing the blast radius to one section. Error boundaries are the React equivalent of a `try/catch` around a part of your UI.

The quirk: error boundaries must currently be **class components**, because they rely on two lifecycle methods with no hook equivalent — `getDerivedStateFromError` (to switch to the fallback) and `componentDidCatch` (to log). You write one generic boundary and reuse it everywhere; you do not write many.

```tsx
import { Component, type ErrorInfo, type ReactNode } from "react";

interface State { hasError: boolean; error: Error | null; }

class ErrorBoundary extends Component<{ children: ReactNode; fallback?: ReactNode }, State> {
  state: State = { hasError: false, error: null };

  // Called when a descendant throws during render → return new state to show the fallback.
  static getDerivedStateFromError(error: Error): State { return { hasError: true, error }; }

  // Called with the error + which components were involved → log it to your service.
  componentDidCatch(error: Error, info: ErrorInfo) {
    console.error("Caught:", error, info.componentStack); // e.g. Sentry.captureException(error)
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? (
        <div role="alert">
          <h2>Something went wrong.</h2>
          <p>{this.state.error?.message}</p>
          <button onClick={() => this.setState({ hasError: false, error: null })}>Try again</button>
        </div>
      );
    }
    return this.props.children;
  }
}

function App() {
  return (
    <ErrorBoundary fallback={<p>Failed to load this section.</p>}>
      <RiskyComponent />
    </ErrorBoundary>
  );
}
```

> **Gotcha — what error boundaries do NOT catch:** errors in **event handlers** (use `try/catch` there), errors in **async code** (`setTimeout`, promises — `try/catch` + state), errors during **SSR**, and errors thrown by the boundary itself. Error boundaries are only for the *render/commit* phase of descendants. Note that a `use(promise)` rejection *does* propagate to the nearest error boundary (§14), and `<Suspense>` + an error boundary together give you the "loading / error / data" trio declaratively.

### 19.2 Root-level error hooks **🆕 React 19** **[A]**

`createRoot` now accepts callbacks for *all* error categories, giving you a single place to wire up monitoring (Sentry, etc.):

```tsx
import { createRoot } from "react-dom/client";
createRoot(document.getElementById("root")!, {
  // 🆕 caught by an error boundary (handled)
  onCaughtError(error, info) { report(error, { handled: true, stack: info.componentStack }); },
  // 🆕 NOT caught by any boundary (the app will crash) — fatal
  onUncaughtError(error, info) { report(error, { handled: false, fatal: true }); },
  // existed in 18 — errors React auto-recovered from (e.g. hydration fallbacks)
  onRecoverableError(error) { console.warn("Recoverable:", error); },
}).render(<App />);
function report(error: unknown, ctx: Record<string, unknown>) { /* Sentry.captureException(...) */ }
```

---

## 20. Server Components & Server Actions

> **⚡ Version note:** React 19 ships the **specification** for Server Components (RSC) and Server Actions; the *runtime* is provided by a framework. **Next.js (App Router)** is the dominant implementation in 2026; Remix and others also support it. A plain Vite + React SPA does **not** have Server Components unless you add a framework. Learn the concepts here; apply them in Next.js.

### 20.1 The mental model: two execution environments **[A]**

Traditionally, all React components ran in the browser. RSC splits components into two kinds by *where they run*:

- **Server Components** run **only on the server**, never ship to the browser, can be `async`, and can directly access the database, filesystem, and secrets. They add **zero JavaScript** to the client bundle. They *cannot* use state, effects, refs, or event handlers — because there is no interactivity on the server.
- **Client Components** are marked with the `"use client"` directive, ship to the browser as JS, and are the only place for interactivity — `useState`, `useEffect`, event handlers, refs.

```
┌──────────── SERVER ────────────┐        ┌──────────── BROWSER ───────────┐
│ Server Components               │        │ Client Components ("use client")│
│  • async, await data directly   │ render │  • useState / useEffect / refs  │
│  • DB / FS / secrets access     │ ─────► │  • event handlers, interactivity│
│  • ZERO client JS               │ (RSC   │  • shipped as JS to the browser │
│  • no state / effects / events  │  wire) │                                 │
└─────────────────────────────────┘        └─────────────────────────────────┘
```

The big win: fetch data on the server with no loading spinner, no API round-trip, no client bundle cost — and push interactivity to small "islands" of Client Components.

```tsx
// app/products/page.tsx — a Server Component (no "use client" directive)
async function ProductsPage() {
  // Runs on the server: query the DB directly. No useEffect, no loading state needed.
  const products = await db.query("SELECT * FROM products ORDER BY name");
  return (
    <section>
      <h1>Products</h1>
      {products.map((p: { id: number; name: string; price: number }) => (
        <ProductCard key={p.id} product={p} />   // ProductCard is a Client Component (has a button)
      ))}
    </section>
  );
}
export default ProductsPage;
```

```tsx
// ProductCard.tsx — a Client Component (interactive)
"use client";                                // ← everything in/imported by this file ships to the browser
import { useState } from "react";
function ProductCard({ product }: { product: { id: number; name: string; price: number } }) {
  const [inCart, setInCart] = useState(false);
  return (
    <div className="card">
      <h2>{product.name}</h2><p>${product.price}</p>
      <button onClick={() => setInCart((v) => !v)}>{inCart ? "Remove" : "Add to Cart"}</button>
    </div>
  );
}
export default ProductCard;
```

> **The composition rule:** a Client Component **cannot import** a Server Component (the server code can't run in the browser). But a Server Component can **pass a Client Component (or its output) as `children`/props**. So compose Server → Client downward, and pass server-rendered content *into* client wrappers via props, never by import.

### 20.2 Server Actions — mutations without API routes **[A]**

A **Server Action** is an async function marked `"use server"` that runs on the server but can be *called from the client* (e.g. a form submit). It lets you mutate data — write to the DB — without hand-writing an API endpoint and `fetch` call. You pass it straight to `<form action={...}>` (§15); the framework handles serialising the call to the server.

```tsx
// actions.ts — a file of Server Actions
"use server";
import { revalidatePath } from "next/cache";       // Next.js: tell it to refresh cached data

export async function deleteProduct(formData: FormData) {
  const id = Number(formData.get("id"));
  await db.query("DELETE FROM products WHERE id = ?", [id]);
  revalidatePath("/products");                       // re-render the products page with fresh data
}

// Pairs with useActionState for validation + result (return errors, don't throw user errors):
export async function updatePrice(prev: { error: string | null }, formData: FormData) {
  const price = Number(formData.get("price"));
  if (isNaN(price) || price < 0) return { error: "Invalid price" };
  await db.query("UPDATE products SET price = ? WHERE id = ?", [price, Number(formData.get("id"))]);
  revalidatePath("/products");
  return { error: null };
}
```

> **Cross-reference:** Next.js orchestrates all of this — routing, the RSC runtime, `revalidatePath`/`revalidateTag` caching, and streaming. If you are building a full-stack React app in 2026, learning Next.js's App Router is how you actually *use* Server Components and Server Actions.

---

## 21. Migrating from React 18 to 19

### 21.1 What broke, and what to do **[A]**

React 19 removes a long list of APIs that were deprecated for years. Most apps need only a handful of mechanical changes; the codemods automate most of them.

| What changed | React 18 | React 19 | Action |
|---|---|---|---|
| `ReactDOM.render` | Deprecated | **Removed** | Use `createRoot(...).render(...)` |
| `ReactDOM.hydrate` | Deprecated | **Removed** | Use `hydrateRoot(...)` |
| `unmountComponentAtNode` | Deprecated | **Removed** | Call `root.unmount()` |
| Legacy Context (`contextTypes`) | Deprecated | **Removed** | `createContext` + `useContext` |
| String refs (`ref="x"`) | Deprecated | **Removed** | `useRef` / callback refs |
| `propTypes` | In core | **Removed** | TypeScript types |
| `defaultProps` (function comps) | Deprecated | **Deprecated** | ES6 default params |
| `forwardRef` | Required | **Deprecated** | `ref` as a plain prop (§17) |
| `<Context.Provider>` | Required | Deprecated (still works) | `<Context>` directly (§17) |
| `act` in tests | `react-dom/test-utils` | Moved to `react` | Update the import |
| `useFormState` | Experimental | Renamed | `useActionState` (§16) |

### 21.2 Step-by-step **[A]**

```bash
# 1. Upgrade packages
npm install react@19 react-dom@19
npm install -D @types/react@19 @types/react-dom@19
```

```tsx
// 2. ReactDOM.render → createRoot
// BEFORE: ReactDOM.render(<App />, document.getElementById("root"));
import { createRoot } from "react-dom/client";
createRoot(document.getElementById("root")!).render(<App />);
```

```tsx
// 3. Remove forwardRef wrappers (move ref into props)
// BEFORE: const Input = forwardRef<HTMLInputElement, {label:string}>(({label}, ref) => ...);
function Input({ label, ref }: { label: string; ref?: React.Ref<HTMLInputElement> }) {
  return <><label>{label}</label><input ref={ref} /></>;
}
```

```tsx
// 4. (Optional, cleaner) Context provider shorthand
// BEFORE: <MyContext.Provider value={v}>{children}</MyContext.Provider>
<MyContext value={v}>{children}</MyContext>

// 5. Delete propTypes — rely on TypeScript instead.
// 6. defaultProps → ES6 default params:  function X({ p = "default" }) {...}
// 7. Update test import:  import { act } from "react";   // was "react-dom/test-utils"
```

### 21.3 Codemods **[A]**

```bash
# React ships official codemods for the common, mechanical migrations:
npx codemod react/19/replace-reactdom-render
npx codemod react/19/replace-string-ref
npx codemod react/19/replace-act-import
npx codemod react/19/replace-use-form-state    # useFormState → useActionState
npx codemod --list react                         # see every available React 19 codemod
```

> **⚡ Version note:** the single most critical codemod is `replace-reactdom-render` (the only hard *removal* most apps hit). The rest are optional cleanups or automated renames. Run the type-checker after upgrading — `@types/react@19` tightens several types (notably stricter `ref` and `JSX` types).

---

## 22. Tips, Tricks & Gotchas

A grab-bag of the lessons that turn "it works" into "it works correctly and fast." Many restate principles from earlier sections — repetition here is deliberate, because these are exactly where real bugs come from.

### 22.1 State & rendering

```tsx
// TIP — All state updates in one handler are batched into ONE render (React 18+),
// even inside setTimeout/promises. Don't expect intermediate renders.

// TIP — Use a changing `key` to RESET a component's state by remounting it.
function ResetOnUser({ userId }: { userId: number }) {
  // When userId changes, React unmounts+remounts UserForm → all its state resets. Cleaner
  // than an effect that manually clears state (§8.4).
  return <UserForm key={userId} userId={userId} />;
}

// PITFALL — Deriving state from props with useState (it won't stay in sync):
function Bad({ userId }: { userId: number }) {
  const [id] = useState(userId);   // ❌ frozen at first render; ignores later userId changes
}
function Good({ userId }: { userId: number }) {
  const label = `User #${userId}`; // ✅ derive during render; or useMemo if expensive
  return <p>{label}</p>;
}
```

### 22.2 Effects (the most bug-prone area)

```tsx
// PITFALL 1 — Effect with no dependency array that sets state = infinite loop:
useEffect(() => { setCount((c) => c + 1); });   // ❌ runs after EVERY render → re-render → repeat

// PITFALL 2 — Stale closure: an effect/handler captures an old value because a dep is missing:
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1);   // ❌ `count` is frozen at the value from mount → always 0 → 1 forever
  }, 1000);
  return () => clearInterval(id);
}, []);                    // missing `count`
// ✅ Use the updater form so you don't depend on the captured value:
useEffect(() => {
  const id = setInterval(() => setCount((c) => c + 1), 1000);
  return () => clearInterval(id);
}, []);

// PITFALL 3 — Forgetting cleanup → leaked listeners/subscriptions:
useEffect(() => {
  const sub = store.subscribe(handle);
  return () => sub.unsubscribe();   // ✅ always tear down what you set up
}, []);

// REMINDER — "You might not need an effect" (§8.4): don't use effects to transform data,
// respond to events, or reset state. Those have better, effect-free solutions.
```

### 22.3 Conditional rendering & keys

```tsx
// PITFALL — `0` renders! Coerce list lengths/numbers to boolean before &&:
{count > 0 && <Badge n={count} />}     // ✅
{count && <Badge n={count} />}          // ❌ shows a stray "0" when count === 0

// PITFALL — index as key in a dynamic list breaks state/DOM matching on reorder (§7.3).
```

### 22.4 React 19 APIs

```tsx
// TIP — useActionState's previous state can be undefined on the first call; default it:
async function action(prev: MyState | undefined, fd: FormData): Promise<MyState> {
  const state = prev ?? DEFAULT_STATE;
  // ...
  return state;
}

// TIP — use(promise) needs a STABLE promise (§14.2) or you get an infinite Suspense loop.

// TIP — useFormStatus must live in a CHILD of the form, not the same component (§16.2).

// TIP — Return validation errors from actions; throw only for unexpected failures (§16.1).
```

### 22.5 Performance

```tsx
// TIP — Profile first (React DevTools Profiler). Don't add useMemo/useCallback "just in case."
// TIP — Collocate state: keep it as LOW in the tree as possible so updates re-render less.
// TIP — Virtualize long lists (100+ rows): react-window or @tanstack/react-virtual.
// TIP — Stable references for memo'd children: define constant objects/arrays OUTSIDE the
//        component, or wrap them in useMemo/useCallback. New literal each render defeats memo.
const STYLES = { color: "red" };                 // ✅ stable, defined once
function Parent() { return <Child style={STYLES} />; }   // Child (if memo'd) won't re-render needlessly
```

### 22.6 TypeScript with React

```tsx
import type { ReactNode, PropsWithChildren } from "react";

// Type children: PropsWithChildren utility, or explicit ReactNode.
function Card({ title, children }: PropsWithChildren<{ title: string }>) {
  return <section><h2>{title}</h2>{children}</section>;
}

// Generic components for reusable, type-safe widgets:
function Select<T extends string | number>({ options, value, onChange }: {
  options: T[]; value: T; onChange: (v: T) => void;
}) {
  return (
    <select value={String(value)} onChange={(e) => onChange(e.target.value as T)}>
      {options.map((o) => <option key={o} value={String(o)}>{o}</option>)}
    </select>
  );
}

// Derive a string-union type from an array with `as const`:
const ROLES = ["admin", "user", "moderator"] as const;
type Role = (typeof ROLES)[number];              // "admin" | "user" | "moderator"
```

### 22.7 Server Components & Actions

```tsx
// TIP — No hooks in Server Components (no useState/useEffect/useRef). Push interactivity into
//        small "use client" islands.
// TIP — Client Components can't IMPORT Server Components; pass them as children/props instead.
// TIP — In Next.js, Server Actions revalidate cached data with revalidatePath/revalidateTag.
```

---

## 23. Study Path & Build-to-Learn Projects

You learn React by *building*. Each phase below pairs concepts with a project that forces you to use them. Do not rush ahead — Phase 1 must be reflexive before the rest makes sense.

### Phase 1 — Core React (Weeks 1–2) **[B]**
1. **JSX** — how it compiles to function calls; the rules and the `{ }` boundary (§2).
2. **Components & props** — small composable pieces; one-way data flow (§3).
3. **`useState`** — local state, the render model, and **immutability** (§4, §5).
4. **Events** — handlers, SyntheticEvents, passing arguments (§6).
5. **Conditional & list rendering** — ternary/`&&`, `.map()`, **keys** and why (§7).
6. **`useEffect`** — the sync mental model, deps, cleanup, *and when not to use it* (§8).

**Build:** a dark/light theme toggle, a filterable product list, a multi-step form with validation.

### Phase 2 — Intermediate hooks & patterns (Weeks 3–4) **[I]**
1. **`useContext`** — theme/auth without prop drilling (§9.1, §11).
2. **`useReducer`** — complex state machines (cart, wizard) (§9.2).
3. **`useRef` & the DOM** — focus, measure, mutable values, portals (§9.3, §12).
4. **`useMemo` / `useCallback`** — understand *why*, not just how (§9.4–9.5, §13).
5. **Custom hooks** — `useFetch`, `useDebounce`, `useLocalStorage` (§10).
6. **`React.memo`** — preventing needless re-renders (§13.2).

**Build:** a Kanban board (`useReducer`), a modal/toast system (portals), a custom form hook.

### Phase 3 — React 19 new APIs (Week 5) **[I/A]**
1. **`use()`** — promises in Suspense, conditional context (§14).
2. **Actions & `useActionState`** — forms with pending/error/result (§15, §16.1).
3. **`useFormStatus`** — a submit button that knows its form (§16.2).
4. **`useOptimistic`** — instant UI with rollback (§16.3).
5. **`ref` as a prop, context-as-provider, document metadata** (§17).

**Build:** a full CRUD app using only React 19 form Actions — no `fetch` in `useEffect`.

### Phase 4 — Concurrent & performance (Week 6) **[A]**
1. **Suspense** — boundaries, nesting, `lazy` code splitting (§18.2).
2. **`useTransition` / `useDeferredValue`** — responsiveness under load (§18.3–18.4).
3. **The React Compiler** — automatic memoisation, the Rules of React (§13.4).
4. **DevTools Profiler** — find and fix *real* bottlenecks (§13.1).

**Build:** a dashboard with several independent data sections, each its own Suspense boundary.

### Phase 5 — Server Components & full-stack (Weeks 7–8) **[A]**
1. **RSC mental model** — server vs client, the composition rule (§20.1).
2. **`"use client"` / `"use server"`** — the boundary (§20).
3. **Server Actions** — mutations without API routes (§20.2).
4. **Pick Next.js 16 (App Router)** — the dominant framework in 2026.
5. **Error boundaries** — `onCaughtError`/`onUncaughtError` in production (§19).

**Build:** a blog or shop in Next.js: server-fetched data, Server Actions for comments/cart, per-section error boundaries.

### Phase 6 — Production patterns (ongoing)
1. **Server state** — **TanStack Query** for caching/dedup/refetch.
2. **Client state at scale** — **Zustand** or Jotai when Context re-renders bite.
3. **Forms** — **React Hook Form** + Zod for complex validation.
4. **Styling** — **Tailwind CSS** (or CSS Modules) for maintainable styles.
5. **Testing** — Vitest + `@testing-library/react`; test behaviour, not implementation.
6. **Accessibility** — `useId`, ARIA, focus management, keyboard navigation.

### Capstone projects (build these to prove fluency)

| Project | Skills exercised |
|---|---|
| **Real-time Todo** | `useState`/`useReducer`, `useOptimistic`, form Actions |
| **E-commerce catalog** | Server Components, Suspense, `lazy`, document metadata |
| **Multi-step wizard** | `useActionState`, `useFormStatus`, validation, transitions |
| **Chat app** | `useOptimistic`, WebSocket + `useEffect` cleanup, Suspense |
| **Admin dashboard** | data fetching, error boundaries, concurrent features, portals |
| **Full-stack blog** | Next.js + Server Actions + Server Components + caching |

### Key resources (for when you DO have internet)
- `react.dev` — official docs, hooks-first, rewritten for React 19 (read *"You Might Not Need an Effect"*).
- `react.dev/blog` — announcements for new APIs and the Compiler.
- `nextjs.org/docs` — the real-world home of Server Components & Actions.
- `tanstack.com/query` — TanStack Query docs for server state.
- `github.com/facebook/react/blob/main/CHANGELOG.md` — every API change.

---

*Guide accurate as of June 2026. React 19 is stable; the React Compiler is stable and opt-in; Server Components are stable in Next.js 16+ and Remix. All code is TypeScript (TSX) using `@types/react@19`. The throughline: describe the UI as a pure function of state, keep state immutable, let React reconcile — and reach for an effect only to sync with the world outside React.*
