# Go (Golang) — Beginner to Advanced — Complete Language Reference

> **Who this is for:** Anyone going from "I have never written a line of Go" to "I understand the Go *language* deeply enough to read the standard library and reason about memory, concurrency, and the type system" — entirely offline. This is the **pure-language** reference: every concept is explained in prose first (what it is, *why* Go was designed this way, when and how to use it, best practices, and security notes where relevant), then demonstrated with heavily-commented, runnable code. Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Go 1.25 / 1.26** (current in 2026). Modern-Go features you should know and that this guide uses throughout:
> - **Generics** (type parameters) — in the language since 1.18, mature and idiomatic where they fit (§11).
> - **Range-over-function iterators** (`for v := range myIterator`) landed in **1.23** — you can write custom iterators that plug into `range`; the `iter` package and the `slices`/`maps` iterator helpers build on this (§6, §13).
> - **The loop-variable-per-iteration change (Go 1.22)** fixed the infamous "loop variable captured by a goroutine/closure" bug — each iteration now gets a *fresh* variable. Code built with `go 1.22+` behaves differently from older code (§5, §19).
> - **`min`, `max`, `clear` builtins** (1.21); **`log/slog`** structured logging in the stdlib (1.21); enhanced `net/http` routing (1.22); the `slices` and `maps` packages (1.21).
>
> Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**, so cross-platform notes (path separators, `.exe` suffixes, shells) are called out. Always confirm exact API signatures at pkg.go.dev.
>
> **This guide's unique value:** The library already has dedicated Go guides — **`GO_LANG_AND_PATTERNS_GUIDE.md`** (language + production design patterns), plus `GO_NET_HTTP_REST_API_GUIDE.md`, `GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`, `GO_ENT_ORM_GUIDE.md`, `GO_JWT_ARGON2_GUIDE.md`, `GO_GORILLA_WEBSOCKETS_GUIDE.md`, `GO_GRPC_RPC_GUIDE.md`, and `GO_FILESYSTEM_OS_CLI_GUIDE.md`. **This** guide is the definitive treatment of the **language itself**, beginner → mastery. For architecture/patterns see the patterns guide; for the full filesystem/OS/CLI treatment see the FS guide (§14 here is only a concise summary). We deliberately avoid duplicating those.

---

## Table of Contents

1. [Why Go & Its Philosophy; Install & Toolchain](#1-why-go--its-philosophy-install--toolchain) **[B]**
2. [Program Structure: Packages, Imports, Modules](#2-program-structure-packages-imports-modules) **[B]**
3. [Variables, Constants, `iota` & Zero Values](#3-variables-constants-iota--zero-values) **[B]**
4. [Strings, Runes, Bytes, UTF-8 & Formatting](#4-strings-runes-bytes-utf-8--formatting) **[B/I]**
5. [Control Flow: if / for / switch / defer / goto](#5-control-flow-if--for--switch--defer--goto) **[B]**
6. [Composite Types: Arrays, Slices, Maps, Structs](#6-composite-types-arrays-slices-maps-structs) **[B/I]**
7. [Pointers](#7-pointers) **[B/I]**
8. [Functions: Returns, Variadics, Closures, Higher-Order](#8-functions-returns-variadics-closures-higher-order) **[B/I]**
9. [Methods & Interfaces](#9-methods--interfaces) **[I]**
10. [Errors: Wrapping, Sentinels, Custom Types, panic/recover](#10-errors-wrapping-sentinels-custom-types-panicrecover) **[I]**
11. [Generics](#11-generics) **[I/A]**
12. [Concurrency: Goroutines, Channels, sync, context](#12-concurrency-goroutines-channels-sync-context) **[A]**
13. [The Standard Library Tour](#13-the-standard-library-tour) **[I]**
14. [File System / OS / exec (Concise)](#14-file-system--os--exec-concise) **[I]**
15. [Testing, Benchmarking, Fuzzing](#15-testing-benchmarking-fuzzing) **[I/A]**
16. [Modules, Tooling & Project Layout](#16-modules-tooling--project-layout) **[I/A]**
17. [Performance & Memory: Stack/Heap, GC, Profiling](#17-performance--memory-stackheap-gc-profiling) **[A]**
18. [Security Recommendations](#18-security-recommendations) **[I/A]**
19. [Idioms, Gotchas & Best Practices](#19-idioms-gotchas--best-practices) **[A]**
20. [Study Path & Build-to-Learn Projects](#20-study-path--build-to-learn-projects)

---

## 1. Why Go & Its Philosophy; Install & Toolchain

### 1.1 The design philosophy — why Go exists **[B]**

Go was created at Google (announced 2009, version 1.0 in 2012) by Robert Griesemer, Rob Pike, and Ken Thompson. It was born from a concrete frustration: building large server programs with large teams in languages that forced a painful trade-off. You could have *fast execution* (C++) or *fast development with safety* (Java, Python), but rarely both, and none of them made concurrency comfortable. Almost every feature — and every *absent* feature — is a deliberate answer to that pain. Understanding the "why" is what makes the rest of the language feel inevitable instead of arbitrary, so we lead with it.

- **Simplicity is a feature.** Go has a famously small specification — you can hold the entire language in your head. There is **no class inheritance, no method overloading, no exceptions, no implicit numeric conversions, no ternary operator, no constructors, no generics until 1.18 (and even now they are restrained).** The reasoning: code is *read* far more than it is *written*, and a small language means any engineer can read any other engineer's code without surprises. When the language "won't let you" do something clever, that is usually on purpose — it is trading personal expressiveness for team-wide readability.
- **Fast compilation.** Go compiles huge programs in seconds. The dependency model was designed for this: no circular imports, explicit imports only (unused imports are a *compile error*), and no C-style header files that get re-parsed repeatedly. Fast builds change how you work — the edit → compile → test loop feels like a scripting language.
- **Concurrency as a first-class citizen.** Goroutines (extremely cheap, runtime-scheduled "threads") and channels (typed pipes that move values between them) are *language* features, not a bolted-on library. The mantra: **"Do not communicate by sharing memory; instead, share memory by communicating."** (§12).
- **Static single binaries.** `go build` emits one self-contained executable — no interpreter, no VM, no shared libraries to ship. This is why Go dominates cloud/devops tooling (Docker, Kubernetes, Terraform, Prometheus are all Go): you drop one file on a machine and it runs. Cross-compilation is one environment variable away.
- **Garbage collected, but performance-aware.** You get memory safety without manual `malloc`/`free`, via a low-latency concurrent GC tuned for server workloads (§17).
- **An opinionated standard library and toolchain.** Formatting (`gofmt`), testing, benchmarking, profiling, dependency management, vetting, and documentation all ship in the box. There is exactly one official way to format code, which ends all style arguments permanently.

**When to reach for Go:** network services and APIs, CLIs, devops/infrastructure tooling, data pipelines, anything where you want predictable performance, painless deployment, and good concurrency. **When not to:** heavy numeric/scientific computing (Python's ecosystem wins), GUI-heavy desktop or mobile apps, or domains where a richer type system (sum types, pattern matching) genuinely pays off (Rust, Haskell).

### 1.2 Install **[B]**

- **Windows:** `winget install GoLang.Go`, or download the MSI from go.dev/dl. The installer puts `go` on your `PATH`.
- **macOS:** `brew install go`, or the official pkg.
- **Linux:** extract the official tarball to `/usr/local/go` and add `/usr/local/go/bin` to `PATH`, or use your distro's package manager.

Verify the install and inspect the environment:

```bash
go version          # e.g. go version go1.24.0 windows/amd64
go env GOPATH       # where downloaded modules + installed binaries live (default ~/go)
go env GOROOT       # where the Go installation itself lives
go env GOOS GOARCH  # your current target OS + CPU architecture
```

> **⚡ Version note:** Since Go 1.21 the toolchain can *auto-download* the exact Go version a project requires via the `toolchain` directive in `go.mod`. You install one Go and it fetches others on demand — so "which Go do I have" matters less than it used to.

### 1.3 `go run`, `go build`, `go install` **[B]**

```bash
go run .              # compile + run the package in the current directory (no binary left on disk)
go run main.go        # compile + run a single file
go build              # compile to an executable in the current dir (myapp.exe on Windows)
go build -o bin/app   # choose the output path
go install            # build and place the binary in $GOPATH/bin (which should be on your PATH)
```

`go run` is for development — it gives you the scripting-language feel. `go build` produces the artifact you ship. `go install` is for installing CLI tools (your own or others': `go install github.com/some/tool@latest`). On Windows the binary gets an `.exe` suffix automatically; on Linux/macOS it does not.

### 1.4 The toolchain — your daily tools **[B]**

Go's tooling is part of the language experience. Memorize these:

```bash
go fmt ./...      # format every package, recursively, to the One True Style (gofmt under the hood)
go vet ./...      # static analysis: catches suspicious constructs (bad Printf verbs, lock copies, etc.)
go test ./...     # run all tests in all packages
go doc fmt.Printf # read documentation for a symbol, offline, from your local source
go build ./...    # type-check + compile everything (great "does it still compile" check)
go mod tidy       # reconcile go.mod/go.sum with what the code actually imports
```

- **`gofmt`** removes the entire category of "how should we indent / where do braces go" debate. Indentation is **tabs** (this matters — see every `go` code block in this guide). Configure your editor to run `gofmt` (or `goimports`, which also fixes imports) on save.
- **`go vet`** is not a linter for style; it finds *likely bugs* the compiler allows. Run it in CI.
- **`go doc`** is your offline documentation. `go doc strings` lists a package's exported API; `go doc strings.Builder` drills into a type. This is invaluable when you have no internet.
- For deeper static analysis, the community standard is **`golangci-lint`** (an aggregator that runs `staticcheck`, `errcheck`, `govet`, and dozens more). It is not part of the Go distribution but is near-universal in real projects.

### 1.5 GOPATH vs modules — a one-paragraph history **[B]**

Before 2018, all Go code had to live inside a single `$GOPATH/src` directory tree, and there was no real dependency versioning — you got "whatever was on master." This was the **GOPATH era** and you will still see it referenced in old tutorials. **Modules** (introduced in 1.11, mandatory since 1.16) replaced it: a project can now live *anywhere* on disk, declares its dependencies and their versions explicitly in `go.mod`, and verifies them with checksums in `go.sum`. **You should always use modules.** GOPATH still exists, but now it is just a cache (`$GOPATH/pkg/mod`) and a bin directory (`$GOPATH/bin`). The next section covers modules in practice.

> **Best practice recap (§1):** install one recent Go; let your editor run `gofmt`/`goimports` on save; wire `go vet` and `golangci-lint` into CI; use `go doc` for offline docs; never write GOPATH-style code — always `go mod init` a new project.

---

## 2. Program Structure: Packages, Imports, Modules

### 2.1 Packages — the unit of code organization **[B]**

A **package** is Go's unit of compilation, naming, and encapsulation. Every `.go` file begins with a `package` clause, and every file in the same directory must declare the same package name. A package groups related code; importing a package gives you access to its *exported* identifiers. The reasoning: Go wants a flat, fast-to-compile dependency graph, so the package — not the file or the class — is the boundary the compiler and the visibility rules operate on.

Two package names are special:

- **`package main`** — tells the toolchain "this is an executable program, not a library." A `main` package must contain a `func main()`, which is the entry point. `go build` on a `main` package produces a runnable binary.
- Any other name (`package strings`, `package myutil`) — a **library** package, meant to be imported by others. `go build` on it just type-checks/compiles; it produces no runnable binary.

```go
// File: main.go  — an executable program.
package main // executable; must have func main()

import "fmt" // bring in the standard library's fmt package

// main is THE entry point. No arguments, no return value.
// The runtime calls it once; when it returns, the program exits with status 0.
func main() {
	fmt.Println("Hello, Go") // Println comes from the imported fmt package
}
```

### 2.2 Imports — explicit, and unused ones are errors **[B]**

You import packages by their *path* (for the stdlib this looks like the package name, e.g. `"fmt"`, `"net/http"`; for third-party code it is the full module path). Within your code you refer to a package by its *name* (the last path element by default).

```go
import (
	"fmt"      // referred to as fmt
	"net/http" // referred to as http (last path element)
	"strings"

	"github.com/google/uuid" // third-party; referred to as uuid
)
```

Key rules and *why* they exist:

- **Unused imports are a compile error**, not a warning. The rationale: dead imports slow compilation and mislead readers, so Go forbids them outright. (`goimports` auto-removes them, which is why most people stop noticing.)
- **Import aliases** rename a package locally: `import crand "crypto/rand"` lets you use `crand` to avoid clashing with `math/rand`.
- **The blank import** `import _ "github.com/lib/pq"` imports a package *only for its side effects* (its `init` functions run) without binding a name — common for registering database drivers or image formats.
- **The dot import** `import . "fmt"` dumps a package's exported names into your namespace so you can write `Println` instead of `fmt.Println`. It is almost always a bad idea (it hides where names come from); you mostly see it in some test files.

```go
import (
	_ "github.com/jackc/pgx/v5/stdlib" // side-effect: registers the "pgx" sql driver
	crand "crypto/rand"                // alias: avoid clash with math/rand
)
```

### 2.3 Exported vs unexported — the capitalization rule **[B, important]**

Go has **no `public`/`private`/`protected` keywords.** Visibility is decided by a single, mechanical rule:

- An identifier (function, type, variable, constant, struct field, or method) whose name **starts with an uppercase letter is *exported*** — visible to code in *other* packages that import this one.
- An identifier that **starts with a lowercase letter is *unexported*** — visible only *within its own package*.

The logic: visibility should be obvious at the *use site* without looking anything up. When you read `user.Name` you instantly know `Name` is part of the package's public API; `user.cache` is clearly internal. This also makes the rule trivial for tooling to enforce. The boundary is the **package**, not the file or the type — code in the same package can touch each other's unexported members freely (this is how a package's own test file can test internals).

```go
package account

type Account struct {
	ID      string // exported field — visible to importers
	balance int64  // unexported field — only this package can read/write it
}

// Deposit is exported: part of the package's public API.
func (a *Account) Deposit(cents int64) { a.balance += cents }

// validate is unexported: an internal helper, invisible outside this package.
func (a *Account) validate() bool { return a.balance >= 0 }
```

> **Idiom:** keep struct fields unexported and expose behaviour through exported methods when you must maintain invariants (like "balance never goes negative"). Export fields freely for plain data carriers (DTOs, config structs).

### 2.4 `init` functions and package initialization order **[I]**

Each package may define one or more `func init()` (no args, no returns). They run **automatically**, after all package-level variables are initialized and **before** `main` starts. Within a package, `init`s run in the order the files are presented to the compiler (alphabetical by filename); across packages, a package's dependencies are initialized first. Use `init` sparingly — for registering drivers, validating required config, or building lookup tables — because implicit work that runs before `main` is hard to trace and test.

```go
package config

import "fmt"

var defaults = loadDefaults() // package-level vars initialize first

func init() { // runs once, automatically, before main()
	fmt.Println("config package initialized") // e.g. validate env, register things
}
```

### 2.5 `go.mod`, `go.sum`, and versioning **[B, important]**

A **module** is a tree of packages versioned together, rooted at a `go.mod` file. The module path is normally the repo URL (`github.com/you/myapp`) and *also* serves as the import prefix for the module's own packages — so a package in `./internal/store` is imported as `github.com/you/myapp/internal/store`.

```bash
go mod init github.com/you/myapp     # create go.mod
go get github.com/google/uuid        # add a dependency (records it + checksum)
go get github.com/google/uuid@v1.6.0 # pin an exact version
go get -u ./...                      # upgrade dependencies to latest minor/patch
go mod tidy                          # add missing + drop unused deps (run this often)
go mod download                      # pre-fetch deps (CI / Docker layer caching)
```

A typical `go.mod`:

```go
module github.com/you/myapp

go 1.24 // the LANGUAGE VERSION the module targets (affects semantics, e.g. loop vars)

require (
	github.com/google/uuid v1.6.0
	golang.org/x/sync v0.7.0
)

require golang.org/x/sys v0.20.0 // indirect  (a transitive dependency)
```

- **`go.mod`** lists your *direct* dependencies, the minimum *language version*, and (marked `// indirect`) transitive deps that need pinning. Edit it via `go get`/`go mod tidy`, not by hand.
- **`go.sum`** stores cryptographic checksums of every dependency version your build uses, so builds are **reproducible and tamper-evident** — if a published version is ever altered, your build fails loudly. **Commit both files.**
- **Semantic Import Versioning:** Go encodes major versions ≥ 2 in the *import path* itself: `github.com/foo/bar/v2`. This lets v1 and v2 of a library coexist in one build (to the compiler they are different packages). v0 and v1 share the unsuffixed path.

The `go` directive is not cosmetic: with `go 1.22` or higher, the Go 1.22 per-iteration loop-variable semantics apply (§5/§19); lower values get the old shared-variable behaviour. That is why the directive is a *language version*, not just metadata.

> **Best practice recap (§2):** one `main` package per binary; keep packages small and named after a noun; rely on the capitalization rule for your public API; use `init` sparingly; run `go mod tidy` before every commit; commit `go.mod` *and* `go.sum`.

---

## 3. Variables, Constants, `iota` & Zero Values

### 3.1 Declaring variables: `var` vs `:=` **[B]**

Go gives you two ways to introduce a variable, and the difference is mostly about *where* and *how explicit* you want to be.

- **`var name type = value`** — the full form. You can omit the type (Go infers it from the value) or the value (you get the *zero value*, §3.4). `var` works at *package level* and inside functions.
- **`name := value`** — *short variable declaration.* It declares **and** initializes, inferring the type. It is concise and the workhorse inside functions, but it **only works inside a function body** (not at package scope), and it requires at least one *new* variable on the left.

The reasoning for two forms: package-level state and "I want an explicit type or a zero value" read best with `var`; the common local "make a thing from this expression" reads best with `:=`. Go nudges you toward `:=` for locals (less noise) and `var` where you need clarity or package scope.

```go
package main

import "fmt"

// Package-level: MUST use var (:= is illegal here).
var appName = "demo"      // type inferred as string
var maxRetries int = 3    // explicit type
var debug bool            // no value -> zero value (false)

func main() {
	count := 0              // short decl; type inferred as int
	name := "Ada"           // string
	pi, tau := 3.14159, 6.28318 // multiple at once

	count = count + 1       // = is ASSIGNMENT (var must already exist); := is DECLARATION
	fmt.Println(appName, count, name, pi, tau, debug, maxRetries)

	// := needs at least one NEW var on the left. This is legal because err is new:
	x := 1
	x, err := 2, fmt.Errorf("oops") // x reused (assigned), err declared
	_ = err
	fmt.Println(x)
}
```

> **Gotcha — shadowing:** inside a new block (an `if`, `for`, etc.) `:=` can *accidentally* create a new variable that shadows an outer one of the same name, so your outer variable never gets updated. `go vet` and `shadow`/`golangci-lint` catch many cases. Be deliberate: use `=` when you mean "assign to the existing variable."

### 3.2 Basic types **[B]**

Go's built-in types are small and explicit:

| Category | Types | Notes |
|---|---|---|
| Boolean | `bool` | `true`/`false`; no implicit conversion from ints. |
| Signed ints | `int`, `int8`, `int16`, `int32`, `int64` | `int` is 32- or 64-bit depending on platform (usually 64). Default integer type. |
| Unsigned ints | `uint`, `uint8`, `uint16`, `uint32`, `uint64`, `uintptr` | `byte` is an alias for `uint8`. |
| Floats | `float32`, `float64` | `float64` is the default; use it unless you have a reason. |
| Complex | `complex64`, `complex128` | Rare outside scientific code. |
| Text | `string` | Immutable UTF-8 bytes (§4). `rune` = `int32` (a Unicode code point). |

The logic behind sized types: Go is a systems-capable language, so it exposes exact widths for binary protocols, file formats, and performance. But for everyday code you use `int` (counts, indices) and `float64` (real numbers) and `string`.

### 3.3 Type conversions are always explicit **[B, important]**

Go does **no implicit numeric conversion.** You cannot add an `int` to an `int64` or assign a `float64` to an `int` without writing the conversion. The reasoning: implicit conversions are a classic source of silent bugs (truncation, sign changes, precision loss), so Go forces you to *say* the conversion, making the cost visible.

```go
var i int = 42
var f float64 = float64(i)   // must convert explicitly
var u uint = uint(f)         // float -> uint truncates toward zero (42)
var b byte = byte(300)       // wraps modulo 256 -> 44; be careful with overflow

// var bad float64 = i       // COMPILE ERROR: cannot use int as float64
// var bad2 int = f          // COMPILE ERROR: cannot use float64 as int

// String <-> number is NOT a conversion; use strconv (a conversion would give a rune!):
// string(65) == "A"  (interprets 65 as a code point, almost never what you want)
```

> **Gotcha:** `string(someInt)` does **not** stringify the number — it returns the Unicode character for that code point. To turn `65` into `"65"`, use `strconv.Itoa(65)` or `fmt.Sprint(65)` (§4). `go vet` warns about `string(int)`.

### 3.4 Zero values — Go's "no nulls for value types" philosophy **[B, important]**

**Every variable in Go is *always* initialized.** If you declare a variable without a value, it gets its type's **zero value** — never garbage, never "undefined." The zero values are:

| Type | Zero value |
|---|---|
| numeric (`int`, `float64`, …) | `0` |
| `bool` | `false` |
| `string` | `""` (empty string) |
| pointer, slice, map, channel, function, interface | `nil` |
| struct | a struct with every field set to its own zero value |

**Why this matters (the philosophy):** by guaranteeing a meaningful zero value, Go eliminates an entire class of "uninitialized variable" bugs and lets you design types that are **useful without a constructor.** A `var buf bytes.Buffer` is immediately ready to use; `var mu sync.Mutex` is an unlocked mutex; `var s []int` is a usable (empty) slice you can `append` to. This is the "**make the zero value useful**" design rule. Value types (numbers, bools, structs, arrays) therefore *cannot* be `nil` — there is no "null int." Only the reference-like types (pointer, slice, map, channel, func, interface) have `nil` as their zero value, and even those often behave sensibly when nil (a nil slice has length 0; ranging over it does nothing).

```go
var n int        // 0
var s string     // ""
var ok bool      // false
var p *int       // nil
var sl []int     // nil, but len(sl)==0 and you can append to it
var m map[string]int // nil — reading is fine (returns zero), WRITING panics (§6)

type User struct {
	Name string // ""
	Age  int    // 0
}
var u User // {Name:"" Age:0} — fully formed, no constructor needed
```

### 3.5 Constants & `iota` **[B/I]**

A **constant** is a value fixed at compile time. Constants can be *typed* or *untyped*; untyped constants are a quietly powerful feature. An **untyped constant** has a "default type" but adapts to the context it is used in, with arbitrary precision until assigned. That is why `const Pi = 3.14159` can be used where a `float32` *or* `float64` is wanted, and why you can write `const Big = 1 << 62` without picking a type. Constants may only be of basic types (numbers, strings, bools) — you cannot make a constant slice or struct.

```go
const Pi = 3.14159          // untyped float constant — flexible
const Greeting = "hello"    // untyped string constant
const MaxUsers int = 1000   // typed constant — locked to int

const (
	KB = 1 << (10 * 1) // 1024
	MB = 1 << (10 * 2) // 1048576
	GB = 1 << (10 * 3)
)
```

**`iota`** is a constant generator that auto-increments within a `const` block: it is `0` on the first line and increases by one per line. It exists to make enumerations concise and self-maintaining (insert a value in the middle and the rest renumber automatically). This is Go's idiomatic way to build enums.

```go
type Weekday int

const (
	Sunday Weekday = iota // 0
	Monday                // 1 (the expression `Weekday = iota` is implicitly repeated)
	Tuesday               // 2
	Wednesday             // 3
	Thursday              // 4
	Friday                // 5
	Saturday              // 6
)

// iota with expressions builds nice scaled enums:
type ByteSize float64

const (
	_           = iota             // skip 0 using the blank identifier
	KiB ByteSize = 1 << (10 * iota) // 1 << 10
	MiB                             // 1 << 20
	GiB                             // 1 << 30
)

// Give your enum a String() method (§9) so it prints nicely; tools like
// `stringer` (go generate) can auto-generate this.
func (d Weekday) String() string {
	return [...]string{"Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"}[d]
}
```

> **Best practice recap (§3):** prefer `:=` for locals, `var` for package scope / explicit zero values; remember conversions are always explicit (and `string(int)` is a trap — use `strconv`); lean on zero values and design types whose zero value is useful; use `iota` for enums and give enum types a `String()` method.

---

## 4. Strings, Runes, Bytes, UTF-8 & Formatting

This is one of the most misunderstood corners of Go, so we go slowly. Getting the string/byte/rune model right prevents a whole class of bugs with non-ASCII text.

### 4.1 What a string actually is **[B/I]**

A Go **`string` is an immutable, read-only sequence of *bytes*.** It is not a sequence of characters, and it is not tied to any particular encoding *by the language* — but by overwhelming convention (and the way string literals and the stdlib work) the bytes hold **UTF-8** encoded text. Two consequences fall out of "immutable bytes":

1. **Indexing a string gives you a `byte` (`uint8`), not a character.** `s[0]` is the first *byte*. For ASCII that happens to equal the character; for anything multi-byte it does not.
2. **`len(s)` is the number of bytes, not the number of characters.** `len("héllo")` is 6, not 5, because `é` is two bytes in UTF-8.

The reasoning: strings are used constantly and must be cheap to pass around (a string is just a pointer + length under the hood, and slicing shares the bytes). Immutability makes that sharing safe. Choosing bytes as the element type keeps the model honest about what is really in memory; the language gives you `rune` and `range` to work at the character level when you need to.

```go
s := "héllo" // 'é' (U+00E9) is 2 bytes in UTF-8
fmt.Println(len(s))          // 6  -> BYTES, not characters
fmt.Println(s[1])            // 195 -> the first BYTE of 'é' (a number!), not 'é'
fmt.Printf("%c\n", s[0])     // h   -> byte 0 happens to be ASCII 'h'
```

### 4.2 Runes — the character-level view **[B/I]**

A **`rune` is an alias for `int32`** and represents a single **Unicode code point** (roughly, "a character"). When you need to think in characters rather than bytes, you work with runes. Two tools convert between the byte view and the rune view:

- **`for i, r := range s`** iterates the string *by rune*, decoding UTF-8 as it goes. `i` is the **byte index** where the rune starts; `r` is the `rune`. This is the correct way to walk characters.
- **`[]rune(s)`** decodes the whole string into a slice of runes (so `len([]rune(s))` is the character count). **`[]byte(s)`** gives the raw bytes. Both copy.

```go
s := "héllo"

// Byte count vs rune (character) count:
fmt.Println(len(s))            // 6  (bytes)
fmt.Println(len([]rune(s)))    // 5  (characters / code points)

// range decodes UTF-8 -> you get runes and their starting byte offsets:
for i, r := range s {
	// i jumps from 1 to 3 across the 2-byte 'é'
	fmt.Printf("byte %d: %c (U+%04X)\n", i, r, r)
}

// A rune literal uses single quotes and is just a number:
var heart rune = '♥' // U+2665; fmt.Printf("%c", heart) prints ♥
fmt.Println(heart)    // 9829
```

> **Mental model:** **byte = storage unit, rune = character.** Index/`len` work in bytes; `range` and `[]rune` work in characters. Reach for runes only when you actually need character semantics (counting visible characters, reversing text, validating a single character); a lot of real code can stay at the byte level (e.g. searching for an ASCII delimiter).

### 4.3 Building and mutating strings efficiently **[I]**

Because strings are immutable, `s = s + x` in a loop allocates a brand-new string every iteration — O(n²) and a classic performance trap. To build a string piece by piece, use **`strings.Builder`** (writes into a growable buffer, then hands you the final string with no extra copy). For byte-level manipulation use **`[]byte`** / **`bytes.Buffer`**.

```go
var b strings.Builder           // zero value is ready to use (§3.4)
for i := 0; i < 3; i++ {
	fmt.Fprintf(&b, "item-%d ", i) // Builder implements io.Writer
}
result := b.String()             // "item-0 item-1 item-2 "
```

### 4.4 The `strings`, `strconv`, and `unicode` packages **[B/I]**

- **`strings`** — the everyday text toolbox: `Contains`, `HasPrefix`/`HasSuffix`, `Index`, `Split`, `Join`, `Replace`/`ReplaceAll`, `ToUpper`/`ToLower`, `TrimSpace`/`Trim`/`TrimPrefix`, `Fields` (split on whitespace), `Repeat`, `EqualFold` (case-insensitive compare), and the `Builder`/`Reader` types.
- **`strconv`** — convert between strings and other types **safely**: `Itoa`/`Atoi` (int↔string), `ParseInt`/`ParseFloat`/`ParseBool`, `FormatInt`/`FormatFloat`, and `Quote`. These return an `error` so you can handle bad input — unlike `fmt.Sprint`, they are the right tool when the input might be malformed (e.g. parsing user input or config).
- **`unicode`** — classify runes: `IsLetter`, `IsDigit`, `IsSpace`, `IsUpper`, `ToLower`, etc. Use these instead of ASCII range checks if you want correct behaviour for non-English text.

```go
n, err := strconv.Atoi("42")        // string -> int, with error if not numeric
if err != nil { /* handle bad input */ }
s := strconv.Itoa(42)               // int -> "42"
f, _ := strconv.ParseFloat("3.14", 64)
ok, _ := strconv.ParseBool("true")

upper := strings.ToUpper("héllo")   // "HÉLLO"
parts := strings.Split("a,b,c", ",") // ["a" "b" "c"]
joined := strings.Join(parts, "-")   // "a-b-c"
_ = n; _ = s; _ = f; _ = ok; _ = upper; _ = joined
```

### 4.5 Formatting with `fmt` — the verb reference **[B]**

`fmt` is Go's formatted I/O package. The core functions: **`Print`/`Println`/`Printf`** (write to stdout), **`Sprint`/`Sprintf`** (return a string), **`Fprintf`** (write to any `io.Writer`), and **`Errorf`** (build an `error`, §10). `Printf`-style functions use **verbs** — `%`-prefixed placeholders:

| Verb | Meaning | Example output |
|---|---|---|
| `%v` | default format for any value | `{Ada 30}` |
| `%+v` | struct with field names | `{Name:Ada Age:30}` |
| `%#v` | Go-syntax representation | `main.User{Name:"Ada", Age:30}` |
| `%T` | the value's type | `main.User` |
| `%d` | integer (base 10) | `42` |
| `%b` `%o` `%x` `%X` | binary / octal / hex | `101010` `52` `2a` `2A` |
| `%f` `%.2f` | float; with precision | `3.140000` `3.14` |
| `%e` `%g` | scientific / compact float | `3.14e+00` `3.14` |
| `%s` | string (or `Stringer`/`error`) | `hello` |
| `%q` | quoted/escaped string or rune | `"hello"` |
| `%c` | the character for a rune | `A` |
| `%t` | boolean | `true` |
| `%p` | pointer address | `0xc0000b4000` |
| `%w` | wrap an error (only in `Errorf`, §10) | — |

```go
type User struct{ Name string; Age int }
u := User{"Ada", 30}

fmt.Printf("%v\n", u)   // {Ada 30}
fmt.Printf("%+v\n", u)  // {Name:Ada Age:30}   <- great for debugging
fmt.Printf("%#v\n", u)  // main.User{Name:"Ada", Age:30}
fmt.Printf("%T\n", u)   // main.User
fmt.Printf("%5d|%-5d|%05d\n", 42, 42, 42) // width & flags:   "   42|42   |00042"
fmt.Printf("%.3f\n", 3.14159)             // "3.142"
fmt.Printf("%q\n", "hi\tthere")           // "\"hi\\tthere\""

// %v is what gets called when you print most things; implement String() (§9)
// to control how YOUR type renders under %v / %s.
```

> **Best practice recap (§4):** internalize byte-vs-rune (index/`len` = bytes, `range`/`[]rune` = characters); never `+=` strings in a loop — use `strings.Builder`; use `strconv` (not `fmt`) when parsing might fail and you need the error; reach for `%+v` when debugging structs and give your types a `String()` method for clean output.

---

## 5. Control Flow: if / for / switch / defer / goto

Go's control flow is deliberately minimal: **one loop keyword** (`for`), no `while`, no `do-while`, no parentheses around conditions, and braces are mandatory. The minimalism is the point — fewer forms means less to read and fewer ways to write a subtle bug.

### 5.1 `if` — with an optional init statement **[B]**

`if` takes a boolean condition (no parentheses; braces required). Its distinctive feature is the optional **init statement** that runs before the condition, with its variables scoped to the `if`/`else`. This is the idiomatic home for the `(value, error)` pattern: do the call, then check the error, and the variables don't leak into the surrounding scope.

```go
// Plain if/else if/else:
if score >= 90 {
	grade = "A"
} else if score >= 80 {
	grade = "B"
} else {
	grade = "C"
}

// if WITH an init statement — n and err exist only inside this if/else.
// This is the canonical Go error-handling shape:
if n, err := strconv.Atoi(input); err != nil {
	fmt.Println("not a number:", err)
} else {
	fmt.Println("parsed:", n)
}
// n and err are NOT visible here — scope ends with the if.
```

> **Idiom:** when a function returns `(result, error)`, handle the error first and return early ("guard clauses"), keeping the happy path un-indented. Deeply nested `if`s are a code smell in Go.

### 5.2 `for` — the only loop, in all its forms **[B]**

Go has exactly one loop keyword, but `for` flexes into every shape you need:

```go
// 1) Classic three-part loop (init; condition; post):
for i := 0; i < 5; i++ {
	fmt.Println(i)
}

// 2) Condition-only — this is Go's "while":
n := 10
for n > 0 {
	n--
}

// 3) Infinite loop — Go's "while(true)"; exit with break/return:
for {
	if done() {
		break
	}
}

// 4) range — iterate over slices/arrays/strings/maps/channels (and, since 1.23,
//    custom iterator functions). For slices/arrays you get index + value:
nums := []int{10, 20, 30}
for i, v := range nums {
	fmt.Println(i, v) // 0 10 / 1 20 / 2 30
}
for _, v := range nums { // use _ to ignore the index
	fmt.Println(v)
}

// range over a map yields key, value (in RANDOM order — never rely on order):
for k, v := range map[string]int{"a": 1, "b": 2} {
	fmt.Println(k, v)
}

// 5) range over an integer (Go 1.22+): iterate 0..n-1 with no slice needed.
for i := range 5 { // 0,1,2,3,4
	fmt.Println(i)
}
```

> **⚡ Version note — the loop variable change (Go 1.22):** Before 1.22, the `for` loop variable (`v` above) was a **single variable reused** across all iterations. If you captured it in a closure or goroutine, every closure saw the *final* value — the single most infamous Go gotcha. **Since Go 1.22 (with `go 1.22+` in `go.mod`), each iteration gets a fresh variable**, so the bug is gone. See §19 for the full before/after example; it still bites you in older codebases.

`break` and `continue` work as usual; both accept an optional **label** to break/continue an *outer* loop — useful for nested loops:

```go
outer:
for i := 0; i < 3; i++ {
	for j := 0; j < 3; j++ {
		if i*j > 2 {
			break outer // break the OUTER loop, not just the inner one
		}
	}
}
```

### 5.3 `switch` — expression, tagless, type, and fallthrough **[B/I]**

Go's `switch` is more powerful and safer than C's. Key differences and the reasoning:

- **No implicit fall-through.** Each `case` breaks automatically — you almost never want C's accidental fall-through, so Go reverses the default. Use the explicit `fallthrough` keyword on the rare occasion you want it.
- **Cases can be any value, not just constants**, and a case can list multiple values.
- **A "tagless" switch** (`switch {`) is a clean replacement for a long `if/else if` chain.
- **Type switch** (`switch v := x.(type)`) branches on the *dynamic type* of an interface value (§9).

```go
// Expression switch with multiple values per case and a default:
switch day {
case "Sat", "Sun":
	fmt.Println("weekend")
default:
	fmt.Println("weekday")
}

// switch with an init statement (like if):
switch os := runtime.GOOS; os {
case "windows":
	fmt.Println("on Windows")
case "linux", "darwin":
	fmt.Println("unix-like")
}

// Tagless switch — replaces an if/else-if ladder, reads top to bottom:
switch {
case score >= 90:
	grade = "A"
case score >= 80:
	grade = "B"
default:
	grade = "F"
}

// Explicit fallthrough (rare): continue into the NEXT case unconditionally.
switch n {
case 1:
	fmt.Println("one")
	fallthrough // execution falls into case 2
case 2:
	fmt.Println("two")
}
```

### 5.4 `defer` — deferred calls, LIFO, and argument timing **[B/I, important]**

`defer` schedules a function call to run **when the surrounding function returns** — whether it returns normally or via a panic. Its primary purpose is **cleanup that should always happen**: closing files, unlocking mutexes, closing rows, releasing resources. Putting the cleanup right next to the acquisition makes leaks far less likely and keeps the two visually paired.

Two rules you must internalize:

1. **Deferred calls run in Last-In-First-Out (LIFO) order.** The most recently deferred runs first. This naturally unwinds nested resources in reverse acquisition order.
2. **Arguments to a deferred call are evaluated *immediately* (when `defer` runs), but the call itself runs later.** This trips people up constantly. The *function value* and its *arguments* are captured now; the *body* executes at return.

```go
func process(path string) error {
	f, err := os.Open(path)
	if err != nil {
		return err
	}
	defer f.Close() // runs when process returns — pairs cleanup with acquisition

	mu.Lock()
	defer mu.Unlock() // unlocks no matter how we return (even on panic)

	// ... use f ...
	return nil
}

// LIFO + argument timing demo:
func demo() {
	for i := 0; i < 3; i++ {
		defer fmt.Println("deferred", i) // i is captured NOW, by value
	}
	fmt.Println("body done")
}
// Output:
// body done
// deferred 2     <- LIFO: last deferred runs first
// deferred 1
// deferred 0     <- each captured its own i at defer time
```

A defer with a closure can read/modify **named return values** (§8), which enables clean error-decoration and recover patterns:

```go
func writeFile(name string) (err error) { // NAMED return value 'err'
	f, err := os.Create(name)
	if err != nil {
		return err
	}
	// A closure deferred here can SEE and OVERRIDE the named return value.
	defer func() {
		if cerr := f.Close(); cerr != nil && err == nil {
			err = cerr // surface a Close() error if the body didn't already fail
		}
	}()
	_, err = f.Write([]byte("data"))
	return err
}
```

> **Gotcha:** `defer` inside a *loop* does **not** run at the end of each iteration — it runs at function return. Deferring `Close` in a long loop can leak file descriptors until the function exits. Either move the work into a helper function (so each call's defers fire promptly) or close explicitly inside the loop.

### 5.5 `goto` — exists, rarely justified **[A]**

Go has `goto`, which jumps to a label in the same function. It cannot jump into a block or over variable declarations. In practice you almost never need it — labeled `break`/`continue` and early returns cover the legitimate cases. The honest niche is breaking out of deeply nested generated code or consolidating error cleanup in low-level code; even there, prefer structured alternatives.

```go
	i := 0
loop:
	if i < 3 {
		fmt.Println(i)
		i++
		goto loop // jumps back to the label
	}
```

> **Best practice recap (§5):** put the error check in the `if` init statement and return early; remember `for` is the only loop and `range 5` works since 1.22; let `switch` (tagless) replace `if/else` ladders; pair every resource acquisition with a `defer` cleanup, and remember LIFO order + immediate argument evaluation; avoid `defer` in tight loops; treat `goto` as a last resort.

---

## 6. Composite Types: Arrays, Slices, Maps, Structs

These four build every Go data structure. The slice is the most important — and the most misunderstood — so we dwell on its internals.

### 6.1 Arrays — fixed size, value semantics **[B]**

An **array** has a length that is *part of its type*: `[3]int` and `[4]int` are different, incompatible types. Arrays are **value types** — assigning one or passing it to a function **copies the whole array**. That copying cost (and the rigid fixed length) is why arrays are rarely used directly; they exist mainly as the *backing store* underneath slices. You will use them for fixed-size things like hashes (`[32]byte`).

```go
var a [3]int          // [0 0 0] — zero-valued, fixed length 3
a[0] = 10
b := a                // COPY — b is independent of a
b[0] = 99
fmt.Println(a, b)     // [10 0 0] [99 0 0]
fmt.Println(len(a))   // 3; length is fixed and part of the type

primes := [...]int{2, 3, 5, 7} // [...] lets the compiler count -> [4]int
_ = primes
```

### 6.2 Slices — the slice header (ptr/len/cap), explained deeply **[B/I, the core concept]**

A **slice** is Go's dynamic, flexible view into a contiguous block of memory. This is *the* data structure you use for "a list of things." Understanding it deeply is the single highest-leverage thing in this section, because almost every slice bug comes from not knowing how it works.

A slice value is a tiny **3-word header**:

```text
slice = { ptr -> backing array, len, cap }
```

- **`ptr`** points to the first element the slice can see, inside a *backing array* that lives elsewhere in memory.
- **`len`** (length) is how many elements the slice currently exposes — what `len(s)` returns and what you can index/range.
- **`cap`** (capacity) is how many elements exist from `ptr` to the end of the backing array — the room available before a reallocation is needed (`cap(s)`).

Because the header *points at* a backing array, **a slice is a reference-like value**: copying the header (assigning a slice, passing it to a function) is cheap and the copies **share the same backing array.** This single fact explains both slices' efficiency and their aliasing gotchas.

```go
s := []int{10, 20, 30, 40, 50} // len 5, cap 5, backed by an array of 5 ints
fmt.Println(len(s), cap(s))     // 5 5

// Slicing creates a NEW header pointing INTO THE SAME backing array:
sub := s[1:3]                   // elements 20,30 ; len 2, cap 4 (index 1..end)
fmt.Println(sub, len(sub), cap(sub)) // [20 30] 2 4

// Because they share storage, writing through one is visible through the other:
sub[0] = 999
fmt.Println(s)                  // [10 999 30 40 50]  <- s changed too! (ALIASING)
```

### 6.3 `make`, `append`, and growth **[B/I]**

You create slices with a literal, with `make` (when you know the size/capacity up front), or by slicing something. **`make([]T, len, cap)`** pre-allocates — supplying a capacity hint avoids repeated reallocation when you know roughly how many elements you'll add (a real performance win in hot loops).

**`append`** adds elements and returns a (possibly new) slice header. The mechanism, which you must understand:

1. If there is spare capacity (`len < cap`), `append` writes into the existing backing array and returns a header with a larger `len` — **same backing array.**
2. If there is no room (`len == cap`), `append` **allocates a bigger backing array**, copies the elements over, adds the new ones, and returns a header pointing at the **new** array. The old array is untouched.

This is why you **must always write `s = append(s, x)`** — the returned header may differ from the input. The growth strategy roughly doubles capacity for small slices and grows more gently for large ones (an implementation detail; don't depend on the exact factor).

```go
s := make([]int, 0, 4) // len 0, cap 4 — room for 4 before reallocating
fmt.Println(len(s), cap(s)) // 0 4

s = append(s, 1, 2, 3, 4) // fits in cap -> same backing array
fmt.Println(len(s), cap(s)) // 4 4
s = append(s, 5)          // exceeds cap -> NEW backing array (cap grows, e.g. to 8)
fmt.Println(len(s), cap(s)) // 5 8 (cap is implementation-defined)

// min/max/clear builtins (Go 1.21+) — handy with slices/numbers:
fmt.Println(min(3, 7), max(3, 7)) // 3 7
clear(s)                          // zeroes all elements (for maps, deletes all keys)
```

### 6.4 The aliasing gotchas (and how to avoid them) **[I, important]**

Because `append` *sometimes* mutates shared storage and *sometimes* reallocates, slices that share a backing array can corrupt each other in surprising ways:

```go
// Gotcha 1: append into a sub-slice can overwrite the parent's data.
original := []int{1, 2, 3, 4, 5}
head := original[:2]            // [1 2], but cap is 5 — it can still "see" 3,4,5
head = append(head, 99)         // writes into index 2 of the SHARED array!
fmt.Println(original)           // [1 2 99 4 5]  <- original was clobbered

// Fix A: a 3-index slice limits capacity so append is forced to reallocate.
//   s[low:high:max]  -> len = high-low, cap = max-low
safe := original[:2:2]          // len 2, cap 2 -> next append MUST allocate fresh
safe = append(safe, 99)         // does NOT touch original
fmt.Println(original)           // unchanged

// Fix B: copy / slices.Clone to get a fully independent slice.
indep := slices.Clone(original) // (Go 1.21+) deep-enough copy of the elements
indep[0] = -1
fmt.Println(original[0])        // still 1
```

**`copy(dst, src)`** copies `min(len(dst), len(src))` elements and is the explicit way to duplicate without aliasing. Removing an element idiomatically also uses sharing-aware operations (`slices.Delete` since 1.21 handles this correctly).

```go
dst := make([]int, 3)
n := copy(dst, []int{1, 2, 3, 4}) // copies 3 (len of dst); n == 3
_ = n
```

> **Three rules that prevent most slice bugs:** (1) always `s = append(s, …)`; (2) if you hand out a sub-slice that the caller might `append` to, give it a bounded capacity with a 3-index slice (`s[a:b:b]`) or `slices.Clone`; (3) never assume a slice you received is yours to mutate — copy it if you need to keep an independent version.

### 6.5 Maps — hash tables with comma-ok and random order **[B/I]**

A **map** is Go's built-in hash table: `map[K]V` maps keys of type `K` to values of type `V`. The key type must be **comparable** (usable with `==`): numbers, strings, bools, pointers, and structs/arrays of comparable types qualify; slices, maps, and functions do **not**. Maps are reference-like (a map value is a pointer to the underlying hash table), so copying a map variable shares the same table.

Critical behaviours and the reasoning behind them:

- **A `nil` map (the zero value) can be *read* but not *written*.** Reading a missing key returns the value's zero value; *writing* to a nil map **panics.** So you must `make` a map (or use a literal) before adding keys. The asymmetry exists so the zero value is still useful for lookups (e.g. a nil map is a fine "empty config").
- **The comma-ok idiom distinguishes "missing" from "present but zero".** `v, ok := m[k]` sets `ok=false` if the key is absent. Without it you can't tell a stored `0` from an absent key.
- **Iteration order is randomized** *on purpose* — Go shuffles it so no one accidentally depends on an order that isn't guaranteed. If you need order, collect keys into a slice and sort them.
- **`delete(m, k)`** removes a key (a no-op if absent). **`clear(m)`** (1.21+) empties the whole map.

```go
m := make(map[string]int)  // empty, ready to write
m["apples"] = 3
m["pears"] = 0

v := m["bananas"]          // 0 (zero value) — but does the key exist? unknown
v, ok := m["bananas"]      // v=0, ok=false -> key absent
v, ok = m["pears"]         // v=0, ok=true  -> key present, value is 0
fmt.Println(v, ok)

delete(m, "apples")        // remove a key
fmt.Println(len(m))        // count of keys

// Deterministic iteration: sort the keys first.
keys := make([]string, 0, len(m))
for k := range m {
	keys = append(keys, k)
}
sort.Strings(keys)
for _, k := range keys {
	fmt.Println(k, m[k])
}

var nilMap map[string]int
fmt.Println(nilMap["x"])   // 0 — reading a nil map is fine
// nilMap["x"] = 1         // PANIC: assignment to entry in nil map
```

> **Gotcha:** you cannot take the address of a map element (`&m[k]` is illegal) because the table can move entries during growth. To mutate a struct stored in a map, either read-modify-write the whole value (`x := m[k]; x.F = 1; m[k] = x`) or store pointers (`map[K]*V`).

### 6.6 Structs — composition over inheritance **[B/I]**

A **struct** is a typed collection of named fields — Go's way to model a record/entity. Structs are **value types** (copied on assignment/pass, like arrays), so use pointers (`*T`) when you want shared, mutable state or want to avoid copying large structs (§7).

```go
type User struct {
	ID    int
	Name  string
	Email string
}

u := User{ID: 1, Name: "Ada", Email: "ada@x.io"} // keyed literal (preferred — robust to field reordering)
u2 := User{2, "Bob", "bob@x.io"}                  // positional literal (fragile; avoid)
p := &User{Name: "Carol"}                         // &T literal -> *User; other fields zero
fmt.Println(u.Name, p.Name)                       // dot access; works through pointers too (auto-deref)
```

**Struct tags** are string metadata attached to fields, read at runtime via reflection. They are how packages like `encoding/json` learn how to map fields to external representations. Tags don't change Go semantics; they're instructions for libraries.

```go
type Product struct {
	ID    int     `json:"id"`
	Name  string  `json:"name"`
	Price float64 `json:"price,omitempty"` // omit from JSON if zero
	hidden string `json:"-"`                // never serialized (also unexported)
}
```

**Embedding** is Go's composition mechanism — and its deliberate alternative to inheritance. You embed a type by listing it *without a field name*; the outer struct then **promotes** the embedded type's fields and methods as if they were its own. This gives you code reuse (you get the embedded behaviour for free) without the fragility of an inheritance hierarchy. The mantra is **"composition over inheritance"**: you assemble behaviour from parts, and there is no `super`, no overriding surprises — just promotion, which you can shadow by redeclaring a method on the outer type.

```go
type Animal struct {
	Name string
}
func (a Animal) Speak() string { return a.Name + " makes a sound" }

type Dog struct {
	Animal      // embedded (no field name) — Dog "has-a" Animal, gets its fields+methods
	Breed string
}

d := Dog{Animal: Animal{Name: "Rex"}, Breed: "Lab"}
fmt.Println(d.Name)     // "Rex"  — promoted field from Animal
fmt.Println(d.Speak())  // "Rex makes a sound" — promoted method
// You can still reach the embedded value explicitly: d.Animal.Name
```

**Anonymous structs** declare a one-off struct type inline, with no name. They're handy for short-lived shapes: a quick table-test row, a JSON payload you assemble once, or grouping a few values.

```go
point := struct {
	X, Y int
}{X: 1, Y: 2} // declared and instantiated in one go

// Common in tests and ad-hoc JSON:
resp := struct {
	Status string `json:"status"`
	Count  int    `json:"count"`
}{Status: "ok", Count: 3}
_ = point; _ = resp
```

> **Best practice recap (§6):** prefer slices to arrays; truly understand the slice header and `append` aliasing (3-index slices / `slices.Clone` to stay safe); `make` maps before writing and use comma-ok; never rely on map order; use keyed struct literals; model relationships with **embedding (composition)**, not inheritance; use struct tags for serialization and unexported fields to protect invariants.

---

## 7. Pointers

### 7.1 What pointers are, and why Go has them (without the footguns) **[B/I]**

A **pointer** holds the *memory address* of a value. `*T` is "pointer to a `T`"; `&x` takes the address of `x`; `*p` *dereferences* the pointer to get (or set) the value it points at. Pointers let you (a) **share** a value so changes are seen by everyone holding the pointer, and (b) **avoid copying** large values when you pass them around.

Go keeps pointers but removes their most dangerous features on purpose:

- **No pointer arithmetic.** You cannot do `p++` to walk through memory (the unsafe escape hatch `unsafe.Pointer` exists but is out of scope for normal code). This eliminates buffer-overrun-style bugs.
- **No dangling pointers.** The garbage collector keeps memory alive as long as a pointer references it, so you can safely return a pointer to a local variable — Go moves it to the heap automatically (escape analysis, §17). In C that would be a use-after-free; in Go it's normal and safe.
- **Nil is the zero value**, and dereferencing `nil` panics (a clear, immediate failure) rather than corrupting memory.

```go
x := 10
p := &x        // p is *int, holds the address of x
fmt.Println(*p) // 10 — dereference to read
*p = 20         // dereference to write -> changes x
fmt.Println(x)  // 20

var np *int     // nil (zero value)
// fmt.Println(*np) // PANIC: nil pointer dereference
if np != nil {  // always guard before dereferencing a maybe-nil pointer
	fmt.Println(*np)
}
```

### 7.2 Why pointers change behaviour: value vs pointer semantics **[B/I]**

Go passes **everything by value** — function arguments are copied. So if you pass a `User` struct, the function gets a *copy* and cannot affect your original. If you pass a `*User`, the function gets a copy *of the pointer*, which still points at your original — so it *can* mutate it. This is the whole reason pointers matter for function parameters and method receivers (§9).

```go
type Counter struct{ n int }

func incValue(c Counter)  { c.n++ } // operates on a COPY — caller unaffected
func incPtr(c *Counter)   { c.n++ } // follows the pointer -> mutates the caller's value

func main() {
	c := Counter{}
	incValue(c)
	fmt.Println(c.n) // 0 — the copy was incremented, not c
	incPtr(&c)
	fmt.Println(c.n) // 1 — c itself was modified
}
```

### 7.3 `new` vs `&T{}` **[I]**

There are two ways to get a pointer to a fresh value:

- **`&T{...}`** — a composite literal preceded by `&`. Allocates a `T`, initializes the fields you specify, and returns `*T`. This is the idiomatic, common form because you usually want to set fields.
- **`new(T)`** — allocates a zero-valued `T` and returns `*T`. Equivalent to `&T{}` with no fields set. It's rarely used for structs (the literal reads better) but is handy for getting a pointer to a basic type's zero value (`new(int)` → `*int` pointing at `0`).

```go
u1 := &User{Name: "Ada"} // idiomatic: pointer + initialized fields
u2 := new(User)          // *User pointing at a zero User{} ; same as &User{}
pi := new(int)           // *int -> 0
*pi = 42
fmt.Println(u1.Name, u2.Name, *pi)
```

### 7.4 When to use a pointer (and when not) **[I]**

Pointers aren't free — they add a level of indirection, can push values onto the heap (GC pressure), and introduce nil-ness you must guard. Use them deliberately:

**Use a pointer when:**
- you need the callee/method to **mutate** the caller's value;
- the value is **large** and copying it would be wasteful (a struct with many fields, or one embedded in a hot path);
- the type contains something that **must not be copied** (a `sync.Mutex`, a `sync.WaitGroup`) — copying it breaks correctness;
- you want to represent **"optional"/"absent"** distinctly from the zero value (a `*int` can be `nil` to mean "not set", whereas `0` is ambiguous).

**Prefer a value when:**
- the type is **small** (a couple of words) — copying is as cheap as or cheaper than chasing a pointer, and values are simpler and allocation-free;
- you want **immutability**/clear ownership — a copy can't be mutated by the callee;
- the type's zero value is meaningful and you have no nil to worry about.

> **Best practice recap (§7):** pointers exist to share and to avoid copying — not as a default. No arithmetic, no dangling refs (the GC has your back, returning `&local` is fine). Use `&T{}` over `new` for structs. Guard maybe-nil pointers before dereferencing. Never copy types that contain a mutex. Be *consistent* about value vs pointer within a type's method set (§9).

---

## 8. Functions: Returns, Variadics, Closures, Higher-Order

Functions in Go are **first-class values**: you can store them in variables, pass them as arguments, return them, and put them in slices/maps. That, plus multiple return values and closures, gives Go a surprisingly functional flavour despite its imperative surface.

### 8.1 Multiple return values **[B, fundamental]**

Go functions can return more than one value, and this is not a minor convenience — it is the foundation of Go's entire error-handling model. The overwhelmingly common pattern is **`(result, error)`**: a function returns its result *and* whether it succeeded, and the caller checks the error immediately. Because errors are ordinary return values (not exceptions), the control flow is explicit and local — you can see exactly where things can fail.

```go
// Returns the quotient AND an error (instead of throwing).
func divide(a, b float64) (float64, error) {
	if b == 0 {
		return 0, fmt.Errorf("division by zero")
	}
	return a / b, nil // nil error means success
}

func main() {
	q, err := divide(10, 2)
	if err != nil {       // the idiomatic check, right at the call site
		log.Fatal(err)
	}
	fmt.Println(q) // 5

	// The "comma ok" cousin appears in map reads, type assertions, channel receives.
	_, err = divide(1, 0)
	fmt.Println(err) // division by zero
}
```

### 8.2 Named return values **[I]**

You can name the return values in the signature. Named returns are pre-declared (to their zero values) and you can `return` with no arguments ("naked return") to return their current values. Their best, idiomatic use is letting a deferred closure inspect or modify the result (e.g. wrapping an error or recovering from a panic, §5.4/§10). Use them for documentation and that defer pattern; avoid *naked* returns in long functions — they hurt readability because the reader has to scroll up to see what's being returned.

```go
// Named returns document intent AND let the defer adjust 'err'.
func readConfig(path string) (cfg Config, err error) {
	defer func() {
		if err != nil {
			err = fmt.Errorf("readConfig %q: %w", path, err) // decorate on the way out
		}
	}()
	data, err := os.ReadFile(path)
	if err != nil {
		return // naked return: returns current cfg (zero) and err
	}
	err = json.Unmarshal(data, &cfg)
	return cfg, err // explicit return is clearer for most functions
}
```

### 8.3 Variadic functions **[B/I]**

A **variadic** parameter (`...T`) lets a function accept any number of trailing arguments of type `T`; inside the function it is a `[]T`. `fmt.Println` is variadic. To pass an existing slice to a variadic parameter, "spread" it with `slice...`.

```go
func sum(nums ...int) int { // nums is a []int inside the function
	total := 0
	for _, n := range nums {
		total += n
	}
	return total
}

fmt.Println(sum(1, 2, 3))      // 6 — pass any number of args
fmt.Println(sum())             // 0 — zero args is fine (nums is empty)

xs := []int{4, 5, 6}
fmt.Println(sum(xs...))        // spread a slice into the variadic param -> 15
```

> **Gotcha:** the spread (`xs...`) passes the slice's backing array directly — the function could mutate your slice. And you can't mix: it's `sum(xs...)` *or* `sum(1, 2, 3)`, not both in one call.

### 8.4 Closures — functions that capture state **[I, important]**

A **closure** is a function literal that *captures* (refers to) variables from its surrounding scope. The captured variables live on as long as the closure does — they are shared *by reference*, not copied. Closures are everywhere in Go: callbacks, `defer` bodies, goroutine bodies, custom sort comparators, middleware, and stateful generators.

```go
// A closure that captures and mutates `count` across calls -> a counter generator.
func makeCounter() func() int {
	count := 0          // captured by the returned closure
	return func() int {
		count++          // each call mutates the SAME captured variable
		return count
	}
}

c := makeCounter()
fmt.Println(c(), c(), c()) // 1 2 3 — state persists in the closure
d := makeCounter()
fmt.Println(d())           // 1 — a fresh, independent `count`
```

> **⚡ Version note — closures in loops:** capturing the loop variable in a closure was the classic Go bug pre-1.22 (all closures saw the final value). **Since Go 1.22 each iteration has its own variable, so this now does what you expect.** Full example in §19. When in doubt or on older Go, copy into a local: `v := v` before capturing.

### 8.5 Higher-order functions & first-class values **[I]**

Because functions are values, you can pass behaviour around — the basis for strategies, callbacks, and the functional-options pattern. The stdlib leans on this: `slices.SortFunc`, `http.HandlerFunc`, `sync.OnceFunc`, etc.

```go
// A higher-order function: takes a transform function as an argument.
func mapInts(xs []int, f func(int) int) []int {
	out := make([]int, len(xs))
	for i, x := range xs {
		out[i] = f(x)
	}
	return out
}

doubled := mapInts([]int{1, 2, 3}, func(n int) int { return n * 2 }) // [2 4 6]

// Functions in variables, and as map values (a tiny dispatch table):
ops := map[string]func(a, b int) int{
	"add": func(a, b int) int { return a + b },
	"mul": func(a, b int) int { return a * b },
}
fmt.Println(ops["add"](2, 3), ops["mul"](2, 3)) // 5 6
_ = doubled
```

### 8.6 Recursion **[B/I]**

Go supports recursion normally. There is **no tail-call optimization**, so very deep recursion can overflow the goroutine stack (though Go stacks grow dynamically, so the practical limit is high). For deep or unbounded recursion, prefer an explicit loop with your own stack. Recursion shines for naturally recursive structures (trees, JSON, filesystem walks).

```go
func factorial(n int) int {
	if n <= 1 { // base case — every recursion needs one
		return 1
	}
	return n * factorial(n-1) // recursive case
}
fmt.Println(factorial(5)) // 120
```

> **Best practice recap (§8):** return `(result, error)` and check the error immediately; use named returns mainly for the defer-decorates-error pattern (avoid naked returns in long funcs); know that variadic spread shares the backing array; reach for closures for callbacks/state but mind loop capture on pre-1.22 Go; pass functions as values to keep code flexible.

---

## 9. Methods & Interfaces

This is the conceptual heart of Go's type system. Methods attach behaviour to types; interfaces describe behaviour abstractly; and Go's *implicit* interface satisfaction is what makes the whole thing feel light and decoupled. Take your time here.

### 9.1 Methods — functions with a receiver **[B/I]**

A **method** is a function with a special **receiver** parameter written before the name. The receiver binds the method to a type. You can define methods on any type *you* declare in your package — not just structs (you can give a named slice or int type methods too). The reasoning: behaviour belongs with data, but Go keeps it decoupled from classes — methods are just functions with nicer call syntax (`u.Greet()` instead of `Greet(u)`).

```go
type User struct {
	Name string
}

// Value receiver: gets a COPY of the User. Good for read-only methods.
func (u User) Greet() string {
	return "Hi, I'm " + u.Name
}

// Pointer receiver: gets a pointer to the User -> can MUTATE it.
func (u *User) Rename(newName string) {
	u.Name = newName
}

func main() {
	u := User{Name: "Ada"}
	fmt.Println(u.Greet()) // "Hi, I'm Ada"
	u.Rename("Bob")        // Go auto-takes &u because u is addressable
	fmt.Println(u.Name)    // "Bob"
}
```

You can also attach methods to non-struct named types — this is how enums get a `String()`:

```go
type Celsius float64
func (c Celsius) Fahrenheit() float64 { return float64(c)*9/5 + 32 }
fmt.Println(Celsius(100).Fahrenheit()) // 212
```

### 9.2 Value vs pointer receivers — the addressability rule **[I, important]**

Choosing value vs pointer receiver is one of the most common Go questions. The rules:

**Use a pointer receiver when:**
- the method **modifies** the receiver (a value receiver mutates a copy — the change is lost);
- the struct is **large** (avoid copying on every call);
- the type contains a **`sync.Mutex`** or similar that must not be copied;
- for **consistency** — if *any* method needs a pointer receiver, give *all* methods on that type pointer receivers, so the method set is uniform.

**Use a value receiver when** the type is small and immutable-ish (a `time.Time`, a small config), or you specifically want copy semantics.

**The addressability subtlety:** Go automatically takes the address (`u.Rename()` becomes `(&u).Rename()`) — *but only if `u` is addressable.* A variable is addressable; a map element or a function's return value is not. So calling a pointer-receiver method on a non-addressable value is a compile error:

```go
type Counter struct{ n int }
func (c *Counter) Inc() { c.n++ }

m := map[string]Counter{"a": {}}
// m["a"].Inc() // COMPILE ERROR: m["a"] is not addressable, can't take its address
c := m["a"]      // copy out
c.Inc()          // works on the addressable local
m["a"] = c       // write back
```

There is a corresponding rule for **method sets and interfaces** (next): a `*T` has *all* of `T`'s methods plus the pointer-receiver ones; a `T` value has only the value-receiver methods. So if a method has a pointer receiver, **only a pointer satisfies an interface requiring it.**

### 9.3 Interfaces — implicit satisfaction **[I, the big idea]**

An **interface** is a set of method signatures — a description of *behaviour* without any implementation. A type **satisfies** an interface if it has all the interface's methods. The revolutionary part: **satisfaction is implicit.** There is no `implements` keyword. If your type happens to have the right methods, it *is* the interface, automatically — even for interfaces defined in packages you've never seen.

Why this design is powerful:

- **Decoupling.** The *consumer* of behaviour defines the interface it needs; the *provider* doesn't even have to know the interface exists. This inverts the dependency: your business logic declares `type Store interface { Save(...) error }`, and a Postgres type satisfies it just by having a `Save` method. You can swap implementations (Postgres, in-memory fake for tests, a mock) without touching the consumer.
- **No ceremony, no premature coupling.** You don't have to design an interface hierarchy up front. You extract an interface only when a second implementation or a test fake makes it worthwhile.

```go
// The interface describes WHAT we need, not WHO provides it.
type Speaker interface {
	Speak() string
}

type Dog struct{}
func (Dog) Speak() string { return "Woof" } // Dog now satisfies Speaker — implicitly

type Robot struct{}
func (Robot) Speak() string { return "Beep" } // so does Robot

// A function that accepts the INTERFACE works with any satisfying type:
func announce(s Speaker) { fmt.Println(s.Speak()) }

func main() {
	announce(Dog{})   // Woof
	announce(Robot{}) // Beep — no shared base class, no "implements", it just works
}
```

> **Compile-time assertion idiom:** to *guarantee* a type satisfies an interface (and get a clear error if you break it), add `var _ Speaker = (*Dog)(nil)` near the type. It documents intent and fails the build if `Dog` ever stops satisfying `Speaker`.

### 9.4 "Accept interfaces, return structs" **[I, key idiom]**

A guiding principle: **functions should accept interfaces (the minimal behaviour they need) and return concrete types (structs).** Accepting an interface maximizes caller flexibility — any satisfying type works. Returning a concrete struct gives the caller the full, documented API and avoids forcing them through a narrow interface they may not want. Constructors therefore return `*Thing`, not `SomeInterface`.

```go
// Accept the smallest interface you need (io.Reader), not a concrete *os.File:
func countBytes(r io.Reader) (int, error) {
	return io.Copy(io.Discard, r) // works with files, network conns, strings.Reader, ...
}

// Return a concrete type from constructors:
func NewServer(addr string) *Server { return &Server{addr: addr} } // not an interface
```

### 9.5 The empty interface and `any` **[I]**

An interface with **no methods** is satisfied by *every* type (every type has "no methods" at minimum). It is written `interface{}` and, since Go 1.18, has the alias **`any`** (use `any` — it reads better). It means "a value of any type," and it's how you write heterogeneous containers or APIs like `fmt.Println(args ...any)`. The cost: you lose static type information and must *recover* the concrete type via a type assertion or type switch before you can do much with it. Prefer generics (§11) when you want "any type" *with* type safety.

```go
var x any = 42         // holds an int
x = "now a string"     // holds a string
x = []int{1, 2, 3}     // holds a slice
fmt.Printf("%T\n", x)  // []int
```

### 9.6 Type assertions & type switches **[I]**

To get the concrete value back out of an interface, use a **type assertion** `v, ok := x.(T)`. The two-result form is safe (`ok` is false instead of panicking if the type doesn't match); the single-result form `x.(T)` panics on mismatch — only use it when you're certain. A **type switch** branches on the dynamic type and is the clean way to handle several possibilities.

```go
var x any = "hello"

// Safe type assertion (comma-ok form):
if s, ok := x.(string); ok {
	fmt.Println("a string of length", len(s))
}

// Type switch — handle multiple concrete types:
func describe(v any) string {
	switch t := v.(type) { // t has the matched concrete type in each case
	case nil:
		return "nil"
	case int:
		return fmt.Sprintf("int: %d", t)
	case string:
		return fmt.Sprintf("string of len %d", len(t))
	case []int:
		return fmt.Sprintf("slice of %d ints", len(t))
	default:
		return fmt.Sprintf("unknown type %T", t)
	}
}
```

### 9.7 Interface composition **[I]**

Small interfaces compose into bigger ones by embedding. This is why the stdlib has `io.Reader` and `io.Writer` as one-method interfaces and `io.ReadWriter` as their composition. Small interfaces are easy to satisfy, easy to fake in tests, and reusable — Go strongly favours them ("the bigger the interface, the weaker the abstraction").

```go
type Reader interface{ Read(p []byte) (n int, err error) }
type Writer interface{ Write(p []byte) (n int, err error) }

// Composition by embedding — a ReadWriter is anything that is both.
type ReadWriter interface {
	Reader
	Writer
}
```

### 9.8 The `nil` interface gotcha (typed nil) **[I/A, classic bug]**

An interface value has **two** parts internally: a *type* and a *value*. An interface is `nil` only when **both** are nil. If you store a *typed* nil pointer in an interface, the interface holds a non-nil type and is therefore **not equal to `nil`** — even though the underlying pointer is nil. This causes the infamous bug where a function "returns nil" but the caller's `err != nil` is true.

```go
type MyError struct{ Msg string }
func (e *MyError) Error() string { return e.Msg }

func doThing() error {
	var e *MyError = nil // a typed nil pointer
	return e             // BUG: returns an interface with type=*MyError, value=nil
}

func main() {
	err := doThing()
	fmt.Println(err == nil) // FALSE! — the interface is non-nil (it has a type)
}
// FIX: return a literal nil error, or only assign the pointer to the error when it's real:
//   if somethingFailed { return &MyError{...} }
//   return nil
```

### 9.9 Common stdlib interfaces you must know **[I]**

The standard library is built from a handful of tiny interfaces. Knowing them lets your types "plug in" to enormous amounts of existing code:

- **`fmt.Stringer`** — `String() string`. Implement it and `fmt` will use it under `%v`/`%s` and `Println`. Give it to your enums and domain types for readable output.
- **`error`** — `Error() string`. The error interface (§10). Any type with an `Error()` method *is* an error.
- **`io.Reader`** — `Read(p []byte) (int, error)`. The universal "stream of bytes in" abstraction; files, network conns, `strings.Reader`, gzip readers all satisfy it, so functions taking `io.Reader` work with all of them.
- **`io.Writer`** — `Write(p []byte) (int, error)`. The universal "stream of bytes out": files, `http.ResponseWriter`, `bytes.Buffer`, `os.Stdout`. This is why `fmt.Fprintf(w, ...)` works with anything.
- **`sort.Interface`**, **`io.Closer`**, **`json.Marshaler`** and friends round out the set.

```go
// Implement Stringer so your type prints nicely everywhere.
type Color int
const ( Red Color = iota; Green; Blue )
func (c Color) String() string {
	return [...]string{"Red", "Green", "Blue"}[c]
}
fmt.Println(Green) // "Green" — fmt found and used String()

// Implement io.Writer and you can fmt.Fprintf into your own type.
type UpperWriter struct{ w io.Writer }
func (u UpperWriter) Write(p []byte) (int, error) {
	return u.w.Write(bytes.ToUpper(p))
}
```

> **Best practice recap (§9):** be consistent with value vs pointer receivers (any mutation/mutex/large struct → all pointer); remember only `*T` satisfies an interface needing a pointer-receiver method; define interfaces where they're *consumed* and keep them small; accept interfaces, return structs; use `any` only when you must, prefer generics; recover concrete types with comma-ok assertions or type switches; beware the typed-nil interface; implement `Stringer`/`io.Reader`/`io.Writer`/`error` to integrate with the stdlib.

---

## 10. Errors: Wrapping, Sentinels, Custom Types, panic/recover

Go's error handling is one of its most debated and most important design choices. Once you internalize the philosophy — **errors are ordinary values, handled explicitly, not exceptions** — the verbosity stops feeling like a chore and starts feeling like clarity.

### 10.1 The `error` interface and the core philosophy **[B/I]**

`error` is just a one-method interface in the language:

```go
type error interface {
	Error() string
}
```

Any value with an `Error() string` method *is* an error. Functions that can fail return an `error` as their **last** return value; `nil` means success. The reasoning behind rejecting exceptions: exceptions create invisible control-flow paths (any line might throw, unwinding the stack to who-knows-where), which is exactly the kind of hidden complexity Go fights. Returning errors makes every failure point visible at the call site, forces the caller to decide what to do, and keeps the stack honest. The cost is the famous `if err != nil` repetition — accepted as a fair price for explicitness.

```go
func half(n int) (int, error) {
	if n%2 != 0 {
		return 0, errors.New("n must be even")
	}
	return n / 2, nil
}

v, err := half(7)
if err != nil {
	// handle it: log, return, retry, default — a DECISION, made here and now
	fmt.Println("error:", err)
	return
}
fmt.Println(v)
```

### 10.2 Creating errors: `errors.New` and `fmt.Errorf` **[B]**

- **`errors.New("message")`** — a simple, static error string.
- **`fmt.Errorf("...%v...", args)`** — a formatted error, for adding context. Use the `%w` verb (next section) when you want to *wrap* an underlying error so callers can still inspect it.

```go
err1 := errors.New("connection refused")
err2 := fmt.Errorf("failed to dial %s on port %d", host, port) // formatted context
```

> **Style:** error strings should be lowercase and not end with punctuation, because they are often wrapped (`fmt.Errorf("load config: %w", err)`) and would otherwise read like "...: Failed to read file.: ...". `go vet`'s `stylecheck`/`golint` flag capitalized error strings.

### 10.3 Wrapping with `%w`, and `errors.Is` / `errors.As` **[I, important]**

Modern Go (since 1.13) supports **error wrapping**: you attach context to an error while preserving the original so callers can still examine it. You wrap with `fmt.Errorf` and the **`%w`** verb, building a chain. To inspect the chain you use:

- **`errors.Is(err, target)`** — is `target` *anywhere* in the chain? Use it to test against **sentinel** errors (like `io.EOF`, `sql.ErrNoRows`). It replaces `err == ErrFoo`, which breaks once the error is wrapped.
- **`errors.As(err, &target)`** — is there an error of a particular *type* in the chain? If so, it assigns it to `target` so you can read its fields. Use it for **custom error types**.
- **`errors.Unwrap(err)`** — peel one layer (rarely needed directly).

The logic: wrapping lets each layer add "what I was trying to do" context (`"create user: save to db: connection refused"`) while the *bottom* layer's identity/type stays inspectable. You get great error messages *and* programmatic handling.

```go
var ErrNotFound = errors.New("not found") // a sentinel (see 10.4)

func getUser(id int) (*User, error) {
	if id <= 0 {
		// Wrap the sentinel with context using %w.
		return nil, fmt.Errorf("getUser(%d): %w", id, ErrNotFound)
	}
	return &User{}, nil
}

_, err := getUser(-1)
fmt.Println(err)                       // getUser(-1): not found
fmt.Println(errors.Is(err, ErrNotFound)) // TRUE — sentinel found through the wrap
```

```go
// errors.As recovers a custom error type from anywhere in the chain:
var verr *ValidationError
if errors.As(err, &verr) {
	fmt.Println("validation failed on field:", verr.Field)
}
```

### 10.4 Sentinel errors **[I]**

A **sentinel** is a predeclared, exported error value that callers compare against to detect a specific, expected condition — `io.EOF`, `sql.ErrNoRows`, `os.ErrNotExist`. Define them as package-level `var`s and check with `errors.Is`. Use sentinels sparingly: they couple callers to your exact value, so reserve them for conditions callers genuinely need to branch on.

```go
package store

import "errors"

var (
	ErrNotFound  = errors.New("store: not found")
	ErrDuplicate = errors.New("store: duplicate key")
)
// Caller: if errors.Is(err, store.ErrNotFound) { return 404 }
```

### 10.5 Custom error types **[I]**

When an error needs to carry **structured data** (a field name, an HTTP status, a retry-after), define a type that implements `error`. Callers recover it with `errors.As` and read its fields. Implement `Unwrap()` if your type also wraps an inner error, so it participates in `errors.Is`/`As` chains.

```go
type ValidationError struct {
	Field string
	Msg   string
	err   error // optional wrapped cause
}

func (e *ValidationError) Error() string {
	return fmt.Sprintf("validation: field %q: %s", e.Field, e.Msg)
}
func (e *ValidationError) Unwrap() error { return e.err } // participate in the chain

// Usage:
func validate(name string) error {
	if name == "" {
		return &ValidationError{Field: "name", Msg: "must not be empty"}
	}
	return nil
}
```

### 10.6 `panic` and `recover` — for the truly exceptional **[I/A]**

`panic` aborts normal flow and starts unwinding the stack, running deferred functions as it goes; if it reaches the top of a goroutine, the program crashes with a stack trace. `recover` (callable only inside a deferred function) stops a panic and returns the panic value, letting you regain control.

**The philosophy:** `panic` is **not** Go's error handling. Use it only for *programmer errors* and truly unrecoverable situations — a nil dereference, an impossible switch case, a failed must-succeed initialization (`regexp.MustCompile`, `template.Must`). For *expected* failures (file missing, bad input, network down) **return an error.** The one common place to `recover` is a **boundary** that must not crash the whole process — e.g. an HTTP server recovering per-request so one bad handler doesn't take down the server (the stdlib's `net/http` already does this; web frameworks add their own).

```go
func safeDivide(a, b int) (result int, err error) {
	defer func() {
		if r := recover(); r != nil {          // recover only works inside a defer
			err = fmt.Errorf("recovered: %v", r) // convert the panic into an error
		}
	}()
	return a / b, nil // a/0 panics -> recovered above, returned as err
}

// MustCompile-style: panic at init for programmer error, since there's no caller to handle it.
var emailRe = regexp.MustCompile(`^\S+@\S+$`) // panics if the regex is invalid (a bug)
```

> **Gotcha:** a panic in a goroutine that nothing recovers crashes the *entire program*, not just that goroutine. Recover inside the goroutine if it must be resilient. Also: `recover()` only does anything when called *directly* in a deferred function.

### 10.7 Error-handling best practices **[I]**

- **Handle each error once.** Either *handle* it (log, retry, default) **or** *return* it (wrapped) — not both. Logging and returning the same error produces duplicate, confusing logs.
- **Add context as you go up the stack** with `%w`, but don't repeat what the caller already knows. Aim for a readable chain: `"handle signup: create user: save: unique violation"`.
- **Check with `errors.Is`/`errors.As`**, never `==` (breaks under wrapping) or string matching (brittle).
- **Decide at the boundary** what an error means to the outside world (map domain errors → HTTP status codes / exit codes) — keep that mapping in one place.
- **Don't wrap when you want to hide internals** from callers (e.g. don't leak a DB driver error to an API client); translate it instead.

> **Best practice recap (§10):** errors are values — return and check them; wrap with `%w` to add context while keeping the cause inspectable; test with `errors.Is`/`errors.As`; use sentinels for expected conditions and custom types for structured data; reserve `panic`/`recover` for programmer errors and process-boundary safety nets; handle an error exactly once.

---

## 11. Generics

Generics (type parameters) arrived in **Go 1.18** after a decade of deliberation. They let you write functions and types that work over *many* types **while keeping full type safety** — no `any`, no reflection, no runtime type assertions. Go's generics are intentionally restrained: they solve the "I'm writing the same code for `int` and `string` and `float64`" problem without becoming a metaprogramming playground.

### 11.1 The problem generics solve **[I]**

Before generics, "works with any type" meant either copy-pasting per type, or using `any` and losing type safety (and paying for boxing/assertions). A `Max` function had to be written for each numeric type, or take `any` and assert. Generics give you one implementation that the compiler specializes, type-checked at compile time.

```go
// Before generics: either duplicate...
func MaxInt(a, b int) int { if a > b { return a }; return b }
func MaxFloat(a, b float64) float64 { if a > b { return a }; return b }
// ...or use any and lose safety + pay runtime costs.
```

### 11.2 Type parameters and constraints **[I/A]**

A **type parameter** is declared in square brackets after the function/type name: `func F[T any](x T)`. The thing in the brackets after the name (`T`) is the type variable; the part after it (`any`, `comparable`, or an interface) is the **constraint** — it limits which types are allowed *and* tells the compiler what operations are valid on values of that type.

- **`any`** — no constraint (any type). You can pass it around but not, say, compare or add it.
- **`comparable`** — types usable with `==`/`!=` (needed for map keys, set membership).
- **A custom constraint interface** — lists the types (a *type set*) or methods allowed. The `golang.org/x/exp/constraints` package (and patterns like `~int`) express things like "any integer type."

The `~` (tilde) in a constraint means "this type *or any type whose underlying type is this*", so your own `type Celsius float64` still satisfies a `~float64` constraint.

```go
import "cmp" // Go 1.21+: cmp.Ordered constraint + cmp.Compare/Less

// Generic Max over any ordered type (numbers, strings). cmp.Ordered is the
// stdlib constraint for "supports < <= >= >".
func Max[T cmp.Ordered](a, b T) T {
	if a > b {
		return a
	}
	return b
}

fmt.Println(Max(3, 7))         // 7      (T inferred as int)
fmt.Println(Max(2.5, 1.5))     // 2.5    (float64)
fmt.Println(Max("apple", "pear")) // pear (string)

// A custom constraint with a type set. ~ allows named types with these underlyings.
type Number interface {
	~int | ~int64 | ~float64
}
func Sum[T Number](xs []T) T {
	var total T // zero value of T
	for _, x := range xs {
		total += x // allowed: the constraint guarantees + works
	}
	return total
}
fmt.Println(Sum([]int{1, 2, 3}))       // 6
fmt.Println(Sum([]float64{1.5, 2.5}))  // 4
```

### 11.3 Generic types **[A]**

You can parameterize *types*, not just functions — the basis for type-safe containers (stacks, sets, linked lists, result wrappers). Methods on a generic type use the receiver's type parameters but cannot introduce new ones.

```go
// A type-safe stack of any element type.
type Stack[T any] struct {
	items []T
}

func (s *Stack[T]) Push(v T) { s.items = append(s.items, v) }
func (s *Stack[T]) Pop() (T, bool) {
	var zero T
	if len(s.items) == 0 {
		return zero, false // return the zero value of T when empty
	}
	last := s.items[len(s.items)-1]
	s.items = s.items[:len(s.items)-1]
	return last, true
}

func main() {
	var s Stack[string] // a stack specialized to string at the call site
	s.Push("a"); s.Push("b")
	v, ok := s.Pop()
	fmt.Println(v, ok) // b true
}
```

### 11.4 Type inference **[I]**

The compiler usually **infers** type arguments from the function arguments, so you rarely write them explicitly. When inference can't figure it out (e.g. the type parameter only appears in the return type), supply them: `MakeSlice[int](3)`.

```go
Max(3, 7)        // inferred: Max[int]
Max[float64](3, 7) // explicit when you want a specific type, or inference can't tell
```

### 11.5 Generics vs interfaces — which to use **[A]**

This is the key judgement call. They overlap but solve different problems:

- **Use an interface** when you need **runtime polymorphism / dynamic dispatch** — a heterogeneous collection (`[]Shape` holding circles and squares), or behaviour that varies per value at runtime. Interfaces are about *behaviour*.
- **Use generics** when you have **one algorithm that's identical across types** and you want to preserve the concrete type — containers, `Map`/`Filter`/`Reduce`, `Max`/`Min`/`Sum`. Generics are about *type-parametric code* with no boxing and full static checking.
- If a plain interface (like `io.Reader`) already expresses what you need, **use it** — don't reach for generics just because you can. Generics add cognitive weight; introduce them when duplication or `any`-loss of safety is the real pain.

### 11.6 Pitfalls **[A]**

- **Over-genericizing.** Most application code doesn't need generics. They earn their place in *libraries* and *utilities* (collections, the `slices`/`maps` packages), less so in business logic.
- **No method-level type parameters.** A method can't add its own type parameters beyond the receiver's — sometimes you must make it a package-level generic function instead.
- **Constraints limit operations.** Inside a generic function you can only do what the constraint guarantees. With `any` you can't even compare; you must widen the constraint (e.g. `comparable`) to unlock `==`.
- **Zero values need `var zero T`.** You can't write a literal for an unknown `T`; declare `var zero T` to get its zero value.

> **Best practice recap (§11):** reach for generics to remove genuine type-duplicated code with safety, especially in libraries; use the stdlib constraints (`cmp.Ordered`, `comparable`) and `~` for named types; let inference do the work; choose interfaces for runtime polymorphism and generics for type-parametric algorithms; don't over-genericize application code.

---

## 12. Concurrency: Goroutines, Channels, sync, context

Concurrency is Go's signature feature and the reason many teams choose it. This is a long section because the model rewards real understanding. **Concurrency is not parallelism:** concurrency is *structuring* a program as independent activities that *can* run out of order; parallelism is *actually* running them simultaneously on multiple cores. Go gives you cheap concurrency and lets the runtime use available cores for parallelism.

### 12.1 The philosophy: "share memory by communicating" **[A]**

Most languages do concurrency by sharing memory between threads and guarding it with locks — which is error-prone (races, deadlocks, forgotten unlocks). Go offers an alternative, summed up by Rob Pike's slogan: **"Do not communicate by sharing memory; instead, share memory by communicating."** Rather than many goroutines poking at the same variable under a lock, you give *ownership* of data to one goroutine and pass it to others through **channels**. The data has a clear owner at every moment, which sidesteps most races by design. Locks (`sync`) still exist and are the right tool for protecting small pieces of shared state (a counter, a cache), but channels are the idiomatic way to coordinate *workflow*.

### 12.2 Goroutines — cheap, runtime-scheduled concurrency **[A]**

A **goroutine** is a function running concurrently, launched with the `go` keyword. Goroutines are *not* OS threads — they are lightweight (a few KB of stack that grows as needed) and multiplexed by the Go runtime onto a small pool of OS threads (the M:N scheduler). You can run hundreds of thousands of them; spawning one is cheap. This is why Go can handle a goroutine *per connection* in a server, a model that would melt under OS threads.

```go
func say(s string) { fmt.Println(s) }

func main() {
	go say("async")   // launches a goroutine; main does NOT wait for it
	say("sync")       // runs in the main goroutine

	// PROBLEM: main might exit before the goroutine runs at all.
	// When main() returns, the program exits and kills all goroutines.
	time.Sleep(10 * time.Millisecond) // crude fix for demo ONLY — use sync/channels in real code
}
```

Two foundational facts:

1. **When `main` returns, the program exits**, abandoning any still-running goroutines. You must *synchronize* (WaitGroup, channel, errgroup) so main waits for the work it cares about.
2. **Goroutines do not return values.** A goroutine's function result is discarded. To get results out, send them on a channel or write to shared state under a lock.

### 12.3 Channels — typed pipes between goroutines **[A]**

A **channel** carries values of one type between goroutines, with the send (`ch <- v`) and receive (`v := <-ch`) operations synchronizing the two sides. Channels come in two flavours:

- **Unbuffered** (`make(chan T)`): a send blocks until another goroutine receives, and vice versa. This is a *synchronization point* — the two goroutines "rendezvous". Use it when you want a guarantee that the value was handed off.
- **Buffered** (`make(chan T, n)`): holds up to `n` values; a send blocks only when the buffer is full, a receive blocks only when it's empty. Use it to decouple producer/consumer speeds or limit in-flight work (a semaphore).

```go
ch := make(chan int)      // unbuffered
go func() { ch <- 42 }()  // send blocks until main receives
v := <-ch                 // receive -> v == 42 (this is the rendezvous)

buf := make(chan string, 2) // buffered, capacity 2
buf <- "a"                  // doesn't block (room in buffer)
buf <- "b"                  // doesn't block
// buf <- "c"               // WOULD block: buffer full, no receiver yet
fmt.Println(<-buf, <-buf)   // a b
```

**Closing, ranging, and directions:**

- **`close(ch)`** signals "no more values will be sent." Receivers can keep draining buffered values; once empty, receives return the zero value with `ok=false`.
- **`v, ok := <-ch`** — `ok` is false when the channel is closed and drained. This is how you detect closure.
- **`for v := range ch`** receives until the channel is closed — the idiomatic consumer loop.
- **Direction-typed channels** document intent and are enforced by the compiler: `chan<- T` is send-only, `<-chan T` is receive-only. Pass these in function signatures to prevent a producer from accidentally receiving, etc.

```go
func produce(out chan<- int) { // send-only param
	for i := 0; i < 3; i++ {
		out <- i
	}
	close(out) // the SENDER closes — never the receiver
}

func main() {
	ch := make(chan int)
	go produce(ch)
	for v := range ch { // ranges until ch is closed
		fmt.Println(v)  // 0 1 2
	}
}
```

> **Channel rules that prevent crashes:** sending on a **closed** channel **panics**; closing a **closed** channel **panics**; closing a **nil** channel panics. Therefore: **only the sender closes**, and only once. **Receiving from a closed channel is safe** (returns zero, `ok=false`). Operations on a **nil** channel **block forever** — occasionally useful in `select` to disable a case.

### 12.4 `select` — waiting on multiple channels **[A]**

`select` blocks until one of several channel operations can proceed, then runs that case. If several are ready it picks one at random (fairness). It is how you build timeouts, cancellation, multiplexing, and non-blocking sends/receives.

```go
select {
case v := <-ch1:
	fmt.Println("from ch1:", v)
case ch2 <- 99:
	fmt.Println("sent to ch2")
case <-time.After(time.Second): // timeout: time.After returns a channel that fires once
	fmt.Println("timed out")
default:
	fmt.Println("nothing ready right now") // default makes select NON-BLOCKING
}
```

- **`time.After`** gives you a one-shot timeout channel. (For repeated timeouts in a loop, prefer a `time.Timer` you can `Reset`, or `context` with a deadline, to avoid leaking timers.)
- **`default`** turns a `select` into a non-blocking poll — runs immediately if nothing else is ready.
- A **nil channel** case is never selected; setting a channel variable to `nil` is a clean way to disable a branch.

### 12.5 The `sync` package — locks and coordination **[A]**

When channels are overkill (you just need to protect a shared variable), use `sync`:

- **`sync.Mutex`** — mutual exclusion: `Lock()`/`Unlock()` around a critical section so only one goroutine touches the shared state at a time. The zero value is an unlocked mutex (useful zero value!). **Never copy a mutex** after first use.
- **`sync.RWMutex`** — allows many concurrent readers *or* one writer. Use when reads vastly outnumber writes.
- **`sync.WaitGroup`** — wait for a set of goroutines to finish: `Add(n)`, each goroutine `defer wg.Done()`, then `wg.Wait()`.
- **`sync.Once`** — run an initializer exactly once even under concurrency (lazy singletons, one-time setup).
- **`sync.Pool`** — a free-list of reusable temporary objects to reduce allocation/GC pressure in hot paths (e.g. buffers). Items can vanish at any GC, so only use for things you can recreate.

```go
// Mutex protecting a counter:
type SafeCounter struct {
	mu sync.Mutex
	n  int
}
func (c *SafeCounter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock() // defer guarantees unlock even on panic/early return
	c.n++
}

// WaitGroup to wait for N goroutines:
func main() {
	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		wg.Add(1)              // increment BEFORE launching (avoid a race with Wait)
		go func() {
			defer wg.Done()     // signals this goroutine is finished
			fmt.Println("working", i) // i is per-iteration safe on Go 1.22+
		}()
	}
	wg.Wait() // blocks until all 5 have called Done()
}

// Once for one-time init:
var once sync.Once
var conn *DB
func GetDB() *DB {
	once.Do(func() { conn = connect() }) // runs the func exactly once, even concurrently
	return conn
}
```

### 12.6 `sync/atomic` — lock-free primitives **[A]**

For single-value counters/flags, atomic operations are faster than a mutex. Go 1.19+ provides typed atomics (`atomic.Int64`, `atomic.Bool`, `atomic.Pointer[T]`) that are harder to misuse than the older free functions.

```go
var ops atomic.Int64
// In many goroutines:
ops.Add(1)            // atomic increment, no lock
total := ops.Load()   // atomic read
_ = total
```

> Use atomics only for simple independent values. The moment you need to keep *two* fields consistent together, use a mutex — atomics don't compose into a multi-field invariant.

### 12.7 Data races and `-race` **[A, critical]**

A **data race** occurs when two goroutines access the same memory concurrently and at least one is a write, with no synchronization. Races are undefined behaviour: corrupted values, torn reads, crashes — and they're often invisible until production. Go ships a **race detector**: run your tests and programs with **`-race`** and it instruments memory accesses to catch races at runtime.

```bash
go test -race ./...   # run this in CI — non-negotiable for concurrent code
go run -race .
go build -race -o app.exe .
```

```go
// RACY: two goroutines write the same map with no lock -> -race will flag it (and it can crash).
m := map[int]int{}
go func() { m[1] = 1 }()
go func() { m[2] = 2 }() // concurrent map write -> data race / possible panic
// Fix: guard with sync.Mutex, use sync.Map, or confine the map to one goroutine + channels.
```

### 12.8 `context` — cancellation, deadlines, and values **[A, essential]**

`context.Context` is how Go propagates **cancellation, deadlines, and request-scoped values** across API boundaries and goroutines. A long-running operation (HTTP request, DB query, RPC) takes a `ctx` and watches `ctx.Done()` (a channel that closes when the context is cancelled or times out). When the caller cancels, every downstream goroutine sees it and can stop — this is how you avoid wasting work and leaking goroutines after a client disconnects.

```go
// Cancellation: derive a child context with a cancel function.
ctx, cancel := context.WithCancel(context.Background())
defer cancel() // always call cancel to release resources, even on the happy path

// Deadline/timeout: auto-cancel after a duration.
ctx, cancel = context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

// A worker that respects cancellation:
func worker(ctx context.Context, jobs <-chan int) {
	for {
		select {
		case <-ctx.Done(): // cancelled or timed out
			fmt.Println("stopping:", ctx.Err()) // context.Canceled / DeadlineExceeded
			return
		case j, ok := <-jobs:
			if !ok {
				return
			}
			fmt.Println("processing", j)
		}
	}
}
```

**Context best practices (these are widely enforced):**

- **Pass `ctx` as the *first* parameter**, named `ctx`: `func Do(ctx context.Context, ...)`. **Never store a Context in a struct** — pass it through the call chain.
- **`context.Background()`** is the root (in `main`, init, tests); **`context.TODO()`** is a placeholder when you haven't wired context through yet.
- **Always call the `cancel` function** (usually via `defer`) — not doing so leaks the context's resources/timer.
- **`context.WithValue`** carries request-scoped data (a request ID, an auth subject) — use it sparingly for *cross-cutting* values, **never** for passing normal function parameters, and use a private key type to avoid collisions.

### 12.9 Common concurrency patterns **[A]**

**Worker pool** — a fixed number of workers pull from a jobs channel, bounding concurrency (so you don't spawn a million goroutines):

```go
func workerPool(jobs <-chan int, results chan<- int, workers int) {
	var wg sync.WaitGroup
	for w := 0; w < workers; w++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := range jobs { // each worker pulls until jobs is closed
				results <- j * j  // do the work, send the result
			}
		}()
	}
	go func() { wg.Wait(); close(results) }() // close results once all workers finish
}
```

**Fan-out / fan-in** — fan-out: start several goroutines reading from one channel (the worker pool above is fan-out). Fan-in: merge several result channels into one:

```go
func fanIn(chans ...<-chan int) <-chan int {
	out := make(chan int)
	var wg sync.WaitGroup
	for _, c := range chans {
		wg.Add(1)
		go func(c <-chan int) {
			defer wg.Done()
			for v := range c {
				out <- v
			}
		}(c)
	}
	go func() { wg.Wait(); close(out) }()
	return out
}
```

**Pipeline** — stages connected by channels, each stage a goroutine transforming a stream:

```go
func gen(nums ...int) <-chan int {
	out := make(chan int)
	go func() { defer close(out); for _, n := range nums { out <- n } }()
	return out
}
func square(in <-chan int) <-chan int {
	out := make(chan int)
	go func() { defer close(out); for n := range in { out <- n * n } }()
	return out
}
// Usage: for v := range square(gen(1,2,3)) { ... } -> 1 4 9
```

**Rate limiting** — a `time.Ticker` paces work:

```go
limiter := time.NewTicker(200 * time.Millisecond) // at most 5 ops/sec
defer limiter.Stop()
for req := range requests {
	<-limiter.C // wait for the next tick before handling
	handle(req)
}
```

**`errgroup`** — `golang.org/x/sync/errgroup` runs a group of goroutines, returns the **first** error, and (with `WithContext`) cancels the rest. It's the modern, ergonomic way to run concurrent subtasks that can fail:

```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, urls []string) error {
	g, ctx := errgroup.WithContext(ctx) // cancels ctx on the first error
	for _, u := range urls {
		u := u // pre-1.22 capture safety; harmless on 1.22+
		g.Go(func() error {
			return fetch(ctx, u) // returning an error cancels siblings
		})
	}
	return g.Wait() // waits for all; returns the first non-nil error
}
```

### 12.10 Avoiding goroutine leaks **[A]**

A goroutine that blocks forever (on a channel that never receives/closes, or a `select` with no exit) **leaks** — it lives until the program ends, holding memory and resources. The discipline: **every goroutine must have a guaranteed way to exit** — a closed channel, a `ctx.Done()` case, or a finite loop. When you launch a goroutine, immediately answer "what makes this return?"

> **Best practice recap (§12):** prefer channels for coordinating workflow, mutexes/atomics for guarding small shared state; only the sender closes a channel (once); use `select` for timeouts/cancellation; thread `context` as the first arg and always `cancel`; never copy a `Mutex`/`WaitGroup`; bound concurrency with worker pools; use `errgroup` for fail-fast concurrent subtasks; guarantee every goroutine can exit; and **always run `go test -race`**.

---

## 13. The Standard Library Tour

Go's standard library is large, high-quality, and stable — reading its source is the best way to learn idiomatic Go. This is a guided tour of the packages you'll use constantly. (HTTP *servers*, ORMs, JWT, websockets, gRPC each have their own dedicated guide in this library; here we cover language-adjacent stdlib essentials and the HTTP *client*.)

### 13.1 `fmt` — formatted I/O **[B]**

Covered in §4.5 (verbs). The functions you'll reach for: `Println`/`Printf` (stdout), `Sprintf` (build a string), `Fprintf` (write to any `io.Writer`), `Errorf` (build/wrap errors), and `Sscanf`/`Scan` (parse). Implement `Stringer` to control how your types print.

### 13.2 `strings` and `bytes` **[B]**

`strings` operates on immutable `string`; `bytes` mirrors it for mutable `[]byte`. They have near-identical APIs (`Contains`, `Split`, `Join`, `HasPrefix`, `Index`, `Replace`, `TrimSpace`, `Builder`/`Buffer`). Use `bytes` when you're already working with byte slices (network/file data) to avoid converting to string and back (which allocates).

```go
s := "  Hello, World  "
fmt.Println(strings.TrimSpace(s))            // "Hello, World"
fmt.Println(strings.Split("a,b,c", ","))     // [a b c]
fmt.Println(strings.ReplaceAll("aaa", "a", "b")) // "bbb"

var buf bytes.Buffer            // mutable byte buffer, implements io.Writer & io.Reader
buf.WriteString("hello ")
fmt.Fprintf(&buf, "world %d", 1)
fmt.Println(buf.String())       // "hello world 1"
```

### 13.3 `strconv` — string ↔ number **[B]**

Covered in §4.4: `Atoi`/`Itoa`, `ParseInt`/`ParseFloat`/`ParseBool`, `FormatInt`/`FormatFloat`, `Quote`. Use it (not `fmt`) when conversion can fail and you need the error.

### 13.4 `time` — durations, instants, formatting **[B/I]**

`time` handles points in time (`time.Time`), spans (`time.Duration`), timers/tickers, and the famously unusual formatting (the reference layout is **`Mon Jan 2 15:04:05 MST 2006`** — i.e. `01/02 03:04:05PM '06 -0700`, the numbers 1-2-3-4-5-6-7). You write the layout *by example* rather than with `%`-codes.

```go
now := time.Now()
fmt.Println(now.Format("2006-01-02 15:04:05"))   // format BY EXAMPLE using the magic date
fmt.Println(now.Format(time.RFC3339))            // built-in layout constant

d := 90 * time.Minute
fmt.Println(d.Hours())                            // 1.5

deadline := now.Add(24 * time.Hour)               // arithmetic with durations
fmt.Println(deadline.Sub(now))                    // 24h0m0s
fmt.Println(now.Before(deadline))                 // true

t, err := time.Parse("2006-01-02", "2026-06-23")  // parse with the same layout idiom
_ = t; _ = err

// Always be explicit about UTC vs local in storage; store UTC, display local.
fmt.Println(now.UTC())
```

> **Gotcha:** don't compare times with `==` (it compares wall clock + monotonic + location); use `Equal`. Use `time.Since(start)` for elapsed time.

### 13.5 `encoding/json` — marshal/unmarshal **[B/I]**

`json.Marshal` turns Go values into JSON; `json.Unmarshal` parses JSON into Go values. **Only exported fields** are serialized; **struct tags** control field names and options (`omitempty`, `-`, `,string`). For streaming, use `json.NewEncoder(w)`/`json.NewDecoder(r)` which read/write directly to an `io.Writer`/`io.Reader` (better for HTTP bodies and large data).

```go
type User struct {
	ID    int    `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email,omitempty"` // omitted when empty
	Token string `json:"-"`                // never serialized (sensitive)
}

u := User{ID: 1, Name: "Ada"}
b, _ := json.Marshal(u)
fmt.Println(string(b))           // {"id":1,"name":"Ada"}

b2, _ := json.MarshalIndent(u, "", "  ") // pretty-printed
_ = b2

var parsed User
_ = json.Unmarshal([]byte(`{"id":2,"name":"Bob"}`), &parsed) // pass a POINTER to fill it
fmt.Println(parsed.Name)         // Bob

// Streaming (idiomatic for HTTP): decode straight from a reader.
// json.NewDecoder(req.Body).Decode(&parsed)
```

> **Security note:** untrusted JSON can be abused (deeply nested input, huge numbers). Bound the body size (`http.MaxBytesReader`), and use `Decoder.DisallowUnknownFields()` when you want to reject unexpected fields. To distinguish "field absent" from "field present and zero," use pointer fields (`*int`).

### 13.6 `sort`, `slices`, `maps` **[I]**

⚡ Since **Go 1.21** the generic **`slices`** and **`maps`** packages largely supersede the older reflection-based `sort` helpers — they're type-safe and faster. Know both, prefer `slices`/`maps`.

```go
// slices (1.21+): generic, type-safe operations over any slice.
xs := []int{3, 1, 2}
slices.Sort(xs)                              // [1 2 3]
slices.SortFunc(xs, func(a, b int) int { return b - a }) // custom comparator -> [3 2 1]
fmt.Println(slices.Contains(xs, 2))          // true
fmt.Println(slices.Index(xs, 2))             // position or -1
fmt.Println(slices.Max(xs), slices.Min(xs))  // 3 1
ys := slices.Clone(xs)                        // independent copy (avoids aliasing, §6)
slices.Reverse(ys)
fmt.Println(slices.Equal(xs, ys))            // element-wise equality

// maps (1.21+): keys/values/clone/equal.
m := map[string]int{"a": 1, "b": 2}
keys := slices.Sorted(maps.Keys(m))          // 1.23: maps.Keys returns an iterator
fmt.Println(keys)                            // [a b]

// Old sort still works and is needed pre-1.21 or for sort.Interface:
names := []string{"c", "a", "b"}
sort.Strings(names)                          // [a b c]
```

> **⚡ Version note (iterators, 1.23):** `maps.Keys`/`maps.Values` now return `iter.Seq` iterators (range-over-func), so you pair them with `slices.Sorted`/`slices.Collect`. On 1.21/1.22 the signatures differed — check `go doc maps`.

### 13.7 `regexp` — regular expressions **[I]**

Go's `regexp` uses RE2 syntax, which guarantees linear-time matching (no catastrophic backtracking — a security plus). **Compile once** (e.g. a package-level `var` with `MustCompile`) and reuse; compiling in a hot loop is wasteful.

```go
var emailRe = regexp.MustCompile(`^[\w.+-]+@[\w-]+\.[\w.-]+$`) // compiled once at init

fmt.Println(emailRe.MatchString("a@b.com"))   // true
re := regexp.MustCompile(`(\w+)=(\d+)`)
m := re.FindStringSubmatch("count=42")        // ["count=42" "count" "42"]
fmt.Println(m[1], m[2])                        // count 42
fmt.Println(re.ReplaceAllString("x=1 y=2", "$1:$2")) // x:1 y:2
```

### 13.8 `io` and `bufio` — streaming **[I]**

`io` defines the universal `Reader`/`Writer`/`Closer` interfaces (§9.9) and helpers (`io.Copy`, `io.ReadAll`, `io.Discard`, `io.MultiWriter`). `bufio` wraps a reader/writer with a buffer to reduce syscalls and adds conveniences like `Scanner` (read line by line) and `Writer.Flush`.

```go
// Copy any reader to any writer without loading it all into memory:
n, _ := io.Copy(os.Stdout, strings.NewReader("stream me\n")) // streams through a small buffer
_ = n

// bufio.Scanner: read input line by line (great for files/stdin).
sc := bufio.NewScanner(strings.NewReader("line1\nline2\nline3"))
for sc.Scan() {            // advances to the next token (default: lines)
	fmt.Println(sc.Text()) // the current line, without the newline
}
// Check sc.Err() after the loop. For very long lines, raise sc.Buffer(...).
```

### 13.9 `net/http` — the client side **[I]**

This guide covers the **client**; for building servers/routers see `GO_NET_HTTP_REST_API_GUIDE.md` and the Gin guide. The key client lesson: **don't use `http.Get`/the `http.DefaultClient` for anything serious** — it has *no timeout*, so a hung server hangs your program forever. Always construct an `http.Client` with a timeout, always `defer resp.Body.Close()`, and always pass a `context` so the request is cancellable.

```go
func fetch(ctx context.Context, url string) ([]byte, error) {
	client := &http.Client{Timeout: 10 * time.Second} // ALWAYS set a timeout

	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil) // ctx-aware
	if err != nil {
		return nil, err
	}
	req.Header.Set("Accept", "application/json")

	resp, err := client.Do(req)
	if err != nil {
		return nil, fmt.Errorf("fetch %s: %w", url, err)
	}
	defer resp.Body.Close() // ALWAYS close the body, or you leak connections

	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("fetch %s: status %d", url, resp.StatusCode)
	}
	// Bound the read so a malicious/huge response can't exhaust memory:
	return io.ReadAll(io.LimitReader(resp.Body, 10<<20)) // cap at 10 MiB
}
```

### 13.10 `log/slog` — structured logging **[I, modern]**

⚡ Since **Go 1.21** the stdlib has **`log/slog`** for **structured** logging (key/value attributes, levels, JSON or text output) — this is the modern standard, replacing the old unstructured `log` package for production. Structured logs are machine-parseable (great for log aggregation) and carry context (request IDs, user IDs) as fields rather than baked into a message string.

```go
// JSON handler at info level, writing to stderr.
logger := slog.New(slog.NewJSONHandler(os.Stderr, &slog.HandlerOptions{Level: slog.LevelInfo}))
slog.SetDefault(logger) // optional: make it the package-level default

slog.Info("user login",
	"user_id", 42,            // structured key/value attributes
	"ip", "10.0.0.1",
)
// {"time":"...","level":"INFO","msg":"user login","user_id":42,"ip":"10.0.0.1"}

slog.Error("db query failed", "err", err, "query", "SELECT ...")

// Attach context that rides along on every log line from this logger:
reqLog := logger.With("request_id", "abc-123")
reqLog.Info("handling request") // includes request_id automatically
```

> **Security note:** never log secrets, passwords, tokens, full card numbers, or PII. Redact sensitive fields before logging. Prefer levels (`Debug`/`Info`/`Warn`/`Error`) and configure verbosity by environment.

### 13.11 `math`, `math/rand`, `math/rand/v2` **[B/I]**

`math` has constants (`math.Pi`, `math.MaxInt64`) and functions (`Sqrt`, `Floor`, `Abs`, `Pow`). For randomness, ⚡ Go 1.22 introduced **`math/rand/v2`** (cleaner API, auto-seeded — no more `rand.Seed`). **Crucial security rule: `math/rand` is NOT cryptographically secure.** For tokens, passwords, salts, session IDs, or anything an attacker shouldn't predict, use **`crypto/rand`** (§18).

```go
fmt.Println(math.Sqrt(16), math.Max(3, 7), math.Abs(-5)) // 4 7 5

// Non-security randomness (games, jitter, sampling): math/rand/v2 (Go 1.22+), auto-seeded.
fmt.Println(rand.IntN(100)) // 0..99   (import "math/rand/v2")
fmt.Println(rand.Float64()) // 0.0..1.0

// SECURITY-sensitive randomness: crypto/rand (see §18).
```

> **Best practice recap (§13):** lean on the stdlib before reaching for dependencies; set timeouts and close bodies on every HTTP client call; bound reads of untrusted input; prefer `slices`/`maps` over reflection-based `sort`; compile regexes once; use `slog` for structured logs (and never log secrets); use `crypto/rand` (never `math/rand`) for anything security-sensitive.

---

## 14. File System / OS / exec (Concise)

> **This is a summary.** For the full treatment — directory walking, `io/fs`, file modes/permissions, atomic writes, temp files, watching, cross-platform path handling, signals, and building robust CLIs — see **`GO_FILESYSTEM_OS_CLI_GUIDE.md`** in this library. Here we cover just enough to be self-contained.

### 14.1 Reading and writing whole files **[I]**

For small files, the one-shot helpers are simplest. They handle opening and closing for you.

```go
// Read an entire file into memory:
data, err := os.ReadFile("config.json") // returns []byte
if err != nil {
	return err
}

// Write an entire file (creates or truncates). 0644 = owner rw, others r (ignored on Windows).
err = os.WriteFile("out.txt", []byte("hello"), 0644)
```

For large files or streaming, open a handle and use `bufio`/`io.Copy` instead of loading everything into RAM:

```go
f, err := os.Open("big.log") // read-only handle
if err != nil {
	return err
}
defer f.Close()
sc := bufio.NewScanner(f)
for sc.Scan() { /* process sc.Text() line by line */ }
```

### 14.2 Paths — use `filepath` for cross-platform correctness **[I]**

`path/filepath` joins and manipulates paths using the **OS-appropriate separator** (`\` on Windows, `/` elsewhere). Use it for filesystem paths; use `path` (always `/`) only for URLs and slash-separated things like embedded FS keys.

```go
p := filepath.Join("data", "users", "ada.json") // data\users\ada.json on Windows
fmt.Println(filepath.Ext(p))                     // .json
fmt.Println(filepath.Base(p))                    // ada.json
fmt.Println(filepath.Dir(p))                     // data\users
```

### 14.3 Environment variables and program arguments **[B]**

```go
// Env vars:
home := os.Getenv("HOME")                 // "" if unset
val, ok := os.LookupEnv("API_KEY")        // ok distinguishes unset from empty
os.Setenv("DEBUG", "1")

// Command-line args: os.Args[0] is the program path; [1:] are the real args.
fmt.Println(os.Args)        // [./app foo bar]
// For real CLIs use the `flag` package (or cobra) — see the FS/OS/CLI guide.
```

### 14.4 Running external commands with `os/exec` **[I, security-sensitive]**

`os/exec` runs other programs. The **single most important rule** is on the next line and again in §18: **build the command from an explicit argument slice — never by concatenating strings into a shell** — to avoid command injection.

```go
// SAFE: args are passed as a slice; no shell parses/expands them.
cmd := exec.CommandContext(ctx, "git", "log", "--oneline", "-n", "5")
out, err := cmd.Output() // runs it, captures stdout
if err != nil {
	return err
}
fmt.Println(string(out))

// DANGEROUS — DO NOT DO THIS with untrusted input:
//   exec.Command("sh", "-c", "git log " + userInput)  // userInput="; rm -rf /" => disaster
```

Use `CombinedOutput()` for stdout+stderr, `cmd.Stdin`/`Stdout`/`Stderr` to wire pipes, and `CommandContext` so a timeout/cancellation kills the child process.

### 14.5 Exit codes and signals **[I]**

`os.Exit(code)` ends the program immediately with a status code — **but it does *not* run deferred functions**, so prefer returning an error up to `main` and exiting there. Catch OS signals (Ctrl+C, SIGTERM) with `signal.NotifyContext` for graceful shutdown:

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()
// <-ctx.Done() fires on Ctrl+C / SIGTERM -> begin graceful shutdown.
```

> **Best practice recap (§14):** use `os.ReadFile`/`WriteFile` for small files and streaming for large ones; always `defer Close()`; use `filepath` for OS paths; read config from env (with `LookupEnv` to detect unset); **never** build shell commands from untrusted strings — pass an arg slice to `exec.Command`; use `signal.NotifyContext` for graceful shutdown; and see the FS/OS/CLI guide for the deep dive.

---

## 15. Testing, Benchmarking, Fuzzing

Testing is built into the language and the toolchain — no framework required. The convention is simple and the tooling is excellent, which is why Go codebases tend to be well-tested.

### 15.1 The `testing` package and conventions **[I]**

Tests live in files ending in **`_test.go`**, in the same package as the code (or `package foo_test` for black-box tests). A test is a function `func TestXxx(t *testing.T)`. You signal failure with `t.Error`/`t.Errorf` (continue) or `t.Fatal`/`t.Fatalf` (stop this test). Go has **no built-in assertion library** by design — you write plain `if` checks, which keeps tests explicit and dependency-free.

```go
// file: math.go
package mathx
func Add(a, b int) int { return a + b }

// file: math_test.go
package mathx
import "testing"

func TestAdd(t *testing.T) {
	got := Add(2, 3)
	want := 5
	if got != want {
		t.Errorf("Add(2,3) = %d; want %d", got, want) // Errorf: report and continue
	}
}
```

```bash
go test ./...            # run all tests
go test -v ./...         # verbose: show each test
go test -run TestAdd     # run tests matching a regex
go test -race ./...      # with the race detector (do this for concurrent code)
go test -cover ./...     # report coverage percentage
go test -coverprofile=cover.out ./... && go tool cover -html=cover.out # visual coverage
```

### 15.2 Table-driven tests and subtests **[I, the idiom]**

The dominant Go testing style is **table-driven**: define a slice of test cases (input + expected output) and loop over them, running each as a **subtest** with `t.Run`. This makes adding a case trivial, isolates failures (each subtest reports independently), and lets you run one case by name.

```go
func TestAdd_Table(t *testing.T) {
	tests := []struct {
		name string // a descriptive subtest name
		a, b int
		want int
	}{
		{"positives", 2, 3, 5},
		{"with zero", 0, 7, 7},
		{"negatives", -1, -1, -2},
	}

	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) { // subtest: isolated, named, individually runnable
			if got := Add(tc.a, tc.b); got != tc.want {
				t.Errorf("Add(%d,%d) = %d; want %d", tc.a, tc.b, got, tc.want)
			}
		})
	}
}
// Run one case: go test -run TestAdd_Table/negatives
```

Helpers: `t.Helper()` (marks a function as a helper so failures report the *caller's* line), `t.Cleanup(fn)` (register teardown, runs at test end — cleaner than defer for shared setup), `t.Parallel()` (run subtests concurrently), `t.TempDir()` (auto-cleaned temp directory).

### 15.3 Benchmarks **[I/A]**

Benchmarks measure performance. A `func BenchmarkXxx(b *testing.B)` runs its body `b.N` times (the framework picks `N` to get a stable measurement). Run with `go test -bench`.

```go
func BenchmarkAdd(b *testing.B) {
	for i := 0; i < b.N; i++ { // b.N is chosen by the framework
		Add(2, 3)
	}
}
// go test -bench=. -benchmem   # -benchmem also reports allocations/op
// Output: BenchmarkAdd-8   1000000000   0.30 ns/op   0 B/op   0 allocs/op
```

> Use `b.ReportAllocs()`/`-benchmem` to track allocations (often the real perf lever, §17). Use `b.ResetTimer()` after expensive setup so it isn't counted. Compare runs with `benchstat`.

### 15.4 Fuzzing **[A]**

⚡ **Native fuzzing** (Go 1.18+) automatically generates inputs to find crashes and edge cases — invaluable for parsers, decoders, and anything handling untrusted input. A `func FuzzXxx(f *testing.F)` seeds a corpus with `f.Add` and runs `f.Fuzz` with generated inputs; the property you assert should hold for *all* inputs.

```go
func FuzzReverse(f *testing.F) {
	f.Add("hello")          // seed corpus
	f.Add("héllo")
	f.Fuzz(func(t *testing.T, s string) {
		r := Reverse(s)
		if Reverse(r) != s { // property: reversing twice returns the original
			t.Errorf("double reverse changed %q -> %q", s, Reverse(r))
		}
	})
}
// go test -fuzz=FuzzReverse   (runs until it finds a failing input or you stop it)
```

### 15.5 Examples (testable docs) **[I]**

An `func ExampleXxx()` with an `// Output:` comment is **both documentation and a test**: `go test` runs it and checks the output matches, and `go doc`/pkg.go.dev shows it as example code. This keeps docs from rotting.

```go
func ExampleAdd() {
	fmt.Println(Add(1, 2))
	// Output: 3
}
```

### 15.6 Test doubles and `testify` **[I]**

Because interfaces are implicit (§9), **fakes/mocks are trivial**: define the interface where it's consumed, and in tests pass a small struct that satisfies it. You rarely need a mocking framework. For terser assertions and mocks many teams add **`testify`** (`assert`/`require`/`mock`) — it's the most common third-party testing dependency, though plain `if` checks remain perfectly idiomatic.

```go
// Production code depends on a small interface:
type Clock interface{ Now() time.Time }

// Test passes a fake — no framework needed:
type fakeClock struct{ t time.Time }
func (f fakeClock) Now() time.Time { return f.t }
// useService(fakeClock{t: someFixedTime})  -> deterministic tests
```

> **Best practice recap (§15):** write table-driven tests with `t.Run` subtests; use `t.Helper`/`t.Cleanup`/`t.TempDir`; run `-race` and track `-cover`; benchmark with `-benchmem` and optimize allocations; fuzz anything parsing untrusted input; write `Example` functions as living docs; exploit implicit interfaces to fake dependencies instead of reaching for heavy mocking tools.

---

## 16. Modules, Tooling & Project Layout

### 16.1 The `go mod` workflow recap **[I]**

§2.5 introduced modules; the day-to-day workflow: `go mod init` once; `go get` to add/upgrade deps; `go mod tidy` to reconcile; commit `go.mod`+`go.sum`. Add **semantic import versioning** awareness (v2+ in the path) and you have the whole model.

```bash
go list -m all              # list every module in the build graph
go mod why github.com/x/y   # explain why a dependency is needed
go mod graph                # the full dependency graph (for debugging conflicts)
go get -u=patch ./...       # upgrade only patch versions (safer)
```

### 16.2 Workspaces — `go.work` **[I/A]**

⚡ **Workspaces** (Go 1.18+) let you work on **several local modules together** without editing each `go.mod` with `replace` directives. A `go.work` file at a parent directory lists the modules; builds use your local copies. This is ideal when you're developing a library and an app that uses it side by side.

```bash
go work init ./api ./shared   # create go.work referencing two local modules
go work use ./newmodule       # add another local module to the workspace
# Now edits in ./shared are immediately seen by ./api with no replace directives.
```

> Commit `go.work` only for multi-module repos that everyone shares; for personal local-dev setups, many keep it out of version control (add to `.gitignore`).

### 16.3 Build tags / build constraints **[A]**

**Build tags** conditionally include/exclude files from a build — for OS-specific code, optional features, or integration tests. The modern syntax is a `//go:build` line at the very top of the file (before `package`). Files can also target an OS/arch by *filename* suffix (`foo_windows.go`, `bar_linux.go`).

```go
//go:build linux || darwin
// (blank line required after the build-constraint block)

package platform
// ... this file is only compiled on Linux or macOS ...
```

```bash
go test -tags=integration ./...   # include files marked //go:build integration
```

### 16.4 Cross-compilation — `GOOS` / `GOARCH` **[I]**

One of Go's superpowers: produce a binary for any platform from any platform, with no extra toolchain, by setting two environment variables. This is why building a Linux container image from a Windows dev machine is trivial.

```bash
# PowerShell (Windows) -> build a Linux amd64 binary:
$env:GOOS="linux"; $env:GOARCH="amd64"; go build -o app .
# bash / git-bash:
GOOS=linux GOARCH=amd64 go build -o app .
# See all supported pairs:
go tool dist list
```

> Pure-Go code cross-compiles freely. Code using **cgo** (C interop) does *not* cross-compile without a C cross-toolchain — another reason to avoid cgo when you can (`CGO_ENABLED=0` for fully static binaries).

### 16.5 `go generate` **[I]**

`go generate` runs commands specified in `//go:generate` comments — a convention for code generation (stringers, mocks, protobuf, ent schemas). It is *not* run by `go build`; you run it explicitly and commit the generated output.

```go
//go:generate stringer -type=Weekday
// Running `go generate ./...` invokes the stringer tool to generate Weekday.String().
```

### 16.6 Embedding files — `//go:embed` **[I]**

⚡ The `embed` package (Go 1.16+) bakes files into the binary at compile time, so your static assets (HTML templates, migrations, config, a web UI) ship inside the single executable — no separate files to deploy. This pairs perfectly with Go's "one static binary" story.

```go
import _ "embed"

//go:embed banner.txt
var banner string          // file contents as a string

//go:embed templates/*.html
var templatesFS embed.FS   // a whole directory tree as a read-only fs.FS

func main() {
	fmt.Println(banner)
	// template.ParseFS(templatesFS, "templates/*.html") ...
}
```

### 16.7 Standard project layout **[I/A]**

Go is unopinionated about directory structure, but conventions have emerged:

- **`cmd/<appname>/main.go`** — the entry point(s); keep `main` thin (wire dependencies, then call into packages).
- **`internal/`** — packages here are importable **only by code within the same module** (the compiler enforces it). Put implementation details here so other modules can't depend on them.
- **`pkg/`** — (optional, somewhat contested) library code intended for external reuse.
- **`api/`, `web/`, `configs/`, `migrations/`, `scripts/`** — non-Go assets.
- **Organize packages by feature/domain, not by technical layer.** Avoid god-packages like `models`/`utils`/`helpers`. A package should be a cohesive noun (`user`, `billing`, `auth`).

> Architecture and packaging are covered in depth in **`GO_LANG_AND_PATTERNS_GUIDE.md`** (clean/layered/hexagonal, repository, DI). This guide focuses on the language; defer to that one for project design.

> **Best practice recap (§16):** keep `main` thin under `cmd/`; hide implementation in `internal/`; package by feature; use `go.work` for multi-module local dev; use `//go:build` tags and filename suffixes for platform code; cross-compile with `GOOS`/`GOARCH` and prefer `CGO_ENABLED=0` for static binaries; commit generated code; embed assets with `//go:embed`.

---

## 17. Performance & Memory: Stack/Heap, GC, Profiling

The golden rule first: **measure before optimizing.** Go's defaults are fast, and "clever" optimizations usually make code worse without measurable gain. Profile, find the real bottleneck, fix *that*, measure again.

### 17.1 Stack vs heap, and escape analysis **[A]**

Every value lives either on a goroutine's **stack** (cheap: allocated/freed automatically as functions enter/return, no GC involvement) or on the **heap** (managed by the garbage collector, more expensive). Unlike C, **you don't choose** — the compiler does, via **escape analysis**: if it can prove a value doesn't outlive the function, it stays on the stack; if a reference "escapes" (you return a pointer to it, store it in a longer-lived structure, send it on a channel, or it's captured by a goroutine), it's moved to the heap so it stays alive.

The practical upshot: **fewer heap allocations = less GC work = faster, more predictable programs.** You can see the compiler's decisions:

```bash
go build -gcflags="-m" .     # prints escape-analysis decisions ("escapes to heap", "does not escape")
```

```go
// Returning a pointer to a local forces it onto the heap (it escapes):
func newUser() *User { u := User{}; return &u } // u escapes -> heap

// Keeping it local lets it stay on the stack:
func sumLocal() int { x := [1024]int{}; return len(x) } // x stays on the stack
```

### 17.2 The garbage collector — what to know **[A]**

Go's GC is a **concurrent, tri-color mark-and-sweep** collector tuned for **low latency** (short pauses) rather than maximum throughput — the right trade-off for servers that must stay responsive. It runs concurrently with your program; pauses are typically sub-millisecond. You mostly don't tune it, but two knobs exist:

- **`GOGC`** (default 100) controls how much the heap grows before the next GC. Higher = fewer GCs but more memory; lower = more frequent GCs, less memory.
- **`GOMEMLIMIT`** (Go 1.19+) sets a soft memory ceiling — very useful in containers to avoid OOM kills; the GC works harder as you approach it.

The best way to "tune the GC" is usually to **allocate less** (§17.4), which reduces GC frequency for free.

### 17.3 Profiling with `pprof` **[A]**

`runtime/pprof` and `net/http/pprof` produce profiles (CPU, heap, goroutine, mutex, block) that `go tool pprof` visualizes. This is how you find *actual* hotspots instead of guessing.

```go
// Easiest: profile a benchmark.
//   go test -bench=. -cpuprofile=cpu.out -memprofile=mem.out
//   go tool pprof cpu.out         # then: top, list <func>, web (needs graphviz)

// For a running server, expose pprof endpoints (DEV/internal only — never public!):
import _ "net/http/pprof" // registers handlers on the default mux at /debug/pprof/
// go func() { log.Println(http.ListenAndServe("localhost:6060", nil)) }()
//   go tool pprof http://localhost:6060/debug/pprof/heap
```

> **Security note:** the `net/http/pprof` endpoints expose internals and can be abused for DoS — never expose them on a public interface; bind to `localhost` or put them behind auth/an internal-only port.

### 17.4 Common allocations to avoid **[A]**

When profiling shows allocation pressure, these are the usual culprits and fixes:

- **String concatenation in loops** → use `strings.Builder` (§4.3).
- **Growing slices without a capacity hint** → `make([]T, 0, n)` when you know the size; each reallocation copies (§6.3).
- **Unnecessary `[]byte`↔`string` conversions** → they copy; use `bytes` when already in bytes, and the compiler optimizes some conversions (e.g. `string(b)` as a map key).
- **Interface boxing of small values / `any`** → can allocate; generics avoid it.
- **Reusable temporary buffers** → `sync.Pool` for hot paths (e.g. per-request buffers).
- **Logging/`fmt.Sprintf` in hot loops** → defer formatting; use leveled logging so debug formatting is skipped in production.

### 17.5 When to optimize **[A]**

Optimize when a **profile** shows a function is hot *and* it's on a path that matters (request latency, throughput, memory ceiling). Don't trade readability for micro-gains the profiler can't even see. Algorithmic improvements (better data structure, fewer DB round-trips, caching) usually dwarf low-level tweaks. Always re-benchmark to confirm the change actually helped.

> **Best practice recap (§17):** measure first (`pprof`, benchmarks); reduce heap allocations to reduce GC work (check with `-gcflags=-m`); set `GOMEMLIMIT` in containers; never expose pprof publicly; prefer algorithmic wins; only optimize hot paths the profiler identifies, and verify with before/after benchmarks.

---

## 18. Security Recommendations

Security is not a separate phase — it's a set of habits woven through the code. These are the language-level practices every Go developer should internalize. (The web guides cover transport/auth specifics; this is the foundation.)

### 18.1 Input validation **[I]**

**Treat all external input as hostile** — request bodies, query params, headers, file uploads, env vars, CLI args, and data from other services. Validate *type, range, length, and format* at the boundary, before the data reaches your logic. Use `strconv` (which returns errors) over assumptions, bound sizes (`http.MaxBytesReader`, `io.LimitReader`), and reject rather than sanitize when you can. Validation libraries (`go-playground/validator`, used by Gin) help, but the principle is yours to enforce.

### 18.2 Avoiding command injection in `os/exec` **[I/A, critical]**

The number-one `os/exec` mistake (repeated from §14 because it matters): **never** pass user input through a shell. `exec.Command` runs the program *directly* with an explicit argument slice — no shell, no metacharacter expansion — which is safe. The danger is only when *you* invoke a shell (`sh -c`, `cmd /C`) with a concatenated string.

```go
// SAFE: the OS executes "convert" directly; args are literal, not shell-parsed.
out, err := exec.Command("convert", userFilename, "-resize", "100x100", outName).Output()
_ = out; _ = err

// UNSAFE: a shell interprets the whole string -> injection.
//   exec.Command("bash", "-c", "convert " + userFilename + " out.png")
//   userFilename = "x.png; curl evil.sh | bash"  => arbitrary code execution
```

Also validate that the *program* and *paths* are what you expect (allowlist binaries; clean/validate file paths to prevent path traversal with `filepath.Clean` and a base-dir check).

### 18.3 SQL injection — parameterized queries **[I/A, critical]**

**Never build SQL by string concatenation.** Use **parameterized queries** (placeholders) so the driver sends data separately from the SQL, making injection impossible. `database/sql` and every reputable driver/ORM support this.

```go
// SAFE: ? / $1 placeholders; the driver binds `email` as data, not SQL.
row := db.QueryRowContext(ctx, "SELECT id, name FROM users WHERE email = $1", email)

// UNSAFE: never do this.
//   db.Query("SELECT * FROM users WHERE email = '" + email + "'")
//   email = "x' OR '1'='1"  => returns every row / worse
```

### 18.4 Safe TLS defaults **[I/A]**

Go's `crypto/tls` defaults are secure — **don't weaken them.** The cardinal sin is `InsecureSkipVerify: true`, which disables certificate verification and opens you to man-in-the-middle attacks; never use it outside throwaway local testing. Set a minimum TLS version (1.2+, prefer 1.3) and let Go pick cipher suites.

```go
tlsCfg := &tls.Config{
	MinVersion: tls.VersionTLS12, // never below 1.2; 1.3 preferred
	// InsecureSkipVerify: true,  // <-- NEVER in production: disables cert checks (MITM)
}
client := &http.Client{
	Timeout:   10 * time.Second,
	Transport: &http.Transport{TLSClientConfig: tlsCfg},
}
_ = client
```

### 18.5 Handling secrets **[I]**

- **Never hardcode** secrets (API keys, DB passwords, signing keys) in source — they end up in git history forever. Load from **environment variables** or a secrets manager (Vault, cloud secret stores).
- **Never log secrets** (§13.10). Redact before logging.
- Keep secrets out of error messages returned to clients.
- For comparing secrets/tokens/MACs, use **`crypto/subtle.ConstantTimeCompare`** to avoid timing attacks — a normal `==` on strings can leak length/contents via timing.

```go
import "crypto/subtle"

// Constant-time comparison resists timing attacks (use for tokens/HMACs, not == ):
equal := subtle.ConstantTimeCompare([]byte(provided), []byte(expected)) == 1
_ = equal
```

### 18.6 `crypto/*` — hashing and secure randomness **[I/A]**

- **Passwords:** never use `md5`/`sha1`/`sha256` directly for passwords — they're too fast to brute-force. Use a slow, salted password hash: **`bcrypt`** (`golang.org/x/crypto/bcrypt`), **`scrypt`**, or **Argon2** (`golang.org/x/crypto/argon2`). (See `GO_JWT_ARGON2_GUIDE.md` for the full treatment.)
- **General hashing/integrity:** `crypto/sha256` for checksums; `crypto/hmac` for keyed message authentication.
- **Secure randomness:** **`crypto/rand`**, never `math/rand`, for tokens, session IDs, salts, nonces, keys. `math/rand` is predictable; a predictable session token is a vulnerability.

```go
import (
	"crypto/rand"
	"encoding/hex"
)

// Cryptographically secure random token:
func secureToken(nBytes int) (string, error) {
	b := make([]byte, nBytes)
	if _, err := rand.Read(b); err != nil { // crypto/rand.Read — NOT math/rand
		return "", err
	}
	return hex.EncodeToString(b), nil
}

// Password hashing with bcrypt (cost >= 10):
//   hash, _ := bcrypt.GenerateFromPassword([]byte(pw), bcrypt.DefaultCost)
//   err := bcrypt.CompareHashAndPassword(hash, []byte(pw))  // nil == match
```

### 18.7 `govulncheck` — scan for known vulnerabilities **[I]**

⚡ Go ships an official vulnerability scanner. **`govulncheck`** checks your dependencies against the Go vulnerability database and — cleverly — reports only vulnerabilities in code paths you *actually call*, cutting noise. Run it in CI.

```bash
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...   # report known vulns reachable from your code
```

> **Best practice recap (§18):** validate and bound all external input; pass arg slices to `exec.Command` (never a shell string); use parameterized SQL; keep TLS defaults (never `InsecureSkipVerify`); load secrets from env/secret stores and never log them; compare secrets with `subtle.ConstantTimeCompare`; hash passwords with bcrypt/argon2 and generate tokens with `crypto/rand`; run `govulncheck` and `go vet` in CI.

---

## 19. Idioms, Gotchas & Best Practices

A consolidated reference of the traps that bite Go developers and the idioms that mark fluent Go. Re-read this after you've written real code — it'll mean more.

### 19.1 The loop variable capture gotcha (before vs after 1.22) **[A]**

The single most infamous Go bug. **Before Go 1.22**, the loop variable was *one* variable reused across iterations, so closures/goroutines all captured the *same* variable and saw its final value:

```go
// PRE-1.22 BEHAVIOUR (go directive < 1.22):
funcs := []func(){}
for i := 0; i < 3; i++ {
	funcs = append(funcs, func() { fmt.Println(i) }) // all close over the SAME i
}
for _, f := range funcs {
	f() // prints 3, 3, 3  (not 0, 1, 2!) — i was 3 when the loop ended
}
// Old fix: shadow with a per-iteration copy:  i := i  before capturing.
```

⚡ **Since Go 1.22 (with `go 1.22+` in `go.mod`), each iteration gets a fresh variable**, so the same code prints `0, 1, 2` — the bug is gone. The same applies to `range` loops and to goroutines launched in loops. **Caveat:** the behaviour depends on the `go` directive version in `go.mod`; old code, and code built with an older language version, still has the trap. When in doubt, the `i := i` copy is harmless on both.

### 19.2 Nil maps **[B/I]**

Reading a nil map is fine (zero value); **writing panics.** `make` it first.

```go
var m map[string]int
_ = m["x"]      // 0 — OK
// m["x"] = 1   // panic: assignment to entry in nil map
m = make(map[string]int) // now writable
```

### 19.3 Slice aliasing **[I]**

Covered in §6.4. The rules: always `s = append(...)`, bound capacity with 3-index slices when handing out sub-slices, and `slices.Clone` for independence.

### 19.4 The typed-nil interface **[A]**

Covered in §9.8. Returning a typed nil pointer as an `error` makes `err != nil`. Return a literal `nil` error.

### 19.5 `defer` in loops **[I]**

Covered in §5.4. Defers run at *function* return, not loop-iteration end. Extract a helper or close explicitly.

### 19.6 Ignoring errors **[B/I]**

Don't silently discard errors with `_` unless you genuinely don't care (and say so in a comment). The `errcheck` linter flags unchecked errors. In particular, **check the error from deferred `Close()` on writes** (§5.4) — a failed flush can lose data.

### 19.7 Naming conventions **[I]**

- **MixedCaps**, never snake_case: `userID`, `HTTPServer` (keep initialisms all-caps: `ID`, `URL`, `HTTP`).
- **Short names for short scopes**: `i`, `r`, `buf`, `ctx` — long descriptive names for package-level identifiers.
- **Package names**: short, lowercase, no underscores, a singular noun (`user`, not `users` or `user_utils`). **Avoid stutter**: `user.New()` not `user.NewUser()`; `http.Server` not `http.HTTPServer`.
- **Getters** drop the `Get` prefix: `u.Name()`, not `u.GetName()`. Setters keep `Set`.
- **Interfaces** for a single method often end in `-er`: `Reader`, `Writer`, `Stringer`.
- **Errors**: variables `ErrXxx`; types `XxxError`.

### 19.8 The idiom summary **[A]**

- **Accept interfaces, return structs** (§9.4); **define interfaces where consumed**; **keep interfaces small**.
- **Make the zero value useful** (§3.4) so types work without constructors.
- **Handle errors explicitly, once** (§10); wrap with `%w`.
- **Don't panic** for expected failures; recover only at boundaries.
- **`gofmt` is law** — don't fight the formatter; run `go vet`/`golangci-lint`.
- **Prefer composition (embedding) over inheritance** (§6.6).
- **Pass `context` first; always `cancel`**; never store it in a struct.
- **Bound concurrency and guarantee goroutine exit** (§12.10); run `-race`.
- **Prefer the standard library**; add dependencies deliberately.
- **Comment the *why*, not the *what*;** exported identifiers get a doc comment starting with their name.

### 19.9 Quick gotcha reference table **[A]**

| Gotcha | Symptom | Fix |
|---|---|---|
| Write to nil map | panic at runtime | `make(map...)` first |
| `append` aliasing | parent slice mutated unexpectedly | 3-index slice `s[a:b:b]` / `slices.Clone` |
| Forgetting `s = append(s,...)` | appended data lost | always reassign |
| Typed-nil interface | `err != nil` when "nil" returned | return literal `nil` |
| `defer` in a loop | fd/resource leak until func returns | helper func / explicit close |
| Loop var capture (pre-1.22) | all closures see final value | upgrade `go` directive / `v := v` |
| `string(intVal)` | gets a Unicode char, not digits | `strconv.Itoa` |
| Map iteration order | flaky/ordered-dependent tests | sort keys explicitly |
| Copying a `sync.Mutex` | races/deadlocks | use pointer receivers; never copy |
| `math/rand` for tokens | predictable, insecure | `crypto/rand` |
| Goroutine leak | rising memory, stuck goroutines | `ctx.Done()`/closed channel exit |
| No HTTP client timeout | hangs forever | `http.Client{Timeout: ...}` |

> **Best practice recap (§19):** know the classic traps cold (nil maps, slice aliasing, typed-nil, defer-in-loop, loop capture); follow Go naming conventions; embrace the core idioms (accept interfaces/return structs, useful zero values, explicit single error handling, composition, context-first); let `gofmt`/`vet`/lint enforce the rest.

---

## 20. Study Path & Build-to-Learn Projects

### Suggested order

The language builds on itself; follow this sequence and *write code at each step* — reading alone won't stick.

1. **§1–§3** — install, run/build, packages, variables, constants, zero values. Write tiny programs; get `go run`/`go build`/`gofmt`/`go test` into muscle memory.
2. **§4–§5** — strings/runes/bytes (truly internalize byte-vs-rune) and control flow (master `defer` timing + LIFO).
3. **§6–§7** — slices, maps, structs, pointers. The slice header and `append` aliasing (§6.2–§6.4) are the highest-leverage beginner concepts — don't move on until they're clear.
4. **§8–§9** — functions, methods, **interfaces**. Implicit interface satisfaction is the conceptual core of Go; understand "accept interfaces, return structs" and the typed-nil trap.
5. **§10** — errors. Wrapping with `%w`, `errors.Is`/`As`, when to panic. This shapes every Go program you'll write.
6. **§11** — generics, once interfaces are solid (know when each applies).
7. **§12** — concurrency. Build the worker pool, pipeline, and fan-in yourself; always run with `-race`. Learn `context` thoroughly.
8. **§13–§14** — the stdlib tour and the concise FS/OS section (then the full FS/OS/CLI guide for I/O depth).
9. **§15–§16** — testing/benchmarking/fuzzing and modules/tooling/layout. Make table-driven tests a habit.
10. **§17–§18** — performance/memory and security. Re-read after you've built something real.
11. **§19** — the gotchas. They'll mean far more once you've hit a few of them.

Then move to **`GO_LANG_AND_PATTERNS_GUIDE.md`** for architecture and production design patterns, and the transport-specific guides (`net/http`, Gin, ent/GORM, JWT, websockets, gRPC) as you build services.

### Build-to-learn projects (increasing depth)

Each project targets specific sections; build the simplest version first, then refactor toward the idioms.

- **CLI tool (e.g. a `wc`/`grep`/todo clone)** — flags (`flag` package), reading files/stdin (`bufio.Scanner`), structs, errors, exit codes. Exercises §3–§8, §10, §14. *Stretch:* JSON persistence, subcommands.
- **A small, well-tested library** — e.g. a generic `Set[T]`, an LRU cache, or a string utility package. Focus on a clean exported API, `Example` tests, table-driven tests, benchmarks, and a `doc.go`. Exercises §9, §11, §15. *Stretch:* fuzz the parser.
- **Concurrent downloader / link checker** — fetch many URLs with a bounded worker pool, `context` timeouts, `errgroup`, progress reporting, graceful Ctrl+C. Exercises §12 end-to-end (and §13 HTTP client, §18 client hardening). *Stretch:* rate limiting, retries with backoff.
- **JSON API client** — wrap a public REST API: typed request/response structs with tags, a configured `http.Client` (timeouts, context), error mapping, retries, and tests against `httptest`. Exercises §9, §10, §13. *Stretch:* pagination via an iterator (1.23 range-over-func).
- **Capstone: a concurrent in-memory key-value store with a CLI/HTTP front-end** — mutex- or channel-guarded map, TTL expiry via a background goroutine, structured logging (`slog`), graceful shutdown, table-driven + race tests, and `pprof` to profile it under load. Exercises almost everything: §6, §9, §10, §12, §13, §15, §17.

For each: start with the simplest thing that compiles and works, get it under test, then refactor toward the idioms and patterns — that's how you learn *why* each rule exists, which is the entire point.

### Cross-references in this library

- **`GO_LANG_AND_PATTERNS_GUIDE.md`** — clean/layered/hexagonal architecture, repository, factory, adapter, DI, functional options, and a complete worked service. (This guide is the language; that one is the architecture.)
- **`GO_NET_HTTP_REST_API_GUIDE.md`** — stdlib HTTP servers, routing, `httptest`, graceful shutdown.
- **`GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md`** — the Gin framework (binding, middleware, uploads, and live reload with Air).
- **`GO_ENT_ORM_GUIDE.md`** — ORM-based data access.
- **`GO_JWT_ARGON2_GUIDE.md`** — auth tokens and password hashing (pairs with §18).
- **`GO_GORILLA_WEBSOCKETS_GUIDE.md`**, **`GO_GRPC_RPC_GUIDE.md`**, **`FTP_SERVER_GO_AND_NODE_GUIDE.md`** — other transports/protocols.
- **`GO_FILESYSTEM_OS_CLI_GUIDE.md`** — the full file/OS/exec/CLI treatment (this guide's §14 is the summary).

Read the standard library source — it is the best Go style guide there is. Happy hacking.
