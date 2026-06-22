# Go — Language & Production Patterns — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I've never written Go" to "I design maintainable, testable, scalable Go backends" — without an internet connection. Every concept is explained in prose first (what it is, *why* Go does it that way, when to use it) and then demonstrated with heavily-commented, runnable code. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Go 1.23 / 1.24** (current in 2026). Things worth knowing about modern Go:
> - **Generics** (type parameters) have been in the language since 1.18 and are now mature and idiomatic where they fit (§8).
> - **Range-over-function iterators** (`for x := range myIterator`) landed in 1.23 — you can now write custom iterators that plug into `range`. The `iter` package and `slices`/`maps` iterator helpers are part of this (§9, §17).
> - **`log/slog`** (structured logging, stdlib since 1.21) is the standard way to log in production (§17).
> - The **loop-variable-per-iteration** change (Go 1.22) fixed the infamous "loop variable captured by goroutine" gotcha — each iteration now gets a fresh variable. Older code and older Go behave differently (§9 gotcha).
> - `min`, `max`, `clear` builtins (1.21); enhanced `net/http` routing (1.22).
>
> Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**, so cross-platform notes (path separators, `.exe` binaries, shells) are called out. Confirm exact APIs at pkg.go.dev.
>
> **This guide's unique value:** The library already has dedicated Go guides for `net/http`, Gin, GORM/ent, JWT, websockets, gRPC, and File System/OS/CLI. So here we keep **(a)** the *language fundamentals* (beginner → advanced) in depth, and **(b)** a large, detailed **production design-patterns** section. File/OS/exec gets only a concise section that cross-references `GO_FILESYSTEM_OS_CLI_GUIDE.md`.

---

## Table of Contents

**Part A — Language Fundamentals**

1. [Why Go, Install & Tooling](#1-why-go-install--tooling) **[B]**
2. [Syntax: Packages, Variables, Constants & Basic Types](#2-syntax-packages-variables-constants--basic-types) **[B]**
3. [Control Flow: if / for / switch / defer](#3-control-flow-if--for--switch--defer) **[B]**
4. [Composite Types: Arrays, Slices, Maps, Structs](#4-composite-types-arrays-slices-maps-structs) **[B/I]**
5. [Functions: Multiple Returns, Variadics, Closures](#5-functions-multiple-returns-variadics-closures) **[B/I]**
6. [Methods & Interfaces](#6-methods--interfaces) **[I]**
7. [Errors: Wrapping, Sentinels, Custom Types, panic/recover](#7-errors-wrapping-sentinels-custom-types-panicrecover) **[I]**
8. [Generics](#8-generics) **[I/A]**
9. [Concurrency: Goroutines, Channels, sync, context](#9-concurrency-goroutines-channels-sync-context) **[A]**
10. [File System, OS & exec (Concise)](#10-file-system-os--exec-concise) **[I]**

**Part B — Production Design Patterns**

11. [Clean / Layered / Hexagonal Architecture](#11-clean--layered--hexagonal-architecture) **[A]**
12. [Repository Pattern](#12-repository-pattern) **[A]**
13. [Factory Pattern](#13-factory-pattern) **[I/A]**
14. [Adapter Pattern & Anti-Corruption Layer](#14-adapter-pattern--anti-corruption-layer) **[A]**
15. [Dependency Injection in Go](#15-dependency-injection-in-go) **[A]**
16. [Strategy Pattern & Functional Options](#16-strategy-pattern--functional-options) **[A]**
17. [Other Production Patterns](#17-other-production-patterns) **[A]**
18. [Putting It Together: A Complete Users Service](#18-putting-it-together-a-complete-users-service) **[A]**
19. [Testing Patterns](#19-testing-patterns) **[I/A]**
20. [Gotchas & Best Practices](#20-gotchas--best-practices) **[A]**
21. [Study Path & Build-to-Learn Projects](#21-study-path--build-to-learn-projects)

---

# Part A — Language Fundamentals

## 1. Why Go, Install & Tooling

### 1.1 The design philosophy — why Go exists **[B]**

Go was created at Google (2009, public; 1.0 in 2012) by Robert Griesemer, Rob Pike, and Ken Thompson to solve a very specific pain: building large server software with large teams. The languages available at the time forced a trade-off — you could have *fast execution* (C++) or *fast development and safety* (Java/Python), but rarely both, and none of them made concurrency pleasant. Go's design is a series of deliberate, opinionated answers to that pain. Understanding the *why* makes the rest of the language feel inevitable rather than arbitrary.

- **Simplicity is a feature, not a limitation.** Go has a tiny specification — you can hold the whole language in your head. There is no inheritance, no method overloading, no exceptions, no implicit conversions, no ternary operator. The reasoning: code is read far more often than it is written, and a small language means any engineer can read any other engineer's code without surprises. When you feel the language "won't let you" do something clever, that is usually intentional.
- **Fast compilation.** Go compiles a large program in seconds. The dependency model (no circular imports, explicit imports, no header files) was designed specifically so the compiler never re-parses the same code repeatedly. Fast builds change how you work — the edit/compile/test loop feels like a scripting language.
- **Concurrency as a first-class citizen.** Goroutines (cheap, lightweight threads managed by the runtime) and channels (typed pipes for communicating between them) are built into the language, not bolted on as a library. The famous mantra is **"Do not communicate by sharing memory; instead, share memory by communicating."** (§9).
- **Static single binaries.** `go build` produces one self-contained executable with no external runtime, no interpreter, no shared libraries to ship. This is why Go dominates cloud/devops tooling (Docker, Kubernetes, Terraform are all Go): you drop one file on a server and it runs. Cross-compilation is trivial (`GOOS=linux GOARCH=amd64 go build`).
- **Garbage collected, but performance-aware.** You get memory safety without manual `malloc/free`, but with a low-latency GC tuned for server workloads.
- **A strong, opinionated standard library and toolchain.** Formatting (`gofmt`), testing, profiling, dependency management, and documentation all ship in the box. There is one official way to format code, which ends all style debates.

When to reach for Go: network services, APIs, CLIs, devops/infra tooling, anything where you want predictable performance and easy deployment. When *not* to: heavy numeric/scientific computing (Python's ecosystem wins), or UI-heavy desktop/mobile apps.

### 1.2 Install **[B]**

- **Windows:** `winget install GoLang.Go` or download the MSI from go.dev/dl. The installer adds `go` to your `PATH`.
- **macOS:** `brew install go` or the pkg installer.
- **Linux:** download the tarball and extract to `/usr/local/go`, or use your package manager.

Verify:

```bash
go version        # e.g. go version go1.24.0 windows/amd64
go env GOPATH     # where downloaded modules & built binaries live (default: ~/go)
```

> **⚡ Version note:** Modern Go (1.21+) can auto-download the toolchain version a project requires via the `toolchain` directive in `go.mod`. You install one Go and it fetches others as needed.

### 1.3 Running and building **[B]**

```bash
go run .            # compile + run the package in the current directory (no binary left behind)
go run main.go      # compile + run a single file
go build            # compile to an executable in the current dir (app.exe on Windows)
go build -o bin/app # choose the output path
go install          # build and drop the binary into $GOPATH/bin (on your PATH)
```

`go run` is for development (it's the scripting-language feel). `go build` is for producing the artifact you ship. On Windows you get `app.exe`; on Linux/macOS just `app`.

Cross-compile (no toolchains to install — it's built in):

```bash
# Build a Linux binary from your Windows machine:
GOOS=linux GOARCH=amd64 go build -o app .       # bash / git-bash
# PowerShell equivalent:
$env:GOOS="linux"; $env:GOARCH="amd64"; go build -o app .
```

### 1.4 Modules — dependency management **[B, important]**

A **module** is a collection of packages versioned together, defined by a `go.mod` file at its root. Before modules (the old `GOPATH` era) Go had no real dependency management; modules (since 1.11, mandatory since 1.16) fixed that. A module path is usually the repo URL, e.g. `github.com/you/myapp` — this doubles as the import prefix for the module's packages.

```bash
go mod init github.com/you/myapp   # create go.mod with your module path
go get github.com/google/uuid      # add a dependency (records it in go.mod + go.sum)
go get github.com/google/uuid@v1.6.0  # pin a specific version
go mod tidy                        # add missing + remove unused deps; do this often
go mod download                    # pre-download deps (useful in CI / Docker)
go mod vendor                      # copy deps into ./vendor (optional, for hermetic builds)
```

`go.mod` records direct dependencies and the minimum Go version; `go.sum` records cryptographic checksums so builds are reproducible and tamper-evident. Commit both. A typical `go.mod`:

```go
module github.com/you/myapp

go 1.24

require (
    github.com/gin-gonic/gin v1.10.0
    github.com/google/uuid v1.6.0
)
```

### 1.5 Project layout **[B/I]**

Go does not force a layout, but a strong community convention has emerged (see §11 for the architectural reasoning). A small project can be flat; a larger one looks like:

```
myapp/
  go.mod
  go.sum
  cmd/
    api/
      main.go        # entry point: package main, func main()
  internal/          # code that CANNOT be imported by other modules (compiler-enforced!)
    user/
      service.go
      repository.go
  pkg/               # (optional) library code you intend others to import
  configs/
  README.md
```

The **`internal/`** directory is special and compiler-enforced: packages under `internal/` can only be imported by code rooted at the parent of `internal/`. This is Go's built-in encapsulation at the package level — put implementation details there so they can't leak into your public API or other modules.

### 1.6 The tooling — fmt, vet, test, doc **[B]**

The Go toolchain is famously batteries-included. The three commands you'll run constantly:

```bash
gofmt -w .        # format files in place (or `go fmt ./...`). There is ONE canonical style.
go vet ./...      # static analysis: catches suspicious constructs (bad Printf args, etc.)
go test ./...     # run all tests in all packages (./... means "this dir and below")
```

- **`gofmt`** ends every style argument. Configure your editor to run it on save. Tabs for indentation, no debate.
- **`go vet`** is a free, fast linter for real bugs (e.g. a `%d` format verb given a string). Run it in CI.
- **`go test`** is the built-in test runner; tests live in `_test.go` files (§19).
- **`go doc fmt.Println`** shows documentation in the terminal — fully offline. `go doc -all net/http` dumps a package's API.
- Popular third-party additions: **`golangci-lint`** (aggregates many linters), **`staticcheck`** (excellent extra checks).

```bash
go doc strings.Builder          # offline docs for a type
go test -v -run TestName ./...  # verbose, single test
go test -race ./...             # the RACE DETECTOR — instruments code to find data races. Use it!
go test -cover ./...            # coverage percentage
go test -bench=. ./...          # run benchmarks
```

> The `-race` flag is one of Go's superpowers. It catches concurrent unsynchronized access to memory at runtime. Always run your tests with `-race` in CI.

---

## 2. Syntax: Packages, Variables, Constants & Basic Types

### 2.1 Every file belongs to a package **[B]**

Every Go source file starts with a `package` declaration. The package named `main` with a `func main()` is the program entry point; any other name is a library package. Within a package, all files share the same scope — you don't import between files of the same package. **Capitalization controls visibility:** an identifier starting with an uppercase letter is *exported* (public, importable from other packages); lowercase is *unexported* (package-private). This is the entire access-control mechanism — no `public`/`private` keywords.

```go
package main // executable program

import (
    "fmt"      // standard library package
    "strings"  // imports are explicit; unused imports are a COMPILE ERROR (keeps code clean)
)

// Exported: starts uppercase -> visible to importers.
func Greet(name string) string {
    return "Hello, " + name
}

// unexported: starts lowercase -> only visible inside this package.
func shout(s string) string {
    return strings.ToUpper(s)
}

func main() {
    fmt.Println(Greet("Go"))
    fmt.Println(shout("quiet"))
}
```

> **Gotcha:** Unused imports *and* unused local variables are compile errors, not warnings. This is intentional — it keeps code free of dead weight. Use the blank identifier `_` to deliberately ignore something.

### 2.2 Variables & the zero-value philosophy **[B]**

There are two ways to declare variables. The verbose `var` form is used at package scope or when you want an explicit type or a zero-valued variable; the short `:=` form (only valid *inside* functions) declares and infers the type from the right-hand side. You'll use `:=` most of the time.

The crucial design idea is the **zero value**: every type in Go has a well-defined default value, and a variable declared without an initializer is *guaranteed* to hold it — never garbage, never undefined. Numbers zero, booleans `false`, strings `""`, and pointers/slices/maps/channels/interfaces/functions are `nil`. This eliminates a whole class of "uninitialized variable" bugs and lets types be *useful before initialization* (e.g. the zero `sync.Mutex` is a ready-to-use unlocked mutex; the zero `bytes.Buffer` is an empty buffer ready to write to). Design your own structs so their zero value is meaningful too.

```go
package main

import "fmt"

func main() {
    var a int            // zero value: 0
    var b string         // zero value: "" (empty string, NOT nil)
    var c bool           // zero value: false
    var d []int          // zero value: nil (a nil slice — usable, len 0)
    var e *int           // zero value: nil pointer

    fmt.Println(a, b == "", c, d == nil, e == nil) // 0 true false true true

    // Short declaration with type inference (functions only):
    name := "Ada"        // string
    age := 36            // int
    pi := 3.14           // float64

    // Multiple assignment (great for swaps — no temp variable):
    x, y := 1, 2
    x, y = y, x          // x=2, y=1
    fmt.Println(name, age, pi, x, y)

    // var block for grouping:
    var (
        host = "localhost"
        port = 8080
    )
    fmt.Println(host, port)
}
```

> **Gotcha:** `:=` declares *new* variables; `=` assigns to existing ones. In a `:=` with multiple names, at least one must be new (the others are just reassigned). Outside functions you must use `var`.

### 2.3 Constants & `iota` **[B]**

Constants are compile-time values (`const`). They can be *untyped*, which is a subtle but powerful feature: an untyped constant like `const big = 1 << 40` has arbitrary precision and only takes on a type when used in a context that needs one, so it flows naturally into `int`, `int64`, `float64`, etc. without explicit conversion. Constants cannot be computed at runtime (no `const x = time.Now()`).

`iota` is a constant generator: within a `const` block it starts at 0 and increments by 1 for each line. It's Go's idiom for enumerations.

```go
package main

import "fmt"

const Pi = 3.14159         // untyped float constant
const Greeting = "hi"      // untyped string constant

// Enum with iota:
type Weekday int
const (
    Sunday Weekday = iota // 0
    Monday                // 1 (the expression "iota" is implicitly repeated)
    Tuesday               // 2
    Wednesday             // 3
)

// iota arithmetic — byte-size constants (a classic idiom):
const (
    _  = iota             // skip 0 with the blank identifier
    KB = 1 << (10 * iota) // 1 << 10 = 1024
    MB                    // 1 << 20
    GB                    // 1 << 30
)

func main() {
    fmt.Println(Monday)       // 1
    fmt.Println(KB, MB, GB)   // 1024 1048576 1073741824
}
```

> To make enums print nicely, generate a `String()` method with the `stringer` tool (`//go:generate stringer -type=Weekday`).

### 2.4 Basic types **[B]**

| Category | Types | Notes |
|---|---|---|
| Signed integers | `int8 int16 int32 int64 int` | `int` is 64-bit on modern platforms; use it unless you need a specific width |
| Unsigned | `uint8 uint16 uint32 uint64 uint uintptr` | `byte` is an alias for `uint8` |
| Floats | `float32 float64` | use `float64` by default |
| Complex | `complex64 complex128` | rarely needed |
| Bool | `bool` | `true`/`false`; **no** implicit int↔bool conversion |
| String | `string` | immutable UTF-8 byte sequence |
| Rune | `rune` | alias for `int32`; one Unicode code point |

Go has **no implicit numeric conversions** — adding an `int` to a `float64` is a compile error. You must convert explicitly. This prevents silent precision/sign bugs.

```go
var i int = 10
var f float64 = float64(i) // explicit conversion required
var u uint = uint(f)       // truncates toward zero
// var bad = i + f         // COMPILE ERROR: mismatched types int and float64
```

### 2.5 Strings, `[]byte`, runes & UTF-8 **[B/I]**

A Go `string` is an **immutable, read-only slice of bytes** — and those bytes are conventionally UTF-8 encoded. This distinction matters constantly:

- Indexing a string (`s[i]`) gives you the **byte** at that position, not the character.
- `len(s)` is the number of **bytes**, not characters. A string of three emoji has `len > 3`.
- Ranging over a string with `for i, r := range s` decodes **runes** (Unicode code points) and gives you the byte index plus the `rune`.
- A `rune` is an `int32` holding one Unicode code point.

Because strings are immutable, building one up byte-by-byte with `+=` is O(n²) — use `strings.Builder` (or a `bytes.Buffer`). Converting between `string` and `[]byte` *copies* the data (because strings are immutable and slices are mutable).

```go
package main

import (
    "fmt"
    "strings"
    "unicode/utf8"
)

func main() {
    s := "héllo, 世界" // contains multi-byte UTF-8 characters

    fmt.Println(len(s))                  // 14 (BYTES, not runes!)
    fmt.Println(utf8.RuneCountInString(s)) // 9 (actual characters)

    // Indexing gives a byte (uint8):
    fmt.Println(s[0])        // 104 (the byte 'h')
    fmt.Printf("%c\n", s[0]) // h

    // Ranging decodes runes; i jumps by the byte-width of each rune:
    for i, r := range s {
        fmt.Printf("%d=%c ", i, r)
    }
    fmt.Println()

    // Convert to []rune to index by character:
    runes := []rune(s)
    fmt.Printf("%c\n", runes[7]) // 世

    // Convert to []byte (copies):
    b := []byte(s)
    b[0] = 'H'
    fmt.Println(string(b)) // Héllo, 世界  (string(b) copies back)

    // Efficient string building:
    var sb strings.Builder
    for i := 0; i < 3; i++ {
        sb.WriteString("go ")
    }
    fmt.Println(sb.String()) // "go go go "
}
```

> **Gotcha:** `string(65)` is `"A"` (interprets the int as a rune), NOT `"65"`. To get the digits use `strconv.Itoa(65)`. `go vet` warns about this.

---

## 3. Control Flow: if / for / switch / defer

### 3.1 `if` — with an optional init statement **[B]**

`if` needs no parentheses around the condition, but always needs braces. Its signature feature is the **init statement**: you can run a short statement before the condition, and any variables it declares are scoped to the `if`/`else` block only. This is the idiomatic way to handle the "call something, check the error, use the result" pattern — it keeps the result variable from leaking into the surrounding scope.

```go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    // Plain if:
    n := 7
    if n%2 == 0 {
        fmt.Println("even")
    } else {
        fmt.Println("odd")
    }

    // if with init statement — `v` and `err` exist ONLY in this if/else:
    if v, err := strconv.Atoi("42"); err == nil {
        fmt.Println("parsed:", v)
    } else {
        fmt.Println("bad number:", err)
    }
    // v and err are out of scope here — can't accidentally reuse a stale value.
}
```

### 3.2 `for` — the only loop **[B]**

Go has exactly one loop keyword: `for`. It covers every looping need, which is another simplicity win — there is no `while`, no `do/while`, no separate `foreach`. The four forms:

```go
package main

import "fmt"

func main() {
    // 1. Classic three-part for:
    for i := 0; i < 3; i++ {
        fmt.Print(i, " ")
    }
    fmt.Println()

    // 2. "while" — just the condition:
    n := 3
    for n > 0 {
        fmt.Print(n, " ")
        n--
    }
    fmt.Println()

    // 3. Infinite loop — break to exit:
    count := 0
    for {
        count++
        if count == 3 {
            break
        }
    }
    fmt.Println("count:", count)

    // 4. range — iterate slices, arrays, maps, strings, channels:
    nums := []int{10, 20, 30}
    for index, value := range nums {
        fmt.Printf("nums[%d]=%d ", index, value)
    }
    fmt.Println()

    m := map[string]int{"a": 1, "b": 2}
    for key, val := range m {
        fmt.Printf("%s=%d ", key, val) // NOTE: map iteration order is RANDOM by design
    }
    fmt.Println()

    // Ignore the index with _:
    sum := 0
    for _, v := range nums {
        sum += v
    }
    fmt.Println("sum:", sum)
}
```

> **⚡ Version note (range-over-int, 1.22):** `for i := range 5 { ... }` loops `i` from 0 to 4. Handy for "do N times".
>
> **⚡ Version note (range-over-func, 1.23):** `range` can now iterate a *function* (a custom iterator). See §9.7 / §17 — this is how `slices.All`, `maps.Keys`, etc. now work.

`break` and `continue` work as expected; you can also `break`/`continue` to a **label** to control an outer loop:

```go
outer:
for i := 0; i < 3; i++ {
    for j := 0; j < 3; j++ {
        if i*j > 2 {
            break outer // breaks the OUTER loop, not just the inner one
        }
    }
}
```

### 3.3 `switch` — flexible and clean **[B/I]**

Go's `switch` is more powerful and safer than C's. Two big differences: cases **do not fall through** by default (no accidental fall-through bugs, no need to write `break` everywhere — use `fallthrough` if you actually want it), and a switch with no expression acts as a clean `if/else if` chain. Cases can be expressions, not just constants.

```go
package main

import "fmt"

func classify(n int) string {
    switch {            // expressionless switch == if/else-if chain
    case n < 0:
        return "negative"
    case n == 0:
        return "zero"
    default:
        return "positive"
    }
}

func main() {
    // Value switch with multiple values per case:
    day := "Sat"
    switch day {
    case "Sat", "Sun":
        fmt.Println("weekend")
    default:
        fmt.Println("weekday")
    }

    // switch with init statement:
    switch x := 5; x {
    case 5:
        fmt.Println("five")
        fallthrough // explicitly continue into the next case
    case 6:
        fmt.Println("...and six (via fallthrough)")
    }

    fmt.Println(classify(-3))
}
```

### 3.4 Type switch **[I]**

A **type switch** inspects the dynamic type held by an interface value (§6). It's the idiomatic way to handle "this could be one of several types". Each case binds the value to its concrete type.

```go
func describe(v any) string { // `any` is the empty interface (§6.3)
    switch x := v.(type) {     // the special .(type) form, only valid in a switch
    case int:
        return fmt.Sprintf("int: %d", x)
    case string:
        return fmt.Sprintf("string of length %d", len(x))
    case bool:
        return fmt.Sprintf("bool: %t", x)
    case nil:
        return "nil"
    default:
        return fmt.Sprintf("unknown type %T", x)
    }
}
```

### 3.5 `defer` — and exactly when it runs **[B/I]**

`defer` schedules a function call to run when the *surrounding function returns* — whether it returns normally or via a panic. This is Go's answer to `try/finally`: it's how you guarantee cleanup (closing files, unlocking mutexes, closing response bodies) right next to the line that acquired the resource, so you can't forget it and reviewers can see the pairing at a glance.

Key rules — internalize these because they cause subtle bugs:

1. **Deferred calls run in LIFO order** (last deferred, first run) — like unwinding a stack.
2. **Arguments are evaluated when `defer` is reached**, not when the deferred call runs. So `defer fmt.Println(i)` snapshots `i` *now*.
3. A deferred function can read and modify **named return values** (§5.2) — this is how you do cleanup that adjusts the error being returned.

```go
package main

import "fmt"

func deferOrder() {
    for i := 0; i < 3; i++ {
        defer fmt.Print(i, " ") // args evaluated now; runs in reverse at return
    }
    fmt.Print("body done -> ")
    // Prints: body done -> 2 1 0
}

func main() {
    deferOrder()
    fmt.Println()
}

// Real-world pairing: open then defer close.
// f, err := os.Open("x.txt")
// if err != nil { return err }
// defer f.Close()   // guaranteed to run no matter how the function exits
```

> **Gotcha:** Calling `defer` inside a loop that runs many times accumulates deferred calls until the *function* returns — they do NOT run at the end of each iteration. If you open files in a loop, wrap the work in a helper function (so each call's defers fire per-iteration) or close manually.

---

## 4. Composite Types: Arrays, Slices, Maps, Structs

### 4.1 Arrays vs slices — and the slice header **[B/I, critical]**

This is one of the most important sections in the language. Getting it wrong causes real production bugs.

An **array** has a *fixed size that is part of its type*: `[3]int` and `[4]int` are different, incompatible types. Arrays are values — assigning or passing an array **copies** all its elements. Because the size is fixed at compile time, arrays are rarely used directly.

A **slice** is the workhorse. A slice is a small struct — the **slice header** — with three fields:

```
slice header = { ptr → backing array, len, cap }
```

- `ptr` points into a **backing array** (which the slice does not own exclusively — several slices can share one).
- `len` is the number of elements currently visible.
- `cap` is how many elements exist from `ptr` to the end of the backing array.

So a slice is a *view* into an array. This is why slices are cheap to pass around (you copy 3 words, not the data) but also why aliasing surprises happen: two slices over the same backing array see each other's writes.

`append` grows a slice. If there's spare capacity (`len < cap`), it writes in place and returns a slice with a larger `len` over the *same* backing array. If capacity is exhausted, it **allocates a new, larger backing array** (typically ~doubling, then growing more slowly for large slices), copies the elements, and returns a slice pointing at the *new* array. This reallocation is the source of the classic gotchas below.

```go
package main

import "fmt"

func main() {
    // Slice literal:
    s := []int{1, 2, 3}
    fmt.Println(len(s), cap(s)) // 3 3

    // make([]T, len, cap) — preallocate when you know the size (perf!):
    buf := make([]int, 0, 10) // len 0, cap 10 — append up to 10 times with NO realloc
    fmt.Println(len(buf), cap(buf)) // 0 10

    // append:
    s = append(s, 4, 5)
    fmt.Println(s) // [1 2 3 4 5]

    // Slicing: s[low:high] -> elements [low, high), len = high-low.
    sub := s[1:3] // {2, 3} — SHARES the backing array with s!
    sub[0] = 99
    fmt.Println(s)   // [1 99 3 4 5]  <- mutating sub changed s!
    fmt.Println(sub) // [99 3]

    // Three-index slice s[low:high:max] limits capacity to control aliasing:
    safe := s[1:3:3] // len 2, cap 2 -> next append MUST reallocate (won't clobber s)
    safe = append(safe, 777)
    fmt.Println(s, safe) // s unchanged past index 2
}
```

**The append-aliasing gotcha (very common):**

```go
original := []int{1, 2, 3, 4, 5}
first := original[:3] // len 3, cap 5 (shares backing array, has spare capacity!)
first = append(first, 100) // cap available -> writes IN PLACE at index 3
fmt.Println(original) // [1 2 3 100 5]  <- append silently overwrote original[3]!
```

The fix is the three-index slice (`original[:3:3]`) which sets `cap == len`, forcing the next append to reallocate. Or copy explicitly with `slices.Clone` / `copy`.

**Slices are passed by value, but the value is the header.** A function that does `s[i] = x` mutates the caller's data (same backing array), but a function that does `s = append(s, x)` may reallocate and the caller won't see it unless you return the new slice. This is why `append` is always used as `s = append(s, ...)`.

```go
func modifyElement(s []int) { s[0] = -1 }      // visible to caller (shared array)
func tryAppend(s []int)     { s = append(s, 9) } // NOT visible (local header reassigned)
```

Other slice essentials:

```go
copy(dst, src)            // copies min(len(dst),len(src)) elements; returns count
s = slices.Delete(s, i, j) // remove [i:j) (Go 1.21+ slices package)
s = slices.Clone(s)        // independent copy
b, _ := slices.BinarySearch(sortedSlice, target)
slices.Sort(s)             // generic sort, no boilerplate
slices.Contains(s, x)      // membership
clear(s)                   // zeroes all elements (1.21+)
```

### 4.2 Maps **[B/I]**

A map is Go's hash table: `map[KeyType]ValueType`. Keys must be **comparable** (anything you can use with `==`: numbers, strings, bools, pointers, structs of comparables — but NOT slices, maps, or functions). Maps are reference-like: a `nil` map (the zero value) can be read (returns the value's zero value) but **writing to a nil map panics** — you must initialize with `make` or a literal first.

The **comma-ok idiom** is essential: indexing a missing key returns the value's zero value silently, so you can't distinguish "key absent" from "key present with zero value" unless you use the two-value form.

```go
package main

import "fmt"

func main() {
    // Initialize (nil map writes PANIC):
    ages := make(map[string]int)
    ages["alice"] = 30
    ages["bob"] = 25

    // Literal:
    scores := map[string]int{"x": 1, "y": 2}

    // Comma-ok: distinguish missing from zero.
    if age, ok := ages["alice"]; ok {
        fmt.Println("alice is", age)
    }
    fmt.Println(ages["nobody"]) // 0 (zero value) — NOT an error

    // Delete:
    delete(ages, "bob")

    // Length:
    fmt.Println(len(ages))

    // Iteration order is RANDOMIZED on purpose (don't rely on it):
    for k, v := range scores {
        fmt.Println(k, v)
    }

    // A nil map is fine to READ but panics on WRITE:
    var m map[string]int
    fmt.Println(m["k"]) // 0, OK
    // m["k"] = 1       // PANIC: assignment to entry in nil map
}
```

> **Gotcha:** You cannot take the address of a map element (`&m[k]` is illegal) because the map may rehash and move it. To mutate a struct stored in a map, read it out, modify, write it back — or store pointers (`map[string]*User`).
>
> **Gotcha:** Maps are not safe for concurrent read+write. Use a `sync.Mutex` or `sync.Map` (§9).

### 4.3 Structs & composition **[B/I]**

A struct groups fields into a single type — Go's equivalent of a record/object. Field visibility follows the capitalization rule. You construct structs with literals; the zero value of a struct has every field at its own zero value.

The defining philosophy: **Go has no inheritance.** Instead it uses **composition** via *embedding*. When you embed a type inside a struct (write the type with no field name), the embedded type's fields and methods are *promoted* — accessible as if they belonged to the outer struct. This gives you code reuse without the fragility of inheritance hierarchies. The mantra is **"composition over inheritance,"** and Go enforces it by simply not offering inheritance.

```go
package main

import "fmt"

type Address struct {
    Street string
    City   string
}

type User struct {
    Name    string
    Age     int
    Address // EMBEDDED (no field name) — its fields are promoted into User
}

func (a Address) FullLocation() string { return a.Street + ", " + a.City }

func main() {
    // Construct with field names (preferred — robust to field reordering):
    u := User{
        Name: "Ada",
        Age:  36,
        Address: Address{
            Street: "1 Logic Ln",
            City:   "London",
        },
    }

    // Promoted fields: access City directly through u, not u.Address.City:
    fmt.Println(u.City)              // "London" (promoted)
    fmt.Println(u.Address.City)      // also works
    fmt.Println(u.FullLocation())    // promoted METHOD too

    // Pointer to struct — & gives a *User; Go auto-dereferences field access:
    p := &u
    p.Age = 37 // same as (*p).Age = 37
    fmt.Println(u.Age)

    // Anonymous struct (handy for one-off data, e.g. JSON or test cases):
    point := struct{ X, Y int }{X: 1, Y: 2}
    fmt.Println(point)

    // Struct comparison: == works if all fields are comparable.
    fmt.Println(Address{"a", "b"} == Address{"a", "b"}) // true
}
```

Struct tags are string metadata read by libraries via reflection (e.g. JSON encoding, DB mapping):

```go
type Product struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Price int    `json:"price,omitempty"` // omit from JSON when zero
}
```

---

## 5. Functions: Multiple Returns, Variadics, Closures

### 5.1 Multiple return values **[B]**

Functions can return multiple values, and Go leans on this heavily — most notably for the `(result, error)` convention that replaces exceptions (§7). This is a deliberate design choice: errors are ordinary values you handle explicitly, right where they occur, rather than control-flow that jumps somewhere distant.

```go
package main

import (
    "errors"
    "fmt"
)

// Returns a result AND an error — the canonical Go signature.
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

func main() {
    if q, err := divide(10, 2); err == nil {
        fmt.Println("result:", q) // 5
    }
    if _, err := divide(1, 0); err != nil {
        fmt.Println("error:", err) // division by zero
    }
}
```

### 5.2 Named return values **[B/I]**

You can name the return values in the signature. They act like pre-declared variables initialized to their zero values, and a bare `return` returns their current values. The most idiomatic use is letting a `defer` inspect or modify the returned `err` — useful for wrapping errors or recovering from panics. Use named returns sparingly; for long functions they can hurt readability.

```go
func readConfig() (cfg string, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered: %v", r) // defer modifies the named return!
        }
    }()
    cfg = "loaded"
    return // bare return: returns cfg and err as they currently stand
}
```

### 5.3 Variadic functions **[B]**

A final parameter written `...T` accepts any number of `T` arguments, received as a `[]T` slice. `fmt.Println` is variadic. To pass an existing slice as the variadic args, "spread" it with `slice...`.

```go
func sum(nums ...int) int { // nums is a []int inside the function
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

func main() {
    fmt.Println(sum(1, 2, 3))      // 6
    fmt.Println(sum())             // 0 (zero args -> nil slice)
    xs := []int{4, 5, 6}
    fmt.Println(sum(xs...))        // 15 — spread the slice
}
```

### 5.4 First-class functions & closures **[B/I]**

Functions are values in Go: you can store them in variables, pass them as arguments, return them, and put them in structs. A **closure** is a function value that *captures* variables from the enclosing scope by reference — it keeps them alive and can mutate them across calls. Closures power the functional-options pattern, middleware, and lazy/stateful helpers you'll see throughout Part B.

```go
package main

import "fmt"

// Returns a closure that remembers its own running total.
func makeCounter() func() int {
    count := 0           // captured by the returned closure
    return func() int {
        count++          // mutates the captured variable; persists between calls
        return count
    }
}

// Higher-order function: takes a function as an argument.
func apply(nums []int, fn func(int) int) []int {
    out := make([]int, len(nums))
    for i, n := range nums {
        out[i] = fn(n)
    }
    return out
}

func main() {
    next := makeCounter()
    fmt.Println(next(), next(), next()) // 1 2 3

    doubled := apply([]int{1, 2, 3}, func(n int) int { return n * 2 })
    fmt.Println(doubled) // [2 4 6]
}
```

> **⚡ Version note (loop-var capture, fixed in 1.22):** Before Go 1.22, a `for` loop reused one variable across iterations, so launching goroutines with `go func(){ use(i) }()` captured the *same* `i` and they often all saw the final value. As of 1.22 each iteration gets a fresh variable and the bug is gone. If you maintain old code (or run an old toolchain), copy the variable: `i := i` inside the loop. (§9 covers this in the goroutine context.)

---

## 6. Methods & Interfaces

This section is the conceptual heart of the language and the foundation for every pattern in Part B. Spend time here.

### 6.1 Methods & receivers — value vs pointer **[I, critical]**

A **method** is a function with a *receiver* — an extra parameter written before the name that binds the function to a type. You can define methods on any type you declare in your package (structs, but also named types over `int`, `string`, slices, etc.) — not just on other packages' types.

The receiver can be a **value** (`func (u User)`) or a **pointer** (`func (u *User)`), and choosing correctly is a real decision:

- A **value receiver** operates on a *copy*. Mutations don't affect the original. Good for small, immutable-ish types where copying is cheap and you don't need to mutate.
- A **pointer receiver** operates on the *original* via its address. Required if the method must **mutate** the receiver, and preferred for **large** structs (to avoid copying) or types that contain a `sync.Mutex` (copying a mutex is a bug).

The practical rule: **be consistent within a type** — if any method needs a pointer receiver, give *all* its methods pointer receivers, so the method set is uniform. Go conveniently lets you call a pointer-receiver method on an addressable value (`u.Mutate()` works even though `u` is a value) by auto-taking the address, but this convenience does *not* extend to interface satisfaction (see the gotcha in §6.2).

```go
package main

import "fmt"

type Counter struct{ n int }

// Pointer receiver: mutates the original.
func (c *Counter) Inc() { c.n++ }

// Value receiver: reads a copy (fine, no mutation).
func (c Counter) Value() int { return c.n }

// Methods can be on non-struct named types too:
type Celsius float64
func (c Celsius) ToF() float64 { return float64(c)*9/5 + 32 }

func main() {
    c := Counter{}
    c.Inc()           // Go auto-takes &c because c is addressable
    c.Inc()
    fmt.Println(c.Value()) // 2

    temp := Celsius(100)
    fmt.Println(temp.ToF()) // 212
}
```

### 6.2 Interfaces & implicit satisfaction **[I, critical]**

An **interface** is a named set of method signatures. Any type that has all those methods *automatically* satisfies the interface — there is **no `implements` keyword, no explicit declaration**. This is called **structural / implicit satisfaction**, and it is the single most important design decision in Go's type system.

Why is implicit satisfaction so powerful? Because it **inverts the dependency direction**. The package that *defines* an interface doesn't need to know about the types that satisfy it, and the types that satisfy it don't need to import the interface. You can write an interface that some standard-library or third-party type already happens to satisfy, retroactively. This decoupling is exactly what makes the Repository, Adapter, and DI patterns in Part B clean — you define small interfaces *where they're consumed*, and any conforming implementation slots in.

The corollary best practice: **"Accept interfaces, return structs."** Functions should accept the smallest interface they need (so callers can pass anything that fits), but return concrete types (so callers get the full functionality and you don't over-abstract). And: **keep interfaces small** — the most reused interfaces in Go (`io.Reader`, `io.Writer`, `fmt.Stringer`, `error`) have *one* method.

```go
package main

import "fmt"

// A small interface — one method.
type Speaker interface {
    Speak() string
}

type Dog struct{ Name string }
func (d Dog) Speak() string { return d.Name + " says Woof" } // Dog satisfies Speaker AUTOMATICALLY

type Robot struct{ ID int }
func (r Robot) Speak() string { return fmt.Sprintf("Unit-%d: beep", r.ID) }

// This function depends only on the BEHAVIOR, not the concrete type.
func announce(s Speaker) {
    fmt.Println(s.Speak())
}

func main() {
    announce(Dog{Name: "Rex"})  // Dog satisfies Speaker
    announce(Robot{ID: 7})      // so does Robot — no registration needed

    // Interface values hold (concrete type, value):
    var s Speaker = Dog{Name: "Fido"}
    fmt.Printf("%T\n", s) // main.Dog
}
```

> **Gotcha (pointer receivers & interfaces):** If a method has a *pointer* receiver, then only a `*T` (not a `T`) satisfies the interface. So if `func (d *Dog) Speak()`, then `var s Speaker = Dog{}` is a **compile error** — you need `var s Speaker = &Dog{}`. The auto-address-taking convenience from §6.1 does NOT apply when assigning to an interface, because the interface might store an unaddressable copy.

### 6.3 The empty interface, `any`, and type assertions **[I]**

The interface with zero methods (`interface{}`) is satisfied by *every* type — it's Go's "value of any type", and since Go 1.18 it has the friendlier alias **`any`**. Use it sparingly: it throws away type information, so you need a type assertion or type switch to get back to a concrete type. Most code should prefer generics (§8) or specific interfaces over `any`.

A **type assertion** `x.(T)` extracts the concrete type `T` from an interface value. The two-value form `v, ok := x.(T)` is safe (returns `ok=false` instead of panicking on mismatch); the single-value form panics on mismatch.

```go
package main

import "fmt"

func main() {
    var v any = "hello"

    // Safe assertion (comma-ok):
    if s, ok := v.(string); ok {
        fmt.Println("string:", s)
    }

    // Unsafe assertion — panics if wrong:
    // n := v.(int) // panic: interface conversion: interface {} is string, not int

    // Type switch (the clean multi-type approach — see §3.4):
    describe := func(x any) {
        switch t := x.(type) {
        case int:
            fmt.Println("int", t)
        case string:
            fmt.Println("string", t)
        default:
            fmt.Printf("other %T\n", t)
        }
    }
    describe(42)
    describe("hi")
    describe(3.14)
}
```

### 6.4 Interface composition **[I]**

Interfaces compose by embedding other interfaces — the composite requires all the embedded methods. The standard library's `io.ReadWriter` is literally `Reader` + `Writer`. This lets you build up capability sets from small pieces, mirroring composition-over-inheritance for behavior.

```go
type Reader interface { Read(p []byte) (n int, err error) }
type Writer interface { Write(p []byte) (n int, err error) }

// Composed interface — anything that is BOTH a Reader and a Writer:
type ReadWriter interface {
    Reader
    Writer
}
```

> **Best practice (interface pollution):** Don't create an interface "just in case." Define one when you have (or clearly will have) **multiple implementations** or you need it for **testing/decoupling** at a boundary. Premature interfaces add indirection with no payoff. More in §20.

---

## 7. Errors: Wrapping, Sentinels, Custom Types, panic/recover

### 7.1 The `error` interface — errors are values **[I, critical]**

Go has no exceptions for ordinary error handling. Instead, **errors are ordinary values**: `error` is just a one-method interface, and functions that can fail return an `error` as their last result. You handle it with a plain `if err != nil`. The philosophy: failure is part of a function's normal contract, so make it explicit and local rather than an invisible control-flow jump. Yes, you write `if err != nil` a lot — that visibility is the point; you always know exactly where things can fail and you're forced to decide what to do.

```go
type error interface {
    Error() string
}
```

```go
package main

import (
    "errors"
    "fmt"
    "strconv"
)

func parse(s string) (int, error) {
    n, err := strconv.Atoi(s)
    if err != nil {
        // Return early on error — the dominant Go control-flow shape.
        return 0, err
    }
    return n, nil
}

func main() {
    n, err := parse("not-a-number")
    if err != nil {
        fmt.Println("failed:", err)
        return
    }
    fmt.Println(n)
    _ = errors.New // (errors imported for later sections)
}
```

### 7.2 Wrapping with `%w`, and `errors.Is` / `errors.As` **[I]**

As an error propagates up the call stack, each layer should add context ("what was I trying to do?") without discarding the original. The `fmt.Errorf` verb **`%w`** *wraps* an error, creating a chain you can later unwrap. Then:

- **`errors.Is(err, target)`** walks the chain looking for a specific *sentinel* error value (use this instead of `err == target`, which breaks once the error is wrapped).
- **`errors.As(err, &target)`** walks the chain looking for an error of a specific *type* and, if found, assigns it into `target` so you can read its fields.

```go
package main

import (
    "errors"
    "fmt"
)

// Sentinel error: an exported, comparable value callers can check for.
var ErrNotFound = errors.New("not found")

func fetchUser(id int) error {
    // Pretend the DB returned ErrNotFound; we add context while preserving it:
    return fmt.Errorf("fetchUser(%d): %w", id, ErrNotFound)
}

func main() {
    err := fetchUser(42)
    fmt.Println(err) // fetchUser(42): not found

    // errors.Is sees through the wrapping:
    if errors.Is(err, ErrNotFound) {
        fmt.Println("handle 404 case")
    }
}
```

### 7.3 Sentinel errors vs custom error types **[I]**

Two ways to let callers distinguish *kinds* of errors:

- **Sentinel errors** — package-level `var ErrX = errors.New(...)`. Cheap; check with `errors.Is`. Good when the error carries no data beyond its identity (e.g. `io.EOF`, `sql.ErrNoRows`).
- **Custom error types** — a struct implementing `error`. Use when the error needs to carry **data** (a field, a wrapped cause, an HTTP status). Check with `errors.As` to recover the typed value.

```go
package main

import (
    "errors"
    "fmt"
)

// Custom error type carrying structured data:
type ValidationError struct {
    Field string
    Msg   string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %q: %s", e.Field, e.Msg)
}

func validateAge(age int) error {
    if age < 0 {
        return &ValidationError{Field: "age", Msg: "must be non-negative"}
    }
    return nil
}

func main() {
    err := validateAge(-5)

    // errors.As recovers the concrete type so we can read its fields:
    var ve *ValidationError
    if errors.As(err, &ve) {
        fmt.Printf("bad field: %s (%s)\n", ve.Field, ve.Msg)
    }
}
```

### 7.4 When to `panic` / `recover` **[I/A]**

`panic` is **not** Go's error handling — it's for *truly unrecoverable* situations or programmer bugs (an impossible state, a nil dereference, an out-of-range index, a failed invariant at startup). A panic unwinds the stack, running deferred functions, and crashes the program unless **recovered**. `recover` (only meaningful inside a deferred function) stops the unwinding and returns the panic value.

Rules of thumb:
- **Don't** use panic for expected/recoverable errors (bad user input, missing file, network failure) — return an `error`.
- **Do** use panic for "this should be impossible" and for `MustХ` initialization helpers (`regexp.MustCompile`) that fail at startup.
- **Do** recover at process boundaries — e.g. an HTTP middleware that catches a panic in one handler so it doesn't take down the whole server, converting it to a 500.

```go
package main

import "fmt"

// A recovery middleware idea: wrap risky work so one panic doesn't kill the process.
func safeCall(fn func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered from panic: %v", r)
        }
    }()
    fn()
    return nil
}

func main() {
    err := safeCall(func() {
        panic("boom")
    })
    fmt.Println(err) // recovered from panic: boom
}
```

---

## 8. Generics

### 8.1 What they are and the problem they solve **[I/A]**

Before Go 1.18, writing a "min of a slice" or a generic container meant either duplicating code per type or using `interface{}` and losing type safety (and paying boxing/assertion costs). **Generics** (type parameters) let you write a single function or type that works for a *set* of types while staying fully type-checked at compile time.

A function declares **type parameters** in square brackets before its ordinary parameters, each with a **constraint** — an interface that specifies which types are allowed (and, for operators like `<`, that they support those operations). The `constraints` package (`golang.org/x/exp/constraints`) and the built-in `comparable` and `any` constraints cover most needs.

```go
package main

import "fmt"

// `T any` — works for ANY type. Returns the last element.
func Last[T any](s []T) T {
    return s[len(s)-1]
}

// A constraint as an inline union: only these types are allowed,
// and the ~ means "any type whose underlying type is this" (covers named types).
type Ordered interface {
    ~int | ~int64 | ~float64 | ~string
}

func Max[T Ordered](a, b T) T { // works because Ordered guarantees `>` is valid
    if a > b {
        return a
    }
    return b
}

// Generic over BOTH key and value; K must be comparable (map key requirement).
func Keys[K comparable, V any](m map[K]V) []K {
    out := make([]K, 0, len(m))
    for k := range m {
        out = append(out, k)
    }
    return out
}

func main() {
    fmt.Println(Last([]int{1, 2, 3}))       // 3 (T inferred as int)
    fmt.Println(Last([]string{"a", "b"}))   // b
    fmt.Println(Max(3, 7))                   // 7
    fmt.Println(Max("apple", "banana"))      // banana
    fmt.Println(Keys(map[string]int{"x": 1, "y": 2}))
}
```

### 8.2 When generics help vs when interfaces are better **[A]**

This is a judgment call worth getting right:

- **Use generics** when the logic is *identical* across types and you want to preserve the concrete type (containers, algorithms over slices/maps, `Map/Filter/Reduce`-style helpers). The stdlib `slices` and `maps` packages are generics.
- **Use interfaces** when you care about *behavior* and want different types to do *different things* behind a common contract (a `Repository`, a `Notifier`, a `PaymentProvider`). This is polymorphism, and it's what Part B's patterns use.

A good heuristic: if you'd otherwise write the same code N times changing only the type, reach for generics. If you'd write *different* code per type behind one method set, reach for an interface. Don't reach for generics to be clever — Go culture still prefers the simplest thing that works.

> **⚡ Version note:** The stdlib now ships generic `slices` and `maps` packages (1.21+), and `cmp` for ordering. Prefer these over hand-rolled generics for common operations: `slices.Sort`, `slices.Index`, `slices.Max`, `maps.Keys` (now returns an iterator, 1.23+), etc.

---

## 9. Concurrency: Goroutines, Channels, sync, context

Concurrency is Go's headline feature. This section is long because it's where most production bugs (and most of Go's leverage) live.

### 9.1 The philosophy: share memory by communicating **[A]**

Most languages do concurrency by sharing mutable memory between threads and guarding it with locks — which is error-prone (deadlocks, races, forgotten locks). Go offers an alternative model inspired by CSP (Communicating Sequential Processes): **"Do not communicate by sharing memory; instead, share memory by communicating."** Rather than multiple goroutines locking a shared variable, you have one goroutine *own* a piece of data and others send/receive messages over **channels** to interact with it. Ownership passes along with the message. Locks still exist (`sync.Mutex`) and are the right tool for simple shared counters/caches, but channels are the idiom for coordinating work and passing ownership.

### 9.2 Goroutines **[A]**

A **goroutine** is a function running concurrently, scheduled by the Go runtime onto OS threads. They are extremely cheap (a few KB of stack that grows as needed), so you can have hundreds of thousands. Start one with the `go` keyword. The catch: `main` returning kills all goroutines immediately, and goroutines don't automatically wait for each other — you need synchronization (a `WaitGroup` or channels) to coordinate.

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup // counts outstanding goroutines

    for i := 0; i < 3; i++ {
        wg.Add(1) // increment BEFORE starting the goroutine
        go func() {
            defer wg.Done()      // decrement when this goroutine finishes
            fmt.Println("worker", i) // i is per-iteration (Go 1.22+), safe to capture
        }()
    }

    wg.Wait() // block until the counter hits zero
    fmt.Println("all done")
}
```

> **⚡ Gotcha (pre-1.22 loop capture):** On Go ≤1.21 the `i` above was shared across iterations, so goroutines often printed the same (final) value. Fixed in 1.22 (each iteration gets a fresh `i`). On old toolchains, write `i := i` inside the loop or pass `i` as an argument: `go func(i int){...}(i)`.

### 9.3 Channels — typed pipes **[A]**

A **channel** is a typed conduit you send to (`ch <- v`) and receive from (`v := <-ch`). Channels synchronize goroutines: by default they're **unbuffered**, meaning a send blocks until another goroutine receives (a rendezvous — it's a synchronization point, not just data transfer). A **buffered** channel (`make(chan T, n)`) holds up to `n` values before sends block, decoupling sender and receiver up to the buffer size.

Closing a channel (`close(ch)`) signals "no more values." Receiving from a closed channel returns the zero value immediately with `ok == false` (the comma-ok form), and ranging over a channel loops until it's closed. **Rule: only the sender closes a channel**, never the receiver, and never send on a closed channel (it panics).

```go
package main

import "fmt"

func main() {
    // Unbuffered channel — send blocks until received.
    ch := make(chan int)
    go func() {
        ch <- 42 // blocks until main receives
    }()
    fmt.Println(<-ch) // 42

    // Buffered channel — holds 2 without a receiver.
    buf := make(chan string, 2)
    buf <- "a"
    buf <- "b"
    // buf <- "c" // would block (buffer full)
    fmt.Println(<-buf, <-buf)

    // Producer/consumer with close + range:
    nums := make(chan int)
    go func() {
        for i := 0; i < 3; i++ {
            nums <- i
        }
        close(nums) // sender closes when done
    }()
    for n := range nums { // loops until channel is closed
        fmt.Println("got", n)
    }

    // Comma-ok detects a closed channel:
    done := make(chan int)
    close(done)
    if v, ok := <-done; !ok {
        fmt.Println("closed; zero value:", v) // closed; zero value: 0
    }
}
```

**Channel directions** document intent and are enforced by the compiler: `chan<- T` is send-only, `<-chan T` is receive-only. Use them in function signatures so a function that's meant to only produce can't accidentally receive.

```go
func produce(out chan<- int) { out <- 1; close(out) } // send-only param
func consume(in <-chan int)  { for v := range in { _ = v } } // receive-only param
```

### 9.4 `select` — waiting on multiple channels **[A]**

`select` blocks until one of several channel operations can proceed, then runs that case (choosing randomly if several are ready). It's the heart of concurrency control: timeouts, cancellation, multiplexing. A `default` case makes it non-blocking.

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    c1 := make(chan string)
    c2 := make(chan string)

    go func() { time.Sleep(50 * time.Millisecond); c1 <- "from c1" }()
    go func() { time.Sleep(20 * time.Millisecond); c2 <- "from c2" }()

    // Wait for whichever arrives first, with a timeout safety net:
    for i := 0; i < 2; i++ {
        select {
        case msg := <-c1:
            fmt.Println(msg)
        case msg := <-c2:
            fmt.Println(msg)
        case <-time.After(time.Second):
            fmt.Println("timed out")
            return
        }
    }

    // Non-blocking receive with default:
    idle := make(chan int)
    select {
    case v := <-idle:
        fmt.Println(v)
    default:
        fmt.Println("nothing ready, moving on") // taken immediately
    }
}
```

### 9.5 `sync` primitives — Mutex, WaitGroup, Once **[A]**

Channels aren't always the right tool. For *protecting shared state* (a counter, a cache, a config map), a **`sync.Mutex`** is simpler and faster. Lock before touching the shared data, `defer Unlock()` right after. **`sync.RWMutex`** allows many concurrent readers but exclusive writers — use it for read-heavy data. **`sync.WaitGroup`** (seen above) waits for a set of goroutines. **`sync.Once`** runs a function exactly once even if called from many goroutines — the idiomatic lazy-singleton initializer (§17).

```go
package main

import (
    "fmt"
    "sync"
)

type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock() // always unlock, even if the body panics
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}

func main() {
    c := &SafeCounter{}
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() { defer wg.Done(); c.Inc() }()
    }
    wg.Wait()
    fmt.Println(c.Value()) // 1000 (deterministic thanks to the mutex)

    // sync.Once — run exactly once:
    var once sync.Once
    for i := 0; i < 3; i++ {
        once.Do(func() { fmt.Println("init runs only once") })
    }
}
```

> **Gotcha:** Never copy a `sync.Mutex` (or a struct containing one) after first use — copying a locked mutex gives you two independent locks. This is why types with a mutex use **pointer receivers** and you pass them around as pointers. `go vet` catches lock copies.

### 9.6 `context` — cancellation, deadlines, request scope **[A, critical for backends]**

`context.Context` is how Go propagates **cancellation signals, deadlines, and request-scoped values** across API boundaries and goroutines. In a server, when a client disconnects or a request times out, you want every goroutine doing work for that request to stop promptly instead of leaking. Context makes that possible: a parent context cancels and all derived child contexts cancel with it.

The conventions, which are nearly universal in Go backends:
- A `Context` is the **first parameter** of a function, named `ctx`.
- You never store a Context in a struct; you pass it down the call chain.
- `ctx.Done()` returns a channel that closes when the context is cancelled; select on it.
- Always call the `cancel` function returned by `WithCancel`/`WithTimeout` (use `defer cancel()`) to release resources.

```go
package main

import (
    "context"
    "fmt"
    "time"
)

// A worker that respects cancellation.
func work(ctx context.Context, id int) {
    for {
        select {
        case <-ctx.Done(): // fires on cancel, timeout, or deadline
            fmt.Printf("worker %d stopping: %v\n", id, ctx.Err())
            return
        case <-time.After(30 * time.Millisecond):
            fmt.Printf("worker %d tick\n", id)
        }
    }
}

func main() {
    // Cancel everything after 100ms (e.g. a request timeout):
    ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
    defer cancel() // ALWAYS — releases the timer even if we return early

    go work(ctx, 1)
    <-ctx.Done()                 // wait for the deadline
    time.Sleep(20 * time.Millisecond) // let the worker print its stop message
    fmt.Println("done:", ctx.Err())   // done: context deadline exceeded
}
```

`context.Background()` is the root (top of `main`, tests). `context.TODO()` is a placeholder when you haven't wired context through yet. Request-scoped values go via `context.WithValue` — but use it sparingly, only for truly request-scoped data (request ID, auth subject), never for passing optional function parameters.

### 9.7 Common concurrency patterns **[A]**

These compose goroutines + channels + context into reusable shapes. They're the backbone of scalable Go backends.

**Worker pool** — a fixed number of workers consume tasks from a channel, bounding concurrency (you don't want to spawn 1M goroutines for 1M tasks hitting a database).

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    const numWorkers = 3
    jobs := make(chan int, 100)    // tasks to do
    results := make(chan int, 100) // results out

    var wg sync.WaitGroup
    // Start a fixed pool of workers:
    for w := 1; w <= numWorkers; w++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := range jobs { // each worker pulls jobs until the channel closes
                results <- j * j  // do the work
            }
        }(w)
    }

    // Feed jobs, then close so workers' range loops end:
    for j := 1; j <= 9; j++ {
        jobs <- j
    }
    close(jobs)

    // Close results once all workers are done (in a separate goroutine so we can range):
    go func() { wg.Wait(); close(results) }()

    sum := 0
    for r := range results {
        sum += r
    }
    fmt.Println("sum of squares:", sum) // 285
}
```

**Pipeline** — stages connected by channels, each stage a goroutine transforming a stream. Combined with **fan-out** (multiple goroutines reading one stage to parallelize) and **fan-in** (merging several channels into one).

```go
package main

import (
    "fmt"
    "sync"
)

// Stage 1: produce numbers.
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

// Stage 2: square them.
func sq(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}

// Fan-in: merge multiple channels into one.
func merge(cs ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    for _, c := range cs {
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

func main() {
    in := gen(2, 3, 4)
    // Fan-out: two squaring stages read the same input concurrently.
    c1 := sq(in)
    c2 := sq(in)
    // Fan-in: merge their outputs.
    for result := range merge(c1, c2) {
        fmt.Println(result) // 4, 9, 16 in some order
    }
}
```

> **⚡ Version note (range-over-func iterators, 1.23):** You can now write a function with signature `func(yield func(T) bool)` and `range` over it: `for v := range mySeq { ... }`. This standardizes custom iteration (the `iter.Seq`/`iter.Seq2` types). The new `slices.All`, `maps.Keys`, `maps.Values` return such iterators. It's distinct from channel-based streaming above — iterators are synchronous/pull-style and don't spawn goroutines.

```go
// A range-over-func iterator (Go 1.23+). `yield` returns false if the consumer breaks early.
func Count(n int) func(yield func(int) bool) {
    return func(yield func(int) bool) {
        for i := 0; i < n; i++ {
            if !yield(i) { // respect early break
                return
            }
        }
    }
}
// Usage: for i := range Count(3) { fmt.Println(i) } // 0 1 2
```

**Concurrency gotchas to burn in:**
- **Goroutine leaks:** a goroutine blocked forever on a channel that's never written/closed never dies — and holds its stack and captured memory. Always have an exit path (close the channel, or `ctx.Done()`).
- **Sending on a closed channel panics.** Only the owner/sender closes.
- **Always run `go test -race`.** The race detector finds unsynchronized access you'll never reproduce reliably otherwise.
- Use **`errgroup`** (`golang.org/x/sync/errgroup`) for "run N goroutines, wait for all, collect the first error, cancel the rest" — it's the production-grade WaitGroup.

```go
// errgroup sketch:
// g, ctx := errgroup.WithContext(ctx)
// g.Go(func() error { return fetchA(ctx) })
// g.Go(func() error { return fetchB(ctx) })
// if err := g.Wait(); err != nil { ... } // first error; ctx cancelled for the rest
```

---

## 10. File System, OS & exec (Concise)

This is intentionally brief — **see `GO_FILESYSTEM_OS_CLI_GUIDE.md` for the full treatment** (file walking, `io/fs`, `bufio`, permissions, temp files, signals, flags/CLI design, robust subprocess handling, etc.). Here's the essential surface so language examples are self-contained.

```go
package main

import (
    "fmt"
    "os"
    "os/exec"
)

func main() {
    // Read an entire file into memory (small files only):
    data, err := os.ReadFile("config.txt") // returns []byte
    if err != nil {
        fmt.Println("read error:", err)
    } else {
        fmt.Println(string(data))
    }

    // Write a whole file (0644 = rw-r--r-- permission bits, ignored on Windows):
    _ = os.WriteFile("out.txt", []byte("hello\n"), 0644)

    // Environment variables:
    home := os.Getenv("HOME")          // "" if unset
    val, ok := os.LookupEnv("PATH")    // ok distinguishes unset from empty
    _ = home; _ = val; _ = ok
    os.Setenv("MY_FLAG", "1")

    // Command-line args: os.Args[0] is the program name.
    fmt.Println("args:", os.Args)

    // Run an external command and capture its output:
    out, err := exec.Command("go", "version").Output()
    if err == nil {
        fmt.Print(string(out))
    }

    // Exit with a status code (skips deferred funcs! — flush/cleanup first):
    // os.Exit(1)
}
```

> **Windows note:** Use `filepath.Join` (not string concatenation with `/`) so paths use the right separator. Built binaries are `*.exe`. `exec.Command("go", ...)` resolves via `PATH` cross-platform.

For everything beyond this — `bufio` scanning, `os.Open`/streaming, `filepath.WalkDir`, `io.Copy`, signal handling (`os/signal`), `flag`/CLI parsing, `context`-aware `exec.CommandContext` — go to **`GO_FILESYSTEM_OS_CLI_GUIDE.md`**.

---

# Part B — Production Design Patterns

> The rest of this guide is the centerpiece: how to structure Go backends so they stay **maintainable, testable, and scalable** as they grow. Each pattern is introduced with the *problem it solves* and its *trade-offs*, then shown in real Go. We use plain Go and **Gin** where a web layer clarifies things (see `GO_GIN_GUIDE.md` for Gin depth). Patterns build toward the complete example in §18.

## 11. Clean / Layered / Hexagonal Architecture

### 11.1 The problem **[A]**

As a backend grows, the dangerous failure mode is *coupling*: HTTP handler code that talks directly to SQL, business rules scattered across request handlers, a database swap that touches a hundred files, and tests that need a real database to run. Everything depends on everything, so nothing can change in isolation.

**Layered / hexagonal (a.k.a. "ports and adapters") architecture** fixes this by separating concerns into layers and enforcing one rule about which way dependencies point.

### 11.2 The layers & the dependency rule **[A]**

Think in three or four concentric layers:

1. **Domain / Entities** (innermost): your core types and pure business rules (`User`, `Order`, validation). Depends on *nothing* — no frameworks, no DB, no HTTP.
2. **Service / Use-cases**: orchestrates business operations (`RegisterUser`, `PlaceOrder`). Depends on the domain and on *interfaces* (ports) for the things it needs (a `UserRepository`, an `EmailSender`).
3. **Adapters / Infrastructure** (outer): concrete implementations of those interfaces — a Postgres repository, an SMTP email client, a Stripe adapter. Depends inward (it implements the interfaces the service defined).
4. **Transport / Delivery** (outermost): HTTP/gRPC handlers, CLI. Translates external requests into service calls.

**The dependency rule: dependencies point *inward*.** The domain knows nothing about the database or HTTP. The service defines interfaces for what it needs; outer layers implement them. This is the key inversion — and Go's *implicit interface satisfaction* (§6.2) makes it natural, because the service can define a `Repository` interface without importing any database package, and the database package implements it without importing the service.

```
            ┌─────────────────────────────────────────┐
            │  Transport (HTTP handlers, Gin, gRPC)     │  outermost
            │   ┌─────────────────────────────────────┐ │
            │   │  Service / Use-cases                  │ │
            │   │   ┌─────────────────────────────────┐ │ │
            │   │   │  Domain (entities, business rules)│ │ │  innermost
            │   │   └─────────────────────────────────┘ │ │
            │   │   defines interfaces (ports) ───────► │ │
            │   └─────────────────────────────────────┘ │
            │  Adapters (Postgres, SMTP) implement them  │
            └─────────────────────────────────────────┘
                    dependencies point INWARD
```

### 11.3 The standard project layout **[A]**

```
myapp/
  cmd/
    api/
      main.go              # composition root: wire everything, start the server
  internal/                # compiler-enforced privacy (§1.5)
    user/
      user.go              # domain entity + domain errors  (innermost)
      service.go           # UserService — use-cases; defines the Repository interface (port)
      repository_memory.go # in-memory adapter (for tests/dev)
      repository_pg.go      # Postgres adapter (production)
      transport_http.go     # Gin handlers (outermost) -> call the service
  pkg/                     # (optional) reusable libs safe for other modules to import
  go.mod
```

> **Why `internal/<feature>/` instead of `internal/handlers`, `internal/models`?** Organizing by **feature** (vertical slices) rather than by **technical layer** keeps everything related to "user" in one place, reduces cross-package churn, and makes deletion/refactoring easy. The layering lives in *file names and interfaces* within the feature package, or in sub-packages when a feature grows large. Avoid a giant `models/` package that every other package imports (it becomes a coupling magnet).

The next sections (§12–§17) each implement one piece of this architecture as a reusable pattern; §18 assembles them into a working service.

---

## 12. Repository Pattern

### 12.1 The problem & the idea **[A]**

Business logic should not know *how* data is stored. If `RegisterUser` contains raw SQL, then (a) you can't unit-test it without a database, (b) switching from Postgres to Mongo (or adding a cache) ripples through your logic, and (c) the storage details pollute the business rules.

The **Repository pattern** introduces an interface that represents your *collection of domain objects* in storage-agnostic terms (`Save`, `FindByID`, `Delete`) — defined in terms of **domain types, not DB rows**. The service depends only on that interface (the "port"). Concrete implementations (the "adapters") live elsewhere. Crucially, **the interface is defined in the consumer's package (the service), not the database package** — that's what keeps the dependency pointing inward (§11.2) and lets you swap implementations or inject a fake in tests.

### 12.2 Domain entity + repository interface **[A]**

```go
// internal/user/user.go — the DOMAIN. No DB, no HTTP imports.
package user

import "errors"

// Domain entity:
type User struct {
    ID    string
    Email string
    Name  string
}

// Domain-level sentinel errors (callers check with errors.Is):
var (
    ErrNotFound      = errors.New("user not found")
    ErrEmailTaken    = errors.New("email already registered")
)

// Repository is the PORT: the service depends on this interface, not on any DB.
// Defined HERE (consumer side) so the dependency points inward.
// Note: methods take a context.Context first (cancellation/timeouts, §9.6).
type Repository interface {
    Save(ctx context.Context, u User) error
    FindByID(ctx context.Context, id string) (User, error)
    FindByEmail(ctx context.Context, email string) (User, error)
    Delete(ctx context.Context, id string) error
}
```

(We'll add the `context` import; shown abbreviated for focus.)

### 12.3 In-memory implementation (for tests & dev) **[A]**

A simple map-backed implementation that satisfies `Repository`. It's perfect for fast unit tests and local development — no database required. Because it implements the same interface, the service can't tell the difference.

```go
// internal/user/repository_memory.go
package user

import (
    "context"
    "sync"
)

// MemoryRepository is a concurrency-safe in-memory adapter.
type MemoryRepository struct {
    mu    sync.RWMutex            // protects the map (§9.5)
    byID  map[string]User
}

// NewMemoryRepository is a FACTORY (§13) returning a ready-to-use repo.
func NewMemoryRepository() *MemoryRepository {
    return &MemoryRepository{byID: make(map[string]User)}
}

func (r *MemoryRepository) Save(ctx context.Context, u User) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    // Enforce unique email (a storage-level invariant):
    for _, existing := range r.byID {
        if existing.Email == u.Email && existing.ID != u.ID {
            return ErrEmailTaken
        }
    }
    r.byID[u.ID] = u
    return nil
}

func (r *MemoryRepository) FindByID(ctx context.Context, id string) (User, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    u, ok := r.byID[id]
    if !ok {
        return User{}, ErrNotFound
    }
    return u, nil
}

func (r *MemoryRepository) FindByEmail(ctx context.Context, email string) (User, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    for _, u := range r.byID {
        if u.Email == email {
            return u, nil
        }
    }
    return User{}, ErrNotFound
}

func (r *MemoryRepository) Delete(ctx context.Context, id string) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    if _, ok := r.byID[id]; !ok {
        return ErrNotFound
    }
    delete(r.byID, id)
    return nil
}
```

### 12.4 A SQL implementation (pgx) **[A]**

The production adapter uses a real database but exposes the *same* interface. Note how it maps DB errors to **domain** errors (`pgx.ErrNoRows` → `ErrNotFound`) — this is part of the anti-corruption boundary (§14): the rest of the app never sees database-specific errors.

```go
// internal/user/repository_pg.go
package user

import (
    "context"
    "errors"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
)

// PostgresRepository implements Repository against a real database.
type PostgresRepository struct {
    pool *pgxpool.Pool
}

func NewPostgresRepository(pool *pgxpool.Pool) *PostgresRepository {
    return &PostgresRepository{pool: pool}
}

func (r *PostgresRepository) Save(ctx context.Context, u User) error {
    // UPSERT: insert or update on conflict. Parameterized to prevent SQL injection.
    _, err := r.pool.Exec(ctx,
        `INSERT INTO users (id, email, name) VALUES ($1, $2, $3)
         ON CONFLICT (id) DO UPDATE SET email = $2, name = $3`,
        u.ID, u.Email, u.Name)
    return err
}

func (r *PostgresRepository) FindByID(ctx context.Context, id string) (User, error) {
    var u User
    err := r.pool.QueryRow(ctx,
        `SELECT id, email, name FROM users WHERE id = $1`, id).
        Scan(&u.ID, &u.Email, &u.Name)
    if errors.Is(err, pgx.ErrNoRows) {
        return User{}, ErrNotFound // translate DB error -> DOMAIN error
    }
    if err != nil {
        return User{}, err
    }
    return u, nil
}

func (r *PostgresRepository) FindByEmail(ctx context.Context, email string) (User, error) {
    var u User
    err := r.pool.QueryRow(ctx,
        `SELECT id, email, name FROM users WHERE email = $1`, email).
        Scan(&u.ID, &u.Email, &u.Name)
    if errors.Is(err, pgx.ErrNoRows) {
        return User{}, ErrNotFound
    }
    return u, err
}

func (r *PostgresRepository) Delete(ctx context.Context, id string) error {
    tag, err := r.pool.Exec(ctx, `DELETE FROM users WHERE id = $1`, id)
    if err != nil {
        return err
    }
    if tag.RowsAffected() == 0 {
        return ErrNotFound
    }
    return nil
}
```

### 12.5 Trade-offs **[A]**

- **Benefit:** business logic is storage-agnostic and unit-testable with a fake/in-memory repo (no DB in tests); swapping or augmenting storage (add caching, change DB) is localized to one file.
- **Cost:** an extra layer of indirection. For a tiny CRUD app it can be overkill — don't add it until you have either a real second implementation *or* a testing need. (See "premature abstraction" in §20.)
- **Don't leak storage concepts** into the interface (no `*sql.Tx` in the signature unless you've designed a transaction abstraction). Keep it in domain terms. For cross-repository transactions, a common approach is a `UnitOfWork`/`Transactor` abstraction or passing a transaction-bound repository.

> See `GO_GORM_ENT_GUIDE.md` for repository implementations using GORM/ent specifically.

---

## 13. Factory Pattern

### 13.1 The idea **[I/A]**

Go has no constructors, so the language convention is a **factory function** named `NewT` that builds and returns a `T` (or `*T`), performing validation and initialization so callers get a fully-formed, valid value. This is the most common pattern in all of Go. Factories matter because they (a) centralize construction logic, (b) can enforce invariants (return an error if inputs are bad), (c) can return an **interface** rather than a concrete type to decouple callers, and (d) can pick *which* concrete implementation to build based on config (an "abstract factory").

### 13.2 Simple constructor `NewX` **[I]**

```go
package user

import (
    "errors"
    "strings"
)

// Constructor with validation: callers can't build an invalid User.
func NewUser(id, email, name string) (User, error) {
    email = strings.TrimSpace(strings.ToLower(email))
    if !strings.Contains(email, "@") {
        return User{}, errors.New("invalid email")
    }
    if name == "" {
        return User{}, errors.New("name required")
    }
    return User{ID: id, Email: email, Name: name}, nil
}
```

### 13.3 Factory returning an interface **[A]**

Returning an interface lets you swap implementations without callers caring. (Note the tension with "return structs" from §20 — return an interface when *choosing among implementations* is the factory's whole job, as here.)

```go
package storage

import "fmt"

type Repository interface { /* Save, FindByID, ... */ }

type Backend string
const (
    BackendMemory   Backend = "memory"
    BackendPostgres Backend = "postgres"
)

// Abstract factory: pick the implementation at runtime from config.
func NewRepository(backend Backend, dsn string) (Repository, error) {
    switch backend {
    case BackendMemory:
        return NewMemoryRepository(), nil
    case BackendPostgres:
        return NewPostgresRepository(dsn) // (this overload would return (Repository, error))
    default:
        return nil, fmt.Errorf("unknown backend %q", backend)
    }
}
```

### 13.4 When to use / not **[A]**

- Always provide a `NewX` for any non-trivial type (anything needing initialization, validation, or a non-meaningful zero value).
- Return a **concrete struct** by default (callers get full functionality, you avoid premature abstraction). Return an **interface** only when the factory's job is to *select* an implementation, or when you genuinely have several.
- Don't build an "AbstractFactoryFactory" — Go culture frowns on Java-style over-engineering. Keep it to a function.

---

## 14. Adapter Pattern & Anti-Corruption Layer

### 14.1 The problem **[A]**

Your code needs to use a third-party SDK or external service (a payment provider, an email API, a cloud client). If you call its types directly everywhere, your domain becomes coupled to that vendor: their types leak into your code, their breaking changes ripple through you, and you can't test without hitting their service. Worse, their model may not match yours (their `Customer` ≠ your `User`).

The **Adapter pattern** wraps the foreign interface behind one *you* define in *your* terms. The layer of adapters that does this translation is called an **Anti-Corruption Layer (ACL)** — it stops the external model from "corrupting" your domain model.

### 14.2 Define your interface, adapt the vendor to it **[A]**

```go
package notify

import (
    "context"
    "fmt"

    vendor "github.com/example/emailsdk" // the third-party SDK
)

// YOUR domain interface — defined in your terms, what your app actually needs.
// Your service depends on THIS, not on the vendor SDK.
type Emailer interface {
    Send(ctx context.Context, to, subject, body string) error
}

// Adapter: wraps the vendor client and satisfies Emailer.
type sendgridAdapter struct {
    client *vendor.Client
    from   string
}

// Factory returns the INTERFACE (the rest of the app never imports the vendor pkg).
func NewSendgridEmailer(apiKey, from string) Emailer {
    return &sendgridAdapter{
        client: vendor.NewClient(apiKey),
        from:   from,
    }
}

// Translate YOUR call into the VENDOR's API shape:
func (a *sendgridAdapter) Send(ctx context.Context, to, subject, body string) error {
    msg := vendor.NewMessage(a.from, to, subject, body) // vendor-specific construction
    resp, err := a.client.SendWithContext(ctx, msg)
    if err != nil {
        return fmt.Errorf("sendgrid send: %w", err) // translate/wrap vendor errors
    }
    if resp.StatusCode >= 400 {
        return fmt.Errorf("sendgrid rejected: status %d", resp.StatusCode)
    }
    return nil // your callers see a plain error, never vendor types
}
```

Now your service depends only on `notify.Emailer`. To swap SendGrid for SES, write another adapter; nothing else changes. To test the service, inject a fake `Emailer` (§19). The vendor SDK is imported in exactly one file.

### 14.3 Adapting incompatible signatures **[I]**

Adapters also bridge mismatched function shapes. A frequent stdlib example: `http.HandlerFunc` is an adapter that turns a plain function into the `http.Handler` interface. You'll write small adapters like this constantly.

```go
// The stdlib's own adapter pattern:
//   type HandlerFunc func(ResponseWriter, *Request)
//   func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) { f(w, r) }
// It lets a function value satisfy the Handler interface.
```

### 14.4 Trade-offs **[A]**

- **Benefit:** vendor lock-in is contained; tests don't need the real service; vendor breaking changes touch one file.
- **Cost:** an extra type and translation code. Worth it for external dependencies you don't control; overkill for stable stdlib types you'd never swap.

---

## 15. Dependency Injection in Go

### 15.1 What DI really is (you don't need a framework) **[A]**

"Dependency injection" sounds heavy, but in Go it's just: **a component receives its dependencies (as interfaces) rather than creating them itself.** Instead of a service doing `repo := NewPostgresRepository(...)` inside itself (hard-coding the choice, impossible to fake), it accepts a `Repository` interface in its constructor. The *caller* decides which concrete implementation to pass. This is **constructor injection**, and it's almost always all you need.

The payoff: components are decoupled from concrete dependencies (swap implementations freely), and they're trivially testable (pass a fake). The whole object graph gets assembled in one place — the **composition root**, which is `main()`.

### 15.2 Constructor injection + wiring in main **[A]**

```go
package user

import "context"

// Service depends on the Repository INTERFACE (a port), injected via the constructor.
type Service struct {
    repo Repository // an interface — the concrete type is chosen by the caller
}

// Constructor injection: dependencies come IN, the service doesn't build them.
func NewService(repo Repository) *Service {
    return &Service{repo: repo}
}

func (s *Service) Get(ctx context.Context, id string) (User, error) {
    return s.repo.FindByID(ctx, id)
}
```

```go
// cmd/api/main.go — the COMPOSITION ROOT. The ONE place that knows concrete types.
package main

func main() {
    // 1. Build the concrete dependencies (decide implementations here):
    repo := user.NewMemoryRepository()      // or NewPostgresRepository(pool) in prod
    emailer := notify.NewSendgridEmailer(apiKey, "no-reply@app.com")

    // 2. Inject them into the service:
    svc := user.NewService(repo /*, emailer */)

    // 3. Inject the service into the transport layer:
    handler := user.NewHTTPHandler(svc)

    // 4. Start the server (transport details elided; see §18).
    _ = handler
}
```

Everything below `main` depends only on interfaces; `main` is the only place that names concrete types and decides the wiring. To switch to Postgres in production, you change *one line* in `main`.

### 15.3 A note on wire / fx **[A]**

For large graphs the manual wiring in `main` can get long. Two tools help:

- **`google/wire`** — compile-time DI. You declare "providers" and `wire` *generates* the wiring code. No runtime reflection, no runtime overhead — it just writes the `main`-style code you'd write by hand. Good when manual wiring becomes unwieldy.
- **`uber-go/fx`** — runtime DI with a lifecycle (start/stop hooks), used in big service fleets. More magic, runtime reflection.

For most apps, **manual constructor injection is the recommended default** — it's explicit, debuggable, and framework-free. Reach for `wire` only when the boilerplate genuinely hurts.

---

## 16. Strategy Pattern & Functional Options

### 16.1 Strategy pattern — interchangeable algorithms **[A]**

The **Strategy pattern** lets you select an algorithm at runtime by holding it behind an interface. In Go it's natural: define an interface for the behavior, provide implementations, and inject the chosen one. Because Go has first-class functions, a single-method strategy is often just a *function type* — even lighter than an interface.

```go
package pricing

// Strategy as an interface (multi-method or stateful strategies):
type DiscountStrategy interface {
    Apply(total float64) float64
}

type NoDiscount struct{}
func (NoDiscount) Apply(total float64) float64 { return total }

type PercentOff struct{ Percent float64 }
func (p PercentOff) Apply(total float64) float64 { return total * (1 - p.Percent/100) }

// The context holds whichever strategy was injected:
type Checkout struct{ discount DiscountStrategy }
func NewCheckout(d DiscountStrategy) *Checkout { return &Checkout{discount: d} }
func (c *Checkout) Total(subtotal float64) float64 { return c.discount.Apply(subtotal) }

// --- Strategy as a plain function type (lighter, for single-method strategies): ---
type RoundingFunc func(float64) float64
func RoundUp(x float64) float64   { /* ... */ return x }
func RoundHalf(x float64) float64 { /* ... */ return x }
// A field of type RoundingFunc can hold either function — no interface needed.
```

### 16.2 Functional options — the idiomatic Go config pattern **[A, very common]**

**The problem:** a constructor needs many optional parameters with sensible defaults. You don't want a `NewServer(addr, timeout, maxConns, tls, logger, ...)` with ten arguments (unreadable, can't tell what `nil, 0, true` means), and you don't want a giant config struct that callers must fully populate. You want: required args positional, optional args named with defaults, and the API extensible without breaking callers.

**The solution — functional options:** the constructor takes required args plus a variadic `...Option`, where `Option` is a function that mutates the config. Each option is a small `WithX(...)` function returning an `Option`. Callers pass only the options they care about; everything else defaults. Adding a new option later doesn't break existing call sites. This is *the* idiomatic Go configuration pattern; you'll see it across the ecosystem (gRPC, AWS SDK, many libraries).

```go
package server

import (
    "log/slog"
    "time"
)

type Server struct {
    addr       string
    timeout    time.Duration
    maxConns   int
    logger     *slog.Logger
}

// Option mutates the Server during construction. This is the whole pattern.
type Option func(*Server)

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}
func WithMaxConns(n int) Option {
    return func(s *Server) { s.maxConns = n }
}
func WithLogger(l *slog.Logger) Option {
    return func(s *Server) { s.logger = l }
}

// Constructor: required `addr` is positional; everything else is optional via opts.
func New(addr string, opts ...Option) *Server {
    // 1. Start with sensible DEFAULTS:
    s := &Server{
        addr:     addr,
        timeout:  30 * time.Second,
        maxConns: 100,
        logger:   slog.Default(),
    }
    // 2. Apply each option in order (later options can override earlier):
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Usage:
//   s := server.New(":8080")                                   // all defaults
//   s := server.New(":8080", server.WithTimeout(5*time.Second),
//                            server.WithMaxConns(500))          // override two
```

**Trade-offs:** more code than a config struct, but far more ergonomic and forward-compatible for libraries with many knobs. For a handful of always-required fields, a plain struct or positional args is simpler — don't reach for options unless optionality is real.

---

## 17. Other Production Patterns

### 17.1 Service layer / use-case pattern **[A]**

The **service layer** holds your *application logic / use-cases* — the orchestration that isn't pure domain rules and isn't transport. A use-case like `RegisterUser` validates input, checks the repo for an existing email, creates the entity, persists it, and sends a welcome email. It depends only on interfaces (repo, emailer), so it's the most valuable thing to unit-test. Keep transport (HTTP parsing) and storage (SQL) *out* of it.

```go
package user

import (
    "context"
    "errors"

    "github.com/google/uuid"
)

type Service struct {
    repo    Repository
    emailer Emailer // injected interface (from §14)
}

func NewService(repo Repository, emailer Emailer) *Service {
    return &Service{repo: repo, emailer: emailer}
}

// A use-case: orchestrates domain + ports. Pure of HTTP and SQL.
func (s *Service) Register(ctx context.Context, email, name string) (User, error) {
    // Reject duplicates (business rule):
    if _, err := s.repo.FindByEmail(ctx, email); err == nil {
        return User{}, ErrEmailTaken
    } else if !errors.Is(err, ErrNotFound) {
        return User{}, err // a real error, not "absent"
    }

    u, err := NewUser(uuid.NewString(), email, name) // factory + validation (§13)
    if err != nil {
        return User{}, err
    }
    if err := s.repo.Save(ctx, u); err != nil {
        return User{}, err
    }

    // Side effect via injected adapter; failure here shouldn't necessarily fail signup:
    _ = s.emailer.Send(ctx, u.Email, "Welcome", "Thanks for joining!")
    return u, nil
}
```

### 17.2 Middleware / decorator pattern **[A]**

A **decorator** wraps a component to add behavior *without changing the component*, preserving its interface. In web servers this is **middleware**: logging, auth, recovery, rate-limiting wrap your handlers. Because Go has first-class functions and small interfaces, middleware is just "a function that takes a handler and returns a handler."

**net/http middleware** (see `GO_NET_HTTP_REST_API_GUIDE.md` for depth):

```go
package mw

import (
    "log/slog"
    "net/http"
    "time"
)

// Middleware = handler in, handler out.
type Middleware func(http.Handler) http.Handler

// Logging decorator: times the request and logs structured fields.
func Logging(logger *slog.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            next.ServeHTTP(w, r) // call the wrapped handler
            logger.Info("request",
                "method", r.Method, "path", r.URL.Path,
                "duration", time.Since(start))
        })
    }
}

// Recovery decorator: a panic in any handler becomes a 500 instead of a crash.
func Recovery(logger *slog.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            defer func() {
                if rec := recover(); rec != nil {
                    logger.Error("panic recovered", "err", rec)
                    http.Error(w, "internal error", http.StatusInternalServerError)
                }
            }()
            next.ServeHTTP(w, r)
        })
    }
}

// Compose middleware (applied outermost-first):
func Chain(h http.Handler, mws ...Middleware) http.Handler {
    for i := len(mws) - 1; i >= 0; i-- {
        h = mws[i](h)
    }
    return h
}
```

**Gin middleware** is the same idea with Gin's signature (see `GO_GIN_GUIDE.md`):

```go
// func AuthMiddleware() gin.HandlerFunc {
//     return func(c *gin.Context) {
//         if c.GetHeader("Authorization") == "" {
//             c.AbortWithStatusJSON(401, gin.H{"error": "unauthorized"})
//             return
//         }
//         c.Next() // proceed to the next handler
//     }
// }
// router.Use(AuthMiddleware())
```

### 17.3 Observer / pub-sub via channels **[A]**

The **Observer pattern** notifies interested parties of events without the publisher knowing who's listening. In Go, channels make this natural: subscribers get a channel; the publisher fans events out to all subscriber channels.

```go
package events

import "sync"

type Event struct{ Name string }

type Bus struct {
    mu   sync.RWMutex
    subs []chan Event
}

func NewBus() *Bus { return &Bus{} }

// Subscribe returns a channel the caller ranges over.
func (b *Bus) Subscribe() <-chan Event {
    ch := make(chan Event, 16) // buffered so a slow subscriber doesn't block the publisher
    b.mu.Lock()
    b.subs = append(b.subs, ch)
    b.mu.Unlock()
    return ch
}

// Publish fans the event out to all subscribers (non-blocking via select+default).
func (b *Bus) Publish(e Event) {
    b.mu.RLock()
    defer b.mu.RUnlock()
    for _, ch := range b.subs {
        select {
        case ch <- e: // delivered
        default:      // subscriber's buffer full -> drop (or handle backpressure)
        }
    }
}
```

> For cross-process pub/sub you'd use NATS/Kafka/Redis, but the in-process channel version is great for decoupling components within one service.

### 17.4 Singleton — and why sparingly **[A]**

A **singleton** is a single shared instance. Go's idiomatic singleton uses `sync.Once` for thread-safe lazy initialization. **Use sparingly:** global singletons are hidden global state — they make testing hard (you can't inject a fake) and couple everything to one instance. Prefer constructor injection (§15). Legitimate uses: a shared logger, a metrics registry, a connection pool created once at startup — and even those are better *created in `main` and injected*.

```go
package db

import "sync"

var (
    instance *Pool
    once     sync.Once
)

// GetPool lazily creates the pool exactly once, safely, even under concurrency.
func GetPool() *Pool {
    once.Do(func() {
        instance = newPool() // runs at most once
    })
    return instance
}
```

### 17.5 Result / error-handling patterns **[A]**

Go uses `(value, error)` rather than a `Result` type. Production conventions that keep error handling clean:
- **Wrap with context as you go up** (`%w`, §7.2) so logs show the full chain.
- **Define error categories at the domain layer** (sentinels/types) and map them to HTTP statuses *at the transport layer only* — domain code stays HTTP-ignorant.
- **Handle each error once**: either handle it (log + recover) or return it — never both (double-logging).

```go
// Transport layer maps DOMAIN errors -> HTTP status (only the edge knows about HTTP):
func statusForError(err error) int {
    switch {
    case errors.Is(err, user.ErrNotFound):
        return http.StatusNotFound
    case errors.Is(err, user.ErrEmailTaken):
        return http.StatusConflict
    default:
        return http.StatusInternalServerError
    }
}
```

### 17.6 Graceful shutdown **[A]**

A production server must finish in-flight requests and close resources (DB pools, queues) before exiting, instead of dropping connections when killed. The pattern: listen for OS signals (`SIGINT`/`SIGTERM`), then call `server.Shutdown(ctx)` with a timeout. (See `GO_NET_HTTP_REST_API_GUIDE.md` for more.)

```go
package main

import (
    "context"
    "errors"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func runServer(srv *http.Server) error {
    // Start serving in a goroutine so main can wait for a signal.
    serverErr := make(chan error, 1)
    go func() {
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            serverErr <- err
        }
    }()

    // Block until an interrupt/terminate signal arrives.
    stop := make(chan os.Signal, 1)
    signal.Notify(stop, syscall.SIGINT, syscall.SIGTERM)

    select {
    case err := <-serverErr:
        return err
    case <-stop:
        // Give in-flight requests up to 10s to complete:
        ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
        defer cancel()
        return srv.Shutdown(ctx) // stop accepting new conns, drain existing ones
    }
}
```

### 17.7 Configuration management **[A]**

Load config from environment variables (12-factor) and/or files, into a typed struct, validated once at startup so the app fails fast on misconfiguration. Use options/defaults for the rest.

```go
package config

import (
    "fmt"
    "os"
    "strconv"
)

type Config struct {
    Port        int
    DatabaseURL string
    LogLevel    string
}

func Load() (Config, error) {
    cfg := Config{
        Port:     8080,    // defaults
        LogLevel: "info",
    }
    if v := os.Getenv("PORT"); v != "" {
        p, err := strconv.Atoi(v)
        if err != nil {
            return Config{}, fmt.Errorf("invalid PORT: %w", err)
        }
        cfg.Port = p
    }
    cfg.DatabaseURL = os.Getenv("DATABASE_URL")
    if cfg.DatabaseURL == "" {
        return Config{}, fmt.Errorf("DATABASE_URL is required") // fail fast
    }
    if v := os.Getenv("LOG_LEVEL"); v != "" {
        cfg.LogLevel = v
    }
    return cfg, nil
}
```

> Popular libs: **`spf13/viper`** (files + env + flags), **`kelseyhightower/envconfig`** / **`caarlos0/env`** (env → struct via tags). For most services, a small hand-written loader like the above is plenty.

### 17.8 Context propagation **[A]**

Thread `ctx` from the transport layer (where the request begins) through the service and into the repository, so cancellation/timeouts flow end-to-end (a cancelled HTTP request cancels the DB query). The earlier `Repository`/`Service` methods all take `ctx context.Context` first for exactly this reason. Never start a `ctx`-less goroutine for request work; pass the request's context.

### 17.9 Structured logging with `slog` **[A]**

**`log/slog`** (stdlib since 1.21) replaces ad-hoc `log.Printf` with **structured, leveled logging** — key/value pairs that log aggregators (Loki, Datadog, CloudWatch) can index and query. Create one logger in `main`, inject it (don't reach for a global). Attach request-scoped fields (request ID, user) with `logger.With(...)`.

```go
package main

import (
    "log/slog"
    "os"
)

func main() {
    // JSON handler for production (machine-parseable); TextHandler for local dev.
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))

    logger.Info("server starting", "port", 8080, "env", "prod")
    logger.Error("db connect failed", "err", "timeout", "attempt", 3)

    // Derive a logger with fixed fields (e.g. per request):
    reqLog := logger.With("request_id", "abc-123")
    reqLog.Info("handling request", "path", "/users")
    // Output (JSON): {"time":...,"level":"INFO","msg":"handling request",
    //                 "request_id":"abc-123","path":"/users"}
}
```

---

## 18. Putting It Together: A Complete Users Service

This section assembles §11–§17 into a small but complete, idiomatic backend: Gin transport → service layer → repository interface with two implementations → factory constructors → functional options → DI wired in `main`. It demonstrates maintainability (clear layers), testability (interfaces + fakes), and scalability (swap storage, add middleware). The code is split by file as it would be in a real project under `internal/user/`.

### 18.1 Domain (`internal/user/user.go`)

```go
package user

import (
    "context"
    "errors"
    "strings"
)

// --- Domain entity ---
type User struct {
    ID    string `json:"id"`
    Email string `json:"email"`
    Name  string `json:"name"`
}

// --- Domain errors (transport maps these to HTTP statuses) ---
var (
    ErrNotFound   = errors.New("user not found")
    ErrEmailTaken = errors.New("email already registered")
    ErrInvalid    = errors.New("invalid user data")
)

// --- Factory with validation (§13): you cannot build an invalid User ---
func NewUser(id, email, name string) (User, error) {
    email = strings.TrimSpace(strings.ToLower(email))
    name = strings.TrimSpace(name)
    if !strings.Contains(email, "@") {
        return User{}, errors.Join(ErrInvalid, errors.New("email must contain @"))
    }
    if name == "" {
        return User{}, errors.Join(ErrInvalid, errors.New("name is required"))
    }
    return User{ID: id, Email: email, Name: name}, nil
}

// --- Repository PORT (§12): defined here, in the consumer's package ---
type Repository interface {
    Save(ctx context.Context, u User) error
    FindByID(ctx context.Context, id string) (User, error)
    FindByEmail(ctx context.Context, email string) (User, error)
    Delete(ctx context.Context, id string) error
}
```

### 18.2 In-memory repository (`internal/user/repository_memory.go`)

```go
package user

import (
    "context"
    "sync"
)

type MemoryRepository struct {
    mu   sync.RWMutex
    byID map[string]User
}

func NewMemoryRepository() *MemoryRepository {
    return &MemoryRepository{byID: make(map[string]User)}
}

func (r *MemoryRepository) Save(ctx context.Context, u User) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    for _, e := range r.byID {
        if e.Email == u.Email && e.ID != u.ID {
            return ErrEmailTaken
        }
    }
    r.byID[u.ID] = u
    return nil
}

func (r *MemoryRepository) FindByID(ctx context.Context, id string) (User, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    if u, ok := r.byID[id]; ok {
        return u, nil
    }
    return User{}, ErrNotFound
}

func (r *MemoryRepository) FindByEmail(ctx context.Context, email string) (User, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    for _, u := range r.byID {
        if u.Email == email {
            return u, nil
        }
    }
    return User{}, ErrNotFound
}

func (r *MemoryRepository) Delete(ctx context.Context, id string) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    if _, ok := r.byID[id]; !ok {
        return ErrNotFound
    }
    delete(r.byID, id)
    return nil
}
```

*(The Postgres implementation from §12.4 is the second implementation; it satisfies the same `Repository` interface.)*

### 18.3 Service with functional options (`internal/user/service.go`)

```go
package user

import (
    "context"
    "errors"
    "log/slog"

    "github.com/google/uuid"
)

// Service is the use-case layer (§17.1). Depends only on the Repository interface.
type Service struct {
    repo   Repository
    logger *slog.Logger
    idGen  func() string // injectable for deterministic tests (strategy via func, §16)
}

// --- Functional options (§16.2) ---
type Option func(*Service)

func WithLogger(l *slog.Logger) Option { return func(s *Service) { s.logger = l } }
func WithIDGenerator(f func() string) Option { return func(s *Service) { s.idGen = f } }

// Constructor: required repo positional; everything else optional with defaults.
func NewService(repo Repository, opts ...Option) *Service {
    s := &Service{
        repo:   repo,
        logger: slog.Default(),
        idGen:  uuid.NewString, // default ID strategy
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Use-case: register a new user.
func (s *Service) Register(ctx context.Context, email, name string) (User, error) {
    if _, err := s.repo.FindByEmail(ctx, email); err == nil {
        return User{}, ErrEmailTaken
    } else if !errors.Is(err, ErrNotFound) {
        return User{}, err
    }

    u, err := NewUser(s.idGen(), email, name)
    if err != nil {
        return User{}, err
    }
    if err := s.repo.Save(ctx, u); err != nil {
        return User{}, err
    }
    s.logger.Info("user registered", "id", u.ID, "email", u.Email)
    return u, nil
}

func (s *Service) Get(ctx context.Context, id string) (User, error) {
    return s.repo.FindByID(ctx, id)
}

func (s *Service) Delete(ctx context.Context, id string) error {
    return s.repo.Delete(ctx, id)
}
```

### 18.4 Transport with Gin (`internal/user/transport_http.go`)

```go
package user

import (
    "errors"
    "net/http"

    "github.com/gin-gonic/gin"
)

// Handler depends only on the Service (injected). No SQL here.
type Handler struct {
    svc *Service
}

func NewHandler(svc *Service) *Handler { return &Handler{svc: svc} }

// RegisterRoutes wires endpoints onto a Gin router group.
func (h *Handler) RegisterRoutes(r gin.IRoutes) {
    r.POST("/users", h.create)
    r.GET("/users/:id", h.get)
    r.DELETE("/users/:id", h.delete)
}

func (h *Handler) create(c *gin.Context) {
    var req struct {
        Email string `json:"email"`
        Name  string `json:"name"`
    }
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid body"})
        return
    }
    u, err := h.svc.Register(c.Request.Context(), req.Email, req.Name) // propagate ctx
    if err != nil {
        c.JSON(statusFor(err), gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusCreated, u)
}

func (h *Handler) get(c *gin.Context) {
    u, err := h.svc.Get(c.Request.Context(), c.Param("id"))
    if err != nil {
        c.JSON(statusFor(err), gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusOK, u)
}

func (h *Handler) delete(c *gin.Context) {
    if err := h.svc.Delete(c.Request.Context(), c.Param("id")); err != nil {
        c.JSON(statusFor(err), gin.H{"error": err.Error()})
        return
    }
    c.Status(http.StatusNoContent)
}

// Map DOMAIN errors -> HTTP status. The ONLY place that knows about HTTP codes.
func statusFor(err error) int {
    switch {
    case errors.Is(err, ErrNotFound):
        return http.StatusNotFound
    case errors.Is(err, ErrEmailTaken):
        return http.StatusConflict
    case errors.Is(err, ErrInvalid):
        return http.StatusBadRequest
    default:
        return http.StatusInternalServerError
    }
}
```

### 18.5 Composition root (`cmd/api/main.go`)

```go
package main

import (
    "log/slog"
    "os"

    "github.com/gin-gonic/gin"
    "github.com/you/myapp/internal/user"
)

func main() {
    // --- Build dependencies (the one place that picks concrete types) ---
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    // Swap this single line for user.NewPostgresRepository(pool) in production:
    repo := user.NewMemoryRepository()

    // Inject repo + options into the service (DI, §15):
    svc := user.NewService(repo, user.WithLogger(logger))

    // Inject the service into the transport layer:
    handler := user.NewHandler(svc)

    // --- Wire HTTP ---
    r := gin.New()
    r.Use(gin.Recovery()) // middleware/decorator (§17.2)
    handler.RegisterRoutes(r)

    logger.Info("listening", "addr", ":8080")
    if err := r.Run(":8080"); err != nil {
        logger.Error("server failed", "err", err)
        os.Exit(1)
    }
}
```

**What this demonstrates:**
- *Maintainability:* each layer has one job; HTTP, business logic, and storage never bleed into each other.
- *Testability:* the service depends on the `Repository` interface, so tests inject `MemoryRepository` (or a fake) — no database needed (§18.6).
- *Scalability:* swap one line in `main` to go from in-memory to Postgres; add caching by wrapping the repo (decorator); add cross-cutting concerns via middleware.

### 18.6 Unit-testing the service with a fake repo

Because `Service` accepts the `Repository` interface, a test can pass a hand-written fake to drive any scenario — including error paths a real DB can't easily produce.

```go
package user

import (
    "context"
    "errors"
    "testing"
)

// A fake repository whose behavior the test fully controls (§19).
type fakeRepo struct {
    saveErr    error
    findByEmail func(string) (User, error)
    saved      []User
}

func (f *fakeRepo) Save(_ context.Context, u User) error {
    if f.saveErr != nil {
        return f.saveErr
    }
    f.saved = append(f.saved, u)
    return nil
}
func (f *fakeRepo) FindByEmail(_ context.Context, e string) (User, error) {
    if f.findByEmail != nil {
        return f.findByEmail(e)
    }
    return User{}, ErrNotFound // default: nobody exists
}
func (f *fakeRepo) FindByID(context.Context, string) (User, error) { return User{}, ErrNotFound }
func (f *fakeRepo) Delete(context.Context, string) error           { return nil }

func TestRegister_Success(t *testing.T) {
    repo := &fakeRepo{}
    // Deterministic ID via the injectable strategy (no random UUIDs in assertions):
    svc := NewService(repo, WithIDGenerator(func() string { return "fixed-id" }))

    u, err := svc.Register(context.Background(), "a@b.com", "Ada")
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if u.ID != "fixed-id" || u.Email != "a@b.com" {
        t.Errorf("unexpected user: %+v", u)
    }
    if len(repo.saved) != 1 {
        t.Errorf("expected 1 saved user, got %d", len(repo.saved))
    }
}

func TestRegister_DuplicateEmail(t *testing.T) {
    repo := &fakeRepo{
        findByEmail: func(string) (User, error) { return User{ID: "x"}, nil }, // exists!
    }
    svc := NewService(repo)
    _, err := svc.Register(context.Background(), "a@b.com", "Ada")
    if !errors.Is(err, ErrEmailTaken) {
        t.Errorf("expected ErrEmailTaken, got %v", err)
    }
}
```

---

## 19. Testing Patterns

### 19.1 Go's built-in testing **[I]**

Tests live in `_test.go` files in the same package; functions named `TestXxx(t *testing.T)` are tests. No assertion library is required — you call `t.Errorf`/`t.Fatalf`. Run with `go test ./...`. This zero-dependency, fast-feedback testing is core to Go culture.

```go
func TestAdd(t *testing.T) {
    got := Add(2, 3)
    if got != 5 {
        t.Errorf("Add(2,3) = %d; want 5", got) // Errorf continues; Fatalf stops the test
    }
}
```

### 19.2 Table-driven tests — the idiomatic style **[I]**

Instead of one test function per case, list the cases in a slice of structs and loop. This is *the* Go testing idiom: adding a case is one line, failures pinpoint the case name, and the logic isn't duplicated. Use `t.Run` for named subtests so you can run/filter individually.

```go
func TestNewUser(t *testing.T) {
    tests := []struct {
        name    string
        email   string
        uname   string
        wantErr bool
    }{
        {"valid", "a@b.com", "Ada", false},
        {"no at sign", "abc", "Ada", true},
        {"empty name", "a@b.com", "", true},
        {"lowercased+trimmed", "  A@B.COM ", "Ada", false},
    }
    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) { // named subtest -> go test -run TestNewUser/valid
            _, err := NewUser("id", tc.email, tc.uname)
            if (err != nil) != tc.wantErr {
                t.Errorf("NewUser(%q,%q) err=%v, wantErr=%v", tc.email, tc.uname, err, tc.wantErr)
            }
        })
    }
}
```

### 19.3 Interface fakes & mocks **[I/A]**

Because dependencies are interfaces, the easiest "mock" is a hand-written **fake** struct (as in §18.6) — explicit, no magic, no codegen. Reach for tooling only when you have many methods/expectations:

- **`testify`** (`stretchr/testify`) — popular helpers: `assert`/`require` (readable assertions) and `mock` (expectation-based mocks). `require.NoError(t, err)` stops on failure; `assert.Equal(t, want, got)` continues.
- **`mockgen`** (`go.uber.org/mock`) / **`moq`** — generate mock implementations of an interface from a `//go:generate` directive, so you don't hand-write large fakes.

```go
// With testify (illustrative):
// require.NoError(t, err)
// assert.Equal(t, "fixed-id", u.ID)
```

> Hand-written fakes are recommended by default — they're simplest and most readable. Generate mocks when the interface is large or you need call-count/argument assertions.

### 19.4 DI for testability **[A]**

The reason the whole architecture is testable comes back to **dependency injection** (§15): every external dependency (repo, emailer, clock, ID generator) enters through a constructor as an interface or function, so a test substitutes a controlled fake. Note the `WithIDGenerator` and the injectable clock idea — inject *time* and *randomness* too, so tests are deterministic. A handler test can use `net/http/httptest` (see `GO_NET_HTTP_REST_API_GUIDE.md`).

```go
// Injecting a clock makes time-dependent logic testable:
// type Clock func() time.Time
// svc := NewService(repo, WithClock(func() time.Time { return fixedTime }))
```

---

## 20. Gotchas & Best Practices

A consolidated checklist. Many of these were flagged inline; here they are together.

**Interfaces & abstraction**
- **Accept interfaces, return structs.** Functions take the *smallest* interface they need; constructors return concrete types. This maximizes caller flexibility without over-abstracting.
- **Avoid interface pollution.** Don't create an interface with one implementation "for the future." Add it when you have a *second* implementation or a *real* testing/decoupling need. Premature interfaces add indirection and obscure the code.
- **Define interfaces where they're consumed**, not where they're implemented. The service package owns the `Repository` interface; the Postgres package just implements it. This keeps the dependency pointing inward (§11).
- **Keep interfaces small.** One- or two-method interfaces compose and get reused; big interfaces are hard to satisfy and fake.

**Slices, maps, values**
- `append` may reallocate; always write `s = append(s, ...)`. Beware aliasing when slicing (`s[:n]` shares the backing array) — use three-index slices or `slices.Clone`.
- Writing to a `nil` map panics; `make` it first. Map iteration order is random — never depend on it.
- You can't take the address of a map element; store `*T` or read-modify-write.

**Pointers & receivers**
- Be **consistent** with value vs pointer receivers within a type. If any method mutates or the struct is large or holds a mutex, use pointer receivers everywhere.
- A `nil` pointer of a concrete type stored in an interface makes the interface **non-nil** (the classic "typed nil" bug). Returning `var e *MyError = nil` as an `error` yields `err != nil`. Return a literal `nil` error instead.

**Concurrency**
- Run `go test -race` in CI. Protect shared state with a mutex or confine it to one goroutine + channels.
- Avoid goroutine leaks: every goroutine needs a guaranteed exit (closed channel or `ctx.Done()`).
- Never copy a `sync.Mutex`/`WaitGroup` after use. Only the sender closes a channel.
- Pass `context.Context` as the first parameter; don't store it in structs.

**Errors**
- Wrap with `%w` and add context; check with `errors.Is`/`errors.As`, not `==` or string matching.
- Don't `panic` for expected failures. Recover only at boundaries (e.g. HTTP middleware).
- Handle an error once — log *or* return, not both.

**Package & project design**
- Organize by **feature**, not by technical layer (avoid god-packages like `models`/`utils`).
- Use `internal/` to hide implementation packages from other modules.
- A package name should be a short noun; avoid stutter (`user.User`, not `user.UserStruct`; never `user.NewUserService` when `user.NewService` reads fine).
- Don't over-engineer: no Java-style `AbstractFactoryBeanFactory`. Start simple; add patterns when the pain is real. The simplest design that meets the need is the Go way.

**General**
- Let `gofmt` and `go vet` (and `golangci-lint`) run automatically. Don't fight the formatter.
- Prefer the standard library; reach for dependencies deliberately.
- Make the **zero value useful** when designing types, so callers can use them without ceremony.

---

## 21. Study Path & Build-to-Learn Projects

### Suggested order

1. **§1–§3** — install, run, syntax, control flow. Write a few tiny programs; get `go run`/`go build`/`gofmt`/`go test` into muscle memory.
2. **§4–§5** — slices, maps, structs, functions. Truly understand the slice header and `append` aliasing (§4.1) — it's the most common beginner bug.
3. **§6–§7** — methods, interfaces, errors. This is the conceptual core; everything in Part B rests on implicit interface satisfaction and `(value, error)`.
4. **§8** — generics, once interfaces are solid (know when each applies).
5. **§9** — concurrency. Build the worker pool and pipeline yourself; always run with `-race`.
6. **§10** + the **File System / OS / CLI guide** for I/O and tooling.
7. **§11–§17** — architecture and patterns, in order. Implement each small example.
8. **§18–§19** — build and test the complete Users service; this is where it clicks.
9. **§20** — re-read the gotchas after you've written real code; they'll mean more.

### Build-to-learn projects (in increasing depth)

- **CLI todo app** — flags, file persistence (JSON), structs, errors. (Pairs with the File System / OS / CLI guide.)
- **URL shortener** — `net/http` (or Gin), a `Repository` interface with in-memory + SQLite/Postgres impls, DI in `main`. Exercises §11–§15, §18 directly.
- **Concurrent web scraper / link checker** — worker pool, fan-in/out, `context` timeouts, `errgroup`. Exercises §9.
- **Rate-limited API gateway** — middleware/decorator chain (logging, auth, rate limit), graceful shutdown, structured logging, config from env. Exercises §17.
- **Event-driven order service** — service layer, repository, an adapter around a fake payment SDK (anti-corruption layer), pub/sub via channels, table-driven tests with fakes. Exercises the full Part B.
- **Pluggable storage library** — a package exposing a `Store` interface with multiple backends selected by an abstract factory and configured via functional options. Exercises §13, §16.

For each project: start with the simplest thing that works, then refactor toward the layered architecture only when you feel the coupling pain — that way you'll understand *why* each pattern exists, which is the entire point of this guide.

### Cross-references in this library

- **`GO_NET_HTTP_REST_API_GUIDE.md`** — stdlib HTTP servers, routing, `httptest`, graceful shutdown in depth.
- **`GO_GIN_GUIDE.md`** — Gin framework specifics (binding, middleware, groups).
- **`GO_GORM_ENT_GUIDE.md`** — ORM-based repository implementations.
- **`GO_JWT_GUIDE.md`** — auth tokens (pairs with the middleware section).
- **`GO_WEBSOCKETS_GUIDE.md`**, **`GO_GRPC_GUIDE.md`** — other transports.
- **`GO_FILESYSTEM_OS_CLI_GUIDE.md`** — the full file/OS/exec/CLI treatment (this guide's §10 is the summary).

Happy hacking. Read the standard library source — it's the best Go style guide there is.
