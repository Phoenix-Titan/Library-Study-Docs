# JavaScript — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone learning JavaScript the language — from first variable to closures, prototypes, async, and metaprogramming — offline. Every concept has runnable, commented code. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced. This guide covers the **language** (engine-agnostic) plus browser essentials; for deep server/runtime APIs see the **Node.js guide** in this library.
>
> **Version note:** This guide targets **ECMAScript 2024 / 2025** (current in 2026). Modern features assumed available in current browsers, Node 22+, Deno, and Bun:
> - Immutable array methods: `toSorted()`, `toReversed()`, `with()`, `toSpliced()`.
> - `Object.groupBy()` / `Map.groupBy()`, `Promise.withResolvers()`, `Array.fromAsync()`.
> - `structuredClone()` for deep cloning, top-level `await` in modules.
> - The `using` / `await using` declarations for **explicit resource management** (Symbol.dispose) — newest, check support.
> - **Temporal** (modern date/time) and **decorators** are landing — flagged as **⚡ Version note** where used.
>
> JavaScript ≠ TypeScript: TS adds static types on top (pointer in §18). The author is on **Windows 11**; the FS/OS section (§14) uses Node and notes Windows specifics. Confirm fast-moving APIs at MDN.

---

## Table of Contents

1. [What JS Is & Running It](#1-what-js-is--running-it) **[B]**
2. [Variables & Types](#2-variables--types) **[B]**
3. [Operators & Modern Syntax](#3-operators--modern-syntax) **[B]**
4. [Functions](#4-functions) **[B/I]**
5. [Scope, Hoisting & Closures](#5-scope-hoisting--closures) **[I]**
6. [`this`, call/apply/bind](#6-this-callapplybind) **[I]**
7. [Objects](#7-objects) **[B/I]**
8. [Prototypes & Classes](#8-prototypes--classes) **[I]**
9. [Arrays](#9-arrays) **[B/I]**
10. [Strings, Numbers, Dates & Regex](#10-strings-numbers-dates--regex) **[B/I]**
11. [Collections, Iterators & Generators](#11-collections-iterators--generators) **[I]**
12. [Asynchronous JavaScript](#12-asynchronous-javascript) **[I/A]**
13. [Modules](#13-modules) **[I]**
14. [File System, OS & Command Execution (via a Runtime)](#14-file-system-os--command-execution-via-a-runtime) **[I]**
15. [Error Handling](#15-error-handling) **[I]**
16. [The DOM & Browser APIs](#16-the-dom--browser-apis) **[B/I]**
17. [Advanced & Metaprogramming](#17-advanced--metaprogramming) **[A]**
18. [Tooling Overview](#18-tooling-overview)
19. [Gotchas & Best Practices](#19-gotchas--best-practices) **[A]**
20. [Study Path & Build-to-Learn Projects](#20-study-path--build-to-learn-projects)

---

## 1. What JS Is & Running It

JavaScript is a dynamically-typed, single-threaded, prototype-based language with first-class functions. It runs in **browsers** (V8/Chrome, SpiderMonkey/Firefox, JavaScriptCore/Safari) and on servers via **Node.js**, **Deno**, and **Bun**.

```js
// In a browser: open DevTools console, or add <script src="app.js"></script> / <script type="module">.
// With Node: `node app.js`. With Deno: `deno run app.js`. With Bun: `bun app.js`.
console.log("hello");        // prints to console
```

```js
"use strict";   // opt into strict mode: turns silent errors into thrown errors,
                // forbids accidental globals, etc. ES MODULES are strict automatically.
```

- A `.js` `<script>` is a classic script (shared global scope).
- A `<script type="module">` (or `.mjs` / `"type":"module"`) is an **ES module**: its own scope, `import`/`export`, deferred, strict.

---

## 2. Variables & Types

### 2.1 Declarations **[B]**

```js
const PI = 3.14159;     // can't be reassigned (but objects it points to can still mutate)
let count = 0;          // block-scoped, reassignable
count = 1;
var old = "avoid";      // function-scoped, hoisted — legacy; prefer let/const

// Rule of thumb: use const by default, let when you must reassign, never var.
```

### 2.2 Types **[B]**

Seven primitives + objects:

```js
// Primitives (immutable, compared by value):
const n = 42;            // number  (one type for ints & floats; IEEE-754 double)
const big = 9007199254740993n;  // bigint (arbitrary-precision integers, note the n)
const s = "text";        // string
const b = true;          // boolean
const u = undefined;     // a variable declared but not assigned
const z = null;          // intentional "no value"
const sym = Symbol("id");// symbol (unique identifier)

// Everything else is an object (compared by reference):
const arr = [1, 2, 3];
const obj = { a: 1 };
const fn = () => {};

console.log(typeof n);       // "number"
console.log(typeof s);       // "string"
console.log(typeof undefined);// "undefined"
console.log(typeof null);    // "object"  <- historical bug, remember it
console.log(typeof fn);      // "function"
console.log(Array.isArray(arr)); // true (typeof arr is "object")
```

### 2.3 Coercion, truthiness & equality **[B, important]**

```js
// Falsy values: false, 0, -0, 0n, "", null, undefined, NaN. Everything else is truthy.
if ("")        {} else { console.log("empty string is falsy"); }
if ([])        { console.log("empty array is TRUTHY"); }   // surprises people

// == does type coercion (avoid); === compares value AND type (use this):
console.log(1 == "1");    // true  (coerced)
console.log(1 === "1");   // false (different types)
console.log(null == undefined);  // true
console.log(null === undefined); // false
console.log(NaN === NaN);        // false! use Number.isNaN(x)

// Explicit conversion is clearer than relying on coercion:
Number("42"); parseInt("42px", 10); String(42); Boolean(0);
```

---

## 3. Operators & Modern Syntax

```js
// Arithmetic: + - * / % **    String concat with +
console.log(2 ** 10);          // 1024
console.log(7 % 3);            // 1

// Template literals (backticks) — interpolation & multiline:
const name = "Ada";
console.log(`Hello, ${name}! 1+1 = ${1 + 1}`);

// Optional chaining ?. — short-circuits to undefined if a link is null/undefined:
const user = { profile: { email: "a@x.com" } };
console.log(user?.profile?.email);   // "a@x.com"
console.log(user?.address?.city);    // undefined (no error)
console.log(user.save?.());          // calls only if save exists

// Nullish coalescing ?? — fallback ONLY for null/undefined (not for 0 or ""):
const port = config.port ?? 8080;    // 0 would be kept; || would replace it
let cache;
cache ??= new Map();                 // assign only if null/undefined

// Spread / rest:
const a = [1, 2], b = [3, 4];
const both = [...a, ...b];            // [1,2,3,4]  (spread)
const clone = { ...user };           // shallow object copy
function sum(...nums) { return nums.reduce((t, n) => t + n, 0); }  // rest

// Destructuring:
const [first, second, ...others] = [10, 20, 30, 40];
const { profile: { email } = {} } = user;         // nested + default
const { x = 0, y = 0 } = point;                    // defaults
function greet({ name, greeting = "Hi" }) { return `${greeting} ${name}`; }  // param destructure
```

---

## 4. Functions

### 4.1 Three ways to define **[B]**

```js
function declared(a, b) { return a + b; }       // declaration (hoisted)
const expr = function (a, b) { return a + b; };  // expression
const arrow = (a, b) => a + b;                   // arrow (concise, lexical `this`)

const square = x => x * x;                       // single param, single expression
const noArgs = () => 42;
const makeObj = () => ({ a: 1 });                // wrap object literal in ()
```

### 4.2 Parameters **[B/I]**

```js
function connect(host, port = 5432, { ssl = false } = {}) {  // defaults + destructured opts
  return `${host}:${port} ssl=${ssl}`;
}
connect("db");                       // db:5432 ssl=false
connect("db", 6543, { ssl: true });

// Arrow functions DON'T have their own `this` or `arguments` (see §6) — that's the point.
```

### 4.3 Higher-order functions **[I]**

```js
const nums = [1, 2, 3, 4];
const doubled = nums.map(n => n * 2);
const evens = nums.filter(n => n % 2 === 0);
const total = nums.reduce((acc, n) => acc + n, 0);

// Functions that return functions:
const multiplier = factor => n => n * factor;
const triple = multiplier(3);
console.log(triple(5));              // 15

// IIFE (Immediately Invoked Function Expression) — old way to make a private scope:
(function () { /* isolated */ })();
```

---

## 5. Scope, Hoisting & Closures

### 5.1 Scope & hoisting **[I]**

```js
// `var` is function-scoped and hoisted (initialized as undefined):
console.log(v);   // undefined (not an error)
var v = 1;

// `let`/`const` are block-scoped and in the "temporal dead zone" until declared:
// console.log(x);  // ReferenceError
let x = 1;

function f() {
  if (true) { let block = "only here"; }
  // console.log(block);  // ReferenceError — block-scoped
}
```

### 5.2 Closures **[I, fundamental]**

**What it is.** A closure is the combination of a function *and* the variables it "remembers" from the scope where it was **defined** (not where it's called). When a function is created inside another function and references the outer function's variables, JavaScript keeps those variables alive for as long as the inner function exists — even after the outer function has returned.

**Why it works this way.** Functions in JS are values that can be returned, stored, and passed around. If a returned function still refers to a local variable of its parent, that variable obviously can't be thrown away when the parent returns — so the engine keeps the variable on the heap, bound to that function. The set of variables a function "closes over" is fixed lexically (by where the code is written), which is why it's called *lexical* scope.

**What it's used for.** Closures are how you get **private state** (data only accessible through the functions you return), **factory functions** (functions that build customized functions), and **callbacks that remember context** (event handlers, `setTimeout`, array methods). They're the foundation of the module pattern and much of functional JS.

```js
function makeCounter() {
  let count = 0;                 // captured by the returned function
  return {
    inc() { return ++count; },
    get() { return count; },
  };
}
const c = makeCounter();
console.log(c.inc(), c.inc(), c.get());   // 1 2 2  (count is private state)

// Classic gotcha — closures share the loop variable with var:
const fns = [];
for (var i = 0; i < 3; i++) fns.push(() => i);
console.log(fns.map(f => f()));   // [3, 3, 3]  — all close over the SAME i
// Fix: use let (a fresh binding per iteration):
const fns2 = [];
for (let j = 0; j < 3; j++) fns2.push(() => j);
console.log(fns2.map(f => f()));  // [0, 1, 2]
```

---

## 6. `this`, call/apply/bind

**The core rule.** In a regular function, `this` is decided **when and how the function is called**, not where it was written. The same function can have a different `this` on every call. This trips up everyone at first, so memorize the four binding rules, in priority order: **[I]**

1. **`new` binding** — `new Foo()` makes `this` the brand-new object being constructed.
2. **Explicit binding** — `fn.call(obj)`, `fn.apply(obj)`, or `fn.bind(obj)` force `this` to be `obj`.
3. **Method (implicit) binding** — `obj.fn()` makes `this` the object left of the dot (`obj`).
4. **Default binding** — a plain `fn()` call sets `this` to `undefined` in strict mode (or the global object in sloppy mode).

**The big exception: arrow functions.** Arrow functions have **no `this` of their own**; they capture `this` lexically from the surrounding scope at definition time, and it can never be reassigned (not even by `call`/`bind`). That's exactly why arrows are the right choice for callbacks (event handlers, `setTimeout`, array methods) where you want to keep the outer `this` — and the wrong choice for object methods that need a dynamic `this`.

**Why this design?** It lets one function be reused across many objects (the method borrows whatever object it's attached to), which is the basis of prototypes and classes (§8). The cost is the "detached method" footgun below.

```js
const obj = {
  name: "Ada",
  greet() { return `Hi ${this.name}`; },   // `this` is obj when called as obj.greet()
};
console.log(obj.greet());       // Hi Ada

const detached = obj.greet;
// console.log(detached());     // `this` is undefined (strict) — "Cannot read name"

// Arrow functions capture `this` LEXICALLY (from the enclosing scope):
const timer = {
  seconds: 0,
  start() {
    setInterval(() => { this.seconds++; }, 1000);  // arrow keeps `this` = timer
  },
};

// Explicitly set `this`:
function intro(greeting) { return `${greeting}, ${this.name}`; }
console.log(intro.call(obj, "Hello"));        // call: args listed
console.log(intro.apply(obj, ["Hey"]));       // apply: args as array
const bound = intro.bind(obj);                // bind: returns a new fn with fixed this
console.log(bound("Yo"));
```

---

## 7. Objects

```js
// Literal, shorthand, computed keys:
const role = "admin";
const id = 7;
const user = {
  name: "Ada",                 // property
  greet() { return "hi"; },    // method shorthand
  [role]: true,                // computed key -> admin: true
  id,                          // shorthand for id: id
};

// Access / mutate:
console.log(user.name, user["name"]);
user.email = "a@x.com";
delete user.id;
console.log("name" in user);             // true
console.log(Object.keys(user));          // ['name','greet'... ]
console.log(Object.values(user));
console.log(Object.entries(user));       // [['name','Ada'], ...]

// Merge / copy (shallow):
const merged = { ...user, role: "eng" }; // or Object.assign({}, user, {...})

// Getters / setters:
const temp = {
  _c: 0,
  get f() { return this._c * 9 / 5 + 32; },
  set f(v) { this._c = (v - 32) * 5 / 9; },
};
temp.f = 212; console.log(temp._c);       // 100

// Freeze (shallow immutability):
const frozen = Object.freeze({ a: 1 });
frozen.a = 2;                             // silently ignored (throws in strict mode)

// JSON:
const json = JSON.stringify(user, null, 2);   // object -> string (pretty)
const back = JSON.parse(json);                 // string -> object

// Deep clone (modern):
const deep = structuredClone(user);
```

---

## 8. Prototypes & Classes

### 8.1 The prototype chain **[I]**

**What it is.** JavaScript doesn't have classes at its core (classes are a newer convenience). Its real inheritance mechanism is the **prototype**: every object has a hidden link to another object — its prototype. When you read a property, the engine looks on the object itself; if it's not there, it follows the link to the prototype, then *that* object's prototype, and so on up the **prototype chain** until it finds the property or reaches `null`.

**Why it works this way / what it's for.** This lets many objects **share** behavior without each carrying its own copy. All arrays share the methods on `Array.prototype` (`map`, `filter`…); defining a method once on a prototype makes it available to every instance and saves memory. It also enables inheritance: put shared methods on a parent prototype and have child objects link to it. Writing a property always sets it on the *own* object (it never modifies the prototype), so instances can override inherited values.

```js
const animal = { eats: true };
const dog = Object.create(animal);   // dog's prototype is animal
dog.barks = true;
console.log(dog.eats);               // true (found on the prototype)
console.log(Object.getPrototypeOf(dog) === animal);  // true
```

### 8.2 Classes (syntactic sugar over prototypes) **[I]**

```js
class Animal {
  #id;                          // private field (the # is part of the name)
  static count = 0;             // static (class-level) field

  constructor(name) {
    this.name = name;
    this.#id = ++Animal.count;
  }
  speak() { return `${this.name} makes a sound`; }   // on the prototype
  get id() { return this.#id; }                       // getter
  static create(name) { return new Animal(name); }    // static method
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);                // must call before using `this`
    this.breed = breed;
  }
  speak() { return `${this.name} barks`; }   // override
  fetch() { return `${super.speak()} ... then fetches`; }  // call parent
}

const d = new Dog("Rex", "Lab");
console.log(d.speak(), d.id, d instanceof Animal);   // Rex barks 1 true
```

---

## 9. Arrays

```js
const a = [3, 1, 2];

// Iterate / transform (these do NOT mutate):
a.forEach((x, i) => console.log(i, x));
const mapped = a.map(x => x * 2);            // [6,2,4]
const odds = a.filter(x => x % 2);           // [3,1]
const sum = a.reduce((t, x) => t + x, 0);    // 6
console.log(a.find(x => x > 1));             // 3 (first match)
console.log(a.findIndex(x => x > 1));        // 0
console.log(a.some(x => x > 2), a.every(x => x > 0));  // true true
console.log(a.includes(2));                  // true
console.log([[1], [2, 3]].flat());           // [1,2,3]
console.log([1, 2].flatMap(x => [x, x * 10])); // [1,10,2,20]

// Mutating methods (change the array in place):
a.push(9); a.pop(); a.unshift(0); a.shift();
a.splice(1, 1, "x");        // remove 1 at index 1, insert "x"
a.sort((p, q) => p - q);    // numeric sort (default sort is lexicographic!)
a.reverse();

// Immutable equivalents (ES2023) — return a NEW array:
const sorted = a.toSorted((p, q) => p - q);
const reversed = a.toReversed();
const replaced = a.with(0, 99);   // copy with index 0 set to 99

// Build & destructure:
console.log(Array.from("abc"));            // ['a','b','c']
console.log(Array.from({ length: 3 }, (_, i) => i));  // [0,1,2]
const [head, ...tail] = [1, 2, 3];          // head=1, tail=[2,3]

// Group (ES2024):
const people = [{ age: 30, name: "A" }, { age: 30, name: "B" }, { age: 40, name: "C" }];
console.log(Object.groupBy(people, p => p.age));   // { 30: [...], 40: [...] }
```

---

## 10. Strings, Numbers, Dates & Regex

```js
// Strings (immutable):
const s = "Hello, World";
console.log(s.length, s.toUpperCase(), s.includes("World"));
console.log(s.slice(0, 5), s.indexOf("World"), s.replace("World", "JS"));
console.log("  trim me  ".trim(), "a,b,c".split(","), ["a","b"].join("-"));
console.log("ab".padStart(5, "*"), "5".padEnd(3, "0"));
console.log("na".repeat(3));

// Numbers & Math:
console.log((1234.5678).toFixed(2));       // "1234.57"
console.log(Number.isInteger(5), Number.parseFloat("3.14px"));
console.log(Math.max(1, 9, 3), Math.min(...[1, 9, 3]));
console.log(Math.round(2.5), Math.floor(2.9), Math.ceil(2.1), Math.abs(-3));
console.log(Math.random());                 // [0, 1)

// Dates (the legacy Date API — clunky):
const now = new Date();
console.log(now.toISOString(), now.getFullYear());
console.log(new Date("2026-06-21").getTime());   // epoch millis
// ⚡ Version note: the modern Temporal API (Temporal.Now, PlainDate, etc.) is arriving —
// prefer it when available; until then many use date-fns / Day.js.

// Regular expressions:
const re = /(\d{4})-(\d{2})-(\d{2})/;
const m = "date: 2026-06-21".match(re);
console.log(m[0], m[1]);                    // "2026-06-21", "2026"
console.log(/\d+/g.test("a1b2"));           // true
console.log("a1b2".replace(/\d/g, "#"));    // "a#b#"
console.log([..."a1b2".matchAll(/\d/g)].map(x => x[0]));  // ['1','2']
// Named groups:
const m2 = "2026-06".match(/(?<year>\d{4})-(?<month>\d{2})/);
console.log(m2.groups.year);                // "2026"
```

---

## 11. Collections, Iterators & Generators

```js
// Map — keys of ANY type, insertion order, easy size:
const map = new Map();
map.set("a", 1).set(42, "num").set(obj, "by ref");
console.log(map.get("a"), map.has(42), map.size);
for (const [k, v] of map) console.log(k, v);

// Set — unique values:
const set = new Set([1, 1, 2, 3]);
console.log([...set]);                       // [1,2,3]
set.add(4); console.log(set.has(2));

// Dedupe an array:
console.log([...new Set([1, 1, 2])]);        // [1,2]

// WeakMap/WeakSet — keys held weakly (don't prevent GC); for metadata/caches.

// Iterators & for...of vs for...in:
for (const x of [10, 20]) console.log(x);    // VALUES (of any iterable)
for (const i in [10, 20]) console.log(i);    // KEYS/indices ("0","1") — avoid for arrays

// Generators — lazy sequences:
function* range(start, end) {
  for (let i = start; i < end; i++) yield i;
}
console.log([...range(0, 5)]);               // [0,1,2,3,4]
const gen = range(0, 3);
console.log(gen.next());                     // { value: 0, done: false }

// Symbols — unique keys; also used to customize behavior (Symbol.iterator):
const ID = Symbol("id");
const o = { [ID]: 123 };
```

---

## 12. Asynchronous JavaScript

### 12.1 The event loop **[I, fundamental]**

**The problem it solves.** JavaScript runs on a **single thread** — one call stack, one thing at a time. If a network request blocked that thread, the whole page (or server) would freeze until it returned. The solution is to be *asynchronous*: slow operations (timers, network `fetch`, file I/O) are handed off to the environment (the browser or Node), which runs them elsewhere and hands back a **callback** to run when they finish.

**How the loop works.** Finished callbacks don't interrupt running code — they wait in queues. The **event loop** is the simple rule that schedules them: run the current synchronous code until the **call stack** is empty; then completely drain the **microtask queue** (resolved promise continuations — `.then` callbacks and code after `await`); then take **one** task from the **macrotask queue** (a timer callback, an I/O event); then drain microtasks again; repeat forever.

**Why the ordering matters.** Because microtasks are *fully* drained before the next macrotask, a promise always resolves before a `setTimeout(…, 0)` queued at the same moment. Understanding this explains otherwise-baffling output and timing bugs. The other practical lesson: a long synchronous computation blocks the loop — nothing else runs until it finishes — so keep synchronous work short.

```js
console.log("1");
setTimeout(() => console.log("4 (macrotask)"), 0);
Promise.resolve().then(() => console.log("3 (microtask)"));
console.log("2");
// Output order: 1, 2, 3, 4  — microtasks beat timers.
```

### 12.2 Promises **[I]**

```js
const p = new Promise((resolve, reject) => {
  setTimeout(() => resolve("done"), 100);     // or reject(new Error(...))
});
p.then(v => console.log(v))
 .catch(err => console.error(err))
 .finally(() => console.log("cleanup"));

// Combinators:
const a = Promise.resolve(1), b = Promise.resolve(2);
await Promise.all([a, b]);        // [1,2]; rejects if ANY rejects
await Promise.allSettled([a, b]); // array of {status, value/reason} — never rejects
await Promise.race([a, b]);       // first to settle (resolve OR reject)
await Promise.any([a, b]);        // first to FULFILL

// New: get resolve/reject out (ES2024):
const { promise, resolve, reject } = Promise.withResolvers();
```

### 12.3 async / await **[I]**

```js
async function load(id) {
  try {
    const res = await fetch(`/api/users/${id}`);   // pauses here without blocking the thread
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    const user = await res.json();
    return user;
  } catch (err) {
    console.error("load failed:", err);
    throw err;                                       // rethrow or return a default
  }
}

// Run in parallel — start both, then await:
const [u1, u2] = await Promise.all([load(1), load(2)]);

// PITFALL: awaiting in a loop is sequential (slow). Map to promises then Promise.all:
const ids = [1, 2, 3];
const users = await Promise.all(ids.map(load));      // concurrent
// const slow = []; for (const id of ids) slow.push(await load(id));  // sequential
```

---

## 13. Modules

```js
// math.js — named + default exports
export const PI = 3.14159;
export function add(a, b) { return a + b; }
export default class Calculator {}

// main.js
import Calculator, { PI, add } from "./math.js";    // default + named
import * as math from "./math.js";                   // namespace import
import { add as plus } from "./math.js";             // rename

// Dynamic import (lazy, returns a promise) — for code-splitting / conditional loading:
const { default: Calc } = await import("./math.js");
```

CommonJS (older Node style) for contrast: `const x = require("./m")` / `module.exports = {...}`. Prefer ESM in new code; see the Node guide for interop details.

---

## 14. File System, OS & Command Execution (via a Runtime)

> **Important:** the browser **cannot** read your disk, list directories, run programs, or read environment variables — that's by design (security/sandbox). For that you need a **runtime**: Node.js, Deno, or Bun. The examples below are **Node.js**. The author is on Windows 11; Windows specifics are noted.

### 14.1 Files & directories (Node `fs/promises`)

```js
import { readFile, writeFile, mkdir, readdir, rm, stat } from "node:fs/promises";

// Read / write text (always specify encoding to get a string, not a Buffer):
const text = await readFile("notes.txt", "utf8");
await writeFile("out.txt", "hello\n", "utf8");        // overwrites
await writeFile("log.txt", "more\n", { flag: "a" });  // append

// Directories:
await mkdir("data/reports", { recursive: true });     // like mkdir -p
const entries = await readdir("data", { withFileTypes: true });
for (const e of entries) console.log(e.name, e.isDirectory());
await rm("tmp", { recursive: true, force: true });    // delete

// Existence / metadata:
try { await stat("file.txt"); console.log("exists"); }
catch { console.log("missing"); }
```

### 14.2 Paths (`node:path`) — cross-platform

```js
import path from "node:path";
console.log(path.join("data", "reports", "2026.csv"));  // data\reports\2026.csv on Windows
console.log(path.resolve("file.txt"));                  // absolute
console.log(path.basename("/a/b.txt"), path.dirname("/a/b.txt"), path.extname("b.txt"));
// In ESM, get the current file's dir:
import { fileURLToPath } from "node:url";
const __dirname = path.dirname(fileURLToPath(import.meta.url));
// (Node 20.11+: import.meta.dirname / import.meta.filename directly.)
```

### 14.3 OS & system info (`node:os`, `process`)

```js
import os from "node:os";
console.log(os.platform(), os.arch(), os.release());   // 'win32' 'x64' ...
console.log(os.cpus().length, os.totalmem(), os.freemem());
console.log(os.hostname(), os.homedir(), os.userInfo().username);

console.log(process.platform, process.version);         // runtime info
console.log(process.cwd());                             // working directory
console.log(process.env.PATH);                          // environment variable
console.log(process.argv);                              // CLI args (argv[2..] are yours)
process.env.MY_FLAG = "1";                              // set for this process & children
```

### 14.4 Executing external commands (`node:child_process`)

```js
import { execFile, spawn } from "node:child_process";
import { promisify } from "node:util";
const execFileP = promisify(execFile);

// Capture output of a command (args as an ARRAY — no shell, injection-safe):
const { stdout } = await execFileP("git", ["rev-parse", "--abbrev-ref", "HEAD"]);
console.log("branch:", stdout.trim());

// Run real tools:
await execFileP("npm", ["install"], { cwd: "frontend" });
await execFileP("python", ["build.py", "--release"]);
// (Windows: npm is npm.cmd — execFile may need { shell: true } or use spawn with shell.)

// Stream output live with spawn:
const child = spawn("ping", process.platform === "win32" ? ["-n", "3", "127.0.0.1"]
                                                          : ["-c", "3", "127.0.0.1"]);
child.stdout.on("data", d => process.stdout.write(d));
child.on("close", code => console.log("exit", code));

// NEVER build a shell string from untrusted input (command injection):
// exec(`rm ${userInput}`)  // DANGER. Use execFile/spawn with an args array.
```

### 14.5 Building a CLI

```js
import { parseArgs } from "node:util";   // built into Node (no dependency)
const { values, positionals } = parseArgs({
  options: {
    width:   { type: "string", short: "w", default: "800" },
    verbose: { type: "boolean", short: "v" },
  },
  allowPositionals: true,
});
console.log(values.width, values.verbose, positionals);
process.exit(0);   // 0 = success
```

### 14.6 Deno & Bun (brief)

```js
// Deno (secure by default — needs --allow-read / --allow-run flags):
const text = await Deno.readTextFile("notes.txt");
const cmd = new Deno.Command("git", { args: ["status"] });
const { stdout } = await cmd.output();

// Bun:
const file = await Bun.file("notes.txt").text();
await Bun.write("out.txt", "hi");
```

---

## 15. Error Handling

```js
try {
  JSON.parse("{ bad json");
} catch (err) {
  if (err instanceof SyntaxError) console.error("bad JSON:", err.message);
  else throw err;                  // rethrow what you don't handle
} finally {
  // always runs (cleanup)
}

// Built-in error types: Error, TypeError, RangeError, SyntaxError, ReferenceError.
// Custom errors:
class ValidationError extends Error {
  constructor(field, message) {
    super(message);
    this.name = "ValidationError";
    this.field = field;
  }
}
throw new ValidationError("email", "invalid email");

// Async errors: a rejected promise in async/await is caught by try/catch.
// Unhandled rejections crash Node by default — always .catch() or try/catch.
```

---

## 16. The DOM & Browser APIs

Everything so far is the JavaScript *language*. In a browser, JS's main job is to read and change the page. The page is represented as the **DOM (Document Object Model)** — a live, in-memory tree of objects that mirrors your HTML. Each HTML tag becomes a **node** (specifically an *element node*); text becomes *text nodes*. JS can read this tree, change it, and react to user actions — and the browser instantly re-renders.

> **Why a tree?** HTML nests (`<body>` contains `<div>` contains `<p>`…), so the natural representation is a tree of parent/child nodes. The global `document` object is your entry point to it; `window` is the global object (the tab itself).

### 16.1 Selecting elements

You almost always start by *finding* the element(s) you want to work with. There are several methods; in modern code you mostly use `querySelector`/`querySelectorAll`, which take **CSS selectors** (the same syntax you use in stylesheets).

```js
// By id (fastest, returns the single element or null):
const form = document.getElementById("login");

// querySelector returns the FIRST match (or null); takes any CSS selector:
const saveBtn = document.querySelector("#save");          // id selector
const firstItem = document.querySelector(".item");        // class selector
const nestedLink = document.querySelector("nav ul li a"); // descendant selector
const checked = document.querySelector("input[type=checkbox]:checked");

// querySelectorAll returns a STATIC NodeList of ALL matches:
const items = document.querySelectorAll(".item");
items.forEach(el => console.log(el.textContent));         // NodeList has forEach
const itemsArray = [...items];                            // spread to a real array for map/filter

// You can also search WITHIN an element, not just the whole document:
const linksInNav = document.querySelector("nav").querySelectorAll("a");
```

> **Live vs static collections:** `getElementsByClassName`/`getElementsByTagName` return *live* `HTMLCollection`s that auto-update as the DOM changes (and lack `forEach`). `querySelectorAll` returns a *static* `NodeList` — a snapshot. Prefer `querySelector`/`querySelectorAll` for predictability. Always handle the `null` case — `querySelector` returns `null` if nothing matches, and `null.textContent` throws.

### 16.2 Reading & changing content

Once you have an element, you read or write its content. The key choice is **`textContent` vs `innerHTML`**:

```js
const el = document.querySelector("#message");

// textContent — plain text. SAFE: never interprets HTML. Use this by default.
el.textContent = "Hello & welcome <b>";   // shows the literal characters, no bold

// innerHTML — parses the string AS HTML and builds nodes from it.
el.innerHTML = "<strong>Hello</strong>";   // actually makes bold text
// ⚠️ SECURITY (XSS): NEVER put untrusted/user input into innerHTML — it can inject
// <script>/<img onerror> and run attacker code. Use textContent, or sanitize.

console.log(el.textContent);   // read the text inside (and descendants)
console.log(el.innerHTML);     // read the HTML markup inside
```

### 16.3 Attributes, properties, classes & styles

HTML *attributes* (what's in the markup) and DOM *properties* (live values on the object) are related but not identical. For standard things, prefer the property; for custom/`data-*` use the attribute methods.

```js
const img = document.querySelector("img");

// Attributes (string-based, mirror the HTML):
img.getAttribute("src");
img.setAttribute("alt", "A cat");
img.hasAttribute("loading");
img.removeAttribute("hidden");

// Properties (typed, live) — often more convenient:
img.src = "/cat.png";          // same as setAttribute("src", ...) but absolute URL when read
const box = document.querySelector("input");
box.value = "typed text";      // form value (NOT reflected as an attribute)
box.disabled = true;           // boolean property

// data-* attributes via the dataset object:
// <div data-user-id="42"> -> el.dataset.userId === "42"
const card = document.querySelector(".card");
console.log(card.dataset.userId);
card.dataset.state = "open";   // sets data-state="open"

// classList — the right way to manage classes:
el.classList.add("active");
el.classList.remove("hidden");
el.classList.toggle("open");           // add if absent, remove if present
el.classList.toggle("open", isOpen);   // force on/off based on a boolean
console.log(el.classList.contains("active"));

// Inline styles (camelCase property names; values are strings):
el.style.color = "red";
el.style.backgroundColor = "#eee";     // CSS background-color
el.style.display = "none";
// For anything non-trivial, toggle a CLASS and keep the CSS in a stylesheet —
// it's cleaner and keeps style concerns out of JS.
```

### 16.4 Traversing the tree

From any node you can move to its relatives — useful when you have one element and need a nearby one.

```js
const li = document.querySelector("li");
li.parentElement;          // the containing element (e.g. the <ul>)
li.children;               // HTMLCollection of child ELEMENTS
li.firstElementChild;      // first child element
li.lastElementChild;
li.nextElementSibling;     // the next <li>
li.previousElementSibling;
li.closest(".list");       // nearest ANCESTOR (or self) matching a selector — very handy
li.matches(".item");       // does this element match the selector? -> boolean
// (The "...Element..." variants skip text/whitespace nodes; the older
//  parentNode/childNodes/nextSibling include text nodes too.)
```

### 16.5 Creating, inserting & removing nodes

To build UI dynamically, create elements, configure them, then attach them to the tree.

```js
// Create and configure:
const li = document.createElement("li");
li.textContent = "New task";
li.className = "item";
li.dataset.id = "7";

const list = document.querySelector("#tasks");

// Insert (modern methods accept multiple nodes/strings):
list.append(li);            // add as LAST child
list.prepend(li);           // add as FIRST child
li.before(otherNode);       // insert as a previous sibling
li.after(otherNode);        // insert as a next sibling
li.replaceWith(newNode);    // swap it out

// Remove:
li.remove();                // delete this element

// Building many nodes efficiently with a DocumentFragment (one reflow, not N):
const frag = document.createDocumentFragment();
for (const name of ["a", "b", "c"]) {
  const item = document.createElement("li");
  item.textContent = name;
  frag.append(item);
}
list.append(frag);          // single insertion -> better performance
```

> **Performance note:** each DOM mutation can trigger layout ("reflow"). Batch inserts with a `DocumentFragment`, or build an HTML string and set `innerHTML` once (for trusted content), rather than appending in a tight loop.

### 16.6 Events — reacting to the user

Events are how the page responds to clicks, typing, scrolling, page load, etc. You register a **listener** (callback) with `addEventListener(type, handler)`; the browser calls it with an **event object**.

```js
const btn = document.querySelector("#save");

btn.addEventListener("click", (event) => {
  console.log("clicked!", event.type);
  console.log(event.target);        // the element that was clicked
  console.log(event.currentTarget); // the element the listener is attached to
  event.preventDefault();           // cancel default behavior (e.g. form submit, link nav)
});

// Common event types:
//   Mouse: click, dblclick, mousedown/up, mousemove, mouseenter/leave
//   Keyboard: keydown, keyup  (event.key === "Enter", event.ctrlKey, ...)
//   Form: submit, input, change, focus, blur
//   Window/document: DOMContentLoaded, load, resize, scroll

// Keyboard example:
document.addEventListener("keydown", (e) => {
  if (e.key === "Escape") closeModal();
  if (e.ctrlKey && e.key === "s") { e.preventDefault(); save(); }
});

// Remove a listener (must be the SAME function reference):
function onScroll() { /* ... */ }
window.addEventListener("scroll", onScroll);
window.removeEventListener("scroll", onScroll);
```

**Bubbling & event delegation.** When you click an element, the event travels *up* the tree (bubbling): the target fires first, then its parent, then grandparent, etc. You can exploit this: attach **one** listener to a container and inspect `event.target` — instead of adding listeners to hundreds of children. This is **event delegation**, and it automatically covers elements added later.

```js
// Instead of a listener per <li>, one listener on the <ul>:
document.querySelector("#tasks").addEventListener("click", (e) => {
  const li = e.target.closest("li");      // find the clicked row, if any
  if (!li) return;                        // clicked empty space -> ignore
  if (e.target.matches(".delete")) {      // clicked a delete button inside the row
    li.remove();
  } else {
    li.classList.toggle("done");
  }
});

// event.stopPropagation() stops bubbling; preventDefault() stops the default action.
// The third arg controls the CAPTURE phase (top-down) and options like { once: true }:
el.addEventListener("click", handler, { once: true });   // auto-removed after first call
```

### 16.7 Forms & user input

```js
const form = document.querySelector("#signup");

form.addEventListener("submit", (e) => {
  e.preventDefault();                       // stop the browser navigating/reloading
  // Read all fields at once with FormData:
  const data = new FormData(form);
  const values = Object.fromEntries(data);  // { email: "...", password: "..." }
  console.log(values.email);

  // Built-in validation:
  if (!form.checkValidity()) {
    form.reportValidity();                  // show native error bubbles
    return;
  }
  submitToServer(values);
});

// React to typing live:
const search = document.querySelector("#q");
search.addEventListener("input", (e) => {
  console.log("current value:", e.target.value);
});
```

### 16.8 Running code at the right time

Your script may run before the DOM exists. Either put `<script>` at the end of `<body>`, use `<script defer>`/`type="module"` (deferred automatically), or wait for the event:

```js
document.addEventListener("DOMContentLoaded", () => {
  // The HTML is parsed and the DOM is ready (images may still be loading).
  init();
});
window.addEventListener("load", () => {
  // Everything (images, stylesheets) finished loading.
});
```

### 16.9 Talking to a server: `fetch`

`fetch` makes HTTP requests and returns a promise. Combine it with the DOM to load and render data.

```js
async function loadUsers() {
  try {
    const res = await fetch("/api/users", {
      method: "GET",
      headers: { "Accept": "application/json" },
    });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);   // fetch does NOT reject on 404/500
    const users = await res.json();

    const list = document.querySelector("#users");
    list.innerHTML = "";                                  // clear
    for (const u of users) {
      const li = document.createElement("li");
      li.textContent = u.name;                            // textContent = safe from XSS
      list.append(li);
    }
  } catch (err) {
    console.error("load failed:", err);
  }
}

// POST JSON:
await fetch("/api/users", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: "Ada" }),
});
```

> **Gotcha:** `fetch` only rejects on *network* failure. A 404 or 500 still *resolves* — always check `res.ok` / `res.status` yourself.

### 16.10 Browser storage

```js
// localStorage — persists across sessions; sessionStorage — cleared when the tab closes.
// Both store STRINGS only, so JSON-encode objects.
localStorage.setItem("theme", "dark");
console.log(localStorage.getItem("theme"));     // "dark"
localStorage.removeItem("theme");
localStorage.clear();

const prefs = { theme: "dark", lang: "en" };
localStorage.setItem("prefs", JSON.stringify(prefs));
const saved = JSON.parse(localStorage.getItem("prefs") ?? "{}");
// For larger/structured/offline data, use IndexedDB (a built-in async database).
```

### 16.11 Other useful browser APIs (pointers)

- **`history` / `location`** — read the URL (`location.href`, `location.search`), navigate (`location.assign`), or do SPA routing (`history.pushState`).
- **`setTimeout` / `setInterval`** — schedule work; `requestAnimationFrame` for smooth animations.
- **`navigator`** — info about the browser/device (geolocation, clipboard, online status).
- **Web APIs** — `Canvas`/WebGL (graphics), Web Audio, WebSockets (live connections), Service Workers (offline/PWA), `IntersectionObserver` (lazy-load/scroll effects).

For real apps you'll usually reach for a framework (React/Next.js, etc. — see their guides), but everything they do is built on these DOM and browser APIs.

---

## 17. Advanced & Metaprogramming

```js
// Proxy — intercept operations on an object:
const target = { a: 1 };
const proxy = new Proxy(target, {
  get(obj, prop) { return prop in obj ? obj[prop] : `<missing ${String(prop)}>`; },
  set(obj, prop, value) { console.log(`set ${String(prop)}=${value}`); obj[prop] = value; return true; },
});
console.log(proxy.a, proxy.zzz);   // 1, "<missing zzz>"
proxy.b = 2;                        // logs "set b=2"

// Reflect — the default implementations of those operations:
Reflect.has(target, "a");          // true
Reflect.ownKeys(target);

// Tagged template literals:
function html(strings, ...values) {
  return strings.reduce((acc, s, i) => acc + s + (values[i] ?? ""), "");
}
const safe = html`<p>${"hi"}</p>`;

// Symbols to customize behavior:
const range = {
  from: 1, to: 3,
  [Symbol.iterator]() {            // makes the object iterable with for...of / spread
    let n = this.from, last = this.to;
    return { next: () => n <= last ? { value: n++, done: false } : { value: undefined, done: true } };
  },
};
console.log([...range]);           // [1,2,3]

// ⚡ Explicit resource management (newest): `using` auto-disposes at scope end.
// {
//   using file = openResource();   // file[Symbol.dispose]() called automatically
// }

// Memory: objects are garbage-collected when unreachable. Watch for leaks:
// lingering event listeners, closures holding big data, growing global maps.
```

---

## 18. Tooling Overview

- **Package managers:** `npm` (default), `pnpm` (fast, disk-efficient), `yarn`, `bun`.
- **Bundlers / dev servers:** Vite (the 2026 default for apps), esbuild, Rollup, webpack (legacy).
- **TypeScript:** static types over JS — strongly recommended for non-trivial projects (`tsc`, or run TS directly with tsx/Bun/Node's type-stripping). See the TypeScript-flavored guides in this library (NestJS, Fastify, Next.js).
- **Lint/format:** ESLint + Prettier, or the all-in-one **Biome**.
- **Testing:** Vitest (Vite-native), Jest, or Node's built-in `node:test`.

---

## 19. Gotchas & Best Practices

```js
// 1) Floating point: 0.1 + 0.2 === 0.30000000000000004. Compare with a tolerance,
//    or work in integer cents for money.
console.log(Math.abs(0.1 + 0.2 - 0.3) < Number.EPSILON);   // true

// 2) NaN is not equal to itself:
console.log(Number.isNaN(NaN));    // true (use this, not === NaN)

// 3) Default sort is string-based:
console.log([10, 9, 2].sort());           // [10, 2, 9]  WRONG for numbers
console.log([10, 9, 2].sort((a, b) => a - b));  // [2, 9, 10]

// 4) Mutating while iterating, sharing references (shallow copies), and forgetting
//    that const objects are still mutable.

// 5) `this` surprises: prefer arrow functions for callbacks that need the outer this.

// 6) Always handle promise rejections (.catch / try-catch); an unhandled rejection
//    can crash a Node process.
```

**Best practices:** `const` by default; `===` always; small pure functions; prefer immutable array methods; modules over globals; handle every async error; lint + format automatically; add types (TypeScript) as projects grow.

---

## 20. Study Path & Build-to-Learn Projects

**Order:** §1–4 → §5–6 (scope/closures/this — the make-or-break concepts) → §7–11 (objects, classes, arrays, collections) → §12 (async — essential) → §13–15 → §16 (browser) → §17–19 (advanced).

**Projects:**
1. **Interactive to-do app** (browser) — DOM, events, `localStorage`, modules. Exercises §16, §13.
2. **Quiz / typing game** — timers, the event loop, state, closures. Exercises §5, §12.
3. **CLI file tool** (Node) — read/transform files, shell out to git, parse args. Exercises §14.
4. **Fetch dashboard** — `fetch`, `Promise.all`, async/await, error handling, render results. Exercises §12, §16.
5. **Tiny reactive library** — Proxy-based state with subscribers; teaches metaprogramming. Exercises §17.

**Next:** TypeScript, a framework (React/Next.js, or Node frameworks Fastify/NestJS — all in this library), and bundler/tooling fluency. For the FS/OS material in depth, see the Node.js guide; for another language's take, the Go File System / OS / CLI guide.

---

*Part of the offline developer study library. Written for ECMAScript 2024/2025 as of 2026. Confirm fast-moving APIs against MDN.*
