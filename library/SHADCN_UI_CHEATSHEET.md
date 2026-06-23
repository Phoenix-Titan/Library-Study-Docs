# shadcn/ui — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I keep hearing about shadcn but I don't get what it *is*" to "I scaffold a design system I own, build my own CVA variants, wire react-hook-form + zod, and ship accessible Radix-powered UIs in Next.js" — without an internet connection. Every concept is explained in prose first (what it is, *why* it works the way it does, when to reach for it, how to use it, the key props), then shown with heavily-commented code. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections and components are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets the **shadcn/ui CLI and ecosystem as of 2026** — i.e. the `shadcn` CLI (the package renamed from `shadcn-ui` to `shadcn`), **React 19** and **Next.js (App Router)**, **Tailwind CSS v4** (the CSS-first `@theme` / `@import "tailwindcss"` setup, no `tailwind.config.js` required), and **Radix UI primitives**. The big shifts that are now baseline:
> - **Tailwind v4** — theming lives in CSS via `@theme inline` and the `@custom-variant dark`, not a JS config. The CLI's `init` writes v4-style tokens using the **oklch** color space.
> - **The registry / `components.json`** — a machine-readable schema that lets the CLI add components, and lets *you* publish your own registry.
> - **Sonner replaces the old Toast** — the deprecated `toast`/`useToast` component is gone from new installs; notifications use Sonner.
> - **React 19 `ref` as a prop** — many components no longer need `React.forwardRef`; refs are passed like normal props. Older generated components still use `forwardRef` and both are correct.
> - **`data-slot` attributes** — newer generated components tag their parts with `data-slot="..."` so you can style any sub-part from the outside.
>
> shadcn/ui is **not a dependency you install and import** — it is *code generation*. You run a CLI, the component's TypeScript source is written into *your* repo, and you own it forever. That single idea changes everything about how you use, customize, and reason about it. Where a feature is newer or fast-moving it is flagged with **⚡ Version note**. The author is on **Windows 11**; commands are shown for `npx`/`pnpm dlx`. This guide pairs with `REACT_19_GUIDE.md`, `TAILWIND_CHEATSHEET.md`, `REACT_HOOK_FORM_GUIDE.md`, `NEXTJS_16_GUIDE.md`, and `TANSTACK_QUERY_GUIDE.md` in this library; it is cross-referenced throughout. Confirm exact details at ui.shadcn.com when you have a connection.

---

## Table of Contents

1. [The shadcn Philosophy — What It Actually Is](#1-the-shadcn-philosophy--what-it-actually-is) **[B]**
2. [Install & the CLI (`init`, `add`)](#2-install--the-cli-init-add) **[B]**
3. [Anatomy of a Component (button.tsx dissected)](#3-anatomy-of-a-component-buttontsx-dissected) **[I]**
4. [`cn()`, `clsx`, tailwind-merge & `asChild`](#4-cn-clsx-tailwind-merge--aschild) **[I]**
5. [Theming — CSS Variables, Tokens & Dark Mode](#5-theming--css-variables-tokens--dark-mode) **[I]**
6. [Forms & Inputs Reference](#6-forms--inputs-reference) **[B/I]**
7. [The Form Pattern (react-hook-form + zod) In Depth](#7-the-form-pattern-react-hook-form--zod-in-depth) **[I/A]**
8. [Buttons & Actions Reference](#8-buttons--actions-reference) **[B]**
9. [Overlays & Dialogs Reference](#9-overlays--dialogs-reference) **[I]**
10. [Navigation Reference](#10-navigation-reference) **[I]**
11. [Data Display Reference](#11-data-display-reference) **[I]**
12. [Feedback & Status Reference](#12-feedback--status-reference) **[B/I]**
13. [Layout & Utility Reference](#13-layout--utility-reference) **[I]**
14. [Composing & Customizing Components](#14-composing--customizing-components) **[A]**
15. [Building Your Own Variants with CVA](#15-building-your-own-variants-with-cva) **[A]**
16. [Accessibility from Radix](#16-accessibility-from-radix) **[I/A]**
17. [Next.js Integration — Server vs Client Components](#17-nextjs-integration--server-vs-client-components) **[I/A]**
18. [Common Gotchas](#18-common-gotchas) **[I]**
19. [Study Path & Build-to-Learn Projects](#19-study-path--build-to-learn-projects)

---

## 1. The shadcn Philosophy — What It Actually Is

### 1.1 The one idea that explains everything **[B]**

Almost every UI library you have used — Material UI, Chakra, Ant Design, Bootstrap — works the same way: you `npm install` a package, you `import { Button } from "the-library"`, and the component's code lives inside `node_modules`, hidden from you. You configure it through props and a theme object, but you cannot *open the file* and change how the button is built. When the maintainers ship a breaking change, you are dragged along. When you need a tweak the API doesn't expose, you fight the abstraction.

**shadcn/ui inverts this.** It is *not* a package you install and import from. It is a **CLI that copies source code into your project**. You run `npx shadcn@latest add button`, and a real file — `components/ui/button.tsx` — appears in *your* repository. From that moment, that code is *yours*. You can rename it, restyle it, delete props, add props, or throw it away. There is no `node_modules` version of the Button to conflict with. Updates are not `npm update`; they are "re-run the add command and merge the diff, or just edit your copy."

The official tagline captures it: **"This is not a component library. It is how you build your component library."** shadcn gives you well-built starting points; you grow them into *your* design system.

### 1.2 What it is built on **[B]**

A shadcn component is a thin, opinionated layer over three lower-level tools. Understanding the layers is the difference between copy-pasting and *mastering*:

| Layer | What it provides | Why it matters |
|---|---|---|
| **Radix UI primitives** | Unstyled, fully-accessible behavior: focus management, keyboard navigation, ARIA roles, portals, controlled/uncontrolled state. | Accessibility and hard interaction logic (a focus-trapped modal, a roving-tabindex menu) are *solved* for you. You never write that code. |
| **Tailwind CSS** | Utility classes for all the visual styling — colors, spacing, radius, shadows. | Styling is co-located in `className`, themeable via CSS variables, and trivially overridable. See `TAILWIND_CHEATSHEET.md`. |
| **CVA (class-variance-authority)** | A tiny function that maps `variant`/`size` props to sets of Tailwind classes. | Gives you a clean, type-safe `variant="destructive" size="sm"` API instead of stringing classes together by hand. |

A fourth helper, `cn()` (built from `clsx` + `tailwind-merge`), glues it together by merging class strings intelligently (§4). That is the *entire* technology stack. There is no proprietary styling engine, no runtime CSS-in-JS, no magic. If you know Radix + Tailwind + CVA, you know how every shadcn component is built — because they are all built the same way.

### 1.3 Trade-offs vs MUI / Chakra / Ant **[I]**

shadcn is not automatically "better" — it is a different deal with different costs. Choose deliberately:

**What you gain with shadcn**
- **Full ownership / zero lock-in.** The code is in your repo. No upstream breaking change can touch a deployed app.
- **Total customizability.** Need a button with a built-in loading spinner and an icon slot? Edit `button.tsx`. No prop-API archaeology.
- **Tiny, transparent footprint.** You ship only the components you added, and you can read every line. Tree-shaking is moot because nothing unused is even in your repo.
- **Tailwind-native.** Styling matches the rest of a Tailwind codebase; designers and devs share one vocabulary.
- **Accessibility for free** via Radix, without inheriting a giant component framework.

**What you give up / the costs**
- **You maintain the code.** Bugs in your copy are yours to fix. There is no `npm update` that silently patches them — you re-pull from the registry and reconcile.
- **No central theme object.** MUI/Chakra give you a single `theme` with deep `sx`/`styled` theming. shadcn theming is CSS variables + Tailwind; powerful but less "one object to rule them all." (See `MATERIAL_UI_GUIDE.md` for the contrast.)
- **More assembly required.** Several "components" (Combobox, Data Table, Date Picker) are *recipes* you compose yourself from primitives, not single imports.
- **You must know Tailwind.** If your team dislikes utility classes, the whole model fights you.

**Rule of thumb:** Reach for shadcn when you want a bespoke, owned design system on a Tailwind codebase and you value control. Reach for MUI/Chakra when you want a batteries-included, centrally-themed library and you value speed-of-defaults over ownership.

### 1.4 The mental model going forward **[B]**

For the rest of this guide, hold these truths:
1. Every component is a *file in your repo* you can open and edit.
2. Every component is *Radix behavior + Tailwind styling + CVA variants*.
3. Most components are **families** of sub-components (`Dialog`, `DialogTrigger`, `DialogContent`…) that you compose.
4. Styling overrides go through `className`, merged by `cn()`.
5. Interactive components run on the **client** — in Next.js App Router they need `"use client"` (§17).

---

## 2. Install & the CLI (`init`, `add`)

### 2.1 Prerequisites **[B]**

shadcn assumes a React project with **Tailwind CSS** and **TypeScript** path aliases (`@/*`) configured. The supported hosts are Next.js, Vite, Astro, Remix/React Router, TanStack Start, Laravel, and Gatsby. The smoothest path in 2026 is a fresh Next.js app:

```bash
# Create a Next.js app (TypeScript + Tailwind v4 wired up by the wizard).
npx create-next-app@latest my-app
cd my-app
```

If you are on Vite instead, install Tailwind v4 first (`npm install tailwindcss @tailwindcss/vite`), add the plugin to `vite.config.ts`, and ensure your `tsconfig` has the `@/*` path alias — the `init` command checks for these.

### 2.2 `init` — set up the project **[B]**

`init` is the one-time bootstrap. It detects your framework, installs the base dependencies (`clsx`, `tailwind-merge`, `class-variance-authority`, `lucide-react`, `tw-animate-css`), creates the `cn()` helper, writes theme tokens into your global CSS, and creates `components.json` — the config file every later `add` command reads.

```bash
# Run the interactive setup wizard.
npx shadcn@latest init
```

The wizard asks a few questions:
- **Base color** — `neutral`, `gray`, `zinc`, `stone`, or `slate`. This seeds your token palette (you can re-theme later).
- (In newer CLIs many older prompts — style, RSC, CSS-variables-yes/no — are decided automatically for the framework.)

What `init` creates / modifies:

```
components.json          # ← the CLI's config + your conventions (see §2.4)
lib/utils.ts             # ← the cn() helper
app/globals.css          # ← @import "tailwindcss" + your CSS-variable theme tokens
package.json             # ← base deps added
```

```ts
// lib/utils.ts — created by init. You will import cn() in every component.
import { clsx, type ClassValue } from "clsx"
import { twMerge } from "tailwind-merge"

// Merge any number of class inputs into ONE conflict-resolved string.
// clsx handles conditionals/arrays/objects; twMerge dedupes Tailwind classes.
export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

### 2.3 `add` — copy components into your repo **[B]**

Once initialized, `add` writes component source files into `components/ui/` (the path is configurable in `components.json`). It also installs any extra npm deps a component needs (e.g. adding `dialog` installs `@radix-ui/react-dialog`).

```bash
# Add a single component.
npx shadcn@latest add button

# Add several at once.
npx shadcn@latest add button card input dialog dropdown-menu

# Add EVERYTHING (handy for exploring; usually you add as needed).
npx shadcn@latest add --all

# Overwrite existing files without the confirm prompt (CAUTION: loses your edits).
npx shadcn@latest add button --overwrite

# Add from a URL / remote registry (your own or a third party).
npx shadcn@latest add https://example.com/registry/fancy-button.json
```

After adding, import from your local alias path — **never** from an npm package:

```tsx
// Correct: import from YOUR copy.
import { Button } from "@/components/ui/button"
// Wrong: there is no "shadcn/ui" package to import from.
```

Running `add` for a component you already have will prompt before overwriting. **This is the update story:** to pull upstream improvements, re-run `add` and reconcile the diff with your local edits (use git to review). There is no `npm update` for components.

### 2.4 `components.json` — the config that drives the CLI **[I]**

This file records your project conventions so the CLI knows where to put files and how to write imports. You rarely edit it after `init`, but understanding it removes mystery.

```jsonc
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",          // visual style preset baked into generated code
  "rsc": true,                  // project uses React Server Components (Next.js)?
  "tsx": true,                  // generate .tsx (vs .jsx)
  "tailwind": {
    "css": "app/globals.css",   // where the theme tokens live
    "baseColor": "neutral",     // seed palette chosen at init
    "cssVariables": true        // use CSS variables for theming (recommended)
  },
  "aliases": {                  // import aliases — must match tsconfig paths
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  },
  "iconLibrary": "lucide"       // default icon set used in generated code
}
```

⚡ **Version note:** the `style` field used to offer `default` vs `new-york`; new-york is the modern default and `default` is deprecated. `cssVariables: true` is strongly recommended — it is what makes dark mode and re-theming a one-place change (§5).

### 2.5 The registry concept **[A]**

The CLI can fetch components from any **registry** — a set of JSON files describing files, dependencies, and tokens. ui.shadcn.com is the default registry, but `add <url>` lets you pull from third-party registries or your own. Teams use this to publish *internal* shared components (a company `<PriceCard>`) installable with one command, code-owned by each consuming app. Defining your own registry is advanced; the takeaway is that "add a component by URL" is a first-class, supported workflow, not a hack.

---

## 3. Anatomy of a Component (button.tsx dissected)

### 3.1 Why dissect the Button **[I]**

The Button is the Rosetta Stone of shadcn. Every other component follows the same construction — CVA for variants, `cn()` for class merging, a Radix primitive (or `Slot`) for behavior, and a ref forwarded to the underlying element. Learn this one file deeply and you can read, edit, and *write* any shadcn component. Here is a representative `components/ui/button.tsx`, annotated line by line.

```tsx
// components/ui/button.tsx
import * as React from "react"
// `Slot` is Radix's "render-as-child" primitive — it merges this component's
// props/behavior ONTO the single child element instead of rendering a <button>.
// This is what powers the `asChild` prop (§4.4).
import { Slot } from "@radix-ui/react-slot"
// CVA: build a function that returns class strings based on variant props.
import { cva, type VariantProps } from "class-variance-authority"
// Our local class-merge helper from §2.2.
import { cn } from "@/lib/utils"

// ── 1. THE VARIANT DEFINITION ───────────────────────────────────────────────
// cva(base, config). The first arg is the ALWAYS-applied base classes.
const buttonVariants = cva(
  // Base: layout, focus ring, disabled state, icon sizing — shared by all buttons.
  "inline-flex items-center justify-center gap-2 whitespace-nowrap rounded-md " +
    "text-sm font-medium transition-colors focus-visible:outline-none " +
    "focus-visible:ring-2 focus-visible:ring-ring disabled:pointer-events-none " +
    "disabled:opacity-50 [&_svg]:size-4 [&_svg]:shrink-0",
  {
    // `variants` maps a prop name → its possible values → the classes for each.
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive:
          "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline:
          "border border-input bg-background hover:bg-accent hover:text-accent-foreground",
        secondary:
          "bg-secondary text-secondary-foreground hover:bg-secondary/80",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "text-primary underline-offset-4 hover:underline",
      },
      size: {
        default: "h-9 px-4 py-2",
        sm: "h-8 rounded-md px-3 text-xs",
        lg: "h-10 rounded-md px-8",
        icon: "h-9 w-9", // square — for icon-only buttons
      },
    },
    // `defaultVariants` are used when the caller omits the prop.
    defaultVariants: { variant: "default", size: "default" },
  }
)

// ── 2. THE PROPS TYPE ────────────────────────────────────────────────────────
// Native <button> props  +  the variant/size props CVA generated  +  asChild.
export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean
}

// ── 3. THE COMPONENT ─────────────────────────────────────────────────────────
// React.forwardRef lets a parent attach a ref to the real DOM <button>.
// (⚡ In React 19, newer generated code may take `ref` as a plain prop and skip
//  forwardRef entirely — both forms are correct; this is the classic form.)
const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    // If asChild, render <Slot> (clone the child); else render a real <button>.
    const Comp = asChild ? Slot : "button"
    return (
      <Comp
        // cn() merges: generated variant classes  +  any className the caller passed.
        // tailwind-merge ensures caller's `bg-blue-500` BEATS the variant's bg.
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        {...props} // spread the rest: onClick, disabled, type, aria-*, etc.
      />
    )
  }
)
Button.displayName = "Button" // nicer name in React DevTools

// Export BOTH the component and the variants fn (so others can reuse the classes).
export { Button, buttonVariants }
```

### 3.2 The four building blocks, named **[I]**

Every shadcn component is some combination of these:

1. **CVA `cva(...)` call** — declares the styling API (`variant`, `size`, …) and maps each to Tailwind classes. Exported so other components (e.g. a `Badge`, or a link styled as a button) can reuse the same class recipe.
2. **`cn(...)` merge** — combines the CVA output with the caller's `className`, letting overrides win.
3. **A Radix primitive or `Slot`** — supplies behavior. The Button uses `Slot` (for `asChild`); a Dialog uses `@radix-ui/react-dialog`; a Select uses `@radix-ui/react-select`, etc.
4. **`forwardRef` (or React 19 ref-as-prop)** — so the consumer can reach the real DOM node (focus it, measure it, anchor a tooltip).

### 3.3 Why `buttonVariants` is exported separately **[I]**

Because the *classes* are useful without the *component*. A common pattern: you want a Next.js `<Link>` that looks exactly like a button. Instead of `<Button asChild>`, you can apply the classes directly:

```tsx
import Link from "next/link"
import { buttonVariants } from "@/components/ui/button"
import { cn } from "@/lib/utils"

// A real <a> (good for SSR/prefetch) that LOOKS like an outline button.
<Link
  href="/pricing"
  className={cn(buttonVariants({ variant: "outline", size: "lg" }))}
>
  See pricing
</Link>
```

This separation of "the styles" from "the element" is a recurring shadcn idiom and a direct payoff of the CVA approach.

---

## 4. `cn()`, `clsx`, tailwind-merge & `asChild`

### 4.1 The problem `cn()` solves **[I]**

Tailwind classes are just strings, and **the last conflicting class does not automatically win** — CSS specificity is equal, so the one defined later in the *stylesheet* wins, not the one written later in your `className`. So if a component renders `class="px-2 px-4"`, the result is unpredictable from the markup. You need a tool that, given two conflicting utilities for the *same* CSS property, keeps the *last* one. That tool is **tailwind-merge**. And to assemble class strings conditionally (objects, arrays, falsy values), you use **clsx**. `cn()` composes both.

```ts
// What cn() actually does, conceptually:
cn("px-2 py-1", condition && "px-4", undefined, ["text-sm"])
// 1. clsx flattens & drops falsy →  "px-2 py-1 px-4 text-sm"
// 2. tailwind-merge resolves conflicts → "py-1 px-4 text-sm"  (px-4 beats px-2)
```

### 4.2 Why this makes overrides "just work" **[I]**

Because components pass `cn(variantClasses, className)` with the caller's `className` **last**, your overrides win even against the component's own defaults — *for the same property*. This is the secret to customizing without `!important` or fighting specificity:

```tsx
// The Button's default is bg-primary. Passing className with bg-blue-600 LAST
// means tailwind-merge drops bg-primary and keeps bg-blue-600. It just works.
<Button className="bg-blue-600 hover:bg-blue-700">Custom blue</Button>

// Non-conflicting classes simply add on:
<Button className="w-full mt-4">Full width</Button>
```

**Gotcha:** tailwind-merge only resolves classes it *recognizes* as conflicting. Arbitrary values, plugin classes, or unusual orderings can occasionally slip past it. When an override mysteriously doesn't win, that is the usual cause — check the produced class string in DevTools.

### 4.3 Using `cn()` in your own components **[I]**

Adopt `cn()` everywhere you accept a `className` prop, so consumers of *your* components get the same override power:

```tsx
function StatCard({ className, ...props }: React.ComponentProps<"div">) {
  return (
    <div
      // Base styles first, caller's className last so it can override.
      className={cn("rounded-xl border bg-card p-6 shadow-sm", className)}
      {...props}
    />
  )
}
```

### 4.4 `asChild` and Radix `Slot` **[I]**

`asChild` is one of the most important and most misunderstood props in the whole ecosystem. By default a component renders its own element (`<Button>` → `<button>`). But sometimes you need the component's *styling and behavior* applied to a **different** element — most often a router `<Link>` (which must render an `<a>` for SEO/prefetch), or to avoid an illegal nesting like `<button>` inside `<button>`.

`asChild` tells the component: "don't render your own tag; instead clone my single child and merge your props onto it." Under the hood this is Radix's `<Slot>`, which merges className, event handlers, refs, and ARIA onto the child.

```tsx
// WITHOUT asChild: renders <button><a>…</a></button>  ← invalid nested interactive.
<Button><Link href="/x">Go</Link></Button>            // ❌

// WITH asChild: renders a single <a class="…button classes…">  ← correct.
<Button asChild>
  <Link href="/x">Go</Link>
</Button>                                              // ✅
```

Rules for `asChild`:
- The child must be a **single** React element (not text, not a fragment of many).
- The child must **forward props/ref** to a DOM node (Next's `Link` and standard elements do).
- Triggers (`DialogTrigger`, `DropdownMenuTrigger`, `TooltipTrigger`, …) almost always want `asChild` so the trigger *is* your button rather than wrapping it.

---

## 5. Theming — CSS Variables, Tokens & Dark Mode

### 5.1 The token model **[I]**

shadcn does not hardcode colors like `bg-zinc-900`. Instead components reference **semantic tokens** — `bg-primary`, `text-muted-foreground`, `border-border` — that resolve to **CSS custom properties** defined once in your global stylesheet. Change the variable, and every component using that token restyles instantly. This indirection is what makes one-line re-theming and automatic dark mode possible.

Every color token comes as a **pair**: a surface color and the readable text color to put *on* it. `--primary` / `--primary-foreground`, `--card` / `--card-foreground`, `--destructive` / `--destructive-foreground`. Always use the matching foreground on a background and contrast is handled for you.

| Token (Tailwind class) | Used for |
|---|---|
| `bg-background` / `text-foreground` | Base page surface and default text. |
| `bg-primary` / `text-primary-foreground` | Primary brand actions (default buttons, active states). |
| `bg-secondary` / `text-secondary-foreground` | Secondary surfaces/buttons. |
| `bg-muted` / `text-muted-foreground` | Subtle fills and de-emphasized text (captions, placeholders). |
| `bg-accent` / `text-accent-foreground` | Hover/active highlights in menus and lists. |
| `bg-card` / `text-card-foreground` | Card surfaces. |
| `bg-popover` / `text-popover-foreground` | Floating surfaces (popovers, dropdowns). |
| `bg-destructive` / `text-destructive-foreground` | Danger/delete actions. |
| `border-border` | Default borders. |
| `ring-ring` | Focus rings. |
| `bg-input` | Input field borders/backgrounds. |

### 5.2 The token definitions (Tailwind v4) **[I]**

⚡ **Version note:** Tailwind **v4** moved theming into CSS. There is no `tailwind.config.js` with a `theme.extend.colors`. Instead `globals.css` declares the variables in `:root` (light) and `.dark` (dark), then maps them to Tailwind color names with `@theme inline`. Colors use **oklch** (perceptually uniform — see `CSS_GUIDE.md` §6). This is what `init` writes:

```css
/* app/globals.css */
@import "tailwindcss";          /* v4: the whole framework in one import */
@import "tw-animate-css";       /* animation utilities used by overlays */

/* Tells Tailwind that `dark:` activates when a .dark ancestor exists. */
@custom-variant dark (&:is(.dark *));

/* ── LIGHT THEME (default) ──────────────────────────────────────────────── */
:root {
  --radius: 0.625rem;                 /* global corner radius */
  --background: oklch(1 0 0);          /* white */
  --foreground: oklch(0.145 0 0);      /* near-black text */
  --primary: oklch(0.205 0 0);
  --primary-foreground: oklch(0.985 0 0);
  --muted: oklch(0.97 0 0);
  --muted-foreground: oklch(0.556 0 0);
  --destructive: oklch(0.577 0.245 27.325);
  --border: oklch(0.922 0 0);
  --ring: oklch(0.708 0 0);
  /* …card, popover, secondary, accent, input, chart-*, sidebar-* … */
}

/* ── DARK THEME ─────────────────────────────────────────────────────────── */
.dark {
  --background: oklch(0.145 0 0);      /* near-black */
  --foreground: oklch(0.985 0 0);      /* near-white */
  --primary: oklch(0.985 0 0);
  --primary-foreground: oklch(0.205 0 0);
  /* …every token re-declared with dark values… */
}

/* ── MAP VARIABLES → TAILWIND COLOR NAMES ───────────────────────────────── */
/* `@theme inline` is what makes `bg-primary`, `text-muted-foreground`, etc.
   exist as real Tailwind utilities pointing at the variables above. */
@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-primary: var(--primary);
  --color-primary-foreground: var(--primary-foreground);
  --color-muted: var(--muted);
  --color-muted-foreground: var(--muted-foreground);
  --color-destructive: var(--destructive);
  --color-border: var(--border);
  --color-ring: var(--ring);
  --radius-lg: var(--radius);
  /* …one mapping per token… */
}
```

To re-brand, change a handful of variables (e.g. set `--primary` to your brand color in both `:root` and `.dark`). Every button, badge, ring, and active state updates. You do **not** edit component files to re-theme — that is the whole point of the token layer.

### 5.3 Dark mode with `next-themes` **[I]**

Dark mode is just "is the `.dark` class on `<html>`?" The `next-themes` library manages that class, persists the choice to `localStorage`, respects the OS preference, and avoids the flash of wrong theme on load.

```bash
npm install next-themes
```

```tsx
// components/theme-provider.tsx
"use client" // next-themes uses context + effects → client component
import { ThemeProvider as NextThemesProvider } from "next-themes"

export function ThemeProvider({ children, ...props }: React.ComponentProps<typeof NextThemesProvider>) {
  return <NextThemesProvider {...props}>{children}</NextThemesProvider>
}
```

```tsx
// app/layout.tsx — wrap the app once.
import { ThemeProvider } from "@/components/theme-provider"

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    // suppressHydrationWarning: next-themes sets the class before hydration,
    // so the server/client html attribute will differ — this silences the warning.
    <html lang="en" suppressHydrationWarning>
      <body>
        <ThemeProvider
          attribute="class"      // toggle by adding/removing the `class` (→ .dark)
          defaultTheme="system"  // follow OS by default
          enableSystem
          disableTransitionOnChange // no color-transition flash when switching
        >
          {children}
        </ThemeProvider>
      </body>
    </html>
  )
}
```

```tsx
// A theme toggle button (client component). useTheme reads/sets the theme.
"use client"
import { Moon, Sun } from "lucide-react"
import { useTheme } from "next-themes"
import { Button } from "@/components/ui/button"

export function ThemeToggle() {
  const { setTheme, theme } = useTheme()
  return (
    <Button
      variant="outline"
      size="icon"
      onClick={() => setTheme(theme === "dark" ? "light" : "dark")}
      aria-label="Toggle theme"
    >
      {/* Show the sun in light mode, moon in dark mode (CSS handles the swap). */}
      <Sun className="size-4 dark:hidden" />
      <Moon className="hidden size-4 dark:block" />
    </Button>
  )
}
```

### 5.4 Best practices for theming **[I]**

- **Never hardcode raw palette classes** (`bg-zinc-800`) in app code — use tokens (`bg-card`) so dark mode and re-branding keep working.
- **Always pair foregrounds with backgrounds** (`bg-primary text-primary-foreground`) for guaranteed contrast.
- **Re-theme in CSS, not in components.** Component files should stay close to upstream so future re-`add`s merge cleanly.
- **Add custom tokens** for brand needs (e.g. `--success`/`--success-foreground`) by declaring them in `:root`/`.dark` and mapping them in `@theme inline`, then use `bg-success` everywhere.

---

## 6. Forms & Inputs Reference

> Every entry below explains *what the component is, when to use it, its key props,* and shows commented usage. These render styled, accessible inputs; §7 then shows the full validated-form pattern that ties them together.

### Input **[B]**
A single-line text field — a styled native `<input>`. **Use for:** text, email, password, search, number, tel, url fields. **Key props:** all native `<input>` attributes (`type`, `placeholder`, `value`/`defaultValue`, `onChange`, `disabled`, `readOnly`, `required`) plus `className`. Because it forwards every native prop, it drops straight into react-hook-form via `{...field}`.

```tsx
import { Input } from "@/components/ui/input"
// Controlled email input. `type` changes browser keyboard + validation hints.
<Input type="email" placeholder="you@example.com" value={email}
       onChange={(e) => setEmail(e.target.value)} />
```

### Textarea **[B]**
Multi-line text input — a styled native `<textarea>`. **Use for:** messages, descriptions, comments, bios. **Key props:** native `<textarea>` props (`rows`, `placeholder`, `value`, `onChange`, `disabled`) + `className`. For auto-growing height you add a small effect or a CSS field-sizing trick yourself.

```tsx
import { Textarea } from "@/components/ui/textarea"
<Textarea placeholder="Type your message…" rows={4} />
```

### Label **[B]**
An accessible label tied to a control (built on `@radix-ui/react-label`). **Use for:** labeling any input; clicking the label focuses the field, and screen readers announce it. **Key props:** `htmlFor` (must equal the input's `id`). In the Form pattern, `FormLabel` wraps this and wires the association automatically.

```tsx
import { Label } from "@/components/ui/label"
<Label htmlFor="email">Email</Label>
<Input id="email" type="email" />
```

### Checkbox **[B]**
Toggle a single boolean (Radix checkbox). **Use for:** "agree to terms," opt-ins, multi-select rows. **Key props:** `checked`, `defaultChecked`, `onCheckedChange` (receives `boolean | "indeterminate"`), `disabled`. Note `onCheckedChange` (not `onChange`) and the indeterminate state.

```tsx
import { Checkbox } from "@/components/ui/checkbox"
// Coerce to boolean since the value can be "indeterminate".
<Checkbox id="terms" onCheckedChange={(v) => setAgreed(v === true)} />
<Label htmlFor="terms">Accept terms</Label>
```

### Radio Group **[B]**
Choose exactly ONE option from several (Radix radio group). **Use for:** mutually-exclusive choices (plan tiers, yes/no, shipping speed). **Key props:** on the group: `value`/`defaultValue`, `onValueChange`; on each `RadioGroupItem`: `value`, `id`, `disabled`.

```tsx
import { RadioGroup, RadioGroupItem } from "@/components/ui/radio-group"
<RadioGroup defaultValue="month" onValueChange={setPlan}>
  <div className="flex items-center gap-2">
    <RadioGroupItem value="month" id="m" /><Label htmlFor="m">Monthly</Label>
  </div>
  <div className="flex items-center gap-2">
    <RadioGroupItem value="year" id="y" /><Label htmlFor="y">Yearly</Label>
  </div>
</RadioGroup>
```

### Switch **[B]**
An on/off toggle styled like an iOS switch (Radix switch). **Use for:** boolean *settings* that apply immediately (notifications on, public profile). Prefer a Checkbox inside forms; prefer a Switch for instant settings. **Key props:** `checked`, `onCheckedChange`, `disabled`.

```tsx
import { Switch } from "@/components/ui/switch"
<Switch checked={notify} onCheckedChange={setNotify} aria-label="Notifications" />
```

### Select **[I]**
A dropdown to pick one value from a list (Radix select — accessible, keyboard-navigable, portaled). **Use for:** country, category, sort order — moderate option counts. For long, *searchable* lists use a Combobox instead. **Family:** `Select` > `SelectTrigger` > `SelectValue` + `SelectContent` > `SelectItem` (and optional `SelectGroup`, `SelectLabel`, `SelectSeparator`). **Key props:** `value`, `defaultValue`, `onValueChange` on `Select`.

```tsx
import { Select, SelectTrigger, SelectValue, SelectContent, SelectItem } from "@/components/ui/select"
<Select onValueChange={setFruit}>
  <SelectTrigger className="w-48">
    {/* SelectValue shows the chosen item or the placeholder. */}
    <SelectValue placeholder="Pick a fruit" />
  </SelectTrigger>
  <SelectContent>
    <SelectItem value="apple">Apple</SelectItem>
    <SelectItem value="banana">Banana</SelectItem>
  </SelectContent>
</Select>
```

### Combobox **[I]**
A *searchable / filterable* select. **There is no `Combobox` import** — it is a **recipe** you compose from `Popover` + `Command` + a `Button` trigger. **Use for:** long option lists where typing-to-filter helps (users, repos, tags, framework pickers). Because you build it, you control the filtering, empty state, and item rendering. See `Command` below for the inner piece.

```tsx
// Skeleton of the recipe (state + handlers omitted for brevity).
<Popover open={open} onOpenChange={setOpen}>
  <PopoverTrigger asChild>
    <Button variant="outline" role="combobox">{label ?? "Select…"}</Button>
  </PopoverTrigger>
  <PopoverContent className="p-0">
    <Command>
      <CommandInput placeholder="Search…" />
      <CommandList>
        <CommandEmpty>No results.</CommandEmpty>
        <CommandGroup>
          {items.map((it) => (
            <CommandItem key={it.value} value={it.value}
              onSelect={(v) => { setValue(v); setOpen(false) }}>
              {it.label}
            </CommandItem>
          ))}
        </CommandGroup>
      </CommandList>
    </Command>
  </PopoverContent>
</Popover>
```

### Command **[I]**
A command palette / fuzzy-search menu (⌘K style), built on the `cmdk` library. **Use for:** quick-search launchers, autocomplete menus, the inner engine of a Combobox. **Family:** `Command` > `CommandInput`, `CommandList`, `CommandEmpty`, `CommandGroup`, `CommandItem`, `CommandSeparator`, `CommandShortcut`. Wrap it in `CommandDialog` to make it a global ⌘K overlay. **Key props:** `CommandItem` takes `value` and `onSelect`; filtering is automatic on the items' text/value.

```tsx
import { Command, CommandInput, CommandList, CommandEmpty, CommandGroup, CommandItem } from "@/components/ui/command"
<Command className="rounded-lg border shadow-md">
  <CommandInput placeholder="Type a command or search…" />
  <CommandList>
    <CommandEmpty>No results found.</CommandEmpty>
    <CommandGroup heading="Pages">
      <CommandItem onSelect={() => router.push("/dashboard")}>Dashboard</CommandItem>
      <CommandItem onSelect={() => router.push("/settings")}>Settings</CommandItem>
    </CommandGroup>
  </CommandList>
</Command>
```

### Input OTP **[I]**
A segmented one-time-password / PIN input (built on `input-otp`). **Use for:** 2FA codes, email/phone verification PINs. **Family:** `InputOTP` > `InputOTPGroup` > `InputOTPSlot` (and `InputOTPSeparator`). **Key props:** `maxLength`, `value`, `onChange` on `InputOTP`; each `InputOTPSlot` takes its `index`.

```tsx
import { InputOTP, InputOTPGroup, InputOTPSlot } from "@/components/ui/input-otp"
<InputOTP maxLength={6} value={code} onChange={setCode}>
  <InputOTPGroup>
    {[0, 1, 2, 3, 4, 5].map((i) => <InputOTPSlot key={i} index={i} />)}
  </InputOTPGroup>
</InputOTP>
```

### Slider **[I]**
Drag a handle to pick a number in a range (Radix slider). **Use for:** volume, price range, opacity, zoom. **Key props:** `min`, `max`, `step`, `value`/`defaultValue` (**arrays** — pass two values for a range slider), `onValueChange`.

```tsx
import { Slider } from "@/components/ui/slider"
<Slider defaultValue={[50]} max={100} step={1} />          // single handle
<Slider defaultValue={[20, 80]} max={100} step={1} />      // range (two handles)
```

### Calendar **[I]**
An inline date-picker grid (built on `react-day-picker`). **Use for:** standalone date selection and as the base of a Date Picker. **Key props:** `mode` (`"single"` | `"multiple"` | `"range"`), `selected`, `onSelect`, `disabled` (a date or matcher fn). It is the calendar surface only — wrap it in a Popover for a field-style picker.

```tsx
import { Calendar } from "@/components/ui/calendar"
<Calendar mode="single" selected={date} onSelect={setDate} className="rounded-md border" />
```

### Date Picker **[I]**
A calendar inside a Popover with a button trigger that shows the chosen date. **No single import** — compose `Popover` + `Calendar` + `Button` (often with `date-fns` for formatting). **Use for:** form date fields, "due date" pickers.

```tsx
import { format } from "date-fns"
<Popover>
  <PopoverTrigger asChild>
    <Button variant="outline">
      <CalendarIcon className="mr-2 size-4" />
      {date ? format(date, "PPP") : "Pick a date"}
    </Button>
  </PopoverTrigger>
  <PopoverContent className="w-auto p-0">
    <Calendar mode="single" selected={date} onSelect={setDate} initialFocus />
  </PopoverContent>
</Popover>
```

### Form **[I/A]**
The wrapper that integrates **react-hook-form** + **zod** with accessible, error-aware markup. **Use for:** any real form with validation and error messages. **Family:** `Form` > `FormField` > `FormItem` > `FormLabel`, `FormControl`, `FormDescription`, `FormMessage`. This is important enough to get its own section — see [§7](#7-the-form-pattern-react-hook-form--zod-in-depth).

---

## 7. The Form Pattern (react-hook-form + zod) In Depth

### 7.1 Why this pattern exists **[I]**

Building a *correct* form by hand is deceptively hard: you must track each field's value, validate it, show errors, associate labels and error messages with inputs for screen readers (`aria-describedby`, `aria-invalid`), focus the first invalid field, and avoid re-rendering the whole form on every keystroke. shadcn's `Form` solves all of this by gluing together two best-in-class libraries:

- **react-hook-form (RHF)** — manages form state with *uncontrolled* inputs and refs, so typing in one field does not re-render the others. Fast, minimal re-renders. (Deep dive in `REACT_HOOK_FORM_GUIDE.md`.)
- **zod** — a schema/validation library. You describe the shape and rules of your data once; zod validates at runtime *and* infers the TypeScript type for you, so your form values are fully typed.
- **`@hookform/resolvers/zod`** — the bridge that lets RHF use a zod schema as its validator.

shadcn's `Form` components are a thin accessibility layer on top: they generate the `id`/`aria-*` wiring and auto-render the active error message in `FormMessage`. You write the schema and the fields; everything else is handled.

### 7.2 Install **[B]**

```bash
# The Form component pulls these in, but install explicitly if needed:
npm install react-hook-form zod @hookform/resolvers
npx shadcn@latest add form input button   # add the UI pieces
```

### 7.3 The full, annotated pattern **[I]**

```tsx
"use client" // RHF uses hooks + state → must be a Client Component (§17).

import { z } from "zod"
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import {
  Form, FormField, FormItem, FormLabel,
  FormControl, FormDescription, FormMessage,
} from "@/components/ui/form"
import { Input } from "@/components/ui/input"
import { Button } from "@/components/ui/button"

// ── 1. SCHEMA: the single source of truth for shape + rules + types. ─────────
const schema = z.object({
  username: z.string().min(2, "At least 2 characters").max(30),
  email: z.string().email("Enter a valid email"),
  // .refine() for cross-field or custom rules:
  age: z.coerce.number().int().min(18, "Must be 18 or older"),
})

// Infer the TS type FROM the schema — never write the type twice.
type FormValues = z.infer<typeof schema>

export function SignupForm() {
  // ── 2. THE FORM INSTANCE ───────────────────────────────────────────────
  const form = useForm<FormValues>({
    resolver: zodResolver(schema), // validate against the zod schema
    defaultValues: { username: "", email: "", age: 18 }, // controlled-from-start
  })

  // ── 3. SUBMIT HANDLER — only runs if validation PASSES. ─────────────────
  // `values` is fully typed as FormValues.
  async function onSubmit(values: FormValues) {
    // e.g. await fetch("/api/signup", { method: "POST", body: JSON.stringify(values) })
    console.log(values)
  }

  return (
    // Spread the form instance into <Form> so child fields can read context.
    <Form {...form}>
      {/* handleSubmit validates, then calls onSubmit only on success. */}
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        <FormField
          control={form.control}      // connect this field to the form
          name="username"             // must match a schema key (typed!)
          render={({ field }) => (     // `field` = { value, onChange, onBlur, ref, name }
            <FormItem>
              <FormLabel>Username</FormLabel>
              <FormControl>
                {/* Spreading {...field} wires value/onChange/onBlur/ref + ARIA. */}
                <Input placeholder="jane_doe" {...field} />
              </FormControl>
              <FormDescription>Your public display name.</FormDescription>
              <FormMessage /> {/* auto-renders this field's zod error, if any */}
            </FormItem>
          )}
        />

        <FormField control={form.control} name="email" render={({ field }) => (
          <FormItem>
            <FormLabel>Email</FormLabel>
            <FormControl><Input type="email" placeholder="you@example.com" {...field} /></FormControl>
            <FormMessage />
          </FormItem>
        )} />

        {/* disable submit while submitting to prevent double-posts */}
        <Button type="submit" disabled={form.formState.isSubmitting}>
          {form.formState.isSubmitting ? "Creating…" : "Create account"}
        </Button>
      </form>
    </Form>
  )
}
```

### 7.4 What each Form part does **[I]**

| Part | Responsibility |
|---|---|
| `<Form {...form}>` | A context provider — passes the RHF instance down so fields self-wire. It is essentially RHF's `FormProvider`. |
| `<FormField name control render>` | A render-prop wrapper around RHF's `Controller`. Gives the render fn a `field` object you spread onto the input. |
| `<FormItem>` | Groups one field's parts and generates a unique `id` used to link label/control/message via ARIA. |
| `<FormLabel>` | A `Label` auto-pointed at the control; turns red when the field is invalid. |
| `<FormControl>` | Wraps the actual input and injects `id`, `aria-describedby`, `aria-invalid`. |
| `<FormDescription>` | Helper text, linked to the input via `aria-describedby`. |
| `<FormMessage>` | Renders the field's current validation error automatically (nothing to wire). |

### 7.5 Using non-text fields in a Form **[I/A]**

For Select, Checkbox, Switch, RadioGroup, Slider, etc., you cannot blindly spread `{...field}` (they use `onValueChange`/`onCheckedChange`, not `onChange`). Map the `field` props explicitly:

```tsx
// Select inside a Form: bridge value + onValueChange manually.
<FormField control={form.control} name="role" render={({ field }) => (
  <FormItem>
    <FormLabel>Role</FormLabel>
    <Select onValueChange={field.onChange} defaultValue={field.value}>
      <FormControl>
        <SelectTrigger><SelectValue placeholder="Choose a role" /></SelectTrigger>
      </FormControl>
      <SelectContent>
        <SelectItem value="admin">Admin</SelectItem>
        <SelectItem value="user">User</SelectItem>
      </SelectContent>
    </Select>
    <FormMessage />
  </FormItem>
)} />

// Checkbox inside a Form: checked + onCheckedChange.
<FormField control={form.control} name="terms" render={({ field }) => (
  <FormItem className="flex items-center gap-2">
    <FormControl>
      <Checkbox checked={field.value} onCheckedChange={field.onChange} />
    </FormControl>
    <FormLabel>I accept the terms</FormLabel>
    <FormMessage />
  </FormItem>
)} />
```

### 7.6 Server-side validation & errors **[A]**

zod schemas are isomorphic — reuse the *same* schema on the server (a Next.js Route Handler or Server Action) to validate the incoming body, so you never trust the client. To surface a server error on a specific field, call `form.setError("email", { message: "Already taken" })` after the request returns. Reuse one schema, validate twice, type once. See `NEXTJS_16_GUIDE.md` for Server Actions and `REACT_HOOK_FORM_GUIDE.md` for advanced RHF (`watch`, `useFieldArray`, `trigger`).

**Best practices:** keep one schema as the source of truth; always set `defaultValues` (avoids uncontrolled→controlled warnings); disable the submit button with `formState.isSubmitting`; never spread `{...field}` onto components that don't use a native `onChange`.

---

## 8. Buttons & Actions Reference

### Button **[B]**
The workhorse action element (dissected fully in §3). **Use for:** submits, triggers, and — via `asChild` — links. **Key props:** `variant` (`default` | `destructive` | `outline` | `secondary` | `ghost` | `link`), `size` (`default` | `sm` | `lg` | `icon`), `asChild`, `disabled`, plus all native button props. For an icon-only button use `size="icon"` and add an `aria-label`.

```tsx
import { Button } from "@/components/ui/button"
import { Plus } from "lucide-react"
<Button>Save</Button>
<Button variant="destructive" size="sm">Delete</Button>
<Button variant="outline" size="icon" aria-label="Add"><Plus /></Button>
<Button variant="ghost" disabled>Loading…</Button>
<Button asChild><a href="/signup">Sign up</a></Button> {/* render as a link */}
```

### Toggle **[B]**
A button that holds an on/off *pressed* state (Radix toggle). **Use for:** toolbar buttons like bold/italic, a single sticky toggle. **Key props:** `pressed`, `defaultPressed`, `onPressedChange`, `variant` (`default` | `outline`), `size`, `disabled`. Add an `aria-label` for icon-only toggles.

```tsx
import { Toggle } from "@/components/ui/toggle"
import { Bold } from "lucide-react"
<Toggle aria-label="Bold" pressed={bold} onPressedChange={setBold}>
  <Bold className="size-4" />
</Toggle>
```

### Toggle Group **[I]**
A set of toggles where one (`single`) or many (`multiple`) can be active (Radix toggle-group). **Use for:** text-align pickers, view switchers (grid/list), formatting toolbars. **Key props:** `type` (`"single"` | `"multiple"`), `value`/`defaultValue`, `onValueChange`, `variant`, `size`.

```tsx
import { ToggleGroup, ToggleGroupItem } from "@/components/ui/toggle-group"
import { AlignLeft, AlignCenter, AlignRight } from "lucide-react"
<ToggleGroup type="single" value={align} onValueChange={setAlign}>
  <ToggleGroupItem value="left" aria-label="Left"><AlignLeft /></ToggleGroupItem>
  <ToggleGroupItem value="center" aria-label="Center"><AlignCenter /></ToggleGroupItem>
  <ToggleGroupItem value="right" aria-label="Right"><AlignRight /></ToggleGroupItem>
</ToggleGroup>
```

---

## 9. Overlays & Dialogs Reference

> All overlays in this section **portal to the document body** by default (so they escape `overflow: hidden`/`z-index` traps) and are powered by Radix, which handles focus-trapping, `Esc`-to-close, scroll-locking, and ARIA. They are interactive → **client components** in Next.js (§17).

### Dialog **[I]**
A modal window centered over a dimmed overlay (Radix dialog). **Use for:** confirmations, focused forms, detail views. **Family:** `Dialog` > `DialogTrigger` + `DialogContent` > `DialogHeader` (`DialogTitle`, `DialogDescription`), `DialogFooter`, `DialogClose`. **Key props:** `open` + `onOpenChange` (omit to run uncontrolled); `DialogTrigger asChild`. **Accessibility:** always include a `DialogTitle` (Radix warns without it) — visually hide it with `sr-only` if your design has no visible title.

```tsx
import { Dialog, DialogTrigger, DialogContent, DialogHeader, DialogTitle, DialogDescription, DialogFooter, DialogClose } from "@/components/ui/dialog"
<Dialog>
  <DialogTrigger asChild><Button>Edit profile</Button></DialogTrigger>
  <DialogContent>
    <DialogHeader>
      <DialogTitle>Edit profile</DialogTitle>
      <DialogDescription>Make changes and save.</DialogDescription>
    </DialogHeader>
    {/* form goes here */}
    <DialogFooter>
      <DialogClose asChild><Button variant="outline">Cancel</Button></DialogClose>
      <Button type="submit">Save</Button>
    </DialogFooter>
  </DialogContent>
</Dialog>
```

### Alert Dialog **[I]**
A modal that **demands a deliberate choice** and cannot be dismissed by clicking outside or pressing `Esc` away (Radix alert-dialog). **Use for:** destructive/irreversible confirmations ("Delete this account?"). **Family:** `AlertDialog` > `AlertDialogTrigger` + `AlertDialogContent` > `AlertDialogHeader` (`AlertDialogTitle`, `AlertDialogDescription`), `AlertDialogFooter` > `AlertDialogAction`, `AlertDialogCancel`. The difference from Dialog is intent: this is for "are you sure?" gates, not general content.

```tsx
import { AlertDialog, AlertDialogTrigger, AlertDialogContent, AlertDialogHeader, AlertDialogTitle, AlertDialogDescription, AlertDialogFooter, AlertDialogAction, AlertDialogCancel } from "@/components/ui/alert-dialog"
<AlertDialog>
  <AlertDialogTrigger asChild><Button variant="destructive">Delete</Button></AlertDialogTrigger>
  <AlertDialogContent>
    <AlertDialogHeader>
      <AlertDialogTitle>Are you absolutely sure?</AlertDialogTitle>
      <AlertDialogDescription>This action cannot be undone.</AlertDialogDescription>
    </AlertDialogHeader>
    <AlertDialogFooter>
      <AlertDialogCancel>Cancel</AlertDialogCancel>
      <AlertDialogAction onClick={handleDelete}>Delete</AlertDialogAction>
    </AlertDialogFooter>
  </AlertDialogContent>
</AlertDialog>
```

### Sheet **[I]**
A panel that slides in from a screen edge (Radix dialog under the hood). **Use for:** mobile nav menus, filter panels, side forms, detail drawers. **Key props:** `side` (`"top"` | `"right"` | `"bottom"` | `"left"`), `open`, `onOpenChange`. **Family:** `Sheet` > `SheetTrigger` + `SheetContent` > `SheetHeader` (`SheetTitle`, `SheetDescription`), `SheetFooter`, `SheetClose`.

```tsx
import { Sheet, SheetTrigger, SheetContent, SheetHeader, SheetTitle } from "@/components/ui/sheet"
<Sheet>
  <SheetTrigger asChild><Button variant="outline">Open menu</Button></SheetTrigger>
  <SheetContent side="left">
    <SheetHeader><SheetTitle>Navigation</SheetTitle></SheetHeader>
    {/* nav links */}
  </SheetContent>
</Sheet>
```

### Drawer **[I]**
A bottom drawer optimized for touch, with drag-to-dismiss (built on **Vaul**). **Use for:** mobile-first sheets and action menus where a native-feeling swipe matters. **Family:** `Drawer` > `DrawerTrigger` + `DrawerContent` > `DrawerHeader` (`DrawerTitle`, `DrawerDescription`), `DrawerFooter`, `DrawerClose`. A common responsive pattern: render a `Dialog` on desktop and a `Drawer` on mobile (switch on a media-query hook).

### Popover **[I]**
A floating panel anchored to a trigger, dismissed by clicking outside (Radix popover). **Use for:** extra options, mini-forms, color pickers, the shell of a Combobox/Date Picker. **Family:** `Popover` > `PopoverTrigger` + `PopoverContent`. **Key props:** on content: `align` (`start`/`center`/`end`), `side`, `sideOffset`.

```tsx
import { Popover, PopoverTrigger, PopoverContent } from "@/components/ui/popover"
<Popover>
  <PopoverTrigger asChild><Button variant="outline">Options</Button></PopoverTrigger>
  <PopoverContent className="w-80">Place content here.</PopoverContent>
</Popover>
```

### Hover Card **[I]**
A popover that opens on hover/focus after a delay (Radix hover-card). **Use for:** rich previews — profile cards, link/footnote previews — on **desktop only** (hover doesn't exist on touch). **Family:** `HoverCard` > `HoverCardTrigger` + `HoverCardContent`. **Key props:** `openDelay`, `closeDelay`.

### Tooltip **[I]**
A tiny label shown on hover/focus (Radix tooltip). **Use for:** explaining icon-only buttons; never put essential info or interactive content in one. **Setup:** wrap the app (or a section) in `TooltipProvider` once, then use `Tooltip` > `TooltipTrigger` + `TooltipContent`. ⚡ **Version note:** newer shadcn versions include the provider inside the component so you may not need to add it manually — but a single top-level `TooltipProvider` is still the safe convention.

```tsx
import { TooltipProvider, Tooltip, TooltipTrigger, TooltipContent } from "@/components/ui/tooltip"
import { Info } from "lucide-react"
<TooltipProvider>
  <Tooltip>
    <TooltipTrigger asChild>
      <Button size="icon" variant="ghost" aria-label="Info"><Info /></Button>
    </TooltipTrigger>
    <TooltipContent>More information</TooltipContent>
  </Tooltip>
</TooltipProvider>
```

### Dropdown Menu **[I]**
A click-triggered menu of actions (Radix dropdown-menu). **Use for:** "⋯" row menus, account menus, contextual actions. **Family:** `DropdownMenu` > `DropdownMenuTrigger` + `DropdownMenuContent` > `DropdownMenuItem`, `DropdownMenuLabel`, `DropdownMenuSeparator`, `DropdownMenuCheckboxItem`, `DropdownMenuRadioGroup`/`DropdownMenuRadioItem`, `DropdownMenuSub`/`DropdownMenuSubTrigger`/`DropdownMenuSubContent`, `DropdownMenuShortcut`. **Gotcha:** putting a Dialog/AlertDialog trigger directly inside a menu item is fiddly because both manage focus — control the dialog's `open` from state instead.

```tsx
import { DropdownMenu, DropdownMenuTrigger, DropdownMenuContent, DropdownMenuItem, DropdownMenuLabel, DropdownMenuSeparator } from "@/components/ui/dropdown-menu"
import { MoreHorizontal } from "lucide-react"
<DropdownMenu>
  <DropdownMenuTrigger asChild>
    <Button variant="ghost" size="icon" aria-label="Actions"><MoreHorizontal /></Button>
  </DropdownMenuTrigger>
  <DropdownMenuContent align="end">
    <DropdownMenuLabel>My account</DropdownMenuLabel>
    <DropdownMenuSeparator />
    <DropdownMenuItem onSelect={() => router.push("/profile")}>Profile</DropdownMenuItem>
    <DropdownMenuItem onSelect={logout}>Log out</DropdownMenuItem>
  </DropdownMenuContent>
</DropdownMenu>
```

### Context Menu **[I]**
A right-click (context) menu (Radix context-menu). **Use for:** desktop-style right-click actions on rows, canvas items, files. **Family:** mirrors Dropdown Menu — `ContextMenu`, `ContextMenuTrigger` (wraps the right-clickable area), `ContextMenuContent`, `ContextMenuItem`, etc.

### Menubar **[I]**
A desktop-app-style horizontal menu bar (Radix menubar). **Use for:** editor/IDE-style top menus (File, Edit, View). **Family:** `Menubar` > `MenubarMenu` > `MenubarTrigger` + `MenubarContent` > `MenubarItem`, `MenubarSeparator`, `MenubarSub`, `MenubarShortcut`.

---

## 10. Navigation Reference

### Navigation Menu **[I]**
An accessible top-navigation bar with dropdown/mega-menu panels (Radix navigation-menu). **Use for:** marketing-site headers with grouped link panels. **Family:** `NavigationMenu` > `NavigationMenuList` > `NavigationMenuItem` > `NavigationMenuTrigger` + `NavigationMenuContent` (or a direct `NavigationMenuLink`). Heavier than a Dropdown Menu — reserve it for true site nav, not action menus.

### Tabs **[B]**
Switch between panels occupying the same space (Radix tabs). **Use for:** settings sections, product detail tabs, dashboard views. **Family:** `Tabs` > `TabsList` > `TabsTrigger value` + `TabsContent value` (triggers and content link by matching `value`). **Key props:** `defaultValue` (uncontrolled) or `value` + `onValueChange` (controlled), `orientation`.

```tsx
import { Tabs, TabsList, TabsTrigger, TabsContent } from "@/components/ui/tabs"
<Tabs defaultValue="account">
  <TabsList>
    <TabsTrigger value="account">Account</TabsTrigger>
    <TabsTrigger value="password">Password</TabsTrigger>
  </TabsList>
  <TabsContent value="account">Account settings…</TabsContent>
  <TabsContent value="password">Password settings…</TabsContent>
</Tabs>
```

### Breadcrumb **[B]**
Path/hierarchy navigation showing where the user is. **Use for:** "Home / Products / Shoes" trails. **Family:** `Breadcrumb` > `BreadcrumbList` > `BreadcrumbItem` > `BreadcrumbLink` (links) / `BreadcrumbPage` (current page, non-link), with `BreadcrumbSeparator` and `BreadcrumbEllipsis`. Use `asChild` on `BreadcrumbLink` to render a Next `<Link>`.

### Pagination **[I]**
Page-number navigation for paged data. **Use for:** paged tables and lists. **Family:** `Pagination` > `PaginationContent` > `PaginationItem` > `PaginationLink`, `PaginationPrevious`, `PaginationNext`, `PaginationEllipsis`. These render anchors by default; in an SPA, wire `onClick`/`asChild` to your router and current-page state — the component is presentational, you own the paging logic.

### Sidebar **[A]**
A complete, composable application sidebar system — collapsible, mobile-aware (becomes a Sheet on small screens), keyboard-toggleable, and state-persisted via cookie. **Use for:** dashboard/app navigation shells. **Family:** `SidebarProvider` (wraps the layout) > `Sidebar` > `SidebarHeader`, `SidebarContent` (`SidebarGroup`, `SidebarGroupLabel`, `SidebarMenu`, `SidebarMenuItem`, `SidebarMenuButton`, `SidebarMenuSub`), `SidebarFooter`; plus `SidebarTrigger` (toggle) and `SidebarInset` (the main content area beside it). This is the most "batteries-included" piece in the catalog and a strong differentiator for dashboard work.

```tsx
// Minimal shell.
<SidebarProvider>
  <Sidebar>
    <SidebarContent>
      <SidebarGroup>
        <SidebarMenu>
          <SidebarMenuItem>
            <SidebarMenuButton asChild><a href="/">Home</a></SidebarMenuButton>
          </SidebarMenuItem>
        </SidebarMenu>
      </SidebarGroup>
    </SidebarContent>
  </Sidebar>
  <SidebarInset>
    <SidebarTrigger /> {/* toggles the sidebar */}
    {/* page content */}
  </SidebarInset>
</SidebarProvider>
```

---

## 11. Data Display Reference

### Card **[B]**
A bordered container that groups related content (a plain styled `<div>` family, no Radix). **Use for:** stats, products, pricing tiers, dashboard widgets, form containers. **Family:** `Card` > `CardHeader` (`CardTitle`, `CardDescription`, `CardAction`), `CardContent`, `CardFooter`. Style internals with Tailwind via `className`.

```tsx
import { Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter } from "@/components/ui/card"
<Card>
  <CardHeader>
    <CardTitle>Total revenue</CardTitle>
    <CardDescription>This month</CardDescription>
  </CardHeader>
  <CardContent className="text-3xl font-bold">$12,400</CardContent>
  <CardFooter className="text-sm text-muted-foreground">+12% vs last month</CardFooter>
</Card>
```

### Table **[I]**
A styled static HTML table (plain markup, no Radix). **Use for:** simple, presentational tabular data. **Family:** `Table` > `TableHeader` > `TableRow` > `TableHead`; `TableBody` > `TableRow` > `TableCell`; plus `TableCaption` and `TableFooter`. For sorting/filtering/pagination/selection you graduate to the Data Table recipe below.

```tsx
import { Table, TableHeader, TableBody, TableRow, TableHead, TableCell } from "@/components/ui/table"
<Table>
  <TableHeader><TableRow><TableHead>Name</TableHead><TableHead>Email</TableHead></TableRow></TableHeader>
  <TableBody>
    {users.map((u) => (
      <TableRow key={u.id}><TableCell>{u.name}</TableCell><TableCell>{u.email}</TableCell></TableRow>
    ))}
  </TableBody>
</Table>
```

### Data Table **[A]**
A powerful data grid built by combining the `Table` components with **@tanstack/react-table** (headless sorting, filtering, pagination, row selection, column visibility). **No single import** — it is a recipe you assemble with `useReactTable`, `getCoreRowModel`, `getSortedRowModel`, etc., plus a column definitions array. **Use for:** admin dashboards and real data grids. Pairs naturally with **TanStack Query** for server data and server-side pagination (see `TANSTACK_QUERY_GUIDE.md`). A big value-add for dashboard projects.

```tsx
// The core wiring (column defs and toolbar omitted for brevity).
const table = useReactTable({
  data, columns,
  getCoreRowModel: getCoreRowModel(),
  getSortedRowModel: getSortedRowModel(),
  getPaginationRowModel: getPaginationRowModel(),
})
// Then render table.getHeaderGroups() / table.getRowModel().rows into <Table>.
```

### Avatar **[B]**
A user profile image with an initials fallback if the image fails or is missing (Radix avatar). **Use for:** account/user pictures, comment authors, member lists. **Family:** `Avatar` > `AvatarImage` + `AvatarFallback`. The fallback shows automatically when the image can't load — always provide meaningful initials.

```tsx
import { Avatar, AvatarImage, AvatarFallback } from "@/components/ui/avatar"
<Avatar>
  <AvatarImage src="/me.jpg" alt="Jane Doe" />
  <AvatarFallback>JD</AvatarFallback>
</Avatar>
```

### Badge **[B]**
A small status/category pill (a styled span with CVA variants). **Use for:** tags, counts, statuses (active/draft/error). **Key props:** `variant` (`default` | `secondary` | `destructive` | `outline`), `asChild` (render as a link). Like Button, it exports `badgeVariants` for reuse.

```tsx
import { Badge } from "@/components/ui/badge"
<Badge>Active</Badge>
<Badge variant="secondary">New</Badge>
<Badge variant="destructive">Error</Badge>
```

### Accordion **[B]**
Collapsible stacked sections (Radix accordion). **Use for:** FAQs, grouped settings, progressive disclosure. **Key props:** `type` (`"single"` | `"multiple"`), `collapsible` (allow closing the open one in single mode), `defaultValue`/`value`. **Family:** `Accordion` > `AccordionItem value` > `AccordionTrigger` + `AccordionContent`.

```tsx
import { Accordion, AccordionItem, AccordionTrigger, AccordionContent } from "@/components/ui/accordion"
<Accordion type="single" collapsible>
  <AccordionItem value="q1">
    <AccordionTrigger>Do I own the code?</AccordionTrigger>
    <AccordionContent>Yes — it lives in your repo and you can edit it freely.</AccordionContent>
  </AccordionItem>
</Accordion>
```

### Collapsible **[B]**
A single show/hide region (Radix collapsible). **Use for:** "show more" toggles, expandable rows. **Family:** `Collapsible` > `CollapsibleTrigger` + `CollapsibleContent`. **Key props:** `open`, `onOpenChange`, `defaultOpen`. (Accordion is the multi-section version of this.)

### Carousel **[I]**
A sliding content slider (built on **Embla Carousel**). **Use for:** image galleries, testimonials, product showcases. **Family:** `Carousel` > `CarouselContent` > `CarouselItem` + `CarouselPrevious`, `CarouselNext`. **Key props:** `opts` (Embla options like `loop`, `align`), `orientation`, `setApi` (grab the Embla API for custom controls/autoplay).

### Chart **[I]**
Composable charts wrapping **Recharts** with theme-aware (token-driven) styling. **Use for:** dashboards, analytics, reports. **Family:** `ChartContainer` (provides the `config` mapping series → label + color) + `ChartTooltip`/`ChartTooltipContent`, `ChartLegend`/`ChartLegendContent`, wrapped around Recharts elements (`Bar`, `Line`, `Area`, `Pie`). Colors come from `--chart-1..5` tokens so charts match light/dark automatically. A strong dashboard differentiator.

### Aspect Ratio **[B]**
Locks content (image/video/iframe) to a fixed width:height ratio (Radix aspect-ratio). **Use for:** responsive media that must not cause layout shift. **Key props:** `ratio` (e.g. `16 / 9`). (Plain CSS `aspect-ratio` covers many cases too — see `CSS_GUIDE.md`.)

```tsx
import { AspectRatio } from "@/components/ui/aspect-ratio"
<AspectRatio ratio={16 / 9}>
  <img src="/cover.jpg" alt="" className="size-full object-cover" />
</AspectRatio>
```

---

## 12. Feedback & Status Reference

### Sonner (Toast) **[B]**
Notification toasts via the **Sonner** library. ⚡ **Version note:** Sonner **replaces** the old deprecated `Toast`/`useToast` component — new installs use Sonner exclusively. **Use for:** transient feedback ("Saved!", "Something went wrong", async results, undo prompts). **Setup:** render `<Toaster />` once near the app root, then call `toast()` from anywhere — no context wiring needed.

```tsx
// app/layout.tsx — render once.
import { Toaster } from "@/components/ui/sonner"
// …inside <body>: <Toaster richColors position="top-right" />

// Anywhere (client code) — import the function from the sonner package.
import { toast } from "sonner"
toast.success("Profile saved")
toast.error("Something went wrong")
toast("Event created", {
  description: "Friday at 7:00 PM",
  action: { label: "Undo", onClick: () => undo() },
})
// For async flows, toast.promise shows loading → success/error automatically:
toast.promise(saveUser(), { loading: "Saving…", success: "Saved!", error: "Failed" })
```

### Alert **[B]**
An inline static callout box (not a toast, not a modal — it sits in the page). **Use for:** persistent page-level notices, warnings, info banners. **Key props:** `variant` (`default` | `destructive`). **Family:** `Alert` > `AlertTitle` + `AlertDescription`; an icon as the first child is styled automatically.

```tsx
import { Alert, AlertTitle, AlertDescription } from "@/components/ui/alert"
import { AlertCircle } from "lucide-react"
<Alert variant="destructive">
  <AlertCircle className="size-4" />
  <AlertTitle>Session expired</AlertTitle>
  <AlertDescription>Please sign in again to continue.</AlertDescription>
</Alert>
```

### Progress **[B]**
A horizontal progress bar (Radix progress). **Use for:** uploads, loading percentages, multi-step progress. **Key props:** `value` (0–100). For indeterminate progress, animate it yourself or use a Skeleton/spinner instead.

```tsx
import { Progress } from "@/components/ui/progress"
<Progress value={66} />
```

### Skeleton **[B]**
A pulsing gray placeholder shown while content loads. **Use for:** loading states that mirror the eventual layout (avatar circle, text lines, cards) to reduce layout shift. **Usage:** it is just a styled box — size and shape it with Tailwind classes.

```tsx
import { Skeleton } from "@/components/ui/skeleton"
<div className="flex items-center gap-4">
  <Skeleton className="size-12 rounded-full" />        {/* avatar */}
  <div className="space-y-2">
    <Skeleton className="h-4 w-[200px]" />              {/* line */}
    <Skeleton className="h-4 w-[160px]" />
  </div>
</div>
```

---

## 13. Layout & Utility Reference

### Separator **[B]**
A thin divider line (Radix separator). **Use for:** separating sections, menu groups, list items. **Key props:** `orientation` (`"horizontal"` | `"vertical"`), `decorative` (true = purely visual, hidden from screen readers). A vertical separator needs a height from its flex parent.

```tsx
import { Separator } from "@/components/ui/separator"
<Separator className="my-4" />
<Separator orientation="vertical" className="h-6" />
```

### Scroll Area **[I]**
Custom, cross-browser-consistent scrollbars (Radix scroll-area). **Use for:** scrollable panels, long lists, chat windows, command menus — anywhere you want a styled scrollbar that matches the design in every browser. **Family:** `ScrollArea` (+ `ScrollBar` with `orientation="horizontal"` for horizontal scroll). Give it a fixed height/width so there is something to scroll.

```tsx
import { ScrollArea } from "@/components/ui/scroll-area"
<ScrollArea className="h-72 w-48 rounded-md border p-4">
  {/* long list */}
</ScrollArea>
```

### Resizable **[A]**
Draggable split panes (built on `react-resizable-panels`). **Use for:** code editors, side-by-side diff views, dashboards with adjustable regions. **Family:** `ResizablePanelGroup` (`direction="horizontal" | "vertical"`) > `ResizablePanel` (with `defaultSize`/`minSize`) + `ResizableHandle` (add `withHandle` for a visible grip).

```tsx
import { ResizablePanelGroup, ResizablePanel, ResizableHandle } from "@/components/ui/resizable"
<ResizablePanelGroup direction="horizontal">
  <ResizablePanel defaultSize={30}>Sidebar</ResizablePanel>
  <ResizableHandle withHandle />
  <ResizablePanel defaultSize={70}>Main</ResizablePanel>
</ResizablePanelGroup>
```

---

## 14. Composing & Customizing Components

### 14.1 The three levels of customization **[A]**

Because you own the code, customization has a ladder. Climb only as high as you need:

1. **`className` override (lightest).** Pass Tailwind classes; `cn()` + tailwind-merge let them win for conflicting properties. Use for one-off tweaks (`<Button className="w-full">`). No file editing.
2. **Edit the component file (medium).** Open `components/ui/button.tsx` and change base classes, add a default, or tweak the markup. This affects *every* use across the app — your design system, your call.
3. **Wrap or extend (heaviest).** Build a higher-level component on top of the primitive when you want new behavior, not just styling.

### 14.2 Wrapping a component to add behavior **[A]**

A classic example: a Button that shows a spinner and disables itself while loading. Rather than re-implementing Button, wrap it:

```tsx
// components/loading-button.tsx
import * as React from "react"
import { Loader2 } from "lucide-react"
import { Button, type ButtonProps } from "@/components/ui/button"

interface LoadingButtonProps extends ButtonProps {
  loading?: boolean
}

export function LoadingButton({ loading, disabled, children, ...props }: LoadingButtonProps) {
  return (
    // Disable while loading; render a spinner before the label.
    <Button disabled={loading || disabled} {...props}>
      {loading && <Loader2 className="size-4 animate-spin" />}
      {children}
    </Button>
  )
}
```

This composes cleanly: it inherits every Button variant/size/prop via `ButtonProps`, adds one new concern, and keeps the underlying `button.tsx` untouched (so re-`add`ing it later is painless).

### 14.3 Extending the primitive's parts **[A]**

For family components, you can pre-compose a opinionated version. Example: a `ConfirmDialog` that wraps `AlertDialog` so call sites are one line:

```tsx
"use client"
import { AlertDialog, AlertDialogTrigger, AlertDialogContent, AlertDialogHeader, AlertDialogTitle, AlertDialogDescription, AlertDialogFooter, AlertDialogAction, AlertDialogCancel } from "@/components/ui/alert-dialog"

export function ConfirmDialog({
  trigger, title, description, onConfirm, confirmLabel = "Confirm",
}: {
  trigger: React.ReactNode; title: string; description: string
  onConfirm: () => void; confirmLabel?: string
}) {
  return (
    <AlertDialog>
      <AlertDialogTrigger asChild>{trigger}</AlertDialogTrigger>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>{title}</AlertDialogTitle>
          <AlertDialogDescription>{description}</AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel>Cancel</AlertDialogCancel>
          <AlertDialogAction onClick={onConfirm}>{confirmLabel}</AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  )
}
// Usage: <ConfirmDialog trigger={<Button variant="destructive">Delete</Button>} title="Sure?" description="Permanent." onConfirm={del} />
```

### 14.4 Customization best practices **[A]**
- **Prefer `className` over editing files** for visual tweaks — it keeps your copy close to upstream so future re-`add`s merge cleanly.
- **Edit the file** when a change is truly system-wide (your brand radius, your default size).
- **Wrap, don't fork**, when adding behavior — new file, compose the primitive.
- **Keep edits committed in git** so a re-`add --overwrite` diff is reviewable and recoverable.
- **Style sub-parts via `data-slot`** (newer components) when you need to reach inside from a parent (§16).

---

## 15. Building Your Own Variants with CVA

### 15.1 Why CVA, and when to add a variant **[A]**

CVA (class-variance-authority) turns a styling decision tree into a typed function. Instead of sprinkling conditional class strings, you declare the axes of variation (`variant`, `size`, `tone`, …) and the classes for each value once. The payoff: a clean prop API, autocompletion, and a single place to change styling. Add a new variant when you find yourself repeatedly passing the same cluster of override classes — promote that pattern into a named variant.

### 15.2 Adding a variant to an existing component **[A]**

Say you want a `success` button. Open `button.tsx` and add an entry to the `variant` map — that's the whole change; TypeScript picks it up automatically:

```tsx
const buttonVariants = cva("…base…", {
  variants: {
    variant: {
      default: "bg-primary text-primary-foreground hover:bg-primary/90",
      destructive: "bg-destructive text-destructive-foreground hover:bg-destructive/90",
      // ── NEW: add a success variant. (Define --success tokens in globals.css.)
      success: "bg-success text-success-foreground hover:bg-success/90",
      outline: "border border-input bg-background hover:bg-accent",
      // …existing…
    },
    size: { /* …unchanged… */ },
  },
  defaultVariants: { variant: "default", size: "default" },
})
// Now <Button variant="success"> is valid AND type-checked. No other edits needed.
```

### 15.3 Building a component from scratch with CVA **[A]**

Here is a full custom `<Callout>` showing every CVA feature — base classes, multiple variant axes, `defaultVariants`, and `compoundVariants` (classes that apply only when *several* variant values combine):

```tsx
// components/ui/callout.tsx
import * as React from "react"
import { cva, type VariantProps } from "class-variance-authority"
import { cn } from "@/lib/utils"

const calloutVariants = cva(
  // BASE: applied to every callout regardless of variants.
  "rounded-lg border p-4 text-sm",
  {
    variants: {
      // Axis 1: semantic tone.
      tone: {
        info: "border-blue-200 bg-blue-50 text-blue-900",
        warning: "border-amber-200 bg-amber-50 text-amber-900",
        danger: "border-red-200 bg-red-50 text-red-900",
      },
      // Axis 2: emphasis.
      emphasis: {
        subtle: "",
        strong: "font-medium shadow-sm",
      },
    },
    // compoundVariants: extra classes ONLY when BOTH conditions match.
    compoundVariants: [
      { tone: "danger", emphasis: "strong", class: "ring-2 ring-red-400" },
    ],
    // Defaults when the caller omits a prop.
    defaultVariants: { tone: "info", emphasis: "subtle" },
  }
)

// VariantProps<typeof calloutVariants> = { tone?, emphasis? } — typed for free.
export interface CalloutProps
  extends React.HTMLAttributes<HTMLDivElement>,
    VariantProps<typeof calloutVariants> {}

export function Callout({ className, tone, emphasis, ...props }: CalloutProps) {
  return (
    // Merge variant classes with the caller's className (caller wins on conflicts).
    <div className={cn(calloutVariants({ tone, emphasis, className }))} {...props} />
  )
}
// <Callout tone="danger" emphasis="strong">Critical!</Callout>
```

### 15.4 CVA best practices **[A]**
- **Put truly-shared classes in the base string**, variant-specific ones in `variants`.
- **Always set `defaultVariants`** so omitting a prop is well-defined.
- **Use `compoundVariants`** for "only when X *and* Y" styling instead of awkward conditionals.
- **Export the `cva` function** (like `buttonVariants`) so the classes can be reused on other elements (§3.3).
- **Keep `className` last in `cn()`** so consumer overrides win.

---

## 16. Accessibility from Radix

### 16.1 What you get for free **[I]**

The biggest hidden value of shadcn is that the *hard* accessibility work is done by Radix primitives. Without writing a line, every relevant component ships with:

- **Correct ARIA roles & relationships** — `role="dialog"`, `aria-expanded`, `aria-controls`, `aria-selected`, `aria-describedby`, etc., wired between triggers and content.
- **Keyboard interaction** — `Tab`/`Shift+Tab` order, `Arrow` keys to move within menus/tabs/radio groups (roving tabindex), `Enter`/`Space` to activate, `Esc` to close overlays, `Home`/`End` where appropriate.
- **Focus management** — modals **trap focus** inside while open and **restore focus** to the trigger on close; menus move focus to the first item.
- **Scroll locking & dismissal** — overlays lock body scroll and close on outside-click/`Esc` (except Alert Dialog, which intentionally requires a choice).
- **Portals** — floating content renders at the body so it isn't clipped, while ARIA keeps the logical relationship intact.

### 16.2 What you still must do **[I/A]**

Radix gives you the machinery; you must still feed it correct content:

- **Label icon-only controls.** Add `aria-label` to icon buttons, toggles, and triggers that have no visible text.
- **Provide required titles.** Dialog/Sheet need a `DialogTitle`/`SheetTitle` (Radix warns otherwise). If your design hides it, wrap it in an `sr-only` (screen-reader-only) utility, e.g. `<DialogTitle className="sr-only">Settings</DialogTitle>`.
- **Use the matching foreground tokens** so text meets contrast (§5).
- **Don't put essential or interactive content in Tooltips/HoverCards** — they're hover/focus-only and unreliable on touch.
- **Respect reduced motion** — Tailwind's `motion-reduce:` variants and `tw-animate-css` honor `prefers-reduced-motion`; keep custom animations behind it too.

### 16.3 Styling sub-parts with `data-slot` and `data-state` **[A]**

⚡ **Version note:** newer generated components tag their internal parts with `data-slot="..."` and expose Radix `data-state="open|closed|checked|active"` attributes. This lets you style internals from a parent or theme without editing the component:

```tsx
// Target a sub-part from the outside via its data-slot.
<Card className="[&_[data-slot=card-title]]:text-primary">…</Card>

// Style based on Radix state (e.g. an accordion trigger when open).
// In the component's classes: data-[state=open]:rotate-180  (rotate the chevron)
```

`data-state` is also how shadcn drives open/close animations (the overlay's enter/exit classes key off `data-[state=open]`/`data-[state=closed]`).

---

## 17. Next.js Integration — Server vs Client Components

### 17.1 The server/client split **[I]**

Next.js App Router renders components on the **server by default** (React Server Components). Server Components cannot use state, effects, refs, or browser-only APIs — they run once on the server and send HTML. **Client Components** (marked with the `"use client"` directive at the top of the file) hydrate in the browser and *can* use `useState`, `useEffect`, event handlers, and context. (Full treatment in `NEXTJS_16_GUIDE.md` and `REACT_19_GUIDE.md`.)

This matters for shadcn because **any interactive component needs the client**:

| Needs `"use client"` (interactive/stateful) | Fine in a Server Component (presentational) |
|---|---|
| Dialog, Sheet, Drawer, Popover, Dropdown/Context Menu, Tooltip, Tabs, Accordion, Select, Combobox, Command | Card, Badge, Separator, Skeleton, Alert, Avatar, AspectRatio |
| Form (RHF uses hooks), Checkbox/Switch/Slider when controlled | Button *as plain markup* (no `onClick`) and `buttonVariants` on a `<Link>` |
| Carousel, Resizable, Sidebar (provider uses state), Sonner `<Toaster />` | Table (static), Breadcrumb |

### 17.2 Where the directive goes **[I]**

You don't have to mark whole pages as client. The pattern is to keep pages/layouts as Server Components and push `"use client"` down into the small interactive leaf components. The shadcn component files that need it already include the directive at the top; your *own* wrapper that uses hooks/handlers needs its own `"use client"`.

```tsx
// app/dashboard/page.tsx — a SERVER component (no directive). Can fetch data directly.
import { Card, CardHeader, CardTitle, CardContent } from "@/components/ui/card"
import { StatsChart } from "./stats-chart" // a "use client" leaf

export default async function Page() {
  const data = await getStats() // server-side data fetch, no useEffect needed
  return (
    <Card> {/* Card is presentational → fine on the server */}
      <CardHeader><CardTitle>Revenue</CardTitle></CardHeader>
      <CardContent>
        <StatsChart data={data} /> {/* interactive chart → client leaf */}
      </CardContent>
    </Card>
  )
}
```

```tsx
// app/dashboard/stats-chart.tsx — the interactive leaf.
"use client" // required: charts/tooltips use client-side state
import { ChartContainer /* … */ } from "@/components/ui/chart"
export function StatsChart({ data }: { data: Stat[] }) { /* … */ }
```

### 17.3 Provider placement & common pitfalls **[I/A]**

- **Render `<Toaster />` once** in `app/layout.tsx` (it's a client component) so `toast()` works app-wide.
- **Wrap the tree in `ThemeProvider`** (next-themes) in the root layout; add `suppressHydrationWarning` to `<html>` (§5.3).
- **`<TooltipProvider>`** near the root (if your version still requires it).
- **Passing data server→client is fine; passing functions is not** — you can't hand an `onClick` from a Server Component to a Client one; lift interactivity into the client leaf.
- **`asChild` + Next `<Link>`** is the canonical way to make Buttons navigate while staying server-rendered anchors: `<Button asChild><Link href="/x">Go</Link></Button>`.

---

## 18. Common Gotchas

| Issue | Cause & fix |
|---|---|
| "useState/hooks only work in Client Components" error | An interactive component (Dialog, Tabs, Form…) used without `"use client"`. Add the directive to the file using it (§17). |
| Tooltip never appears | Not wrapped in `<TooltipProvider>` (older versions). Add one near the app root. |
| Toast never appears | `<Toaster />` not rendered, or `toast` imported from the wrong place. Render `<Toaster />` once at root; import `toast` from `"sonner"`. |
| Radix warns "Dialog requires a Title" | Missing `DialogTitle`. Add one; hide it with `className="sr-only"` if your design has no visible title (§16). |
| `className` override doesn't win | tailwind-merge didn't recognize the conflict (arbitrary value/plugin class), or the class wasn't passed *last* into `cn()`. Check the produced class string in DevTools (§4.2). |
| Button-as-link renders nested `<button><a>` | Use `asChild`: `<Button asChild><Link/></Button>` (§4.4). |
| Select/Dropdown content clipped inside `overflow-hidden` | They portal to `<body>` by default — usually fine; if hidden, it's a `z-index`/stacking-context issue, not clipping. |
| Re-running `add` wiped my edits | `add --overwrite` overwrites your file. Keep edits in git; review the diff and re-apply, or prefer `className`/wrappers over editing the file (§14). |
| Spreading `{...field}` breaks Select/Checkbox in a Form | They use `onValueChange`/`onCheckedChange`, not `onChange`. Map `field.value`/`field.onChange` manually (§7.5). |
| Dark mode flashes wrong theme on load | Missing `suppressHydrationWarning` on `<html>` and/or `disableTransitionOnChange`; ensure next-themes is set up per §5.3. |
| Old `Toast`/`useToast` examples don't exist | They're deprecated/removed — use Sonner (§12). |
| Colors don't change in dark mode | You hardcoded a palette class (`bg-zinc-900`) instead of a token (`bg-card`). Use semantic tokens (§5). |
| Tailwind classes from shadcn not applying | Tailwind v4 isn't scanning your files, or `globals.css` lacks `@import "tailwindcss"`. Verify the v4 setup (§5.2). |

---

## 19. Study Path & Build-to-Learn Projects

### 19.1 Suggested order

1. **[B]** Read §1 (philosophy) until "you own the code" is obvious, then run `init` + `add button card input` in a scratch Next.js app (§2).
2. **[I]** Dissect `button.tsx` (§3) and `cn()`/`asChild` (§4) — open the real files in your repo and read them.
3. **[I]** Wire one real validated form with react-hook-form + zod (§7); make `FormMessage` show an error.
4. **[I]** Set up theming + dark mode with next-themes (§5); re-brand by changing `--primary`.
5. **[I]** Build with overlays and navigation (§9–§10): a Dialog form, a Dropdown Menu, Tabs.
6. **[A]** Compose recipes: a Combobox, a Date Picker, and a TanStack Data Table (§11, §14).
7. **[A]** Add your own CVA variant and build a from-scratch component (§15).
8. **[A]** Internalize the server/client split (§17) and accessibility duties (§16).

### 19.2 Build-to-learn projects

- **Auth flow (covers: Form, Input, Input OTP, Button, Alert, Sonner).** Sign-up + login + 2FA screens with zod validation, server-side error mapping, and toast feedback.
- **Settings page (covers: Tabs, Switch, Select, Form, Separator, Card).** Multiple sections, instant-apply switches, a validated profile form, and a save toast.
- **Admin dashboard (covers: Sidebar, Data Table, Chart, Card, Dialog, Dropdown Menu, Skeleton).** A real grid with sorting/filtering/pagination (TanStack Table + Query), charts wired to tokens, row-action menus, and a create/edit Dialog form.
- **Marketing site header (covers: Navigation Menu, Sheet, Button asChild, Badge, Theme toggle).** Desktop mega-menu, mobile Sheet menu, and a dark-mode toggle.
- **Command palette (covers: Command, CommandDialog, Dialog, keyboard).** A global ⌘K launcher that navigates and runs actions.
- **Your own design system (covers: CVA, theming, registry).** Fork three components, add brand variants and tokens, and (stretch) publish them to a private registry your other apps can `add` by URL.

> When you have a connection, every component has a live, copy-pasteable demo at **ui.shadcn.com/docs/components** — but the goal of this guide is that you can build the whole thing offline, from the principles up, owning every line.
