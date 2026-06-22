# Node.js — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** JavaScript developers who want to run JS *outside the browser* to build CLIs, servers, file/automation tools, and background workers — and to understand *why* the runtime behaves as it does, offline. You should know the JavaScript language first (see the JavaScript guide); this guide teaches **Node itself**. Every concept is explained in prose — what it is, why it works that way, when to use it — and then shown in heavily-commented code. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Node.js 22 / 24 LTS** (current in 2026). Modern features used here:
> - **Built-in test runner** `node:test` + `node:assert` (stable) — run with `node --test`. No Jest needed for most projects.
> - **Native `--watch`** (restart on file change) and **`--env-file=.env`** (load env vars without the `dotenv` package).
> - **Native `fetch`**, `Blob`, `structuredClone`, `AbortController`, `URL` are all global — no imports.
> - **`node:sqlite`** — a built-in SQLite module (stabilizing) for zero-dependency local storage.
> - **`util.parseArgs`** for argument parsing; a **permission model** (`--permission` + `--allow-fs-read`) for sandboxing untrusted code.
> - ESM is first-class; most callback APIs have promise versions under `node:fs/promises`, `node:timers/promises`, etc.
>
> Always import core modules with the **`node:` prefix** (`import fs from "node:fs"`) — it's explicit and can never be shadowed by an npm package of the same name. The author is on **Windows 11**, so path and process-spawning specifics are flagged. Fast-moving details are marked **⚡ Version note**. Confirm exact APIs at nodejs.org/api.

---

## Table of Contents

1. [What Node Is & Running It](#1-what-node-is--running-it) **[B]**
2. [Modules: CommonJS vs ESM](#2-modules-commonjs-vs-esm) **[B/I]**
3. [npm & Packages](#3-npm--packages) **[B/I]**
4. [The Event Loop & Async Model](#4-the-event-loop--async-model) **[I/A]**
5. [File System](#5-file-system) **[I]**
6. [Paths & OS / Process Info](#6-paths--os--process-info) **[I]**
7. [Executing External Commands](#7-executing-external-commands) **[I]**
8. [Streams](#8-streams) **[I/A]**
9. [Events (EventEmitter)](#9-events-eventemitter) **[I]**
10. [Other Core Modules](#10-other-core-modules) **[I]**
11. [Building CLIs](#11-building-clis) **[I]**
12. [Concurrency & Scaling](#12-concurrency--scaling) **[A]**
13. [Error Handling & Graceful Shutdown](#13-error-handling--graceful-shutdown) **[I/A]**
14. [Networking: HTTP & fetch](#14-networking-http--fetch) **[I]**
15. [Testing & Debugging](#15-testing--debugging) **[I]**
16. [Data & Persistence](#16-data--persistence) **[I]**
17. [Performance & Production](#17-performance--production) **[A]**
18. [Project Structure & Best Practices](#18-project-structure--best-practices)
19. [Study Path & Build-to-Learn Projects](#19-study-path--build-to-learn-projects)

---

## 1. What Node Is & Running It

**What it is.** Node.js is a *runtime* — a program that executes JavaScript on your machine instead of inside a web page. It bundles three things: **V8** (Google's JavaScript engine, the same one in Chrome, which compiles your JS to machine code), **libuv** (a C library that provides the *event loop* and asynchronous, non-blocking I/O across operating systems), and a **standard library** of modules (`fs`, `http`, `crypto`, …) that expose the filesystem, network, processes, and OS to JavaScript.

**Why it exists / why it matters.** Browsers deliberately sandbox JS — it can't read your disk or open arbitrary network sockets. Node removes that sandbox so you can write servers, build tools, and scripts in the same language as the front end. Its defining design choice is being **non-blocking and event-driven**: instead of dedicating one thread per connection (the traditional server model), a single thread juggles thousands of in-flight operations by reacting to events as I/O completes. That makes Node excellent for I/O-bound work (APIs, proxies, real-time apps) on modest hardware.

**How you run it.**

```bash
node --version                 # check the installed version
node app.js                    # run a script file
node --watch app.js            # re-run automatically when the file changes (built in)
node --env-file=.env app.js    # load KEY=value pairs from .env into process.env (built in)
node                           # start the REPL (interactive prompt) — great for experiments
node -e "console.log(process.platform)"   # evaluate a one-liner and exit
```

**Managing versions.** Projects often pin a specific Node version. A *version manager* lets you install and switch between them per project: **fnm** (fast, Rust-based), **nvm**, or **volta**. On Windows: `winget install Schniz.fnm`, then `fnm install 22 && fnm use 22`. Declaring `"engines": { "node": ">=22" }` in `package.json` documents the requirement.

---

## 2. Modules: CommonJS vs ESM

**What modules are and why.** A module is a single file with its own scope. Without modules, every `var` would leak into one giant global namespace and collide. Modules let you split a program into files that *explicitly* export some values and import others — making dependencies visible and code reusable. Node supports two module systems, and knowing which you're in matters because the syntax and loading rules differ.

**ESM (ECMAScript Modules)** is the modern, standardized system (the same `import`/`export` browsers use). **CommonJS (CJS)** is Node's original system (`require`/`module.exports`), still everywhere in the ecosystem. New projects should prefer ESM.

```js
// ----- ESM (modern, recommended) -----
// Turn it on with "type": "module" in package.json, OR name the file .mjs.
import { readFile } from "node:fs/promises";   // named import
import express from "express";                  // default import
export function hello() { return "hi"; }        // named export
export default class App {}                      // default export
```

```js
// ----- CommonJS (classic Node) -----
// The default when "type" isn't "module"; or name the file .cjs.
const { readFile } = require("node:fs/promises");
function hello() { return "hi"; }
module.exports = { hello };       // export an object
module.exports.default = App;     // attach more
```

**How the two differ (and why it bites people).** CJS loads modules **synchronously** at the moment `require()` runs, and `module.exports` is a plain mutable object resolved at runtime — so you can `require` conditionally inside a function. ESM is **asynchronous and static**: imports are hoisted and resolved *before* the module body runs, which enables tree-shaking and top-level `await` but means you can't `import` conditionally (use dynamic `import()` for that). You also can't use `require` in an ESM file, and `__dirname`/`__filename` don't exist in ESM.

| | CommonJS | ESM |
|---|---|---|
| Import | `require()` | `import` |
| Export | `module.exports` | `export` |
| Loading | synchronous, runtime | asynchronous, static (hoisted) |
| Conditional import | `if (x) require(...)` | `await import(...)` |
| `__dirname`/`__filename` | available | use `import.meta.dirname`/`.filename` |
| Top-level `await` | ❌ | ✅ |

```js
// Getting the current file/directory:
// CJS: __dirname and __filename are provided for free.
// ESM (Node 20.11+):
console.log(import.meta.dirname);    // the directory this file lives in
console.log(import.meta.filename);   // this file's full path
console.log(import.meta.url);        // file:// URL form
// Older ESM pattern (pre-20.11), still common:
import { fileURLToPath } from "node:url";
import { dirname } from "node:path";
const __dirname = dirname(fileURLToPath(import.meta.url));

// Dynamic import — works in BOTH systems, returns a promise. Use it to load a
// module lazily, conditionally, or by a computed path:
const { default: config } = await import(`./config.${process.env.NODE_ENV}.js`);
```

---

## 3. npm & Packages

**What npm is.** npm is two things: the **registry** (a huge public store of reusable packages) and the **CLI** that installs them and runs project scripts. A package is a folder with a `package.json` describing it.

### 3.1 package.json — the project manifest **[B]**

`package.json` records your project's metadata, dependencies, and scripts. It's the file every Node tool reads first.

```json
{
  "name": "myapp",
  "version": "1.0.0",
  "type": "module",                       // make .js files ESM
  "engines": { "node": ">=22" },          // documents the required Node version
  "scripts": {
    "dev": "node --watch src/index.js",   // `npm run dev`
    "start": "node src/index.js",
    "test": "node --test"
  },
  "dependencies": { "express": "^5.0.0" },     // needed at runtime
  "devDependencies": { "typescript": "^5.6.0" },// only needed to build/test
  "exports": { ".": "./src/index.js" }          // the public entry point
}
```

### 3.2 Installing & running **[B]**

```bash
npm init -y                  # create a package.json with defaults
npm install express          # add a runtime dependency (updates package-lock.json)
npm install -D typescript     # add a dev dependency
npm install -g some-cli       # install a CLI tool globally
npm ci                        # install EXACTLY from the lockfile (use in CI — reproducible)
npm run dev                   # run a script defined in package.json
npx cowsay hi                 # download+run a package binary once, without installing
npm outdated / npm update     # see/update outdated deps
npm uninstall express
```

**Semantic versioning (semver) — why the `^` matters.** Versions are `MAJOR.MINOR.PATCH`. The range prefix controls how much npm may upgrade:
- `^1.2.3` → allows `>=1.2.3 <2.0.0` (new features/fixes, no breaking changes). The common default.
- `~1.2.3` → allows `>=1.2.3 <1.3.0` (patches only). Conservative.
- `1.2.3` → exact, no upgrades.

The **lockfile** (`package-lock.json`) records the *exact* resolved version of every dependency (and their dependencies). Commit it so everyone — and CI — installs identical trees. Use `npm ci` (not `npm install`) in automated builds because it installs strictly from the lockfile and fails if they disagree.

**Alternatives:** **pnpm** (fast, saves disk by hard-linking a global content-addressed store), **yarn**, and **bun**. For multi-package repos, npm/pnpm **workspaces** (`"workspaces": ["packages/*"]`) let several packages share one install.

---

## 4. The Event Loop & Async Model

This is the single most important thing to understand about Node. **[I/A]**

**The core idea.** Your JavaScript runs on **one thread**. When you ask for something slow — read a file, query a database, fetch a URL — Node doesn't sit and wait. It hands the operation to libuv (which uses the OS or a small background thread pool), registers a **callback**, and immediately moves on. When the operation finishes, its callback is queued, and the **event loop** runs it when the thread is free. This is how one thread serves thousands of concurrent connections: it's almost never *waiting*, only *reacting*.

**The phases.** The event loop cycles through phases, each with its own callback queue:

1. **timers** — `setTimeout` / `setInterval` callbacks whose time has elapsed
2. **pending callbacks** — certain deferred system callbacks
3. **poll** — wait for and process I/O events (file/network reads completing) — this is where the loop spends most idle time
4. **check** — `setImmediate` callbacks
5. **close** — `close` event callbacks (e.g. a socket closing)

**The crucial detail: microtasks.** Between *every* callback, Node drains two higher-priority queues completely: first the **`process.nextTick`** queue, then the **promise microtask** queue (resolved `.then`/`await` continuations, `queueMicrotask`). Microtasks always run before the loop moves to the next timer or I/O callback. This ordering explains output that surprises beginners:

```js
console.log("1: sync");
setTimeout(() => console.log("5: timeout (macrotask)"), 0);
setImmediate(() => console.log("6: immediate (check phase)"));
Promise.resolve().then(() => console.log("4: promise (microtask)"));
process.nextTick(() => console.log("3: nextTick (highest priority)"));
console.log("2: sync");
// Order: 1, 2 (all sync first), then 3 (nextTick), 4 (promise microtask),
// then 5/6 (the macrotask phases). nextTick beats promises; both beat timers.
```

**Why you must never block the loop.** Because everything shares one thread, a long *synchronous* computation (a tight CPU loop, `JSON.parse` of a 200 MB string, `fs.readFileSync` of a huge file) freezes the entire process — no requests served, no timers fire, until it finishes. The rules: prefer the **async** version of every API, keep request handlers fast, and push heavy CPU work to **worker threads** (§12).

```js
// Turn an old callback-style API into a promise you can await:
import { promisify } from "node:util";
import { exec } from "node:child_process";
const execAsync = promisify(exec);
const { stdout } = await execAsync("git --version");

// Most core modules already offer promise variants — prefer them:
import { readFile } from "node:fs/promises";   // instead of fs.readFile(cb)
```

---

## 5. File System

**What it is.** `node:fs` reads and writes files and directories. It comes in **three flavors**, and choosing the right one matters:
- **Promise API** (`node:fs/promises`) — `await`-able, the default for application code.
- **Callback API** (`fs.readFile(path, cb)`) — the original; fine but leads to nesting.
- **Sync API** (`fs.readFileSync`) — blocks the event loop; only acceptable at startup, in CLI scripts, or build tools where nothing else is running.

```js
import { readFile, writeFile, appendFile, mkdir, readdir, rm, stat,
         rename, copyFile, access } from "node:fs/promises";
import { constants } from "node:fs";

// --- Reading & writing text ---
// Pass an encoding to get a string back; omit it to get a raw Buffer (binary).
const text = await readFile("config.json", "utf8");
await writeFile("out.txt", "hello\n", "utf8");          // creates or TRUNCATES (overwrites)
await appendFile("app.log", "another line\n", "utf8");  // adds to the end

// --- Directories ---
await mkdir("data/reports", { recursive: true });       // like `mkdir -p`: makes parents too
const entries = await readdir("data", { withFileTypes: true });  // Dirent objects
for (const e of entries) {
  console.log(e.name, e.isDirectory() ? "(dir)" : "(file)");
}
const everything = await readdir("src", { recursive: true });    // deep listing (Node 20+)

// --- Delete / move / copy ---
await rm("tmp", { recursive: true, force: true });      // delete a tree; force = no error if missing
await rename("old.txt", "new.txt");                     // move/rename
await copyFile("a.txt", "b.txt");

// --- Existence & metadata ---
// Prefer "try the op and handle failure" over checking-then-acting (avoids race conditions):
try {
  await access("file.txt", constants.R_OK);             // throws if not readable
  console.log("readable");
} catch {
  console.log("missing or no permission");
}
const info = await stat("file.txt");
console.log(info.size, info.isFile(), info.mtime);      // bytes, type, modified time
```

**Watching for changes** (rebuild-on-save, live tooling):

```js
import { watch } from "node:fs";
watch("src", { recursive: true }, (eventType, filename) => {
  console.log(`${eventType} -> ${filename}`);   // 'change' | 'rename'
});
```

> **When to use streams instead:** the calls above load the *whole* file into memory. For large files (logs, video, big CSV) that's wasteful or impossible — use **streams** (§8) to process data in chunks.

---

## 6. Paths & OS / Process Info

**Why `node:path` exists.** File paths differ across operating systems — Windows uses `\` and drive letters, Unix uses `/`. Building paths by string concatenation breaks cross-platform. `node:path` joins and parses paths correctly for whatever OS you're on.

```js
import path from "node:path";

console.log(path.join("data", "reports", "2026.csv"));  // data\reports\2026.csv on Windows
console.log(path.resolve("file.txt"));                  // absolute path from the cwd
console.log(path.basename("/a/b.txt"));                 // "b.txt"
console.log(path.dirname("/a/b.txt"));                  // "/a"
console.log(path.extname("archive.tar.gz"));            // ".gz"
console.log(path.parse("/a/b.txt"));                    // { root, dir, base, name, ext }
console.log(path.sep);                                  // "\\" on Windows, "/" elsewhere
// Force a style regardless of platform with path.posix / path.win32.
```

**`node:os` — facts about the machine.** Use it for diagnostics, choosing sensible defaults (e.g. a worker count), or locating user directories.

```js
import os from "node:os";
console.log(os.platform());        // 'win32' | 'linux' | 'darwin'
console.log(os.arch());            // 'x64' | 'arm64'
console.log(os.cpus().length);     // logical CPU count — size your worker pool from this
console.log(os.totalmem(), os.freemem());
console.log(os.hostname(), os.homedir(), os.tmpdir());
console.log(os.userInfo().username);
```

**`process` — the running program itself.** A global object (no import) describing and controlling the current process: its arguments, environment, working directory, and lifecycle.

```js
console.log(process.pid);            // process id
console.log(process.platform);       // same as os.platform()
console.log(process.version);        // Node version, e.g. 'v22.4.0'
console.log(process.cwd());          // current working directory
process.chdir("..");                 // change it
console.log(process.argv);           // ['node', '/abs/script.js', ...your args]
console.log(process.env.PATH);       // an environment variable
process.env.MY_FLAG = "1";           // set one (visible to this process and children)
process.stdout.write("no newline");  // write raw to stdout (console.log adds \n)
process.exitCode = 1;                // set the exit code; let the program finish naturally
```

> **Tip:** prefer setting `process.exitCode` over calling `process.exit()`. `exit()` terminates *immediately* and can cut off pending writes (e.g. a half-flushed log). Setting `exitCode` lets the event loop drain first.

---

## 7. Executing External Commands

**What & why.** Real tools often shell out to other programs — run `git`, kick off `npm install`, call a Python script, invoke `ffmpeg`. `node:child_process` spawns those programs as separate OS processes and lets you feed them input, capture their output, and read their exit code. There are four functions; picking the right one is the whole skill:

| Function | What it does | Use when |
|---|---|---|
| `execFile(file, args)` | runs a binary with an args array; **no shell**; buffers all output | you want a command's output and it's a known program (safest) |
| `spawn(cmd, args)` | streams output as it arrives; **no shell** by default | long-running commands, or large/continuous output (builds, logs) |
| `exec(cmdString)` | runs a string **through a shell** | you need shell features (pipes, `&&`) on a *trusted, fixed* string |
| `fork(modulePath)` | spawns another **Node** process with a message channel | parallel Node work that needs to talk back |

**Why "no shell" is safer.** `exec` passes your string to `/bin/sh` or `cmd.exe`, which interprets metacharacters (`;`, `|`, `$()`). If any part of that string came from user input, an attacker can inject commands — **command injection**. `execFile`/`spawn` take the program and an **array of arguments** that are passed directly to the OS, so arguments can never be reinterpreted as new commands. Default to these.

```js
import { execFile, spawn } from "node:child_process";
import { promisify } from "node:util";
const execFileP = promisify(execFile);

// --- Capture output safely (args array, no shell) ---
const { stdout } = await execFileP("git", ["rev-parse", "--abbrev-ref", "HEAD"]);
console.log("branch:", stdout.trim());

// --- Run real tools with a working directory and custom env ---
await execFileP("npm", ["install"], { cwd: "frontend" });
await execFileP("python", ["build.py", "--release"],
                { env: { ...process.env, MODE: "prod" } });

// Why set { cwd } instead of running `cd frontend && npm install`?
// Each spawned command is its own process; a `cd` in one would NOT affect the next.
// Pass the working directory explicitly with the cwd option.

// --- Stream output live (better for builds / long commands) ---
const isWin = process.platform === "win32";
const child = spawn(isWin ? "npm.cmd" : "npm", ["run", "build"],
                    { stdio: "inherit" });   // pipe child's stdio straight to ours
child.on("close", (code) => console.log("build exited with", code));

// --- DANGER: never interpolate untrusted input into exec's shell string ---
// exec(`rm -rf ${userInput}`)   // an attacker passes "x; rm -rf /"
// Use execFile/spawn with an args array instead.
```

> **⚡ Windows note:** `npm`, `npx`, and many CLIs are `.cmd` batch shims, not real `.exe` files. `execFile("npm", ...)` can fail with `ENOENT`. Fixes: call `"npm.cmd"` explicitly, pass `{ shell: true }`, or use `spawn(..., { shell: true })`. On Unix the bare name works.

---

## 8. Streams

**The problem streams solve.** Reading a 5 GB file with `readFile` tries to hold all 5 GB in memory at once — it'll be slow or crash. A **stream** processes data in small **chunks** as it flows, so memory stays flat no matter the size. Streams are everywhere in Node: files, network sockets, HTTP requests/responses, compression. **[I/A]**

**The four types.** **Readable** (you read from it — a file being read, an incoming request), **Writable** (you write to it — a file being written, an outgoing response), **Duplex** (both — a TCP socket), and **Transform** (a duplex that modifies data passing through — gzip, encryption).

**Backpressure — the key concept.** If a fast source pours data into a slow destination (e.g. reading from a fast SSD, writing over a slow network), the destination's buffer fills up. Backpressure is the mechanism that tells the source to *pause* until the destination catches up, preventing unbounded memory growth. You almost never manage this by hand — `pipeline()` does it for you, which is exactly why you should use it instead of manual `.pipe()` chains (which don't propagate errors cleanly).

```js
import { createReadStream, createWriteStream } from "node:fs";
import { pipeline } from "node:stream/promises";
import { createGzip } from "node:zlib";

// Read -> compress -> write, with backpressure and error handling done for you:
await pipeline(
  createReadStream("big.log"),     // Readable source
  createGzip(),                    // Transform (gzip each chunk)
  createWriteStream("big.log.gz"), // Writable destination
);
console.log("compressed");

// Consume a readable stream chunk-by-chunk with async iteration:
for await (const chunk of createReadStream("data.txt", { encoding: "utf8" })) {
  process.stdout.write(chunk);     // handle one piece at a time; memory stays flat
}

// A custom Transform that upper-cases everything flowing through it:
import { Transform } from "node:stream";
const upper = new Transform({
  transform(chunk, _encoding, callback) {
    callback(null, chunk.toString().toUpperCase());   // (error, transformedChunk)
  },
});
await pipeline(createReadStream("in.txt"), upper, createWriteStream("out.txt"));
```

---

## 9. Events (EventEmitter)

**What it is and why.** Much of Node is built on an **event** pattern: an object emits named events, and any number of listeners react. It's the in-process version of pub/sub and underpins streams, servers, and sockets. You'll both *consume* emitters (`server.on("request", ...)`) and *create* your own when modeling something that produces a stream of happenings over time.

```js
import { EventEmitter } from "node:events";

class Job extends EventEmitter {
  run() {
    this.emit("start");                                   // notify listeners
    setTimeout(() => this.emit("done", { ok: true }), 100);
  }
}

const job = new Job();
job.on("start", () => console.log("started"));            // every time
job.once("done", (result) => console.log("done", result));// only the first time
job.on("error", (err) => console.error("job error:", err));// ALWAYS handle 'error'
job.run();
```

> **Critical gotcha:** an emitted `"error"` event with **no** listener is special — Node *throws* it, crashing the process. Always attach an `error` listener to emitters that can fail (streams, servers, child processes). Also, Node warns when an emitter has more than 10 listeners for one event (a common leak signal); raise it deliberately with `emitter.setMaxListeners(n)` if you really need more.

---

## 10. Other Core Modules

Node's standard library is large; these are the ones you reach for constantly. **[I]**

```js
// --- crypto: hashing, HMAC, random values, key derivation ---
import { createHash, randomUUID, randomBytes, scryptSync, timingSafeEqual } from "node:crypto";
console.log(createHash("sha256").update("hello").digest("hex"));  // content hash
console.log(randomUUID());                       // a random v4 UUID (ids)
console.log(randomBytes(16).toString("hex"));    // a secure random token (CSRF, sessions)
const key = scryptSync("password", "salt", 32);  // derive a key from a password (slow on purpose)
// Use timingSafeEqual to compare secrets/tokens to avoid timing attacks.

// --- buffer: raw binary data (files, network, encodings) ---
const buf = Buffer.from("hello", "utf8");
console.log(buf.toString("base64"));             // "aGVsbG8="
console.log(buf.length, buf[0]);

// --- url: parse and build URLs safely ---
const u = new URL("https://example.com/path?q=node&page=2");
console.log(u.hostname);                          // "example.com"
console.log(u.searchParams.get("q"));             // "node"
u.searchParams.set("page", "3");                  // mutate query params

// --- util: helpers (promisify, parseArgs, deep inspection) ---
import { promisify, inspect } from "node:util";
console.log(inspect({ a: { b: { c: 1 } } }, { depth: null, colors: true }));

// --- timers/promises: delays you can await without blocking ---
import { setTimeout as sleep } from "node:timers/promises";
await sleep(500);                                 // pause 500ms, non-blocking

// --- readline/promises: interactive prompts ---
import readline from "node:readline/promises";
const rl = readline.createInterface({ input: process.stdin, output: process.stdout });
// const name = await rl.question("Your name? "); rl.close();
```

---

## 11. Building CLIs

**Why Node for CLIs.** Node makes excellent command-line tools: fast startup, the npm ecosystem, and easy distribution (`npm install -g`). The skill is parsing arguments, reading input, and returning a correct exit code. Node now ships `util.parseArgs`, so simple CLIs need no dependency.

```js
#!/usr/bin/env node
// ^ the "shebang": on Unix it lets the file run directly as ./cli.js.
import { parseArgs } from "node:util";

const { values, positionals } = parseArgs({
  allowPositionals: true,
  options: {
    output:  { type: "string",  short: "o", default: "dist" },  // --output / -o <dir>
    verbose: { type: "boolean", short: "v", default: false },   // --verbose / -v
    help:    { type: "boolean", short: "h" },
  },
});

if (values.help) {
  console.log("usage: build [files...] -o <dir> [-v]");
  process.exit(0);                  // 0 = success
}
if (values.verbose) console.log("building", positionals, "->", values.output);

// Reading piped stdin (e.g.  cat data.txt | node tool.js):
if (!process.stdin.isTTY) {        // isTTY is false when input is piped/redirected
  let input = "";
  for await (const chunk of process.stdin) input += chunk;
  console.log("received", input.length, "chars on stdin");
}

process.exit(0);
```

To ship a command name, add `"bin": { "mytool": "./cli.js" }` to `package.json`; then `npm install -g .` creates a global `mytool`. For larger CLIs reach for **commander** or **yargs** (subcommands, help generation); for nice output, **chalk** (colors), **ora** (spinners), **@clack/prompts** (interactive questions).

---

## 12. Concurrency & Scaling

**The situation.** Your JS is single-threaded, so a single Node process uses one CPU core for JS execution. To use more cores — for CPU-bound work or to scale a server — you create more execution units. Three tools, three purposes: **[A]**

| Tool | What it is | Use for |
|---|---|---|
| `worker_threads` | real threads in the *same* process, sharing memory via `SharedArrayBuffer` | CPU-bound work (parsing, image processing, crypto) without freezing the main loop |
| `cluster` | several Node *processes* sharing one server port | scaling an HTTP server across all cores |
| `child_process.fork` | a separate Node process with a message channel | isolating risky work, or coarse parallel jobs |

```js
// --- worker_threads: offload CPU work so the event loop stays responsive ---
import { Worker, isMainThread, parentPort, workerData } from "node:worker_threads";
import { fileURLToPath } from "node:url";

if (isMainThread) {
  // Main thread: spawn a worker and receive its result asynchronously.
  const worker = new Worker(fileURLToPath(import.meta.url), { workerData: 42 });
  worker.on("message", (result) => console.log("fib =", result));
  worker.on("error", console.error);
} else {
  // Worker thread: do the heavy computation, then post the answer back.
  const fib = (n) => (n < 2 ? n : fib(n - 1) + fib(n - 2));
  parentPort.postMessage(fib(workerData));
}
```

```js
// --- cluster: one worker process per CPU, all sharing port 3000 ---
import cluster from "node:cluster";
import os from "node:os";
import http from "node:http";

if (cluster.isPrimary) {
  for (let i = 0; i < os.cpus().length; i++) cluster.fork();   // fork N workers
  cluster.on("exit", (worker) => cluster.fork());              // restart crashed ones
} else {
  http.createServer((req, res) => res.end(`handled by ${process.pid}`)).listen(3000);
}
// In production you'd usually let a process manager (PM2) or your container
// orchestrator (Kubernetes) handle multi-instance scaling instead.
```

---

## 13. Error Handling & Graceful Shutdown

**The model.** Synchronous errors are caught with `try/catch`. Asynchronous errors come in two shapes: rejected **promises** (caught by `try/catch` around `await`) and error-first **callbacks** (`(err, data) => ...`). The danger is *unhandled* async errors — an unhandled promise rejection or an emitter `error` with no listener will, by default, crash the process. **[I/A]**

```js
// Async errors -> wrap awaits in try/catch:
async function main() {
  try {
    await doWork();
  } catch (err) {
    console.error("failed:", err);
    process.exitCode = 1;          // signal failure, let the process drain
  }
}

// Last-resort safety nets: log and exit. Don't try to "keep going" — the process
// is now in an unknown state. Let a supervisor restart it cleanly.
process.on("uncaughtException", (err) => { console.error("uncaught:", err); process.exit(1); });
process.on("unhandledRejection", (reason) => { console.error("unhandled:", reason); process.exit(1); });
```

**Graceful shutdown — why it matters.** When your container is stopped (`docker stop`, Kubernetes rollout), the OS sends **SIGTERM**, then kills the process after a grace period. If you exit instantly you may drop in-flight requests or leave DB connections dangling. Graceful shutdown means: stop accepting new work, finish what's in progress, close resources, then exit.

```js
function shutdown(signal) {
  console.log(`\n${signal} received — shutting down gracefully...`);
  server.close(() => {            // stop accepting new connections; finish active ones
    db.end();                     // close the database pool
    process.exit(0);
  });
  // Safety: if cleanup hangs, force-exit after 10s. .unref() so this timer
  // doesn't itself keep the process alive.
  setTimeout(() => process.exit(1), 10_000).unref();
}
process.on("SIGINT", () => shutdown("SIGINT"));    // Ctrl-C in a terminal
process.on("SIGTERM", () => shutdown("SIGTERM"));  // container/orchestrator stop

// AbortController — cancel async work / add timeouts (works with fetch, fs, timers):
const ac = new AbortController();
const timer = setTimeout(() => ac.abort(), 5000);  // abort after 5s
try {
  const res = await fetch("https://example.com", { signal: ac.signal });
} finally {
  clearTimeout(timer);
}
```

---

## 14. Networking: HTTP & fetch

**Outbound requests.** Node has a global `fetch` (the same API as the browser) — no library needed. Pair it with the DOM-free logic of a server or CLI to call other APIs.

```js
const res = await fetch("https://api.example.com/users", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: "Ada" }),
});
// IMPORTANT: fetch only rejects on a NETWORK failure. A 404 or 500 still resolves —
// you must check res.ok / res.status yourself.
if (!res.ok) throw new Error(`HTTP ${res.status}`);
const data = await res.json();
```

**A minimal server.** The core `http` module is enough to understand how servers work; for real apps use a framework (Fastify, NestJS, Express — each has its own guide), which adds routing, validation, and middleware on top of exactly this.

```js
import http from "node:http";

const server = http.createServer((req, res) => {
  // req: the incoming request (method, url, headers, a readable body stream)
  // res: the writable response
  if (req.method === "GET" && req.url === "/health") {
    res.writeHead(200, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ ok: true }));
  } else {
    res.writeHead(404).end("not found");
  }
});

server.listen(3000, () => console.log("listening on http://localhost:3000"));
```

---

## 15. Testing & Debugging

**Built-in testing.** Node ships a full test runner (`node:test`) and assertion library (`node:assert`) — no Jest/Mocha install required for most projects. Tests live in `*.test.js` files or a `test/` folder.

```js
// file: math.test.js   ->   run with:  node --test
import { test, describe, before, mock } from "node:test";
import assert from "node:assert/strict";          // "strict" = uses === semantics

describe("math", () => {
  test("adds numbers", () => {
    assert.equal(2 + 3, 5);
  });

  test("handles async", async () => {
    const value = await Promise.resolve(42);
    assert.equal(value, 42);
  });

  test("can mock functions", () => {
    const fn = mock.fn((x) => x * 2);
    fn(21);
    assert.equal(fn.mock.calls.length, 1);
    assert.equal(fn.mock.calls[0].result, 42);
  });
});
```

```bash
node --test                                  # run all tests
node --test --watch                          # re-run on change
node --test --experimental-test-coverage     # coverage report
node --inspect app.js                         # start the debugger; attach VS Code or chrome://inspect
```

**Debugging tips.** `node --inspect` opens a debugger you can attach from VS Code (set breakpoints, step through). For quick tracing, `console.log` works, but `util.debuglog("myapp")` gives namespaced logs you enable via `NODE_DEBUG=myapp`. In production, use a structured logger like **pino** (fast JSON logs) rather than `console.log`. Alternatives to the built-in runner: **Vitest**, **Jest**.

---

## 16. Data & Persistence

**Built-in SQLite.** Node now includes a SQLite module — a real SQL database in a single file, zero setup, perfect for CLIs, desktop apps, and small services. (See the SQLite3 guide for depth.)

```js
import { DatabaseSync } from "node:sqlite";       // ⚡ Version note: stabilizing in 22.5+
const db = new DatabaseSync("app.db");
db.exec("CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, name TEXT)");
const insert = db.prepare("INSERT INTO users (name) VALUES (?)");  // prepared statement
insert.run("Ada");                                                 // ? prevents SQL injection
console.log(db.prepare("SELECT * FROM users").all());
```

```js
// JSON config files and env-based config:
import { readFile } from "node:fs/promises";
const config = JSON.parse(await readFile("config.json", "utf8"));
// Or run with:  node --env-file=.env app.js   then read process.env.DATABASE_URL
```

For full databases see the **PostgreSQL**, **MongoDB**, **SQLite3**, **Prisma**, and **Redis** guides in this library.

---

## 17. Performance & Production

- **Profile before optimizing.** Guessing wastes time. Use `node --prof app.js` then `node --prof-process`, the `--cpu-prof`/`--heap-prof` flags, or Chrome DevTools via `--inspect`. Measure, find the real hot path, then fix it.
- **Never block the event loop.** The #1 production issue. Move CPU work to **worker threads** (§12); stream large I/O (§8); avoid huge synchronous `JSON.parse`/`readFileSync` in request paths.
- **Hunt memory leaks.** Common causes: ever-growing global `Map`/array caches, timers never cleared, event listeners added but never removed, closures capturing large objects. Take heap snapshots (DevTools) and compare them over time to find what keeps growing.
- **Security.** Never `eval`/`exec` untrusted input; validate every external input; keep dependencies patched (`npm audit`); consider the **permission model** (`node --permission --allow-fs-read=./data app.js`) to restrict what untrusted code can touch.
- **Secrets & config** belong in environment variables or a secrets manager — never hard-coded or committed. Validate required config at startup and fail fast if it's missing.
- **Run it properly.** Use a process manager (**PM2**, systemd) or a container (see the Docker guide), behind a reverse proxy; emit structured JSON logs; expose a `/health` endpoint; and handle SIGTERM (§13) for zero-downtime deploys.

---

## 18. Project Structure & Best Practices

```
myapp/
├── package.json
├── package-lock.json     (commit it)
├── .env                  (gitignored — secrets/config)
├── .gitignore            (node_modules, .env, dist…)
├── src/
│   ├── index.js          (entry / bootstrap)
│   ├── config.js         (read & validate env at startup)
│   ├── routes/           (HTTP handlers)
│   ├── services/         (business logic)
│   └── lib/              (shared helpers)
└── test/                 (*.test.js)
```

**Best practices, distilled:** prefer **ESM** with `node:` prefixes; use **async/await** over raw callbacks; handle *every* promise rejection; validate environment/config at startup and fail fast; keep modules small with clear exports; use `npm ci` in CI; pin Node via `engines` plus a version manager; and let **ESLint + Prettier** (or **Biome**) handle linting/formatting automatically.

---

## 19. Study Path & Build-to-Learn Projects

**Order to learn in:** §1–3 (run code, modules, npm) → **§4 the event loop** (build the mental model first — everything else makes sense after) → §5–7 (filesystem, paths/OS, executing commands — immediately useful) → §8–9 (streams, events) → §11 (CLIs) → §13–14 (errors, HTTP) → then §12, §15–17 (scaling, testing, production) as you need them.

**Projects that cement the concepts:**
1. **A CLI tool** — read/transform files, parse args with `parseArgs`, shell out to `git`. Exercises §5, §7, §11.
2. **A log processor** — stream a multi-GB file through a transform and gzip the result. Exercises §8 and proves you understand backpressure.
3. **A dev task runner** — watch a directory and run a build command on every change. Exercises §5, §7, §9.
4. **A REST API** — start with core `http`, then rebuild it with **Fastify** or **NestJS** (their own guides) and add a database. Exercises §14.
5. **A worker pool** — distribute CPU-bound jobs across `worker_threads` and compare timings to the single-threaded version. Exercises §12.

**Where to go next:** a web framework (Fastify / NestJS / Express), a database layer (Prisma / `pg` / MongoDB), and Docker for deployment — all covered elsewhere in this library. For the language fundamentals underneath Node, see the JavaScript guide; for the same systems concepts in another language, the Go File System / OS / CLI guide.

---

*Part of the offline developer study library. Written for Node.js 22/24 LTS as of 2026. Confirm fast-moving APIs against nodejs.org/api.*
