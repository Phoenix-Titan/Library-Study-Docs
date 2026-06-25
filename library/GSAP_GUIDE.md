# GSAP (GreenSock) — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I've heard GSAP is the gold standard for web animation" to "I ship production-grade, performant, maintainable animations in React and Next.js" — without an internet connection. Every concept is explained in prose first: *what it is*, *the logic / why it works this way*, *what it's for and when to reach for it*, *how to use it*, *the key parameters*, *best practices*, *performance*, and *the gotchas* — and only then a heavily-commented, runnable code block. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. This guide assumes you are comfortable with JavaScript and React basics — if not, read the **[JavaScript](JAVASCRIPT_GUIDE.md)** and **[React 19](REACT_19_GUIDE.md)** guides first. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **GSAP 3.13+** (the 3.x line, current in 2026), the official **`@gsap/react`** package with the **`useGSAP()`** hook, **React 19**, and **Next.js 16 (App Router)**. The single most important fact about GSAP in 2026:
> - **GSAP is now 100% FREE for everyone — including every plugin.** After **Webflow acquired GreenSock in 2024**, the entire toolset was released free of charge. The plugins that used to require a paid "Club GreenSock" membership — **SplitText, MorphSVG, DrawSVG, ScrollSmoother, MotionPathHelper, Physics2D, PhysicsProps, InertiaPlugin, GSDevTools, CustomEase, CustomBounce, CustomWiggle, ScrambleText**, and the rest — are now bundled in the standard `gsap` npm package. There is **no more paid tier, no special private npm registry, no auth token**. You `npm install gsap` and import everything. This guide uses the formerly-premium plugins freely.
> - **`@gsap/react`** is the official React adapter. Its **`useGSAP()`** hook is the modern, correct way to use GSAP in React: it wraps your animation code in a `gsap.context()` scoped to a ref, auto-reverts (cleans up) every animation/ScrollTrigger on unmount or dependency change, and survives React 18/19 StrictMode double-invocation. This guide centers the entire React/Next workflow on `useGSAP()`.
> - GSAP is **client-only** — it touches the DOM and `window`. In Next.js App Router every component that uses it needs **`"use client"`**, and you must guard against SSR/hydration mismatches. Section 9 covers this in depth.
>
> Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**; package-manager commands work identically across platforms. Confirm exact APIs at gsap.com/docs for changes after this guide's cutoff.

---

## Table of Contents

1. [What GSAP Is & Why](#1-what-gsap-is--why) **[B]**
2. [Setup, Install, Imports & Registering Plugins](#2-setup-install-imports--registering-plugins) **[B]**
3. [The Tween — `to` / `from` / `fromTo` / `set`](#3-the-tween--to--from--fromto--set) **[B]**
4. [Easing in Depth](#4-easing-in-depth) **[B/I]**
5. [Timelines — Sequencing & Orchestration](#5-timelines--sequencing--orchestration) **[I]**
6. [Staggers, Keyframes, Repeat & Yoyo](#6-staggers-keyframes-repeat--yoyo) **[I]**
7. [The React Integration Done Right — `useGSAP()`](#7-the-react-integration-done-right--usegsap) **[I/A]**
8. [ScrollTrigger — Scroll-Driven Animation](#8-scrolltrigger--scroll-driven-animation) **[I/A]**
9. [Next.js Specifics — `"use client"`, SSR & Hydration](#9-nextjs-specifics--use-client-ssr--hydration) **[I/A]**
10. [The Plugin Suite (All Free)](#10-the-plugin-suite-all-free) **[A]**
11. [The Flip Plugin with React](#11-the-flip-plugin-with-react) **[A]**
12. [Responsive & Accessible Animation](#12-responsive--accessible-animation) **[I/A]**
13. [Performance & Production](#13-performance--production) **[A]**
14. [Architecture & Maintainability](#14-architecture--maintainability) **[A]**
15. [Gotchas & Best Practices](#15-gotchas--best-practices) **[I/A]**
16. [Worked Example — A Production Scroll-Telling Page (Next.js 16)](#16-worked-example--a-production-scroll-telling-page-nextjs-16) **[A]**
17. [Study Path & Build-to-Learn Projects](#17-study-path--build-to-learn-projects)

---

## 1. What GSAP Is & Why

GSAP — the **GreenSock Animation Platform** — is a JavaScript library for animating *anything* a browser can change: DOM elements, CSS properties, SVG, canvas, WebGL objects, even plain JavaScript object properties. It is the most battle-tested, widely-deployed animation engine on the web; it powers award-winning marketing sites, interactive storytelling, game UIs, and product launches. Before you write a single tween, it pays to understand *what GSAP actually does under the hood* and *why* you would reach for it over the alternatives, because that decision shapes everything downstream.

### 1.1 What GSAP actually is — a property tweening engine [B]

At its core GSAP is a **tweening engine**. "Tween" is short for "in-between": you tell GSAP the *end state* of some properties ("move this box to `x: 300` and fade it to `opacity: 0.5`"), and on every animation frame GSAP computes the *in-between* values — interpolating from where the property is now to where you said it should end — and writes them to the DOM. It does this inside a single, highly-optimised `requestAnimationFrame` loop (GSAP's "ticker") that drives *every* animation on the page. That central loop is a big part of why GSAP is smooth: instead of dozens of independent timers fighting each other, there is one heartbeat synchronising everything.

Critically, **GSAP does not care what kind of value it is animating.** It can tween a CSS `transform`, an SVG `stroke-dashoffset`, a hex color, a `<canvas>` particle's `radius`, or `myObject.score` from `0` to `1000`. It just reads a starting number, interpolates toward a target number applying an *ease*, and calls a setter. This generality is GSAP's superpower and the reason it spans use cases that CSS and most React animation libraries cannot.

### 1.2 GSAP vs CSS animations & transitions [B]

You do *not* always need GSAP. Knowing where plain CSS suffices keeps your bundle small. Here is the decision logic:

| Approach | Best for | Limitation |
|---|---|---|
| **CSS `transition`** | Simple two-state changes you can express in CSS: hover color, a button growing on `:hover`, a class-toggle slide | Only two states; no sequencing; can't pause/reverse/seek; no physics; no callbacks; hard to drive from JS data |
| **CSS `@keyframes`** | Looping decorative animations independent of app state: spinners, pulses, marquees | Fixed at author time; clumsy to coordinate with JS/React state; no fine control; no easing beyond `cubic-bezier` |
| **GSAP** | Anything that needs *control* (play/pause/reverse/seek/timeScale), *sequencing* (timelines), *scroll-linking*, *physics-based inertia*, *SVG morphing/drawing*, *text splitting*, *layout transitions*, callbacks, or runtime-computed values | ~`40–70 KB` of JS (core + plugins you use); client-only |

**The logic / why a library:** CSS transitions and keyframes are *fire-and-forget*. Once a CSS animation starts you cannot, from JavaScript, smoothly say "reverse halfway through" or "jump to 40% and scrub from there with the mouse." You also cannot orchestrate ten elements into a precisely-timed sequence where element 3 starts 0.2s before element 2 finishes. GSAP gives you a **timeline**: a controllable, seekable container of tweens with frame-accurate positioning. It also handles the cross-browser transform quirks, sub-pixel rounding, and unit conversions that make hand-rolled animation miserable. And via its plugins it does things CSS simply *can't*: morph one SVG path into another, draw an SVG stroke on, snap a scroll, throw an element with realistic momentum.

### 1.3 GSAP vs Motion (Framer Motion) — the honest comparison [B/I]

Both GSAP and **[Motion](MOTION_ANIMATION_GUIDE.md)** (the library formerly called Framer Motion) are excellent. They make different bets, and a senior developer picks per-project.

| | **Motion** | **GSAP** |
|---|---|---|
| **Paradigm** | **Declarative** — you describe states as React props (`initial`/`animate`/`exit`), the library diffs and animates | **Imperative** — you call functions (`gsap.to(...)`) and build timelines; more like scripting |
| **React fit** | Born in React; `motion.div`, `AnimatePresence`, layout animations feel native to JSX | Framework-agnostic core + the `@gsap/react` adapter; you animate **refs**, not components |
| **Sequencing** | Variants + `staggerChildren`; good, but complex sequences get awkward | **Timelines** are best-in-class — precise, controllable, scrubbable |
| **Scroll** | `useScroll` + `useTransform`; solid for progress-linked effects | **ScrollTrigger** — the industry standard for pinning, scrubbing, snapping, batching |
| **Plugins** | Smaller surface; layout/shared-element built in | Huge free suite: SplitText, MorphSVG, DrawSVG, Flip, MotionPath, Inertia, ScrollSmoother |
| **Exit animations** | First-class via `AnimatePresence` | Manual — you run an animation, then call your unmount in `onComplete` |
| **Mental model** | "Describe the UI per state" (very React) | "Script the motion over time" (very animator) |

**When to pick which:** Reach for **Motion** when your animations map cleanly onto React state (enter/exit on mount/unmount, simple gesture and layout transitions) and you want them to read like declarative JSX. Reach for **GSAP** when you need *timeline orchestration*, *scroll-driven storytelling* (ScrollTrigger), *SVG morphing/drawing*, *text-splitting*, *physics/inertia*, *FLIP layout transitions across arbitrary DOM*, or fine imperative control (scrubbing to a point, reversing mid-flight, syncing to audio). Many production codebases use **both**: Motion for component-level UI transitions, GSAP for the hero/scroll showpieces. They coexist fine. The comparison is covered again from Motion's side in the **[Motion guide](MOTION_ANIMATION_GUIDE.md)**.

### 1.4 The 2026 landscape — GSAP is now completely free [B]

For most of its life GSAP's core was free but its most powerful plugins (SplitText, MorphSVG, DrawSVG, ScrollSmoother, the physics plugins, InertiaPlugin, GSDevTools) lived behind a paid **"Club GreenSock"** membership, distributed through a private npm registry that needed an auth token. **That era is over.** Webflow acquired GreenSock in 2024 and made the **entire platform free for everyone, commercial use included.** As of GSAP 3.13:

- Every plugin ships inside the public `gsap` package on the **standard npm registry**. No token, no `.npmrc` edits, no Club account.
- You `import { SplitText } from "gsap/SplitText"` exactly the way you import `ScrollTrigger`.
- Code you wrote against the old Club registry should be migrated to the public package; the APIs are the same.

**⚡ Version note:** If you find a tutorial telling you to add a private GreenSock registry to `.npmrc` or to buy a membership to use SplitText/MorphSVG/ScrollSmoother — it predates the 2024 acquisition. Ignore it. Everything is in `npm install gsap`.

### 1.5 Cross-references [B]

GSAP touches CSS heavily (it animates transforms and colors — see the **[CSS](CSS_GUIDE.md)** guide for the rendering pipeline). In React you'll combine it with refs and effects (**[React 19](REACT_19_GUIDE.md)**), and in Next.js the `"use client"` boundary is a recurring theme (**[Next.js 16](NEXTJS_16_GUIDE.md)**). If you also use **[Tailwind CSS](TAILWIND_CHEATSHEET.md)**, GSAP animates the rendered element regardless of how its initial styles were authored — they compose fine.

---

## 2. Setup, Install, Imports & Registering Plugins

Getting GSAP into a project is genuinely simple in 2026 — one package, no auth. The two things beginners stumble on are **importing plugins from their submodule paths** and **registering plugins before use**. Get those right and everything works.

### 2.1 Installing [B]

GSAP is a single npm package that contains the core *and* every plugin:

```bash
# That's the whole install. Core + ALL plugins, free, public registry.
npm install gsap

# For React/Next, also install the official React adapter:
npm install @gsap/react
```

There is nothing else to configure — no token, no special registry, no peer-dependency dance beyond having React installed for `@gsap/react`.

### 2.2 Importing the core and registering plugins [B]

GSAP's core lives at the package root; **each plugin lives at its own submodule path** (`gsap/ScrollTrigger`, `gsap/SplitText`, etc.). Plugins must be **registered once** with `gsap.registerPlugin(...)` before you use them. Registration is what wires the plugin's capabilities into the core engine; forget it and you'll get a console warning and a no-op tween.

```js
import { gsap } from "gsap";                       // the core engine
import { ScrollTrigger } from "gsap/ScrollTrigger"; // a plugin (submodule path)
import { SplitText } from "gsap/SplitText";         // formerly premium, now free
import { Flip } from "gsap/Flip";

// Register ONCE per app — typically at module top-level or in an app-init file.
// Registering the same plugin twice is harmless, but do it in a single place.
gsap.registerPlugin(ScrollTrigger, SplitText, Flip);

// Now the core knows about these plugins and tweens can use their features.
```

**Best practice:** register plugins exactly once, in a place that runs before any animation. In React/Next, the cleanest spot is *inside* the component (or a shared init module) guarded by `"use client"` — never at the top of a file that a Server Component might import (see Section 9). The `useGSAP` hook does **not** auto-register plugins; you still call `registerPlugin` yourself.

### 2.3 Which CSS properties GSAP can animate, and `gsap.set` for initial state [B]

GSAP's CSS plugin is built into the core (you don't import it separately). It understands `x`/`y`/`rotation`/`scale` as shorthands for `transform`, handles colors, units, and even `autoAlpha` (opacity + `visibility` together). More on this in Section 3.

### 2.4 The CDN note (vanilla / non-bundler) [B]

If you're dropping GSAP into a plain HTML page with no bundler (a quick CodePen, a static demo, a `.html` file you open locally), you can load it from a CDN as global UMD scripts. In that mode there is no `import`; `gsap`, `ScrollTrigger`, etc. become globals, and you still call `gsap.registerPlugin(ScrollTrigger)`.

```html
<!-- Vanilla / no-bundler usage. Core + a plugin as global scripts. -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.13.0/gsap.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.13.0/ScrollTrigger.min.js"></script>
<script>
  // No imports — gsap and ScrollTrigger are global UMD objects here.
  gsap.registerPlugin(ScrollTrigger);
  gsap.to(".box", { x: 200, duration: 1 });
</script>
```

**For React/Next you should use the npm package**, not the CDN — bundling lets tree-shaking drop unused plugins and gives you types. The CDN is for prototyping and no-build environments.

### 2.5 A minimal vanilla example end-to-end [B]

```js
import { gsap } from "gsap";

// Animate every element with class "card": move up 40px and fade in over 0.6s.
// `gsap.to` animates FROM current state TO the values you specify.
gsap.to(".card", {
  y: -40,          // translateY by -40px (transform — GPU-friendly)
  opacity: 1,      // fade to fully visible
  duration: 0.6,   // seconds (GSAP uses seconds, not ms)
  ease: "power2.out",
});
```

That is a complete, working GSAP animation. The rest of this guide is about doing this *well* — controllably, in sequences, scroll-driven, and cleaned up properly inside React.

---

## 3. The Tween — `to` / `from` / `fromTo` / `set`

The **tween** is the atom of GSAP. Everything — every timeline, every ScrollTrigger animation — is ultimately made of tweens. A tween animates one or more **targets** through a set of **vars** (the properties to animate plus configuration like `duration` and `ease`). There are four creation methods, and choosing the right one is half of writing clean GSAP.

### 3.1 The four methods and what each is for [B]

- **`gsap.to(targets, vars)`** — animate **from the current state TO** the values in `vars`. The everyday workhorse. "Move this to x:300."
- **`gsap.from(targets, vars)`** — animate **FROM the values in `vars` TO the current state**. Ideal for entrance animations: you set the *starting* offset/opacity and GSAP brings the element to its natural CSS position. "Start 50px low and invisible, arrive at rest."
- **`gsap.fromTo(targets, fromVars, toVars)`** — you specify **both** ends explicitly. Use when you can't rely on the current state (e.g. re-running an animation, or when the start must be deterministic regardless of where the element currently is).
- **`gsap.set(targets, vars)`** — apply values **instantly** (a zero-duration tween). For establishing initial state before animating, or jumping something into place.

```js
// to: current → specified. (most common)
gsap.to(".box", { x: 300, rotation: 360, duration: 1 });

// from: specified → current. (great for entrances — element ends at its CSS position)
gsap.from(".hero h1", { y: 50, opacity: 0, duration: 0.8 });

// fromTo: explicit both ends. (deterministic, re-runnable)
gsap.fromTo(".bar",
  { scaleX: 0 },            // from
  { scaleX: 1, duration: 1 } // to
);

// set: instant, no animation. (initialize state)
gsap.set(".panel", { autoAlpha: 0, y: 20 }); // hidden, shifted — ready to animate in later
```

**The `from`/`fromTo` FOUC gotcha:** because `gsap.from` reads the element's *current* (final) state as the destination, there is a brief moment before the tween's first frame where the element is rendered at its natural position. With entrance animations this can cause a flash (FOUC — flash of unstyled/unanimated content). The fix is to set the hidden state *immediately* (in CSS or via `gsap.set`) so the element is never visible before the animation, then use `to`/`fromTo`. This matters a lot in React/Next; Section 15 returns to it.

### 3.2 Targets — strings, elements, refs, arrays [B]

The first argument can be:
- a **CSS selector string** (`".card"`, `"#hero"`) — GSAP queries the DOM;
- a **DOM element** or **array of elements** (`document.querySelector(...)`, `ref.current`);
- a **NodeList / array** — GSAP animates all of them;
- in React, **always prefer refs** over selector strings (see Section 7) so animations are scoped and survive re-renders.

### 3.3 The vars object — the animatable properties [B]

Inside `vars` you mix **properties to animate** with **special configuration keys**. The most-used animatable properties:

| Property | Animates | Notes |
|---|---|---|
| `x`, `y` | `transform: translate()` | Pixels by default; GPU-friendly. Use these, not `left`/`top`. |
| `xPercent`, `yPercent` | translate by % of own size | Great for centering / responsive offsets |
| `rotation` | `transform: rotate()` | Degrees. Also `rotationX`, `rotationY` (3D) |
| `scale`, `scaleX`, `scaleY` | `transform: scale()` | `1` = natural size |
| `skewX`, `skewY` | `transform: skew()` | Degrees |
| `opacity` | opacity | `0`–`1` |
| `autoAlpha` | opacity **+** `visibility` | `0` also sets `visibility:hidden` (removes from hit-testing) — preferred for show/hide |
| `backgroundColor`, `color`, `borderColor` | colors | Accepts hex, rgb, hsl, named |
| `width`, `height`, `top`, `left` | layout box | **Triggers layout/reflow — avoid in hot paths**; prefer transforms |
| `xPercent`/`x` combo | offset by % then px | e.g. `xPercent: -50, x: 10` |

```js
gsap.to(".card", {
  x: 120,                 // transform translateX
  y: -30,
  rotation: 15,           // degrees
  scale: 1.1,
  backgroundColor: "#f43f5e",
  borderRadius: "16px",   // GSAP parses units in strings
  duration: 0.7,
});
```

### 3.4 Units and relative values [B]

Numbers default to pixels for length properties and degrees for rotations. You can pass **strings with explicit units** (`"50%"`, `"3rem"`, `"45deg"`) and **relative values** with `"+="`/`"-="`/`"*="`:

```js
gsap.to(".box", { x: "+=100" });   // move 100px RIGHT of wherever it is now
gsap.to(".box", { rotation: "-=90" }); // rotate 90deg counter-clockwise from current
gsap.to(".box", { width: "50%" });  // animate width to 50% of parent
```

### 3.5 The core configuration keys [B]

These keys in `vars` are *not* animated — they configure the tween:

| Key | Type | Meaning |
|---|---|---|
| `duration` | number (s) | How long the tween runs. Default `0.5`. |
| `delay` | number (s) | Wait this long before starting. |
| `ease` | string/fn | The acceleration curve (Section 4). Default `"power1.out"`. |
| `repeat` | number | Times to repeat after first play. `-1` = infinite. |
| `yoyo` | boolean | On repeat, alternate direction (forward, back, forward…). |
| `repeatDelay` | number | Pause between repeats. |
| `stagger` | number/object | Offset start times across multiple targets (Section 6). |
| `paused` | boolean | Create the tween but don't auto-play (call `.play()` later). |
| `onComplete` | function | Callback when the tween finishes. |
| `onStart`, `onUpdate`, `onRepeat`, `onReverseComplete` | function | Lifecycle callbacks. |
| `overwrite` | `true`/`"auto"`/`false` | How to handle conflicting tweens on the same property (Section 15). |

```js
gsap.to(".loader", {
  rotation: 360,
  duration: 1,
  ease: "none",      // linear — constant speed spinner
  repeat: -1,        // forever
  // onComplete won't fire (infinite), but onUpdate fires every frame:
  onUpdate() { /* runs ~60×/sec while animating */ },
});
```

### 3.6 The tween instance — control after creation [B/I]

Every `gsap.to/from/fromTo` **returns a Tween instance** you can hold onto and control:

```js
const tween = gsap.to(".box", { x: 300, duration: 2, paused: true });

tween.play();        // start
tween.pause();       // freeze
tween.reverse();     // run backwards
tween.restart();     // back to 0 and play
tween.seek(1);       // jump to the 1-second mark
tween.progress(0.5); // jump to 50% (getter/setter)
tween.timeScale(2);  // play at 2× speed
tween.kill();        // stop and remove (frees it from the ticker)
```

This control surface is exactly what CSS cannot give you, and it's the foundation for interactive and scroll-driven work.

---

## 4. Easing in Depth

**Easing** is the single biggest lever on how an animation *feels*. An ease maps linear time (0→1) onto a non-linear progress curve, so an element can start slow and finish fast, overshoot and settle, or bounce. Default browser motion looks robotic precisely because it's linear or a generic `ease`. GSAP ships a rich, named ease vocabulary and lets you author your own.

### 4.1 The mental model — what an ease *is* [B]

Imagine the animation's timeline on the X axis (0% → 100% of the duration) and the *progress* of the property on the Y axis (0% → 100% of the change). A **linear** ease is a straight diagonal line: equal time, equal progress. An **ease-out** is steep at the start and flattens at the end — fast then slowing — which feels like something decelerating to rest (the most natural for UI entrances). An **ease-in** is the reverse (slow start, fast finish), good for exits. An **ease-in-out** is slow-fast-slow. Picturing this curve is the "ease visualizer" concept — GSAP's website has an interactive one, but the curve in your head is enough.

### 4.2 The named ease families [B]

GSAP eases are written `"family.type"` where `type` is `in`, `out`, or `inOut`:

| Family | Feel | Typical use |
|---|---|---|
| `none` (a.k.a. `"linear"`) | constant speed | spinners, scroll-scrub, marquees |
| `power1` … `power4` | progressively sharper curves (quad, cubic, quart, quint) | the everyday workhorses; `power2.out` for entrances |
| `back` | overshoots past the target then settles | playful pops, buttons, badges |
| `elastic` | springs past and oscillates | bouncy, attention-grabbing |
| `bounce` | bounces like a dropped ball at the end | drops, landing effects |
| `circ` | circular curve, gentle then sharp | smooth, organic moves |
| `expo` | extreme acceleration/deceleration | dramatic reveals |
| `sine` | gentle sinusoidal | subtle, calm motion |
| `steps(n)` | jumps in `n` discrete steps | sprite sheets, typewriter, ticking |

```js
gsap.to(".pop", { scale: 1, duration: 0.6, ease: "back.out(1.7)" });
// back.out(1.7): overshoots; the number controls overshoot strength.

gsap.to(".ball", { y: 300, duration: 1, ease: "bounce.out" });

gsap.to(".spinner", { rotation: 360, duration: 1, ease: "none", repeat: -1 });

gsap.to(".sprite", { backgroundPosition: "-960px 0", duration: 1, ease: "steps(12)" });
```

**Configurable eases:** several families accept a parameter. `back.out(amount)` controls overshoot; `elastic.out(amplitude, period)` controls how far and how fast it oscillates; `steps(n)` sets the count.

### 4.3 Choosing an ease — practical guidance [B/I]

- **UI entering** (cards, modals, text in): `power2.out` or `power3.out` — decelerate to rest, feels responsive.
- **UI leaving**: `power2.in` — accelerate away.
- **Both ends visible** (a slide that moves and stops on screen): `power2.inOut`.
- **Playful / branded** pops: `back.out(1.7)`.
- **Scroll-scrubbed** animations: almost always `none` (linear) so the motion tracks the scrollbar 1:1.
- **Avoid** `elastic`/`bounce` for frequent/utility animations — they're attention-grabbing and tire users fast.

### 4.4 Custom eases — CustomEase, CustomBounce, CustomWiggle (now free) [I]

When the named eases don't capture the exact feel you want, **CustomEase** lets you define an ease from an SVG-path-like string (the same format design tools export) or a cubic-bezier. **CustomBounce** and **CustomWiggle** generate bounce/wiggle curves with fine control. All three were Club-only before 2024 and are **now free**.

```js
import { gsap } from "gsap";
import { CustomEase } from "gsap/CustomEase";
gsap.registerPlugin(CustomEase);

// Define a reusable named ease from a path. Name it once, reference by name anywhere.
CustomEase.create("myEase", "M0,0 C0.25,0.1 0.25,1 1,1");

gsap.to(".box", { x: 400, duration: 1, ease: "myEase" });
```

```js
import { CustomBounce } from "gsap/CustomBounce";
gsap.registerPlugin(CustomEase, CustomBounce); // CustomBounce needs CustomEase

// Generates "myBounce" (the motion) and "myBounce-squash" (the squash-and-stretch).
CustomBounce.create("myBounce", { strength: 0.6, squash: 3 });
gsap.to(".ball", { y: 400, duration: 2, ease: "myBounce" });
gsap.to(".ball", { scaleX: 1.4, scaleY: 0.6, duration: 2, ease: "myBounce-squash" });
```

**Best practice:** define custom eases once at app init (so the name resolves everywhere) and reference them by string. Don't recreate the same CustomEase on every render.

---

## 5. Timelines — Sequencing & Orchestration

A single tween animates one moment. Real interfaces need *choreography*: the logo fades, then the nav items stagger in, then the hero text rises, then a button pops — with precise overlaps. The **Timeline** is GSAP's orchestration container, and it's the feature that most clearly separates GSAP from CSS and from simpler libraries. Mastering timelines is the inflection point from "I can make things move" to "I can direct a sequence."

### 5.1 What a timeline is and why it beats chained delays [I]

A timeline is a **container of tweens (and nested timelines) arranged on a shared playhead.** Instead of manually computing `delay` for each tween (fragile — change one duration and every later delay is wrong), you **append** tweens and let the timeline track time for you. The whole timeline is itself controllable: `play/pause/reverse/seek/timeScale/progress` apply to the entire sequence as one unit. You can reverse a ten-step sequence with one call.

```js
// Create a timeline. Default vars (like ease/duration) can be shared via `defaults`.
const tl = gsap.timeline({ defaults: { duration: 0.6, ease: "power2.out" } });

// Append tweens. By default each starts when the previous ENDS (sequential).
tl.from(".logo", { opacity: 0, y: -20 })
  .from(".nav li", { opacity: 0, y: -10, stagger: 0.1 })
  .from(".hero h1", { opacity: 0, y: 40 })
  .from(".cta", { scale: 0, ease: "back.out(1.7)" });

// Change ANY duration above and the rest still sequence correctly — no manual delays.
```

### 5.2 The position parameter — the heart of timeline control [I]

Each `tl.to/from/fromTo/add` accepts an optional **position parameter** as its last argument, controlling *where on the timeline* the tween is placed. This is the most important timeline concept:

| Position value | Meaning |
|---|---|
| *(omitted)* | At the **end** of the timeline (sequential — the default). |
| `0` (a number) | At an absolute time in seconds from the start. |
| `"+=0.5"` | `0.5s` **after** the timeline's current end (a gap). |
| `"-=0.5"` | `0.5s` **before** the current end (an **overlap** — the most-used). |
| `"<"` | At the **start** of the *previous* tween. |
| `">"` | At the **end** of the *previous* tween (same as default). |
| `"<0.2"` | `0.2s` after the start of the previous tween. |
| `">-0.3"` | `0.3s` before the end of the previous tween. |
| `"myLabel"` | At a named **label** (see below). |
| `"myLabel+=0.3"` | Relative to a label. |

```js
const tl = gsap.timeline();
tl.to(".a", { x: 100, duration: 1 })
  .to(".b", { y: 100, duration: 1 }, "-=0.5")  // start 0.5s before .a finishes (overlap)
  .to(".c", { rotation: 360, duration: 1 }, "<") // start at the SAME time as .b
  .to(".d", { opacity: 0, duration: 1 }, "+=0.25"); // 0.25s gap after .c ends
```

**Labels** are named time markers you can jump to and position against — invaluable for keeping a long sequence readable and seekable:

```js
const tl = gsap.timeline();
tl.addLabel("intro")
  .from(".title", { opacity: 0, y: 30 }, "intro")
  .addLabel("reveal", "+=0.2")
  .from(".image", { scale: 0.8, opacity: 0 }, "reveal")
  .from(".caption", { opacity: 0 }, "reveal+=0.15");

tl.seek("reveal"); // jump the playhead straight to the reveal moment
```

### 5.3 Nesting timelines [I]

A timeline can contain other timelines via `tl.add(childTimeline, position)`. This is how you build large sequences from reusable, self-contained pieces (a "hero intro" timeline, a "card grid" timeline) and then arrange those pieces on a master timeline. Each child stays controllable on its own, and the parent controls them all.

```js
function makeIntro() {
  const tl = gsap.timeline();
  tl.from(".logo", { opacity: 0 }).from(".tagline", { y: 20, opacity: 0 });
  return tl; // return a reusable timeline
}

const master = gsap.timeline();
master.add(makeIntro())                    // nest the intro
      .add(makeCardsTimeline(), "-=0.3")   // then the cards, overlapping slightly
      .to(".bg", { backgroundColor: "#111" }, 0); // and a bg shift from the very start
```

### 5.4 Controlling timelines [I]

The control API mirrors tweens — but now it drives the whole sequence:

```js
const tl = gsap.timeline({ paused: true });
// ...build it...

tl.play();
tl.pause();
tl.reverse();          // run the entire sequence backwards
tl.restart();
tl.seek(2);            // jump to 2s
tl.seek("reveal");     // jump to a label
tl.progress(0.5);      // jump to 50% of total duration
tl.timeScale(1.5);     // 1.5× speed (or 0.5 for slow-mo)
tl.duration();         // total computed duration (read-only-ish)
tl.kill();             // tear down
```

**Best practice:** build timelines **paused** when they're triggered by events or scroll, and `.play()`/`.reverse()` them in response. In React, keep the timeline in a ref (Section 7) so handlers can control it.

---

## 6. Staggers, Keyframes, Repeat & Yoyo

Three features that make multi-element and multi-step animation concise: **stagger** (offset many elements), **keyframes** (multi-step within one tween), and **repeat/yoyo** (looping).

### 6.1 Stagger — animating many targets with offset timing [I]

When a tween has multiple targets, **`stagger`** offsets each one's start time, producing the classic cascade (list items flying in one after another). In its simplest form it's a number (seconds between each). As an object it unlocks distribution, direction, and grid-aware staggering.

```js
// Simple: each .card starts 0.1s after the previous.
gsap.from(".card", { y: 40, opacity: 0, duration: 0.5, stagger: 0.1 });

// Advanced: object form.
gsap.from(".card", {
  y: 40, opacity: 0, duration: 0.5,
  stagger: {
    each: 0.1,          // 0.1s between each (use `amount` to spread a TOTAL time instead)
    from: "center",     // origin: "start" | "center" | "end" | "edges" | index | [x,y]
    grid: "auto",       // treat targets as a grid for 2D stagger
    ease: "power2.in",  // distribute the stagger timing with its own ease
  },
});
```

`each` vs `amount`: `each: 0.1` means a fixed gap per item (total grows with count). `amount: 1` means "spread the *whole* stagger across 1 second total no matter how many items" (gap shrinks as count grows). Use `amount` when you want a consistent overall duration regardless of list length.

### 6.2 Keyframes — multiple steps in one tween [I]

Sometimes you want one element to pass through several states in sequence without writing three tweens. **Keyframes** express that inside a single tween. Two syntaxes:

```js
// Array syntax: each object is a step; can carry its own duration/ease/delay.
gsap.to(".box", {
  keyframes: [
    { x: 100, duration: 0.4 },
    { y: 100, duration: 0.4, ease: "power2.inOut" },
    { x: 0, y: 0, duration: 0.6 },
  ],
});

// Object syntax: each PROPERTY gets an array of values spread across the duration.
gsap.to(".box", {
  keyframes: {
    x: [0, 100, 100, 0],
    y: [0, 0, 100, 0],
    ease: "power1.inOut",
  },
  duration: 2,
});
```

Keyframes keep a multi-step motion as a single controllable tween (one item on a timeline) instead of three.

### 6.3 Repeat, yoyo & repeatDelay [I]

`repeat` loops a tween or timeline; `yoyo` makes alternate iterations run backwards (so it returns smoothly instead of snapping back); `repeatDelay` pauses between iterations.

```js
gsap.to(".pulse", {
  scale: 1.2,
  duration: 0.8,
  ease: "sine.inOut",
  repeat: -1,        // infinite
  yoyo: true,        // grow then shrink then grow… (a breathing pulse)
  repeatDelay: 0.2,  // brief pause at each end
});
```

**Performance note:** an infinite `repeat: -1` tween runs *forever* and keeps the ticker busy. In React, make sure such tweens are cleaned up on unmount — `useGSAP` does this automatically (Section 7). For purely decorative infinite loops with no JS logic, a CSS `@keyframes` animation may be lighter.

---

## 7. The React Integration Done Right — `useGSAP()`

This is the section that makes the difference between a demo and a production app. GSAP itself is framework-agnostic; the challenge in React is **lifecycle and cleanup**. React mounts, re-renders, and unmounts components — and in development **StrictMode mounts every component twice** to surface bugs. If you create tweens/ScrollTriggers carelessly, you leak them: duplicate animations stack up, ScrollTriggers from old renders keep firing, infinite tweens never stop, and route changes orphan animations pointing at removed DOM. The official **`@gsap/react`** package and its **`useGSAP()`** hook exist to solve exactly this.

### 7.1 The problem, concretely — why naive code leaks [I/A]

Consider the obvious-but-wrong approach:

```jsx
// ⚠️ ANTI-PATTERN — leaks in StrictMode, on re-render, and on unmount.
function Box() {
  const ref = useRef(null);
  useEffect(() => {
    gsap.to(ref.current, { x: 300, repeat: -1, yoyo: true });
    // No cleanup! In StrictMode this runs TWICE → two stacked tweens.
    // On unmount the infinite tween keeps running against a detached node.
  }, []);
  return <div ref={ref} className="box" />;
}
```

The historically-correct manual fix was `gsap.context()` plus a cleanup function:

```jsx
// The OLD correct way — useful to understand, but verbose. `useGSAP` automates it.
function Box() {
  const container = useRef(null);
  useEffect(() => {
    const ctx = gsap.context(() => {
      gsap.to(".box", { x: 300, repeat: -1, yoyo: true });
    }, container); // scope selectors to this container
    return () => ctx.revert(); // cleanup: kills tweens, reverts inline styles
  }, []);
  return <div ref={container}><div className="box" /></div>;
}
```

`gsap.context()` does two things: it **scopes selector strings** to a container element (so `".box"` only matches inside *this* component, not the whole page), and it **records every animation created inside it** so a single `ctx.revert()` can tear them all down. `useGSAP` wraps this pattern into a hook so you stop writing the boilerplate — and so cleanup is never forgotten.

### 7.2 `useGSAP()` — the modern, correct way [I/A]

`useGSAP(callback, options)` runs your animation code inside a `gsap.context()` and **automatically reverts that context** on unmount and whenever the dependency array changes. It's a drop-in replacement for the `useEffect` + `context` + cleanup pattern.

```jsx
"use client"; // required in Next.js App Router — GSAP is client-only

import { useRef } from "react";
import { gsap } from "gsap";
import { useGSAP } from "@gsap/react";

function Hero() {
  const container = useRef(null); // the scope element

  useGSAP(
    () => {
      // Selector strings here are SCOPED to `container` — ".title" only matches
      // inside this component. Every animation created here is auto-cleaned up.
      gsap.from(".title", { y: 40, opacity: 0, duration: 0.8 });
      gsap.from(".subtitle", { y: 20, opacity: 0, duration: 0.8, delay: 0.2 });
    },
    { scope: container } // <-- scope ties selectors + cleanup to this ref
  );

  return (
    <section ref={container}>
      <h1 className="title">Welcome</h1>
      <p className="subtitle">Built with GSAP</p>
    </section>
  );
}
```

**What you get for free:** StrictMode-safe (the double-mount no longer double-animates, because the first context is reverted before the second runs), automatic cleanup on unmount (no leaked ScrollTriggers or infinite tweens), and scoped selectors (no accidental cross-component matches).

### 7.3 `scope`, dependencies, and the options object [I/A]

`useGSAP(callback, options)` options:

| Option | Purpose |
|---|---|
| `scope` | A ref to the container element. Scopes selector strings *and* defines what gets reverted. **Always set this.** |
| `dependencies` | Array like `useEffect`'s deps. When a value changes, the context reverts and the callback re-runs. Default `[]` (run once). |
| `revertOnUpdate` | If `true`, fully revert (including inline styles) before re-running when deps change. Default `false` (it re-runs but doesn't roll back prior styles). |

```jsx
function ProgressBar({ value }) {
  const ref = useRef(null);
  // Re-run the animation whenever `value` changes; context reverts between runs.
  useGSAP(
    () => {
      gsap.to(".fill", { scaleX: value / 100, duration: 0.4, ease: "power2.out" });
    },
    { scope: ref, dependencies: [value] }
  );
  return <div ref={ref} className="track"><div className="fill" /></div>;
}
```

**⚡ Version note:** `useGSAP` follows React's deps rules — list every reactive value the callback reads. Passing the alternate object signature `useGSAP(() => {...}, { dependencies: [...], scope })` is the current recommended form.

### 7.4 `contextSafe` — animating from event handlers [I/A]

`useGSAP`'s callback runs once on mount (and on dep changes). But event handlers — a click, a hover — fire *later*, **outside** the context. Animations you create directly inside a handler are **not** recorded by the context and therefore **won't be cleaned up**. The fix is **`contextSafe`**, returned from `useGSAP`. Wrap any animation-creating handler with it, and the animations it creates join the context (and get cleaned up).

```jsx
function Button() {
  const container = useRef(null);

  // Destructure contextSafe from the hook's return value.
  const { contextSafe } = useGSAP({ scope: container });

  // Wrap the handler so animations created on click are tracked + cleaned up.
  const onClick = contextSafe(() => {
    gsap.to(".dot", { x: "random(-100, 100)", y: "random(-100, 100)", duration: 0.5 });
  });

  return (
    <div ref={container}>
      <button onClick={onClick}>Move</button>
      <span className="dot" />
    </div>
  );
}
```

**Rule of thumb:** animations that run on mount → put them in the `useGSAP` callback. Animations that run in response to user events → wrap the handler in `contextSafe`. This one rule prevents the most common React+GSAP leak.

### 7.5 Refs over selectors — and when each is fine [I/A]

Inside `useGSAP` with a `scope`, **selector strings are safe** because they're scoped to your container. But **refs are more robust** for single, specific elements: they survive className changes, don't depend on the DOM being queryable at the exact moment, and are the only sane option when you need *the specific instance* (e.g. one item in a `.map`). A common pattern is a ref for the scope/container plus selector strings for groups inside it, and explicit refs for elements you address individually.

```jsx
function Card() {
  const root = useRef(null);
  const badge = useRef(null);

  useGSAP(() => {
    gsap.from(".line", { opacity: 0, y: 10, stagger: 0.1 }); // group → scoped selector
    gsap.from(badge.current, { scale: 0, ease: "back.out(2)" }); // single → ref
  }, { scope: root });

  return (
    <article ref={root}>
      <span ref={badge} className="badge">New</span>
      <p className="line">Line one</p>
      <p className="line">Line two</p>
    </article>
  );
}
```

### 7.6 Holding a timeline in a ref for control [I/A]

When handlers must control a timeline (play on hover, reverse on leave), create it inside `useGSAP` and store it in a ref so handlers can reach it. Because it was created inside the context, it's cleaned up automatically.

```jsx
function Menu() {
  const root = useRef(null);
  const tl = useRef(null);

  useGSAP(() => {
    tl.current = gsap.timeline({ paused: true })
      .to(".panel", { height: "auto", duration: 0.4, ease: "power2.out" })
      .from(".item", { opacity: 0, y: 10, stagger: 0.05 }, "-=0.2");
  }, { scope: root });

  return (
    <nav ref={root}
         onMouseEnter={() => tl.current.play()}
         onMouseLeave={() => tl.current.reverse()}>
      {/* ...panel + items... */}
    </nav>
  );
}
```

---

## 8. ScrollTrigger — Scroll-Driven Animation

**ScrollTrigger** is GSAP's most celebrated plugin and the reason much of the "award-winning website" aesthetic exists: animations that play, pin, scrub, and snap as you scroll. It connects any tween or timeline to the scroll position with frame-accurate precision and excellent performance. It is also the plugin most prone to cleanup and SSR mistakes, so this section pairs the features with the React/Next discipline from Sections 7 and 9.

### 8.1 The mental model [I/A]

ScrollTrigger watches a **trigger element** as it passes through the viewport and maps that travel to an animation. You define **where the relationship starts** (`start`) and **ends** (`end`) in terms of "this point on the trigger" meeting "this point on the viewport," and choose what happens in between: either **scrub** (tie the animation's playhead directly to scroll progress) or **toggleActions** (play/reverse the animation at the boundaries). It can also **pin** the trigger (lock it in place while you scroll "through" it) and **snap** the scroll to defined points.

### 8.2 Core configuration [I/A]

```js
gsap.to(".panel", {
  x: -1000,
  ease: "none",
  scrollTrigger: {
    trigger: ".panel",     // element whose position drives the animation
    start: "top top",      // "[trigger point] [viewport point]": trigger's top hits viewport's top
    end: "+=2000",         // 2000px of scroll after start (or "bottom bottom", etc.)
    scrub: 1,              // tie playhead to scroll; number = smoothing seconds
    pin: true,             // lock the trigger in place during the scroll range
    markers: true,         // DEV ONLY — draws start/end/scroller markers
    toggleActions: "play none none reverse", // used when scrub is OFF
  },
});
```

| Key | Meaning |
|---|---|
| `trigger` | The element ScrollTrigger watches. |
| `start` / `end` | `"triggerPos viewportPos"` (e.g. `"top center"`), or `"+=N"`, or a function. |
| `scrub` | `true` ties playhead to scroll exactly; a number adds that many seconds of catch-up smoothing. |
| `pin` | `true` (or an element) to fix it in place during the range. |
| `markers` | Visual debugging markers — **remove for production**. |
| `toggleActions` | Four states `onEnter onLeave onEnterBack onLeaveBack`, each one of `play/pause/resume/reverse/restart/reset/complete/none`. Used when **not** scrubbing. |
| `snap` | Snap scroll to progress points (number, array, or `"labels"`). |
| `pinSpacing` | Whether to add padding so content below doesn't jump (default `true`). |
| `toggleClass` | Add/remove a CSS class while active. |
| `onEnter`/`onLeave`/`onUpdate`/`onToggle` | Callbacks with the ScrollTrigger instance. |

### 8.3 Scrub vs toggleActions — the key choice [I/A]

- **`scrub`**: the animation's progress *is* the scroll progress. Scroll down → it advances; scroll up → it reverses. Use for parallax, horizontal scroll sections, progress-tied reveals. Pair with `ease: "none"` so motion tracks the scrollbar linearly.
- **`toggleActions`**: the animation **plays on its own timing** when a boundary is crossed. Use for "fade this in when it enters the viewport" one-shots. The string is four actions for the four events:

```js
// "play none none reverse": play on enter, do nothing on leave/enterBack, reverse on leaveBack.
gsap.from(".reveal", {
  y: 60, opacity: 0, duration: 0.8,
  scrollTrigger: { trigger: ".reveal", start: "top 80%", toggleActions: "play none none reverse" },
});
```

### 8.4 Pinning [I/A]

`pin: true` fixes the trigger while the user scrolls through the `start`–`end` range — the foundation of "sticky" storytelling and horizontal-scroll galleries. ScrollTrigger inserts spacer markup so the page length stays correct. Pin the *container*, and animate its children with a scrubbed timeline:

```js
const tl = gsap.timeline({
  scrollTrigger: {
    trigger: ".horizontal",
    start: "top top",
    end: () => "+=" + document.querySelector(".track").scrollWidth, // dynamic distance
    scrub: 1,
    pin: true,
  },
});
tl.to(".track", { xPercent: -100, ease: "none" }); // scroll vertically → move horizontally
```

### 8.5 ScrollTrigger inside `useGSAP` — the correct React pattern [I/A]

ScrollTriggers created inside a `useGSAP` context are **reverted automatically** — this is the single most important reason to use the hook. Created outside it, they leak across route changes and StrictMode mounts.

```jsx
"use client";
import { useRef } from "react";
import { gsap } from "gsap";
import { useGSAP } from "@gsap/react";
import { ScrollTrigger } from "gsap/ScrollTrigger";

gsap.registerPlugin(useGSAP, ScrollTrigger); // register once (useGSAP is registerable too)

function Reveal() {
  const root = useRef(null);
  useGSAP(() => {
    gsap.from(".item", {
      y: 50, opacity: 0, stagger: 0.1,
      scrollTrigger: { trigger: root.current, start: "top 75%" },
    });
    // No manual ScrollTrigger.kill() needed — the context reverts on unmount.
  }, { scope: root });

  return <div ref={root}>{/* .item elements */}</div>;
}
```

**⚡ Version note:** `ScrollTrigger.refresh()` recalculates start/end positions. Call it after content height changes (images loading, fonts swapping, async data) so triggers don't compute against the wrong layout. When using SplitText or web fonts, refresh on `document.fonts.ready` (Section 15).

### 8.6 Responsive ScrollTrigger with `matchMedia` [I/A]

Scroll distances and pins that work on desktop often break on mobile. **`gsap.matchMedia()`** runs different setups per media query and **auto-reverts** the ones that no longer match when the viewport changes — perfect with `useGSAP`:

```jsx
useGSAP(() => {
  const mm = gsap.matchMedia();

  mm.add("(min-width: 768px)", () => {
    // Desktop-only: a pinned, scrubbed horizontal section.
    gsap.to(".track", {
      xPercent: -100, ease: "none",
      scrollTrigger: { trigger: ".horizontal", pin: true, scrub: 1, end: "+=2000" },
    });
  });

  mm.add("(max-width: 767px)", () => {
    // Mobile: simple fade-ins, no pinning.
    gsap.from(".card", { opacity: 0, y: 30, stagger: 0.1,
      scrollTrigger: { trigger: ".cards", start: "top 80%" } });
  });
  // matchMedia reverts the non-matching branch automatically on resize.
}, { scope: root });
```

### 8.7 `ScrollTrigger.batch` — efficient many-element reveals [A]

When dozens of elements should reveal on enter, creating a ScrollTrigger per element is wasteful. **`ScrollTrigger.batch`** groups them and animates whichever batch enters together — far fewer triggers, smoother scrolling.

```js
ScrollTrigger.batch(".card", {
  start: "top 85%",
  onEnter: (batch) => gsap.from(batch, { opacity: 0, y: 40, stagger: 0.1, overwrite: true }),
});
```

### 8.8 Snapping to sections [A]

`snap` makes the scroll position **settle onto meaningful points** after the user stops scrolling — the basis of "full-page section" sites. It works *with* a scrubbed animation: you snap to the progress values you care about. The simplest form snaps to evenly-spaced steps; the array form snaps to specific progress positions; the object form gives you control over duration and inertia.

```js
// Snap a scrubbed timeline to 5 evenly-spaced stops (0, .25, .5, .75, 1):
ScrollTrigger.create({
  trigger: ".panels",
  start: "top top",
  end: "+=3000",
  pin: true,
  scrub: 1,
  snap: {
    snapTo: 1 / 4,          // a number = snap to increments of this (5 stops here)
    duration: { min: 0.2, max: 0.6 }, // clamp the settle time
    delay: 0.1,             // wait after scroll stops before snapping
    ease: "power1.inOut",
    inertia: false,         // ignore flick velocity; always snap to nearest
  },
});

// Snap to specific labels of a timeline (great for narrative steps):
snap: { snapTo: "labelsDirectional" }   // snaps to timeline labels, in the scroll direction
```

> **⚡ Gotcha:** snapping fights the user if `duration` is too long or you snap too aggressively on a long page — it can feel like the page is "grabbing" the scroll. Use a short duration, a small `delay`, and only snap where discrete stops are genuinely meaningful (panels, steps), never on free-flowing content.

### 8.9 Horizontal scroll driven by vertical scrolling [A]

One of the most-requested effects: a **horizontal section** that scrolls sideways while the user scrolls down. The recipe is "pin a tall-enough region and tween the track's `x` against scroll progress." The key insight is computing the distance to travel from the track's actual width, and setting `end` to that distance so the pin lasts exactly as long as the horizontal travel.

```jsx
"use client";
import { useRef } from "react";
import gsap from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";
import { useGSAP } from "@gsap/react";
gsap.registerPlugin(ScrollTrigger, useGSAP);

export default function HorizontalScroll() {
  const root = useRef(null);
  const track = useRef(null);

  useGSAP(() => {
    const panels = gsap.utils.toArray(".panel", track.current);
    // Distance to travel = total track width minus one viewport.
    const distance = () => track.current.scrollWidth - window.innerWidth;

    gsap.to(track.current, {
      x: () => -distance(),          // function = re-evaluated on ScrollTrigger.refresh()
      ease: "none",
      scrollTrigger: {
        trigger: root.current,
        pin: true,
        scrub: 1,
        // end is also a function of distance, so it's correct after resize:
        end: () => "+=" + distance(),
        invalidateOnRefresh: true,   // recompute x/end when the layout changes (CRITICAL here)
        anticipatePin: 1,
      },
    });
  }, { scope: root });

  return (
    <section ref={root} className="overflow-hidden">
      <div ref={track} className="flex w-max">
        {[1, 2, 3, 4].map((n) => (
          <div key={n} className="panel w-screen h-screen grid place-items-center">Panel {n}</div>
        ))}
      </div>
    </section>
  );
}
```

> **Why the functions matter:** `x` and `end` are written as **functions**, not fixed numbers. With `invalidateOnRefresh: true`, ScrollTrigger re-runs them on every `refresh()` (resize, font load, orientation change), so the horizontal distance stays correct. Hard-coding a pixel value is the #1 reason horizontal-scroll sections break on mobile or after the layout shifts.

### 8.10 Parallax, progress & the callback API [A]

ScrollTrigger exposes **lifecycle callbacks** and a live **progress** value (0→1) that you can read for effects beyond a single scrubbed tween — progress bars, parallax layers at different speeds, and triggering side effects (analytics, lazy media, nav highlighting).

```js
// Multi-layer parallax: each layer moves a different amount against the same scroll.
gsap.utils.toArray(".layer").forEach((layer) => {
  const depth = layer.dataset.depth || 1;          // e.g. data-depth="0.3"
  gsap.to(layer, {
    yPercent: -20 * depth,                          // farther layers move less
    ease: "none",
    scrollTrigger: { trigger: layer.closest(".scene"), scrub: true, start: "top bottom", end: "bottom top" },
  });
});

// A reading-progress bar + callbacks for side effects:
ScrollTrigger.create({
  trigger: "article",
  start: "top top",
  end: "bottom bottom",
  onUpdate: (self) => {
    // self.progress is 0→1; self.direction is 1 (down) or -1 (up); self.velocity is px/s.
    gsap.set(".progress-bar", { scaleX: self.progress, transformOrigin: "left center" });
  },
  onEnter:     () => console.log("entered"),
  onLeave:     () => markSectionRead(),    // fire once when scrolled past
  onEnterBack: () => highlightNav("article"),
  onLeaveBack: () => unhighlightNav("article"),
});
```

> **Best practice:** prefer **`scrub` + a tween** over reading `progress` in `onUpdate` and manually setting styles — GSAP's scrub is smoother and frame-throttled. Reach for `onUpdate`/`progress` only when you need a value GSAP can't tween directly (driving a canvas, a WebGL uniform, a React state update — and if you set React state here, throttle it, or you'll re-render every frame).

---

## 9. Next.js Specifics — `"use client"`, SSR & Hydration

GSAP runs in the browser. Next.js App Router renders components on the **server** by default, where there is no `window`, no `document`, and no layout to measure. This mismatch is the source of nearly every "GSAP doesn't work in Next.js" problem. The rules below make it reliable.

### 9.1 GSAP is client-only — the `"use client"` rule [I/A]

Any component that imports `gsap`, `@gsap/react`, or a plugin, or that calls `useGSAP`/`useEffect`/`useRef`, **must** be a Client Component. Put `"use client";` as the very first line of the file. Without it, Next tries to render on the server, hits browser APIs, and errors (or worse, silently no-ops).

```jsx
"use client"; // MUST be first — makes this a Client Component

import { useRef } from "react";
import { gsap } from "gsap";
import { useGSAP } from "@gsap/react";

export default function AnimatedHeading() {
  const ref = useRef(null);
  useGSAP(() => { gsap.from(".h", { y: 30, opacity: 0 }); }, { scope: ref });
  return <h1 ref={ref} className="h">Hello</h1>;
}
```

**Architecture tip:** keep Server Components as the default (they're faster and SEO-friendly) and isolate animation into small `"use client"` leaf components. A server-rendered page can import and render a client `<AnimatedHeading />` — only that leaf ships GSAP to the browser.

### 9.2 Where to register plugins [I/A]

Call `gsap.registerPlugin(...)` **inside a client module**, never at the top of a file a Server Component imports. The safest spots: at the top level of the `"use client"` component file that uses the plugin, or in a small shared `"use client"` init module imported by your client components. Registering twice is harmless, so registering in each client file is acceptable; a single shared module is cleaner.

```jsx
"use client";
import { gsap } from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";
import { useGSAP } from "@gsap/react";

// Top-level of a client file is fine — runs only on the client.
gsap.registerPlugin(useGSAP, ScrollTrigger);
```

### 9.3 SSR/hydration mismatches and FOUC [I/A]

The classic Next.js animation bug: the server renders the element in its **final** state (because `gsap.from` hasn't run — it only runs on the client after hydration). For a split second the user sees the finished layout, *then* GSAP yanks it to the start and animates in — a flash (FOUC). Two robust fixes:

1. **Set the hidden/start state in CSS** so the server-rendered HTML already looks "pre-animation," then animate **to** the visible state. This guarantees no flash because the markup is never visible in its final state before JS runs.

```css
/* The element renders hidden on the server; GSAP animates it in after hydration. */
.fade-up { opacity: 0; transform: translateY(30px); }
```
```jsx
"use client";
function Section() {
  const ref = useRef(null);
  useGSAP(() => {
    gsap.to(".fade-up", { opacity: 1, y: 0, duration: 0.8, ease: "power2.out" });
  }, { scope: ref });
  return <div ref={ref}><p className="fade-up">Content</p></div>;
}
```

2. **Use `gsap.set` before the animation** in the same `useGSAP` callback to force the start state synchronously before paint. Combined with option 1 for belt-and-suspenders.

**Why not just `gsap.from`?** Because between server render and the first client frame, the element is briefly at its natural position. CSS-first hiding removes that window entirely.

### 9.4 Dynamic import for heavy/measure-dependent components [I/A]

Some components must not render on the server at all — e.g. ones that measure layout immediately, or third-party canvas/WebGL pieces. Load them client-side only with `next/dynamic` and `ssr: false`:

```jsx
// In a Server (or client) component:
import dynamic from "next/dynamic";

// This component never renders on the server — no hydration mismatch possible.
const ScrollScene = dynamic(() => import("./ScrollScene"), { ssr: false });

export default function Page() {
  return <main><ScrollScene /></main>;
}
```

Use this sparingly — it disables SSR for that subtree (worse for SEO and first paint). Most GSAP components work fine as plain `"use client"` leaves; reserve `ssr: false` for genuinely measure-dependent or non-isomorphic code.

### 9.5 Cleanup on route changes [I/A]

App Router navigations unmount the old page's components. If your ScrollTriggers/tweens weren't created inside a `useGSAP` context, they survive the navigation, keep firing against removed DOM, and pollute the next page (e.g. duplicate ScrollTriggers, broken scroll positions). **`useGSAP` reverts on unmount**, so it cleans up automatically on navigation. This is the strongest practical argument for routing *all* React GSAP code through `useGSAP`.

**⚡ Version note:** After significant navigations or layout shifts (e.g. a modal that changed page height), call `ScrollTrigger.refresh()` so trigger positions recompute. With `useGSAP` the old triggers are already gone; refresh ensures the new ones measure correctly.

---

## 10. The Plugin Suite (All Free)

Everything below is bundled in `npm install gsap` with **no membership** since the 2024 acquisition. Import from the submodule path, register, and go. This section gives each plugin a focused purpose and example; Flip gets its own Section 11 because it pairs so tightly with React.

### 10.1 SplitText — animating text by char/word/line [A]

**SplitText** breaks a text element into wrapped `<div>`/`<span>` pieces for **characters, words, and lines**, so you can animate them individually (the staggered "letters fly in" hero effect). It handles line detection responsively. Formerly Club-only; now free.

```js
import { SplitText } from "gsap/SplitText";
gsap.registerPlugin(SplitText);

const split = new SplitText(".hero h1", { type: "chars, words" });
gsap.from(split.chars, { y: 40, opacity: 0, stagger: 0.03, ease: "back.out(1.7)", duration: 0.6 });

// CLEANUP is mandatory: split.revert() restores the original markup (and accessibility).
// In React, do this inside useGSAP so it's automatic.
```

**Critical gotchas:** (1) **Always `revert()`** — SplitText mutates the DOM; leaving it split harms screen readers and re-splits stack up. (2) Split **after fonts load** (`document.fonts.ready`) or line breaks compute against the fallback font and shift. (3) Splitting causes a brief FOUC; hide the text until split+animation are ready.

### 10.2 ScrollSmoother — smooth/inertia scrolling [A]

**ScrollSmoother** (built on ScrollTrigger) adds smooth, momentum-based scrolling and easy parallax via `data-speed`/`data-lag` attributes. Formerly premium; now free. It requires a specific wrapper/content DOM structure.

```js
import { ScrollSmoother } from "gsap/ScrollSmoother";
gsap.registerPlugin(ScrollTrigger, ScrollSmoother);

ScrollSmoother.create({
  wrapper: "#smooth-wrapper",
  content: "#smooth-content",
  smooth: 1.2,        // smoothing time in seconds
  effects: true,      // enable data-speed / data-lag parallax on children
});
// In React: create inside useGSAP and let the context revert it on unmount.
```

**Caution:** smooth scrolling can hurt accessibility and feel laggy on low-end devices; gate it behind `prefers-reduced-motion` and consider desktop-only via `matchMedia`.

### 10.3 DrawSVG — drawing SVG strokes on [A]

**DrawSVG** animates an SVG stroke as if it's being drawn, by tweening the visible portion of the stroke (`stroke-dasharray`/`offset` under the hood). Great for signatures, line illustrations, animated icons. Formerly premium; now free.

```js
import { DrawSVGPlugin } from "gsap/DrawSVGPlugin";
gsap.registerPlugin(DrawSVGPlugin);

gsap.from("#signature path", { drawSVG: "0%", duration: 2, ease: "power1.inOut", stagger: 0.2 });
// drawSVG "0%" → "100%": the stroke draws on. Accepts "20% 80%" to reveal a segment.
```

### 10.4 MorphSVG — morphing one SVG shape into another [A]

**MorphSVG** smoothly morphs one SVG path's `d` data into another's, even with differing point counts (it intelligently maps points). The canonical "icon A turns into icon B" effect. Formerly premium; now free.

```js
import { MorphSVGPlugin } from "gsap/MorphSVGPlugin";
gsap.registerPlugin(MorphSVGPlugin);

gsap.to("#start", { morphSVG: "#end", duration: 1, ease: "power2.inOut" });
// Morphs #start's path shape into #end's path shape.
```

### 10.5 MotionPath — moving along a path [A]

**MotionPathPlugin** moves an element **along an SVG path or a set of points**, optionally auto-rotating it to face the direction of travel. For orbiting elements, curved entrances, following a drawn route. Core-adjacent and free.

```js
import { MotionPathPlugin } from "gsap/MotionPathPlugin";
gsap.registerPlugin(MotionPathPlugin);

gsap.to(".rocket", {
  duration: 4,
  ease: "none",
  motionPath: {
    path: "#route",      // an SVG path element
    align: "#route",     // align coordinate space to the path
    autoRotate: true,    // rotate the rocket to face its travel direction
  },
});
```

`MotionPathHelper` (also now free) lets you edit paths visually in-browser during development.

### 10.6 Observer — unified input handling [A]

**Observer** normalizes wheel, touch, and pointer events into one consistent API with velocity and direction — the clean way to build wheel/swipe-driven sections without juggling raw event listeners.

```js
import { Observer } from "gsap/Observer";
gsap.registerPlugin(Observer);

Observer.create({
  target: window,
  type: "wheel,touch,pointer",
  onUp: () => goToSection(prev),    // scroll/swipe up
  onDown: () => goToSection(next),  // scroll/swipe down
  tolerance: 10,
  preventDefault: true,
});
```

### 10.7 Draggable + InertiaPlugin — drag & throw [A]

**Draggable** makes elements draggable (and spinnable/rotatable); paired with **InertiaPlugin** (formerly premium, now free) it adds realistic momentum so a flicked element glides and settles. For sliders, carousels, knobs, kanban cards.

```js
import { Draggable } from "gsap/Draggable";
import { InertiaPlugin } from "gsap/InertiaPlugin";
gsap.registerPlugin(Draggable, InertiaPlugin);

Draggable.create(".card", {
  type: "x",
  inertia: true,          // throw with momentum (needs InertiaPlugin)
  bounds: ".container",   // constrain within an element
  snap: (value) => Math.round(value / 100) * 100, // settle on a grid
});
```

### 10.8 Plugin reference table [A]

| Plugin | Import path | Does | Status 2026 |
|---|---|---|---|
| ScrollTrigger | `gsap/ScrollTrigger` | Scroll-driven anim, pin, scrub, snap | Free (always was) |
| SplitText | `gsap/SplitText` | Split text into chars/words/lines | **Now free** |
| ScrollSmoother | `gsap/ScrollSmoother` | Smooth/inertia scroll + parallax | **Now free** |
| DrawSVGPlugin | `gsap/DrawSVGPlugin` | Draw-on SVG strokes | **Now free** |
| MorphSVGPlugin | `gsap/MorphSVGPlugin` | Morph SVG shapes | **Now free** |
| MotionPathPlugin | `gsap/MotionPathPlugin` | Move along a path | Free |
| Flip | `gsap/Flip` | State-based layout animation | Free |
| Observer | `gsap/Observer` | Unified wheel/touch/pointer | Free |
| Draggable | `gsap/Draggable` | Drag/spin elements | Free |
| InertiaPlugin | `gsap/InertiaPlugin` | Momentum/throw physics | **Now free** |
| CustomEase / CustomBounce / CustomWiggle | `gsap/CustomEase` … | Author custom eases | **Now free** |
| GSDevTools | `gsap/GSDevTools` | Visual timeline scrubber for dev | **Now free** |

---

## 11. The Flip Plugin with React

**Flip** deserves its own section because it solves a problem React developers hit constantly and pairs beautifully with React state. FLIP stands for **First, Last, Invert, Play** — a technique for animating *layout changes you can't (or don't want to) animate directly*: an element moving to a new grid position, a list reordering, a thumbnail expanding into a full view (shared-element transition), an item moving between two containers. You change the DOM/state however you like, and Flip animates the *transition* between the before and after layouts.

### 11.1 Why Flip exists — the React layout-animation problem [A]

In React you change a layout by changing state: filtering a grid, reordering a list, toggling an element between two parents. React re-renders and the browser *snaps* to the new layout instantly — no animation, because you never tweened anything; the DOM just changed. Animating these by hand is brutal: you'd have to measure old and new positions and compute transforms. **Flip automates exactly that.** You record the layout *before* the change (`Flip.getState`), let React apply the change, then call `Flip.from(state, ...)` — Flip measures the new layout, inverts the difference as a transform, and animates it away, producing a smooth move/resize even across DOM reparenting.

### 11.2 The four-step pattern [A]

1. **Capture** the current state of the targets: `const state = Flip.getState(targets)`.
2. **Mutate** — change React state / classes / DOM so the layout changes.
3. **Animate** from the captured state: `Flip.from(state, { duration, ease, ... })`.
4. Flip computes First→Last, **inverts** the delta, and **plays** it to zero.

The subtlety in React: step 2 (the state change) is asynchronous — the DOM updates *after* the re-render commits. So you capture *before* `setState`, and run `Flip.from` *after* the DOM has updated. `useGSAP` with the changing value as a dependency is the clean way to run step 3 post-commit.

### 11.3 Flip with `useGSAP` — reorder/filter a grid [A]

```jsx
"use client";
import { useRef, useState } from "react";
import { gsap } from "gsap";
import { useGSAP } from "@gsap/react";
import { Flip } from "gsap/Flip";

gsap.registerPlugin(useGSAP, Flip);

function Gallery({ items }) {
  const root = useRef(null);
  const [filter, setFilter] = useState("all");
  const stateRef = useRef(null); // holds the captured Flip state across the render

  // Capture BEFORE the state change triggers a re-render.
  const changeFilter = (next) => {
    stateRef.current = Flip.getState(root.current.querySelectorAll(".tile"));
    setFilter(next); // React re-renders → layout changes
  };

  // After the DOM updates (filter changed → deps fire), animate from the captured state.
  useGSAP(() => {
    if (!stateRef.current) return; // skip the initial mount
    Flip.from(stateRef.current, {
      duration: 0.6,
      ease: "power2.inOut",
      absolute: true,          // take elements out of flow during the move (smoother)
      stagger: 0.03,
      onEnter: (els) => gsap.fromTo(els, { opacity: 0, scale: 0 }, { opacity: 1, scale: 1 }),
      onLeave: (els) => gsap.to(els, { opacity: 0, scale: 0 }),
    });
  }, { scope: root, dependencies: [filter] });

  const visible = items.filter((i) => filter === "all" || i.cat === filter);
  return (
    <div ref={root}>
      <button onClick={() => changeFilter("all")}>All</button>
      <button onClick={() => changeFilter("photos")}>Photos</button>
      <div className="grid">
        {visible.map((i) => <div key={i.id} className="tile">{i.label}</div>)}
      </div>
    </div>
  );
}
```

### 11.4 Shared-element transition (thumbnail → detail) [A]

Flip can animate an element from one position/size to a completely different one elsewhere in the tree — the "thumbnail expands into a hero" effect — by capturing state, toggling which element is shown/sized, and flipping. Give the source and destination the same logical identity (e.g. via a `data-flip-id`) so Flip matches them across the DOM change.

```jsx
// Pattern: capture before expanding, expand via state, Flip.from after.
const onOpen = () => {
  const state = Flip.getState(".thumb, .detail");
  setOpen(true); // swaps which is visible / changes layout
  // in the deps-driven useGSAP:
  // Flip.from(state, { duration: 0.5, ease: "power3.inOut", absolute: true });
};
```

**Key Flip options:** `absolute` (position elements absolutely during the flip for smoother reparenting), `nested` (handle nested flipping elements), `onEnter`/`onLeave` (animate elements that newly appear/disappear), `targets`, `scale` (animate via scale instead of width/height for performance).

**Why Flip + React is a power combo:** you keep writing idiomatic React (change state, re-render) and Flip retrofits smooth motion onto layout changes you'd otherwise be unable to animate. It's the GSAP analogue to Motion's `layout` prop, but works across arbitrary DOM and reparenting.

### 11.5 The Flip options that matter, in depth [A]

`Flip.from(state, vars)` accepts options that solve the real-world problems you hit once you go past the basic case:

| Option | What it does | When you need it |
|---|---|---|
| `absolute: true` | Temporarily positions the flipping targets `absolute` during the animation | Reparenting, or when surrounding layout would otherwise reflow mid-flip (the most common fix for "the flip looks janky") |
| `absoluteOnLeave: true` | Absolutely positions only *leaving* elements | Smooth removal without collapsing siblings early |
| `nested: true` | Correctly handles a flipping element **inside** another flipping element | Cards that move *and* contain moving children |
| `scale: true` | Animates size via `scaleX/scaleY` (transform) instead of `width/height` (layout) | Performance — transforms are GPU-friendly; use unless scaling distorts text/borders |
| `onEnter` / `onLeave` | Callbacks to animate elements that newly **appear** / **disappear** between states | Filtering/adding/removing items (see 11.6) |
| `targets` | Limit the flip to a subset of the captured state | You captured a broad state but only want some elements to animate |
| `props` | Also animate non-layout CSS props (e.g. `backgroundColor`, `borderRadius`) through the flip | A card that changes color *and* position in one motion |
| `simple: true` | Skip rotation/skew matrix math for a small perf win | Flat translate/scale only, no rotated ancestors |

`Flip.fit(target, source, vars)` is a different, underused tool: it **resizes/repositions one element to match another** (without the capture/apply dance) — perfect for "make this box exactly cover that box," picture-in-picture, or snapping a draggable into a slot.

```js
// Make the #player element morph to exactly fit whichever container is active:
Flip.fit("#player", activeContainer, { duration: 0.6, ease: "power2.inOut", scale: true, absolute: true });
```

### 11.6 Entering & leaving elements during a Flip (filter/sort a list) [A]

The killer React use case: a filtered/sorted list where items **stay** (move to new positions), **leave** (animate out), and **enter** (animate in) — all in one coordinated motion. Capture the state of everything *before* the data change, let React re-render, then `Flip.from` with `onEnter`/`onLeave` handlers and a stagger.

```jsx
"use client";
import { useRef, useState } from "react";
import gsap from "gsap";
import { Flip } from "gsap/Flip";
import { useGSAP } from "@gsap/react";
gsap.registerPlugin(Flip, useGSAP);

export default function FilterableGrid({ items }) {
  const root = useRef(null);
  const [filter, setFilter] = useState("all");
  const stateRef = useRef(null);

  // Capture BEFORE the filter state change re-renders the list:
  const setFilterAnimated = (next) => {
    stateRef.current = Flip.getState(".item", { props: "opacity" });
    setFilter(next);
  };

  // After re-render, flip from the captured state. `filter` in deps drives it.
  useGSAP(() => {
    if (!stateRef.current) return;
    Flip.from(stateRef.current, {
      duration: 0.5,
      ease: "power2.inOut",
      scale: true,
      absolute: true,                 // smooth reflow as items reposition
      stagger: 0.04,
      // Items that are NEW this render (weren't in the captured state):
      onEnter: (els) =>
        gsap.fromTo(els, { opacity: 0, scale: 0.8 }, { opacity: 1, scale: 1, duration: 0.4, stagger: 0.04 }),
      // Items leaving (in the state but gone now) — Flip keeps them around to animate out:
      onLeave: (els) =>
        gsap.to(els, { opacity: 0, scale: 0.8, duration: 0.3 }),
    });
  }, { scope: root, dependencies: [filter] });

  const visible = filter === "all" ? items : items.filter((i) => i.tag === filter);
  return (
    <div ref={root}>
      <nav>{["all", "a", "b"].map((f) => <button key={f} onClick={() => setFilterAnimated(f)}>{f}</button>)}</nav>
      <div className="grid grid-cols-3 gap-4">
        {visible.map((i) => <div key={i.id} data-flip-id={i.id} className="item">{i.label}</div>)}
      </div>
    </div>
  );
}
```

> **The mental model:** `Flip.getState` is a *snapshot* of positions/sizes; React then freely changes the DOM; `Flip.from` *diffs* the live DOM against the snapshot and animates the difference — moving survivors, and handing you new/removed elements via `onEnter`/`onLeave`. The `data-flip-id` is what lets Flip match an element across the re-render even if React recreated the node. This is the single most powerful GSAP+React pattern; master it and most "how do I animate this list change?" questions answer themselves.

---

## 12. Responsive & Accessible Animation

Production animation must adapt to viewport size and respect users who are sensitive to motion. Two tools carry most of this: **`gsap.matchMedia()`** for responsive setups and the **`prefers-reduced-motion`** media query for accessibility. Skipping these ships animations that break on phones or trigger vestibular discomfort — both serious quality and inclusivity failures.

### 12.1 `gsap.matchMedia()` — responsive, auto-reverting setups [I/A]

`matchMedia` runs a setup function only while its media query matches and **reverts it automatically** when the query stops matching (on resize/orientation change). This is the correct way to have different animations per breakpoint without manually tearing down on resize. It composes with `useGSAP` (the context reverts everything on unmount; matchMedia reverts per-query on resize).

```jsx
useGSAP(() => {
  const mm = gsap.matchMedia();

  mm.add(
    { isDesktop: "(min-width: 768px)", isMobile: "(max-width: 767px)" },
    (ctx) => {
      const { isDesktop } = ctx.conditions;
      gsap.from(".panel", {
        x: isDesktop ? -200 : 0,           // horizontal on desktop, none on mobile
        y: isDesktop ? 0 : 40,             // vertical entrance on mobile
        opacity: 0, duration: 0.8,
      });
    }
  );
}, { scope: root });
```

### 12.2 `prefers-reduced-motion` — the accessibility baseline [I/A]

The OS-level "reduce motion" setting (Windows: Settings → Accessibility → Visual effects → Animation effects) surfaces as the `(prefers-reduced-motion: reduce)` media query. When a user has it on, **disable or drastically simplify** large/parallax/scroll-jacking motion. The cleanest implementation uses `matchMedia` to branch:

```jsx
useGSAP(() => {
  const mm = gsap.matchMedia();

  // Full motion for users who haven't requested reduced motion.
  mm.add("(prefers-reduced-motion: no-preference)", () => {
    gsap.from(".hero", { y: 60, opacity: 0, duration: 1, ease: "power3.out" });
    // ...parallax, pins, splits...
  });

  // Reduced: skip movement, allow a gentle fade (or nothing).
  mm.add("(prefers-reduced-motion: reduce)", () => {
    gsap.set(".hero", { opacity: 1, y: 0 });       // just show it
    // optionally: gsap.from(".hero", { opacity: 0, duration: 0.3 });
  });
}, { scope: root });
```

**Best practices for accessibility & SEO:**
- **Never animate content into existence in a way that hides it from crawlers or assistive tech permanently.** Set the *final* state as the source of truth; animations should reveal, not gate, content.
- With **SplitText**, always `revert()` so screen readers read the original, unsplit text. Splitting leaves text in fragmented spans — bad for AT if left in place.
- Avoid scroll-jacking (hijacking native scroll) unless essential; it frustrates keyboard and screen-reader users. If you use ScrollSmoother, ensure keyboard navigation still works and respect reduced-motion.
- Don't tie *essential information* to animation completion — content must be usable even if JS/animation fails.

---

## 13. Performance & Production

GSAP is already one of the fastest animation engines available, but you can still build janky experiences by animating the wrong properties or thrashing layout. The frame budget at 60fps is ~`16.7ms`; miss it and animations stutter. These practices keep you inside it.

### 13.1 Animate transforms and opacity (the GPU rule) [A]

The browser renders via **Style → Layout → Paint → Composite** (see the **[CSS](CSS_GUIDE.md)** guide). Animating `width`/`height`/`top`/`left`/`margin` forces **layout** (reflow) every frame — expensive, often page-wide. Animating `transform` (`x`/`y`/`scale`/`rotation`) and `opacity` only touches the **composite** step, which runs on the GPU compositor thread. **Prefer `x`/`y` over `left`/`top`, `scale` over `width`/`height`.** This single rule is the largest performance lever in GSAP work.

```js
// ✅ GPU-friendly — composite only:
gsap.to(".box", { x: 300, scale: 1.2, opacity: 0.5 });
// ❌ Layout-thrashing — avoid in animations:
gsap.to(".box", { left: 300, width: 400 });
```

### 13.2 `will-change`, `force3D`, and layer promotion [A]

GSAP automatically applies `transform: translateZ(0)` (via `force3D: "auto"`) when helpful, promoting animated elements to their own GPU layer. You can hint the browser with CSS `will-change: transform` on elements you're about to animate — but **don't leave it on permanently or apply it to many elements**, as each promoted layer costs memory. Add it just before animating and remove it after for heavy cases. `force3D: true` forces 3D layering; `force3D: false` disables it (occasionally needed to avoid blurry text on certain GPUs).

### 13.3 Reducing layout thrash (batch reads/writes) [A]

"Layout thrashing" is interleaving DOM **reads** (`offsetWidth`, `getBoundingClientRect`) and **writes** (style changes) so the browser must recompute layout repeatedly within one frame. GSAP batches its own writes, but *your* code (e.g. measuring inside `onUpdate`) can thrash. Read all measurements first, then write. For scroll, let ScrollTrigger do the measuring (it caches and refreshes deliberately) rather than calling `getBoundingClientRect` every frame.

### 13.4 `gsap.ticker` — the central loop [A]

Every GSAP animation is driven by one `requestAnimationFrame` loop exposed as `gsap.ticker`. You can hook into it for custom per-frame work that stays synced with all animations, set the target FPS, or add/remove listeners — better than running your own `rAF` loop alongside GSAP's.

```js
const onTick = (time, deltaTime) => { /* synced per-frame work */ };
gsap.ticker.add(onTick);
// ...later, ALWAYS remove it (in React, inside the useGSAP cleanup path):
gsap.ticker.remove(onTick);
gsap.ticker.fps(30); // cap the loop at 30fps if appropriate
```

### 13.5 Lazy-loading animations & code-splitting [A]

GSAP plus several plugins adds real bytes. Strategies: import **only the plugins you use** (tree-shaking drops the rest); dynamically import heavy scroll/scene components with `next/dynamic` so their GSAP code loads only when needed; defer below-the-fold ScrollTriggers (they don't run until scrolled into range anyway, but their *setup* and the element refs still cost — `ScrollTrigger.batch` and lazy mounting help).

### 13.6 Measuring [A]

Use the browser's **Performance** panel to record a scroll/interaction and look for long frames, forced reflows ("Recalculate Style"/"Layout" purple bars), and dropped frames. Enable ScrollTrigger `markers: true` while tuning, then remove them. GSAP's free **GSDevTools** gives a visual scrubber to step through timelines frame by frame during development.

---

## 14. Architecture & Maintainability

"Production-grade" means animations that a team can read, reuse, and not break six months later. GSAP code sprawls fast if every component hand-rolls timelines inline. These patterns keep an animated React/Next codebase maintainable.

### 14.1 Encapsulate animations in custom hooks [A]

Wrap a reusable animation in a custom hook that takes a ref (or returns one) and runs `useGSAP` internally. The component stays declarative; the animation logic is testable and reused.

```jsx
"use client";
import { useRef } from "react";
import { gsap } from "gsap";
import { useGSAP } from "@gsap/react";

// Reusable "fade up on mount" hook. Returns the ref to attach.
export function useFadeUp(options = {}) {
  const ref = useRef(null);
  useGSAP(() => {
    gsap.from(ref.current, {
      y: options.distance ?? 30,
      opacity: 0,
      duration: options.duration ?? 0.8,
      ease: options.ease ?? "power2.out",
      delay: options.delay ?? 0,
    });
  }, { scope: ref });
  return ref;
}

// Usage stays clean:
function Card() {
  const ref = useFadeUp({ delay: 0.1 });
  return <div ref={ref} className="card">…</div>;
}
```

### 14.2 Reusable animation components [A]

For wrapping arbitrary children, build a component that owns the scope ref and runs a scroll-reveal. This is the React-idiomatic way to make `<Reveal>…</Reveal>` available across the app.

```jsx
"use client";
export function Reveal({ children, y = 40, start = "top 80%" }) {
  const ref = useRef(null);
  useGSAP(() => {
    gsap.from(ref.current, {
      y, opacity: 0, duration: 0.8, ease: "power2.out",
      scrollTrigger: { trigger: ref.current, start },
    });
  }, { scope: ref });
  return <div ref={ref}>{children}</div>;
}
```

### 14.3 Centralize tokens, eases, and durations [A]

Hard-coded `0.8`s and `"power2.out"`s scattered everywhere drift over time. Define a small animation config and reference it, so timing/feel is consistent and tunable in one place.

```js
// animation/config.js
export const DUR = { fast: 0.3, base: 0.6, slow: 1 };
export const EASE = { out: "power2.out", inOut: "power3.inOut", pop: "back.out(1.7)" };
// Register custom eases here once, at import time, so names resolve app-wide.
```

### 14.4 TypeScript with GSAP [A]

GSAP ships its own type definitions — no `@types` package needed. Tween/timeline instances, plugin configs, and `useGSAP` are typed. Type your refs and the values you pass.

```tsx
import { gsap } from "gsap";
import type { GSAPTimeline } from "gsap"; // namespaced types are available

function useMenu() {
  const root = useRef<HTMLDivElement>(null);
  const tl = useRef<GSAPTimeline | null>(null);
  useGSAP(() => {
    tl.current = gsap.timeline({ paused: true }).to(".panel", { height: "auto" });
  }, { scope: root });
  return { root, tl };
}
```

Most APIs are typed via the global `gsap` namespace (e.g. `gsap.core.Timeline`, `gsap.core.Tween`). Plugin vars (like `scrollTrigger`) are typed once the plugin is imported.

### 14.5 Testing considerations [A]

Animation is hard to assert on directly. Practical approaches: (1) test the **logic** around animations (state changes, handlers, that the right elements render) rather than pixel positions; (2) in **jsdom** (Jest/Vitest), GSAP runs but there's no real layout — ScrollTrigger and measurement-based plugins won't behave meaningfully, so mock or skip them in unit tests; (3) reserve real motion verification for **end-to-end** tools (Playwright) that can assert an element reached a final state or a class toggled; (4) since `useGSAP` cleans up, you can assert components unmount without leaking. Keep animation declarative and side-effect-light so the surrounding logic stays unit-testable.

---

## 15. Gotchas & Best Practices

A consolidated checklist of the mistakes that bite real projects, with the fix for each.

### 15.1 Cleanup leaks — always scope through `useGSAP` [I/A]

The #1 React+GSAP bug. Tweens, infinite loops, and especially **ScrollTriggers** created outside a managed context survive unmount and route changes, then duplicate and fire against detached DOM. **Fix:** route *all* React GSAP through `useGSAP` (it reverts on unmount/dep-change), and wrap event-handler animations in `contextSafe`. Never create a ScrollTrigger in a bare `useEffect` without manual `.kill()`.

### 15.2 StrictMode double-invocation [I/A]

React 18/19 StrictMode mounts components twice in dev to surface side-effect bugs. Naive effects create two sets of animations. **Fix:** `useGSAP` reverts the first context before the second runs — it's StrictMode-safe by design. If you must use raw `useEffect`, return `() => ctx.revert()`.

### 15.3 Selector strings vs refs & scope [I/A]

A global selector like `gsap.to(".box", ...)` matches *every* `.box` on the page, including other components' — a classic cross-contamination bug. **Fix:** always pass `scope` to `useGSAP` so selector strings are scoped to your container; use **refs** for single, specific elements.

### 15.4 FOUC on entrance, scroll, and SplitText [I/A]

Because `gsap.from` reads the final state, and because React/Next render the final markup on the server, content can flash in its end state before animating. **Fixes:** set the start/hidden state in **CSS** so server-rendered HTML looks pre-animation, then animate **to** the visible state; for SplitText, hide the text until split+animation are ready; call `ScrollTrigger.refresh()` and split after `document.fonts.ready`.

### 15.5 Register plugins once, in client code [I/A]

Forgetting `registerPlugin` → silent no-op + console warning. Registering at the top of a module a Server Component imports → server error. **Fix:** `gsap.registerPlugin(...)` in `"use client"` code, once (a shared init module or per client file; duplicates are harmless).

### 15.6 `useGSAP` dependencies [I/A]

Like `useEffect`, stale deps mean the animation reads old values; missing deps mean it never re-runs when it should. **Fix:** list every reactive value the callback reads in `dependencies`. Use `revertOnUpdate: true` if a dep change should fully roll back prior inline styles before re-running.

### 15.7 Conflicting tweens & `overwrite` [I/A]

Two tweens animating the *same property* of the *same target* at once fight each other and stutter. **Fix:** GSAP's `overwrite` controls this. `overwrite: "auto"` (smartest — only kills conflicting *properties* on the same target), `overwrite: true` (kill all other tweens of the target), `overwrite: false` (default — let them coexist). For hover-in/hover-out handlers that can interrupt each other, `overwrite: "auto"` or `"true"` prevents jitter.

```js
// Hover handlers that interrupt cleanly instead of fighting:
const onEnter = () => gsap.to(el, { scale: 1.1, overwrite: "auto" });
const onLeave = () => gsap.to(el, { scale: 1, overwrite: "auto" });
```

### 15.8 ScrollTrigger position miscalculation [I/A]

Triggers compute `start`/`end` at creation time. If layout changes afterward (lazy images load, fonts swap, accordions open, async data arrives), positions are wrong. **Fix:** call `ScrollTrigger.refresh()` after layout-affecting changes; refresh on `document.fonts.ready`; use `invalidateOnRefresh: true` for triggers whose values depend on measured sizes.

### 15.9 `from` re-run pitfalls [I/A]

Re-running a `gsap.from` (e.g. on a dep change) can read an already-animated state as the "destination," producing wrong results. **Fix:** prefer `fromTo` (deterministic both ends) for animations that may re-run, and combine with `useGSAP`'s `revertOnUpdate`.

### 15.10 Don't fight React for the DOM [I/A]

Let React own *structure* and GSAP own *motion*. Don't have GSAP add/remove DOM nodes React is also managing, or set inline styles React will overwrite on its next render. Animate transforms/opacity (which React doesn't touch), and drive structural change through React state — then animate the transition with Flip.

### 15.11 Best-practice summary table [I/A]

| Do | Don't |
|---|---|
| Route all React GSAP through `useGSAP` with `scope` | Create tweens/ScrollTriggers in bare `useEffect` without cleanup |
| Wrap handler animations in `contextSafe` | Create handler animations directly (they leak) |
| Animate `x`/`y`/`scale`/`opacity` | Animate `left`/`top`/`width`/`height` in hot paths |
| Set hidden start state in CSS to avoid FOUC | Rely on `gsap.from` alone in SSR (flash) |
| Register plugins once in client code | Register in server-imported modules |
| `revert()` SplitText; refresh on fonts ready | Leave text split (a11y) or measure before fonts load |
| Respect `prefers-reduced-motion` | Ship parallax/scroll-jacking unconditionally |
| Use `matchMedia` for responsive + reduced motion | Manually tear down on resize |

---

## 16. Worked Example — A Production Scroll-Telling Page (Next.js 16)

This section ties the whole guide together into **one realistic, production-shaped component**: a scroll-telling section that pins, plays a scrubbed multi-stage timeline as you scroll, reveals content on enter, respects reduced-motion, survives resize, and cleans up perfectly on route change. Every decision here reflects a rule from earlier sections — read the comments as a checklist of *why*, not just *what*.

### 16.1 The structure & the FOUC-free strategy [A]

The cardinal SSR rule (§9.3): **set the "animated-from" state in CSS, not JS**, so the server-rendered HTML already looks right and there's no flash before hydration. We hide/offset the animated bits with a class, then GSAP animates *from* that state and clears it. We also gate everything behind `prefers-reduced-motion` (§12.2) and scope all of it through `useGSAP` (§7) so cleanup is automatic.

```css
/* scene.module.css — the initial state lives in CSS so SSR matches the pre-animation frame */
.fadeUp   { opacity: 0; transform: translateY(40px); }      /* GSAP animates FROM this, then clears it */
.panel    { min-height: 100svh; display: grid; place-items: center; }
@media (prefers-reduced-motion: reduce) {
  .fadeUp { opacity: 1; transform: none; }                  /* reduced-motion users see the final state immediately */
}
```

### 16.2 The component [A]

```jsx
"use client";                                   // §9.1 — GSAP is browser-only; this whole tree is client
import { useRef } from "react";
import gsap from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";
import { useGSAP } from "@gsap/react";
import styles from "./scene.module.css";

gsap.registerPlugin(ScrollTrigger, useGSAP);     // §9.2 — register once, in client code

export default function ScrollStory() {
  const root = useRef(null);                      // scope element — every selector is resolved within it

  useGSAP(() => {
    // §12.2 — honor reduced motion. matchMedia auto-reverts when the query stops matching,
    // and its cleanup is tied to useGSAP's scope, so we get correct teardown for free.
    const mm = gsap.matchMedia();

    mm.add(
      { motionOK: "(prefers-reduced-motion: no-preference)" },
      (ctx) => {
        if (!ctx.conditions.motionOK) return;     // reduced-motion users: CSS already shows final state, do nothing

        // 1) Reveal-on-enter for every .fadeUp (clearProps so the inline styles don't linger — §15.4)
        gsap.utils.toArray(`.${styles.fadeUp}`, root.current).forEach((el) => {
          gsap.from(el, {
            opacity: 0, y: 40, duration: 0.8, ease: "power2.out", clearProps: "opacity,transform",
            scrollTrigger: { trigger: el, start: "top 85%", toggleActions: "play none none reverse" }, // §8.3
          });
        });

        // 2) A pinned, scrubbed multi-stage timeline — the "story" beats (§5 + §8.2/8.4)
        const tl = gsap.timeline({
          scrollTrigger: {
            trigger: ".pinned",
            start: "top top",
            end: "+=2000",                         // pin lasts 2000px of scroll
            pin: true,
            scrub: 1,                              // §8.3 — tie progress to scroll, 1s catch-up smoothing
            anticipatePin: 1,
            invalidateOnRefresh: true,             // §8.9 — recompute on resize/font-load
            snap: { snapTo: "labelsDirectional", duration: 0.3, ease: "power1.inOut" }, // §8.8 — settle on beats
          },
        });
        tl.addLabel("beat1")
          .to(".headline", { xPercent: -10, opacity: 1, ease: "none" })
          .addLabel("beat2")
          .from(".bg-layer", { yPercent: 20, ease: "none" }, "<")   // parallax against the same scroll (§8.10)
          .addLabel("beat3")
          .to(".caption", { opacity: 1, y: 0, ease: "none" });

        // 3) A reading-progress bar driven by the section's own progress (§8.10)
        ScrollTrigger.create({
          trigger: root.current,
          start: "top top",
          end: "bottom bottom",
          onUpdate: (self) =>
            gsap.set(".progress", { scaleX: self.progress, transformOrigin: "left center" }),
        });
      }
    );
    // No manual cleanup needed: useGSAP reverts everything created in this scope on unmount/route change (§7.2, §9.5).
  }, { scope: root });

  return (
    <main ref={root}>
      <div className="progress" style={{ position: "fixed", top: 0, left: 0, height: 3, width: "100%", background: "tomato", transform: "scaleX(0)", zIndex: 50 }} />
      <section className={styles.panel}><h1 className={styles.fadeUp}>Scroll to begin</h1></section>

      <section className="pinned" style={{ position: "relative", overflow: "hidden" }}>
        <div className="bg-layer" style={{ position: "absolute", inset: 0, background: "linear-gradient(#222,#000)" }} />
        <h2 className="headline" style={{ opacity: 0.2, position: "relative" }}>The story unfolds</h2>
        <p className="caption" style={{ opacity: 0, transform: "translateY(20px)", position: "relative" }}>…one beat at a time.</p>
      </section>

      <section className={styles.panel}><p className={styles.fadeUp}>The end — content continues normally.</p></section>
    </main>
  );
}
```

### 16.3 What this example demonstrates (the checklist) [A]

Every rule in the guide shows up here — this is the mental checklist for *any* production GSAP+React scene:

| Concern | How it's handled here | Section |
|---|---|---|
| Client-only | `"use client"` on the whole tree | §9.1 |
| Register once | `registerPlugin` at module top | §9.2 |
| No FOUC | Initial state in CSS module, `clearProps` after | §9.3 / §15.4 |
| Reduced motion | `matchMedia` gate; CSS shows final state | §12.2 |
| Scoped + auto-cleanup | `useGSAP({ scope })`; nothing manual | §7.2 |
| Route-change safety | `useGSAP` reverts on unmount | §9.5 |
| Scrub vs reveal | `scrub` for the story, `toggleActions` for reveals | §8.3 |
| Pin correctness on resize | `invalidateOnRefresh` + functional ends | §8.9 |
| Snap to beats | `snapTo: "labelsDirectional"` | §8.8 |
| Parallax & progress | layered `yPercent` + `onUpdate` bar | §8.10 |

> **Make it yours:** drop this into a Next.js 16 app, then extend it — add a `SplitText` headline (§10.1), swap the pinned section for the **horizontal scroll** recipe (§8.9), or make the final panel a **Flip** gallery (§11.6). Resize the window and reload mid-scroll to confirm `invalidateOnRefresh` keeps the pin honest, and navigate away with React DevTools open to confirm zero orphan tweens.

---

## 17. Study Path & Build-to-Learn Projects

Knowledge sticks when you build. Follow this progression; each project layers new concepts on the last, and each is framed in **Next.js 16 + React 19** with `useGSAP` and proper cleanup so you practice the production workflow from day one.

### 17.1 The learning sequence

1. **Fundamentals (Sections 1–4).** Install `gsap` + `@gsap/react`. In a single `"use client"` component, animate a box with `to`/`from`/`fromTo`/`set`. Cycle through every ease family until you can predict the feel from the name. Build a relative-value playground (`"+="`, `xPercent`).
2. **Timelines & stagger (Sections 5–6).** Build a multi-step intro timeline using the position parameter and labels. Add a staggered list reveal. Make the whole timeline play/pause/reverse from buttons (store it in a ref).
3. **React integration (Section 7).** Rebuild the above purely through `useGSAP` — scope every animation, move click animations into `contextSafe`, confirm StrictMode doesn't double-fire, and verify cleanup with React DevTools (unmount the component, ensure no orphan tweens).
4. **ScrollTrigger (Section 8).** Add scroll reveals (`toggleActions`), a scrubbed parallax, and a pinned horizontal section. Make it responsive and reduced-motion-aware with `matchMedia`.
5. **Next.js hardening (Section 9).** Eliminate FOUC via CSS-first hidden states, verify no hydration warnings, test navigation cleanup between routes, and dynamically import one measure-heavy scene.
6. **Plugins & Flip (Sections 10–11).** Add a SplitText hero (with `revert` and fonts-ready refresh), a DrawSVG signature, and a Flip-powered filterable gallery.
7. **Polish (Sections 12–15).** Audit accessibility (reduced motion, SplitText revert, no scroll-jacking), profile in the Performance panel, refactor animations into reusable hooks/components, and apply the gotchas checklist.

### 17.2 Build-to-learn projects (Next.js 16, all with `useGSAP`)

- **Scroll-telling landing page.** A long-scroll page with pinned sections, scrubbed parallax layers, and reveal-on-enter content. Practices ScrollTrigger pin/scrub/toggleActions, `matchMedia` responsiveness, and FOUC-free SSR. The capstone of scroll work.
- **Animated navigation.** A header whose menu opens via a paused timeline (`play`/`reverse` on hover/click), with staggered items and a morphing hamburger→close icon (MorphSVG). Practices timelines-in-refs, `contextSafe`, and a plugin.
- **Flip image gallery.** A filterable/reorderable grid where tiles smoothly move, enter, and leave as the filter changes, plus a thumbnail→detail shared-element transition. Practices the Flip four-step pattern with React state and `useGSAP` dependencies — the most "React-y" GSAP skill.
- **SplitText hero.** A landing hero where the headline animates in by characters/words/lines on load, correctly handling fonts-ready timing, FOUC prevention, accessibility `revert`, and reduced-motion fallback. Practices SplitText end-to-end and a11y discipline.
- **Stretch: an interactive product showcase** combining ScrollSmoother, MotionPath, Draggable+Inertia, and a master timeline — the "everything plugin" project that forces clean architecture (reusable hooks, centralized eases/durations, and rigorous cleanup).

### 17.3 Where to go next

Compare your GSAP solutions against equivalent **[Motion (animation)](MOTION_ANIMATION_GUIDE.md)** implementations to internalize the imperative-vs-declarative tradeoff and learn when to reach for each. Strengthen the foundations with the **[JavaScript](JAVASCRIPT_GUIDE.md)**, **[React 19](REACT_19_GUIDE.md)**, **[Next.js 16](NEXTJS_16_GUIDE.md)**, **[CSS](CSS_GUIDE.md)**, and **[Tailwind CSS](TAILWIND_CHEATSHEET.md)** guides — GSAP sits on top of all of them, and the better you know the rendering pipeline and React lifecycle, the more confidently you'll animate.

---

*This guide targets GSAP 3.13+ with the official `@gsap/react` package and `useGSAP()`, React 19, and Next.js 16, current in 2026. As of the 2024 Webflow acquisition, GSAP and all its plugins are 100% free. gsap.com/docs is the canonical source for any API changes after this guide's cutoff.*
