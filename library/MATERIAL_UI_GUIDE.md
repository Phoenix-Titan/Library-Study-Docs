# Material UI (MUI) — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I've added a `<Button>` once" to "I architect a themed, accessible, performant MUI app in React 19 or Next.js" — with no internet connection. Every concept is explained in *prose first*: **what it is**, **the logic/why it works this way**, **what it's for and when to reach for it**, **how to use it**, its **key props**, **best practices**, and the **gotchas** that bite people — and *then* a heavily-commented, runnable code block. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **MUI v6 and v7** (`@mui/material@6` / `@mui/material@7`), **React 19**, and the ecosystem as it stands in **2026**. Key facts worth knowing up front:
> - **MUI v7** (released 2025) is a mostly-incremental upgrade over v6: ESM-first package layout, the **Grid** rewrite is now the default `Grid` (the old `Grid2` name is gone), and improved tree-shaking. The component API you learn here is identical across v6/v7 except where flagged.
> - **CSS variables / `colorSchemes`** is the modern theming + dark-mode mechanism (stable since v6). It eliminates the SSR dark-mode flash.
> - **Pigment CSS** — a zero-runtime CSS-in-JS compiler — is the long-term direction to replace Emotion's runtime cost, but as of 2026 it is still opt-in/experimental. The default engine is **Emotion**, and that is what this guide assumes.
> - **React 19** is fully supported. The new `ref`-as-a-prop and Actions/`useActionState` features interoperate cleanly with MUI inputs (see §11).
>
> Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**; shell commands are shown for npm. Confirm exact prop names at `mui.com` when you next have internet — MUI's docs have superb live prop tables.
>
> **Cross-references:** This guide assumes React fundamentals from `REACT_19_GUIDE.md`. The Next.js integration in §3 pairs with `NEXTJS_16_GUIDE.md`. Form handling in §11 builds on `REACT_HOOK_FORM_GUIDE.md`. For the "why pick MUI vs the alternatives" discussion see `SHADCN_UI_CHEATSHEET.md` and `TAILWIND_CHEATSHEET.md`.

---

## Table of Contents

1. [What MUI Is & When to Choose It](#1-what-mui-is--when-to-choose-it) **[B]**
2. [Setup & Install](#2-setup--install) **[B]**
3. [Next.js App Router Setup](#3-nextjs-app-router-setup) **[I]**
4. [Theming In Depth](#4-theming-in-depth) **[I/A]**
5. [The `sx` Prop](#5-the-sx-prop) **[B/I]**
6. [Styling Approaches: sx vs styled vs Overrides](#6-styling-approaches-sx-vs-styled-vs-overrides) **[I]**
7. [The Layout System: Box, Container, Stack, Grid](#7-the-layout-system-box-container-stack-grid) **[B/I]**
8. [Core Component Reference](#8-core-component-reference) **[B/I]**
9. [Icons — @mui/icons-material](#9-icons--muiicons-material) **[B]**
10. [DataGrid — @mui/x-data-grid](#10-datagrid--muix-data-grid) **[A]**
11. [Forms with MUI + React Hook Form](#11-forms-with-mui--react-hook-form) **[I/A]**
12. [Dark Mode & Theme Toggling](#12-dark-mode--theme-toggling) **[I]**
13. [Responsive Design](#13-responsive-design) **[I]**
14. [Performance, Tips & Gotchas](#14-performance-tips--gotchas) **[A]**
15. [Study Path & Build-to-Learn Projects](#15-study-path--build-to-learn-projects)

---

## 1. What MUI Is & When to Choose It

### What it is

**Material UI (MUI)** is a React component library: a versioned set of pre-built, accessible, styleable components (`Button`, `TextField`, `Dialog`, `DataGrid`, and ~70 more) that you install from npm and import into your app. It implements Google's **Material Design** language by default — the elevation shadows, the ripple effect on click, the floating labels on text fields — but every visual token is overridable through a central **theme**, so you are not locked into looking "Google-y."

The mental model that matters most: **you do not own the component source.** When you `npm install @mui/material`, you get compiled, versioned components. You customize them from the *outside* — via props, the `sx` prop, `styled()`, and theme overrides — not by editing their code. This is the fundamental philosophical split from `shadcn/ui`, where the CLI copies component source *into your repo* and you own and edit it directly.

### The logic / why it works this way

MUI's architecture rests on three pillars:

1. **A central theme as the single source of truth.** Colors, fonts, spacing, breakpoints, and per-component defaults all live in one `theme` object you pass to a `ThemeProvider`. Change `primary.main` once and every button, link, and focused input updates. This is the opposite of scattering hardcoded hex values across hundreds of files.
2. **CSS-in-JS via Emotion.** Every component's styles are generated as scoped CSS classes at runtime by the Emotion engine. This gives you dynamic, theme-aware, prop-driven styling with no global CSS collisions — at the cost of a runtime that must run in the browser (which is why SSR needs special handling, §3).
3. **Accessibility built in.** Focus management, ARIA roles, keyboard navigation, and screen-reader labels are handled inside the components. A `<Dialog>` traps focus and restores it on close; a `<Select>` is keyboard-navigable. You get this for free, which is a large part of MUI's value.

### When to choose MUI — and when not to

This is the decision every team faces. Here is the honest trade-off table:

| Library | Model | Best when | Trade-offs |
|---|---|---|---|
| **MUI** | Installed npm package, themed centrally | You want a *complete* component set fast (data grids, date pickers, autocomplete), a centralized theme, and built-in a11y. Internal tools, dashboards, admin panels, B2B SaaS. | Larger bundle; Emotion runtime cost; default look is recognizably "Material"; deep customization fights the defaults. |
| **shadcn/ui** | Source copied into your repo (you own it) | You want full control over markup, a Tailwind-native workflow, and a modern minimal aesthetic. Marketing sites, design-forward products. | You maintain the code; no premium data grid; more assembly required. See `SHADCN_UI_CHEATSHEET.md`. |
| **Tailwind (alone)** | Utility CSS, no components | Maximum design freedom, smallest runtime, you build components yourself. | You build *everything* (modals, comboboxes, a11y) from scratch. See `TAILWIND_CHEATSHEET.md`. |
| **Chakra UI** | Installed npm package, themed | Similar to MUI but lighter, less opinionated visually, great DX. | Smaller component catalog than MUI (no first-party data grid of MUI's depth). |

**Choose MUI when:** you need breadth and speed, especially data-heavy UIs. The `DataGrid` (§10) and date pickers alone justify it for many dashboards. The theme system scales beautifully across large teams because design decisions are centralized.

**Avoid MUI when:** bundle size is critical (a content site that must hit a Lighthouse 100), you want a non-Material aesthetic with minimal fighting, or your team is already deep in Tailwind and wants to own their markup.

> **Gotcha — "MUI looks like Google."** New users complain everything looks like a Google product. That is the *default* theme, not a limitation. With 30 minutes of theming (§4) — your brand palette, your border radius, `textTransform: 'none'` on buttons, a custom font — it stops looking like Material and starts looking like *yours*.

### The package ecosystem

You rarely install just one package. Here is the map:

| Package | Purpose | Required? |
|---|---|---|
| `@mui/material` | Core components (Button, TextField, Dialog, …) | **Yes** |
| `@emotion/react`, `@emotion/styled` | The CSS-in-JS engine (peer deps) | **Yes** (with default engine) |
| `@mui/icons-material` | 2,100+ SVG icons as React components | Very common |
| `@mui/x-data-grid` | Feature-rich data table (community = free; Pro/Premium = paid) | Optional |
| `@mui/x-date-pickers` | Date/time pickers | Optional |
| `@mui/lab` | Incubator for not-yet-stable components | Optional |
| `@mui/system` | Low-level `Box`/`sx`/`styled` utilities (bundled into material) | Included |
| `@mui/material-nextjs` | Next.js SSR style-extraction helpers | Next.js only |

> **⚡ Version note (engine):** MUI v7 keeps **Emotion** as the default but the `@mui/material` package is now ESM-first. If you hit `require()` / ESM interop errors after upgrading, it is almost always a bundler/transpiler config issue, not your code.

---

## 2. Setup & Install

### What setup actually does

Three things must be true before a single MUI component renders correctly: (1) the packages are installed, (2) a `ThemeProvider` wraps your app so components can read the theme via context, and (3) `CssBaseline` is mounted so the browser's default styles are normalized and the theme's background/font are applied. Skip any one and you get either crashes, unstyled output, or a white page with mismatched fonts.

### Install (Vite / Create React App / any SPA)

```bash
# Core + the Emotion engine (peer dependencies — MUI will warn if missing)
npm install @mui/material @emotion/react @emotion/styled

# Almost always wanted:
npm install @mui/icons-material

# Optional power-ups:
npm install @mui/x-data-grid          # data table
npm install @mui/x-date-pickers       # date/time pickers
```

### Minimum app wiring

The provider is the heart of it. `ThemeProvider` uses React context to make the theme available to every descendant; that is *why* components deep in the tree can read `primary.main` without prop-drilling.

```tsx
// src/main.tsx  (Vite)
import React from 'react';
import ReactDOM from 'react-dom/client';
import { ThemeProvider, createTheme, CssBaseline } from '@mui/material';
import App from './App';

// createTheme() with no args = MUI defaults: light mode, blue primary, Roboto font.
// Define it at MODULE scope (outside any component) so it is created ONCE, not on
// every render — recreating the theme on each render is a classic performance bug.
const theme = createTheme();

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    {/* ThemeProvider publishes `theme` via context to all descendants */}
    <ThemeProvider theme={theme}>
      {/* CssBaseline: a CSS reset + applies theme background/font to <body>.
          It renders NO markup itself — it just injects global styles. */}
      <CssBaseline />
      <App />
    </ThemeProvider>
  </React.StrictMode>
);
```

### The Roboto font (recommended, and offline-friendly)

Material Design is *designed* for Roboto; the default `theme.typography` assumes it. You can install it as an npm package so it works without internet — the right call for offline study.

```bash
npm install @fontsource/roboto
```

```tsx
// Import the weights MUI uses, at the top of main.tsx. These resolve to local
// CSS+font files in node_modules — no Google Fonts CDN, works fully offline.
import '@fontsource/roboto/300.css';
import '@fontsource/roboto/400.css';
import '@fontsource/roboto/500.css';
import '@fontsource/roboto/700.css';
```

> **Best practice:** If you ship a *custom* font, set `theme.typography.fontFamily` to it (§4) AND import the font files. Setting the theme alone without loading the font gives you a silent fallback to the system font.

> **Gotcha — missing Emotion peer deps.** If you install only `@mui/material` you get cryptic runtime errors. Emotion is a *peer* dependency; install `@emotion/react` and `@emotion/styled` explicitly.

---

## 3. Next.js App Router Setup

### The problem this solves

Next.js App Router renders **React Server Components (RSC)** by default — components that run only on the server and ship zero JS. MUI components, however, rely on the Emotion runtime and (for interactive ones) React hooks and browser APIs. Two issues arise:

1. **Style extraction for SSR.** Emotion generates CSS as it renders. On the server, those generated styles must be *collected* and inlined into the initial HTML; otherwise the first paint is unstyled (a "flash of unstyled content," FOUC) until the client re-runs Emotion. MUI ships `AppRouterCacheProvider` to do this collection automatically for the App Router.
2. **The server/client boundary.** Any MUI component that uses state, effects, or event handlers must live in a file marked `'use client'`. `ThemeProvider` itself uses context (a client feature), so your theme setup is a client component.

### Install

```bash
npm install @mui/material @emotion/react @emotion/styled @mui/material-nextjs
```

### Root layout — the SSR cache provider

```tsx
// app/layout.tsx — a Server Component (no 'use client' here).
import type { Metadata } from 'next';
import { AppRouterCacheProvider } from '@mui/material-nextjs/v15-appRouter';
// ⚡ Version note: pick the import path that matches your Next.js major:
//   '@mui/material-nextjs/v14-appRouter'  for Next 14
//   '@mui/material-nextjs/v15-appRouter'  for Next 15/16 (current in 2026)
import { Providers } from './providers';

export const metadata: Metadata = { title: 'My MUI App' };

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {/* AppRouterCacheProvider must be the OUTERMOST wrapper. It captures
            Emotion's generated styles during the server render and inlines
            them, preventing FOUC. */}
        <AppRouterCacheProvider>
          <Providers>{children}</Providers>
        </AppRouterCacheProvider>
      </body>
    </html>
  );
}
```

### The client provider (theme + baseline)

```tsx
// app/providers.tsx
'use client'; // REQUIRED: ThemeProvider relies on React context, a client feature.

import { ThemeProvider, createTheme, CssBaseline } from '@mui/material';
import type { ReactNode } from 'react';

// Module-scope theme — created once. With CSS variables mode (§12) this also
// gives you a flash-free dark mode under SSR.
const theme = createTheme({
  cssVariables: true, // emit CSS custom properties (recommended for Next.js SSR)
  palette: { primary: { main: '#1976d2' } },
});

export function Providers({ children }: { children: ReactNode }) {
  return (
    <ThemeProvider theme={theme}>
      <CssBaseline />
      {children}
    </ThemeProvider>
  );
}
```

### Using MUI in Server vs Client components

```tsx
// app/page.tsx — a Server Component. Purely presentational MUI components
// (no hooks/handlers inside the boundary you control) render fine in RSC.
import { Container, Typography } from '@mui/material';
import { InteractiveSection } from './InteractiveSection'; // a 'use client' island

export default function HomePage() {
  return (
    <Container maxWidth="md" sx={{ py: 4 }}>
      {/* Static content — server-rendered, no client JS for this part */}
      <Typography variant="h1" gutterBottom>Hello MUI + App Router</Typography>
      {/* Interactivity is isolated in a client island */}
      <InteractiveSection />
    </Container>
  );
}
```

```tsx
// app/InteractiveSection.tsx
'use client'; // needed: this uses useState + an event handler
import { useState } from 'react';
import { Button, Typography } from '@mui/material';

export function InteractiveSection() {
  const [count, setCount] = useState(0);
  return (
    <>
      <Typography>Clicked {count} times</Typography>
      <Button variant="contained" onClick={() => setCount((c) => c + 1)}>
        Click me
      </Button>
    </>
  );
}
```

> **Best practice — the "client island" pattern.** Keep your `'use client'` boundary as *low* in the tree as possible: wrap only the interactive piece, not the whole page. This maximizes how much stays server-rendered (smaller JS bundle, faster first load). See `NEXTJS_16_GUIDE.md` for the broader RSC model.

> **Gotcha — "You're importing a component that needs `useState`…"** This RSC error means an interactive MUI component (Dialog, Drawer, Menu, anything with state/handlers) was imported into a server component. Fix: move that usage into a file with `'use client'` at the top.

> **⚡ Version note (Pages Router):** If you are on the legacy `pages/` router, the setup is different and more manual — you create an Emotion cache and extract styles in a custom `_document.tsx`. Use `@mui/material-nextjs/v15-pagesRouter`. The App Router approach above is strongly preferred for new projects.

---

## 4. Theming In Depth

The theme is the most important concept in MUI. Master it and everything else composes cleanly; ignore it and you will fight the library with one-off `sx` overrides forever.

### What the theme is and why it exists

`createTheme(options)` returns a deeply-nested, fully-resolved configuration object describing **every design token** in your app: the color palette, typography scale, spacing unit, shape (border radius), breakpoints, z-index layers, transitions, and per-component default props and style overrides. `ThemeProvider` publishes it via context; components and your `sx`/`styled` code read from it.

The *logic*: design systems are about consistency, and consistency comes from a single source of truth. Instead of typing `#1976d2` in fifty places, you define `palette.primary.main` once. Instead of guessing `16px` vs `15px` paddings, you use `spacing(2)`. When the brand color changes, you edit one line. This centralization is the whole point.

### `createTheme` — a complete, annotated theme

```tsx
// theme.ts
import { createTheme } from '@mui/material/styles';

const theme = createTheme({
  // Emit CSS custom properties (--mui-palette-primary-main, etc.). Strongly
  // recommended: enables flash-free dark mode and lets you reference tokens in
  // plain CSS. (§12)
  cssVariables: true,

  // ─── PALETTE — the color system ────────────────────────────────────────────
  palette: {
    mode: 'light', // 'light' | 'dark' (or use colorSchemes for both — §12)

    // Each color "intention" has main/light/dark/contrastText. You MUST give
    // `main`; MUI auto-derives light/dark/contrastText if you omit them.
    primary: {
      main: '#1976d2',
      light: '#42a5f5',
      dark: '#1565c0',
      contrastText: '#fff', // text color placed ON the primary background
    },
    secondary: { main: '#9c27b0' },

    // Semantic colors — used by Alert, helper text, status chips, etc.
    error:   { main: '#d32f2f' },
    warning: { main: '#ed6c02' },
    info:    { main: '#0288d1' },
    success: { main: '#2e7d32' },

    // Surfaces and text. CssBaseline applies background.default to <body>.
    background: { default: '#f5f5f5', paper: '#ffffff' },
    text: {
      primary:   'rgba(0,0,0,0.87)',
      secondary: 'rgba(0,0,0,0.60)',
      disabled:  'rgba(0,0,0,0.38)',
    },

    // A custom palette color (requires a TS augmentation — see below).
    // neutral: { main: '#64748B' },
  },

  // ─── TYPOGRAPHY — the type scale ───────────────────────────────────────────
  typography: {
    fontFamily: '"Roboto", "Helvetica", "Arial", sans-serif',
    fontSize: 14,            // base px; rem values are computed relative to this
    fontWeightLight: 300,
    fontWeightRegular: 400,
    fontWeightMedium: 500,
    fontWeightBold: 700,
    h1: { fontSize: '2.5rem', fontWeight: 700, lineHeight: 1.2 },
    h2: { fontSize: '2rem',   fontWeight: 600 },
    body1: { fontSize: '1rem', lineHeight: 1.6 },
    // Disable the Material ALL-CAPS button text app-wide — a very common tweak:
    button: { textTransform: 'none', fontWeight: 600 },
  },

  // ─── SPACING — the 8px grid ────────────────────────────────────────────────
  // theme.spacing(1) = 8px, spacing(2) = 16px. Drives every sx p/m/gap value.
  spacing: 8, // or a function: (factor) => `${0.25 * factor}rem`

  // ─── SHAPE — global corner radius ──────────────────────────────────────────
  shape: { borderRadius: 8 }, // Material default is 4; 8–12 looks more modern.

  // ─── BREAKPOINTS — responsive cutoffs (min-widths) ─────────────────────────
  breakpoints: {
    values: { xs: 0, sm: 600, md: 900, lg: 1200, xl: 1536 },
  },

  // ─── COMPONENT OVERRIDES — per-component defaults & styles ──────────────────
  components: {
    MuiButton: {
      // defaultProps: change the DEFAULT value of any prop, app-wide.
      defaultProps: { disableElevation: true, variant: 'contained' },
      // styleOverrides: inject CSS into a component's named "slots".
      styleOverrides: {
        // `root` is the outermost slot. The callback receives theme + ownerState
        // (the component's current props) so you can vary by variant/color.
        root: ({ ownerState, theme }) => ({
          borderRadius: ownerState.variant === 'outlined' ? 4 : 20,
        }),
      },
    },
    MuiTextField: {
      defaultProps: { variant: 'outlined', size: 'small', fullWidth: true },
    },
    MuiCard: {
      styleOverrides: { root: { boxShadow: '0 2px 8px rgba(0,0,0,0.12)' } },
    },
  },
});

export default theme;
```

### Palette — how MUI resolves colors

When you write `color="primary"` on a `Button`, MUI looks up `theme.palette.primary.main` for the background and `primary.contrastText` for the text. When you write `color="error"` on an `Alert`, it uses `palette.error`. This is why your *whole app* recolors from a single palette edit. The `light`/`dark` shades are used for hover states, borders, and subtle backgrounds — let MUI auto-compute them unless you have brand-exact values.

### Typography — the variant system

`theme.typography` defines named text styles (`h1`–`h6`, `subtitle1/2`, `body1/2`, `caption`, `overline`, `button`). The `<Typography variant="h2">` component pulls the matching style. Defining the scale here means consistent text everywhere; you never hand-set `font-size` on headings.

### Spacing — the multiplier

Every spacing value in `sx` (`p`, `m`, `gap`, etc.) is multiplied by `theme.spacing` (default 8px). So `sx={{ p: 2 }}` = 16px. This enforces a consistent rhythm. To bypass it for a literal value, pass a string: `sx={{ p: '10px' }}`.

### Breakpoints — the responsive contract

The five breakpoints define where layouts change. They power responsive `sx` objects (`{ xs: …, md: … }`), `useMediaQuery`, and `theme.breakpoints.up('md')` in `styled()`. They are **min-width / mobile-first**: `md` means "≥ 900px." (§13 covers responsive design fully.)

### `CssBaseline` — what it actually does

```tsx
// Place INSIDE ThemeProvider so it can read theme tokens.
<ThemeProvider theme={theme}>
  <CssBaseline />
  <App />
</ThemeProvider>
// CssBaseline applies global styles:
//  • box-sizing: border-box on everything (sane sizing)
//  • removes default <body> margin
//  • sets <body> background to theme.palette.background.default
//  • sets the base font-family/size from theme.typography
//  • smooths font rendering
// It renders no DOM of its own — think of it as Normalize.css, theme-aware.
```

### Reading theme tokens in your own code

```tsx
import { useTheme } from '@mui/material/styles';

function Banner() {
  const theme = useTheme(); // grab the resolved theme inside a component
  return (
    <div style={{
      backgroundColor: theme.palette.primary.main,
      padding: theme.spacing(2),               // 16px
      borderRadius: theme.shape.borderRadius,  // 8px
      color: theme.palette.primary.contrastText,
    }}>
      Themed banner
    </div>
  );
}
```

### TypeScript — augmenting the theme for custom tokens

If you add a custom palette color or typography variant, TypeScript doesn't know about it until you *augment* MUI's type declarations. This is module augmentation — you extend MUI's interfaces.

```tsx
// theme.d.ts  (or any .d.ts included in tsconfig)
import '@mui/material/styles';

declare module '@mui/material/styles' {
  // Tell TS that palette has a `neutral` color shaped like `primary`.
  interface Palette { neutral: Palette['primary']; }
  interface PaletteOptions { neutral?: PaletteOptions['primary']; }
}

// Teach the Button's `color` prop to accept "neutral":
declare module '@mui/material/Button' {
  interface ButtonPropsColorOverrides { neutral: true; }
}

// Now this is type-safe:
// const theme = createTheme({ palette: { neutral: { main: '#64748B' } } });
// <Button color="neutral">Cancel</Button>
```

> **Best practice:** Keep `createTheme` at module scope and import it. Never call it inside a render — it is a non-trivial computation and a new object identity each time defeats memoization and forces full re-styles.

> **Gotcha — nested themes.** A `<ThemeProvider>` deeper in the tree *replaces* the theme for its subtree by default, not merges. To extend the outer theme, pass it as the first arg: `createTheme(outerTheme, { … })`.

---

## 5. The `sx` Prop

### What it is and why it exists

The `sx` prop is MUI's signature styling tool — a superset of CSS that you pass as an object to any MUI component (and to `Box`). It looks like inline styles but is far more powerful: it resolves **theme tokens**, supports **responsive breakpoint objects**, accepts **shorthand spacing aliases**, allows **pseudo-classes and nested selectors**, and (unlike `style=`) generates real cached CSS classes via Emotion.

The *logic*: writing one-off styles should be ergonomic and theme-aware without forcing you to create a new `styled()` component or a separate CSS file for every tweak. `sx` is the "I just need to nudge this one element" tool, and it stays connected to your theme so values like `p: 2` and `color: 'primary.main'` mean the same thing everywhere.

### Key capabilities (with the why for each)

```tsx
import { Box } from '@mui/material';

<Box
  sx={{
    // 1) STANDARD CSS — any CSS property, camelCased.
    display: 'flex',
    alignItems: 'center',
    backgroundColor: '#fafafa',

    // 2) SPACING SHORTHANDS — multiplied by theme.spacing (default 8px).
    //    Why: terse, consistent rhythm. p/m + t/b/l/r/x/y axes.
    p: 2,            // padding: 16px
    px: 3,           // padding-left + padding-right: 24px
    py: 1,           // padding-top + padding-bottom: 8px
    mt: 'auto',      // margin-top: auto (string passes through literally)
    gap: 2,          // gap: 16px (for flex/grid)

    // 3) THEME TOKEN STRINGS — dotted paths into the theme, resolved for you.
    color: 'primary.main',          // → theme.palette.primary.main
    bgcolor: 'background.paper',    // → theme.palette.background.paper
    borderColor: 'divider',         // → theme.palette.divider
    boxShadow: 3,                   // → theme.shadows[3]
    borderRadius: 1,                // → theme.shape.borderRadius * 1

    // 4) RESPONSIVE VALUES — object keyed by breakpoint (mobile-first).
    //    xs is the base; each key overrides at that breakpoint and up.
    flexDirection: { xs: 'column', md: 'row' },
    width: { xs: '100%', md: '50%' },
    fontSize: { xs: '0.875rem', sm: '1rem', lg: '1.25rem' },

    // 5) PSEUDO-CLASSES & NESTED SELECTORS — '&' = this element.
    '&:hover': { bgcolor: 'action.hover' },
    '&:focus-visible': { outline: '2px solid', outlineColor: 'primary.main' },
    '& .child': { color: 'text.secondary' }, // style a descendant
  }}
>
  Content
</Box>
```

### Responsive values: object vs array

```tsx
// Object form (clearest): keys are breakpoints, mobile-first.
<Box sx={{ p: { xs: 1, sm: 2, md: 4 } }} />

// Array form: index maps to [xs, sm, md, lg, xl]. Use `null` to skip.
<Box sx={{ p: [1, 2, 4] }} />   // xs=8px, sm=16px, md=32px
```

### The callback form — full theme access

When token strings aren't enough (e.g. you need `theme.breakpoints` or conditional logic on `theme.palette.mode`), pass a function. It receives the resolved theme.

```tsx
<Box
  sx={(theme) => ({
    // Branch on the active color mode:
    backgroundColor: theme.palette.mode === 'dark'
      ? theme.palette.grey[800]
      : theme.palette.grey[100],
    // Use a breakpoint helper for a media query:
    [theme.breakpoints.up('md')]: { padding: theme.spacing(4) },
  })}
>
  Adaptive surface
</Box>
```

### The array form for merging / conditional styles

`sx` accepts an *array*; entries are merged left-to-right and falsy entries are skipped. This is the idiomatic way to apply conditional styles and to let a parent component pass through `sx`.

```tsx
<Box
  sx={[
    { p: 2, borderRadius: 1 },          // base styles
    isActive && { bgcolor: 'primary.light' }, // applied only when active
    isError && { borderColor: 'error.main' },
  ]}
>
  Card
</Box>
```

> **Best practice:** Use `sx` for one-off and dynamic styles. If the same `sx` object appears on many elements, promote it to a `styled()` component (§6) — `styled` resolves its base styles once, while an inline `sx` object is re-created each render (cheap, but not free in long lists).

> **Gotcha — `color` strings vs CSS colors.** `color: 'primary.main'` is a *theme path*. `color: '#1976d2'` is a literal. `color: 'primary'` is **not** valid (there's no `.main`). Likewise `bgcolor` is the shorthand; `backgroundColor` also works but won't accept the `'primary.main'` token shorthand on every property — when in doubt use the documented shorthands (`bgcolor`, `color`, `borderColor`).

---

## 6. Styling Approaches: sx vs styled vs Overrides

MUI gives you four ways to style. They are not competing — each has a niche. Choosing well is what separates clean codebases from `sx`-spaghetti.

### The decision table

| Approach | What it is | Reach for it when | Runtime cost |
|---|---|---|---|
| **`sx` prop** | Inline style object on one element | One-off or dynamic styling of a single instance | Low (Emotion caches classes; object re-created per render) |
| **`styled()`** | A reusable styled React component | The same styled element is used in many places, or has complex variants | Low; base styles resolved once at definition |
| **Theme `styleOverrides`** | Global CSS for a component across the app | You want *every* `Button`/`Card`/etc. to look a certain way | Zero extra (runs at theme creation) |
| **Theme `defaultProps`** | Default values for a component's props | You want every `TextField` to default to `size="small"` etc. | Zero |
| **Plain `style={}`** | Native React inline style | Truly per-frame dynamic values (e.g. a JS-driven transform) | None (no class, no theme) |

### 1) `sx` — one-offs (covered in §5)

```tsx
<Button sx={{ mt: 2, px: 4 }}>Submit</Button>
```

### 2) `styled()` — reusable styled components

`styled()` wraps any MUI component or HTML tag and returns a new component with baked-in styles. The base styles are computed once, making it efficient for components rendered many times (rows, cards, list items). It also has clean access to the theme and to the component's own props.

```tsx
import { styled } from '@mui/material/styles';
import { Button } from '@mui/material';

// Wrap Button — PillButton accepts ALL Button props plus our styling.
const PillButton = styled(Button)(({ theme }) => ({
  borderRadius: 999,
  paddingInline: theme.spacing(3),
  textTransform: 'none',
  fontWeight: 600,
}));

// Usage — fully typed, behaves exactly like Button:
<PillButton variant="contained" startIcon={<span>★</span>}>Save</PillButton>;
```

#### Styling an HTML element with a custom prop

When you put a *custom* prop on a styled HTML element, you must stop it from leaking to the DOM (React warns about unknown DOM attributes). Use `shouldForwardProp`.

```tsx
import { styled } from '@mui/material/styles';

interface HighlightProps { active?: boolean; }

const Highlight = styled('div', {
  // Block `active` from reaching the real <div> — it's a styling prop, not HTML.
  shouldForwardProp: (prop) => prop !== 'active',
})<HighlightProps>(({ theme, active }) => ({
  transition: 'background-color .2s',
  backgroundColor: active ? theme.palette.primary.light : 'transparent',
  padding: theme.spacing(1.5),
  borderRadius: theme.shape.borderRadius,
}));

<Highlight active={isSelected}>Row content</Highlight>;
```

### 3) Theme `styleOverrides` — global component styling

When you want a rule to apply to *every* instance of a component, put it in the theme. This is how you brand the whole app at once — no per-instance `sx`.

```tsx
const theme = createTheme({
  components: {
    MuiButton: {
      styleOverrides: {
        // `root` slot, with ownerState so you can branch by props.
        root: ({ ownerState }) => ({
          borderRadius: 12,
          ...(ownerState.size === 'large' && { paddingInline: 32 }),
        }),
        // Slot-specific overrides — every component documents its slots.
        startIcon: { marginRight: 6 },
      },
    },
  },
});
```

### 4) Theme `defaultProps` — global default props

Change the *default* value of props app-wide. The cleanest way to, say, make all text fields small and outlined without repeating yourself.

```tsx
const theme = createTheme({
  components: {
    MuiTextField: { defaultProps: { size: 'small', variant: 'outlined', fullWidth: true } },
    MuiButton:    { defaultProps: { disableElevation: true } },
  },
});
// Now <TextField label="X" /> is already small/outlined/fullWidth.
```

> **Best practice — the layering strategy.** Set the *baseline* look in the theme (`defaultProps` + `styleOverrides`), use `styled()` for reusable bespoke components, and reserve `sx` for genuine one-offs. If you find yourself copy-pasting the same `sx` block, that's a signal to move it up a layer.

> **Gotcha — specificity wars.** Theme `styleOverrides` and `sx` both generate Emotion classes; `sx` generally wins (it's applied last) for the same property. If an override "won't stick," check whether something more specific (a slot override, a `&.Mui-*` state class, or an inline `style`) is overriding it.

---

## 7. The Layout System: Box, Container, Stack, Grid

MUI's layout primitives let you build responsive page structure without writing raw flexbox/grid CSS by hand — though it's all flexbox/grid underneath. Learn these four and you can lay out almost anything.

### Box — the universal building block

`Box` is a `<div>` (by default) supercharged with the `sx` prop. It is the single most-used component in any MUI app: use it anywhere you'd reach for a styled `<div>`. The `component` prop lets it render any element/component while keeping `sx`.

```tsx
import { Box, Avatar, Typography } from '@mui/material';

// A flex row — the everyday use.
<Box sx={{ display: 'flex', alignItems: 'center', gap: 2, p: 2,
           bgcolor: 'background.paper', borderRadius: 1, boxShadow: 1 }}>
  <Avatar src="/avatar.jpg" />
  <Typography>John Doe</Typography>
</Box>

// `component` changes the rendered tag (semantic HTML matters for a11y/SEO).
<Box component="section" sx={{ py: 6 }}>…</Box>

// A full-height app shell using flex column.
<Box sx={{ display: 'flex', flexDirection: 'column', minHeight: '100vh' }}>
  <Header />
  <Box component="main" sx={{ flex: 1 }}>Content grows to fill</Box>
  <Footer />
</Box>
```

### Container — centered, width-constrained content

`Container` centers your content horizontally and caps its width at a breakpoint, with responsive horizontal padding ("gutters"). It is the standard outer wrapper for page sections so text doesn't stretch uncomfortably wide on big monitors.

```tsx
import { Container } from '@mui/material';

// maxWidth: 'xs' | 'sm' | 'md' | 'lg' | 'xl' | false (no cap)
<Container maxWidth="lg" sx={{ py: 4 }}>
  {/* content is centered and capped at ~1200px */}
</Container>

// Edge-to-edge hero: remove the side gutters.
<Container maxWidth={false} disableGutters>
  <FullBleedHero />
</Container>
```

### Stack — consistent spacing between children

`Stack` is a flex container whose job is to space its children evenly — replacing the old "margin on every child" hack. Set `direction` and `spacing`; both can be responsive.

```tsx
import { Stack, Divider, TextField, Button } from '@mui/material';

// Vertical (default direction='column'), 16px gaps.
<Stack spacing={2}>
  <TextField label="Email" />
  <TextField label="Password" type="password" />
  <Button type="submit" variant="contained">Log in</Button>
</Stack>

// Horizontal with dividers between items.
<Stack direction="row" spacing={2}
       divider={<Divider orientation="vertical" flexItem />}
       alignItems="center">
  <span>Profile</span><span>Settings</span><span>Logout</span>
</Stack>

// Responsive: stack on mobile, row on tablet+.
<Stack direction={{ xs: 'column', sm: 'row' }} spacing={{ xs: 1, sm: 3 }}>
  <Card>…</Card><Card>…</Card>
</Stack>
```

> **`Stack` vs `Box`:** Use `Stack` when the *only* thing you need is even spacing in one direction. Use `Box` when you need finer flex control (justify, wrap, custom alignment per child). They overlap; `Stack` is just the ergonomic shortcut.

### Grid — the 12-column responsive grid (the v6/v7 API)

⚡ **Version note — the big one.** MUI rewrote `Grid`. The **old** API (`<Grid item xs={12} md={6}>`) is gone. In **v6** the new grid was imported as `Grid2`/`Unstable_Grid2`; in **v7** it is simply the default `Grid` and the `Grid2` name was removed. The new API:
- There is **no `item` prop** anymore — every `Grid` inside a `container` is an item.
- Column spans are set with a single **`size`** prop: `size={{ xs: 12, md: 6 }}` (or `size={6}`, or `size="grow"`).
- It uses CSS `gap` under the hood (cleaner than the old negative-margin trick).

```tsx
// v7: import the default Grid. (v6: import Grid from '@mui/material/Grid2')
import { Grid } from '@mui/material';

// 12-column responsive layout. spacing is the gap (× theme.spacing).
<Grid container spacing={2}>
  {/* size replaces item + xs/md. Full width row: */}
  <Grid size={12}><Header /></Grid>

  {/* Main + sidebar: 8/4 on desktop, stacked (12/12) on mobile. */}
  <Grid size={{ xs: 12, md: 8 }}><MainContent /></Grid>
  <Grid size={{ xs: 12, md: 4 }}><Sidebar /></Grid>
</Grid>

// 'grow' takes the remaining space (like flex-grow:1):
<Grid container spacing={2}>
  <Grid size={4}>Fixed third</Grid>
  <Grid size="grow">Fills the rest</Grid>
</Grid>

// Responsive card grid — 1/2/3/4 columns as the viewport grows:
<Grid container spacing={3}>
  {items.map((item) => (
    <Grid key={item.id} size={{ xs: 12, sm: 6, md: 4, lg: 3 }}>
      <Card><CardContent><Typography>{item.title}</Typography></CardContent></Card>
    </Grid>
  ))}
</Grid>

// Nested grids: a Grid item can itself be a container.
<Grid container spacing={2}>
  <Grid size={6}>
    <Grid container spacing={1}>
      <Grid size={6}><Box sx={{ bgcolor: 'primary.light', p: 2 }}>A</Box></Grid>
      <Grid size={6}><Box sx={{ bgcolor: 'secondary.light', p: 2 }}>B</Box></Grid>
    </Grid>
  </Grid>
</Grid>
```

> **When to use Grid vs Stack vs CSS grid via `sx`:** Use `Grid` for true 12-column page/card layouts that reflow by breakpoint. Use `Stack` for simple one-directional spacing. For bespoke grids, `<Box sx={{ display: 'grid', gridTemplateColumns: 'repeat(3, 1fr)', gap: 2 }}>` is often simpler than `Grid` and avoids the wrapper overhead.

> **Gotcha — migrating old Grid code.** If you copy a pre-v6 snippet with `<Grid item xs={12} md={6}>`, it will not work. Convert to `<Grid size={{ xs: 12, md: 6 }}>` and remove `item`.

---

## 8. Core Component Reference

This is the working catalog. For each component: a one-line "what it's for," the **key props**, the **gotchas**, then commented code. Components are grouped: Text & Display → Inputs → Data Display → Navigation → Feedback/Overlays.

### 8.1 Text & Display

#### Typography — semantic, theme-styled text

Renders text with a theme-defined style. The crucial idea: **`variant` controls the *look*, `component` controls the *HTML tag*.** That separation lets you have an `<h2>` that looks like `h4`, keeping your heading hierarchy correct for accessibility/SEO while matching the design.

- **Key props:** `variant` (h1–h6, subtitle1/2, body1/2, caption, overline, button), `component`, `color`, `align`, `gutterBottom`, `noWrap`.

```tsx
import { Typography } from '@mui/material';

<Typography variant="h1" component="h1" gutterBottom>Page Title</Typography>
<Typography variant="body1" color="text.secondary">Secondary paragraph.</Typography>

{/* Looks like H4, but is semantically a paragraph: */}
<Typography variant="h4" component="p">Big intro line</Typography>

{/* Truncate long text with an ellipsis: */}
<Typography noWrap sx={{ maxWidth: 200 }}>Very long text that will be cut…</Typography>
```

> **Gotcha:** Nesting block-level `Typography` inside another can produce invalid HTML (`<p>` inside `<p>`). Use `component="span"` for inline text inside a paragraph.

### 8.2 Inputs & Forms

#### Button — the primary action element

- **Key props:** `variant` (`contained` | `outlined` | `text`), `color`, `size`, `startIcon`/`endIcon`, `fullWidth`, `disabled`, `loading` (v6+), `href`, `component`.

```tsx
import { Button } from '@mui/material';
import SendIcon from '@mui/icons-material/Send';
import DeleteIcon from '@mui/icons-material/Delete';

<Button variant="contained" color="primary" onClick={handleClick}>Submit</Button>
<Button variant="outlined" color="secondary" disabled>Disabled</Button>
<Button variant="text" size="small">Cancel</Button>

{/* Icons placed before/after the label: */}
<Button variant="contained" startIcon={<SendIcon />}>Send</Button>
<Button variant="outlined" color="error" endIcon={<DeleteIcon />}>Delete</Button>

<Button fullWidth variant="contained" sx={{ mt: 2 }}>Log in</Button>

{/* Built-in loading state (MUI v6+): shows a spinner, disables clicks. */}
<Button loading={isSaving} loadingPosition="start" startIcon={<SendIcon />}>
  Save
</Button>
```

> **Best practice:** One `contained` (primary) button per view; secondary actions are `outlined` or `text`. This visual hierarchy guides the user to the main action.

#### IconButton — a clickable icon (no label)

For toolbar/compact actions. **Always** give it an accessible name (`aria-label`) since there's no visible text, and wrap it in a `Tooltip` for discoverability.

```tsx
import { IconButton, Tooltip } from '@mui/material';
import FavoriteIcon from '@mui/icons-material/Favorite';

<Tooltip title="Add to favorites">
  <IconButton aria-label="add to favorites" color="error">
    <FavoriteIcon />
  </IconButton>
</Tooltip>
```

#### TextField — the all-in-one text input

`TextField` is a composite: it bundles the `<input>`, the floating `<label>`, helper/error text, and adornments into one component with built-in label animation and validation styling. It is the workhorse of MUI forms.

- **Key props:** `label`, `value`/`onChange` (controlled), `type`, `variant` (`outlined` | `filled` | `standard`), `size`, `error` (bool), `helperText`, `required`, `multiline` + `rows`, `slotProps` (v6+ way to reach inner slots), `select` (turns it into a Select).

```tsx
import { TextField, InputAdornment } from '@mui/material';
import SearchIcon from '@mui/icons-material/Search';
import { useState } from 'react';

const [value, setValue] = useState('');

{/* Controlled input — value comes from state, onChange writes it back. */}
<TextField
  label="Email Address" type="email"
  value={value} onChange={(e) => setValue(e.target.value)}
  variant="outlined" size="small" fullWidth required
  helperText="We'll never share your email."
  placeholder="you@example.com"
/>

{/* Error state: red border + red helper text. */}
<TextField label="Password" type="password"
  error={!!errors.password} helperText={errors.password?.message} />

{/* Multiline = textarea. */}
<TextField label="Description" multiline rows={4} fullWidth />

{/* Adornments via slotProps (v6+). (Old API was InputProps.startAdornment.) */}
<TextField
  label="Search"
  slotProps={{
    input: {
      startAdornment: (
        <InputAdornment position="start"><SearchIcon /></InputAdornment>
      ),
    },
  }}
/>
```

> **⚡ Version note:** In v6+ the preferred way to pass props to inner pieces is **`slotProps`** (e.g. `slotProps={{ input: {...}, htmlInput: {...} }}`). The older `InputProps`/`inputProps` still work but are being phased toward the unified `slotProps` API.

> **Gotcha — controlled vs uncontrolled.** If you pass `value` you MUST pass `onChange`, or the field becomes read-only and React warns. For uncontrolled fields (e.g. with React Hook Form's `register`), use `defaultValue` instead of `value`.

#### Select — a dropdown of options

A `Select` needs a wrapping `FormControl` + `InputLabel` to get the floating label behavior. Pair `labelId` (on Select) with the `InputLabel`'s `id` for accessibility, and repeat the label text in Select's `label` prop so the outline notch sizes correctly.

```tsx
import { FormControl, InputLabel, Select, MenuItem, Checkbox, ListItemText } from '@mui/material';
import { useState } from 'react';

const [age, setAge] = useState('');

<FormControl fullWidth size="small">
  <InputLabel id="age-label">Age</InputLabel>
  <Select labelId="age-label" value={age} label="Age"
          onChange={(e) => setAge(e.target.value)}>
    <MenuItem value=""><em>None</em></MenuItem>
    <MenuItem value={10}>Ten</MenuItem>
    <MenuItem value={20}>Twenty</MenuItem>
  </Select>
</FormControl>

{/* Multi-select with checkboxes and a summary render. */}
<Select multiple value={selected} onChange={handleChange}
        renderValue={(s) => (s as string[]).join(', ')}>
  {options.map((opt) => (
    <MenuItem key={opt} value={opt}>
      <Checkbox checked={selected.includes(opt)} />
      <ListItemText primary={opt} />
    </MenuItem>
  ))}
</Select>
```

#### Autocomplete — searchable combobox

Use over `Select` when the option list is long, needs filtering, supports multiple tags, free typing, or async loading.

- **Key props:** `options`, `value`/`onChange`, `getOptionLabel`, `isOptionEqualToValue`, `multiple`, `freeSolo`, `loading`, `renderInput` (required — render a `TextField`).

```tsx
import { Autocomplete, TextField } from '@mui/material';

const options = ['Apple', 'Banana', 'Cherry'];

{/* Basic single-select. renderInput wires the Autocomplete to a TextField. */}
<Autocomplete options={options}
  onChange={(_e, value) => setFruit(value)}
  renderInput={(params) => <TextField {...params} label="Fruit" />} />

{/* Objects: tell Autocomplete how to label and compare them. */}
<Autocomplete
  options={movies}
  getOptionLabel={(o) => o.title}
  isOptionEqualToValue={(o, v) => o.id === v.id}  // avoids "controlled" warnings
  renderInput={(params) => <TextField {...params} label="Movie" />} />

{/* Multiple → renders deletable chips. */}
<Autocomplete multiple options={options} value={tags}
  onChange={(_e, v) => setTags(v)}
  renderInput={(params) => <TextField {...params} label="Tags" />} />

{/* freeSolo allows arbitrary typed values (e.g. a search box). */}
<Autocomplete freeSolo options={options}
  renderInput={(params) => <TextField {...params} label="Search" />} />
```

> **Gotcha:** With object options you almost always need `isOptionEqualToValue`, otherwise MUI compares by reference and complains the value isn't in the options list.

#### Checkbox, Radio, Switch — boolean & choice inputs

Each is usually wrapped in `FormControlLabel` to attach a clickable label. `Radio`s live inside a `RadioGroup` (which owns the shared value). `Switch` is a styled boolean toggle (use for instant on/off settings, not form submission of yes/no).

```tsx
import { Checkbox, Radio, RadioGroup, Switch,
         FormControlLabel, FormControl, FormLabel } from '@mui/material';

{/* Checkbox — controlled via checked + onChange(e.target.checked). */}
<FormControlLabel
  control={<Checkbox checked={agree} onChange={(e) => setAgree(e.target.checked)} />}
  label="Accept terms and conditions" />

{/* Radio group — RadioGroup owns the value; each Radio has a `value`. */}
<FormControl>
  <FormLabel>Plan</FormLabel>
  <RadioGroup row value={plan} onChange={(e) => setPlan(e.target.value)}>
    <FormControlLabel value="free" control={<Radio />} label="Free" />
    <FormControlLabel value="pro"  control={<Radio />} label="Pro" />
  </RadioGroup>
</FormControl>

{/* Switch — for settings toggles. */}
<FormControlLabel
  control={<Switch checked={dark} onChange={(e) => setDark(e.target.checked)} />}
  label="Dark mode" />
```

#### Slider — numeric range input

```tsx
import { Slider, Box, Typography } from '@mui/material';

<Box sx={{ width: 300 }}>
  <Typography>Volume: {volume}</Typography>
  <Slider value={volume} onChange={(_e, v) => setVolume(v as number)}
          min={0} max={100} step={1} marks valueLabelDisplay="auto"
          aria-label="Volume" />
</Box>

{/* Range: value is a two-element array. disableSwap stops thumbs crossing. */}
<Slider value={range} onChange={(_e, v) => setRange(v as number[])}
        min={0} max={1000} step={10} valueLabelDisplay="on" disableSwap />
```

### 8.3 Data Display

#### Card — a content surface with optional media and actions

Compose from `CardHeader`, `CardMedia`, `CardContent`, and `CardActions`. Wrap the whole thing in `CardActionArea` if the *entire* card should be one clickable link.

```tsx
import { Card, CardHeader, CardMedia, CardContent, CardActions,
         Avatar, Typography, IconButton, Button } from '@mui/material';
import FavoriteIcon from '@mui/icons-material/Favorite';

<Card sx={{ maxWidth: 345 }}>
  <CardHeader avatar={<Avatar sx={{ bgcolor: 'error.main' }}>R</Avatar>}
              title="Shrimp Paella" subheader="Sep 14, 2026" />
  <CardMedia component="img" height="194" image="/paella.jpg" alt="Paella" />
  <CardContent>
    <Typography variant="body2" color="text.secondary">
      A perfect combination of saffron-infused rice and seafood…
    </Typography>
  </CardContent>
  <CardActions>
    <IconButton aria-label="favorite"><FavoriteIcon /></IconButton>
    <Button size="small">Learn More</Button>
  </CardActions>
</Card>
```

#### Paper — the base elevated surface

`Paper` is the primitive that `Card`, `Menu`, `Dialog`, etc. are built on: a background with elevation (shadow). Use it directly for simple raised panels.

```tsx
import { Paper, Typography } from '@mui/material';

{/* elevation: 0–24 controls shadow depth. */}
<Paper elevation={3} sx={{ p: 3, maxWidth: 400, mx: 'auto' }}>
  <Typography variant="h6">Raised panel</Typography>
</Paper>

{/* variant="outlined": a border instead of a shadow. */}
<Paper variant="outlined" sx={{ p: 2 }}>Bordered panel</Paper>
```

#### Accordion — collapsible sections

```tsx
import { Accordion, AccordionSummary, AccordionDetails, Typography } from '@mui/material';
import ExpandMoreIcon from '@mui/icons-material/ExpandMore';
import { useState } from 'react';

// Controlled: only one panel open at a time.
const [expanded, setExpanded] = useState<string | false>(false);
const handle = (panel: string) =>
  (_e: React.SyntheticEvent, isOpen: boolean) => setExpanded(isOpen ? panel : false);

{['p1', 'p2'].map((p, i) => (
  <Accordion key={p} expanded={expanded === p} onChange={handle(p)}>
    <AccordionSummary expandIcon={<ExpandMoreIcon />}>
      <Typography fontWeight={500}>Section {i + 1}</Typography>
    </AccordionSummary>
    <AccordionDetails><Typography>Content {i + 1}</Typography></AccordionDetails>
  </Accordion>
))}
```

#### List — vertical item lists

Use `ListItemButton` (not `ListItemText` alone) when items are clickable/navigable — it provides the hover/focus/selected states and ripple.

```tsx
import { List, ListItem, ListItemButton, ListItemIcon, ListItemText,
         ListSubheader, Divider } from '@mui/material';
import InboxIcon from '@mui/icons-material/Inbox';

<List subheader={<ListSubheader>Mail</ListSubheader>}
      sx={{ maxWidth: 360, bgcolor: 'background.paper' }}>
  <ListItem disablePadding>
    <ListItemButton selected onClick={() => setView('inbox')}>
      <ListItemIcon><InboxIcon /></ListItemIcon>
      <ListItemText primary="Inbox" secondary="42 unread" />
    </ListItemButton>
  </ListItem>
  <Divider />
</List>
```

#### Table — static/simple tables

For small, presentational tables. For sorting/filtering/pagination over many rows, use the **DataGrid** (§10) instead.

```tsx
import { Table, TableBody, TableCell, TableContainer,
         TableHead, TableRow, Paper } from '@mui/material';

<TableContainer component={Paper}>
  <Table aria-label="dessert table">
    <TableHead>
      <TableRow>
        <TableCell>Dessert</TableCell>
        <TableCell align="right">Calories</TableCell>
      </TableRow>
    </TableHead>
    <TableBody>
      {rows.map((row) => (
        <TableRow key={row.name}>
          <TableCell component="th" scope="row">{row.name}</TableCell>
          <TableCell align="right">{row.calories}</TableCell>
        </TableRow>
      ))}
    </TableBody>
  </Table>
</TableContainer>
```

#### Chip, Avatar, Badge, Tooltip

```tsx
import { Chip, Avatar, Badge, Tooltip, Stack, Button } from '@mui/material';
import MailIcon from '@mui/icons-material/Mail';

{/* Chip — compact tags/filters. onDelete adds an X; onClick makes it actionable. */}
<Stack direction="row" spacing={1}>
  <Chip label="Primary" color="primary" />
  <Chip label="Outlined" variant="outlined" />
  <Chip label="Deletable" onDelete={() => removeTag()} />
</Stack>

{/* Avatar — image, initials, or icon. */}
<Avatar alt="Jane" src="/jane.jpg" />
<Avatar sx={{ bgcolor: 'secondary.main' }}>JD</Avatar>

{/* Badge — small overlay indicator (counts/dots). */}
<Badge badgeContent={4} color="error"><MailIcon /></Badge>

{/* Tooltip — hover/focus hint. For a DISABLED element, wrap in <span>, because
    disabled elements don't fire the pointer events the tooltip listens for. */}
<Tooltip title="Log in to continue">
  <span><Button disabled>Purchase</Button></span>
</Tooltip>
```

### 8.4 Navigation

#### AppBar & Toolbar — the top app bar

`AppBar` is the colored bar; `Toolbar` provides correct height/padding for its contents. Use `display` breakpoints to swap a desktop nav for a mobile hamburger.

```tsx
import { AppBar, Toolbar, Typography, IconButton, Button, Box } from '@mui/material';
import MenuIcon from '@mui/icons-material/Menu';

<AppBar position="sticky"> {/* 'fixed' | 'sticky' | 'static' | 'absolute' */}
  <Toolbar>
    {/* Hamburger — only on mobile (hidden at md+). */}
    <IconButton edge="start" color="inherit" aria-label="open menu"
                sx={{ mr: 2, display: { md: 'none' } }} onClick={toggleDrawer}>
      <MenuIcon />
    </IconButton>

    {/* flexGrow:1 pushes the nav links to the right. */}
    <Typography variant="h6" sx={{ flexGrow: 1 }}>My App</Typography>

    {/* Desktop links — hidden on mobile. */}
    <Box sx={{ display: { xs: 'none', md: 'flex' }, gap: 1 }}>
      <Button color="inherit">Home</Button>
      <Button color="inherit">About</Button>
    </Box>
  </Toolbar>
</AppBar>
```

> **Gotcha — `position="fixed"` overlaps content.** A fixed AppBar floats above the page, hiding the top of your content. Add a spacer `<Toolbar />` (empty) right after it, or `sx={{ mt: 8 }}` on your main content.

#### Drawer — slide-in side panel

```tsx
import { Drawer, Box, List, ListItem, ListItemButton, ListItemText } from '@mui/material';

<Drawer anchor="left" open={open} onClose={() => setOpen(false)}>
  {/* role/onClick close the drawer when a nav item is chosen. */}
  <Box sx={{ width: 250 }} role="presentation" onClick={() => setOpen(false)}>
    <List>
      {['Home', 'About', 'Contact'].map((item) => (
        <ListItem key={item} disablePadding>
          <ListItemButton component="a" href={`/${item.toLowerCase()}`}>
            <ListItemText primary={item} />
          </ListItemButton>
        </ListItem>
      ))}
    </List>
  </Box>
</Drawer>

{/* Permanent variant = a fixed sidebar (no overlay). */}
<Drawer variant="permanent" sx={{ '& .MuiDrawer-paper': { width: 240 } }}>
  …
</Drawer>
```

#### Tabs — switch between views

```tsx
import { Tabs, Tab, Box } from '@mui/material';
import { useState } from 'react';

const [tab, setTab] = useState(0);

<Box sx={{ borderBottom: 1, borderColor: 'divider' }}>
  <Tabs value={tab} onChange={(_e, v) => setTab(v)} aria-label="main tabs">
    <Tab label="Overview" id="tab-0" aria-controls="panel-0" />
    <Tab label="Settings" id="tab-1" aria-controls="panel-1" />
  </Tabs>
</Box>

{/* Render only the active panel. role/aria wire up screen-reader semantics. */}
{[<Overview />, <Settings />].map((panel, i) => (
  <Box key={i} role="tabpanel" hidden={tab !== i} id={`panel-${i}`}
       aria-labelledby={`tab-${i}`} sx={{ p: 3 }}>
    {tab === i && panel}
  </Box>
))}
```

#### Menu — button-anchored dropdown

The pattern: track an `anchorEl` (the element to position against). Open when set, close when nulled.

```tsx
import { Menu, MenuItem, Button, Divider, ListItemIcon, ListItemText } from '@mui/material';
import Logout from '@mui/icons-material/Logout';
import { useState } from 'react';

function UserMenu() {
  const [anchorEl, setAnchorEl] = useState<null | HTMLElement>(null);
  const open = Boolean(anchorEl);
  return (
    <>
      <Button onClick={(e) => setAnchorEl(e.currentTarget)}>Account</Button>
      <Menu anchorEl={anchorEl} open={open} onClose={() => setAnchorEl(null)}
            anchorOrigin={{ horizontal: 'right', vertical: 'bottom' }}
            transformOrigin={{ horizontal: 'right', vertical: 'top' }}>
        <MenuItem onClick={() => { goProfile(); setAnchorEl(null); }}>
          <ListItemText>Profile</ListItemText>
        </MenuItem>
        <Divider />
        <MenuItem onClick={logout}>
          <ListItemIcon><Logout fontSize="small" /></ListItemIcon>
          <ListItemText>Logout</ListItemText>
        </MenuItem>
      </Menu>
    </>
  );
}
```

#### Breadcrumbs & Pagination

```tsx
import { Breadcrumbs, Link, Typography, Pagination } from '@mui/material';

<Breadcrumbs separator="›" aria-label="breadcrumb">
  <Link underline="hover" color="inherit" href="/">Home</Link>
  <Link underline="hover" color="inherit" href="/products">Products</Link>
  <Typography color="text.primary">Sneakers</Typography>
</Breadcrumbs>

<Pagination count={10} page={page} onChange={(_e, v) => setPage(v)}
            color="primary" shape="rounded" showFirstButton showLastButton />
```

### 8.5 Feedback & Overlays

#### Dialog — modal for confirmations and forms

Compose from `DialogTitle`, `DialogContent` (+ `DialogContentText`), and `DialogActions`. MUI traps focus inside and restores it on close — a key accessibility feature you get free.

```tsx
import { Dialog, DialogTitle, DialogContent, DialogContentText,
         DialogActions, Button } from '@mui/material';

<Dialog open={open} onClose={() => setOpen(false)}
        maxWidth="sm" fullWidth aria-labelledby="confirm-title">
  <DialogTitle id="confirm-title">Confirm Delete</DialogTitle>
  <DialogContent>
    <DialogContentText>
      This action cannot be undone. Delete this item?
    </DialogContentText>
  </DialogContent>
  <DialogActions>
    <Button onClick={() => setOpen(false)}>Cancel</Button>
    <Button onClick={handleDelete} color="error" variant="contained" autoFocus>
      Delete
    </Button>
  </DialogActions>
</Dialog>
```

#### Snackbar & Alert — toasts and inline messages

`Snackbar` positions a transient, auto-dismissing notification. Put an `Alert` inside it for the colored severity styling. `Alert` standalone is an *inline* message (not a toast).

```tsx
import { Snackbar, Alert } from '@mui/material';

<Snackbar open={open} autoHideDuration={4000} onClose={() => setOpen(false)}
          anchorOrigin={{ vertical: 'bottom', horizontal: 'center' }}>
  <Alert onClose={() => setOpen(false)} severity="success"
         variant="filled" sx={{ width: '100%' }}>
    Saved successfully!
  </Alert>
</Snackbar>

{/* Inline alert (always visible, not a toast). */}
<Alert severity="warning" sx={{ mb: 2 }}>Your session expires in 5 minutes.</Alert>
```

> **Best practice:** For app-wide toasts, build a small context/provider that exposes `notify(message, severity)` and renders a single `Snackbar`, instead of scattering Snackbar state across components.

#### Progress, Backdrop, Skeleton — loading states

```tsx
import { CircularProgress, LinearProgress, Backdrop, Skeleton,
         Card, CardContent, Stack } from '@mui/material';

{/* Indeterminate spinners/bars (unknown duration). */}
<CircularProgress />
<LinearProgress />
{/* Determinate (known %): */}
<LinearProgress variant="determinate" value={75} />

{/* Full-screen blocking overlay. zIndex above the drawer. */}
<Backdrop open={isLoading} sx={{ color: '#fff', zIndex: (t) => t.zIndex.drawer + 1 }}>
  <CircularProgress color="inherit" />
</Backdrop>

{/* Skeleton — placeholder matching the content's SHAPE (better than a spinner
    for perceived performance). Match the real layout for a seamless swap. */}
{isLoading ? (
  <Card sx={{ maxWidth: 345 }}>
    <Skeleton variant="rectangular" height={194} />
    <CardContent>
      <Skeleton variant="text" sx={{ fontSize: '1.5rem' }} />
      <Skeleton variant="text" width="80%" />
      <Stack direction="row" spacing={1} sx={{ mt: 2 }}>
        <Skeleton variant="circular" width={40} height={40} />
        <Skeleton variant="text" width="60%" />
      </Stack>
    </CardContent>
  </Card>
) : (
  <RealCard />
)}
```

---

## 9. Icons — @mui/icons-material

### What it is

`@mui/icons-material` packages the entire Material Symbols set (~2,100 icons) as individual React components. Each icon is an SVG wrapped in MUI's `SvgIcon`, so it inherits `fontSize` and `color` from the theme just like text — that consistency is the point.

### Install & import

```bash
npm install @mui/icons-material
```

```tsx
// ALWAYS prefer named PATH imports — they tree-shake to just that one icon.
import DeleteIcon from '@mui/icons-material/Delete';
import EditIcon from '@mui/icons-material/Edit';
import SearchIcon from '@mui/icons-material/Search';

// Each Material icon has style variants — suffix the name:
import HomeOutlinedIcon from '@mui/icons-material/HomeOutlined'; // outlined
import HomeRoundedIcon from '@mui/icons-material/HomeRounded';   // rounded
import HomeTwoToneIcon from '@mui/icons-material/HomeTwoTone';   // two-tone
```

### Props

```tsx
{/* Size: fontSize keyword or sx override. */}
<HomeIcon />                       {/* 24px (medium) */}
<HomeIcon fontSize="small" />      {/* 20px */}
<HomeIcon fontSize="large" />      {/* 35px */}
<HomeIcon sx={{ fontSize: 48 }} /> {/* custom */}

{/* Color: any palette intention or a literal. */}
<CheckCircleIcon color="success" />
<DeleteIcon color="error" />
<SearchIcon sx={{ color: 'primary.main' }} />
```

> **Gotcha — the barrel import bombs your bundle.** `import { Delete, Edit } from '@mui/icons-material'` pulls the *entire* icon set into dev builds (thousands of modules → painfully slow HMR), and can bloat production if tree-shaking misfires. Use path imports: `import DeleteIcon from '@mui/icons-material/Delete'`. This is the single most common MUI performance mistake.

---

## 10. DataGrid — @mui/x-data-grid

### What it is and when to reach for it

The `DataGrid` is MUI's spreadsheet-grade table: sorting, filtering, pagination, column resizing/reordering, row selection, inline editing, CSV export, and (in the paid Pro/Premium tiers) virtualization for huge datasets, tree data, grouping, and aggregation. The **community** edition is free and covers sorting/filtering/pagination/selection — enough for most CRUD dashboards.

Use it instead of the basic `Table` (§8.3) whenever you have *many rows* or need *interaction* (sort/filter/paginate). Use the plain `Table` for small static tables where the DataGrid's footprint isn't justified.

The core mental model: you give it **`columns`** (definitions: which field, header, width, type, custom renderers) and **`rows`** (your data, each needing a unique `id`). The grid handles everything else.

### Install

```bash
npm install @mui/x-data-grid
```

### Basic usage (client-side data)

```tsx
import { DataGrid, type GridColDef } from '@mui/x-data-grid';
import { Box, Button } from '@mui/material';
import { useState } from 'react';

// Column definitions describe the SHAPE of the table.
const columns: GridColDef[] = [
  { field: 'id', headerName: 'ID', width: 70 },
  { field: 'firstName', headerName: 'First name', width: 130 },
  { field: 'lastName', headerName: 'Last name', width: 130 },
  {
    field: 'fullName', headerName: 'Full name', width: 160,
    // valueGetter computes a value not present in the row data.
    // ⚡ v6/v7 signature: (value, row) — NOT the old params object.
    valueGetter: (_value, row) => `${row.firstName ?? ''} ${row.lastName ?? ''}`,
  },
  { field: 'age', headerName: 'Age', type: 'number', width: 90, editable: true },
  {
    field: 'actions', headerName: 'Actions', width: 120, sortable: false,
    // renderCell injects custom JSX into a cell. params.row is the full row.
    renderCell: (params) => (
      <Button size="small" onClick={() => editRow(params.row)}>Edit</Button>
    ),
  },
];

// Rows: an array of objects. Each MUST have a unique `id` (or pass getRowId).
const rows = [
  { id: 1, firstName: 'Jon', lastName: 'Snow', age: 35 },
  { id: 2, firstName: 'Cersei', lastName: 'Lannister', age: 42 },
];

function UserTable() {
  const [selection, setSelection] = useState<number[]>([]);
  return (
    // The grid needs an explicit height (it doesn't grow with content).
    <Box sx={{ height: 400, width: '100%' }}>
      <DataGrid
        rows={rows}
        columns={columns}
        // initialState seeds uncontrolled features like pagination.
        initialState={{ pagination: { paginationModel: { page: 0, pageSize: 10 } } }}
        pageSizeOptions={[5, 10, 25]}
        checkboxSelection
        onRowSelectionModelChange={(ids) => setSelection(ids as number[])}
        onRowClick={(params) => console.log(params.row)}
        // Style inner slots via the MuiDataGrid-* class names:
        sx={{ '& .MuiDataGrid-columnHeaders': { bgcolor: 'primary.main', color: '#fff' } }}
      />
    </Box>
  );
}
```

### Server-side data (large datasets / API-backed)

For big data you don't load everything into the browser. Switch the grid into *server mode* so it reports user intent (page change, sort, filter) via callbacks, and you fetch the matching slice from your API. Pair this with `TANSTACK_QUERY_GUIDE.md` for caching/loading.

```tsx
<DataGrid
  rows={rows}                 // only the CURRENT page of rows
  columns={columns}
  rowCount={totalCount}       // total across all pages, from the API
  paginationMode="server"     // grid no longer slices locally…
  sortingMode="server"        // …it asks YOU to fetch sorted/filtered data
  filterMode="server"
  loading={isFetching}        // shows the built-in loading overlay
  onPaginationModelChange={(m) => fetchData({ page: m.page, pageSize: m.pageSize })}
  onSortModelChange={(m) => fetchData({ sortBy: m[0]?.field, sortDir: m[0]?.sort })}
  onFilterModelChange={(m) => fetchData({ filters: m.items })}
/>
```

> **⚡ Version note (v6/v7 API changes):** Several callbacks changed signatures between v5 and v6+. Most notably `valueGetter`/`valueFormatter` now receive `(value, row, column, apiRef)` instead of a single `params` object. If you copy an old snippet and `params.row` is undefined, this is why.

> **Gotcha — "no rows / collapsed grid."** The DataGrid renders nothing visible if its container has no height. It does **not** auto-size to content. Give the wrapper a fixed `height`, or use the `autoHeight` prop (fine for small grids, not for virtualized large ones).

---

## 11. Forms with MUI + React Hook Form

Forms are where MUI components meet validation. There are two tiers: simple controlled state for tiny forms, and **React Hook Form (RHF)** for anything real. This section assumes RHF basics from `REACT_HOOK_FORM_GUIDE.md`.

### The controlled-state baseline (small forms)

For a two-field form, plain `useState` is fine. The pattern: each `TextField` is controlled (`value` + `onChange`), and you validate on submit, feeding errors into `error`/`helperText`.

```tsx
import { Box, TextField, Button } from '@mui/material';
import { useState } from 'react';

function LoginForm() {
  const [form, setForm] = useState({ email: '', password: '' });
  const [errors, setErrors] = useState<Record<string, string>>({});

  const change = (e: React.ChangeEvent<HTMLInputElement>) =>
    setForm((f) => ({ ...f, [e.target.name]: e.target.value }));

  const submit = (e: React.FormEvent) => {
    e.preventDefault();
    const errs: Record<string, string> = {};
    if (!form.email.includes('@')) errs.email = 'Enter a valid email';
    if (form.password.length < 8) errs.password = 'Min 8 characters';
    setErrors(errs);
    if (Object.keys(errs).length === 0) { /* submit */ }
  };

  return (
    <Box component="form" onSubmit={submit} noValidate sx={{ maxWidth: 400 }}>
      <TextField name="email" label="Email" type="email" fullWidth margin="normal"
                 value={form.email} onChange={change}
                 error={!!errors.email} helperText={errors.email} />
      <TextField name="password" label="Password" type="password" fullWidth margin="normal"
                 value={form.password} onChange={change}
                 error={!!errors.password} helperText={errors.password} />
      <Button type="submit" fullWidth variant="contained" sx={{ mt: 2 }}>Sign In</Button>
    </Box>
  );
}
```

### Why React Hook Form for real forms

`useState`-per-field re-renders the whole form on every keystroke and makes validation logic sprawl. **RHF** keeps inputs *uncontrolled* (reading values via refs), so typing in one field doesn't re-render the others — a big performance win on large forms — and centralizes validation (especially with a schema library like Zod). The trick is wiring RHF to MUI's components, which split into two cases.

### The two integration patterns — the key distinction

- **`register()`** works for components that expose a native `<input>` and forward standard `ref`/`onChange`/`onBlur`. MUI's `TextField` qualifies. Spread `{...register('field')}` onto it.
- **`Controller`** is required for components that *don't* behave like a native input — `Select`, `Autocomplete`, `Checkbox`, `Switch`, `Slider`, date pickers. These have non-standard value/onChange shapes, so `Controller` adapts RHF's controlled API to them.

That single rule — *native input → `register`; custom MUI control → `Controller`* — is the whole game.

```tsx
// npm install react-hook-form zod @hookform/resolvers
import { useForm, Controller, type SubmitHandler } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { Box, TextField, Button, FormControl, InputLabel, Select, MenuItem,
         FormControlLabel, Checkbox, FormHelperText } from '@mui/material';

// 1) Schema = single source of truth for validation + inferred types.
const schema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Min 8 characters'),
  role: z.string().min(1, 'Select a role'),
  agree: z.boolean().refine((v) => v, 'You must agree'),
});
type FormData = z.infer<typeof schema>;

function RegistrationForm() {
  const { register, control, handleSubmit, formState: { errors, isSubmitting } } =
    useForm<FormData>({ resolver: zodResolver(schema), defaultValues: { agree: false } });

  const onSubmit: SubmitHandler<FormData> = async (data) => { await registerUser(data); };

  return (
    <Box component="form" onSubmit={handleSubmit(onSubmit)} sx={{ maxWidth: 400 }}>
      {/* TextField → register(). Spread connects ref/name/onChange/onBlur. */}
      <TextField label="Email" fullWidth margin="normal"
        {...register('email')}
        error={!!errors.email} helperText={errors.email?.message} />

      <TextField label="Password" type="password" fullWidth margin="normal"
        {...register('password')}
        error={!!errors.password} helperText={errors.password?.message} />

      {/* Select → Controller (Select's onChange/value aren't native-input-shaped). */}
      <Controller name="role" control={control} render={({ field }) => (
        <FormControl fullWidth margin="normal" error={!!errors.role}>
          <InputLabel>Role</InputLabel>
          <Select {...field} label="Role">
            <MenuItem value="user">User</MenuItem>
            <MenuItem value="admin">Admin</MenuItem>
          </Select>
          {errors.role && <FormHelperText>{errors.role.message}</FormHelperText>}
        </FormControl>
      )} />

      {/* Checkbox → Controller. Note `checked={field.value}` (boolean, not value). */}
      <Controller name="agree" control={control} render={({ field }) => (
        <FormControlLabel
          control={<Checkbox {...field} checked={field.value} />}
          label="I agree to the terms" />
      )} />
      {errors.agree && <FormHelperText error>{errors.agree.message}</FormHelperText>}

      <Button type="submit" fullWidth variant="contained"
              disabled={isSubmitting} sx={{ mt: 2 }}>
        {isSubmitting ? 'Registering…' : 'Register'}
      </Button>
    </Box>
  );
}
```

> **React 19 note:** React 19's Actions/`useActionState` and `<form action={fn}>` are an alternative for server-driven forms. They compose with MUI inputs (use `name` + `defaultValue`, read `FormData` on submit), but for rich client-side validation RHF + Zod remains the more ergonomic choice. See `REACT_19_GUIDE.md`.

> **Gotcha — Checkbox value vs checked.** A `Controller`-wrapped `Checkbox` needs `checked={field.value}` explicitly; if you only spread `{...field}` the checkbox won't reflect/update its boolean state correctly.

---

## 12. Dark Mode & Theme Toggling

There are two ways to do dark mode in MUI. The modern one (CSS variables + `colorSchemes`) is strongly preferred because it eliminates the SSR flash. The manual one is simpler to grasp and fine for client-only SPAs.

### Method 1 (recommended): CSS variables + `colorSchemes` + `useColorScheme`

⚡ **Version note:** Since v6, enabling `cssVariables: true` and defining `colorSchemes.light`/`colorSchemes.dark` lets MUI emit CSS custom properties and switch modes by toggling a `data-mui-color-scheme` attribute on `<html>` — *without re-rendering the React tree or recreating the theme*. Because the scheme can be set before hydration, there is **no flash of the wrong theme** on SSR. This is the right default for Next.js.

```tsx
// theme.ts — define BOTH schemes once.
import { createTheme } from '@mui/material/styles';

export const theme = createTheme({
  cssVariables: { colorSchemeSelector: 'class' }, // or true for the default selector
  colorSchemes: {
    light: { palette: { primary: { main: '#1976d2' } } },
    dark: {
      palette: {
        primary: { main: '#90caf9' },
        background: { default: '#121212', paper: '#1e1e1e' },
      },
    },
  },
});
```

```tsx
// DarkModeToggle.tsx — useColorScheme reads/sets the mode. No custom context needed.
'use client';
import { useColorScheme } from '@mui/material/styles';
import { IconButton, Tooltip } from '@mui/material';
import DarkModeIcon from '@mui/icons-material/DarkMode';
import LightModeIcon from '@mui/icons-material/LightMode';

export function DarkModeToggle() {
  const { mode, setMode } = useColorScheme();
  // mode can be 'light' | 'dark' | 'system' | undefined (before mount).
  if (!mode) return null; // avoid hydration mismatch before the scheme is known
  return (
    <Tooltip title="Toggle theme">
      <IconButton color="inherit"
                  onClick={() => setMode(mode === 'dark' ? 'light' : 'dark')}>
        {mode === 'dark' ? <LightModeIcon /> : <DarkModeIcon />}
      </IconButton>
    </Tooltip>
  );
}
```

> MUI persists the chosen scheme to `localStorage` automatically and can follow the OS preference via `mode = 'system'`. That's a lot of behavior you'd otherwise hand-roll.

### Method 2: manual palette-mode toggle (client-only)

Recreate the theme when the mode changes. Simple, but the whole tree re-renders on toggle and it can flash under SSR — so use it only in SPAs.

```tsx
'use client';
import { createTheme, ThemeProvider, CssBaseline } from '@mui/material';
import { createContext, useContext, useMemo, useState, type ReactNode } from 'react';

const ColorModeContext = createContext({ toggle: () => {} });
export const useColorMode = () => useContext(ColorModeContext);

export function ColorModeProvider({ children }: { children: ReactNode }) {
  // Initialize from localStorage so the choice survives reloads.
  const [mode, setMode] = useState<'light' | 'dark'>(() =>
    (typeof window !== 'undefined'
      ? (localStorage.getItem('mode') as 'light' | 'dark')
      : null) ?? 'light');

  const colorMode = useMemo(() => ({
    toggle: () => setMode((prev) => {
      const next = prev === 'light' ? 'dark' : 'light';
      localStorage.setItem('mode', next);
      return next;
    }),
  }), []);

  // Recreate the theme only when `mode` changes (memoized).
  const theme = useMemo(() => createTheme({ palette: { mode } }), [mode]);

  return (
    <ColorModeContext.Provider value={colorMode}>
      <ThemeProvider theme={theme}><CssBaseline />{children}</ThemeProvider>
    </ColorModeContext.Provider>
  );
}
```

> **Best practice:** Define mode-specific colors so dark mode is *designed*, not just inverted. Pure inversion often yields harsh contrast — set `background.default`/`paper` and tune `primary` for dark explicitly (as in Method 1).

---

## 13. Responsive Design

MUI is mobile-first: styles apply at a breakpoint *and up*. The breakpoint scale (min-widths) is the backbone:

| Key | Min width | Typical device |
|---|---|---|
| `xs` | 0px | Phones |
| `sm` | 600px | Large phones / small tablets |
| `md` | 900px | Tablets / small laptops |
| `lg` | 1200px | Desktops |
| `xl` | 1536px | Large monitors |

### Responsive `sx` (the everyday tool)

```tsx
<Box sx={{
  display: 'flex',
  flexDirection: { xs: 'column', md: 'row' }, // stack on phones, row on tablets+
  gap: { xs: 1, md: 3 },
  px: { xs: 2, sm: 4, lg: 8 },
  fontSize: { xs: '0.875rem', md: '1rem' },
}} />

{/* Show/hide per breakpoint — the responsive visibility idiom. */}
<Box sx={{ display: { xs: 'none', md: 'block' } }}>Desktop only</Box>
<Box sx={{ display: { xs: 'block', md: 'none' } }}>Mobile only</Box>
```

### Breakpoints inside `styled()`

```tsx
import { styled } from '@mui/material/styles';
import { Box } from '@mui/material';

const Panel = styled(Box)(({ theme }) => ({
  padding: theme.spacing(2),
  [theme.breakpoints.up('md')]:   { padding: theme.spacing(4), display: 'flex' },
  [theme.breakpoints.down('sm')]: { flexDirection: 'column' },
  [theme.breakpoints.between('sm', 'lg')]: { background: theme.palette.grey[100] },
}));
```

### `useMediaQuery` — responding in JS (not just CSS)

When you must change *behavior* or *which component renders* (not merely styles), use the hook. It returns a boolean and re-renders on breakpoint changes.

```tsx
import { useMediaQuery, useTheme } from '@mui/material';

function Adaptive() {
  const theme = useTheme();
  const isDesktop = useMediaQuery(theme.breakpoints.up('md'));     // ≥ 900px
  const isMobile  = useMediaQuery(theme.breakpoints.down('sm'));   // < 600px
  const prefersDark = useMediaQuery('(prefers-color-scheme: dark)'); // raw query
  return isDesktop ? <DesktopLayout /> : <MobileLayout />;
}
```

> **Best practice — prefer CSS over JS for responsiveness.** Responsive `sx`/`styled` is rendered once and adapts purely in CSS. `useMediaQuery` causes a JS re-render and, under SSR, the server can't know the screen size (returns `false` first), risking a hydration flash. Use the hook only when you genuinely need to branch *logic* or swap whole components.

### SSR caveat with `useMediaQuery`

```tsx
// On the server the viewport is unknown → defaults to no-match. To assume mobile
// until hydration (avoids a desktop-then-mobile flash), set defaultMatches:
const isMobile = useMediaQuery(theme.breakpoints.down('md'), { defaultMatches: true });
```

---

## 14. Performance, Tips & Gotchas

### Bundle size — imports that matter

```tsx
// Components: both forms tree-shake fine with modern bundlers.
import { Button, TextField } from '@mui/material';        // OK
import Button from '@mui/material/Button';                // also OK (explicit)

// Icons: ALWAYS path-import. The barrel import is the #1 perf footgun.
import DeleteIcon from '@mui/icons-material/Delete';      // ✓
// import { Delete } from '@mui/icons-material';          // ✗ pulls thousands of modules
```

### `sx` vs `styled()` — the cost

```tsx
// sx: the style OBJECT is recreated each render (cheap, Emotion caches the class),
//     but in long lists that adds up. Fine for one-offs and dynamic styles.
<Box sx={{ display: 'flex', p: 2 }} />

// styled(): base styles resolved ONCE at definition. Prefer for components that
//           render many times (table rows, list items, repeated cards).
const Row = styled(Box)({ display: 'flex', padding: 16 });
```

### Don't recreate the theme on render

```tsx
const theme = createTheme({ /* … */ });   // ✓ module scope — created once
function App() {
  const bad = createTheme({ /* … */ });   // ✗ new object every render → full re-style
}
```

### z-index layering (use the theme, don't guess)

```tsx
// MUI's z-index scale: mobileStepper 1000, fab 1050, speedDial 1050,
// appBar 1100, drawer 1200, modal 1300, snackbar 1400, tooltip 1500.
<Box sx={{ zIndex: (t) => t.zIndex.modal + 1 }}>Above any modal</Box>
```

### Hydration mismatches (Next.js)

```tsx
// useMediaQuery / useColorScheme can differ server vs client on first paint.
// Option A: render after mount.
const [mounted, setMounted] = useState(false);
useEffect(() => setMounted(true), []);
if (!mounted) return null;            // or a Skeleton
// Option B: useColorScheme → guard on `mode` being defined (see §12).
// Option C: useMediaQuery → { defaultMatches: true } (see §13).
```

### Common mistakes → fixes

| Mistake | Fix |
|---|---|
| Barrel-importing icons | Path import: `@mui/icons-material/IconName` |
| `ThemeProvider` outside `AppRouterCacheProvider` | Cache provider must be outermost (Next.js) |
| Old `<Grid item xs={12}>` in v6/v7 | `<Grid size={12}>` — no `item` prop |
| DataGrid renders blank | Give the wrapper a height (or `autoHeight`) |
| `valueGetter`/`renderCell` using `params.row` | v6/v7 signature is `(value, row, …)` |
| `TextField` controlled without `onChange` | Add `onChange`, or use `defaultValue` |
| `Tooltip` on a disabled `Button` | Wrap the button in `<span>` |
| Missing `key` on mapped `MenuItem`/`Grid` | Always key dynamic lists |
| `useColorScheme` without `colorSchemes` | Add `colorSchemes` (or use manual mode) |
| Recreating theme in render | Define at module scope |

### Handy patterns

```tsx
// Merge conditional styles via the sx array form:
<Box sx={[{ p: 2 }, isActive && { bgcolor: 'primary.light' }]} />

// Render MUI components AS router links via `component`:
import { Link as RouterLink } from 'react-router-dom';
<Button component={RouterLink} to="/about">About</Button>
<ListItemButton component={RouterLink} to="/profile">Profile</ListItemButton>

// Extend an outer theme in a nested provider (merge, don't replace):
const inner = createTheme(useTheme(), { palette: { primary: { main: '#f00' } } });
<ThemeProvider theme={inner}>…</ThemeProvider>

// Use Box for semantic, styled layout elements instead of bare divs:
<Box component="section" sx={{ py: 6 }}>…</Box>
```

---

## 15. Study Path & Build-to-Learn Projects

Build, don't just read. Each phase ends in a project that forces the concepts into your fingers.

### Phase 1 — Foundation (Week 1) **[B]**
1. Scaffold a Vite + React 19 app; install MUI + Emotion; wire `ThemeProvider` + `CssBaseline`.
2. Style entirely with `sx`: build a profile card from `Box`, `Avatar`, `Typography`, `Button`.
3. Customize the theme: change `primary.main`, `typography.fontFamily`, `shape.borderRadius`; watch it cascade.

**Build:** a polished profile card.

### Phase 2 — Layout (Week 2) **[B/I]**
4. Build an app shell (top nav, sidebar, main, footer) with `Box` flexbox only.
5. Convert spaced groups to `Stack`.
6. Build a responsive image gallery with the new `Grid` (`size={{ xs:12, sm:6, md:4, lg:3 }}`).
7. Wrap page content in `Container maxWidth="lg"`.

**Build:** a responsive landing page.

### Phase 3 — Forms (Week 3) **[I]**
8. Controlled sign-up form: `TextField`, `Select`, `Checkbox`, `Switch`.
9. Add manual validation with `error`/`helperText`.
10. Refactor to React Hook Form + Zod — `register` for TextFields, `Controller` for Select/Checkbox.

**Build:** a validated multi-section registration form.

### Phase 4 — Navigation & feedback (Week 4) **[I]**
11. Responsive `AppBar` + `Drawer` (hamburger on mobile, links on desktop).
12. Add `Tabs`, a confirmation `Dialog`, and a global `Snackbar`/`Alert` toast system.

**Build:** an admin dashboard shell with toasts.

### Phase 5 — Data & advanced (Week 5) **[A]**
13. `DataGrid` with sorting, filtering, pagination over a fake API; then switch to server mode.
14. Dark mode via `cssVariables` + `colorSchemes` + `useColorScheme`.
15. Global restyle of `Button`/`TextField`/`Card` via `components.styleOverrides` + `defaultProps`.
16. Rebuild the dashboard in Next.js App Router with `AppRouterCacheProvider` + the client provider pattern (§3).

**Build:** a full CRUD user-management dashboard (Next.js, dark mode, DataGrid, RHF forms).

### Phase 6 — Mastery (ongoing)
- Read each component's official prop table the first time you use it.
- Reskin one layout with three different brand palettes to internalize theming.
- Profile bundle size; confirm icon path-imports; audit `sx` in long lists → promote to `styled()`.
- Study MUI's free templates for real composition patterns.

---

### Quick Reference Card

| Task | Code |
|---|---|
| Install | `npm i @mui/material @emotion/react @emotion/styled` |
| Theme | `createTheme({})` → `<ThemeProvider theme={theme}>` |
| CSS reset | `<CssBaseline />` inside `ThemeProvider` |
| Spacing unit | `theme.spacing(1)` = 8px |
| sx padding | `sx={{ px: 2 }}` = left/right 16px |
| Responsive sx | `sx={{ fontSize: { xs: '1rem', md: '1.5rem' } }}` |
| Grid (v6/v7) | `<Grid container><Grid size={{ xs:12, md:6 }}>…` |
| Dark mode | `cssVariables:true` + `colorSchemes` + `useColorScheme()` |
| useMediaQuery | `useMediaQuery(theme.breakpoints.up('md'))` |
| Global override | `components: { MuiButton: { styleOverrides: { root: {} } } }` |
| Icon import | `import DeleteIcon from '@mui/icons-material/Delete'` |
| Router link | `<Button component={RouterLink} to="/x">` |
| Next.js SSR | `<AppRouterCacheProvider>` → `<Providers>` → children |
| RHF + MUI | `register()` for TextField; `Controller` for Select/Checkbox |
| DataGrid | `<DataGrid rows={} columns={} />` in a fixed-height `Box` |
