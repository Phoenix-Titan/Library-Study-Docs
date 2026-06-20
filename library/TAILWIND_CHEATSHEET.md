# Tailwind CSS Cheatsheet (2026 / v4-ready)

A practical reference for **Tailwind CSS**. Each utility lists the class and a brief description of what it does.

> **Tailwind v4 note:** v4 uses a CSS-first setup. In your main CSS file you now write `@import "tailwindcss";` (instead of the old `@tailwind base; @tailwind components; @tailwind utilities;`). Theme tokens are defined in CSS with `@theme { ... }` instead of `tailwind.config.js`. **All the utility classes below are unchanged** — v4 mostly changes configuration, not the classes you write in HTML/JSX.

---

## Table of Contents
1. [Setup & Core Concepts](#1-setup--core-concepts)
2. [Responsive & State Variants](#2-responsive--state-variants)
3. [Layout](#3-layout)
4. [Flexbox](#4-flexbox)
5. [Grid](#5-grid)
6. [Spacing (Padding / Margin / Gap)](#6-spacing)
7. [Sizing (Width / Height)](#7-sizing)
8. [Typography](#8-typography)
9. [Colors & Backgrounds](#9-colors--backgrounds)
10. [Borders & Outline](#10-borders--outline)
11. [Effects (Shadow / Opacity / Blend)](#11-effects)
12. [Filters & Backdrop](#12-filters--backdrop)
13. [Transitions & Animation](#13-transitions--animation)
14. [Transforms](#14-transforms)
15. [Interactivity](#15-interactivity)
16. [Tables](#16-tables)
17. [SVG, Accessibility & Misc](#17-svg-accessibility--misc)
18. [Arbitrary Values & Custom Tricks](#18-arbitrary-values--custom-tricks)
19. [Dark Mode](#19-dark-mode)
20. [Most-Used Combos (copy-paste)](#20-most-used-combos)

---

## 1. Setup & Core Concepts

```bash
# Install (v4)
npm install tailwindcss @tailwindcss/postcss
# or with the Vite plugin
npm install tailwindcss @tailwindcss/vite
```

```css
/* app.css (v4) */
@import "tailwindcss";

@theme {
  --color-brand: #4f46e5;     /* now usable as bg-brand, text-brand */
  --font-display: "Geist", sans-serif;
}
```

| Concept | Meaning |
|---|---|
| **Utility-first** | Style by composing small single-purpose classes directly in markup. |
| **Mobile-first** | Unprefixed classes apply to all sizes; prefixes (`md:`, `lg:`) add styles *from that breakpoint up*. |
| **JIT** | Classes are generated on-demand; arbitrary values like `w-[327px]` work out of the box. |
| `cn()` / `clsx` | Helper to merge conditional classes (very common in React/shadcn). |

---

## 2. Responsive & State Variants

Prefix any utility with these. Example: `md:flex`, `hover:bg-blue-600`, `dark:text-white`.

**Breakpoints (min-width, mobile-first):**

| Prefix | Min width | Typical device |
|---|---|---|
| `sm:` | 640px | large phone / small tablet |
| `md:` | 768px | tablet |
| `lg:` | 1024px | laptop |
| `xl:` | 1280px | desktop |
| `2xl:` | 1536px | large desktop |
| `max-md:` | < 768px | apply *below* a breakpoint |

**State / interaction variants:**

| Variant | Triggers on |
|---|---|
| `hover:` | mouse over |
| `focus:` | element focused |
| `focus-visible:` | keyboard focus only |
| `focus-within:` | a child is focused |
| `active:` | being clicked |
| `disabled:` | disabled form element |
| `checked:` | checked checkbox/radio |
| `required:` / `invalid:` / `valid:` | input validation state |
| `first:` / `last:` / `odd:` / `even:` | position in a list |
| `group-hover:` | parent has `group` and is hovered |
| `peer-checked:` | sibling with `peer` is checked |
| `aria-expanded:` / `data-[state=open]:` | matches ARIA / data attributes |
| `motion-safe:` / `motion-reduce:` | respects reduced-motion setting |

**Group & peer pattern:**
```html
<div class="group">
  <p class="text-gray-500 group-hover:text-black">Hover the box</p>
</div>

<input type="checkbox" class="peer hidden" id="t">
<label for="t" class="peer-checked:bg-green-500">Toggle</label>
```

---

## 3. Layout

| Class | Description |
|---|---|
| `block` / `inline-block` / `inline` | Display type. |
| `flex` / `inline-flex` | Flex container. |
| `grid` / `inline-grid` | Grid container. |
| `hidden` | `display: none`. |
| `contents` | Element box disappears, children remain. |
| `container` | Centers + sets max-width per breakpoint. |
| `box-border` / `box-content` | Box-sizing model. |
| **Position** | |
| `static` `relative` `absolute` `fixed` `sticky` | Positioning. |
| `inset-0` | top/right/bottom/left = 0 (full overlay). |
| `top-0` `right-4` `bottom-2` `left-1/2` | Edge offsets. |
| `z-0` `z-10` `z-50` `z-[999]` | Stacking order. |
| **Overflow** | |
| `overflow-auto` `overflow-hidden` `overflow-scroll` | Scroll behavior. |
| `overflow-x-auto` `overflow-y-hidden` | Per-axis. |
| `truncate` | One-line ellipsis (`overflow-hidden text-ellipsis whitespace-nowrap`). |
| **Float / Object** | |
| `float-left` `float-right` `clear-both` | Floats. |
| `object-cover` `object-contain` `object-fill` | How `img`/`video` fits its box. |
| `object-center` `object-top` | Image focal position. |
| `aspect-square` `aspect-video` `aspect-[4/3]` | Aspect ratio. |
| `isolate` | New stacking context. |

---

## 4. Flexbox

| Class | Description |
|---|---|
| `flex-row` / `flex-col` | Main-axis direction. |
| `flex-row-reverse` / `flex-col-reverse` | Reversed direction. |
| `flex-wrap` / `flex-nowrap` | Wrap items or not. |
| `justify-start` `justify-center` `justify-end` | Align on **main** axis. |
| `justify-between` `justify-around` `justify-evenly` | Distribute space on main axis. |
| `items-start` `items-center` `items-end` | Align on **cross** axis. |
| `items-stretch` `items-baseline` | Stretch / baseline align. |
| `content-center` `content-between` | Align wrapped lines. |
| `self-start` `self-center` `self-end` `self-auto` | Override align for one item. |
| `flex-1` | Grow & shrink to fill (`flex: 1 1 0%`). |
| `flex-auto` `flex-initial` `flex-none` | Common flex shorthands. |
| `grow` / `grow-0` | Allow / prevent growing. |
| `shrink` / `shrink-0` | Allow / prevent shrinking (`shrink-0` stops squishing). |
| `order-1` `order-last` `order-first` | Visual order. |
| `gap-4` `gap-x-2` `gap-y-6` | Spacing between items (works in flex & grid). |

**Classic centering:** `flex items-center justify-center`

---

## 5. Grid

| Class | Description |
|---|---|
| `grid-cols-3` | 3 equal columns. |
| `grid-cols-[200px_1fr]` | Custom column track sizes. |
| `grid-rows-4` | 4 rows. |
| `col-span-2` `row-span-3` | Item spans multiple cells. |
| `col-start-2` `col-end-4` | Explicit placement. |
| `auto-cols-fr` `auto-rows-min` | Implicit track sizing. |
| `grid-flow-row` `grid-flow-col` `grid-flow-dense` | Auto-placement direction. |
| `place-items-center` | Shorthand: align + justify items. |
| `place-content-between` | Align + justify the whole grid. |
| `gap-6` `gap-x-4 gap-y-8` | Gutters. |

**Responsive card grid:** `grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6`

---

## 6. Spacing

Scale: `0, 0.5, 1, 1.5, 2, 2.5, 3, 4, 5, 6, 8, 10, 12, 16, 20, 24, 32...` where `1 = 0.25rem (4px)`. So `p-4` = 16px.

| Class | Description |
|---|---|
| `p-4` | Padding all sides. |
| `px-6` / `py-2` | Padding horizontal / vertical. |
| `pt-2 pr-4 pb-2 pl-4` | Padding individual side. |
| `ps-4` / `pe-4` | Padding inline-start / -end (RTL-aware). |
| `m-4` `mx-auto` `my-8` | Margin (use `mx-auto` to center a block). |
| `mt-2 mr-4 mb-2 ml-4` | Margin per side. |
| `-mt-4` | Negative margin. |
| `space-x-4` / `space-y-2` | Adds margin *between* children (no gap on ends). |
| `gap-4` | Preferred spacing inside flex/grid containers. |

---

## 7. Sizing

| Class | Description |
|---|---|
| `w-4` `w-64` `w-px` | Fixed width (scale or pixel). |
| `w-full` `w-screen` | 100% / 100vw. |
| `w-1/2` `w-1/3` `w-2/3` | Fractional width. |
| `w-min` `w-max` `w-fit` | Content-based widths. |
| `max-w-md` `max-w-7xl` `max-w-prose` | Max width (great for content columns). |
| `min-w-0` | Allow flex children to shrink/truncate. |
| `h-12` `h-full` `h-screen` | Heights (`h-screen` = 100vh). |
| `min-h-screen` | Minimum full-viewport height (hero/page wrapper). |
| `size-10` | Sets width AND height (v3.4+) — perfect for icons/avatars. |

---

## 8. Typography

| Class | Description |
|---|---|
| `text-xs` `text-sm` `text-base` `text-lg` `text-xl` ... `text-9xl` | Font size. |
| `font-thin` `font-normal` `font-medium` `font-semibold` `font-bold` `font-black` | Font weight. |
| `italic` / `not-italic` | Italic style. |
| `font-sans` `font-serif` `font-mono` | Font family. |
| `text-left` `text-center` `text-right` `text-justify` | Alignment. |
| `leading-none` `leading-tight` `leading-normal` `leading-relaxed` `leading-7` | Line height. |
| `tracking-tight` `tracking-normal` `tracking-wide` | Letter spacing. |
| `uppercase` `lowercase` `capitalize` `normal-case` | Text casing. |
| `underline` `line-through` `no-underline` `overline` | Text decoration. |
| `decoration-2` `decoration-dotted` `underline-offset-4` | Decoration style/offset. |
| `text-ellipsis` `truncate` | Overflow ellipsis. |
| `whitespace-nowrap` `whitespace-pre` | Whitespace handling. |
| `break-words` `break-all` `text-balance` `text-pretty` | Wrapping (`text-balance` = nice headline wraps). |
| `line-clamp-3` | Limit to N lines with ellipsis. |
| `list-disc` `list-decimal` `list-none` | List markers. |
| `indent-8` | First-line indent. |
| `antialiased` | Smoother font rendering. |
| `prose` `prose-lg` | (Typography plugin) auto-styles article HTML. |

---

## 9. Colors & Backgrounds

Color format: `{utility}-{color}-{shade}` where shade is `50–950`. Example: `bg-blue-500`, `text-gray-700`.

| Class | Description |
|---|---|
| `text-red-500` | Text color. |
| `bg-slate-900` | Background color. |
| `bg-white/80` | Background with 80% opacity (slash = alpha). |
| `bg-transparent` `bg-current` `bg-inherit` | Keyword colors. |
| `border-gray-200` | Border color. |
| `fill-blue-500` `stroke-red-500` | SVG colors. |
| `placeholder-gray-400` | Placeholder text color. |
| `caret-blue-500` | Text caret color. |
| `accent-indigo-600` | Native checkbox/radio accent. |
| **Gradients** | |
| `bg-gradient-to-r` | Direction (`-t -b -l -r -tr -bl` etc.). |
| `from-purple-500` `via-pink-500` `to-red-500` | Gradient stops. |
| `bg-[radial-gradient(...)]` | Arbitrary gradient. |
| **Background image/position** | |
| `bg-cover` `bg-contain` | Sizing. |
| `bg-center` `bg-top` `bg-no-repeat` | Position/repeat. |
| `bg-fixed` `bg-local` | Attachment. |
| `bg-[url('/hero.jpg')]` | Arbitrary background image. |

Default palettes: `slate, gray, zinc, neutral, stone, red, orange, amber, yellow, lime, green, emerald, teal, cyan, sky, blue, indigo, violet, purple, fuchsia, pink, rose`.

---

## 10. Borders & Outline

| Class | Description |
|---|---|
| `border` | 1px border all sides. |
| `border-2` `border-4` `border-8` | Border width. |
| `border-t` `border-x` `border-b-2` | Per-side width. |
| `border-dashed` `border-dotted` `border-double` | Border style. |
| `rounded` `rounded-md` `rounded-lg` `rounded-xl` `rounded-2xl` `rounded-full` | Border radius. |
| `rounded-t-lg` `rounded-tl-xl` | Radius per corner/side. |
| `divide-y` `divide-x-2` | Borders *between* children. |
| `divide-gray-200` | Divider color. |
| `outline` `outline-2` `outline-offset-2` | Outline (focus rings). |
| `outline-none` | Remove outline (pair with a custom `focus:ring`). |
| `ring` `ring-2` `ring-4` | Box-shadow ring (great for focus states). |
| `ring-blue-500` `ring-offset-2` `ring-offset-white` | Ring color & offset. |

---

## 11. Effects

| Class | Description |
|---|---|
| `shadow-sm` `shadow` `shadow-md` `shadow-lg` `shadow-xl` `shadow-2xl` | Box shadow. |
| `shadow-none` | Remove shadow. |
| `shadow-blue-500/50` | Colored shadow. |
| `opacity-0` `opacity-50` `opacity-100` | Element opacity. |
| `mix-blend-multiply` `mix-blend-overlay` | Blend mode with elements behind. |
| `bg-blend-screen` | Blend background layers. |

---

## 12. Filters & Backdrop

| Class | Description |
|---|---|
| `blur-sm` `blur-md` `blur-lg` | Blur element. |
| `brightness-50` `brightness-125` | Brightness. |
| `contrast-50` `contrast-150` | Contrast. |
| `grayscale` `grayscale-0` | Desaturate. |
| `saturate-150` | Saturation. |
| `sepia` `invert` `hue-rotate-90` | Color filters. |
| `drop-shadow-md` | Shadow that follows shape (good for PNG/SVG). |
| `backdrop-blur-md` | Blur what's *behind* (frosted glass). |
| `backdrop-brightness-75` `backdrop-saturate-150` | Backdrop adjustments. |

**Glassmorphism:** `bg-white/30 backdrop-blur-md border border-white/20`

---

## 13. Transitions & Animation

| Class | Description |
|---|---|
| `transition` | Transition common properties. |
| `transition-all` `transition-colors` `transition-transform` `transition-opacity` | Which properties animate. |
| `duration-150` `duration-300` `duration-700` | Transition duration (ms). |
| `delay-100` | Delay. |
| `ease-linear` `ease-in` `ease-out` `ease-in-out` | Timing function. |
| `animate-spin` | Infinite spin (spinners). |
| `animate-ping` | Radar ping (notification dot). |
| `animate-pulse` | Fade pulse (skeletons). |
| `animate-bounce` | Bounce (scroll hints). |
| `animate-none` | Disable animation. |
| `will-change-transform` | Hint browser for smoother animation. |

**Smooth button:** `transition-colors duration-200 hover:bg-blue-700`

---

## 14. Transforms

| Class | Description |
|---|---|
| `scale-95` `scale-105` `scale-x-110` | Scale. |
| `rotate-45` `-rotate-12` | Rotate. |
| `translate-x-4` `-translate-y-1/2` | Move. |
| `skew-x-6` | Skew. |
| `origin-center` `origin-top-left` | Transform origin. |
| `transform-gpu` | Force GPU acceleration. |

**Center absolutely:** `absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2`

---

## 15. Interactivity

| Class | Description |
|---|---|
| `cursor-pointer` `cursor-not-allowed` `cursor-wait` `cursor-grab` | Cursor style. |
| `pointer-events-none` `pointer-events-auto` | Toggle mouse interaction. |
| `select-none` `select-text` `select-all` | Text selection behavior. |
| `resize` `resize-none` `resize-y` | Textarea resize handle. |
| `scroll-smooth` | Smooth scrolling. |
| `scroll-mt-20` | Scroll margin (offset for sticky headers + anchors). |
| `snap-x snap-mandatory` + `snap-center` | Scroll snapping (carousels). |
| `appearance-none` | Strip native styling (custom selects/inputs). |
| `touch-pan-y` `touch-none` | Touch action. |
| `accent-pink-600` | Native control accent color. |
| `caret-transparent` | Hide caret. |
| `aria-disabled:opacity-50` | Style by ARIA state. |

---

## 16. Tables

| Class | Description |
|---|---|
| `table` `table-row` `table-cell` | Table display. |
| `table-auto` `table-fixed` | Column sizing algorithm. |
| `border-collapse` `border-separate` | Border model. |
| `border-spacing-2` | Spacing in separate mode. |
| `caption-top` `caption-bottom` | Caption placement. |

---

## 17. SVG, Accessibility & Misc

| Class | Description |
|---|---|
| `fill-current` `stroke-current` | Use current text color for SVG. |
| `sr-only` | Visually hide but keep for screen readers. |
| `not-sr-only` | Undo `sr-only`. |
| `forced-color-adjust-auto` | High-contrast mode handling. |
| `[content:''] before:` | Pseudo-element content. |
| `before:` / `after:` | Style pseudo-elements (`before:content-['→']`). |
| `marker:text-blue-500` | Style list bullets. |
| `selection:bg-yellow-200` | Style highlighted text. |
| `placeholder:italic` | Style placeholder. |
| `file:mr-4 file:rounded` | Style file input button. |
| `first-letter:text-4xl` | Drop-cap styling. |

---

## 18. Arbitrary Values & Custom Tricks

When the scale doesn't have what you need, use square brackets:

| Pattern | Example |
|---|---|
| Arbitrary size | `w-[327px]` `h-[42vh]` `mt-[7px]` |
| Arbitrary color | `bg-[#1da1f2]` `text-[rgb(20,30,40)]` |
| Arbitrary font | `text-[13px]` `leading-[1.7]` |
| Arbitrary grid | `grid-cols-[1fr_2fr_1fr]` |
| CSS variable | `bg-[var(--brand)]` |
| Arbitrary property | `[mask-type:luminance]` |
| Arbitrary variant | `[&:nth-child(3)]:bg-red-100` |
| Target children | `[&>p]:mt-2` `[&_svg]:size-4` |
| Important | `!font-bold` (forces `!important`) |

---

## 19. Dark Mode

```css
/* v4: enable class-based dark mode */
@custom-variant dark (&:where(.dark, .dark *));
```

| Class | Description |
|---|---|
| `dark:bg-gray-900` | Applies in dark mode. |
| `dark:text-gray-100` | Dark-mode text. |
| `dark:hover:bg-gray-800` | Combine variants. |

Toggle by adding/removing the `dark` class on `<html>`. Default mode is `media` (follows OS) unless you switch to `class`.

---

## 20. Most-Used Combos

```html
<!-- Centered page wrapper -->
<div class="min-h-screen flex items-center justify-center bg-gray-50">

<!-- Responsive container -->
<div class="mx-auto max-w-7xl px-4 sm:px-6 lg:px-8">

<!-- Primary button -->
<button class="inline-flex items-center justify-center rounded-lg bg-indigo-600 px-5 py-2.5 text-sm font-semibold text-white shadow-sm transition-colors hover:bg-indigo-700 focus-visible:outline-2 focus-visible:outline-indigo-600 disabled:opacity-50">

<!-- Card -->
<div class="rounded-xl border border-gray-200 bg-white p-6 shadow-sm hover:shadow-md transition-shadow">

<!-- Responsive card grid -->
<div class="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3">

<!-- Sticky navbar -->
<header class="sticky top-0 z-50 w-full border-b bg-white/80 backdrop-blur-md">

<!-- Input field -->
<input class="w-full rounded-md border border-gray-300 px-3 py-2 text-sm shadow-sm focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500 focus:outline-none">

<!-- Hero heading -->
<h1 class="text-4xl font-bold tracking-tight text-gray-900 sm:text-6xl text-balance">

<!-- Avatar -->
<img class="size-10 rounded-full object-cover ring-2 ring-white">

<!-- Badge -->
<span class="inline-flex items-center rounded-full bg-green-100 px-2.5 py-0.5 text-xs font-medium text-green-800">

<!-- Skeleton loader -->
<div class="h-4 w-3/4 animate-pulse rounded bg-gray-200">

<!-- Glass panel -->
<div class="rounded-2xl border border-white/20 bg-white/30 p-6 shadow-lg backdrop-blur-md">

<!-- Two-column responsive layout -->
<div class="flex flex-col gap-8 lg:flex-row">
```

---

### Quick mental model
- **Layout** → pick `flex` or `grid`, then `gap-*`, then alignment (`items-*`, `justify-*`).
- **Spacing** → `p-*` inside, `m-*`/`gap-*` between, `space-y-*` for stacked lists.
- **Responsive** → design mobile first, add `md:`/`lg:` to scale up.
- **States** → `hover:`, `focus-visible:`, `disabled:`, `dark:`.
- **Stuck?** → arbitrary value `[...]` always works.

> Bookmark the official docs search (press `/` on tailwindcss.com) — it's the fastest way to confirm a class.
