# Testing in Go — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone who can write a little Go but has never written a test, all the way up to engineers who want to run a fast, deterministic, production-grade test suite in CI. You will start from `go test` and a single `TestXxx` function and finish able to write unit tests, table-driven tests, mocks and fakes, HTTP handler tests, benchmarks, fuzz tests, and full integration and end-to-end tests against a real PostgreSQL database in a container. This guide is deliberately **explain-first**: every idea leads with prose — *what it is, why it exists, when to reach for it, how it works, the best practice, and the gotcha* — and only then shows heavily-commented, runnable code. Testing is a skill you learn by understanding *why* a test is fast or slow, isolated or flaky; the reasoning matters as much as the syntax.
>
> **Version note:** This guide targets **Go 1.25 / 1.26** (2026-current). It uses the standard library **`testing`** and **`net/http/httptest`** packages (no dependency needed), **testify v1.10+** (`github.com/stretchr/testify`) for assertions and suites, **gomock via `go.uber.org/mock` v0.5+** (the maintained successor to the archived `github.com/golang/mock`), **Testcontainers-go v0.34+** (`github.com/testcontainers/testcontainers-go`) for real-database integration tests, and **`pgregory.net/rapid`** for property-based testing. Native **fuzzing** has been in the standard toolchain since **Go 1.18**; the benchmark helper **`testing.B.Loop`** landed in **Go 1.24**, and the **`testing/synctest`** package for deterministic concurrency testing graduated to stable in **Go 1.25** (experimental in 1.24). Version-specific behavior is flagged inline with **⚡**. The author is on **Windows 11**; shell commands are shown for PowerShell and POSIX where they differ. Always cross-check current signatures at **pkg.go.dev** before shipping.
>
> **This guide's place in the library:** Testing is the discipline that keeps every other guide's code honest. The worked examples here test the exact stack the sibling guides build: [Go Gin REST API + File Upload](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) is the HTTP app whose handlers we test in §7 and end-to-end in §13; [pgx](GO_PGX_GUIDE.md) and [Go Ent ORM](GO_ENT_ORM_GUIDE.md) are the data layers we integration-test in §12 against a real database from [PostgreSQL](POSTGRESQL_GUIDE.md); [GitHub Actions CI/CD](GITHUB_ACTIONS_CICD_GUIDE.md) is where §18 runs the whole suite on every push; and [Docker](DOCKER_GUIDE.md) is what Testcontainers drives under the hood. Read this top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.

---

## Table of Contents

1. [Why Test and the Testing Mindset](#1-why-test-and-the-testing-mindset) **[B]**
2. [The testing Package and go test](#2-the-testing-package-and-go-test) **[B]**
3. [Table-Driven Tests with Subtests and Parallelism](#3-table-driven-tests-with-subtests-and-parallelism) **[B]**
4. [Assertions — Stdlib versus testify](#4-assertions--stdlib-versus-testify) **[B/I]**
5. [Test Lifecycle Setup Teardown and Golden Files](#5-test-lifecycle-setup-teardown-and-golden-files) **[I]**
6. [Test Doubles — Fakes Stubs Spies and Mocks](#6-test-doubles--fakes-stubs-spies-and-mocks) **[I]**
7. [HTTP Testing with httptest and Gin](#7-http-testing-with-httptest-and-gin) **[I]**
8. [Coverage and the Race Detector](#8-coverage-and-the-race-detector) **[I]**
9. [Benchmarks and Profiling](#9-benchmarks-and-profiling) **[I/A]**
10. [Fuzzing](#10-fuzzing) **[A]**
11. [Example Tests as Executable Documentation](#11-example-tests-as-executable-documentation) **[B/I]**
12. [Integration Tests with Testcontainers and Postgres](#12-integration-tests-with-testcontainers-and-postgres) **[A]**
13. [End-to-End Tests](#13-end-to-end-tests) **[A]**
14. [Property-Based Testing](#14-property-based-testing) **[A]**
15. [Testing Time and Concurrency](#15-testing-time-and-concurrency) **[A]**
16. [Testing the Filesystem OS and exec Code](#16-testing-the-filesystem-os-and-exec-code) **[I]**
17. [Mutation Testing](#17-mutation-testing) **[A]**
18. [Running Tests in CI with GitHub Actions](#18-running-tests-in-ci-with-github-actions) **[I]**
19. [Gotchas and Best Practices](#19-gotchas-and-best-practices) **[A]**
20. [Study Path and Build-to-Learn Projects](#20-study-path-and-build-to-learn-projects)

---

## 1. Why Test and the Testing Mindset

### 1.1 What a test actually is **[B]**

A **test** is nothing more than *ordinary code that runs your other code and checks that the result is what you expected*. There is no magic. In Go a test is a normal function in a file ending in `_test.go`; you call the function you want to check, look at what it returned, and if it is wrong you tell the test framework "this failed." That is the whole idea. Everything else in this guide — table-driven tests, mocks, fuzzing, containers — is a technique layered on top of that one primitive.

The reason we bother writing this second body of code is that **software changes, and change breaks things silently.** When you fix a bug or add a feature six months from now, you have no way to know you didn't re-break something you wrote today — unless a test remembers the expected behavior for you and screams the moment it stops holding. Tests are *executable specifications*: they encode "this is what the code is supposed to do" in a form the machine can re-check in milliseconds, forever, for free. A codebase with good tests is one you can change fearlessly; a codebase without them is one where every change is a gamble.

### 1.2 Why Go makes this easy **[B]**

Go bakes testing into the language toolchain. There is no framework to install, no config file, no test runner to wire up: the `testing` package ships with the standard library and the `go test` command is part of the `go` tool. This is a deliberate design choice — the Go team wanted testing to be so frictionless that there is no excuse not to do it. Because everyone uses the same built-in machinery, every Go project's tests look and run the same way, and knowledge transfers instantly between codebases.

### 1.3 The kinds of tests — a taxonomy **[B]**

"Test" is an umbrella word. Before you write one, you should know which *kind* you mean, because the kind determines how fast it is, how much it isolates, and what it can catch. Here is the full taxonomy you will meet in this guide:

| Kind | What it checks | Speed | Isolation | Covered in |
|---|---|---|---|---|
| **Unit test** | One function/type in isolation, no I/O | Microseconds | Total (no network/DB/disk) | §2–§6 |
| **Integration test** | Two or more real components together (e.g. your repository + a real Postgres) | Milliseconds–seconds | Partial (real DB, no network to the world) | §12 |
| **End-to-end (e2e) test** | The whole app through its real entry point (HTTP router → handler → DB → response) | Seconds | None (exercises everything) | §13 |
| **Benchmark** | *How fast* a piece of code runs, not whether it is correct | Varies | Total | §9 |
| **Fuzz test** | Correctness/robustness against *randomly generated* inputs looking for crashes | Continuous | Total | §10 |
| **Example test** | That documented example code compiles and prints what the docs claim | Microseconds | Total | §11 |
| **Property-based test** | That an invariant holds across *many generated* inputs | Milliseconds | Total | §14 |
| **Mutation test** | *How good your tests are* — do they catch deliberately injected bugs | Slow (reruns suite) | N/A (meta-test) | §17 |

You do not use all of these on every project, but a healthy Go service uses at least unit, integration, e2e, and benchmarks, and adds fuzz and property tests wherever inputs are hostile or invariants are subtle.

### 1.4 The test pyramid and the test trophy **[B]**

Two mental models help you decide *how many of each kind* to write.

The classic **test pyramid** (Mike Cohn) says: write **many** fast, cheap unit tests at the base; **fewer** integration tests in the middle; and **very few** slow, brittle end-to-end tests at the top. The reasoning is economic — unit tests are fast and pinpoint failures precisely, while e2e tests are slow, flaky, and when they fail they only tell you "something in this whole chain broke" without saying where. Most of your confidence should come cheaply from the base.

```text
        /\
       /e2e\        few   — slow, whole-system, brittle
      /------\
     /  integ \     some  — real DB/deps, medium speed
    /----------\
   /    unit    \   many  — microseconds, isolated, precise
  /--------------/
```

The modern **test trophy** (Kent C. Dodds) rebalances this for services that are mostly glue around I/O. It argues the *integration* layer often gives the best confidence-per-second, because a unit test of a handler with everything mocked can pass while the real wiring is broken. The trophy is wide in the middle: a thin base of static analysis (`go vet`, the compiler, `golangci-lint`), a solid band of unit tests, the **largest** band of integration tests, and a thin cap of e2e.

The practical synthesis for Go, and the philosophy of this guide: **write unit tests for logic-heavy pure code (parsers, calculations, state machines) and integration tests for I/O-heavy glue code (repositories, handlers). Do not mock the thing you are actually trying to gain confidence in.** §6 and §12 return to this repeatedly.

### 1.5 What "efficient tests" means **[B]**

Throughout this guide "a good test" means one with four properties. Memorize these; they are the rubric you will judge every test against:

- **Fast.** Unit tests run in microseconds so you can run the whole suite on every save. Slowness compounds: a suite that takes five minutes gets run once an hour instead of once a minute, and its bug-catching value collapses.
- **Isolated (independent).** A test must not depend on another test having run first, on shared mutable global state, or on the order tests run in. Isolated tests can run in parallel and in random order (§18's `-shuffle`). The enemy is *shared state*.
- **Deterministic (repeatable).** The same code must produce the same pass/fail every time. A test that passes sometimes and fails sometimes is **flaky** (§19) — worse than no test, because it trains the team to ignore red. Determinism means no dependence on wall-clock time (§15), random seeds you don't control, network to the outside world, or goroutine scheduling.
- **Readable.** A test is documentation. When it fails at 3am, the failure message and the test body must make it obvious *what* broke and *what was expected*. A clever test nobody can read is a liability.

Hold these four up against every example that follows. When we prefer a fake over a mock, inject a clock instead of calling `time.Now`, or use a subtest name that describes the case, it is always in service of fast/isolated/deterministic/readable.

---

## 2. The testing Package and go test

### 2.1 The absolute minimum — your first test **[B]**

Let's ground everything in a real, tiny example. Suppose we have a function to test. By convention it lives in a file like `math.go` inside a package:

```go
// file: calc/calc.go
package calc

// Add returns the sum of two integers. This is the "production" code —
// the thing we want to gain confidence in.
func Add(a, b int) int {
	return a + b
}
```

The test for it goes in a file **in the same directory** whose name ends in `_test.go`. The Go tool treats `_test.go` files specially: they are compiled and run only under `go test`, and never included in your normal build or shipped in your binary. That is why you can put test-only helpers and imports there without bloating production.

```go
// file: calc/calc_test.go
package calc

import "testing"

// A test is a function whose name starts with "Test", takes exactly one
// parameter of type *testing.T, and returns nothing. go test discovers
// every such function by name (via reflection over the compiled package)
// and runs it. The name after "Test" must start with an uppercase letter
// or be empty — "TestAdd" is found, "Testadd" is NOT.
func TestAdd(t *testing.T) {
	// 1. Arrange: set up inputs and the expected result.
	got := Add(2, 3)
	want := 5

	// 2. Assert: compare reality to expectation. If they differ, we call
	//    t.Errorf, which marks THIS test as failed and records a message,
	//    but keeps running the rest of the function.
	if got != want {
		t.Errorf("Add(2, 3) = %d; want %d", got, want)
	}
}
```

Run it from the package directory:

```console
$ go test
PASS
ok      example.com/calc   0.002s
```

That is a complete, working Go test. Notice there is no assertion library, no `describe`/`it`, no annotations — just an `if` and a call to report failure. This minimalism is intentional and it scales surprisingly far.

### 2.2 The `*testing.T` object — how you talk to the framework **[B]**

Every test receives a `*testing.T`. This is your handle to the test framework: it is how you report failures, log diagnostics, control the test's lifecycle, and spawn subtests. The methods fall into a few groups. The most important distinction — the one beginners get wrong most often — is **`Error` versus `Fatal`**:

| Method | What it does | Keeps running? |
|---|---|---|
| `t.Log(args...)` / `t.Logf(fmt, ...)` | Record a diagnostic message (only shown on failure, or always with `-v`) | Yes |
| `t.Error(args...)` / `t.Errorf(fmt, ...)` | Mark the test **failed**, record a message | **Yes** — execution continues |
| `t.Fatal(args...)` / `t.Fatalf(fmt, ...)` | Mark the test **failed**, then **stop this test** immediately | **No** — via `runtime.Goexit` |
| `t.Fail()` | Mark failed with no message | Yes |
| `t.FailNow()` | Mark failed and stop (what `Fatal` calls) | No |
| `t.Skip(args...)` / `t.Skipf(...)` | Mark **skipped** and stop (e.g. "needs network") | No |
| `t.Helper()` | Mark this function a helper so failures point at the caller | — |
| `t.Cleanup(fn)` | Register `fn` to run when the test ends (§5) | — |

**When to use which:** reach for `t.Errorf` by default — it lets one test report *multiple* independent problems in a single run, which is more informative than stopping at the first. Reach for `t.Fatalf` when continuing makes no sense because a precondition failed: if you couldn't open a file or a constructor returned an error, the rest of the test would only produce noise (or panic on a nil pointer), so stop now.

```go
func TestParseConfig(t *testing.T) {
	cfg, err := ParseConfig("testdata/valid.yaml")
	if err != nil {
		// A nil cfg means every line below would panic. There is no point
		// continuing, so Fatal: stop this test right here.
		t.Fatalf("ParseConfig returned unexpected error: %v", err)
	}

	// From here cfg is safe to use. These are independent checks, so use
	// Error (not Fatal): if Port is wrong AND Host is wrong we want to see
	// BOTH in one run, not fix one and rerun to discover the next.
	if cfg.Port != 8080 {
		t.Errorf("Port = %d; want 8080", cfg.Port)
	}
	if cfg.Host != "localhost" {
		t.Errorf("Host = %q; want %q", cfg.Host, "localhost")
	}
}
```

> **⚡ Fatal only works on the test's own goroutine.** `t.Fatal`/`t.FailNow` call `runtime.Goexit`, which only unwinds the goroutine it runs on. If you call `t.Fatal` from a goroutine you spawned inside a test, it will **not** stop the test correctly and the behavior is undefined. From helper goroutines, send the error back over a channel and `Fatal` on the main test goroutine, or use `t.Error` (which is goroutine-safe for reporting). §15 revisits this.

### 2.3 Anatomy of a failure message — write good ones **[B]**

The `if got != want { t.Errorf(...) }` shape is the beating heart of Go testing, and the *message* you pass is what you will read when it fails. The community convention is:

```go
t.Errorf("FunctionName(inputs) = %v; want %v", got, want)
```

That is: what you called, what it produced (`got`), and what you expected (`want`). Use `%v` for general values, `%q` for strings (so you can see quotes, whitespace, and empty strings), `%d` for ints, `%+v` for structs with field names, and `%#v` for a Go-syntax dump. A message like `t.Error("failed")` is nearly useless; `t.Errorf("Add(2,3) = %d; want 5", got)` tells you everything at a glance. The `got`/`want` naming is a near-universal Go convention — follow it and every Go dev instantly reads your tests.

### 2.4 Running and filtering tests with go test **[B]**

`go test` is the command that compiles your test files together with the package and runs the discovered tests. You will spend a lot of time with its flags:

| Command | What it does |
|---|---|
| `go test` | Run tests in the current directory's package |
| `go test ./...` | Run tests in **every** package under the current dir (the standard "test everything") |
| `go test -v` | **Verbose**: print each test's name, `t.Log` output, and PASS/FAIL as it goes |
| `go test -run TestAdd` | Run only tests whose name **matches the regexp** `TestAdd` |
| `go test -run 'TestAdd/negative'` | Run only the `negative` **subtest** of `TestAdd` (the `/` splits levels — §3) |
| `go test -count=1` | Disable the test result cache — force a real rerun |
| `go test -failfast` | Stop at the first failing test instead of running the rest |
| `go test -shuffle=on` | Randomize test execution order to catch order-dependence (§18) |
| `go test -timeout 30s` | Fail the run if it hangs longer than 30s (default is 10m) |

Two behaviors surprise newcomers. First, `-run` takes a **regular expression**, not a literal — `go test -run Add` runs `TestAdd`, `TestAddNegative`, and `TestReadAddress`, because the regex is unanchored. Anchor it with `-run '^TestAdd$'` to be exact. Second, `go test` **caches** passing results: if nothing about a package changed, a second `go test` prints `(cached)` and doesn't rerun. That is a feature (fast CI), but when a test reads an external resource that changed, add `-count=1` to bypass the cache.

```console
$ go test -v -run '^TestAdd$' ./calc
=== RUN   TestAdd
--- PASS: TestAdd (0.00s)
PASS
ok      example.com/calc   0.003s
```

### 2.5 Package layout — white-box versus black-box tests **[B]**

Here is a subtlety unique to Go that has real design consequences. A `_test.go` file in directory `calc/` may declare **one of two packages**:

- **`package calc`** — the *same* package as the code under test. This is a **white-box** test: it can see and call the package's *unexported* (lowercase) identifiers. Use it to test internal helpers directly.
- **`package calc_test`** — a *separate* package (Go makes a special exception allowing this `_test` suffix in the same directory). This is a **black-box** test: it can only see the package's **exported** API, exactly as a real consumer would. It must `import` the package to use it.

Why would you deliberately restrict yourself to the exported API? Because **testing through the public API tests behavior, not implementation.** A black-box test survives internal refactors — you can rename a private helper, change an algorithm, restructure the internals, and if the public behavior is unchanged the test still passes. A white-box test coupled to internals breaks on refactors that changed nothing a user can observe, which trains people to "fix the test" reflexively. The best practice: **default to black-box (`package foo_test`); drop to white-box (`package foo`) only when you genuinely need to test an unexported function that is too complex to reach through the public API.**

```go
// file: calc/calc_blackbox_test.go
package calc_test // black-box: separate package, sees only exported API

import (
	"testing"

	"example.com/calc" // we must import what we're testing
)

func TestAdd_BlackBox(t *testing.T) {
	// We can only reach calc.Add (exported). calc's private helpers are
	// invisible here — exactly the constraint a real user has.
	if got := calc.Add(2, 2); got != 4 {
		t.Errorf("Add(2,2) = %d; want 4", got)
	}
}
```

> **Both can coexist.** Go lets `calc/` contain both a `package calc` test file and a `package calc_test` test file at once. A common pattern: black-box files for the public behavior, one white-box file for the gnarly internal helper. The `Example` tests in §11 almost always live in the `_test` package so the docs show real usage.

### 2.6 Where tests live and the testdata convention **[B]**

Go has no `tests/` directory. **Test files live right next to the code they test** — `calc.go` and `calc_test.go` in the same folder. This keeps a function and its test one keystroke apart and makes it obvious at a glance which code is untested (no neighboring `_test.go`). The one special directory name is **`testdata/`**: the Go tool *ignores* any directory named `testdata`, so you can put fixture files (sample inputs, golden outputs — §5) there without them being treated as a package or compiled. Reference them with a relative path from the test, because `go test` sets the working directory to the package directory:

```go
data, err := os.ReadFile("testdata/sample_input.json") // relative to the package dir
```

---

## 3. Table-Driven Tests with Subtests and Parallelism

### 3.1 Why table-driven tests are THE Go idiom **[B]**

Once you have written three tests for `Add` — one for positives, one for negatives, one for zero — you will notice they are the same four lines with different numbers. Copy-pasting that shape is how test files become thousand-line walls of near-identical functions where adding a case means duplicating a block and nobody can see the forest. Go's answer, and the single most important testing pattern in the language, is the **table-driven test**: you describe your cases as *data* — a slice of structs — and loop over them running the same assertion logic for each.

The payoff is enormous. Adding a new case becomes adding **one line** to the table. The test logic exists exactly once, so a bug in the assertion is fixed in one place. The table reads like a specification: a reader scans the rows and sees every input/output pair the function promises to handle. This is not a niche trick — it is how idiomatic Go is tested, from the standard library outward. Learn it now and use it everywhere.

```go
package calc

import "testing"

func TestAdd_Table(t *testing.T) {
	// The table: each row is one test case. Naming the struct type inline
	// keeps it local. The `name` field labels the case in output; give it a
	// short human description of WHAT this row exercises.
	tests := []struct {
		name string // description of the case
		a, b int    // inputs
		want int    // expected output
	}{
		{name: "two positives", a: 2, b: 3, want: 5},
		{name: "with zero", a: 0, b: 7, want: 7},
		{name: "two negatives", a: -4, b: -6, want: -10},
		{name: "mixed signs", a: -5, b: 5, want: 0},
	}

	// Loop over the rows. `tc` (test case) holds the current row.
	for _, tc := range tests {
		// t.Run creates a SUBTEST named after the case (see 3.2). Everything
		// inside runs as its own test with its own *testing.T.
		t.Run(tc.name, func(t *testing.T) {
			got := Add(tc.a, tc.b)
			if got != tc.want {
				// Note the case name is implicit in the subtest, so the
				// message just needs the values.
				t.Errorf("Add(%d, %d) = %d; want %d", tc.a, tc.b, got, tc.want)
			}
		})
	}
}
```

### 3.2 Subtests with t.Run — structure and selective runs **[B]**

`t.Run(name, func(t *testing.T){...})` creates a **subtest**: a test nested inside another, with its own `*testing.T`, its own pass/fail status, and its own place in the output. Subtests are what make the table above so much better than a bare loop. Without `t.Run`, a loop that hits `t.Error` on row 3 gives you one failure with no clue which row; the whole test is one unit. With `t.Run`, each row is independently reported:

```console
$ go test -v -run TestAdd_Table
=== RUN   TestAdd_Table
=== RUN   TestAdd_Table/two_positives
=== RUN   TestAdd_Table/with_zero
=== RUN   TestAdd_Table/two_negatives
=== RUN   TestAdd_Table/mixed_signs
--- PASS: TestAdd_Table (0.00s)
    --- PASS: TestAdd_Table/two_positives (0.00s)
    --- PASS: TestAdd_Table/with_zero (0.00s)
    --- PASS: TestAdd_Table/two_negatives (0.00s)
    --- PASS: TestAdd_Table/mixed_signs (0.00s)
PASS
```

Notice the framework **replaced spaces in the case name with underscores** — `two positives` became `two_positives` — so the name is usable on the command line. That gives you the second superpower: **run one case in isolation** with `-run`, using `/` to descend into subtests:

```console
$ go test -run 'TestAdd_Table/two_negatives'   # run ONLY that one row
$ go test -run 'TestAdd_Table/negative'        # regex: any subtest containing "negative"
```

When a table test fails in CI, you copy the `TestFoo/case_name` path straight from the output into `-run` and iterate on that single case locally. This is why you should always give table rows descriptive names and never leave `name` empty.

### 3.3 t.Parallel — running tests concurrently for speed **[B/I]**

By default Go runs the tests **within a single package sequentially**, one after another. For fast unit tests that is fine. But when tests involve waiting — a real HTTP call, a database round-trip, a deliberate `time.Sleep` — running them one at a time wastes wall-clock time they spend blocked. `t.Parallel()` tells the framework "this test is safe to run at the same time as other parallel tests," letting the suite overlap their waiting.

Here is the mechanism, because it is subtle. When a test calls `t.Parallel()`, it **pauses** at that point and yields; the test function does not continue. Once the *enclosing* test function (or the package) has launched all its tests, the framework **resumes all the paused parallel ones together**, up to a concurrency limit (`GOMAXPROCS` by default, tunable with `-parallel N`). So `t.Parallel()` is best read as "signal that I'm parallel, then wait my turn to be run alongside my siblings."

```go
func TestParallelExample(t *testing.T) {
	tests := []struct {
		name string
		in   string
		want string
	}{
		{"lowercase", "abc", "ABC"},
		{"mixed", "aBc", "ABC"},
		{"already upper", "ABC", "ABC"},
	}

	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			t.Parallel() // this subtest may run alongside the others

			got := strings.ToUpper(tc.in)
			if got != tc.want {
				t.Errorf("ToUpper(%q) = %q; want %q", tc.in, got, tc.want)
			}
		})
	}
}
```

Two rules keep parallel tests from becoming flaky:

1. **Parallel tests must be isolated.** If two tests that run concurrently both touch the same global variable, the same file, or the same database row, you get a data race or nondeterministic result. Run parallel suites under `-race` (§8) to catch shared-state bugs. This is fast/isolated/deterministic in action: parallelism *forces* isolation, which is why it is a good discipline even when you don't need the speed.
2. **The loop-variable trap (pre-Go 1.22).** This one bit thousands of people. Before Go 1.22, the loop variable `tc` was **shared and reused** across iterations. Because a parallel subtest *pauses* and runs *later* — after the loop has finished and moved `tc` to its final value — every parallel subtest would see the **last** row's data. The classic fix was to shadow the variable inside the loop with `tc := tc`.

> **⚡ Go 1.22+ fixed loop-variable capture.** As of **Go 1.22** each loop iteration gets its **own** copy of the loop variable, so the `tc := tc` shadow is **no longer needed** and modern code omits it. If you see `tc := tc` (or `tt := tt`) at the top of a subtest loop in older code or tutorials, that is the pre-1.22 workaround — harmless but obsolete. If you must support a `go.mod` declaring `go 1.21` or earlier, keep the shadow. This guide targets 1.25/1.26, so we omit it.

### 3.4 Table-driven tests for error cases **[I]**

Real functions return errors, and your table should cover both the happy path and the failures. The pattern is to add a `wantErr` field. For simple "did it error at all" checks a `bool` suffices; when you care *which* error, compare with `errors.Is` against a sentinel or check the message. Testing error paths is where a lot of real bugs hide, so give them first-class rows.

```go
func Divide(a, b int) (int, error) {
	if b == 0 {
		return 0, fmt.Errorf("divide %d by zero: %w", a, ErrDivByZero)
	}
	return a / b, nil
}

func TestDivide(t *testing.T) {
	tests := []struct {
		name    string
		a, b    int
		want    int
		wantErr error // the sentinel we expect via errors.Is, or nil for success
	}{
		{name: "ok", a: 10, b: 2, want: 5, wantErr: nil},
		{name: "by zero", a: 1, b: 0, want: 0, wantErr: ErrDivByZero},
	}

	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			got, err := Divide(tc.a, tc.b)

			// errors.Is walks the wrapped error chain looking for the target,
			// so it works even though Divide wraps ErrDivByZero with %w.
			if !errors.Is(err, tc.wantErr) {
				t.Fatalf("Divide(%d,%d) error = %v; want errors.Is …%v", tc.a, tc.b, err, tc.wantErr)
			}
			// Only check the value on the success path.
			if tc.wantErr == nil && got != tc.want {
				t.Errorf("Divide(%d,%d) = %d; want %d", tc.a, tc.b, got, tc.want)
			}
		})
	}
}
```

---

## 4. Assertions — Stdlib versus testify

### 4.1 The stdlib philosophy — no assertion library **[B]**

Coming from JUnit, RSpec, or Jest, the first thing you look for in Go is `assertEquals`. It is not there, and that is on purpose. The Go team argues that an assertion is just an `if` plus a `t.Error`, and that a dedicated assertion DSL hides control flow and tempts people to write cryptic one-liners. So the *idiomatic standard-library* way to assert is exactly what you have already seen:

```go
if got != want {
	t.Errorf("Foo() = %v; want %v", got, want)
}
```

For comparing things `==` can't handle — slices, maps, structs with nested fields — the standard tools are **`reflect.DeepEqual`** (works on anything, but is loose about types and unexported fields) and, better, **`github.com/google/go-cmp/cmp`**, which the Go team recommends for test comparisons because it produces a readable *diff* of what differed and lets you configure how to compare:

```go
import "github.com/google/go-cmp/cmp"

func TestBuildUser(t *testing.T) {
	got := BuildUser("Ada", 36)
	want := User{Name: "Ada", Age: 36, Roles: []string{"admin"}}

	// cmp.Diff returns "" when equal, or a human-readable diff when not.
	// This is far more useful than "got X want Y" for big structs: it points
	// at the exact field that differs.
	if diff := cmp.Diff(want, got); diff != "" {
		t.Errorf("BuildUser mismatch (-want +got):\n%s", diff)
	}
}
```

`cmp.Diff` is the stdlib-friendly workhorse for structured data and is worth adopting even if you never touch testify. Note it **panics on unexported fields** unless you tell it how to handle them (`cmpopts.IgnoreUnexported` or an `Equal` method) — a deliberate nudge to compare through public API.

### 4.2 testify — the community assertion library **[B/I]**

Despite the stdlib philosophy, the most-used testing dependency in the Go ecosystem by far is **testify** (`github.com/stretchr/testify`). It exists because, in practice, large suites full of `if got != want { t.Errorf(...) }` are verbose, and testify collapses each into a single readable line with a good failure message for free. It is not "un-idiomatic" — it is pragmatic, and it dominates real codebases. This guide teaches **both** so you can read any project and choose per-team.

testify has three sub-packages you will use:

| Package | Import | Purpose |
|---|---|---|
| `assert` | `github.com/stretchr/testify/assert` | Assertions that **report and continue** (like `t.Error`) |
| `require` | `github.com/stretchr/testify/require` | Assertions that **report and stop** the test (like `t.Fatal`) |
| `suite` | `github.com/stretchr/testify/suite` | xUnit-style test structs with setup/teardown methods |

The single most important testify decision is **`assert` versus `require`**, and it maps exactly onto the `Error`-versus-`Fatal` distinction from §2.2:

- **`require`** stops the test on failure. Use it for **preconditions**: if `require.NoError(t, err)` fails, the value is unusable and the next line would panic, so stop.
- **`assert`** records the failure and keeps going. Use it for **independent final checks**: assert several fields and see all the wrong ones in one run.

```go
package user_test

import (
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

func TestFetchUser_Testify(t *testing.T) {
	u, err := FetchUser(42)

	// PRECONDITION: if this errored, u is nil and every line below panics.
	// require => stop here on failure.
	require.NoError(t, err)
	require.NotNil(t, u)

	// INDEPENDENT CHECKS: we want to see ALL mismatches, so assert => continue.
	assert.Equal(t, 42, u.ID)
	assert.Equal(t, "Ada Lovelace", u.Name)
	assert.True(t, u.Active)
	assert.Len(t, u.Roles, 2)
	// Every assert takes an optional trailing message + args for context:
	assert.Equal(t, "admin", u.Roles[0], "first role should be admin for user %d", u.ID)
}
```

> **⚡ testify argument order gotcha.** testify assertions are `assert.Equal(t, expected, actual)` — **expected first, actual second**. This is the *opposite* of the stdlib `got`/`want` mental order and a perennial source of backwards failure messages ("expected 5, got 3" when it's really the reverse). Burn `Equal(t, want, got)` into memory. When a testify diff looks inverted, this is almost always why.

### 4.3 The most useful testify assertions **[I]**

testify has dozens of assertions; these are the ones you will actually reach for. Each exists in both `assert.` and `require.` form.

| Assertion | Passes when |
|---|---|
| `Equal(t, want, got)` | `want` deeply equals `got` (uses `ObjectsAreEqual`/reflect) |
| `EqualValues(t, want, got)` | Equal after type conversion (e.g. `int32(5)` vs `int(5)`) |
| `NoError(t, err)` / `Error(t, err)` | `err` is nil / non-nil |
| `ErrorIs(t, err, target)` | `errors.Is(err, target)` is true (wrapped-error aware) |
| `ErrorContains(t, err, "sub")` | error message contains substring |
| `True(t, b)` / `False(t, b)` | boolean is true / false |
| `Nil(t, v)` / `NotNil(t, v)` | value is / isn't nil |
| `Len(t, coll, n)` | `len(coll) == n` |
| `Contains(t, coll, elem)` | slice/map/string contains the element/substring |
| `ElementsMatch(t, a, b)` | same elements ignoring order |
| `Empty(t, v)` / `NotEmpty(t, v)` | zero value / non-zero |
| `Panics(t, fn)` / `NotPanics(t, fn)` | calling `fn` does / doesn't panic |
| `InDelta(t, want, got, δ)` | floats within δ (never compare floats with `Equal`) |
| `JSONEq(t, wantJSON, gotJSON)` | two JSON strings are semantically equal |

`assert.Equal` uses reflect-based deep equality under the hood, so it handles slices, maps, and nested structs without ceremony — that is a big part of its appeal over hand-rolled comparisons. For floats, always use `InDelta`; `Equal` on floats fails on rounding.

### 4.4 testify suites — xUnit-style setup and teardown **[I]**

Sometimes a group of tests share expensive setup — a database handle, a temp directory, a seeded fixture. The `suite` package gives you a *struct* whose methods are tests, with lifecycle hooks (`SetupSuite`, `SetupTest`, `TearDownTest`, `TearDownSuite`) that run around them. This is the familiar xUnit shape and is handy when many tests share state, though for most Go code the plain `TestMain`/`t.Cleanup` approach (§5) is lighter. Use suites when the shared setup genuinely warrants a struct to hang it on.

```go
package repo_test

import (
	"testing"

	"github.com/stretchr/testify/suite"
)

// A suite is a struct embedding suite.Suite (which gives it Assert/Require
// helpers and a *testing.T). Fields hold shared state.
type UserRepoSuite struct {
	suite.Suite
	repo *UserRepo // shared across the suite's tests
}

// SetupSuite runs ONCE before any test in the suite — expensive one-time setup.
func (s *UserRepoSuite) SetupSuite() {
	s.repo = NewUserRepo( /* ...open a connection... */ )
}

// SetupTest runs before EACH test method — per-test isolation (reset state).
func (s *UserRepoSuite) SetupTest() {
	s.repo.Truncate() // start each test from a clean slate
}

// Any method named TestXxx on the suite is a test. Use s.Require()/s.Assert()
// or the promoted s.NoError/s.Equal helpers from the embedded Suite.
func (s *UserRepoSuite) TestCreateThenGet() {
	id, err := s.repo.Create("Ada")
	s.Require().NoError(err)

	got, err := s.repo.Get(id)
	s.Require().NoError(err)
	s.Equal("Ada", got.Name)
}

// A single ordinary Test function is the bridge that hands the suite to
// `go test`. Without this, go test never discovers the methods above.
func TestUserRepoSuite(t *testing.T) {
	suite.Run(t, new(UserRepoSuite))
}
```

### 4.5 Which to use — a recommendation **[I]**

There is no universally right answer, but a defensible default: **use the standard library (`if`/`t.Error`) plus `cmp.Diff` for structured comparisons in libraries and small projects, and adopt testify's `assert`/`require` in application/service code where the density of assertions makes the terseness pay off.** The most important thing is **consistency within a codebase** — pick one and stick to it, because a file mixing three assertion styles is harder to read than any single style. Whatever you choose, keep failure messages informative; the tool is secondary to the message.

---

## 5. Test Lifecycle Setup Teardown and Golden Files

### 5.1 The problem lifecycle hooks solve **[I]**

Real tests need to *set things up* before they run and *clean things up* after: open a temp directory, start a fake server, seed a database, and then tear it all down so the next test starts clean. Do this badly and you get the two classic failure modes: **leaked resources** (a temp file or goroutine that outlives the test and pollutes the next one) and **order-dependence** (test B only passes because test A happened to leave state behind). Go gives you four precise tools — `TestMain`, `t.Cleanup`, `t.Helper`, and `t.TempDir` — that make setup/teardown correct *and* impossible to forget. Master these and your tests stay isolated and deterministic even as they grow.

### 5.2 t.Cleanup — teardown that can't be forgotten **[I]**

The old way to clean up was `defer`. It works, but it has two weaknesses: a `defer` in a *helper* function runs when the helper returns, not when the test ends (so you can't set up in a helper and tear down at test end), and multiple `defer`s scatter the teardown logic. **`t.Cleanup(fn)`** fixes both: it registers `fn` to run when the test *and all its subtests* finish, in **last-registered-first-run** (LIFO) order, no matter how the test exits — normal return, `t.Fatal`, or panic. Crucially, cleanups registered inside a helper still fire at the *test's* end, so you can pair "create resource" with "destroy resource" right next to each other inside a helper and trust it.

```go
func TestWithTempServer(t *testing.T) {
	// Start a resource...
	srv := startTestServer(t)

	// ...and register its teardown immediately, so the setup and teardown
	// live side by side and teardown ALWAYS runs — even if an assertion
	// below calls t.Fatal.
	t.Cleanup(func() {
		srv.Close()
	})

	resp, err := http.Get(srv.URL + "/health")
	if err != nil {
		t.Fatalf("GET failed: %v", err) // srv.Close() still runs via Cleanup
	}
	_ = resp.Body.Close()
}

// A helper that both creates a resource AND schedules its cleanup. The caller
// gets a ready-to-use thing and never has to remember to tear it down.
func mustTempFile(t *testing.T) *os.File {
	t.Helper()
	f, err := os.CreateTemp("", "test-*.txt")
	if err != nil {
		t.Fatalf("CreateTemp: %v", err)
	}
	// This runs at the END OF THE TEST, not when mustTempFile returns.
	t.Cleanup(func() {
		f.Close()
		os.Remove(f.Name())
	})
	return f
}
```

### 5.3 t.Helper — clean failure locations **[I]**

When you factor assertions into a helper function, failures reported inside it point at the *helper's* line number, not the test line that called it — so every failure looks like it came from the same place and you can't tell which call failed. **`t.Helper()`**, called as the first line of a helper, tells the framework "don't blame me in failure output; blame my caller." The result is that `t.Errorf` inside the helper reports the file and line of the *test* that invoked it. Any function that takes a `*testing.T` and might fail should call `t.Helper()` first.

```go
// requireStatus is a custom assertion helper. Because of t.Helper(), a failure
// is reported at the CALLER's line, so you can see which check failed.
func requireStatus(t *testing.T, got, want int) {
	t.Helper() // <-- must be the first statement
	if got != want {
		t.Fatalf("status = %d; want %d", got, want)
	}
}
```

### 5.4 t.TempDir — automatic per-test scratch space **[I]**

Tests that touch the filesystem should never write to a fixed path (two parallel tests would collide) or into the repo. **`t.TempDir()`** returns a **fresh, unique, empty directory** for this test and **automatically deletes it** (via an internal `t.Cleanup`) when the test ends. It is the correct, zero-boilerplate way to give a test a private sandbox on disk. Each call — including each subtest — gets its own directory, so it is inherently parallel-safe.

```go
func TestWriteReport(t *testing.T) {
	dir := t.TempDir() // unique dir, auto-removed at test end

	path := filepath.Join(dir, "report.txt")
	if err := WriteReport(path, "hello"); err != nil {
		t.Fatalf("WriteReport: %v", err)
	}

	got, err := os.ReadFile(path)
	if err != nil {
		t.Fatalf("ReadFile: %v", err)
	}
	if string(got) != "hello" {
		t.Errorf("report = %q; want %q", got, "hello")
	}
	// No cleanup code needed — t.TempDir handles removal.
}
```

### 5.5 TestMain — package-wide setup and teardown **[I]**

Sometimes setup is too expensive to do per-test and belongs *once per package* — spin up a shared database container, compile an asset, set an env var. **`TestMain(m *testing.M)`** is a special function the tool calls *instead of* running tests directly; you do your global setup, call `m.Run()` (which runs all the tests and returns an exit code), do your teardown, then `os.Exit` with that code. If you define `TestMain`, **you own** the run — forgetting `m.Run()` means no tests execute, and forgetting to propagate its exit code means CI never sees failures.

```go
package integration_test

import (
	"os"
	"testing"
)

var testDB *sql.DB // shared across the whole package's tests

func TestMain(m *testing.M) {
	// --- global SETUP: runs once before ANY test in this package ---
	db, cleanup, err := startSharedDatabase()
	if err != nil {
		fmt.Fprintf(os.Stderr, "setup failed: %v\n", err)
		os.Exit(1) // non-zero: CI sees the failure
	}
	testDB = db

	// Run every test in the package; code returns the exit code (0 = all pass).
	code := m.Run()

	// --- global TEARDOWN: runs once after ALL tests, pass or fail ---
	cleanup() // NOTE: deferred funcs do NOT run after os.Exit, so call directly

	os.Exit(code) // MUST propagate the code or failures are invisible
}
```

> **⚡ `defer` does not run after `os.Exit`.** Because `TestMain` ends in `os.Exit(code)`, any `defer cleanup()` you wrote will be **skipped**. Call teardown *before* `os.Exit`, as above. This is the single most common `TestMain` bug — a container that never gets stopped.

### 5.6 Fixtures, testdata, and golden files **[I]**

A **fixture** is fixed input data a test feeds to the code. Small fixtures live inline in the table; large ones — a 5KB sample JSON payload, a binary file, an HTML page — live in the `testdata/` directory (§2.6), which the Go tool ignores. You load them with `os.ReadFile("testdata/whatever")`.

A **golden file** is the *flip side*: instead of hand-writing the expected output for a function that produces something big (a rendered template, a formatted report, a marshaled document), you let the test **write the actual output to a `.golden` file the first time**, eyeball it once to confirm it's correct, commit it, and thereafter the test **compares** its output against that committed golden file. When behavior legitimately changes, you re-run with an `-update` flag to regenerate the golden and review the diff in code review. This is the standard Go pattern (the `gofmt` and template test suites use it) for "output too large to write by hand but easy to verify by eye."

```go
// A package-level flag: `go test -update` regenerates golden files instead
// of asserting against them. Define it once per package.
var update = flag.Bool("update", false, "update golden files")

func TestRenderInvoice(t *testing.T) {
	got := RenderInvoice(sampleInvoice) // produces a big formatted string

	golden := filepath.Join("testdata", "invoice.golden")

	if *update {
		// Regeneration mode: write the current output as the new expectation.
		if err := os.WriteFile(golden, []byte(got), 0o644); err != nil {
			t.Fatalf("write golden: %v", err)
		}
		t.Logf("updated golden file %s", golden)
	}

	// Assertion mode: compare against the committed golden file.
	want, err := os.ReadFile(golden)
	if err != nil {
		t.Fatalf("read golden (run `go test -update` to create it): %v", err)
	}
	if got != string(want) {
		t.Errorf("RenderInvoice output differs from golden.\n"+
			"Run `go test -update` if the change is intentional.\n"+
			"got:\n%s\nwant:\n%s", got, want)
	}
}
```

The golden-file discipline pays off enormously for code that generates text or documents: adding a test case costs nothing (the golden is auto-generated), and an unintended change to output shows up as a failing diff you must consciously accept with `-update`.

---

## 6. Test Doubles — Fakes Stubs Spies and Mocks

### 6.1 Why you need test doubles and what a "seam" is **[I]**

A unit test must be *fast* and *isolated*, but real code talks to slow, nondeterministic things: databases, HTTP APIs, the clock, the filesystem, a payment gateway. You cannot let a unit test hit a live payment API — it would be slow, flaky, and possibly expensive. A **test double** is a stand-in you substitute for a real dependency during a test: it looks like the real thing to your code but is controllable and instant. The general term "test double" (from the movie stunt-double) covers several specific kinds.

For a double to be substitutable, your code must depend on an **interface**, not a concrete type — the interface is the **seam** where you can swap the real implementation for a double. This is the deepest design lesson in this section: *testable code and well-decoupled code are the same thing.* If a function reaches out and calls `http.Get` or `sql.Open` directly, there is no seam and you cannot unit-test it. If instead it accepts an interface, you can pass the real one in production and a double in the test. Design for the seam first; the test follows.

```go
// The seam: your code depends on THIS interface, not on a concrete DB or API.
type UserStore interface {
	GetUser(ctx context.Context, id int) (*User, error)
	SaveUser(ctx context.Context, u *User) error
}

// The service under test accepts the interface (dependency injection). In
// production you pass a real Postgres-backed UserStore; in tests, a double.
type UserService struct {
	store UserStore
}

func NewUserService(store UserStore) *UserService {
	return &UserService{store: store}
}

func (s *UserService) Promote(ctx context.Context, id int) error {
	u, err := s.store.GetUser(ctx, id)
	if err != nil {
		return fmt.Errorf("promote: %w", err)
	}
	u.Role = "admin"
	return s.store.SaveUser(ctx, u)
}
```

### 6.2 The five kinds of double **[I]**

The terms get used loosely, but they mean distinct things and knowing them helps you pick the right one:

| Double | What it is | Use when |
|---|---|---|
| **Dummy** | A value passed only to satisfy a signature, never used | Filling a required arg you don't care about |
| **Stub** | Returns hard-coded answers to calls | You need the dependency to *return* something specific |
| **Spy** | A stub that also *records* how it was called | You need to assert the code *called* it correctly |
| **Fake** | A real, working, but simplified implementation (e.g. in-memory DB) | You want realistic behavior without the real dependency |
| **Mock** | A double pre-programmed with *expectations*, verified at the end | You want to assert exact interactions strictly |

The practically important split is **fakes versus mocks**, because it reflects two testing philosophies (§6.6). A **fake** behaves like the real thing — an in-memory map that acts as a database — and you test the *end result* (state). A **mock** is programmed with "I expect `SaveUser` to be called once with this argument" and *fails if the interaction doesn't match* — you test the *behavior* (interactions). Both are valid; overusing mocks is a common trap.

### 6.3 Hand-written fakes and spies **[I]**

For most interfaces, the simplest and most maintainable double is one you **write by hand** — no code generator, no library. A hand-written fake is just a struct implementing the interface with in-memory logic. It is readable, refactors with your code, and doubles as living documentation of how the interface behaves.

```go
// InMemoryUserStore is a hand-written FAKE: a real, working UserStore backed
// by a map instead of a database. It's fast, deterministic, and needs no I/O.
type InMemoryUserStore struct {
	mu    sync.Mutex // real map access is guarded so it's parallel-safe
	users map[int]*User

	// Spy fields: record interactions so tests can assert on them.
	SaveCalls int
}

func NewInMemoryUserStore() *InMemoryUserStore {
	return &InMemoryUserStore{users: make(map[int]*User)}
}

func (s *InMemoryUserStore) GetUser(_ context.Context, id int) (*User, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	u, ok := s.users[id]
	if !ok {
		return nil, ErrNotFound // behaves like a real store's miss
	}
	// Return a copy so callers can't mutate our internal state — subtle but
	// important for test isolation.
	cp := *u
	return &cp, nil
}

func (s *InMemoryUserStore) SaveUser(_ context.Context, u *User) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.SaveCalls++ // spy: count how many times we were asked to save
	cp := *u
	s.users[u.ID] = &cp
	return nil
}

func TestPromote_WithFake(t *testing.T) {
	store := NewInMemoryUserStore()
	// Seed the fake with realistic starting state.
	_ = store.SaveUser(context.Background(), &User{ID: 1, Role: "member"})
	store.SaveCalls = 0 // reset the spy after seeding

	svc := NewUserService(store)
	if err := svc.Promote(context.Background(), 1); err != nil {
		t.Fatalf("Promote: %v", err)
	}

	// STATE assertion (the fake way): check the end result.
	got, _ := store.GetUser(context.Background(), 1)
	if got.Role != "admin" {
		t.Errorf("role = %q; want admin", got.Role)
	}
	// INTERACTION assertion (the spy way): check it saved exactly once.
	if store.SaveCalls != 1 {
		t.Errorf("SaveCalls = %d; want 1", store.SaveCalls)
	}
}
```

A **stub** that just needs to return a canned value (or error) is even simpler — often a struct with function fields you set per test:

```go
// FuncStore is a configurable stub/spy: set the func fields per test to
// control behavior, including forcing errors to test failure paths.
type FuncStore struct {
	GetUserFn  func(ctx context.Context, id int) (*User, error)
	SaveUserFn func(ctx context.Context, u *User) error
}

func (f FuncStore) GetUser(ctx context.Context, id int) (*User, error) {
	return f.GetUserFn(ctx, id)
}
func (f FuncStore) SaveUser(ctx context.Context, u *User) error {
	return f.SaveUserFn(ctx, u)
}

func TestPromote_GetError(t *testing.T) {
	// Force GetUser to fail so we can test Promote's error handling — trivial
	// with a stub, impossible with a real DB without contriving a failure.
	store := FuncStore{
		GetUserFn: func(context.Context, int) (*User, error) {
			return nil, ErrNotFound
		},
	}
	svc := NewUserService(store)
	err := svc.Promote(context.Background(), 999)
	if !errors.Is(err, ErrNotFound) {
		t.Errorf("err = %v; want ErrNotFound", err)
	}
}
```

### 6.4 Generated mocks with go.uber.org/mock (gomock) **[I]**

When an interface is large or you need strict interaction checking with argument matchers, hand-writing doubles gets tedious. **gomock** generates a mock implementation from your interface and gives you a fluent API to set expectations: "expect `GetUser` called once with id 1, return this user." The canonical tool today is **`go.uber.org/mock`** — Uber's maintained fork that replaced the archived `github.com/golang/mock`.

> **⚡ Use `go.uber.org/mock`, not `github.com/golang/mock`.** Google archived the original `golang/mock` in 2023; the community-maintained continuation is `go.uber.org/mock` with the generator binary `mockgen`. The API is identical, so old tutorials' *code* still applies — only the import paths changed. Install with `go install go.uber.org/mock/mockgen@latest`.

You generate mocks with a `//go:generate` directive next to the interface, then run `go generate ./...`:

```go
//go:generate mockgen -source=store.go -destination=mock_store_test.go -package=user_test

type UserStore interface {
	GetUser(ctx context.Context, id int) (*User, error)
	SaveUser(ctx context.Context, u *User) error
}
```

The generated `MockUserStore` is then wired up in a test like this:

```go
func TestPromote_WithGomock(t *testing.T) {
	ctrl := gomock.NewController(t) // ctrl verifies expectations at test end
	mockStore := NewMockUserStore(ctrl)

	ctx := context.Background()
	existing := &User{ID: 1, Role: "member"}

	// EXPECTATION 1: GetUser(ctx, 1) is called exactly once, return `existing`.
	// gomock.Any() matches any context; the 1 must match exactly.
	mockStore.EXPECT().
		GetUser(gomock.Any(), 1).
		Return(existing, nil).
		Times(1)

	// EXPECTATION 2: SaveUser is called once with a user whose Role == "admin".
	// A custom matcher asserts on the argument's content.
	mockStore.EXPECT().
		SaveUser(gomock.Any(), gomock.Cond(func(u *User) bool {
			return u.Role == "admin"
		})).
		Return(nil).
		Times(1)

	svc := NewUserService(mockStore)
	if err := svc.Promote(ctx, 1); err != nil {
		t.Fatalf("Promote: %v", err)
	}
	// When the test ends, ctrl (registered via NewController(t)) automatically
	// fails the test if any EXPECT was not satisfied. No manual Finish needed
	// in modern gomock when you pass t to NewController.
}
```

> **⚡ `gomock.NewController(t)` auto-finishes since v1.5+.** Old tutorials show `defer ctrl.Finish()`. Passing the `*testing.T` to `NewController` registers the verification via `t.Cleanup` automatically, so the explicit `defer ctrl.Finish()` is no longer required (it's harmless if present). `gomock.Cond` is the modern content matcher.

An alternative generator, **mockery** (`github.com/vektra/mockery`), produces testify-style mocks driven by a `.mockery.yaml` config rather than per-interface directives, and integrates with `testify/mock`. Teams that already standardize on testify often prefer it; the concepts are identical.

### 6.5 When to mock and when NOT to **[I]**

The most damaging testing anti-pattern is **over-mocking**: mocking everything until the test only verifies that your code calls the mocks in the order you wrote it to. Such a test is a mirror — it asserts the implementation against itself, passes even when the real integration is broken, and breaks on every harmless refactor. The guidance:

- **Mock at architectural boundaries you don't own** — a third-party payment API, an email sender, a clock. These are slow/external and you want to control them.
- **Prefer a fake over a mock for things you'll test heavily** (your own repository interface): a fake is reusable across many tests and asserts *state*, which couples less to implementation.
- **Do not mock the database if what you're testing IS the database interaction.** A mocked `sql.DB` proves nothing about whether your SQL is correct. Use a real Postgres in a container (§12) instead. This is the trophy philosophy: mock the fringes, integrate the core.

### 6.6 State verification versus behavior verification **[I]**

Stepping back: fakes lead to **state verification** ("after calling Promote, the stored user's role is admin") and mocks lead to **behavior verification** ("Promote called SaveUser once with an admin user"). State verification is more robust — it tests *what the user observes* and survives refactoring — so prefer it by default. Behavior verification is necessary when there is no observable state to check: you *must* assert that "send email" was called, because the whole point of the code is the side effect and there's no return value to inspect. Choose the style that matches what actually matters about the code, and lean toward state.

---

## 7. HTTP Testing with httptest and Gin

### 7.1 Why HTTP needs special tooling **[I]**

Most Go you write in a service is an HTTP handler, and handlers are exactly the code you most want to test — they parse requests, call business logic, and shape responses, and any of those can go wrong. The naive approach is to start a real server on a real port and fire real requests at it, but that is slow, needs a free port, and is racy. The standard library's **`net/http/httptest`** package solves this with two tools that make handler testing fast and deterministic: **`httptest.NewRecorder`** (test a handler with *no network at all*) and **`httptest.NewServer`** (a real server on a random loopback port for when you need the full stack). Understanding when to use each is the key.

### 7.2 ResponseRecorder — testing a handler with no network **[I]**

A Go HTTP handler is just a function `func(http.ResponseWriter, *http.Request)`. You do not need a network to call it — you can construct a request in memory and pass a fake `ResponseWriter` that records what the handler writes. **`httptest.NewRecorder()`** returns exactly that fake writer: an `*httptest.ResponseRecorder` that captures the status code, headers, and body. This is the fastest, most isolated way to test a handler — no ports, no sockets, microseconds per test.

```go
// The handler under test: a plain net/http handler.
func HealthHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	_, _ = w.Write([]byte(`{"status":"ok"}`))
}

func TestHealthHandler(t *testing.T) {
	// Build an in-memory request. The nil body is fine for GET. The URL path
	// is what the handler sees; the "target" can be a full URL or just a path.
	req := httptest.NewRequest(http.MethodGet, "/health", nil)

	// The recorder stands in for the real ResponseWriter and captures output.
	rec := httptest.NewRecorder()

	// Call the handler DIRECTLY — no server, no network.
	HealthHandler(rec, req)

	// rec.Result() gives an *http.Response built from what the handler wrote.
	res := rec.Result()
	defer res.Body.Close()

	if res.StatusCode != http.StatusOK {
		t.Errorf("status = %d; want %d", res.StatusCode, http.StatusOK)
	}
	if ct := res.Header.Get("Content-Type"); ct != "application/json" {
		t.Errorf("Content-Type = %q; want application/json", ct)
	}
	body, _ := io.ReadAll(res.Body)
	if !strings.Contains(string(body), `"status":"ok"`) {
		t.Errorf("body = %s; want it to contain status ok", body)
	}
}
```

### 7.3 Testing requests with bodies and JSON **[I]**

Handlers that accept `POST` bodies need a request *with* a body and the right `Content-Type`. Build the body as an `io.Reader` (commonly `strings.NewReader` or `bytes.NewReader` of marshaled JSON), and assert on the decoded response rather than raw string matching so your test doesn't break on whitespace or field order.

```go
func TestCreateUserHandler(t *testing.T) {
	// Marshal the request payload to JSON and wrap it in a reader.
	payload := `{"name":"Ada","email":"ada@example.com"}`
	req := httptest.NewRequest(http.MethodPost, "/users", strings.NewReader(payload))
	req.Header.Set("Content-Type", "application/json")

	rec := httptest.NewRecorder()
	CreateUserHandler(rec, req)

	res := rec.Result()
	defer res.Body.Close()

	if res.StatusCode != http.StatusCreated {
		t.Fatalf("status = %d; want 201", res.StatusCode)
	}

	// Decode the response body into a struct and assert on FIELDS, not text.
	var got struct {
		ID    int    `json:"id"`
		Name  string `json:"name"`
		Email string `json:"email"`
	}
	if err := json.NewDecoder(res.Body).Decode(&got); err != nil {
		t.Fatalf("decode response: %v", err)
	}
	if got.Name != "Ada" {
		t.Errorf("name = %q; want Ada", got.Name)
	}
	if got.ID == 0 {
		t.Error("expected a non-zero assigned ID")
	}
}
```

### 7.4 Testing middleware **[I]**

Middleware wraps a handler to add cross-cutting behavior — auth, logging, rate limiting. You test it by wrapping a tiny *spy handler* that records whether it was reached and with what context, then asserting the middleware let the request through (or blocked it) correctly. This isolates the middleware's decision logic from any real downstream handler.

```go
// AuthMiddleware rejects requests without a valid bearer token.
func AuthMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.Header.Get("Authorization") != "Bearer secret" {
			http.Error(w, "unauthorized", http.StatusUnauthorized)
			return
		}
		next.ServeHTTP(w, r)
	})
}

func TestAuthMiddleware(t *testing.T) {
	tests := []struct {
		name       string
		token      string
		wantStatus int
		wantCalled bool // did the request reach the inner handler?
	}{
		{"valid token", "Bearer secret", http.StatusOK, true},
		{"missing token", "", http.StatusUnauthorized, false},
		{"wrong token", "Bearer nope", http.StatusUnauthorized, false},
	}

	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			called := false
			// Spy inner handler: records that it was reached.
			inner := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
				called = true
				w.WriteHeader(http.StatusOK)
			})

			req := httptest.NewRequest(http.MethodGet, "/", nil)
			if tc.token != "" {
				req.Header.Set("Authorization", tc.token)
			}
			rec := httptest.NewRecorder()

			AuthMiddleware(inner).ServeHTTP(rec, req)

			if rec.Code != tc.wantStatus {
				t.Errorf("status = %d; want %d", rec.Code, tc.wantStatus)
			}
			if called != tc.wantCalled {
				t.Errorf("inner called = %v; want %v", called, tc.wantCalled)
			}
		})
	}
}
```

### 7.5 Testing a Gin app in-process **[I]**

The sibling [Gin guide](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) builds a real REST API. Gin's `*gin.Engine` implements `http.Handler`, so you test it the same way — build the engine with all its routes and middleware, then serve in-memory requests through `engine.ServeHTTP(rec, req)`. This exercises the *entire* routing and middleware chain in-process, with no network, which is the sweet spot for testing a Gin handler: realistic (real router, real binding) yet fast and isolated.

```go
// Build the router the same way production does, but inject a fake store so
// the test stays isolated from the database.
func newTestRouter(store UserStore) *gin.Engine {
	gin.SetMode(gin.TestMode) // quiet Gin's debug logging in tests
	r := gin.New()
	h := &Handlers{store: store}
	r.POST("/users", h.CreateUser)
	r.GET("/users/:id", h.GetUser)
	return r
}

func TestGin_CreateAndGetUser(t *testing.T) {
	store := NewInMemoryUserStore() // the fake from §6
	router := newTestRouter(store)

	// --- CREATE ---
	body := `{"name":"Grace"}`
	req := httptest.NewRequest(http.MethodPost, "/users", strings.NewReader(body))
	req.Header.Set("Content-Type", "application/json")
	rec := httptest.NewRecorder()

	router.ServeHTTP(rec, req) // route through the FULL Gin engine

	if rec.Code != http.StatusCreated {
		t.Fatalf("create status = %d; want 201; body=%s", rec.Code, rec.Body.String())
	}

	var created struct {
		ID   int    `json:"id"`
		Name string `json:"name"`
	}
	if err := json.Unmarshal(rec.Body.Bytes(), &created); err != nil {
		t.Fatalf("unmarshal create: %v", err)
	}

	// --- GET the user we just created ---
	getReq := httptest.NewRequest(http.MethodGet, fmt.Sprintf("/users/%d", created.ID), nil)
	getRec := httptest.NewRecorder()
	router.ServeHTTP(getRec, getReq)

	if getRec.Code != http.StatusOK {
		t.Fatalf("get status = %d; want 200", getRec.Code)
	}
	if !strings.Contains(getRec.Body.String(), "Grace") {
		t.Errorf("get body = %s; want it to contain Grace", getRec.Body.String())
	}
}
```

### 7.6 httptest.NewServer — a real server, and faking outbound calls **[I]**

`ResponseRecorder` tests *your* handlers. The other direction — code that *makes* HTTP calls to some external API — needs the opposite tool: **`httptest.NewServer`** spins up a real HTTP server on a random localhost port that returns *whatever you program*, so you can point your client at it and control its responses (including errors, slow responses, and bad status codes) deterministically. This is how you test an API-client package without hitting the real third-party service.

```go
func TestWeatherClient_Fetch(t *testing.T) {
	// A fake upstream server that returns a canned weather payload.
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Assert the client sent the request we expect (spy behavior).
		if r.URL.Path != "/v1/current" {
			t.Errorf("client hit path %q; want /v1/current", r.URL.Path)
		}
		w.Header().Set("Content-Type", "application/json")
		_, _ = w.Write([]byte(`{"temp_c": 21.5}`))
	}))
	t.Cleanup(srv.Close) // shut the server down at test end

	// Point the client at the fake server's URL instead of the real API.
	client := NewWeatherClient(srv.URL)

	temp, err := client.CurrentTemp(context.Background(), "London")
	if err != nil {
		t.Fatalf("CurrentTemp: %v", err)
	}
	if temp != 21.5 {
		t.Errorf("temp = %v; want 21.5", temp)
	}
}
```

For this to work your client must let you *override its base URL* — another instance of the "design for a seam" rule from §6. A client that hard-codes `https://api.weather.com` cannot be pointed at the test server.

---

## 8. Coverage and the Race Detector

### 8.1 What coverage is and how to read it **[I]**

**Code coverage** measures which lines (more precisely, which *statements* and *branches*) of your production code were executed while the tests ran. It answers "what did my tests actually touch?" and it is the fastest way to find code you *forgot* to test. Go builds coverage into `go test`: add `-cover` for a summary percentage, or `-coverprofile` to write a detailed profile you can render as an annotated HTML report showing exactly which lines ran (green) and which didn't (red).

```console
$ go test -cover ./...
ok      example.com/calc     0.003s  coverage: 100.0% of statements
ok      example.com/user     0.010s  coverage: 78.4% of statements

# Write a profile, then view it as color-coded HTML in the browser:
$ go test -coverprofile=coverage.out ./...
$ go tool cover -html=coverage.out           # opens an annotated source view
$ go tool cover -func=coverage.out           # per-function coverage in the terminal
```

### 8.2 Coverage is a guide, not a goal **[I]**

Here is the crucial judgment call, because coverage numbers are routinely misused. **High coverage does not mean good tests, and 100% is rarely the right target.** Coverage tells you a line *ran*; it says nothing about whether you *asserted* the right thing about it — a test with zero assertions can hit 100% coverage and catch nothing. Worse, mandating a high coverage percentage in CI incentivizes people to write shallow tests that execute lines without checking behavior, or to test trivial getters to pad the number while the gnarly logic stays under-tested.

Use coverage the right way: as a **discovery tool**. Open the HTML report and look at the *red* lines — an untested error branch, a whole function nobody exercises — and ask "should this be tested?" Often the answer reveals a missing test for exactly the edge case that would have caught a real bug. Chase *meaningful* coverage of branching logic and error paths; don't chase a number. A pragmatic team target is "cover the code that matters and watch that coverage doesn't *drop*," not "hit 90%."

> **⚡ Coverage of integration tests (Go 1.20+).** Since Go 1.20 you can collect coverage from a *running binary* (integration/e2e tests), not just unit tests, by building with `go build -cover` and setting `GOCOVERDIR`. This lets you measure how much of your code your e2e suite exercises — useful for the trophy model where integration tests carry the load. See `go help build` and the `-coverpkg` flag to include packages beyond the one under test.

### 8.3 What a data race is **[I]**

A **data race** occurs when two goroutines access the *same* memory location *concurrently* and at least one of them is a *write*, with no synchronization (mutex, channel, atomic) ordering the accesses. The result is undefined behavior: corrupted values, torn reads, or crashes that appear only under load, on certain hardware, once a week in production — the worst kind of bug because it is nondeterministic and nearly impossible to reproduce by staring at the code. Concurrency is a headline Go feature, so data races are a real and present danger in Go code.

```go
// A textbook data race: two goroutines increment the same counter with no
// synchronization. `counter++` is read-modify-write — three steps that can
// interleave and lose updates. The final value is unpredictable.
func racy() int {
	counter := 0
	var wg sync.WaitGroup
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			counter++ // <-- DATA RACE: concurrent unsynchronized write
		}()
	}
	wg.Wait()
	return counter // often < 100, and varies run to run
}
```

### 8.4 The race detector — -race **[I]**

You cannot reliably find data races by reading code or by running tests normally — the racy program above often *passes* by luck. Go ships a **race detector** that instruments memory accesses at runtime and *reports* races the moment they actually happen during a run. You enable it with the **`-race`** flag, and the rule is simple and strong: **always run your test suite with `-race`.** It is the single highest-value testing flag in Go. It catches races that would otherwise reach production, and it works especially well with `t.Parallel()` tests, which naturally exercise concurrency.

```console
$ go test -race ./...
==================
WARNING: DATA RACE
Write at 0x00c0000b4008 by goroutine 8:
  example.com/pkg.racy.func1()
      /path/counter.go:12 +0x64
Previous write at 0x00c0000b4008 by goroutine 7:
  ...
==================
--- FAIL: TestRacy (0.01s)
    testing.go:1465: race detected during execution of test
FAIL
```

Two practical notes. First, `-race` makes tests run roughly 2–10x slower and use more memory, because of the instrumentation — so run it in CI on every push, and locally when touching concurrent code, but you needn't have it on for every quick local run. Second, the detector only reports races it *observes* during the run; it is not a proof of absence. Races on rarely-taken paths need tests that exercise those paths (and enough iterations) to surface. The fix for the example is a mutex, an atomic, or a channel:

```go
func fixed() int64 {
	var counter atomic.Int64 // atomic makes the increment a single indivisible op
	var wg sync.WaitGroup
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			counter.Add(1) // safe: no race
		}()
	}
	wg.Wait()
	return counter.Load() // always 100
}
```

---

## 9. Benchmarks and Profiling

### 9.1 What a benchmark is and why it's a test **[I]**

A **benchmark** measures *how fast* a piece of code runs and *how much memory* it allocates, rather than whether it is correct. Go treats benchmarks as first-class citizens of the same `testing` machinery: a benchmark is a function `BenchmarkXxx(b *testing.B)` in a `_test.go` file, run by `go test -bench`. You reach for benchmarks when performance is a feature — a hot path, a serializer, a parser — and you want *data* instead of guesses about whether a change made things faster or slower. Crucially, benchmarks make optimization **empirical**: you measure, change, measure again, and keep the change only if the numbers improve. Optimizing without benchmarking is superstition.

### 9.2 The b.N loop and the classic form **[I]**

The tricky part of benchmarking is that a single call to a fast function is too quick to time accurately. Go solves this by handing you `b.N` — a count the framework *chooses and increases automatically* until the benchmark runs long enough (by default ~1s) to produce a stable per-operation measurement. Your job in the classic form is to run the operation exactly `b.N` times in a loop. The framework reports nanoseconds *per operation* (`ns/op`), which is what you compare.

```go
func BenchmarkFibonacci(b *testing.B) {
	// The framework picks b.N and grows it until timing is stable. You must
	// run the operation b.N times — do NOT hard-code an iteration count.
	for i := 0; i < b.N; i++ {
		Fibonacci(20)
	}
}
```

```console
$ go test -bench=BenchmarkFibonacci -benchmem
BenchmarkFibonacci-16    260143    4602 ns/op    0 B/op    0 allocs/op
#                  ^cpus  ^b.N runs ^time/op      ^mem/op   ^allocs/op
```

The `-benchmem` flag adds the two most important numbers after time: **`B/op`** (bytes allocated per operation) and **`allocs/op`** (number of heap allocations per operation). Allocations are frequently the real performance story in Go — reducing `allocs/op` to zero on a hot path often matters more than shaving nanoseconds, because allocations create garbage-collector pressure. Always benchmark with `-benchmem`.

### 9.3 b.Loop — the modern form (Go 1.24+) **[I]**

> **⚡ `b.Loop` replaces the `for i := 0; i < b.N; i++` idiom in Go 1.24+.** The classic loop has two long-standing footguns: the compiler could *optimize away* the benchmarked call if its result was unused (making it look infinitely fast), and any per-iteration setup inside the loop got wrongly counted. **`for b.Loop()`**, added in **Go 1.24**, fixes both — it runs the body the right number of times, **prevents the compiler from eliminating** the calls inside it, and only times the loop body (setup before the loop is excluded automatically). Prefer `b.Loop` in all new code on Go 1.24+.

```go
func BenchmarkParseModern(b *testing.B) {
	input := buildLargeInput() // setup: NOT timed, and runs once

	// for b.Loop() is the new idiom. The result is implicitly kept "alive"
	// so the compiler can't delete the call as dead code.
	for b.Loop() {
		_ = Parse(input)
	}
}
```

### 9.4 ResetTimer, StopTimer, and excluding setup **[I]**

When your benchmark needs expensive setup that must run *inside* the function (because it depends on `b` or must happen per-run), you must keep that setup out of the measured time or your numbers are meaningless. The tools are **`b.ResetTimer()`** (zero the clock after setup), **`b.StopTimer()`/`b.StartTimer()`** (pause around per-iteration setup), and — as noted — `b.Loop` handles the common case for free. In the classic `b.N` form you still need them:

```go
func BenchmarkProcessClassic(b *testing.B) {
	data := loadHugeFixture() // expensive one-time setup

	b.ResetTimer() // <-- discard the setup time; start measuring from here

	for i := 0; i < b.N; i++ {
		Process(data)
	}
}

func BenchmarkWithPerIterSetup(b *testing.B) {
	for i := 0; i < b.N; i++ {
		b.StopTimer()
		item := freshMutableItem() // per-iteration setup we don't want to time
		b.StartTimer()

		Transform(item) // only THIS is measured
	}
}
```

### 9.5 Comparing benchmarks with benchstat **[I/A]**

A single benchmark run is noisy — background processes, CPU frequency scaling, and cache effects make one number unreliable. The correct workflow is to run each benchmark *multiple times* (`-count=10`), save the output, and use **`benchstat`** (`golang.org/x/perf/cmd/benchstat`) to compute the mean, variance, and — when comparing two files — whether the difference is *statistically significant*. This is how you honestly answer "did my optimization actually help?"

```console
# Baseline: run 10x, save results.
$ go test -bench=BenchmarkParse -benchmem -count=10 > old.txt

# ...make your optimization...
$ go test -bench=BenchmarkParse -benchmem -count=10 > new.txt

# Compare. benchstat prints a delta and a p-value; "~" means no significant
# change, so you don't fool yourself with noise.
$ benchstat old.txt new.txt
name     old time/op    new time/op    delta
Parse-16   4.60µs ± 2%    3.11µs ± 1%   -32.4%  (p=0.000 n=10+10)
```

### 9.6 Profiling — finding WHERE the time goes **[A]**

A benchmark tells you *how slow* code is; a **profile** tells you *where* the time (or memory) is spent, so you optimize the right line instead of guessing. `go test` can emit CPU and memory profiles directly from a benchmark, which you then explore with **`go tool pprof`**. This is the essential loop for serious optimization: benchmark to get a number, profile to find the hot spot, fix it, benchmark again.

```console
# Emit a CPU profile (and a mem profile) from a benchmark run.
$ go test -bench=BenchmarkParse -cpuprofile=cpu.out -memprofile=mem.out

# Explore interactively: `top` shows the hottest functions; `list Parse`
# shows line-by-line time inside a function; `web` draws a graph (needs graphviz).
$ go tool pprof cpu.out
(pprof) top10
(pprof) list Parse
```

`pprof` is a deep tool (flame graphs, allocation profiling, block/mutex profiles), but even `top10` and `list` on a CPU profile will usually point you straight at the one function eating your budget. For production profiling of a running service, the same machinery is exposed via `net/http/pprof` — out of scope here, but it is the same `pprof` view.

---

## 10. Fuzzing

### 10.1 What fuzzing is and why it finds bugs you can't imagine **[A]**

Every test you have written so far checks inputs *you* thought of. But the bugs that reach production are usually the inputs you *didn't* think of — the empty string, the emoji in a name field, the 10MB payload, the malformed UTF-8 that crashes your parser. **Fuzzing** attacks this blind spot: it *generates* huge numbers of random and mutated inputs, feeds them to your code, and watches for crashes, panics, or violated invariants. It is the closest thing to an automated adversary trying to break your function. Fuzzing is spectacularly effective on code that parses or transforms untrusted input — decoders, parsers, validators, anything handling bytes from the network.

> **⚡ Native fuzzing has been built into `go test` since Go 1.18.** You do not need a third-party fuzzer (`go-fuzz` is now legacy). Fuzz targets are `FuzzXxx(f *testing.F)` functions and run with `go test -fuzz`.

### 10.2 Anatomy of a fuzz target **[A]**

A fuzz test has three parts: a **seed corpus** (starting examples you add with `f.Add`), the **fuzz function** (`f.Fuzz` with a callback whose arguments the fuzzer fills with generated data), and inside the callback a **property to check**. The most powerful and easiest property is a *round-trip*: if you encode something and decode it back, you must get the original. Any input that violates that is a bug the fuzzer will hunt down and hand you.

```go
// Suppose we have symmetric encode/decode functions we want to harden.
func FuzzEncodeDecodeRoundTrip(f *testing.F) {
	// Seed corpus: representative starting inputs. The fuzzer MUTATES these
	// to explore nearby inputs, so good seeds accelerate discovery.
	f.Add("hello")
	f.Add("")
	f.Add("with spaces and \n newline")
	f.Add("unicode: 世界 🌍")

	// The fuzz function. The callback's parameters (after *testing.T) are
	// what the fuzzer generates — here, a string `in`.
	f.Fuzz(func(t *testing.T, in string) {
		encoded := Encode(in)
		decoded, err := Decode(encoded)
		if err != nil {
			// Encode output should always be decodable. If not, that's a bug.
			t.Fatalf("Decode(Encode(%q)) errored: %v", in, err)
		}
		// The invariant: round-trip must reproduce the original exactly.
		if decoded != in {
			t.Errorf("round-trip mismatch: got %q; want %q", decoded, in)
		}
	})
}
```

### 10.3 Running the fuzzer and what happens when it finds a bug **[A]**

Without `-fuzz`, a `FuzzXxx` function runs only against its seed corpus — so it acts as a normal regression test in CI (fast, deterministic). To actually *fuzz*, pass `-fuzz` with a pattern; it then runs **continuously**, generating inputs until you stop it or it finds a failure. When it finds an input that crashes or fails your check, it does something wonderful: it **minimizes** the input to the smallest failing case and **saves it to `testdata/fuzz/`**, which becomes a permanent seed. The next plain `go test` replays that saved case, so once a fuzzer finds a bug, it stays found.

```console
# Run as normal regression tests (seed corpus only), part of every CI run:
$ go test ./...

# Actively fuzz one target for 30 seconds (omit -fuzztime to run until Ctrl-C):
$ go test -fuzz=FuzzEncodeDecodeRoundTrip -fuzztime=30s

fuzz: elapsed: 3s, execs: 412300 (137432/sec), new interesting: 42
--- FAIL: FuzzEncodeDecodeRoundTrip (0.02s)
    round-trip mismatch: got "" want "\x00"
    Failing input written to testdata/fuzz/FuzzEncodeDecodeRoundTrip/a1b2c3...
    To re-run: go test -run=FuzzEncodeDecodeRoundTrip/a1b2c3...
```

That last line is the payoff: the failing input is now a committed test-data file, and `go test -run` replays exactly it. You fix the bug, the replay passes, and you have a permanent regression test for a case no human would have typed.

### 10.4 Fuzzing best practices **[A]**

Fuzzing works best when you give it a **checkable property** and a **good seed corpus**. Beyond round-trips, useful properties include "never panics" (just running the target catches panics for free), "output always satisfies an invariant" (e.g. a sanitizer's output never contains `<script>`), and "matches a slow reference implementation." Keep the fuzz function *fast* (the fuzzer runs it millions of times) and *deterministic* (no time, no network). Fuzz targets accept only a fixed set of argument types (`[]byte`, `string`, `int`, `bool`, etc.), so to fuzz structured input, generate the bytes and parse them inside the callback. Run fuzzing on a schedule (nightly) rather than on every commit, since a real fuzzing session takes minutes to hours.

---

## 11. Example Tests as Executable Documentation

### 11.1 Why examples are tests **[B/I]**

Documentation that drifts out of sync with the code is worse than none — it actively misleads. Go's answer is the **example function**: a function named `ExampleXxx` that shows how to use your API *and*, if it ends with an `// Output:` comment, is **run and verified** by `go test`. If the code's printed output no longer matches the comment, the test fails. This means your usage examples *cannot* rot: they compile (so they can't reference deleted functions) and their output is checked (so they can't lie about behavior). And they surface beautifully in `go doc` and on pkg.go.dev, right beside the thing they document. Examples are the rare artifact that is simultaneously documentation, a test, and a compile check.

### 11.2 Writing an example with verified output **[B/I]**

An example lives in a `_test.go` file (usually the black-box `_test` package so it reads like real client code), prints to standard output, and declares the expected output in a trailing comment. The framework captures stdout and compares it — trimmed of surrounding whitespace — against the comment.

```go
package strutil_test

import (
	"fmt"

	"example.com/strutil"
)

// ExampleReverse documents strutil.Reverse. The name "ExampleReverse" ties it
// to the Reverse function in docs. Because of the // Output: line, go test
// RUNS this and fails if the printed text differs.
func ExampleReverse() {
	fmt.Println(strutil.Reverse("hello"))
	// Output: olleh
}

// Naming convention maps examples to symbols:
//   ExampleReverse         -> documents func Reverse
//   ExampleUser            -> documents type User
//   ExampleUser_Greet      -> documents method User.Greet
//   ExampleReverse_unicode -> a SECOND example of Reverse (suffix after _)
func ExampleReverse_unicode() {
	fmt.Println(strutil.Reverse("世界"))
	// Output: 界世
}
```

### 11.3 Unordered output and examples without output **[I]**

Two variations matter. If your example prints things whose *order* isn't guaranteed (iterating a map), use **`// Unordered output:`** — the framework then compares the lines as a set, ignoring order, so the example doesn't flake. And if you write an example with **no `// Output:` comment at all**, it is *compiled but not run* — useful when the example has no deterministic output (it starts a server, say) but you still want it to appear in docs and be guaranteed to compile.

```go
func ExampleCounts() {
	for word, n := range Counts("a b a c b a") {
		fmt.Printf("%s: %d\n", word, n)
	}
	// Order of map iteration is random, so assert as an unordered set:
	// Unordered output:
	// a: 3
	// b: 2
	// c: 1
}

// No Output comment: compiled (so it can't reference dead code) but not run.
// Good for examples that start servers or have nondeterministic output.
func ExampleServer() {
	srv := NewServer()
	srv.Start()
	defer srv.Stop()
	fmt.Println("server running") // not verified without an Output comment
}
```

---

## 12. Integration Tests with Testcontainers and Postgres

### 12.1 Why integration tests, and what they cost **[A]**

A unit test with a mocked database proves your Go logic is right, but it proves *nothing* about whether your SQL is valid, your migrations apply, your `NULL` handling works, or your transaction semantics are correct — because no real database ever saw the query. **Integration tests** fill that gap by running your code against the *real* dependency: an actual PostgreSQL server. This is where you catch the bugs that mocks structurally cannot: a typo in a column name, a constraint violation, a type mismatch between Go and Postgres, a query that works on SQLite but not Postgres. Following the trophy philosophy (§1.4), for a data-layer package the integration test *is* the test that matters.

The cost is speed and setup: a real database is slower (milliseconds not microseconds) and must exist somewhere. The old approaches — a shared CI database, or asking every developer to install Postgres locally — are fragile and non-isolated (tests pollute each other's data; versions drift). The modern answer is **Testcontainers**.

### 12.2 What Testcontainers-go does **[A]**

**Testcontainers-go** (`github.com/testcontainers/testcontainers-go`) programmatically starts a **real Docker container** — a genuine PostgreSQL 17 — from your test code, waits until it is ready, hands you its connection string, and tears it down when the test finishes. Every test run gets a *pristine, isolated, correctly-versioned* database with zero manual setup: no "install Postgres," no shared CI instance, no leftover state. It needs a Docker daemon available (locally: Docker Desktop; in CI: GitHub Actions provides one — §18). This is the single biggest quality-of-life improvement in Go integration testing in years, and it is the approach the [pgx](GO_PGX_GUIDE.md) and [Ent](GO_ENT_ORM_GUIDE.md) guides assume.

### 12.3 Gating integration tests with a build tag **[A]**

Integration tests are slow and need Docker, so you do *not* want them running on every `go test ./...` during rapid local iteration. The idiomatic gate is a **build constraint** (build tag): put `//go:build integration` at the top of integration test files, and they compile *only* when you pass `-tags=integration`. Your fast unit suite runs by default; the heavy integration suite runs on demand and in CI.

```go
//go:build integration

// ^ This line (followed by a BLANK line, then package) means the file is
// compiled ONLY when you run `go test -tags=integration`. A normal `go test`
// skips it entirely, keeping the default suite fast.

package repo_test
```

```console
$ go test ./...                      # fast: unit tests only, no Docker needed
$ go test -tags=integration ./...    # includes the container-backed tests
```

### 12.4 Spinning up Postgres and the shared-container pattern **[A]**

Starting a container takes a couple of seconds, so you usually start **one** per package in `TestMain` (§5.5) and share it across all the package's integration tests, isolating individual tests with the transaction-rollback trick in §12.5 rather than a fresh container each. Here is the full `TestMain` using the `postgres` module, which knows how to wait for Postgres to be genuinely ready to accept connections.

```go
//go:build integration

package repo_test

import (
	"context"
	"database/sql"
	"fmt"
	"os"
	"testing"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/modules/postgres"
	"github.com/testcontainers/testcontainers-go/wait"
)

var testPool *pgxpool.Pool // shared connection pool for the whole package

func TestMain(m *testing.M) {
	ctx := context.Background()

	// Start a real PostgreSQL 17 container. The postgres module sets up the
	// user/password/db and gives us a ready-to-use container handle.
	pgC, err := postgres.Run(ctx,
		"postgres:17-alpine",
		postgres.WithDatabase("testdb"),
		postgres.WithUsername("test"),
		postgres.WithPassword("test"),
		// WaitStrategy: don't proceed until Postgres logs that it's accepting
		// connections TWICE (it restarts once during init) — avoids a flaky
		// "connection refused" race.
		testcontainers.WithWaitStrategy(
			wait.ForLog("database system is ready to accept connections").
				WithOccurrence(2).
				WithStartupTimeout(60*time.Second),
		),
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "start container: %v\n", err)
		os.Exit(1)
	}

	// Ask the container for its dynamically-assigned connection string.
	dsn, err := pgC.ConnectionString(ctx, "sslmode=disable")
	if err != nil {
		fmt.Fprintf(os.Stderr, "conn string: %v\n", err)
		os.Exit(1)
	}

	testPool, err = pgxpool.New(ctx, dsn)
	if err != nil {
		fmt.Fprintf(os.Stderr, "pool: %v\n", err)
		os.Exit(1)
	}

	// Apply the schema/migrations once against the fresh database.
	if err := applyMigrations(ctx, testPool); err != nil {
		fmt.Fprintf(os.Stderr, "migrate: %v\n", err)
		os.Exit(1)
	}

	code := m.Run() // run all integration tests in the package

	// Teardown BEFORE os.Exit (defer won't run — §5.5).
	testPool.Close()
	_ = testcontainers.TerminateContainer(pgC)
	os.Exit(code)
}
```

### 12.5 The transaction-per-test rollback pattern **[A]**

The shared container raises a problem: if test A inserts a user and test B counts users, B sees A's data and the tests are no longer isolated. You could `TRUNCATE` every table between tests, but there is a faster, cleaner idiom beloved in the Go community: **wrap each test in a database transaction and roll it back at the end.** Every test runs inside its own transaction, sees a consistent view, does its inserts/updates, asserts — and then the transaction is *rolled back*, so the database is byte-for-byte unchanged for the next test. It is fast (no truncation), perfectly isolating, and even parallel-safe if each test gets its own transaction/connection.

```go
//go:build integration

// withTx gives a test its own transaction and guarantees rollback at test end,
// so nothing it writes ever persists. The repository under test must accept a
// pgx.Tx (or a common interface implemented by both *pgxpool.Pool and pgx.Tx)
// so it can run inside this transaction — another "design for a seam" case.
func withTx(t *testing.T) pgx.Tx {
	t.Helper()
	ctx := context.Background()

	tx, err := testPool.Begin(ctx)
	if err != nil {
		t.Fatalf("begin tx: %v", err)
	}
	// Rollback at test end. If the test already committed (it shouldn't),
	// Rollback returns ErrTxClosed, which we ignore.
	t.Cleanup(func() {
		_ = tx.Rollback(ctx)
	})
	return tx
}

func TestUserRepo_CreateAndGet(t *testing.T) {
	tx := withTx(t)                 // isolated transaction, auto-rolled-back
	repo := NewUserRepo(tx)         // repo runs its queries inside the tx
	ctx := context.Background()

	id, err := repo.Create(ctx, "Ada")
	if err != nil {
		t.Fatalf("Create: %v", err)
	}

	got, err := repo.Get(ctx, id)
	if err != nil {
		t.Fatalf("Get: %v", err)
	}
	if got.Name != "Ada" {
		t.Errorf("name = %q; want Ada", got.Name)
	}
	// When the test ends, the transaction rolls back: this "Ada" never existed
	// as far as the next test is concerned.
}
```

### 12.6 Testing an Ent or pgx repository against the real DB **[A]**

The same pattern tests either data layer from the sibling guides. For a **pgx** repository you pass the `pgx.Tx` straight in as above. For an **Ent** client, Ent can be constructed over a `database/sql` connection bound to a single transaction, or you use Ent's own `client.Tx(ctx)` and roll it back. The principle is identical: run the real queries against real Postgres inside a transaction, assert on the results, roll back. This is the layer where you finally learn whether your `ON CONFLICT`, your `RETURNING`, your JSONB column, and your foreign-key constraints actually behave — none of which a mock could ever tell you.

```go
//go:build integration

// Ent variant sketch: open a client on the pool, start an Ent transaction,
// exercise the repository, and roll back for isolation.
func TestEntUserRepo(t *testing.T) {
	ctx := context.Background()
	client := entClientFromPool(t, testPool) // helper wiring Ent onto the pool

	tx, err := client.Tx(ctx)
	if err != nil {
		t.Fatalf("ent begin tx: %v", err)
	}
	t.Cleanup(func() { _ = tx.Rollback() }) // discard all writes at test end

	u, err := tx.User.Create().SetName("Grace").Save(ctx)
	if err != nil {
		t.Fatalf("create: %v", err)
	}

	got, err := tx.User.Get(ctx, u.ID)
	if err != nil {
		t.Fatalf("get: %v", err)
	}
	if got.Name != "Grace" {
		t.Errorf("name = %q; want Grace", got.Name)
	}
}
```

> **⚡ Reuse containers across runs with Ryuk and `testcontainers.WithReuse`.** Testcontainers starts a small "Ryuk" reaper container that guarantees your test containers are cleaned up even if the test process is killed. For faster local iteration, `WithReuse(true)` (plus a fixed container name) keeps one container alive between runs. In CI, prefer fresh containers per run for cleanliness. Also consider a `testcontainers.properties` file or `TESTCONTAINERS_RYUK_DISABLED` only if you fully understand the cleanup implications.

---

## 13. End-to-End Tests

### 13.1 What e2e means for a Go service **[A]**

An **end-to-end test** exercises your application the way a real client does: through its actual public entry point, with the real router, real middleware, real handlers, and real database, asserting on the real HTTP responses. Where a §7 handler test calls one handler with a fake store, and a §12 integration test calls one repository against a real DB, an e2e test wires the *whole thing together* — router → middleware → handler → repository → Postgres → JSON response — and drives it over HTTP. It gives the highest confidence that the system actually works, at the cost of being the slowest and most brittle, which is why the pyramid keeps e2e tests few.

### 13.2 Standing up the whole app with httptest.Server **[A]**

The clean way to e2e-test a Go HTTP service in-process is to build your *real* application (router plus a real DB from a Testcontainer) and hand its router to **`httptest.NewServer`**, which serves it on a real loopback port. You then use an ordinary `http.Client` to make real requests against that URL. This is genuinely end-to-end — real HTTP over TCP, the full middleware stack, real database — yet still fully self-contained and automated. Combine §12's container with §7's server and you have a complete e2e harness.

```go
//go:build integration

// e2eApp builds the REAL application against the shared test database and
// serves it on a random port. Returns the base URL and relies on t.Cleanup
// for shutdown.
func e2eApp(t *testing.T) string {
	t.Helper()

	// Build the production router, injecting the real DB-backed store.
	store := NewPostgresUserStore(testPool)
	router := BuildRouter(store) // the SAME BuildRouter main() uses

	srv := httptest.NewServer(router) // real server on 127.0.0.1:<random>
	t.Cleanup(srv.Close)
	return srv.URL
}
```

### 13.3 An e2e flow — register, log in, use a token **[A]**

A representative e2e test walks a *user journey* across multiple endpoints, carrying state (like an auth token) between steps, exactly as a real client would. This is where you validate that auth actually protects routes, that the token issued by `/login` is accepted by `/me`, and that the JSON contracts line up end to end.

```go
//go:build integration

func TestE2E_RegisterLoginAndFetchProfile(t *testing.T) {
	base := e2eApp(t)
	client := &http.Client{Timeout: 5 * time.Second}
	ctx := context.Background()

	// --- STEP 1: register a new user ---
	regBody := `{"email":"ada@example.com","password":"s3cr3t-pw"}`
	regReq, _ := http.NewRequestWithContext(ctx, http.MethodPost,
		base+"/api/register", strings.NewReader(regBody))
	regReq.Header.Set("Content-Type", "application/json")

	regResp, err := client.Do(regReq)
	if err != nil {
		t.Fatalf("register request: %v", err)
	}
	defer regResp.Body.Close()
	if regResp.StatusCode != http.StatusCreated {
		t.Fatalf("register status = %d; want 201", regResp.StatusCode)
	}

	// --- STEP 2: log in and capture the JWT from the response ---
	loginBody := `{"email":"ada@example.com","password":"s3cr3t-pw"}`
	loginReq, _ := http.NewRequestWithContext(ctx, http.MethodPost,
		base+"/api/login", strings.NewReader(loginBody))
	loginReq.Header.Set("Content-Type", "application/json")

	loginResp, err := client.Do(loginReq)
	if err != nil {
		t.Fatalf("login request: %v", err)
	}
	defer loginResp.Body.Close()
	if loginResp.StatusCode != http.StatusOK {
		t.Fatalf("login status = %d; want 200", loginResp.StatusCode)
	}

	var login struct {
		Token string `json:"token"`
	}
	if err := json.NewDecoder(loginResp.Body).Decode(&login); err != nil {
		t.Fatalf("decode login: %v", err)
	}
	if login.Token == "" {
		t.Fatal("expected a non-empty token")
	}

	// --- STEP 3: use the token to hit a PROTECTED endpoint ---
	meReq, _ := http.NewRequestWithContext(ctx, http.MethodGet, base+"/api/me", nil)
	meReq.Header.Set("Authorization", "Bearer "+login.Token)

	meResp, err := client.Do(meReq)
	if err != nil {
		t.Fatalf("me request: %v", err)
	}
	defer meResp.Body.Close()
	if meResp.StatusCode != http.StatusOK {
		t.Fatalf("me status = %d; want 200", meResp.StatusCode)
	}

	var profile struct {
		Email string `json:"email"`
	}
	if err := json.NewDecoder(meResp.Body).Decode(&profile); err != nil {
		t.Fatalf("decode profile: %v", err)
	}
	if profile.Email != "ada@example.com" {
		t.Errorf("profile email = %q; want ada@example.com", profile.Email)
	}

	// --- STEP 4 (negative): the protected route must REJECT no token ---
	noAuthReq, _ := http.NewRequestWithContext(ctx, http.MethodGet, base+"/api/me", nil)
	noAuthResp, err := client.Do(noAuthReq)
	if err != nil {
		t.Fatalf("noauth request: %v", err)
	}
	defer noAuthResp.Body.Close()
	if noAuthResp.StatusCode != http.StatusUnauthorized {
		t.Errorf("unauthenticated /me status = %d; want 401", noAuthResp.StatusCode)
	}
}
```

### 13.4 Asserting on JSON APIs cleanly **[A]**

E2e tests drown in JSON marshaling boilerplate. Two habits keep them readable. First, write small helpers — `postJSON(t, client, url, body)` and `decodeJSON(t, resp, &target)` — that fold the repetitive request-building and error-checking into one `t.Helper()`-marked call. Second, for asserting response *bodies*, prefer `testify`'s `assert.JSONEq` (compares JSON semantically, ignoring key order and whitespace) or decode into a struct and compare fields, rather than brittle string matching. A negative test (bad token → 401, missing field → 400) is as important as the happy path; auth and validation bugs are exactly what e2e is there to catch.

```go
// A tiny helper collapses the request/response ceremony so the TEST reads as
// a sequence of business steps, not HTTP plumbing.
func postJSON(t *testing.T, client *http.Client, url, body string) *http.Response {
	t.Helper()
	req, err := http.NewRequest(http.MethodPost, url, strings.NewReader(body))
	if err != nil {
		t.Fatalf("build request: %v", err)
	}
	req.Header.Set("Content-Type", "application/json")
	resp, err := client.Do(req)
	if err != nil {
		t.Fatalf("do request: %v", err)
	}
	return resp
}
```

---

## 14. Property-Based Testing

### 14.1 The idea — assert properties, not examples **[A]**

Example-based tests (everything up to now) assert that *specific inputs* produce *specific outputs*: `Add(2,3) == 5`. **Property-based testing** flips this: instead of examples, you state a *property* that must hold for **all** inputs — "sorting a list twice gives the same result as sorting it once," "encoding then decoding returns the original," "the reverse of the reverse is the original" — and the framework generates *hundreds of random inputs* trying to falsify it. When it finds a counterexample, it **shrinks** it to the minimal failing case. It is fuzzing's disciplined cousin: fuzzing hunts crashes on unstructured bytes; property testing checks *logical invariants* over *structured, typed* generated values.

Property tests shine for pure, algorithmic code: parsers, serializers, math, data structures, anything with a clean invariant. They routinely find edge cases (empty inputs, duplicates, integer overflow, ordering) that you would never enumerate by hand.

### 14.2 rapid — property testing in Go **[A]**

The modern Go library is **`pgregory.net/rapid`**. It integrates with `go test` (uses `*testing.T`), generates values with a rich set of generators (`rapid.Int()`, `rapid.String()`, `rapid.SliceOf(...)`, custom struct generators), and **automatically shrinks** counterexamples to the smallest input that still fails — so instead of "failed on `[847, -12, 847, 0, 33]`" you get "failed on `[0, 0]`", which is far easier to debug. An older alternative, `gopter`, exists but rapid is the ergonomic current choice.

```go
package sortutil_test

import (
	"sort"
	"testing"

	"pgregory.net/rapid"
)

// Property: sorting is IDEMPOTENT — sorting an already-sorted slice changes
// nothing. We assert this over hundreds of generated slices, not a few examples.
func TestSort_Idempotent(t *testing.T) {
	rapid.Check(t, func(t *rapid.T) {
		// Generate a random slice of ints. rapid picks the length AND values,
		// and will shrink toward small/simple values on failure.
		xs := rapid.SliceOf(rapid.Int()).Draw(t, "xs")

		once := append([]int(nil), xs...)
		sort.Ints(once)

		twice := append([]int(nil), once...)
		sort.Ints(twice)

		// The property: sorting again must not change an already-sorted slice.
		if !slices.Equal(once, twice) {
			t.Fatalf("sort not idempotent: %v then %v", once, twice)
		}
	})
}

// Property: Reverse is its own inverse — reversing twice yields the original.
func TestReverse_Involutive(t *testing.T) {
	rapid.Check(t, func(t *rapid.T) {
		s := rapid.String().Draw(t, "s")
		if got := Reverse(Reverse(s)); got != s {
			t.Fatalf("Reverse(Reverse(%q)) = %q; want original", s, got)
		}
	})
}
```

### 14.3 Finding good properties **[A]**

The hard part of property testing is *identifying* the property, because "what's true for all inputs?" takes practice. A handful of reusable patterns cover most cases: **round-trip** (`decode(encode(x)) == x`), **idempotence** (`f(f(x)) == f(x)`), **invariance** (a property preserved by the operation, e.g. "sorting preserves length and multiset of elements"), **commutativity/associativity** (`f(a,b) == f(b,a)`), and the **model/oracle** pattern (compare a fast implementation against an obviously-correct slow one). When you can name one of these for your function, a property test will often find bugs your example tests missed — for essentially free, since rapid generates the inputs.

---

## 15. Testing Time and Concurrency

### 15.1 Why `time.Now` and `time.Sleep` ruin tests **[A]**

Two things make tests flaky more than anything else: **real time** and **real concurrency**. Code that calls `time.Now()` directly produces a different value every run, so any test asserting on it is nondeterministic. Code that coordinates goroutines with `time.Sleep(100 * time.Millisecond)` — "wait long enough for the other goroutine to finish" — is a bet against the scheduler: it is *slow* (you pay the sleep every run) and *flaky* (on a loaded CI machine 100ms isn't enough and it fails intermittently). Both violate the fast/deterministic rule. The fixes are structural: **inject** the clock, and **synchronize** instead of sleeping.

### 15.2 Injecting a clock **[A]**

The root cause is that `time.Now()` is an *ambient dependency* — your code reaches out to the global clock, giving you no seam to control it (the same lesson as §6). The fix is to make time a *dependency you pass in*: define a tiny `Clock` interface, use the real clock in production, and a controllable fake clock in tests. Now a test can set "now" to any instant and advance it deliberately, making time-dependent logic fully deterministic.

```go
// The seam: code depends on this instead of calling time.Now() directly.
type Clock interface {
	Now() time.Time
}

// Production implementation: the real wall clock.
type realClock struct{}

func (realClock) Now() time.Time { return time.Now() }

// A token expires after a TTL. It takes a Clock, so tests control "now".
type TokenService struct {
	clock Clock
	ttl   time.Duration
}

func (s *TokenService) IsExpired(issuedAt time.Time) bool {
	return s.clock.Now().Sub(issuedAt) > s.ttl
}

// --- test ---

// fakeClock returns whatever time we tell it to, and lets us advance it.
type fakeClock struct{ t time.Time }

func (c *fakeClock) Now() time.Time      { return c.t }
func (c *fakeClock) Advance(d time.Duration) { c.t = c.t.Add(d) }

func TestToken_Expiry(t *testing.T) {
	base := time.Date(2026, 1, 1, 12, 0, 0, 0, time.UTC)
	clk := &fakeClock{t: base}
	svc := &TokenService{clock: clk, ttl: 30 * time.Minute}

	issued := clk.Now()

	// Not expired yet — advance 10 minutes, still within the 30m TTL.
	clk.Advance(10 * time.Minute)
	if svc.IsExpired(issued) {
		t.Error("token should NOT be expired after 10m")
	}

	// Advance past the TTL — now it must be expired. No real waiting occurs;
	// the test runs in microseconds and is 100% deterministic.
	clk.Advance(25 * time.Minute) // total 35m > 30m TTL
	if !svc.IsExpired(issued) {
		t.Error("token SHOULD be expired after 35m")
	}
}
```

For richer needs (timers, tickers, scheduled work), a well-known library is `github.com/benbjohnson/clock`, which provides a mock clock whose timers you advance manually. The principle is unchanged: never let production code read the global clock without a seam.

### 15.3 Synchronize, don't sleep **[A]**

To test that a goroutine did something, do not `time.Sleep` and hope — **synchronize explicitly** so the test proceeds the *instant* the work is done and no sooner. The tools are channels (the goroutine sends when finished), `sync.WaitGroup` (the test waits for a known number of goroutines), and for "wait until a condition becomes true," a bounded poll loop rather than a fixed sleep. This makes concurrency tests both fast (no wasted waiting) and reliable (no scheduler bet).

```go
func TestWorker_ProcessesJob(t *testing.T) {
	done := make(chan Result, 1) // buffered so the worker never blocks

	w := NewWorker(func(r Result) {
		done <- r // signal completion by sending the result
	})
	w.Submit(Job{ID: 7})

	// Wait for the signal OR a timeout — deterministic and bounded. The test
	// finishes the moment the worker is done; the timeout only fires on a real
	// hang, turning a deadlock into a clear failure instead of an infinite wait.
	select {
	case r := <-done:
		if r.JobID != 7 {
			t.Errorf("processed job %d; want 7", r.JobID)
		}
	case <-time.After(2 * time.Second):
		t.Fatal("worker did not finish within 2s")
	}
}
```

### 15.4 synctest — deterministic concurrency testing **[A]**

> **⚡ `testing/synctest` is stable in Go 1.25 (experimental in 1.24).** The standard library now ships a package that makes concurrent, time-based code *fully deterministic* in tests. Inside `synctest.Test`, all goroutines run in an isolated "bubble" with a **fake clock**: `time.Sleep`, timers, and tickers advance *instantly and virtually* once every goroutine in the bubble is blocked, and `synctest.Wait` blocks until all goroutines are idle. This lets you test timeouts, retries with backoff, and rate limiters that involve real `time` calls — *without* injecting a clock and *without* real waiting or flakiness.

```go
// Requires Go 1.25+. Tests a 1-second timeout WITHOUT waiting one real second.
func TestContextTimeout_Synctest(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		ctx, cancel := context.WithTimeout(context.Background(), time.Second)
		defer cancel()

		start := time.Now()
		<-ctx.Done() // inside the bubble, virtual time jumps to the deadline

		// Elapsed VIRTUAL time is exactly 1s, measured instantly in real time.
		if elapsed := time.Since(start); elapsed != time.Second {
			t.Errorf("elapsed = %v; want exactly 1s of virtual time", elapsed)
		}
		if err := ctx.Err(); err != context.DeadlineExceeded {
			t.Errorf("err = %v; want DeadlineExceeded", err)
		}
	})
}
```

`synctest` is a genuine breakthrough for the historically painful category of "testing code that waits." When you are on Go 1.25+, prefer it over injected clocks for timeout/retry logic; fall back to clock injection (§15.2) on older versions.

---

## 16. Testing the Filesystem OS and exec Code

### 16.1 The problem with code that touches the OS **[I]**

Code that reads files, inspects the environment, or shells out to other programs is hard to test for the usual reason: the real OS is slow, stateful, and machine-dependent. A test that reads `/etc/passwd` or runs `git` behaves differently on your Windows laptop, a Linux CI runner, and a colleague's Mac. Go gives you three clean techniques to make this code testable and deterministic — the in-memory filesystem `fstest.MapFS`, the per-test sandbox `t.TempDir`, and the "fake subprocess" pattern for `os/exec` — and, as always, the enabling move is to **design a seam** so your code doesn't hard-wire the real OS.

### 16.2 fstest.MapFS — an in-memory filesystem **[I]**

Since Go 1.16, the standard `io/fs` package defines an `fs.FS` interface for read-only filesystems, and `testing/fstest.MapFS` is an **in-memory implementation** you build from a map of filename → file. If your code accepts an `fs.FS` instead of calling `os.ReadFile` directly, you can hand it a `MapFS` in tests: no disk, no fixtures on disk, instant and deterministic, with whatever file tree you want conjured in a few lines. This is the cleanest way to test file-*reading* logic.

```go
import (
	"io/fs"
	"testing/fstest"
)

// The seam: accept fs.FS, not a hard-coded os path. In production you pass
// os.DirFS("/some/dir"); in tests, an fstest.MapFS.
func CountLines(fsys fs.FS, name string) (int, error) {
	data, err := fs.ReadFile(fsys, name)
	if err != nil {
		return 0, err
	}
	if len(data) == 0 {
		return 0, nil
	}
	return bytes.Count(data, []byte("\n")) + 1, nil
}

func TestCountLines(t *testing.T) {
	// Build a fake filesystem entirely in memory — no disk involved.
	fsys := fstest.MapFS{
		"poem.txt":  {Data: []byte("roses\nviolets\nsugar")},
		"empty.txt": {Data: []byte("")},
	}

	got, err := CountLines(fsys, "poem.txt")
	if err != nil {
		t.Fatalf("CountLines: %v", err)
	}
	if got != 3 {
		t.Errorf("lines = %d; want 3", got)
	}
}
```

> **`fstest.TestFS`** is a bonus: it *checks your own `fs.FS` implementation* for correctness against the interface's contract. If you write a custom filesystem, run it through `fstest.TestFS` to catch conformance bugs.

### 16.3 t.TempDir for code that must write real files **[I]**

`fstest.MapFS` is read-only, so for code that *writes* files (or that insists on real `os` paths) use **`t.TempDir()`** from §5.4: a real, unique, auto-cleaned directory. This exercises the genuine `os` write path — permissions, `MkdirAll`, file modes — against a real disk, but sandboxed so parallel tests don't collide and nothing leaks.

```go
func TestSaveConfig(t *testing.T) {
	dir := t.TempDir() // real, private, auto-removed

	path := filepath.Join(dir, "conf", "app.json")
	if err := SaveConfig(path, Config{Port: 9000}); err != nil {
		t.Fatalf("SaveConfig: %v", err)
	}

	// Verify it really landed on disk with the right content.
	raw, err := os.ReadFile(path)
	if err != nil {
		t.Fatalf("read back: %v", err)
	}
	if !bytes.Contains(raw, []byte(`"port":9000`)) {
		t.Errorf("config = %s; want it to contain port 9000", raw)
	}
}
```

### 16.4 Faking os/exec subprocesses **[A]**

Code that shells out (`exec.Command("git", "status")`) is the hardest to test, because you can't rely on `git` being installed or behaving identically everywhere. The canonical Go technique — used by the standard library itself — is the **`TestHelperProcess` trick**: your code calls commands through an *injectable* runner, and the test points that runner at **the test binary re-invoking itself** with a special env var, so a normal Go test function *acts as* the fake subprocess and prints whatever output you want. It sounds baroque but it is self-contained and needs no external binary.

```go
// Seam: an execCommand variable the test can swap out. Production uses the
// real exec.Command; the test replaces it with a fake that runs the test
// binary itself.
var execCommand = exec.Command

func GitBranch() (string, error) {
	out, err := execCommand("git", "rev-parse", "--abbrev-ref", "HEAD").Output()
	if err != nil {
		return "", err
	}
	return strings.TrimSpace(string(out)), nil
}

// fakeExecCommand builds an *exec.Cmd that re-runs THIS test binary, invoking
// only TestHelperProcess, and flags it via an env var.
func fakeExecCommand(command string, args ...string) *exec.Cmd {
	cs := append([]string{"-test.run=TestHelperProcess", "--", command}, args...)
	cmd := exec.Command(os.Args[0], cs...) // os.Args[0] is the test binary
	cmd.Env = append(os.Environ(), "GO_WANT_HELPER_PROCESS=1")
	return cmd
}

// TestHelperProcess is NOT a real test — it's the fake subprocess. It returns
// early unless the env var marks it as the helper invocation.
func TestHelperProcess(t *testing.T) {
	if os.Getenv("GO_WANT_HELPER_PROCESS") != "1" {
		return
	}
	// Play the role of `git`: print a fake branch and exit cleanly.
	fmt.Fprintln(os.Stdout, "main")
	os.Exit(0)
}

func TestGitBranch(t *testing.T) {
	execCommand = fakeExecCommand      // swap in the fake
	defer func() { execCommand = exec.Command }() // restore after

	got, err := GitBranch()
	if err != nil {
		t.Fatalf("GitBranch: %v", err)
	}
	if got != "main" {
		t.Errorf("branch = %q; want main", got)
	}
}
```

### 16.5 Environment variables and t.Setenv **[I]**

Code that reads `os.Getenv` needs controlled env vars in tests. **`t.Setenv(key, value)`** (Go 1.17+) sets an environment variable for the duration of the test and *automatically restores* the previous value at the end via cleanup — no manual save/restore. One important constraint: `t.Setenv` **cannot be used in a parallel test** (it panics if the test called `t.Parallel()`), because environment variables are process-global and would race between parallel tests. That restriction is a *feature*: it stops you from creating a subtle shared-state bug.

```go
func TestReadConfigFromEnv(t *testing.T) {
	t.Setenv("APP_PORT", "8080") // auto-restored at test end
	t.Setenv("APP_ENV", "test")

	cfg := LoadFromEnv()
	if cfg.Port != 8080 {
		t.Errorf("port = %d; want 8080", cfg.Port)
	}
	// Do NOT call t.Parallel() in a test that uses t.Setenv — it will panic.
}
```

---

## 17. Mutation Testing

### 17.1 The question mutation testing answers **[A]**

Coverage (§8) tells you which lines *ran*, but not whether your tests would *notice* if those lines were wrong. A test that executes a line without asserting anything meaningful gives false confidence. **Mutation testing** answers the deeper question — *"how good are my tests, really?"* — by deliberately introducing small bugs ("mutants") into your production code (change `>` to `>=`, `+` to `-`, `true` to `false`, delete a statement) and rerunning your test suite against each mutated version. If a mutant makes a test fail, the mutant is **killed** — good, your tests caught the injected bug. If all tests still **pass** with the bug in place, the mutant **survived** — a hole in your suite: you have code whose behavior no test actually pins down.

The output is a **mutation score** (percent of mutants killed), and more usefully, a list of *surviving* mutants pointing at exactly the assertions you're missing. It is a test of your tests. It is expensive (the suite reruns once per mutant, so it can take a long time), so you run it periodically on critical packages, not on every commit.

### 17.2 gremlins — mutation testing for Go **[A]**

The maintained Go tool is **gremlins** (`github.com/go-gremlins/gremlins`). You install it, point it at a package, and it reports killed vs. surviving mutants. An older tool, `go-mutesting`, also exists. The workflow:

```console
$ go install github.com/go-gremlins/gremlins/cmd/gremlins@latest

$ gremlins unleash ./...
...
Mutant: CONDITIONALS_BOUNDARY at pricing.go:42:15   KILLED
Mutant: ARITHMETIC_BASE      at pricing.go:58:9    LIVED    <-- a test gap!
...
Mutation testing completed in 41s
Killed: 34, Lived: 3, Timed out: 0, Not covered: 5
Test efficacy: 91.89%
```

The `LIVED` line is gold: gremlins changed an arithmetic operator on `pricing.go:58` and *every test still passed*, which means no test actually checks that calculation's result. You go add an assertion that would catch it, rerun, and the mutant dies. Treat surviving mutants as a prioritized to-do list for strengthening assertions, especially in high-stakes logic (money, auth, permissions) where a silent bug is costly. Do not chase a perfect score — some mutants are equivalent (semantically identical to the original) and unkillable — but a rising score on your critical packages is a real, honest signal of test quality that coverage alone cannot give.

---

## 18. Running Tests in CI with GitHub Actions

### 18.1 Why CI runs your tests **[I]**

Tests only protect you if they *actually run* on every change — a suite that a developer forgets to run locally catches nothing. **Continuous Integration (CI)** runs your full test suite automatically on every push and pull request, on a clean machine, so a broken test blocks the merge before it reaches `main`. This closes the loop: the tests you wrote become an always-on gate. The sibling [GitHub Actions CI/CD guide](GITHUB_ACTIONS_CICD_GUIDE.md) covers Actions in depth; here we focus on the *test-specific* concerns — running the whole suite, the race detector, coverage, container-backed integration tests, and flake detection.

### 18.2 A complete Go test workflow **[I]**

Here is a GitHub Actions workflow that runs the full quality gate: vet, unit tests with `-race` and coverage, and (in a separate job with Docker available) the Testcontainers-backed integration suite. GitHub-hosted Linux runners include a running Docker daemon, so Testcontainers works out of the box.

```yaml
# file: .github/workflows/test.yml
name: test

on:
  push:
    branches: [main]
  pull_request:

jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: '1.26'
          cache: true            # cache the module and build cache automatically

      - name: Vet
        run: go vet ./...

      # -race catches data races; -shuffle=on catches order-dependence;
      # -coverprofile records coverage. This is the core unit-test gate.
      - name: Test
        run: go test -race -shuffle=on -coverprofile=coverage.out ./...

      - name: Coverage summary
        run: go tool cover -func=coverage.out | tail -1

  integration:
    runs-on: ubuntu-latest       # Linux runners have a Docker daemon for Testcontainers
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.26'
          cache: true

      # The build tag gates these; Testcontainers starts Postgres itself, so no
      # `services:` block is needed.
      - name: Integration tests
        run: go test -tags=integration -race ./...
```

### 18.3 Caching for speed **[I]**

CI that takes fifteen minutes gets bypassed; fast CI gets trusted. Two caches matter for Go. The `actions/setup-go@v5` action with `cache: true` automatically caches the **module download cache** (`$GOMODCACHE`) and the **build cache** (`$GOCACHE`), keyed on your `go.sum` — so dependencies aren't re-downloaded and unchanged packages aren't recompiled between runs. That alone often halves CI time. For Testcontainers, image pulls dominate integration-test time; pinning small images (`postgres:17-alpine`) and letting the runner's Docker layer cache work keeps it reasonable.

### 18.4 Detecting flaky tests with shuffle and count **[I]**

A **flaky test** passes sometimes and fails sometimes without any code change (§19.1). CI is where you *hunt* them, because a flake that slips into `main` erodes trust in the whole suite. Two flags are your traps:

- **`-shuffle=on`** randomizes the *order* tests run in. A test that only passes because an earlier test left state behind (hidden order-dependence) will start failing under shuffle, exposing the shared-state bug. The failure prints the *seed* so you can reproduce the exact order with `-shuffle=<seed>`.
- **`-count=N`** runs each test N times in one invocation. `go test -count=10 -run TestSuspect ./...` reruns a suspected flake ten times; if it fails even once, it is flaky. This is also how you *confirm a fix*.

```console
# Reproduce a shuffle failure using the seed CI reported:
$ go test -shuffle=1730000000000000000 ./...

# Stress a suspected-flaky test to smoke out nondeterminism:
$ go test -run '^TestSuspect$' -count=50 -race ./...
```

A good CI policy: run `-race -shuffle=on` on every push (cheap insurance), and a nightly job that runs the suite with `-count` several times to surface flakes before they bite. When a test is found flaky, *fix or quarantine it immediately* — a tolerated flake trains the team to ignore red, which is how real failures get missed.

### 18.5 The `go test ./...` mental model in CI **[I]**

`go test ./...` compiles and tests **every package** in the module, each in its **own process**, and by default runs *packages* in parallel (governed by `-p`, defaulting to `GOMAXPROCS`) while tests *within* a package run sequentially unless they call `t.Parallel()`. This has a practical consequence for CI: because each package is a separate process, `TestMain` and package-level globals are *per package*, so a shared Testcontainer defined in one package's `TestMain` is not shared with another package. If several packages each spin up a container, integration runs get slow — a reason to concentrate integration tests in a few packages, or share one container via a small helper package with `sync.Once`. Understanding this model explains both the parallelism you get for free and the isolation boundaries you must respect.

---

## 19. Gotchas and Best Practices

This section is the distilled hard-won wisdom — the mistakes that produce flaky, slow, or useless tests, and the habits that produce fast, trustworthy ones. Skim it now; reread it after your first painful flake.

### 19.1 Flaky tests — causes and cures **[A]**

A flaky test is one that passes and fails nondeterministically on unchanged code. Flakes are corrosive: they train the team to hit "re-run" and to ignore red, so that when a *real* bug turns a test red, nobody believes it. Treat every flake as a bug to fix now, not later. The usual causes and their fixes:

- **Time dependence** — `time.Now`, `time.Sleep`, timeouts too tight for a loaded runner. *Fix:* inject a clock (§15.2) or use `synctest` (§15.4); synchronize instead of sleeping (§15.3).
- **Ordering / shared state** — a test that depends on another running first, or on a shared global/table. *Fix:* isolate state (fresh fixtures, transaction rollback §12.5); run `-shuffle=on` to expose it.
- **Concurrency / data races** — nondeterministic goroutine interleaving. *Fix:* run `-race` (§8.4); synchronize with channels/WaitGroups.
- **Randomness without a fixed seed** — `rand` or map-iteration order leaking into assertions. *Fix:* seed deterministically in tests; use `// Unordered output:` (§11.3) or compare as sets.
- **External dependencies** — real network, real time-of-day, DNS. *Fix:* fake them (`httptest` §7.6, containers §12); never call the live internet in a test.

### 19.2 Do not test implementation — test behavior **[A]**

The most valuable tests assert *what the code does* (observable behavior through its public API), not *how it does it* (internal calls, private helpers, field layout). Behavior tests survive refactoring — you can rewrite the internals and they still pass, which is exactly what you want, because a refactor that changed nothing observable should not break a single test. Implementation tests (heavy mocking that asserts call sequences, white-box tests of every private function) break constantly on harmless changes, so people learn to "fix the test" mechanically, which destroys the test's value as a truth-teller. Prefer black-box tests (§2.5), prefer fakes over interaction-mocks (§6.5), and ask of every test: *"would this still pass if I refactored the internals but kept the behavior? Would it fail if the behavior actually broke?"* If the answers aren't yes-and-yes, the test is testing the wrong thing.

### 19.3 The over-mocking trap **[A]**

Reiterated because it is so common: mocking every dependency until a test only verifies that your code calls the mocks you told it to call produces a test that *always passes* (it's checking the code against itself) and gives *false confidence* (the real integration is never exercised). Mock the *edges you don't own* (third-party APIs, email, the clock); use *real* implementations or fakes for the core you're actually testing; and for anything where the integration IS the point (SQL correctness, HTTP routing), use a real dependency (§12). A codebase drowning in mocks is usually a codebase whose tests catch nothing.

### 19.4 The pre-1.22 loop-variable capture bug **[A]**

Covered in §3.3 but it belongs on any gotcha list because it silently broke countless table-driven parallel tests. Before Go 1.22, the loop variable was shared across iterations; a `t.Parallel()` subtest that captured it ran *later*, after the loop finished, and saw the *last* row's values — so every parallel subtest tested the same case and the others were silently skipped. **Go 1.22+ gives each iteration its own variable, fixing this**, so modern code needs no `tc := tc` shadow. If your module's `go.mod` declares `go 1.21` or below, keep the shadow; on 1.22+ omit it. Beware old tutorials that present the shadow as mandatory.

### 19.5 A best-practices checklist **[A]**

| Practice | Why |
|---|---|
| Name tests and subtests descriptively (`TestX/empty_input`) | The name is the first line of the failure report |
| Use `got`/`want` in messages, `want` first in testify | Consistent, instantly readable failures |
| One logical assertion focus per test | A test should fail for exactly one reason |
| Table-driven + subtests for multiple cases | Add a case in one line; isolate failures |
| `require` for preconditions, `assert` for checks | Stop when continuing would panic; else see all failures |
| `t.Cleanup` over `defer` for teardown | Runs at test end even from helpers; LIFO; survives Fatal |
| `t.TempDir`/`t.Setenv` over manual setup | Auto-cleaned, parallel-safe, can't leak |
| Run `-race` and `-shuffle=on` in CI | Catch races and order-dependence early |
| Prefer fakes/real deps over interaction-mocks | Test behavior, not implementation |
| Inject clocks; never `time.Sleep` to synchronize | Deterministic, fast |
| Real DB via Testcontainers for data-layer tests | Mocks can't validate SQL |
| Gate slow tests with `//go:build integration` | Keep the default suite fast |
| Keep tests fast, isolated, deterministic, readable | The whole point (§1.5) |
| Never call the live internet in a test | Flake and slowness guaranteed |
| Fix or delete flaky tests immediately | A tolerated flake poisons trust in the suite |

### 19.6 Minor but common mistakes **[I]**

A few small traps worth internalizing: **forgetting `t.Helper()`** in an assertion helper (failures point at the wrong line); **using `t.Fatal` from a spawned goroutine** (it doesn't stop the test — §2.2); **comparing floats with `==` or `assert.Equal`** (use `InDelta` — rounding will bite you); **not closing `resp.Body`** in HTTP tests (leaks connections, can hang the suite); **relying on the test cache** when a test reads changing external state (add `-count=1`); and **asserting on error *strings*** instead of `errors.Is`/`errors.As` (brittle — messages change, wrapped errors break substring checks). None are catastrophic alone, but each is a paper cut that a careful reviewer catches.

---

## 20. Study Path and Build-to-Learn Projects

Reading about testing builds recognition; *writing* tests builds skill. This path takes you from your first `TestXxx` to a fully-tested service, in the order the concepts compound. Do the projects — each one forces you to confront the exact difficulties the corresponding sections prepared you for.

### 20.1 Suggested reading order **[B→A]**

If you are brand new, read straight through §1–§8, writing a test for every concept as you meet it, then stop and do Project 1 before continuing. Sections §9–§11 (benchmarks, fuzzing, examples) can be read any time after §3. Sections §12–§13 (integration, e2e) need Docker and are best tackled after you are comfortable with §6's seams. Sections §14–§17 are advanced techniques to layer on once the fundamentals are automatic. Re-read §19 after your first real flake — it will land differently.

| Stage | Sections | You can now |
|---|---|---|
| **Foundations** | §1–§3 | Write table-driven unit tests with subtests |
| **Fluency** | §4–§6 | Assert cleanly, manage lifecycle, use test doubles |
| **Services** | §7–§8, §11 | Test handlers, measure coverage, run `-race`, write examples |
| **Performance** | §9 | Benchmark and profile hot paths |
| **Real systems** | §12–§13, §16, §18 | Integration + e2e tests against real deps, in CI |
| **Mastery** | §10, §14, §15, §17 | Fuzz, property-test, tame time/concurrency, audit test quality |

### 20.2 Project 1 — a fully unit-tested library (Foundations) **[B]**

Build a small pure-logic package — a `moneyutil` package that parses and formats currency, or a `slug` generator that turns titles into URL slugs — and test it *thoroughly* with nothing but the standard library. Requirements: every function has a table-driven test with named subtests covering happy path, empty input, and at least two edge cases; error paths use `errors.Is`; you hit 100% coverage of the *branching* logic (verify with `go tool cover -html`) and can explain any line you chose *not* to cover; and you add at least one `Example` function with verified `// Output:`. Goal: internalize §2–§4 and §11 until the table-driven shape is muscle memory.

### 20.3 Project 2 — a service layer with test doubles (Fluency) **[I]**

Add a service that depends on a repository *interface* — say an `OrderService` with an `OrderStore`. Write a hand-rolled in-memory fake (§6.3) and test the service against it (state verification); then generate a gomock mock (§6.4) and write the *same* tests as interaction verification, so you feel the difference firsthand. Force and test every error path by making the fake return errors. Use `t.Cleanup` and `t.Helper` for setup. Goal: master seams, dependency injection, and the fake-versus-mock judgment from §6.

### 20.4 Project 3 — a tested HTTP API (Services) **[I]**

Wrap the service in a Gin (or `net/http`) API with register/login/CRUD endpoints and JWT auth middleware. Test every handler with `httptest.NewRecorder` (§7.2), test the auth middleware in a table (§7.4), and test the router end-to-end in-process with `engine.ServeHTTP` and a fake store (§7.5). Add an outbound API client and test it against `httptest.NewServer` (§7.6). Run the whole suite with `-race` and set up the GitHub Actions workflow from §18. Goal: confidently test the request→response path, the layer most Go jobs live in.

### 20.5 Project 4 — integration and e2e against real Postgres (Real systems) **[A]**

Replace the fake store with a real pgx- or Ent-backed Postgres repository (borrow from the [pgx](GO_PGX_GUIDE.md) / [Ent](GO_ENT_ORM_GUIDE.md) guides). Write integration tests gated behind `//go:build integration` that spin up Postgres with Testcontainers in `TestMain` (§12.4), isolate each test with the transaction-rollback pattern (§12.5), and assert real SQL behavior. Then write an e2e test (§13) that stands up the whole app on `httptest.NewServer` against the container and walks the register→login→protected-route journey, including negative auth cases. Wire both into the CI workflow's integration job. Goal: the trophy's middle band — the tests that actually prove the system works.

### 20.6 Project 5 — hardening with fuzzing, properties, and mutation (Mastery) **[A]**

Return to Project 1's parser/formatter and *attack* it. Write a fuzz target (§10) for its round-trip property and run `go test -fuzz` until it finds (or fails to find) a bug — if it finds one, fix it and confirm the saved corpus file replays. Add a `rapid` property test (§14) asserting an invariant. Then run `gremlins` (§17) over your best-tested package and *kill the surviving mutants* by adding the missing assertions — watch your mutation score climb. Finally, find any time- or goroutine-dependent code and make it deterministic with an injected clock or `synctest` (§15). Goal: move from "my code has tests" to "my tests are provably good."

### 20.7 Where to go next **[A]**

Once these are second nature, deepen in the directions your work demands: contract testing (Pact) for microservice boundaries; load/soak testing (k6, vegeta) for capacity; snapshot testing for large outputs beyond golden files; and observability-driven testing (asserting on emitted metrics/traces). But the fundamentals in this guide — fast, isolated, deterministic, readable tests, at the right level of the pyramid, testing behavior over implementation — carry you through all of them. The tools change; the discipline does not.

Happy testing. A green suite you trust is one of the great pleasures of software engineering — build one.
