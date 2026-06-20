# Motion (Framer Motion) Complete Animation Guide — React & Next.js

A full, study-friendly reference for **Motion** (formerly Framer Motion), the premier animation library for React. Covers every major API, Next.js integration, performance patterns, and practical recipes — enough to build production-quality animations with zero internet access.

> **Version note:** This guide targets **Motion v11+** (the `motion` package at motion.dev), running with **React 18/19** and **Next.js 15/16**. The library was rebranded from "Framer Motion" to "Motion" in late 2024; the old `framer-motion` package still works but the canonical package is now `motion` with imports from `motion/react`. Where behaviour differs by version it is flagged with **⚡ Version note**.

---

## Table of Contents

1. [What Motion Is & the Rebrand (Framer Motion → Motion)](#1-what-motion-is--the-rebrand)
2. [Setup & Imports](#2-setup--imports)
3. [The `motion` Component](#3-the-motion-component)
4. [Transitions](#4-transitions)
5. [Gestures](#5-gestures)
6. [Variants](#6-variants)
7. [AnimatePresence — Exit Animations & Mount/Unmount](#7-animatepresence)
8. [Layout Animations & Shared Element Transitions](#8-layout-animations)
9. [Scroll Animations](#9-scroll-animations)
10. [Motion Values & Hooks](#10-motion-values--hooks)
11. [Keyframes, Colors & SVG Path Animation](#11-keyframes-colors--svg)
12. [Using Motion with Next.js App Router](#12-motion-with-nextjs)
13. [Accessibility & Reduced Motion](#13-accessibility)
14. [Performance & Bundle Size](#14-performance)
15. [Practical Recipes](#15-practical-recipes)
16. [Tips, Tricks & Gotchas](#16-tips-tricks--gotchas)
17. [Study Path](#17-study-path)

---

## 1. What Motion Is & the Rebrand

### What is Motion?

**Motion** is the most popular animation library for React. It gives you a declarative, prop-driven API for animating DOM elements — you describe the *what* (start state, end state, transition type), not the *how* (requestAnimationFrame loops, keyframe math).

Key selling points vs. CSS animations:
- Animate layout changes (height: auto → fixed, element reordering) that CSS cannot handle.
- Exit animations on unmount — normally impossible in React without extra work.
- Spring physics that feel natural instead of cubic-bezier guesswork.
- Scroll-linked effects, drag, gestures, and shared-element transitions.
- Tiny mental model: add a `motion.` prefix to any HTML element, set `animate`, done.

### The Rebrand: Framer Motion → Motion

| What changed | Details |
|---|---|
| Old package name | `framer-motion` |
| New package name | `motion` |
| Old import path | `import { motion } from "framer-motion"` |
| New import path | `import { motion } from "motion/react"` |
| Website | motion.dev (was framer.com/motion) |
| Backwards compat | `framer-motion` still publishes updates and re-exports the same API — existing projects don't break |
| Vanilla JS API | `import { animate } from "motion"` (no React) |

> **⚡ Version note:** The rebrand happened with `motion` v11 (late 2024). If you see `framer-motion` in tutorials, the API is nearly identical — just swap the import path. Both packages coexist fine; don't install both in the same project.

### React API vs. Vanilla JS API

```
motion/react   → React components (motion.div, AnimatePresence, hooks, etc.)
motion         → Framework-agnostic imperative API (animate(), scroll(), inView(), etc.)
```

This guide focuses on `motion/react`. The vanilla `motion` package is useful for animating non-React contexts (vanilla JS, Web Components, other frameworks).

---

## 2. Setup & Imports

### Installation

```bash
# The modern package (recommended for new projects)
npm install motion

# Legacy — still works, still maintained
npm install framer-motion
```

### Core imports from `motion/react`

```tsx
// ─── Core component & presence ───────────────────────────────────────────────
import { motion, AnimatePresence } from "motion/react"

// ─── Hooks ───────────────────────────────────────────────────────────────────
import {
  useMotionValue,       // create an animatable value
  useTransform,         // derive a new value from one or more motion values
  useSpring,            // apply spring physics to a motion value
  useScroll,            // track scroll progress of the page or an element
  useAnimate,           // imperative animation (replaces the old useAnimation)
  useInView,            // detect when an element enters the viewport
  useReducedMotion,     // respect prefers-reduced-motion
  useMotionValueEvent,  // subscribe to motion value changes
} from "motion/react"

// ─── Providers & config ───────────────────────────────────────────────────────
import { MotionConfig, LazyMotion, domAnimation, m } from "motion/react"

// ─── Layout ───────────────────────────────────────────────────────────────────
import { LayoutGroup, Reorder } from "motion/react"
```

### Verify the setup

```tsx
// src/components/TestMotion.tsx
"use client" // required in Next.js App Router

import { motion } from "motion/react"

export function TestMotion() {
  return (
    <motion.div
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: 0.5 }}
      className="p-8 bg-blue-500 text-white rounded-xl"
    >
      Motion is working!
    </motion.div>
  )
}
```

---

## 3. The `motion` Component

### How it works

Any standard HTML or SVG element has a `motion` counterpart. The `motion` prefix adds three animation-related props: `initial`, `animate`, and `exit`. You describe CSS properties (and Motion shorthand transforms) as plain JS objects.

```tsx
// Every HTML element is available
<motion.div />
<motion.span />
<motion.button />
<motion.img />
<motion.section />
<motion.h1 />
<motion.ul />
<motion.li />

// SVG elements too
<motion.svg />
<motion.path />
<motion.circle />
<motion.rect />
```

### The three key animation props

| Prop | Purpose |
|---|---|
| `initial` | The starting state (before the component mounts or the animation begins). |
| `animate` | The target state to animate toward. |
| `exit` | The state to animate to when the component is *removed* from the tree (requires `<AnimatePresence>`). |

```tsx
"use client"
import { motion } from "motion/react"

export function FadeIn() {
  return (
    <motion.div
      initial={{ opacity: 0, scale: 0.8 }}   // start: invisible + slightly small
      animate={{ opacity: 1, scale: 1 }}      // end: visible + full size
      exit={{ opacity: 0, scale: 0.8 }}       // leave: shrink + fade
    >
      Hello
    </motion.div>
  )
}
```

### Animatable CSS properties

You can animate almost any CSS property by its camelCase name:

```tsx
<motion.div
  animate={{
    // ─── Layout ───────────────────────────────
    width: 200,
    height: 100,
    borderRadius: 20,
    padding: 16,

    // ─── Colors ───────────────────────────────
    backgroundColor: "#3b82f6",
    color: "#ffffff",
    borderColor: "rgba(0,0,0,0.2)",
    boxShadow: "0px 10px 30px rgba(0,0,0,0.2)",

    // ─── Typography ───────────────────────────
    fontSize: 24,
    letterSpacing: 2,
    opacity: 1,
  }}
/>
```

### Motion's transform shorthand

Instead of the CSS `transform` string, Motion gives you individual props:

| Motion prop | CSS equivalent | Notes |
|---|---|---|
| `x` | `translateX` | pixels by default |
| `y` | `translateY` | pixels by default |
| `z` | `translateZ` | pixels |
| `rotate` | `rotate` | degrees |
| `rotateX` / `rotateY` / `rotateZ` | `rotateX(...)` etc. | degrees |
| `scale` | `scale` | 1 = normal |
| `scaleX` / `scaleY` | `scaleX(...)` etc. | 1 = normal |
| `skew` | `skew` | degrees |
| `skewX` / `skewY` | `skewX(...)` etc. | degrees |
| `perspective` | `perspective` | pixels |

```tsx
"use client"
import { motion } from "motion/react"

export function TransformDemo() {
  return (
    <motion.div
      initial={{ x: -100, opacity: 0, rotate: -10 }}
      animate={{ x: 0, opacity: 1, rotate: 0 }}
      transition={{ type: "spring", stiffness: 100 }}
      className="w-32 h-32 bg-purple-500 rounded-lg"
    />
  )
}
```

### Conditional animate — toggle between states

```tsx
"use client"
import { useState } from "react"
import { motion } from "motion/react"

export function ToggleBox() {
  const [isOpen, setIsOpen] = useState(false)

  return (
    <motion.div
      animate={{
        // animate changes to any prop when isOpen flips
        width: isOpen ? 300 : 100,
        height: isOpen ? 200 : 100,
        backgroundColor: isOpen ? "#10b981" : "#3b82f6",
        borderRadius: isOpen ? 16 : 50,
      }}
      transition={{ duration: 0.4, ease: "easeInOut" }}
      onClick={() => setIsOpen(!isOpen)}
      className="cursor-pointer"
    />
  )
}
```

### `style` prop with motion values

The `style` prop on a `motion.*` element accepts both normal style values *and* `MotionValue` objects (covered in Section 10).

```tsx
<motion.div style={{ x: myMotionValue, opacity: anotherMotionValue }} />
```

---

## 4. Transitions

The `transition` prop controls *how* an animation moves from A to B.

### Duration & ease (tween)

```tsx
<motion.div
  animate={{ x: 100 }}
  transition={{
    type: "tween",      // default for most properties (explicit)
    duration: 0.6,      // seconds
    ease: "easeOut",    // easing curve
    delay: 0.2,         // seconds before animation starts
  }}
/>
```

### Ease values

| String shorthand | Description |
|---|---|
| `"linear"` | Constant speed |
| `"easeIn"` | Starts slow, ends fast |
| `"easeOut"` | Starts fast, ends slow (most natural for enter) |
| `"easeInOut"` | Slow start AND end |
| `"circIn"` / `"circOut"` / `"circInOut"` | Circular easing (sharper) |
| `"backIn"` / `"backOut"` / `"backInOut"` | Overshoots slightly |
| `"anticipate"` | Pulls back before moving forward |
| `[0.17, 0.67, 0.83, 0.67]` | Custom cubic bezier (4-point array) |

### Spring physics

Springs are the most natural-feeling motion. Use them for interactions (button presses, drag-and-drop).

```tsx
<motion.div
  animate={{ x: 200, rotate: 45 }}
  transition={{
    type: "spring",
    stiffness: 300,   // higher = faster/snappier (100–500 typical)
    damping: 20,      // higher = less bounce (10–30 typical)
    mass: 1,          // higher = heavier/slower
    velocity: 0,      // initial velocity
    restDelta: 0.001, // stops when below this distance (fine-tune)
  }}
/>
```

**Spring tuning quick reference:**

| Feel | Stiffness | Damping |
|---|---|---|
| Snappy / UI click | 400–600 | 25–35 |
| Bouncy / playful | 200–300 | 10–15 |
| Heavy / slow | 80–120 | 20–30 |
| Overdamped (no bounce) | 200 | 40+ |

### Inertia (deceleration)

Used for drag-momentum. The element decelerates naturally after a drag ends.

```tsx
<motion.div
  drag
  dragConstraints={{ left: 0, right: 200 }}
  transition={{
    type: "inertia",
    velocity: 50,     // initial velocity when released
    power: 0.8,       // amplifies velocity
    timeConstant: 700, // ms — higher = longer deceleration
    bounceStiffness: 400,
    bounceDamping: 40,
  }}
/>
```

### Repeating animations

```tsx
<motion.div
  animate={{ rotate: 360 }}
  transition={{
    duration: 2,
    repeat: Infinity,         // number or Infinity
    repeatType: "loop",       // "loop" | "reverse" | "mirror"
    repeatDelay: 0.5,         // pause between repeats
    ease: "linear",
  }}
/>
```

| `repeatType` | Behaviour |
|---|---|
| `"loop"` | Jumps back to start and repeats forward |
| `"reverse"` | Plays forward, then backward (like `animation-direction: alternate`) |
| `"mirror"` | Same as reverse but easing is also mirrored |

### Per-property transitions

Each property inside `animate` can have its own transition:

```tsx
<motion.div
  animate={{ x: 100, opacity: 1, rotate: 180 }}
  transition={{
    // default for all properties
    duration: 0.5,
    // overrides per property
    x: { type: "spring", stiffness: 200 },
    opacity: { duration: 0.3, ease: "easeIn" },
    rotate: { delay: 0.4, duration: 1 },
  }}
/>
```

### Stagger in transitions (see also Variants)

```tsx
// Stagger is defined in the parent's transition; individual children use their own transition
// Full stagger pattern: see Section 6 (Variants)
```

---

## 5. Gestures

Motion gives you interactive gesture props that accept the same target-state objects as `animate`.

### Hover, tap, focus

```tsx
"use client"
import { motion } from "motion/react"

export function InteractiveCard() {
  return (
    <motion.button
      // Animate while hovered
      whileHover={{
        scale: 1.05,
        boxShadow: "0px 10px 30px rgba(0,0,0,0.2)",
        y: -4,
      }}
      // Animate while pressed
      whileTap={{
        scale: 0.97,
        y: 0,
      }}
      // Animate while keyboard-focused
      whileFocus={{
        outline: "2px solid #3b82f6",
        outlineOffset: 4,
      }}
      transition={{ type: "spring", stiffness: 400, damping: 25 }}
      className="px-6 py-3 bg-blue-500 text-white rounded-xl cursor-pointer"
    >
      Click me
    </motion.button>
  )
}
```

### whileInView — scroll-triggered animations

```tsx
"use client"
import { motion } from "motion/react"

export function RevealSection() {
  return (
    <motion.div
      initial={{ opacity: 0, y: 60 }}
      whileInView={{ opacity: 1, y: 0 }}
      // viewport controls when the animation fires
      viewport={{
        once: true,         // only animate once (don't reverse when scrolling back)
        amount: 0.3,        // fire when 30% of the element is visible
        margin: "0px 0px -100px 0px", // rootMargin equivalent (shrinks viewport bottom by 100px)
      }}
      transition={{ duration: 0.6, ease: "easeOut" }}
    >
      I animate when scrolled into view
    </motion.div>
  )
}
```

### Drag

```tsx
"use client"
import { useRef } from "react"
import { motion } from "motion/react"

export function DraggableCard() {
  const constraintRef = useRef(null)

  return (
    // The parent defines the drag boundary
    <div ref={constraintRef} className="relative w-80 h-80 bg-gray-100 rounded-2xl overflow-hidden">
      <motion.div
        drag                            // enable drag in both axes
        // drag="x"                     // or restrict to x or y axis
        dragConstraints={constraintRef} // stay inside the parent
        // dragConstraints={{ top: 0, bottom: 200, left: 0, right: 200 }} // or pixel values
        dragElastic={0.2}               // how much the element can go beyond constraints (0=rigid, 1=full elastic)
        dragMomentum={true}             // coasts to a stop after release
        dragSnapToOrigin={false}        // snap back to start position
        whileDrag={{ scale: 1.1, zIndex: 10, cursor: "grabbing" }}
        className="absolute top-4 left-4 w-24 h-24 bg-purple-500 rounded-xl cursor-grab"
      />
    </div>
  )
}
```

### Drag with callbacks

```tsx
<motion.div
  drag
  onDragStart={(event, info) => {
    console.log("started", info.point)  // { x, y } on the page
  }}
  onDrag={(event, info) => {
    console.log("dragging delta", info.delta) // { x, y } moved this frame
    console.log("offset from start", info.offset) // { x, y }
    console.log("velocity", info.velocity)   // { x, y }
  }}
  onDragEnd={(event, info) => {
    // Snap-to-side example: if dragged far left, keep left
    if (info.offset.x < -100) doSomething()
  }}
/>
```

---

## 6. Variants

Variants let you define named animation states in an object, then reference them by name in the prop. The real power: **they propagate to children automatically**.

### Basic variants

```tsx
"use client"
import { motion } from "motion/react"

// Define states as named objects
const cardVariants = {
  hidden: {
    opacity: 0,
    y: 50,
    scale: 0.9,
  },
  visible: {
    opacity: 1,
    y: 0,
    scale: 1,
  },
}

export function AnimatedCard() {
  return (
    <motion.div
      variants={cardVariants}
      initial="hidden"
      animate="visible"
      transition={{ duration: 0.5 }}
    >
      Card content
    </motion.div>
  )
}
```

### Parent → child propagation (orchestration)

When a parent has `animate="visible"`, all children with `variants` automatically receive the same `animate` value — you don't need to repeat it on every child.

```tsx
"use client"
import { motion } from "motion/react"

// Parent controls the orchestration
const containerVariants = {
  hidden: {},
  visible: {
    transition: {
      staggerChildren: 0.1,   // each child starts 100ms after the previous
      delayChildren: 0.2,     // wait 200ms before the first child starts
      // staggerDirection: -1, // reverse stagger order
      // when: "beforeChildren", // parent animates before its children
      // when: "afterChildren",  // parent waits for children to finish
    },
  },
}

// Child defines its own animation states (same names as parent)
const itemVariants = {
  hidden: {
    opacity: 0,
    y: 20,
  },
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
      animate="visible"
      className="space-y-3"
    >
      {items.map((item) => (
        <motion.li
          key={item}
          variants={itemVariants}
          // No initial/animate needed — inherited from parent!
          className="p-4 bg-white rounded-lg shadow"
        >
          {item}
        </motion.li>
      ))}
    </motion.ul>
  )
}
```

### Dynamic variants (functions)

Variant states can be functions that receive a `custom` prop:

```tsx
"use client"
import { motion } from "motion/react"

const itemVariants = {
  hidden: { opacity: 0, x: -50 },
  visible: (i: number) => ({
    opacity: 1,
    x: 0,
    transition: {
      delay: i * 0.1,           // each item uses its index as delay
      type: "spring",
      stiffness: 200,
    },
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
          custom={i}             // passed to the variant function
        >
          {item}
        </motion.li>
      ))}
    </ul>
  )
}
```

### Variants with gestures

Variants work with gesture props too, allowing coordinated multi-element hover effects:

```tsx
const cardVariants = {
  rest: { scale: 1 },
  hover: { scale: 1.02 },
}
const overlayVariants = {
  rest: { opacity: 0 },
  hover: { opacity: 1 },
}

export function HoverCard() {
  return (
    // Parent controls hover state; child reacts automatically
    <motion.div
      variants={cardVariants}
      initial="rest"
      whileHover="hover"
      className="relative w-64 h-40 bg-blue-500 rounded-xl overflow-hidden"
    >
      <motion.div
        variants={overlayVariants}
        className="absolute inset-0 bg-black/20 flex items-center justify-center text-white"
      >
        Hover overlay
      </motion.div>
    </motion.div>
  )
}
```

---

## 7. AnimatePresence

`AnimatePresence` enables **exit animations** — animating elements *out* before they are removed from the DOM.

### The problem it solves

React removes elements from the DOM immediately when they unmount. `AnimatePresence` intercepts the removal, runs the `exit` animation, then removes the element.

### Basic usage

```tsx
"use client"
import { useState } from "react"
import { motion, AnimatePresence } from "motion/react"

export function ToggleMessage() {
  const [show, setShow] = useState(true)

  return (
    <>
      <button onClick={() => setShow(!show)}>Toggle</button>

      {/* AnimatePresence MUST wrap the conditionally rendered element */}
      <AnimatePresence>
        {show && (
          <motion.div
            key="message"               // key is required for AnimatePresence to track the element
            initial={{ opacity: 0, height: 0 }}
            animate={{ opacity: 1, height: "auto" }}
            exit={{ opacity: 0, height: 0 }}  // runs when show becomes false
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

> **Gotcha:** The `key` prop is required on the direct child of `AnimatePresence`. Without it, Motion cannot track which element is entering vs. exiting.

### AnimatePresence `mode`

| mode | Behaviour |
|---|---|
| `"sync"` (default) | Entering and exiting happen at the same time |
| `"wait"` | Wait for the exiting element to fully disappear before entering the new one |
| `"popLayout"` | Exiting element is popped out of layout flow (positioned absolutely during exit) |

```tsx
// Page transition example — wait for old page to leave before new one enters
<AnimatePresence mode="wait">
  <motion.div
    key={currentPage}          // change key to trigger exit/enter cycle
    initial={{ opacity: 0, x: 20 }}
    animate={{ opacity: 1, x: 0 }}
    exit={{ opacity: 0, x: -20 }}
    transition={{ duration: 0.25 }}
  >
    {pageContent}
  </motion.div>
</AnimatePresence>
```

### Animating list items (add/remove)

```tsx
"use client"
import { useState } from "react"
import { motion, AnimatePresence } from "motion/react"

let nextId = 4

export function AnimatedList() {
  const [items, setItems] = useState([
    { id: 1, text: "Item 1" },
    { id: 2, text: "Item 2" },
    { id: 3, text: "Item 3" },
  ])

  const addItem = () => {
    setItems([...items, { id: nextId++, text: `Item ${nextId - 1}` }])
  }

  const removeItem = (id: number) => {
    setItems(items.filter((item) => item.id !== id))
  }

  return (
    <>
      <button onClick={addItem}>Add Item</button>
      <ul className="space-y-2 mt-4">
        <AnimatePresence>
          {items.map((item) => (
            <motion.li
              key={item.id}                    // unique, stable key
              initial={{ opacity: 0, x: -20, height: 0 }}
              animate={{ opacity: 1, x: 0, height: "auto" }}
              exit={{ opacity: 0, x: 20, height: 0 }}
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

### Modal with AnimatePresence

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
          // Backdrop
          <motion.div
            key="backdrop"
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            onClick={() => setIsOpen(false)}
            className="fixed inset-0 bg-black/50 flex items-center justify-center z-50"
          >
            {/* Modal panel — stop backdrop click from propagating */}
            <motion.div
              key="modal"
              initial={{ opacity: 0, scale: 0.9, y: 20 }}
              animate={{ opacity: 1, scale: 1, y: 0 }}
              exit={{ opacity: 0, scale: 0.9, y: 20 }}
              transition={{ type: "spring", stiffness: 300, damping: 25 }}
              onClick={(e) => e.stopPropagation()}
              className="bg-white rounded-2xl p-8 max-w-md w-full shadow-2xl"
            >
              <h2 className="text-xl font-bold">Modal Title</h2>
              <p className="mt-2 text-gray-600">Modal content here.</p>
              <button
                onClick={() => setIsOpen(false)}
                className="mt-4 px-4 py-2 bg-blue-500 text-white rounded-lg"
              >
                Close
              </button>
            </motion.div>
          </motion.div>
        )}
      </AnimatePresence>
    </>
  )
}
```

---

## 8. Layout Animations

### The `layout` prop

Add `layout` to a `motion.*` element and Motion will automatically animate it whenever its size or position changes — even if caused by DOM changes elsewhere (other elements being added/removed, CSS changes, etc.).

```tsx
"use client"
import { useState } from "react"
import { motion } from "motion/react"

export function ExpandingCard() {
  const [expanded, setExpanded] = useState(false)

  return (
    <motion.div
      layout                    // animate ANY layout change automatically
      onClick={() => setExpanded(!expanded)}
      className="bg-blue-500 text-white rounded-xl p-6 cursor-pointer w-64"
    >
      <motion.h2 layout="position">Card Title</motion.h2>
      {expanded && (
        <motion.p
          initial={{ opacity: 0 }}
          animate={{ opacity: 1 }}
          className="mt-2"
        >
          Expanded content that changes the card's height. Motion animates the
          height change automatically — no need to know the height in advance.
        </motion.p>
      )}
    </motion.div>
  )
}
```

**`layout` values:**

| Value | Animates |
|---|---|
| `true` (or just `layout`) | Size AND position |
| `"position"` | Position only (not size) |
| `"size"` | Size only (not position) |
| `"preserve-aspect"` | Keeps aspect ratio during resize |

### Shared layout animations with `layoutId` (Magic Move)

`layoutId` creates a "magic move" — when one element with a `layoutId` unmounts and another with the same `layoutId` mounts (even in a different part of the tree), Motion smoothly transitions between them.

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
            layoutId={item}             // same layoutId as the expanded view
            onClick={() => setSelected(item)}
            className="h-32 bg-blue-500 rounded-xl cursor-pointer"
          />
        ))}
      </div>

      {/* Full-screen expanded view */}
      {selected && (
        <motion.div
          layoutId={selected}           // matches the clicked thumbnail
          onClick={() => setSelected(null)}
          className="fixed inset-10 bg-blue-500 rounded-3xl cursor-pointer z-50"
        />
      )}
    </>
  )
}
```

> **Tip:** Combine `layoutId` with `AnimatePresence` for the exit animation when the expanded view closes.

### `LayoutGroup`

Wrap sibling subtrees in `LayoutGroup` so that layout changes in one trigger layout animations in the others:

```tsx
import { LayoutGroup } from "motion/react"

export function TabInterface() {
  return (
    // Without LayoutGroup, the active-tab indicator won't animate across tabs
    <LayoutGroup>
      <Tab id="tab1" />
      <Tab id="tab2" />
      <Tab id="tab3" />
    </LayoutGroup>
  )
}

function Tab({ id }: { id: string }) {
  const isActive = /* ... */ true
  return (
    <div className="relative">
      <span>{id}</span>
      {isActive && (
        <motion.div
          layoutId="active-underline"     // same layoutId across all tab instances
          className="absolute bottom-0 left-0 right-0 h-0.5 bg-blue-500"
        />
      )}
    </div>
  )
}
```

---

## 9. Scroll Animations

### `whileInView` (simplest approach)

Already covered in Section 5 — the simplest scroll animation. The element animates when it enters the viewport.

```tsx
<motion.div
  initial={{ opacity: 0, y: 40 }}
  whileInView={{ opacity: 1, y: 0 }}
  viewport={{ once: true, amount: 0.2 }}
  transition={{ duration: 0.5 }}
>
  Scroll-triggered content
</motion.div>
```

### `useScroll` — scroll progress tracking

```tsx
"use client"
import { useScroll, useTransform, motion } from "motion/react"

export function ScrollProgressBar() {
  // Track the page scroll progress
  const { scrollYProgress } = useScroll()
  // scrollYProgress is a MotionValue from 0 (top) to 1 (bottom)

  return (
    // A bar that grows as you scroll down
    <motion.div
      style={{ scaleX: scrollYProgress }}
      className="fixed top-0 left-0 right-0 h-1 bg-blue-500 origin-left z-50"
    />
  )
}
```

### `useScroll` with element tracking

```tsx
"use client"
import { useRef } from "react"
import { useScroll, useTransform, motion } from "motion/react"

export function ParallaxSection() {
  const ref = useRef(null)

  // Track scroll progress *within the element's viewport window*
  const { scrollYProgress } = useScroll({
    target: ref,
    offset: ["start end", "end start"],
    // offset[0] = when does progress reach 0? ("start of element" hits "end of viewport")
    // offset[1] = when does progress reach 1? ("end of element" hits "start of viewport")
  })

  // Map scroll progress 0→1 to a y offset for parallax
  const y = useTransform(scrollYProgress, [0, 1], ["-20%", "20%"])
  const opacity = useTransform(scrollYProgress, [0, 0.3, 0.7, 1], [0, 1, 1, 0])

  return (
    <div ref={ref} className="relative h-screen overflow-hidden">
      <motion.div
        style={{ y, opacity }}
        className="absolute inset-0 bg-gradient-to-b from-blue-500 to-purple-600"
      />
      <div className="relative z-10 flex items-center justify-center h-full text-white text-4xl font-bold">
        Parallax Section
      </div>
    </div>
  )
}
```

### `useScroll` offset reference table

| Offset string | Meaning |
|---|---|
| `"start start"` | When the element's top hits the viewport top |
| `"start end"` | When the element's top hits the viewport bottom (element just enters view) |
| `"end start"` | When the element's bottom hits the viewport top (element just exits view) |
| `"end end"` | When the element's bottom hits the viewport bottom |
| `"center center"` | When the element's center hits the viewport center |

You can also use pixel values: `"100px"`, or percentage strings like `"20%"`.

### Scroll-linked rotation / scale

```tsx
"use client"
import { useScroll, useTransform, motion } from "motion/react"

export function RotatingLogo() {
  const { scrollY } = useScroll()
  // As you scroll 0→500px, rotate from 0 to 360 degrees
  const rotate = useTransform(scrollY, [0, 500], [0, 360])
  const scale = useTransform(scrollY, [0, 300], [1, 0.5])

  return (
    <motion.div
      style={{ rotate, scale }}
      className="w-24 h-24 bg-blue-500 rounded-full fixed top-4 right-4"
    />
  )
}
```

### `useInView` hook (imperative)

```tsx
"use client"
import { useRef } from "react"
import { useInView, motion } from "motion/react"

export function CounterOnView() {
  const ref = useRef(null)
  const isInView = useInView(ref, {
    once: true,       // stop tracking after first entry
    amount: 0.5,      // 50% must be visible
    margin: "0px 0px -100px 0px", // shrink the detection zone
  })

  return (
    <div ref={ref}>
      <motion.span
        animate={{ opacity: isInView ? 1 : 0 }}
        transition={{ duration: 0.5 }}
      >
        {isInView ? "Visible!" : "Not yet"}
      </motion.span>
    </div>
  )
}
```

---

## 10. Motion Values & Hooks

Motion values are the underlying reactive primitive. They power the style prop and are the bridge between JavaScript logic and GPU-accelerated rendering.

### `useMotionValue`

```tsx
"use client"
import { useMotionValue, motion } from "motion/react"

export function DragTracker() {
  // Creates a reactive value NOT connected to React's render cycle
  const x = useMotionValue(0)

  return (
    <motion.div
      drag="x"
      style={{ x }}       // x updates without re-renders
      className="w-20 h-20 bg-blue-500 rounded-full cursor-grab"
    />
  )
}
```

> **Key concept:** Updating a `MotionValue` does NOT cause a React re-render. This is intentional — Motion updates the DOM directly for performance. Use `useMotionValueEvent` when you need to sync to React state.

### `useTransform`

Derive one value from another:

```tsx
"use client"
import { useMotionValue, useTransform, motion } from "motion/react"

export function ColorDrag() {
  const x = useMotionValue(0)

  // Map x position -200→200 to a color
  const backgroundColor = useTransform(
    x,
    [-200, 0, 200],
    ["#ef4444", "#3b82f6", "#10b981"]  // red → blue → green
  )

  // Map x to rotation
  const rotate = useTransform(x, [-200, 200], [-30, 30])

  // Map x to opacity (fade out at extremes)
  const opacity = useTransform(x, [-200, -100, 0, 100, 200], [0.2, 1, 1, 1, 0.2])

  return (
    <motion.div
      drag="x"
      dragConstraints={{ left: -200, right: 200 }}
      style={{ x, backgroundColor, rotate, opacity }}
      className="w-20 h-20 rounded-lg cursor-grab"
    />
  )
}
```

`useTransform` with a function (for complex mappings):

```tsx
// Map x to a custom formula
const scale = useTransform(x, (latest) => 1 + Math.abs(latest) / 400)
```

`useTransform` to combine multiple values:

```tsx
import { useTransform } from "motion/react"

// Combine x and y into a CSS filter
const blur = useTransform(
  [x, y],
  ([latestX, latestY]) => `blur(${Math.abs(latestX) / 20 + Math.abs(latestY) / 20}px)`
)
```

### `useSpring`

Apply spring physics to any motion value:

```tsx
"use client"
import { useMotionValue, useSpring, motion } from "motion/react"

export function SpringCursor() {
  const mouseX = useMotionValue(0)
  const mouseY = useMotionValue(0)

  // The spring lags behind the mouse — smooth follow effect
  const springX = useSpring(mouseX, { stiffness: 150, damping: 15 })
  const springY = useSpring(mouseY, { stiffness: 150, damping: 15 })

  const handleMouseMove = (e: React.MouseEvent) => {
    mouseX.set(e.clientX)
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

### `useAnimate` — imperative animations

Use when you need programmatic control (e.g., animate on button press, sequence multiple elements):

```tsx
"use client"
import { useAnimate } from "motion/react"

export function SequenceAnimation() {
  // scope: a ref to the root element; animate: the imperative animate function
  const [scope, animate] = useAnimate()

  const runSequence = async () => {
    // Animate elements using CSS selectors relative to scope
    await animate("h1", { opacity: [0, 1], y: [20, 0] }, { duration: 0.4 })
    await animate("p", { opacity: [0, 1] }, { duration: 0.3 })
    await animate(".cta", { scale: [0.9, 1] }, { type: "spring" })
    // animate returns a Promise — await sequences the animations
  }

  return (
    <div ref={scope}>
      <h1 style={{ opacity: 0 }}>Title</h1>
      <p style={{ opacity: 0 }}>Description</p>
      <button className="cta" onClick={runSequence}>
        Run Sequence
      </button>
    </div>
  )
}
```

### `useMotionValueEvent`

Subscribe to motion value changes (bridge to React state):

```tsx
"use client"
import { useState } from "react"
import { useMotionValue, useMotionValueEvent, motion } from "motion/react"

export function PositionDisplay() {
  const x = useMotionValue(0)
  const [xPos, setXPos] = useState(0)

  // Subscribe without causing infinite re-renders
  useMotionValueEvent(x, "change", (latest) => {
    setXPos(Math.round(latest))
  })

  return (
    <>
      <p>X position: {xPos}px</p>
      <motion.div
        drag="x"
        dragConstraints={{ left: -200, right: 200 }}
        style={{ x }}
        className="w-20 h-20 bg-blue-500 rounded-lg cursor-grab"
      />
    </>
  )
}
```

---

## 11. Keyframes, Colors & SVG

### Keyframe arrays

Pass an array of values to animate through multiple states:

```tsx
<motion.div
  animate={{
    // Animate through multiple values in sequence
    x: [0, 100, -50, 80, 0],   // bounce around and return
    backgroundColor: ["#3b82f6", "#ef4444", "#10b981"],
    borderRadius: ["10%", "50%", "10%"],
  }}
  transition={{
    duration: 2,
    ease: "easeInOut",
    times: [0, 0.2, 0.5, 0.8, 1], // where each keyframe falls (0–1)
    // If times is omitted, keyframes are evenly distributed
    repeat: Infinity,
    repeatType: "reverse",
  }}
  className="w-20 h-20"
/>
```

### null keyframe (jump to)

Use `null` to use the element's *current* value at that keyframe position:

```tsx
<motion.div
  animate={{
    x: [null, 100, 0],  // start from wherever it is, go to 100, come back to 0
  }}
/>
```

### Color animation

Motion can animate between any CSS color formats:

```tsx
<motion.div
  animate={{
    backgroundColor: "#3b82f6",            // hex
    color: "rgb(255, 255, 255)",           // rgb
    borderColor: "hsl(220, 80%, 60%)",     // hsl
    boxShadow: "0px 0px 20px rgba(59, 130, 246, 0.5)", // with color in shadow
  }}
/>
```

### SVG path animation

Animate `pathLength`, `pathOffset`, and `pathSpacing` to draw SVGs:

```tsx
"use client"
import { motion } from "motion/react"

export function DrawingCheckmark() {
  return (
    <svg viewBox="0 0 50 50" width="100" height="100">
      <motion.path
        d="M 10 30 L 22 42 L 40 12"
        fill="none"
        stroke="#10b981"
        strokeWidth={4}
        strokeLinecap="round"
        // pathLength is normalized to 0→1 regardless of actual SVG length
        initial={{ pathLength: 0, opacity: 0 }}
        animate={{ pathLength: 1, opacity: 1 }}
        transition={{
          pathLength: { duration: 0.8, ease: "easeOut" },
          opacity: { duration: 0.1 },
        }}
      />
    </svg>
  )
}
```

### Animated SVG icon (circle loading)

```tsx
"use client"
import { motion } from "motion/react"

export function LoadingCircle() {
  return (
    <svg viewBox="0 0 50 50" width={48} height={48}>
      <motion.circle
        cx={25}
        cy={25}
        r={20}
        fill="none"
        stroke="#3b82f6"
        strokeWidth={4}
        strokeLinecap="round"
        initial={{ pathLength: 0, rotate: -90 }}
        animate={{ pathLength: 0.75, rotate: 270 }}
        transition={{
          duration: 1,
          repeat: Infinity,
          ease: "easeInOut",
          repeatType: "reverse",
        }}
      />
    </svg>
  )
}
```

---

## 12. Motion with Next.js App Router

### The fundamental rule: `"use client"`

Motion components are **client-side only** — they use browser APIs (`requestAnimationFrame`, `window`, CSS transitions). In Next.js App Router, **all components are Server Components by default**. You must add `"use client"` to any file that uses Motion.

```tsx
// app/components/AnimatedHero.tsx

"use client" // ← REQUIRED — motion components cannot run on the server

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
// app/page.tsx — this is a Server Component (no "use client")

import { AnimatedHero } from "@/components/AnimatedHero"

export default function HomePage() {
  // Fetch data server-side...
  return (
    <main>
      {/* AnimatedHero is a Client Component but can be used here */}
      <AnimatedHero />
      <p>Server-rendered content here</p>
    </main>
  )
}
```

### Recommended pattern: thin client wrapper

Keep as much logic in Server Components as possible. Only the animated parts need `"use client"`:

```tsx
// components/FadeInSection.tsx — reusable animated wrapper
"use client"
import { motion } from "motion/react"

interface FadeInSectionProps {
  children: React.ReactNode
  delay?: number
  className?: string
}

export function FadeInSection({ children, delay = 0, className }: FadeInSectionProps) {
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
// app/about/page.tsx — Server Component passes children to the wrapper
import { FadeInSection } from "@/components/FadeInSection"

export default async function AboutPage() {
  const data = await fetchAboutData()  // server-side fetch

  return (
    <main>
      <FadeInSection delay={0}>
        <h1>{data.title}</h1>
      </FadeInSection>
      <FadeInSection delay={0.1}>
        <p>{data.description}</p>
      </FadeInSection>
    </main>
  )
}
```

### Page transition with Next.js App Router

The App Router does NOT re-mount the layout between page navigations. Use `template.tsx` instead of `layout.tsx` if you need the wrapping component to re-mount (and thus re-run animations) on every navigation.

```tsx
// app/template.tsx — re-mounts on every navigation (unlike layout.tsx)
"use client"
import { motion, AnimatePresence } from "motion/react"

export default function Template({ children }: { children: React.ReactNode }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 10 }}
      animate={{ opacity: 1, y: 0 }}
      exit={{ opacity: 0, y: -10 }}
      transition={{ duration: 0.25, ease: "easeInOut" }}
    >
      {children}
    </motion.div>
  )
}
```

> **⚡ Version note:** `exit` animations in `template.tsx` require `AnimatePresence` to be wrapped around the `{children}` in the parent layout. This is tricky with the App Router because the exit fires before the new page mounts. The `animate` (enter) is reliable; `exit` in templates is an ongoing ecosystem discussion. For most projects, animate-only page transitions (no exit) are the pragmatic choice.

### Layout animation caution with SSR

Layout animations (`layout` prop) require the browser to measure element positions. Avoid them in components that render server-side HTML — they can cause a flash or layout shift on first paint. Strategies:

```tsx
// Option 1: Only use layout in client-only components (safest)
"use client"
import { motion } from "motion/react"
// layout prop is safe here

// Option 2: Disable initial animation to prevent hydration mismatch
<motion.div
  layout
  initial={false}   // skip the initial animation, only animate subsequent changes
>
```

### Reusable `<Reveal>` component (production-ready)

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
  children,
  direction = "up",
  delay = 0,
  duration = 0.5,
  className,
  once = true,
}: RevealProps) {
  const prefersReducedMotion = useReducedMotion()

  const directionMap = {
    up: { y: 40 },
    down: { y: -40 },
    left: { x: 40 },
    right: { x: -40 },
  }

  // Respect the user's accessibility preference
  if (prefersReducedMotion) {
    return <div className={className}>{children}</div>
  }

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

## 13. Accessibility

### `useReducedMotion`

The `prefers-reduced-motion` CSS media query tells you if a user has requested minimal motion (vestibular disorders, motion sensitivity, or personal preference). Always respect it.

```tsx
"use client"
import { useReducedMotion, motion } from "motion/react"

export function AccessibleAnimation() {
  const prefersReducedMotion = useReducedMotion()

  return (
    <motion.div
      // If reduced motion: skip the y movement, only fade
      initial={{ opacity: 0, y: prefersReducedMotion ? 0 : 40 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{
        duration: prefersReducedMotion ? 0.1 : 0.6,
      }}
    >
      Accessible animated content
    </motion.div>
  )
}
```

### `MotionConfig` — global defaults

`MotionConfig` lets you set defaults for all Motion components in its subtree:

```tsx
// app/layout.tsx
import { MotionConfig } from "motion/react"

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <MotionConfig
          // Reduce all animation durations when user prefers it
          reducedMotion="user"   // "user" | "always" | "never"
          // "user" → checks prefers-reduced-motion automatically
          // "always" → always reduce (useful for testing)
          // "never" → ignore the OS setting
          transition={{ duration: 0.3 }}  // default transition for all children
        >
          {children}
        </MotionConfig>
      </body>
    </html>
  )
}
```

> **Tip:** `reducedMotion="user"` in `MotionConfig` is the easiest way to make your entire site respect `prefers-reduced-motion` globally. It removes movement but keeps opacity/color transitions (which are still useful feedback).

### Accessible animation patterns

```tsx
// ✅ Good: convey state with opacity+color, not only position
<motion.button
  animate={{
    backgroundColor: isActive ? "#3b82f6" : "#e5e7eb",
    scale: isActive ? 1.02 : 1,  // subtle scale OK
  }}
/>

// ✅ Good: skip motion but keep opacity on reduced-motion
const prefersReduced = useReducedMotion()
<motion.div
  initial={{ opacity: 0 }}
  animate={{ opacity: 1 }}
  // Only add movement if not reduced
  {...(!prefersReduced && { y: [40, 0] })}
/>

// ✅ Good: use aria-live for content that appears after animation
<motion.div
  animate={{ opacity: 1 }}
  aria-live="polite"
>
  {message}
</motion.div>
```

---

## 14. Performance

### What to animate (GPU-accelerated)

| Property | GPU-composited? | Notes |
|---|---|---|
| `transform` (x, y, scale, rotate) | Yes | Always prefer these |
| `opacity` | Yes | Cheap to animate |
| `filter` (blur, brightness) | Yes on most browsers | Moderate cost |
| `width` / `height` | **No** | Causes layout recalculation |
| `padding` / `margin` | **No** | Causes layout recalculation |
| `background-color` | No | Paint cost, but minor |
| `box-shadow` | No | Paint cost |
| `clip-path` | Partial | OK for simple clips |

**Rule of thumb:** Animate `transform` and `opacity` whenever possible. Use `layout` prop sparingly (only when you truly need to animate unmeasurable sizes like `height: auto`).

### `will-change`

Tell the browser to promote an element to its own compositor layer in advance:

```tsx
<motion.div
  style={{ willChange: "transform" }}  // or "transform, opacity"
  animate={{ x: 100 }}
/>
```

> **Caution:** Overusing `will-change` consumes GPU memory. Only add it to elements that animate frequently (e.g., a cursor follower, a persistent scroll-linked animation). Don't add it to every animated element.

### `LazyMotion` — reduce bundle size

The default `motion` import includes all features (~33kb gzip). Use `LazyMotion` to only include what you need:

```tsx
// app/layout.tsx or a provider component
"use client"
import { LazyMotion, domAnimation } from "motion/react"

// domAnimation includes: animate, gestures, drag, layout (most common features)
// domMax includes: everything including FLIP layout animations

export function MotionProvider({ children }: { children: React.ReactNode }) {
  return (
    <LazyMotion features={domAnimation} strict>
      {children}
    </LazyMotion>
  )
}
```

```tsx
// When using LazyMotion, import from the `m` namespace instead of `motion`
"use client"
import { m } from "motion/react"  // lightweight — works with LazyMotion

export function LightweightAnimation() {
  return (
    <m.div
      initial={{ opacity: 0 }}
      animate={{ opacity: 1 }}
    />
  )
}
```

**Bundle size reference:**

| Import | Approx gzip size |
|---|---|
| `motion` (full) | ~33kb |
| `m` + `LazyMotion` + `domAnimation` | ~18kb |
| `m` + `LazyMotion` + `domMax` | ~25kb |
| Vanilla `motion` animate() | ~5kb |

### Async feature loading (code-split)

```tsx
import { LazyMotion } from "motion/react"

// Load features asynchronously — they're excluded from the initial bundle
const loadFeatures = () => import("motion/react").then((m) => m.domAnimation)

export function MotionProvider({ children }: { children: React.ReactNode }) {
  return (
    <LazyMotion features={loadFeatures}>
      {children}
    </LazyMotion>
  )
}
```

### Avoid layout thrashing with `layout`

```tsx
// ❌ Bad: layout animations on hundreds of list items will be slow
{items.map(item => (
  <motion.div layout key={item.id}>{item.text}</motion.div>
))}

// ✅ Better: only put layout on elements that actually need it
// ✅ Better: use transform-only animations for lists (variants with y/opacity)
```

---

## 15. Practical Recipes

### Recipe 1: Fade-up section reveal

```tsx
// components/FadeUp.tsx
"use client"
import { motion, useReducedMotion } from "motion/react"

export function FadeUp({
  children,
  delay = 0,
}: {
  children: React.ReactNode
  delay?: number
}) {
  const reduced = useReducedMotion()

  return (
    <motion.div
      initial={{ opacity: 0, y: reduced ? 0 : 50 }}
      whileInView={{ opacity: 1, y: 0 }}
      viewport={{ once: true, amount: 0.15 }}
      transition={{ duration: 0.6, delay, ease: [0.21, 0.47, 0.32, 0.98] }}
    >
      {children}
    </motion.div>
  )
}

// Usage in a page:
// <FadeUp delay={0}><h1>Title</h1></FadeUp>
// <FadeUp delay={0.1}><p>Subtitle</p></FadeUp>
// <FadeUp delay={0.2}><Button>CTA</Button></FadeUp>
```

### Recipe 2: Staggered card grid

```tsx
// components/StaggerGrid.tsx
"use client"
import { motion } from "motion/react"

const container = {
  hidden: {},
  show: {
    transition: {
      staggerChildren: 0.08,
      delayChildren: 0.1,
    },
  },
}

const card = {
  hidden: { opacity: 0, y: 30, scale: 0.96 },
  show: {
    opacity: 1,
    y: 0,
    scale: 1,
    transition: { type: "spring", stiffness: 200, damping: 20 },
  },
}

interface StaggerGridProps {
  items: { id: string; title: string; description: string }[]
}

export function StaggerGrid({ items }: StaggerGridProps) {
  return (
    <motion.div
      variants={container}
      initial="hidden"
      whileInView="show"
      viewport={{ once: true, amount: 0.1 }}
      className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6"
    >
      {items.map((item) => (
        <motion.div
          key={item.id}
          variants={card}
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

### Recipe 3: Animated modal / drawer

```tsx
// components/Drawer.tsx
"use client"
import { motion, AnimatePresence } from "motion/react"

interface DrawerProps {
  isOpen: boolean
  onClose: () => void
  children: React.ReactNode
}

export function Drawer({ isOpen, onClose, children }: DrawerProps) {
  return (
    <AnimatePresence>
      {isOpen && (
        <>
          {/* Backdrop */}
          <motion.div
            key="backdrop"
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            transition={{ duration: 0.2 }}
            onClick={onClose}
            className="fixed inset-0 bg-black/40 z-40"
          />

          {/* Drawer panel — slides in from the right */}
          <motion.aside
            key="drawer"
            initial={{ x: "100%" }}
            animate={{ x: 0 }}
            exit={{ x: "100%" }}
            transition={{ type: "spring", stiffness: 300, damping: 30 }}
            className="fixed top-0 right-0 bottom-0 w-80 bg-white shadow-2xl z-50 p-6 overflow-y-auto"
          >
            <button
              onClick={onClose}
              className="mb-4 text-gray-400 hover:text-gray-800"
            >
              ✕ Close
            </button>
            {children}
          </motion.aside>
        </>
      )}
    </AnimatePresence>
  )
}
```

### Recipe 4: Hover lift card

```tsx
// components/HoverCard.tsx
"use client"
import { motion } from "motion/react"

export function HoverCard({ children }: { children: React.ReactNode }) {
  return (
    <motion.div
      whileHover={{
        y: -8,
        scale: 1.02,
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

### Recipe 5: Scroll progress bar

```tsx
// components/ScrollProgressBar.tsx
"use client"
import { useScroll, useSpring, motion } from "motion/react"

export function ScrollProgressBar() {
  const { scrollYProgress } = useScroll()
  // Apply spring to make it feel smoother
  const scaleX = useSpring(scrollYProgress, {
    stiffness: 100,
    damping: 30,
    restDelta: 0.001,
  })

  return (
    <motion.div
      style={{ scaleX }}
      className="fixed top-0 left-0 right-0 h-1 bg-gradient-to-r from-blue-500 to-purple-500 origin-left z-[9999]"
    />
  )
}
```

### Recipe 6: Page transition wrapper

```tsx
// app/template.tsx — re-mounts on each navigation
"use client"
import { motion } from "motion/react"

export default function Template({ children }: { children: React.ReactNode }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 8 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: 0.3, ease: "easeOut" }}
    >
      {children}
    </motion.div>
  )
}
```

### Recipe 7: Animated counter

```tsx
// components/AnimatedCounter.tsx
"use client"
import { useEffect, useRef } from "react"
import { useInView, useMotionValue, useSpring, motion } from "motion/react"

export function AnimatedCounter({ target }: { target: number }) {
  const ref = useRef<HTMLSpanElement>(null)
  const isInView = useInView(ref, { once: true })
  const motionValue = useMotionValue(0)
  const springValue = useSpring(motionValue, { duration: 2000, bounce: 0 })

  useEffect(() => {
    if (isInView) motionValue.set(target)
  }, [isInView, motionValue, target])

  useEffect(() => {
    return springValue.on("change", (latest) => {
      if (ref.current) {
        ref.current.textContent = Math.round(latest).toLocaleString()
      }
    })
  }, [springValue])

  return <span ref={ref}>0</span>
}

// Usage: <AnimatedCounter target={1234567} />
```

### Recipe 8: Tab indicator (active underline magic move)

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
        <button
          key={tab}
          onClick={() => setActive(tab)}
          className="relative px-4 py-2 text-sm font-medium rounded-lg z-10"
        >
          {/* Active pill slides between tabs */}
          {active === tab && (
            <motion.div
              layoutId="active-tab-pill"
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

### "use client" in Next.js App Router

Every file that uses `motion`, `AnimatePresence`, or any Motion hook **must** have `"use client"` as its first line. Forgetting this is the #1 source of Motion errors in Next.js projects.

```
Error: Cannot use hook "useMotionValue" in a server component
```

Fix: Add `"use client"` at the top of the file.

### AnimatePresence: `exit` not firing

The most common cause: you forgot `key` on the child, or you're conditionally rendering at the wrong level.

```tsx
// ❌ Wrong — key is on a wrapper, not the animated element
<AnimatePresence>
  <div key="wrapper">
    {show && <motion.div exit={{ opacity: 0 }}>Content</motion.div>}
  </div>
</AnimatePresence>

// ✅ Correct — AnimatePresence watches its DIRECT children
<AnimatePresence>
  {show && (
    <motion.div key="content" exit={{ opacity: 0 }}>
      Content
    </motion.div>
  )}
</AnimatePresence>
```

Also check: the `exit` prop is on the same element that has `initial`/`animate`. The `exit` only fires on **unmount** — if the element stays mounted (e.g. CSS `display: none`), it won't fire.

### Layout animation jank

Layout animations can look wrong if:
- A parent has `overflow: hidden` — clip it with `overflow: clip` or animate without layout.
- Fonts are loading — layout shifts during font swap confuse Motion. Use `font-display: block` or preload fonts.
- The element has `position: absolute` or `fixed` — layout animations work best on flow-positioned elements.
- Multiple nested `layout` elements — can compound and look wrong. Add `layoutId` instead for the container.

```tsx
// Debug layout animations:
<motion.div layout layoutId="debug" onLayoutAnimationStart={() => console.log("layout start")} />
```

### Reduced motion: respect it globally

```tsx
// In your root layout or _app:
<MotionConfig reducedMotion="user">
  {children}
</MotionConfig>
```

This is the single easiest accessibility win. Don't skip it.

### AnimatePresence `mode="wait"` for smooth page transitions

When switching between pages or tabs, `mode="wait"` ensures the exit animation completes before the enter animation starts, preventing overlap:

```tsx
<AnimatePresence mode="wait">
  <motion.div key={pathname}>...</motion.div>
</AnimatePresence>
```

### Don't animate `height: auto` without `layout`

```tsx
// ❌ This won't work — "auto" is not a valid animation target
<motion.div animate={{ height: "auto" }} />

// ✅ Use layout prop instead
<motion.div layout style={{ height: isOpen ? "auto" : 0 }} />

// ✅ Or: use a fixed pixel value if you know it
<motion.div animate={{ height: isOpen ? 200 : 0 }} />
```

### Exit animations and conditional rendering

The pattern `{condition && <Component />}` works with AnimatePresence, but a common mistake is unmounting the `AnimatePresence` itself:

```tsx
// ❌ Wrong — AnimatePresence is removed before exit animation runs
{show && (
  <AnimatePresence>
    <motion.div exit={{ opacity: 0 }}>Content</motion.div>
  </AnimatePresence>
)}

// ✅ Correct — AnimatePresence stays mounted, controls child exit
<AnimatePresence>
  {show && <motion.div key="content" exit={{ opacity: 0 }}>Content</motion.div>}
</AnimatePresence>
```

### `initial={false}` to skip mount animation

Sometimes you don't want elements to animate when the page first loads (e.g., a tab panel that starts visible):

```tsx
// Disable initial animation (only animate on subsequent changes)
<motion.div initial={false} animate={{ opacity: isVisible ? 1 : 0 }} />

// On AnimatePresence — disables initial animation for ALL present children
<AnimatePresence initial={false}>
  {items.map(item => <motion.li key={item.id} exit={{ opacity: 0 }}>{item.text}</motion.li>)}
</AnimatePresence>
```

### Framer Motion vs. Motion imports

```tsx
// Both work. Prefer motion/react for new projects.
import { motion } from "framer-motion"   // old (still maintained)
import { motion } from "motion/react"    // new (recommended)

// Do NOT mix both in the same project — they'll fight over the same DOM elements.
```

### Motion values bypass React renders — that's intentional

```tsx
// This does NOT cause re-renders (by design):
const x = useMotionValue(0)
x.set(100)  // updates DOM directly

// To read the value in React render:
const [xValue, setXValue] = useState(0)
useMotionValueEvent(x, "change", setXValue)
```

### Pointer events during drag

When dragging, the element captures all pointer events. To prevent child clicks from interfering:

```tsx
<motion.div
  drag
  style={{ touchAction: "none" }}  // prevents scroll interference on touch devices
>
  <button onClick={...}>This won't accidentally fire during drag</button>
</motion.div>
```

### `LayoutGroup` and unique `layoutId`s

`layoutId` values must be unique within a `LayoutGroup` (and globally if no `LayoutGroup` is used). If you have multiple instances of a component that use `layoutId`, each instance needs a unique prefix:

```tsx
function Tab({ id, isActive }: { id: string; isActive: boolean }) {
  return (
    <div>
      <span>{id}</span>
      {isActive && (
        <motion.div layoutId={`underline-${id}`} />  // prefix with something unique
      )}
    </div>
  )
}
```

### Server-side rendering and `initial`

On the server, Motion cannot know what state the element is in. By default, `initial` is applied during hydration (the client-side takeover), causing a brief flash. Strategies:

```tsx
// Option 1: Match initial to the final state (no animation on first load)
<motion.div initial={{ opacity: 1 }} animate={{ opacity: 1 }} />

// Option 2: Use whileInView (won't fire until the user scrolls to it)
<motion.div
  initial={{ opacity: 0 }}
  whileInView={{ opacity: 1 }}
  viewport={{ once: true }}
/>

// Option 3: Add a tiny delay to run after hydration
<motion.div
  initial={{ opacity: 0 }}
  animate={{ opacity: 1 }}
  transition={{ delay: 0.05 }}
/>
```

---

## 17. Study Path

Follow this sequence to go from zero to confident with Motion animations.

### Stage 1 — Core concepts (Days 1–2)

1. Read Section 3 (motion component) and animate a `div` with `initial`/`animate`/`exit`.
2. Experiment with the `transition` prop — try tween vs. spring, adjust stiffness/damping.
3. Add `whileHover` and `whileTap` to buttons. Build a card with a lift effect.
4. **Build:** An animated landing page hero with a fade-up heading, subtitle, and CTA button with stagger delay.

### Stage 2 — Presence & variants (Days 3–4)

1. Read Section 6 (Variants) — build a staggered list with parent/child propagation.
2. Read Section 7 (AnimatePresence) — build a modal that animates in and out.
3. Combine variants + AnimatePresence for a list where items can be added and removed.
4. **Build:** A to-do list where adding/removing items animates smoothly.

### Stage 3 — Scroll & layout (Days 5–6)

1. Read Section 9 (Scroll) — add `whileInView` to every section of your landing page.
2. Build a scroll progress bar with `useScroll` + `useTransform`.
3. Read Section 8 (Layout) — experiment with the `layout` prop on an accordion.
4. Try `layoutId` for a tab underline indicator.
5. **Build:** A portfolio page with scroll-triggered reveals, a scroll progress bar, and an animated tab interface.

### Stage 4 — Motion values & hooks (Days 7–8)

1. Read Section 10 — use `useMotionValue` + `useTransform` to create a drag-based color changer.
2. Use `useSpring` to build a cursor-follower component.
3. Use `useAnimate` to sequence a multi-step animation imperatively.
4. **Build:** An interactive drag card that changes color, shadow, and rotation based on drag position.

### Stage 5 — Next.js integration (Days 9–10)

1. Read Section 12 — understand the `"use client"` requirement.
2. Create a reusable `<Reveal>` and `<FadeUp>` component.
3. Add `app/template.tsx` for page transition animations.
4. Read Section 13 — add `MotionConfig reducedMotion="user"` to your root layout.
5. Read Section 14 — switch to `LazyMotion` + `domAnimation` to reduce bundle size.
6. **Build:** A full Next.js 16 marketing site with animated page transitions, staggered hero, scroll-triggered sections, and an animated navigation bar.

### Stage 6 — Advanced patterns (Days 11–14)

1. Build a magic-move image gallery with `layoutId`.
2. Build a Reorder list (drag-to-reorder) with `Reorder.Group` and `Reorder.Item`.
3. Animate SVG paths — a loading spinner and a checkmark reveal.
4. Create a parallax hero section using `useScroll` + `useTransform` with `offset`.
5. **Build:** A full-featured product page — animated hero parallax, scroll-linked sticky header opacity, staggered feature cards, animated testimonials carousel, and a slide-in contact drawer.

### Key projects to build (portfolio-worthy)

| Project | Key Motion features used |
|---|---|
| Landing page | FadeUp reveals, stagger, whileInView, page transition |
| Dashboard | AnimatePresence modals, tab indicators (layoutId), data loading skeletons |
| E-commerce PDP | Shared layout (image zoom), drawer cart, hover cards, scroll parallax |
| Portfolio | SVG path drawing on scroll, magnetic buttons, cursor follower, page transitions |
| Kanban board | Drag (drag + dragConstraints), list reorder, AnimatePresence for card removal |

### Reference cheatsheet

```tsx
// Animate on mount
<motion.div initial={{ opacity: 0 }} animate={{ opacity: 1 }} />

// Animate on scroll entry
<motion.div initial={{ opacity: 0 }} whileInView={{ opacity: 1 }} viewport={{ once: true }} />

// Animate on gesture
<motion.button whileHover={{ scale: 1.05 }} whileTap={{ scale: 0.95 }} />

// Animate exit
<AnimatePresence>
  {show && <motion.div key="x" exit={{ opacity: 0 }} />}
</AnimatePresence>

// Stagger children
const container = { hidden: {}, show: { transition: { staggerChildren: 0.1 } } }
const item = { hidden: { y: 20, opacity: 0 }, show: { y: 0, opacity: 1 } }
<motion.ul variants={container} initial="hidden" animate="show">
  <motion.li variants={item} />
</motion.ul>

// Scroll progress
const { scrollYProgress } = useScroll()
<motion.div style={{ scaleX: scrollYProgress }} />

// Derive values
const x = useMotionValue(0)
const opacity = useTransform(x, [-100, 0, 100], [0, 1, 0])

// Imperative sequence
const [scope, animate] = useAnimate()
await animate(".heading", { opacity: 1 })
await animate(".body", { opacity: 1 })

// Reduce motion
<MotionConfig reducedMotion="user">{children}</MotionConfig>

// Bundle size
<LazyMotion features={domAnimation}>
  <m.div animate={{ x: 100 }} />  // m instead of motion
</LazyMotion>
```

---

*This guide was written for Motion v11+ / `motion` package, React 18/19, and Next.js 15/16. The motion.dev documentation is the canonical source for any API changes after August 2025.*
