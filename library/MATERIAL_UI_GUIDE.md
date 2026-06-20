# Material UI (MUI) — Comprehensive Offline Reference Guide (2026)

> **What MUI is:** A production-ready React component library that implements Google's **Material Design** specification. You install it via npm, import ready-made components, and customize through a powerful **theming system**. Unlike shadcn/ui, you do NOT own the source — MUI components are versioned npm packages. Current stable major: **MUI v6** (`@mui/material@6`), with **v7** in active development. The styling engine is **Emotion** (CSS-in-JS) by default, with an experimental **Pigment CSS** zero-runtime compiler coming for v7.

---

## Table of Contents

1. [What MUI Is — Packages & Architecture](#1-what-mui-is--packages--architecture)
2. [Setup & Install](#2-setup--install)
3. [Next.js App Router Setup](#3-nextjs-app-router-setup)
4. [Theming — createTheme, ThemeProvider, Palette, Typography, Spacing](#4-theming)
5. [The `sx` Prop — The Primary Styling Tool](#5-the-sx-prop)
6. [Styling Approaches: sx vs styled() vs Theme Overrides](#6-styling-approaches)
7. [Layout Components: Box, Container, Stack, Grid](#7-layout-components)
8. [Core Component Reference](#8-core-component-reference)
   - [Text & Display](#text--display)
   - [Inputs & Forms](#inputs--forms)
   - [Data Display](#data-display)
   - [Navigation](#navigation)
   - [Feedback & Overlays](#feedback--overlays)
9. [Icons — @mui/icons-material](#9-icons----muiicons-material)
10. [DataGrid — @mui/x-data-grid](#10-datagrid----muix-data-grid)
11. [Forms with MUI — Controlled Inputs & react-hook-form](#11-forms-with-mui)
12. [Dark Mode & Theme Toggling](#12-dark-mode--theme-toggling)
13. [Responsive Design — Breakpoints & useMediaQuery](#13-responsive-design)
14. [Performance, Tips, Tricks & Gotchas](#14-performance-tips-tricks--gotchas)
15. [Study Path](#15-study-path)

---

## 1. What MUI Is — Packages & Architecture

### The Package Ecosystem

| Package | Purpose | Install |
|---|---|---|
| `@mui/material` | Core component library (Button, TextField, etc.) | Required |
| `@mui/icons-material` | 2,000+ SVG icons as React components | Optional |
| `@mui/x-data-grid` | Feature-rich data table (community edition free) | Optional |
| `@mui/x-date-pickers` | Date/time picker components | Optional |
| `@mui/lab` | Incubator for experimental components (MaskedInput, etc.) | Optional |
| `@mui/system` | Low-level `Box`, `sx`, `styled` utilities (bundled with material) | Included |
| `@emotion/react` | CSS-in-JS runtime (peer dependency) | Required |
| `@emotion/styled` | `styled()` API (peer dependency) | Required |

### Material Design Foundation

MUI implements **Material Design 3** (Material You) design tokens — elevation, shape, color roles (primary, secondary, error, warning, info, success), and motion. You are not required to follow Material Design strictly; the theme is fully overridable.

### The Styling Engine: Emotion

By default every MUI component is styled with **Emotion**. When you write:

```tsx
<Button sx={{ mt: 2, color: 'primary.main' }} />
```

Emotion generates a unique CSS class at runtime, injects a `<style>` tag, and applies it. This works great in SPAs. For SSR (Next.js), you need to extract styles on the server — covered in Section 3.

⚡ **Version note:** MUI v7 is introducing **Pigment CSS**, a zero-runtime CSS-in-JS tool (like Linaria/vanilla-extract) that moves style generation to build time. Pigment CSS eliminates Emotion's runtime cost but requires a bundler plugin. As of 2026, Pigment CSS is experimental — use Emotion unless you are bleeding-edge.

---

## 2. Setup & Install

### Standard React / Vite Project

```bash
# Install MUI + Emotion (peer deps)
npm install @mui/material @emotion/react @emotion/styled

# Optional but very common
npm install @mui/icons-material

# Optional data-grid
npm install @mui/x-data-grid
```

### Minimum App Setup

```tsx
// src/main.tsx  (Vite / CRA)
import React from 'react';
import ReactDOM from 'react-dom/client';
import { ThemeProvider, createTheme, CssBaseline } from '@mui/material';
import App from './App';

// createTheme with no args = MUI defaults (light mode, default palette)
const theme = createTheme();

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <ThemeProvider theme={theme}>
      {/* CssBaseline: CSS reset + applies background-color from theme */}
      <CssBaseline />
      <App />
    </ThemeProvider>
  </React.StrictMode>
);
```

### Roboto Font (recommended)

```html
<!-- index.html — Google Fonts CDN (or install @fontsource/roboto) -->
<link
  rel="stylesheet"
  href="https://fonts.googleapis.com/css2?family=Roboto:wght@300;400;500;700&display=swap"
/>
```

```bash
# Alternative: self-hosted font (works offline)
npm install @fontsource/roboto
```

```tsx
// Import at the top of main.tsx
import '@fontsource/roboto/300.css';
import '@fontsource/roboto/400.css';
import '@fontsource/roboto/500.css';
import '@fontsource/roboto/700.css';
```

---

## 3. Next.js App Router Setup

Next.js App Router uses React Server Components (RSC) by default. MUI components use hooks and browser APIs, so they must render on the client. The official solution is `AppRouterCacheProvider` from `@mui/material-nextjs`.

### Install

```bash
npm install @mui/material @emotion/react @emotion/styled @mui/material-nextjs
```

### Layout File

```tsx
// app/layout.tsx  — Root layout (Server Component)
import type { Metadata } from 'next';
import { AppRouterCacheProvider } from '@mui/material-nextjs/v15-appRouter';
// ⚡ Version note: use 'v14-appRouter' for Next.js 14, 'v15-appRouter' for Next.js 15

export const metadata: Metadata = { title: 'My MUI App' };

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {/*
          AppRouterCacheProvider handles SSR style extraction for Emotion.
          It MUST wrap everything — it is a server component wrapper.
        */}
        <AppRouterCacheProvider>
          {children}
        </AppRouterCacheProvider>
      </body>
    </html>
  );
}
```

### Client ThemeProvider

```tsx
// app/providers.tsx  — "use client" because ThemeProvider uses context
'use client';

import { ThemeProvider, createTheme, CssBaseline } from '@mui/material';
import type { ReactNode } from 'react';

const theme = createTheme({
  palette: {
    primary: { main: '#1976d2' },
  },
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

```tsx
// app/layout.tsx  — Wire in the providers
import { AppRouterCacheProvider } from '@mui/material-nextjs/v15-appRouter';
import { Providers } from './providers';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <AppRouterCacheProvider>
          <Providers>{children}</Providers>
        </AppRouterCacheProvider>
      </body>
    </html>
  );
}
```

### Using MUI Components in RSC Pages

```tsx
// app/page.tsx — Server Component (no 'use client' needed for static content)
// MUI components that are purely presentational CAN be imported in RSCs
// as long as they don't use hooks internally.
// Safer: keep interactive components in separate 'use client' files.

import { Typography, Container } from '@mui/material';
import { InteractiveSection } from './InteractiveSection'; // 'use client'

export default function HomePage() {
  return (
    <Container maxWidth="md" sx={{ py: 4 }}>
      <Typography variant="h1">Hello MUI + App Router</Typography>
      <InteractiveSection />
    </Container>
  );
}
```

⚡ **Version note:** Many MUI components that contain `useEffect`, `useState`, or event handlers will throw "You're importing a component that needs X" in RSC. When in doubt, add `'use client'` to the file that imports the MUI component.

---

## 4. Theming

The theme is the single source of truth for every visual token in your app.

### createTheme — Full Example

```tsx
// theme.ts
import { createTheme } from '@mui/material/styles';

const theme = createTheme({
  // ─── Palette ────────────────────────────────────────────────────────────────
  palette: {
    mode: 'light', // 'light' | 'dark'
    primary: {
      main: '#1976d2',    // Required
      light: '#42a5f5',   // Optional; MUI auto-calculates if omitted
      dark: '#1565c0',
      contrastText: '#fff',
    },
    secondary: {
      main: '#9c27b0',
    },
    error:   { main: '#d32f2f' },
    warning: { main: '#ed6c02' },
    info:    { main: '#0288d1' },
    success: { main: '#2e7d32' },
    // Custom color (must augment the type — see TypeScript section below)
    // neutral: { main: '#64748B' },
    background: {
      default: '#f5f5f5',
      paper: '#ffffff',
    },
    text: {
      primary: 'rgba(0,0,0,0.87)',
      secondary: 'rgba(0,0,0,0.60)',
      disabled: 'rgba(0,0,0,0.38)',
    },
  },

  // ─── Typography ─────────────────────────────────────────────────────────────
  typography: {
    fontFamily: '"Roboto", "Helvetica", "Arial", sans-serif',
    fontSize: 14,           // Base font size (px) — affects rem calculation
    h1: { fontSize: '2.5rem', fontWeight: 700 },
    h2: { fontSize: '2rem',   fontWeight: 600 },
    body1: { fontSize: '1rem', lineHeight: 1.6 },
    button: { textTransform: 'none' }, // Disable ALL-CAPS buttons globally
  },

  // ─── Spacing ────────────────────────────────────────────────────────────────
  // Default: 8px. theme.spacing(1) = 8px, theme.spacing(2) = 16px, etc.
  spacing: 8, // or a custom function: (factor) => `${factor * 0.5}rem`

  // ─── Shape ──────────────────────────────────────────────────────────────────
  shape: {
    borderRadius: 8, // Default for all components. 4 = Material default.
  },

  // ─── Breakpoints ────────────────────────────────────────────────────────────
  breakpoints: {
    values: {
      xs: 0,
      sm: 600,
      md: 900,
      lg: 1200,
      xl: 1536,
    },
  },

  // ─── Component Overrides ────────────────────────────────────────────────────
  components: {
    MuiButton: {
      defaultProps: {
        disableElevation: true, // Remove box-shadow from all Buttons
        variant: 'contained',   // Default variant
      },
      styleOverrides: {
        root: {
          borderRadius: 20, // Pill-shaped buttons everywhere
        },
        containedPrimary: {
          // Target a specific variant+color combo
          '&:hover': { backgroundColor: '#1565c0' },
        },
      },
    },
    MuiTextField: {
      defaultProps: {
        variant: 'outlined',
        size: 'small',
      },
    },
    MuiCard: {
      styleOverrides: {
        root: {
          boxShadow: '0 2px 8px rgba(0,0,0,0.12)',
        },
      },
    },
  },
});

export default theme;
```

### CssBaseline

Always include `<CssBaseline />` inside `<ThemeProvider>`:

```tsx
// Applies:
// - box-sizing: border-box globally
// - Removes default margin/padding from body
// - Sets background-color to theme.palette.background.default
// - Sets font-family to theme.typography.fontFamily
<ThemeProvider theme={theme}>
  <CssBaseline />
  <App />
</ThemeProvider>
```

### Accessing Theme Tokens in Code

```tsx
import { useTheme } from '@mui/material/styles';

function MyComponent() {
  const theme = useTheme();

  return (
    <div style={{
      backgroundColor: theme.palette.primary.main,
      padding: theme.spacing(2),       // 16px
      borderRadius: theme.shape.borderRadius,
    }}>
      Content
    </div>
  );
}
```

### TypeScript — Augmenting the Theme

```tsx
// theme.d.ts — extend MUI types to add custom palette colors
import '@mui/material/styles';

declare module '@mui/material/styles' {
  interface Palette {
    neutral: Palette['primary'];
  }
  interface PaletteOptions {
    neutral?: PaletteOptions['primary'];
  }
}

// Then in createTheme:
const theme = createTheme({
  palette: {
    neutral: { main: '#64748B' },
  },
});
```

---

## 5. The `sx` Prop

The `sx` prop is the primary, recommended way to style individual MUI components. It accepts an object of CSS properties with superpowers.

### Basic Usage

```tsx
import { Box } from '@mui/material';

// sx is available on ALL MUI components (Box, Button, Card, etc.)
<Box
  sx={{
    // Standard CSS properties (camelCase)
    backgroundColor: '#f5f5f5',
    borderRadius: '8px',
    padding: '16px',

    // Shorthand aliases (MUI-specific)
    p: 2,          // padding: theme.spacing(2) = 16px
    px: 3,         // padding-left + padding-right: 24px
    py: 1,         // padding-top + padding-bottom: 8px
    m: 'auto',     // margin: auto
    mt: 2,         // margin-top: 16px
    mb: { xs: 2, md: 4 }, // responsive margin-bottom

    // Theme token access (dot notation as string)
    color: 'primary.main',        // theme.palette.primary.main
    bgcolor: 'background.paper',  // theme.palette.background.paper
    fontWeight: 'typography.fontWeightBold', // theme.typography.fontWeightBold

    // Pseudo-classes
    '&:hover': { opacity: 0.8 },
    '&:focus-visible': { outline: '2px solid', outlineColor: 'primary.main' },

    // CSS media queries (manual)
    '@media (max-width: 600px)': { fontSize: '0.875rem' },
  }}
>
  Content
</Box>
```

### Spacing Shorthands

```tsx
// Spacing values are multiplied by theme.spacing (default 8px)
sx={{
  p: 0,    // 0px
  p: 1,    // 8px
  p: 2,    // 16px
  p: 3,    // 24px
  p: '10px', // literal value (string bypasses multiplier)
}}
```

### Responsive Values

```tsx
// Pass an object with breakpoint keys (mobile-first)
sx={{
  // xs is the base (mobile), overridden at sm, md, lg, xl
  fontSize: { xs: '1rem', sm: '1.25rem', md: '1.5rem' },
  display: { xs: 'block', md: 'flex' },
  width: { xs: '100%', md: '50%', lg: '33%' },

  // Or use an array (index = xs, sm, md, lg, xl)
  padding: [1, 2, 3], // xs=8px, sm=16px, md=24px
}}
```

### Theme Function Access

```tsx
// Use a callback to access the full theme object
sx={(theme) => ({
  backgroundColor: theme.palette.mode === 'dark'
    ? theme.palette.grey[800]
    : theme.palette.grey[100],
  [theme.breakpoints.up('md')]: {
    padding: theme.spacing(4),
  },
})}
```

---

## 6. Styling Approaches

### Summary Table

| Approach | When to Use | Runtime Cost |
|---|---|---|
| `sx` prop | One-off styles on a single component | Low (Emotion caches classes) |
| `styled()` | Reusable styled component, complex variants | Low (same as sx) |
| Theme `styleOverrides` | Globally override a MUI component across the entire app | Zero (runs at theme creation) |
| Theme `defaultProps` | Set default prop values globally | Zero |
| Inline `style` | Truly dynamic values (e.g. JS-animated values) | None (no CSS class) |

### 1. `sx` Prop — One-off Styles

```tsx
<Button sx={{ mt: 2, px: 4 }}>Submit</Button>
```

### 2. `styled()` — Reusable Styled Component

```tsx
import { styled } from '@mui/material/styles';
import { Button } from '@mui/material';

// styled() wraps any MUI or HTML element
const PillButton = styled(Button)(({ theme }) => ({
  borderRadius: theme.shape.borderRadius * 4,
  paddingLeft: theme.spacing(3),
  paddingRight: theme.spacing(3),
  // Respond to color prop variations:
  // '&.MuiButton-containedPrimary': { ... }
}));

// Usage — PillButton accepts all Button props
<PillButton variant="contained" onClick={handleClick}>
  Click Me
</PillButton>
```

```tsx
// styled() on an HTML element with shouldForwardProp
import { styled } from '@mui/material/styles';

interface HighlightProps { active?: boolean; }

const Highlight = styled('div', {
  // Prevents 'active' from being forwarded to the DOM element
  shouldForwardProp: (prop) => prop !== 'active',
})<HighlightProps>(({ theme, active }) => ({
  backgroundColor: active ? theme.palette.primary.light : 'transparent',
  transition: 'background-color 0.2s',
}));

<Highlight active={isSelected}>Card content</Highlight>
```

### 3. Theme Component Overrides — Global Styles

```tsx
// In createTheme — affects EVERY instance of the component
const theme = createTheme({
  components: {
    MuiButton: {
      // defaultProps — change prop defaults
      defaultProps: {
        variant: 'contained',
        disableElevation: true,
      },
      // styleOverrides — CSS overrides per slot
      styleOverrides: {
        root: ({ theme, ownerState }) => ({
          // ownerState = the component's current props
          borderRadius: ownerState.variant === 'outlined' ? 4 : 20,
        }),
        startIcon: {
          marginRight: 4,
        },
      },
    },
    MuiTextField: {
      defaultProps: {
        size: 'small',
        variant: 'outlined',
        fullWidth: true,
      },
    },
  },
});
```

---

## 7. Layout Components

### Box

`Box` is a generic `div` with the `sx` prop. It is the building block for everything.

```tsx
import { Box } from '@mui/material';

// Basic flex container
<Box
  sx={{
    display: 'flex',
    alignItems: 'center',
    gap: 2,
    p: 2,
    bgcolor: 'background.paper',
    borderRadius: 1,
    boxShadow: 1,
  }}
>
  <Avatar src="/avatar.jpg" />
  <Typography>John Doe</Typography>
</Box>

// The 'component' prop changes the underlying HTML element
<Box component="section" sx={{ py: 6 }}>
  <Typography component="h2" variant="h4">About</Typography>
</Box>

// Box as a flex column
<Box sx={{ display: 'flex', flexDirection: 'column', gap: 2, minHeight: '100vh' }}>
  <Header />
  <Box component="main" sx={{ flex: 1 }}>Content</Box>
  <Footer />
</Box>
```

### Container

Centers and constrains content width. Use as the outermost layout wrapper for page sections.

```tsx
import { Container } from '@mui/material';

// maxWidth: 'xs' | 'sm' | 'md' | 'lg' | 'xl' | false
<Container maxWidth="lg">
  <Typography variant="h3">My Page</Typography>
  {/* Content is constrained to 1200px and centered */}
</Container>

// disableGutters removes horizontal padding
<Container maxWidth="xl" disableGutters>
  <FullWidthHero />
</Container>
```

### Stack

A flex container that adds consistent spacing between children. Replaces "margin hacks" for stacking elements.

```tsx
import { Stack } from '@mui/material';

// Vertical stack (default direction = 'column')
<Stack spacing={2}>
  <TextField label="Email" />
  <TextField label="Password" type="password" />
  <Button type="submit">Login</Button>
</Stack>

// Horizontal stack with dividers
<Stack
  direction="row"
  spacing={2}
  divider={<Divider orientation="vertical" flexItem />}
  alignItems="center"
>
  <Typography>Profile</Typography>
  <Typography>Settings</Typography>
  <Typography>Logout</Typography>
</Stack>

// Responsive direction
<Stack direction={{ xs: 'column', sm: 'row' }} spacing={{ xs: 1, sm: 2 }}>
  <Card>...</Card>
  <Card>...</Card>
</Stack>
```

### Grid — v6+ API

⚡ **Version note:** MUI v6 introduced a **new Grid component** with a breaking API change. The old `<Grid item xs={12} md={6}>` pattern is replaced with `<Grid size={{ xs: 12, md: 6 }}>` (or `size="grow"`). Import from `@mui/material/Grid2` (also exported as `Unstable_Grid2` in v5).

```tsx
// v6+ Grid API — import Grid (Grid2 under the hood)
import Grid from '@mui/material/Grid2';
// or: import { Grid2 as Grid } from '@mui/material';

// 12-column grid layout
<Grid container spacing={2}>
  {/* size prop replaces item + xs/sm/md props */}
  <Grid size={12}>
    <Header />
  </Grid>
  <Grid size={{ xs: 12, md: 8 }}>
    <MainContent />
  </Grid>
  <Grid size={{ xs: 12, md: 4 }}>
    <Sidebar />
  </Grid>
</Grid>

// 'grow' — takes remaining space
<Grid container spacing={2}>
  <Grid size={4}>Fixed width</Grid>
  <Grid size="grow">Fills remaining space</Grid>
</Grid>

// Responsive card grid
<Grid container spacing={3}>
  {items.map((item) => (
    <Grid key={item.id} size={{ xs: 12, sm: 6, md: 4, lg: 3 }}>
      <Card>
        <CardContent>
          <Typography variant="h6">{item.title}</Typography>
        </CardContent>
      </Card>
    </Grid>
  ))}
</Grid>

// Nested grids
<Grid container spacing={2}>
  <Grid size={6}>
    <Grid container spacing={1}>
      <Grid size={6}><Box sx={{ bgcolor: 'primary.light', p: 2 }}>A</Box></Grid>
      <Grid size={6}><Box sx={{ bgcolor: 'secondary.light', p: 2 }}>B</Box></Grid>
    </Grid>
  </Grid>
</Grid>
```

---

## 8. Core Component Reference

### Text & Display

---

#### Typography

Renders semantic HTML text with theme-consistent styles.

- **Key props:** `variant`, `component`, `color`, `align`, `noWrap`, `gutterBottom`
- **Variants:** `h1`–`h6`, `subtitle1`, `subtitle2`, `body1`, `body2`, `caption`, `overline`, `button`

```tsx
import { Typography } from '@mui/material';

<Typography variant="h1" component="h1" gutterBottom>
  Page Title
</Typography>

<Typography variant="body1" color="text.secondary">
  Paragraph text in a secondary color.
</Typography>

<Typography variant="caption" display="block" sx={{ mt: 1 }}>
  Helper text below a field
</Typography>

{/* Change rendered element without changing visual style */}
<Typography variant="h4" component="p">
  Looks like H4, renders as paragraph
</Typography>

{/* Truncate with ellipsis */}
<Typography noWrap sx={{ maxWidth: 200 }}>
  This very long text will be truncated with an ellipsis at the end
</Typography>
```

---

### Inputs & Forms

---

#### Button

```tsx
import { Button } from '@mui/material';

{/* variant: 'contained' | 'outlined' | 'text' */}
<Button variant="contained" color="primary" onClick={handleClick}>
  Submit
</Button>

<Button variant="outlined" color="secondary" disabled>
  Disabled
</Button>

<Button variant="text" size="small">
  Cancel
</Button>

{/* With icons */}
import SendIcon from '@mui/icons-material/Send';
import DeleteIcon from '@mui/icons-material/Delete';

<Button variant="contained" startIcon={<SendIcon />}>
  Send
</Button>

<Button variant="outlined" color="error" endIcon={<DeleteIcon />}>
  Delete
</Button>

{/* Full width */}
<Button fullWidth variant="contained" sx={{ mt: 2 }}>
  Login
</Button>

{/* Loading state (MUI v6+) */}
<Button loading={isLoading} variant="contained">
  Save
</Button>
```

#### IconButton

Clickable icon with no label text. Good for toolbar actions.

```tsx
import { IconButton, Tooltip } from '@mui/material';
import FavoriteIcon from '@mui/icons-material/Favorite';
import MoreVertIcon from '@mui/icons-material/MoreVert';

<Tooltip title="Add to favorites">
  <IconButton aria-label="add to favorites" color="error">
    <FavoriteIcon />
  </IconButton>
</Tooltip>

{/* size: 'small' | 'medium' | 'large' */}
<IconButton size="small" onClick={openMenu}>
  <MoreVertIcon />
</IconButton>
```

#### TextField

The Swiss Army knife for user text input. Wraps `<input>` with label, helper text, and validation styling.

```tsx
import { TextField } from '@mui/material';

{/* Controlled input */}
const [value, setValue] = useState('');

<TextField
  label="Email Address"
  type="email"
  value={value}
  onChange={(e) => setValue(e.target.value)}
  variant="outlined"   // 'outlined' | 'filled' | 'standard'
  size="small"         // 'small' | 'medium'
  fullWidth
  required
  helperText="We'll never share your email."
  placeholder="you@example.com"
/>

{/* Error state */}
<TextField
  label="Password"
  type="password"
  error={!!errors.password}
  helperText={errors.password?.message}
/>

{/* Multiline (textarea) */}
<TextField
  label="Description"
  multiline
  rows={4}
  fullWidth
  variant="outlined"
/>

{/* With adornments */}
import { InputAdornment } from '@mui/material';
import SearchIcon from '@mui/icons-material/Search';

<TextField
  label="Search"
  InputProps={{
    startAdornment: (
      <InputAdornment position="start">
        <SearchIcon />
      </InputAdornment>
    ),
  }}
/>
```

#### Select

```tsx
import { FormControl, InputLabel, Select, MenuItem } from '@mui/material';

const [age, setAge] = useState('');

<FormControl fullWidth size="small">
  <InputLabel id="age-label">Age</InputLabel>
  <Select
    labelId="age-label"
    value={age}
    label="Age"
    onChange={(e) => setAge(e.target.value)}
  >
    <MenuItem value="">
      <em>None</em>
    </MenuItem>
    <MenuItem value={10}>Ten</MenuItem>
    <MenuItem value={20}>Twenty</MenuItem>
    <MenuItem value={30}>Thirty</MenuItem>
  </Select>
</FormControl>

{/* Multiple select */}
<Select multiple value={selectedItems} onChange={handleChange} renderValue={(selected) => selected.join(', ')}>
  {options.map((opt) => (
    <MenuItem key={opt} value={opt}>
      <Checkbox checked={selectedItems.includes(opt)} />
      <ListItemText primary={opt} />
    </MenuItem>
  ))}
</Select>
```

#### Checkbox, Radio, Switch

```tsx
import {
  Checkbox, Radio, RadioGroup, Switch,
  FormControlLabel, FormControl, FormLabel
} from '@mui/material';

{/* Checkbox */}
<FormControlLabel
  control={<Checkbox checked={checked} onChange={(e) => setChecked(e.target.checked)} />}
  label="Accept terms and conditions"
/>

{/* Radio group */}
<FormControl>
  <FormLabel>Gender</FormLabel>
  <RadioGroup value={gender} onChange={(e) => setGender(e.target.value)} row>
    <FormControlLabel value="female" control={<Radio />} label="Female" />
    <FormControlLabel value="male"   control={<Radio />} label="Male" />
    <FormControlLabel value="other"  control={<Radio />} label="Other" />
  </RadioGroup>
</FormControl>

{/* Switch */}
<FormControlLabel
  control={<Switch checked={darkMode} onChange={(e) => setDarkMode(e.target.checked)} />}
  label="Dark Mode"
/>
```

#### Autocomplete

Combo-box with filtering, free-solo input, async options, and multi-select.

```tsx
import { Autocomplete, TextField } from '@mui/material';

const options = ['Apple', 'Banana', 'Cherry', 'Date', 'Elderberry'];

{/* Basic */}
<Autocomplete
  options={options}
  renderInput={(params) => <TextField {...params} label="Fruit" />}
  onChange={(event, value) => setFruit(value)}
/>

{/* With objects */}
const movieOptions = [
  { title: 'The Shawshank Redemption', year: 1994 },
  { title: 'The Godfather', year: 1972 },
];

<Autocomplete
  options={movieOptions}
  getOptionLabel={(option) => option.title}
  renderInput={(params) => <TextField {...params} label="Movie" />}
  isOptionEqualToValue={(option, value) => option.title === value.title}
/>

{/* Multiple selection */}
<Autocomplete
  multiple
  options={options}
  renderInput={(params) => <TextField {...params} label="Tags" />}
  value={tags}
  onChange={(e, newValue) => setTags(newValue)}
/>

{/* Free-solo (user can type anything) */}
<Autocomplete
  freeSolo
  options={options}
  renderInput={(params) => <TextField {...params} label="Search" />}
/>
```

#### Slider

```tsx
import { Slider, Typography, Box } from '@mui/material';

{/* Basic */}
<Box sx={{ width: 300 }}>
  <Typography>Volume: {volume}</Typography>
  <Slider
    value={volume}
    onChange={(e, newValue) => setVolume(newValue as number)}
    min={0}
    max={100}
    step={1}
    marks
    valueLabelDisplay="auto"
    aria-label="Volume"
  />
</Box>

{/* Range slider */}
<Slider
  value={priceRange}
  onChange={(e, newValue) => setPriceRange(newValue as number[])}
  min={0}
  max={1000}
  step={10}
  valueLabelDisplay="on"
  disableSwap  // Prevents thumbs from crossing
/>
```

---

### Data Display

---

#### Card

Surface-level container for related content and actions.

```tsx
import {
  Card, CardContent, CardMedia, CardActions,
  CardHeader, Avatar, Typography, Button, IconButton
} from '@mui/material';
import FavoriteIcon from '@mui/icons-material/Favorite';
import ShareIcon from '@mui/icons-material/Share';

<Card sx={{ maxWidth: 345 }}>
  <CardHeader
    avatar={<Avatar sx={{ bgcolor: 'error.main' }}>R</Avatar>}
    title="Shrimp and Chorizo Paella"
    subheader="September 14, 2016"
  />
  <CardMedia
    component="img"
    height="194"
    image="/paella.jpg"
    alt="Paella dish"
  />
  <CardContent>
    <Typography variant="body2" color="text.secondary">
      This impressive paella is a perfect combination of rich saffron...
    </Typography>
  </CardContent>
  <CardActions disableSpacing>
    <IconButton aria-label="add to favorites">
      <FavoriteIcon />
    </IconButton>
    <IconButton aria-label="share">
      <ShareIcon />
    </IconButton>
    <Button size="small">Learn More</Button>
  </CardActions>
</Card>
```

#### Paper

Base surface component. Provides elevation (shadow) and background.

```tsx
import { Paper } from '@mui/material';

{/* elevation: 0–24 */}
<Paper elevation={3} sx={{ p: 3, maxWidth: 400, mx: 'auto' }}>
  <Typography variant="h6">Paper Surface</Typography>
  <Typography variant="body2">Content on a raised surface.</Typography>
</Paper>

{/* variant: 'elevation' | 'outlined' */}
<Paper variant="outlined" sx={{ p: 2 }}>
  Outlined paper (no shadow, has border)
</Paper>
```

#### Accordion

Expandable/collapsible content sections.

```tsx
import { Accordion, AccordionSummary, AccordionDetails, Typography } from '@mui/material';
import ExpandMoreIcon from '@mui/icons-material/ExpandMore';

{/* Controlled expansion */}
const [expanded, setExpanded] = useState<string | false>(false);
const handleChange = (panel: string) => (e: React.SyntheticEvent, isExpanded: boolean) => {
  setExpanded(isExpanded ? panel : false);
};

{['panel1', 'panel2', 'panel3'].map((panel, i) => (
  <Accordion key={panel} expanded={expanded === panel} onChange={handleChange(panel)}>
    <AccordionSummary expandIcon={<ExpandMoreIcon />} id={`${panel}-header`}>
      <Typography fontWeight={500}>Section {i + 1}</Typography>
    </AccordionSummary>
    <AccordionDetails>
      <Typography>Content for section {i + 1}</Typography>
    </AccordionDetails>
  </Accordion>
))}
```

#### List

```tsx
import {
  List, ListItem, ListItemButton, ListItemIcon,
  ListItemText, ListSubheader, Divider
} from '@mui/material';
import InboxIcon from '@mui/icons-material/Inbox';
import DraftsIcon from '@mui/icons-material/Drafts';

<List
  subheader={<ListSubheader>Mail</ListSubheader>}
  sx={{ width: '100%', maxWidth: 360, bgcolor: 'background.paper' }}
>
  <ListItem disablePadding>
    <ListItemButton selected onClick={() => setSelected('inbox')}>
      <ListItemIcon><InboxIcon /></ListItemIcon>
      <ListItemText primary="Inbox" secondary="42 unread" />
    </ListItemButton>
  </ListItem>
  <Divider />
  <ListItem disablePadding>
    <ListItemButton>
      <ListItemIcon><DraftsIcon /></ListItemIcon>
      <ListItemText primary="Drafts" />
    </ListItemButton>
  </ListItem>
</List>
```

#### Table

```tsx
import {
  Table, TableBody, TableCell, TableContainer,
  TableHead, TableRow, Paper
} from '@mui/material';

const rows = [
  { name: 'Frozen yoghurt', calories: 159, fat: 6 },
  { name: 'Ice cream sandwich', calories: 237, fat: 9 },
];

<TableContainer component={Paper}>
  <Table sx={{ minWidth: 650 }} aria-label="simple table">
    <TableHead>
      <TableRow>
        <TableCell>Dessert</TableCell>
        <TableCell align="right">Calories</TableCell>
        <TableCell align="right">Fat (g)</TableCell>
      </TableRow>
    </TableHead>
    <TableBody>
      {rows.map((row) => (
        <TableRow key={row.name} sx={{ '&:last-child td, &:last-child th': { border: 0 } }}>
          <TableCell component="th" scope="row">{row.name}</TableCell>
          <TableCell align="right">{row.calories}</TableCell>
          <TableCell align="right">{row.fat}</TableCell>
        </TableRow>
      ))}
    </TableBody>
  </Table>
</TableContainer>
```

#### Chip

Small labels for tags, categories, or filters.

```tsx
import { Chip, Stack } from '@mui/material';
import FaceIcon from '@mui/icons-material/Face';

<Stack direction="row" spacing={1}>
  <Chip label="Basic" />
  <Chip label="Primary" color="primary" />
  <Chip label="Outlined" variant="outlined" color="secondary" />
  <Chip label="Clickable" onClick={() => console.log('clicked')} />
  <Chip label="Deletable" onDelete={() => console.log('deleted')} />
  <Chip icon={<FaceIcon />} label="With Icon" color="success" />
</Stack>
```

#### Avatar & Badge

```tsx
import { Avatar, Badge, Stack } from '@mui/material';

{/* Avatar variants */}
<Stack direction="row" spacing={2}>
  <Avatar alt="Jane Doe" src="/jane.jpg" />                  {/* Image */}
  <Avatar sx={{ bgcolor: 'secondary.main' }}>JD</Avatar>     {/* Initials */}
  <Avatar variant="square"><PersonIcon /></Avatar>            {/* Icon */}
</Stack>

{/* Badge — adds a small indicator over a child element */}
<Badge badgeContent={4} color="error">
  <MailIcon />
</Badge>

<Badge variant="dot" color="success" overlap="circular">
  <Avatar src="/user.jpg" />
</Badge>
```

#### Tooltip

```tsx
import { Tooltip, Button } from '@mui/material';

<Tooltip title="This action cannot be undone" arrow placement="top">
  <Button color="error">Delete</Button>
</Tooltip>

{/* Tooltip on a disabled element — need a wrapper span */}
<Tooltip title="Log in to continue">
  <span>
    <Button disabled>Purchase</Button>
  </span>
</Tooltip>
```

---

### Navigation

---

#### AppBar & Toolbar

```tsx
import {
  AppBar, Toolbar, Typography, IconButton,
  Button, Box, Menu, MenuItem
} from '@mui/material';
import MenuIcon from '@mui/icons-material/Menu';

function NavBar() {
  return (
    <AppBar position="sticky">  {/* 'fixed' | 'sticky' | 'static' | 'relative' */}
      <Toolbar>
        {/* Hamburger menu for mobile */}
        <IconButton
          size="large"
          edge="start"
          color="inherit"
          aria-label="menu"
          sx={{ mr: 2, display: { md: 'none' } }}
          onClick={toggleDrawer}
        >
          <MenuIcon />
        </IconButton>

        {/* Logo / brand */}
        <Typography variant="h6" component="div" sx={{ flexGrow: 1 }}>
          My App
        </Typography>

        {/* Desktop nav links */}
        <Box sx={{ display: { xs: 'none', md: 'flex' }, gap: 1 }}>
          <Button color="inherit">Home</Button>
          <Button color="inherit">About</Button>
          <Button color="inherit">Contact</Button>
        </Box>
      </Toolbar>
    </AppBar>
  );
}
```

#### Drawer

Side panel for navigation or supplementary content.

```tsx
import { Drawer, Box, List, ListItem, ListItemButton, ListItemText } from '@mui/material';

const [open, setOpen] = useState(false);
const navItems = ['Home', 'About', 'Services', 'Contact'];

<>
  <Button onClick={() => setOpen(true)}>Open Menu</Button>

  <Drawer
    anchor="left"          // 'left' | 'right' | 'top' | 'bottom'
    open={open}
    onClose={() => setOpen(false)}
  >
    <Box sx={{ width: 250 }} role="presentation" onClick={() => setOpen(false)}>
      <List>
        {navItems.map((item) => (
          <ListItem key={item} disablePadding>
            <ListItemButton component="a" href={`/${item.toLowerCase()}`}>
              <ListItemText primary={item} />
            </ListItemButton>
          </ListItem>
        ))}
      </List>
    </Box>
  </Drawer>
</>

{/* Permanent drawer (sidebar layout) */}
<Drawer variant="permanent" sx={{ '& .MuiDrawer-paper': { width: 240 } }}>
  <Box sx={{ overflow: 'auto' }}>
    <List>{/* nav items */}</List>
  </Box>
</Drawer>
```

#### Tabs

```tsx
import { Tabs, Tab, Box } from '@mui/material';

const [tab, setTab] = useState(0);

<Box sx={{ borderBottom: 1, borderColor: 'divider' }}>
  <Tabs value={tab} onChange={(e, v) => setTab(v)} aria-label="main tabs">
    <Tab label="Overview" id="tab-0" aria-controls="tabpanel-0" />
    <Tab label="Settings" id="tab-1" aria-controls="tabpanel-1" />
    <Tab label="History"  id="tab-2" aria-controls="tabpanel-2" />
  </Tabs>
</Box>

{/* Tab panels */}
{[<Overview />, <Settings />, <History />].map((panel, i) => (
  <Box
    key={i}
    role="tabpanel"
    id={`tabpanel-${i}`}
    aria-labelledby={`tab-${i}`}
    hidden={tab !== i}
    sx={{ p: 3 }}
  >
    {tab === i && panel}
  </Box>
))}
```

#### Menu

Dropdown menu, typically anchored to a button.

```tsx
import { Menu, MenuItem, Button, Divider, ListItemIcon, ListItemText } from '@mui/material';
import ContentCopy from '@mui/icons-material/ContentCopy';
import Logout from '@mui/icons-material/Logout';

function UserMenu() {
  const [anchorEl, setAnchorEl] = useState<null | HTMLElement>(null);
  const open = Boolean(anchorEl);

  return (
    <>
      <Button onClick={(e) => setAnchorEl(e.currentTarget)} aria-haspopup="true">
        Account
      </Button>
      <Menu
        anchorEl={anchorEl}
        open={open}
        onClose={() => setAnchorEl(null)}
        transformOrigin={{ horizontal: 'right', vertical: 'top' }}
        anchorOrigin={{ horizontal: 'right', vertical: 'bottom' }}
      >
        <MenuItem onClick={() => { navigate('/profile'); setAnchorEl(null); }}>
          <ListItemText>Profile</ListItemText>
        </MenuItem>
        <MenuItem>
          <ListItemIcon><ContentCopy fontSize="small" /></ListItemIcon>
          <ListItemText>Copy link</ListItemText>
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

{/* Breadcrumbs */}
<Breadcrumbs aria-label="breadcrumb" separator="›">
  <Link underline="hover" color="inherit" href="/">Home</Link>
  <Link underline="hover" color="inherit" href="/products">Products</Link>
  <Typography color="text.primary">Sneakers</Typography>
</Breadcrumbs>

{/* Pagination */}
const [page, setPage] = useState(1);

<Pagination
  count={10}
  page={page}
  onChange={(e, value) => setPage(value)}
  color="primary"
  shape="rounded"
  showFirstButton
  showLastButton
/>
```

---

### Feedback & Overlays

---

#### Dialog

Modal overlay for confirmations, forms, and important messages.

```tsx
import {
  Dialog, DialogTitle, DialogContent, DialogContentText,
  DialogActions, Button
} from '@mui/material';

const [open, setOpen] = useState(false);

<>
  <Button variant="outlined" onClick={() => setOpen(true)}>
    Open Dialog
  </Button>

  <Dialog
    open={open}
    onClose={() => setOpen(false)}
    maxWidth="sm"   // 'xs' | 'sm' | 'md' | 'lg' | 'xl' | false
    fullWidth
    aria-labelledby="dialog-title"
  >
    <DialogTitle id="dialog-title">Confirm Delete</DialogTitle>
    <DialogContent>
      <DialogContentText>
        Are you sure you want to delete this item? This action cannot be undone.
      </DialogContentText>
    </DialogContent>
    <DialogActions>
      <Button onClick={() => setOpen(false)}>Cancel</Button>
      <Button onClick={handleDelete} color="error" variant="contained" autoFocus>
        Delete
      </Button>
    </DialogActions>
  </Dialog>
</>
```

#### Snackbar & Alert

Brief notifications that slide in and auto-dismiss.

```tsx
import { Snackbar, Alert, Button } from '@mui/material';

const [open, setOpen] = useState(false);

<>
  <Button onClick={() => setOpen(true)}>Show Notification</Button>

  <Snackbar
    open={open}
    autoHideDuration={4000}
    onClose={() => setOpen(false)}
    anchorOrigin={{ vertical: 'bottom', horizontal: 'center' }}
  >
    {/* Alert inside Snackbar for styled toasts */}
    <Alert
      onClose={() => setOpen(false)}
      severity="success"   // 'error' | 'warning' | 'info' | 'success'
      variant="filled"     // 'standard' | 'outlined' | 'filled'
      sx={{ width: '100%' }}
    >
      File saved successfully!
    </Alert>
  </Snackbar>
</>

{/* Alert standalone (inline message, not a toast) */}
<Alert severity="warning" sx={{ mb: 2 }}>
  Your session expires in 5 minutes.
</Alert>
```

#### Backdrop & CircularProgress

```tsx
import { Backdrop, CircularProgress, LinearProgress } from '@mui/material';

{/* Full-screen loading overlay */}
<Backdrop
  sx={{ color: '#fff', zIndex: (theme) => theme.zIndex.drawer + 1 }}
  open={isLoading}
>
  <CircularProgress color="inherit" />
</Backdrop>

{/* Circular spinner (inline) */}
<CircularProgress />
<CircularProgress color="secondary" size={60} thickness={5} />

{/* Determinate progress */}
<CircularProgress variant="determinate" value={progress} />

{/* Linear progress bar */}
<LinearProgress />                                     {/* Indeterminate */}
<LinearProgress variant="determinate" value={75} />   {/* 75% */}
<LinearProgress variant="buffer" value={60} valueBuffer={80} />
```

#### Skeleton

Placeholder loading state that matches the shape of expected content.

```tsx
import { Skeleton, Card, CardContent, Stack } from '@mui/material';

{/* Match content shape */}
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
  <ActualCard />
)}

{/* Animation: 'pulse' (default) | 'wave' | false */}
<Skeleton animation="wave" variant="rectangular" width={210} height={60} />
```

---

## 9. Icons — @mui/icons-material

### Installation

```bash
npm install @mui/icons-material
```

### Usage

Every icon is a React component. Import by name from the package.

```tsx
// Named imports (best for tree-shaking)
import DeleteIcon from '@mui/icons-material/Delete';
import EditIcon from '@mui/icons-material/Edit';
import SearchIcon from '@mui/icons-material/Search';
import HomeIcon from '@mui/icons-material/Home';
import CheckCircleIcon from '@mui/icons-material/CheckCircle';

// Or destructured (works but may pull more into bundle)
import { Delete, Edit, Search } from '@mui/icons-material';
```

### Icon Props

```tsx
{/* size: 'small' | 'medium' | 'large' | custom sx */}
<HomeIcon />                              {/* 24px default */}
<HomeIcon fontSize="small" />            {/* 20px */}
<HomeIcon fontSize="large" />            {/* 35px */}
<HomeIcon sx={{ fontSize: 48 }} />       {/* Custom size */}

{/* color: any palette color or inherit */}
<CheckCircleIcon color="success" />
<DeleteIcon color="error" />
<EditIcon color="action" />              {/* theme.palette.action.active */}
<SearchIcon sx={{ color: 'primary.main' }} />
```

### Finding Icon Names

Browse all 2,000+ icons at `https://mui.com/material-ui/material-icons/`. The component name is the icon name in PascalCase + "Icon" suffix (or without — both work).

```tsx
import HomeOutlinedIcon from '@mui/icons-material/HomeOutlined';  // Outlined variant
import HomeTwoToneIcon from '@mui/icons-material/HomeTwoTone';    // Two-tone variant
import HomeRoundedIcon from '@mui/icons-material/HomeRounded';    // Rounded variant
```

---

## 10. DataGrid — @mui/x-data-grid

The DataGrid is MUI's most powerful premium component. The community version (`@mui/x-data-grid`) is free with pagination, sorting, and filtering.

### Install

```bash
npm install @mui/x-data-grid
```

### Basic Usage

```tsx
import { DataGrid, GridColDef, GridValueGetterParams } from '@mui/x-data-grid';

// Column definitions
const columns: GridColDef[] = [
  { field: 'id', headerName: 'ID', width: 70 },
  { field: 'firstName', headerName: 'First name', width: 130 },
  { field: 'lastName', headerName: 'Last name', width: 130 },
  {
    field: 'fullName',
    headerName: 'Full name',
    width: 160,
    // Computed column — not in data
    valueGetter: (value, row) => `${row.firstName || ''} ${row.lastName || ''}`,
  },
  {
    field: 'age',
    headerName: 'Age',
    type: 'number',
    width: 90,
    editable: true,
  },
  {
    field: 'actions',
    headerName: 'Actions',
    width: 120,
    sortable: false,
    // Custom cell renderer
    renderCell: (params) => (
      <Button size="small" onClick={() => handleEdit(params.row)}>
        Edit
      </Button>
    ),
  },
];

// Row data
const rows = [
  { id: 1, lastName: 'Snow',    firstName: 'Jon',    age: 35 },
  { id: 2, lastName: 'Lannister', firstName: 'Cersei', age: 42 },
  { id: 3, lastName: 'Stark',   firstName: 'Arya',   age: 16 },
];

// The component
function UserTable() {
  const [selectionModel, setSelectionModel] = useState<number[]>([]);

  return (
    <Box sx={{ height: 400, width: '100%' }}>
      <DataGrid
        rows={rows}
        columns={columns}
        // Pagination
        initialState={{
          pagination: { paginationModel: { page: 0, pageSize: 10 } },
        }}
        pageSizeOptions={[5, 10, 25]}
        // Selection
        checkboxSelection
        onRowSelectionModelChange={(ids) => setSelectionModel(ids as number[])}
        // Filtering (built-in)
        disableColumnFilter={false}
        // Sorting
        sortingOrder={['asc', 'desc']}
        // Row click
        onRowClick={(params) => console.log(params.row)}
        // Styling
        sx={{
          '& .MuiDataGrid-columnHeaders': { bgcolor: 'primary.main', color: '#fff' },
        }}
      />
    </Box>
  );
}
```

### Server-Side Data

```tsx
<DataGrid
  rows={rows}
  columns={columns}
  rowCount={totalCount}      // Total rows from API
  paginationMode="server"    // Tell DataGrid not to paginate locally
  filterMode="server"
  sortingMode="server"
  onPaginationModelChange={(model) => {
    // Fetch new page from API
    fetchData({ page: model.page, pageSize: model.pageSize });
  }}
  onSortModelChange={(model) => {
    fetchData({ sortField: model[0]?.field, sortOrder: model[0]?.sort });
  }}
  loading={isFetching}
/>
```

---

## 11. Forms with MUI

### Controlled Input Pattern

Always use controlled inputs (value + onChange) with MUI form components.

```tsx
// Minimal login form — controlled
function LoginForm() {
  const [form, setForm] = useState({ email: '', password: '' });
  const [errors, setErrors] = useState<Record<string, string>>({});

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const { name, value } = e.target;
    setForm((prev) => ({ ...prev, [name]: value }));
  };

  const validate = () => {
    const errs: Record<string, string> = {};
    if (!form.email.includes('@')) errs.email = 'Enter a valid email';
    if (form.password.length < 8) errs.password = 'Min 8 characters';
    setErrors(errs);
    return Object.keys(errs).length === 0;
  };

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (validate()) {
      // submit logic
    }
  };

  return (
    <Box component="form" onSubmit={handleSubmit} noValidate sx={{ mt: 1 }}>
      <TextField
        name="email"
        label="Email Address"
        type="email"
        value={form.email}
        onChange={handleChange}
        error={!!errors.email}
        helperText={errors.email}
        fullWidth
        margin="normal"
        required
        autoFocus
      />
      <TextField
        name="password"
        label="Password"
        type="password"
        value={form.password}
        onChange={handleChange}
        error={!!errors.password}
        helperText={errors.password}
        fullWidth
        margin="normal"
        required
      />
      <Button type="submit" fullWidth variant="contained" sx={{ mt: 2, mb: 2 }}>
        Sign In
      </Button>
    </Box>
  );
}
```

### Integration with react-hook-form

```tsx
// Install: npm install react-hook-form zod @hookform/resolvers
import { useForm, Controller, SubmitHandler } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import {
  TextField, Button, Box, FormControlLabel,
  Checkbox, MenuItem, Select, FormControl, InputLabel
} from '@mui/material';

const schema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Min 8 characters'),
  role: z.string().min(1, 'Select a role'),
  agree: z.boolean().refine((v) => v, 'You must agree'),
});

type FormData = z.infer<typeof schema>;

function RegistrationForm() {
  const {
    register,
    control,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<FormData>({ resolver: zodResolver(schema) });

  const onSubmit: SubmitHandler<FormData> = async (data) => {
    await registerUser(data);
  };

  return (
    <Box component="form" onSubmit={handleSubmit(onSubmit)} sx={{ maxWidth: 400 }}>

      {/* TextField — use register() for simple inputs */}
      <TextField
        label="Email"
        fullWidth
        margin="normal"
        {...register('email')}
        error={!!errors.email}
        helperText={errors.email?.message}
      />

      <TextField
        label="Password"
        type="password"
        fullWidth
        margin="normal"
        {...register('password')}
        error={!!errors.password}
        helperText={errors.password?.message}
      />

      {/* Select — use Controller for components that don't use native onChange */}
      <Controller
        name="role"
        control={control}
        defaultValue=""
        render={({ field }) => (
          <FormControl fullWidth margin="normal" error={!!errors.role}>
            <InputLabel>Role</InputLabel>
            <Select {...field} label="Role">
              <MenuItem value="user">User</MenuItem>
              <MenuItem value="admin">Admin</MenuItem>
            </Select>
          </FormControl>
        )}
      />

      {/* Checkbox — always use Controller */}
      <Controller
        name="agree"
        control={control}
        defaultValue={false}
        render={({ field }) => (
          <FormControlLabel
            control={<Checkbox {...field} checked={field.value} />}
            label="I agree to the terms"
          />
        )}
      />

      <Button
        type="submit"
        fullWidth
        variant="contained"
        disabled={isSubmitting}
        sx={{ mt: 2 }}
      >
        {isSubmitting ? 'Registering...' : 'Register'}
      </Button>
    </Box>
  );
}
```

**Key rule:** Use `register()` for native HTML inputs (TextField, input, textarea). Use `Controller` (or `useController`) for custom MUI components that don't forward native HTML events (Select, Autocomplete, Checkbox, Slider, DatePicker).

---

## 12. Dark Mode & Theme Toggling

### Method 1: Manual Palette Mode Toggle (simplest)

```tsx
// providers.tsx
'use client';

import { createTheme, ThemeProvider, CssBaseline } from '@mui/material';
import { createContext, useContext, useState, useMemo, type ReactNode } from 'react';

// Context for toggling
const ColorModeContext = createContext({ toggle: () => {} });
export const useColorMode = () => useContext(ColorModeContext);

export function ColorModeProvider({ children }: { children: ReactNode }) {
  const [mode, setMode] = useState<'light' | 'dark'>('light');

  const colorMode = useMemo(() => ({
    toggle: () => setMode((prev) => (prev === 'light' ? 'dark' : 'light')),
  }), []);

  const theme = useMemo(() => createTheme({
    palette: { mode },
    // Other theme config...
  }), [mode]);

  return (
    <ColorModeContext.Provider value={colorMode}>
      <ThemeProvider theme={theme}>
        <CssBaseline />
        {children}
      </ThemeProvider>
    </ColorModeContext.Provider>
  );
}
```

```tsx
// DarkModeToggle.tsx
import { IconButton, Tooltip } from '@mui/material';
import DarkModeIcon from '@mui/icons-material/DarkMode';
import LightModeIcon from '@mui/icons-material/LightMode';
import { useTheme } from '@mui/material/styles';
import { useColorMode } from './providers';

export function DarkModeToggle() {
  const theme = useTheme();
  const { toggle } = useColorMode();

  return (
    <Tooltip title="Toggle dark mode">
      <IconButton onClick={toggle} color="inherit">
        {theme.palette.mode === 'dark' ? <LightModeIcon /> : <DarkModeIcon />}
      </IconButton>
    </Tooltip>
  );
}
```

### Method 2: useColorScheme (MUI v6+ CSS Variables)

⚡ **Version note:** MUI v6 introduced `colorSchemes` + `useColorScheme` when you opt into **CSS variables mode**. This is the recommended approach for v6+ as it avoids a flash of unstyled content (FOUC) on SSR.

```tsx
// theme.ts — enable CSS variables mode
import { createTheme } from '@mui/material/styles';

const theme = createTheme({
  colorSchemes: {
    light: {
      palette: {
        primary: { main: '#1976d2' },
      },
    },
    dark: {
      palette: {
        primary: { main: '#90caf9' },
        background: { default: '#121212', paper: '#1e1e1e' },
      },
    },
  },
});

export default theme;
```

```tsx
// DarkModeToggle.tsx — useColorScheme works without extra context
'use client';
import { useColorScheme, IconButton, Tooltip } from '@mui/material';
import DarkModeIcon from '@mui/icons-material/DarkMode';
import LightModeIcon from '@mui/icons-material/LightMode';

export function DarkModeToggle() {
  const { mode, setMode } = useColorScheme();

  return (
    <Tooltip title="Toggle dark mode">
      <IconButton
        onClick={() => setMode(mode === 'light' ? 'dark' : 'light')}
        color="inherit"
      >
        {mode === 'dark' ? <LightModeIcon /> : <DarkModeIcon />}
      </IconButton>
    </Tooltip>
  );
}
```

### Persisting Dark Mode to localStorage

```tsx
// In the manual toggle approach — persist to localStorage
const [mode, setMode] = useState<'light' | 'dark'>(() => {
  if (typeof window !== 'undefined') {
    return (localStorage.getItem('colorMode') as 'light' | 'dark') ?? 'light';
  }
  return 'light';
});

const colorMode = useMemo(() => ({
  toggle: () => setMode((prev) => {
    const next = prev === 'light' ? 'dark' : 'light';
    localStorage.setItem('colorMode', next);
    return next;
  }),
}), []);
```

---

## 13. Responsive Design

MUI's default breakpoints (mobile-first):

| Key | Min width | Typical use |
|---|---|---|
| `xs` | 0px | Mobile phones |
| `sm` | 600px | Tablets |
| `md` | 900px | Small laptops |
| `lg` | 1200px | Desktops |
| `xl` | 1536px | Large screens |

### Breakpoints in sx

```tsx
// sx responsive values
<Box
  sx={{
    // Mobile: column, Desktop: row
    display: 'flex',
    flexDirection: { xs: 'column', md: 'row' },
    gap: { xs: 1, md: 3 },

    // Show/hide per breakpoint
    display: { xs: 'none', md: 'block' }, // Hide on mobile
    display: { xs: 'block', md: 'none' }, // Show only on mobile

    // Responsive font size
    fontSize: { xs: '1rem', sm: '1.25rem', lg: '1.5rem' },

    // Responsive padding
    px: { xs: 2, sm: 4, lg: 8 },
  }}
>
```

### Breakpoints in styled()

```tsx
const ResponsiveBox = styled(Box)(({ theme }) => ({
  padding: theme.spacing(2),

  [theme.breakpoints.up('md')]: {
    padding: theme.spacing(4),
    display: 'flex',
  },

  [theme.breakpoints.between('sm', 'lg')]: {
    backgroundColor: theme.palette.grey[100],
  },

  [theme.breakpoints.down('sm')]: {
    // Mobile-specific styles
    flexDirection: 'column',
  },
}));
```

### useMediaQuery

```tsx
import { useMediaQuery, useTheme } from '@mui/material';

function AdaptiveComponent() {
  const theme = useTheme();

  // Returns true if viewport >= md (900px)
  const isDesktop = useMediaQuery(theme.breakpoints.up('md'));

  // Boolean — re-renders when breakpoint changes
  const isMobile = useMediaQuery(theme.breakpoints.down('sm'));

  // Custom query
  const isLandscape = useMediaQuery('(orientation: landscape)');

  // Prefers dark mode (system preference)
  const prefersDark = useMediaQuery('(prefers-color-scheme: dark)');

  return (
    <div>
      {isDesktop ? <DesktopLayout /> : <MobileLayout />}
    </div>
  );
}
```

### SSR + useMediaQuery (Next.js)

```tsx
// On the server, screen size is unknown — useMediaQuery returns false by default.
// Use initializeMode for SSR if you need server-rendered responsive layouts.
const isMobile = useMediaQuery(theme.breakpoints.down('md'), {
  defaultMatches: true, // Assume mobile until hydration
  noSsr: false,         // Allow SSR behavior
});
```

---

## 14. Performance, Tips, Tricks & Gotchas

### Bundle Size — Tree-Shaking

```tsx
// GOOD — named path imports (tree-shaked by bundler)
import Button from '@mui/material/Button';
import TextField from '@mui/material/TextField';
import Grid from '@mui/material/Grid2';

// ALSO GOOD — named imports from main (modern bundlers handle this fine)
import { Button, TextField } from '@mui/material';

// AVOID for icons — always use direct path imports
// BAD:
import { Delete, Edit, Home } from '@mui/icons-material'; // pulls in ALL icons
// GOOD:
import DeleteIcon from '@mui/icons-material/Delete';
import EditIcon from '@mui/icons-material/Edit';
```

### sx vs styled() — Performance

```tsx
// sx creates a new style object on every render — use for one-off/dynamic styles
// styled() creates the style ONCE at component definition time — use for reusable components

// Bad — creates a new class on every render due to inline object
// (But still cached by Emotion — not as bad as it sounds for static values)
<Box sx={{ display: 'flex', p: 2 }}>...</Box>

// Better — for components rendered many times (e.g. in a list):
const FlexBox = styled(Box)({ display: 'flex', padding: 16 });
// Then use: <FlexBox>...</FlexBox>
```

### "use client" in Next.js App Router

```tsx
// These MUI components REQUIRE 'use client' in Next.js App Router:
// - Any component that uses hooks: useState, useEffect, useContext, useRef
// - useTheme, useMediaQuery, useColorScheme
// - Dialog, Drawer, Menu (useState-based open/close)
// - Snackbar, Tooltip (event handlers)
// - DataGrid (complex interaction)
// - react-hook-form + Controller

// Pattern: Create thin 'use client' wrapper components
// Server: renders static content
// Client: renders interactive islands

'use client'; // Add this to any file that imports interactive MUI components
```

### Hydration Mismatches

```tsx
// Problem: useMediaQuery returns false on server, true on client → mismatch
// Solution 1: defaultMatches option
const isMobile = useMediaQuery(theme.breakpoints.down('md'), { defaultMatches: true });

// Solution 2: Render after mount
const [mounted, setMounted] = useState(false);
useEffect(() => setMounted(true), []);
if (!mounted) return <Skeleton />; // Or null
return <ActualComponent />;
```

### Emotion SSR — Missing Styles

```tsx
// If you see unstyled flash on first load (SSR), ensure:
// 1. AppRouterCacheProvider wraps everything (Next.js App Router)
// 2. getInitialProps / _document.tsx setup (Next.js Pages Router)
// 3. CssBaseline is inside ThemeProvider

// Next.js Pages Router setup (_document.tsx)
import createEmotionServer from '@emotion/server/create-instance';
import createEmotionCache from '../lib/createEmotionCache';

// This is more involved — see official MUI Next.js Pages Router example
```

### zIndex Management

```tsx
// MUI manages zIndex through the theme.zIndex object
// Drawer: 1200, AppBar: 1100, Modal: 1300, Snackbar: 1400, Tooltip: 1500

// Reference in sx:
<Box sx={{ zIndex: (theme) => theme.zIndex.modal + 1 }}>
  Above the modal
</Box>
```

### Avoiding Double Scrollbars

```tsx
// CssBaseline sets html and body to height: 100%, which can cause issues.
// Override if needed:
<Box
  component="main"
  sx={{
    flexGrow: 1,
    overflow: 'auto',
    height: '100vh',
  }}
>
```

### Common Mistakes

| Mistake | Fix |
|---|---|
| Importing from `@mui/icons-material` barrel | Use direct path: `@mui/icons-material/IconName` |
| `ThemeProvider` outside `AppRouterCacheProvider` | `AppRouterCacheProvider` must be the outermost wrapper |
| Using `<Grid item xs={12}>` in v6 | Use `<Grid size={12}>` — no `item` prop in v6 |
| `TextField` with no `label` and no `aria-label` | Always provide accessible label |
| `Dialog` z-index behind custom elements | Use `theme.zIndex.modal` reference |
| `useColorScheme` called without `colorSchemes` in createTheme | Add `colorSchemes` key or use manual mode approach |
| Wrapping `Tooltip` around a disabled `Button` | Add `<span>` wrapper — disabled elements block pointer events |
| Forgetting `key` on mapped `MenuItem` | Always key dynamic list items |

### Tips

```tsx
// Tip 1: sx accepts arrays for merging styles (useful in component APIs)
<Box sx={[{ p: 2 }, isActive && { bgcolor: 'primary.light' }]}>
  Content
</Box>

// Tip 2: Clone theme and extend it (nested ThemeProvider)
import { createTheme, useTheme } from '@mui/material/styles';
const outerTheme = useTheme();
const innerTheme = createTheme(outerTheme, {
  palette: { primary: { main: '#ff0000' } },
});
<ThemeProvider theme={innerTheme}>...</ThemeProvider>

// Tip 3: Use 'component' prop to render MUI components as router Links
import { Link as RouterLink } from 'react-router-dom';
<Button component={RouterLink} to="/about">About</Button>
<ListItemButton component={RouterLink} to="/profile">Profile</ListItemButton>

// Tip 4: MUI's sx prop works on any element via the Box component
// Use Box as a layout div instead of unstyled divs
<Box component="section" sx={{ ... }}>...</Box>
<Box component="span" sx={{ ... }}>...</Box>
<Box component="article" sx={{ ... }}>...</Box>

// Tip 5: Avoid re-creating theme on every render
// createTheme is expensive — define it OUTSIDE the component tree
const theme = createTheme({ ... }); // module scope ✓
function App() {
  const theme = createTheme({ ... }); // BAD — recreates every render ✗
}
```

---

## 15. Study Path

Follow this sequence to build real intuition, not just memorize the API.

### Phase 1 — Foundation (Week 1)

1. **Install + setup:** Create a Vite React project with MUI + Emotion. Wire up `ThemeProvider` and `CssBaseline`.
2. **Learn `sx`:** Build a profile card using only `Box`, `Typography`, `Avatar`, and `Button` with `sx` for all styling.
3. **Theme basics:** Customize `primary.main`, `typography.fontFamily`, and `shape.borderRadius`. See it change everywhere.

**Build:** A personal profile card component.

### Phase 2 — Layout Mastery (Week 2)

4. **Box + Flexbox:** Build a page layout (top nav, sidebar, main content, footer) using only `Box`.
5. **Stack:** Refactor all "vertical spaced elements" to use `Stack spacing`.
6. **Grid v6:** Build a responsive image gallery — 1 column on mobile, 2 on tablet, 4 on desktop.
7. **Container:** Wrap page content in `Container` with `maxWidth="lg"`.

**Build:** A fully responsive landing page.

### Phase 3 — Forms (Week 3)

8. **Controlled inputs:** Build a sign-up form with `TextField`, `Select`, `Checkbox`, `Switch`.
9. **Validation:** Add manual validation with error states and helper text.
10. **react-hook-form + zod:** Refactor the form to use `react-hook-form` with `Controller` for non-native components.

**Build:** A multi-step registration form.

### Phase 4 — Navigation & Patterns (Week 4)

11. **AppBar + Drawer:** Build a responsive navigation — hamburger menu on mobile, horizontal nav on desktop.
12. **Tabs:** Add a tabbed content section.
13. **Dialog:** Add a confirmation dialog for a destructive action.
14. **Snackbar + Alert:** Add toast notifications for success and error states.

**Build:** A full admin dashboard shell (nav + sidebar + content area + toasts).

### Phase 5 — Data & Advanced (Week 5)

15. **DataGrid:** Fetch a list of users from a fake API and display in a DataGrid with sorting, filtering, and pagination.
16. **Dark mode:** Implement a dark/light toggle using `colorSchemes` + `useColorScheme`.
17. **Theme overrides:** Globally restyle `Button`, `TextField`, and `Card` via `components.styleOverrides`.
18. **Next.js App Router:** Migrate or rebuild the dashboard in Next.js with `AppRouterCacheProvider` + client provider pattern.

**Build:** A full CRUD user management dashboard.

### Phase 6 — Polish (Ongoing)

- Read the official MUI docs for every component you use (they have excellent prop tables and demos).
- Explore the MUI component playground at `https://mui.com/material-ui/`.
- Study MUI's free templates for real-world composition patterns.
- Practice theming: build the same layout with 3 different brand color palettes.

---

### Quick Reference Card

| Task | Code |
|---|---|
| Install | `npm install @mui/material @emotion/react @emotion/styled` |
| Theme setup | `createTheme({})` → `<ThemeProvider theme={theme}>` |
| CSS reset | `<CssBaseline />` inside `<ThemeProvider>` |
| Spacing unit | `theme.spacing(1)` = 8px (default) |
| sx shorthand px | `sx={{ px: 2 }}` = `padding-left/right: 16px` |
| Responsive sx | `sx={{ fontSize: { xs: '1rem', md: '1.5rem' } }}` |
| Grid v6 | `<Grid size={{ xs: 12, md: 6 }}>` |
| Dark mode | `createTheme({ colorSchemes: { light: {}, dark: {} } })` + `useColorScheme()` |
| useMediaQuery | `useMediaQuery(theme.breakpoints.up('md'))` |
| Global override | `createTheme({ components: { MuiButton: { styleOverrides: { root: {} } } } })` |
| Icons import | `import DeleteIcon from '@mui/icons-material/Delete'` |
| Router link | `<Button component={RouterLink} to="/path">` |
| Next.js SSR | `<AppRouterCacheProvider>` → `<Providers>` → children |
