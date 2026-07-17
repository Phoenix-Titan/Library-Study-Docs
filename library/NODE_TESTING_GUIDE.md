# Testing in Node.js — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone who writes JavaScript or TypeScript and has never written an automated test in their life — as well as developers who write a few tests but suspect they are doing it wrong (slow suites, brittle mocks, flaky CI). By the end you will understand *why* we test, the full taxonomy — unit, integration, end-to-end, snapshot, property-based, mutation, benchmark, fuzz — and be able to write fast, trustworthy, production-grade tests with the modern 2026 toolchain. Every concept is explained **prose-first**: *what* it is, *why* it exists, *when* you reach for it, *how* to use it, and the *gotcha* that bites people — and only then the heavily-commented, runnable code. Read it top-to-bottom the first time; afterwards use the Table of Contents as a lookup.
>
> **Version note:** This guide targets **Node.js 22 LTS and 24** with the stabilised built-in **`node:test`** runner and **`node:assert`**. The third-party runners are **Vitest 3.x** (the recommended default for new TypeScript projects), **Jest 30.x** (mature, ubiquitous, best where already established), with **Supertest 7.x** for HTTP assertions, **Playwright 1.5x** for end-to-end browser tests, **@nestjs/testing 11** for NestJS, **fast-check 3.x** for property-based testing, **Testcontainers for Node 10.x** for real-database integration, **c8 / V8** for coverage, and **TypeScript 5.7+** run through **`tsx`**, **`ts-jest`**, or Vitest's native transform. Fast-moving details are flagged **⚡ Version note**. The author is on **Windows 11**, so shell commands are shown for PowerShell and POSIX where they differ.
>
> **This guide's place in the library:** Testing sits on top of everything else you build. It assumes you can already write the code under test — see [Node.js](NODEJS_GUIDE.md) and [JavaScript](JAVASCRIPT_GUIDE.md) for the language and runtime (TypeScript is covered inline in §6), Express (covered in the Node.js guide) and [NestJS](NESTJS_GUIDE.md) / [NestJS + GraphQL](NESTJS_GRAPHQL_GUIDE.md) for the HTTP frameworks whose tests we write in §9–§10, and [PostgreSQL](POSTGRESQL_GUIDE.md) / [Prisma](PRISMA_ORM_GUIDE.md) for the database layer we integration-test with Testcontainers in §11. The pipeline that runs all of this on every push is covered in [GitHub Actions CI/CD](GITHUB_ACTIONS_CICD_GUIDE.md), and §18 shows the testing-specific parts. Cross-links appear inline throughout. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.

---

## Table of Contents

1. [Why Test, the Pyramid and the Testing Trophy](#1-why-test-the-pyramid-and-the-testing-trophy) **[B]**
2. [The Runner Landscape](#2-the-runner-landscape) **[B]**
3. [Testing Fundamentals](#3-testing-fundamentals) **[B]**
4. [Async Testing Done Right](#4-async-testing-done-right) **[B/I]**
5. [Mocking and Test Doubles](#5-mocking-and-test-doubles) **[I]**
6. [TypeScript Testing](#6-typescript-testing) **[I]**
7. [Coverage and Watch Mode](#7-coverage-and-watch-mode) **[I]**
8. [Snapshot Testing](#8-snapshot-testing) **[I]**
9. [HTTP and API Integration with Supertest](#9-http-and-api-integration-with-supertest) **[I]**
10. [NestJS Testing](#10-nestjs-testing) **[I/A]**
11. [Database Integration with Testcontainers](#11-database-integration-with-testcontainers) **[A]**
12. [End-to-End Browser Tests with Playwright](#12-end-to-end-browser-tests-with-playwright) **[A]**
13. [Property-Based Testing with fast-check](#13-property-based-testing-with-fast-check) **[A]**
14. [Benchmarking](#14-benchmarking) **[I]**
15. [Fuzzing](#15-fuzzing) **[A]**
16. [Filesystem and OS Code Testing](#16-filesystem-and-os-code-testing) **[I]**
17. [Mutation Testing with Stryker](#17-mutation-testing-with-stryker) **[A]**
18. [Continuous Integration with GitHub Actions](#18-continuous-integration-with-github-actions) **[A]**
19. [Gotchas and Best Practices](#19-gotchas-and-best-practices) **[I/A]**
20. [Study Path and Build-to-Learn Projects](#20-study-path-and-build-to-learn-projects)

---

## 1. Why Test, the Pyramid and the Testing Trophy

### 1.1 What a test actually is **[B]**

An **automated test** is nothing mystical: it is ordinary code that runs your *other* code with a known input, then checks that the output is what you expected. If the check passes, the test is silent; if it fails, the test throws an error and the test runner reports it. That is the entire idea. A test runner is just a program that finds your test files, executes them, catches the thrown errors, and prints a green/red summary.

Here is the smallest possible test, written by hand with zero libraries, so you can see there is no magic:

```js
// add.js — the code under test ("SUT" = System Under Test)
function add(a, b) {
  return a + b;
}
module.exports = { add };
```

```js
// add.manual-test.js — a test with no framework at all
const assert = require('node:assert'); // Node's built-in assertion library
const { add } = require('./add');

// "Arrange" the inputs, "Act" by calling the function, "Assert" the result.
const result = add(2, 3);
assert.strictEqual(result, 5, 'add(2,3) should be 5'); // throws if not equal

console.log('OK: add works'); // only prints if the assertion did NOT throw
```

Run it with `node add.manual-test.js`. If `add` were buggy (say it returned `a - b`), `assert.strictEqual` would throw an `AssertionError` and the process would exit non-zero. Everything else in this guide — `describe`, `it`, `expect`, mocks, coverage — is convenience layered on top of exactly this: **run code, assert, report.**

### 1.2 Why we test at all **[B]**

You could test everything by hand: run the app, click around, eyeball the result. People do, and it works — once. The problem is *repetition* and *regression*. Every time you change code, you risk breaking something that used to work (a **regression**). Manual re-checking of the whole app after every change does not scale past a few features; automated tests do. The concrete payoffs:

- **Confidence to change code.** Refactoring without tests is guesswork. With a good suite you can restructure aggressively and trust a red test to catch a mistake in seconds.
- **Executable specification.** A well-named test documents intended behaviour better than a comment, because it cannot drift out of date without failing.
- **Faster feedback than humans.** A machine re-checks a thousand behaviours in the time it takes you to reload a page once.
- **Design pressure.** Code that is hard to test is usually hard to *use* — too many dependencies, hidden global state, doing five things at once. Writing the test first often reveals a better shape for the code.
- **Fewer 3 a.m. incidents.** Bugs caught on your laptop cost minutes; bugs caught in production cost customers.

Testing is not about proving code is *correct* (you cannot, in general). It is about **buying confidence** at a reasonable cost. The rest of this section is about spending that budget wisely.

### 1.3 The test pyramid **[B]**

The classic mental model, from Mike Cohn, is the **test pyramid**. It sorts tests by *scope* — how much of the system each one exercises — and argues for having many small tests and few large ones.

| Layer | Scope | Speed | Count | What it catches |
|-------|-------|-------|-------|-----------------|
| **Unit** | One function/class in isolation | Microseconds–ms | Many (hundreds–thousands) | Logic bugs, edge cases |
| **Integration** | Several units together, or a unit + a real dependency (DB, HTTP) | Milliseconds–seconds | Some (dozens–hundreds) | Wiring, contracts, serialization, SQL |
| **End-to-end (E2E)** | The whole system through its real UI/API | Seconds | Few (a handful–dozens) | User journeys, deployment, config |

The shape is a pyramid because the ideal distribution is *wide at the bottom, narrow at the top*. Unit tests are cheap and fast, so you can afford thousands; E2E tests are slow, flaky, and expensive to maintain, so you keep them few and reserve them for critical journeys. The **anti-pattern** is the "ice-cream cone" (or "inverted pyramid"): mostly slow E2E tests and few unit tests, which gives you a suite that takes 40 minutes, fails randomly, and no one trusts.

### 1.4 The testing trophy **[B]**

For JavaScript specifically, Kent C. Dodds popularised a refinement called the **testing trophy**. Its motivation: in JS/TS a huge share of real bugs live at the *seams between modules* — serialization, wrong props, a mis-shaped API response — which pure unit tests (each module mocked away from its neighbours) systematically miss. So the trophy shifts weight toward **integration** tests. From bottom to top:

- **Static** (the base) — TypeScript, ESLint, and the type checker. These catch a whole class of bugs (typos, wrong argument types, null access) *before a single test runs*, at zero runtime cost. Treat `tsc --noEmit` and `eslint` as the cheapest tests you own.
- **Unit** — small, isolated logic tests. Still valuable, especially for pure functions with tricky edge cases.
- **Integration** (the largest band) — several real units working together, mocking only the awkward boundaries (network, clock, third-party). This is where Dodds says to spend most effort, because these tests resemble how the code is actually used and catch the bugs that matter, while staying much faster than E2E.
- **E2E** (the small top) — the real thing in a real browser, for the few flows you cannot afford to break.

His guiding slogan is worth memorising: **"The more your tests resemble the way your software is used, the more confidence they can give you."** A test that reaches into a class's private fields and asserts on them resembles nothing a user does; a test that hits your Express route and checks the JSON resembles exactly what a client does.

> Pyramid vs trophy is not a religious war. The pyramid's "push work down to fast tests" and the trophy's "integration catches JS's real bugs" are both true. In practice for a Node/TS backend: lean on the type checker, write plenty of fast unit tests for pure logic, write a strong band of integration tests (Supertest against your real app, Testcontainers against a real DB), and a thin layer of Playwright E2E for the money paths.

### 1.5 What an "efficient" test is **[B]**

A production-grade suite is not just "has tests." Aim for tests that are **FIRST**:

- **Fast** — the whole unit suite should run in seconds so you actually run it on save. Slow suites get skipped, and a skipped suite protects nothing.
- **Isolated / Independent** — each test sets up its own world and cleans up. No test may depend on another test having run first, or on execution order. Shared mutable state is the number-one cause of "passes alone, fails in the suite."
- **Repeatable** — same result every run, on every machine, regardless of date, timezone, or network. This means controlling time (`Date.now`), randomness (`Math.random`), and external services (mock or containerise them).
- **Self-validating** — the test decides pass/fail by itself with assertions. A test that prints output for a human to eyeball is not a test.
- **Timely** — written close to the code, ideally alongside or just after (TDD writes them just before). Tests written six months later tend to codify bugs as "expected."

Two more properties matter enormously in practice:

- **Test behaviour, not implementation.** Assert on *what the code does* (its observable output/effects), not *how* it does it. A test that breaks every time you rename a private method is a maintenance tax with no safety benefit. See §19.
- **Deterministic.** A test that fails 1-in-20 for no code reason (a **flaky** test) is worse than no test — it trains the team to ignore red. Determinism (§4, §19) is a first-class goal, not an afterthought.

---

## 2. The Runner Landscape

### 2.1 What a test runner does and why you need one **[B]**

The hand-written test in §1.1 works but is missing everything that makes a suite pleasant: it does not *discover* test files automatically, group related tests, run setup/teardown, isolate failures (one thrown error stops the whole file), report results nicely, measure coverage, run in parallel, or watch for changes. A **test runner** provides all of that. In the Node/JS world you have three serious choices in 2026, and — crucially — they share almost the same surface API (`describe`, `it`/`test`, hooks, assertions), so learning one teaches you most of the others.

### 2.2 The three runners at a glance **[B]**

| | **`node:test`** (built-in) | **Vitest 3** | **Jest 30** |
|---|---|---|---|
| Install | Nothing — ships with Node | `npm i -D vitest` | `npm i -D jest` |
| ESM support | Native | Native (Vite) | Native-ish (historically painful) |
| TypeScript | Via `tsx`/`--experimental-strip-types` | **Native** (Vite transform) | Via `ts-jest` or Babel |
| Speed | Fast, minimal | **Very fast** (Vite, parallel workers) | Fast, mature |
| Assertions | `node:assert` | `expect` (Chai/Jest-compatible) | `expect` (built-in) |
| Mocking | `mock` from `node:test` | `vi.*` | `jest.*` |
| Watch mode | `--watch` | Excellent | Good |
| Snapshots | Basic (`--experimental`) / via lib | Yes | Yes (the original) |
| Ecosystem | Growing | Large, momentum | Huge, mature |
| Best for | Zero-dependency libs, small tools | **New TS projects** | Existing Jest codebases, React (CRA legacy) |

**The decision guide, in one breath:**

- **New TypeScript project?** Use **Vitest.** It runs `.ts` with no config, is the fastest of the three, has a Jest-compatible `expect` (so Jest knowledge transfers), and integrates with the Vite ecosystem you probably already use for the frontend.
- **Writing a library and want zero dependencies / minimal supply-chain surface?** Use **`node:test`.** It is built into Node, needs no `package.json` devDependency, and is now stable. Perfect for open-source packages where every dependency is a liability.
- **Working in a codebase that already uses Jest?** Stay on **Jest.** It is mature and battle-tested; there is rarely a good reason to migrate a large working suite. Jest 30 modernised ESM and performance considerably.

### 2.3 The same test in all three runners **[B]**

To prove how similar they are, here is one tiny suite for the `add` function written three ways. Read all three; notice that only the imports and the assertion style differ.

**With `node:test` + `node:assert`:**

```js
// add.node.test.js  —  run with:  node --test
import { test, describe } from 'node:test';   // the runner primitives
import assert from 'node:assert/strict';       // "/strict" makes === the default

describe('add', () => {
  test('adds two positive numbers', () => {
    assert.equal(add(2, 3), 5);                // strict-equal because of /strict
  });

  test('handles negatives', () => {
    assert.equal(add(-2, -3), -5);
  });
});

function add(a, b) { return a + b; }
```

**With Vitest:**

```js
// add.vitest.test.js  —  run with:  npx vitest
import { describe, it, expect } from 'vitest'; // imported explicitly (or set globals:true)

describe('add', () => {
  it('adds two positive numbers', () => {
    expect(add(2, 3)).toBe(5);                 // .toBe uses Object.is (like ===)
  });

  it('handles negatives', () => {
    expect(add(-2, -3)).toBe(-5);
  });
});

function add(a, b) { return a + b; }
```

**With Jest:**

```js
// add.jest.test.js  —  run with:  npx jest
// describe/it/expect are GLOBAL in Jest — no import needed.
describe('add', () => {
  it('adds two positive numbers', () => {
    expect(add(2, 3)).toBe(5);
  });

  it('handles negatives', () => {
    expect(add(-2, -3)).toBe(-5);
  });
});

function add(a, b) { return a + b; }
```

The differences are cosmetic: `assert.equal(x, y)` vs `expect(x).toBe(y)`, and whether the primitives are imported (`node:test`, Vitest) or global (Jest, and Vitest with `globals: true`). Everything you learn about structuring tests applies to all three.

### 2.4 Getting each one set up **[B]**

**`node:test` — nothing to install.** With Node 22/24, just name files `*.test.js` (or put them in a `test/` directory) and run:

```bash
node --test                 # discovers and runs *.test.js across the project
node --test --watch         # re-run on file change
node --test --experimental-test-coverage   # built-in coverage report
```

**Vitest** — install and add a script:

```bash
npm i -D vitest
```

```json
// package.json (excerpt)
{
  "scripts": {
    "test": "vitest run",       // one-shot (for CI); omit "run" for watch mode
    "test:watch": "vitest",     // interactive watch
    "coverage": "vitest run --coverage"
  }
}
```

An optional `vitest.config.ts` lets you turn on Jest-style globals and pick the environment:

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,          // makes describe/it/expect global, like Jest (no imports)
    environment: 'node',    // 'node' for backend; 'jsdom'/'happy-dom' for DOM tests
    coverage: { provider: 'v8' }, // V8 coverage (fast, no instrumentation) — see §7
  },
});
```

**Jest** — install and initialise:

```bash
npm i -D jest
npx jest --init            # interactive: creates jest.config.js
```

```js
// jest.config.js (minimal for Node)
/** @type {import('jest').Config} */
module.exports = {
  testEnvironment: 'node',       // 'jsdom' for browser-like tests
  // For TypeScript, add a transform — see §6.
};
```

> ⚡ **Version note:** `node:test` graduated from experimental in Node 20 and is fully stable in 22/24; its coverage flag `--experimental-test-coverage` is the one piece still stabilising. Jest 30 (2025) dropped a lot of legacy weight and modernised ESM; Vitest 3 tightened its config API. Pin your runner version in `package.json` and read that runner's own changelog before a major upgrade — test runners change their reporter and config surfaces more often than most libraries.

### 2.5 Which one this guide uses **[B]**

To keep examples runnable and framework-neutral, this guide **shows Vitest by default** (because its `expect` API is the lingua franca and it needs no TypeScript config), **shows `node:test` wherever zero-dependency matters** (filesystem tests in §16, and any pure-Node snippet), and **notes the Jest equivalent** whenever the API differs (`vi.fn` ↔ `jest.fn`, `vi.mock` ↔ `jest.mock`, and so on — the mapping in §5.9). If you use Jest, mentally substitute `vi` → `jest` and the code works almost verbatim.

---

## 3. Testing Fundamentals

Every runner gives you the same four building blocks: a way to **group** tests, a way to **declare** a test, a way to **assert**, and **hooks** for setup/teardown. Master these and you can write 90% of all tests you will ever write.

### 3.1 describe, it and test **[B]**

- **`test(name, fn)`** (alias **`it`**) declares a single test case. The `name` is a human sentence describing the expected behaviour; `fn` contains the arrange/act/assert. `it` reads nicely with a `describe` ("it adds two numbers"); `test` reads nicely alone ("test adds two numbers"). They are identical — pick a convention and stick to it.
- **`describe(name, fn)`** groups related tests into a **suite**. Groups nest, produce indented output, and let hooks (below) apply to just that group. Grouping is organisational, not behavioural — it does not change what runs.

```ts
import { describe, it, expect } from 'vitest';

// A pure function we want to specify thoroughly.
function slugify(input: string): string {
  return input
    .trim()
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')   // non-alphanumerics -> single hyphen
    .replace(/^-+|-+$/g, '');       // trim leading/trailing hyphens
}

describe('slugify', () => {
  it('lowercases and hyphenates words', () => {
    expect(slugify('Hello World')).toBe('hello-world');
  });

  it('collapses runs of separators into one hyphen', () => {
    expect(slugify('a  --  b')).toBe('a-b');
  });

  it('trims leading and trailing separators', () => {
    expect(slugify('  !Hi!  ')).toBe('hi');
  });

  // Grouping edge cases in a nested describe keeps output readable.
  describe('edge cases', () => {
    it('returns empty string for punctuation-only input', () => {
      expect(slugify('!!!')).toBe('');
    });
  });
});
```

**Naming tests well matters more than beginners expect.** The name is what you read in the failure report at 3 a.m. Prefer *behaviour* descriptions ("rejects a negative amount") over *implementation* descriptions ("calls validateAmount"). A good pattern: `it('<does something observable> when <condition>')`.

### 3.2 The Arrange-Act-Assert shape **[B]**

Almost every good test has three visually distinct phases, often separated by blank lines. Keeping them separate makes tests skimmable:

```ts
it('applies a 10 percent discount to the subtotal', () => {
  // Arrange — build the inputs and the world the test needs.
  const cart = { subtotal: 200, discountRate: 0.1 };

  // Act — perform the single action under test.
  const total = applyDiscount(cart);

  // Assert — verify the observable outcome.
  expect(total).toBe(180);
});
```

A test should ideally **act once**. If you find yourself with multiple act/assert pairs, that is often two tests wearing a trench coat — split them, so a failure points at exactly one behaviour.

### 3.3 Assertions — node:assert strict vs expect **[B]**

An **assertion** is the statement that throws on failure. There are two families you will meet.

**`node:assert` (the built-in).** Always import the **strict** variant — `import assert from 'node:assert/strict'` — so that `assert.equal` uses deep strict equality (`===` semantics, no `==` coercion). The plain `node:assert` uses loose `==` for `equal`, which is a legacy footgun.

```js
import assert from 'node:assert/strict';

assert.equal(1 + 1, 2);                      // strict === for primitives
assert.deepEqual({ a: 1 }, { a: 1 });        // deep structural equality
assert.notEqual(cost, 0);
assert.ok(user.isActive);                    // truthiness check
assert.throws(() => JSON.parse('{bad'));     // asserts the fn throws
await assert.rejects(fetchUser(-1), /not found/); // asserts a promise rejects
assert.match('hello@x.com', /@/);            // regex match on a string
```

**`expect` (Jest / Vitest).** A fluent, chainable "matcher" API. It reads like English and produces rich diff output on failure, which is why most people prefer it for application code.

```ts
import { expect } from 'vitest';

expect(1 + 1).toBe(2);                        // Object.is — for primitives & identity
expect({ a: 1 }).toEqual({ a: 1 });           // deep value equality (ignores identity)
expect({ a: 1, b: 2 }).toMatchObject({ a: 1 }); // subset match — only listed keys
expect([1, 2, 3]).toContain(2);
expect('hello@x.com').toMatch(/@/);
expect(user.isActive).toBe(true);
expect(() => JSON.parse('{bad')).toThrow();
expect(value).toBeNull();
expect(value).toBeDefined();
expect(0.1 + 0.2).toBeCloseTo(0.3);           // floating-point-safe numeric compare
expect(list).toHaveLength(3);
expect(spy).toHaveBeenCalledWith('arg');      // for mocks — see §5
```

The single most common beginner mistake: **`toBe` vs `toEqual`.** `toBe` compares with `Object.is` — for objects and arrays that means *reference identity*, so `expect({a:1}).toBe({a:1})` **fails** because they are two different objects. Use `toBe` for primitives (numbers, strings, booleans) and for "is it literally the same object?"; use `toEqual` (or `assert.deepEqual`) for comparing the *contents* of objects and arrays.

| You want to check… | expect matcher | node:assert |
|---|---|---|
| Primitive equality / same reference | `toBe(x)` | `assert.equal` |
| Deep value equality | `toEqual(x)` | `assert.deepEqual` |
| Object contains a subset of fields | `toMatchObject(x)` | *(manual)* |
| Truthy / falsy | `toBeTruthy()` / `toBeFalsy()` | `assert.ok` |
| Throws | `toThrow(msg?)` | `assert.throws` |
| Promise rejects | `rejects.toThrow()` | `assert.rejects` |
| Floating-point near | `toBeCloseTo(x)` | *(manual epsilon)* |
| Array/string length | `toHaveLength(n)` | *(manual)* |

### 3.4 Hooks — beforeEach, afterEach, beforeAll, afterAll **[B]**

**Hooks** run setup and teardown code around your tests so you do not repeat it. There are four, and understanding *when each fires* is essential to avoiding shared-state bugs.

- **`beforeEach(fn)`** — runs before **every** test in its scope. Use it to build a *fresh* copy of any mutable state, so tests cannot pollute each other. This is your default choice.
- **`afterEach(fn)`** — runs after every test. Use it to clean up: reset mocks, close a connection opened per-test, restore a stubbed global.
- **`beforeAll(fn)`** — runs **once** before all tests in scope. Use it for expensive, *read-only* setup shared across tests: start a database container, compile something, open a pooled connection.
- **`afterAll(fn)`** — runs once after all tests. Use it to tear down what `beforeAll` created (stop the container, close the pool).

```ts
import { describe, it, expect, beforeEach, afterEach, beforeAll, afterAll } from 'vitest';

describe('ShoppingCart', () => {
  let cart: ShoppingCart;

  beforeAll(() => {
    // Runs ONCE. Good for expensive, immutable setup (e.g. load a fixture file).
  });

  beforeEach(() => {
    // Runs before EACH test — a brand-new cart, so tests never share state.
    cart = new ShoppingCart();
  });

  afterEach(() => {
    // Per-test cleanup. Here nothing is needed, but this is where you'd
    // restore mocks or close per-test resources.
  });

  afterAll(() => {
    // Runs ONCE at the end. Tear down what beforeAll created.
  });

  it('starts empty', () => {
    expect(cart.itemCount).toBe(0);
  });

  it('adds an item', () => {
    cart.add({ id: 'sku1', price: 10 });
    expect(cart.itemCount).toBe(1);
  });
});
```

**The golden rule of hooks:** prefer `beforeEach` + fresh state over `beforeAll` + shared state. `beforeAll` is an *optimisation* you reach for only when setup is genuinely expensive and the shared thing is genuinely read-only. The moment tests mutate a `beforeAll` object, you have re-introduced order-dependence and flakiness. In `node:test` the same hooks exist under the same names, imported from `node:test`.

### 3.5 only, skip and todo **[B]**

While developing you often want to run *just one* test, or park a test you have not finished. Every runner supports three modifiers:

- **`.only`** — run *only* this test (or suite) and ignore all its siblings. Invaluable when debugging a single failure. **Danger:** committing a stray `.only` silently disables the rest of your suite in CI. Add an ESLint rule (`no-only-tests`) or a CI grep to block it.
- **`.skip`** — do not run this test, but report it as skipped (not passed). Use it to temporarily disable a test with a `// TODO: re-enable when X lands` comment rather than deleting it.
- **`.todo`** — declare a test that does not exist yet. It shows up in the report as a reminder. Great for jotting down cases while writing code, TDD-style.

```ts
describe('parser', () => {
  it.only('parses the happy path', () => { /* only this runs */ });

  it.skip('handles malformed input', () => { /* skipped, still reported */ });

  it.todo('supports unicode identifiers'); // no body — a reminder to write it
});
```

You can also *filter* from the command line without editing code: `vitest run -t "happy path"` (or `jest -t`, or `node --test --test-name-pattern="happy path"`) runs only tests whose name matches — cleaner than `.only` because there is nothing to accidentally commit.

### 3.6 Where tests live and how they are named **[B]**

Two conventions coexist; pick one per project:

- **Co-located:** `src/cart.ts` next to `src/cart.test.ts`. Pros: easy to find, obvious when a file has no tests, moves with the code. This is the modern default, especially with Vitest.
- **Separate folder:** all tests under `test/` or `__tests__/` mirroring `src/`. Pros: keeps `src/` pure; some teams prefer it for shipping libraries.

Runners discover tests by filename glob. The near-universal patterns are `*.test.ts` and `*.spec.ts` (`.spec` is common in the Angular/NestJS world; they are interchangeable). Whatever you choose, be consistent — the glob in your config depends on it.

---

## 4. Async Testing Done Right

Most real code is asynchronous — it reads files, queries databases, calls HTTP APIs. Testing async code has one central rule: **the test runner must know when your asynchronous work is finished**, otherwise it declares the test passed while your assertions are still pending in the background, or it times out waiting for something that never resolves. Getting this right is the difference between a trustworthy suite and one full of false greens.

### 4.1 The core rule — return or await the promise **[B]**

If a test does asynchronous work, its function must **either be `async` and `await` the work, or return the promise.** Both tell the runner "wait for this before judging pass/fail." The `async` form is almost always clearer.

```ts
import { describe, it, expect } from 'vitest';

// The code under test returns a promise.
async function fetchUserName(id: number): Promise<string> {
  // pretend this hits a database
  return id === 1 ? 'Ada' : 'Unknown';
}

describe('fetchUserName', () => {
  // CORRECT — async test, await the result, then assert.
  it('returns the name for a known id', async () => {
    const name = await fetchUserName(1);
    expect(name).toBe('Ada');
  });

  // ALSO CORRECT — return the promise chain (the runner awaits the returned promise).
  it('returns Unknown for an unknown id', () => {
    return fetchUserName(999).then((name) => {
      expect(name).toBe('Unknown');
    });
  });
});
```

**The classic silent-failure bug** is forgetting the `await`/`return`:

```ts
// WRONG — no await, no return. The runner sees a synchronous function that
// throws nothing, so it PASSES instantly. The assertion runs later, in the
// background, and even if it throws, the test is already "green".
it('is a false green', () => {
  fetchUserName(1).then((name) => {
    expect(name).toBe('WRONG VALUE'); // this never fails the test!
  });
});
```

If you take one thing from this section: **every async test body must have an `await` (or `return`) in front of the async work.** ESLint's `no-floating-promises` (with `@typescript-eslint`) catches most of these at lint time — turn it on.

### 4.2 Testing that a promise rejects **[B/I]**

Testing the *failure* path is as important as the success path. The naive `try/catch` approach has a subtle trap: if the code *doesn't* throw, the `catch` never runs and the test silently passes. Guard against that, or use the built-in rejection matchers.

```ts
import { describe, it, expect } from 'vitest';

async function withdraw(balance: number, amount: number): Promise<number> {
  if (amount > balance) throw new Error('Insufficient funds');
  return balance - amount;
}

describe('withdraw', () => {
  // BEST — the runner's rejection matcher. Clear, and fails if it does NOT reject.
  it('rejects when amount exceeds balance', async () => {
    await expect(withdraw(100, 500)).rejects.toThrow('Insufficient funds');
    //    ^ the leading await is REQUIRED — .rejects returns a promise
  });

  // node:assert equivalent:
  // await assert.rejects(withdraw(100, 500), /Insufficient funds/);

  // Manual try/catch — only correct if you FAIL when no error is thrown.
  it('rejects (manual, done safely)', async () => {
    try {
      await withdraw(100, 500);
      expect.unreachable('should have thrown'); // fail if we get here
    } catch (err) {
      expect((err as Error).message).toMatch(/Insufficient/);
    }
  });
});
```

`expect(...).rejects` and `.resolves` unwrap the promise so you can chain any matcher: `await expect(p).resolves.toEqual({...})`, `await expect(p).rejects.toBeInstanceOf(TypeError)`. Forgetting the leading `await` here is the number-one async testing bug — the assertion becomes a floating promise and never runs.

### 4.3 The done-callback and why to avoid it **[I]**

Old callback-style APIs (and old tutorials) use a **`done` callback**: the test takes a `done` argument and the runner waits until you call it. It still works, but it is error-prone — forget to call `done` and the test hangs until timeout; call it twice and you get confusing errors; throw before calling it and you get an unhelpful timeout instead of the real error.

```ts
// Legacy style — avoid in new code.
it('reads a stream (callback style)', (done) => {
  const chunks: Buffer[] = [];
  stream.on('data', (c) => chunks.push(c));
  stream.on('end', () => {
    try {
      expect(Buffer.concat(chunks).toString()).toBe('hello');
      done();               // must remember this
    } catch (err) {
      done(err);            // must forward errors or the failure is a timeout
    }
  });
});
```

The modern fix: wrap the callback/event API in a promise (or use `events.once` / `stream/promises`) and write a normal `async` test:

```ts
import { once } from 'node:events';

it('reads a stream (promise style)', async () => {
  const chunks: Buffer[] = [];
  stream.on('data', (c) => chunks.push(c));
  await once(stream, 'end');                  // resolves when 'end' fires — no done()
  expect(Buffer.concat(chunks).toString()).toBe('hello');
});
```

Prefer promises everywhere. If you must test a genuinely callback-only API, promisify it with `node:util`'s `promisify` and await it.

### 4.4 Timeouts **[I]**

Every runner imposes a **per-test timeout** (Vitest and Jest default to 5000 ms) so a hung promise fails the test instead of freezing the whole suite. When a test legitimately needs longer (spinning up a container, a slow integration call), raise it *for that test* rather than globally.

```ts
// Vitest / Jest: third argument to it() is the timeout in milliseconds.
it('starts a database container', async () => {
  const container = await new PostgreSqlContainer().start();
  expect(container.getPort()).toBeGreaterThan(0);
  await container.stop();
}, 60_000); // 60s — containers are slow to boot

// node:test uses an options object:
// test('slow thing', { timeout: 60_000 }, async () => { ... });
```

If a test *times out* unexpectedly, the usual cause is an unresolved promise (a mock that never resolves, an `await` on something that never fires) or a real hang in the code. Do not just raise the timeout to hide it — a test that needs 30 seconds for logic that should take milliseconds is telling you something.

### 4.5 Testing concurrent and parallel behaviour **[I/A]**

Two different things get called "parallel":

- **Test-file parallelism** — runners execute different *test files* in separate worker processes/threads simultaneously for speed. This is why tests must be isolated: two files hitting the same database row will collide. Vitest and Jest do this by default; control it with `--pool`, `poolOptions`, `--maxWorkers`.
- **Concurrent tests within a file** — `it.concurrent(...)` runs the marked tests in the same file at the same time. Only safe when they share no mutable state. Use sparingly; the speed win is usually not worth the isolation risk for backend tests.

For code that itself does concurrency (e.g. `Promise.all`, a queue, a rate limiter), test the *observable* outcome — ordering, that all items were processed, that the limiter never exceeded N in flight — using deterministic fakes rather than real timing. Real `setTimeout`-based timing in tests is a top source of flake; see fake timers in §5.6.

---

## 5. Mocking and Test Doubles

### 5.1 What test doubles are and why they exist **[I]**

A **test double** is a stand-in you substitute for a real dependency during a test — the way a stunt double stands in for an actor. You use one when the real dependency is *slow* (a network call), *non-deterministic* (the current time, a random ID), *has side effects you don't want* (sending a real email, charging a real card), or *is simply not available* in the test environment. The umbrella term is "test double"; the sub-types (from Gerard Meszaros's taxonomy) are worth knowing because people use them loosely:

| Double | What it does | Example |
|---|---|---|
| **Dummy** | Passed to satisfy a signature, never used | A `logger` argument you don't care about |
| **Stub** | Returns canned answers to calls | `getUser()` always returns a fixed user |
| **Spy** | A real or stub function that *records* how it was called | Assert `sendEmail` was called once with the right address |
| **Mock** | A spy with *pre-set expectations* it verifies | "expect `save` to be called exactly once" |
| **Fake** | A working but simplified implementation | An in-memory repository instead of a real DB |

In JS tooling the words blur: `vi.fn()` / `jest.fn()` creates a function that is simultaneously a stub (you can set its return value) and a spy (it records calls). Do not agonise over the vocabulary; understand *what behaviour you need* (canned return? call recording? full fake?) and reach for the matching tool.

### 5.2 Mock functions — vi.fn and jest.fn **[I]**

The workhorse. `vi.fn()` (Vitest) / `jest.fn()` (Jest) creates a fresh function that records every call and lets you program its behaviour. It exposes a `.mock` property with the call history you assert against.

```ts
import { describe, it, expect, vi } from 'vitest';

describe('mock functions', () => {
  it('records calls and returns programmed values', () => {
    const fn = vi.fn();                 // a blank spy/stub

    fn.mockReturnValue(42);             // program a return value
    const out = fn('a', 'b');

    expect(out).toBe(42);
    expect(fn).toHaveBeenCalled();               // was it called at all?
    expect(fn).toHaveBeenCalledTimes(1);         // exactly once
    expect(fn).toHaveBeenCalledWith('a', 'b');   // with these arguments
    expect(fn.mock.calls[0]).toEqual(['a', 'b']); // raw access to call history
  });

  it('programs async return values and sequences', async () => {
    const fetchThing = vi.fn();
    fetchThing
      .mockResolvedValueOnce({ id: 1 })   // first call resolves to this
      .mockResolvedValueOnce({ id: 2 })   // second call
      .mockRejectedValue(new Error('gone')); // all later calls reject

    expect(await fetchThing()).toEqual({ id: 1 });
    expect(await fetchThing()).toEqual({ id: 2 });
    await expect(fetchThing()).rejects.toThrow('gone');
  });

  it('runs a fake implementation', () => {
    const add = vi.fn((a: number, b: number) => a + b); // real behaviour
    expect(add(2, 3)).toBe(5);
    expect(add).toHaveBeenCalledWith(2, 3);
  });
});
```

Key methods: `mockReturnValue` / `mockReturnValueOnce`, `mockResolvedValue` / `mockRejectedValue` (async), `mockImplementation` / `mockImplementationOnce` (custom body), and `mockClear()` (wipe call history) / `mockReset()` (also wipe behaviour) / `mockRestore()` (restore a spied-on original — see next). In Jest, replace `vi` with `jest`; the method names are identical.

### 5.3 Spying on existing methods — spyOn **[I]**

Often you do not want to replace a whole module — just observe or override *one method* of a real object while leaving the rest intact. `vi.spyOn(object, 'method')` wraps the real method so you can assert on calls and optionally replace its behaviour, then restore it.

```ts
import { it, expect, vi, afterEach } from 'vitest';

const analytics = {
  track(event: string) { /* real: sends over the network */ },
};

function checkout(analyticsSvc: typeof analytics) {
  analyticsSvc.track('checkout_started');
  return 'ok';
}

afterEach(() => vi.restoreAllMocks()); // undo every spy after each test

it('tracks a checkout event without hitting the network', () => {
  // Wrap the real method and stub its body so no network call happens.
  const spy = vi.spyOn(analytics, 'track').mockImplementation(() => {});

  checkout(analytics);

  expect(spy).toHaveBeenCalledWith('checkout_started');
  // After restoreAllMocks(), analytics.track is the real method again.
});
```

Always pair `spyOn` with a restore (`vi.restoreAllMocks()` in `afterEach`, or `spy.mockRestore()`) or the override leaks into later tests — a classic shared-state bug.

### 5.4 Module mocking — vi.mock and jest.mock **[I/A]**

Sometimes the dependency is a whole imported module (a database client, an email SDK, `node:fs`). **Module mocking** replaces the module for the duration of the test file so that any code importing it gets your fake instead of the real thing.

```ts
// userService.ts — the code under test imports a mailer module
import { sendEmail } from './mailer';

export async function registerUser(email: string) {
  // ... create the user ...
  await sendEmail(email, 'Welcome!'); // we do NOT want a real email in tests
  return { email };
}
```

```ts
// userService.test.ts
import { describe, it, expect, vi } from 'vitest';
import { registerUser } from './userService';
import { sendEmail } from './mailer';

// Replace the ENTIRE ./mailer module. The factory returns the fake shape.
vi.mock('./mailer', () => ({
  sendEmail: vi.fn().mockResolvedValue(undefined),
}));

describe('registerUser', () => {
  it('sends a welcome email', async () => {
    await registerUser('ada@x.com');
    // sendEmail is now the mock from the factory above.
    expect(sendEmail).toHaveBeenCalledWith('ada@x.com', 'Welcome!');
  });
});
```

**Hoisting gotcha:** both `vi.mock` and `jest.mock` are *hoisted to the top of the file* by the runner's transform, so they run **before** the imports — that is why the mock is in place by the time `userService` imports `mailer`. A consequence: the factory cannot reference outer variables that are not yet initialised. Vitest gives you `vi.hoisted(() => ...)` to define values that need to exist before the mock; Jest has the `mockName`/`__mocks__` conventions. If you need to *change* mock return values per test, grab the mocked function via `vi.mocked(sendEmail)` and reprogram it in `beforeEach`.

```ts
import { vi } from 'vitest';
// Access the mock with full types to reprogram it per-test:
const mockedSend = vi.mocked(sendEmail);
// beforeEach(() => mockedSend.mockResolvedValue(undefined));
```

### 5.5 ESM mocking caveats **[A]**

Mocking is harder under native **ES Modules** than under CommonJS, and this trips up everyone eventually. In CommonJS, `require` returns a mutable object, so a mock can swap properties on it after the fact. In native ESM, imports are **live read-only bindings** resolved by the spec — you cannot just reassign `mailer.sendEmail` from outside. This is why:

- **Vitest** handles ESM mocking well because it controls the module graph via Vite; `vi.mock` works for ESM as shown above. This is a major reason to prefer Vitest for modern ESM/TS projects.
- **Jest** historically needed `jest.unstable_mockModule` plus dynamic `import()` for true ESM; Jest 30 improved this but it is still fiddlier than Vitest.
- **`node:test`** has a built-in `mock.module()` (still stabilising) but the cleanest path with plain Node ESM is often **not to module-mock at all** — inject dependencies instead (next section).

```ts
// node:test module mocking (experimental) — replace a module's named export.
import { mock } from 'node:test';
mock.module('./mailer.js', {
  namedExports: { sendEmail: mock.fn(async () => {}) },
});
```

> ⚡ **Version note:** ESM mocking APIs are the fastest-moving corner of this whole topic. `node:test`'s `mock.module` and Jest's ESM story both changed across 2024–2026 releases. If a module-mock snippet from an old blog post fails under native ESM, that is expected — check your runner's *current* docs, and strongly consider dependency injection (below), which sidesteps the whole problem.

### 5.6 Fake timers **[I/A]**

Code that uses `setTimeout`, `setInterval`, or reads `Date.now()` is *time-dependent*, which makes tests slow (waiting for real timers) and flaky (timing races). **Fake timers** replace the clock with one *you* control, so you can advance time instantly and deterministically.

```ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';

// Code under test: a debounce that fires the callback 300ms after the last call.
function debounce(fn: () => void, ms: number) {
  let t: ReturnType<typeof setTimeout>;
  return () => { clearTimeout(t); t = setTimeout(fn, ms); };
}

describe('debounce', () => {
  beforeEach(() => vi.useFakeTimers());   // swap in the fake clock
  afterEach(() => vi.useRealTimers());    // ALWAYS restore, or other tests break

  it('fires once after the delay, no real waiting', () => {
    const spy = vi.fn();
    const debounced = debounce(spy, 300);

    debounced();
    debounced();                 // resets the timer
    expect(spy).not.toHaveBeenCalled();

    vi.advanceTimersByTime(300); // jump forward instantly — no real 300ms wait
    expect(spy).toHaveBeenCalledTimes(1);
  });

  it('controls the current date too', () => {
    vi.setSystemTime(new Date('2026-01-01T00:00:00Z'));
    expect(new Date().getUTCFullYear()).toBe(2026); // Date.now() is now frozen
  });
});
```

Useful controls: `advanceTimersByTime(ms)`, `runAllTimers()` (flush every pending timer), `runOnlyPendingTimers()`, `setSystemTime(date)` (freeze `Date`), and in Jest the identical `jest.useFakeTimers()` / `jest.advanceTimersByTime()`. The one iron rule: **restore real timers in `afterEach`**, or a fake clock leaks into unrelated tests and produces baffling hangs.

### 5.7 Mocking fetch and HTTP — nock and MSW **[I/A]**

Your code calls out over HTTP; your tests must not. Two dominant approaches:

**MSW (Mock Service Worker)** — the modern favourite. It intercepts at the network layer by defining *request handlers*, so your code uses the real `fetch`/axios unchanged and MSW answers. The same handlers work in Node tests and in the browser, which is a big win.

```ts
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';
import { beforeAll, afterEach, afterAll, it, expect } from 'vitest';

// Declare how the fake server responds to specific requests.
const server = setupServer(
  http.get('https://api.example.com/users/:id', ({ params }) =>
    HttpResponse.json({ id: params.id, name: 'Ada' }),
  ),
);

beforeAll(() => server.listen({ onUnhandledRequest: 'error' })); // fail on stray calls
afterEach(() => server.resetHandlers());  // undo per-test overrides
afterAll(() => server.close());

it('parses the user from the API', async () => {
  const res = await fetch('https://api.example.com/users/7');
  const user = await res.json();
  expect(user).toEqual({ id: '7', name: 'Ada' });
});
```

**nock** — the older, HTTP-interceptor approach specific to Node. It patches Node's `http`/`https` and matches requests you declare. Still widely used, but MSW's shared browser/Node handlers and cleaner API have made it the default recommendation for new code.

```ts
import nock from 'nock';
import { it, expect, afterEach } from 'vitest';

afterEach(() => nock.cleanAll());

it('mocks an HTTP GET with nock', async () => {
  nock('https://api.example.com').get('/ping').reply(200, { ok: true });
  const res = await fetch('https://api.example.com/ping');
  expect(await res.json()).toEqual({ ok: true });
});
```

Set MSW's `onUnhandledRequest: 'error'` (or nock's `disableNetConnect()`) so that any *accidental* real network call in a test fails loudly rather than silently reaching the internet.

### 5.8 Dependency injection — the cleaner alternative **[I/A]**

Module mocking and monkey-patching are powerful but fragile: they couple your test to *how* a module is imported, break under ESM, and reset globally. The structurally cleaner technique is **dependency injection (DI)** — instead of a function reaching out to grab its dependencies, you *pass them in*. Then a test just passes a fake; no mocking machinery at all.

```ts
// HARD to test: reaches out and grabs concrete dependencies itself.
import { db } from './db';
import { clock } from './clock';
export async function createOrderHard(items: Item[]) {
  return db.orders.insert({ items, createdAt: clock.now() });
}

// EASY to test: dependencies are parameters (injected).
interface Deps { insertOrder(o: Order): Promise<Order>; now(): Date; }

export async function createOrder(items: Item[], deps: Deps): Promise<Order> {
  return deps.insertOrder({ items, createdAt: deps.now() });
}
```

```ts
import { it, expect, vi } from 'vitest';

it('creates an order with a frozen time — no mocks needed', async () => {
  const fakeDeps = {
    insertOrder: vi.fn(async (o) => ({ ...o, id: 'order_1' })),
    now: () => new Date('2026-07-17T00:00:00Z'), // deterministic clock, injected
  };

  const order = await createOrder([{ sku: 'a' }], fakeDeps);

  expect(order.id).toBe('order_1');
  expect(fakeDeps.insertOrder).toHaveBeenCalledOnce();
  expect(order.createdAt.toISOString()).toBe('2026-07-17T00:00:00.000Z');
});
```

Notice there is no `vi.mock`, no hoisting, no ESM headache — just a plain object passed in. This is exactly what NestJS's DI container gives you for free (§10). DI is why NestJS/Angular code is so testable: every dependency is already a constructor parameter you can override. When you can choose the design, prefer DI over module mocking.

### 5.9 sinon, and the vi/jest cheat-sheet **[I]**

**sinon** is a standalone, framework-agnostic library for spies, stubs, and fake timers that predates the built-in mocking of Jest/Vitest. You will still meet it in older codebases and in Mocha-based suites (Mocha has no built-in mocking). It works the same conceptually — `sinon.stub()`, `sinon.spy()`, `sinon.useFakeTimers()` — but if you are on Vitest or Jest you rarely need it; their built-ins cover the same ground. Know it exists so you can read it.

The Vitest↔Jest mapping, since they are nearly identical:

| Vitest | Jest | Purpose |
|---|---|---|
| `vi.fn()` | `jest.fn()` | Create a mock function |
| `vi.spyOn(o, 'm')` | `jest.spyOn(o, 'm')` | Spy on an existing method |
| `vi.mock('./mod')` | `jest.mock('./mod')` | Replace a module |
| `vi.mocked(x)` | `jest.mocked(x)` | Typed access to a mock |
| `vi.useFakeTimers()` | `jest.useFakeTimers()` | Fake clock |
| `vi.advanceTimersByTime(n)` | `jest.advanceTimersByTime(n)` | Advance fake clock |
| `vi.clearAllMocks()` | `jest.clearAllMocks()` | Reset call history |
| `vi.restoreAllMocks()` | `jest.restoreAllMocks()` | Restore spied originals |

### 5.10 When NOT to mock — the over-mocking warning **[I/A]**

The most damaging testing mistake is **over-mocking**: replacing so many dependencies that the test only exercises your mocks, not your code. A test where you mock the database, the HTTP client, the clock, *and* the collaborators, then assert that "the service called the mock with these args," proves only that *the test matches the current implementation* — it passes even when the real integration is broken, and it breaks every time you refactor internals. This is why the testing trophy (§1.4) pushes toward integration tests.

Guidelines for what to mock:

- **Mock the boundaries you cannot control or afford:** third-party network APIs, payment providers, email/SMS, the system clock, randomness, the filesystem when you need determinism.
- **Do NOT mock your own domain logic** just to isolate one function — if two of your modules always work together, test them together.
- **Prefer a real (or in-memory/containerised) dependency over a mock** when it is fast enough: a Testcontainers Postgres (§11) gives far more confidence than a hand-rolled DB mock, and catches real SQL bugs.
- **If a test needs five mocks to set up, that is a design smell** — the unit under test probably has too many responsibilities or dependencies. Fix the design (often with DI), do not pile on more mocks.

A useful rule of thumb: *mock across process boundaries, use the real thing within them.*

---

## 6. TypeScript Testing

### 6.1 Why TypeScript changes testing **[I]**

TypeScript's type checker is, in the testing-trophy sense, your **cheapest and largest layer of tests** — it catches whole categories of bugs (wrong argument types, null access, typos in property names, mis-shaped objects) before any test runs, across your *entire* codebase, for free. So "testing in TypeScript" is two jobs: (1) run your normal runtime tests, but written in `.ts`, and (2) treat type-checking itself as a required CI gate. Do not let one substitute for the other — types prove shapes, tests prove behaviour.

### 6.2 Running TypeScript tests — the three transform strategies **[I]**

Your test files are `.ts`, but Node executes JavaScript. Something must transform the TypeScript first. There are three strategies, and which runner you use largely decides for you.

- **Vitest — native, zero config.** Vitest transforms `.ts` (and `.tsx`) through Vite/esbuild automatically. You write `foo.test.ts`, run `vitest`, and it just works. This is the least-friction option and a primary reason to choose Vitest for TS projects.
- **Jest — `ts-jest` or Babel.** Jest needs a transform. `ts-jest` runs the TypeScript compiler (slower, but *type-checks* as it transforms — it can fail a test file that does not type-check). `@babel/preset-typescript` (via `babel-jest`) is faster but **strips types without checking them**, so it will happily run type-invalid code — you must run `tsc --noEmit` separately.
- **`node:test` — `tsx` or Node's native stripping.** Run your tests through **`tsx`** (`tsx --test`) which transpiles TS on the fly, or use Node's built-in type-stripping (Node 22.6+ behind `--experimental-strip-types`, stabilising in 24) which *erases* types without checking them.

```bash
# Vitest — nothing extra:
npx vitest run

# Jest with ts-jest — install and configure a preset:
npm i -D jest ts-jest @types/jest
npx ts-jest config:init        # writes a jest.config with the ts-jest transform

# node:test through tsx (transpiles TS, no type-checking):
npm i -D tsx
npx tsx --test              # discovers and runs *.test.ts

# node:test with Node's native type stripping (Node 22.6+ / stable in 24):
node --experimental-strip-types --test
```

```js
// jest.config.js using ts-jest
/** @type {import('jest').Config} */
module.exports = {
  testEnvironment: 'node',
  preset: 'ts-jest',       // transform *.ts with the TypeScript compiler
};
```

> ⚡ **Version note:** Node's built-in TypeScript support (type *stripping*, not type *checking*) moved from `--experimental-strip-types` toward on-by-default across Node 22→24. It erases annotations and cannot run features that need type information to emit code (certain `enum`s, `namespace`s, `experimentalDecorators` in some configs). For NestJS (heavy decorators/metadata) and anything needing emit, use `ts-jest`, `tsx`, or Vitest with the SWC plugin rather than raw stripping.

### 6.3 Types in tests — catching bugs the compiler can catch **[I]**

Write your tests in TypeScript so the compiler checks *them* too. Typed mocks (`vi.mocked`, `jest.mocked`) ensure your fake matches the real signature — if the real function grows a parameter, the typed mock flags the test.

```ts
import { describe, it, expect, vi } from 'vitest';

interface PaymentGateway {
  charge(amountCents: number, token: string): Promise<{ id: string }>;
}

async function payInvoice(gw: PaymentGateway, cents: number, token: string) {
  const { id } = await gw.charge(cents, token);
  return `receipt_${id}`;
}

describe('payInvoice', () => {
  it('returns a receipt id', async () => {
    // A fully-typed fake — TypeScript enforces it matches PaymentGateway.
    const gateway: PaymentGateway = {
      charge: vi.fn().mockResolvedValue({ id: 'ch_123' }),
    };

    const receipt = await payInvoice(gateway, 500, 'tok_abc');

    expect(receipt).toBe('receipt_ch_123');
    // vi.mocked gives typed access to the mock's assertions:
    expect(vi.mocked(gateway.charge)).toHaveBeenCalledWith(500, 'tok_abc');
  });
});
```

### 6.4 Type-level testing — asserting on the types themselves **[A]**

Sometimes the *type* is the thing you want to test — a generic utility, a conditional type, an inferred return. Vitest ships `expectTypeOf` / `assertType` for **type-level assertions** that are checked at compile time (they run no runtime code). Libraries like `tsd` and `expect-type` do the same standalone.

```ts
import { expectTypeOf, it } from 'vitest';

type NonEmpty<T> = [T, ...T[]];

function first<T>(list: NonEmpty<T>): T {
  return list[0];
}

it('infers the element type', () => {
  // These assertions are verified by `tsc`, not at runtime.
  expectTypeOf(first([1, 2, 3])).toEqualTypeOf<number>();
  expectTypeOf(first(['a'])).toEqualTypeOf<string>();
  // @ts-expect-error — an empty array is not assignable to NonEmpty<T>
  first([]);
});
```

Run these with `vitest --typecheck` (which invokes `tsc` under the hood for the type assertions). This is niche — mostly for library authors publishing types — but invaluable there.

### 6.5 Type-checking as a CI gate **[I]**

However you transform tests, add a **separate, mandatory `tsc --noEmit` step** to CI. Runners that strip types (Babel-Jest, `tsx`, native stripping) do *not* fail on type errors, so without this step a type-invalid codebase can go green. Vitest and `ts-jest` catch more, but a dedicated `tsc` pass over the whole project (source *and* tests) is the authoritative check.

```json
// package.json
{
  "scripts": {
    "typecheck": "tsc --noEmit",           // fails the build on any type error
    "test": "vitest run",
    "ci": "npm run typecheck && npm run test"
  }
}
```

Treat `typecheck` as non-negotiable in CI (§18). It is faster than the test suite and catches a class of bugs no runtime test will.

---

## 7. Coverage and Watch Mode

### 7.1 What coverage is and what it is not **[I]**

**Code coverage** measures how much of your source code was *executed* while the tests ran. A coverage tool instruments (or observes) your code, runs the suite, and reports the percentage of lines, statements, branches, and functions that were touched. It is reported in four dimensions:

| Metric | Question it answers |
|---|---|
| **Statements** | What fraction of statements ran? |
| **Branches** | For each `if`/`?:`/`&&`, did both the true and false paths run? |
| **Functions** | What fraction of functions were called? |
| **Lines** | What fraction of source lines ran? |

**Branch coverage is the one that matters most** — it is easy to hit 100% line coverage while never testing the `else` path where the bugs hide. The crucial mental correction for beginners: **coverage measures what was *executed*, not what was *verified*.** A test that calls a function but asserts nothing gives 100% coverage and zero confidence. High coverage is necessary-ish but wildly insufficient; it tells you what you *definitely have not* tested (the red lines), which is its real value. Treat low coverage as a red flag and high coverage as a *non-guarantee*, not a trophy.

### 7.2 The two engines — V8 and Istanbul **[I]**

There are two ways to measure coverage in Node:

- **V8 coverage** (used by **c8**, and Vitest's default provider) reads coverage data directly from the V8 JavaScript engine's built-in instrumentation. It is **fast** (no source transformation) and needs no build step, which is why it is the modern default. Historically its source-mapping to original TS was slightly less precise, but that gap has largely closed.
- **Istanbul** instruments your source by rewriting it with counters before running. It is **more mature and precise** for branch reporting and has the richest ecosystem (`nyc` is its CLI), at the cost of a transform step and some speed.

For most projects, V8/c8 is the right default. Choose Istanbul if you need its extra precision or specific reporters.

### 7.3 Running coverage in each runner **[I]**

```bash
# node:test — built-in V8 coverage:
node --test --experimental-test-coverage

# c8 wrapping any command (great for node:test or plain scripts):
npx c8 --reporter=text --reporter=html node --test

# Vitest — pick a provider (v8 is default; 'istanbul' also available):
npx vitest run --coverage

# Jest — Istanbul-based, built in:
npx jest --coverage
```

```ts
// vitest.config.ts — coverage configuration with thresholds
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',                        // or 'istanbul'
      reporter: ['text', 'html', 'lcov'],    // console + browsable HTML + CI format
      include: ['src/**/*.ts'],
      exclude: ['**/*.test.ts', 'src/types/**', 'src/**/index.ts'],
      thresholds: {                          // FAIL the run if coverage drops below:
        statements: 80,
        branches: 75,
        functions: 80,
        lines: 80,
      },
    },
  },
});
```

The `lcov` reporter emits a file that services like Codecov and most CI dashboards ingest; the `html` reporter produces a clickable report where uncovered lines are highlighted red — open `coverage/index.html` and *look at the red lines*, that is the actual workflow.

### 7.4 Thresholds — what target to set and how **[I]**

A coverage **threshold** fails the build when coverage drops below a number, preventing new untested code from sneaking in. Advice for setting them sanely:

- **Do not chase 100%.** The last 10% is usually error-handling and defensive branches that cost more to test than they are worth; mandating 100% pushes people to write assertion-free "coverage tests." A pragmatic target is **70–85% branches** for most backends, higher for critical libraries.
- **Ratchet, don't mandate overnight.** On a legacy codebase, set the threshold to *current* coverage and forbid it going down. Coverage then only ever rises.
- **Measure the right files.** `exclude` generated code, type-only files, config, and test files themselves, or your denominator is polluted and the number is meaningless.
- **Per-file thresholds** catch the case where a well-tested module's high coverage masks a completely untested new module in the same project.

### 7.5 Watch mode — the inner development loop **[I]**

**Watch mode** re-runs affected tests automatically whenever you save a file, giving sub-second feedback as you code — it is how you *actually* run tests during development (you run the full suite in CI). Vitest's watch mode is best-in-class: it uses the module graph to re-run only the tests affected by the file you changed.

```bash
vitest                         # watch is the DEFAULT (use `vitest run` for one-shot)
node --test --watch            # node:test watch
jest --watch                   # only tests related to changed files (needs git)
jest --watchAll                # all tests on any change
```

In an interactive Vitest/Jest watch session you can press keys to filter: `p` to filter by filename, `t` to filter by test-name pattern, `f` to re-run only failures. The workflow that makes TDD pleasant: put a `.only` or a name filter on the test you are working on, keep watch running on a second monitor, and let it turn green as you type.

---

## 8. Snapshot Testing

### 8.1 What snapshots are and the one honest use for them **[I]**

A **snapshot test** serialises a value to a string, saves it to a file the first time it runs (the "snapshot"), and on every later run compares the fresh value against the saved one — failing if they differ. The appeal: you assert on a large, complex output (a rendered component, a big config object, an error message) without hand-writing every field. The first run *records*; subsequent runs *guard against change*.

```ts
import { it, expect } from 'vitest';

function buildInvoice(items: { name: string; cents: number }[]) {
  const subtotal = items.reduce((s, i) => s + i.cents, 0);
  return { items, subtotal, tax: Math.round(subtotal * 0.2), currency: 'USD' };
}

it('matches the invoice snapshot', () => {
  const invoice = buildInvoice([{ name: 'Book', cents: 1500 }]);
  // First run: writes the snapshot to __snapshots__/*.snap and passes.
  // Later runs: fails if `invoice` serialises differently.
  expect(invoice).toMatchSnapshot();
});
```

When `buildInvoice` legitimately changes, you review the diff and run with `--update` (`vitest -u`, `jest -u`) to re-record. `node:test` supports snapshots too via `--experimental-test-snapshots` / `t.assert.snapshot`.

### 8.2 Inline snapshots — the better default **[I]**

**Inline snapshots** write the expected value *into the test file itself* rather than a separate `.snap` file. This is usually superior because the expectation is visible right there in the test — no jumping to a snapshot file, and diffs show up in code review.

```ts
it('formats an inline snapshot', () => {
  const invoice = buildInvoice([{ name: 'Book', cents: 1500 }]);
  // The runner FILLS IN the argument automatically on first run / with -u:
  expect(invoice).toMatchInlineSnapshot(`
    {
      "currency": "USD",
      "items": [ { "cents": 1500, "name": "Book" } ],
      "subtotal": 1500,
      "tax": 300,
    }
  `);
});
```

### 8.3 The misuse — why snapshots get a bad name **[I]**

Snapshots are widely *abused*, and an abused snapshot suite is worse than no tests:

- **Rubber-stamping updates.** When a snapshot fails, the lazy fix is `-u` to accept the new value without reading the diff. Do that reflexively and the snapshot guards nothing — it just records whatever the code currently does, bugs included.
- **Huge, opaque snapshots.** A 500-line serialised DOM tells you *nothing* on failure except "something changed somewhere." Nobody reviews it.
- **Non-deterministic content.** Timestamps, random IDs, and ordering make snapshots fail spuriously. Use property matchers (`expect.any(Date)`) or normalise the value before snapshotting.

**Use snapshots when:** the output is large but *stable*, you would otherwise hand-write dozens of brittle field assertions, and a human will actually review update diffs. **Prefer explicit assertions when:** you care about a *specific* property (assert exactly that), or the value is small enough to write out. A good habit: snapshot the *shape*, assert explicitly on the *values that matter*.

```ts
it('snapshots with volatile fields masked', () => {
  const user = { id: crypto.randomUUID(), name: 'Ada', createdAt: new Date() };
  // Match structure but do not pin the volatile fields:
  expect(user).toMatchSnapshot({
    id: expect.any(String),
    createdAt: expect.any(Date),
  });
});
```

---

## 9. HTTP and API Integration with Supertest

### 9.1 What Supertest is and why in-process testing wins **[I]**

**Supertest** lets you make HTTP requests against an Express/Fastify/Nest app and assert on the response — status, headers, JSON body — *without starting a real network server on a port*. You hand it your app object; it boots the app on an ephemeral port (or in-process), fires the request through the real routing/middleware stack, and gives you the response to assert on. This is the heart of the testing trophy's integration layer for a backend: it exercises your routes, middleware, validation, serialization, and error handling *together*, exactly as a real client would, but in-process so it is fast and needs no running server.

```bash
npm i express
npm i -D supertest @types/supertest
```

### 9.2 A testable Express app **[I]**

The key structural rule: **export your `app` without calling `app.listen`.** Separate "define the app" from "start the server" so tests can import the app and Supertest can drive it, while production still calls `listen`.

```ts
// app.ts — defines the app but does NOT listen. This is what tests import.
import express, { type Request, type Response, type NextFunction } from 'express';

interface User { id: number; name: string; }
const users: User[] = [{ id: 1, name: 'Ada' }];

export function createApp() {
  const app = express();
  app.use(express.json()); // parse JSON request bodies

  app.get('/health', (_req, res) => res.status(200).json({ ok: true }));

  app.get('/users/:id', (req, res) => {
    const user = users.find((u) => u.id === Number(req.params.id));
    if (!user) return res.status(404).json({ error: 'Not found' });
    res.json(user);
  });

  app.post('/users', (req, res) => {
    const { name } = req.body as { name?: unknown };
    if (typeof name !== 'string' || name.trim() === '') {
      return res.status(422).json({ error: 'name is required' });
    }
    const user = { id: users.length + 1, name };
    users.push(user);
    res.status(201).location(`/users/${user.id}`).json(user);
  });

  // Centralised error handler (4 args marks it as an error middleware).
  app.use((err: Error, _req: Request, res: Response, _next: NextFunction) => {
    res.status(500).json({ error: err.message });
  });

  return app;
}
```

```ts
// server.ts — the production entry point actually listens.
import { createApp } from './app';
createApp().listen(3000, () => console.log('listening on :3000'));
```

### 9.3 Asserting status, JSON and headers **[I]**

```ts
// app.test.ts
import request from 'supertest';
import { describe, it, expect } from 'vitest';
import { createApp } from './app';

const app = createApp(); // a fresh in-process app; no listen() needed

describe('GET /health', () => {
  it('returns 200 and ok true', async () => {
    const res = await request(app).get('/health');
    expect(res.status).toBe(200);
    expect(res.body).toEqual({ ok: true });
  });
});

describe('GET /users/:id', () => {
  it('returns the user as JSON with the right content-type', async () => {
    const res = await request(app).get('/users/1');
    expect(res.status).toBe(200);
    expect(res.headers['content-type']).toMatch(/application\/json/);
    expect(res.body).toMatchObject({ id: 1, name: 'Ada' });
  });

  it('returns 404 for an unknown id', async () => {
    const res = await request(app).get('/users/999');
    expect(res.status).toBe(404);
    expect(res.body).toEqual({ error: 'Not found' });
  });
});

describe('POST /users', () => {
  it('creates a user and returns 201 with a Location header', async () => {
    const res = await request(app)
      .post('/users')
      .send({ name: 'Grace' })            // JSON body
      .set('Accept', 'application/json'); // request header

    expect(res.status).toBe(201);
    expect(res.headers.location).toBe('/users/2');
    expect(res.body).toMatchObject({ name: 'Grace' });
  });

  it('rejects a missing name with 422', async () => {
    const res = await request(app).post('/users').send({});
    expect(res.status).toBe(422);
    expect(res.body.error).toMatch(/name/);
  });
});
```

Supertest also has its own chainable expectations (`.expect(200)`, `.expect('Content-Type', /json/)`) that throw if unmet — handy but I recommend awaiting the response and using your runner's `expect` so the diffs and matchers are consistent with the rest of your suite.

### 9.4 Testing authentication flows **[I]**

Real APIs are protected. To test authenticated routes you either (a) log in through the real flow and reuse the returned token/cookie, or (b) mint a valid token directly for the test. Option (a) is higher-confidence; (b) is faster for routes where auth is not the thing under test.

```ts
// auth.test.ts — a token-based (JWT) auth flow
import request from 'supertest';
import { describe, it, expect, beforeAll } from 'vitest';
import { createApp } from './app-with-auth';

const app = createApp();
let token: string;

beforeAll(async () => {
  // (a) Log in through the REAL endpoint and capture the token.
  const res = await request(app)
    .post('/auth/login')
    .send({ email: 'ada@x.com', password: 'correct-horse' });
  expect(res.status).toBe(200);
  token = res.body.accessToken;
});

describe('GET /me (protected)', () => {
  it('401s without a token', async () => {
    const res = await request(app).get('/me');
    expect(res.status).toBe(401);
  });

  it('returns the profile with a valid bearer token', async () => {
    const res = await request(app)
      .get('/me')
      .set('Authorization', `Bearer ${token}`); // attach the credential
    expect(res.status).toBe(200);
    expect(res.body.email).toBe('ada@x.com');
  });
});
```

For **cookie/session** auth, use Supertest's `agent`, which persists cookies across requests just like a browser:

```ts
const agent = request.agent(app);              // remembers Set-Cookie between calls
await agent.post('/auth/login').send({ email: 'ada@x.com', password: 'x' });
const res = await agent.get('/me');            // the session cookie rides along
expect(res.status).toBe(200);
```

For login flows in NestJS the mechanics are the same — you just build the app through `@nestjs/testing` instead of `createApp` (§10.5). And when the protected routes touch a database, combine this with Testcontainers (§11) so you are asserting against a real datastore rather than a mock.

---

## 10. NestJS Testing

### 10.1 Why NestJS is a joy to test **[I]**

NestJS is built on **dependency injection** (§5.8): every service, repository, and controller declares its dependencies as constructor parameters, and a DI container wires them at runtime. That architecture is *exactly* what makes code testable — in a test you construct a **testing module** and tell the container "when someone asks for `MailService`, give them *this fake* instead." No `vi.mock`, no monkey-patching, no ESM headaches — you override providers cleanly through the framework. `@nestjs/testing`'s `Test.createTestingModule` is the tool for this. NestJS testing splits into two clear layers: **unit tests** (a single service/controller with its dependencies faked) and **e2e tests** (the whole application bootstrapped and driven with Supertest).

```bash
npm i -D @nestjs/testing supertest @types/supertest
```

### 10.2 Unit-testing a service with overridden providers **[I/A]**

Consider a `UsersService` that depends on a `UsersRepository`. To unit-test the service's *logic*, we replace the repository with a fake so no database is touched.

```ts
// users.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';

export interface User { id: number; name: string; email: string; }

// The dependency, expressed as an injectable class (its real impl hits a DB).
@Injectable()
export class UsersRepository {
  async findById(_id: number): Promise<User | null> { throw new Error('real DB'); }
  async create(_data: Omit<User, 'id'>): Promise<User> { throw new Error('real DB'); }
}

@Injectable()
export class UsersService {
  constructor(private readonly repo: UsersRepository) {} // injected dependency

  async getUser(id: number): Promise<User> {
    const user = await this.repo.findById(id);
    if (!user) throw new NotFoundException(`User ${id} not found`);
    return user;
  }

  async register(name: string, email: string): Promise<User> {
    return this.repo.create({ name, email });
  }
}
```

```ts
// users.service.spec.ts
import { Test, type TestingModule } from '@nestjs/testing';
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { UsersService, UsersRepository, type User } from './users.service';

describe('UsersService', () => {
  let service: UsersService;
  // A fully-typed fake repository — just an object of mock functions.
  const repo = {
    findById: vi.fn<(id: number) => Promise<User | null>>(),
    create: vi.fn<(d: Omit<User, 'id'>) => Promise<User>>(),
  };

  beforeEach(async () => {
    vi.clearAllMocks(); // fresh call history each test — critical for isolation

    // Build a testing module and OVERRIDE the real repository with our fake.
    const moduleRef: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        { provide: UsersRepository, useValue: repo }, // <-- swap the provider
      ],
    }).compile();

    service = moduleRef.get(UsersService);
  });

  it('returns the user when found', async () => {
    repo.findById.mockResolvedValue({ id: 1, name: 'Ada', email: 'ada@x.com' });
    await expect(service.getUser(1)).resolves.toMatchObject({ name: 'Ada' });
    expect(repo.findById).toHaveBeenCalledWith(1);
  });

  it('throws NotFoundException when the user is missing', async () => {
    repo.findById.mockResolvedValue(null);
    await expect(service.getUser(42)).rejects.toThrow('User 42 not found');
  });

  it('creates a user through the repository', async () => {
    repo.create.mockResolvedValue({ id: 5, name: 'Grace', email: 'g@x.com' });
    const user = await service.register('Grace', 'g@x.com');
    expect(user.id).toBe(5);
    expect(repo.create).toHaveBeenCalledWith({ name: 'Grace', email: 'g@x.com' });
  });
});
```

The pattern to internalise: `providers: [RealThingUnderTest, { provide: Dependency, useValue: fake }]`. You can also override with `useClass` (a fake class) or `useFactory` (a computed fake). For providers registered by token (e.g. a repository injected via `@InjectRepository(User)`), override by the *same token*: `{ provide: getRepositoryToken(User), useValue: fakeRepo }`.

### 10.3 Unit-testing a controller **[I]**

Controllers are thin — they translate HTTP to service calls. Test them the same way, faking the service, to verify they call the right method and shape the response. Alternatively, and often better, skip controller unit tests entirely and cover controllers via e2e tests (§10.5), which exercise the routing and pipes too.

```ts
// users.controller.spec.ts
import { Test } from '@nestjs/testing';
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

describe('UsersController', () => {
  let controller: UsersController;
  const service = { getUser: vi.fn() };

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      controllers: [UsersController],
      providers: [{ provide: UsersService, useValue: service }],
    }).compile();
    controller = moduleRef.get(UsersController);
  });

  it('delegates GET /users/:id to the service', async () => {
    service.getUser.mockResolvedValue({ id: 1, name: 'Ada' });
    await expect(controller.findOne(1)).resolves.toMatchObject({ name: 'Ada' });
    expect(service.getUser).toHaveBeenCalledWith(1);
  });
});
```

### 10.4 Overriding guards, interceptors and pipes **[I/A]**

For unit tests you often want to *disable* an auth guard so you can test the handler logic without a real token. The testing module builder supports `overrideGuard`, `overrideInterceptor`, `overridePipe`, and `overrideProvider`:

```ts
const moduleRef = await Test.createTestingModule({ /* ... */ })
  .overrideGuard(AuthGuard)                     // replace the real guard...
  .useValue({ canActivate: () => true })        // ...with one that always allows
  .compile();
```

In an e2e test you usually do the opposite — keep the real guard and supply a real token — so the auth path is genuinely covered.

### 10.5 End-to-end tests with a real Nest app and Supertest **[A]**

An e2e test boots the *entire* application — real modules, real pipes, real guards, real validation — and drives it over HTTP with Supertest. This is the highest-confidence backend test: it proves the wiring works end to end. The only thing you typically fake is truly external I/O (payment, email); the database is best made real via Testcontainers (§11).

```ts
// app.e2e-spec.ts
import { Test } from '@nestjs/testing';
import { type INestApplication, ValidationPipe } from '@nestjs/common';
import request from 'supertest';
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { AppModule } from '../src/app.module';

describe('Users (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    // Compile the WHOLE application graph, exactly like production bootstrap.
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleRef.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true })); // mirror main.ts
    await app.init(); // initialise but do NOT listen on a port — Supertest handles it
  });

  afterAll(async () => {
    await app.close(); // release DB pools, timers, handles — prevents hanging CI
  });

  it('POST /users validates the body (422 on bad input)', async () => {
    const res = await request(app.getHttpServer()) // the underlying http.Server
      .post('/users')
      .send({ name: '' }); // invalid per the DTO
    expect(res.status).toBe(400); // ValidationPipe rejects with 400 by default
  });

  it('POST then GET round-trips a user', async () => {
    const created = await request(app.getHttpServer())
      .post('/users')
      .send({ name: 'Ada', email: 'ada@x.com' })
      .expect(201);

    const fetched = await request(app.getHttpServer())
      .get(`/users/${created.body.id}`)
      .expect(200);

    expect(fetched.body).toMatchObject({ name: 'Ada', email: 'ada@x.com' });
  });
});
```

Two things that bite people in Nest e2e tests: **always `await app.close()`** in `afterAll` (otherwise open DB connections and timers keep the process alive and CI hangs — see §19), and **mirror your `main.ts` setup** (global pipes, prefixes, filters) in the test bootstrap, or the app under test behaves differently from production. For the GraphQL variant, drive `POST /graphql` with query strings via Supertest — see the testing section of the [NestJS + GraphQL](NESTJS_GRAPHQL_GUIDE.md) guide.

---

## 11. Database Integration with Testcontainers

### 11.1 Why a real database beats a mocked one **[A]**

Mocking a database means writing a fake that returns canned rows — which tests your *assumptions* about the database, not the database. It cannot catch a malformed SQL query, a wrong migration, a missing index causing a constraint violation, a JSON-column serialization quirk, or a transaction-isolation bug. Those are exactly the bugs that reach production. The high-confidence alternative is to run your tests against a **real Postgres** — and **Testcontainers** makes that painless by spinning up a throwaway database in a Docker container from within your test suite, then throwing it away afterwards. You get real SQL, real constraints, real behaviour, fully isolated and reproducible on any machine (and in CI) that has Docker.

```bash
npm i -D @testcontainers/postgresql testcontainers
# Requires a running Docker daemon (Docker Desktop on Windows/macOS).
```

### 11.2 Spinning up Postgres for the suite **[A]**

Start one container for the whole test file in `beforeAll` (starting a container per test would be far too slow), and stop it in `afterAll`.

```ts
// db.int-test.ts
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { Pool } from 'pg';
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest';

describe('user repository (real Postgres)', () => {
  let container: StartedPostgreSqlContainer;
  let pool: Pool;

  beforeAll(async () => {
    // Boot a disposable Postgres. First run pulls the image; later runs are fast.
    container = await new PostgreSqlContainer('postgres:17-alpine')
      .withDatabase('testdb')
      .withUsername('test')
      .withPassword('test')
      .start();

    // Connect using the container's dynamically-assigned host/port.
    pool = new Pool({ connectionString: container.getConnectionUri() });

    // Apply the schema (in a real app, run your migration tool here instead).
    await pool.query(`
      CREATE TABLE users (
        id    SERIAL PRIMARY KEY,
        name  TEXT NOT NULL,
        email TEXT NOT NULL UNIQUE
      );
    `);
  }, 120_000); // generous timeout — pulling an image can be slow the first time

  afterAll(async () => {
    await pool?.end();        // close the connection pool...
    await container?.stop();  // ...then destroy the container
  });

  // ... tests below (see the isolation strategies next) ...

  it('enforces the unique email constraint', async () => {
    await pool.query(`INSERT INTO users (name, email) VALUES ('Ada', 'a@x.com')`);
    // The REAL constraint fires — a mock could never catch this.
    await expect(
      pool.query(`INSERT INTO users (name, email) VALUES ('Ada2', 'a@x.com')`),
    ).rejects.toThrow(/duplicate key value/);
  });
});
```

### 11.3 Per-test isolation — truncate vs transaction rollback **[A]**

Tests share the one container, so each test must start from a known, clean state or they pollute each other. Two strategies:

**Truncate between tests** (simple, robust). In `beforeEach`, wipe the tables. `TRUNCATE ... RESTART IDENTITY CASCADE` clears rows and resets sequences so IDs are predictable.

```ts
beforeEach(async () => {
  await pool.query('TRUNCATE TABLE users RESTART IDENTITY CASCADE');
});
```

**Transaction-per-test rollback** (fast, elegant). Begin a transaction in `beforeEach`, run the test inside it, and `ROLLBACK` in `afterEach` — every change vanishes, and nothing is ever committed. This is faster than truncating and keeps the schema untouched. The catch: your code under test must use the *same connection* that holds the transaction (inject the client), and code that itself issues `COMMIT`/`BEGIN` needs care (use savepoints).

```ts
import { afterEach, beforeEach } from 'vitest';
let client: import('pg').PoolClient;

beforeEach(async () => {
  client = await pool.connect();
  await client.query('BEGIN');    // open a transaction
  // Hand `client` to the repository under test so it uses THIS connection.
});

afterEach(async () => {
  await client.query('ROLLBACK'); // undo everything the test did
  client.release();               // return the connection to the pool
});
```

Truncate is the safer default; reach for transaction rollback when suite speed matters and you can thread the transactional client through your data layer.

### 11.4 Seeding and testing a repository layer **[A]**

Most tests need some baseline data (**seed** data). Keep seeding explicit and per-test so each test's assumptions are visible. Here is a repository under test with a helper that seeds and a test that verifies real query behaviour.

```ts
// A thin repository the app uses (constructor-injected pool = testable).
class UserRepository {
  constructor(private readonly db: Pool) {}
  async create(name: string, email: string) {
    const { rows } = await this.db.query(
      'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *',
      [name, email], // parameterised — never string-concatenate SQL
    );
    return rows[0];
  }
  async findByEmail(email: string) {
    const { rows } = await this.db.query('SELECT * FROM users WHERE email = $1', [email]);
    return rows[0] ?? null;
  }
}

it('creates and finds a user by email', async () => {
  const repo = new UserRepository(pool);
  const created = await repo.create('Ada', 'ada@x.com');
  expect(created).toMatchObject({ id: expect.any(Number), name: 'Ada' });

  const found = await repo.findByEmail('ada@x.com');
  expect(found.id).toBe(created.id);        // round-trips through real Postgres
  expect(await repo.findByEmail('nobody@x.com')).toBeNull();
});
```

### 11.5 Testing Prisma and Drizzle layers **[A]**

The same container approach works for ORMs — point the ORM at the container's connection string and run its migrations against it.

```ts
// Prisma against a Testcontainers Postgres:
import { PrismaClient } from '@prisma/client';
import { execSync } from 'node:child_process';

let prisma: PrismaClient;

beforeAll(async () => {
  const container = await new PostgreSqlContainer('postgres:17-alpine').start();
  const url = container.getConnectionUri();
  // Point Prisma at the ephemeral DB and apply migrations to it.
  process.env.DATABASE_URL = url;
  execSync('npx prisma migrate deploy', { env: { ...process.env, DATABASE_URL: url } });
  prisma = new PrismaClient({ datasources: { db: { url } } });
}, 120_000);

// afterAll: await prisma.$disconnect(); await container.stop();
```

For **Drizzle**, do the same: create the pool from `container.getConnectionUri()`, run `migrate()` from `drizzle-orm/<driver>/migrator`, then exercise your queries. See the [Prisma](PRISMA_ORM_GUIDE.md) and [PostgreSQL](POSTGRESQL_GUIDE.md) guides for the schema/migration mechanics; the *testing* pattern is identical regardless of ORM. One performance tip for large suites: use Testcontainers' **reuse** feature or a **template database** (`CREATE DATABASE ... TEMPLATE`) so you migrate once and clone per file rather than re-migrating repeatedly.

---

## 12. End-to-End Browser Tests with Playwright

### 12.1 What E2E browser testing is and when to use it **[A]**

An **end-to-end browser test** drives a *real browser* (Chromium, Firefox, WebKit) through your *real, deployed-like application* the way a user would: it navigates to a URL, clicks buttons, fills forms, and asserts on what appears on screen. It is the top of the pyramid/trophy — the highest-confidence, slowest, most expensive test. Use it for the handful of **critical user journeys** whose breakage would be a disaster (login, checkout, the primary happy path), *not* for exhaustively covering every field and edge case (do that with faster tests below). **Playwright** is the modern standard: it is fast, reliable, auto-waits for elements (killing a whole class of flake), runs across all three browser engines, and ships superb debugging tools (trace viewer, codegen).

```bash
npm init playwright@latest      # scaffolds config, example tests, installs browsers
# or: npm i -D @playwright/test && npx playwright install
```

### 12.2 A login-to-dashboard flow **[A]**

Here is a canonical admin-dashboard journey: visit the login page, sign in, and assert the dashboard shows the expected data. Note the **user-facing selectors** (`getByRole`, `getByLabel`) — they resemble how a human finds elements, so they are robust to markup changes and encourage accessible HTML.

```ts
// e2e/login.spec.ts
import { test, expect } from '@playwright/test';

test.describe('admin dashboard', () => {
  test('logs in and sees the data table', async ({ page }) => {
    // 1. Navigate to the app (baseURL is set in the config, see 12.4).
    await page.goto('/login');

    // 2. Fill the form using ACCESSIBLE selectors (by label / role), not CSS.
    await page.getByLabel('Email').fill('admin@example.com');
    await page.getByLabel('Password').fill('correct-horse-battery');
    await page.getByRole('button', { name: 'Sign in' }).click();

    // 3. Playwright AUTO-WAITS for navigation and elements — no manual sleeps.
    await expect(page).toHaveURL(/.*\/dashboard/);
    await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible();

    // 4. Assert on real rendered data.
    const rows = page.getByRole('row');
    await expect(rows).toHaveCount(11);           // 10 data rows + 1 header
    await expect(page.getByText('Total users')).toBeVisible();
  });

  test('rejects bad credentials', async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('Email').fill('admin@example.com');
    await page.getByLabel('Password').fill('wrong');
    await page.getByRole('button', { name: 'Sign in' }).click();
    // The alert role is how a screen reader would find the error, too.
    await expect(page.getByRole('alert')).toContainText(/invalid credentials/i);
  });
});
```

### 12.3 Selectors, auto-waiting and avoiding flake **[A]**

Playwright's assertions are **auto-retrying**: `await expect(locator).toBeVisible()` polls until the condition holds or a timeout expires, so you almost never need `waitForTimeout` (a fixed sleep — the number-one cause of flaky E2E tests; ban it). Selector priority, best to worst:

| Preference | Selector | Why |
|---|---|---|
| Best | `getByRole('button', { name })` | Matches accessibility tree — robust and a11y-checking |
| Good | `getByLabel`, `getByPlaceholder`, `getByText` | User-visible, resilient |
| OK | `getByTestId('...')` (a `data-testid` attr) | Stable when semantics are absent |
| Avoid | `page.locator('.btn-primary > span:nth-child(2)')` | Breaks on any markup/style change |

Prefer role/label selectors; fall back to `data-testid` for elements with no accessible name; avoid brittle CSS/XPath chains.

### 12.4 Config, fixtures and reusing login **[A]**

The Playwright config sets the base URL, browsers, and can auto-start your dev server. Its **fixtures** system lets you share setup — most importantly, **authenticate once and reuse the storage state** so every test does not repeat the login (a big speed-up).

```ts
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  use: {
    baseURL: 'http://localhost:3000',   // so page.goto('/login') works
    trace: 'on-first-retry',            // capture a trace when a test retries (12.5)
  },
  // Boot the app before the suite and shut it down after — no manual server juggling.
  webServer: {
    command: 'npm run start',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
  ],
  retries: process.env.CI ? 2 : 0,      // retry flaky tests in CI only
});
```

```ts
// global-setup pattern: log in once, save the session, reuse it everywhere.
// auth.setup.ts
import { test as setup, expect } from '@playwright/test';

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('admin@example.com');
  await page.getByLabel('Password').fill('correct-horse-battery');
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page).toHaveURL(/.*\/dashboard/);
  // Persist cookies + localStorage so other tests start already logged in.
  await page.context().storageState({ path: 'playwright/.auth/admin.json' });
});
```

Then a project can declare `use: { storageState: 'playwright/.auth/admin.json' }`, and its tests begin authenticated — no repeated login.

### 12.5 Debugging — trace viewer and codegen **[A]**

Two tools make Playwright pleasant when a test fails in CI where you cannot watch it:

- **Trace viewer.** With `trace: 'on-first-retry'`, a failed retry records a full trace — a DOM snapshot at every step, network log, console, and screenshots. Open it with `npx playwright show-trace trace.zip` and *scrub through the test like a video*. This turns "it fails in CI and I can't reproduce" into a five-minute diagnosis.
- **Codegen.** `npx playwright codegen http://localhost:3000` opens a browser that *records your clicks as test code*, picking good selectors automatically. Great for scaffolding a new test, then hand-editing the assertions.

```bash
npx playwright test                       # run all E2E tests headless
npx playwright test --headed              # watch it in a real browser window
npx playwright test --debug               # step through with the inspector
npx playwright show-trace trace.zip       # post-mortem a failure's trace
```

> ⚡ **Version note:** Playwright ships frequent releases (1.5x in 2026) and bundles specific browser builds, so `npx playwright install` must be re-run after upgrading the package to fetch matching browsers — in CI, cache `~/.cache/ms-playwright` and run `install --with-deps`. Cypress is the main alternative; Playwright's multi-browser support, speed, and trace viewer have made it the default recommendation for new projects.

---

## 13. Property-Based Testing with fast-check

### 13.1 What property-based testing is and why it finds bugs you can't imagine **[A]**

Normal ("example-based") tests check specific inputs you thought of: `add(2, 3) === 5`. But you only test the cases you *imagine*, and bugs hide in the cases you *didn't* — the empty string, the negative zero, the 10,000-character unicode name, the array with a duplicate. **Property-based testing** flips this: instead of picking inputs, you state a **property** that must hold for *all* inputs ("for any two integers, `add(a, b) === add(b, a)`"), and the library — **fast-check** — generates hundreds of random inputs trying to *falsify* it. When it finds a failing case, it **shrinks** it: automatically simplifies the counterexample to the smallest input that still fails (e.g. from a giant random string down to `""`), handing you a minimal reproduction.

```bash
npm i -D fast-check
```

### 13.2 A real example — round-trip and invariant properties **[A]**

The two most useful property shapes are **round-trip** ("decode(encode(x)) === x for all x") and **invariants** ("the output always satisfies P"). Here we test a `slugify` and an `encode/decode` pair.

```ts
import { describe, it, expect } from 'vitest';
import fc from 'fast-check';

function slugify(s: string): string {
  return s.trim().toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-+|-+$/g, '');
}

describe('slugify properties', () => {
  it('never produces leading or trailing hyphens, for ANY string', () => {
    fc.assert(
      // fc.string() generates arbitrary strings; fast-check runs this ~100x.
      fc.property(fc.string(), (input) => {
        const slug = slugify(input);
        // The PROPERTY: whatever the input, the slug has no edge hyphens.
        expect(slug.startsWith('-')).toBe(false);
        expect(slug.endsWith('-')).toBe(false);
      }),
    );
  });

  it('is idempotent — slugifying a slug changes nothing', () => {
    fc.assert(
      fc.property(fc.string(), (input) => {
        const once = slugify(input);
        expect(slugify(once)).toBe(once); // applying twice === applying once
      }),
    );
  });
});

// Round-trip property: base64 encode then decode must return the original bytes.
describe('base64 round-trip', () => {
  it('decode(encode(x)) === x for any byte array', () => {
    fc.assert(
      fc.property(fc.uint8Array(), (bytes) => {
        const encoded = Buffer.from(bytes).toString('base64');
        const decoded = new Uint8Array(Buffer.from(encoded, 'base64'));
        expect(decoded).toEqual(bytes); // the invariant that must always hold
      }),
    );
  });
});
```

### 13.3 Generators, shrinking and reproducing failures **[A]**

fast-check ships **arbitraries** (generators) for every common type, which you compose to describe your input space precisely:

```ts
import fc from 'fast-check';

fc.integer({ min: 0, max: 100 });         // bounded ints
fc.string();                               // arbitrary strings (incl. unicode)
fc.array(fc.integer());                    // arrays of ints
fc.record({ name: fc.string(), age: fc.nat() }); // objects with a fixed shape
fc.constantFrom('GET', 'POST', 'PUT');     // pick from a set
fc.emailAddress();                          // realistic emails
fc.date();                                  // arbitrary dates

// Model-based / stateful example: an amount is always non-negative after clamping.
it('clamp never returns a negative', () => {
  fc.assert(fc.property(fc.integer(), (n) => Math.max(0, n) >= 0));
});
```

When a property fails, fast-check prints the **shrunk counterexample** and a **seed**. Because generation is seeded, you can reproduce the exact failing run deterministically — invaluable in CI:

```ts
// Reproduce a specific failure by pinning the seed/path fast-check reported:
fc.assert(fc.property(fc.string(), predicate), { seed: 42, path: '3:2', endOnFailure: true });
```

Property tests shine for pure logic with clear invariants: parsers, serializers, sorting, math, validation, state machines. They complement (not replace) example tests — keep a few readable example tests as documentation, and add properties to hunt the edge cases you would never enumerate by hand.

---

## 14. Benchmarking

### 14.1 What benchmarking is and how it differs from testing **[I]**

A **benchmark** measures *how fast* code runs, not whether it is correct. You reach for it when performance is a feature — a hot path, a serialization routine, a parser — and you want to (a) know its current cost and (b) catch performance *regressions* when someone's refactor accidentally makes it 5× slower. Benchmarking is subtle: JIT warm-up, garbage collection, and noise mean a naive `Date.now()` around a loop lies. Use a real benchmarking harness that runs many iterations, discards warm-up, and reports statistics. **Vitest** has `bench` built in (backed by **tinybench**); tinybench also works standalone.

### 14.2 vitest bench and tinybench **[I]**

```ts
// sort.bench.ts — run with: vitest bench
import { bench, describe } from 'vitest';

const data = Array.from({ length: 10_000 }, () => Math.floor(Math.random() * 1e6));

describe('sorting 10k numbers', () => {
  // Each bench runs many iterations; Vitest reports ops/sec and variance.
  bench('native Array.sort with comparator', () => {
    [...data].sort((a, b) => a - b);
  });

  bench('Array.sort default (lexicographic — WRONG for numbers, shown for contrast)', () => {
    [...data].sort();
  });
});
```

```ts
// Standalone tinybench — no runner needed:
import { Bench } from 'tinybench';

const bench = new Bench({ time: 500 }); // run each task for ~500ms
bench
  .add('JSON.parse', () => JSON.parse('{"a":1,"b":[1,2,3]}'))
  .add('structuredClone', () => structuredClone({ a: 1, b: [1, 2, 3] }));

await bench.run();
console.table(bench.table()); // ops/sec, average, margin of error per task
```

Read the output for **ops/sec** (higher is better) and the **margin of error / standard deviation** — a result with ±30% variance is noise, not signal. To catch regressions in CI, save a baseline (`vitest bench --outputJson`) and compare against it, but be aware CI runners are noisy shared machines; treat CI benchmark numbers as rough guardrails, not precise measurements. Reserve microbenchmarks for genuinely hot code — most code's speed is dominated by I/O, which a microbenchmark does not measure.

---

## 15. Fuzzing

### 15.1 What fuzzing is and when it matters **[A]**

**Fuzzing** feeds your code a relentless stream of malformed, random, and adversarial inputs to make it crash, hang, or misbehave — it is property-based testing's aggressive cousin, aimed at *robustness and security* rather than logic. Where fast-check generates inputs to falsify a *property you wrote*, a coverage-guided fuzzer mutates inputs to maximise *code coverage*, steering toward untested branches to trigger unhandled exceptions, infinite loops, prototype-pollution, or ReDoS (catastrophic-backtracking regexes). It matters most for code that parses **untrusted input**: file-format parsers, protocol decoders, deserializers, and anything at a security boundary. In the Node world, **Jazzer.js** (from Code Intelligence) is the coverage-guided fuzzer.

```bash
npm i -D @jazzer.js/core
```

### 15.2 A fuzz target with Jazzer.js **[A]**

You write a **fuzz target** — a function taking raw bytes — and let the fuzzer drive it, hunting for inputs that throw.

```js
// parse.fuzz.js — run with: npx jazzer parse.fuzz
const { parseConfig } = require('./config-parser');

/**
 * @param { import('@jazzer.js/core').FuzzedDataProvider } data
 * The fuzzer calls this thousands of times with mutated byte buffers,
 * steering toward inputs that reach new code paths and crash the parser.
 */
module.exports.fuzz = function (data) {
  const input = data.consumeString(data.remainingBytes());
  try {
    parseConfig(input);            // the code under test
  } catch (err) {
    // Catch EXPECTED validation errors so only UNEXPECTED crashes fail the fuzz run.
    if (err instanceof SyntaxError) return; // this is a normal, handled rejection
    throw err;                     // anything else is a real bug the fuzzer found
  }
};
```

When Jazzer finds a crashing input, it writes it to a file so you can add it as a **regression test** (a permanent example test that pins the fixed bug). Run fuzzing occasionally or on a schedule rather than on every commit — it is open-ended and CPU-hungry. It is a specialist tool: most application code does not need it, but for a parser handling untrusted bytes it can find memory-safety-adjacent and denial-of-service bugs no other technique will.

---

## 16. Filesystem and OS Code Testing

### 16.1 The challenge — real I/O is stateful and shared **[I]**

Code that touches the **filesystem** or runs **child processes** is hard to test because it has side effects on a resource shared with the rest of your machine: a test that writes to `./output.txt` collides with another test doing the same, leaves garbage behind, and behaves differently on Windows vs Linux (path separators, line endings, permissions). Two strategies: **use a real but isolated temp directory** (high confidence, my default for filesystem code), or **mock the `fs` module** (fast, but tests your assumptions rather than the OS). We cover both, using `node:test` to honour the zero-dependency spirit of OS-level code.

### 16.2 Real filesystem tests with a temp directory **[I]**

`fs.mkdtemp` creates a *unique* temp directory (its name ends in random characters), so parallel tests never collide. Create one per test, do real I/O in it, and delete it afterwards — you get real filesystem behaviour with perfect isolation.

```js
// file-store.test.js — run with: node --test
import { test, describe, before, after, beforeEach, afterEach } from 'node:test';
import assert from 'node:assert/strict';
import { mkdtemp, rm, writeFile, readFile, readdir } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

// The code under test: appends a line to a log file inside a given directory.
async function appendLog(dir, line) {
  const file = join(dir, 'app.log');
  const existing = await readFile(file, 'utf8').catch(() => ''); // '' if absent
  await writeFile(file, existing + line + '\n');
  return file;
}

describe('appendLog', () => {
  let dir;

  beforeEach(async () => {
    // A fresh, uniquely-named temp dir per test — no collisions, ever.
    // os.tmpdir() keeps it on the right volume for Windows AND POSIX.
    dir = await mkdtemp(join(tmpdir(), 'filestore-'));
  });

  afterEach(async () => {
    // Recursively remove it so tests leave the machine clean.
    await rm(dir, { recursive: true, force: true });
  });

  test('creates the file and writes the first line', async () => {
    const file = await appendLog(dir, 'hello');
    assert.equal(await readFile(file, 'utf8'), 'hello\n');
  });

  test('appends without clobbering existing content', async () => {
    await appendLog(dir, 'line1');
    await appendLog(dir, 'line2');
    const contents = await readFile(join(dir, 'app.log'), 'utf8');
    assert.equal(contents, 'line1\nline2\n');
    assert.deepEqual(await readdir(dir), ['app.log']); // exactly one file created
  });
});
```

Always build paths with `path.join` (never hard-code `/` or `\`) and root temp dirs at `os.tmpdir()` so the same test passes on Windows and Linux — this is the FS/OS-portability discipline the [Go filesystem guide](GO_FILESYSTEM_OS_CLI_GUIDE.md) covers for Go; the principles are identical here.

### 16.3 Mocking the fs module **[I]**

When you want to test *logic* around the filesystem without real I/O (faster, and lets you simulate errors like `EACCES` that are hard to produce for real), mock `node:fs`. This tests your error-handling branches deterministically.

```ts
import { describe, it, expect, vi } from 'vitest';
import { readFile } from 'node:fs/promises';
import { loadConfig } from './load-config';

vi.mock('node:fs/promises'); // auto-mock the module

describe('loadConfig', () => {
  it('returns defaults when the file is missing', async () => {
    // Simulate ENOENT without touching a real disk.
    vi.mocked(readFile).mockRejectedValue(
      Object.assign(new Error('no file'), { code: 'ENOENT' }),
    );
    expect(await loadConfig('/nope.json')).toEqual({ debug: false });
  });

  it('parses a real config when present', async () => {
    vi.mocked(readFile).mockResolvedValue('{"debug":true}');
    expect(await loadConfig('/app.json')).toEqual({ debug: true });
  });
});
```

Use real temp dirs when the *filesystem behaviour* is what matters (creation, appends, directory listing, permissions); mock `fs` when you are testing *decision logic* around rare error conditions. `memfs` is a popular in-memory `fs` implementation if you want realistic filesystem semantics without a mock or a disk.

### 16.4 Mocking child_process **[I]**

Code that shells out (`exec`, `spawn`) must not run real commands in tests — that is slow, platform-dependent, and possibly destructive. Mock `node:child_process` to return canned output and to assert *which command* your code tried to run.

```ts
import { describe, it, expect, vi } from 'vitest';
import { execFile } from 'node:child_process';
import { promisify } from 'node:util';
import { getGitBranch } from './git-info';

vi.mock('node:child_process');

describe('getGitBranch', () => {
  it('parses the current branch from git output', async () => {
    // execFile's callback signature is (err, stdout, stderr) — simulate success.
    vi.mocked(execFile).mockImplementation(((_cmd, _args, cb: any) => {
      cb(null, 'main\n', '');
      return {} as any;
    }) as any);

    expect(await getGitBranch()).toBe('main');
    // Assert we invoked git safely with execFile + arg array (no shell injection):
    expect(execFile).toHaveBeenCalledWith(
      'git', ['rev-parse', '--abbrev-ref', 'HEAD'], expect.any(Function),
    );
  });
});
```

Asserting on the *arguments* is as important as the output: it verifies you used `execFile` with an argument array (injection-safe) rather than building a shell string. For higher confidence in a controlled integration test, you can run a real, harmless command (`node -e "..."`) in a temp dir — but never a destructive or environment-dependent one.

---

## 17. Mutation Testing with Stryker

### 17.1 The question coverage can't answer **[A]**

Coverage (§7) tells you which lines *ran*, but not whether your assertions would actually *catch a bug* in those lines. You can execute a line with no meaningful assertion and score 100% coverage while testing nothing. **Mutation testing** answers the real question: *"if I deliberately broke the code, would a test fail?"* A mutation-testing tool makes tiny changes ("mutants") to your source — flips a `>` to `>=`, a `+` to `-`, `true` to `false`, removes a statement — then runs your suite against each mutant. If a test fails, the mutant is **killed** (good — your tests caught the bug). If all tests still pass, the mutant **survived** (bad — a real bug in that spot would ship undetected). Your **mutation score** (killed / total) is a far more honest measure of test quality than coverage. **Stryker** is the Node/JS/TS mutation tester.

```bash
npm i -D @stryker-mutator/core
# plus a runner plugin: @stryker-mutator/vitest-runner OR @stryker-mutator/jest-runner
npx stryker init   # scaffolds stryker.config.json
```

### 17.2 Running Stryker and reading the report **[A]**

```json
// stryker.config.json
{
  "$schema": "https://raw.githubusercontent.com/stryker-mutator/stryker-js/master/packages/api/schema/stryker-core.json",
  "packageManager": "npm",
  "testRunner": "vitest",
  "reporters": ["html", "clear-text", "progress"],
  "coverageAnalysis": "perTest",
  "mutate": ["src/**/*.ts", "!src/**/*.test.ts"]
}
```

```bash
npx stryker run
```

Stryker prints a score and writes an HTML report highlighting every **surviving mutant** — each one is a precise instruction for a missing assertion. A survivor on `if (age >= 18)` mutated to `if (age > 18)` means you never tested the boundary (exactly 18); add that case and re-run. Because Stryker runs the whole suite once per mutant, it is **slow** — scope it with `mutate` to critical modules (payment, auth, pricing), use `coverageAnalysis: "perTest"` so it only runs the tests that cover each mutant, and run it nightly or pre-release rather than on every push. Do not chase 100% mutation score globally; use it as a sharp scalpel on the code where correctness truly matters.

---

## 18. Continuous Integration with GitHub Actions

### 18.1 Why tests must run in CI, not just on your laptop **[A]**

Tests that only run on your machine protect only your machine. **Continuous Integration** runs the whole suite automatically on every push and pull request, on a clean, standardised environment — catching "works on my machine" bugs (missing dependency, uncommitted file, platform difference) and *blocking merges* when tests fail. This section covers the testing-specific mechanics of a GitHub Actions pipeline; for the full CI/CD story (deployment, environments, secrets) see the [GitHub Actions CI/CD](GITHUB_ACTIONS_CICD_GUIDE.md) guide.

### 18.2 A matrix workflow with caching **[A]**

A **matrix** runs the suite across multiple Node versions (and OSes) in parallel, so you catch version-specific breakage. Dependency **caching** makes each run fast by reusing `~/.npm` between runs.

```yaml
# .github/workflows/test.yml
name: test
on:
  push: { branches: [main] }
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false                 # let all matrix legs finish even if one fails
      matrix:
        node: [22, 24]                 # test on both LTS and current
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'                 # cache ~/.npm keyed on package-lock.json

      - run: npm ci                    # clean, lockfile-exact install

      - run: npm run typecheck         # the cheapest test layer FIRST (fail fast)
      - run: npm run lint
      - run: npm run test -- --coverage # unit + integration with coverage

      - name: Upload coverage
        if: matrix.node == 24          # upload once, from the primary leg
        uses: actions/upload-artifact@v4
        with: { name: coverage, path: coverage/ }
```

### 18.3 Services and Docker for integration tests **[A]**

Integration tests need a real database. Two options in CI: a **service container** (GitHub spins up Postgres alongside your job) or **Testcontainers** (§11), which works unchanged in Actions because the runner has Docker. Service containers are simplest when every test in the job shares one DB:

```yaml
  integration:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:17-alpine
        env: { POSTGRES_PASSWORD: test, POSTGRES_DB: testdb }
        ports: ['5432:5432']
        # Wait until Postgres is accepting connections before tests start:
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 10s --health-timeout 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 24, cache: 'npm' }
      - run: npm ci
      - run: npm run test:integration
        env:
          DATABASE_URL: postgres://postgres:test@localhost:5432/testdb
```

### 18.4 Sharding, parallelism and Playwright in CI **[A]**

Large suites split across machines with **sharding** — each CI job runs a fraction of the tests, cutting wall-clock time. Vitest (`--shard=1/4`) and Playwright (`--shard=1/4`) both support it:

```yaml
  e2e:
    runs-on: ubuntu-latest
    strategy:
      matrix: { shard: [1, 2, 3, 4] }        # 4 parallel jobs, each a quarter
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 24, cache: 'npm' }
      - run: npm ci
      - run: npx playwright install --with-deps chromium   # browsers + OS libs
      - run: npx playwright test --shard=${{ matrix.shard }}/4
      - uses: actions/upload-artifact@v4
        if: failure()                          # keep traces ONLY when a shard fails
        with: { name: playwright-trace-${{ matrix.shard }}, path: test-results/ }
```

### 18.5 Reporters, flaky-test detection and retries **[A]**

- **Reporters** shape CI output. Use a compact reporter locally and a machine-readable one in CI: `vitest run --reporter=dot --reporter=junit --outputFile=junit.xml` produces a JUnit XML that GitHub and dashboards parse into a test-results UI. Playwright's `--reporter=github` annotates failures directly on the PR diff.
- **Flaky-test detection.** A flaky test passes and fails without code changes. Configure **retries in CI only** (`retries: process.env.CI ? 2 : 0`) so a transient blip does not fail the build — *but* treat retries as a smell, not a cure: log which tests needed a retry and fix them, because a genuinely flaky test erodes trust in the whole suite. Playwright's HTML report flags tests that passed only on retry as "flaky".
- **Fail fast on the cheap layers.** Order CI steps cheapest-first: `typecheck` → `lint` → unit → integration → e2e. A type error should fail in 20 seconds, not after a 15-minute e2e run.
- **Required checks.** In branch-protection settings, mark the test job a *required status check* so a red suite genuinely blocks merge — CI that can be ignored protects nothing.

---

## 19. Gotchas and Best Practices

This section collects the mistakes that cost the most debugging time, each with the fix.

### 19.1 Test behaviour, not implementation **[I]**

The single most important principle. A test coupled to *how* code works internally (private methods, call order, internal state) breaks on every refactor even when behaviour is unchanged, and passes when behaviour breaks but the internals look the same. Assert on **observable outputs and effects**: return values, thrown errors, what was written to the database, what HTTP response came back.

```ts
// BAD — asserts on internal implementation. Breaks if you rename or reorder internals.
expect(service._validate).toHaveBeenCalledBefore(service._persist);

// GOOD — asserts on the observable outcome a caller actually cares about.
const result = await service.register('Ada', 'ada@x.com');
expect(result).toMatchObject({ id: expect.any(Number), name: 'Ada' });
```

### 19.2 Shared state between tests **[I]**

The most common cause of "passes alone, fails in the suite" and order-dependent flakiness. A module-level array, a singleton, a database row, a mock's un-cleared call history — any state that outlives a test pollutes the next one.

```ts
// BAD — module-level state accumulates across tests.
const cache = new Map();
it('a', () => { cache.set('x', 1); /* ... */ });
it('b', () => { expect(cache.size).toBe(0); }); // FAILS — 'a' polluted it

// GOOD — fresh state per test via beforeEach.
let cache: Map<string, number>;
beforeEach(() => { cache = new Map(); });
```

Also clear mocks between tests: set `clearMocks: true` in the Vitest/Jest config, or call `vi.clearAllMocks()` in `beforeEach`. Never rely on test execution order — runners may parallelise or reorder.

### 19.3 Non-deterministic time and randomness **[I]**

Tests that read `Date.now()`, `new Date()`, `Math.random()`, or `crypto.randomUUID()` produce different results every run and fail at midnight, at month boundaries, or in other timezones. Control them: fake timers / `setSystemTime` for time (§5.6), and inject or seed randomness.

```ts
// BAD — asserts on a value derived from the real clock; fails tomorrow.
expect(makeToken().expiresAt).toBe(/* some fixed date */);

// GOOD — freeze time so "now" is deterministic.
vi.useFakeTimers();
vi.setSystemTime(new Date('2026-07-17T00:00:00Z'));
expect(makeToken().expiresAt.toISOString()).toBe('2026-07-17T01:00:00.000Z');
vi.useRealTimers();
```

Also beware **timezone dependence**: run CI with a fixed `TZ` (e.g. `TZ=UTC`) so date-formatting tests behave identically everywhere.

### 19.4 Leaking async handles and open connections **[A]**

If a test opens a database pool, a server, a timer, or a file handle and never closes it, the Node process cannot exit — CI *hangs* until it times out, or you see "a worker process failed to exit gracefully." Every resource opened in `beforeAll`/`beforeEach` needs a matching close in `afterAll`/`afterEach`.

```ts
let pool: Pool;
beforeAll(() => { pool = new Pool({ /* ... */ }); });
afterAll(async () => { await pool.end(); });     // WITHOUT this, CI hangs

// NestJS: await app.close(); Playwright/containers: stop them. Clear intervals:
afterEach(() => { clearInterval(handle); });
```

Diagnose leaks with Vitest's `--reporter=verbose` plus `--no-file-parallelism`, or Jest's `--detectOpenHandles`, which points at the un-closed resource.

### 19.5 ESM vs CommonJS mocking **[A]**

As covered in §5.5: module mocking behaves differently under native ESM (live read-only bindings) than CommonJS (mutable `require` cache). If a `vi.mock`/`jest.mock` "doesn't take," check whether you are in ESM, whether the mock is hoisted above the import, and whether the code imports the *same specifier* you mocked (mocking `'./mailer'` does nothing if the code imports `'./mailer.js'`). When in doubt, prefer dependency injection (§5.8) which sidesteps module resolution entirely.

### 19.6 Over-mocking and testing the mock **[I]**

Recapping §5.10: if a test mocks everything and asserts that "the mock was called with X," it proves only that the code matches its own current implementation. Such tests give false confidence and break on every refactor. Mock across process boundaries (network, third-party, clock); use real or containerised dependencies within your own system.

### 19.7 Assertion-free and conditional-assertion tests **[I]**

A test with no assertion passes as long as the code does not throw — it is coverage theatre. Worse is an assertion inside an `if` or a `.then` that never runs. Always assert; put assertions on the main path; and for async, ensure the assertion is awaited (§4.1). Vitest's `expect.hasAssertions()` / `expect.assertions(n)` guards make "at least one assertion ran" explicit for tricky async tests.

### 19.8 A best-practices checklist **[I/A]**

| Do | Don't |
|---|---|
| Name tests by observable behaviour | Name tests after methods called |
| Fresh state in `beforeEach` | Share mutable state across tests |
| `await` every async assertion | Leave floating promises |
| Mock only external boundaries | Mock your own domain logic |
| Control time and randomness | Read the real clock/RNG in assertions |
| Close every resource in teardown | Leak pools, servers, timers |
| One logical assertion focus per test | Test five behaviours in one `it` |
| Type-check tests as a CI gate | Rely on runtime tests for type safety |
| Use real DB via Testcontainers for data logic | Hand-roll DB mocks |
| Keep E2E few and for critical journeys | Build an ice-cream-cone suite |
| Ban `.only` and `waitForTimeout` in CI | Commit stray `.only` / fixed sleeps |
| Ratchet coverage thresholds up | Chase 100% with assertion-free tests |

---

## 20. Study Path and Build-to-Learn Projects

The only way to internalise testing is to write tests for real code. Follow this path in order; each step builds on the last.

### 20.1 A staged learning path

1. **Fundamentals (§1–§4).** Install Vitest in a scratch project. Write a `math.ts` with `add`, `divide` (throws on divide-by-zero), and an async `fetchWithRetry`. Test every branch, including the throw and the rejection. Get comfortable with `describe`/`it`/`expect`, hooks, and `async`/`await` tests. Then rewrite the same tests in `node:test` to feel the difference.
2. **Mocking and DI (§5).** Write a `NotificationService` that depends on an `EmailClient` and a `Clock`. Test it first with `vi.mock`, then refactor to dependency injection and test it with plain fakes — feel how much simpler the DI version is. Add fake timers to test a debounce/retry-with-backoff.
3. **TypeScript and coverage (§6–§8).** Turn on `tsc --noEmit` and coverage thresholds. Drive branch coverage to 85% on a small module, then look at the HTML report and write tests for the red lines. Add one snapshot test and one property test, and deliberately make each fail to see the output.
4. **Integration (§9, §11).** Build a small Express (or NestJS) todo API. Test it end-to-end with Supertest — status codes, validation errors, auth. Then add a real Postgres via Testcontainers and test the repository layer against it with per-test truncation.
5. **E2E and CI (§12, §18).** Put a minimal frontend on the API, write one Playwright login-to-list journey, and wire a GitHub Actions workflow: matrix typecheck + lint + unit + integration, with a sharded Playwright job.
6. **Advanced (§13, §15, §17).** Add fast-check properties to your validation logic, run Stryker on your pricing/auth module and kill the survivors, and (optional) fuzz a parser with Jazzer.js.

### 20.2 Build-to-learn projects

- **Project A — "Bank account" library (unit + property + mutation).** A pure-logic `Account` with `deposit`, `withdraw` (rejects overdraft), `transfer`, and interest accrual. Write exhaustive unit tests, then fast-check properties (balance never goes negative; sum is conserved across a transfer), then run Stryker and drive the mutation score above 90%. This teaches assertion quality better than any tutorial.
- **Project B — "URL shortener" API (integration + DB + auth).** Express or NestJS, Postgres via Testcontainers, JWT auth. Test the full stack with Supertest: create-short-url, redirect, per-user listing, 404s, auth failures. Add transaction-rollback isolation and get the suite under a few seconds. This is the testing-trophy sweet spot.
- **Project C — "Admin dashboard" (E2E + CI).** A small React/Next front end over Project B. One Playwright journey: log in, create a link, see it in the table, delete it. Wire the whole thing into GitHub Actions with sharding, coverage upload, and trace-on-failure. This teaches the realities of E2E and CI: flake, waiting, and debugging failures you cannot watch.
- **Project D — "Config parser" (fuzz + snapshot).** A parser for a small config format. Golden-file/snapshot tests for valid inputs, property tests for round-tripping, and a Jazzer.js fuzz target that must survive thousands of malformed inputs without an unhandled crash. This teaches robustness testing on untrusted input.

Work through these and you will not just *know* the tools — you will have the judgement for *which* test to reach for, how much to mock, and where the confidence-per-second is highest. That judgement, more than any single API, is what separates a production-grade suite from a pile of brittle tests. Happy testing.

