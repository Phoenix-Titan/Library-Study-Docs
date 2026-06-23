# Tailwind CSS v4 — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I copied a Tailwind class once and it worked" to "I architect a design system with `@theme` tokens, custom variants, and a clean `cn()` pattern in React/Next/shadcn" — without an internet connection. Every concept is explained in prose first (what it is, *why* utility-first CSS exists and what it trades off, when to reach for it, how to use it, best practices), then shown with heavily-commented code and reference tables you can scan. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Tailwind CSS v4 (2026)**. v4 is a major rewrite with a **new high-performance engine (Oxide)** and a **CSS-first configuration model**. The headline changes vs v3:
> - **No `tailwind.config.js` by default.** You configure Tailwind *in your CSS* with the new `@theme` at-rule. A JS config is still loadable via `@config` for migration, but it is no longer the default path.
> - **One import line:** `@import "tailwindcss";` replaces the old `@tailwind base; @tailwind components; @tailwind utilities;` trio.
> - **First-class Vite plugin** (`@tailwindcss/vite`) and a PostCSS plugin (`@tailwindcss/postcss`); the engine auto-detects your content/template files — no `content: []` array to maintain.
> - **Design tokens become real CSS variables.** Every theme value (`--color-*`, `--spacing`, `--font-*`…) is exposed as a `var()` you can use anywhere, including in arbitrary values.
> - **Modern CSS under the hood:** native cascade layers (`@layer`), `oklch()` colors, container queries built in, `@property` for animatable variables.
>
> **All the utility *class names* you write in HTML/JSX are essentially the same as v3** — v4 changes *configuration and the engine*, not the day-to-day vocabulary (a few utilities were renamed; see §15). This guide **pairs with `CSS_GUIDE.md`** in this library: Tailwind utilities are thin, named wrappers over the real CSS properties taught there. When a class maps to a CSS property you want to understand deeply, cross-reference the CSS guide. Where a feature is v4-specific or fast-moving it is flagged with **⚡ Version note**. The author is on **Windows 11**. Confirm details at tailwindcss.com/docs or MDN.

---

## Table of Contents

1. [The Utility-First Philosophy](#1-the-utility-first-philosophy) **[B]**
2. [v4 Setup & How Tailwind Builds Your CSS](#2-v4-setup--how-tailwind-builds-your-css) **[B/I]**
3. [How Utilities Map to CSS](#3-how-utilities-map-to-css) **[B]**
4. [Layout & Display](#4-layout--display) **[B/I]**
5. [Flexbox](#5-flexbox) **[B/I]**
6. [Grid](#6-grid) **[I]**
7. [Spacing (Padding / Margin / Gap)](#7-spacing) **[B]**
8. [Sizing (Width / Height)](#8-sizing) **[B]**
9. [Typography](#9-typography) **[B/I]**
10. [Colors & Backgrounds](#10-colors--backgrounds) **[B/I]**
11. [Borders, Outline & Ring](#11-borders-outline--ring) **[B/I]**
12. [Effects, Filters & Backdrop](#12-effects-filters--backdrop) **[I]**
13. [Transitions, Animation & Transforms](#13-transitions-animation--transforms) **[I]**
14. [Interactivity, Tables, SVG & Accessibility](#14-interactivity-tables-svg--accessibility) **[I]**
15. [Responsive Design — Mobile-First Breakpoints](#15-responsive-design--mobile-first-breakpoints) **[I]**
16. [State Variants — hover/focus/group/peer/has](#16-state-variants--hoverfocusgrouppeerhas) **[I/A]**
17. [Dark Mode](#17-dark-mode) **[I]**
18. [Arbitrary Values & Arbitrary Properties](#18-arbitrary-values--arbitrary-properties) **[I]**
19. [Theme Customization with `@theme`](#19-theme-customization-with-theme) **[I/A]**
20. [`@apply`, Reusing Styles & the `cn()` Pattern](#20-apply-reusing-styles--the-cn-pattern) **[I/A]**
21. [Plugins (Typography, Forms) & Framework Integration](#21-plugins-typography-forms--framework-integration) **[A]**
22. [Gotchas](#22-gotchas) **[I/A]**
23. [Most-Used Combos (copy-paste)](#23-most-used-combos) **[B/I]**
24. [Study Path & Build-to-Learn Projects](#24-study-path--build-to-learn-projects)

---

## 1. The Utility-First Philosophy

### 1.1 What "utility-first" actually means **[B]**

Traditional CSS (taught in `CSS_GUIDE.md`) is **semantic / component-based**: you invent a meaningful class name for a thing — `.card`, `.btn-primary` — and then write a separate block of CSS describing how that thing looks. Your markup says *what* something is; a stylesheet elsewhere says *how it looks*.

**Utility-first** flips this. Instead of inventing names, you compose many tiny, single-purpose classes — each mapping to essentially one CSS declaration — directly in your markup:

```html
<!-- Component-based (traditional): one name, styles live in a .css file -->
<div class="card">…</div>
<!-- somewhere in styles.css:
     .card { display:flex; gap:1rem; padding:1rem; border-radius:.75rem;
             border:1px solid #e5e7eb; background:#fff; } -->

<!-- Utility-first (Tailwind): the styles ARE the class list -->
<div class="flex gap-4 p-4 rounded-xl border border-gray-200 bg-white">…</div>
```

Each Tailwind class is a **utility**: `p-4` is "padding: 1rem", `flex` is "display: flex", `text-center` is "text-align: center". You build a design by *stacking* utilities, the way you'd build a sentence from words. There is no `card` class to name, no CSS file to switch to, no question of "where is this styled."

### 1.2 The logic — *why* would anyone do this? **[B]**

Utility-first looks strange at first ("isn't that just inline styles with extra steps?"). It exists to solve real, chronic problems with hand-written CSS at scale:

- **Naming is hard and wasteful.** A huge fraction of CSS effort goes into inventing class names (`.card__header--featured`) that carry no styling information. Utilities skip naming entirely — you describe the look directly.
- **CSS only grows.** In a big component-based codebase, nobody dares delete a CSS rule because they can't prove it's unused. Stylesheets accumulate dead code forever. With Tailwind, the generated CSS is derived *from the classes actually present in your markup*, so unused styles simply never get generated — your CSS stays small and stops growing linearly with the project.
- **Local reasoning.** With utilities, everything affecting an element is visible *on that element*. You never have to hunt across files, fight the cascade, or fear that editing `.card` breaks some other page using `.card`. Changes are local and safe.
- **A constrained design system by default.** `p-4`, `p-6`, `p-8` come from a fixed spacing scale; colors come from a fixed palette. You're nudged toward consistent, harmonious values instead of `padding: 13px` here and `15px` there. The framework *is* a lightweight design system.
- **It's not inline styles.** Inline `style=""` can't do hover, focus, media queries, dark mode, or pseudo-elements, and shares no constraints. Tailwind utilities can (`hover:`, `md:`, `dark:`, `before:`), come from a scale, and are deduplicated in the final CSS. They're the ergonomic opposite of inline styles despite the surface resemblance.

### 1.3 The trade-offs — be honest **[B/I]**

Utility-first is a genuine engineering trade-off, not a free win:

| Pro | Con |
|---|---|
| No naming; faster authoring once fluent | Long, "ugly" class lists; markup looks busy |
| CSS stays tiny (only used utilities ship) | Requires a build step / the Tailwind engine |
| Local reasoning; safe to change/delete | A real learning curve (a new vocabulary) |
| Built-in design constraints (scale, palette) | Repetition across similar elements (mitigated by components/loops) |
| Responsive/state/dark variants inline | Logic and presentation mixed in markup (some dislike this) |
| No specificity wars (flat, single-class-ish) | Hard to read at a glance until you know the names |

**The key mental shift:** repetition of utility *strings* is not duplication of *concept*. When you have a real reusable thing (a button used 50 times), you don't copy 12 classes 50 times — you make a **component** (a React/Vue component, a loop, or rarely an `@apply` class, §20) and write the utilities once. Tailwind is for the *leaves* of your UI; components handle reuse.

> **When NOT to choose Tailwind:** tiny static pages where a build step is overkill, email HTML (no support), or teams that strongly value framework-agnostic semantic CSS. For most modern app/component work (React, Next, Vue, shadcn ecosystems) it's an excellent default. The two approaches also blend: many teams use Tailwind for layout/spacing and extract complex widgets into components.

---

## 2. v4 Setup & How Tailwind Builds Your CSS

### 2.1 The mental model: a build-time engine **[B/I]**

Tailwind is **not a CSS file you ship** and it is **not a runtime library**. It is a **build-time engine** that:

1. **Scans your source files** (`.html`, `.jsx`, `.tsx`, `.vue`, …) for strings that look like class names.
2. For every utility it finds (`flex`, `p-4`, `md:grid-cols-3`, `hover:bg-blue-600`, even arbitrary `w-[327px]`), it **generates the corresponding CSS rule** on demand. This on-demand generation is called **JIT (Just-In-Time)**, and it's why arbitrary values work and why your output only contains what you used.
3. **Outputs a single small CSS file** that you link in the page.

Because generation is driven by *what appears in your markup*, two rules follow that trip up beginners (see §22): class names must appear as **complete static strings** (you can't build `` `text-${color}-500` `` dynamically), and editing markup re-triggers a build.

### 2.2 Installing with Vite (recommended) **[B]**

The Vite plugin is the fastest, simplest path for modern apps (React, Vue, Svelte, vanilla Vite, and the basis of many Next setups).

```bash
# 1) Install Tailwind v4 and the Vite plugin
npm install tailwindcss @tailwindcss/vite
```

```js
// 2) vite.config.js — register the plugin. That's the whole config.
import { defineConfig } from "vite";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [
    tailwindcss(),   // scans your files & generates CSS automatically
  ],
});
```

```css
/* 3) src/style.css — ONE line pulls in Tailwind's base, components & utilities.
      This replaces v3's three @tailwind directives. */
@import "tailwindcss";
```

```js
// 4) Import that CSS once (e.g. in main.jsx / main.ts) and you're done.
import "./style.css";
```

That's it — no `content` array, no `tailwind.config.js`. The engine auto-detects template files in your project and watches them.

### 2.3 Installing with PostCSS (framework-agnostic) **[I]**

If your toolchain uses PostCSS directly (some bundlers, some Next setups), use the PostCSS plugin instead:

```bash
npm install tailwindcss @tailwindcss/postcss
```

```js
// postcss.config.mjs
export default {
  plugins: {
    "@tailwindcss/postcss": {},   // v4's PostCSS plugin (note: not "tailwindcss")
  },
};
```

```css
/* your main CSS, same as before */
@import "tailwindcss";
```

> **⚡ Version note:** In v3 the PostCSS plugin was just `tailwindcss` and you needed `autoprefixer` + `postcss-import` alongside it. In **v4 those are built in** — the single `@tailwindcss/postcss` plugin handles imports and vendor prefixing for you.

### 2.4 The Play CDN (prototyping only) **[B]**

For a quick experiment or a CodePen-style demo with no build step, there's a browser script. **Never use this in production** — it ships the whole engine and generates CSS at runtime, which is slow.

```html
<!-- Prototyping ONLY. No build tool, but heavy and not for production. -->
<script src="https://cdn.tailwindcss.com"></script>
```

### 2.5 What `@import "tailwindcss"` actually pulls in **[I]**

That one line expands into three concerns, organized as **cascade layers** (the native CSS `@layer`, see `CSS_GUIDE.md` §3.6) so their priority is deterministic:

```css
/* Conceptually, @import "tailwindcss"; gives you these layers in this order: */
@layer theme, base, components, utilities;
/* - theme      → your design tokens become CSS variables (from @theme)
   - base       → "Preflight": a modern reset (margins removed, box-sizing:border-box,
                  headings unstyled, images block-level, etc.)
   - components → a place for component classes / plugin components
   - utilities  → the millions of utility classes (these win, being the last layer) */
```

Two practical consequences of the layering: **Preflight** means you start from a clean slate (no default heading sizes, no list bullets — you opt into everything), and **utilities live in the last layer**, so a utility reliably overrides component/base styles without specificity tricks. You can disable or reorder pieces (§19.6) if you ever need to.

### 2.6 First end-to-end example **[B]**

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <!-- REQUIRED for mobile breakpoints to behave (see CSS_GUIDE §13.1). -->
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <link rel="stylesheet" href="/dist/style.css" /> <!-- the built output -->
  </head>
  <body class="bg-gray-50 text-gray-900">
    <main class="mx-auto max-w-2xl p-8">
      <h1 class="text-3xl font-bold">Hello, Tailwind v4</h1>
      <p class="mt-2 text-gray-600">Composed entirely from utilities.</p>
    </main>
  </body>
</html>
```

---

## 3. How Utilities Map to CSS

### 3.1 The naming convention **[B]**

Most utilities follow a predictable pattern, so you can often *guess* the class. Internalizing the pattern is faster than memorizing thousands of names.

```
{property-abbreviation}-{value}
```

| Utility | Generated CSS | Pattern |
|---|---|---|
| `p-4` | `padding: 1rem;` | `p` = padding, `4` = scale step (4 × 0.25rem) |
| `mt-2` | `margin-top: 0.5rem;` | `mt` = margin-top |
| `text-center` | `text-align: center;` | `text-{keyword}` |
| `text-lg` | `font-size: 1.125rem; line-height:…` | `text-{size}` |
| `text-red-500` | `color: oklch(…);` | `text-{color}-{shade}` |
| `bg-blue-600` | `background-color: oklch(…);` | `bg-{color}-{shade}` |
| `flex` | `display: flex;` | bare keyword |
| `justify-between` | `justify-content: space-between;` | `justify-{value}` |
| `rounded-lg` | `border-radius: 0.5rem;` | `rounded-{size}` |
| `w-1/2` | `width: 50%;` | fractions allowed |
| `gap-6` | `gap: 1.5rem;` | scale step |

Notice `text-` is overloaded (size, color, alignment) — Tailwind disambiguates by the *value*: `text-lg` (size), `text-red-500` (color), `text-center` (alignment). You learn these quickly.

### 3.2 The spacing scale — the unit that underpins everything **[B]**

Numeric utilities (`p-`, `m-`, `gap-`, `w-`, `h-`, `top-`, `space-`, …) share one **spacing scale**. The base unit is `0.25rem` (4px at the default root size), and the number is a multiplier:

```
0    → 0
0.5  → 0.125rem (2px)
1    → 0.25rem  (4px)
2    → 0.5rem   (8px)
3    → 0.75rem  (12px)
4    → 1rem     (16px)   ← p-4 = 16px padding
6    → 1.5rem   (24px)
8    → 2rem     (32px)
10   → 2.5rem   (40px)
12   → 3rem     (48px)
16   → 4rem     (64px)
…    → continues (20, 24, 32, 40, 48, 56, 64, 80, 96)
```

**Why a scale?** Constraining spacing to multiples of 4px enforces visual rhythm and consistency. You stop agonizing over `13px` vs `15px` — the scale makes "near misses" impossible, which is half the value of a design system. In v4 the whole scale derives from a single `--spacing` token (default `0.25rem`), so you can rescale everything at once (§19).

### 3.3 Negative values, fractions, and `*` (every-direction) **[B]**

```html
<!-- Negative: prefix the utility with a minus -->
<div class="-mt-4">          <!-- margin-top: -1rem -->
<div class="-translate-x-2"> <!-- transform: translateX(-0.5rem) -->

<!-- Fractions for sizing -->
<div class="w-1/2"></div>    <!-- width: 50% -->
<div class="w-2/3"></div>    <!-- width: 66.666% -->

<!-- Axis & side suffixes are extremely common: -->
<!-- x = left+right, y = top+bottom; t/r/b/l = single side -->
<div class="px-6 py-2"></div>     <!-- padding-left/right:1.5rem, top/bottom:0.5rem -->
<div class="mt-2 mb-4"></div>     <!-- margin-top:0.5rem; margin-bottom:1rem -->
```

### 3.4 The opacity-via-slash shorthand **[B/I]**

Any color utility accepts a `/N` alpha suffix — one of Tailwind's most-loved conveniences:

```html
<div class="bg-black/50"></div>   <!-- background: rgb(0 0 0 / 50%) -->
<div class="text-white/80"></div> <!-- color with 80% alpha -->
<div class="border-blue-500/25"></div>
```

This compiles to a modern `oklch(... / <alpha>)` (or `rgb(... / <alpha>)`) — no need for a separate `opacity` utility on the element, which would also fade children (see `CSS_GUIDE.md` §6.4).

---

## 4. Layout & Display

**What this category is:** the utilities that set an element's `display` type, its `position`, how overflow is handled, and how replaced elements (images/video) fit their box. This is the structural skeleton you reach for first. For the underlying theory (normal flow, stacking contexts, the box model) read `CSS_GUIDE.md` §4, §9, §12 — Tailwind doesn't change *how* these work, it just gives them short names.

**The logic / when to use:** before any alignment or spacing, decide the *display model*. Is this a block, a flex container, or a grid? Is something pulled out of flow with `absolute`/`fixed`/`sticky`? Get the display and position right and the rest follows.

**Best practices:** prefer `flex`/`grid` + `gap` for layout over floats and margins; use `sticky top-0` for headers; reach for `relative` on a parent before positioning a child `absolute` (so the child anchors to it); add `min-w-0`/`overflow-hidden` when flex children refuse to shrink (the classic `min-width:auto` trap, §22).

```html
<!-- Display models -->
<div class="block">…</div>          <!-- display: block -->
<div class="hidden md:block">…</div><!-- hide on mobile, show from md up -->
<div class="flex">…</div>           <!-- display: flex (see §5) -->
<div class="grid">…</div>           <!-- display: grid (see §6) -->

<!-- Positioning: anchor an absolute child to a relative parent -->
<div class="relative">
  <span class="absolute top-2 right-2">×</span> <!-- pinned to top-right -->
</div>

<!-- Sticky header that stays at the top while scrolling -->
<header class="sticky top-0 z-50 bg-white/80 backdrop-blur">…</header>

<!-- Full-cover overlay: inset-0 = top/right/bottom/left all 0 -->
<div class="absolute inset-0 bg-black/40"></div>

<!-- Aspect ratio + object-fit for a responsive media box -->
<img class="aspect-video w-full object-cover" src="/hero.jpg" alt="" />

<!-- One-line truncation with ellipsis -->
<p class="truncate">A very long single line of text that will be cut…</p>
```

| Class | CSS / Description |
|---|---|
| `block` `inline-block` `inline` | `display` type |
| `flex` `inline-flex` | Flex container (§5) |
| `grid` `inline-grid` | Grid container (§6) |
| `hidden` | `display: none` |
| `contents` | Box disappears, children remain in flow |
| `container` | `width:100%` + max-width steps per breakpoint |
| `box-border` `box-content` | `box-sizing` model (border-box is the default via Preflight) |
| `static` `relative` `absolute` `fixed` `sticky` | `position` |
| `inset-0` | `top/right/bottom/left: 0` (full overlay) |
| `inset-x-0` `inset-y-0` | Both horizontal / vertical edges |
| `top-0` `right-4` `bottom-2` `left-1/2` | Single edge offset (scale, fraction, or `-` negative) |
| `z-0` `z-10` `z-50` `z-[999]` | `z-index` (stacking; needs non-static position) |
| `isolate` | `isolation: isolate` — new stacking context (§22) |
| `overflow-auto` `overflow-hidden` `overflow-scroll` `overflow-clip` | Overflow handling |
| `overflow-x-auto` `overflow-y-hidden` | Per-axis overflow |
| `truncate` | `overflow:hidden; text-overflow:ellipsis; white-space:nowrap` |
| `float-left` `float-right` `clear-both` | Floats (rarely needed now) |
| `object-cover` `object-contain` `object-fill` | How img/video fills its box |
| `object-center` `object-top` `object-bottom` | Focal position of the media |
| `aspect-square` `aspect-video` `aspect-[4/3]` | `aspect-ratio` |

---

## 5. Flexbox

**What it is:** utilities for CSS Flexbox — one-dimensional layout (a row *or* a column) with powerful alignment and space distribution. For the full conceptual model (main vs cross axis, grow/shrink/basis) see `CSS_GUIDE.md` §10.

**The logic / when to use:** Flexbox is the right tool when you're arranging items along **one axis** — a navbar, a toolbar, a row of buttons, a centered hero, a card's internal stack. Reach for Grid (§6) instead when you need **two-dimensional** control (rows *and* columns aligned together).

**Best practices:** use `gap-*` for spacing between items (cleaner than margins, no collapsing); the canonical centering recipe is `flex items-center justify-center`; add `flex-wrap` so rows don't overflow on small screens; put `shrink-0` on things that must keep their size (icons, avatars) and `min-w-0` on the flexible text child so it can truncate.

```html
<!-- Navbar: logo left, links right. justify-between pushes them apart. -->
<nav class="flex items-center justify-between p-4">
  <span class="font-bold">Logo</span>
  <div class="flex items-center gap-6">       <!-- gap = space BETWEEN links -->
    <a href="#">Home</a>
    <a href="#">About</a>
  </div>
</nav>

<!-- Perfect centering (both axes) -->
<div class="flex min-h-screen items-center justify-center">
  <div>Centered both ways</div>
</div>

<!-- A flexible text item that can truncate next to a fixed icon -->
<div class="flex items-center gap-3">
  <img class="size-10 shrink-0 rounded-full" src="/avatar.jpg" alt="" /> <!-- keep size -->
  <p class="min-w-0 truncate">A long name that should ellipsize, not push…</p>
</div>

<!-- Responsive direction: stack on mobile, row on large screens -->
<div class="flex flex-col gap-8 lg:flex-row">…</div>
```

| Class | CSS / Description |
|---|---|
| `flex-row` `flex-col` | Main-axis direction |
| `flex-row-reverse` `flex-col-reverse` | Reversed direction |
| `flex-wrap` `flex-nowrap` `flex-wrap-reverse` | Wrapping behavior |
| `justify-start` `justify-center` `justify-end` | Align on the **main** axis |
| `justify-between` `justify-around` `justify-evenly` | Distribute space (main axis) |
| `items-start` `items-center` `items-end` | Align on the **cross** axis |
| `items-stretch` `items-baseline` | Stretch / baseline align |
| `content-center` `content-between` | Align *wrapped lines* (multi-row) |
| `self-start` `self-center` `self-end` `self-auto` | Override cross-axis align for one item |
| `flex-1` | `flex: 1 1 0%` — grow & shrink to fill |
| `flex-auto` `flex-initial` `flex-none` | Common `flex` shorthands |
| `grow` `grow-0` | Allow / prevent growing |
| `shrink` `shrink-0` | Allow / prevent shrinking |
| `basis-1/2` `basis-64` `basis-auto` | `flex-basis` |
| `order-1` `order-first` `order-last` `order-none` | Visual order |
| `gap-4` `gap-x-2` `gap-y-6` | Space between items (works in flex *and* grid) |

> **Centering, the canonical recipe:** `flex items-center justify-center`. Memorize it — you'll use it constantly.

---

## 6. Grid

**What it is:** utilities for CSS Grid — two-dimensional layout where you define columns and/or rows and place items into the resulting cells. See `CSS_GUIDE.md` §11 for the deep model (tracks, `fr`, `minmax`, `auto-fit`, areas, subgrid).

**The logic / when to use:** Grid shines for **2D layouts** — card galleries, dashboards, page skeletons (header/sidebar/main/footer), anything where you want columns *and* rows to line up. The mantra is "**grid for the page-level structure, flex for the components inside**."

**Best practices:** the single most common pattern is a responsive card grid — start `grid-cols-1`, bump columns up at breakpoints. For *self-sizing* responsive grids without media queries, use an arbitrary `repeat(auto-fit, minmax(…))` track. Use `gap-*` for gutters; use `col-span-*`/`row-span-*` to make a single item occupy more cells.

```html
<!-- Responsive card grid: 1 col on mobile → 2 at sm → 3 at lg -->
<div class="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3">
  <div class="rounded-lg border p-4">Card</div>
  <div class="rounded-lg border p-4">Card</div>
  <div class="rounded-lg border p-4">Card</div>
</div>

<!-- Auto-fit grid: as many 250px-min columns as fit, no media queries needed -->
<div class="grid gap-4 grid-cols-[repeat(auto-fit,minmax(250px,1fr))]">…</div>

<!-- Explicit page skeleton: a fixed sidebar + flexible main -->
<div class="grid min-h-screen grid-cols-[240px_1fr]">
  <aside class="border-r">Sidebar</aside>
  <main class="p-6">Content</main>
</div>

<!-- An item spanning two columns -->
<div class="grid grid-cols-3 gap-4">
  <div class="col-span-2">Wide (2 cols)</div>
  <div>Narrow</div>
</div>
```

| Class | CSS / Description |
|---|---|
| `grid-cols-3` | 3 equal columns (`repeat(3, minmax(0,1fr))`) |
| `grid-cols-none` `grid-cols-subgrid` | No template / inherit parent tracks (subgrid) |
| `grid-cols-[200px_1fr]` | Custom column track sizes (arbitrary) |
| `grid-rows-4` `grid-rows-[…]` | Rows |
| `col-span-2` `row-span-3` `col-span-full` | Item spans multiple cells |
| `col-start-2` `col-end-4` `row-start-1` | Explicit line-based placement |
| `auto-cols-fr` `auto-rows-min` `auto-rows-auto` | Implicit (auto) track sizing |
| `grid-flow-row` `grid-flow-col` `grid-flow-dense` | Auto-placement direction/packing |
| `place-items-center` | Shorthand for `align-items` + `justify-items` |
| `place-content-between` | Align + justify the whole grid |
| `place-self-end` | Per-item place-self |
| `gap-6` `gap-x-4 gap-y-8` | Gutters |

---

## 7. Spacing

**What it is:** padding (`p-*`), margin (`m-*`), gap (`gap-*`), and the "between-children" helper (`space-*`). All draw from the one spacing scale (§3.2).

**The logic / when to use which:** think of it as three jobs. **Padding** is space *inside* a box (between border and content). **Margin** is space *outside* a box (pushing neighbors away). **Gap** is space *between* the children of a flex/grid container — and it's the modern default for laying out lists/rows because it doesn't collapse and adds no edge spacing. Reach for `gap` first; use margin for one-off nudges; use padding for internal breathing room.

**Best practices:** prefer `gap-*` over `space-*` (gap is cleaner and works everywhere flex/grid does); `mx-auto` centers a block with a max-width; remember margins between block siblings *collapse* (CSS_GUIDE §4.4) — flex/grid + gap sidesteps that entirely; logical `ps-*`/`pe-*` are RTL-aware.

```html
<!-- Padding inside a card; gap between stacked children -->
<div class="flex flex-col gap-4 p-6 rounded-xl border">
  <h3 class="font-semibold">Title</h3>
  <p class="text-gray-600">Body text with breathing room.</p>
</div>

<!-- Center a constrained block horizontally -->
<div class="mx-auto max-w-4xl px-4">…</div>

<!-- space-y adds margin BETWEEN children (no gap on the ends). Prefer gap in flex/grid. -->
<ul class="space-y-2">
  <li>One</li><li>Two</li><li>Three</li>
</ul>

<!-- Negative margin to pull an element outward -->
<div class="-mt-8">Overlaps the section above</div>
```

| Class | CSS / Description |
|---|---|
| `p-4` | Padding, all sides |
| `px-6` `py-2` | Padding horizontal / vertical |
| `pt-2 pr-4 pb-2 pl-4` | Padding per side |
| `ps-4` `pe-4` | Padding inline-start / -end (RTL-aware logical) |
| `m-4` `mx-auto` `my-8` | Margin (`mx-auto` centers a block) |
| `mt-2 mr-4 mb-2 ml-4` | Margin per side |
| `-mt-4` | Negative margin |
| `space-x-4` `space-y-2` | Margin *between* children only |
| `space-x-reverse` | Adjust between-spacing for reversed flex |
| `gap-4` `gap-x-2 gap-y-6` | **Preferred** spacing inside flex/grid |

---

## 8. Sizing

**What it is:** width (`w-*`), height (`h-*`), their min/max variants, and the combined `size-*` (width AND height).

**The logic / when to use:** sizes come from the spacing scale (`w-64` = 16rem), plus fractions (`w-1/2`), content keywords (`w-fit`, `w-max`, `w-min`), and viewport/percent keywords (`w-full`, `w-screen`, `h-dvh`). **`max-w-*` is the responsive workhorse** — set a max width so content never gets uncomfortably wide, while `w-full` lets it shrink on small screens. Use `min-h-screen` for full-height page wrappers and heroes.

**Best practices:** prefer `max-w-* + mx-auto` for readable content columns (`max-w-prose` ≈ 65ch); use `size-*` for square things (icons, avatars); on mobile prefer `min-h-dvh` over `min-h-screen` so toolbars don't clip content (CSS_GUIDE §5.3); add `min-w-0` to flex children that must truncate.

```html
<!-- Responsive content column: full width, capped, centered -->
<article class="mx-auto w-full max-w-prose px-4">…</article>

<!-- Square avatar via size-* (sets width AND height) -->
<img class="size-12 rounded-full object-cover" src="/me.jpg" alt="" />

<!-- Full-viewport-height hero (dvh handles mobile toolbars) -->
<section class="min-h-dvh flex items-center justify-center">…</section>

<!-- Two columns: fixed sidebar, fluid main, via fractions -->
<div class="flex">
  <aside class="w-64 shrink-0">Sidebar</aside>
  <main class="w-full">Main</main>
</div>
```

| Class | CSS / Description |
|---|---|
| `w-4` `w-64` `w-px` | Fixed width (scale / 1px) |
| `w-full` `w-screen` `w-svw` `w-dvw` | 100% / viewport widths |
| `w-1/2` `w-1/3` `w-2/3` | Fractional width |
| `w-min` `w-max` `w-fit` | Content-based widths |
| `max-w-md` `max-w-7xl` `max-w-prose` `max-w-none` | Max width (content caps) |
| `min-w-0` `min-w-full` | Min width (`min-w-0` lets flex kids shrink) |
| `h-12` `h-full` `h-screen` `h-dvh` | Heights |
| `min-h-screen` `min-h-dvh` | Minimum full-viewport height |
| `max-h-96` `max-h-screen` | Max height |
| `size-10` `size-full` | Width **and** height together |

---

## 9. Typography

**What it is:** font size/weight/family, line-height (`leading-*`), letter-spacing (`tracking-*`), alignment, decoration, wrapping/overflow, and lists. (The `prose` classes from the Typography *plugin* are covered in §21.) Theory lives in `CSS_GUIDE.md` §7.

**The logic / when to use:** `text-{size}` utilities bundle a sensible `font-size` *and* `line-height` together (e.g. `text-lg` ≈ 1.125rem with comfortable leading), so you usually set one class and get good vertical rhythm for free. Override leading/tracking only when the default doesn't fit a specific design.

**Best practices:** set `font-family` once high up (on `<body>` via a base layer or just inherit); use `text-balance` on headings and `text-pretty` on body copy for nicer line breaks; clamp multi-line overflow with `line-clamp-N`; `tracking-tight` reads better on large headings; respect the user — don't lock font sizes below ~14px for body text.

```html
<!-- Heading with balanced wrapping and tight tracking -->
<h1 class="text-4xl font-bold tracking-tight text-balance sm:text-6xl">
  A headline that wraps evenly across lines
</h1>

<!-- Body copy: readable size, relaxed leading, pretty wrapping -->
<p class="text-base/relaxed text-gray-600 text-pretty">…</p>
<!-- ↑ text-base/relaxed = font-size base WITH line-height "relaxed" (slash shorthand) -->

<!-- Clamp a description to 3 lines with an ellipsis -->
<p class="line-clamp-3">A long product description that gets cut after three lines…</p>

<!-- Uppercase label with wide tracking -->
<span class="text-xs font-semibold uppercase tracking-wide text-gray-500">Label</span>
```

| Class | CSS / Description |
|---|---|
| `text-xs` … `text-base` … `text-9xl` | Font size (bundles a default line-height) |
| `text-base/relaxed` `text-lg/7` | Font size **with** explicit line-height (slash) |
| `font-thin` … `font-medium` `font-semibold` `font-bold` `font-black` | Font weight |
| `italic` `not-italic` | Font style |
| `font-sans` `font-serif` `font-mono` | Font family (mapped to `--font-*` tokens) |
| `text-left` `text-center` `text-right` `text-justify` | Text alignment |
| `leading-none` `leading-tight` `leading-relaxed` `leading-7` | Line height |
| `tracking-tight` `tracking-normal` `tracking-wide` | Letter spacing |
| `uppercase` `lowercase` `capitalize` `normal-case` | Casing |
| `underline` `line-through` `no-underline` `overline` | Decoration |
| `decoration-2` `decoration-dotted` `underline-offset-4` | Decoration style/offset |
| `truncate` `text-ellipsis` `line-clamp-3` | Overflow handling |
| `whitespace-nowrap` `whitespace-pre` `whitespace-normal` | Whitespace |
| `break-words` `break-all` `text-balance` `text-pretty` | Wrapping |
| `list-disc` `list-decimal` `list-none` `list-inside` | List markers |
| `indent-8` `antialiased` | First-line indent / font smoothing |

---

## 10. Colors & Backgrounds

**What it is:** text/background/border/SVG colors drawn from Tailwind's palette, the `/alpha` shorthand, gradients, and background image/size/position utilities.

**The logic / when to use:** colors follow `{utility}-{color}-{shade}`, where shade runs `50` (lightest) → `950` (darkest). `bg-blue-600` is a mid-dark blue background; `text-gray-500` is muted body text. **In v4 the default palette is authored in `oklch()`** (perceptually uniform, wider gamut than v3's sRGB hex), so tints/shades look more even and gradients are smoother. Every color is also a CSS variable (`--color-blue-600`), usable in arbitrary values.

**Best practices:** lean on the numbered scale for consistency (don't reach for arbitrary hex unless a brand color isn't in the palette — and if it recurs, add it to `@theme`, §19); use `/alpha` for overlays and translucent surfaces; for gradients use `bg-linear-to-r` (v4) with `from-`/`via-`/`to-` stops; layer a dark gradient over an image for legible hero text.

```html
<!-- Text, background, border colors from the palette -->
<div class="bg-slate-900 text-slate-100 border border-slate-700 p-4">…</div>

<!-- Translucent surface via /alpha (overlay that doesn't fade children) -->
<div class="bg-black/60 text-white p-6">Modal backdrop content</div>

<!-- Linear gradient (v4 syntax) with three stops -->
<div class="bg-linear-to-r from-indigo-500 via-purple-500 to-pink-500 h-32"></div>

<!-- Hero image with a readable dark gradient overlay (layered backgrounds) -->
<div class="relative h-80 bg-[url('/hero.jpg')] bg-cover bg-center">
  <div class="absolute inset-0 bg-linear-to-t from-black/70 to-transparent"></div>
  <h2 class="relative p-6 text-white">Legible over the photo</h2>
</div>
```

| Class | CSS / Description |
|---|---|
| `text-red-500` | Text color |
| `bg-slate-900` | Background color |
| `bg-white/80` | Background with 80% alpha (slash) |
| `bg-transparent` `bg-current` `bg-inherit` | Keyword colors |
| `border-gray-200` | Border color |
| `fill-blue-500` `stroke-red-500` | SVG fill / stroke color |
| `placeholder-gray-400` | Placeholder text color |
| `caret-blue-500` `accent-indigo-600` | Caret / native control accent |
| `bg-linear-to-r` `bg-linear-to-tr` | Linear gradient direction (**v4** name) |
| `bg-radial` `bg-conic` | Radial / conic gradient (**v4**) |
| `from-purple-500` `via-pink-500` `to-red-500` | Gradient color stops |
| `from-10%` `to-90%` | Gradient stop positions |
| `bg-cover` `bg-contain` | Background sizing |
| `bg-center` `bg-top` `bg-no-repeat` | Background position / repeat |
| `bg-fixed` `bg-local` | Background attachment |
| `bg-[url('/hero.jpg')]` | Arbitrary background image |

> **⚡ Version note — gradient utility rename:** v3 used `bg-gradient-to-r`. **v4 renames it to `bg-linear-to-r`** (and adds `bg-radial`, `bg-conic`). If you're porting v3 markup, update these. The `from-`/`via-`/`to-` stop classes are unchanged.

Default palette families: `slate, gray, zinc, neutral, stone, red, orange, amber, yellow, lime, green, emerald, teal, cyan, sky, blue, indigo, violet, purple, fuchsia, pink, rose` — each with shades `50–950`.

---

## 11. Borders, Outline & Ring

**What it is:** border width/style/color, border-radius (`rounded-*`), dividers between children (`divide-*`), outlines, and the **ring** (a box-shadow-based outline ideal for focus states).

**The logic / when to use:** borders define a box's edge; `rounded-*` softens corners; `divide-*` draws lines *between* list children without you adding borders to each. The **ring** is Tailwind-specific and important: it's a configurable box-shadow that surrounds an element without affecting layout — perfect for focus rings and selection highlights, because (unlike a border) adding it doesn't shift anything.

**Best practices:** use `focus-visible:ring-2 ring-offset-2` for accessible keyboard focus (the offset creates a gap so the ring reads clearly); pair `outline-none` with a *replacement* ring — never remove focus styling with nothing behind it (accessibility); `rounded-full` makes pills and circles; `divide-y` is the clean way to separate stacked rows.

> **⚡ Version note — `ring` width changed:** in v3, bare `ring` meant a **3px** ring. In **v4, bare `ring` is 1px** (consistent with `border`); use `ring-2`, `ring-4`, etc. for thicker rings. Update v3 markup that relied on `ring` being 3px.

```html
<!-- Card with subtle border and large radius -->
<div class="rounded-xl border border-gray-200 p-6">…</div>

<!-- Accessible focus ring (doesn't shift layout); offset adds a gap -->
<button class="rounded-lg bg-indigo-600 px-4 py-2 text-white
               outline-none focus-visible:ring-2 focus-visible:ring-indigo-500
               focus-visible:ring-offset-2">
  Save
</button>

<!-- Dividers between list items, no per-item borders -->
<ul class="divide-y divide-gray-200">
  <li class="py-3">First</li>
  <li class="py-3">Second</li>
</ul>

<!-- Pill / circular avatar -->
<span class="rounded-full bg-green-100 px-3 py-1 text-green-800">Active</span>
```

| Class | CSS / Description |
|---|---|
| `border` | 1px border, all sides |
| `border-2` `border-4` `border-8` `border-0` | Border width |
| `border-t` `border-x` `border-b-2` | Per-side width |
| `border-dashed` `border-dotted` `border-double` | Border style |
| `rounded` `rounded-md` `rounded-lg` `rounded-xl` `rounded-2xl` `rounded-full` | Radius |
| `rounded-t-lg` `rounded-tl-xl` | Radius per side / corner |
| `divide-y` `divide-x-2` | Borders *between* children |
| `divide-gray-200` | Divider color |
| `outline` `outline-2` `outline-offset-2` `outline-none` | Outline (and removal) |
| `ring` `ring-2` `ring-4` `ring-inset` | Ring (box-shadow outline) — **v4 `ring` = 1px** |
| `ring-blue-500` `ring-offset-2` `ring-offset-white` | Ring color & offset |

---

## 12. Effects, Filters & Backdrop

**What it is:** shadows (`shadow-*`, `drop-shadow-*`), element opacity, blend modes, CSS filters (blur, brightness, grayscale…), and **backdrop** filters (which affect what's *behind* a translucent element — the frosted-glass effect).

**The logic / when to use:** `shadow-*` lifts cards/menus off the page to convey elevation. `drop-shadow-*` (a filter) follows the *shape* of transparent images/SVGs, where a box `shadow` would just draw a rectangle. **Backdrop filters** are how you build "glassmorphism" — a translucent panel that blurs the content scrolling behind it (e.g. a sticky nav).

**Best practices:** use shadows sparingly and consistently (a couple of elevation levels, not ten); animate `shadow` on hover for tactile cards (`transition-shadow hover:shadow-md`); for glass, combine a translucent background + `backdrop-blur` + a faint border; remember backdrop filters can be GPU-heavy on large areas.

```html
<!-- Elevated card that lifts on hover -->
<div class="rounded-xl bg-white p-6 shadow-sm transition-shadow hover:shadow-lg">…</div>

<!-- drop-shadow on a transparent logo (follows the shape, not a box) -->
<img class="drop-shadow-md" src="/logo.png" alt="" />

<!-- Frosted-glass sticky bar (glassmorphism) -->
<header class="sticky top-0 border-b border-white/20 bg-white/30 backdrop-blur-md">…</header>

<!-- Desaturate an image until hovered -->
<img class="grayscale transition hover:grayscale-0" src="/photo.jpg" alt="" />
```

| Class | CSS / Description |
|---|---|
| `shadow-sm` `shadow` `shadow-md` `shadow-lg` `shadow-xl` `shadow-2xl` `shadow-none` | Box shadow (elevation) |
| `shadow-blue-500/50` | Colored shadow |
| `inset-shadow-sm` | Inner shadow (**v4**) |
| `opacity-0` `opacity-50` `opacity-100` | Element opacity (fades children too — §22) |
| `mix-blend-multiply` `mix-blend-overlay` | Blend with elements behind |
| `bg-blend-screen` | Blend background layers |
| `blur-sm` `blur-md` `blur-lg` | Blur the element |
| `brightness-50` `brightness-125` `contrast-150` | Brightness / contrast |
| `grayscale` `grayscale-0` `saturate-150` `sepia` `invert` `hue-rotate-90` | Color filters |
| `drop-shadow-md` | Shadow following the element's shape |
| `backdrop-blur-md` | Blur what's **behind** (frosted glass) |
| `backdrop-brightness-75` `backdrop-saturate-150` | Backdrop adjustments |

> **Glassmorphism recipe:** `bg-white/30 backdrop-blur-md border border-white/20`.

---

## 13. Transitions, Animation & Transforms

**What it is:** `transition-*` (which properties animate + duration/delay/easing), `animate-*` (named keyframe animations like spin/pulse/bounce), and **transforms** (`scale`/`rotate`/`translate`/`skew`).

**The logic / when to use:** `transition` smoothly interpolates a property when it changes (e.g. on hover). The cheapest, smoothest things to animate are **`transform` and `opacity`** (the GPU compositor handles them — CSS_GUIDE §14.6), so prefer `transition-transform`/`transition-opacity`/`transition-colors` over `transition-all`. Use the `animate-*` set for spinners (`animate-spin`), skeleton loaders (`animate-pulse`), and attention pings (`animate-ping`).

**Best practices:** add `transition` + a `duration-*` + an easing to interactive elements for polish; avoid `transition-all` (it can animate layout properties and cause jank — §22); use the absolute-centering transform trick (`top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2`); honor reduced-motion with the `motion-reduce:` variant.

```html
<!-- Smooth button color transition on hover -->
<button class="bg-blue-600 transition-colors duration-200 hover:bg-blue-700">…</button>

<!-- Lift + grow on hover (transform = GPU-cheap) -->
<div class="transition-transform duration-200 hover:-translate-y-1 hover:scale-105">…</div>

<!-- Spinner -->
<svg class="size-5 animate-spin" viewBox="0 0 24 24">…</svg>

<!-- Skeleton loading bar -->
<div class="h-4 w-3/4 animate-pulse rounded bg-gray-200"></div>

<!-- Absolutely center an element with transforms -->
<div class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2">Centered</div>

<!-- Respect reduced-motion: disable the bounce for sensitive users -->
<div class="motion-safe:animate-bounce motion-reduce:animate-none">↓</div>
```

| Class | CSS / Description |
|---|---|
| `transition` | Transition the common set of properties |
| `transition-all` `transition-colors` `transition-transform` `transition-opacity` | Which properties animate |
| `duration-150` `duration-300` `duration-700` | Duration (ms) |
| `delay-100` | Delay (ms) |
| `ease-linear` `ease-in` `ease-out` `ease-in-out` | Timing function |
| `animate-spin` | Infinite spin (spinners) |
| `animate-ping` | Radar ping (notification dot) |
| `animate-pulse` | Fade pulse (skeletons) |
| `animate-bounce` | Bounce (scroll hints) |
| `animate-none` | Disable animation |
| `scale-95` `scale-105` `scale-x-110` | Scale transform |
| `rotate-45` `-rotate-12` | Rotate |
| `translate-x-4` `-translate-y-1/2` | Translate (move) |
| `skew-x-6` `origin-top-left` | Skew / transform origin |
| `transform-gpu` `will-change-transform` | GPU hints for smoother animation |

---

## 14. Interactivity, Tables, SVG & Accessibility

**What it is:** cursor & pointer-events, text selection, resize, scroll behavior/snap, native-control styling, table display utilities, SVG color helpers, pseudo-element utilities (`before:`/`after:`), and accessibility helpers like `sr-only`.

**The logic / when to use:** these are the "everything else" controls that make UIs feel finished — a `cursor-not-allowed` on a disabled button, `select-none` on a drag handle, `scroll-smooth` for anchor jumps, `snap-*` for carousels, and `sr-only` to give screen-reader-only labels to icon buttons.

**Best practices:** always give icon-only buttons an accessible name (`aria-label` or an `sr-only` span); use `scroll-mt-*` so anchored sections aren't hidden behind a sticky header; `fill-current`/`stroke-current` make inline SVGs inherit the surrounding text color (so one icon adapts to context); `before:`/`after:` need a `content-[…]` to appear.

```html
<!-- Icon button with a screen-reader-only label -->
<button class="rounded p-2 hover:bg-gray-100" aria-label="Close">
  <svg class="size-5 fill-current" viewBox="0 0 24 24"><!-- X icon --></svg>
  <span class="sr-only">Close dialog</span>
</button>

<!-- Disabled state with cursor + reduced opacity -->
<button class="disabled:cursor-not-allowed disabled:opacity-50" disabled>Submit</button>

<!-- Scroll-snap carousel -->
<div class="flex snap-x snap-mandatory overflow-x-auto">
  <img class="snap-center shrink-0" src="1.jpg" alt="" />
  <img class="snap-center shrink-0" src="2.jpg" alt="" />
</div>

<!-- Pseudo-element decoration (requires content-[…]) -->
<a class="after:ml-1 after:content-['→']" href="#">Read more</a>

<!-- Anchor target offset so a sticky header doesn't cover it -->
<section id="pricing" class="scroll-mt-20">…</section>
```

| Class | CSS / Description |
|---|---|
| `cursor-pointer` `cursor-not-allowed` `cursor-wait` `cursor-grab` | Cursor |
| `pointer-events-none` `pointer-events-auto` | Toggle mouse interaction |
| `select-none` `select-text` `select-all` | Text selection |
| `resize` `resize-none` `resize-y` | Textarea resize handle |
| `scroll-smooth` `scroll-mt-20` | Smooth scroll / anchor offset |
| `snap-x snap-mandatory` + `snap-center` | Scroll snapping (carousels) |
| `appearance-none` | Strip native control styling |
| `touch-pan-y` `touch-none` | Touch action |
| `table` `table-row` `table-cell` | Table display |
| `table-auto` `table-fixed` | Column sizing algorithm |
| `border-collapse` `border-separate` `border-spacing-2` | Table border model |
| `caption-top` `caption-bottom` | Caption placement |
| `fill-current` `stroke-current` | SVG uses current text color |
| `sr-only` `not-sr-only` | Visually hide (keep for screen readers) / undo |
| `before:` `after:` + `content-['…']` | Pseudo-element content |
| `marker:text-blue-500` `selection:bg-yellow-200` | Style markers / selected text |
| `placeholder:italic` `file:rounded` `first-letter:text-4xl` | Placeholder / file button / drop cap |

---

## 15. Responsive Design — Mobile-First Breakpoints

### 15.1 The prefix logic — and why it's mobile-first **[I]**

Tailwind is **mobile-first**: an **unprefixed** utility applies at *all* screen sizes, and a **breakpoint-prefixed** utility (`md:`, `lg:`) applies **from that breakpoint up**. There is no "this size only" by default — each prefix is a `min-width` media query that *adds* styles as the screen grows.

This is the single most common point of confusion, so internalize it:

```html
<!-- Reads as: "1 column by default; from sm up use 2; from lg up use 3."
     The base (grid-cols-1) is your MOBILE design; you LAYER bigger styles on top. -->
<div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3">…</div>

<!-- "Centered text on mobile, left-aligned from md up." -->
<p class="text-center md:text-left">…</p>

<!-- WRONG mental model: `md:flex` does NOT mean "flex only on tablet."
     It means "flex from 768px UP." To target a narrow range, see max-* below. -->
```

**Why mobile-first?** Mobile is the constraint — it's easier to start simple (one column, stacked) and progressively *enhance* for more space than to cram a desktop layout down. It also means your base styles are the smallest, and you only pay for complexity at larger sizes.

### 15.2 The breakpoint scale **[I]**

| Prefix | Min width | `@media` | Typical device |
|---|---|---|---|
| (none) | 0 | always | mobile (the base) |
| `sm:` | 640px | `min-width: 40rem` | large phone / small tablet |
| `md:` | 768px | `min-width: 48rem` | tablet |
| `lg:` | 1024px | `min-width: 64rem` | laptop |
| `xl:` | 1280px | `min-width: 80rem` | desktop |
| `2xl:` | 1536px | `min-width: 96rem` | large desktop |

### 15.3 `max-*`, range stacking, and container queries **[I/A]**

```html
<!-- max-* applies BELOW a breakpoint (a max-width query). -->
<div class="max-md:hidden">Visible only from md up (hidden below md)</div>

<!-- Stack a min and a max to target a RANGE (e.g. md only): -->
<div class="md:max-lg:bg-yellow-100">Yellow only between md and lg</div>
```

> **⚡ Version note — container queries are built in (v4):** v3 needed a plugin; **v4 ships container queries natively.** Mark a parent `@container`, then size children by the *parent's* width with `@sm:`, `@md:`, `@lg:` variants — so a component adapts to *where it's placed*, not the viewport. This is huge for reusable components (a card that reflows in a sidebar vs a wide column). See `CSS_GUIDE.md` §13.5 for the underlying CSS.

```html
<!-- The card adapts to ITS OWN width, not the screen's. -->
<div class="@container">
  <div class="flex flex-col @md:flex-row @md:gap-6">
    <img class="@md:w-1/3" src="…" alt="" />
    <div>…</div>
  </div>
</div>
```

### 15.4 Responsive best practices **[I]**

- **Design the mobile (base) layout first**, then add `sm:`/`md:`/`lg:` overrides — don't start desktop and remove things.
- Don't over-specify: only add a breakpoint where the layout actually needs to change.
- Prefer **container queries** for *components* and viewport breakpoints for *page-level* structure.
- Remember the `<meta name="viewport">` tag — without it, breakpoints behave oddly on phones (§22).

---

## 16. State Variants — hover/focus/group/peer/has

### 16.1 What variants are **[I]**

A **variant** is a prefix that wraps a utility in a condition — a pseudo-class, attribute selector, or media query. `hover:bg-blue-700` means "apply `bg-blue-700` *when hovered*." You can **stack** variants left-to-right: `dark:md:hover:bg-blue-700` = "in dark mode, from md up, on hover." The responsive prefixes from §15 are just variants too — everything composes.

```html
<button class="bg-blue-600 hover:bg-blue-700 focus-visible:ring-2 active:scale-95
               disabled:opacity-50">
  Interactive
</button>
```

### 16.2 The interaction & state variant table **[I]**

| Variant | Triggers on |
|---|---|
| `hover:` | pointer over |
| `focus:` | element focused |
| `focus-visible:` | keyboard focus only (prefer for focus rings) |
| `focus-within:` | a descendant is focused |
| `active:` | being pressed |
| `disabled:` | disabled form element |
| `checked:` | checked checkbox/radio |
| `required:` `invalid:` `valid:` | input validation state |
| `first:` `last:` `odd:` `even:` | position among siblings |
| `empty:` `only:` | no children / sole child |
| `aria-expanded:` `aria-checked:` | matches an ARIA attribute |
| `data-[state=open]:` | matches a `data-*` attribute (great with Radix/shadcn) |
| `motion-safe:` `motion-reduce:` | respects the reduced-motion setting |
| `dark:` | dark mode active (§17) |
| `group-*:` / `peer-*:` | parent / sibling state (below) |
| `has-[…]:` | element *contains* a match (below) |

### 16.3 `group-*` — react to a parent's state **[I/A]**

Normally a variant reacts to the element's *own* state. **`group`** lets a child react to an *ancestor's* state. Mark the parent with the `group` class, then use `group-hover:`, `group-focus:`, etc. on any descendant. This is how you build "hover the card → reveal the button / change the icon color."

```html
<!-- Hovering the whole card changes the title color and reveals the arrow. -->
<a href="#" class="group block rounded-xl border p-6">
  <h3 class="font-semibold text-gray-900 group-hover:text-indigo-600">Card title</h3>
  <p class="text-gray-600">Description…</p>
  <span class="mt-2 inline-block opacity-0 transition group-hover:opacity-100">→</span>
</a>
```

**Named groups** disambiguate when you nest groups: mark the parent `group/menu` and target it with `group-hover/menu:`.

```html
<li class="group/item">
  <div class="group/btn">
    <!-- reacts to the OUTER group, not the inner one -->
    <span class="hidden group-hover/item:block">Shown when the row is hovered</span>
  </div>
</li>
```

### 16.4 `peer-*` — react to a *sibling's* state **[I/A]**

**`peer`** is like `group`, but for **siblings**: mark one element `peer`, and a *later sibling* can style itself based on the peer's state with `peer-checked:`, `peer-focus:`, `peer-invalid:`, etc. The classic use is **custom checkboxes/toggles** and **floating labels**, with no JavaScript.

> **Important constraint:** `peer-*` only works on an element that comes *after* the `peer` in the DOM (CSS sibling combinators only look forward). Put the `peer` first.

```html
<!-- Custom toggle: hide the native checkbox, style a sibling label by its state. -->
<input type="checkbox" id="t" class="peer sr-only" />
<label for="t"
  class="cursor-pointer rounded-full bg-gray-300 px-4 py-1
         peer-checked:bg-green-500 peer-checked:text-white">
  Toggle
</label>

<!-- Floating label + validation styling, pure CSS -->
<input type="email" placeholder=" "
  class="peer border-b focus:border-indigo-500 invalid:border-red-500" />
<p class="hidden text-sm text-red-500 peer-invalid:peer-focus:block">
  Enter a valid email
</p>
```

> **group vs peer in one line:** `group-*` = "style me based on my **ancestor**." `peer-*` = "style me based on a **previous sibling**." Both eliminate JavaScript for many interactive patterns.

### 16.5 `has-*` — style a parent by what it contains **[A]**

**`has-[…]:`** wraps the CSS `:has()` relational selector (CSS_GUIDE §2.7) — it lets an element style itself based on what it *contains* or what *follows*. This is the "parent selector" Tailwind exposes as a variant.

```html
<!-- A label that turns red when its checkbox child is checked -->
<label class="rounded border p-3 has-[:checked]:border-indigo-500 has-[:checked]:bg-indigo-50">
  <input type="checkbox" /> Subscribe
</label>

<!-- A form field group that highlights when it contains an invalid input -->
<div class="rounded p-2 has-[:invalid]:bg-red-50">
  <input type="email" required />
</div>

<!-- group-has / peer-has compose too -->
<div class="group">
  <span class="group-has-[a]:underline">Underlined when the group contains a link</span>
</div>
```

> **⚡ Version note:** `has-*`, `group-has-*`, and `peer-has-*` rely on baseline `:has()` support, solid across evergreen browsers in 2026.

---

## 17. Dark Mode

### 17.1 The two strategies **[I]**

Dark mode in Tailwind is the `dark:` variant: `dark:bg-gray-900` applies *when dark mode is active*. There are two ways to decide "active":

1. **`media` (default):** follows the OS setting via `@media (prefers-color-scheme: dark)`. Zero JavaScript, but the user can't override the OS choice on your site.
2. **`class` / `selector` (manual toggle):** dark mode turns on when a `.dark` class (or a chosen selector) is present on an ancestor — usually `<html>`. This lets you build a toggle button. **Most apps want this.**

In **v4**, you configure the manual strategy with a custom variant in your CSS (there's no JS config by default):

```css
@import "tailwindcss";

/* Enable CLASS-based dark mode: dark: utilities apply when .dark is on an ancestor. */
@custom-variant dark (&:where(.dark, .dark *));
```

### 17.2 Using `dark:` and toggling it **[I]**

```html
<!-- These two classes mean "light by default, dark when .dark is active." -->
<body class="bg-white text-gray-900 dark:bg-gray-900 dark:text-gray-100">
  <div class="border-gray-200 dark:border-gray-700">…</div>
</body>
```

```js
// Toggle dark mode and persist the choice (class strategy).
const root = document.documentElement;        // <html>
function setTheme(dark) {
  root.classList.toggle("dark", dark);         // add/remove .dark
  localStorage.theme = dark ? "dark" : "light";
}
// On load: respect saved choice, else fall back to the OS preference.
const prefersDark = matchMedia("(prefers-color-scheme: dark)").matches;
setTheme(localStorage.theme ? localStorage.theme === "dark" : prefersDark);
```

> **Avoid the "flash of wrong theme":** apply the `.dark` class *before paint* by running a tiny inline script in `<head>` (read `localStorage`/`matchMedia` and set the class) — don't wait for your bundle to load. This is why theme toggles often have an inline boot script.

### 17.3 Best practices **[I]**

- Define your colors as **`@theme` tokens** (§19) and use semantic names (`--color-surface`, `--color-fg`) so dark mode is mostly "swap the token values," not hundreds of `dark:` classes.
- Don't just invert — dark UIs usually want slightly desaturated colors and softer shadows.
- Test contrast in *both* themes (CSS_GUIDE §17.3).
- You can stack: `dark:hover:bg-gray-800`, `dark:focus-visible:ring-gray-500`.

```html
<!-- Stacked variants: dark mode + hover -->
<button class="bg-gray-100 hover:bg-gray-200 dark:bg-gray-800 dark:hover:bg-gray-700">…</button>
```

---

## 18. Arbitrary Values & Arbitrary Properties

### 18.1 The escape hatch — when the scale doesn't have it **[I]**

Tailwind's scales cover most needs, but sometimes you need an exact value the design system doesn't include — a `327px` width from a spec, a brand hex, a one-off grid track. **Square-bracket arbitrary values** let you inline *any* CSS value into a utility, and the JIT engine generates the rule on the spot. This is what makes Tailwind feel unrestrictive despite being constrained.

```html
<div class="w-[327px]"></div>          <!-- width: 327px -->
<div class="top-[7%]"></div>           <!-- top: 7% -->
<div class="bg-[#1da1f2]"></div>       <!-- a brand blue not in the palette -->
<div class="text-[13px] leading-[1.7]"></div>
<div class="grid-cols-[1fr_2fr_1fr]"></div>  <!-- underscores = spaces in CSS -->
<div class="bg-[var(--brand)]"></div>  <!-- reference any CSS variable -->
```

> **Note on spaces:** CSS values that contain spaces (like grid templates or `calc()`) use **underscores** inside the brackets, which Tailwind converts to spaces: `grid-cols-[1fr_2fr]`. If you need a literal underscore, escape it `\_`.

### 18.2 Arbitrary *properties* and arbitrary *variants* **[I]**

Beyond values, you can set an entirely arbitrary CSS property, or write an arbitrary selector variant:

```html
<!-- Arbitrary PROPERTY: a CSS declaration with no dedicated utility -->
<div class="[mask-type:luminance]"></div>
<div class="[scrollbar-width:none]"></div>

<!-- Arbitrary VARIANT: an inline selector. & = "this element". -->
<li class="[&:nth-child(3)]:bg-red-100">…</li>     <!-- style the 3rd child -->
<div class="[&>p]:mt-2 [&_svg]:size-4">…</div>     <!-- style direct <p> / any <svg> inside -->

<!-- Arbitrary media/feature variant -->
<div class="[@supports(backdrop-filter:blur(0))]:backdrop-blur">…</div>
```

The `[&_…]` form is especially useful for styling children you don't control (e.g. HTML from a CMS or a third-party component) without writing a separate stylesheet.

### 18.3 The `!important` modifier and best practices **[I]**

```html
<!-- v4: the ! goes at the END of the utility (v3 put it at the start). -->
<p class="font-bold!">Forced !important</p>
```

> **⚡ Version note — important syntax moved:** v3 used a leading bang (`!font-bold`); **v4 uses a trailing bang (`font-bold!`)**. Update when porting.

**Best practices for arbitrary values:**
- Treat them as a **last resort**, not a habit — they bypass the design system's consistency.
- If an arbitrary value **recurs**, that's a signal to add it as a **token in `@theme`** (§19) so it becomes a first-class, reusable utility.
- Avoid arbitrary values that are computed dynamically in code — the class string must be statically present for the JIT to find it (§22).

---

## 19. Theme Customization with `@theme`

### 19.1 The v4 model: configuration *is* CSS **[I/A]**

In v3 you customized Tailwind in `tailwind.config.js`. **v4 moves configuration into your CSS** via the `@theme` at-rule. You declare **design tokens** as special CSS custom properties with conventional namespaces (`--color-*`, `--font-*`, `--spacing`, `--breakpoint-*`, `--radius-*`, …), and Tailwind **generates matching utilities** *and* exposes each token as a real `var()` you can use anywhere.

```css
@import "tailwindcss";

@theme {
  /* A new color → generates bg-brand, text-brand, border-brand, etc. */
  --color-brand: oklch(60% 0.15 264);
  --color-brand-muted: oklch(92% 0.04 264);

  /* A font family → generates font-display, and sets the --font-display var */
  --font-display: "Geist", system-ui, sans-serif;

  /* A custom radius → rounded-card */
  --radius-card: 0.875rem;

  /* A custom breakpoint → enables tablet:flex, etc. */
  --breakpoint-tablet: 60rem;
}
```

```html
<!-- Those tokens are now utilities AND variables: -->
<div class="bg-brand font-display rounded-card tablet:flex">…</div>
<div class="shadow-[0_2px_8px_var(--color-brand)]">…</div> <!-- token in an arbitrary value -->
```

### 19.2 Why this is better **[I]**

- **One source of truth:** your tokens live with your styles, in plain CSS, and double as `var()`s for hand-written CSS, inline styles, or arbitrary values — no JS/CSS sync problem.
- **Themeable at runtime:** because tokens are real CSS variables, you can override them per-scope or per-theme (light/dark, per-brand) just like any custom property (CSS_GUIDE §5.6).
- **No config file to maintain** for most projects.

### 19.3 Common token namespaces **[I]**

| Namespace | Generates utilities like | Example token |
|---|---|---|
| `--color-*` | `bg-*` `text-*` `border-*` `fill-*` `ring-*` | `--color-brand: oklch(…)` |
| `--font-*` | `font-*` | `--font-display: …` |
| `--text-*` | `text-*` (font sizes) | `--text-hero: 4rem` |
| `--spacing` | the entire spacing scale base | `--spacing: 0.25rem` |
| `--breakpoint-*` | responsive prefixes | `--breakpoint-tablet: 60rem` |
| `--container-*` | `max-w-*` container sizes & `@container` sizes | `--container-prose: 65ch` |
| `--radius-*` | `rounded-*` | `--radius-card: .875rem` |
| `--shadow-*` | `shadow-*` | `--shadow-soft: …` |
| `--ease-*` `--animate-*` | `ease-*` / `animate-*` | `--ease-snappy: …` |

### 19.4 Extending vs replacing, and referencing existing tokens **[A]**

By default `@theme` **adds to** the defaults. To **replace** a whole namespace (e.g. ship only your own colors, dropping Tailwind's), set it to `initial` first:

```css
@theme {
  --color-*: initial;          /* wipe the default palette… */
  --color-bg: oklch(98% 0 0);  /* …and define only what you want */
  --color-fg: oklch(20% 0 0);
  --color-brand: oklch(60% 0.15 264);
}
```

You can also keep tokens *out* of the generated CSS variables (so they don't bloat output) using `@theme inline { … }` when a token is only used to derive utilities, not referenced as a `var()`.

### 19.5 Custom keyframes and animations **[A]**

```css
@theme {
  --animate-fade-in: fade-in 0.3s ease-out;   /* → animate-fade-in utility */
}
/* Define the keyframes normally (inside @layer or top-level). */
@keyframes fade-in {
  from { opacity: 0; transform: translateY(4px); }
  to   { opacity: 1; transform: translateY(0); }
}
```

### 19.6 Other v4 directives worth knowing **[A]**

```css
/* Add your own utilities/components in the right layer. */
@utility tap-target { min-block-size: 44px; min-inline-size: 44px; }  /* → class="tap-target" */

/* Define a reusable custom variant (selector-based). */
@custom-variant pointer-fine (@media (pointer: fine));   /* → pointer-fine:hover:… */

/* Tell the engine where to look (only if auto-detection misses something). */
@source "../node_modules/some-ui-lib/dist";

/* Migration bridge: load an old JS config if you must. */
@config "./tailwind.config.js";
```

> **⚡ Version note:** `@theme`, `@utility`, `@custom-variant`, `@source`, and `@config` are all **v4 CSS-first directives**. If you find a tutorial editing `tailwind.config.js`, it's v3-era — the v4 equivalent is almost always one of these.

---

## 20. `@apply`, Reusing Styles & the `cn()` Pattern

### 20.1 `@apply` — and when NOT to use it **[I]**

`@apply` lets you pull existing utilities *into* a regular CSS rule, so you can create a named class out of utilities:

```css
@layer components {
  .btn-primary {
    /* Compose utilities into one class. */
    @apply inline-flex items-center rounded-lg bg-indigo-600 px-4 py-2
           text-sm font-semibold text-white transition-colors hover:bg-indigo-700;
  }
}
```

```html
<button class="btn-primary">Save</button>
```

This *feels* like the obvious way to "DRY up" repeated class lists — **but the Tailwind team and most experienced users recommend avoiding it for that purpose.** The reasons:

- It **re-introduces the very problems utilities solved**: you're back to inventing names, switching between HTML and CSS, and growing a stylesheet that can accumulate dead classes.
- It **hides the design system** — you can no longer see at a glance what `.btn-primary` does.
- The right reuse mechanism is usually a **component** (a React/Vue/Svelte component, a template partial, or a server include), which keeps the utilities visible *and* gives you props, variants, and logic.

**When `@apply` *is* legitimate:**
- Styling markup you don't control / can't add classes to (third-party HTML, CMS output, `prose` content).
- A handful of truly global primitives in a design system where a component isn't available (e.g. base form-control styling shared across frameworks).
- Avoiding repetition in places without a component layer (some plain-HTML or email-template contexts).

> **Rule of thumb:** if you *can* make a component, make a component. Reach for `@apply` only when you can't.

### 20.2 The real reuse pattern: components + a `cn()` helper **[I/A]**

In React/Vue/etc., you reuse by wrapping the utilities in a component and accepting a `className`/variant prop. The challenge is **merging** classes safely: a caller might pass `bg-red-500` to override the component's default `bg-indigo-600`, and you need the caller's class to *win* — but two `bg-*` utilities both in the list means whichever comes *last in the generated CSS* wins, not whichever you intended. Plain string concatenation can't resolve that conflict.

The community solution is the **`cn()` helper**, combining **`clsx`** (conditional class joining) with **`tailwind-merge`** (intelligently de-duplicates conflicting Tailwind classes, keeping the *last* one). This is the exact pattern shadcn/ui uses.

```bash
npm install clsx tailwind-merge
```

```ts
// lib/utils.ts — the canonical cn() helper.
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

// clsx handles conditionals/arrays/objects; twMerge resolves Tailwind conflicts.
export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

```tsx
// A reusable Button. Defaults are listed first; caller's className can override.
import { cn } from "@/lib/utils";

type Props = React.ButtonHTMLAttributes<HTMLButtonElement> & {
  variant?: "primary" | "ghost";
};

export function Button({ className, variant = "primary", ...props }: Props) {
  return (
    <button
      className={cn(
        // base
        "inline-flex items-center rounded-lg px-4 py-2 text-sm font-semibold transition-colors",
        // variant
        variant === "primary" && "bg-indigo-600 text-white hover:bg-indigo-700",
        variant === "ghost" && "bg-transparent text-gray-900 hover:bg-gray-100",
        // caller overrides win because twMerge keeps the last conflicting class
        className
      )}
      {...props}
    />
  );
}
```

```tsx
// Usage: the caller's bg-red-600 correctly replaces the default bg-indigo-600.
<Button className="bg-red-600 hover:bg-red-700">Delete</Button>
```

**Why `tailwind-merge` and not just `clsx`?** `clsx("bg-indigo-600", "bg-red-600")` produces *both* classes; the CSS cascade then picks by source order, which is unpredictable from the caller's view. `twMerge` understands that `bg-indigo-600` and `bg-red-600` target the same property and keeps only the last — giving intuitive override behavior.

### 20.3 Variant-driven components with CVA **[A]**

For components with many variants (size × color × state), **`class-variance-authority` (CVA)** organizes the class logic declaratively. It's common in shadcn-style codebases.

```ts
import { cva, type VariantProps } from "class-variance-authority";
import { cn } from "@/lib/utils";

const button = cva(
  "inline-flex items-center justify-center rounded-lg font-semibold transition-colors",
  {
    variants: {
      variant: {
        primary: "bg-indigo-600 text-white hover:bg-indigo-700",
        ghost: "hover:bg-gray-100",
      },
      size: { sm: "h-8 px-3 text-sm", md: "h-10 px-4", lg: "h-12 px-6 text-lg" },
    },
    defaultVariants: { variant: "primary", size: "md" },
  }
);

type Props = React.ButtonHTMLAttributes<HTMLButtonElement> &
  VariantProps<typeof button>;

export function Button({ className, variant, size, ...props }: Props) {
  return <button className={cn(button({ variant, size }), className)} {...props} />;
}
```

---

## 21. Plugins (Typography, Forms) & Framework Integration

### 21.1 Loading plugins in v4 **[I]**

Preflight intentionally strips default styling, which is perfect for app UIs but inconvenient for two cases: long-form *prose* (articles/markdown) and *form controls*. Two official plugins fix these. In v4 you load a plugin from CSS with `@plugin`:

```css
@import "tailwindcss";
@plugin "@tailwindcss/typography";  /* adds the `prose` classes */
@plugin "@tailwindcss/forms";       /* sensible base styles for form controls */
```

```bash
npm install -D @tailwindcss/typography @tailwindcss/forms
```

### 21.2 Typography plugin — `prose` **[I]**

The Typography plugin gives you the **`prose`** class: drop it on a container of raw HTML (e.g. rendered Markdown) and it styles headings, paragraphs, lists, code, blockquotes, and links into a polished article — without you classing each element (which utilities can't reach since the HTML is generated).

```html
<article class="prose prose-lg dark:prose-invert max-w-prose">
  <!-- Raw HTML from a CMS/Markdown renderer gets beautiful typographic defaults. -->
  <h1>Title</h1>
  <p>Body with <a href="#">links</a>, <code>code</code>, lists…</p>
</article>
```

- `prose-lg`/`prose-sm` scale it; `dark:prose-invert` flips it for dark mode.
- Override individual elements with `prose-headings:` / `prose-a:` modifiers: `prose-a:text-indigo-600`.

### 21.3 Forms plugin **[I]**

The Forms plugin resets native inputs/selects/checkboxes to a clean, consistent base that's *easy to extend with utilities* — without it you fight inconsistent browser defaults.

```html
<input type="email"
  class="w-full rounded-md border-gray-300 shadow-sm
         focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500" />
```

### 21.4 React / Next.js integration **[A]**

- **Vite-based React:** use `@tailwindcss/vite` (§2.2). Put `cn()` in `lib/utils.ts`.
- **Next.js (App Router):** Tailwind integrates via PostCSS — `npm install tailwindcss @tailwindcss/postcss`, add the PostCSS config (§2.3), and `@import "tailwindcss";` in `app/globals.css` (imported in the root `layout.tsx`). It works in both Server and Client Components since it's just CSS classes.
- **Conditional classes:** always merge with `cn()` so server-rendered defaults and client overrides resolve predictably.
- **Avoid dynamic class construction** (`` `text-${c}-500` ``) — Next's bundler can't see those strings either; use full static strings or `@theme` tokens (§22).

```tsx
// app/globals.css imported once in the root layout makes Tailwind available app-wide.
// A typical Next component:
import { cn } from "@/lib/utils";
export default function Card({ active }: { active: boolean }) {
  return (
    <div className={cn("rounded-xl border p-6", active && "ring-2 ring-indigo-500")}>
      …
    </div>
  );
}
```

### 21.5 shadcn/ui **[A]**

**shadcn/ui** is not a component *library* you install as a dependency — it's a collection of accessible components (built on Radix primitives) that you **copy into your own codebase** and own. It's built directly on the patterns in this guide: **Tailwind utilities + `cn()` + CVA + `@theme` design tokens** (often using semantic CSS-variable colors like `bg-background`, `text-foreground`, `border-border`). Because you own the code, you customize by editing the components and your `@theme` tokens directly.

```html
<!-- shadcn components lean on semantic token utilities you define in @theme: -->
<div class="bg-background text-foreground border border-border rounded-lg p-6">…</div>
```

> **The throughline:** shadcn = Radix (behavior/a11y) + Tailwind (styling) + `cn()`/CVA (composition) + `@theme` tokens (theming). Everything in §16, §19, and §20 is the foundation it stands on.

---

## 22. Gotchas

A consolidated list of the traps that cost Tailwind beginners the most time.

**1. Dynamic class names don't work.** The JIT engine scans source for **complete static strings**. `` `text-${color}-500` `` or `'p-' + n` produce nothing because that exact class never appears literally. *Fix:* map to full class strings (`const c = { red:'text-red-500', blue:'text-blue-500' }[color]`), or use a CSS variable / arbitrary value, or list the candidates somewhere static (`@source inline(...)`).

**2. Conflicting utilities — last in the *CSS* wins, not last in the class list.** `class="bg-red-500 bg-blue-500"` doesn't reliably pick blue; the winner is whichever utility comes later in the *generated stylesheet*. *Fix:* don't put two of the same property utility on one element; in components use `cn()`/`tailwind-merge` (§20) to resolve overrides predictably.

**3. Forgetting the viewport meta tag.** Without `<meta name="viewport" content="width=device-width, initial-scale=1">`, phones render at ~980px and your `sm:`/`md:` breakpoints feel "broken." *Fix:* add the tag (§2.6).

**4. `peer-*` only looks forward.** A `peer-checked:` element must come *after* the `peer` in the DOM, because CSS sibling combinators only target following siblings. *Fix:* put the `peer` element first (§16.4).

**5. `opacity-50` fades children too.** Element opacity applies to the whole subtree; you can't make a child fully opaque again. *Fix:* use a `/alpha` color (`bg-black/50`) instead of `opacity-*` when you only want a translucent background (§3.4, CSS_GUIDE §6.4).

**6. Flex/grid child won't shrink or truncate (`min-width: auto`).** Flex and grid items default to `min-width: auto` = their content size, so long text blocks shrinking. *Fix:* add `min-w-0` (and/or `overflow-hidden`) to the flex child you want to truncate (§5).

**7. `transition-all` animates unintended properties.** It can tween layout properties and cause jank/flashes. *Fix:* name the property — `transition-colors`, `transition-transform`, `transition-opacity` (§13).

**8. `100vh` overflows on mobile.** Toolbars make `h-screen`/`min-h-screen` too tall. *Fix:* use `h-dvh`/`min-h-dvh` (§8, CSS_GUIDE §5.3).

**9. Preflight removed my default styles.** No bullet points, no heading sizes, no default button look — that's Preflight (the reset) doing its job. *Fix:* style with utilities, use `list-disc`, or add the Typography plugin's `prose` for article content (§21).

**10. v3 → v4 renames.** Common breakers: gradients `bg-gradient-to-*` → `bg-linear-to-*` (§10); bare `ring` is now 1px not 3px (§11); important is `font-bold!` not `!font-bold` (§18); shadows/blur scale names shifted slightly (`shadow-sm` is the smallest). *Fix:* run the official `npx @tailwindcss/upgrade` codemod when migrating.

**11. Editing `tailwind.config.js` does nothing in v4.** v4 reads CSS-first config. *Fix:* use `@theme`, `@plugin`, `@custom-variant`, `@utility` in your CSS (§19); only fall back to JS config via `@config` for migration.

**12. The build doesn't pick up a new file/template.** v4 auto-detects most files, but generated files or files outside the project root may be missed. *Fix:* add `@source "path"` to point the scanner at them (§19.6).

**13. Arbitrary values with spaces.** `grid-cols-[1fr 2fr]` fails; CSS spaces inside brackets must be **underscores**: `grid-cols-[1fr_2fr]` (§18).

---

## 23. Most-Used Combos

Copy-paste starting points. Each is plain, correct v4 markup.

```html
<!-- Centered page wrapper -->
<div class="flex min-h-dvh items-center justify-center bg-gray-50">…</div>

<!-- Responsive content container -->
<div class="mx-auto max-w-7xl px-4 sm:px-6 lg:px-8">…</div>

<!-- Primary button (accessible focus, disabled state) -->
<button class="inline-flex items-center justify-center rounded-lg bg-indigo-600
               px-5 py-2.5 text-sm font-semibold text-white shadow-sm
               transition-colors hover:bg-indigo-700
               focus-visible:ring-2 focus-visible:ring-indigo-500 focus-visible:ring-offset-2
               disabled:cursor-not-allowed disabled:opacity-50">
  Save changes
</button>

<!-- Card that lifts on hover -->
<div class="rounded-xl border border-gray-200 bg-white p-6 shadow-sm
            transition-shadow hover:shadow-md">…</div>

<!-- Responsive card grid -->
<div class="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3">…</div>

<!-- Sticky, frosted navbar -->
<header class="sticky top-0 z-50 w-full border-b border-gray-200 bg-white/80 backdrop-blur-md">…</header>

<!-- Form input -->
<input class="w-full rounded-md border border-gray-300 px-3 py-2 text-sm shadow-sm
              focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500 focus:outline-none" />

<!-- Hero heading (balanced wrapping) -->
<h1 class="text-4xl font-bold tracking-tight text-gray-900 text-balance sm:text-6xl">…</h1>

<!-- Avatar -->
<img class="size-10 rounded-full object-cover ring-2 ring-white" src="…" alt="" />

<!-- Badge / pill -->
<span class="inline-flex items-center rounded-full bg-green-100 px-2.5 py-0.5
             text-xs font-medium text-green-800">Active</span>

<!-- Skeleton loader -->
<div class="h-4 w-3/4 animate-pulse rounded bg-gray-200"></div>

<!-- Glass panel -->
<div class="rounded-2xl border border-white/20 bg-white/30 p-6 shadow-lg backdrop-blur-md">…</div>

<!-- Reveal-on-hover (group) -->
<a class="group block rounded-xl border p-6">
  <h3 class="group-hover:text-indigo-600">Title</h3>
  <span class="opacity-0 transition group-hover:opacity-100">→</span>
</a>

<!-- Dark-mode-aware surface -->
<div class="bg-white text-gray-900 dark:bg-gray-900 dark:text-gray-100">…</div>

<!-- Two-column responsive layout -->
<div class="flex flex-col gap-8 lg:flex-row">…</div>
```

### Quick mental model
- **Layout** → pick `flex` (1D) or `grid` (2D), then `gap-*`, then alignment (`items-*`, `justify-*`).
- **Spacing** → `p-*` inside, `m-*`/`gap-*` between, `space-y-*` for stacked lists (prefer `gap`).
- **Responsive** → design mobile first; add `md:`/`lg:` to enhance upward; `@container` for components.
- **States** → `hover:`, `focus-visible:`, `disabled:`, `dark:`; `group-*`/`peer-*`/`has-*` for relational.
- **Reuse** → make a **component** + `cn()`; avoid `@apply` unless you must.
- **Stuck?** → an arbitrary value `[...]` always works; if it recurs, promote it to a `@theme` token.

---

## 24. Study Path & Build-to-Learn Projects

**Suggested order:** §1 (the philosophy — understand *why* before the *what*) → §2–3 (setup + how utilities map to CSS) → §4–8 (layout, flex, grid, spacing, sizing — the structural core; pair each with the matching `CSS_GUIDE.md` section) → §9–14 (typography, color, borders, effects, motion, interactivity) → §15–17 (responsive, state variants, dark mode — the "Tailwind magic") → §18 (arbitrary escape hatch) → §19–20 (theming with `@theme`, reuse with `cn()`) → §21 (plugins + framework integration) → §22 (re-read the gotchas after each project).

**Build these, in order, to cement everything:**

1. **A pricing card** — image/icon, title, price, feature list, CTA button. Practice the box model via utilities (`p-*`, `rounded-*`, `border`, `shadow-*`), a hover transition, and an accessible `focus-visible` ring. *Then* wrap it in a `@container` and use `@md:` variants so it reflows between a sidebar and a wide column. Exercises §4–§13, §15.3.

2. **A responsive navbar** — logo left, links right (`justify-between`), collapsing to a stacked menu below `md` (`hidden md:flex` + a toggle). Build a custom toggle/hamburger using the **`peer`** pattern with no JS. Add `focus-visible` styles. Exercises §5, §15, §16.4.

3. **A landing page (grid + flex)** — page skeleton with `grid` (header / hero / features / footer); a feature section using `grid-cols-[repeat(auto-fit,minmax(250px,1fr))]`; each card's internals with flex; a sticky frosted header. The canonical "grid for the page, flex for components." Exercises §4–§6, §10, §12.

4. **A dark-mode theme with `@theme` tokens** — define a semantic palette (`--color-bg`, `--color-fg`, `--color-brand`) in `oklch()` inside `@theme`, enable class-based dark mode with `@custom-variant`, and wire a persistent JS toggle with an anti-flash inline boot script. Exercises §17, §19.

5. **(Stretch) A small component library** — build `Button`, `Input`, `Card`, and `Badge` as React components using **`cn()` + CVA**, with size/variant props, and a shared `@theme` token set. Add the Typography and Forms plugins; render a Markdown article with `prose`. This pulls together §19, §20, §21 — and is essentially how shadcn/ui is structured.

**Next steps after this guide:** the official docs search (`/` on tailwindcss.com) for confirming exact class names; `CSS_GUIDE.md` for the CSS theory under every utility; the React/Next guides in this library for component architecture; and `MOTION_ANIMATION_GUIDE.md` when CSS transitions/animations aren't enough.

---

*Part of the offline developer study library. Written for **Tailwind CSS v4 (2026)** — CSS-first `@theme` configuration, the new Oxide engine, no `tailwind.config.js` by default, built-in container queries, `oklch()` colors, and the `cn()`/CVA component pattern. Pairs with `CSS_GUIDE.md`. Confirm fast-moving details at tailwindcss.com/docs and MDN.*
