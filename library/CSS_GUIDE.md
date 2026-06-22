# CSS & CSS3 — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I've seen `color: red` once" to "I architect responsive, accessible, maintainable stylesheets using modern layout and the modern cascade" — without an internet connection. Every concept is explained in prose first (what it is, *why* it behaves the way it does, when to reach for it, how to use it), then shown with heavily-commented, runnable code. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **modern CSS as of 2026** — i.e. the features that are now *baseline*, widely supported across all current evergreen browsers (Chrome, Edge, Firefox, Safari). The things that used to be "cutting edge" and are now safe to use everywhere:
> - **Native CSS nesting** — write nested rules without Sass.
> - **Container queries** (`@container`) — style a component based on *its own* size, not the viewport.
> - **`:has()`** — the long-awaited "parent selector" and powerful relational selector.
> - **Cascade layers** (`@layer`) — explicit control over the order of the cascade, ending specificity wars.
> - **Subgrid** — grid items that align to their parent grid's tracks.
> - **`oklch()` / `oklab()`** — perceptually-uniform color spaces, now the recommended way to define colors.
> - **Logical properties** (`margin-inline`, `padding-block`, `inset`…) — writing-mode/internationalization aware.
> - **`aspect-ratio`**, **`gap` on flexbox**, **`clamp()`** for fluid type, **scroll-snap**, **`:is()`/`:where()`**.
>
> This guide is about **real, hand-written CSS** — not a framework. There is already a `TAILWIND_CHEATSHEET.md` in this library for the utility-class approach; §16 explains how the two relate. Where a feature is newer or fast-moving it is flagged with **⚡ Version note**. The author is on **Windows 11**; browser DevTools shortcuts are noted where relevant. Confirm exact support at MDN (developer.mozilla.org) or caniuse.com.

---

## Table of Contents

1. [What CSS Is & How It's Applied](#1-what-css-is--how-its-applied) **[B]**
2. [Selectors In Depth](#2-selectors-in-depth) **[B/I]**
3. [The Cascade, Specificity & Inheritance](#3-the-cascade-specificity--inheritance) **[I]**
4. [The Box Model](#4-the-box-model) **[B/I]**
5. [Units & Values](#5-units--values) **[B/I]**
6. [Colors](#6-colors) **[B/I]**
7. [Typography](#7-typography) **[B/I]**
8. [Backgrounds, Borders & Shadows](#8-backgrounds-borders--shadows) **[B/I]**
9. [Display Models & Normal Flow](#9-display-models--normal-flow) **[I]**
10. [Flexbox](#10-flexbox) **[I]**
11. [CSS Grid](#11-css-grid) **[I/A]**
12. [Positioning, z-index & Stacking](#12-positioning-z-index--stacking) **[I]**
13. [Responsive Design & Container Queries](#13-responsive-design--container-queries) **[I/A]**
14. [Transitions, Animations & Transforms](#14-transitions-animations--transforms) **[I/A]**
15. [Modern CSS Features](#15-modern-css-features) **[I/A]**
16. [Architecture & Methodology](#16-architecture--methodology) **[I/A]**
17. [Accessibility & Best Practices](#17-accessibility--best-practices) **[I]**
18. [Gotchas](#18-gotchas) **[I/A]**
19. [Study Path & Build-to-Learn Projects](#19-study-path--build-to-learn-projects)

---

## 1. What CSS Is & How It's Applied

### 1.1 What CSS actually is **[B]**

**CSS** stands for **Cascading Style Sheets**. HTML describes the *structure and meaning* of a document (this is a heading, this is a paragraph, this is a list). CSS describes its *presentation* — color, size, spacing, layout, animation. The browser combines the two: it parses HTML into a tree of elements (the **DOM**), parses CSS into a set of rules, then for every element works out which rules apply and what the final value of every style property should be. That final, fully-resolved set of styles is the **computed style**, and it's what gets painted to the screen.

The word **"cascading"** is the heart of it: many rules from many sources can target the same element, and CSS has a precise, deterministic algorithm for deciding which one wins. Most of the confusion beginners feel about CSS is really confusion about the cascade — which is why §3 is the most important section in this guide.

CSS is **declarative**: you state *what* you want ("paragraphs should be dark gray with 1.6 line-height"), not *how* to compute it step by step. The browser's layout and paint engines do the rest.

### 1.2 The anatomy of a rule **[B]**

A stylesheet is a list of **rules**. Each rule has a **selector** (which elements it targets) and a **declaration block** (what to do to them). Each declaration is a **property** and a **value**, separated by a colon and terminated by a semicolon.

```css
/* This whole thing is a RULE.                                              */
/* `p` is the SELECTOR — it matches every <p> element.                      */
p {
  /* Each line below is a DECLARATION. */
  color: #333333;        /* property: color   value: #333333 (a hex color) */
  font-size: 16px;       /* property: font-size  value: 16px               */
  line-height: 1.6;      /* the semicolon ends each declaration             */
}                        /* the braces { } enclose the declaration BLOCK    */

/* Comments use /* ... */ — there is NO // line-comment in CSS.            */
```

Key syntax facts that trip people up:

- The **semicolon separates** declarations. The last one's semicolon is optional but *always include it* — adding a new line later without it is a classic bug.
- Whitespace and line breaks are insignificant — `p{color:red}` works, but don't write that.
- CSS is **forgiving / fault-tolerant**: if the browser hits a property it doesn't understand or a value it can't parse, it *skips that one declaration* and keeps going. It does not throw an error or abandon the rule. This is why you can write fallbacks (an old value then a new value); the browser keeps the last one it understands.

### 1.3 The three ways to apply CSS **[B]**

There are exactly three places CSS can live, and you should overwhelmingly prefer the last one.

**1. Inline styles** — a `style` attribute directly on an element. Highest specificity, impossible to reuse, can't use media queries or pseudo-classes well. Avoid except for one-off dynamic styles set by JavaScript.

```html
<!-- Inline: styles ONLY this one element. Hard to maintain, very "sticky"
     in the cascade because inline beats almost everything (see §3). -->
<p style="color: red; font-size: 20px;">Urgent notice</p>
```

**2. Internal / embedded styles** — a `<style>` block in the HTML `<head>`. Good for a single-page demo, an HTML email, or critical above-the-fold CSS. It only styles that one document.

```html
<head>
  <style>
    /* Applies to the whole document, but lives inside this one HTML file. */
    body { font-family: system-ui, sans-serif; }
    h1   { color: navy; }
  </style>
</head>
```

**3. External stylesheet** — a separate `.css` file linked from the `<head>`. **This is the right way** for real projects: one stylesheet can style many pages, the browser caches it, and your concerns are cleanly separated.

```html
<head>
  <!-- rel="stylesheet" tells the browser this link is CSS. href is the path. -->
  <link rel="stylesheet" href="/css/styles.css" />
</head>
```

```css
/* styles.css — an external file. Same syntax as inside <style>. */
body { margin: 0; font-family: system-ui, sans-serif; }
```

You can also import one stylesheet from another with `@import "other.css";`, but `@import` blocks rendering and is slower than multiple `<link>` tags — prefer `<link>` (or a bundler) for production. `@import` does have a legitimate modern use with cascade layers (see §3.6).

### 1.4 How the browser actually applies styles **[B/I]**

Understanding the pipeline demystifies a lot of "why isn't this working" moments:

1. **Parse HTML → DOM tree.** Every element becomes a node.
2. **Parse CSS → CSSOM** (CSS Object Model) — the rules in a structured form.
3. **Style / cascade:** for each element, the browser gathers every declaration that matches it, resolves conflicts via the cascade (§3), inherits values where appropriate, and produces the element's **computed style**.
4. **Layout (a.k.a. reflow):** using box model + display + positioning, the browser computes the size and position of every box.
5. **Paint:** fill in pixels — colors, text, images, borders, shadows.
6. **Composite:** the browser may put some things on separate GPU layers and combine them (this is why `transform`/`opacity` animations are cheap — §14.6).

The practical takeaway: changing a property that affects size/position triggers **layout** (expensive). Changing only paint properties (e.g. `color`) triggers **paint** (cheaper). Changing `transform`/`opacity` can often be handled by the **compositor** alone (cheapest). This hierarchy governs animation performance.

> **DevTools tip:** In any browser press **F12**, pick an element in the *Elements/Inspector* panel, and look at the *Styles* and *Computed* tabs. The Styles tab shows every matching rule with overridden ones struck through — it is the single best tool for understanding the cascade in practice.

---

## 2. Selectors In Depth

A **selector** is the pattern that decides which elements a rule applies to. Mastering selectors is half of being good at CSS — they let you target exactly what you want without adding classes everywhere.

### 2.1 The basic selectors **[B]**

```css
/* TYPE (element) selector — matches by tag name. Low specificity. */
p        { line-height: 1.6; }
h1       { font-size: 2rem; }

/* CLASS selector — matches elements with class="card". The workhorse of
   real CSS: reusable, low-but-not-too-low specificity. Starts with a dot. */
.card    { border: 1px solid #ddd; }

/* ID selector — matches the ONE element with id="header". Starts with #.
   High specificity (see §3) — avoid for styling; reserve IDs for anchors/JS. */
#header  { position: sticky; top: 0; }

/* UNIVERSAL selector — matches every element. Use sparingly. */
*        { box-sizing: border-box; }   /* the famous global reset (see §4) */
```

An element can carry many classes (`class="card card--featured is-active"`), and you target combinations by chaining with no space:

```css
/* Matches an element that has BOTH .card AND .is-active.
   No space between them = "same element". */
.card.is-active { border-color: dodgerblue; }

/* Combine type + class: a <button> that also has class="primary". */
button.primary { background: royalblue; color: white; }
```

### 2.2 Attribute selectors **[I]**

Match on the presence or value of an HTML attribute. Great for styling form inputs by type, links by destination, or elements by data-attributes.

```css
[disabled]            { opacity: 0.5; }              /* has the attribute at all */
[type="email"]        { /* exactly equals */ }
input[type="checkbox"]{ /* type + attribute combined */ }

[href^="https://"]    { /* value STARTS WITH (^=) — external links */ }
[href$=".pdf"]        { /* value ENDS WITH ($=) — PDF links */ }
[href*="example"]     { /* value CONTAINS substring (*=)  */ }
[class~="card"]       { /* contains this word in a space-separated list (~=) */ }
[lang|="en"]          { /* equals "en" or starts with "en-" (|=) — for langs */ }

/* Case-insensitive match: add `i` before the closing bracket. */
[type="EMAIL" i]      { /* matches type="email", "EMAIL", "Email"… */ }
```

### 2.3 Grouping **[B]**

A comma means "apply this rule to all of these selectors." It's pure deduplication — each selector is matched independently.

```css
/* Same styles for h1, h2 and h3. The comma reads as "or". */
h1, h2, h3 {
  font-family: Georgia, serif;
  margin-block: 0.5em;   /* top & bottom margin (logical property, §15.4) */
}
```

> **Gotcha:** an invalid selector inside a comma list used to invalidate the *whole* group in older CSS. With `:is()`/`:where()` (§2.7) that's no longer a problem — they're "forgiving."

### 2.4 Combinators — relationships between elements **[I]**

Combinators express how two elements relate in the tree. This is where selectors get powerful.

```css
/* DESCENDANT combinator = a SPACE.
   Matches <a> anywhere inside .nav, at any depth. */
.nav a { color: white; }

/* CHILD combinator = >  .
   Matches <li> that is a DIRECT child of <ul> (not grandchildren). */
ul > li { list-style: square; }

/* NEXT-SIBLING (adjacent) combinator = +  .
   Matches a <p> that immediately FOLLOWS an <h2> (same parent). */
h2 + p { margin-top: 0; }            /* tighten the gap after a heading */

/* SUBSEQUENT-SIBLING (general) combinator = ~ .
   Matches every <p> that comes after an <h2> and shares its parent. */
h2 ~ p { color: #555; }
```

Mental model for the tree:

```html
<ul>                    <!-- parent -->
  <li>A                 <!-- child of ul; descendant of ul -->
    <span>x</span>      <!-- descendant of ul, but NOT a child -->
  </li>
  <li>B</li>            <!-- sibling of the first <li> -->
</ul>
```

`ul > li` matches A and B. `ul li` (descendant) matches A, B, *and* the span's `<li>` ancestors' contents — anything nested. `li + li` matches B (the li immediately after another li).

### 2.5 Pseudo-classes — state & position **[I]**

A **pseudo-class** (single colon `:`) targets an element in a particular *state* or *structural position* — something that isn't expressed by a class in the HTML. The browser keeps these up to date as the user interacts.

```css
/* USER-INTERACTION states */
a:hover            { text-decoration: underline; }  /* pointer over it */
a:active           { color: red; }                  /* being clicked/pressed */
button:focus       { outline: 2px solid royalblue; }/* keyboard/programmatic focus */
button:focus-visible { outline: 2px solid royalblue; } /* focus, but only when a
                       focus ring is appropriate (keyboard, not mouse) — prefer this */
input:disabled     { background: #eee; }
input:checked      { /* checkbox/radio is checked */ }
input:required     { /* has the required attribute */ }
input:invalid      { border-color: crimson; }       /* fails validation */
input:placeholder-shown { /* the placeholder is currently visible (empty) */ }

/* LINK states — order matters: LoVe HAte = :link :visited :hover :active */
a:link    { color: blue; }    /* unvisited */
a:visited { color: purple; }  /* visited (privacy-limited which props work) */

/* STRUCTURAL / positional pseudo-classes */
li:first-child     { font-weight: bold; }  /* first among its siblings */
li:last-child      { border-bottom: none; }
li:only-child      { /* the sole child */ }
p:first-of-type    { /* first <p> among siblings (ignores other types) */ }
tr:nth-child(even) { background: #f7f7f7; } /* zebra striping */
tr:nth-child(odd)  { background: white; }
li:nth-child(3)    { /* the 3rd child */ }
li:nth-child(3n)   { /* every 3rd (3,6,9…) */ }
li:nth-child(3n+1) { /* 1st, 4th, 7th… (an+b formula) */ }
li:nth-last-child(2){ /* 2nd from the END */ }

/* NEGATION */
li:not(.special)   { opacity: 0.8; }        /* every li WITHOUT .special */
input:not([type="hidden"]) { /* … */ }

/* :empty — element with no children/text at all */
p:empty { display: none; }

/* :root — the <html> element; the canonical place for global custom properties */
:root { --brand: #4f46e5; }
```

**How `nth-child` math works:** the formula is `an+b`. The browser tries `n = 0, 1, 2, 3…` and the result is the 1-based position to match. `2n+1` → positions 1,3,5… (odd). `3n` → 3,6,9. Keywords `even`/`odd` are shortcuts for `2n`/`2n+1`.

> **Gotcha — `:nth-child` vs `:nth-of-type`:** `p:nth-child(2)` means "an element that is the 2nd child of its parent **and** happens to be a `<p>`" — if the 2nd child is a `<div>`, nothing matches. `p:nth-of-type(2)` means "the 2nd `<p>` among its siblings," counting only `<p>` elements. Use `of-type` when the markup mixes element types.

### 2.6 Pseudo-elements — style part of an element **[I]**

A **pseudo-element** (double colon `::`) lets you style a *part* of an element, or insert generated content, that isn't a real DOM node.

```css
/* ::before and ::after insert generated content. They REQUIRE the `content`
   property to appear at all (even an empty string). They sit inside the
   element, before/after its real content. */
.note::before {
  content: "💡 ";          /* generated text */
  font-weight: bold;
}
.tooltip::after {
  content: "";              /* empty but present — used as a decorative shape */
  position: absolute;
  border: 6px solid transparent;
  border-top-color: black;  /* a little CSS triangle */
}

/* Style the first line / first letter of a block of text */
p::first-line   { font-variant: small-caps; }
p::first-letter { font-size: 3em; float: left; }  /* drop cap */

/* Style the user's text selection */
::selection     { background: gold; color: black; }

/* Style a form control's placeholder text */
input::placeholder { color: #999; font-style: italic; }

/* Style list markers (modern) */
li::marker      { color: tomato; }
```

> **Why two colons?** Original CSS used a single colon for both (`:before`). CSS3 introduced `::` to *distinguish pseudo-elements from pseudo-classes*. Browsers still accept the old single-colon form for the original four (`:before`, `:after`, `:first-line`, `:first-letter`) for backwards compatibility, but write `::` in new code.

### 2.7 The modern relational & logical selectors **[I/A]**

These changed how CSS can be written. They are baseline-supported in 2026.

**`:is()`** — takes a selector list and matches if any of them match. It *deduplicates* long selectors and takes on the **highest specificity** of its arguments.

```css
/* WITHOUT :is() — repetitive */
section h1, article h1, aside h1 { font-size: 1.5rem; }
/* WITH :is() — concise, equivalent */
:is(section, article, aside) h1 { font-size: 1.5rem; }
```

**`:where()`** — identical to `:is()` *except its specificity is always zero*. This is enormously useful for writing base/reset styles that are trivially easy to override.

```css
/* These reset styles add ZERO specificity, so any later .class easily wins. */
:where(ul, ol) { margin: 0; padding: 0; list-style: none; }
```

**`:has()` — the relational pseudo-class, a.k.a. the "parent selector."** For decades CSS could only style *downward*. `:has()` lets a selector match based on what an element *contains* or what *follows* it. This is one of the most powerful additions to the language.

```css
/* A .card that CONTAINS an <img> — style the container based on its content. */
.card:has(img) { padding: 0; }

/* A form label whose associated input is invalid (sibling relationship). */
label:has(+ input:invalid) { color: crimson; }

/* A <figure> that has a <figcaption> gets a border. */
figure:has(figcaption) { border: 1px solid #ddd; }

/* "Quantity query": a grid that has MORE than 4 children switches layout.
   :has() + :nth-last-child makes this possible with pure CSS. */
.grid:has(> :nth-child(5)) { /* there are at least 5 children */ }

/* Combine with :not() for "doesn't contain": */
.section:not(:has(h2)) { /* a section with no <h2> inside */ }
```

> **⚡ Version note:** `:has()` became baseline across all evergreen browsers during 2023–2024 and is fully safe in 2026. Because it's *forgiving* like `:is()`, an unknown selector inside it won't break the rule. Be mindful of performance with extremely broad `:has()` on huge documents, but for normal component-sized usage it's fine.

### 2.8 Selector reference table

| Selector | Matches | Example |
|---|---|---|
| `*` | every element | `* { box-sizing: border-box; }` |
| `E` | type/tag | `p`, `h1`, `button` |
| `.cls` | class | `.card` |
| `#id` | id | `#main` |
| `[attr]` / `[attr=v]` | attribute | `[disabled]`, `[type="text"]` |
| `[attr^=v]` `$=` `*=` | starts/ends/contains | `[href^="https"]` |
| `A B` | descendant | `.nav a` |
| `A > B` | direct child | `ul > li` |
| `A + B` | next sibling | `h2 + p` |
| `A ~ B` | following siblings | `h2 ~ p` |
| `:hover :focus :active` | interaction state | `a:hover` |
| `:nth-child(an+b)` | positional | `li:nth-child(2n)` |
| `:not(s)` | negation | `li:not(.x)` |
| `:is(a, b)` | any of (max specificity) | `:is(h1,h2) a` |
| `:where(a, b)` | any of (zero specificity) | `:where(h1,h2)` |
| `:has(s)` | relational / parent | `.card:has(img)` |
| `::before ::after` | generated content | `.x::before { content:"" }` |
| `::first-line ::marker ::selection` | element part | `li::marker` |

---

## 3. The Cascade, Specificity & Inheritance

This is the single most misunderstood part of CSS. When your style "isn't applying," 90% of the time it *is* applying — it's just being **overridden** by another rule that wins the cascade. Once you understand the algorithm, CSS stops feeling random.

When more than one declaration sets the same property on the same element, the browser picks a winner using these criteria **in order**. Each step is a tiebreaker for the previous:

1. **Origin & importance** (where the rule came from + `!important`).
2. **Cascade layers** (`@layer`) — newer, sits inside the origin/importance step.
3. **Specificity** — how specific the selector is.
4. **Source order** — last one wins.

### 3.1 Origin and importance **[I]**

Styles come from three *origins*: the **user agent** (the browser's built-in defaults — why an `<h1>` is big and bold before you write any CSS), the **user** (custom stylesheets a person sets, rare), and the **author** (your stylesheet). Normally author beats user beats user-agent. `!important` *flips* part of this ordering. The full precedence, low to high:

```
1. user-agent normal
2. user normal
3. author normal
4. author !important     <-- !important reverses author vs user vs UA
5. user !important
6. user-agent !important   (and CSS transitions sit above even this)
```

The practical points: your normal author styles always beat the browser's defaults, and `!important` is a sledgehammer that jumps the queue.

### 3.2 Specificity — the scoring system **[I]**

When two declarations are in the same origin/layer/importance, the more **specific** selector wins. Specificity is a tuple of three numbers, conventionally written `(a, b, c)`:

- **a** = number of **ID** selectors (`#id`)
- **b** = number of **class**, attribute, and **pseudo-class** selectors (`.cls`, `[type]`, `:hover`)
- **c** = number of **type** (element) and **pseudo-element** selectors (`div`, `::before`)

Compare left-to-right like version numbers: any ID beats any number of classes; any class beats any number of types. The universal selector `*` and combinators (`>`, `+`, `~`, space) add **nothing**.

```css
/* Specificity of each selector, shown as (a,b,c): */
*                         /* (0,0,0) */
li                        /* (0,0,1) — one type */
ul li                     /* (0,0,2) — two types */
.card                     /* (0,1,0) — one class */
.card li                  /* (0,1,1) — one class + one type */
.card.active              /* (0,2,0) — two classes */
a:hover                   /* (0,1,1) — type + pseudo-class */
[type="text"]             /* (0,1,0) — attribute counts as a class */
#header                   /* (1,0,0) — one id */
#header .nav a:hover      /* (1,2,1) — id + class + pseudo-class + type */
```

So `#header` `(1,0,0)` beats `.card.active.featured.x.y.z` `(0,6,0)` — a single ID outranks any pile of classes. *This* is why over-using IDs for styling causes pain: nothing short of another ID or `!important` can override them.

**Special cases worth memorizing:**

- **Inline `style="…"`** behaves like a 4th, highest column — it beats any selector-based rule of the same importance.
- **`!important`** is even higher and lives in the origin/importance step (§3.1), above specificity entirely.
- **`:not()`, `:is()`, `:has()`** themselves add no specificity, but the *most specific selector inside them* does. (`:where()` is the exception — always 0.)
- The pseudo-element `::before` counts as a **type** `(0,0,1)`.

```css
/* :is() takes the HIGHEST specificity of its arguments. */
:is(#id, .class, p) span   /* counts the #id → (1,0,1), not the p */

/* :where() contributes ZERO no matter what's inside → great for resets. */
:where(#id, .class) span   /* (0,0,1) — only the trailing span counts */
```

### 3.3 Source order — the final tiebreaker **[B/I]**

If two declarations have the *same* origin, layer, importance **and** specificity, **the one that appears later in the source wins.** This is why the order of your stylesheets and rules matters, and why "just move it lower" sometimes fixes things.

```css
.btn { color: blue; }
.btn { color: green; }   /* WINS — same specificity, later in source → green */
```

### 3.4 `!important` — and why to avoid it **[I]**

Appending `!important` to a declaration lifts it into the high-importance tier (§3.1), bypassing specificity. It *feels* like a quick fix, but it starts an arms race: the only way to override an `!important` is *another* `!important` with higher specificity. Codebases drowning in `!important` are unmaintainable.

```css
.alert { color: red !important; }   /* beats even #id selectors of normal weight */
```

**Legitimate uses:** overriding inline styles you can't change (e.g. from a third-party widget), or utility classes that *must* always win by design (some utility frameworks use it intentionally). **Better long-term answer:** cascade layers (§3.6) give you ordering control without `!important`.

### 3.5 Inheritance **[B/I]**

Some properties **inherit**: if you don't set them on an element, the element takes the value from its parent. This is mostly **typography and text** properties — `color`, `font-family`, `font-size`, `line-height`, `text-align`, `letter-spacing`, `visibility`, `cursor`, `list-style`. Setting `font-family` on `<body>` cascades to every descendant, which is exactly what you want.

**Box-model and layout properties do NOT inherit** — `margin`, `padding`, `border`, `width`, `background`, `display`, `position`. If they did, nesting boxes would be chaos (every child would repeat the parent's padding).

You can control inheritance explicitly with these keyword values, which work on *any* property:

```css
.child {
  color: inherit;   /* force this property to take the parent's COMPUTED value */
  margin: initial;  /* reset to the property's spec default (margin → 0) */
  border: unset;    /* = inherit if the property naturally inherits, else initial */
  display: revert;  /* roll back to the user-agent/user value (undo author styles) */
}

/* The nuclear option — reset EVERY property on an element in one go: */
.reset-me { all: unset; }    /* useful when un-styling a <button> to look like text */
```

> **Why does `color` inherit but `background-color` doesn't?** Text needs a consistent color through nested spans/strongs — inheriting `color` is convenient. But `background-color` *visually composites* (a child with a transparent background shows the parent's), so inheriting it would double-paint. The spec chose inheritance per-property based on what's ergonomic.

### 3.6 Cascade layers — `@layer` **[A]**

**The problem they solve:** in a big codebase, base styles, component styles, utilities, and third-party CSS all compete by specificity and source order, leading to specificity wars and `!important` spam. **Cascade layers let you define explicit, named buckets and declare their priority order — and layer order beats specificity.** A declaration in a *later* layer beats one in an *earlier* layer *even if the earlier one has higher specificity*.

```css
/* 1) Declare the order ONCE, up front. Later layers WIN over earlier ones.
      Anything not in a layer ("unlayered") beats ALL layers. */
@layer reset, base, components, utilities;

/* 2) Put rules into layers. */
@layer reset {
  *, *::before, *::after { box-sizing: border-box; }
  body { margin: 0; }
}

@layer base {
  /* Even though this is an ID (high specificity)… */
  #main p { color: gray; }
}

@layer components {
  /* …this low-specificity class WINS, because `components` is a later layer
     than `base`. Layer order outranks specificity. No !important needed. */
  .prose p { color: black; }
}

@layer utilities {
  /* Utilities sit last → they reliably override component styles. */
  .text-center { text-align: center; }
}
```

You can also import a whole stylesheet directly into a layer (a rare *good* use of `@import`):

```css
@import url("bootstrap.css") layer(vendor);  /* corral third-party CSS into one layer */
```

**Precedence recap, highest priority last:** unlayered author styles > last declared layer > … > first declared layer. And `!important` *inverts* layer order (an `!important` in an *early* layer beats one in a later layer) — another reason to lean on layers instead of `!important`.

> **⚡ Version note:** `@layer` is baseline-supported across evergreen browsers and is the modern best practice for taming large stylesheets and integrating third-party CSS without fighting specificity.

### 3.7 Specificity reference table

| Selector | a (id) | b (class/attr/pseudo-class) | c (type/pseudo-element) | Score |
|---|---|---|---|---|
| `*` | 0 | 0 | 0 | (0,0,0) |
| `p` | 0 | 0 | 1 | (0,0,1) |
| `.card` | 0 | 1 | 0 | (0,1,0) |
| `[type=text]` | 0 | 1 | 0 | (0,1,0) |
| `:hover` | 0 | 1 | 0 | (0,1,0) |
| `::before` | 0 | 0 | 1 | (0,0,1) |
| `#id` | 1 | 0 | 0 | (1,0,0) |
| `nav ul li a` | 0 | 0 | 4 | (0,0,4) |
| `.nav .item.active` | 0 | 3 | 0 | (0,3,0) |
| `#hd .nav a:hover` | 1 | 2 | 1 | (1,2,1) |
| inline `style=""` | — | — | — | beats all selectors |
| `!important` | — | — | — | beats specificity entirely |

---

## 4. The Box Model

Everything in CSS is a rectangular **box**. The box model defines how an element's content area, padding, border, and margin combine to produce the space it occupies. Misunderstanding it is the #1 cause of "why is my layout 20px too wide."

### 4.1 The four layers **[B]**

From the inside out, every box has:

1. **Content** — the text/image itself. Sized by `width`/`height`.
2. **Padding** — transparent space *inside* the border, between content and border. Takes the element's background.
3. **Border** — a line around the padding. Has width, style, color.
4. **Margin** — transparent space *outside* the border, pushing other elements away. Never has a background.

```css
.box {
  width: 300px;            /* content width (by default — see box-sizing below) */
  padding: 20px;           /* space inside the border, all four sides */
  border: 5px solid #333;  /* 5px line on all sides */
  margin: 40px;            /* space outside, separating from neighbors */
  background: lightyellow; /* fills content + padding, NOT margin */
}
```

```
        ┌─────────────── margin (transparent) ───────────────┐
        │   ┌─────────── border ───────────┐                  │
        │   │   ┌─────── padding ───────┐   │                  │
        │   │   │      content          │   │                  │
        │   │   └───────────────────────┘   │                  │
        │   └───────────────────────────────┘                  │
        └──────────────────────────────────────────────────────┘
```

### 4.2 `box-sizing` — and why `border-box` is the right default **[B/I]**

Here's the historical wart: by default (`box-sizing: content-box`), `width` sets the **content** width *only*. Padding and border are then **added on top**. So the example above is actually `300 + 20*2 + 5*2 = 350px` wide. This makes sizing painful — set `width: 50%` plus padding and your box overflows its container.

`box-sizing: border-box` fixes this: `width` now includes padding and border. A `300px` box with `20px` padding and `5px` border *is* 300px wide; the content area shrinks to fit. This is far more intuitive, so virtually every modern stylesheet sets it globally:

```css
/* The single most common line in any modern reset.
   `inherit` on * + setting it on :root lets components opt out if ever needed,
   but most people just write `box-sizing: border-box` directly. */
*, *::before, *::after {
  box-sizing: border-box;   /* width/height now INCLUDE padding + border */
}
```

```css
.with-border-box {
  box-sizing: border-box;
  width: 300px;          /* the box is EXACTLY 300px wide… */
  padding: 20px;         /* …content area becomes 300 - 40 = 260px */
  border: 5px solid;     /* …content shrinks further; total still 300px */
}
```

### 4.3 Margin & padding shorthands **[B]**

```css
/* One value → all four sides */
margin: 10px;

/* Two values → vertical | horizontal (top/bottom, left/right) */
margin: 10px 20px;

/* Three values → top | horizontal | bottom */
margin: 10px 20px 30px;

/* Four values → top | right | bottom | left  (clockwise from top, "TRBL") */
margin: 10px 20px 30px 40px;

/* Individual sides */
margin-top: 10px;  margin-right: 20px;
margin-bottom: 30px; margin-left: 40px;

/* `auto` on horizontal margins centers a block element with a set width */
.container { width: 800px; margin: 0 auto; }   /* centered horizontally */

/* Negative margins are allowed — they pull elements together/outward */
.pull-up { margin-top: -10px; }
```

Padding works identically (same shorthand rules), but padding **cannot be negative** and `auto` does nothing on padding.

### 4.4 Margin collapsing — the confusing one **[I]**

**Vertical margins between block elements collapse**: when two vertical margins meet, the result is *not* their sum — it's the *larger* of the two. This surprises everyone the first time.

```css
.a { margin-bottom: 30px; }
.b { margin-top: 20px; }
/* The gap between .a and .b is 30px (the larger), NOT 50px. */
```

**Why does this exist?** It comes from typesetting: if every paragraph has a top and bottom margin, you don't want double spacing between consecutive paragraphs — you want one consistent gap. Collapsing gives that automatically.

**Three situations where margins collapse:**

1. **Adjacent siblings** — bottom margin of one meets top margin of the next (above).
2. **Parent and first/last child** — if there's no border, padding, or content between them, a child's top margin can "escape" through the parent and collapse with the parent's top margin (a frequent "why is there space above my container" bug).
3. **Empty blocks** — an empty element's own top and bottom margins can collapse together.

**Margins do NOT collapse when:** the boxes are in a **flex or grid container**, are **floated** or **absolutely positioned**, or there's a **border/padding/overflow** boundary between them. Horizontal margins *never* collapse.

```css
/* Fixes for the parent/child escape: any of these creates a boundary. */
.parent { padding-top: 1px; }              /* padding stops the escape */
.parent { border-top: 1px solid transparent; }
.parent { overflow: auto; }                /* establishes a BFC (§9.5) */
.parent { display: flow-root; }            /* the clean, modern fix */
```

> **Modern escape hatch:** in **flex and grid layouts margins never collapse**, and you'd typically use `gap` for spacing instead of margins. So in modern layout work you rarely hit collapsing at all — it's mostly a flow-layout phenomenon.

### 4.5 `width`/`height`, min/max, and `overflow` **[B/I]**

```css
.box {
  width: 50%;          /* relative to the containing block's width */
  max-width: 600px;    /* never grow past 600px (responsive cap) */
  min-width: 200px;    /* never shrink below 200px */
  height: 100px;
  min-height: 100vh;   /* at least the viewport height */
  /* `auto` (the default for width) = fill available inline space for blocks */
}

/* When content is bigger than the box, `overflow` decides what happens. */
.scrollbox {
  height: 200px;
  overflow: auto;      /* show scrollbars only if needed */
  /* visible (default, content spills out) | hidden (clip) | scroll (always) |
     clip (clip, no scroll) | auto (scroll if needed) */
  overflow-x: hidden;  /* axis-specific control */
  overflow-y: auto;
}
```

> **Gotcha — percentage heights:** `height: 50%` only works if the *parent* has an explicit height (or you're in a flex/grid context). A percentage height resolves against the parent's height, and if the parent's height is `auto` (content-based), the percentage is indeterminate and the browser falls back to `auto`. This is the classic "my full-height div doesn't fill" problem — set a height up the chain, use `100vh`/`100dvh`, or use flex/grid. (See §18.)

---

## 5. Units & Values

CSS values can be **absolute** (fixed) or **relative** (computed from something else — font size, parent, viewport). Choosing the right unit is what makes a layout scale and respect user preferences.

### 5.1 Absolute units **[B]**

```css
.x { width: 200px; }    /* px = CSS pixel — the workhorse absolute unit */
/* Print-oriented units exist (pt, cm, mm, in) but you rarely use them on screen. */
```

A CSS pixel isn't necessarily one device pixel (high-DPI "retina" screens pack several device pixels per CSS pixel), but for layout purposes treat `px` as a fixed, predictable unit. Use it for things that genuinely shouldn't scale with font size — hairline borders, small fixed offsets.

### 5.2 Font-relative units: `em`, `rem`, `ch`, `ex` **[B/I]**

```css
/* em = relative to THIS element's font-size (or the parent's for font-size itself).
   Compounds when nested — a common foot-gun. */
.card { font-size: 20px; padding: 1.5em; }  /* padding = 30px (1.5 × 20) */

/* rem = relative to the ROOT (<html>) font-size. Does NOT compound.
   The modern default for sizing because it's predictable and respects the
   user's browser font-size setting (accessibility!). */
:root { font-size: 16px; }     /* 1rem = 16px (also the browser default) */
.title { font-size: 2rem; }    /* = 32px, regardless of nesting */
.gap   { margin-bottom: 1.5rem; }

/* ch = width of the "0" character in the current font — great for line length */
.prose { max-width: 65ch; }    /* ~65 characters per line = comfortable reading */

/* ex = x-height; rarely used. */
```

**When to use which:** Use **`rem`** for most sizing (font sizes, spacing, widths) — predictable and accessible. Use **`em`** when you *want* something to scale with its own font size (e.g. padding inside a button that should grow with the button's text). Use **`ch`** to constrain line length for readability. Avoid setting `:root { font-size }` in `px` if you want to respect user zoom of the base font — but `16px` matches the default and is fine; many prefer `font-size: 100%` to defer to the user.

> **Why `rem` over `px` for font sizes?** If a user has bumped their browser's default font size up for readability, `px` font sizes ignore that preference, but `rem` scales with it. Respecting that is an accessibility win.

### 5.3 Viewport units: `vw`, `vh`, `vmin`, `vmax`, and the dynamic variants **[I]**

```css
.hero { height: 100vh; }   /* 100% of the viewport HEIGHT */
.full { width: 100vw; }    /* 100% of the viewport WIDTH */
/* 1vw = 1% of viewport width; 1vh = 1% of viewport height.            */
/* vmin = 1% of the SMALLER dimension; vmax = 1% of the LARGER.        */
.square { width: 50vmin; height: 50vmin; }  /* stays square, scales w/ screen */
```

> **⚡ Version note — mobile viewport units:** On mobile, the browser's URL/tool bars appear and disappear, changing the visible height. `100vh` is the *largest* possible height, so content can get cut off behind the toolbar. Modern CSS adds **dynamic viewport units**: `dvh`/`dvw` (dynamic — updates as bars show/hide), `svh`/`svw` (small — assumes bars visible), `lvh`/`lvw` (large — assumes bars hidden). For a true full-screen section on mobile, prefer **`height: 100dvh`**.

### 5.4 Percentages **[B/I]**

A percentage is *relative to something*, and **what it's relative to depends on the property**:

```css
.col   { width: 50%; }         /* 50% of the containing block's WIDTH */
.box   { padding: 10%; }       /* % padding/margin is relative to parent WIDTH —
                                  even padding-top! This enables aspect-ratio hacks. */
.tall  { height: 50%; }        /* relative to parent HEIGHT (needs explicit parent height) */
.lh    { line-height: 150%; }  /* relative to the element's font-size */
```

The quirk that all of `padding`/`margin` percentages (including top/bottom) resolve against the **width** is deliberate — it makes the old "padding-bottom hack" for aspect ratios work. These days use `aspect-ratio` instead (§15.6).

### 5.5 `calc()`, `min()`, `max()`, `clamp()` **[I]**

These math functions let you mix units and compute values at render time — incredibly powerful for responsive design.

```css
/* calc() — arithmetic, can MIX units. Spaces around + and - are REQUIRED. */
.sidebar { width: calc(100% - 250px); }    /* fill remaining space beside a 250px col */
.box     { padding: calc(1rem + 2vw); }    /* mix fixed + fluid */

/* min() — use the SMALLEST of the values. Great as a responsive cap. */
.container { width: min(90%, 1200px); }    /* 90% on small screens, capped at 1200px */

/* max() — use the LARGEST. Enforce a floor. */
.box { width: max(50%, 300px); }           /* never narrower than 300px */

/* clamp(MIN, PREFERRED, MAX) — the responsive sweet spot. Returns PREFERRED,
   but never below MIN or above MAX. The backbone of fluid typography. */
h1 { font-size: clamp(1.5rem, 1rem + 3vw, 3rem); }
/*  ↑ at least 1.5rem, scales with viewport (1rem + 3vw), caps at 3rem. */
```

**How fluid type with `clamp()` works:** the middle expression usually combines a fixed base (`1rem`) with a viewport term (`3vw`) so the value grows smoothly as the screen widens, while `min`/`max` keep it readable at the extremes. This replaces a stack of media-query font-size overrides with one line.

### 5.6 Custom properties (CSS variables) **[I]**

**What they are:** user-defined properties whose names start with `--`. Unlike Sass variables (which are compiled away), CSS custom properties are *live* — they exist in the browser, **inherit** down the tree, can be read/changed by JavaScript, can differ per element, and respond to media queries and `:hover`. They are the foundation of theming and dark mode.

```css
/* Define on :root so they're globally available (they INHERIT to all descendants). */
:root {
  --brand: #4f46e5;
  --space: 1rem;
  --radius: 8px;
  --font-body: system-ui, sans-serif;
}

/* USE them with var(). Second argument is an optional FALLBACK if undefined. */
.btn {
  background: var(--brand);
  padding: var(--space);
  border-radius: var(--radius);
  color: var(--btn-fg, white);   /* falls back to white if --btn-fg isn't set */
}

/* Because they inherit and cascade, you can OVERRIDE per-scope. */
.card.warning {
  --brand: #d97706;   /* every var(--brand) inside this card now uses orange */
}

/* They can be combined with calc() and changed by state/media. */
.box { gap: calc(var(--space) * 2); }
```

**Theming / dark mode example** — flip a whole palette by reassigning variables:

```css
:root {                       /* light theme (default) */
  --bg: white;
  --fg: #111;
  --card: #f5f5f5;
}
:root[data-theme="dark"] {    /* JS toggles <html data-theme="dark"> */
  --bg: #0d1117;
  --fg: #e6edf3;
  --card: #161b22;
}
/* Respect the OS setting automatically when no manual choice is made. */
@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) { --bg: #0d1117; --fg: #e6edf3; --card: #161b22; }
}
body  { background: var(--bg); color: var(--fg); }
.card { background: var(--card); }
```

```js
// Read & write custom properties from JavaScript:
const root = document.documentElement;
root.style.setProperty("--brand", "tomato");           // set
getComputedStyle(root).getPropertyValue("--brand");    // read
```

> **⚡ Version note — `@property`:** You can now *register* a custom property with a type, so it can be *animated*. A plain `--x` is just a string to the browser and can't tween; a registered one can.
> ```css
> @property --angle {
>   syntax: "<angle>";        /* declare the type so it can animate */
>   initial-value: 0deg;
>   inherits: false;
> }
> .spinner { background: conic-gradient(from var(--angle), red, blue); }
> /* now animating --angle from 0deg→360deg actually animates the gradient */
> ```

---

## 6. Colors

Color in CSS has quietly become deep. You can still write `red`, but modern color spaces give you better gradients, accessible contrast, and easy palette generation.

### 6.1 The classic notations **[B]**

```css
.a { color: red; }                 /* named color (147 keywords + transparent) */
.b { color: #ff0000; }             /* hex: #RRGGBB */
.c { color: #f00; }                /* short hex: #RGB → #ff0000 */
.d { color: #ff000080; }           /* 8-digit hex: #RRGGBBAA (last byte = alpha) */
.e { color: rgb(255 0 0); }        /* modern space-separated rgb (no commas needed) */
.f { color: rgb(255 0 0 / 50%); }  /* with alpha after a slash */
.g { color: hsl(0 100% 50%); }     /* Hue Saturation Lightness */
.h { color: hsl(0 100% 50% / 0.5);}/* HSL + alpha */
```

**Why `hsl` is friendlier than hex/rgb:** `hsl(hue, saturation, lightness)` maps to how humans think about color. Hue is an angle on the color wheel (0/360 = red, 120 = green, 240 = blue). Want the same color but lighter? Just raise the lightness. Want a hover state? Nudge one number. With hex you'd have to recompute three.

```css
.btn        { background: hsl(220 90% 50%); }
.btn:hover  { background: hsl(220 90% 40%); }   /* same hue, darker = -10% L */
```

### 6.2 `oklch()` / `oklab()` — the modern, perceptually-uniform spaces **[I/A]**

**The problem with HSL/RGB:** they are *not perceptually uniform*. In HSL, `lightness: 50%` yellow looks much brighter than `lightness: 50%` blue, and interpolating between two HSL colors can pass through muddy grays. This makes consistent palettes and smooth gradients hard.

**`oklch()`** fixes this. It's `oklch(Lightness Chroma Hue)`:
- **L** — perceptual lightness, `0%`–`100%` (or `0`–`1`). Equal L *looks* equally bright across hues.
- **C** — chroma (colorfulness/saturation), `0` to ~`0.4`.
- **H** — hue angle in degrees.

Because L is perceptual, you can build accessible palettes (same L = same visual weight), generate tints/shades by changing only L, and get gradients that don't go muddy. It also covers a **wider gamut** (P3 displays) than sRGB hex.

```css
:root {
  --brand:       oklch(60% 0.15 264);   /* a calm blue */
  --brand-hover: oklch(52% 0.15 264);   /* same hue & chroma, lower L = darker */
  --brand-tint:  oklch(92% 0.05 264);   /* light tint for backgrounds */
}
.btn        { background: var(--brand); color: white; }
.btn:hover  { background: var(--brand-hover); }

/* oklab() is the cartesian cousin (L a b); use oklch unless you need a/b math. */
.x { color: oklab(60% 0.1 -0.1); }
```

> **⚡ Version note:** `oklch()`/`oklab()` are baseline in 2026 and are now the *recommended* way to author colors in new design systems. For maximum legacy safety you can still provide an sRGB fallback first (the browser keeps the last value it understands):
> ```css
> .btn { background: #2563eb; background: oklch(60% 0.15 264); }
> ```

### 6.3 `color-mix()` and relative color **[A]**

```css
/* color-mix() blends two colors in a chosen color space. Perfect for hover
   states, tints, and overlays derived from a single brand variable. */
.btn        { --c: oklch(60% 0.15 264); background: var(--c); }
.btn:hover  { background: color-mix(in oklch, var(--c), black 12%); } /* 12% darker */
.badge      { background: color-mix(in oklch, var(--c), white 80%); } /* light tint */

/* Relative color syntax — derive a new color from an existing one. */
.alpha { color: rgb(from var(--c) r g b / 50%); }  /* same color, 50% alpha */
```

### 6.4 Opacity & alpha **[B]**

```css
/* `opacity` makes the WHOLE element (and its children) semi-transparent,
   and creates a new stacking context (§12.4). */
.ghost { opacity: 0.5; }

/* Alpha in the COLOR affects only that color, not children — usually preferred. */
.overlay { background: rgb(0 0 0 / 0.5); }   /* 50% black overlay */
```

> **Gotcha:** `opacity: 0.5` on a parent fades the text *and* every child — you can't make a child fully opaque again. If you only want a translucent *background*, set alpha on `background-color`, not `opacity`.

### 6.5 Gradients **[I]**

Gradients are generated *images*, so they go anywhere an image does (`background`, `border-image`, `mask`).

```css
/* LINEAR — color transition along a line. Angle or keyword direction. */
.a { background: linear-gradient(to right, red, orange, yellow); }
.b { background: linear-gradient(45deg, #4f46e5, #06b6d4); }
.c { background: linear-gradient(to bottom, black 0%, transparent 100%); } /* fade */

/* Color STOPS control where each color sits; two stops = a hard edge (stripe). */
.stripes {
  background: linear-gradient(90deg, red 0 50%, blue 50% 100%);  /* half red half blue */
}

/* RADIAL — emanates from a point. */
.d { background: radial-gradient(circle at center, white, transparent); }

/* CONIC — sweeps around a center. Great for pie charts / color wheels. */
.pie { background: conic-gradient(red 0 25%, green 25% 50%, blue 50% 100%); }

/* Interpolate in oklch for smoother, non-muddy gradients (modern). */
.smooth { background: linear-gradient(in oklch, blue, red); }

/* Gradients stack like layered images (comma-separated), first = top layer. */
.layered {
  background:
    linear-gradient(rgb(0 0 0 / .4), rgb(0 0 0 / .4)),  /* dark overlay on top */
    url("photo.jpg") center / cover no-repeat;          /* image underneath */
}
```

---

## 7. Typography

Text is most of the web. Good typography is mostly about a few properties used well, a sensible scale, and consistent vertical rhythm.

### 7.1 Font properties **[B]**

```css
body {
  /* font-family: a PRIORITIZED LIST. The browser uses the first available
     font, falling back rightward. Always end with a generic family. */
  font-family: "Inter", system-ui, -apple-system, Segoe UI, Roboto, sans-serif;

  font-size: 1rem;        /* base text size */
  font-weight: 400;       /* 100–900; 400 = normal, 700 = bold */
  font-style: normal;     /* normal | italic | oblique */
  line-height: 1.6;       /* UNITLESS is best (= 1.6 × font-size); scales with text */
  letter-spacing: 0.01em; /* tracking */
  font-variant: small-caps;
}

/* The `font` SHORTHAND (order-sensitive; size & family are required): */
h1 { font: 700 2rem/1.2 "Inter", sans-serif; }  /* weight size/line-height family */
```

**`system-ui`** resolves to the operating system's native UI font (Segoe UI on Windows, San Francisco on macOS) — zero download, instantly familiar, great default.

> **Why unitless `line-height`?** A unitless value is a multiplier applied per element using *its own* font size. If you write `line-height: 24px` on `body`, a large heading inherits the literal `24px` and lines overlap. `line-height: 1.5` inherits the *ratio*, so each element computes a sensible height from its own size.

### 7.2 Web fonts: `@font-face` & variable fonts **[I]**

`@font-face` lets you ship a custom font file. Define it once, then use the `font-family` name anywhere.

```css
@font-face {
  font-family: "Inter";              /* the name you'll reference */
  src: url("/fonts/Inter.woff2") format("woff2");  /* woff2 = best compression */
  font-weight: 100 900;              /* a RANGE → this is a VARIABLE font */
  font-display: swap;                /* show fallback text immediately, swap when loaded */
}
body { font-family: "Inter", sans-serif; }
```

**Variable fonts** pack many weights/widths/styles into one file along continuous **axes**, so `font-weight: 437` is valid and you ship one file instead of a dozen. Control axes with `font-variation-settings` for custom axes:

```css
.heavy { font-weight: 850; }                       /* standard weight axis */
.wide  { font-variation-settings: "wdth" 125; }    /* a custom width axis */
```

**`font-display`** controls the loading behavior. `swap` (show fallback, then swap) avoids invisible text (FOIT); `optional` is best for performance-critical text. This is an important UX/CLS consideration.

### 7.3 Text layout & decoration **[B/I]**

```css
.text {
  text-align: left;          /* left | right | center | justify */
  text-decoration: underline; /* shorthand: line style color thickness */
  text-decoration: underline wavy red 2px;
  text-transform: uppercase; /* uppercase | lowercase | capitalize */
  text-indent: 2em;          /* first-line indent */
  white-space: nowrap;       /* prevent wrapping; `pre` keeps whitespace */
  word-break: break-word;
  overflow-wrap: anywhere;   /* break long URLs/words to avoid overflow */
  hyphens: auto;             /* automatic hyphenation (needs lang attr) */
  text-wrap: balance;        /* ⚡ even out ragged lines in headings (modern) */
  text-wrap: pretty;         /* ⚡ avoid orphans in body text (modern) */
}

/* Truncate to one line with an ellipsis: */
.ellipsis {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
/* Clamp to N lines (widely supported): */
.clamp {
  display: -webkit-box;
  -webkit-line-clamp: 3;
  line-clamp: 3;
  -webkit-box-orient: vertical;
  overflow: hidden;
}
```

> **⚡ Version note:** `text-wrap: balance` (great for multi-line headings) and `text-wrap: pretty` (prevents single-word last lines in paragraphs) are recent, widely-supported niceties. They degrade gracefully — older browsers just ignore them.

### 7.4 Vertical rhythm **[I]**

**Vertical rhythm** means consistent vertical spacing so text sits on an invisible baseline grid, giving a calm, organized feel. The practical recipe: pick a base `line-height`, derive your spacing scale from it, and apply margins in one direction to avoid collapse confusion.

```css
:root { --rhythm: 1.5rem; }   /* base spacing unit tied to line-height */

body { line-height: 1.5; font-size: 1rem; }

/* "Lobotomized owl": add top margin to every element that FOLLOWS a sibling.
   Gives consistent flow spacing without per-element margins or collapse issues. */
* + * { margin-top: var(--rhythm); }

/* (In flex/grid layouts, prefer `gap` over margins — see §10/§11.) */
```

---

## 8. Backgrounds, Borders & Shadows

### 8.1 Backgrounds **[B/I]**

```css
.hero {
  background-color: #111;                 /* solid fallback */
  background-image: url("/img/bg.jpg");    /* one or many (comma-separated) */
  background-repeat: no-repeat;            /* repeat | repeat-x | repeat-y | no-repeat */
  background-position: center top;         /* where the image sits */
  background-size: cover;                  /* cover (fill, crop) | contain (fit, letterbox) | 200px 100px */
  background-attachment: fixed;            /* fixed = parallax-ish; scroll (default) */
  background-clip: padding-box;            /* how far the bg extends (border/padding/content-box) */
}

/* The shorthand (very common; `/` separates position and size): */
.card {
  background: #fff url("/img/x.png") no-repeat center / cover;
}

/* Multiple layered backgrounds (first listed = topmost): */
.layered {
  background:
    linear-gradient(rgb(0 0 0 / .5), rgb(0 0 0 / .5)),  /* tint overlay */
    url("/img/photo.jpg") center / cover;
}
```

**`cover` vs `contain`:** `cover` scales the image to *fill* the box (cropping overflow) — use for hero/banner images. `contain` scales to *fit entirely* inside (possibly leaving empty space) — use when the whole image must be visible (logos).

### 8.2 Borders & `border-radius` **[B]**

```css
.box {
  border: 2px solid #333;          /* shorthand: width style color */
  border-style: dashed;            /* solid|dashed|dotted|double|groove|none… */
  border-top: 4px solid royalblue; /* one side */
  border-inline: 1px solid #ddd;   /* logical: left+right (§15.4) */

  border-radius: 8px;              /* rounded corners (all four) */
  border-radius: 8px 0 8px 0;      /* per-corner: TL TR BR BL */
  border-radius: 50%;              /* a circle (on a square box) / ellipse */
  border-radius: 100vmax;          /* "pill" shape for buttons */
}

/* A border that doesn't shift layout: use outline or box-shadow instead, since
   adding a border changes the box size unless box-sizing:border-box. */
```

### 8.3 `box-shadow` **[B/I]**

```css
.card {
  /* offset-x | offset-y | blur | spread | color */
  box-shadow: 0 4px 12px rgb(0 0 0 / 0.15);

  /* `inset` puts the shadow INSIDE the box (e.g. pressed look). */
  box-shadow: inset 0 2px 4px rgb(0 0 0 / 0.2);

  /* Multiple shadows, comma-separated — layer them for realistic depth. */
  box-shadow:
    0 1px 2px rgb(0 0 0 / 0.1),
    0 4px 8px rgb(0 0 0 / 0.08),
    0 12px 24px rgb(0 0 0 / 0.06);
}
```

**Anatomy:** offset-x/y move the shadow; **blur** softens it (bigger = fuzzier); **spread** grows/shrinks it before blurring; negative spread tucks it under. Layering several low-opacity shadows looks far more natural than one harsh one. `text-shadow` works similarly (no spread/inset).

### 8.4 `outline` vs `border` **[I]**

`outline` draws *outside* the border and **does not take up space** (doesn't affect layout) — and it can't be rounded the same way historically (modern browsers do follow `border-radius` for outlines now). Its crucial role: **focus indicators**. Never remove focus outlines without providing a visible alternative (§17).

```css
button:focus-visible {
  outline: 2px solid royalblue;   /* width style color */
  outline-offset: 2px;            /* push it away from the edge */
}
/* NEVER do `outline: none` with no replacement — it breaks keyboard accessibility. */
```

---

## 9. Display Models & Normal Flow

Before flexbox and grid, you must understand **normal flow** — the default way the browser lays boxes out when you don't intervene. Everything else is a deviation from this baseline.

### 9.1 Block vs inline **[B/I]**

Every element has an **outer display type** that determines how it participates in flow:

- **Block-level** boxes (`<div>`, `<p>`, `<h1>`, `<section>`…) stack **vertically**, each starting on a new line, and by default **fill the full available width** of their container. You can set `width`/`height` and all margins/padding.
- **Inline-level** boxes (`<span>`, `<a>`, `<strong>`, `<em>`…) flow **horizontally** within a line, wrapping like words in a sentence. They size to their content. **`width`/`height` are ignored**, and **vertical margins/padding don't affect layout** (they paint but don't push other lines).

```css
span      { display: inline; }       /* default for span — sits in the text flow */
div       { display: block; }        /* default for div — stacks, full width */

/* inline-block: flows inline (sits next to text) BUT respects width/height
   and vertical padding/margin. The classic way to lay out items in a row
   before flexbox; still handy for things like buttons in a sentence. */
.tag { display: inline-block; padding: 4px 8px; width: 80px; }

/* `display: none` removes the element from flow AND rendering entirely
   (no box, not read by most assistive tech). vs visibility:hidden which
   keeps the SPACE but hides it. */
.hidden  { display: none; }
.invisible { visibility: hidden; }   /* occupies space, just not painted */
```

### 9.2 The two-value display syntax **[I]**

Modern `display` is really two values: **outer** (how the box behaves toward siblings — `block`/`inline`) and **inner** (how its *children* are laid out — `flow`/`flex`/`grid`/`table`). `display: flex` is shorthand for `display: block flex` (a block-level box whose children are flex items).

```css
.a { display: flex; }          /* = block flex  (block outside, flex inside) */
.b { display: inline-flex; }   /* = inline flex (inline outside, flex inside) */
.c { display: grid; }          /* = block grid */
.d { display: inline-grid; }   /* = inline grid */
```

The mental unlock: setting `display: flex` on a **container** doesn't change how *it* sits among its siblings (still a block); it changes how its **children** are arranged.

### 9.3 The visual formatting model & in-flow vs out-of-flow **[I]**

In **normal flow**, boxes are laid out top-to-bottom (block) and left-to-right within lines (inline), respecting the writing mode. Elements are **in-flow** by default. Some things take an element **out of flow** so it no longer affects its siblings' positions:

- `position: absolute` / `fixed` (§12)
- `float: left/right` (legacy; now mostly for wrapping text around images)

Out-of-flow elements don't reserve space — siblings act as if they aren't there.

### 9.4 `float` (legacy, but still used) **[I]**

`float` was the old hack for multi-column layouts (now obsolete — use grid/flex). Its remaining legitimate use is letting text **wrap around** an image or pull-quote.

```css
img.figure {
  float: left;           /* text flows around the right side */
  margin: 0 1rem 1rem 0;
  shape-outside: circle(); /* optionally wrap text to a shape */
}
.clearfix::after {       /* old technique to contain floats — rarely needed now */
  content: ""; display: block; clear: both;
}
```

### 9.5 Block Formatting Contexts (BFC) **[A]**

A **BFC** is a mini-layout region where floats are contained and margins don't collapse across its boundary. You create one with `overflow: auto/hidden`, `display: flow-root` (the purpose-built keyword), floats, flex/grid containers, or absolute positioning. It's the mechanism behind several margin-collapse and float-clearing fixes.

```css
.contains-floats { display: flow-root; }  /* clean BFC — contains child floats,
                                              stops margin escape (§4.4) */
```

---

## 10. Flexbox

**Flexbox** (the Flexible Box Layout) lays out items in a **single dimension** — a row *or* a column — and excels at distributing space and aligning items along that line. Reach for flexbox for one-dimensional things: navbars, toolbars, button groups, centering, a row of cards that should share space.

### 10.1 The core concept: main axis & cross axis **[I]**

This is *the* thing to internalize. A flex container has two axes:

- **Main axis** — the direction items flow. Set by `flex-direction`. `row` (default) → main axis is horizontal; `column` → main axis is vertical.
- **Cross axis** — perpendicular to the main axis.

`justify-content` aligns along the **main** axis. `align-items` aligns along the **cross** axis. If you remember "justify = main, align = cross," flexbox stops being confusing. When you flip `flex-direction` to `column`, the axes swap, so `justify-content` now controls *vertical* distribution and `align-items` controls *horizontal*.

### 10.2 Container properties **[I]**

```css
.container {
  display: flex;                  /* turn children into flex items */

  flex-direction: row;            /* row | row-reverse | column | column-reverse */

  flex-wrap: nowrap;              /* nowrap (default, may overflow) | wrap | wrap-reverse */
  /* Shorthand for direction + wrap: */
  flex-flow: row wrap;

  /* MAIN-axis distribution of items: */
  justify-content: flex-start;    /* flex-start | flex-end | center |
                                     space-between | space-around | space-evenly */

  /* CROSS-axis alignment of items within the line: */
  align-items: stretch;           /* stretch (default) | flex-start | flex-end |
                                     center | baseline */

  /* CROSS-axis distribution of MULTIPLE LINES (only matters when wrapping): */
  align-content: flex-start;      /* same keywords as justify-content */

  gap: 1rem;                      /* space BETWEEN items — modern, replaces margins */
  gap: 1rem 2rem;                 /* row-gap column-gap */
}
```

**`justify-content` keywords visualized** (main axis, items `[A][B][C]`):

```
flex-start:    [A][B][C]..........
center:        ....[A][B][C]......
flex-end:      ..........[A][B][C]
space-between: [A].....[B].....[C]   (no edge gaps)
space-around:  ..[A]...[B]...[C]..   (half gaps at edges)
space-evenly:  ...[A]..[B]..[C]...   (equal gaps everywhere)
```

### 10.3 Item properties **[I]**

These go on the **children**, not the container.

```css
.item {
  /* flex-grow: share of EXTRA space this item absorbs (0 = don't grow). */
  flex-grow: 1;
  /* flex-shrink: how readily it shrinks when space is tight (1 = default). */
  flex-shrink: 1;
  /* flex-basis: the item's starting size along the main axis before grow/shrink. */
  flex-basis: 200px;             /* auto (use width/content) | a length | 0 */

  /* The `flex` SHORTHAND (grow shrink basis) — learn these idioms: */
  flex: 1;                       /* = 1 1 0  → equal-width columns that fill space */
  flex: auto;                    /* = 1 1 auto → grow/shrink from content size */
  flex: none;                    /* = 0 0 auto → fixed, won't grow or shrink */
  flex: 0 0 250px;               /* fixed 250px sidebar that never flexes */

  /* Override the container's cross-axis alignment for THIS item only. */
  align-self: center;

  /* Reorder visually WITHOUT changing the DOM (use cautiously — a11y). */
  order: 2;                      /* default 0; higher = later */
}
```

**The `flex: 1` insight:** `flex: 1` expands to `flex-grow:1; flex-shrink:1; flex-basis:0`. Because the basis is `0`, all items start from zero and *grow in equal proportion*, producing equal-width columns regardless of content. `flex: auto` (basis `auto`) instead starts from content size, so wider content keeps more space. Knowing this distinction explains most "why aren't my flex columns equal" questions.

### 10.4 Common flexbox layouts **[I]**

```css
/* 1) Perfect centering — the famous one-liner. */
.center {
  display: flex;
  justify-content: center;   /* horizontal (main axis = row) */
  align-items: center;       /* vertical (cross axis) */
  min-height: 100vh;
}

/* 2) Navbar: logo on the left, links pushed to the right. */
.navbar { display: flex; align-items: center; gap: 1rem; }
.navbar .logo { margin-right: auto; }   /* auto margin eats all free space → pushes rest right */
/* (Alternatively: justify-content: space-between with two groups.) */

/* 3) Holy-grail-ish: fixed sidebar + fluid main. */
.layout { display: flex; gap: 1rem; }
.layout .sidebar { flex: 0 0 250px; }   /* fixed width */
.layout .main    { flex: 1; }           /* fills the rest */

/* 4) Responsive card row that wraps and keeps a minimum width. */
.cards { display: flex; flex-wrap: wrap; gap: 1rem; }
.cards .card { flex: 1 1 250px; }        /* grow/shrink, ~250px ideal → wraps neatly */

/* 5) Sticky footer: push footer to the bottom even on short pages. */
body { display: flex; flex-direction: column; min-height: 100dvh; }
body main { flex: 1; }                   /* main grows, footer sits at bottom */
```

> **Auto margins are a flexbox superpower:** an `auto` margin on a flex item absorbs all free space on that side, letting you push one item (or group) to an edge — cleaner than `space-between` when grouping is uneven.

A complete, annotated navbar putting several of these ideas together:

```html
<nav class="nav">
  <a class="nav__logo" href="/">Acme</a>
  <ul class="nav__links">
    <li><a href="/features">Features</a></li>
    <li><a href="/pricing">Pricing</a></li>
    <li><a href="/docs">Docs</a></li>
  </ul>
  <a class="nav__cta" href="/signup">Sign up</a>
</nav>
```

```css
.nav {
  display: flex;          /* one-dimensional row → flexbox is the right tool */
  align-items: center;    /* vertically center logo, links, and button together */
  gap: 1.5rem;            /* consistent spacing; no per-item margins */
  padding: 0.75rem 1.5rem;
}
.nav__links {
  display: flex;          /* nested flex row for the link list */
  gap: 1rem;
  list-style: none;       /* drop the <li> bullets */
  margin: 0; padding: 0;  /* zero out the <ul> defaults */
  margin-inline-end: auto;/* auto margin EATS free space → pushes the CTA to the far right */
}
.nav__cta {
  padding: 0.5rem 1rem;
  background: var(--brand, royalblue);
  color: white; border-radius: 6px;
}
```

### 10.5 Flexbox property reference

| Property | Goes on | Does |
|---|---|---|
| `display: flex` | container | turn children into flex items |
| `flex-direction` | container | main axis: `row`/`column`(+`-reverse`) |
| `flex-wrap` | container | allow items to wrap to new lines |
| `justify-content` | container | distribute along **main** axis |
| `align-items` | container | align along **cross** axis |
| `align-content` | container | distribute **lines** (when wrapping) |
| `gap` | container | spacing between items |
| `flex-grow` | item | share of extra space (0 = none) |
| `flex-shrink` | item | willingness to shrink (1 = default) |
| `flex-basis` | item | starting main-axis size |
| `flex` | item | shorthand: grow shrink basis |
| `align-self` | item | override cross-axis alignment |
| `order` | item | visual reorder (not DOM order) |

### 10.6 When to use flexbox vs grid

Use **flexbox** when you're laying out items in *one direction* and want them to **share/distribute space based on their content** (a row of buttons, a navbar, a wrapping tag list). Use **grid** (§11) when you need *two-dimensional* control — rows **and** columns aligned together (a page layout, an image gallery, a form with aligned labels). They compose beautifully: a grid page whose cells contain flex rows.

---

## 11. CSS Grid

**CSS Grid** is a **two-dimensional** layout system: you define rows and columns, then place items into the resulting cells. Where flexbox distributes along one line, grid lets you align content in both dimensions at once — it is the most powerful layout tool CSS has ever had.

### 11.1 Defining the grid **[I]**

```css
.grid {
  display: grid;

  /* Define COLUMNS: three columns. The `fr` unit = a fraction of free space. */
  grid-template-columns: 1fr 1fr 1fr;      /* three equal columns */
  grid-template-columns: 200px 1fr;        /* fixed sidebar + fluid main */
  grid-template-columns: 1fr 2fr 1fr;      /* middle column twice as wide */

  /* Define ROWS (often left to `auto` so they size to content). */
  grid-template-rows: auto 1fr auto;       /* header / body fills / footer */

  gap: 1rem;                               /* gutters between tracks */
  gap: 1rem 2rem;                          /* row-gap column-gap */
}
```

**The `fr` unit** ("fractional unit") distributes *leftover* space after fixed sizes and gaps are accounted for. `1fr 1fr` = two equal columns. `1fr 2fr` = one-third / two-thirds. It's like `flex-grow` for grid tracks, and it's why grid layouts are so easy to make responsive.

### 11.2 `repeat()`, `minmax()`, `auto-fit`/`auto-fill` **[I/A]**

```css
/* repeat() avoids repetition: */
grid-template-columns: repeat(3, 1fr);          /* = 1fr 1fr 1fr */
grid-template-columns: repeat(4, 100px);

/* minmax(min, max): a track at least `min`, at most `max`. */
grid-template-columns: minmax(200px, 1fr) 3fr;  /* sidebar floor of 200px */

/* THE responsive grid pattern — fills as many columns as fit, no media queries:
   auto-fit packs items, minmax keeps each at least 250px, fr lets them grow. */
.gallery {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 1rem;
}
```

**`auto-fit` vs `auto-fill`:** both create as many tracks as fit. **`auto-fill`** keeps the *empty* leftover tracks (so existing items don't stretch to fill the row). **`auto-fit`** *collapses* empty tracks, letting the present items expand to fill the row. For a responsive gallery you usually want `auto-fit`.

### 11.3 Placing items: lines, spans, and areas **[I/A]**

Grid lines are numbered starting at 1 (the left/top edge). You place an item by telling it which lines to start and end on.

```css
.item {
  grid-column: 1 / 3;        /* start at line 1, end at line 3 → spans 2 columns */
  grid-column: 1 / span 2;   /* equivalent: start at 1, span 2 columns */
  grid-row: 2 / 4;           /* spans rows 2–3 */
  grid-column: 1 / -1;       /* line 1 to the LAST line → full width */
}
```

**Named template areas** — the most readable way to lay out a page. You "draw" the layout as a string map, then assign each item a name.

```css
.page {
  display: grid;
  grid-template-columns: 200px 1fr;
  grid-template-rows: auto 1fr auto;
  grid-template-areas:
    "header  header"     /* the same name spanning two cells = it spans them */
    "sidebar main"
    "footer  footer";
  min-height: 100dvh;
  gap: 1rem;
}
.page > header  { grid-area: header; }
.page > .side   { grid-area: sidebar; }
.page > main    { grid-area: main; }
.page > footer  { grid-area: footer; }
/* A "." in the template = an empty cell. Rearrange the layout by editing
   ONLY the area strings — no markup changes. Brilliant for responsiveness. */
```

### 11.4 Aligning within the grid **[I]**

Grid has the same alignment vocabulary as flexbox, doubled for two axes:

```css
.grid {
  /* Align the WHOLE grid within the container if it's smaller than the container: */
  justify-content: center;   /* along the inline (column/row-of-columns) axis */
  align-content: center;     /* along the block axis */

  /* Align each ITEM within its cell (the common case): */
  justify-items: stretch;    /* inline axis: start | end | center | stretch */
  align-items: stretch;      /* block axis */
}
.cell {
  justify-self: center;      /* override for one item, inline axis */
  align-self: end;           /* override for one item, block axis */
  /* Shorthand: `place-self: center;` = align-self + justify-self */
}
/* `place-items: center;` on the container = the easiest true 2-axis centering. */
```

### 11.5 Subgrid **[A]**

**The problem:** nested grids don't know about their parent's tracks, so a card's internal rows can't align with sibling cards' rows — leading to ragged layouts. **`subgrid`** lets a nested grid *adopt its parent's track lines* on a given axis, so children align across separate grid items.

```css
.cards {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 1rem;
}
.card {
  display: grid;
  grid-row: span 3;
  /* Inherit the PARENT grid's row tracks so every card's title/body/footer
     line up across the row, regardless of content length. */
  grid-template-rows: subgrid;
}
```

> **⚡ Version note:** `subgrid` reached baseline support across all evergreen browsers and is safe in 2026. It's the cleanest solution for "make these cards' internal sections align."

### 11.6 `grid-auto-flow` and implicit tracks **[A]**

When you place more items than your explicit template defines, the **implicit grid** kicks in, creating new tracks sized by `grid-auto-rows`/`grid-auto-columns`. `grid-auto-flow: dense` lets the algorithm backfill gaps.

```css
.masonry-ish {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
  grid-auto-rows: 150px;          /* size of auto-created rows */
  grid-auto-flow: dense;          /* backfill holes left by spanning items */
  gap: 8px;
}
.tall { grid-row: span 2; }
.wide { grid-column: span 2; }
```

### 11.7 Grid vs flexbox — decision table

| Situation | Use |
|---|---|
| Row of buttons / navbar | Flexbox |
| Wrapping tag list / chips | Flexbox (`flex-wrap`) |
| Distribute items by content size in one line | Flexbox |
| Page layout (header/sidebar/main/footer) | Grid (areas) |
| Image gallery / card grid | Grid (`auto-fit` + `minmax`) |
| Aligning rows AND columns together | Grid |
| Cards whose internal sections must align | Grid + subgrid |
| Centering one thing | Either (`place-items:center` or flex center) |

---

## 12. Positioning, z-index & Stacking

`position` lets you take elements out of (or offset within) normal flow. The key concept threaded through all of it is the **containing block** — the box an offset is measured against.

### 12.1 The five `position` values **[I]**

```css
/* STATIC — the default. The element sits in normal flow; top/left/etc. are
   ignored. You rarely write this except to reset another value. */
.a { position: static; }

/* RELATIVE — stays in flow (still reserves its space) but you can NUDGE it with
   top/right/bottom/left relative to where it WOULD have been. Also makes the
   element a containing block for absolutely-positioned descendants. */
.b { position: relative; top: 10px; left: 20px; }

/* ABSOLUTE — removed from flow (no longer reserves space). Positioned relative
   to its nearest POSITIONED ancestor (the containing block). If none exists,
   relative to the initial containing block (the viewport-ish root). */
.tooltip { position: absolute; top: 100%; left: 0; }

/* FIXED — removed from flow, positioned relative to the VIEWPORT; stays put
   while the page scrolls. Classic for sticky headers/back-to-top buttons. */
.fab { position: fixed; bottom: 1rem; right: 1rem; }

/* STICKY — a hybrid: behaves like relative until you scroll past a threshold,
   then "sticks" like fixed within its scrolling container. */
.tableheader { position: sticky; top: 0; }   /* sticks to the top when scrolled */
```

### 12.2 The containing block — what offsets are relative to **[I]**

This trips people up constantly. For `position: absolute`, the offsets (`top`/`left`/…) are measured from the **nearest ancestor that is itself positioned** (`relative`, `absolute`, `fixed`, or `sticky`). If no ancestor is positioned, they're measured from the *initial containing block* (essentially the viewport).

**The fundamental pattern:** to position a child absolutely *inside* a parent, give the parent `position: relative`. This is the "positioning context" idiom — without it, the absolute child escapes to the page root.

```css
.card {
  position: relative;     /* establishes the containing block… */
}
.card .badge {
  position: absolute;     /* …so this is positioned relative to .card, */
  top: 8px; right: 8px;   /*    pinned to the card's top-right corner. */
}
```

> **⚡ Note:** `transform`, `filter`, `perspective`, `will-change`, and `contain` on an ancestor *also* establish a containing block for `position: fixed` descendants — a surprising source of "my fixed element isn't fixed to the viewport" bugs.

### 12.3 `inset` shorthand & full-cover **[I]**

```css
/* `inset` is shorthand for top/right/bottom/left (logical-aware). */
.overlay {
  position: absolute;
  inset: 0;               /* = top:0; right:0; bottom:0; left:0 → fill the parent */
}
.overlay2 { position: absolute; inset: 10px 20px; } /* block | inline */
```

### 12.4 Stacking contexts & `z-index` **[I/A]**

When boxes overlap, **`z-index`** controls the front-to-back order — higher is in front. But there are two rules people miss:

1. **`z-index` only works on positioned elements** (`position` is anything but `static`) — and on flex/grid children. On a static element it does nothing.
2. **`z-index` is scoped to a *stacking context*.** A stacking context is a self-contained layer; `z-index` values only compare *within the same context*. A child's `z-index: 9999` can never escape its parent's stacking context to sit above the parent's sibling.

**What creates a stacking context?** The root `<html>`; any positioned element with a `z-index` other than `auto`; `opacity` less than 1; `transform`, `filter`, `perspective`, `will-change`, `mix-blend-mode`, `isolation: isolate`; flex/grid children with a `z-index`.

```css
.modal   { position: fixed; inset: 0; z-index: 1000; }   /* needs position! */
.backdrop{ position: fixed; inset: 0; z-index: 999; }

/* Deliberately create a stacking context to keep a component's z-indexes
   self-contained (so it can't be punched through by outside z-index). */
.widget { isolation: isolate; }
```

> **Gotcha — the #1 z-index confusion:** "I set `z-index: 99999` and it's *still* behind that other thing." Almost always the culprit is that the element is inside a parent that forms its own stacking context (e.g. a parent with `opacity: 0.99` or a `transform`). Your huge z-index only competes *inside* that parent. Fix: raise the *parent's* z-index, or avoid creating the unintended context. (See §18.)

---

## 13. Responsive Design & Container Queries

**Responsive design** means a single layout adapts to any screen — phone to ultrawide — without horizontal scrolling or tiny text. The tools are the viewport meta tag, fluid units, media queries, and (the modern leap) container queries.

### 13.1 The viewport meta tag — non-negotiable **[B]**

Without this in your HTML `<head>`, mobile browsers render at a fake ~980px width and zoom out, breaking every media query.

```html
<!-- Tells mobile browsers: use the device width, start at 100% zoom. -->
<meta name="viewport" content="width=device-width, initial-scale=1" />
```

### 13.2 Mobile-first methodology **[I]**

**Write the base (no-media-query) styles for the *smallest* screen, then add `min-width` media queries to *enhance* for larger screens.** Why mobile-first? Mobile styles are usually simpler (single column), small screens are the constrained case worth getting right first, and `min-width` queries layer additively without overrides fighting each other. The alternative (`max-width`, desktop-first) tends to pile up overrides.

```css
/* BASE: mobile. One column, full width. No media query. */
.layout { display: grid; grid-template-columns: 1fr; gap: 1rem; }

/* Enhance at ≥768px (tablet): two columns. */
@media (min-width: 768px) {
  .layout { grid-template-columns: 200px 1fr; }
}

/* Enhance at ≥1200px (desktop): cap the width and center. */
@media (min-width: 1200px) {
  .layout { max-width: 1200px; margin-inline: auto; }
}
```

### 13.3 Media queries in depth **[I]**

```css
/* Width is the most common feature. min-width = "this wide OR WIDER". */
@media (min-width: 600px) { /* … */ }
@media (max-width: 599.98px) { /* … */ }   /* "narrower than 600" */

/* ⚡ Modern RANGE syntax — clearer than min/max-width: */
@media (width >= 600px) { /* … */ }
@media (400px <= width <= 800px) { /* … */ }

/* Combine with `and`, list alternatives with a comma (= OR): */
@media (min-width: 600px) and (orientation: landscape) { /* … */ }
@media (min-width: 600px), print { /* … */ }

/* Feature queries beyond size: */
@media (orientation: portrait) { /* … */ }
@media (prefers-color-scheme: dark) { /* dark mode (§5.6) */ }
@media (prefers-reduced-motion: reduce) { /* a11y (§17) */ }
@media (hover: hover) and (pointer: fine) { /* mouse users — show hover affordances */ }
@media (min-resolution: 2dppx) { /* retina / high-DPI screens */ }
@media print { /* print stylesheet */ }
```

**Choosing breakpoints:** Don't memorize device pixel sizes — they change every year. **Set breakpoints where *your content* breaks.** Widen the browser; the moment the layout looks awkward (line lengths too long, a row too cramped), add a breakpoint there. Common starting points are ~600px, ~900px, ~1200px, but they should serve the design, not specific phones.

### 13.4 `@supports` — feature detection **[I]**

Apply CSS only if the browser supports a feature — useful while a feature is still rolling out.

```css
@supports (display: grid) {
  .layout { display: grid; }
}
@supports not (aspect-ratio: 1) {
  .box { /* fallback for browsers lacking aspect-ratio */ }
}
@supports selector(:has(*)) {       /* test selector support */
  .x:has(img) { /* … */ }
}
```

### 13.5 Container queries — `@container` **[I/A]**

**The fundamental limitation of media queries:** they only know the **viewport** size. But a *component* (say, a card) might appear in a wide main column on one page and a narrow sidebar on another. With media queries the card can't know which — it only knows the screen width. So you end up coupling component styles to page-level breakpoints, which is fragile.

**Container queries let an element respond to the size of its *container*, not the viewport.** Now a card can lay itself out horizontally when it has room and stack vertically when it's cramped — *anywhere* it's placed, automatically. This is the right tool for reusable components.

```css
/* 1) Mark an element as a query CONTAINER. Descendants can query its size.
      `inline-size` = query its width (the usual choice). */
.card-wrapper {
  container-type: inline-size;
  container-name: card;          /* optional name to target a specific container */
}

/* 2) Query the container's size. This card stacks by default… */
.card { display: grid; gap: 1rem; }

/* …and goes side-by-side when ITS CONTAINER is at least 400px wide,
   regardless of the viewport. Drop the name to query the nearest container. */
@container card (min-width: 400px) {
  .card { grid-template-columns: 150px 1fr; }
}
```

```css
/* Shorthand to set type+name at once: */
.sidebar, .main { container: layout / inline-size; }
```

> **⚡ Container query units:** inside a container you can size things relative to the container — `cqw` (1% of container width), `cqh`, `cqi` (inline), `cqb` (block), `cqmin`, `cqmax`. e.g. `font-size: 5cqw` scales text to the container, not the viewport.

> **⚡ Version note:** `@container` (size queries) is baseline across evergreen browsers in 2026 and is the recommended approach for component-level responsiveness. Newer **style queries** (`@container style(--theme: dark)`) are emerging for querying a container's custom-property values — check support before relying on them.

### 13.6 Responsive images (CSS side) **[B/I]**

```css
img { max-width: 100%; height: auto; display: block; }  /* never overflow; keep ratio */
/* (For art direction / resolution switching use HTML <picture>/srcset — see §17.) */
```

---

## 14. Transitions, Animations & Transforms

Motion adds polish and communicates change. CSS gives you two declarative tools — **transitions** (animate *between two states*) and **animations** (multi-step, looping timelines) — plus **transforms** (move/scale/rotate without affecting layout).

### 14.1 Transforms **[I]**

`transform` visually moves/scales/rotates/skews an element **without affecting layout** — surrounding elements don't shift, and it's cheap to animate (compositor-friendly). The transform applies around the element's `transform-origin` (default: its center).

```css
.box {
  transform: translate(20px, 10px);   /* move right 20, down 10 (x, y) */
  transform: translateX(50%);         /* % is relative to the ELEMENT's own size */
  transform: scale(1.2);              /* 120% size; scaleX/scaleY for one axis */
  transform: rotate(15deg);
  transform: skew(10deg, 0);

  /* Combine — applied RIGHT-TO-LEFT (rotate happens, then translate). */
  transform: translateY(-4px) scale(1.05) rotate(2deg);

  transform-origin: top left;         /* pivot point for scale/rotate */
}

/* 3D transforms */
.card3d {
  transform: perspective(800px) rotateY(20deg);
  transform-style: preserve-3d;       /* let children live in 3D space */
  backface-visibility: hidden;        /* hide the back when flipped */
}

/* ⚡ Modern individual transform properties (animate independently, clearer): */
.x { translate: 0 -4px; scale: 1.05; rotate: 2deg; }
```

> **Why `translate` for movement instead of `top`/`left`?** Changing `top`/`left` triggers **layout** (reflow) every frame — expensive and janky. `transform: translate()` is handled by the **compositor** on the GPU, so it animates at 60fps without reflow. Always animate position with `transform`, not box offsets.

### 14.2 Transitions **[I]**

A **transition** smoothly interpolates a property when its value changes (e.g. on `:hover`, or when a class is toggled by JS). You declare *which* properties, *how long*, and the *easing*.

```css
.btn {
  background: royalblue;
  transform: scale(1);
  /* property | duration | timing-function | delay */
  transition: background 200ms ease, transform 150ms ease-out;
  /* Or transition: all 200ms ease;  ← convenient but can animate unintended props */
}
.btn:hover {
  background: navy;        /* the change is animated over 200ms */
  transform: scale(1.05);  /* animated over 150ms */
}

/* Longhand form: */
.x {
  transition-property: opacity, transform;
  transition-duration: 300ms;
  transition-timing-function: cubic-bezier(0.2, 0, 0, 1);
  transition-delay: 0ms;
}
```

**Timing functions (easing)** shape the *rate* of change: `linear` (constant), `ease` (default, slow-fast-slow), `ease-in` (slow start), `ease-out` (slow end — best for things entering), `ease-in-out`, `steps(n)` (jump in discrete steps — for sprite animations), and `cubic-bezier(...)` for custom curves.

> **Gotcha:** you can't transition `display` (it's discrete) the old way — so fade-out-then-hide needed JS. **⚡ 2026:** `transition-behavior: allow-discrete` plus `@starting-style` now make it possible to animate to/from `display: none` purely in CSS.

### 14.3 Keyframe animations **[I/A]**

For multi-step, looping, or self-starting motion, define a `@keyframes` timeline and attach it with `animation`.

```css
/* 1) Define the timeline. `from`/`to` = 0%/100%; add any % steps in between. */
@keyframes fade-in-up {
  from { opacity: 0; transform: translateY(20px); }
  to   { opacity: 1; transform: translateY(0); }
}
@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50%      { transform: scale(1.1); }
}

/* 2) Apply it. */
.modal {
  /* name | duration | timing | delay | iteration-count | direction | fill-mode */
  animation: fade-in-up 300ms ease-out;
}
.heartbeat {
  animation: pulse 1s ease-in-out infinite;   /* loops forever */
}

/* Longhand controls: */
.spinner {
  animation-name: spin;
  animation-duration: 1s;
  animation-timing-function: linear;
  animation-iteration-count: infinite;     /* a number or `infinite` */
  animation-direction: alternate;          /* normal | reverse | alternate | alternate-reverse */
  animation-fill-mode: forwards;           /* keep the END state after finishing */
  animation-delay: 0.2s;
  animation-play-state: paused;            /* pause/run (toggle via class/JS) */
}
@keyframes spin { to { rotate: 360deg; } }
```

`animation-fill-mode: forwards` is a common need — without it, the element snaps back to its pre-animation styles when the animation ends.

### 14.4 `will-change` — a hint, not a fix **[A]**

`will-change` tells the browser "I'm about to animate this property," so it can promote the element to its own GPU layer in advance. Use it **sparingly and temporarily** — over-using it wastes memory and can *hurt* performance. Don't slap it on everything.

```css
.card:hover { will-change: transform; }   /* hint just before the animation */
/* Remove it when the animation is done (often via JS) to free the layer. */
```

### 14.5 Respect motion preferences **[I]**

Always wrap large/looping motion so users who get motion sickness can opt out (see §17).

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

### 14.6 Performance: the compositor-friendly properties **[A]**

The browser can animate some properties **on the compositor thread (GPU) without layout or paint**, hitting a smooth 60fps. These are essentially **`transform`** and **`opacity`** (and `filter` to a degree). Animating layout properties (`width`, `height`, `top`, `margin`, `left`) forces **reflow** every frame and is the usual cause of janky animation.

**Rule of thumb:** animate `transform` and `opacity`; avoid animating anything that changes geometry. Need to "grow" a box smoothly? Animate `transform: scale()` rather than `width`/`height`.

---

## 15. Modern CSS Features

A grab-bag of the features that define writing CSS *today*. Several were previewed in earlier sections; here they're collected with detail.

### 15.1 Native CSS nesting **[I]**

You can now nest rules the way Sass let you — natively, no build step. Nesting keeps related styles together and reduces repetition of the parent selector.

```css
.card {
  padding: 1rem;
  border: 1px solid #ddd;

  /* Nested rule: the `&` is the parent selector (the .card). */
  & .title { font-weight: 700; }      /* = .card .title */

  /* `&` lets you attach states/pseudo-classes to the parent itself. */
  &:hover { border-color: royalblue; }/* = .card:hover */
  &.is-active { background: #eef; }   /* = .card.is-active (compound, no space) */

  /* You can nest media/container queries too: */
  @media (min-width: 600px) {
    flex-direction: row;              /* applies to .card at ≥600px */
  }
}
```

> **⚡ Version note & gotcha:** Native nesting is baseline in 2026. Early implementations required `&` before a *type* selector; current browsers allow `.card { p { … } }` directly, but writing `& p` is the safest, clearest habit. Also note nesting does **not** reduce specificity (unlike `:where()`); deeply nested selectors still pile up specificity, so don't over-nest.

### 15.2 `:has()`, `:is()`, `:where()` (recap) **[I/A]**

Covered in §2.7. Quick reminders: `:is()` deduplicates and takes max specificity; `:where()` is the zero-specificity version (ideal for resets); `:has()` is the relational/parent selector. Together they enable styling patterns that previously required JavaScript (e.g. "style a form group when its input is invalid," "lay a card out differently when it contains an image").

### 15.3 Cascade layers (recap) **[A]**

Covered in §3.6. The headline: `@layer order beats specificity`, giving you predictable, conflict-free control over base/components/utilities/vendor CSS without `!important`.

### 15.4 Logical properties **[I]**

Physical properties (`margin-left`, `top`, `width`) assume left-to-right, top-to-bottom text. **Logical properties** describe space relative to the **writing mode and direction** — `inline` is the text direction (horizontal in English), `block` is the stacking direction (vertical). In an Arabic (RTL) or vertical-Japanese context, the same logical CSS just works, mirroring automatically. For internationalized sites this is the correct default.

```css
.box {
  margin-inline: auto;        /* left+right in LTR; auto-centers a block */
  margin-block: 1rem;         /* top+bottom */
  margin-inline-start: 1rem;  /* "left" in LTR, "right" in RTL — flips automatically */
  padding-block: 0.5rem 1rem; /* block-start block-end */
  border-inline-start: 3px solid royalblue;  /* a leading accent bar that flips for RTL */

  inset-inline-start: 0;      /* logical version of left/right (with position) */
  inline-size: 50%;           /* logical `width` */
  block-size: 200px;          /* logical `height` */
  text-align: start;          /* `start`/`end` instead of left/right */
}
```

**Mapping cheat:** `inline-size`↔width, `block-size`↔height, `inset-block-start`↔top, `inset-inline-start`↔left(LTR). Prefer logical properties in new code; mix with physical only when you truly mean a fixed physical side.

### 15.5 `gap` everywhere **[I]**

`gap` (and `row-gap`/`column-gap`) sets spacing *between* items. Originally grid-only, it now works in **flexbox** and even multi-column layouts. It replaces the old "margin on every item except the last" dance — no edge margins, no `:last-child` hacks.

```css
.row  { display: flex; gap: 1rem; }          /* even spacing, no margins needed */
.grid { display: grid; gap: 1rem 2rem; }     /* row-gap column-gap */
```

### 15.6 `aspect-ratio` **[I]**

Lock a box's width-to-height ratio without the old padding-hack. The browser computes the missing dimension.

```css
.video   { aspect-ratio: 16 / 9; width: 100%; }   /* responsive 16:9 box */
.avatar  { aspect-ratio: 1; width: 64px; }         /* a perfect square */
.thumb img { aspect-ratio: 4 / 3; object-fit: cover; width: 100%; }
```

`object-fit: cover` (paired with `aspect-ratio`) makes an `<img>` fill its box and crop rather than distort — the image equivalent of `background-size: cover`.

### 15.7 Scroll-snap **[I/A]**

Create carousels and "snap-to-section" scrolling declaratively.

```css
.carousel {
  display: flex;
  overflow-x: auto;
  scroll-snap-type: x mandatory;   /* snap on the x-axis; `mandatory` = always land on a slide */
  gap: 1rem;
}
.carousel > .slide {
  flex: 0 0 100%;                  /* one slide per viewport width */
  scroll-snap-align: center;       /* the point that aligns to the snap position */
}
/* For full-page section scrolling: */
html { scroll-snap-type: y proximity; scroll-behavior: smooth; }
section { scroll-snap-align: start; }
```

### 15.8 Other modern niceties **[I/A]**

```css
/* object-fit / object-position — fit replaced elements (img/video) into a box */
img { width: 100%; height: 200px; object-fit: cover; object-position: top; }

/* clip-path — clip an element to a shape */
.hex { clip-path: polygon(50% 0, 100% 25%, 100% 75%, 50% 100%, 0 75%, 0 25%); }

/* filter / backdrop-filter — visual effects; backdrop blurs what's BEHIND. */
.glass { backdrop-filter: blur(10px); background: rgb(255 255 255 / .6); }
.muted { filter: grayscale(1) blur(2px); }

/* accent-color — theme native checkboxes/radios/range with one line */
:root { accent-color: var(--brand); }

/* scrollbar-gutter — reserve space so layout doesn't shift when scrollbars appear */
.scroll-area { scrollbar-gutter: stable; }

/* :focus-within — style a container when ANY descendant is focused */
.field:focus-within { outline: 2px solid royalblue; }

/* @starting-style + transition-behavior — animate entry & to/from display:none */
.popover {
  transition: opacity 200ms, display 200ms allow-discrete;
  opacity: 1;
}
@starting-style { .popover { opacity: 0; } }   /* the state to animate FROM on first render */
```

> **⚡ Version note:** `@starting-style`, `transition-behavior: allow-discrete`, `backdrop-filter`, and `scrollbar-gutter` are all recent. Most are baseline in 2026 but verify on caniuse for your support targets, and provide graceful fallbacks.

---

## 16. Architecture & Methodology

Small projects survive with ad-hoc CSS. Large ones need *conventions* — without them you drown in specificity conflicts and dead code. Here are the approaches that scale.

### 16.1 The core problem: specificity & global scope **[I]**

All CSS is global by default, and the cascade means any rule can affect any element. As a codebase grows, two failure modes appear: (1) **specificity wars** (overriding `#x .y .z` requires ever-more-specific selectors or `!important`), and (2) **fear of deletion** (you can't tell what a rule affects, so nothing gets removed). The methodologies below all attack these by keeping selectors *flat* (low, equal specificity) and *scoped by name*.

### 16.2 BEM — Block, Element, Modifier **[I]**

**BEM** is a naming convention (not a tool) that encodes structure into class names so every selector is a single, flat class — equal specificity, self-documenting, easy to find.

- **Block** — a standalone component: `.card`, `.menu`, `.button`.
- **Element** — a part of a block, joined with `__`: `.card__title`, `.menu__item`.
- **Modifier** — a variant/state, joined with `--`: `.card--featured`, `.button--large`.

```html
<article class="card card--featured">
  <h2 class="card__title">Title</h2>
  <p class="card__body">…</p>
  <button class="card__btn card__btn--primary">Read</button>
</article>
```

```css
/* Every selector is ONE class → specificity (0,1,0) everywhere. No wars. */
.card            { border: 1px solid #ddd; border-radius: 8px; }
.card--featured  { border-color: gold; }       /* modifier */
.card__title     { font-size: 1.25rem; }        /* element */
.card__btn       { padding: .5rem 1rem; }
.card__btn--primary { background: royalblue; color: white; }
```

The verbosity is the point: `card__btn--primary` tells you exactly what it is and where it belongs, and you can delete `.card` confidently knowing all its parts are namespaced.

### 16.3 Organizing files & the cascade order **[I]**

A common, scalable structure (often called ITCSS — inverted-triangle, from generic to specific):

```
1. settings    — design tokens (custom properties): colors, spacing, type scale
2. tools        — (in Sass) mixins/functions
3. generic      — reset / normalize, box-sizing
4. elements     — bare element defaults (h1, a, p) — no classes
5. layout/objects — structural patterns (.container, .grid)
6. components   — .card, .button, .navbar (most of your CSS)
7. utilities    — single-purpose helpers (.text-center, .mt-4) — override everything
```

Map these directly onto **cascade layers** so order is explicit and immune to source-order accidents:

```css
@layer settings, generic, elements, layout, components, utilities;
```

### 16.4 Utility-first vs component-based — and Tailwind **[I]**

Two philosophies:

- **Component-based** (BEM, plain CSS): one meaningful class per component; styles live in CSS. Pros: semantic markup, styles reusable across frameworks. Cons: naming overhead, context-switching between HTML and CSS, dead-CSS accumulation.
- **Utility-first** (Tailwind): compose many tiny single-purpose classes in the HTML (`class="flex items-center gap-4 p-4 rounded-lg"`). Pros: no naming, no dead CSS (you only ship used utilities), styling stays in the markup. Cons: verbose class lists, a learning curve, needs a build step.

Neither is "correct" — many teams blend them (utilities for spacing/layout, components for complex widgets, often extracting repeated utility clusters into a component). **This library has a dedicated `TAILWIND_CHEATSHEET.md`** if you want the utility-first approach; the present guide is the foundation either way — Tailwind utilities are just thin wrappers over the real CSS properties taught here, so understanding this guide makes Tailwind obvious.

### 16.5 Avoiding specificity wars — practical rules **[I]**

- Style with **classes**, not IDs or element selectors (keep specificity flat at `(0,1,0)`).
- Avoid deep descendant selectors (`.nav ul li a span`) — they're fragile and specific.
- Use **cascade layers** to control order instead of escalating specificity.
- Use **`:where()`** for resets/base styles so they're trivially overridable.
- Reserve **`!important`** for genuine utilities or third-party overrides only.
- Keep components **self-contained**; don't reach into another component's internals from the outside.

---

## 17. Accessibility & Best Practices

CSS choices have a direct impact on whether people can actually *use* your site — keyboard users, screen-reader users, people sensitive to motion, people with low vision. A few habits cover most of it.

### 17.1 Visible focus indicators **[I]**

Keyboard and switch users navigate by focus. If you remove focus outlines without replacing them, your site becomes unusable for them. Use `:focus-visible` so the ring shows for keyboard users without cluttering mouse clicks.

```css
/* GOOD: a clear, custom focus ring only when it's appropriate. */
:focus-visible {
  outline: 3px solid royalblue;
  outline-offset: 2px;
  border-radius: 2px;
}
/* NEVER ship this with nothing to replace it: */
/* *:focus { outline: none; }  ← removes keyboard accessibility */
```

### 17.2 `prefers-reduced-motion` **[I]**

Some users experience nausea/vertigo from motion. Honor their OS setting by disabling or toning down non-essential animation (full snippet in §14.5). Keep *essential* feedback (e.g. a quick fade confirming an action) but kill large parallax/auto-playing motion.

### 17.3 Color contrast **[I]**

Text must contrast enough against its background to be readable (WCAG: 4.5:1 for normal text, 3:1 for large text/UI). Don't rely on color alone to convey meaning (e.g. error states need an icon/text, not just red). **`oklch()` helps** here: equal-lightness colors have similar visual weight, and adjusting only the L channel predictably changes contrast. Check pairs in DevTools (the color picker shows a contrast ratio).

```css
/* Don't convey state with color ALONE — add a non-color cue. */
.error { color: #b00020; }
.error::before { content: "⚠ "; }   /* also signals via icon/text */
```

### 17.4 Respect user preferences & don't disable zoom **[I]**

- Use **`rem`** for font sizes so the user's base-size preference is honored (§5.2).
- Never set `user-scalable=no` or `maximum-scale=1` in the viewport meta — it blocks pinch-zoom for low-vision users.
- Support **dark mode** via `prefers-color-scheme` (§5.6).
- Use `@media (prefers-contrast: more)` to offer a higher-contrast variant where useful.

### 17.5 Responsive & accessible images **[I]**

CSS keeps images fluid (`max-width:100%; height:auto`), but resolution/art-direction switching is HTML's job — pair them:

```html
<!-- srcset lets the browser pick the right file for the screen's density/size. -->
<img src="photo-800.jpg"
     srcset="photo-400.jpg 400w, photo-800.jpg 800w, photo-1600.jpg 1600w"
     sizes="(min-width: 800px) 50vw, 100vw"
     alt="A meaningful description"      <!-- alt is REQUIRED for screen readers -->
     loading="lazy" decoding="async" />
```

```css
img { max-width: 100%; height: auto; }   /* the CSS half: never overflow, keep ratio */
```

### 17.6 General best practices checklist **[I]**

- A modern reset (box-sizing, remove default margins) at the top, ideally in a `reset` layer.
- Design **tokens** as custom properties on `:root` (colors, spacing, radii, type scale).
- **Mobile-first**, content-driven breakpoints; prefer **container queries** for components.
- Animate only **`transform`/`opacity`**; respect `prefers-reduced-motion`.
- Keep selectors **flat** (classes), control order with **layers**, avoid `!important`.
- Use **logical properties** for i18n-readiness.
- Constrain line length (`max-width: ~65ch`) for readability.

---

## 18. Gotchas

A consolidated list of the traps that cost everyone hours. Each is explained earlier; this is the quick-reference.

**1. Margin collapsing (§4.4).** Vertical margins between blocks collapse to the larger value, not the sum — and a child's top margin can escape its parent. *Fix:* use `gap` in flex/grid, or add `padding`/`border`/`display: flow-root` to the parent.

**2. `box-sizing` surprises (§4.2).** Without `box-sizing: border-box`, `width` excludes padding/border, so `width:50%` + padding overflows. *Fix:* the global `*{box-sizing:border-box}` reset.

**3. `z-index` does nothing (§12.4).** `z-index` is ignored on `position: static` elements. *Fix:* set `position: relative` (or use a flex/grid child).

**4. `z-index` "trapped" in a stacking context (§12.4).** A huge `z-index` can't escape a parent that forms its own stacking context (caused by `opacity < 1`, `transform`, `filter`, `will-change`…). *Fix:* raise the *parent's* z-index, or avoid the unintended context (`isolation: isolate` deliberately, or remove the stray `opacity`/`transform`).

**5. Percentage heights don't work (§4.5).** `height: 100%` needs the parent to have an explicit height. *Fix:* set height up the chain, use `100dvh`, or switch to flex/grid.

**6. Flex item won't shrink below its content / overflows (`min-width: auto`).** Flex (and grid) items have an implicit **`min-width: auto`** = their content size, so long text or a wide child can stop an item from shrinking and blow out the layout. *Fix:* set `min-width: 0` (or `overflow: hidden`) on the flex item.

```css
.flex-item { min-width: 0; }   /* allow shrinking below content size (e.g. for text-overflow) */
```

**7. Specificity surprises (§3.2).** A single `#id` beats any stack of classes; `!important` beats specificity entirely; inline styles beat selectors. *Fix:* style with classes, use cascade layers, avoid IDs/`!important` for styling.

**8. `transform` creates a containing block & stacking context (§12.2/§12.4).** A parent `transform` makes `position: fixed` children fix to *it*, not the viewport, and creates a stacking context. *Surprising* but by design.

**9. `100vh` is too tall on mobile (§5.3).** Mobile toolbars make `100vh` overflow. *Fix:* `100dvh`.

**10. `opacity` fades children too (§6.4).** You can't make a child fully opaque inside a parent with `opacity < 1`. *Fix:* use an `rgb(... / a)` alpha on the specific property instead.

**11. Collapsed-margin / negative-margin layout drift, and `gap` not collapsing.** `gap` does *not* collapse and adds no edge spacing — unlike margins — which is usually what you want; just remember they behave differently.

**12. `transition: all` animates unintended properties.** Listing `all` can animate layout properties you didn't mean to (causing jank or flashes). *Fix:* name the properties explicitly.

**13. Forgetting the viewport meta tag (§13.1).** Media queries "don't work" on mobile because the page renders at ~980px. *Fix:* add the `<meta name="viewport">` tag.

**14. `::before`/`::after` invisible without `content`.** They won't render at all unless `content` is set (even `content: ""`).

**15. Over-nesting (native nesting/§15.1) inflates specificity.** Unlike `:where()`, nesting *adds* specificity. Keep nesting shallow.

---

## 19. Study Path & Build-to-Learn Projects

**Suggested order:** §1–2 (syntax & selectors) → §3 (cascade — slow down here, it's the keystone) → §4–5 (box model & units) → §9 (display/flow) → §10–11 (flexbox & grid — the layout core) → §6–8 (color, type, decoration) → §12–13 (positioning & responsive) → §14 (motion) → §15–17 (modern features, architecture, a11y) → §18 (gotchas, re-read after each project).

**Build these, in order, to cement everything:**

1. **A card component** — image, title, body, button. Practice the box model, `border-radius`, `box-shadow`, custom properties for spacing, a hover transition, and BEM naming. *Then* make it a **container query** component (§13.5) so it reflows from stacked to side-by-side based on its own width — drop it in a wide column and a narrow sidebar and watch it adapt. Exercises §4, §6–8, §14, §15.6, §16.

2. **A responsive navigation bar** — logo left, links right (auto-margin trick), collapsing to a stacked/menu layout on small screens via a media query. Add `:focus-visible` styles and ensure keyboard usability. Exercises §10, §13, §17.

3. **A full landing-page layout (grid + flexbox)** — page skeleton with `grid-template-areas` (header / hero / features / footer); inside, a feature section using `repeat(auto-fit, minmax(250px, 1fr))`; each card's internals laid out with flexbox; a sticky header (`position: sticky`). This is the canonical "grid for the page, flex for the components" exercise. Exercises §9–12, §13.

4. **A dark-mode theme with custom properties** — define a token palette in `oklch()` on `:root`, a `[data-theme="dark"]` override, *and* automatic `prefers-color-scheme` support; a JS toggle that flips the attribute and persists the choice. Bonus: derive hover/tint colors with `color-mix()`. Exercises §5.6, §6.2–6.3, §17.

5. **(Stretch) A small design system** — organize everything with cascade layers (`@layer reset, base, components, utilities`), logical properties throughout for RTL-readiness, fluid type with `clamp()`, a `prefers-reduced-motion` guard, and a couple of `:has()`-driven interactions (e.g. a form group that highlights when its input is invalid). This pulls together §3.6, §5.5, §14.5, §15, §16.

**Next steps after this guide:** a CSS framework or utility system (see `TAILWIND_CHEATSHEET.md`), Sass/PostCSS for tooling-level conveniences (though native nesting/variables/layers cover much of what Sass used to), CSS-in-JS or CSS Modules for component-scoped styles in app frameworks (see the React/Next guides in this library), and animation libraries (see `MOTION_ANIMATION_GUIDE.md`) when declarative CSS isn't enough.

---

*Part of the offline developer study library. Written for modern CSS (nesting, container queries, `:has()`, cascade layers, subgrid, `oklch()`, logical properties) as of 2026. Confirm fast-moving features against MDN (developer.mozilla.org) and caniuse.com.*
