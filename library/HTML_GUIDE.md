# HTML & HTML5 — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I've never written a web page" to "I write clean, semantic, accessible, performant HTML" — without an internet connection. HTML is the *foundation* of every web page and almost every web app; you cannot skip it. Every concept here is explained in prose first (what it is, why it exists, when to use it, how to use it) and then shown with heavily commented, runnable markup. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets the **HTML Living Standard as of 2026**. "HTML5" is no longer a fixed, versioned spec with a number that increments — since 2019 the WHATWG (Web Hypertext Application Technology Working Group) **Living Standard** is *the* definition of HTML, continuously updated. So when people say "HTML5" today they really mean "modern HTML." Things this guide assumes are available in all current evergreen browsers (Chrome, Edge, Firefox, Safari):
> - **Semantic structural elements** (`<header>`, `<nav>`, `<main>`, `<article>`, `<section>`, `<aside>`, `<footer>`) — standard for over a decade.
> - **Modern form controls** (`type="email"`, `date`, `color`, `range`, etc.) and **native client-side validation** (`required`, `pattern`, `min`/`max`).
> - **Native lazy-loading** of images and iframes (`loading="lazy"`) — broadly supported.
> - **`<dialog>` element** (modal/non-modal dialogs with a top layer) — baseline across browsers since 2022.
> - **The `popover` attribute** — a declarative way to build tooltips, menus, and disclosure UI without JavaScript, baseline across browsers since 2024. Flagged with **⚡ Version note** where relevant.
> - **Responsive images** (`srcset`/`sizes`, `<picture>`), **`<details>`/`<summary>`**, **`<template>`**, **web components** (`<slot>`, custom elements), `contenteditable`, and the drag-and-drop attributes.
>
> Where behaviour is newer or version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**, so any OS-specific notes (file paths, default browsers) are called out. HTML is about *structure and meaning*; **CSS** handles presentation and **JavaScript** handles behaviour — this guide cross-references the **CSS_GUIDE.md** and **JAVASCRIPT_GUIDE.md** in this library but stays self-contained. Confirm exact APIs at the WHATWG HTML Living Standard and MDN.

---

## Table of Contents

1. [What HTML Is & How the Browser Uses It](#1-what-html-is--how-the-browser-uses-it) **[B]**
2. [Document Structure & Boilerplate](#2-document-structure--boilerplate) **[B]**
3. [Syntax, Tags, Attributes & Rules](#3-syntax-tags-attributes--rules) **[B]**
4. [Text Content & Semantics](#4-text-content--semantics) **[B]**
5. [Links & Navigation](#5-links--navigation) **[B]**
6. [Images & Media](#6-images--media) **[B/I]**
7. [Semantic HTML5 & Document Landmarks](#7-semantic-html5--document-landmarks) **[I]**
8. [Tables](#8-tables) **[I]**
9. [Forms — The Big One](#9-forms--the-big-one) **[I]**
10. [Metadata & the `<head>` in Depth](#10-metadata--the-head-in-depth) **[I]**
11. [Accessibility (a11y)](#11-accessibility-a11y) **[I/A]**
12. [Interactive & Modern Elements](#12-interactive--modern-elements) **[I/A]**
13. [HTML5 JavaScript APIs Overview](#13-html5-javascript-apis-overview) **[A]**
14. [SEO & Performance Basics](#14-seo--performance-basics) **[I]**
15. [Best Practices & Validation](#15-best-practices--validation) **[I]**
16. [Gotchas](#16-gotchas) **[B/I/A]**
17. [Study Path & Build-to-Learn Projects](#17-study-path--build-to-learn-projects)

---

## 1. What HTML Is & How the Browser Uses It

### 1.1 What HTML actually is **[B]**

**HTML** stands for **HyperText Markup Language**.

- **HyperText** = text with **links** to other text/documents. The "hyper" is the ability to jump from one document to another — the core idea of the web.
- **Markup** = you take plain text and **mark it up** by wrapping pieces of it in **tags** that say what each piece *means*: "this is a heading," "this is a paragraph," "this is a link." HTML is *not* a programming language — there are no variables, loops, or conditionals. It is a **declarative markup language**: you describe *what* the content is, and the browser decides how to render it.
- **Language** = it has a defined vocabulary (the set of legal elements) and grammar (rules about how they nest).

The mental model: **HTML describes the structure and meaning of content.** It answers "what is this content?" It does *not* (and should not) answer "what colour is it?" (that's **CSS** — see CSS_GUIDE.md) or "what happens when I click it?" (that's **JavaScript** — see JAVASCRIPT_GUIDE.md). Keeping these three concerns separate is one of the oldest and most important best practices in web development (covered in §15).

### 1.2 The three layers of a web page

Every web page is built from up to three layers, and it is worth fixing this picture in your head from the start:

| Layer | Language | Responsibility | Example |
|-------|----------|----------------|---------|
| **Structure / content** | **HTML** | *What* the content is and means | "This is a navigation menu containing three links." |
| **Presentation** | **CSS** | *How* it looks | "Make the menu horizontal, blue, with 16px gaps." |
| **Behaviour** | **JavaScript** | *What it does* when interacted with | "When a link is clicked, smoothly scroll to a section." |

A page can work with HTML alone (it'll just look plain). It can be styled with CSS. It can be made interactive with JS. The principle that the content should be usable even before CSS/JS load is called **progressive enhancement** (§15).

### 1.3 The DOM — HTML's relationship to the page in memory **[B/I]**

This is a concept beginners often miss, and it matters a lot once you touch JavaScript.

When the browser receives your HTML *text*, it **parses** it (reads it character by character) and builds an in-memory tree structure called the **DOM** — the **Document Object Model**. Every tag becomes a **node** (specifically an *element node*); the text inside becomes *text nodes*; attributes become properties on element nodes. Nested tags become parent/child relationships in the tree.

So this HTML:

```html
<body>
  <h1>Hello</h1>
  <p>World</p>
</body>
```

becomes this tree in the browser's memory:

```text
document
└── body
    ├── h1
    │   └── (text) "Hello"
    └── p
        └── (text) "World"
```

Key points:

- The HTML *file* is just text. The **DOM is the live, in-memory representation** the browser actually renders and that JavaScript manipulates. When JS does `document.querySelector('h1').textContent = 'Hi'`, it changes the DOM (and thus the screen), **not** your `.html` file on disk.
- The browser is forgiving: if your HTML is slightly malformed (unclosed tags, etc.), the parser uses standardized **error-recovery rules** to build a DOM anyway. This is why a broken-looking page still shows *something*. But relying on this is fragile — write valid HTML.
- CSS targets DOM nodes (via selectors); JS reads and mutates DOM nodes. So the quality of your HTML structure directly affects how easy your CSS and JS are to write.

### 1.4 How the browser turns HTML into pixels (the rendering pipeline, briefly) **[I]**

You don't need to memorize this, but knowing the rough flow explains many performance tips later (§14):

1. **Parse HTML → DOM tree.** The parser reads top-to-bottom.
2. **Parse CSS → CSSOM tree** (CSS Object Model). Stylesheets are fetched and parsed.
3. **Render tree.** DOM + CSSOM are combined into a tree of *what is actually visible and how it's styled* (elements with `display:none` are excluded).
4. **Layout (a.k.a. reflow).** The browser computes the exact size and position of every box.
5. **Paint.** Pixels are filled in.
6. **Composite.** Layers are combined and shown on screen.

A crucial consequence: by default, **`<script>` tags block parsing** — when the parser hits a `<script>` it stops building the DOM, downloads (if external) and runs the script, *then* continues. Likewise, CSS in the `<head>` is **render-blocking**. This is why `<script>` placement, `defer`/`async`, and stylesheet handling matter for performance (covered in §10 and §14).

### 1.5 The Living Standard — what "HTML5" means in 2026 **[B/I]**

A short history so the version landscape makes sense:

- **HTML 4.01** (1999) and **XHTML** (a stricter, XML-based variant) dominated the early 2000s.
- **HTML5** (the original W3C "version 5") brought semantic elements, native audio/video, canvas, new form types, and many JS APIs. It reached "Recommendation" status in 2014.
- Since **2019**, the W3C and WHATWG agreed that the **WHATWG HTML Living Standard** is the single source of truth. It is *"living"* — there is no HTML6 coming; the spec is continuously revised. New features ship to browsers and are folded into the standard as they go.

So in 2026, when someone says "HTML5," they almost always just mean **modern HTML as defined by the Living Standard**. There is no version number to chase. What matters instead is **browser support** for a given feature — which you check on resources like *Can I Use* / *Baseline* (the cross-browser support status indicator).

### 1.6 Writing, serving, and viewing an `.html` file **[B]**

HTML files are plain text with a `.html` (or legacy `.htm`) extension. You can write them in any text editor; **VS Code** is the common choice (free, great HTML support, Emmet abbreviations, Live Preview).

**Create your first file.** Save the following as `index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>My First Page</title>
</head>
<body>
  <h1>Hello, world!</h1>
  <p>This is my first HTML page.</p>
</body>
</html>
```

**View it three ways:**

1. **Double-click the file** (or drag it into a browser). The URL bar shows a `file://` path — e.g. `file:///C:/Users/Phoniex/Documents/index.html` on Windows. This is the simplest way and works for static pages.

2. **Serve it over HTTP with a local server.** Many features (`fetch`, modules, service workers, some media) require an actual `http://` origin, not `file://`. Easiest options:

   ```bash
   # Using Python (preinstalled on many systems; see PYTHON_GUIDE.md):
   python -m http.server 8000
   #   then open http://localhost:8000  in your browser

   # Using Node.js (see NODEJS_GUIDE.md):
   npx serve            # or:  npx http-server

   # In VS Code: install the "Live Server" extension and click "Go Live"
   #   (it also auto-reloads the page when you save — very handy).
   ```

3. **Inspect it.** Press **F12** (or right-click → Inspect) to open **DevTools**. The **Elements** panel shows the *live DOM* (not your file — remember §1.3), the **Console** shows errors and lets you run JS, and the **Network** panel shows what was downloaded. DevTools is your most important learning tool — keep it open.

> **`file://` vs `http://` gotcha:** Relative links and images usually work over `file://`, but `fetch`, ES modules (`<script type="module">`), `localStorage` on some configs, and cross-file requests may be blocked by browser security. When in doubt, run a local server.

---

## 2. Document Structure & Boilerplate

Every HTML document follows the same skeleton. Let's dissect the boilerplate line by line, because each line earns its place.

```html
<!DOCTYPE html>                                    <!-- 1 -->
<html lang="en">                                   <!-- 2 -->
  <head>                                           <!-- 3 -->
    <meta charset="UTF-8" />                        <!-- 4 -->
    <meta name="viewport"
          content="width=device-width, initial-scale=1.0" />  <!-- 5 -->
    <title>Page Title — Site Name</title>          <!-- 6 -->
    <meta name="description" content="A one-sentence summary." /> <!-- 7 -->
    <link rel="stylesheet" href="styles.css" />    <!-- 8 -->
  </head>
  <body>                                            <!-- 9 -->
    <!-- All visible content goes here -->
    <h1>Visible heading</h1>
    <script src="app.js" defer></script>           <!-- 10 -->
  </body>
</html>
```

### 2.1 `<!DOCTYPE html>` — the doctype **[B]**

The very first line. It is **not** an HTML tag (note the `!` and that it has no closing tag); it is a declaration that tells the browser **"render this page in *standards mode*."**

- **Why it matters:** browsers have an old **"quirks mode"** they fall into if the doctype is missing — they then emulate buggy behaviour from 1990s browsers (different box model, etc.), and your CSS will mysteriously misbehave. The modern doctype is deliberately short and switches on the correct, standards-compliant rendering.
- **It is case-insensitive** (`<!doctype html>` is fine) and must be the first thing in the file (no content or even a blank line before it — well, whitespace before it can trigger quirks mode in some cases, so put it first).
- In old HTML4/XHTML you had long, scary doctypes with URLs. HTML5 simplified it to exactly `<!DOCTYPE html>`. That's all you ever need now.

### 2.2 `<html lang="en">` — the root element **[B]**

The single **root** element that wraps everything. Every other element is a descendant of `<html>`.

- The **`lang` attribute** declares the document's primary human language using a BCP 47 language tag (`en`, `en-US`, `fr`, `de`, `ja`, `zh-Hans`, etc.). **Always set it.** Why?
  - **Screen readers** use it to choose the correct pronunciation/voice. Without it, English read with a French voice is unintelligible.
  - **Search engines** use it for language targeting.
  - **Browser translation** offers ("Translate this page?") rely on it.
  - **CSS** can target it (`:lang(en)`), and typographic features like hyphenation depend on it.
- You can override `lang` on any element for inline language switches: `<span lang="fr">déjà vu</span>`.

### 2.3 `<head>` vs `<body>` — the two halves **[B]**

`<html>` contains exactly two children: `<head>` then `<body>`.

- **`<head>`** = **metadata** *about* the document — information for the browser, search engines, and social platforms, **not** shown in the page content area. The title (shown in the tab), character encoding, viewport settings, links to CSS, `<script>` references, SEO/social tags, favicons all live here. (Covered in depth in §10.)
- **`<body>`** = **everything the user actually sees** — headings, paragraphs, images, links, forms, etc.

A common beginner mistake is putting visible content in `<head>` (it won't show) or putting `<meta>`/`<title>` in `<body>` (browsers may move it or ignore it). Keep the division clean.

### 2.4 `<meta charset="UTF-8">` — character encoding **[B, important]**

This declares how the bytes in the file map to characters.

- **What it is:** a text file is just bytes; the *encoding* says which bytes mean which characters. **UTF-8** is the universal encoding that covers every character in every language plus emoji. It is the default and only sane choice today.
- **Why it matters:** if the declared (or guessed) encoding doesn't match how the file was saved, you get **mojibake** — garbled characters like `Ã©` instead of `é`, or `â€™` instead of `'`. Setting `charset=UTF-8` *and* saving your file as UTF-8 (VS Code does this by default) prevents this.
- **Put it first in `<head>`**, within the first 1024 bytes. The browser starts parsing assuming an encoding; declaring UTF-8 early avoids a re-parse.

### 2.5 The viewport meta — `<meta name="viewport" ...>` **[B, important]**

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
```

- **What it does:** controls how the page maps to a mobile device's screen. Without it, phones assume your page is a desktop page ~980px wide and **shrink the whole thing to fit**, leaving everything tiny and requiring pinch-zoom.
- **`width=device-width`** says "make the layout viewport the actual width of the device in CSS pixels." **`initial-scale=1.0`** says "don't zoom in or out initially."
- **Why it's essential:** modern responsive design (media queries, fluid layouts — see CSS_GUIDE.md) *depends* on this tag. Without it, your `@media (max-width: 600px)` rules effectively never fire on phones because the device reports the wrong width. **Include it on every page.**
- **Do not** add `user-scalable=no` or `maximum-scale=1` — disabling zoom is an accessibility violation that harms people with low vision.

### 2.6 `<title>` **[B]**

```html
<title>How to Make Bread — Cooking Blog</title>
```

The single most important `<head>` tag for users and SEO:

- Shows in the **browser tab** and the **bookmark name**.
- Is the **clickable headline** in search engine results.
- Is read first by **screen readers** when a page loads, and used in browser history.

Best practice: put the **specific page topic first**, then the site name, separated by `—` or `|`. Keep it under ~60 characters so search engines don't truncate it. Every page should have a **unique, descriptive** title.

### 2.7 Lines 7–10

These (description meta, stylesheet `<link>`, deferred `<script>`) are previewed here and explained fully in §10. The short version: `<meta name="description">` is the search-result snippet; `<link rel="stylesheet">` pulls in CSS; `<script defer>` loads JS without blocking parsing.

---

## 3. Syntax, Tags, Attributes & Rules

### 3.1 Anatomy of an element **[B]**

The fundamental unit of HTML is the **element**. Most elements are written as an **opening tag**, some **content**, and a **closing tag**:

```html
<p class="intro">Hello world</p>
│  │     │       │           │ └── closing tag (note the slash)
│  │     │       │           └──── element content (text and/or other elements)
│  │     │       └──────────────── attribute value (quoted)
│  │     └──────────────────────── attribute name
│  └────────────────────────────── tag name (the element type)
└───────────────────────────────── opening tag (angle brackets)
```

- A **tag** is the thing in angle brackets (`<p>` or `</p>`).
- An **element** is the whole package: opening tag + content + closing tag.
- The **closing tag** repeats the name with a leading slash: `</p>`.
- Tag names are **case-insensitive** in HTML (`<P>` works) but by overwhelming convention you write them **lowercase**. Stick to lowercase.

### 3.2 Attributes **[B]**

**Attributes** provide extra information about or configuration of an element. They live **inside the opening tag**, after the tag name, written as `name="value"`:

```html
<a href="https://example.com" target="_blank" rel="noopener">Visit</a>
<img src="cat.jpg" alt="A sleeping cat" width="400" height="300" />
<input type="email" name="email" required placeholder="you@example.com" />
```

Rules and conventions:

- **Always quote values.** Double quotes are conventional (`href="..."`). Unquoted values *can* work for simple cases but break with spaces and are error-prone — always quote.
- Attribute **order does not matter** and names are **case-insensitive** (use lowercase).
- **Boolean attributes** (like `required`, `disabled`, `checked`, `readonly`, `hidden`, `selected`, `multiple`) are *on* if present and *off* if absent. Their **value is irrelevant** — `required`, `required=""`, and `required="required"` all mean the same "true." There is **no** way to write `required="false"`; to turn it off you **remove the attribute entirely**. This trips up beginners constantly.

  ```html
  <input type="checkbox" checked />        <!-- checked -->
  <input type="checkbox" />                <!-- not checked -->
  <button disabled>Can't click</button>    <!-- disabled -->
  ```

### 3.3 Global attributes — usable on (almost) any element **[B/I]**

Some attributes work on virtually every element. These are the **global attributes**. Know these well:

| Attribute | Purpose |
|-----------|---------|
| `id="..."` | A **unique** identifier for the element (unique *per page*). Used for in-page links (`#id`), CSS (`#id`), JS (`getElementById`), and `<label for>`. |
| `class="a b c"` | One or more **space-separated** class names. The main hook for CSS and JS. Reusable (unlike `id`). |
| `style="..."` | Inline CSS. Works but **avoid** for anything beyond quick tests — keep CSS in stylesheets (§15). |
| `title="..."` | Advisory text shown as a **tooltip** on hover. Not reliably accessible (no keyboard/touch access), so don't rely on it for essential info. |
| `hidden` | Boolean; **hides** the element entirely (like `display:none`). `hidden="until-found"` is a newer value that hides but allows find-in-page/fragment navigation to reveal it. |
| `data-*="..."` | **Custom data attributes** — your own attributes for storing arbitrary data, read in JS via `element.dataset`. See below. |
| `lang="..."` | Language of this element's content (overrides `<html lang>`). |
| `dir="ltr\|rtl\|auto"` | Text direction (left-to-right, right-to-left for Arabic/Hebrew, or auto). |
| `tabindex="..."` | Controls keyboard focus order (§11). `0` = focusable in normal order; `-1` = focusable only via script; positive numbers = avoid. |
| `contenteditable` | Makes the element's content user-editable (§12). |
| `role`, `aria-*` | Accessibility semantics and state (§11). |
| `draggable="true"` | Marks an element as draggable (§12). |
| `spellcheck`, `autocapitalize`, `inputmode` | Hints for editable content / on-screen keyboards. |

**`id` rules:** must be **unique** in the document, shouldn't start with a digit historically (modern HTML is lenient but be safe), and is case-sensitive. Duplicating an `id` is invalid and breaks `getElementById`/anchor links.

**`class` vs `id`:** use **classes** for styling and grouping (reusable, many per element, many per page); use **`id`** for a single unique target (one element). Most styling should use classes.

**Custom `data-*` attributes** let you attach machine-readable data to elements without abusing other attributes:

```html
<!-- Store data on the element; JS reads it later -->
<button data-product-id="42" data-price="9.99" data-in-stock>Add to cart</button>

<script>
  const btn = document.querySelector('button');
  // The dataset API: data-product-id  ->  dataset.productId  (camelCased)
  console.log(btn.dataset.productId); // "42"  (always a string)
  console.log(btn.dataset.price);     // "9.99"
  console.log('inStock' in btn.dataset); // true (present even with no value)
</script>
```

`data-*` is the right way to bridge HTML and JS state — see JAVASCRIPT_GUIDE.md for using `dataset`.

### 3.4 Void elements (self-closing / empty elements) **[B]**

Some elements **cannot have content** and therefore have **no closing tag**. These are called **void elements** (or empty elements). They represent something self-contained:

```html
<br />          <!-- line break -->
<hr />          <!-- thematic break (horizontal rule) -->
<img src="x.jpg" alt="..." />   <!-- image -->
<input type="text" />           <!-- form control -->
<meta charset="UTF-8" />        <!-- metadata -->
<link rel="stylesheet" href="x.css" />
<source src="x.mp4" type="video/mp4" />
<track kind="captions" src="x.vtt" />
<area />, <base />, <col />, <embed />, <param />, <wbr />
```

- The trailing slash (`<br />`) is **optional in HTML**. Writing `<br>` is equally valid. The `/>` style is a habit carried over from XHTML/JSX; both are fine — pick one and be consistent. (In **JSX/React** — see REACT_19_GUIDE.md — the slash *is* required, which is why many developers write it everywhere.)
- The key point: **do not** write `</br>` or try to put content inside a void element — there is no such thing.

### 3.5 Nesting rules & the content model **[B/I]**

Elements form a tree (§1.3), so they must **nest properly** — open and close in the correct order, like balanced brackets:

```html
<!-- CORRECT: inner element fully inside outer -->
<p>This is <strong>very <em>important</em></strong> text.</p>

<!-- WRONG: tags cross / overlap. Browser will "fix" it unpredictably. -->
<p>This is <strong>very <em>important</strong></em> text.</p>
```

Beyond balanced nesting, HTML has a **content model**: each element specifies what it's *allowed* to contain. The two broad historical categories you must internalize:

- **Block-level** elements (e.g. `<div>`, `<p>`, `<h1>`–`<h6>`, `<ul>`, `<section>`, `<article>`) start on a new line and take the full available width by default. They can usually contain other block and inline elements.
- **Inline** elements (e.g. `<span>`, `<a>`, `<strong>`, `<em>`, `<img>`, `<code>`) flow within text and only take the space they need. They generally **cannot** contain block-level elements.

> **Note:** "block" vs "inline" is technically a **CSS** `display` concept now; the HTML spec uses *content categories* like "flow content" and "phrasing content." But the practical rule still holds and explains the most common nesting errors.

The classic illegal nestings that browsers silently "fix" (often badly), so **avoid them**:

```html
<!-- ILLEGAL: <p> cannot contain a block element. The browser will auto-close
     the <p> right before the <div>, splitting your structure unexpectedly. -->
<p>Text <div>more</div></p>

<!-- ILLEGAL: an <a> cannot be inside another <a>, and interactive elements
     (button, a, input) should not be nested inside each other. -->
<a href="#"><a href="#">nope</a></a>

<!-- ILLEGAL: <ul>/<ol> may only have <li> (and <script>/<template>) as direct
     children. Put a <div> inside an <li>, not between <li>s. -->
<ul><div>wrong</div><li>ok</li></ul>

<!-- ILLEGAL: a <button> cannot contain interactive content like an <input>. -->
<button><input type="text" /></button>
```

The validator (§15) catches these. When in doubt, validate.

### 3.6 Whitespace handling **[B/I]**

HTML **collapses whitespace**. Any run of spaces, tabs, and newlines in your source is rendered as a **single space**. This is usually helpful (you can indent your source freely) but surprises beginners:

```html
<p>Hello       world

   on   many lines</p>

<!-- Renders as:  "Hello world on many lines"  (all whitespace collapsed) -->
```

Consequences:

- You **cannot** add visual spacing by typing many spaces or blank lines — use **CSS** (`margin`, `padding`, `gap`) for that.
- A single space *between inline elements in the source* becomes a real, rendered space — which famously creates tiny gaps between `inline-block` elements. CSS solves this (fl/grid), not HTML.
- To force a line break inside text, use `<br>` (§4). To preserve whitespace exactly (code listings, ASCII art), use `<pre>` (§4), which is the one element that respects spaces and newlines.
- A **non-breaking space** entity `&nbsp;` produces a space that (a) is not collapsed and (b) won't break onto a new line — useful for keeping "10&nbsp;kg" together. Don't overuse it for layout.

### 3.7 Comments **[B]**

Comments are notes in the source that the browser ignores and does not render:

```html
<!-- This is a comment. It can span
     multiple lines. -->

<!-- TODO: replace this placeholder image before launch -->
<img src="placeholder.jpg" alt="" />
```

- Syntax: `<!--` to open, `-->` to close.
- **Comments are still sent to the browser** and visible in "View Source" — never put secrets, passwords, or sensitive data in HTML comments.
- You **cannot nest** comments, and you can't put `--` inside one.
- Use them to explain *why* something non-obvious is there, to section large files, or to temporarily disable markup.

### 3.8 Character references (entities) **[B/I]**

Some characters are **reserved** in HTML (they're part of the syntax) or hard to type. **Character references** (commonly called **entities**) let you write them safely. They start with `&` and end with `;`:

| You want | Write | Why |
|----------|-------|-----|
| `<` | `&lt;` | `<` starts a tag; literal `<` would be misread |
| `>` | `&gt;` | symmetry with `&lt;` (and avoids edge cases) |
| `&` | `&amp;` | `&` starts an entity; literal `&` is ambiguous |
| `"` | `&quot;` | needed inside double-quoted attribute values |
| `'` | `&#39;` or `&apos;` | apostrophe |
| (non-breaking space) | `&nbsp;` | space that won't collapse or wrap |
| `©` | `&copy;` | copyright |
| `®` | `&reg;` | registered trademark |
| `™` | `&trade;` | trademark |
| `—` | `&mdash;` | em dash |
| `–` | `&ndash;` | en dash |
| `…` | `&hellip;` | ellipsis |
| `€` `£` `¥` | `&euro;` `&pound;` `&yen;` | currency |
| any char | `&#NNNN;` / `&#xHHHH;` | numeric (decimal / hex) code point |

```html
<!-- To literally display HTML code on the page, escape the brackets: -->
<p>To make a paragraph, write <code>&lt;p&gt;...&lt;/p&gt;</code>.</p>
<!-- Renders as:  To make a paragraph, write <p>...</p>. -->

<p>Tom &amp; Jerry &copy; 2026 &mdash; all rights reserved.</p>
<!-- Renders:  Tom & Jerry © 2026 — all rights reserved. -->
```

The three you **must** escape because they're structural are **`<` `>` `&`**. Because your file is UTF-8 (§2.4), you can usually just type `©`, `€`, `—` directly; entities are mainly needed for the reserved characters and for the non-breaking space.

---

## 4. Text Content & Semantics

Now the actual content elements. The recurring theme of this section — and of HTML in general — is **semantics**: choosing the element whose *meaning* matches your content, not the one that happens to *look* right. Looks are CSS's job; meaning is HTML's job, and meaning is what assistive technology, search engines, and future maintainers rely on.

### 4.1 Headings `<h1>`–`<h6>` and the document outline **[B, important]**

Headings define the **hierarchical structure** of your content — the table of contents, essentially. There are six levels, `<h1>` (most important) through `<h6>` (least).

```html
<h1>The Complete Guide to Bread</h1>      <!-- page's main title (one per page) -->
  <h2>Ingredients</h2>                     <!-- a major section -->
    <h3>Flour types</h3>                    <!-- a subsection of Ingredients -->
    <h3>Yeast vs sourdough</h3>
  <h2>Method</h2>                           <!-- another major section -->
    <h3>Mixing</h3>
    <h3>Proving</h3>
      <h4>First rise</h4>                    <!-- sub-subsection -->
```

**Why hierarchy matters (this is not optional polish):**

- **Screen-reader users navigate by headings.** A blind user often jumps heading-to-heading to scan a page, exactly like a sighted user skims bold subtitles. If your headings are out of order or missing, navigation becomes confusing or impossible.
- **Search engines** use headings to understand page topic and structure.
- The headings form the **document outline** — a logical TOC of the page.

**Rules and best practices:**

- **Don't skip levels** when descending: go `h1 → h2 → h3`, not `h1 → h3`. (You *can* go back up freely, e.g. `h3` then a new `h2`.)
- **Use exactly one `<h1>` per page** (the main subject) as the safe, recommended convention. (The spec technically allows more in some sectioning contexts, but the once-promised "automatic outline algorithm" was **never implemented by browsers/AT**, so the practical rule is: one `<h1>`, then a sensible `h2/h3/...` hierarchy.)
- **Choose heading level by meaning, not size.** Need a smaller-looking heading? Keep the correct level for structure and shrink it with **CSS**. Never pick `<h4>` just because you want small text.

### 4.2 Paragraphs `<p>` **[B]**

The workhorse for blocks of running text.

```html
<p>This is a paragraph. It can be as long as you like; the browser wraps the
   text to fit the available width automatically.</p>
<p>This is a second, separate paragraph.</p>
```

- A `<p>` is **block-level** and gets default top/bottom margins (which you control with CSS).
- Remember whitespace collapsing (§3.6): blank lines and indentation in the source don't create extra space — separate paragraphs create the structure.
- A `<p>` **cannot contain block-level elements** (no `<div>`, `<ul>`, another `<p>` inside it). For lists or sub-blocks, close the paragraph first.

### 4.3 Semantic emphasis: `<strong>` & `<em>` vs presentational `<b>` & `<i>` **[B/I]**

These four look similar by default (`<strong>`/`<b>` bold, `<em>`/`<i>` italic) but **mean** different things. This distinction is a classic interview question and a real accessibility issue.

| Element | Meaning | Screen reader behaviour | Default look |
|---------|---------|--------------------------|--------------|
| `<strong>` | Strong **importance**, seriousness, or urgency | May change tone/emphasis | bold |
| `<em>` | **Stress emphasis** — the word you'd stress when speaking | May change inflection | italic |
| `<b>` | Stylistically **bold** with **no extra importance** (keywords, product names) | No special meaning | bold |
| `<i>` | **Off-tone** text: technical terms, foreign phrases, thoughts, ship names | No special meaning | italic |

```html
<!-- SEMANTIC: this word carries genuine importance / emphasis -->
<p><strong>Warning:</strong> never mix bleach and ammonia.</p>
<p>I said the <em>blue</em> one, not the red one.</p>

<!-- PRESENTATIONAL: bold/italic for convention, no added meaning -->
<p>The Latin term <i lang="la">in vitro</i> means "in glass."</p>
<p>Search for the keyword <b>refactor</b> in the codebase.</p>
```

**Rule of thumb:** if the emphasis *changes the meaning* or *conveys importance*, use `<strong>`/`<em>` (semantic). If you just want bold/italic for stylistic convention with no semantic weight, use `<b>`/`<i>` — or better, a CSS class. Either way, **never** use `<strong>`/`<em>` *just* to get bold/italic styling.

### 4.4 `<br>` (line break) and `<hr>` (thematic break) **[B]**

```html
<p>Roses are red,<br />
   Violets are blue.</p>            <!-- <br> forces a line break WITHIN a block -->

<hr />                              <!-- thematic break between sections -->
```

- **`<br>`** is a void element that forces a single line break inside flowing text. **Use it only for content where the break is meaningful** — addresses, poems, song lyrics. **Do not** use a string of `<br><br>` to create spacing between paragraphs; use separate `<p>` elements and CSS margins.
- **`<hr>`** represents a **thematic break** — a shift in topic (a scene change in a story, a separation between sections). It happens to *render* as a horizontal line, but its **meaning** is "topic change." Don't use it purely as a decorative divider; for pure decoration use a CSS border.

### 4.5 Quotations: `<blockquote>`, `<q>`, `<cite>` **[B/I]**

```html
<!-- Long, block-level quotation -->
<blockquote cite="https://example.com/source">
  <p>The best way to predict the future is to invent it.</p>
</blockquote>
<p>— <cite>Alan Kay</cite></p>      <!-- <cite> = the TITLE of a work/source -->

<!-- Short, inline quotation: the browser ADDS quotation marks automatically -->
<p>She said <q>this won't take long</q> and she was right.</p>
```

- **`<blockquote>`** = a block-level quotation (set off from surrounding text). The optional `cite` **attribute** holds a URL of the source (not shown by default).
- **`<q>`** = a short **inline** quotation; the browser inserts the locale-appropriate quotation marks, so **don't type your own** around it.
- **`<cite>`** = the **title of a creative work** (book, film, song, paper) being referenced — *not* the person's name per the spec, though it's commonly used for attribution. Renders italic by default.

### 4.6 Code & preformatted text: `<code>`, `<pre>`, `<kbd>`, `<samp>`, `<var>` **[B/I]**

```html
<!-- Inline code: a snippet within a sentence -->
<p>Run <code>npm install</code> to install dependencies.</p>

<!-- A code block: <pre> preserves whitespace/newlines; <code> marks it as code.
     The combo <pre><code> is the standard way to show multi-line code. -->
<pre><code>function greet(name) {
  return "Hello, " + name;
}</code></pre>

<p>Press <kbd>Ctrl</kbd> + <kbd>S</kbd> to save.</p>   <!-- keyboard input -->
<p>The program printed <samp>Segmentation fault</samp>.</p> <!-- sample output -->
<p>Solve for <var>x</var> in the equation.</p>          <!-- a variable -->
```

- **`<pre>`** ("preformatted") is special: it is the **one element that preserves whitespace and line breaks exactly** (it doesn't collapse — §3.6) and renders in a monospace font. Perfect for code, ASCII art, fixed-width tables.
- **`<code>`** marks text as computer code (monospace by default). For multi-line code, nest `<code>` inside `<pre>`.
- Inside a code block, remember to **escape** `<`, `>`, `&` (§3.8) or your tags will be interpreted as real HTML.
- `<kbd>` = user keyboard input, `<samp>` = sample program output, `<var>` = a variable/placeholder. All semantic niceties.

### 4.7 Lists: `<ul>`, `<ol>`, `<dl>` **[B]**

Three kinds of lists, each with a distinct meaning.

**Unordered list `<ul>`** — items where **order doesn't matter** (bullets):

```html
<ul>
  <li>Eggs</li>
  <li>Milk</li>
  <li>Flour</li>
</ul>
```

**Ordered list `<ol>`** — items where **order matters** (numbered steps, rankings):

```html
<ol>
  <li>Preheat the oven.</li>
  <li>Mix the ingredients.</li>
  <li>Bake for 30 minutes.</li>
</ol>

<!-- Useful <ol> attributes: -->
<ol start="5">   <!-- start numbering at 5 -->
  <li>Fifth item</li>
</ol>
<ol reversed>    <!-- count down -->
  <li>Third place</li>
  <li>Second place</li>
  <li>First place</li>
</ol>
<ol type="A">    <!-- A, B, C... (also "a", "i", "I", "1") -->
  <li>Alpha</li>
</ol>
```

**Description list `<dl>`** — **name/value pairs** (terms and their descriptions, key/value data, metadata, glossaries):

```html
<dl>
  <dt>HTML</dt>                       <!-- dt = the term being defined -->
  <dd>The markup language of the web.</dd>   <!-- dd = its description -->

  <dt>CSS</dt>
  <dd>The styling language of the web.</dd>

  <!-- one term can have multiple descriptions, or vice versa -->
  <dt>Coffee</dt>
  <dt>Java</dt>
  <dd>A hot caffeinated beverage.</dd>
</dl>
```

**Important rules:**

- A `<ul>`/`<ol>` may contain **only `<li>`** elements as direct children (plus `<script>`/`<template>`). Don't put text or a `<div>` directly inside `<ul>`.
- **Lists nest** by putting a `<ul>`/`<ol>` *inside* an `<li>`:

  ```html
  <ul>
    <li>Fruits
      <ul>                 <!-- nested list lives INSIDE the parent <li> -->
        <li>Apple</li>
        <li>Banana</li>
      </ul>
    </li>
    <li>Vegetables</li>
  </ul>
  ```

- Navigation menus are conventionally marked up as a `<ul>` of links inside `<nav>` (§7) — semantically "a list of navigation choices."
- The bullets/numbers are **styling**; change or remove them with CSS (`list-style`), not by switching to the wrong element.

### 4.8 Inline semantic helpers: `<abbr>`, `<time>`, `<mark>`, `<sub>`, `<sup>`, `<small>`, `<span>` **[B/I]**

These add fine-grained meaning to inline text.

```html
<!-- <abbr>: an abbreviation/acronym; the title gives the expansion (tooltip). -->
<p>We use <abbr title="HyperText Markup Language">HTML</abbr> for structure.</p>

<!-- <time>: a machine-readable date/time. The datetime attr is the canonical
     value (ISO 8601); the visible text can be human-friendly. Great for SEO,
     calendars, and "x days ago" scripts. -->
<p>Published on <time datetime="2026-06-21">June 21st, 2026</time>.</p>
<p>The event starts at <time datetime="2026-06-21T19:30">7:30 PM</time>.</p>

<!-- <mark>: highlighted/relevant text, e.g. search-result matches. -->
<p>Your search for <mark>HTML</mark> returned 3 results.</p>

<!-- <sub> / <sup>: subscript and superscript. -->
<p>Water is H<sub>2</sub>O.</p>
<p>E = mc<sup>2</sup>.  The 1<sup>st</sup> place winner.</p>

<!-- <small>: side comments / fine print (legal, copyright). -->
<p><small>© 2026 Acme Corp. All rights reserved.</small></p>

<!-- <span>: the SEMANTICALLY NEUTRAL inline container. Use ONLY when no other
     element fits, purely as a hook for CSS/JS on a piece of inline text. -->
<p>The total is <span class="price">$42.00</span>.</p>
```

- **`<time>`** deserves emphasis: the `datetime` attribute holds a standard, machine-readable timestamp (ISO 8601 format like `2026-06-21` or `2026-06-21T19:30:00Z`), while the element's text is whatever's nice for humans. This lets browsers, search engines, and scripts reliably understand the date.
- **`<span>`** and **`<div>`** are the two **non-semantic** elements (inline and block respectively). They mean *nothing*. Reach for them **only** when no semantic element fits and you just need a styling/scripting hook (§7 discusses "div soup").

---

## 5. Links & Navigation

Links are the defining feature of the web (the "hypertext" in HTML). The anchor element `<a>` creates them.

### 5.1 The anchor element `<a>` and `href` **[B]**

```html
<a href="https://example.com">Visit Example</a>
```

- **`href`** ("hypertext reference") is the destination. The text between the tags is the **clickable link text**, which should be **descriptive** — never "click here" (bad for SEO and for screen-reader users who navigate by a list of link texts; "click here, click here, click here" is useless out of context).

### 5.2 Absolute vs relative URLs **[B, important]**

Understanding URL resolution is essential or your links/images break when you reorganize files.

```html
<!-- ABSOLUTE URL: full address including scheme + domain. Goes to that exact
     site regardless of where the current page lives. Use for EXTERNAL sites. -->
<a href="https://developer.mozilla.org/">MDN</a>

<!-- ROOT-RELATIVE: starts with "/", resolved from the SITE ROOT (domain).
     "/about.html" -> https://yoursite.com/about.html  no matter the current page. -->
<a href="/about.html">About</a>
<a href="/images/logo.png">logo</a>

<!-- DOCUMENT-RELATIVE: resolved relative to the CURRENT page's folder. -->
<a href="contact.html">Contact</a>       <!-- a file in the SAME folder -->
<a href="blog/post1.html">First post</a> <!-- into a SUBfolder -->
<a href="../index.html">Home</a>         <!-- ".." = go UP one folder -->
<a href="../../assets/x.png">image</a>   <!-- up two folders -->
```

Mental model:

- **Absolute** (`https://...`): the complete address. Use for links to *other* websites.
- **Root-relative** (`/path`): from the domain root. Robust within your own site because it doesn't depend on the current page's location — recommended for internal links on a real server. (Note: on `file://` there is no "root," so these may not work when double-clicking files — another reason to run a local server, §1.6.)
- **Document-relative** (`path`, `../path`): from the current file's folder. `.` = current folder, `..` = parent folder. Simple but breaks if you move the page.

### 5.3 Fragment links (anchors / in-page jumps) **[B]**

A `#` in an `href` jumps to an element with that `id` on the page (or another page).

```html
<!-- Table of contents linking to sections on the SAME page -->
<nav>
  <a href="#intro">Introduction</a>
  <a href="#setup">Setup</a>
  <a href="#top">Back to top</a>
</nav>

<h2 id="intro">Introduction</h2>   <!-- the target has a matching id -->
<h2 id="setup">Setup</h2>

<a href="#">           <!-- "#" alone jumps to the top of the page -->
<a href="other.html#setup">  <!-- jump to #setup ON ANOTHER page -->
```

This is exactly how the Table of Contents at the top of this guide works. The browser scrolls so the target element is at the top (CSS `scroll-margin-top` and `scroll-behavior: smooth` refine this — see CSS_GUIDE.md).

### 5.4 `target` — where the link opens **[B]**

```html
<a href="https://example.com" target="_blank" rel="noopener noreferrer">
  Opens in a new tab
</a>
```

- **`target="_blank"`** opens the link in a **new tab/window**. Other values: `_self` (default, same tab), `_parent`, `_top` (for frames).
- **Always pair `target="_blank"` with `rel="noopener"`** (and often `noreferrer`). Why: without it, the newly opened page can access your page via `window.opener` and potentially hijack it (a "tabnabbing" attack), and it can hurt performance. Modern browsers imply `noopener` for `_blank`, but write it explicitly for safety and clarity.
- **UX note:** opening new tabs unexpectedly annoys users and can disorient screen-reader users. Use `_blank` sparingly and ideally indicate it (e.g. an icon or visually-hidden "(opens in new tab)" text).

### 5.5 `rel` — the relationship attribute **[I]**

`rel` describes the relationship between the current page and the link target. Common values:

| `rel` value | Meaning / use |
|-------------|---------------|
| `noopener` | Prevents the new page from accessing `window.opener` (security). |
| `noreferrer` | Don't send the `Referer` header (privacy); also implies `noopener`. |
| `nofollow` | Tell search engines not to pass ranking credit (paid/untrusted links). |
| `sponsored` | Marks paid/affiliate links (SEO). |
| `ugc` | User-generated content links (comments, forums). |
| `external` | The link goes to another site. |
| `me` | This link represents the same person/entity (identity verification, e.g. Mastodon). |

`rel` is also used heavily on `<link>` in the `<head>` (`rel="stylesheet"`, `rel="icon"`, `rel="canonical"`, `rel="preload"` — see §10).

### 5.6 Special link schemes: `mailto:`, `tel:`, `sms:` **[B]**

```html
<a href="mailto:hi@example.com">Email us</a>
<!-- Optional pre-filled fields: -->
<a href="mailto:hi@example.com?subject=Hello&body=Hi%20there&cc=team@example.com">
  Email with subject
</a>

<a href="tel:+15551234567">Call +1 555 123 4567</a>   <!-- dials on phones -->
<a href="sms:+15551234567">Text us</a>
```

- `mailto:` opens the user's email client. The query string (`?subject=...&body=...`) pre-fills fields; spaces must be URL-encoded (`%20`).
- `tel:` is invaluable on mobile — tapping it dials. Use the full international format (`+countrycode...`).

### 5.7 The `download` attribute **[B/I]**

```html
<!-- Instead of NAVIGATING to the file, the browser DOWNLOADS it. -->
<a href="report.pdf" download>Download the report (PDF)</a>

<!-- Optionally rename the downloaded file: -->
<a href="generated-7f3a.pdf" download="Annual-Report-2026.pdf">Download</a>
```

- `download` turns a link into a "save this file" action instead of opening/navigating to it. The optional value sets the **suggested filename**.
- **Security note:** `download` only works for **same-origin** URLs (and `blob:`/`data:` URLs). You can't force-download a file from another domain with it.
- Great for PDFs, images, generated files (e.g. a CSV your JS creates as a `blob:` URL).

### 5.8 Linking images, and other content as links **[B]**

Any content — text, images, even whole blocks — can be a link, as long as you don't nest interactive elements (§3.5):

```html
<a href="/product/42">
  <img src="thumb.jpg" alt="Red running shoes" />
  <span>Red Running Shoes — $89</span>
</a>
```

Wrapping an `<img>` in an `<a>` makes the image clickable. (When an image is itself the link, its `alt` text should describe the *link destination/purpose*, not just the picture — §6.)

---

## 6. Images & Media

### 6.1 The `<img>` element, `src` and `alt` **[B, important]**

```html
<img src="photos/sunset.jpg" alt="The sun setting over a calm ocean"
     width="800" height="533" />
```

- **`src`** is the image source (path/URL — relative or absolute, same rules as links, §5.2).
- **`alt`** ("alternative text") is **text that replaces the image** when it can't be seen. This is one of the most important attributes in all of HTML. **Why `alt` matters:**
  - **Blind/low-vision users** with screen readers hear the `alt` text — it's their only access to the image's content. No `alt` = the image is invisible to them (or the reader announces the meaningless filename).
  - It shows if the image **fails to load** (broken link, slow connection, blocked).
  - **Search engines** read it (image SEO).

**How to write good `alt`:**

```html
<!-- Informative image: describe what it conveys, concisely. -->
<img src="chart.png" alt="Sales doubled from Q1 to Q2 2026" />

<!-- Image that's also a link/button: describe the ACTION/DESTINATION. -->
<a href="/"><img src="logo.png" alt="Acme Corp home" /></a>

<!-- PURELY DECORATIVE image (adds nothing to meaning): use EMPTY alt="" so
     screen readers SKIP it. Do NOT omit the attribute entirely. -->
<img src="divider-flourish.png" alt="" />
```

- **Never omit `alt`.** For meaningful images, describe the content/function. For purely decorative images, use **`alt=""`** (empty) so assistive tech skips them. Omitting `alt` entirely is *invalid* and causes screen readers to read the filename.
- Don't start with "Image of…" — the reader already announces it's an image.

### 6.2 `width`/`height` and avoiding layout shift (CLS) **[B/I, important]**

```html
<img src="hero.jpg" alt="..." width="1200" height="630" />
```

Always set `width` and `height` (the image's **intrinsic pixel dimensions**), even when CSS resizes the image. **Why:** if the browser doesn't know the aspect ratio before the image downloads, it reserves *no* space; when the image finally arrives, everything below it jumps down. This visible jump is **Cumulative Layout Shift (CLS)** — a poor experience and a penalized Core Web Vital (§14).

When you provide `width`/`height`, the browser computes the **aspect ratio** and reserves the correct space immediately, even while you size the image fluidly in CSS:

```css
/* Common CSS pairing: keep images responsive without losing the reserved ratio */
img { max-width: 100%; height: auto; }
```

> The attributes are *unitless numbers* (CSS pixels). They're a ratio hint, not a hard size — your CSS still controls the rendered size.

### 6.3 Responsive images: `srcset` and `sizes` **[I]**

Serving a 2000px image to a 360px phone wastes bandwidth. **`srcset`** lets the browser pick the best image for the device.

**Resolution switching** (same image, different sizes; let the browser pick based on screen density and layout width):

```html
<img
  src="photo-800.jpg"                       <!-- fallback for old browsers -->
  srcset="photo-400.jpg 400w,               /* each candidate + its WIDTH */
          photo-800.jpg 800w,
          photo-1600.jpg 1600w"
  sizes="(max-width: 600px) 100vw,          /* how WIDE the image is displayed */
         (max-width: 1200px) 50vw,
         800px"
  alt="A scenic mountain lake"
  width="800" height="533" />
```

- **`srcset`** lists candidate images with their **intrinsic widths** (the `400w` means "this file is 400px wide").
- **`sizes`** tells the browser **how wide the image will be displayed** at various breakpoints (in CSS units / `vw`). The browser combines `sizes` + the device's pixel density to choose the smallest `srcset` candidate that still looks sharp.
- The browser makes the choice *before* layout, and it's allowed to consider network conditions. You provide options; the browser optimizes.

> **Density descriptors** (simpler, fixed-display-size case): `srcset="logo.png 1x, logo@2x.png 2x"` serves the `2x` file to high-DPI ("Retina") screens.

### 6.4 `<picture>` — art direction and format fallbacks **[I]**

When you need **different images** (not just sizes) — e.g. a cropped version on mobile (**art direction**), or a modern format with a fallback — use `<picture>`:

```html
<!-- ART DIRECTION: a square crop on narrow screens, wide crop otherwise -->
<picture>
  <source media="(max-width: 600px)" srcset="hero-square.jpg" />
  <source media="(min-width: 601px)" srcset="hero-wide.jpg" />
  <img src="hero-wide.jpg" alt="Team photo" width="1200" height="600" />
</picture>

<!-- FORMAT FALLBACK: try AVIF, then WebP, then JPEG. The browser uses the
     FIRST <source> whose type it supports; <img> is the ultimate fallback. -->
<picture>
  <source srcset="photo.avif" type="image/avif" />
  <source srcset="photo.webp" type="image/webp" />
  <img src="photo.jpg" alt="A red bicycle" width="800" height="600" />
</picture>
```

- `<picture>` is a wrapper containing one or more `<source>` elements and **exactly one `<img>`** (which is required — it's the fallback *and* it carries the `alt`, `width`, `height`).
- The browser evaluates `<source>`s top-to-bottom and uses the first match; if none match, it uses the `<img>`.
- **AVIF** and **WebP** are modern formats with much smaller file sizes than JPEG/PNG — using `<picture>` lets you serve them while older browsers fall back to JPEG.

### 6.5 Native lazy-loading: `loading="lazy"` **[B/I]** ⚡

```html
<!-- The browser DEFERS loading this image until it's about to scroll into view. -->
<img src="below-the-fold.jpg" alt="..." width="800" height="600" loading="lazy" />
```

- **What it does:** images far down the page aren't downloaded until the user scrolls near them, saving bandwidth and speeding up initial load. Before this attribute, you needed a JavaScript library; now it's **one attribute, no JS**.
- **When to use it:** for images **below the fold** (off-screen at load). **Do NOT** lazy-load your above-the-fold hero/LCP image — that would *delay* the most important pixel. For the hero image, you can even use `fetchpriority="high"` to load it sooner.
- `loading="lazy"` also works on **`<iframe>`** (§6.8). The default is `loading="eager"`.
- ⚡ **Version note:** native lazy-loading is broadly supported in all current browsers in 2026.

### 6.6 `<figure>` and `<figcaption>` **[B/I]**

When an image (or code block, chart, diagram) has a **caption** and is a self-contained unit referenced from the text, wrap it semantically:

```html
<figure>
  <img src="bridge.jpg" alt="A suspension bridge at dawn" width="800" height="500" />
  <figcaption>The Golden Gate Bridge, photographed in June 2026.</figcaption>
</figure>
```

- `<figure>` groups the content and its caption into one semantic unit; `<figcaption>` (first or last child) is the caption.
- It's not only for images: code samples, quotes, tables, and diagrams can all live in a `<figure>`.
- The `alt` (for screen readers) and the visible `<figcaption>` serve different purposes and may say different things — `alt` describes the image; the caption adds context.

### 6.7 Audio and video: `<audio>`, `<video>`, `<source>`, `<track>` **[I]**

HTML5 introduced **native** media playback — no plugins (remember Flash?).

```html
<!-- VIDEO with controls, a poster image, multiple formats, and captions -->
<video
  controls                       <!-- show the native play/pause/volume UI -->
  width="640" height="360"
  poster="preview.jpg"           <!-- image shown before playback -->
  preload="metadata"             <!-- "none" | "metadata" | "auto" (bandwidth hint) -->
  playsinline>                   <!-- play inline on iOS instead of fullscreen -->

  <!-- Provide multiple formats; the browser picks the first it supports -->
  <source src="clip.webm" type="video/webm" />
  <source src="clip.mp4"  type="video/mp4" />

  <!-- Captions/subtitles for accessibility (a WebVTT .vtt file) -->
  <track kind="captions" src="captions-en.vtt" srclang="en"
         label="English" default />

  <!-- Fallback text for browsers that can't play video at all -->
  Your browser doesn't support HTML video.
  <a href="clip.mp4">Download the video</a>.
</video>

<!-- AUDIO works the same way -->
<audio controls preload="none">
  <source src="song.ogg" type="audio/ogg" />
  <source src="song.mp3" type="audio/mpeg" />
  Your browser doesn't support the audio element.
</audio>
```

Key attributes (both `<audio>` and `<video>`):

| Attribute | Effect |
|-----------|--------|
| `controls` | Show the browser's built-in playback UI. Without it, you must control playback via JS. |
| `autoplay` | Start automatically. **Browsers block autoplay with sound**; it only works if also `muted`. Use sparingly — it's hostile. |
| `muted` | Start muted (required for autoplay to be allowed). |
| `loop` | Repeat when finished. |
| `preload` | `none` / `metadata` / `auto` — how aggressively to prefetch. |
| `poster` (video) | Placeholder image before play. |
| `playsinline` (video) | Prevent forced fullscreen on mobile. |

- **`<source>`**: list multiple encodings; the browser uses the first `type` it supports. MP4 (H.264) is the safest baseline for video; MP3/AAC for audio.
- **`<track kind="captions">`** is an **accessibility must** for video with speech: it attaches a **WebVTT** (`.vtt`) caption/subtitle file. Captions serve deaf/hard-of-hearing users and anyone watching without sound. `kind` can also be `subtitles`, `descriptions`, `chapters`, `metadata`.
- All media elements expose a rich **JavaScript API** (`.play()`, `.pause()`, `.currentTime`, events like `timeupdate`) for custom players — see JAVASCRIPT_GUIDE.md.

### 6.8 `<iframe>` — embedding another document **[I]**

An **inline frame** embeds another HTML page (a map, a video player, a payment widget) inside yours.

```html
<iframe
  src="https://www.openstreetmap.org/export/embed.html"
  width="600" height="450"
  title="Map of our office location"   <!-- REQUIRED for accessibility -->
  loading="lazy"                        <!-- defer offscreen iframes -->
  referrerpolicy="no-referrer"
  sandbox="allow-scripts allow-same-origin"  <!-- restrict what the frame can do -->
  allow="fullscreen">                   <!-- permissions policy (camera, etc.) -->
</iframe>
```

- **`title`** is **required for accessibility** — it labels the frame for screen-reader users (e.g. "Map of our office location"). Don't skip it.
- **`sandbox`** locks the embedded content down (no scripts, forms, popups, etc.) and you re-enable only what's needed via `allow-*` tokens. Use it for untrusted content. An empty `sandbox` (just the attribute) is the most restrictive.
- **`allow`** is the Permissions Policy — grant specific capabilities like `fullscreen`, `camera`, `geolocation`, `clipboard-write`.
- **Security:** iframes are powerful but risky (clickjacking, third-party tracking). The flip side is the `X-Frame-Options` / `Content-Security-Policy: frame-ancestors` **HTTP headers**, which control whether *your* site may be framed by others.
- iframes are heavy; `loading="lazy"` helps a lot when they're below the fold.

### 6.9 Inline SVG basics **[I]**

**SVG** (Scalable Vector Graphics) is an XML-based image format you can embed **directly in HTML**. Unlike a JPEG/PNG (raster, made of pixels), SVG is **vector** (made of math) — it scales to any size **without blurring**, is tiny for simple shapes, and is **stylable/scriptable** because its shapes become DOM nodes.

```html
<!-- Inline SVG: shapes live in the DOM, so CSS and JS can target them -->
<svg width="120" height="120" viewBox="0 0 120 120"
     role="img" aria-label="A red circle with a blue border"
     xmlns="http://www.w3.org/2000/svg">
  <!-- viewBox = the internal coordinate system; the SVG scales to width/height -->
  <circle cx="60" cy="60" r="50" fill="crimson" stroke="navy" stroke-width="4" />
  <rect x="20" y="20" width="30" height="30" fill="gold" />
  <text x="60" y="110" text-anchor="middle" font-size="12">SVG</text>
</svg>
```

Ways to use SVG:

- **Inline `<svg>`** (above): the shapes are DOM nodes — style with CSS (`fill`, `stroke`), animate, script. Best for **icons** and interactive graphics. Add `role="img"` + `aria-label` (or a `<title>` child) for accessibility, or `aria-hidden="true"` if purely decorative.
- **As an image:** `<img src="icon.svg" alt="..." />` — simpler, cached, but not stylable from your page's CSS.
- **As a CSS background** (see CSS_GUIDE.md).

SVG is ideal for logos, icons, charts, and illustrations. Photographs should stay raster (JPEG/AVIF/WebP).

---

## 7. Semantic HTML5 & Document Landmarks

This is one of the most important sections in the guide. **Semantic HTML** means using elements that describe the *meaning and role* of content, rather than wrapping everything in generic `<div>`s. HTML5 added a set of **structural semantic elements** specifically so pages can express their anatomy.

### 7.1 Why semantics matter (the "why" before the "what") **[I, important]**

A `<div>` and a `<section>` may look identical on screen (both render as plain blocks). So why bother? Three concrete payoffs:

1. **Accessibility.** Semantic elements automatically expose **landmarks** to assistive technology. A screen-reader user can press a key to jump straight to the main content, the navigation, or the site footer — but **only if** you used `<main>`, `<nav>`, `<footer>` instead of `<div>`. Generic `<div>`s create no landmarks, so those users must wade through everything linearly. Semantics turn a wall of text into a navigable document.
2. **SEO & machine understanding.** Search engines, "reader mode," and content scrapers use semantic structure to identify the primary content (`<article>`/`<main>`) versus boilerplate (`<nav>`, `<aside>`, `<footer>`). Better structure → better understanding → better ranking and richer previews.
3. **Maintainability.** `<header>`, `<nav>`, `<main>`, `<footer>` make the code **self-documenting**. Six months later, `</main>` tells you exactly what just ended; `</div>` tells you nothing. It also reduces the need for `id`/`class` names just to identify regions.

The anti-pattern this fights is **"div soup"** (§16): pages built from nothing but `<div>`s and `<span>`s, which are semantically empty and accessibility-hostile.

### 7.2 The structural elements **[I]**

Here is a full, semantic page skeleton, then each element explained:

```html
<body>
  <header>                          <!-- top banner: logo, site title, primary nav -->
    <a href="/" class="logo">Acme</a>
    <nav aria-label="Primary">      <!-- a major navigation block -->
      <ul>
        <li><a href="/">Home</a></li>
        <li><a href="/blog">Blog</a></li>
        <li><a href="/contact">Contact</a></li>
      </ul>
    </nav>
  </header>

  <main>                            <!-- THE unique main content of this page -->
    <article>                       <!-- a self-contained, distributable piece -->
      <header>                      <!-- an article can have its own header -->
        <h1>Why Semantic HTML Matters</h1>
        <p>By Tom · <time datetime="2026-06-21">June 21, 2026</time></p>
      </header>

      <section>                     <!-- a thematic grouping WITH a heading -->
        <h2>Accessibility</h2>
        <p>...</p>
      </section>

      <section>
        <h2>SEO</h2>
        <p>...</p>
      </section>

      <footer>                      <!-- footer for THIS article -->
        <p>Tagged: html, a11y</p>
      </footer>
    </article>

    <aside>                         <!-- tangential content: related links, ads -->
      <h2>Related posts</h2>
      <ul><li><a href="#">...</a></li></ul>
    </aside>
  </main>

  <footer>                          <!-- site-wide footer -->
    <p><small>© 2026 Acme Corp.</small></p>
    <nav aria-label="Footer"> ... </nav>
  </footer>
</body>
```

| Element | Meaning / when to use |
|---------|------------------------|
| `<header>` | Introductory content for its nearest section — site banner at page level, or an article's title block. **Can appear multiple times** (one per `<article>`/`<section>`). |
| `<nav>` | A block of **major navigation** links (primary menu, breadcrumb, pagination, footer nav). Not *every* group of links — just significant navigation. AT users can list/skip nav blocks. |
| `<main>` | The **dominant, unique content** of the page. **Exactly one per page**, and it should **not** be nested inside `<article>`/`<aside>`/`<header>`/`<footer>`/`<nav>`. Repeated boilerplate (header/footer/nav) lives *outside* it. Provides the "skip to main content" landmark. |
| `<section>` | A **thematic grouping** of content, almost always **with a heading**. Think "a chapter/section that would appear in an outline." |
| `<article>` | A **self-contained, independently distributable** unit: a blog post, news story, product card, forum comment, widget. The test: would it make sense on its own / in an RSS feed? Articles can nest (a post and its comments). |
| `<aside>` | Content **tangential** to the surrounding content: sidebars, pull quotes, related links, ads. Removing it shouldn't break the main content. |
| `<footer>` | Footer for its nearest section — site footer (copyright, links) at page level, or an article's footer (author, tags). Can appear multiple times. |

### 7.3 `<section>` vs `<article>` vs `<div>` — the decision **[I, important]**

This is the question everyone trips on. Decide in this order:

1. **Is the content self-contained and would it make sense distributed on its own** (a blog post, a product, a comment, a card you'd syndicate)? → **`<article>`**.
2. **Is it a thematic part of the page that has (or should have) a heading** and belongs in the page's outline (e.g. "Features," "Pricing," "FAQ")? → **`<section>`**.
3. **Do you only need a box to hang CSS/JS on, with no semantic meaning** (a layout wrapper, a flex/grid container, a styling hook)? → **`<div>`**. This is *legitimate* and not "div soup" — `<div>` is the correct tool when there genuinely is no semantic meaning.

Key tests:

- A **`<section>` should almost always have a heading**. If you can't think of a heading for it, it's probably a `<div>`, not a `<section>`.
- Don't use `<section>` as a generic wrapper for styling — that's what `<div>` is for. Misusing `<section>` adds noise to the accessibility tree (extra meaningless regions).
- `<article>` is for *content*, not layout. A "card" component showing one product is an `<article>`; the grid wrapping all the cards is a `<div>` (or a `<ul>`/`<li>` if it's truly a list).

```html
<!-- GOOD: div is the right choice for a pure layout wrapper -->
<div class="grid">
  <article class="card"> ... product 1 ... </article>
  <article class="card"> ... product 2 ... </article>
</div>

<!-- GOOD: section with a heading, part of the page outline -->
<section aria-labelledby="faq-h">
  <h2 id="faq-h">Frequently Asked Questions</h2>
  ...
</section>
```

### 7.4 Landmarks & labeling multiple regions **[I/A]**

Each of `<header>`(page-level → `banner`), `<nav>`(`navigation`), `<main>`(`main`), `<aside>`(`complementary`), `<footer>`(page-level → `contentinfo`), and `<form>`/`<section>` (when labeled) maps to an **ARIA landmark** that AT users can jump between. Best practices:

- If you have **more than one `<nav>`** (e.g. primary + footer), **distinguish them** with `aria-label` so screen-reader users hear "Primary navigation" vs "Footer navigation":

  ```html
  <nav aria-label="Primary"> ... </nav>
  <nav aria-label="Footer"> ... </nav>
  ```

- Same for multiple `<section>`/`<aside>` regions — label them with `aria-labelledby` (pointing at their heading) or `aria-label`.
- Provide a **"skip to main content" link** as the first focusable element so keyboard users can bypass the nav (§11).

> **⚡ Note:** these elements provide landmarks **for free** — that's the whole point. Only add explicit `role="..."` when you genuinely can't use the native element. Native semantics beat ARIA (§11).

---

## 8. Tables

### 8.1 What tables are for (and what they're NOT for) **[I, important]**

A `<table>` represents **two-dimensional tabular data** — information with a meaningful row/column relationship: a price list, a schedule, a comparison matrix, a spreadsheet of results.

**The cardinal rule: never use tables for page layout.** In the 1990s–2000s, before CSS layout matured, people nested tables to position page elements. This is now a serious anti-pattern because:

- It's **inaccessible**: screen readers announce table semantics ("table with 5 rows, 3 columns") for what is just a layout, confusing users.
- It's **rigid and unmaintainable**, and terrible on mobile.

For layout, use **CSS Flexbox and Grid** (see CSS_GUIDE.md). Use `<table>` **only** for actual data.

### 8.2 Table structure **[I]**

```html
<table>
  <caption>Quarterly Sales by Region (2026)</caption>  <!-- accessible title -->

  <thead>                              <!-- the header row group -->
    <tr>
      <th scope="col">Region</th>      <!-- column header -->
      <th scope="col">Q1</th>
      <th scope="col">Q2</th>
    </tr>
  </thead>

  <tbody>                              <!-- the data row group(s) -->
    <tr>
      <th scope="row">North</th>       <!-- row header -->
      <td>$10,000</td>                 <!-- data cell -->
      <td>$12,500</td>
    </tr>
    <tr>
      <th scope="row">South</th>
      <td>$8,000</td>
      <td>$9,200</td>
    </tr>
  </tbody>

  <tfoot>                              <!-- the footer row group (totals etc.) -->
    <tr>
      <th scope="row">Total</th>
      <td>$18,000</td>
      <td>$21,700</td>
    </tr>
  </tfoot>
</table>
```

The elements:

| Element | Role |
|---------|------|
| `<table>` | The whole table. |
| `<caption>` | The table's title/description. **Should be the first child.** Read by screen readers — strongly recommended. |
| `<thead>` / `<tbody>` / `<tfoot>` | Group the header / body / footer rows. They aid semantics, styling, and let `<thead>`/`<tfoot>` repeat when printing long tables. |
| `<tr>` | A table **row** (contains cells). |
| `<th>` | A **header** cell (bold + centered by default). For a column header or a row header. |
| `<td>` | A **data** cell. |

### 8.3 `scope` — making headers accessible **[I, important]**

```html
<th scope="col">...</th>   <!-- this header labels a COLUMN -->
<th scope="row">...</th>   <!-- this header labels a ROW -->
```

`<th>` alone says "this is a header," but **`scope`** says *what it heads*. This is what lets a screen reader announce "Region: North, Q2: $9,200" — associating each data cell with its row and column headers. **Always add `scope`** to your `<th>`s. For complex tables, the `headers` attribute (referencing header cell `id`s) handles cells governed by multiple headers.

### 8.4 Spanning cells: `colspan` and `rowspan` **[I]**

```html
<table>
  <tr>
    <th>Name</th>
    <th colspan="2">Scores</th>     <!-- this cell spans 2 COLUMNS -->
  </tr>
  <tr>
    <td>Alice</td>
    <td>90</td>
    <td>85</td>
  </tr>
  <tr>
    <td rowspan="2">Team B</td>     <!-- this cell spans 2 ROWS down -->
    <td>70</td>
    <td>75</td>
  </tr>
  <tr>
    <!-- no first cell here: the rowspan above covers it -->
    <td>60</td>
    <td>65</td>
  </tr>
</table>
```

- **`colspan="n"`** makes a cell stretch across `n` columns.
- **`rowspan="n"`** makes a cell stretch down across `n` rows.
- When you span, **omit** the cells that the span now covers (don't add empty `<td>`s for them) — that's the common mistake that makes tables misalign.

### 8.5 `<colgroup>`/`<col>` **[A]**

For styling whole columns (e.g. highlighting one column) without adding a class to every cell:

```html
<table>
  <colgroup>
    <col />                         <!-- first column, default -->
    <col style="background: #fffbe6" /> <!-- second column highlighted -->
  </colgroup>
  <!-- ...rows... -->
</table>
```

---

## 9. Forms — The Big One

Forms are how users **send data to a server** (and feed data to JavaScript). This is one of the largest, most important areas of HTML, and the area where accessibility and semantics matter most because broken forms block people from doing things.

### 9.1 The `<form>` element: `action` and `method` **[I]**

```html
<form action="/subscribe" method="post">
  <label for="email">Email</label>
  <input type="email" id="email" name="email" required />
  <button type="submit">Subscribe</button>
</form>
```

- **`action`** = the URL the form data is sent to when submitted. If omitted, it submits to the **current page's URL**.
- **`method`** = the HTTP method:
  - **`get`** (the **default** if you omit `method`) — appends the data to the URL as a query string (`/search?q=html&page=2`). Use for **idempotent, non-sensitive, bookmarkable** requests like searches and filters. **Never** send passwords via GET (they'd appear in the URL, history, and server logs). Limited length.
  - **`post`** — sends the data in the **request body**, not the URL. Use for **anything that changes state** or contains sensitive/large data (logins, sign-ups, file uploads, comments).
- **`enctype`** matters for POST:
  - `application/x-www-form-urlencoded` (default) — for normal text fields.
  - **`multipart/form-data`** — **required for file uploads** (`<input type="file">`). Forgetting this is the #1 reason uploads "don't work."

  ```html
  <form action="/upload" method="post" enctype="multipart/form-data">
    <input type="file" name="avatar" />
    <button>Upload</button>
  </form>
  ```

> **⚡ Gotcha — the default method is GET.** If you write `<form action="/login">` with no `method`, it submits via GET and your credentials end up in the URL. Always set `method="post"` for anything sensitive.

### 9.2 `<label>` and why association is non-negotiable **[I, important]**

```html
<!-- METHOD 1: explicit association via for/id (preferred, most flexible) -->
<label for="username">Username</label>
<input type="text" id="username" name="username" />

<!-- METHOD 2: implicit association by WRAPPING the input -->
<label>
  Username
  <input type="text" name="username" />
</label>
```

A `<label>` is **the human-readable name of a form control**, and associating it correctly is critical:

- **Accessibility:** a properly associated label is what a screen reader announces when the field gets focus ("Username, edit text"). Without it, the user hears only "edit text" and has no idea what to type. This is the single most common form accessibility failure.
- **Usability for everyone:** clicking the label **focuses/toggles the control** — a much bigger click target, which especially helps with small checkboxes and radio buttons on touch screens.

How to associate: either `<label for="X">` matching `<input id="X">` (the `for` ↔ `id` link — note it's `for`, not `name`), **or** wrap the input inside the `<label>`. **Every input needs a label.** A `placeholder` is **not** a label (it vanishes when typing and has poor contrast/AT support). If a visible label truly can't be shown, use `aria-label` — but a real `<label>` is best.

### 9.3 `<input>` and its many `type`s **[I]**

The `<input>` element is a chameleon: the **`type`** attribute changes what it is. Choosing the right type gives you appropriate keyboards on mobile, built-in validation, and native UI for free.

```html
<!-- TEXT-LIKE inputs -->
<input type="text"     name="full_name" placeholder="Jane Doe" />
<input type="email"    name="email" />     <!-- email keyboard + format validation -->
<input type="password" name="pw" />        <!-- masks characters -->
<input type="search"   name="q" />         <!-- search semantics, clear button -->
<input type="url"      name="site" />      <!-- URL keyboard + validation -->
<input type="tel"      name="phone" />     <!-- numeric phone keypad (no validation) -->

<!-- NUMERIC -->
<input type="number" name="qty" min="1" max="10" step="1" />  <!-- spinner + numeric kb -->
<input type="range"  name="vol" min="0" max="100" step="5" value="50" /> <!-- slider -->

<!-- DATE & TIME (native pickers) -->
<input type="date"           name="dob" />        <!-- calendar picker -->
<input type="time"           name="appt" />
<input type="datetime-local" name="when" />
<input type="month"          name="exp" />
<input type="week"           name="wk" />

<!-- CHOICES -->
<input type="checkbox" id="terms" name="terms" />      <!-- on/off, independent -->
<label for="terms">I accept the terms</label>

<!-- Radios: same NAME = mutually exclusive group; only one can be chosen -->
<input type="radio" id="s" name="size" value="small" />  <label for="s">Small</label>
<input type="radio" id="m" name="size" value="medium" /> <label for="m">Medium</label>
<input type="radio" id="l" name="size" value="large" />  <label for="l">Large</label>

<!-- SPECIALIZED -->
<input type="color" name="theme" value="#3366ff" />   <!-- native color picker -->
<input type="file"  name="docs" accept="image/*" multiple /> <!-- file chooser -->
<input type="hidden" name="csrf" value="abc123" />    <!-- not shown; sends data -->

<!-- BUTTON-LIKE input types (prefer the <button> element instead, §9.7) -->
<input type="submit" value="Send" />
<input type="reset"  value="Clear" />
```

Reference of common types:

| `type` | Purpose / notes |
|--------|-----------------|
| `text` | Generic single-line text (the default). |
| `email` | Email; validates format; mobile shows `@` keyboard. Add `multiple` for comma-lists. |
| `password` | Masks input. Use with HTTPS; never send via GET. |
| `number` | Numeric only; `min`/`max`/`step`; spin buttons. Not for things like phone/credit-card (use `text`/`tel` + `pattern`). |
| `tel` | Telephone; numeric keypad on mobile; **no format validation** (numbers vary worldwide) — pair with `pattern`. |
| `url` | Requires a valid absolute URL. |
| `search` | Like text but semantically a search box; may show a clear (×) button. |
| `range` | Slider for imprecise numeric input; show the value separately. |
| `date` / `time` / `datetime-local` / `month` / `week` | Native pickers; value is ISO format. |
| `checkbox` | Independent on/off. Sends `name=value` (default `value` is `"on"`) only when checked. |
| `radio` | One choice from a group sharing the same `name`. Always give each a `value`. |
| `color` | Color picker; value like `#rrggbb`. |
| `file` | File upload; `accept` filters types, `multiple` allows many. Needs `enctype="multipart/form-data"`. |
| `hidden` | Not displayed; carries data (tokens, IDs) with the submission. |
| `submit` / `reset` / `button` | Buttons (prefer `<button>`). |

### 9.4 `<textarea>` — multi-line text **[I]**

```html
<label for="msg">Your message</label>
<textarea id="msg" name="message" rows="5" cols="40"
          maxlength="500" placeholder="Type here..."></textarea>
```

- For **multi-line** text (comments, messages, bios). Unlike `<input>` it's **not void** — it has a closing tag, and its **initial content goes between the tags** (not in a `value` attribute). Beware: whitespace/newlines inside `<textarea>...</textarea>` become part of the value, so keep the tags tight.
- `rows`/`cols` set the initial size (CSS usually overrides). Tip: `style="resize: vertical"` to control user resizing.

### 9.5 `<select>`, `<option>`, `<optgroup>` — dropdowns **[I]**

```html
<label for="country">Country</label>
<select id="country" name="country" required>
  <option value="">-- Choose one --</option>  <!-- empty value + required = forces a pick -->
  <optgroup label="Africa">                    <!-- visually groups options -->
    <option value="ng">Nigeria</option>
    <option value="ke">Kenya</option>
  </optgroup>
  <optgroup label="Europe">
    <option value="de" selected>Germany</option> <!-- selected = default choice -->
    <option value="fr">France</option>
  </optgroup>
</select>

<!-- Multi-select (Ctrl/Cmd-click); submits multiple name=value pairs -->
<select name="toppings" multiple size="4">
  <option value="cheese">Cheese</option>
  <option value="ham">Ham</option>
</select>
```

- The submitted value is each chosen `<option>`'s **`value`** attribute (falls back to its text if `value` is omitted — always set `value`).
- `selected` sets the default; `<optgroup label="...">` groups options under a non-selectable heading; `multiple` allows several choices.
- For a **free-text field with suggestions**, use `<datalist>` (§9.8), not `<select>`.

### 9.6 `<fieldset>` and `<legend>` — grouping controls **[I]**

```html
<fieldset>
  <legend>Shipping address</legend>     <!-- the group's caption -->
  <label for="street">Street</label>
  <input id="street" name="street" type="text" />
  <label for="city">City</label>
  <input id="city" name="city" type="text" />
</fieldset>

<!-- ESSENTIAL for radio/checkbox groups: <legend> gives the group a name -->
<fieldset>
  <legend>Preferred contact method</legend>
  <input type="radio" id="c-email" name="contact" value="email" />
  <label for="c-email">Email</label>
  <input type="radio" id="c-phone" name="contact" value="phone" />
  <label for="c-phone">Phone</label>
</fieldset>
```

- `<fieldset>` groups related controls (renders with a border by default); `<legend>` is the group's caption.
- **Why it matters for a11y:** for a radio/checkbox group, each radio has its own `<label>` ("Email," "Phone"), but **what question are they answering?** The `<legend>` ("Preferred contact method") supplies that group-level context to screen readers. Without it, the user hears the options but not the question.

### 9.7 `<button>` and its `type`s **[I, important]**

```html
<button type="submit">Save</button>   <!-- submits the form -->
<button type="reset">Reset</button>   <!-- clears the form to defaults -->
<button type="button">Toggle</button> <!-- does NOTHING by itself; for JS handlers -->
```

- Prefer the **`<button>` element** over `<input type="submit">` because `<button>` can contain **HTML content** (icons, formatted text), not just a plain string.
- **Always set `type` explicitly.** ⚡ **Gotcha:** a `<button>` **inside a `<form>` defaults to `type="submit"`**. So a `<button>Menu</button>` you meant as a JS toggle will *submit the form and reload the page* when clicked. For any button that isn't meant to submit, write **`type="button"`**.
- Use `type="reset"` rarely — users hate accidentally wiping a long form.

### 9.8 `<datalist>` — autocomplete suggestions **[I]**

```html
<label for="browser">Favorite browser</label>
<input list="browsers" id="browser" name="browser" />  <!-- list points at the datalist id -->
<datalist id="browsers">
  <option value="Chrome"></option>
  <option value="Firefox"></option>
  <option value="Safari"></option>
  <option value="Edge"></option>
</datalist>
```

- A `<datalist>` provides **suggestions** for a normal `<input>` while still allowing **free text** (unlike `<select>`, which restricts to its options). The `<input list="...">` references the datalist's `id`.

### 9.9 Native client-side validation **[I, important]**

HTML can validate input **before** submission, with **no JavaScript** — the browser checks and shows a message, blocking submit until valid. This is "constraint validation."

```html
<form>
  <!-- required: must not be empty -->
  <input type="text" name="name" required />

  <!-- type="email"/"url": format checked automatically -->
  <input type="email" name="email" required />

  <!-- min / max / step: numeric & date ranges -->
  <input type="number" name="age" min="18" max="120" required />

  <!-- minlength / maxlength: text length -->
  <input type="password" name="pw" minlength="8" maxlength="64" required />

  <!-- pattern: a regular expression the value must FULLY match.
       title supplies the error hint shown to the user. -->
  <input type="text" name="zip" pattern="\d{5}"
         title="Enter a 5-digit ZIP code" required />

  <button>Sign up</button>
</form>
```

Key constraints:

| Attribute | Constraint |
|-----------|------------|
| `required` | Field must have a value. |
| `min` / `max` | Min/max for numbers and dates. |
| `step` | Allowed increment (e.g. `step="0.01"` for currency, `step="any"` for any). |
| `minlength` / `maxlength` | Min/max number of characters. |
| `pattern="regex"` | Value must match the regex entirely (anchored). Pair with `title`. |
| `type="email"/"url"/"number"/..."` | The type's own format rules. |

CSS hooks: `:required`, `:optional`, `:valid`, `:invalid`, `:user-invalid` (only flags after interaction — nicer UX) let you style validation states (see CSS_GUIDE.md).

> **Security warning — validation is for UX, not security.** Client-side validation is a *convenience*; it can be trivially bypassed (DevTools, curl, disabling JS). **You must ALWAYS re-validate and sanitize on the server.** Never trust input that came from the browser. See NODEJS_GUIDE.md / your backend guide.

You can also opt **out** of native validation:

```html
<form novalidate> ... </form>           <!-- skip all native validation -->
<button formnovalidate>Save draft</button> <!-- this button bypasses validation -->
```

### 9.10 Helpful form/input attributes **[I]**

| Attribute | Purpose |
|-----------|---------|
| `name` | **The key under which the value is submitted.** No `name` = the field is **not sent** at all. Essential. |
| `value` | The field's current/default value. |
| `placeholder` | Faint hint text inside the field. **Not a substitute for `<label>`.** |
| `autocomplete` | Lets browsers/password managers autofill — use **specific tokens** (`autocomplete="email"`, `"current-password"`, `"new-password"`, `"name"`, `"street-address"`, `"one-time-code"`). Good autocomplete is an accessibility *and* UX win; `autocomplete="off"` is rarely appropriate. |
| `autofocus` | Focus this field on load (use **at most one** per page; can be disorienting). |
| `readonly` | Value shown and **submitted** but not editable. |
| `disabled` | Value not editable **and NOT submitted** (and removed from tab order). |
| `multiple` | Allow multiple values (`email`, `file`, `select`). |
| `inputmode` | Hints the on-screen keyboard (`numeric`, `decimal`, `tel`, `email`, `search`) without changing validation. |
| `form="formId"` | Associates a control with a `<form>` even if it sits **outside** that form in the DOM. |

> ⚡ **`readonly` vs `disabled`:** both prevent editing, but **`disabled` values are NOT submitted** with the form, while `readonly` values **are**. Choose based on whether you need the value sent.

### 9.11 How forms submit, and the bridge to JavaScript **[I/A]**

What happens on submit (no JS):

1. The browser runs native validation (§9.9) unless `novalidate`.
2. It gathers every control that has a `name` (and, for checkboxes/radios, is checked) into name/value pairs.
3. It sends them to `action` using `method` (GET → query string; POST → body) and the page **navigates** (full reload).

To handle the form **with JavaScript instead** (AJAX, single-page apps), you intercept the `submit` event and call `preventDefault()`. The `FormData` API reads the values without manually querying each field:

```html
<form id="signup">
  <input name="email" type="email" required />
  <input name="age" type="number" />
  <button>Join</button>
</form>

<script>
  const form = document.getElementById('signup');
  form.addEventListener('submit', async (event) => {
    event.preventDefault();          // stop the default navigation/reload
    if (!form.checkValidity()) {     // run native validation programmatically
      form.reportValidity();         // show the native messages
      return;
    }
    const data = new FormData(form); // grabs all named controls automatically
    const payload = Object.fromEntries(data.entries());
    // Send it without leaving the page (see JAVASCRIPT_GUIDE.md for fetch):
    await fetch('/subscribe', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload),
    });
  });
</script>
```

This is the seam between HTML and JS: HTML provides the structure, types, and native validation; JS adds dynamic behaviour. Even in JS-driven apps, **keep the real `<form>`, `<label>`, and proper input types** — they give you accessibility, mobile keyboards, autofill, and progressive enhancement for free.

---

## 10. Metadata & the `<head>` in Depth

The `<head>` is invisible but powerful — it controls SEO, social sharing, how resources load, and more. Here is a thorough, realistic `<head>`, dissected.

```html
<head>
  <!-- 1. Encoding FIRST (within first 1024 bytes) -->
  <meta charset="UTF-8" />

  <!-- 2. Viewport for responsive design -->
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />

  <!-- 3. Title (tab + search result headline) -->
  <title>How to Bake Sourdough — The Bread Blog</title>

  <!-- 4. SEO: the search-result snippet (~155 chars) -->
  <meta name="description"
        content="A beginner-friendly guide to baking your first sourdough loaf at home." />

  <!-- 5. Canonical URL: the preferred address if duplicates exist -->
  <link rel="canonical" href="https://breadblog.example/sourdough" />

  <!-- 6. Robots: how search engines should treat this page -->
  <meta name="robots" content="index, follow" />

  <!-- 7. Open Graph (Facebook, LinkedIn, Slack, Discord, iMessage previews) -->
  <meta property="og:title"       content="How to Bake Sourdough" />
  <meta property="og:description" content="A beginner-friendly sourdough guide." />
  <meta property="og:image"       content="https://breadblog.example/og.jpg" />
  <meta property="og:url"         content="https://breadblog.example/sourdough" />
  <meta property="og:type"        content="article" />

  <!-- 8. Twitter/X Card -->
  <meta name="twitter:card"  content="summary_large_image" />
  <meta name="twitter:title" content="How to Bake Sourdough" />
  <meta name="twitter:image" content="https://breadblog.example/og.jpg" />

  <!-- 9. Favicons / app icons -->
  <link rel="icon" href="/favicon.ico" sizes="any" />
  <link rel="icon" href="/icon.svg" type="image/svg+xml" />
  <link rel="apple-touch-icon" href="/apple-touch-icon.png" />
  <link rel="manifest" href="/site.webmanifest" />

  <!-- 10. Theme color for mobile browser UI -->
  <meta name="theme-color" content="#b5651d" />

  <!-- 11. Performance hints -->
  <link rel="preconnect" href="https://fonts.example" />
  <link rel="preload" href="/fonts/main.woff2" as="font" type="font/woff2" crossorigin />

  <!-- 12. CSS (render-blocking; keep it in <head>) -->
  <link rel="stylesheet" href="/styles.css" />

  <!-- 13. JS (deferred so it doesn't block parsing) -->
  <script src="/app.js" defer></script>
</head>
```

### 10.1 `<meta name="description">` **[I]**

A ~155-character summary. Search engines often use it as the **snippet** under your title in results (though they may rewrite it). It doesn't directly affect ranking, but a compelling description improves click-through. Write a unique one per page.

### 10.2 Open Graph & Twitter Cards (social previews) **[I, important]**

When someone pastes your link into Slack, Discord, iMessage, WhatsApp, LinkedIn, or X, the rich preview card (image + title + description) comes from these tags:

- **Open Graph** (`og:*`, used by most platforms): at minimum `og:title`, `og:description`, `og:image`, `og:url`, `og:type`. The image should be ~1200×630px.
- **Twitter/X cards** (`twitter:*`): `twitter:card` (`summary_large_image` for a big image), `twitter:title`, `twitter:image`. X will fall back to OG tags if these are absent.
- Note these use **`property=`** (OG) and **`name=`** (Twitter) — a common copy-paste mistake.

Without these, your shared links look like bare URLs — a missed engagement opportunity.

### 10.3 Favicons & app icons **[I]**

The favicon is the little icon in the browser tab/bookmark. Modern robust setup:

- `rel="icon" type="image/svg+xml"` — a crisp SVG icon (scales, supports dark mode).
- `rel="icon" sizes="any"` `favicon.ico` — legacy fallback.
- `rel="apple-touch-icon"` — icon when added to an iOS home screen.
- `rel="manifest"` — a **Web App Manifest** (`.webmanifest` JSON) describing your site as an installable PWA (name, icons, theme/background color, display mode).

### 10.4 Canonical & robots **[I]**

- **`<link rel="canonical">`** tells search engines the **preferred URL** when the same content is reachable at multiple URLs (e.g. with/without trailing slash, tracking params). Prevents duplicate-content dilution.
- **`<meta name="robots">`** controls indexing: `index`/`noindex` (allow/forbid listing in results) and `follow`/`nofollow` (follow links or not). E.g. `content="noindex, nofollow"` for a private/staging page. (The `robots.txt` *file* is a separate, site-wide crawler control.)

### 10.5 Linking CSS: `<link rel="stylesheet">` **[B/I]**

```html
<link rel="stylesheet" href="styles.css" />
<link rel="stylesheet" href="print.css" media="print" />  <!-- only when printing -->
```

- Goes in `<head>`. CSS is **render-blocking** by design: the browser delays painting until CSS is ready, to avoid a flash of unstyled content (FOUC). Keep stylesheets lean and in the `<head>`.
- The `media` attribute conditionally loads CSS (`media="print"`, `media="(max-width: 600px)"`).
- See CSS_GUIDE.md for the language itself.

### 10.6 Linking JS: `<script>` with `defer`, `async`, `type="module"` **[I, important]**

How and where you load JS dramatically affects performance because, as noted in §1.4, a plain `<script>` **blocks HTML parsing**.

```html
<!-- 1. Plain script in <head>: BLOCKS parsing while it downloads + runs. Avoid. -->
<script src="app.js"></script>

<!-- 2. defer: downloads in parallel, runs AFTER the HTML is fully parsed,
        in ORDER. Best default for scripts that touch the DOM. -->
<script src="app.js" defer></script>

<!-- 3. async: downloads in parallel, runs AS SOON AS it arrives (parsing pauses
        briefly), order NOT guaranteed. Good for independent scripts like analytics. -->
<script src="analytics.js" async></script>

<!-- 4. type="module": an ES module. Deferred by DEFAULT, supports import/export,
        runs in strict mode, has its own scope. The modern way. -->
<script type="module" src="main.js"></script>

<!-- 5. Inline fallback for browsers without module support (rarely needed now) -->
<script nomodule src="legacy.js"></script>
```

| Attribute | Download | Execution | Order | Use for |
|-----------|----------|-----------|-------|---------|
| (none) | blocks parser | immediately, blocking | source order | almost never |
| `defer` | parallel | after parse completes | source order | most app scripts (DOM-dependent) |
| `async` | parallel | immediately on arrival | unordered | independent/3rd-party (analytics) |
| `type="module"` | parallel | deferred by default | dependency order | modern ES-module code |

**Practical rules:**

- For most scripts, put `<script defer>` in the `<head>` (or `type="module"`). It downloads early but runs after the DOM exists, so you don't need to query for elements that aren't there yet.
- The old advice "put scripts at the end of `<body>`" achieves a similar effect and is still fine, but `defer`/`module` is cleaner.
- `async` only for truly independent scripts where order doesn't matter.
- See JAVASCRIPT_GUIDE.md for the language and module system.

### 10.7 Resource hints: `preload`, `prefetch`, `preconnect`, `dns-prefetch` **[A]**

These tell the browser to fetch or warm up resources earlier than it otherwise would:

| Hint | Meaning |
|------|---------|
| `<link rel="preload" as="...">` | "I need this **for the current page**; fetch it **now**, high priority." Common for fonts, the LCP image, critical scripts. `as` is required so the browser sets priority/CORS correctly. |
| `<link rel="prefetch">` | "I'll **probably need this for the NEXT navigation**; fetch it at low priority when idle." For likely next pages. |
| `<link rel="preconnect">` | "Open a connection (DNS + TCP + TLS) to this **origin** now." For third-party origins you'll fetch from (fonts, CDN, API). |
| `<link rel="dns-prefetch">` | Cheaper than preconnect: just resolve DNS ahead of time. |

```html
<link rel="preconnect" href="https://cdn.example" crossorigin />
<link rel="preload" href="/hero.avif" as="image" fetchpriority="high" />
<link rel="prefetch" href="/next-page.html" />
```

Use sparingly — preloading everything defeats the purpose by competing for bandwidth.

---

## 11. Accessibility (a11y)

**Accessibility** means building pages usable by **everyone**, including people who use screen readers, navigate by keyboard only, have low vision or color blindness, or have motor or cognitive differences. It is also a **legal requirement** in many jurisdictions and overlaps heavily with good SEO and general UX. The guiding standard is **WCAG** (Web Content Accessibility Guidelines).

The single most encouraging fact: **most accessibility comes free from writing correct, semantic HTML.** Sections 4, 6, 7, 8, and 9 already covered the big wins — headings, `alt`, landmarks, table `scope`, labels. This section ties them together and adds ARIA, keyboard, and focus.

### 11.1 Semantic structure is the foundation **[I]**

- Use the **right element for the job** (a `<button>` for a button, a `<a>` for a link, an `<h2>` for a subheading). Native elements come with built-in keyboard behaviour, focus, roles, and states that you'd otherwise have to rebuild (badly) with ARIA + JS.
- A `<div onclick>` styled to *look* like a button is **not** a button: it isn't focusable, doesn't respond to Enter/Space, and isn't announced as a button. Just use `<button>`.
- Proper **heading hierarchy** (§4.1) and **landmarks** (§7.4) let AT users navigate. A logical reading order in the **DOM** matters too — don't rely on CSS to reorder content into a sequence that differs from the source, since AT follows the DOM.

### 11.2 ARIA — and "no ARIA is better than bad ARIA" **[I/A, important]**

**ARIA** (Accessible Rich Internet Applications) is a set of attributes — **roles**, **states**, and **properties** — that supplement HTML semantics for custom widgets that HTML can't express natively (tabs, comboboxes, tree views, live regions).

```html
<!-- ROLE: what this thing IS (only when no native element fits) -->
<div role="alert">Your changes were saved.</div>   <!-- announced immediately -->

<!-- PROPERTIES: relationships / labels -->
<nav aria-label="Breadcrumb"> ... </nav>            <!-- names this landmark -->
<button aria-labelledby="t1"></button> <span id="t1">Delete</span>
<input aria-describedby="hint" /> <p id="hint">Use 8+ characters.</p>

<!-- STATES: dynamic conditions JS keeps in sync -->
<button aria-expanded="false" aria-controls="menu">Menu</button>
<div id="menu" hidden> ... </div>
<input aria-invalid="true" aria-errormessage="err" />
<button aria-pressed="true">Bold</button>           <!-- toggle state -->

<!-- LIVE REGIONS: announce dynamic updates without moving focus -->
<div aria-live="polite" id="status"></div>          <!-- e.g. "3 results loaded" -->
```

**The five rules of ARIA (paraphrased), most importantly:**

1. **Don't use ARIA if a native HTML element/attribute already gives you the semantics.** Prefer `<button>` over `<div role="button">`; prefer `<nav>` over `<div role="navigation">`. Native is more robust and free.
2. **Don't change native semantics** (don't put `role="heading"` on a `<button>`).
3. **All interactive ARIA controls must be keyboard-operable.**
4. **Don't suppress focus** on focusable elements you expect users to reach.
5. **Interactive elements must have an accessible name** (label).

> **"No ARIA is better than bad ARIA."** Incorrect ARIA actively *breaks* accessibility — e.g. `role="button"` on a `<div>` that doesn't also handle Enter/Space and `tabindex`, or `aria-hidden="true"` accidentally hiding real content, or a stale `aria-expanded` your JS forgot to update. If you're unsure, use the native element and add ARIA only when you genuinely build a custom widget — and then keep its states in sync with JS.

### 11.3 Keyboard navigation & focus **[I, important]**

Many users navigate entirely by keyboard (Tab to move, Shift+Tab back, Enter/Space to activate, Arrow keys within widgets). Requirements:

- **Everything interactive must be reachable and operable by keyboard.** Native `<a>`, `<button>`, and form controls are by default. Custom widgets need `tabindex` and key handlers.
- **Visible focus indicator:** users must see *where* focus is. **Never** do `outline: none` without providing an equally visible replacement (see `:focus-visible` in CSS_GUIDE.md). Removing focus outlines is a top accessibility failure.
- **`tabindex`:**
  - `tabindex="0"` — insert a normally-non-focusable element into the natural tab order (use for custom widgets).
  - `tabindex="-1"` — focusable **only via script** (`element.focus()`), not by tabbing. Used to move focus programmatically (e.g. to a dialog or a route's new heading).
  - **`tabindex="1"` (or any positive number) — avoid.** It overrides natural order and creates chaos. Order should come from DOM order.
- **Skip link:** make the first focusable element a "Skip to main content" link so keyboard users can bypass the nav:

  ```html
  <body>
    <a href="#main" class="skip-link">Skip to main content</a>
    <header> ...nav... </header>
    <main id="main" tabindex="-1"> ... </main>
  </body>
  ```

  (The `.skip-link` is typically visually hidden until focused — a CSS technique.)

- **Focus management** for dynamic UI: when you open a modal (`<dialog>`, §12), move focus into it and **trap** focus inside; on close, return focus to the trigger. After a client-side route change in an SPA, move focus to the new page's heading. `<dialog>`'s `showModal()` handles much of this for you.

### 11.4 Visual & content accessibility **[I]**

- **Color contrast:** text must contrast enough with its background (WCAG AA: 4.5:1 for normal text, 3:1 for large text). DevTools and contrast checkers measure this. (Implemented in CSS — see CSS_GUIDE.md.)
- **Don't rely on color alone** to convey meaning (e.g. "required fields are red") — add an icon, text, or symbol too, for color-blind users.
- **`alt` text** on meaningful images, `alt=""` on decorative ones (§6.1).
- **Captions/transcripts** for audio/video (§6.7).
- **Form labels** and `<legend>` (§9.2, §9.6).
- **Visually-hidden but screen-reader-available text** (a `.sr-only` / `.visually-hidden` CSS class) provides context for AT without showing it visually — e.g. extra label text. Don't use `display:none`/`hidden` for this, as that hides it from AT too.

### 11.5 Screen-reader basics **[I]**

A screen reader (NVDA on Windows — free, JAWS, VoiceOver on macOS/iOS, TalkBack on Android) reads the page aloud and lets users navigate by element type. Test with one occasionally — it's eye-opening. Also use the browser's **Accessibility tree** inspector (in DevTools) and an automated checker (Lighthouse, axe) — but remember automated tools catch only ~30–40% of issues; **manual keyboard + screen-reader testing** is essential.

---

## 12. Interactive & Modern Elements

HTML has gained genuinely interactive elements that previously required JavaScript libraries. Using the native versions means accessibility, keyboard support, and focus handling come built in.

### 12.1 `<details>` and `<summary>` — native disclosure/accordion **[B/I]**

```html
<details>                                  <!-- collapsible region -->
  <summary>What is your refund policy?</summary>  <!-- always-visible toggle -->
  <p>You can request a full refund within 30 days of purchase.</p>
</details>

<details open>                             <!-- "open" = expanded by default -->
  <summary>Shipping information</summary>
  <p>We ship worldwide within 3–5 business days.</p>
</details>
```

- A **native, no-JS expand/collapse** widget. The `<summary>` is the clickable header; everything else in `<details>` is hidden until expanded. It's keyboard-accessible and announced correctly for free.
- Perfect for **FAQs, accordions, "show more"** sections, and progressive disclosure.
- `open` makes it start expanded. Group multiple `<details>` with the same `name` attribute to make an **exclusive accordion** (opening one closes the others) — ⚡ a newer feature, broadly supported in 2026.

### 12.2 The `<dialog>` element — native modals **[I/A]** ⚡

```html
<dialog id="confirm">
  <form method="dialog">                 <!-- method="dialog" closes the dialog on submit -->
    <h2>Delete this item?</h2>
    <p>This action cannot be undone.</p>
    <button value="cancel">Cancel</button>
    <button value="confirm">Delete</button>
  </form>
</dialog>

<button id="openBtn">Delete…</button>

<script>
  const dlg = document.getElementById('confirm');
  document.getElementById('openBtn')
    .addEventListener('click', () => dlg.showModal());  // open as a MODAL
  dlg.addEventListener('close', () => {
    console.log('Closed with value:', dlg.returnValue); // "confirm" / "cancel"
  });
</script>
```

The native `<dialog>` element handles the hard parts of modals **for you**:

- **`showModal()`** opens it as a true modal in the browser's **top layer** (above everything, no `z-index` wars), automatically adds a dimming **`::backdrop`**, **traps focus** inside, makes the rest of the page inert, and closes on **Esc**. `show()` opens a non-modal dialog. `close()` (or `dlg.close('value')`) closes it.
- A `<form method="dialog">` inside the dialog closes it on submit, and the submitting button's `value` becomes `dialog.returnValue`.
- ⚡ **Version note:** `<dialog>` and its `::backdrop` are baseline across all current browsers (since 2022). It replaces the old pattern of hand-rolled modal `<div>`s, which were notoriously hard to make accessible. **Use `<dialog>` for modals.**

### 12.3 The `popover` attribute — declarative overlays **[I/A]** ⚡

```html
<!-- The trigger uses popovertarget to control a popover by id. NO JS NEEDED. -->
<button popovertarget="menu">Open menu</button>

<div id="menu" popover>                    <!-- popover="auto" by default -->
  <ul>
    <li><a href="#">Profile</a></li>
    <li><a href="#">Settings</a></li>
  </ul>
</div>

<!-- popover="manual" stays open until you explicitly close it (no light-dismiss) -->
<div id="toast" popover="manual">Saved!</div>
```

- The **`popover` attribute** turns any element into a top-layer overlay (tooltip, menu, hint, popup) with **no JavaScript** — the browser handles showing/hiding, the top layer, and (for `popover="auto"`) **light dismiss** (click outside or press Esc to close) and one-popover-at-a-time behaviour.
- A control opens it via **`popovertarget="id"`** (and optionally `popovertargetaction="show|hide|toggle"`).
- `popover="auto"` (default): light-dismiss + auto-close others. `popover="manual"`: you control closing (good for toasts/notifications).
- ⚡ **Version note:** the Popover API is baseline across current browsers as of 2024. Combined with **CSS anchor positioning** (CSS_GUIDE.md), it makes tooltips/menus largely a no-JS affair. `<dialog>` is for *modal* blocking dialogs; `popover` is for *non-modal* transient overlays.

### 12.4 `<template>` — inert, reusable markup **[I/A]**

```html
<!-- Content inside <template> is PARSED but NOT rendered and NOT active
     (images don't load, scripts don't run). It's a stamp for JS to clone. -->
<template id="card-tpl">
  <article class="card">
    <h3 class="title"></h3>
    <p class="body"></p>
  </article>
</template>

<ul id="list"></ul>

<script>
  const tpl = document.getElementById('card-tpl');
  const list = document.getElementById('list');
  for (const item of [{t:'A', b:'1'}, {t:'B', b:'2'}]) {
    const node = tpl.content.cloneNode(true);   // deep-clone the template's content
    node.querySelector('.title').textContent = item.t;
    node.querySelector('.body').textContent  = item.b;
    list.append(node);                          // now it renders
  }
</script>
```

`<template>` holds markup that the browser **parses but doesn't render or activate** until you clone it with JS. It's the efficient, declarative way to define a chunk of DOM once and stamp out copies — the foundation for rendering lists and for web components.

### 12.5 Web Components: custom elements & `<slot>` **[A]**

**Web Components** are a browser-native way to build reusable, encapsulated components (like framework components, but standard). Three pillars:

- **Custom elements** — define your own tags (e.g. `<user-card>`) backed by a JS class.
- **Shadow DOM** — an encapsulated DOM subtree with **scoped CSS** (styles inside don't leak out, outside styles don't leak in).
- **`<template>` + `<slot>`** — define structure and project user-provided content into it.

```html
<!-- A template with named/default slots = "holes" for the user's content -->
<template id="card">
  <style>            /* scoped to this component's shadow DOM */
    .box { border: 1px solid #ccc; padding: 1rem; border-radius: 8px; }
    ::slotted(h2) { margin-top: 0; }   /* style slotted light-DOM content */
  </style>
  <div class="box">
    <slot name="title">Default title</slot>   <!-- named slot -->
    <slot></slot>                              <!-- default slot for the rest -->
  </div>
</template>

<script>
  class UserCard extends HTMLElement {
    constructor() {
      super();
      const root = this.attachShadow({ mode: 'open' });   // create shadow DOM
      const tpl = document.getElementById('card');
      root.append(tpl.content.cloneNode(true));           // stamp the template
    }
  }
  customElements.define('user-card', UserCard);           // register the tag
</script>

<!-- Usage: the light-DOM children fill the slots -->
<user-card>
  <h2 slot="title">Ada Lovelace</h2>      <!-- goes into slot name="title" -->
  <p>First programmer.</p>                <!-- goes into the default slot -->
</user-card>
```

This is a deep topic (lifecycle callbacks, attributes/properties, events) that lives more in JS than HTML — see JAVASCRIPT_GUIDE.md. The HTML takeaways: custom element **tag names must contain a hyphen** (`user-card`, never `usercard`), `<slot>` projects content, and Shadow DOM scopes styles.

### 12.6 `contenteditable` — editable content **[I]**

```html
<div contenteditable="true">
  You can <strong>edit</strong> this text directly in the browser.
</div>

<!-- "plaintext-only" allows editing but strips rich formatting -->
<div contenteditable="plaintext-only">Type plain text here.</div>
```

- Makes an element's content **directly editable** by the user. It's the basis of rich-text editors (though production editors use libraries to tame its quirks).
- Read/save the result via JS (`element.innerHTML` / `textContent`). It does **not** auto-submit with forms — you wire it up yourself.

### 12.7 Native drag-and-drop attributes **[A]**

```html
<!-- draggable="true" makes an element draggable; the rest is JS event handling -->
<div id="item" draggable="true">Drag me</div>
<div id="dropzone">Drop here</div>

<script>
  const item = document.getElementById('item');
  const zone = document.getElementById('dropzone');

  item.addEventListener('dragstart', (e) => {
    e.dataTransfer.setData('text/plain', 'item');   // attach payload
  });
  zone.addEventListener('dragover', (e) => e.preventDefault()); // allow dropping
  zone.addEventListener('drop', (e) => {
    e.preventDefault();
    const data = e.dataTransfer.getData('text/plain');
    zone.append(item);                              // perform the move
  });
</script>
```

- The **`draggable="true"`** attribute is the HTML part; everything else is the JavaScript **Drag-and-Drop API** (`dragstart`, `dragover`, `drop`, the `dataTransfer` object). The API is famously fiddly (you must `preventDefault()` on `dragover` to permit a drop). For complex needs, libraries help. Note native DnD has weak touch support — consider Pointer Events for mobile.

---

## 13. HTML5 JavaScript APIs Overview

"HTML5" popularized a large set of **JavaScript APIs** that browsers expose. They are **JS, not markup** — you won't write them as tags — but they're part of the modern web platform that "HTML5" colloquially includes, and HTML often provides the surface (a `<canvas>` element, a `<form>`, etc.). This is a **map with pointers**; the actual code lives in JAVASCRIPT_GUIDE.md and NODEJS_GUIDE.md.

| API | What it does | HTML touchpoint |
|-----|--------------|-----------------|
| **Canvas** | Pixel-based 2D drawing (games, charts, image editing) via `getContext('2d')`. | The `<canvas>` element. |
| **WebGL / WebGPU** | GPU-accelerated 3D/compute graphics. | `<canvas>`. |
| **Web Storage** | `localStorage` (persists) & `sessionStorage` (per-tab) — simple key/value string storage. | none (pure JS); needs `http(s)://`. |
| **IndexedDB** | A larger, structured, async client-side database. | none. |
| **Geolocation** | Ask (with permission) for the user's location. | none. |
| **History API** | `pushState`/`replaceState` + `popstate` to change the URL without a reload — the backbone of SPA routing. | relates to `<a>` navigation. |
| **Fetch API** | Modern promise-based networking (replaces `XMLHttpRequest`). | submitting `<form>` data via JS (§9.11). |
| **Web Workers** | Run JS on a **background thread** so heavy work doesn't freeze the UI. | none. |
| **Service Workers** | A proxy worker enabling **offline support, caching, push** — the heart of PWAs. | paired with the `<link rel="manifest">`. |
| **WebSocket** | Persistent, bidirectional real-time connection (chat, live data). | none. (See GO_GORILLA_WEBSOCKETS_GUIDE.md for a server.) |
| **Web Audio / WebRTC / Media** | Sound synthesis, peer-to-peer video/audio calls, camera/mic access. | `<audio>`, `<video>`. |
| **Intersection Observer** | Efficiently detect when an element scrolls into view (lazy-load, infinite scroll, animations). | observes any element. |
| **Notifications / Clipboard / Drag-Drop / File** | OS notifications, copy/paste, file reading. | `<input type="file">`, `draggable`. |

```html
<!-- Example: the <canvas> element is the HTML; the drawing is all JS -->
<canvas id="board" width="300" height="150" role="img"
        aria-label="A line chart of monthly sales"></canvas>
<script>
  const ctx = document.getElementById('board').getContext('2d');
  ctx.fillStyle = 'steelblue';
  ctx.fillRect(20, 20, 100, 60);   // x, y, width, height
</script>
```

**Key reminders:** `<canvas>` content is invisible to screen readers — always provide a text alternative (`aria-label` or fallback content). Many of these APIs require a **secure context** (`https://` or `localhost`) and explicit **user permission** (geolocation, camera, notifications). For real usage, go to the JS/Node guides.

---

## 14. SEO & Performance Basics

Good HTML is, by itself, good for **SEO** (Search Engine Optimization) and **performance**. Most of what follows you've already met — here it's framed around making pages fast and discoverable.

### 14.1 SEO essentials in HTML **[I]**

- **One descriptive `<title>` and `<meta name="description">` per page** (§10).
- **Logical heading hierarchy** with a single `<h1>` stating the topic (§4.1).
- **Semantic structure** (`<main>`, `<article>`, `<nav>`) so crawlers identify primary content vs boilerplate (§7).
- **Descriptive link text** (not "click here") and `alt` on images (§5, §6).
- **`<link rel="canonical">`** to consolidate duplicate URLs; **`robots`** meta to control indexing (§10.4).
- **Structured data (JSON-LD)** for rich results (recipes, products, FAQs, breadcrumbs) — embedded in a script:

  ```html
  <script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "Article",
    "headline": "How to Bake Sourdough",
    "datePublished": "2026-06-21",
    "author": { "@type": "Person", "name": "Tom" }
  }
  </script>
  ```

  (`type="application/ld+json"` isn't executed as JS — the browser ignores it; search engines read it.)
- **Mobile-friendliness** (the viewport meta, §2.5) is a ranking factor.
- **A `sitemap.xml`** and clean, readable URLs help crawling (server-side concerns).

### 14.2 Performance & Core Web Vitals **[I]**

Google's **Core Web Vitals** quantify user-perceived performance; HTML choices move these metrics directly:

| Metric | Measures | HTML levers |
|--------|----------|-------------|
| **LCP** (Largest Contentful Paint) | Time to render the biggest element (usually the hero image/heading) | Don't lazy-load the hero; `fetchpriority="high"` and `preload` for it; use efficient image formats; non-blocking JS. |
| **CLS** (Cumulative Layout Shift) | Unexpected layout jumps | Set `width`/`height` (or aspect-ratio) on images/iframes/ads (§6.2); reserve space for late content. |
| **INP** (Interaction to Next Paint) | Responsiveness to clicks/taps | Keep main-thread JS light; `defer`/`async`; offload heavy work to Web Workers. |

Practical HTML performance checklist:

- **Lazy-load below-the-fold** images and iframes (`loading="lazy"`, §6.5).
- **Responsive images** (`srcset`/`<picture>`) so phones don't download desktop-sized files (§6.3–6.4).
- **Modern image formats** (AVIF/WebP) — far smaller than JPEG/PNG.
- **Non-render-blocking JS** (`defer`/`async`/modules, §10.6); keep CSS lean (it's render-blocking).
- **Resource hints** (`preconnect`, `preload`) for critical/third-party resources (§10.7).
- **Minimize DOM size** — thousands of nodes slow layout and INP. Don't ship "div soup."
- Set `width`/`height` on media to keep CLS near zero.

Measure with **Lighthouse** (built into Chrome DevTools) — it audits performance, accessibility, SEO, and best practices and gives actionable fixes.

---

## 15. Best Practices & Validation

### 15.1 Separation of concerns **[I, important]**

Keep the three layers (§1.2) apart:

- **HTML** = structure/content. No inline styling beyond rare necessity; no big inline scripts.
- **CSS** = presentation, in external stylesheets (`<link rel="stylesheet">`).
- **JS** = behaviour, in external files (`<script defer>` / modules), attached via `addEventListener` — **not** inline `onclick="..."` attributes.

Why: maintainability (one place to change styles), caching (external files cache across pages), reusability, and security (inline handlers complicate Content-Security-Policy). It's fine to *start* a quick prototype inline, but factor it out as it grows.

```html
<!-- AVOID: presentation + behaviour tangled into the markup -->
<p style="color:red" onclick="alert('hi')">Click</p>

<!-- PREFER: clean HTML; CSS in a stylesheet; JS via addEventListener -->
<p class="warning" id="greet">Click</p>
<!-- styles.css:  .warning { color: red; } -->
<!-- app.js:      document.getElementById('greet')
                    .addEventListener('click', () => alert('hi')); -->
```

### 15.2 Progressive enhancement **[I]**

Build in layers so the **core content/functionality works for everyone**, then enhance:

1. **Solid semantic HTML** that works with no CSS and no JS (content is readable; forms submit to a server; links navigate).
2. **CSS** layered on for presentation.
3. **JS** layered on for richer interactivity — but the page shouldn't be *useless* if JS fails to load or errors.

This benefits slow networks, old devices, assistive tech, and resilience (a JS error shouldn't blank the page). The opposite, "graceful degradation," starts rich and tries to cope when features are missing — progressive enhancement is the more robust mindset.

### 15.3 Write valid, well-structured HTML **[I]**

- **Proper nesting** and closing of elements (§3.5); respect the content model.
- Use **semantic elements** (§7); reserve `<div>`/`<span>` for when nothing semantic fits.
- **One `<h1>`**, logical heading order (§4.1).
- **Labels** for every form control; `alt` for every image (§9.2, §6.1).
- **Lowercase** tag/attribute names; **quote** attribute values; consistent indentation.
- Declare `<!DOCTYPE html>`, `<html lang>`, `charset`, and viewport on every page (§2).

### 15.4 The W3C Validator **[I]**

The **W3C Markup Validation Service** (validator.w3.org) checks your HTML against the spec and reports errors (unclosed tags, illegal nesting, duplicate `id`s, missing `alt`, etc.). You can validate by **URL, file upload, or pasted source** (the latter two work even offline if you run a local validator like the `vnu.jar` Nu Html Checker).

Why bother when browsers "fix" errors anyway? Because the browser's guesses may not match your intent, and invalid HTML causes subtle, hard-to-debug rendering and accessibility problems. Treat validation like a linter — clean it up. (Editor extensions and `htmlhint`/`html-validate` can run in your build, like ESLint for HTML.)

### 15.5 General hygiene **[I]**

- **Accessibility from the start** — it's far cheaper than retrofitting (§11).
- **Test in multiple browsers** and on mobile; use **Baseline/Can I Use** to confirm support before relying on a new feature.
- **Don't reinvent native elements** — `<dialog>`, `<details>`, `<button>`, real form controls beat hand-rolled `<div>` versions.
- **Keep markup lean** — fewer nodes = faster, more maintainable.

---

## 16. Gotchas

A focused list of the mistakes that bite beginners and intermediates most often. Many were flagged inline; collected here for quick reference.

1. **Block vs inline misuse / illegal nesting.** Putting a `<div>` (block) inside a `<p>`, or block elements inside `<a>`/`<span>` incorrectly. The browser silently "fixes" it by closing tags early, splitting your structure. Know the content model (§3.5) and validate (§15.4).

2. **Missing or wrong `alt`.** Omitting `alt` entirely (invalid; screen readers read the filename) or writing `alt="image"`. Use descriptive `alt` for meaningful images, **`alt=""`** for decorative ones (§6.1).

3. **"Div soup."** Building everything from `<div>`/`<span>` with no semantic elements. Inaccessible (no landmarks), bad for SEO, unmaintainable. Use `<header>/<nav>/<main>/<article>/<section>/<footer>` and real `<button>`/`<a>` (§7).

4. **Unassociated form labels.** A `<label>` not linked to its input (mismatched `for`/`id`, or no label at all). Screen-reader users can't tell what to enter; clicking the label doesn't focus the field. Always associate (§9.2). And remember: **placeholder ≠ label**.

5. **Form `method` defaults to GET.** `<form action="/login">` with no `method` sends credentials in the URL. Use `method="post"` for sensitive/state-changing forms; remember `enctype="multipart/form-data"` for file uploads (§9.1).

6. **`<button>` defaults to `type="submit"` inside a form.** A button you meant for JS reloads the page. Always set `type="button"` for non-submit buttons (§9.7).

7. **Boolean attributes misunderstood.** Writing `required="false"` or `disabled="false"` — these are *still true* because the value is ignored. To turn them off, **remove the attribute** (§3.2).

8. **Encoding issues (mojibake).** Garbled `Ã©` characters from a missing/mismatched `<meta charset="UTF-8">` or a file not actually saved as UTF-8 (§2.4).

9. **Forgetting the viewport meta.** Without it, your responsive CSS doesn't work on phones — the page renders desktop-wide and shrinks (§2.5).

10. **Skipping heading levels / multiple `<h1>` chaos / choosing headings by size.** Breaks the outline and screen-reader navigation. One `<h1>`, no skipped levels, size via CSS (§4.1).

11. **Duplicate `id`s.** `id` must be unique per page; duplicates break `getElementById`, label association, and fragment links (§3.3).

12. **`name` missing on inputs.** A control with no `name` is **not submitted**. (And `disabled` controls aren't submitted either — vs `readonly`, which are.) (§9.10)

13. **`<table>` for layout.** Inaccessible and rigid; use CSS Grid/Flexbox. Tables are for *data* only (§8.1).

14. **Trusting client-side validation.** Native validation is UX only; it's bypassable. Always validate on the server (§9.9).

15. **Render-blocking scripts.** A plain `<script>` in `<head>` blocks parsing. Use `defer`/`async`/`type="module"` (§10.6).

16. **`outline: none` killing focus visibility.** Removes the keyboard-focus indicator — a serious a11y break. Keep a visible focus style (§11.3).

17. **`target="_blank"` without `rel="noopener"`.** Security/perf risk (tabnabbing) (§5.4).

18. **Whitespace confusion.** Expecting many spaces/blank lines to create spacing — HTML collapses them. Use CSS for spacing; `<pre>` to preserve whitespace (§3.6).

19. **`<br>` for spacing.** Stacking `<br><br>` instead of paragraphs/margins. Use `<p>` + CSS (§4.4).

20. **Forgetting `<source>` `type` / wrong `<picture>` order.** Browsers pick the first matching `<source>`; put the most-preferred format first and always include the `<img>` fallback (§6.4, §6.7).

---

## 17. Study Path & Build-to-Learn Projects

HTML is best learned by **building real pages** and inspecting them in DevTools. Here's a sequence from zero to confident, with projects that force you to use each concept. (Style with CSS — see **CSS_GUIDE.md** — and add behaviour with JS — see **JAVASCRIPT_GUIDE.md** — as you go.)

### 17.1 Suggested study order

1. **Foundations (§1–3):** Understand what HTML is, the DOM, the boilerplate, and the syntax rules (elements, attributes, nesting, entities). Write and serve a "Hello world" page; open DevTools and find your elements in the live DOM.
2. **Content & text (§4):** Headings, paragraphs, emphasis, lists, quotes, code. Get the semantic-vs-presentational distinction in your bones.
3. **Links & media (§5–6):** Relative/absolute URLs, fragments, images with proper `alt`, responsive images, audio/video, lazy loading.
4. **Semantics & structure (§7):** Build pages with `<header>/<nav>/<main>/<article>/<section>/<aside>/<footer>`. Learn the `<section>` vs `<div>` decision.
5. **Tables & forms (§8–9):** Forms are the biggest practical skill — input types, labels, validation, fieldsets, submission. Spend real time here.
6. **Head, a11y, performance (§10, §11, §14):** Metadata, social cards, accessibility, Core Web Vitals.
7. **Modern & advanced (§12–13):** `<details>`, `<dialog>`, `popover`, `<template>`, web components, and the JS-API landscape.
8. **Polish (§15–16):** Validate everything, apply best practices, internalize the gotchas.

### 17.2 Build-to-learn projects

Do these in order; each adds difficulty and exercises a cluster of skills. Build with **valid, semantic, accessible HTML first**, then layer CSS/JS.

**Project 1 — A semantic blog post page** *(exercises §2, §4, §5, §6, §7, §10)*
> Build a single article page: a page `<header>` with site title and `<nav>`; a `<main>` containing one `<article>` with its own `<header>` (title, author, `<time>` published date), several `<section>`s with `<h2>` subheadings, a `<figure>` with `<figcaption>`, a `<blockquote>`, inline `<code>` and a `<pre><code>` block, and a related-links `<aside>`; a site `<footer>`. Add a complete `<head>`: title, description, Open Graph tags, canonical, favicon. **Success criteria:** validates with zero errors; navigable by headings/landmarks in a screen reader; produces a rich social-share preview.

**Project 2 — An accessible contact form** *(exercises §9, §11)*
> Build a form with: text, email, tel, a `<textarea>`, a `<select>` with `<optgroup>`, a radio group and a checkbox group each wrapped in a `<fieldset>`/`<legend>`, a `<datalist>`-backed input, a file upload, and submit/reset `<button>`s with explicit `type`. Every control has an associated `<label>`. Use native validation (`required`, `type`, `pattern`, `minlength`). Set the correct `method` and `enctype`. **Success criteria:** fully keyboard-operable; every field announced with its label and any error; `multipart/form-data` set; can't submit while invalid. Then enhance: intercept submit in JS, validate with `checkValidity()`, and send via `fetch` without a reload.

**Project 3 — A responsive media gallery** *(exercises §6, §12, §14)*
> Build an image gallery using `<figure>`/`<figcaption>`, **responsive images** (`srcset`/`sizes`, or `<picture>` with AVIF/WebP + JPEG fallback), and **`loading="lazy"`** on below-the-fold images with `width`/`height` to prevent CLS. Add a native **`<dialog>`** lightbox that opens the full-size image (focus-trapped, Esc-closable) and a **`<details>`** "image info" disclosure per item. **Success criteria:** Lighthouse performance is high; no layout shift; the lightbox is keyboard-accessible and returns focus on close; phones download appropriately small images.

**Project 4 — A complete landing page** *(exercises everything)*
> Combine all of the above into a polished marketing landing page: semantic structure with multiple labeled `<section>`s (hero, features, pricing, FAQ, contact); a skip link and visible focus styles; a sticky `<nav>` with smooth-scroll fragment links to sections; an LCP hero image with `fetchpriority="high"` (not lazy); a features grid of `<article>` cards; a **FAQ** built from `<details>`/`<summary>` (exclusive accordion via shared `name`); a `popover` menu or tooltip; a newsletter form with native validation; full social/SEO `<head>` including JSON-LD structured data; a site `<footer>` with a secondary labeled `<nav>`. **Success criteria:** validates clean; Lighthouse scores high across Performance, Accessibility, SEO, and Best Practices; works (content readable, links/forms functional) with CSS or JS disabled; passes a keyboard-only walkthrough.

### 17.3 Where to go next

- **CSS_GUIDE.md** — make all of the above *look* good: the box model, Flexbox/Grid layout, responsive design, custom properties, animations.
- **JAVASCRIPT_GUIDE.md** — add behaviour: DOM manipulation, events, `fetch`, the APIs from §13, and web components.
- Then frameworks that build on HTML: **REACT_19_GUIDE.md**, **NEXTJS_16_GUIDE.md** (note JSX's stricter, self-closing syntax), and component libraries (**SHADCN_UI_CHEATSHEET.md**, **MATERIAL_UI_GUIDE.md**, **TAILWIND_CHEATSHEET.md**).
- For where forms *go*, see backend guides: **NODEJS_GUIDE.md**, **FASTIFY_GUIDE.md**, **NESTJS_GUIDE.md**, and the various Go API guides.

---

> **Final word:** HTML rewards a "meaning first" mindset. Before reaching for a `<div>`, ask "what *is* this content?" — there's usually a more meaningful element. Get the structure right and accessibility, SEO, and maintainability largely follow for free; presentation (CSS) and behaviour (JS) then layer cleanly on top. Keep DevTools open, validate often, test with the keyboard, and build, build, build.
