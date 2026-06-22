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
16. [Browser Basics](#16-browser-basics) **[B/I]**
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

A closure is a function that remembers the variables from where it was *defined*.

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

`this` depends on **how a function is called**, not where it's defined. **[I]**

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

Every object has an internal link to a **prototype** object; property lookups walk the chain.

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

JS is single-threaded. Long work is offloaded (timers, I/O, fetch); when done, callbacks queue up. The loop runs the **call stack** to empty, drains the **microtask queue** (promises) fully, then takes one **macrotask** (timer/IO), and repeats.

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

## 16. Browser Basics

```js
// Select & manipulate the DOM:
const btn = document.querySelector("#save");
const items = document.querySelectorAll(".item");   // NodeList
btn.textContent = "Save";
btn.classList.add("active");
btn.setAttribute("disabled", "");

// Events:
btn.addEventListener("click", (event) => {
  event.preventDefault();
  console.log("clicked", event.target);
});

// Create & insert:
const li = document.createElement("li");
li.textContent = "new";
document.querySelector("ul")?.append(li);

// fetch (returns a promise):
const res = await fetch("/api/data", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ a: 1 }),
});
const data = await res.json();

// Storage (string only — JSON-encode objects):
localStorage.setItem("token", "abc");
console.log(localStorage.getItem("token"));
localStorage.removeItem("token");
```

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
