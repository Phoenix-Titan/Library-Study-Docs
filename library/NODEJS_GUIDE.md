# Node.js — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** JavaScript developers who want to run JS *outside the browser* — build CLIs, servers, file/automation tools, and background workers — and understand the runtime deeply, offline. Assumes you know the JavaScript language (see the JavaScript guide); this guide is about **Node itself**. Sections are tagged **[B]/[I]/[A]**.
>
> **Version note:** This guide targets **Node.js 22 / 24 LTS** (current in 2026). Modern Node features used here:
> - **Built-in test runner** `node:test` + `node:assert` (stable), run with `node --test`.
> - **Native `--watch`** (restart on change) and **`--env-file=.env`** (no dotenv needed).
> - **Native `fetch`**, `Blob`, `structuredClone`, `AbortController` are global.
> - **`node:sqlite`** — a built-in SQLite module (stabilizing).
> - **`util.parseArgs`** for CLIs; **permission model** (`--permission`/`--allow-fs-read`) for sandboxing.
> - ESM is first-class; many APIs have promise versions under `node:fs/promises` etc.
>
> Always import core modules with the `node:` prefix (`node:fs`) — it's explicit and avoids clashing with an npm package. The author is on **Windows 11**; path/exec specifics are flagged. Fast-moving details are marked **⚡ Version note**. Confirm at nodejs.org/api.

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

Node.js is a JavaScript runtime built on Chrome's **V8** engine plus **libuv** (the C library providing the event loop and async I/O). It gives JS access to the filesystem, network, processes, and OS.

```bash
node --version                 # check version
node app.js                    # run a script
node --watch app.js            # re-run on file changes (built in)
node --env-file=.env app.js    # load env vars from .env (built in)
node                           # REPL
node -e "console.log(process.platform)"   # one-liner
```

**Version managers** (install/switch Node versions): **fnm** (fast), **nvm**, **volta**. On Windows, `winget install Schniz.fnm` then `fnm use 22`.

---

## 2. Modules: CommonJS vs ESM

Node supports two module systems. **[B/I]**

```js
// ----- ESM (modern, recommended) -----
// Enable by: "type": "module" in package.json, or use a .mjs extension.
import fs from "node:fs/promises";
import { join } from "node:path";
export function hello() { return "hi"; }
export default class App {}

// ----- CommonJS (classic Node) -----
// Default for .cjs, or when "type" isn't "module".
const fs = require("node:fs");
module.exports = { hello };
module.exports.default = App;
```

| | CommonJS | ESM |
|---|---|---|
| Import | `require()` | `import` |
| Export | `module.exports` | `export` |
| Loading | synchronous | async, static (+ dynamic `import()`) |
| `__dirname` | available | use `import.meta.dirname` (Node 20.11+) |
| Top-level await | no | yes |

```js
// Getting the current file/dir in ESM:
console.log(import.meta.url);        // file:// URL
console.log(import.meta.dirname);    // directory (Node 20.11+)
console.log(import.meta.filename);   // full file path
// Older pattern:
import { fileURLToPath } from "node:url";
import { dirname } from "node:path";
const __dirname = dirname(fileURLToPath(import.meta.url));

// Dynamic import works in both systems and is async:
const { default: cfg } = await import("./config.js");
```

---

## 3. npm & Packages

### 3.1 package.json **[B]**

```json
{
  "name": "myapp",
  "version": "1.0.0",
  "type": "module",
  "engines": { "node": ">=22" },
  "scripts": {
    "dev": "node --watch src/index.js",
    "start": "node src/index.js",
    "test": "node --test"
  },
  "dependencies": { "express": "^5.0.0" },
  "devDependencies": { "typescript": "^5.6.0" },
  "exports": { ".": "./src/index.js" }
}
```

### 3.2 Working with packages **[B]**

```bash
npm init -y                  # create package.json
npm install express          # add a dependency (writes package-lock.json)
npm install -D typescript     # dev dependency
npm install -g some-cli       # global install
npm ci                        # clean install from lockfile (use in CI)
npm run dev                   # run a script
npx cowsay hi                 # run a package binary without installing
npm update / npm outdated     # manage versions
npm uninstall express
```

- **Semver ranges:** `^1.2.3` allows `<2.0.0` (minor/patch), `~1.2.3` allows `<1.3.0` (patch), `1.2.3` exact.
- **Lockfile** (`package-lock.json`) pins exact resolved versions — commit it.
- Alternatives: **pnpm** (fast, space-efficient via a content-addressed store), **yarn**, **bun**.
- **Workspaces** (monorepos): `"workspaces": ["packages/*"]` in the root package.json.

---

## 4. The Event Loop & Async Model

Node is single-threaded for *your* JS; libuv handles I/O on a thread pool and queues callbacks. **[I/A]**

### 4.1 Phases (simplified)

1. **timers** — `setTimeout`/`setInterval` callbacks
2. **pending callbacks** — some system ops
3. **poll** — retrieve I/O events, run their callbacks
4. **check** — `setImmediate` callbacks
5. **close** — `close` event callbacks

Between every callback, the **microtask queue** (resolved promises + `queueMicrotask`) and **`process.nextTick`** queue are drained — `nextTick` first, then promises.

```js
console.log("sync 1");
setTimeout(() => console.log("timeout"), 0);
setImmediate(() => console.log("immediate"));
Promise.resolve().then(() => console.log("promise (microtask)"));
process.nextTick(() => console.log("nextTick"));
console.log("sync 2");
// Order: sync 1, sync 2, nextTick, promise, then timeout/immediate (order can vary).
```

### 4.2 Don't block the loop **[I, critical]**

A long synchronous loop or `fs.readFileSync` of a huge file freezes *everything* (no requests served, no timers). Offload CPU work (§12) and prefer async APIs.

```js
// promisify a callback API:
import { promisify } from "node:util";
import { exec } from "node:child_process";
const execP = promisify(exec);
```

---

## 5. File System

`node:fs` has three flavors: **promises** (`node:fs/promises` — prefer), **callback** (`fs.readFile(..., cb)`), and **sync** (`fs.readFileSync` — only at startup/CLIs). **[I]**

```js
import { readFile, writeFile, appendFile, mkdir, readdir, rm, stat,
         rename, copyFile, access } from "node:fs/promises";
import { constants } from "node:fs";

// Read / write text (encoding -> string; omit it -> Buffer):
const text = await readFile("config.json", "utf8");
await writeFile("out.txt", "hello\n", "utf8");   // overwrite
await appendFile("app.log", "line\n", "utf8");

// Directories:
await mkdir("data/sub", { recursive: true });    // mkdir -p
const entries = await readdir("data", { withFileTypes: true });
for (const e of entries) console.log(e.name, e.isDirectory() ? "dir" : "file");
// Recursive listing (Node 20+):
const all = await readdir("src", { recursive: true });

// Delete / move / copy:
await rm("tmp", { recursive: true, force: true });
await rename("old.txt", "new.txt");
await copyFile("a.txt", "b.txt");

// Existence & metadata:
try { await access("file.txt", constants.R_OK); console.log("readable"); }
catch { console.log("missing/no access"); }
const s = await stat("file.txt");
console.log(s.size, s.isFile(), s.mtime);

// Watch for changes:
import { watch } from "node:fs";
watch("src", { recursive: true }, (event, filename) => console.log(event, filename));
```

For huge files, use **streams** (§8) instead of reading the whole thing into memory.

---

## 6. Paths & OS / Process Info

```js
import path from "node:path";
import os from "node:os";

// Paths (cross-platform — uses \ on Windows, / elsewhere):
console.log(path.join("a", "b", "c.txt"));   // a\b\c.txt on Windows
console.log(path.resolve("c.txt"));          // absolute from cwd
console.log(path.parse("/a/b.txt"));         // { root, dir, base, ext, name }
console.log(path.extname("x.tar.gz"));       // ".gz"
console.log(path.sep);                       // "\\" on Windows, "/" elsewhere
// path.posix / path.win32 force a style regardless of platform.

// OS info:
console.log(os.platform());        // 'win32' | 'linux' | 'darwin'
console.log(os.arch());            // 'x64' | 'arm64'
console.log(os.cpus().length);     // logical CPU count
console.log(os.totalmem(), os.freemem());
console.log(os.hostname(), os.homedir(), os.tmpdir());
console.log(os.userInfo().username);
console.log(os.networkInterfaces());

// process — the running program:
console.log(process.pid, process.platform, process.version, process.versions.v8);
console.log(process.cwd());          // working dir
process.chdir("..");                 // change it
console.log(process.argv);           // ['node', '/path/script.js', ...your args]
console.log(process.env.PATH);       // env var
process.exitCode = 1;                // set exit code (preferred over process.exit)
process.stdout.write("no newline");  // raw stdout
```

---

## 7. Executing External Commands

`node:child_process` — four functions: **[I]**

| Function | Use |
|---|---|
| `execFile(file, args)` | run a binary with args; buffers output; **no shell** (safe) |
| `spawn(cmd, args)` | stream output; for long-running / large output |
| `exec(cmdString)` | run a string **through a shell** (convenient, injection-risky) |
| `fork(modulePath)` | spawn another Node process with an IPC channel |

```js
import { execFile, spawn } from "node:child_process";
import { promisify } from "node:util";
const execFileP = promisify(execFile);

// Capture output safely (args array, no shell):
const { stdout } = await execFileP("git", ["rev-parse", "--abbrev-ref", "HEAD"]);
console.log("branch:", stdout.trim());

// Run real tools with a working dir and env:
await execFileP("npm", ["install"], { cwd: "frontend" });
await execFileP("python", ["build.py"], { env: { ...process.env, MODE: "prod" } });

// Stream output live (better for builds/long commands):
const isWin = process.platform === "win32";
const child = spawn(isWin ? "npm.cmd" : "npm", ["run", "build"], { stdio: "inherit" });
child.on("close", code => console.log("build exited", code));

// Why set cwd instead of running `cd`? Because each command is its own process;
// `cd` in one wouldn't affect the next. Pass { cwd } instead.

// SECURITY: never interpolate untrusted input into exec()'s shell string.
// Use execFile/spawn with an args array so arguments can't be reinterpreted.
```

> **⚡ Windows note:** npm/npx/many CLIs are `.cmd` shims. `execFile("npm", ...)` may fail with ENOENT; use `"npm.cmd"`, or `{ shell: true }`, or `spawn(..., { shell: true })`.

---

## 8. Streams

Streams process data in **chunks** without loading it all into memory — a Node superpower. **[I/A]**

```js
import { createReadStream, createWriteStream } from "node:fs";
import { pipeline } from "node:stream/promises";
import { createGzip } from "node:zlib";

// Copy + gzip a large file with backpressure handled for you:
await pipeline(
  createReadStream("big.log"),
  createGzip(),
  createWriteStream("big.log.gz"),
);

// Read a stream line-friendly via async iteration:
for await (const chunk of createReadStream("data.txt", { encoding: "utf8" })) {
  process.stdout.write(chunk);
}

// A Transform stream (uppercase each chunk):
import { Transform } from "node:stream";
const upper = new Transform({
  transform(chunk, _enc, cb) { cb(null, chunk.toString().toUpperCase()); },
});
await pipeline(createReadStream("in.txt"), upper, createWriteStream("out.txt"));
```

Stream types: **Readable**, **Writable**, **Duplex**, **Transform**. Use `pipeline()` (handles errors + cleanup) rather than `.pipe()` chains.

---

## 9. Events (EventEmitter)

```js
import { EventEmitter } from "node:events";

class Job extends EventEmitter {
  run() {
    this.emit("start");
    setTimeout(() => this.emit("done", { ok: true }), 100);
  }
}
const job = new Job();
job.on("start", () => console.log("started"));     // listen
job.once("done", res => console.log("done", res)); // listen once
job.run();

// Always handle 'error' events — an unhandled 'error' THROWS and crashes:
job.on("error", err => console.error(err));
// Too many listeners warns at 10 by default: emitter.setMaxListeners(n).
```

---

## 10. Other Core Modules

```js
// crypto — hashing, HMAC, random, encryption:
import { createHash, randomUUID, randomBytes, scryptSync } from "node:crypto";
console.log(createHash("sha256").update("hello").digest("hex"));
console.log(randomUUID());                       // a v4 UUID
console.log(randomBytes(16).toString("hex"));    // secure random token
const key = scryptSync("password", "salt", 32);  // password-based key derivation

// buffer — binary data:
const buf = Buffer.from("hello", "utf8");
console.log(buf.toString("base64"));             // aGVsbG8=

// url:
const u = new URL("https://x.com/p?q=1");
console.log(u.hostname, u.searchParams.get("q"));

// util:
import { promisify, inspect, parseArgs } from "node:util";
console.log(inspect({ a: { b: { c: 1 } } }, { depth: null, colors: true }));

// readline — interactive prompts:
import readline from "node:readline/promises";
const rl = readline.createInterface({ input: process.stdin, output: process.stdout });
// const name = await rl.question("Name? "); rl.close();

// zlib, dns, timers/promises (setTimeout as a promise):
import { setTimeout as sleep } from "node:timers/promises";
await sleep(500);                                 // non-blocking delay
```

---

## 11. Building CLIs

```js
#!/usr/bin/env node
import { parseArgs } from "node:util";

const { values, positionals } = parseArgs({
  allowPositionals: true,
  options: {
    output:  { type: "string",  short: "o", default: "dist" },
    verbose: { type: "boolean", short: "v", default: false },
    help:    { type: "boolean", short: "h" },
  },
});

if (values.help) {
  console.log("usage: build [files...] -o <dir> [-v]");
  process.exit(0);
}
if (values.verbose) console.log("building", positionals, "->", values.output);
process.exit(0);     // explicit exit code
```

Add `"bin": { "mytool": "./cli.js" }` to package.json so `npm install -g` creates a `mytool` command. For richer CLIs: **commander**, **yargs**, **clipanion**; for prompts/colors: **@clack/prompts**, **chalk**, **ora**.

```js
// Read piped stdin (e.g. `cat f | node tool.js`):
let input = "";
for await (const chunk of process.stdin) input += chunk;
console.log(process.stdin.isTTY ? "interactive" : "piped");
```

---

## 12. Concurrency & Scaling

| Tool | Use |
|---|---|
| `worker_threads` | CPU-bound work in real threads (shares memory via SharedArrayBuffer) |
| `child_process.fork` | separate Node processes with IPC messaging |
| `cluster` | fork N workers sharing a server port (scale a server across cores) |

```js
// worker_threads — offload CPU work so the main loop stays responsive:
import { Worker, isMainThread, parentPort, workerData } from "node:worker_threads";
import { fileURLToPath } from "node:url";

if (isMainThread) {
  const worker = new Worker(fileURLToPath(import.meta.url), { workerData: 40 });
  worker.on("message", result => console.log("fib =", result));
} else {
  const fib = n => (n < 2 ? n : fib(n - 1) + fib(n - 2));
  parentPort.postMessage(fib(workerData));
}
```

```js
// cluster — run one worker per CPU, all sharing port 3000:
import cluster from "node:cluster";
import os from "node:os";
import http from "node:http";
if (cluster.isPrimary) {
  for (let i = 0; i < os.cpus().length; i++) cluster.fork();
} else {
  http.createServer((req, res) => res.end("hi from " + process.pid)).listen(3000);
}
```

---

## 13. Error Handling & Graceful Shutdown

```js
// Async errors -> try/catch with await:
async function main() {
  try {
    await doWork();
  } catch (err) {
    console.error("failed:", err);
    process.exitCode = 1;
  }
}

// Last-resort handlers (log + exit; don't keep running in a broken state):
process.on("uncaughtException", err => { console.error("uncaught", err); process.exit(1); });
process.on("unhandledRejection", reason => { console.error("unhandled", reason); process.exit(1); });

// Graceful shutdown — close servers/DB on SIGINT/SIGTERM:
function shutdown(signal) {
  console.log(`\n${signal} received, shutting down...`);
  server.close(() => process.exit(0));
  setTimeout(() => process.exit(1), 10_000).unref();   // force-exit after 10s
}
process.on("SIGINT", () => shutdown("SIGINT"));    // Ctrl-C
process.on("SIGTERM", () => shutdown("SIGTERM"));  // docker/k8s stop

// AbortController for timeouts / cancellation:
const ac = new AbortController();
const t = setTimeout(() => ac.abort(), 5000);
try {
  const res = await fetch("https://example.com", { signal: ac.signal });
} finally { clearTimeout(t); }
```

---

## 14. Networking: HTTP & fetch

```js
// Outbound requests — native fetch (no library needed):
const res = await fetch("https://api.example.com/users", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: "Ada" }),
});
if (!res.ok) throw new Error(`HTTP ${res.status}`);
const data = await res.json();

// A minimal server with the core http module (frameworks like Express/Fastify
// live in their own guides — use them for real apps):
import http from "node:http";
const server = http.createServer((req, res) => {
  if (req.url === "/health") {
    res.writeHead(200, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ ok: true }));
  } else {
    res.writeHead(404).end("not found");
  }
});
server.listen(3000, () => console.log("listening on :3000"));
```

---

## 15. Testing & Debugging

```js
// Built-in test runner — file: math.test.js, run: `node --test`
import { test, describe, before, mock } from "node:test";
import assert from "node:assert/strict";

describe("math", () => {
  test("adds", () => {
    assert.equal(2 + 3, 5);
  });
  test("async", async () => {
    const v = await Promise.resolve(42);
    assert.equal(v, 42);
  });
  test("mocking", () => {
    const fn = mock.fn(() => 1);
    fn();
    assert.equal(fn.mock.calls.length, 1);
  });
});
```

```bash
node --test                 # run all *.test.js / test/ files
node --test --watch         # re-run on change
node --test --experimental-test-coverage   # coverage
node --inspect app.js       # debugger; attach VS Code / chrome://inspect
```

Popular alternatives: **Vitest**, **Jest**. For logging in production use **pino** (fast, structured) rather than `console.log`.

---

## 16. Data & Persistence

```js
// Built-in SQLite (Node 22.5+, stabilizing):
import { DatabaseSync } from "node:sqlite";
const db = new DatabaseSync("app.db");
db.exec("CREATE TABLE IF NOT EXISTS users(id INTEGER PRIMARY KEY, name TEXT)");
const insert = db.prepare("INSERT INTO users(name) VALUES (?)");
insert.run("Ada");
console.log(db.prepare("SELECT * FROM users").all());

// JSON config:
import { readFile } from "node:fs/promises";
const config = JSON.parse(await readFile("config.json", "utf8"));

// Env config: node --env-file=.env app.js, then read process.env.
```

For real databases see the PostgreSQL, MongoDB, SQLite, Prisma, and Redis guides in this library.

---

## 17. Performance & Production

- **Profile** before optimizing: `node --prof app.js` then `node --prof-process`, or `--cpu-prof`/`--heap-prof`, or Chrome DevTools via `--inspect`.
- **Don't block the event loop** — move CPU work to workers; stream large I/O.
- **Memory leaks:** growing global Maps/arrays, uncleared timers, lingering event listeners, closures capturing big data. Take heap snapshots to find them.
- **Security:** never `eval`/`exec` untrusted input; validate inputs; keep deps patched (`npm audit`); use the **permission model** (`node --permission --allow-fs-read=./data app.js`) to sandbox.
- **Secrets:** environment variables / a secrets manager — never commit them.
- **Run in production:** a process manager (**PM2**, systemd) or a container (Docker — see the Docker guide), behind a reverse proxy; log structured JSON; expose a health endpoint; handle SIGTERM for zero-downtime deploys.

---

## 18. Project Structure & Best Practices

```
myapp/
├── package.json
├── .env                  (gitignored)
├── src/
│   ├── index.js          (entry / bootstrap)
│   ├── config.js
│   ├── routes/
│   ├── services/
│   └── lib/
└── test/
```

**Best practices:** ESM + `node:` prefixes; async/await over callbacks; handle every rejection; validate env at startup; small modules with clear exports; `npm ci` in CI; pin Node via `engines` + a version manager; lint/format with ESLint+Prettier or Biome.

---

## 19. Study Path & Build-to-Learn Projects

**Order:** §1–3 (run code, modules, npm) → §4 (event loop — the mental model) → §5–7 (fs, paths/OS, exec — immediately useful) → §8–9 (streams, events) → §11 (CLIs) → §13–14 (errors, HTTP) → §12, §15–17 (scaling, testing, production).

**Projects:**
1. **CLI tool** — read/transform files, parse args with `parseArgs`, shell out to git. Exercises §5, §7, §11.
2. **Log processor** — stream a huge file, transform, gzip the output. Exercises §8.
3. **File watcher / dev task runner** — watch a dir, run a build command on change. Exercises §5, §7, §9.
4. **REST API** — start with core `http`, then move to **Fastify** or **NestJS** (their own guides). Exercises §14.
5. **Worker pool** — distribute CPU work across `worker_threads`. Exercises §12.

**Next:** a web framework (Fastify/NestJS/Express), a database layer (Prisma/pg/MongoDB), and Docker for deployment — all covered elsewhere in this library.

---

*Part of the offline developer study library. Written for Node.js 22/24 LTS as of 2026. Confirm fast-moving APIs against nodejs.org/api.*
