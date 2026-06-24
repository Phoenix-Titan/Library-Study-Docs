# Motion (Animation for React) — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I've never animated anything in React" to "I build production-quality, accessible, performant motion design" — without an internet connection. Every concept comes with prose that explains *what it is*, *the logic / why it works this way*, *what it's for and when you reach for it*, *how to use it*, the *key props/parameters*, *best practices*, and *performance gotchas* — followed by heavily-commented, runnable code. Read top-to-bottom the first time; afterward, use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Motion** (the `motion` package at motion.dev), running with **React 19** and **Next.js 15/16** (current in 2026). The library was **renamed from "Framer Motion" to "Motion"** in late 2024 when it spun out from Framer into an independent project. The canonical package is now `motion`, with React imports from `motion/react`. The old `framer-motion` package still publishes updates and re-exports the identical API, so existing projects don't break — but new code should use `motion`. Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**; OS-specific notes are called out where relevant. Confirm exact APIs at motion.dev for changes after this guide's cutoff.

---

## Table of Contents

1. [Animation Fundamentals & Why a Library](#1-animation-fundamentals--why-a-library) **[B]**
2. [Setup, Install & Imports](#2-setup-install--imports) **[B]**
3. [The `motion` Component](#3-the-motion-component) **[B]**
4. [Transitions — Tween, Spring, Inertia](#4-transitions--tween-spring-inertia) **[B/I]**
5. [Gestures — Hover, Tap, Drag, InView](#5-gestures--hover-tap-drag-inview) **[I]**
6. [Variants & Orchestration](#6-variants--orchestration) **[I]**
7. [AnimatePresence — Exit Animations](#7-animatepresence--exit-animations) **[I]**
8. [Layout Animations & Shared Elements](#8-layout-animations--shared-elements) **[I/A]**
9. [Scroll Animations](#9-scroll-animations) **[I/A]**
10. [Motion Values & Hooks](#10-motion-values--hooks) **[A]**
11. [Keyframes, Colors & SVG Path Animation](#11-keyframes-colors--svg-path-animation) **[I]**
12. [Motion with Next.js App Router](#12-motion-with-nextjs-app-router) **[I/A]**
13. [Accessibility & Reduced Motion](#13-accessibility--reduced-motion) **[I]**
14. [Performance & Bundle Size](#14-performance--bundle-size) **[A]**
15. [Practical Recipes](#15-practical-recipes) **[I]**
16. [Tips, Tricks & Gotchas](#16-tips-tricks--gotchas) **[I/A]**
17. [Study Path & Build-to-Learn Projects](#17-study-path--build-to-learn-projects)

---

## 1. Animation Fundamentals & Why a Library

Before you touch a single `motion` component, it pays to understand *what animation actually is* on the web and *why* a library like Motion exists at all. Animation is the illusion of change over time: a value (a position, a size, a color, an opacity) starts at A and arrives at B, and instead of jumping instantly, the browser draws a series of in-between frames so the eye perceives smooth movement. At 60 frames per second the browser has roughly **16.7 milliseconds** to compute and paint each frame; miss that budget and the animation "janks" (stutters). Everything in this guide ultimately serves one goal: changing values over time *without* missing that frame budget.

### 1.1 What "animatable" means and how the browser renders [B]

The browser turns your HTML+CSS into pixels through a pipeline: **Style → Layout → Paint → Composite**. Understanding this pipeline is the single most important performance concept in the whole guide, because *which* property you animate decides *how much* of that pipeline must re-run every frame.

- **Layout** (also called *reflow*): the browser computes the geometry — where every box sits and how big it is. Animating `width`, `height`, `top`, `left`, `margin`, or `padding` forces layout to recompute, often for the whole page. Expensive.
- **Paint**: the browser fills in pixels — colors, shadows, borders, text. Animating `background-color` or `box-shadow` triggers paint but not layout. Moderately expensive.
- **Composite**: the browser takes already-painted layers and arranges them on the GPU. Animating `transform` (translate/scale/rotate) and `opacity` only touches this final compositor step. **Cheap** — these can run on the GPU's own thread, smoothly, even while JavaScript is busy.

The practical takeaway, repeated throughout this guide: **prefer animating `transform` and `opacity`.** Motion's shorthand props (`x`, `y`, `scale`, `rotate`) all compile to `transform`, which is why they're the workhorses of good motion design.

### 1.2 CSS transitions vs. CSS keyframes vs. a JS library — when to use each [B]

You do *not* always need Motion. Knowing when plain CSS suffices keeps your bundle small and your code simple. Here is the decision logic:

| Approach | Best for | Limitation |
|---|---|---|
| **CSS `transition`** | Simple state changes you can express in CSS: hover color, a button growing on `:hover`, a menu sliding when a class toggles | Can only animate between two states; can't orchestrate sequences; can't animate elements *leaving* the DOM; no spring physics |
| **CSS `@keyframes`** | Looping decorative animations: spinners, pulses, marquees — anything that runs independently of app state | Hard to coordinate with React state; values are fixed at author time, not driven by data; no exit animations |
| **Motion (JS library)** | Anything stateful or interactive: enter/exit on mount/unmount, drag, gesture-driven values, shared-element transitions, scroll-linked effects, spring physics, stagger orchestration, animating to `height: auto` | ~18–34 KB of JavaScript; must run on the client |

**The logic / why a library:** React's render model fights animation in two specific ways that CSS cannot fix. First, when a component unmounts, React rips it out of the DOM *immediately* — there is no opportunity for an exit animation. Second, React re-renders are discrete snapshots, while animation is continuous; driving a smooth 60fps value through `useState` would re-render your component sixty times a second. Motion solves both: it intercepts unmounts (via `AnimatePresence`) to run exit animations, and it drives continuous values (via *motion values*) *outside* React's render cycle so the DOM updates without re-rendering. Add spring physics, FLIP-based layout animations, and a declarative gesture system on top, and you have capabilities that are simply not expressible in hand-written CSS.

**Best practice:** reach for CSS first for trivial hover/focus transitions on static markup; reach for Motion the moment you need exit animations, physics, orchestration, drag, scroll-linking, or layout/shared-element transitions.

### 1.3 Declarative animation — the core mental model [B]

Motion is **declarative**, like React itself. You don't write a `requestAnimationFrame` loop, compute interpolation math, or manage timers. Instead you *describe the states* — "start invisible and 20px low; end visible at rest" — and tell Motion *how* to move between them (the transition). Motion runs the loop, does the interpolation, and updates the DOM. This is the same shift in thinking React asks of you for UI ("describe what the UI should look like for this state") applied to motion ("describe what the element should look like in each state").

```tsx
// The entire mental model in one component:
<motion.div
  initial={{ opacity: 0, y: 20 }}   // WHAT the start state is
  animate={{ opacity: 1, y: 0 }}    // WHAT the end state is
  transition={{ duration: 0.5 }}    // HOW to get from start to end
/>
// You never wrote a loop. Motion interpolates opacity 0→1 and y 20→0 over 0.5s.
```

### 1.4 Cross-references [B]

Motion is a React library, so it assumes you're comfortable with components, props, `useState`, `useRef`, and `useEffect` — see the **React 19** guide. Because Motion runs only in the browser, its interaction with **Next.js**'s Server Components (the `"use client"` boundary) is a recurring theme; Section 12 covers it in depth and cross-references the **Next.js** guide.

---

## 2. Setup, Install & Imports

### 2.1 Installing [B]

Motion ships as a single npm package. For a brand-new project, install `motion`. If you're maintaining an older codebase that already has `framer-motion`, that still works — just don't install both, because two copies will fight over the same DOM elements and produce subtle bugs.

```bash
# The modern package — recommended for all new work
npm install motion

# Legacy package — still maintained, identical API, only the import path differs
npm install framer-motion
```

**The logic / why two packages:** the rebrand from Framer Motion to Motion (late 2024) renamed the npm package from `framer-motion` to `motion` and moved the project to motion.dev as an independent open-source library. To avoid breaking the millions of projects on `framer-motion`, the old package continues to publish and simply re-exports the same code. So `import { motion } from "framer-motion"` and `import { motion } from "motion/react"` give you the *same* component today.

> **⚡ Version note:** If a tutorial imports from `"framer-motion"`, the API is essentially identical — swap the import to `"motion/react"` and it will work. The only structural change is the new top-level *vanilla JS* entry point (see below).

### 2.2 The two entry points: `motion` vs `motion/react` [B]

```
motion         → framework-agnostic, imperative API: animate(), scroll(), inView(), stagger()
                 (vanilla JS, Web Components, any framework). Tiny — ~5 KB.
motion/react   → React components and hooks: motion.div, AnimatePresence, useScroll, etc.
```

This guide focuses on **`motion/react`**. The bare `motion` package is for animating non-React contexts; you'll occasionally see it referenced for the lightweight `animate()` function, but in a React app you'll almost always want `motion/react`.

### 2.3 The imports you'll actually use [B]

Group your imports by purpose so the file reads clearly. Here is essentially the complete surface area you'll touch across this guide:

```tsx
// ─── Core component + presence (the two you import most often) ─────────────────
import { motion, AnimatePresence } from "motion/react"

// ─── Hooks ────────────────────────────────────────────────────────────────────
import {
  useMotionValue,       // create a reactive value that lives OUTSIDE React renders
  useTransform,         // derive a new motion value from one or more existing ones
  useSpring,            // wrap a motion value in spring physics (smooth follow)
  useScroll,            // track page or element scroll progress as a motion value
  useAnimate,           // imperative animation with a scoped ref (sequences)
  useInView,            // boolean: is this element currently in the viewport?
  useReducedMotion,     // boolean: has the user requested reduced motion?
  useMotionValueEvent,  // subscribe to a motion value's changes (bridge to state)
  useMotionTemplate,    // build a string (e.g. CSS) from multiple motion values
} from "motion/react"

// ─── Providers, config & bundle-size helpers ──────────────────────────────────
import { MotionConfig, LazyMotion, domAnimation, domMax, m } from "motion/react"

// ─── Layout helpers ───────────────────────────────────────────────────────────
import { LayoutGroup, Reorder } from "motion/react"
```

### 2.4 Verify the setup [B]

Drop this in a component and render it. If the box fades and slides up on mount, Motion is wired correctly. (In Next.js App Router you need the `"use client"` directive — see Section 12 for the full explanation.)

```tsx
// src/components/TestMotion.tsx
"use client" // required in Next.js App Router; harmless in a plain Vite/CRA app

import { motion } from "motion/react"

export function TestMotion() {
  return (
    <motion.div
      initial={{ opacity: 0, y: 20 }}      // start hidden, 20px below resting position
      animate={{ opacity: 1, y: 0 }}        // animate to visible, at rest
      transition={{ duration: 0.5 }}        // over half a second
      className="p-8 bg-blue-500 text-white rounded-xl"
    >
      Motion is working!
    </motion.div>
  )
}
```

> **Best practice:** keep Motion imports in client components only. In a Next.js app, a single forgotten `"use client"` produces a confusing server-rendering error (covered in §12) — so make a habit of putting the directive at the top of *any* file that imports from `motion/react`.

---

## 3. The `motion` Component

### 3.1 What it is and the logic behind it [B]

The `motion` object is a collection of **enhanced versions of every HTML and SVG element**. `motion.div` is a `<div>` that understands animation props; `motion.button` is a `<button>` that does the same. You use them exactly like the normal element — same children, same `className`, same event handlers — but with three extra animation props (`initial`, `animate`, `exit`) plus gesture props (`whileHover`, etc.) and a `transition`.

**The logic / why a special component:** a plain `<div>` has no concept of "animate to this state." Motion wraps the element so it can (a) attach a *motion value* to each animatable property, (b) run a render loop that interpolates those values over time, and (c) write the results directly to the element's `style` — bypassing React re-renders for performance. The `motion.` prefix is the seam where ordinary React markup gains these superpowers.

```tsx
// Every HTML element has a motion. counterpart:
<motion.div />   <motion.span />  <motion.button />  <motion.img />
<motion.section /> <motion.h1 /> <motion.ul /> <motion.li /> <motion.nav />

// So does every SVG element:
<motion.svg /> <motion.path /> <motion.circle /> <motion.rect /> <motion.g />

// ⚡ For custom components, wrap them once with motion.create() (was motion() in older versions):
const MotionLink = motion.create(Link)  // now <MotionLink animate={...} />
// The wrapped component MUST forward the ref and spread props/style onto a DOM element.
```

> **⚡ Version note:** the helper to wrap a custom component is `motion.create(Component)` in current Motion. Older Framer Motion code used `motion(Component)` as a callable — that still works for back-compat but `motion.create` is the documented form.

### 3.2 The three animation props: `initial`, `animate`, `exit` [B]

These are the heart of the API. Each takes a plain JS object whose keys are CSS properties (camelCased) or Motion transform shorthands, and whose values are the targets.

| Prop | What it is / when it runs |
|---|---|
| `initial` | The state *before* the element appears. Set on first mount, before any animation. Pass `initial={false}` to skip the mount animation entirely (start already at the `animate` state). |
| `animate` | The target state Motion animates *toward*. When this object changes (e.g. driven by React state), Motion animates to the new values. This is the prop you'll use most. |
| `exit` | The state to animate to when the element is *removed* from the React tree. Requires wrapping in `<AnimatePresence>` (Section 7) — otherwise React unmounts instantly and `exit` never gets a chance to run. |

```tsx
"use client"
import { motion } from "motion/react"

export function FadeIn() {
  return (
    <motion.div
      initial={{ opacity: 0, scale: 0.8 }}  // BEFORE mount: invisible + slightly small
      animate={{ opacity: 1, scale: 1 }}     // AFTER mount: fade + scale to full size
      exit={{ opacity: 0, scale: 0.8 }}      // ON unmount: shrink + fade (needs AnimatePresence)
    >
      Hello
    </motion.div>
  )
}
```

> **Why `initial` matters for first paint:** without `initial`, the element renders at its natural CSS state and then snaps to `animate` — often a visible flash. Setting `initial` guarantees a clean starting point. Conversely, `initial={false}` is the right call for elements that should appear already in place (e.g. content present on page load that should only animate on *later* state changes).

### 3.3 Animatable CSS properties [B]

You can animate almost any numeric or color CSS property by its **camelCase** name. Motion knows how to interpolate numbers, pixels, percentages, colors, and even complex values like `box-shadow` and `filter`.

```tsx
<motion.div
  animate={{
    // ─── Geometry (NOTE: these trigger layout — see §14 performance) ───────────
    width: 200,
    height: 100,
    borderRadius: 20,
    padding: 16,
    // ─── Color & paint ─────────────────────────────────────────────────────────
    backgroundColor: "#3b82f6",
    color: "#ffffff",
    borderColor: "rgba(0,0,0,0.2)",
    boxShadow: "0px 10px 30px rgba(0,0,0,0.2)",
    // ─── Typography ────────────────────────────────────────────────────────────
    fontSize: 24,
    letterSpacing: 2,
    // ─── Filters & opacity (compositor-friendly) ───────────────────────────────
    filter: "blur(4px)",
    opacity: 1,
  }}
/>
```

> **Gotcha — bare numbers:** numeric values without units are treated as pixels for length properties (`width: 200` → `200px`) and as plain numbers for unitless properties (`opacity: 1`, `scale: 1.2`). If you need a non-pixel unit, pass a string: `width: "50%"`.

### 3.4 The transform shorthand — your performance workhorse [B]

Rather than animating the CSS `transform` string, Motion gives you **individual props for each transform function**. This is more than convenience: animating these is GPU-composited (the cheap path from §1.1), and Motion can spring/interpolate each independently.

| Motion prop | CSS equivalent | Units / notes |
|---|---|---|
| `x` / `y` / `z` | `translateX/Y/Z` | pixels by default; strings like `"100%"` allowed |
| `rotate` | `rotate` | degrees |
| `rotateX` / `rotateY` / `rotateZ` | `rotateX(...)` etc. | degrees (3D needs `perspective`) |
| `scale` | `scale` | `1` = normal, `1.2` = 120% |
| `scaleX` / `scaleY` | `scaleX/Y(...)` | `1` = normal |
| `skew` / `skewX` / `skewY` | `skew(...)` | degrees |
| `originX` / `originY` / `originZ` | `transform-origin` | `0`–`1` (fraction) or px/% string |
| `perspective` | `perspective` | pixels — set on the parent for 3D children |

```tsx
"use client"
import { motion } from "motion/react"

export function TransformDemo() {
  return (
    <motion.div
      initial={{ x: -100, opacity: 0, rotate: -10 }}  // off-screen-left, tilted, hidden
      animate={{ x: 0, opacity: 1, rotate: 0 }}         // slide in, straighten, reveal
      transition={{ type: "spring", stiffness: 100 }}   // springy entrance
      className="w-32 h-32 bg-purple-500 rounded-lg"
    />
  )
}
```

> **Best practice:** when you have a choice, express movement with `x`/`y` (transform) rather than `left`/`top` (layout), and size changes with `scale` rather than `width`/`height`. The visual result is similar but the performance is night-and-day. The exception is when scaling distorts content (text, child layout) — then animate real size, ideally with the `layout` prop (§8).

### 3.5 Driving animation from React state [B]

Because `animate` re-animates whenever its object changes, the most common pattern is to compute the target from state. Flip the state and Motion smoothly transitions.

```tsx
"use client"
import { useState } from "react"
import { motion } from "motion/react"

export function ToggleBox() {
  const [isOpen, setIsOpen] = useState(false)

  return (
    <motion.div
      // Each property animates whenever isOpen flips, using one transition:
      animate={{
        width: isOpen ? 300 : 100,
        height: isOpen ? 200 : 100,
        backgroundColor: isOpen ? "#10b981" : "#3b82f6",
        borderRadius: isOpen ? 16 : 50,
      }}
      transition={{ duration: 0.4, ease: "easeInOut" }}
      onClick={() => setIsOpen(!isOpen)}   // motion elements take normal event handlers
      className="cursor-pointer"
    />
  )
}
```

### 3.6 The `style` prop and motion values [B]

A `motion.*` element's `style` prop accepts both normal CSS values *and* `MotionValue` objects (Section 10). This is how scroll- and drag-driven animations feed continuous values into an element without React re-rendering.

```tsx
// myMotionValue and opacityValue are MotionValues; the div updates as they change,
// with zero React re-renders. (Full explanation in §10.)
<motion.div style={{ x: myMotionValue, opacity: opacityValue }} />
```

> **Gotcha — `animate` vs `style`:** use `animate` for *target-based* animation (go to this state). Use `style` with motion values for *continuous, externally-driven* values (scroll position, drag, mouse). Don't put the same property in both — they'll conflict over who controls the DOM.

---

## 4. Transitions — Tween, Spring, Inertia

The `transition` prop controls *how* an animation moves from its current value to the target. This is where the *feel* of your UI lives — the difference between a cheap-feeling linear slide and a satisfying, physical spring. There are three transition **types**, and choosing the right one is a core skill.

### 4.1 The big picture: tween vs spring vs inertia [B]

- **`tween`** — duration-based. You specify how *long* the animation takes and an *easing curve*. Deterministic and predictable. Best for opacity fades, color changes, and any animation where exact timing matters (e.g. coordinated with other timed events).
- **`spring`** — physics-based. You specify the *physical properties* of a spring (stiffness, damping, mass) and Motion simulates real spring motion. There's no fixed duration — it settles when the physics says so. Best for anything interactive (buttons, drag release, layout shifts) because it feels natural and responsive.
- **`inertia`** — deceleration-based. The value carries momentum and decelerates to a stop, optionally bouncing off boundaries. Used almost exclusively for *drag momentum* (the element keeps gliding after you release it).

**The logic / why springs feel better:** real objects don't move at a constant speed and stop abruptly — they accelerate, overshoot slightly, and settle. A spring simulation captures this, so spring-driven UI reads as physical and alive. A tween with `ease: "linear"` reads as mechanical. For interactions especially, springs are usually the right default.

### 4.2 Tween: duration & easing [B]

```tsx
<motion.div
  animate={{ x: 100 }}
  transition={{
    type: "tween",      // duration-based (this is the default for most properties)
    duration: 0.6,      // seconds the animation takes
    ease: "easeOut",    // the easing curve (see table below)
    delay: 0.2,         // seconds to wait before starting
  }}
/>
```

**Easing** describes how speed varies across the animation's duration. The right easing dramatically changes perceived quality:

| Easing | Feel / when to use |
|---|---|
| `"linear"` | Constant speed. Mechanical — best for continuous loops (spinners), rarely for UI transitions. |
| `"easeIn"` | Starts slow, ends fast. Good for elements *leaving* (accelerate away). |
| `"easeOut"` | Starts fast, ends slow. The most natural for elements *entering* — they arrive and settle. **Best default for reveals.** |
| `"easeInOut"` | Slow at both ends. Smooth, balanced — good for elements that move and stay (toggles, repositioning). |
| `"circIn"` / `"circOut"` / `"circInOut"` | Circular curve — sharper acceleration than ease. |
| `"backIn"` / `"backOut"` / `"backInOut"` | Overshoots slightly past the target then returns — playful. |
| `"anticipate"` | Pulls *backward* before moving forward — like a wind-up. Great for attention-grabbing entrances. |
| `[0.17, 0.67, 0.83, 0.67]` | A custom cubic-bézier (four control points). Full control; copy curves from easing tools. |

> **Best practice:** use `easeOut` for things appearing and `easeIn` for things disappearing — it matches how we expect objects to enter and leave a space.

### 4.3 Spring: stiffness, damping, mass — explained [B/I]

A spring has no `duration`. Instead you describe a physical spring and Motion runs the simulation. The three parameters that matter:

- **`stiffness`** — how strong the spring is (the constant *k* in physics). Higher = the spring pulls harder toward the target = **faster, snappier** motion. Typical range **100–500**. Think of it as "how eager is it to reach the destination."
- **`damping`** — friction / resistance that absorbs energy. Higher = **less bounce**, settles sooner. Lower = more oscillation (wobble). Typical range **10–40**. Think of it as "how much does it resist overshooting."
- **`mass`** — the weight of the object. Higher = **heavier, slower** to start and stop, more momentum. Default `1`. Increasing mass makes motion feel weightier and laggier.

The interplay: high stiffness + low damping = a fast, bouncy spring. High stiffness + high damping = a fast spring with no bounce (snappy click). Low stiffness + high mass = a slow, heavy drift.

```tsx
<motion.div
  animate={{ x: 200, rotate: 45 }}
  transition={{
    type: "spring",
    stiffness: 300,    // snappy pull toward the target
    damping: 20,       // a little bounce, but settles cleanly
    mass: 1,           // default weight
    velocity: 0,       // initial velocity (handy when handing off from a gesture)
    restDelta: 0.001,  // how close to target before it's considered "at rest" and stops
    restSpeed: 0.001,  // how slow before it's considered at rest
  }}
/>
```

**Alternative spring API — `duration` + `bounce`:** if thinking in physics feels unintuitive, Motion lets you describe a spring by an approximate `duration` (seconds) and a `bounce` value (`0` = no bounce, up to ~`1` = very bouncy). Motion converts this to stiffness/damping internally. This is often easier for designers.

```tsx
transition={{ type: "spring", duration: 0.6, bounce: 0.25 }}
```

**Spring tuning quick reference:**

| Desired feel | stiffness | damping | or duration / bounce |
|---|---|---|---|
| Snappy UI click | 400–600 | 25–35 | `0.3` / `0` |
| Bouncy / playful | 200–300 | 10–15 | `0.6` / `0.4` |
| Heavy / slow drift | 80–120 | 20–30 | `1.0` / `0.1` |
| Overdamped (no bounce at all) | 200 | 40+ | `0.5` / `0` |

> **Best practice:** start from one of these presets, then tweak a single parameter at a time. Changing stiffness and damping together makes it hard to learn what each does.

### 4.4 Inertia: drag momentum [I]

Inertia transitions take a starting `velocity` and decelerate. They're mostly applied automatically by the `drag` gesture (the element coasts after release), but you can tune the behaviour.

```tsx
<motion.div
  drag
  dragConstraints={{ left: 0, right: 200 }}
  // Motion uses an inertia transition automatically on drag release; tune it like so:
  transition={{
    type: "inertia",
    velocity: 50,        // initial velocity when released (px/s)
    power: 0.8,          // multiplier applied to velocity (higher = travels further)
    timeConstant: 700,   // ms — larger = decelerates more slowly (glides longer)
    bounceStiffness: 400,// spring stiffness when it hits a constraint boundary
    bounceDamping: 40,   // spring damping at the boundary
    min: 0, max: 200,    // boundaries it will decelerate toward / bounce off
  }}
/>
```

### 4.5 Repeating animations [B]

For loops (spinners, pulses, attention-grabbers), use `repeat`:

```tsx
<motion.div
  animate={{ rotate: 360 }}
  transition={{
    duration: 2,
    repeat: Infinity,    // a finite number, or Infinity for endless
    repeatType: "loop",  // "loop" | "reverse" | "mirror" (see table)
    repeatDelay: 0.5,    // pause between each repetition
    ease: "linear",      // linear is correct for continuous rotation
  }}
/>
```

| `repeatType` | Behaviour |
|---|---|
| `"loop"` | Jumps back to the start and plays forward again (a hard reset each cycle). |
| `"reverse"` | Plays forward, then backward — like CSS `animation-direction: alternate`. Smooth back-and-forth. |
| `"mirror"` | Like `reverse`, but the *easing* is mirrored too, so acceleration feels symmetric. Best for pulses. |

### 4.6 Per-property transitions [I]

Each property inside `animate` can carry its own transition, overriding the default. This lets a single state change move different properties with different timing — a key tool for sophisticated, layered motion.

```tsx
<motion.div
  animate={{ x: 100, opacity: 1, rotate: 180 }}
  transition={{
    duration: 0.5,                                   // default for any property not overridden
    x: { type: "spring", stiffness: 200 },           // x springs in
    opacity: { duration: 0.3, ease: "easeIn" },      // opacity fades faster, linearly-ish
    rotate: { delay: 0.4, duration: 1 },             // rotation waits, then turns slowly
  }}
/>
```

### 4.7 The `transition` on the target object [I]

You can also attach a `transition` *inside* an `animate` or variant object. This is essential for variants (Section 6), where the transition needs to travel with the named state:

```tsx
<motion.div animate={{ x: 100, transition: { type: "spring", bounce: 0.3 } }} />
```

> **Gotcha — `staggerChildren` lives on the parent's transition.** Stagger orchestration (each child starting after the previous) is configured in the *parent's* transition via `staggerChildren`, not on the children. Full pattern in Section 6.

---

## 5. Gestures — Hover, Tap, Drag, InView

Gestures are Motion's interactive layer. Each gesture prop accepts the **same target-state object as `animate`** — so "grow on hover" is just `whileHover={{ scale: 1.05 }}`. While the gesture is active, Motion animates to that state; when it ends, it animates back to the base `animate`/`initial` state. This automatic return is what makes gestures feel so clean.

### 5.1 Hover, tap, focus [I]

These three cover almost all button and card interactivity. The logic: `whileHover` is active while the pointer is over the element, `whileTap` while it's being pressed (mouse down or touch), and `whileFocus` while keyboard-focused (crucial for accessibility — keyboard users should see the same feedback as mouse users).

```tsx
"use client"
import { motion } from "motion/react"

export function InteractiveCard() {
  return (
    <motion.button
      whileHover={{                                  // active while hovered
        scale: 1.05,
        boxShadow: "0px 10px 30px rgba(0,0,0,0.2)",
        y: -4,
      }}
      whileTap={{ scale: 0.97, y: 0 }}              // active while pressed (gives tactile feedback)
      whileFocus={{ outline: "2px solid #3b82f6", outlineOffset: 4 }} // keyboard a11y
      transition={{ type: "spring", stiffness: 400, damping: 25 }}    // snappy spring feel
      className="px-6 py-3 bg-blue-500 text-white rounded-xl cursor-pointer"
    >
      Click me
    </motion.button>
  )
}
```

> **Best practice:** always pair `whileTap` with `whileHover` on interactive elements — the press-down scale is the single cheapest way to make a UI feel responsive and physical. And include `whileFocus` so keyboard users get equivalent feedback. There are also `onHoverStart`/`onHoverEnd` and `onTap`/`onTapStart`/`onTapCancel` callbacks if you need to run logic, not just animate.

### 5.2 `whileInView` — scroll-triggered reveals [I]

`whileInView` animates the element when it scrolls into the viewport. Internally it uses an `IntersectionObserver`, so it's efficient (no scroll-event spam). This is the simplest, most common scroll animation — fade-up reveals as the user scrolls down a page.

```tsx
"use client"
import { motion } from "motion/react"

export function RevealSection() {
  return (
    <motion.div
      initial={{ opacity: 0, y: 60 }}        // start hidden + below
      whileInView={{ opacity: 1, y: 0 }}      // animate when it enters the viewport
      viewport={{
        once: true,                           // animate ONCE; don't re-hide on scroll-away
        amount: 0.3,                          // trigger when 30% of the element is visible
        margin: "0px 0px -100px 0px",         // shrink the viewport bottom by 100px (fire earlier)
      }}
      transition={{ duration: 0.6, ease: "easeOut" }}
    >
      I animate when scrolled into view
    </motion.div>
  )
}
```

**Key `viewport` options:**

- `once: true` — animate only the first time it enters. Almost always what you want for content reveals; without it, content re-animates every time it scrolls in/out, which is distracting.
- `amount` — `0`–`1` fraction of the element that must be visible to trigger (or `"all"`/`"some"`). Higher = waits until more is on screen.
- `margin` — a CSS `rootMargin` string that grows/shrinks the detection box. Negative bottom margin fires the reveal *before* the element fully enters; positive grows the trigger zone.

> **Gotcha:** `whileInView` is distinct from `useScroll` (Section 9). `whileInView` is *scroll-triggered* (fires once at a threshold, then animates on its own clock). `useScroll` is *scroll-linked* (the animation value is tied frame-by-frame to scroll position, so scrolling back un-animates it). Choose triggered for reveals, linked for parallax/progress.

### 5.3 Drag [I]

Adding `drag` makes an element draggable with the pointer. By itself it's unconstrained; in practice you almost always constrain it to a boundary and add release momentum (handled automatically via an inertia transition).

```tsx
"use client"
import { useRef } from "react"
import { motion } from "motion/react"

export function DraggableCard() {
  const constraintRef = useRef(null)   // a ref to the boundary element

  return (
    // The parent defines the drag area:
    <div ref={constraintRef} className="relative w-80 h-80 bg-gray-100 rounded-2xl overflow-hidden">
      <motion.div
        drag                              // enable dragging on BOTH axes
        // drag="x"                       // ...or restrict to a single axis: "x" or "y"
        dragConstraints={constraintRef}   // keep the element inside the boundary element
        // dragConstraints={{ top: 0, bottom: 200, left: 0, right: 200 }} // ...or pixel bounds
        dragElastic={0.2}                 // 0 = rigid stop at edge; 1 = fully elastic past edge
        dragMomentum={true}               // coast (inertia) after release
        dragSnapToOrigin={false}          // if true, springs back to start on release
        whileDrag={{ scale: 1.1, zIndex: 10, cursor: "grabbing" }} // feedback while dragging
        className="absolute top-4 left-4 w-24 h-24 bg-purple-500 rounded-xl cursor-grab"
      />
    </div>
  )
}
```

**Key drag props:**

- `dragConstraints` — either a ref to a container (drag stays inside it) or a `{ top, right, bottom, left }` pixel object. Without it, the element can be dragged anywhere.
- `dragElastic` — `0`–`1`. How far the element may be pulled *past* its constraints before resisting. `0` is a hard wall; `0.2` gives a satisfying rubber-band feel.
- `dragMomentum` — whether the element coasts (inertia) after release. `true` feels physical; `false` stops dead.
- `dragSnapToOrigin` — return to the start position on release (good for swipe-to-dismiss-or-snap-back patterns).

### 5.4 Drag callbacks and the `info` object [I]

For logic during a drag — swipe-to-dismiss, snap-to-side, reordering — use the callbacks. Each receives the native `event` and an `info` object with rich pointer data.

```tsx
<motion.div
  drag
  onDragStart={(event, info) => {
    console.log("started at page point", info.point)   // { x, y } in page coordinates
  }}
  onDrag={(event, info) => {
    console.log("moved this frame", info.delta)        // { x, y } since last frame
    console.log("offset from start", info.offset)      // { x, y } total since drag began
    console.log("velocity", info.velocity)             // { x, y } px/s — useful for swipe detection
  }}
  onDragEnd={(event, info) => {
    // Swipe-to-dismiss: if flung left fast OR dragged far left, dismiss it
    if (info.velocity.x < -500 || info.offset.x < -100) {
      dismiss()
    }
  }}
/>
```

> **Best practice for touch:** set `style={{ touchAction: "none" }}` on a draggable element so the browser doesn't try to scroll the page while the user drags. For horizontal-only drags inside a vertically scrolling page, use `drag="x"` plus `touchAction: "pan-y"`.

### 5.5 The `Reorder` components — drag-to-reorder lists [A]

For the common "drag list items to reorder" pattern, Motion provides `Reorder.Group` and `Reorder.Item`, which handle the drag, the layout animation of siblings shifting, and calling back with the new order.

```tsx
"use client"
import { useState } from "react"
import { Reorder } from "motion/react"

export function ReorderableList() {
  const [items, setItems] = useState(["Apples", "Bananas", "Cherries", "Dates"])

  return (
    // values = current array; onReorder = called with the new order as you drag
    <Reorder.Group axis="y" values={items} onReorder={setItems} className="space-y-2">
      {items.map((item) => (
        // value identifies which item this is; Motion animates siblings out of the way
        <Reorder.Item
          key={item}
          value={item}
          whileDrag={{ scale: 1.03, boxShadow: "0 8px 24px rgba(0,0,0,0.15)" }}
          className="p-4 bg-white rounded-lg shadow cursor-grab active:cursor-grabbing"
        >
          {item}
        </Reorder.Item>
      ))}
    </Reorder.Group>
  )
}
```

---

## 6. Variants & Orchestration

### 6.1 What variants are and why they exist [I]

A **variant** is a named animation state defined once in an object, then referenced by name in the animation props. Instead of writing the same target object inline on `initial`/`animate`/gesture props, you give states names like `"hidden"` and `"visible"` and pass the *string*.

```tsx
const cardVariants = {
  hidden: { opacity: 0, y: 50, scale: 0.9 },   // a named state
  visible: { opacity: 1, y: 0, scale: 1 },     // another named state
}

<motion.div variants={cardVariants} initial="hidden" animate="visible" />
```

This alone is just tidier. But variants unlock two things impossible with inline objects, and *these* are the real reason to use them:

1. **Propagation:** when a parent changes variant, all `motion` children with the *same variant names* automatically follow — no need to thread `animate` through every child.
2. **Orchestration:** the parent can sequence its children — staggering them, delaying them, and choosing whether the parent animates before or after its children.

**The logic / why propagation works:** when a `motion` element receives a *string* for `animate` (a variant name) instead of an object, it doesn't just animate itself — it passes that variant name down to its `motion` descendants. Each descendant looks up *its own* `variants` object for that name. So one state change at the top cascades through the tree, with each element animating to its own definition of "visible." This is the cleanest way to coordinate many elements.

### 6.2 Basic variants [I]

```tsx
"use client"
import { motion } from "motion/react"

// Define states as named objects (any names you like):
const cardVariants = {
  hidden: { opacity: 0, y: 50, scale: 0.9 },
  visible: { opacity: 1, y: 0, scale: 1 },
}

export function AnimatedCard() {
  return (
    <motion.div
      variants={cardVariants}
      initial="hidden"             // reference the named start state
      animate="visible"            // reference the named end state
      transition={{ duration: 0.5 }}
    >
      Card content
    </motion.div>
  )
}
```

### 6.3 Parent → child propagation + staggerChildren [I]

This is the pattern you'll reach for constantly: a list or grid whose items reveal one after another. Note that the *children carry no `initial`/`animate`* — they inherit the variant name from the parent and look it up in their own `variants`.

```tsx
"use client"
import { motion } from "motion/react"

// The PARENT defines the orchestration in its variant's `transition`:
const containerVariants = {
  hidden: {},                          // parent has nothing to animate itself
  visible: {
    transition: {
      staggerChildren: 0.1,            // each child starts 100ms after the previous — THE stagger
      delayChildren: 0.2,              // wait 200ms before the FIRST child starts
      // staggerDirection: -1,         // reverse the order (last child first)
      // when: "beforeChildren",       // parent finishes its own animation before children begin
      // when: "afterChildren",        // parent waits for all children to finish before it animates
    },
  },
}

// Each CHILD defines its own look for the SAME variant names:
const itemVariants = {
  hidden: { opacity: 0, y: 20 },
  visible: {
    opacity: 1,
    y: 0,
    transition: { type: "spring", stiffness: 200, damping: 20 },
  },
}

const items = ["First", "Second", "Third", "Fourth", "Fifth"]

export function StaggeredList() {
  return (
    <motion.ul
      variants={containerVariants}
      initial="hidden"
      animate="visible"               // this single change cascades to every child
      className="space-y-3"
    >
      {items.map((item) => (
        <motion.li
          key={item}
          variants={itemVariants}
          // NO initial/animate here — inherited from the parent!
          className="p-4 bg-white rounded-lg shadow"
        >
          {item}
        </motion.li>
      ))}
    </motion.ul>
  )
}
```

**Orchestration parameters explained:**

- `staggerChildren` — seconds between each child's start. The heart of staggered reveals.
- `delayChildren` — seconds to wait before the first child begins (after the parent's `when` condition is met).
- `staggerDirection` — `1` (default, first→last) or `-1` (last→first).
- `when` — `"beforeChildren"` makes the parent finish before children animate (e.g. a container fades in, *then* its contents); `"afterChildren"` reverses it (children leave, *then* the container) — invaluable for exit animations.

> **Best practice:** keep `staggerChildren` small (0.05–0.12s). Large staggers on long lists make the last items feel sluggish. For very long lists, prefer `whileInView` per-section over one giant stagger.

### 6.4 Dynamic variants — functions and `custom` [A]

A variant can be a *function* that receives a `custom` prop you pass on the element. This lets each instance compute its own values — most often an index-based delay.

```tsx
"use client"
import { motion } from "motion/react"

const itemVariants = {
  hidden: { opacity: 0, x: -50 },
  // The `visible` state is a function of the custom value (here, the index i):
  visible: (i: number) => ({
    opacity: 1,
    x: 0,
    transition: { delay: i * 0.1, type: "spring", stiffness: 200 },
  }),
}

export function CustomStagger({ items }: { items: string[] }) {
  return (
    <ul>
      {items.map((item, i) => (
        <motion.li
          key={item}
          variants={itemVariants}
          initial="hidden"
          animate="visible"
          custom={i}                  // passed into the variant function as its argument
        >
          {item}
        </motion.li>
      ))}
    </ul>
  )
}
```

> **When to use `custom` vs `staggerChildren`:** `staggerChildren` is simpler and automatic, but it requires the parent/child variant structure. `custom` gives per-element control (varying delays, directions, or values by data) and works without a parent orchestrator — reach for it when each element needs *different* timing or values.

### 6.5 Variants with gestures — coordinated multi-element effects [I]

Because gesture props also accept variant names, a parent's hover state can drive children. This is how you make, say, an overlay fade in when a *card* is hovered (not the overlay itself).

```tsx
const cardVariants = { rest: { scale: 1 }, hover: { scale: 1.02 } }
const overlayVariants = { rest: { opacity: 0 }, hover: { opacity: 1 } }

export function HoverCard() {
  return (
    // The PARENT owns the hover state; the child reacts automatically via the shared name:
    <motion.div
      variants={cardVariants}
      initial="rest"
      whileHover="hover"            // hovering the card sets variant "hover" on the whole subtree
      className="relative w-64 h-40 bg-blue-500 rounded-xl overflow-hidden"
    >
      <motion.div
        variants={overlayVariants} // child looks up "hover" in its OWN variants
        className="absolute inset-0 bg-black/20 flex items-center justify-center text-white"
      >
        Hover overlay
      </motion.div>
    </motion.div>
  )
}
```

> **Gotcha — child variant names must match.** Propagation only reaches children whose `variants` object contains the parent's current variant name. If the names don't match (`"hover"` vs `"hovered"`), the child silently won't animate. This is the most common variants bug.

---

## 7. AnimatePresence — Exit Animations

### 7.1 The problem it solves and the logic [I]

When React removes a component from the tree, it removes it from the DOM *instantly*. There's no chance to animate it out — by the time you'd run the animation, the element is already gone. This is a fundamental clash between React's render model and animation.

`AnimatePresence` bridges the gap. The logic: it *remembers* its children between renders. When a child disappears from the JSX, `AnimatePresence` keeps the real DOM node alive, runs the child's `exit` animation, and only *then* removes it. It's the missing piece that makes `exit` props actually work.

### 7.2 Basic usage and why `key` is mandatory [I]

```tsx
"use client"
import { useState } from "react"
import { motion, AnimatePresence } from "motion/react"

export function ToggleMessage() {
  const [show, setShow] = useState(true)

  return (
    <>
      <button onClick={() => setShow(!show)}>Toggle</button>

      {/* AnimatePresence wraps the CONDITIONALLY RENDERED element */}
      <AnimatePresence>
        {show && (
          <motion.div
            key="message"                            // REQUIRED — see below
            initial={{ opacity: 0, height: 0 }}
            animate={{ opacity: 1, height: "auto" }} // height:"auto" works inside motion!
            exit={{ opacity: 0, height: 0 }}         // runs when `show` becomes false
            transition={{ duration: 0.3 }}
          >
            I animate in and out!
          </motion.div>
        )}
      </AnimatePresence>
    </>
  )
}
```

**Why `key` is required:** `AnimatePresence` tracks children by their `key` to know *which* element entered, stayed, or left between renders. Without a stable, unique key, it can't tell that "the old element is gone and should exit" versus "this is the same element." A missing key is the #1 reason `exit` silently fails. For a single toggled element, any constant string works (`key="message"`); for lists, use the item's stable id.

> **Note:** `height: "auto"` (animating to the natural content height) works inside Motion's `animate`/`exit` even though it's impossible in plain CSS transitions. Motion measures the target height and interpolates to it.

### 7.3 The `mode` prop [I]

`mode` controls how entering and exiting elements coexist when a *new* element replaces an old one (e.g. changing pages or tabs):

| `mode` | Behaviour / when to use |
|---|---|
| `"sync"` (default) | The exiting and entering elements animate **at the same time**, overlapping. Fine when they don't occupy the same space. |
| `"wait"` | **Wait** for the exiting element to fully finish before the new one enters. The cleanest choice for page/tab transitions — no overlap, no layout fighting. |
| `"popLayout"` | The exiting element is "popped" out of layout flow (positioned absolutely) during its exit, so surrounding elements immediately reflow into place. Use when removing an item from a list and you want the gap to close *while* the item animates out. |

```tsx
// Page/tab transition: old fully leaves, then new enters
<AnimatePresence mode="wait">
  <motion.div
    key={currentPage}                 // changing the key triggers an exit→enter cycle
    initial={{ opacity: 0, x: 20 }}
    animate={{ opacity: 1, x: 0 }}
    exit={{ opacity: 0, x: -20 }}
    transition={{ duration: 0.25 }}
  >
    {pageContent}
  </motion.div>
</AnimatePresence>
```

> **Why changing `key` causes a transition:** when `currentPage` changes, React sees a child with a *new* key and treats the old one as removed and the new one as added. `AnimatePresence` runs the old key's `exit`, then mounts the new key with its `initial`→`animate`. Changing the key is the idiomatic way to trigger a swap animation.

### 7.4 Animating list add/remove [I]

```tsx
"use client"
import { useState } from "react"
import { motion, AnimatePresence } from "motion/react"

let nextId = 4

export function AnimatedList() {
  const [items, setItems] = useState([
    { id: 1, text: "Item 1" }, { id: 2, text: "Item 2" }, { id: 3, text: "Item 3" },
  ])

  const addItem = () => setItems([...items, { id: nextId++, text: `Item ${nextId - 1}` }])
  const removeItem = (id: number) => setItems(items.filter((i) => i.id !== id))

  return (
    <>
      <button onClick={addItem}>Add Item</button>
      <ul className="space-y-2 mt-4">
        <AnimatePresence>
          {items.map((item) => (
            <motion.li
              key={item.id}                          // STABLE, UNIQUE key (the id, not the index!)
              initial={{ opacity: 0, x: -20, height: 0 }}
              animate={{ opacity: 1, x: 0, height: "auto" }}
              exit={{ opacity: 0, x: 20, height: 0 }} // removed items slide out + collapse
              transition={{ duration: 0.25 }}
              className="flex items-center justify-between p-3 bg-white rounded shadow"
            >
              {item.text}
              <button onClick={() => removeItem(item.id)}>Remove</button>
            </motion.li>
          ))}
        </AnimatePresence>
      </ul>
    </>
  )
}
```

> **Gotcha — never use the array index as the key in an AnimatePresence list.** Indices shift when items are added/removed, so Motion mis-identifies which item left and animates the wrong one out. Always key by a stable id. Consider `mode="popLayout"` here so remaining items slide up to fill the gap while the removed item animates away.

### 7.5 Modal with backdrop [I]

A canonical use: an overlay + panel that both animate in and out together.

```tsx
"use client"
import { useState } from "react"
import { motion, AnimatePresence } from "motion/react"

export function Modal() {
  const [isOpen, setIsOpen] = useState(false)

  return (
    <>
      <button onClick={() => setIsOpen(true)}>Open Modal</button>

      <AnimatePresence>
        {isOpen && (
          <motion.div
            key="backdrop"
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            onClick={() => setIsOpen(false)}      // click backdrop to close
            className="fixed inset-0 bg-black/50 flex items-center justify-center z-50"
          >
            <motion.div
              key="modal"
              initial={{ opacity: 0, scale: 0.9, y: 20 }}
              animate={{ opacity: 1, scale: 1, y: 0 }}
              exit={{ opacity: 0, scale: 0.9, y: 20 }}
              transition={{ type: "spring", stiffness: 300, damping: 25 }}
              onClick={(e) => e.stopPropagation()} // clicks inside the panel don't close it
              className="bg-white rounded-2xl p-8 max-w-md w-full shadow-2xl"
            >
              <h2 className="text-xl font-bold">Modal Title</h2>
              <p className="mt-2 text-gray-600">Modal content here.</p>
              <button onClick={() => setIsOpen(false)}
                className="mt-4 px-4 py-2 bg-blue-500 text-white rounded-lg">Close</button>
            </motion.div>
          </motion.div>
        )}
      </AnimatePresence>
    </>
  )
}
```

> **Critical gotcha — keep `AnimatePresence` mounted.** `AnimatePresence` must *itself* stay rendered across the toggle; only its *children* should be conditional. If you write `{isOpen && <AnimatePresence>...</AnimatePresence>}`, the whole presence component unmounts before it can run the exit, and the exit silently never plays. Put the condition *inside*, never *around*, `AnimatePresence`.

---

## 8. Layout Animations & Shared Elements

Layout animations are Motion's most "magical" feature and worth understanding deeply, because they let you animate things CSS fundamentally cannot: an element moving to a new position because *other* elements changed, an element growing from `height: 0` to its natural content height, or an element appearing to *fly* across the screen from one component to another.

### 8.1 The FLIP technique — how layout animation works [A]

**The logic / why it's even possible:** you can't smoothly animate `height: auto` or "moved because a sibling was removed" with transitions, because the browser jumps instantly to the new layout. Motion uses a technique called **FLIP** (First, Last, Invert, Play):

1. **First** — measure the element's current position/size *before* the change.
2. **Last** — let React apply the change; measure the *new* position/size.
3. **Invert** — apply a `transform` that visually places the element *back* where it started (so to the eye, nothing moved yet).
4. **Play** — animate that transform away to zero, so the element glides from old to new.

The brilliance: the actual layout changes instantly (so the rest of the page is correct), but the animated element is *visually* corrected with a `transform` — the cheap, GPU-composited property from §1.1. That's why layout animations are smooth even when animating "expensive" things like position-in-a-flex-container or `height: auto`. You opt in with a single prop: `layout`.

### 8.2 The `layout` prop [I/A]

Add `layout` and Motion animates the element whenever its size or position changes *for any reason* — a state toggle, a sibling appearing, a CSS class change, a flex/grid reflow.

```tsx
"use client"
import { useState } from "react"
import { motion } from "motion/react"

export function ExpandingCard() {
  const [expanded, setExpanded] = useState(false)

  return (
    <motion.div
      layout                                 // animate ANY layout change automatically (FLIP)
      onClick={() => setExpanded(!expanded)}
      className="bg-blue-500 text-white rounded-xl p-6 cursor-pointer w-64"
    >
      <motion.h2 layout="position">Card Title</motion.h2>{/* only re-position, don't stretch */}
      {expanded && (
        <motion.p initial={{ opacity: 0 }} animate={{ opacity: 1 }} className="mt-2">
          Expanded content that changes the card's height. Motion animates the height change
          smoothly — you never had to know the pixel height in advance.
        </motion.p>
      )}
    </motion.div>
  )
}
```

**`layout` values:**

| Value | What it animates | When to use |
|---|---|---|
| `layout` / `layout={true}` | Both size and position | The default; general layout changes |
| `layout="position"` | Position only (not size) | Text/children that should *move* but not *stretch* (stretching distorts text) |
| `layout="size"` | Size only (not position) | Containers that resize in place |
| `layout="preserve-aspect"` | Maintains aspect ratio during resize | Images/media that shouldn't squash |

> **Gotcha — distorted children.** When a parent animates `layout` (size), Motion scales it, which can squash text and child content mid-animation. Fix by adding `layout` (or `layout="position"`) to the *children* too, so Motion counter-scales them. This is why the `<h2>` above uses `layout="position"`.

### 8.3 Shared-element transitions with `layoutId` (magic move) [A]

`layoutId` is the showstopper. Give two *different* elements the **same** `layoutId`, and when one unmounts while the other mounts — *even in a completely different part of the tree* — Motion animates between them as if they were one element that flew across the screen and resized. This is the "magic move" you see in image galleries that expand a thumbnail to fullscreen, or a tab indicator that slides between tabs.

**The logic:** Motion treats matching `layoutId`s as the same logical element across mount/unmount. It measures where the old one was, where the new one is, and runs a FLIP animation between them. You get a polished shared-element transition for free.

```tsx
"use client"
import { useState } from "react"
import { motion } from "motion/react"

const items = ["Card 1", "Card 2", "Card 3"]

export function MagicMoveGallery() {
  const [selected, setSelected] = useState<string | null>(null)

  return (
    <>
      {/* Grid of thumbnails */}
      <div className="grid grid-cols-3 gap-4">
        {items.map((item) => (
          <motion.div
            key={item}
            layoutId={item}                    // SAME id as the expanded version below
            onClick={() => setSelected(item)}
            className="h-32 bg-blue-500 rounded-xl cursor-pointer"
          />
        ))}
      </div>

      {/* Full-screen expanded view — different element, SAME layoutId */}
      {selected && (
        <motion.div
          layoutId={selected}                  // Motion flies the thumbnail to this position/size
          onClick={() => setSelected(null)}
          className="fixed inset-10 bg-blue-500 rounded-3xl cursor-pointer z-50"
        />
      )}
    </>
  )
}
```

> **Best practice:** wrap the expanded element in `<AnimatePresence>` so closing it animates back to the thumbnail rather than vanishing. And ensure `layoutId` values are *globally unique* (or unique within a `LayoutGroup`) — duplicates make Motion connect the wrong elements.

### 8.4 `LayoutGroup` — coordinating separate components [A]

By default, layout animations only react to changes within a single component subtree. When you have *sibling* components that should share layout awareness — most classically a row of tabs where the active indicator must slide from one tab component to another — wrap them in `LayoutGroup`. It namespaces and links `layoutId`s across the children so the shared-element transition works between separate components.

```tsx
import { LayoutGroup, motion } from "motion/react"

export function TabInterface() {
  return (
    // Without LayoutGroup, the underline can't animate ACROSS separate Tab components:
    <LayoutGroup>
      <Tab id="tab1" /> <Tab id="tab2" /> <Tab id="tab3" />
    </LayoutGroup>
  )
}

function Tab({ id, isActive }: { id: string; isActive: boolean }) {
  return (
    <div className="relative">
      <span>{id}</span>
      {isActive && (
        <motion.div
          layoutId="active-underline"        // same id in every Tab → the underline slides
          className="absolute bottom-0 left-0 right-0 h-0.5 bg-blue-500"
        />
      )}
    </div>
  )
}
```

> **Performance gotcha:** layout animations require measuring the DOM, which is more expensive than transform animations. Don't slap `layout` on hundreds of list items — for big lists, prefer transform-based variants (animating `y`/`opacity`). Reserve `layout` and `layoutId` for the handful of elements that genuinely need position/size or shared-element animation.

---

## 9. Scroll Animations

Scroll animations come in two flavors that are easy to confuse, and picking the right one is the key decision. **Scroll-*triggered*** animations fire *once* when an element reaches a threshold, then run on their own clock (covered as `whileInView` in §5.2). **Scroll-*linked*** animations bind a value *continuously* to scroll position, so the animation scrubs forward and backward as you scroll — this is how you build parallax, progress bars, and "pin and animate" effects. This section focuses on scroll-linked animation via `useScroll`.

### 9.1 `useScroll` — tracking scroll as a motion value [I/A]

`useScroll` returns motion values that represent scroll position and progress. The most useful is `scrollYProgress`, a value from `0` (top) to `1` (bottom). Because it's a *motion value*, you feed it straight into `style` (or transform it first) and the element updates every frame — no scroll listeners, no React re-renders.

```tsx
"use client"
import { useScroll, motion } from "motion/react"

export function ScrollProgressBar() {
  // With no options, tracks the whole page's vertical scroll:
  const { scrollYProgress } = useScroll()   // a MotionValue: 0 at top → 1 at bottom

  return (
    // scaleX is driven directly by progress; origin-left makes it grow from the left.
    // Animating scaleX (a transform) keeps this on the cheap compositor path.
    <motion.div
      style={{ scaleX: scrollYProgress }}
      className="fixed top-0 left-0 right-0 h-1 bg-blue-500 origin-left z-50"
    />
  )
}
```

`useScroll` returns four motion values: `scrollX`, `scrollY` (absolute pixels) and `scrollXProgress`, `scrollYProgress` (normalized `0`–`1`). Use the progress values for most effects; the pixel values when you need raw distance.

### 9.2 Element-relative tracking and `offset` — the parallax engine [A]

The real power is tracking scroll *relative to a specific element's journey through the viewport*. Pass a `target` ref and an `offset` describing the start and end of the tracked window. Then map that `0`–`1` progress to any visual property with `useTransform` (§10).

```tsx
"use client"
import { useRef } from "react"
import { useScroll, useTransform, motion } from "motion/react"

export function ParallaxSection() {
  const ref = useRef(null)

  const { scrollYProgress } = useScroll({
    target: ref,
    offset: ["start end", "end start"],
    // offset[0] "start end": progress = 0 when the element's START (top) meets the viewport's END (bottom)
    //                        → i.e. the moment the element first appears at the bottom of the screen.
    // offset[1] "end start": progress = 1 when the element's END (bottom) meets the viewport's START (top)
    //                        → i.e. the moment the element fully leaves at the top.
  })

  // Map the 0→1 scroll progress to a vertical offset → parallax (background drifts slower):
  const y = useTransform(scrollYProgress, [0, 1], ["-20%", "20%"])
  // Fade in then out across the journey:
  const opacity = useTransform(scrollYProgress, [0, 0.3, 0.7, 1], [0, 1, 1, 0])

  return (
    <div ref={ref} className="relative h-screen overflow-hidden">
      <motion.div style={{ y, opacity }}
        className="absolute inset-0 bg-gradient-to-b from-blue-500 to-purple-600" />
      <div className="relative z-10 flex items-center justify-center h-full text-white text-4xl font-bold">
        Parallax Section
      </div>
    </div>
  )
}
```

**`offset` reference** — each entry is `"<element edge> <viewport edge>"`:

| Offset string | The progress edge is reached when… |
|---|---|
| `"start start"` | the element's top meets the viewport's top |
| `"start end"` | the element's top meets the viewport's bottom (element just enters from below) |
| `"end start"` | the element's bottom meets the viewport's top (element just exits at top) |
| `"end end"` | the element's bottom meets the viewport's bottom |
| `"center center"` | the element's center meets the viewport's center |

You can also use pixel (`"100px"`) or percentage (`"20%"`) values for fine control.

### 9.3 Scroll-linked rotate / scale [I]

Any transform can be scroll-linked. Map absolute `scrollY` (pixels) when you want effects tied to a fixed scroll distance rather than an element's journey.

```tsx
"use client"
import { useScroll, useTransform, motion } from "motion/react"

export function RotatingLogo() {
  const { scrollY } = useScroll()                          // absolute pixels scrolled
  const rotate = useTransform(scrollY, [0, 500], [0, 360]) // 0→500px scroll → 0°→360°
  const scale = useTransform(scrollY, [0, 300], [1, 0.5])  // 0→300px scroll → full→half size

  return (
    <motion.div style={{ rotate, scale }}
      className="w-24 h-24 bg-blue-500 rounded-full fixed top-4 right-4" />
  )
}
```

### 9.4 `useInView` — the imperative trigger [I]

`whileInView` is declarative; sometimes you need the *boolean* "is it visible right now?" to drive logic (start a counter, lazy-load, fire analytics). `useInView` gives you exactly that.

```tsx
"use client"
import { useRef } from "react"
import { useInView, motion } from "motion/react"

export function VisibilityFlag() {
  const ref = useRef(null)
  const isInView = useInView(ref, {
    once: true,                       // stop tracking after first entry
    amount: 0.5,                      // 50% must be visible
    margin: "0px 0px -100px 0px",     // shrink the detection zone (fire a bit later)
  })

  return (
    <div ref={ref}>
      <motion.span animate={{ opacity: isInView ? 1 : 0 }} transition={{ duration: 0.5 }}>
        {isInView ? "Visible!" : "Not yet"}
      </motion.span>
    </div>
  )
}
```

> **Best practice — smooth scroll-linked values.** Raw `scrollYProgress` updates exactly with the wheel, which can feel twitchy. Wrap it in `useSpring` (§10.3) for a smooth, slightly-lagging follow: `const smooth = useSpring(scrollYProgress, { stiffness: 100, damping: 30 })`. This is the trick behind silky progress bars.

> **Performance gotcha:** scroll-linked animations run every frame while scrolling, so keep them to `transform`/`opacity` only. Animating `width`, `height`, or `box-shadow` on scroll will thrash layout/paint and stutter. Also avoid running many heavy `useTransform` chains on the same scroll value — consolidate where possible.

---

## 10. Motion Values & Hooks

Motion values are the engine under everything in this guide. Understanding them takes you from "I can use the components" to "I can build any custom motion behaviour."

### 10.1 What a motion value is and why it bypasses React [A]

**The logic / the core insight:** a `MotionValue` is a reactive container for a single animating value (a number, string, or color) that lives *entirely outside React's render cycle*. When you call `value.set(100)`, Motion writes the change straight to the DOM via its own render loop — **React does not re-render.** This is deliberate and is *the* reason Motion can drive 60fps animations (drag, scroll, mouse-follow) without grinding your component tree. A motion value is the bridge between continuous JavaScript-driven values and the GPU-composited DOM, skipping React entirely for performance.

You rarely create the bottom-level ones by hand for `animate` (Motion makes them internally). You create them explicitly when *you* need to own and feed a continuous value — drag position, scroll, mouse coordinates.

### 10.2 `useMotionValue` and `useTransform` [A]

`useMotionValue(initial)` creates one. `useTransform` *derives* a new motion value from one or more existing ones — the heart of mapping inputs (drag/scroll) to outputs (color, rotation, opacity).

```tsx
"use client"
import { useMotionValue, useTransform, motion } from "motion/react"

export function ColorDrag() {
  const x = useMotionValue(0)             // owns the horizontal drag position; no re-renders

  // Map x position to a background color across three stops (input range → output range):
  const backgroundColor = useTransform(
    x,
    [-200, 0, 200],                       // input: x positions
    ["#ef4444", "#3b82f6", "#10b981"]     // output: red → blue → green (auto-interpolated)
  )
  const rotate = useTransform(x, [-200, 200], [-30, 30])                 // tilt with drag
  const opacity = useTransform(x, [-200, -100, 0, 100, 200], [0.2, 1, 1, 1, 0.2]) // fade at edges

  return (
    <motion.div
      drag="x"
      dragConstraints={{ left: -200, right: 200 }}
      style={{ x, backgroundColor, rotate, opacity }}  // all driven by the single x value
      className="w-20 h-20 rounded-lg cursor-grab"
    />
  )
}
```

`useTransform` also accepts a **function** for arbitrary math, and can combine **multiple** inputs:

```tsx
// Custom formula on one value:
const scale = useTransform(x, (latest) => 1 + Math.abs(latest) / 400)

// Combine two values into one derived value (note the array input + array destructure):
const blur = useTransform(
  [x, y],
  ([lx, ly]) => `blur(${(Math.abs(lx) + Math.abs(ly)) / 20}px)`
)
```

> **Why not just use `useState` + `useTransform`-style math?** Because every drag/scroll frame would call `setState`, re-rendering the component (and its children) dozens of times per second. Motion values mutate the DOM directly, so the work per frame is tiny and constant regardless of component complexity.

### 10.3 `useSpring` — smoothing any motion value [A]

`useSpring` takes a motion value (or a number) and returns a *new* motion value that follows it through spring physics — lagging slightly and settling smoothly. This is how you turn a jerky input (mouse position, raw scroll) into buttery motion.

```tsx
"use client"
import { useMotionValue, useSpring, motion } from "motion/react"

export function SpringCursor() {
  const mouseX = useMotionValue(0)
  const mouseY = useMotionValue(0)

  // The springs trail the mouse with physics — the follower glides instead of snapping:
  const springX = useSpring(mouseX, { stiffness: 150, damping: 15 })
  const springY = useSpring(mouseY, { stiffness: 150, damping: 15 })

  const handleMouseMove = (e: React.MouseEvent) => {
    mouseX.set(e.clientX)               // feed raw position in; the spring smooths it
    mouseY.set(e.clientY)
  }

  return (
    <div onMouseMove={handleMouseMove} className="fixed inset-0">
      <motion.div
        style={{ x: springX, y: springY }}
        className="absolute w-8 h-8 bg-blue-500 rounded-full pointer-events-none -translate-x-1/2 -translate-y-1/2"
      />
    </div>
  )
}
```

### 10.4 `useMotionTemplate` — composing strings from motion values [A]

When you need a *string* CSS value (like `filter`, `boxShadow`, or `background`) built from several motion values, `useMotionTemplate` is a tagged template that produces a live motion-value string.

```tsx
import { useMotionTemplate, useMotionValue } from "motion/react"

const blurPx = useMotionValue(0)
const brightness = useMotionValue(1)
// A live string that updates whenever either value changes:
const filter = useMotionTemplate`blur(${blurPx}px) brightness(${brightness})`
// <motion.div style={{ filter }} />
```

### 10.5 `useMotionValueEvent` — bridging back to React state [A]

Because motion values bypass React, you sometimes need to *read* one in render (to display a number, or to trigger logic at a threshold). `useMotionValueEvent` subscribes to changes without causing an infinite render loop.

```tsx
"use client"
import { useState } from "react"
import { useMotionValue, useMotionValueEvent, motion } from "motion/react"

export function PositionDisplay() {
  const x = useMotionValue(0)
  const [xPos, setXPos] = useState(0)

  // Subscribe to the motion value; update React state only when you actually need to render it:
  useMotionValueEvent(x, "change", (latest) => setXPos(Math.round(latest)))

  return (
    <>
      <p>X position: {xPos}px</p>
      <motion.div drag="x" dragConstraints={{ left: -200, right: 200 }} style={{ x }}
        className="w-20 h-20 bg-blue-500 rounded-lg cursor-grab" />
    </>
  )
}
```

> **Best practice:** only bridge to state when you truly must render the value. If you just need the value to *drive another visual property*, keep it in motion-value land with `useTransform` — calling `setState` every frame defeats the whole performance benefit.

### 10.6 `useAnimate` — imperative animation & sequences [A]

The declarative props cover most needs, but sometimes you need *imperative* control: animate on a button click, run a multi-step sequence, or animate based on logic that's awkward to express declaratively. `useAnimate` returns a `scope` ref and an `animate` function. You can target the scoped element or any selector inside it, and `await` each step to sequence.

```tsx
"use client"
import { useAnimate } from "motion/react"

export function SequenceAnimation() {
  const [scope, animate] = useAnimate() // scope = ref for the root; animate = imperative fn

  const runSequence = async () => {
    // Each call returns a Promise; awaiting them runs the steps in order:
    await animate("h1", { opacity: [0, 1], y: [20, 0] }, { duration: 0.4 })
    await animate("p", { opacity: [0, 1] }, { duration: 0.3 })
    await animate(".cta", { scale: [0.9, 1] }, { type: "spring" })
  }

  return (
    <div ref={scope}>
      <h1 style={{ opacity: 0 }}>Title</h1>
      <p style={{ opacity: 0 }}>Description</p>
      <button className="cta" onClick={runSequence}>Run Sequence</button>
    </div>
  )
}
```

> **When to use `useAnimate` vs declarative props:** prefer declarative `animate`/variants by default — it's simpler and more "React-y." Reach for `useAnimate` when the animation is a discrete *action* (run this sequence now), when you need to `await` and chain steps with logic between them, or when you're animating something not easily tied to render output.

---

## 11. Keyframes, Colors & SVG Path Animation

### 11.1 Keyframe arrays — multi-step animations [I]

Instead of a single target value, pass an **array** to animate *through* several values in sequence. Motion interpolates between consecutive entries. This is how you do bounces, wiggles, color cycles, and any multi-step motion without chaining transitions.

```tsx
<motion.div
  animate={{
    x: [0, 100, -50, 80, 0],                            // bounce around and return
    backgroundColor: ["#3b82f6", "#ef4444", "#10b981"], // cycle through colors
    borderRadius: ["10%", "50%", "10%"],                // morph shape and back
  }}
  transition={{
    duration: 2,
    ease: "easeInOut",
    times: [0, 0.2, 0.5, 0.8, 1],  // WHERE each keyframe lands on the 0→1 timeline (per-array)
    // Omit `times` and keyframes are spaced evenly.
    repeat: Infinity,
    repeatType: "reverse",
  }}
  className="w-20 h-20"
/>
```

**`times`** maps each keyframe to a point in the timeline (`0`–`1`). Its length must match the keyframe array's length. Use it to make some segments quick and others slow.

### 11.2 The `null` keyframe — "start from wherever you are" [I]

A `null` as the first keyframe tells Motion to use the element's *current* value as the starting point. Invaluable when an animation might interrupt another and you want a seamless handoff rather than a jump.

```tsx
<motion.div animate={{ x: [null, 100, 0] }} />
// "From current x, go to 100, then back to 0" — no snap to a hardcoded start.
```

### 11.3 Color animation [I]

Motion interpolates between any CSS color formats — hex, rgb(a), hsl(a) — even mixing formats across a keyframe array. It handles the channel math for you.

```tsx
<motion.div
  animate={{
    backgroundColor: "#3b82f6",                          // hex
    color: "rgb(255, 255, 255)",                         // rgb
    borderColor: "hsl(220, 80%, 60%)",                   // hsl
    boxShadow: "0px 0px 20px rgba(59, 130, 246, 0.5)",   // color embedded in a shadow
  }}
/>
```

> **Gotcha:** you can't animate *to or from* `"transparent"` cleanly in all cases — prefer `rgba(r,g,b,0)` (same color, zero alpha) so Motion interpolates alpha rather than guessing the RGB of "transparent."

### 11.4 SVG path drawing — `pathLength` [I]

Motion can animate SVG strokes to create the "self-drawing line" effect by animating `pathLength`, `pathOffset`, and `pathSpacing`. The key convenience: `pathLength` is **normalized to 0→1** regardless of the path's real length, so you don't need to know the geometry. `0` = nothing drawn, `1` = fully drawn.

```tsx
"use client"
import { motion } from "motion/react"

export function DrawingCheckmark() {
  return (
    <svg viewBox="0 0 50 50" width="100" height="100">
      <motion.path
        d="M 10 30 L 22 42 L 40 12"   // the checkmark path
        fill="none" stroke="#10b981" strokeWidth={4} strokeLinecap="round"
        initial={{ pathLength: 0, opacity: 0 }}     // undrawn + invisible
        animate={{ pathLength: 1, opacity: 1 }}      // draw the stroke from start to end
        transition={{
          pathLength: { duration: 0.8, ease: "easeOut" }, // the draw itself
          opacity: { duration: 0.1 },                      // pop visible quickly at the start
        }}
      />
    </svg>
  )
}
```

### 11.5 Animated SVG spinner [I]

Combining `pathLength` with rotation makes a clean, dependency-free loading spinner.

```tsx
"use client"
import { motion } from "motion/react"

export function LoadingCircle() {
  return (
    <svg viewBox="0 0 50 50" width={48} height={48}>
      <motion.circle
        cx={25} cy={25} r={20} fill="none" stroke="#3b82f6" strokeWidth={4} strokeLinecap="round"
        initial={{ pathLength: 0, rotate: -90 }}
        animate={{ pathLength: 0.75, rotate: 270 }}   // 3/4 arc, spinning
        transition={{ duration: 1, repeat: Infinity, ease: "easeInOut", repeatType: "reverse" }}
        style={{ originX: "25px", originY: "25px" }}   // rotate around the circle's center
      />
    </svg>
  )
}
```

---

## 12. Motion with Next.js App Router

Next.js is the most common place people use Motion, and its Server Components model creates one rule you must internalize. This section assumes familiarity with the **Next.js** App Router (see that guide); here we focus on the Motion-specific concerns.

### 12.1 The fundamental rule: `"use client"` [I]

**The logic / why it's required:** Motion runs in the browser — it uses `requestAnimationFrame`, reads element geometry, attaches pointer listeners, and uses React state/refs/effects. In the Next.js App Router, **every component is a Server Component by default**, and Server Components cannot use hooks, state, or browser APIs. Therefore any file that imports from `motion/react` must declare `"use client"` at the very top, marking it (and its import subtree) as a Client Component that ships JS to the browser and hydrates there.

```tsx
// app/components/AnimatedHero.tsx
"use client" // ← REQUIRED. Without it you get a server-rendering error.

import { motion } from "motion/react"

export function AnimatedHero() {
  return (
    <motion.h1
      initial={{ opacity: 0, y: -20 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: 0.6 }}
      className="text-5xl font-bold"
    >
      Hello World
    </motion.h1>
  )
}
```

```tsx
// app/page.tsx — a Server Component (NO "use client"); can fetch data and still RENDER client components
import { AnimatedHero } from "@/components/AnimatedHero"

export default async function HomePage() {
  // const data = await db.query(...)  // server-side work is fine here
  return (
    <main>
      <AnimatedHero />                  {/* a Client Component used inside a Server Component */}
      <p>Server-rendered content here</p>
    </main>
  )
}
```

### 12.2 Best-practice pattern: thin client wrappers [I]

You want to keep as much as possible on the server (smaller JS bundles, server data fetching). The idiomatic solution is a small, reusable *client* wrapper that adds animation, while the actual content stays server-rendered and is passed in as `children`.

```tsx
// components/FadeInSection.tsx — a tiny client island that animates whatever you pass it
"use client"
import { motion } from "motion/react"

export function FadeInSection({
  children, delay = 0, className,
}: { children: React.ReactNode; delay?: number; className?: string }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 30 }}
      whileInView={{ opacity: 1, y: 0 }}
      viewport={{ once: true, amount: 0.2 }}
      transition={{ duration: 0.5, delay, ease: "easeOut" }}
      className={className}
    >
      {children}
    </motion.div>
  )
}
```

```tsx
// app/about/page.tsx — Server Component fetches data, wraps server content in client animators
import { FadeInSection } from "@/components/FadeInSection"

export default async function AboutPage() {
  const data = await fetchAboutData()   // runs on the server
  return (
    <main>
      <FadeInSection delay={0}><h1>{data.title}</h1></FadeInSection>
      <FadeInSection delay={0.1}><p>{data.description}</p></FadeInSection>
    </main>
  )
}
```

> **Why this matters:** `children` passed from a Server Component into a Client Component are *still rendered on the server*. So the heading and paragraph here are server-rendered HTML; only the thin animation wrapper is client JS. You get animation without dragging your whole page to the client.

### 12.3 Page transitions with `template.tsx` [I/A]

The App Router preserves `layout.tsx` across navigations (it doesn't re-mount), so an animation in a layout won't replay on each page change. Use **`template.tsx`** instead — Next.js gives each navigation a *fresh* `template` instance, so its enter animation runs on every page.

```tsx
// app/template.tsx — re-mounts on EVERY navigation (unlike layout.tsx)
"use client"
import { motion } from "motion/react"

export default function Template({ children }: { children: React.ReactNode }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 10 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: 0.25, ease: "easeInOut" }}
    >
      {children}
    </motion.div>
  )
}
```

> **⚡ Version note — exit animations between pages are hard.** Reliable *exit* transitions in the App Router are tricky: the new route mounts before you get a clean handle on the old one leaving, so a naive `AnimatePresence` + `exit` in `template.tsx` often won't fire as expected. For most projects, **enter-only** page transitions (as above) are the pragmatic, robust choice. Full exit transitions require coordinating `AnimatePresence` with the pathname and are an evolving ecosystem topic — verify the current recommended approach at motion.dev / the Next.js docs.

### 12.4 SSR, hydration, and layout animations [A]

Two SSR-specific gotchas:

- **`initial` flash:** the server can't run animations, so it renders the element and on hydration the client applies `initial` then animates — sometimes a brief flash. For above-the-fold content that should appear immediately, either set `initial` equal to the resting state (no mount animation) or use `whileInView` (which doesn't fire until scrolled to).
- **Layout animations need measurement:** the `layout` prop measures DOM geometry, which doesn't exist on the server. Use it in client-only components, and pass `initial={false}` so the first render doesn't try to animate from an unmeasured state.

```tsx
// Skip the mount animation; only animate SUBSEQUENT layout changes (avoids hydration mismatch):
<motion.div layout initial={false} />
```

### 12.5 A production-ready, accessible `<Reveal>` [I]

This wrapper bundles the best practices: client island, reduced-motion respect, configurable direction/delay.

```tsx
// components/Reveal.tsx
"use client"
import { motion, useReducedMotion } from "motion/react"

interface RevealProps {
  children: React.ReactNode
  direction?: "up" | "down" | "left" | "right"
  delay?: number
  duration?: number
  className?: string
  once?: boolean
}

export function Reveal({
  children, direction = "up", delay = 0, duration = 0.5, className, once = true,
}: RevealProps) {
  const prefersReducedMotion = useReducedMotion()

  const directionMap = {
    up: { y: 40 }, down: { y: -40 }, left: { x: 40 }, right: { x: -40 },
  }

  // Accessibility: if the user wants reduced motion, render plain content, no movement:
  if (prefersReducedMotion) return <div className={className}>{children}</div>

  return (
    <motion.div
      initial={{ opacity: 0, ...directionMap[direction] }}
      whileInView={{ opacity: 1, x: 0, y: 0 }}
      viewport={{ once, amount: 0.2 }}
      transition={{ duration, delay, ease: "easeOut" }}
      className={className}
    >
      {children}
    </motion.div>
  )
}
```

---

## 13. Accessibility & Reduced Motion

Motion can be a delight or a barrier. Some users experience nausea, dizziness, or migraines from large or parallax motion (vestibular disorders). Operating systems expose a **"reduce motion"** setting, surfaced to the web as the `prefers-reduced-motion` media query. Respecting it is not optional polish — it's a baseline accessibility obligation, and Motion makes it easy.

### 13.1 `useReducedMotion` [I]

This hook returns `true` when the user has requested reduced motion. Use it to *replace* large movement with subtle alternatives (usually a simple opacity fade, which conveys the same "this appeared" feedback without the disorienting travel).

```tsx
"use client"
import { useReducedMotion, motion } from "motion/react"

export function AccessibleAnimation() {
  const prefersReducedMotion = useReducedMotion()

  return (
    <motion.div
      // Drop the vertical travel when reduced motion is requested; keep the fade:
      initial={{ opacity: 0, y: prefersReducedMotion ? 0 : 40 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: prefersReducedMotion ? 0.1 : 0.6 }}
    >
      Accessible animated content
    </motion.div>
  )
}
```

> **Best practice:** don't simply *disable* animation for reduced-motion users — that can remove useful feedback. Instead, *reduce* it: keep opacity/color transitions (calm, informative) and remove or shrink large positional/scale/parallax movement (the disorienting part).

### 13.2 `MotionConfig reducedMotion` — the global switch [I]

The single easiest accessibility win is wrapping your app in `MotionConfig` with `reducedMotion="user"`. It makes *every* Motion component in the subtree automatically drop transform/layout movement when the OS setting is on — while keeping opacity transitions. You also use `MotionConfig` to set default transitions for the whole tree.

```tsx
// app/layout.tsx (or a top-level client provider)
import { MotionConfig } from "motion/react"

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <MotionConfig
          reducedMotion="user"          // "user" = follow the OS setting automatically
          // "always" = always reduce (great for QA/testing); "never" = ignore the OS setting
          transition={{ duration: 0.3 }} // a default transition for every child Motion element
        >
          {children}
        </MotionConfig>
      </body>
    </html>
  )
}
```

| `reducedMotion` value | Effect |
|---|---|
| `"user"` | Automatically reduces motion when the OS `prefers-reduced-motion` is on. **Use this.** |
| `"always"` | Always reduces, regardless of OS. Useful for testing your reduced-motion UI. |
| `"never"` | Ignores the OS setting entirely (don't use without good reason). |

### 13.3 Accessible patterns checklist [I]

```tsx
// ✅ Convey state with opacity/color (and only SUBTLE scale), not only by position:
<motion.button animate={{ backgroundColor: isActive ? "#3b82f6" : "#e5e7eb", scale: isActive ? 1.02 : 1 }} />

// ✅ Announce content that appears after animation to screen readers:
<motion.div animate={{ opacity: 1 }} aria-live="polite">{message}</motion.div>
```

> **Best practices beyond reduced motion:** never gate essential information behind an animation a screen-reader user can't perceive; keep `whileFocus` feedback equal to `whileHover` so keyboard users aren't second-class; avoid `repeat: Infinity` motion that can't be paused near important content; and ensure interactive `motion.div`s that act as buttons have `role`, `tabIndex`, and keyboard handlers (or just use `motion.button`).

---

## 14. Performance & Bundle Size

Motion is fast *if you let it be*. Almost all jank traces back to animating the wrong properties or doing too much per frame. This section is the performance contract.

### 14.1 What's cheap to animate (and what isn't) [A]

Recall the rendering pipeline from §1.1. Properties differ enormously in cost:

| Property | Pipeline stage | Cost | Verdict |
|---|---|---|---|
| `transform` (`x`, `y`, `scale`, `rotate`) | Composite | Cheap (GPU) | ✅ Always prefer |
| `opacity` | Composite | Cheap (GPU) | ✅ Always prefer |
| `filter` (`blur`, `brightness`) | Composite-ish | Moderate | ⚠️ OK in moderation |
| `backgroundColor`, `color` | Paint | Minor | ⚠️ Fine occasionally |
| `boxShadow` | Paint | Expensive to paint | ⚠️ Avoid on scroll/loops |
| `width`, `height`, `padding`, `margin`, `top`, `left` | **Layout** | Expensive (reflow) | ❌ Avoid; use transform or `layout` |
| `clip-path` | Partial | Varies | ⚠️ Simple clips only |

**Rule of thumb:** animate `transform` and `opacity` for anything that runs continuously (scroll, drag, loops). When you genuinely must animate size/position-in-flow (e.g. `height: auto`, list reordering), use the `layout` prop — Motion's FLIP technique converts those into GPU transforms internally (§8.1).

### 14.2 `will-change` — promote, but sparingly [A]

`will-change: transform` hints the browser to pre-promote an element to its own GPU layer, avoiding a hitch when the animation starts. But each promoted layer costs GPU memory, so over-using it *hurts* performance.

```tsx
<motion.div style={{ willChange: "transform" }} animate={{ x: 100 }} />
```

> **Best practice:** only set `will-change` on elements that animate *frequently or continuously* — a cursor follower, a persistent scroll-linked element. Don't put it on every animated element, and don't leave it on after the animation is done. (Motion already manages layer promotion well for most cases; reach for manual `will-change` only when profiling shows a startup hitch.)

### 14.3 Reducing layout thrash [A]

```tsx
// ❌ Slow: layout animations on hundreds of rows force measurement of each, every frame
{items.map((item) => <motion.div layout key={item.id}>{item.text}</motion.div>)}

// ✅ Fast: transform-only variants for the bulk; reserve `layout` for the few that truly move
const item = { hidden: { opacity: 0, y: 20 }, show: { opacity: 1, y: 0 } }
{items.map((item) => <motion.div key={item.id} variants={item}>{item.text}</motion.div>)}
```

Other thrash-reducers: prefer `mode="popLayout"` over many simultaneous `layout` animations; avoid animating into containers with `overflow: hidden` that force reflow; consolidate multiple `useTransform`s on the same scroll value.

### 14.4 `LazyMotion` — shrinking the bundle [A]

The convenient `motion` import bundles *all* features (~34 KB gzip) because it can't know which you'll use. For size-sensitive apps, `LazyMotion` loads only the feature set you choose, and you swap the `motion` component for the lightweight `m` component (which has no built-in features — it relies on whatever `LazyMotion` provides).

**The logic:** `motion.div` ships every feature so it "just works." `m.div` ships *nothing* on its own and asks the nearest `LazyMotion` for features. By picking `domAnimation` (animations, gestures, exit) or `domMax` (adds layout/drag/FLIP), you include only what you need.

```tsx
// A provider near the root (client component):
"use client"
import { LazyMotion, domAnimation } from "motion/react"

// domAnimation: animations + variants + gestures + AnimatePresence (most apps)
// domMax:       everything, including layout animations, layoutId, and drag
export function MotionProvider({ children }: { children: React.ReactNode }) {
  return (
    <LazyMotion features={domAnimation} strict /* `strict` errors if you accidentally use motion.* */>
      {children}
    </LazyMotion>
  )
}
```

```tsx
// In components, import the lightweight `m` instead of `motion`:
"use client"
import { m } from "motion/react"

export function LightweightAnimation() {
  return <m.div initial={{ opacity: 0 }} animate={{ opacity: 1 }} />
}
```

**Bundle size reference (approximate, gzip):**

| Import strategy | Size |
|---|---|
| `motion` (full, all features) | ~34 KB |
| `m` + `LazyMotion` + `domAnimation` | ~18 KB |
| `m` + `LazyMotion` + `domMax` | ~25 KB |
| Vanilla `import { animate } from "motion"` | ~5 KB |

### 14.5 Async-loading features (code splitting) [A]

You can even keep features out of the *initial* bundle and load them lazily — they arrive after first paint.

```tsx
import { LazyMotion } from "motion/react"

// Dynamically import the feature bundle so it's split out of the main chunk:
const loadFeatures = () => import("motion/react").then((res) => res.domAnimation)

export function MotionProvider({ children }: { children: React.ReactNode }) {
  return <LazyMotion features={loadFeatures}>{children}</LazyMotion>
}
```

> **Performance checklist:** (1) animate transform/opacity, (2) use `layout` only where needed, (3) keep scroll/drag work in motion values (no per-frame `setState`), (4) add `will-change` only to hot elements, (5) consider `LazyMotion` + `m` for bundle size, (6) test on a mid-range phone — desktop hides a lot of jank.

---

## 15. Practical Recipes

Production-ready building blocks. Each is a client component; drop them into a Next.js app behind the `"use client"` boundary.

### 15.1 Fade-up section reveal [I]

```tsx
// components/FadeUp.tsx
"use client"
import { motion, useReducedMotion } from "motion/react"

export function FadeUp({ children, delay = 0 }: { children: React.ReactNode; delay?: number }) {
  const reduced = useReducedMotion()
  return (
    <motion.div
      initial={{ opacity: 0, y: reduced ? 0 : 50 }}     // skip travel for reduced-motion users
      whileInView={{ opacity: 1, y: 0 }}
      viewport={{ once: true, amount: 0.15 }}
      transition={{ duration: 0.6, delay, ease: [0.21, 0.47, 0.32, 0.98] }} // custom "soft" curve
    >
      {children}
    </motion.div>
  )
}
// Usage:
// <FadeUp delay={0}><h1>Title</h1></FadeUp>
// <FadeUp delay={0.1}><p>Subtitle</p></FadeUp>
// <FadeUp delay={0.2}><Button>CTA</Button></FadeUp>
```

### 15.2 Staggered card grid [I]

```tsx
// components/StaggerGrid.tsx
"use client"
import { motion } from "motion/react"

const container = {
  hidden: {},
  show: { transition: { staggerChildren: 0.08, delayChildren: 0.1 } },
}
const card = {
  hidden: { opacity: 0, y: 30, scale: 0.96 },
  show: { opacity: 1, y: 0, scale: 1, transition: { type: "spring", stiffness: 200, damping: 20 } },
}

interface StaggerGridProps {
  items: { id: string; title: string; description: string }[]
}

export function StaggerGrid({ items }: StaggerGridProps) {
  return (
    <motion.div
      variants={container}
      initial="hidden"
      whileInView="show"                 // the WHOLE grid orchestrates as it scrolls into view
      viewport={{ once: true, amount: 0.1 }}
      className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6"
    >
      {items.map((item) => (
        <motion.div
          key={item.id}
          variants={card}                // each card inherits "show" and reveals in sequence
          whileHover={{ y: -4, boxShadow: "0 20px 40px rgba(0,0,0,0.12)" }}
          className="p-6 bg-white rounded-2xl shadow-sm border border-gray-100"
        >
          <h3 className="font-semibold text-lg">{item.title}</h3>
          <p className="mt-2 text-gray-500">{item.description}</p>
        </motion.div>
      ))}
    </motion.div>
  )
}
```

### 15.3 Slide-in drawer [I]

```tsx
// components/Drawer.tsx
"use client"
import { motion, AnimatePresence } from "motion/react"

interface DrawerProps { isOpen: boolean; onClose: () => void; children: React.ReactNode }

export function Drawer({ isOpen, onClose, children }: DrawerProps) {
  return (
    <AnimatePresence>          {/* stays mounted; only the children are conditional */}
      {isOpen && (
        <>
          <motion.div          // backdrop
            key="backdrop"
            initial={{ opacity: 0 }} animate={{ opacity: 1 }} exit={{ opacity: 0 }}
            transition={{ duration: 0.2 }}
            onClick={onClose}
            className="fixed inset-0 bg-black/40 z-40"
          />
          <motion.aside        // panel slides in from the right
            key="drawer"
            initial={{ x: "100%" }} animate={{ x: 0 }} exit={{ x: "100%" }}
            transition={{ type: "spring", stiffness: 300, damping: 30 }}
            className="fixed top-0 right-0 bottom-0 w-80 bg-white shadow-2xl z-50 p-6 overflow-y-auto"
          >
            <button onClick={onClose} className="mb-4 text-gray-400 hover:text-gray-800">✕ Close</button>
            {children}
          </motion.aside>
        </>
      )}
    </AnimatePresence>
  )
}
```

### 15.4 Hover-lift card [I]

```tsx
// components/HoverCard.tsx
"use client"
import { motion } from "motion/react"

export function HoverCard({ children }: { children: React.ReactNode }) {
  return (
    <motion.div
      whileHover={{
        y: -8, scale: 1.02,
        boxShadow: "0px 20px 40px rgba(0,0,0,0.15)",
        transition: { type: "spring", stiffness: 400, damping: 20 },
      }}
      whileTap={{ scale: 0.98 }}
      className="rounded-2xl bg-white border border-gray-100 p-6 cursor-pointer"
    >
      {children}
    </motion.div>
  )
}
```

### 15.5 Smooth scroll progress bar [I]

```tsx
// components/ScrollProgressBar.tsx
"use client"
import { useScroll, useSpring, motion } from "motion/react"

export function ScrollProgressBar() {
  const { scrollYProgress } = useScroll()
  // Spring-smooth the raw progress so the bar glides instead of jumping with the wheel:
  const scaleX = useSpring(scrollYProgress, { stiffness: 100, damping: 30, restDelta: 0.001 })

  return (
    <motion.div
      style={{ scaleX }}
      className="fixed top-0 left-0 right-0 h-1 bg-gradient-to-r from-blue-500 to-purple-500 origin-left z-[9999]"
    />
  )
}
```

### 15.6 Animated counter (count-up on view) [I]

```tsx
// components/AnimatedCounter.tsx
"use client"
import { useEffect, useRef } from "react"
import { useInView, useMotionValue, useSpring } from "motion/react"

export function AnimatedCounter({ target }: { target: number }) {
  const ref = useRef<HTMLSpanElement>(null)
  const isInView = useInView(ref, { once: true })
  const motionValue = useMotionValue(0)
  const springValue = useSpring(motionValue, { duration: 2000, bounce: 0 }) // smooth ramp, no bounce

  useEffect(() => {
    if (isInView) motionValue.set(target)         // when visible, animate toward the target
  }, [isInView, motionValue, target])

  useEffect(() => {
    // Write the formatted number to the DOM each frame WITHOUT re-rendering React:
    return springValue.on("change", (latest) => {
      if (ref.current) ref.current.textContent = Math.round(latest).toLocaleString()
    })
  }, [springValue])

  return <span ref={ref}>0</span>
}
// Usage: <AnimatedCounter target={1234567} />
```

### 15.7 Animated tab indicator (layoutId magic move) [I]

```tsx
// components/AnimatedTabs.tsx
"use client"
import { useState } from "react"
import { motion } from "motion/react"

const tabs = ["Design", "Development", "Marketing"]

export function AnimatedTabs() {
  const [active, setActive] = useState(tabs[0])
  return (
    <div className="flex gap-1 bg-gray-100 p-1 rounded-xl w-fit">
      {tabs.map((tab) => (
        <button key={tab} onClick={() => setActive(tab)}
          className="relative px-4 py-2 text-sm font-medium rounded-lg z-10">
          {active === tab && (
            <motion.div
              layoutId="active-tab-pill"    // the pill flies from the old tab to the new one
              className="absolute inset-0 bg-white rounded-lg shadow"
              transition={{ type: "spring", stiffness: 500, damping: 35 }}
            />
          )}
          <span className="relative z-10">{tab}</span>
        </button>
      ))}
    </div>
  )
}
```

---

## 16. Tips, Tricks & Gotchas

A field guide to the mistakes that cost the most debugging time.

### 16.1 Forgetting `"use client"` (Next.js) [I]

Every file importing `motion`, `AnimatePresence`, or any Motion hook **must** start with `"use client"`. This is the #1 Motion error in Next.js.

```
Error: ... hooks can only be used in Client Components ...
```

Fix: add `"use client"` as the first line of the file.

### 16.2 `exit` not firing [I]

Two usual causes: a missing `key`, or `AnimatePresence` placed wrong.

```tsx
// ❌ key on a wrapper, not on the animated element — AnimatePresence can't track it
<AnimatePresence>
  <div key="wrapper">{show && <motion.div exit={{ opacity: 0 }}>Content</motion.div>}</div>
</AnimatePresence>

// ✅ AnimatePresence watches its DIRECT children; key + condition live there
<AnimatePresence>
  {show && <motion.div key="content" exit={{ opacity: 0 }}>Content</motion.div>}
</AnimatePresence>
```

Also: `exit` fires only on **unmount**. Hiding with CSS `display: none` keeps the element mounted, so `exit` never runs.

### 16.3 Unmounting `AnimatePresence` itself [I]

```tsx
// ❌ The whole AnimatePresence is removed before it can run the exit
{show && <AnimatePresence><motion.div exit={{ opacity: 0 }}>Content</motion.div></AnimatePresence>}

// ✅ AnimatePresence stays mounted; only the child is conditional
<AnimatePresence>
  {show && <motion.div key="content" exit={{ opacity: 0 }}>Content</motion.div>}
</AnimatePresence>
```

### 16.4 Don't animate `height: auto` without `layout` (or AnimatePresence) [I]

```tsx
// ❌ "auto" isn't a number Motion can interpolate from a static `animate` on a persistent element
<motion.div animate={{ height: "auto" }} />

// ✅ Inside AnimatePresence, height:"auto" works (Motion measures it):
<AnimatePresence>{open && <motion.div initial={{ height: 0 }} animate={{ height: "auto" }} exit={{ height: 0 }} />}</AnimatePresence>

// ✅ Or use the layout prop for content-driven size changes:
<motion.div layout>{open && <Details />}</motion.div>

// ✅ Or animate a known pixel value:
<motion.div animate={{ height: open ? 200 : 0 }} />
```

### 16.5 Array index as `key` in lists [I]

Keying AnimatePresence/Reorder items by array index breaks tracking when items are added/removed — Motion animates the wrong element out. Always use a stable id.

### 16.6 Layout animation jank [A]

Layout animations look wrong when:
- A parent has `overflow: hidden` (it clips the FLIP transform). Prefer `overflow: clip`, or animate without `layout`.
- Web fonts swap mid-animation and shift layout. Preload fonts or use `font-display`.
- The element is `position: absolute`/`fixed` — `layout` works best on in-flow elements.
- Nested `layout` elements distort each other — add `layout`/`layout="position"` to children to counter-scale, or use `layoutId` on the container instead.

```tsx
// Debug hook to see when a layout animation starts:
<motion.div layout onLayoutAnimationStart={() => console.log("layout start")} />
```

### 16.7 Reduced motion globally [I]

The single easiest a11y win — wrap your root:

```tsx
<MotionConfig reducedMotion="user">{children}</MotionConfig>
```

### 16.8 `initial={false}` to skip the mount animation [I]

```tsx
// Element should appear already in place; only animate on LATER changes:
<motion.div initial={false} animate={{ opacity: isVisible ? 1 : 0 }} />

// On AnimatePresence — disables the enter animation for the first batch of present children:
<AnimatePresence initial={false}>
  {items.map((i) => <motion.li key={i.id} exit={{ opacity: 0 }}>{i.text}</motion.li>)}
</AnimatePresence>
```

### 16.9 `framer-motion` vs `motion` imports [I]

```tsx
import { motion } from "framer-motion"   // legacy package — still works
import { motion } from "motion/react"    // current package — use this for new code
// Never install/import BOTH in one project; two copies fight over the DOM.
```

### 16.10 Motion values bypass React renders — by design [A]

```tsx
const x = useMotionValue(0)
x.set(100)            // updates the DOM directly — NO re-render (intentional, for performance)

// To read it in render, bridge with an event (don't read x.get() during render):
const [val, setVal] = useState(0)
useMotionValueEvent(x, "change", setVal)
```

### 16.11 Pointer/touch during drag [I]

```tsx
<motion.div drag style={{ touchAction: "none" }}>  {/* stop the page scrolling while dragging */}
  <button onClick={...}>Won't fire accidentally mid-drag</button>
</motion.div>
```

### 16.12 Unique `layoutId`s within a `LayoutGroup` [A]

```tsx
function Tab({ id, isActive }: { id: string; isActive: boolean }) {
  return (
    <div>
      <span>{id}</span>
      {isActive && <motion.div layoutId={`underline-${id}`} />} {/* prefix to keep ids unique */}
    </div>
  )
}
```

### 16.13 SSR + `initial` flash [A]

```tsx
// Option 1: match initial to the resting state → no animation on first paint
<motion.div initial={{ opacity: 1 }} animate={{ opacity: 1 }} />
// Option 2: use whileInView so it only fires after scroll (never on first paint)
<motion.div initial={{ opacity: 0 }} whileInView={{ opacity: 1 }} viewport={{ once: true }} />
// Option 3: a tiny delay lets hydration finish before animating
<motion.div initial={{ opacity: 0 }} animate={{ opacity: 1 }} transition={{ delay: 0.05 }} />
```

---

## 17. Study Path & Build-to-Learn Projects

Animation is learned by building. Follow this sequence; at each stage, *build the project* before moving on — reading alone won't make spring tuning or layout animation intuitive.

### Stage 1 — Core component & transitions (Days 1–2) [B]
1. Read §3–§4. Animate a `div` with `initial`/`animate`, then add `exit`.
2. Experiment with `transition`: tween vs spring. Tune `stiffness`/`damping`/`mass` until you can predict the feel.
3. Add `whileHover` + `whileTap` to buttons; build a card that lifts on hover.
4. **Build:** a landing-page hero — fade-up heading, subtitle, and a springy CTA button, each with a slightly increasing delay.

### Stage 2 — Variants & presence (Days 3–4) [I]
1. Read §6. Build a staggered list using parent→child propagation + `staggerChildren`.
2. Read §7. Build a modal that animates in and out with a backdrop.
3. Combine variants + `AnimatePresence` for a list where items add/remove smoothly (`mode="popLayout"`).
4. **Build:** a to-do app where adding and removing items animates cleanly, keyed by stable ids.

### Stage 3 — Scroll & layout (Days 5–6) [I/A]
1. Read §9. Add `whileInView` reveals to every section of a page; build a spring-smoothed scroll progress bar.
2. Read §8. Try `layout` on an accordion; then `layoutId` for a tab underline that slides.
3. Build a parallax hero with `useScroll` + `useTransform` + `offset`.
4. **Build:** a portfolio page with scroll-triggered reveals, a progress bar, an animated tab interface, and one parallax section.

### Stage 4 — Motion values & custom behaviour (Days 7–8) [A]
1. Read §10. Use `useMotionValue` + `useTransform` for a drag card that changes color, rotation, and opacity by position.
2. Use `useSpring` to build a smooth cursor follower.
3. Use `useAnimate` to sequence a multi-step reveal imperatively.
4. **Build:** an interactive draggable card that reacts (color/shadow/rotate) to its drag position, with momentum on release.

### Stage 5 — Next.js integration (Days 9–10) [I/A]
1. Read §12. Internalize the `"use client"` boundary; build thin client wrappers (`<FadeInSection>`, `<Reveal>`).
2. Add `app/template.tsx` for enter page transitions.
3. Read §13. Add `MotionConfig reducedMotion="user"` at the root and verify with the OS setting on.
4. Read §14. Switch a project to `LazyMotion` + `m` and measure the bundle drop.
5. **Build:** a full Next.js marketing site — page transitions, staggered hero, scroll-triggered sections, animated nav, all reduced-motion-aware.

### Stage 6 — Advanced patterns & polish (Days 11–14) [A]
1. Build a magic-move image gallery with `layoutId` + `AnimatePresence`.
2. Build a drag-to-reorder list with `Reorder.Group`/`Reorder.Item`.
3. Animate SVG paths — a self-drawing checkmark and a spinner.
4. Profile a heavy page; eliminate layout-triggering animations; add `will-change` only where it helps.
5. **Build:** a full product page — parallax hero, scroll-linked sticky-header opacity, staggered feature cards, a magic-move gallery, an animated testimonials carousel, and a slide-in cart drawer.

### Portfolio-worthy projects

| Project | Key Motion features exercised |
|---|---|
| Landing page | FadeUp reveals, stagger, `whileInView`, page transitions |
| Dashboard | `AnimatePresence` modals, `layoutId` tab indicators, loading skeletons |
| E-commerce PDP | shared-element image zoom (`layoutId`), cart drawer, hover cards, scroll parallax |
| Portfolio | SVG path drawing, magnetic buttons, cursor follower, page transitions |
| Kanban board | `drag` + `dragConstraints`, `Reorder`, `AnimatePresence` card removal |

### Reference cheatsheet

```tsx
// Animate on mount
<motion.div initial={{ opacity: 0 }} animate={{ opacity: 1 }} />

// Animate on scroll entry (triggered)
<motion.div initial={{ opacity: 0 }} whileInView={{ opacity: 1 }} viewport={{ once: true }} />

// Gestures
<motion.button whileHover={{ scale: 1.05 }} whileTap={{ scale: 0.95 }} />

// Exit on unmount
<AnimatePresence>{show && <motion.div key="x" exit={{ opacity: 0 }} />}</AnimatePresence>

// Stagger children (orchestration)
const container = { hidden: {}, show: { transition: { staggerChildren: 0.1 } } }
const item = { hidden: { y: 20, opacity: 0 }, show: { y: 0, opacity: 1 } }
<motion.ul variants={container} initial="hidden" animate="show">
  <motion.li variants={item} />
</motion.ul>

// Scroll-linked (continuous)
const { scrollYProgress } = useScroll()
<motion.div style={{ scaleX: scrollYProgress }} />

// Derive values
const x = useMotionValue(0)
const opacity = useTransform(x, [-100, 0, 100], [0, 1, 0])

// Imperative sequence
const [scope, animate] = useAnimate()
await animate(".heading", { opacity: 1 }); await animate(".body", { opacity: 1 })

// Layout / shared element
<motion.div layout />
<motion.div layoutId="hero" />

// Reduce motion globally
<MotionConfig reducedMotion="user">{children}</MotionConfig>

// Bundle size
<LazyMotion features={domAnimation}><m.div animate={{ x: 100 }} /></LazyMotion>
```

---

*This guide targets Motion (the `motion` package, formerly Framer Motion) with React 19 and Next.js 15/16, current in 2026. motion.dev is the canonical source for any API changes after this guide's cutoff.*
