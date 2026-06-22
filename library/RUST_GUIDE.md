# Rust — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I've never written Rust" to "I write safe, idiomatic, production Rust — and I understand *why* the compiler keeps telling me no" — without an internet connection. Rust has a famously steep early curve because it asks you to learn a genuinely new idea (ownership) before you can write much code. This guide takes that seriously: the hard concepts (ownership, borrowing, lifetimes) are explained slowly, in prose, with the *reasoning* behind them, before any clever code. Read top-to-bottom the first time; after that, use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets the **Rust 2024 edition** on a stable toolchain (current in 2026). Key context:
> - **Editions** are opt-in language dialects (2015, 2018, 2021, 2024). A 2024-edition crate can depend on a 2018-edition crate and vice-versa — editions are per-crate and the compiler links them together. You select the edition in `Cargo.toml` (`edition = "2024"`). New projects should use 2024.
> - **rustup** manages toolchains (stable / beta / nightly) and targets; **cargo** is the build tool, package manager, and test runner all in one. You will use `cargo` constantly.
> - **The borrow checker** is the part of the compiler that enforces ownership and borrowing rules at compile time. It is the source of most early frustration and most of Rust's value. Modern Rust uses **NLL** (non-lexical lifetimes), which makes the checker much more permissive than the early-days reputation suggests.
> - **Async** in Rust is library-driven: the language gives you `async`/`.await` and the `Future` trait, but you bring your own **runtime** — **tokio** is the de-facto standard in 2026 (with `async-std` largely deprecated). Covered in §13.
> - **No garbage collector.** Memory is freed deterministically when values go out of scope (RAII). This is the whole point — covered in §4.
>
> Where behaviour is version- or ecosystem-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**, so cross-platform notes (path separators, `.exe`, shells) are called out. Confirm exact APIs at doc.rust-lang.org/std and docs.rs.

---

## Table of Contents

1. [Why Rust, Install, Cargo & the Toolchain](#1-why-rust-install-cargo--the-toolchain) **[B]**
2. [Basics: Variables, Types, Functions, Expressions](#2-basics-variables-types-functions-expressions) **[B]**
3. [Control Flow & Pattern Matching](#3-control-flow--pattern-matching) **[B]**
4. [Ownership — The Core Concept](#4-ownership--the-core-concept) **[B/I]**
5. [Borrowing & References](#5-borrowing--references) **[B/I]**
6. [Lifetimes](#6-lifetimes) **[I/A]**
7. [Structs, Enums, Option & Result](#7-structs-enums-option--result) **[B/I]**
8. [Error Handling](#8-error-handling) **[I]**
9. [Generics, Traits & Polymorphism](#9-generics-traits--polymorphism) **[I/A]**
10. [Collections, Strings, Iterators & Closures](#10-collections-strings-iterators--closures) **[I]**
11. [Smart Pointers](#11-smart-pointers) **[I/A]**
12. [Modules, Crates & Cargo](#12-modules-crates--cargo) **[I]**
13. [Concurrency & Async](#13-concurrency--async) **[A]**
14. [File System, OS Info & Command Execution](#14-file-system-os-info--command-execution) **[I]**
15. [Macros & the Type System](#15-macros--the-type-system) **[A]**
16. [Testing](#16-testing) **[I]**
17. [Tooling & Ecosystem](#17-tooling--ecosystem) **[I]**
18. [Gotchas & Best Practices](#18-gotchas--best-practices) **[A]**
19. [Study Path & Build-to-Learn Projects](#19-study-path--build-to-learn-projects)

---

## 1. Why Rust, Install, Cargo & the Toolchain

### 1.1 The value proposition — why does Rust exist? **[B]**

Most programming languages sit on one side of a historic trade-off:

- **Languages with a garbage collector** (Python, Java, Go, JavaScript, C#) are memory-safe and pleasant: you allocate memory and forget about it; a runtime background process reclaims it later. The cost is a runtime, unpredictable pauses (the GC stops your program to collect), higher memory use, and less control over layout and performance.
- **Languages without a GC** (C, C++) give you full control and top-tier performance, but *you* are responsible for freeing memory. Mistakes here — use-after-free, double-free, buffer overflows, data races — are the source of the majority of security vulnerabilities in shipped software (Microsoft and Google have both reported ~70% of their CVEs trace to memory-safety bugs).

Rust's central claim is that **you don't have to choose**. It delivers:

1. **Memory safety without a garbage collector.** Rust proves at *compile time* that your program never uses memory after it's freed, never frees it twice, and never reads uninitialized memory — using the ownership/borrowing system (§4–§6). There is no runtime tracking and no GC pauses. When a value's owner goes out of scope, its memory is freed immediately and deterministically.
2. **Performance on par with C and C++.** Zero-cost abstractions: high-level constructs (iterators, generics, closures) compile down to the same machine code you'd write by hand. You pay no runtime overhead for safety — the checks happen during compilation.
3. **Fearless concurrency.** The *same* rules that make single-threaded code safe also make multi-threaded code safe. The compiler will refuse to compile code that could cause a **data race** (two threads touching the same data, at least one writing, without synchronization). This is enormous: in most languages data races are runtime bugs you discover in production; in Rust they are compile errors.

The price you pay is up front: you must learn ownership, and the borrow checker will reject programs that *might* be valid but that it can't prove are safe. The payoff is that once it compiles, an entire category of bugs simply cannot occur. This is why people say "if it compiles, it usually works."

Rust also brings modern conveniences that have nothing to do with safety but make it pleasant: an excellent built-in build tool and package manager (cargo), algebraic data types with exhaustive pattern matching (§3, §7), a powerful trait system (§9), no null and no exceptions (replaced by `Option` and `Result`, §7–§8), and first-class tooling (§17).

### 1.2 Install via rustup **[B]**

`rustup` is the official toolchain installer and version manager. It installs `rustc` (the compiler), `cargo` (the build tool), and the standard library, and lets you switch between stable/beta/nightly and add cross-compilation targets.

```bash
# Windows: download and run rustup-init.exe from https://rustup.rs
#   (you may first need the "Build Tools for Visual Studio" C++ linker — the installer prompts you)
# macOS / Linux:
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Verify the install:
rustc --version        # rustc 1.85.0 (or later)
cargo --version
rustup --version

# Managing toolchains:
rustup update                      # update to the latest stable
rustup toolchain install nightly   # add the nightly channel
rustup default stable              # set the default channel
rustup component add clippy rustfmt rust-analyzer   # add tools (usually preinstalled)
rustup target add wasm32-unknown-unknown            # add a cross-compile target
```

> **⚡ Windows note:** Rust on Windows uses the MSVC toolchain by default and needs a C++ linker (from Visual Studio Build Tools). There's also a `-gnu` toolchain if you prefer MinGW. Compiled binaries are `myprog.exe`. Paths use `\` but Rust's `Path` handles both; in source code use `/` in string literals or `PathBuf::join` (see §14).

### 1.3 Cargo — the one tool you'll use constantly **[B]**

Cargo creates projects, fetches dependencies, builds, runs, tests, and more. Learn these commands first:

```bash
cargo new myapp            # create a new binary project (with a git repo)
cargo new mylib --lib      # create a library project instead
cargo init                 # turn the current directory into a cargo project

cd myapp
cargo build                # compile (debug build -> target/debug/myapp)
cargo build --release      # optimized build (slower compile, fast binary -> target/release/)
cargo run                  # build + run the binary
cargo run -- arg1 arg2     # pass args to YOUR program (everything after -- )
cargo check                # type-check WITHOUT producing a binary — much faster; use this constantly
cargo test                 # build and run all tests
cargo doc --open           # build and open this crate's API docs in a browser

cargo add serde            # add a dependency (writes it into Cargo.toml, picks a version)
cargo add tokio --features full     # add with feature flags
cargo add serde --features derive
cargo remove serde         # remove a dependency
cargo update               # update dependencies within the semver constraints in Cargo.toml
```

A new project's layout:

```
myapp/
├── Cargo.toml        # project metadata + dependencies (the "manifest")
├── Cargo.lock        # exact resolved versions (commit this for binaries; usually not for libs)
├── .gitignore        # ignores /target by default
└── src/
    └── main.rs       # entry point: `fn main() {}`  (for a lib it's src/lib.rs)
```

`Cargo.toml`:

```toml
[package]
name = "myapp"
version = "0.1.0"
edition = "2024"          # the language edition (see 1.5)

[dependencies]
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
```

The first build downloads and compiles dependencies into `target/` and resolves exact versions into `Cargo.lock`. Subsequent builds are incremental.

### 1.4 Your first program **[B]**

```rust
// src/main.rs
// `fn main()` is the entry point of every binary. It takes no arguments here and returns ().
fn main() {
    // println! is a MACRO (the ! marks it), not a function — it does compile-time
    // format-string checking. {} is a placeholder filled by the argument.
    println!("Hello, world!");

    let name = "Ada";
    // You can also name captured variables directly inside the braces (Rust 2021+):
    println!("Hello, {name}!");          // Hello, Ada!
    println!("{} + {} = {}", 2, 3, 2 + 3); // positional args
}
```

```bash
cargo run        # Hello, world! / Hello, Ada! / 2 + 3 = 5
```

### 1.5 Editions **[B/I]**

An **edition** is a named set of language defaults (2015, 2018, 2021, 2024). Editions let Rust make breaking *syntactic/idiomatic* changes without breaking old code: each crate declares its edition in `Cargo.toml`, and the compiler honors it. Crucially, **crates of different editions interoperate freely** — your 2024 crate can use a library still on 2018. Editions never split the ecosystem the way Python 2 vs 3 did.

The 2024 edition (stable in 2026) refined async, closures capturing, `gen` blocks, and several smaller rules. For a new project just set `edition = "2024"` and `cargo fix --edition` will help migrate an old crate. Editions only matter at crate boundaries; the standard library and most features are shared across all editions.

---

## 2. Basics: Variables, Types, Functions, Expressions

### 2.1 Variables, mutability & shadowing **[B]**

In Rust, **variables are immutable by default**. This is a deliberate safety choice: immutability makes code easier to reason about and is required for some of the borrowing guarantees later. If you want to change a value after binding, you must opt in with `mut`.

```rust
fn main() {
    let x = 5;          // immutable binding
    // x = 6;           // ERROR: cannot assign twice to immutable variable `x`

    let mut y = 5;      // mutable binding (note `mut`)
    y = 6;              // OK
    println!("{y}");    // 6

    // Constants: always immutable, must have a type annotation, computed at compile time.
    // Convention: SCREAMING_SNAKE_CASE. Can be declared in any scope, including global.
    const MAX_POINTS: u32 = 100_000;   // underscores are digit separators
    println!("{MAX_POINTS}");
}
```

**Shadowing** is different from mutation: you can declare a *new* variable with the same name. The new binding shadows the old one. This is useful for transforming a value while keeping a tidy name, and it lets you change the *type* (which `mut` cannot):

```rust
fn main() {
    let spaces = "   ";          // spaces is a &str (string slice)
    let spaces = spaces.len();   // shadowed: now spaces is a usize (a number) — different type, fine
    println!("{spaces}");        // 3

    let n = 5;
    let n = n + 1;     // new binding, value 6
    let n = n * 2;     // new binding, value 12
    println!("{n}");   // 12
}
```

Why allow shadowing? Because each `let` creates a *fresh* immutable binding, you keep the benefits of immutability while still being able to refine a value step by step. With `mut` you'd be stuck with one type for the lifetime of the variable.

### 2.2 Scalar types **[B]**

Rust is **statically and strongly typed**: every value's type is known at compile time. Often the compiler **infers** the type, but it can always be annotated as `let name: Type = ...`.

```rust
fn main() {
    // Integers. Signed: i8 i16 i32 i64 i128 isize.  Unsigned: u8 ... u128 usize.
    // The number after the letter is the bit width. `isize`/`usize` match the
    // pointer size of the machine (64-bit on modern PCs) — used for indexing.
    let a: i32 = -42;          // i32 is the default integer type if none is inferred
    let b: u8 = 255;           // 0..=255
    let big: u64 = 18_000_000_000;
    let hex = 0xff;            // 255   (0x hex, 0o octal, 0b binary literals)
    let byte = b'A';           // 65    (a u8 byte literal)

    // Integer overflow: in DEBUG builds, overflow PANICS (crashes loudly).
    // In RELEASE builds it WRAPS (two's complement) unless you use checked ops.
    let x: u8 = 200;
    let y = x.checked_add(100);    // None  (would overflow) — returns Option
    let z = x.wrapping_add(100);   // 44    (explicitly wraps)
    let s = x.saturating_add(100); // 255   (clamps to max)
    println!("{:?} {z} {s}", y);

    // Floats: f32 and f64 (f64 is the default — IEEE-754 double).
    let pi: f64 = 3.14159;
    let e = 2.71828_f32;       // type suffix on the literal

    // Booleans and characters:
    let ok: bool = true;
    let ch: char = 'z';        // a `char` is a 4-byte Unicode scalar value, NOT a byte!
    let emoji: char = '🦀';     // valid — chars hold any Unicode scalar
    println!("{pi} {e} {ok} {ch} {emoji}");

    // Numeric casting is explicit with `as` (Rust never auto-converts numeric types):
    let i = 65;
    let f = i as f64;          // 65.0
    let c = i as u8 as char;   // 'A'
    println!("{f} {c}");
}
```

> **Why no implicit numeric conversion?** Silent widening/narrowing is a classic bug source (e.g. truncating a large number into a small type). Rust forces every conversion to be visible, so the cost and the risk are explicit in the code.

### 2.3 Compound types: tuples & arrays **[B]**

```rust
fn main() {
    // TUPLE: a fixed-length, ordered group of values of POSSIBLY DIFFERENT types.
    let person: (&str, i32, f64) = ("Ada", 36, 1.7);
    let (name, age, height) = person;   // destructuring
    println!("{name} {age} {height}");
    println!("{} {}", person.0, person.1);  // access by index with .N

    let unit = ();   // the "unit" type/value — an empty tuple. Means "no meaningful value".
                     // Functions that return nothing actually return ().
    let _ = unit;

    // ARRAY: a fixed-length sequence of the SAME type, stored on the STACK.
    let nums: [i32; 5] = [1, 2, 3, 4, 5];   // type is [element; length]
    let zeros = [0; 3];                      // [0, 0, 0]  — value; count
    println!("{}", nums[0]);                 // 1  (indexing)
    println!("{}", nums.len());              // 5
    // Out-of-bounds indexing PANICS at runtime (Rust checks every index — no buffer overruns):
    // println!("{}", nums[99]);             // panics: index out of bounds

    // For a GROWABLE sequence, use Vec (heap-allocated) — see §10.
}
```

The distinction matters: arrays have a length fixed at compile time and live on the stack; `Vec<T>` can grow and lives on the heap. Use arrays for small, fixed collections; `Vec` for everything else.

### 2.4 Functions, statements vs expressions **[B]**

This section contains a concept that trips up newcomers and that the rest of the language leans on heavily: **the difference between statements and expressions**.

- A **statement** performs an action and returns no value. `let x = 5;` is a statement. Statements end with `;`.
- An **expression** evaluates to a value. `5 + 6`, a function call, a `{ ... }` block, an `if`, and a `match` are all expressions.

The key rule: **a block `{ ... }` evaluates to its last expression, *if that expression has no trailing semicolon*.** Putting a semicolon turns the expression into a statement, which evaluates to `()`. This is why Rust functions usually have no `return` keyword — the last expression is the return value.

```rust
// Parameters MUST be type-annotated. The return type follows `->`.
fn add(a: i32, b: i32) -> i32 {
    a + b        // NO semicolon -> this is the return value of the function
}

fn add_with_return(a: i32, b: i32) -> i32 {
    return a + b;   // explicit early-return; needed for returning before the end
}

fn no_return() {    // no `->` means it returns the unit type ()
    println!("side effect only");
}

fn main() {
    println!("{}", add(2, 3));   // 5

    // A block is an expression. This binds the block's last expression to y:
    let y = {
        let a = 3;
        let b = 4;
        a * b      // no semicolon -> the block evaluates to 12
    };
    println!("{y}");   // 12

    // Add a semicolon and the block evaluates to () instead:
    let z = {
        let a = 3;
        a + 1;     // semicolon! block value is now ()
    };
    // z has type () here. This is a common beginner bug: an accidental ; at the
    // end of a function body makes it return () and the compiler complains about types.
    let _ = z;
}
```

Why does Rust work this way? Treating control-flow constructs as expressions makes the language composable: you can assign the result of an `if` or `match` directly to a variable (next section), avoiding the temporary-mutable-variable dance other languages need. The statement/expression rule is the mechanism that makes that work.

### 2.5 Comments, `println!` and `format!` **[B]**

```rust
fn main() {
    // line comment
    /* block comment */
    /// doc comment for the item BELOW it (used by `cargo doc`); supports Markdown
    //! doc comment for the ENCLOSING item (top of a module/file)

    let user = "Ada";
    let score = 95.5;

    // println! prints with a trailing newline; print! without. eprintln! goes to stderr.
    println!("{user} scored {score}");
    println!("{:.1}%", score);          // 95.5  — .1 = one decimal place
    println!("{:>10}", user);           // right-align in width 10
    println!("{:08.2}", score);         // 00095.50  — zero-padded, width 8, 2 decimals
    println!("{:#x}", 255);             // 0xff  — # adds the 0x/0b prefix
    println!("{:?}", (1, 2, 3));        // (1, 2, 3)  — {:?} is the Debug format
    println!("{:#?}", vec![1, 2, 3]);   // pretty-printed multi-line Debug

    // format! returns a String instead of printing — the standard way to build strings:
    let msg: String = format!("{user}: {score:.0}");
    println!("{msg}");                  // Ada: 96
}
```

`{}` uses the `Display` trait (human-facing); `{:?}` uses `Debug` (developer-facing). Your own types get `Debug` for free with `#[derive(Debug)]` (§7, §9).

---

## 3. Control Flow & Pattern Matching

### 3.1 `if` as an expression **[B]**

`if` works as you'd expect, but two things differ from many languages: the condition has **no parentheses** and **must be a `bool`** (Rust does not coerce numbers/strings to bool), and `if` is an **expression** so it can produce a value.

```rust
fn main() {
    let n = 7;
    if n % 2 == 0 {
        println!("even");
    } else if n % 3 == 0 {
        println!("divisible by 3");
    } else {
        println!("odd, not div by 3");
    }

    // if as an expression — both branches MUST yield the same type:
    let label = if n > 5 { "big" } else { "small" };
    println!("{label}");

    // This replaces the ternary `?:` operator (Rust has no ?:).
    let parity = if n % 2 == 0 { "even" } else { "odd" };
    println!("{parity}");
}
```

### 3.2 Loops: `loop`, `while`, `for`, ranges & labels **[B]**

```rust
fn main() {
    // `loop` runs forever until you `break`. Uniquely, `break` can return a VALUE,
    // making `loop` an expression — handy for retry-until-success patterns.
    let mut counter = 0;
    let result = loop {
        counter += 1;
        if counter == 10 {
            break counter * 2;   // break out AND yield 20
        }
    };
    println!("{result}");        // 20

    // while: loop while a condition holds.
    let mut n = 3;
    while n > 0 {
        println!("{n}");
        n -= 1;
    }

    // for: the workhorse. Iterates over anything that implements IntoIterator.
    for i in 0..5 {              // 0..5 is a Range: 0,1,2,3,4 (end-EXCLUSIVE)
        print!("{i} ");
    }
    println!();
    for i in 1..=5 {             // 1..=5 is inclusive: 1,2,3,4,5
        print!("{i} ");
    }
    println!();

    let items = ["a", "b", "c"];
    for item in &items {                       // iterate by reference (don't consume)
        print!("{item} ");
    }
    println!();
    for (idx, item) in items.iter().enumerate() {   // index + value
        println!("{idx}: {item}");
    }

    // Loop LABELS: name loops with 'label so break/continue can target outer loops.
    'outer: for x in 0..5 {
        for y in 0..5 {
            if x * y > 6 {
                println!("breaking at {x},{y}");
                break 'outer;     // breaks the OUTER loop, not just the inner
            }
        }
    }
}
```

### 3.3 `match` — pattern matching, a centerpiece **[B/I]**

`match` is one of Rust's most important features. It compares a value against a series of **patterns** and runs the arm of the first match. Two properties make it powerful: it is an **expression** (returns a value), and it is **exhaustive** — the compiler forces you to handle every possible case (or add a catch-all). Exhaustiveness is a safety feature: when you add a new variant to an enum later, the compiler points you at every `match` that needs updating.

```rust
fn main() {
    let n = 3;
    let name = match n {
        1 => "one",
        2 => "two",
        3 => "three",
        4 | 5 | 6 => "a few",      // | matches multiple values
        7..=10 => "several",        // ..= matches an inclusive range
        _ => "many",                // _ is the catch-all (required to be exhaustive)
    };
    println!("{name}");             // three

    // Binding with @ : test a range AND capture the value.
    let msg = match n {
        x @ 1..=5 => format!("small: {x}"),
        x => format!("other: {x}"),
    };
    println!("{msg}");

    // Destructuring tuples in patterns, with guards (extra `if` conditions):
    let point = (0, 7);
    match point {
        (0, 0) => println!("origin"),
        (x, 0) => println!("on x-axis at {x}"),
        (0, y) => println!("on y-axis at {y}"),
        (x, y) if x == y => println!("on diagonal"),   // `if` guard
        (x, y) => println!("at ({x}, {y})"),
    }
}
```

Patterns also destructure structs and enums (§7) and appear in `let`, function parameters, and `for`. We'll use `match` extensively with `Option`/`Result`.

### 3.4 `if let`, `let else`, `while let` **[I]**

When you only care about *one* pattern, a full `match` is verbose. `if let` is sugar for "match this one pattern, ignore the rest":

```rust
fn main() {
    let maybe: Option<i32> = Some(5);

    // Instead of a full match with a `_ => {}` arm:
    if let Some(x) = maybe {
        println!("got {x}");      // got 5
    } else {
        println!("nothing");
    }

    // `let else`: bind a pattern OR diverge (return/break/panic). Great for early exits,
    // because the bound variable stays in scope AFTER the block (unlike `if let`).
    fn parse(s: &str) -> i32 {
        let Ok(n) = s.parse::<i32>() else {
            return -1;            // must diverge in the else branch
        };
        n * 2                     // `n` is in scope here
    }
    println!("{}", parse("21"));  // 42
    println!("{}", parse("xx"));  // -1

    // `while let`: loop as long as the pattern keeps matching.
    let mut stack = vec![1, 2, 3];
    while let Some(top) = stack.pop() {   // pop returns Option; loop until it's None
        println!("popped {top}");
    }
}
```

---

## 4. Ownership — The Core Concept

This is the most important section in the guide. Ownership is the idea that makes Rust memory-safe without a garbage collector, and it underlies borrowing (§5), lifetimes (§6), and concurrency (§13). Take it slowly. If you understand this section, the rest of Rust falls into place.

### 4.1 The stack and the heap (necessary background) **[B/I]**

To understand ownership you need a mental model of where data lives.

- **The stack** is fast, ordered memory. Values are pushed on and popped off in LIFO order. Every value on the stack must have a **known, fixed size at compile time**. Function calls push a "stack frame" of their local variables; returning pops it. Integers, floats, bools, chars, and fixed-size arrays/tuples of those live on the stack.
- **The heap** is a large, less-organized region for data whose size is unknown at compile time or that must outlive the function that created it. To use the heap you *request* a chunk of a given size; the allocator finds space and returns a **pointer** to it. The pointer itself (a fixed-size address) can live on the stack, while the data it points to lives on the heap. A `String` and a `Vec<T>` work exactly this way: a small fixed-size "handle" (pointer + length + capacity) on the stack, pointing at a growable buffer on the heap.

Heap access is slower (you follow a pointer; data is scattered) and someone must eventually **free** heap memory. *Who frees it, and when* is the question every language must answer. C says "you, manually" (error-prone). Java/Python say "a garbage collector, eventually" (runtime cost). **Rust says: the owner frees it, automatically, when the owner goes out of scope.** That mechanism is ownership.

### 4.2 The three rules of ownership **[B/I]**

Memorize these. Everything else is a consequence.

1. **Each value in Rust has a single variable that is its *owner*.**
2. **There can be only one owner at a time.**
3. **When the owner goes out of scope, the value is dropped (freed).**

"Dropped" means Rust automatically calls a cleanup routine (`Drop`) — for a `String`, that frees its heap buffer. This is **RAII** (Resource Acquisition Is Initialization), the same idea as C++ destructors, but enforced by the compiler so you can't forget and can't double-free.

```rust
fn main() {
    {                          // s does not exist yet
        let s = String::from("hello");   // s OWNS a heap-allocated string
        println!("{s}");       // use s
    }                          // s goes out of scope HERE -> Rust calls drop(s),
                               // freeing the heap memory. Deterministic. No GC.
    // println!("{s}");        // ERROR: s no longer exists
}
```

### 4.3 Move semantics — the surprising part **[B/I]**

Here is the rule that surprises everyone coming from other languages. Consider:

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;            // what happens here?
    // println!("{s1}");    // ERROR: borrow of moved value: `s1`
    println!("{s2}");       // OK
}
```

In Python or Java, `s2 = s1` would make both names point at the same object. In C++, it might *copy* the whole string. Rust does neither by default. Recall that `s1` is a handle (pointer + length + capacity) on the stack pointing at a heap buffer. When you write `let s2 = s1`, Rust copies the *handle* (cheap), so now `s1` and `s2` both point at the same heap buffer.

But rule 2 says there can be only one owner. If both were owners, then when both go out of scope, Rust would try to free the same buffer **twice** — a classic, dangerous "double free" bug. Rust's solution: assigning `s1` to `s2` is a **move**. Ownership transfers to `s2`, and `s1` is **invalidated** — the compiler will not let you use it anymore. Now exactly one owner exists, and the buffer is freed exactly once.

This is why the error above says "borrow of *moved* value." The move is a compile-time bookkeeping operation; at runtime nothing is copied beyond the small handle. It is a zero-cost guarantee of safety.

**Moves also happen when you pass a value to a function or return it:**

```rust
fn takes_ownership(s: String) {       // s moves INTO this function
    println!("{s}");
}   // s dropped here

fn gives_ownership() -> String {
    String::from("made here")          // ownership MOVED OUT to the caller
}

fn main() {
    let a = String::from("hi");
    takes_ownership(a);                // a is MOVED into the function...
    // println!("{a}");                // ERROR: a was moved, no longer usable

    let b = gives_ownership();         // b now owns the returned String
    println!("{b}");
}
```

If a function needs a value but the caller wants to keep using it, you have two options: **borrow** it (pass a reference — §5, the usual choice) or **clone** it (§4.5, an explicit deep copy).

### 4.4 Copy types — why integers don't move **[B/I]**

You may have noticed that earlier code freely reused integers after assigning them. That's because simple scalar types implement the **`Copy`** trait. A `Copy` type is fully stored on the stack with a known size and *no heap resource to manage*, so duplicating it is trivial and harmless. For `Copy` types, assignment **copies** rather than moves, and the original stays valid.

```rust
fn main() {
    let x = 5;
    let y = x;          // x is Copy (i32) -> this COPIES; x is still valid
    println!("{x} {y}"); // 5 5  — both usable

    // Copy types: all integers, floats, bool, char, and tuples/arrays whose
    // elements are ALL Copy. Anything owning heap data (String, Vec) is NOT Copy.
}
```

The rule of thumb: if a type manages a resource (heap memory, a file handle, a socket), it is **not** `Copy`, and assignment moves it. If it's a plain bundle of stack values, it's usually `Copy`.

### 4.5 `Clone` — explicit deep copies **[B/I]**

When you genuinely need an independent duplicate of heap data, call `.clone()`. This is **explicit** on purpose: copying a heap buffer can be expensive, so Rust makes you ask for it so the cost is visible in the source.

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();      // deep copy: a NEW heap buffer with the same contents
    println!("{s1} {s2}");    // both valid — they own separate buffers
}
```

Beginners often sprinkle `.clone()` everywhere to silence the borrow checker. That works but can be wasteful (§18). The idiomatic fix is usually to *borrow* instead — which is the entire next section.

### 4.6 Why this gives safety without a GC — recap **[I]**

Pulling it together: because every value has exactly one owner, and the owner frees it deterministically at end of scope, Rust knows *exactly* when memory should be freed — at compile time, with no runtime tracking. Because moves invalidate the source, the same buffer can never be freed twice and can never be used after it's freed. Because these are compile-time rules, there is **zero runtime cost** and **no garbage collector**. Use-after-free, double-free, and memory leaks-by-forgetting are all eliminated as categories of bug. The price is the discipline of ownership — which feels restrictive at first and liberating once internalized.

---

## 5. Borrowing & References

Moving ownership everywhere would be painful: every function that just wants to *read* a string would consume it. **Borrowing** is the solution. Instead of transferring ownership, you lend out a **reference** — a pointer that the borrow checker tracks to guarantee it's always valid.

### 5.1 References: `&` and `&mut` **[B/I]**

A reference (`&T`) lets you *access* a value without *owning* it. Creating a reference is called **borrowing**. The owner keeps ownership; when the reference goes out of scope, nothing is freed (the reference didn't own anything).

```rust
fn length(s: &String) -> usize {   // &String = "a reference to a String" (borrowed, not owned)
    s.len()
}   // s goes out of scope, but it's just a reference — the String is NOT dropped

fn main() {
    let owner = String::from("hello");
    let len = length(&owner);          // &owner = "lend a reference to owner"
    println!("'{owner}' has length {len}");  // owner is STILL VALID and usable here
}
```

By default references are **immutable** — you can read through them but not modify. To modify, you need a **mutable reference**, `&mut T`, and the value being borrowed must itself be `mut`:

```rust
fn append_world(s: &mut String) {
    s.push_str(", world");        // mutate through the mutable reference
}

fn main() {
    let mut greeting = String::from("hello");   // must be `mut` to lend &mut
    append_world(&mut greeting);                // lend a MUTABLE reference
    println!("{greeting}");                     // hello, world
}
```

### 5.2 The borrowing rules — the heart of the borrow checker **[B/I]**

At any given time, for a particular piece of data, you may have **either**:

- **any number of immutable references (`&T`)**, OR
- **exactly one mutable reference (`&mut T`)**,

**but never both at once.** This is often summarized as "**shared XOR mutable**" or "many readers OR one writer."

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;        // immutable borrow — fine
    let r2 = &s;        // another immutable borrow — also fine (many readers)
    println!("{r1} {r2}");   // last use of r1 and r2

    // After r1/r2 are no longer used, we can take a mutable borrow:
    let r3 = &mut s;    // one writer — fine, because no live shared borrows remain
    r3.push('!');
    println!("{r3}");   // hello!

    // What's NOT allowed: a shared and a mutable borrow ALIVE at the same time:
    // let a = &s;
    // let b = &mut s;          // ERROR: cannot borrow `s` as mutable because it is
    // println!("{a} {b}");     //        also borrowed as immutable
}
```

> **⚡ NLL (non-lexical lifetimes):** A borrow lasts only until its *last use*, not until the end of the enclosing block. This is why `r1`/`r2` above can be "done" before `r3` starts. Older Rust tied borrows to lexical scope and rejected far more code; modern Rust is much more forgiving. If you hit a borrow error, the fix is often just to narrow where references are *used*.

### 5.3 Why these rules exist — they prevent data races and aliasing bugs **[I]**

The shared-XOR-mutable rule is not arbitrary; it eliminates an entire class of bugs:

- **Data races** (in concurrent code): a data race needs two things at once — simultaneous access from multiple places, and at least one of them mutating. The borrowing rule forbids exactly that combination, *at compile time*. That's the foundation of "fearless concurrency" (§13): the single-threaded rule, applied across threads, makes data races impossible.
- **Iterator invalidation / aliasing bugs** (even in single-threaded code): consider modifying a vector while iterating it, or holding a pointer into a buffer that gets reallocated. In C++ this is undefined behavior; in Rust the borrow checker refuses it, because iterating borrows the vector immutably while modifying it would need a mutable borrow.

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    // for x in &v {        // immutable borrow of v for the loop...
    //     v.push(*x);      // ERROR: cannot borrow v as mutable while iterating
    // }
    // The compiler stops you from invalidating the iterator. Collect or index instead.
    let doubled: Vec<i32> = v.iter().map(|x| x * 2).collect();
    v.extend(doubled);
    println!("{v:?}");      // [1, 2, 3, 2, 4, 6]
}
```

### 5.4 Dangling references are impossible **[I]**

In C, you can return a pointer to a local variable; the variable is freed when the function returns, leaving the pointer pointing at garbage (a "dangling pointer"). Rust's borrow checker makes this a compile error: a reference can never outlive the data it points to.

```rust
// fn dangle() -> &String {        // ERROR: missing lifetime / returns a reference to local
//     let s = String::from("hi");
//     &s                          // s is dropped at the end of the function...
// }                               // ...so this reference would dangle. Rejected at compile time.

fn no_dangle() -> String {
    let s = String::from("hi");
    s                              // FIX: move ownership OUT instead of returning a reference
}

fn main() {
    println!("{}", no_dangle());
}
```

This guarantee is enforced by **lifetimes** — the subject of the next section.

### 5.5 Slices — borrowing part of a collection **[I]**

A **slice** (`&[T]` for arrays/vecs, `&str` for strings) is a reference to a *contiguous range* of a collection — a borrowed view, not an owned copy. Slices store a pointer and a length. Because they're borrows, the borrow checker ensures the underlying data outlives the slice.

```rust
fn main() {
    let s = String::from("hello world");
    let hello: &str = &s[0..5];     // a string slice: borrows bytes 0..5 of s
    let world = &s[6..11];
    println!("{hello} {world}");     // hello world

    let nums = [10, 20, 30, 40, 50];
    let middle: &[i32] = &nums[1..4];   // slice of the array: [20, 30, 40]
    println!("{middle:?}");

    // Idiom: take &[T] / &str as parameters so the function accepts arrays, vecs,
    // string literals, and Strings alike (see §10.2 for why &str beats &String).
    fn sum(slice: &[i32]) -> i32 {
        slice.iter().sum()
    }
    println!("{}", sum(&nums));          // pass an array
    println!("{}", sum(&vec![1, 2, 3])); // pass a Vec (coerces to a slice)
}
```

---

## 6. Lifetimes

Lifetimes are the part of Rust people fear most, usually because they misunderstand what lifetimes *are*. Let's fix the mental model first.

### 6.1 What lifetimes actually are **[I/A]**

A **lifetime** is the span of program execution during which a reference is valid. Every reference has a lifetime. Most of the time the compiler figures it out and you never write a thing. **Lifetime annotations do not change how long anything lives** — they don't extend or shorten any value's life. They are purely *descriptive*: they tell the compiler about the *relationships* between the lifetimes of several references, so it can verify that no reference outlives the data it points to.

Think of it this way: the borrow checker already enforces "no dangling references" (§5.4). When references flow through function boundaries or get stored in structs, the compiler sometimes can't *prove* the relationship on its own — so you annotate it. The annotation is a promise you make and the compiler checks; it's not a runtime mechanism.

### 6.2 Why they exist — the ambiguity they resolve **[I/A]**

Consider a function returning the longer of two string slices:

```rust
// fn longest(a: &str, b: &str) -> &str {   // ERROR: missing lifetime specifier
//     if a.len() > b.len() { a } else { b }
// }
```

The compiler rejects this. Why? The returned reference borrows from either `a` or `b`, but the compiler can't tell *which* from the signature alone — and it needs to know, so it can verify the caller doesn't use the result after the chosen input is gone. The lifetime annotation supplies the missing relationship:

```rust
// 'a is a lifetime PARAMETER, like a generic type parameter but for lifetimes.
// This signature says: "a, b, and the return value all share the lifetime 'a —
// i.e. the result is valid only as long as BOTH inputs are valid."
fn longest<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() > b.len() { a } else { b }
}

fn main() {
    let s1 = String::from("longer string");
    let result;
    {
        let s2 = String::from("short");
        result = longest(&s1, &s2);
        println!("{result}");        // OK: used while both s1 and s2 are alive
    }
    // println!("{result}");         // ERROR: s2 dropped, so result might dangle.
                                     // The annotation let the compiler catch this.
}
```

You didn't change any value's actual lifetime; you described the relationship so the compiler could enforce safety at the call sites.

### 6.3 Lifetime elision — why you rarely write them **[I]**

If lifetimes were always required, Rust would be unbearable. They're not, thanks to **elision rules**: the compiler applies three rules to infer lifetimes in common cases, and only when they fail to determine everything do you write annotations.

The rules (you don't need to memorize, just know they exist):
1. Each input reference gets its own distinct lifetime.
2. If there is exactly **one** input lifetime, it is assigned to all output references.
3. If one of the inputs is `&self` or `&mut self` (a method), `self`'s lifetime is assigned to all outputs.

These cover the vast majority of functions, which is why you can write tons of Rust before ever typing `'a`. The `longest` function needs annotations only because it has *two* input references and rule 2 doesn't apply.

```rust
// Elision rule 2 applies: one input lifetime -> output gets it. No annotation needed:
fn first_word(s: &str) -> &str {
    s.split_whitespace().next().unwrap_or("")
}

fn main() {
    println!("{}", first_word("hello world"));   // hello
}
```

### 6.4 Lifetimes in structs **[I/A]**

When a struct *holds a reference* (rather than owning its data), it must declare a lifetime, promising it won't outlive what it borrows:

```rust
// This struct borrows a string slice. The 'a says: an Excerpt cannot outlive the
// &str it holds — the compiler enforces it.
struct Excerpt<'a> {
    part: &'a str,
}

impl<'a> Excerpt<'a> {
    fn part(&self) -> &str {     // elision rule 3: output borrows from &self — no explicit 'a
        self.part
    }
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first = novel.split('.').next().unwrap();
    let e = Excerpt { part: first };       // e borrows from `novel`
    println!("{}", e.part());
    // `novel` must stay alive as long as `e` does — guaranteed by 'a.
}
```

### 6.5 `'static` and when you actually annotate **[A]**

`'static` is a special lifetime meaning "valid for the entire program." String literals (`&'static str`) have it because they're baked into the binary. Don't slap `'static` on things to silence errors — it usually masks a design problem.

**When you actually write lifetimes:** returning a reference derived from multiple reference inputs (like `longest`); storing references in structs/enums; some trait bounds. **When you don't:** the overwhelming majority of everyday code, where elision handles it or where you own your data (return `String`, not `&str`, when in doubt). If lifetimes feel hard, the pragmatic move is often to own data rather than borrow it across boundaries.

---

## 7. Structs, Enums, Option & Result

### 7.1 Structs **[B]**

A `struct` groups related data under named fields. It's Rust's primary tool for modeling domain types.

```rust
// A classic named-field struct.
#[derive(Debug)]               // auto-implement Debug so we can println!("{:?}", ...)
struct User {
    username: String,
    email: String,
    active: bool,
    sign_in_count: u64,
}

fn main() {
    let mut user = User {
        username: String::from("ada"),
        email: String::from("ada@example.com"),
        active: true,
        sign_in_count: 1,
    };
    user.email = String::from("new@example.com");   // mutate (whole struct must be `mut`)
    println!("{}", user.username);
    println!("{user:?}");        // Debug print of the whole struct

    // Struct update syntax: copy remaining fields from another instance.
    let user2 = User {
        email: String::from("ada2@example.com"),
        ..user                   // take the rest from `user` (moves non-Copy fields!)
    };
    println!("{user2:?}");

    // Tuple structs: fields by position, no names. Good for lightweight wrappers.
    struct Point(i32, i32);
    let origin = Point(0, 0);
    println!("{} {}", origin.0, origin.1);

    // Unit struct: no fields. Useful as a marker or for implementing traits.
    struct Marker;
    let _m = Marker;
}
```

### 7.2 Methods and `impl` blocks **[B]**

Behavior lives in `impl` blocks. Methods take `self` in one of three forms, which is just ownership/borrowing again: `&self` (borrow to read — most common), `&mut self` (borrow to mutate), or `self` (consume the value). Functions without a `self` parameter are **associated functions** (like static methods); `new` constructors are the convention.

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    // Associated function (no self) — a constructor, called as Rectangle::new(..)
    fn new(width: u32, height: u32) -> Self {     // `Self` = the type being impl'd
        Rectangle { width, height }                // field init shorthand (name == var)
    }

    fn area(&self) -> u32 {            // &self: borrow to read
        self.width * self.height
    }

    fn scale(&mut self, factor: u32) { // &mut self: borrow to mutate
        self.width *= factor;
        self.height *= factor;
    }

    fn into_square(self) -> Rectangle {   // self: consumes the rectangle
        let side = self.width.max(self.height);
        Rectangle::new(side, side)
    }
}

fn main() {
    let mut r = Rectangle::new(3, 4);
    println!("area = {}", r.area());   // 12  — method-call syntax with .
    r.scale(2);
    println!("{r:?}");                  // Rectangle { width: 6, height: 8 }
    let sq = r.into_square();           // r is consumed here
    println!("{sq:?}");
}
```

### 7.3 Enums — and how they carry data **[B/I]**

An `enum` defines a type by enumerating its possible **variants**. Unlike enums in C, Rust enum variants can each **carry data** of different shapes. This makes enums Rust's tool for "a value that is one of several alternatives" — a *sum type* / *tagged union*. Combined with `match`, they model state and choice precisely and safely.

```rust
// Each variant can hold different data (or none):
#[derive(Debug)]
enum Message {
    Quit,                          // no data
    Move { x: i32, y: i32 },       // named fields, like a struct
    Write(String),                 // a single String
    ChangeColor(u8, u8, u8),       // a tuple of three u8s
}

impl Message {
    fn describe(&self) -> String {
        // match destructures each variant to get at its data:
        match self {
            Message::Quit => "quit".to_string(),
            Message::Move { x, y } => format!("move to ({x}, {y})"),
            Message::Write(text) => format!("write: {text}"),
            Message::ChangeColor(r, g, b) => format!("color #{r:02x}{g:02x}{b:02x}"),
        }
    }
}

fn main() {
    let messages = [
        Message::Quit,
        Message::Move { x: 10, y: 20 },
        Message::Write(String::from("hi")),
        Message::ChangeColor(255, 128, 0),
    ];
    for m in &messages {
        println!("{}", m.describe());
    }
}
```

### 7.4 `Option<T>` — replacing null **[B/I]**

Rust has **no null**. The billion-dollar mistake (null reference) is designed out. Instead, a value that might be absent has type `Option<T>`, an enum with two variants: `Some(value)` or `None`. Because it's a distinct type, the compiler *forces* you to handle the absent case before you can use the value — you cannot accidentally dereference "null."

```rust
fn main() {
    // Option<T> { Some(T), None }
    let some_number: Option<i32> = Some(5);
    let no_number: Option<i32> = None;

    // You can't use an Option as if it were the inner value — you must unwrap it safely:
    // let x = some_number + 1;     // ERROR: Option<i32> is not i32

    // Handle both cases with match:
    let result = match some_number {
        Some(n) => n + 1,
        None => 0,
    };
    println!("{result}");      // 6

    // Common helper methods (cleaner than match for simple cases):
    println!("{}", some_number.unwrap_or(0));         // 5  (default if None)
    println!("{}", no_number.unwrap_or(0));           // 0
    println!("{}", some_number.unwrap_or_else(|| 99)); // 5  (lazy default via closure)
    println!("{:?}", some_number.map(|n| n * 10));    // Some(50)  (transform if Some)
    println!("{}", some_number.is_some());            // true

    // .unwrap() / .expect() extract the value but PANIC if None — use only when you're
    // certain, or in prototypes/tests (see §18 on unwrap in production).
    let config: Option<&str> = Some("debug");
    println!("{}", config.expect("config must be set"));  // debug
}
```

### 7.5 `Result<T, E>` — replacing exceptions **[B/I]**

Rust has **no exceptions** for recoverable errors. A fallible operation returns `Result<T, E>`: either `Ok(value)` on success or `Err(error)` on failure. Like `Option`, it's a normal enum, so error handling is explicit and type-checked — you can't forget to handle an error because the type system tracks it. (Full treatment in §8.)

```rust
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err(String::from("division by zero"))   // failure path
    } else {
        Ok(a / b)                                // success path
    }
}

fn main() {
    match divide(10.0, 2.0) {
        Ok(v) => println!("ok: {v}"),       // ok: 5
        Err(e) => println!("err: {e}"),
    }
    match divide(1.0, 0.0) {
        Ok(v) => println!("ok: {v}"),
        Err(e) => println!("err: {e}"),     // err: division by zero
    }
}
```

> **Why is this better than null/exceptions?** Null lets absence sneak into any reference; exceptions let errors propagate invisibly until they crash somewhere far away. `Option` and `Result` make both *visible in the type signature* and *impossible to ignore silently* — the compiler is your safety net.

---

## 8. Error Handling

### 8.1 Two kinds of errors **[I]**

Rust splits errors into two philosophies:

- **Unrecoverable errors** — bugs and impossible states. Use `panic!`, which unwinds the stack and aborts the thread (and usually the program). A panic means "this should never happen; stop now." Out-of-bounds indexing, failed `unwrap()`, and explicit `panic!("...")` are panics.
- **Recoverable errors** — expected failure conditions (file not found, bad input, network down). Use `Result<T, E>`. The caller decides what to do.

```rust
fn main() {
    // panic! — for truly unrecoverable situations:
    // panic!("the database is on fire");  // aborts with a message + backtrace

    // RUST_BACKTRACE=1 cargo run  shows a full backtrace on panic.

    // Recoverable: prefer Result and let the caller choose.
    let nums = vec![1, 2, 3];
    match nums.get(10) {            // .get returns Option instead of panicking
        Some(n) => println!("{n}"),
        None => println!("out of range"),   // graceful
    }
}
```

### 8.2 The `?` operator — ergonomic propagation **[I]**

Manually matching every `Result` to bubble errors up is tedious. The **`?` operator** does it for you: applied to a `Result`, it returns the `Ok` value, or **early-returns the `Err` from the current function**. It works in any function whose return type is a compatible `Result` (or `Option`). This is the backbone of idiomatic Rust error handling.

```rust
use std::fs;
use std::io;

// Without ? — verbose:
fn read_verbose(path: &str) -> Result<String, io::Error> {
    let result = fs::read_to_string(path);
    match result {
        Ok(content) => Ok(content),
        Err(e) => Err(e),
    }
}

// With ? — the same thing, concise. ? unwraps Ok or returns the Err.
fn read_concise(path: &str) -> Result<String, io::Error> {
    let content = fs::read_to_string(path)?;   // on Err, returns it from the function
    Ok(content.trim().to_string())
}

// ? chains beautifully across multiple fallible calls:
fn first_line_len(path: &str) -> Result<usize, io::Error> {
    let content = fs::read_to_string(path)?;
    let first = content.lines().next().unwrap_or("");
    Ok(first.len())
}

fn main() {
    match read_concise("Cargo.toml") {
        Ok(c) => println!("read {} chars", c.len()),
        Err(e) => println!("error: {e}"),
    }
    let _ = read_verbose;       // silence "unused" for the example
    let _ = first_line_len;
}
```

`main` itself can return `Result`, so `?` works at the top level too:

```rust
use std::error::Error;

fn main() -> Result<(), Box<dyn Error>> {     // Box<dyn Error> = "any error type"
    let content = std::fs::read_to_string("Cargo.toml")?;
    println!("{} bytes", content.len());
    Ok(())                                     // success
}
```

### 8.3 Custom error types **[I/A]**

For libraries, define your own error enum so callers can match on specific failures. Implement `Display` (human message) and `std::error::Error` (the standard error trait) so it composes with `?` and `Box<dyn Error>`.

```rust
use std::fmt;

#[derive(Debug)]
enum ConfigError {
    NotFound(String),
    Invalid { field: String, reason: String },
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            ConfigError::NotFound(path) => write!(f, "config not found: {path}"),
            ConfigError::Invalid { field, reason } =>
                write!(f, "invalid field '{field}': {reason}"),
        }
    }
}

impl std::error::Error for ConfigError {}   // marks it as a standard error (uses Debug+Display)

fn load(path: &str) -> Result<String, ConfigError> {
    if path.is_empty() {
        return Err(ConfigError::NotFound(path.to_string()));
    }
    if !path.ends_with(".toml") {
        return Err(ConfigError::Invalid {
            field: "path".into(),
            reason: "must end with .toml".into(),
        });
    }
    Ok(format!("contents of {path}"))
}

fn main() {
    println!("{:?}", load(""));            // Err(NotFound(""))
    println!("{:?}", load("a.json"));      // Err(Invalid { ... })
    println!("{:?}", load("a.toml"));      // Ok("contents of a.toml")
}
```

### 8.4 `thiserror` and `anyhow` — the ecosystem standard **[I/A]**

Hand-writing `Display`/`Error` impls is boilerplate. Two crates dominate in 2026:

- **`thiserror`** — for **libraries**. A derive macro that generates the `Display`/`Error` boilerplate for your custom error enum. You keep precise, matchable error types with almost no code.
- **`anyhow`** — for **applications/binaries**. Provides `anyhow::Result<T>` (a `Result<T, anyhow::Error>`) that can hold *any* error, plus `.context(...)` to attach human-readable context. You don't care about matching exact types at the top of a binary — you just want a good error message and a backtrace.

```bash
cargo add thiserror anyhow
```

```rust
// --- library style with thiserror ---
use thiserror::Error;

#[derive(Error, Debug)]
enum DataError {
    #[error("file not found: {0}")]            // the format string IS the Display impl
    NotFound(String),
    #[error("parse failed at line {line}")]
    Parse { line: usize },
    #[error(transparent)]                       // wrap another error, forwarding its Display
    Io(#[from] std::io::Error),                 // #[from] auto-converts io::Error via ?
}

fn read_data(path: &str) -> Result<String, DataError> {
    let content = std::fs::read_to_string(path)?;   // io::Error auto-converts to DataError::Io
    if content.is_empty() {
        return Err(DataError::NotFound(path.to_string()));
    }
    Ok(content)
}

// --- application style with anyhow ---
use anyhow::{Context, Result};

fn run() -> Result<()> {
    let content = std::fs::read_to_string("config.toml")
        .context("failed to read config.toml")?;   // attach context to ANY error
    println!("{} bytes", content.len());
    Ok(())
}

fn main() {
    let _ = read_data;
    if let Err(e) = run() {
        // anyhow prints the error and its full context chain:
        eprintln!("error: {e:?}");
    }
}
```

Rule of thumb: **`thiserror` for libraries** (callers need typed errors), **`anyhow` for binaries** (you just want it to work and report well).

---

## 9. Generics, Traits & Polymorphism

### 9.1 Generics **[I]**

Generics let you write code that works for many types without duplication, while staying fully type-checked and zero-cost (the compiler generates a specialized version per concrete type — "monomorphization"). Type parameters go in angle brackets.

```rust
// A generic function: works for any T that can be compared with >.
// The `T: PartialOrd` is a TRAIT BOUND — it constrains T to comparable types.
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    let mut max = list[0];
    for &item in list {
        if item > max { max = item; }
    }
    max
}

// A generic struct: Point can hold any type for its coordinates.
struct Point<T> {
    x: T,
    y: T,
}

impl<T: std::fmt::Display> Point<T> {   // methods can require their own bounds
    fn show(&self) {
        println!("({}, {})", self.x, self.y);
    }
}

fn main() {
    println!("{}", largest(&[3, 7, 2, 9, 4]));          // 9   (T = i32)
    println!("{}", largest(&[1.5, 2.7, 0.3]));          // 2.7 (T = f64)
    Point { x: 1, y: 2 }.show();                        // (1, 2)
    Point { x: "a", y: "b" }.show();                    // (a, b)
}
```

### 9.2 Traits — Rust's interfaces **[I]**

A **trait** defines shared behavior: a set of method signatures that a type can implement. Traits are how Rust does polymorphism — they're like interfaces (Java) or protocols (Swift), but more powerful. If a type implements a trait, it can be used anywhere that trait is required. Traits can also provide **default method implementations**.

```rust
// Define a trait: any "Summary" type must provide summarize().
trait Summary {
    fn summarize(&self) -> String;          // required method (no body)

    fn preview(&self) -> String {           // default method (has a body)
        format!("{}...", &self.summarize()[..self.summarize().len().min(10)])
    }
}

struct Article { title: String, body: String }
struct Tweet { user: String, text: String }

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{}: {}", self.title, self.body)
    }
    // we accept the default preview()
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("@{}: {}", self.user, self.text)
    }
    fn preview(&self) -> String {            // override the default
        format!("tweet by @{}", self.user)
    }
}

fn main() {
    let a = Article { title: "Rust".into(), body: "is great".into() };
    let t = Tweet { user: "ada".into(), text: "loving it".into() };
    println!("{}", a.summarize());   // Rust: is great
    println!("{}", a.preview());     // Rust: is g...   (default)
    println!("{}", t.preview());     // tweet by @ada   (overridden)
}
```

> **Orphan rule:** you can implement a trait for a type only if *either* the trait *or* the type is defined in your crate. This prevents two crates from giving conflicting impls for the same (trait, type) pair.

### 9.3 Trait bounds and `impl Trait` **[I]**

Trait bounds constrain generic parameters to types that have the needed behavior. `impl Trait` in argument position is shorthand for a generic; in return position it means "returns *some* concrete type implementing this trait" (without naming it).

```rust
use std::fmt::Display;

// These three signatures are equivalent ways to say "any T that is Display + Clone":
fn show1<T: Display + Clone>(x: T) { println!("{x}"); }
fn show2<T>(x: T) where T: Display + Clone { println!("{x}"); }  // `where` clause: cleaner for many bounds
fn show3(x: impl Display + Clone) { println!("{x}"); }            // impl Trait argument

// Return position impl Trait: "returns something that implements Iterator", details hidden.
fn even_numbers(max: u32) -> impl Iterator<Item = u32> {
    (0..max).filter(|n| n % 2 == 0)
}

fn main() {
    show1(42); show2("hi"); show3(3.14);
    let evens: Vec<u32> = even_numbers(10).collect();
    println!("{evens:?}");        // [0, 2, 4, 6, 8]
}
```

### 9.4 Derive macros **[I]**

Common traits can be auto-implemented with `#[derive(...)]`, saving enormous boilerplate. The compiler generates the obvious impl based on your fields.

```rust
// Each derive generates a standard implementation:
#[derive(Debug,        // {:?} formatting
         Clone,         // .clone() deep copy
         PartialEq, Eq, // == comparison
         Hash,          // usable as a HashMap key
         Default,       // T::default() zero/empty value
         PartialOrd, Ord)] // ordering / sorting
struct Version {
    major: u32,
    minor: u32,
    patch: u32,
}

fn main() {
    let v1 = Version { major: 1, minor: 2, patch: 0 };
    let v2 = v1.clone();
    println!("{v1:?} == {v2:?} -> {}", v1 == v2);    // equal -> true
    let zero = Version::default();                    // all-zero via Default
    println!("{zero:?}");

    let mut versions = vec![
        Version { major: 2, minor: 0, patch: 0 },
        Version { major: 1, minor: 5, patch: 3 },
    ];
    versions.sort();                                  // uses derived Ord
    println!("{versions:?}");
}
```

### 9.5 Static vs dynamic dispatch: generics vs `dyn Trait` **[I/A]**

There are two ways to write polymorphic code, and the distinction matters for performance and flexibility:

- **Static dispatch (generics / `impl Trait`):** the compiler generates a specialized copy of the code for each concrete type (monomorphization). Calls are resolved at compile time — **fast, no runtime overhead**, but code can bloat and you can't mix different concrete types in one collection.
- **Dynamic dispatch (`dyn Trait`, "trait objects"):** one copy of the code; the concrete type is determined at runtime via a *vtable* (a table of function pointers). The call has a small indirection cost, but you gain flexibility — e.g. a `Vec<Box<dyn Trait>>` can hold *different* concrete types implementing the same trait.

```rust
trait Shape {
    fn area(&self) -> f64;
}
struct Circle { r: f64 }
struct Square { s: f64 }
impl Shape for Circle { fn area(&self) -> f64 { std::f64::consts::PI * self.r * self.r } }
impl Shape for Square { fn area(&self) -> f64 { self.s * self.s } }

// STATIC dispatch: monomorphized; one T per call site. Can't mix types.
fn print_area<T: Shape>(shape: &T) {
    println!("{:.2}", shape.area());
}

fn main() {
    print_area(&Circle { r: 1.0 });
    print_area(&Square { s: 2.0 });

    // DYNAMIC dispatch: a heterogeneous collection of "anything that is a Shape".
    // `dyn Shape` has no fixed size, so it must be behind a pointer (Box, &, Rc...).
    let shapes: Vec<Box<dyn Shape>> = vec![
        Box::new(Circle { r: 1.0 }),
        Box::new(Square { s: 3.0 }),
    ];
    let total: f64 = shapes.iter().map(|s| s.area()).sum();   // vtable call per element
    println!("total area = {total:.2}");
}
```

Rule of thumb: prefer generics (static) for performance; reach for `dyn Trait` when you need a heterogeneous collection or want to keep compile times/binary size down.

### 9.6 Associated types **[A]**

A trait can declare an **associated type** — a placeholder type that the implementor fills in. The canonical example is `Iterator`, whose `Item` type is associated rather than a generic parameter. Associated types are used when there's exactly *one* sensible type per implementor (an iterator yields one kind of item).

```rust
// A custom iterator. `type Item` is the associated type — each impl picks it.
struct Counter { count: u32, max: u32 }

impl Iterator for Counter {
    type Item = u32;                       // this iterator yields u32s

    fn next(&mut self) -> Option<Self::Item> {
        if self.count < self.max {
            self.count += 1;
            Some(self.count)
        } else {
            None                            // None ends iteration
        }
    }
}

fn main() {
    let c = Counter { count: 0, max: 5 };
    // Because we implemented Iterator, ALL the adaptor methods work for free:
    let sum: u32 = c.map(|x| x * 2).filter(|x| x > &4).sum();
    println!("{sum}");        // (6 + 8 + 10) = 24
}
```

---

## 10. Collections, Strings, Iterators & Closures

### 10.1 `Vec<T>` — the growable array **[I]**

`Vec<T>` is the workhorse collection: a heap-allocated, growable list of values of one type.

```rust
fn main() {
    let mut v: Vec<i32> = Vec::new();   // empty, type annotated
    v.push(1);
    v.push(2);
    v.push(3);

    let mut v2 = vec![10, 20, 30];      // vec! macro to initialize

    // Access: indexing PANICS if out of range; .get returns Option (safe).
    println!("{}", v2[0]);              // 10
    println!("{:?}", v2.get(99));       // None

    v2[1] = 99;                         // mutate by index (v2 must be mut)
    let last = v2.pop();                // remove & return last -> Option
    println!("{last:?} {v2:?}");        // Some(30) [10, 99]

    // Iterate:
    for x in &v { print!("{x} "); }     // by ref (don't consume)
    println!();
    for x in &mut v { *x *= 10; }       // mutable ref: *x dereferences to assign
    println!("{v:?}");                  // [10, 20, 30]

    println!("{}", v.len());            // 3
    println!("{}", v.contains(&20));    // true
    v.extend([40, 50]);                 // append another iterable
    v.retain(|&x| x > 20);              // keep only matching elements
    println!("{v:?}");                  // [30, 40, 50]
}
```

### 10.2 `String` vs `&str` — the crucial distinction **[I]**

This confuses every newcomer, so let's be precise. There are two main string types and they play different roles:

- **`String`** is an **owned**, growable, heap-allocated UTF-8 buffer. You own it; you can mutate and grow it; it's dropped when its owner goes out of scope. Think "I own this text and may change it."
- **`&str`** (a "string slice") is a **borrowed**, immutable *view* into UTF-8 text owned by someone else (a `String`, or a string literal baked into the binary). It's just a pointer + length. Think "I'm looking at some text I don't own."

A **string literal** like `"hello"` has type `&'static str` — it lives in the program binary for the whole run. A `String` is what you build at runtime.

**The practical guideline:** take `&str` as a function *parameter* (it accepts both `&str` and `&String` via auto-coercion, so it's maximally flexible), and return/store `String` when you need ownership.

```rust
fn main() {
    let literal: &str = "hello";              // &'static str — baked into the binary
    let owned: String = String::from("hello"); // owned, heap, growable
    let owned2 = "world".to_string();          // another way to make a String

    let mut s = String::from("foo");
    s.push_str("bar");        // append a &str
    s.push('!');              // append a single char
    let combined = s + "?";   // + consumes the left String, borrows the right &str
    println!("{combined}");   // foobar!?

    // Borrow a slice of an owned String:
    let slice: &str = &owned[0..3];   // "hel"
    println!("{slice}");

    // Parameter idiom: take &str so callers can pass anything text-like.
    fn shout(text: &str) -> String {
        text.to_uppercase()
    }
    println!("{}", shout("hi"));        // pass a literal &str
    println!("{}", shout(&owned));      // pass a &String (coerces to &str)

    // UTF-8 caveat: you can't index a string by integer (s[0]) because a "character"
    // may be multiple bytes. Iterate explicitly instead:
    for ch in "héllo".chars() { print!("{ch} "); }   // by Unicode scalar
    println!();
    println!("{}", "héllo".len());        // 6 BYTES (é is 2 bytes), not 5 chars!
    println!("{}", "héllo".chars().count()); // 5 chars
    let _ = (literal, owned2);
}
```

### 10.3 `HashMap<K, V>` **[I]**

A hash map stores key→value pairs with average O(1) lookup. Keys must implement `Eq + Hash`.

```rust
use std::collections::HashMap;

fn main() {
    let mut scores: HashMap<String, i32> = HashMap::new();
    scores.insert("blue".to_string(), 10);
    scores.insert("red".to_string(), 50);

    // Lookup returns Option<&V> (the key might be absent):
    if let Some(s) = scores.get("blue") {
        println!("blue = {s}");           // blue = 10
    }
    println!("{}", scores.get("green").copied().unwrap_or(0));  // 0 (missing -> default)

    // entry API: insert-if-absent, or update in place. The classic "count" pattern:
    let text = "the cat the dog the bird";
    let mut counts: HashMap<&str, i32> = HashMap::new();
    for word in text.split_whitespace() {
        *counts.entry(word).or_insert(0) += 1;   // get-or-create the count, then increment
    }
    println!("{counts:?}");    // {"the": 3, "cat": 1, "dog": 1, "bird": 1} (order varies)

    for (k, v) in &scores {    // iterate (no guaranteed order)
        println!("{k}: {v}");
    }
}
```

> Related collections in `std::collections`: `BTreeMap`/`BTreeSet` (sorted), `HashSet`, `VecDeque` (double-ended queue), `BinaryHeap` (priority queue).

### 10.4 Iterators — lazy, composable, zero-cost **[I]**

Iterators are central to idiomatic Rust. An iterator is anything implementing the `Iterator` trait (one method: `next() -> Option<Item>`). The power comes from **adaptor** methods (`map`, `filter`, `take`, etc.) that transform one iterator into another. Critically, **adaptors are lazy**: they do nothing until a **consumer** (like `collect`, `sum`, `for`) drives the iteration. The whole chain compiles down to a tight loop with no allocation per step — a zero-cost abstraction.

```rust
fn main() {
    let nums = vec![1, 2, 3, 4, 5, 6];

    // A lazy pipeline. Nothing runs until .collect() consumes it.
    let result: Vec<i32> = nums
        .iter()                  // &i32 iterator over the vec (borrows it)
        .filter(|&&x| x % 2 == 0)// keep evens -> 2, 4, 6   (lazy)
        .map(|&x| x * 10)        // transform -> 20, 40, 60  (lazy)
        .collect();              // CONSUMER: runs the pipeline, builds a Vec
    println!("{result:?}");      // [20, 40, 60]

    // Consumers that produce a single value:
    let sum: i32 = nums.iter().sum();                 // 21
    let product: i32 = nums.iter().product();         // 720
    let max = nums.iter().max();                      // Some(6)
    let count = nums.iter().filter(|&&x| x > 3).count();  // 3
    println!("{sum} {product} {max:?} {count}");

    // Useful adaptors / consumers:
    let any_even = nums.iter().any(|&x| x % 2 == 0);  // true
    let all_pos = nums.iter().all(|&x| x > 0);        // true
    let first_big = nums.iter().find(|&&x| x > 4);    // Some(5)
    let total_with_index: usize = nums.iter().enumerate()
        .map(|(i, _)| i).sum();                       // 0+1+2+3+4+5 = 15
    println!("{any_even} {all_pos} {first_big:?} {total_with_index}");

    // Three ways to iterate a collection — choose based on ownership:
    //   .iter()       -> &T      (borrow each element; collection still usable after)
    //   .iter_mut()   -> &mut T  (mutably borrow each element)
    //   .into_iter()  -> T       (consume the collection, take ownership of elements)
    let words = vec!["a".to_string(), "b".to_string()];
    let joined: String = words.into_iter().collect::<Vec<_>>().join("-");  // consumes `words`
    println!("{joined}");        // a-b
}
```

### 10.5 Closures and the three closure traits **[I/A]**

A **closure** is an anonymous function that can **capture** variables from its surrounding scope. Syntax: `|params| body`. How a closure captures determines which of three traits it implements — and this matters because it controls how/where the closure can be called and stored.

- **`Fn`** — captures by *immutable reference* (`&`). Can be called many times; doesn't mutate or consume captures.
- **`FnMut`** — captures by *mutable reference* (`&mut`). Can be called many times and may mutate captured state.
- **`FnOnce`** — captures by *value* (moves captures in). Can be called *at least once*; calling it may consume the captures. Every closure implements at least `FnOnce`.

The compiler picks the *least restrictive* trait automatically based on what the closure body does. You only think about these when writing functions that *accept* closures.

```rust
fn main() {
    // Inferred parameter & return types; captures `factor` by reference (Fn):
    let factor = 3;
    let multiply = |x: i32| x * factor;
    println!("{}", multiply(10));      // 30

    // FnMut: mutates captured state across calls.
    let mut count = 0;
    let mut increment = || { count += 1; count };   // captures &mut count
    println!("{}", increment());       // 1
    println!("{}", increment());       // 2

    // `move` forces capture BY VALUE (the closure takes ownership). Essential for
    // threads/async where the closure outlives the current scope (see §13).
    let data = vec![1, 2, 3];
    let owns = move || println!("{data:?}");   // data is MOVED into the closure
    owns();
    // println!("{data:?}");           // ERROR: data was moved into `owns`

    // Functions accepting closures use the trait bounds:
    fn apply<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 { f(x) }     // Fn: callable repeatedly
    fn apply_once<F: FnOnce() -> String>(f: F) -> String { f() } // FnOnce: called once
    println!("{}", apply(|x| x + 1, 41));      // 42
    let s = String::from("hi");
    println!("{}", apply_once(move || s));     // moves s out, returns it
}
```

---

## 11. Smart Pointers

A **smart pointer** is a struct that acts like a pointer but adds behavior (and usually owns the data it points to). They let you opt into capabilities the basic ownership rules don't give you, each with a clear purpose. Know *which problem each one solves*.

### 11.1 `Box<T>` — heap allocation **[I]**

`Box<T>` puts a single value on the heap and owns it. Use it when: (a) you have a large value you want to move cheaply (only the pointer moves), (b) you need a type whose size isn't known at compile time — classically a **recursive type** — or (c) you want a trait object (`Box<dyn Trait>`, §9.5).

```rust
fn main() {
    let boxed: Box<i32> = Box::new(5);    // 5 lives on the heap; `boxed` owns it
    println!("{}", *boxed);                // 5  (deref with *)

    // Recursive type: a cons-list. Without Box this would be infinitely sized.
    #[derive(Debug)]
    enum List {
        Cons(i32, Box<List>),   // Box gives a known size (pointer) for the recursion
        Nil,
    }
    use List::*;
    let list = Cons(1, Box::new(Cons(2, Box::new(Cons(3, Box::new(Nil))))));
    println!("{list:?}");
}   // the whole heap chain is freed automatically here
```

### 11.2 `Rc<T>` and `Arc<T>` — shared ownership **[I/A]**

Sometimes a value needs **multiple owners** — e.g. a node in a graph referenced from several places — and "single owner" doesn't fit. `Rc<T>` ("reference counted") allows multiple owners by keeping a count of references; the value is dropped only when the last `Rc` goes away. `Rc` is **single-threaded only**. For multiple threads, use **`Arc<T>`** ("atomic reference counted"), which does the same with thread-safe atomic counting (slightly slower). Both give shared, *immutable* access.

```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new(String::from("shared data"));
    let b = Rc::clone(&a);    // does NOT deep-copy; bumps the reference count to 2
    let c = Rc::clone(&a);    // count = 3
    println!("count = {}", Rc::strong_count(&a));   // 3
    println!("{a} {b} {c}");  // all three point at the SAME heap string
    drop(b);
    println!("count = {}", Rc::strong_count(&a));   // 2
}   // remaining Rcs dropped; count hits 0; the String is freed
```

### 11.3 `RefCell<T>` and `Mutex<T>` — interior mutability **[A]**

The borrowing rules (one `&mut` XOR many `&`) are checked *at compile time*. Occasionally you need to mutate data through a shared reference, and you're willing to enforce the rule **at runtime** instead. That's **interior mutability**.

- **`RefCell<T>`** (single-threaded) enforces borrow rules at runtime: `.borrow()` gives a shared ref, `.borrow_mut()` a mutable one. If you violate the rule (e.g. two `borrow_mut` at once), it **panics** at runtime instead of failing to compile. Often combined as `Rc<RefCell<T>>` for "shared, mutable" single-threaded data.
- **`Mutex<T>`** (multi-threaded) is the concurrent analog: `.lock()` blocks until you have exclusive access, returning a guard. Combined as `Arc<Mutex<T>>` for shared mutable state across threads (§13.4).

```rust
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    // Shared + mutable, single-threaded: Rc for sharing, RefCell for mutation.
    let shared = Rc::new(RefCell::new(vec![1, 2, 3]));
    let clone = Rc::clone(&shared);

    shared.borrow_mut().push(4);     // mutate through a shared Rc — checked at RUNTIME
    clone.borrow_mut().push(5);

    println!("{:?}", shared.borrow());   // [1, 2, 3, 4, 5]

    // Beware: two simultaneous borrow_mut() panics:
    // let _a = shared.borrow_mut();
    // let _b = shared.borrow_mut();    // PANIC: already mutably borrowed
}
```

**Choosing a smart pointer:** `Box` for single-owner heap/recursion/trait objects; `Rc` for multiple owners (single thread); `Arc` for multiple owners across threads; add `RefCell` (single thread) or `Mutex` (threads) when you also need to mutate shared data.

---

## 12. Modules, Crates & Cargo

### 12.1 Crates and the module tree **[I]**

A **crate** is a compilation unit: either a binary (has `main`) or a library. A **package** (a `Cargo.toml`) bundles one or more crates. Within a crate, **modules** (`mod`) organize code into a tree and control **privacy** — everything is private by default, and you expose items with `pub`.

```rust
// All in src/main.rs for illustration; in real projects modules go in separate files.
mod network {
    pub mod client {                  // pub makes the module reachable from outside
        pub fn connect() -> String {  // pub makes the function callable from outside
            format!("connected via {}", helper())
        }
        fn helper() -> &'static str { "tcp" }   // private: only usable within `client`
    }

    pub struct Config {
        pub host: String,             // pub fields are individually exposed
        port: u16,                    // private field
    }
    impl Config {
        pub fn new(host: &str) -> Self {
            Config { host: host.to_string(), port: 8080 }
        }
        pub fn port(&self) -> u16 { self.port }   // accessor for the private field
    }
}

// `use` brings a path into scope so you don't repeat the full path.
use network::client;

fn main() {
    println!("{}", client::connect());                // absolute-ish via `use`
    println!("{}", network::client::connect());       // full path also works
    let cfg = network::Config::new("localhost");
    println!("{} {}", cfg.host, cfg.port());
}
```

Path keywords: `crate::` (the crate root), `self::` (current module), `super::` (parent module).

### 12.2 Splitting modules across files **[I]**

`mod foo;` (no body) tells the compiler to load the module from `foo.rs` or `foo/mod.rs`. This is how real projects are laid out:

```
src/
├── main.rs        // contains `mod network;` and `mod utils;`
├── network.rs     // the `network` module  (or network/mod.rs for submodules)
└── utils.rs       // the `utils` module
```

```rust
// src/main.rs
mod network;            // loads src/network.rs
mod utils;              // loads src/utils.rs

use network::connect;

fn main() {
    connect();
    utils::log("started");
}
```

```rust
// src/network.rs
pub fn connect() {
    println!("connecting...");
}
```

### 12.3 Cargo.toml, dependencies & features **[I]**

```toml
[package]
name = "myapp"
version = "0.1.0"
edition = "2024"

[dependencies]
serde = { version = "1", features = ["derive"] }   # enable the "derive" feature
tokio = { version = "1", features = ["full"] }
rand = "0.8"                                         # shorthand: just a version
local_lib = { path = "../local_lib" }               # a path dependency
some_git = { git = "https://github.com/x/y", branch = "main" }   # a git dependency

[dev-dependencies]                                   # only for tests/benches/examples
proptest = "1"

[features]                                           # YOUR crate's optional features
default = ["fast"]                                   # enabled unless --no-default-features
fast = []
extra = ["dep:rayon"]                                # turning on `extra` pulls in rayon

[profile.release]                                    # tune optimization
opt-level = 3
lto = true                                           # link-time optimization
```

**Features** are named, optional bits of functionality (and optional dependencies). They let users compile only what they need. `cargo build --features extra` or `--no-default-features` controls them.

### 12.4 Workspaces — multi-crate projects **[I/A]**

A **workspace** groups several related crates that share one `Cargo.lock` and `target/` directory — ideal for splitting an app into a library plus a CLI, or microservices in one repo.

```toml
# top-level Cargo.toml (the workspace root)
[workspace]
resolver = "2"
members = ["core", "cli", "web"]
```

```
myproject/
├── Cargo.toml          # the [workspace] above
├── core/  (Cargo.toml + src/lib.rs)
├── cli/   (Cargo.toml + src/main.rs, depends on core via { path = "../core" })
└── web/   (Cargo.toml + src/main.rs)
```

`cargo build` from the root builds everything; `cargo run -p cli` runs a specific member. Publishing to **crates.io** (the public registry) is `cargo publish` from a crate directory.

---

## 13. Concurrency & Async

Rust offers two concurrency models: OS **threads** (for CPU-bound parallelism) and **async/await** (for high-volume I/O-bound concurrency). The ownership system makes both far safer than usual — the borrow checker rejects data races at compile time.

### 13.1 Threads and `move` closures **[A]**

`std::thread::spawn` runs a closure on a new OS thread. Because the new thread may outlive the spawning function, the closure must own (`move`) any data it uses.

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];

    // `move` transfers ownership of `data` into the thread's closure — required,
    // because the thread might run after main's stack frame is gone.
    let handle = thread::spawn(move || {
        let sum: i32 = data.iter().sum();
        println!("thread computed {sum}");
        sum
    });

    // .join() blocks until the thread finishes and returns its result.
    let result = handle.join().unwrap();
    println!("main got {result}");          // main got 6
}
```

### 13.2 `Send` and `Sync` — what they guarantee **[A]**

Two marker traits encode thread-safety, and the compiler uses them to enforce it:

- **`Send`**: a type is `Send` if it's safe to **transfer ownership to another thread**. Almost everything is `Send`; notable exceptions are `Rc<T>` (its non-atomic count would race) and raw pointers.
- **`Sync`**: a type is `Sync` if it's safe to **share a reference (`&T`) across threads**. `T` is `Sync` iff `&T` is `Send`.

You rarely implement these by hand — they're auto-derived. Their power is that thread APIs *require* them (`thread::spawn` needs `Send`), so the compiler refuses to let you move thread-unsafe data across threads. That's the mechanism behind "fearless concurrency": misusing `Rc` across threads is a compile error, not a heisenbug.

### 13.3 Channels — message passing **[A]**

Channels let threads communicate by **sending messages** rather than sharing memory ("do not communicate by sharing memory; share memory by communicating"). `std::sync::mpsc` is "multiple producer, single consumer."

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();          // tx = transmitter, rx = receiver

    for id in 0..3 {
        let tx = tx.clone();                 // clone the sender for each producer
        thread::spawn(move || {
            tx.send(format!("hello from {id}")).unwrap();   // send moves the value
        });
    }
    drop(tx);                                // drop the original so rx knows when to stop

    // rx is an iterator: yields messages until all senders are dropped.
    for msg in rx {
        println!("{msg}");                   // prints 3 messages, order not guaranteed
    }
}
```

### 13.4 Shared state with `Arc<Mutex<T>>` **[A]**

When threads must share *mutable* state, combine `Arc` (shared ownership across threads) with `Mutex` (exclusive access). This is the canonical pattern.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    // Arc lets multiple threads own the counter; Mutex guards mutation.
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);   // each thread gets its own Arc handle
        let handle = thread::spawn(move || {
            // .lock() blocks until exclusive; the guard auto-unlocks when dropped.
            let mut num = counter.lock().unwrap();
            *num += 1;                         // safe: only one thread holds the lock
        });
        handles.push(handle);
    }
    for h in handles { h.join().unwrap(); }

    println!("counter = {}", *counter.lock().unwrap());   // 10  (no data race)
}
```

> **Rayon** (`cargo add rayon`) makes data parallelism trivial: `v.par_iter().map(...).sum()` runs across all cores. Great for CPU-bound work where you'd otherwise hand-roll threads.

### 13.5 Async/await with tokio **[A]**

For workloads dominated by **waiting on I/O** (network servers, many concurrent connections), threads are wasteful — you'd need thousands. **Async** lets a small number of threads juggle huge numbers of concurrent tasks. The model:

- An `async fn` returns a **`Future`** — a value representing a computation that isn't done yet. Futures are **lazy**: they do nothing until polled.
- `.await` on a future yields control back to the runtime if the future isn't ready, letting other tasks run; it resumes when the future is ready. Crucially, `.await` does *not* block the OS thread.
- A **runtime** (executor) polls futures and schedules tasks. Rust's std doesn't ship one — **tokio** is the standard choice in 2026.

```bash
cargo add tokio --features full
```

```rust
use std::time::Duration;

// #[tokio::main] sets up the runtime and lets main be async.
#[tokio::main]
async fn main() {
    // Awaiting runs sequentially:
    let a = fetch("A").await;
    println!("{a}");

    // tokio::spawn launches a CONCURRENT task (like a green thread); returns a handle.
    let h1 = tokio::spawn(fetch("B"));
    let h2 = tokio::spawn(fetch("C"));
    // join! awaits multiple futures concurrently — B and C overlap:
    let (b, c) = tokio::join!(h1, h2);
    println!("{} {}", b.unwrap(), c.unwrap());
}

// An async function: its body can .await other futures.
async fn fetch(name: &str) -> String {
    // tokio's sleep is async — it does NOT block the thread, it yields.
    tokio::time::sleep(Duration::from_millis(50)).await;
    format!("fetched {name}")
}
```

> **Async coloring:** `async` functions can only be `.await`ed inside other `async` contexts ("function color"). Mixing sync and async needs care (`tokio::task::spawn_blocking` for CPU-heavy or blocking calls inside async code). For HTTP, `reqwest` (client) and `axum`/`actix-web` (servers) are async and built on tokio.

---

## 14. File System, OS Info & Command Execution

This section is deliberately thorough — reading/writing files, inspecting the environment, and shelling out to other programs are everyday tasks. All of these are in the standard library (`std::fs`, `std::path`, `std::env`, `std::process`, `std::io`) and return `Result`, so pair them with `?` (§8).

### 14.1 Reading & writing files with `std::fs` **[I]**

For small files, the one-shot helpers are simplest:

```rust
use std::fs;
use std::io;

fn main() -> io::Result<()> {
    // Write a whole string (creates or TRUNCATES the file):
    fs::write("notes.txt", "line one\nline two\n")?;

    // Read a whole file into a String (must be valid UTF-8):
    let text = fs::read_to_string("notes.txt")?;
    println!("--- contents ---\n{text}");

    // Read raw bytes (for binary files):
    let bytes: Vec<u8> = fs::read("notes.txt")?;
    println!("{} bytes", bytes.len());

    // Process line by line:
    for (i, line) in text.lines().enumerate() {
        println!("{i}: {line}");
    }
    Ok(())
}
```

For large files or appending, use `File` with buffered I/O (avoids reading everything into memory):

```rust
use std::fs::{File, OpenOptions};
use std::io::{self, BufRead, BufReader, Write, BufWriter};

fn main() -> io::Result<()> {
    // Buffered WRITE — efficient for many small writes:
    let file = File::create("out.txt")?;          // create/truncate
    let mut writer = BufWriter::new(file);
    for i in 0..5 {
        writeln!(writer, "row {i}")?;             // writeln! adds a newline
    }
    writer.flush()?;                               // ensure buffer is written

    // APPEND instead of truncate:
    let mut appender = OpenOptions::new()
        .create(true)
        .append(true)
        .open("out.txt")?;
    writeln!(appender, "appended line")?;

    // Buffered READ — stream lines without loading the whole file:
    let reader = BufReader::new(File::open("out.txt")?);
    for line in reader.lines() {
        let line = line?;                          // each line is a Result
        println!("read: {line}");
    }
    Ok(())
}
```

### 14.2 Directories, metadata, and `read_dir` **[I]**

```rust
use std::fs;
use std::io;

fn main() -> io::Result<()> {
    // Create nested directories (like `mkdir -p` — no error if they exist):
    fs::create_dir_all("data/sub/dir")?;

    // Metadata: size, file vs dir, modified time, permissions:
    let meta = fs::metadata("Cargo.toml")?;
    println!("is_file: {}, size: {} bytes", meta.is_file(), meta.len());

    // Check existence without erroring:
    println!("exists: {}", std::path::Path::new("Cargo.toml").exists());

    // List a directory. read_dir yields Result<DirEntry> items:
    for entry in fs::read_dir(".")? {
        let entry = entry?;
        let path = entry.path();
        let kind = if path.is_dir() { "dir " } else { "file" };
        println!("{kind} {}", path.display());     // .display() for human output
    }

    // Copy, rename/move, remove:
    fs::copy("Cargo.toml", "data/Cargo.bak")?;     // copy a file
    // fs::rename("a.txt", "b.txt")?;              // move/rename
    fs::remove_file("data/Cargo.bak")?;            // delete a file
    fs::remove_dir_all("data")?;                   // delete a dir tree (careful!)
    Ok(())
}
```

### 14.3 Paths: `Path` and `PathBuf` **[I]**

Paths get their own types so cross-platform path handling is correct (Windows `\` vs Unix `/`). The relationship mirrors `&str`/`String`: **`Path`** is a borrowed path slice; **`PathBuf`** is an owned, growable path. Build paths with `.join` (never by string concatenation) so separators are correct on every OS.

```rust
use std::path::{Path, PathBuf};

fn main() {
    // Borrowed view:
    let p = Path::new("/home/ada/file.txt");
    println!("{:?}", p.file_name());     // Some("file.txt")
    println!("{:?}", p.extension());     // Some("txt")
    println!("{:?}", p.parent());        // Some("/home/ada")
    println!("{:?}", p.file_stem());     // Some("file")

    // Owned, built piece by piece (handles separators per-OS):
    let mut path = PathBuf::from("data");
    path.push("logs");                   // data/logs   (or data\logs on Windows)
    path.push("today.log");              // data/logs/today.log
    println!("{}", path.display());

    // join returns a new PathBuf without mutating the original:
    let cfg = Path::new("/etc").join("app").join("config.toml");
    println!("{}", cfg.display());

    // fs functions accept anything that's AsRef<Path>: &str, String, Path, PathBuf.
    let _exists = path.exists();
}
```

### 14.4 Environment & arguments with `std::env` **[I]**

```rust
use std::env;

fn main() {
    // Command-line arguments. args()[0] is the program path; the rest are user args.
    let args: Vec<String> = env::args().collect();
    println!("program: {}", args[0]);
    println!("args: {:?}", &args[1..]);

    // Environment variables (returns Result — the var may be unset):
    match env::var("PATH") {
        Ok(val) => println!("PATH has {} chars", val.len()),
        Err(_) => println!("PATH not set"),
    }
    let home = env::var("HOME").or_else(|_| env::var("USERPROFILE"));   // unix | windows
    println!("home: {home:?}");

    // Set / iterate env vars:
    env::set_var("MY_FLAG", "1");
    for (k, v) in env::vars().take(3) {
        println!("{k}={v}");
    }

    // Working directory & OS info:
    println!("cwd: {:?}", env::current_dir().unwrap());
    println!("exe: {:?}", env::current_exe().unwrap());
    println!("os: {} / arch: {}", env::consts::OS, env::consts::ARCH);  // e.g. windows / x86_64
}
```

### 14.5 Running external commands with `std::process::Command` **[I]**

`std::process::Command` builds and runs external programs — the equivalent of shelling out. You configure it (program, args, env, working directory) then either capture its output, wait for its status, or spawn it to run alongside your program.

```rust
use std::process::{Command, Stdio};

fn main() -> std::io::Result<()> {
    // 1) CAPTURE OUTPUT with .output() — runs to completion, captures stdout/stderr.
    let out = Command::new("git")
        .args(["rev-parse", "--abbrev-ref", "HEAD"])   // git rev-parse --abbrev-ref HEAD
        .output()?;
    if out.status.success() {
        // stdout is raw bytes; convert assuming UTF-8:
        let branch = String::from_utf8_lossy(&out.stdout);
        println!("on branch: {}", branch.trim());
    } else {
        eprintln!("git failed: {}", String::from_utf8_lossy(&out.stderr));
    }

    // 2) Just the EXIT STATUS with .status() — output goes to the terminal directly.
    let status = Command::new("python")
        .args(["--version"])
        .status()?;
    println!("python exited with: {:?}", status.code());   // Some(0) on success

    // 3) SPAWN to run concurrently; control it via the handle.
    let mut child = Command::new("npm")
        .arg("install")
        .current_dir("some_project")        // set working directory
        .env("CI", "true")                   // set an env var for the child
        .stdout(Stdio::piped())              // capture stdout via a pipe
        .spawn()?;                            // returns immediately, process runs in background
    // ... do other work while npm runs ...
    let result = child.wait()?;              // block until it finishes
    println!("npm done: success={}", result.success());

    // 4) List a directory via the shell (cross-platform note below):
    #[cfg(windows)]
    let list = Command::new("cmd").args(["/C", "dir"]).output()?;
    #[cfg(not(windows))]
    let list = Command::new("ls").args(["-la"]).output()?;
    println!("{}", String::from_utf8_lossy(&list.stdout));

    Ok(())
}
```

Key points: `.output()` captures and waits; `.status()` waits and inherits the terminal; `.spawn()` runs in the background and returns a `Child` you can `.wait()` / `.kill()`. Exit codes come from `status.code()` (an `Option<i32>` — `None` if killed by a signal). `String::from_utf8_lossy` safely turns captured bytes into text. Use `#[cfg(windows)]` for OS-specific commands.

### 14.6 Reading stdin **[I]**

```rust
use std::io::{self, BufRead, Write};

fn main() -> io::Result<()> {
    // Prompt and read a single line:
    print!("Enter your name: ");
    io::stdout().flush()?;                  // flush so the prompt shows before reading
    let mut name = String::new();
    io::stdin().read_line(&mut name)?;      // includes the trailing newline
    let name = name.trim();                 // strip it
    println!("Hello, {name}!");

    // Read all of stdin line by line (e.g. for piped input: `cat file | myprog`):
    let stdin = io::stdin();
    let mut total = 0usize;
    for line in stdin.lock().lines() {
        let line = line?;
        total += line.len();
    }
    println!("total chars: {total}");
    Ok(())
}
```

### 14.7 Building CLIs: `std::env::args` vs `clap` **[I]**

For a tiny tool, parse `env::args()` by hand. For anything real — subcommands, flags, `--help`, validation — use **`clap`** with its derive API: you declare a struct describing your arguments and clap generates the parser, help text, and error messages.

```bash
cargo add clap --features derive
```

```rust
use clap::Parser;

// #[derive(Parser)] turns this struct into a full CLI parser.
#[derive(Parser, Debug)]
#[command(name = "greet", version, about = "Greets a person")]
struct Args {
    /// Name of the person to greet (positional argument)
    name: String,

    /// Number of times to greet (flag with default)
    #[arg(short, long, default_value_t = 1)]    // -c / --count
    count: u8,

    /// Shout the greeting (boolean flag)
    #[arg(short, long)]                          // -s / --shout
    shout: bool,
}

fn main() {
    let args = Args::parse();      // parses argv; prints help/errors & exits as needed
    let mut msg = format!("Hello, {}!", args.name);
    if args.shout {
        msg = msg.to_uppercase();
    }
    for _ in 0..args.count {
        println!("{msg}");
    }
}
// Run: cargo run -- Ada --count 3 --shout
//      cargo run -- --help      (auto-generated usage)
```

---

## 15. Macros & the Type System

### 15.1 Why macros exist **[A]**

Macros generate code at compile time. They do things functions can't: take a *variable* number of arguments (`println!`, `vec!`), accept arbitrary syntax, and implement traits automatically (derive). You'll *use* macros constantly and *write* them occasionally. The `!` marks a macro invocation.

### 15.2 Declarative macros: `macro_rules!` **[A]**

Declarative macros match input *patterns* and expand to code, a bit like a `match` over syntax. Here's a simplified `vec!`-style macro:

```rust
// Matches zero or more comma-separated expressions and builds a Vec.
macro_rules! my_vec {
    // $( ... ),*  means "repeat, comma-separated, zero or more times".
    // $x:expr captures each item as an expression.
    ( $( $x:expr ),* $(,)? ) => {       // the $(,)? allows an optional trailing comma
        {
            let mut v = Vec::new();
            $( v.push($x); )*           // expands the push once PER captured $x
            v
        }
    };
}

fn main() {
    let v = my_vec![1, 2, 3];           // expands to push(1); push(2); push(3);
    println!("{v:?}");                  // [1, 2, 3]

    let empty: Vec<i32> = my_vec![];
    println!("{empty:?}");              // []
}
```

Fragment specifiers include `expr`, `ident` (a name), `ty` (a type), `tt` (a token tree), `block`, `stmt`, `pat`, `literal`. Repetition operators: `*` (0+), `+` (1+), `?` (0 or 1).

### 15.3 Procedural & derive macros **[A]**

**Procedural macros** are Rust functions that take a token stream and return a token stream — they run actual code at compile time and live in their own crate (`proc-macro = true`). The three kinds:

- **Derive macros** (`#[derive(Serialize)]`) — generate trait impls from a type definition. This is how `serde`, `thiserror`, and `clap` work — you derive and they write the boilerplate.
- **Attribute macros** (`#[tokio::main]`, `#[get("/")]`) — transform the item they're attached to.
- **Function-like macros** (`sqlx::query!(...)`) — like `macro_rules!` but with full Rust logic, e.g. validating SQL at compile time.

You'll rarely write proc macros (they need `syn` and `quote` crates and a fair bit of ceremony), but understanding that `#[derive(...)]` and `#[tokio::main]` are *code generators* demystifies a lot of the ecosystem.

### 15.4 A note on type-system power **[A]**

Rust's type system does real work for you: exhaustive enums + `match` make illegal states unrepresentable; the **newtype pattern** (`struct Meters(f64)`) prevents mixing up units; **zero-sized types** and marker traits encode invariants with no runtime cost; **typestate** patterns encode a state machine in types so misuse won't compile. The slogan is "**make illegal states unrepresentable**" — push correctness into types so the compiler enforces it.

```rust
// Newtype pattern: distinct types prevent mixing semantically different values.
struct Meters(f64);
struct Seconds(f64);
fn speed(d: Meters, t: Seconds) -> f64 { d.0 / t.0 }   // can't swap args by accident

fn main() {
    println!("{}", speed(Meters(100.0), Seconds(9.58)));   // ~10.44
    // speed(Seconds(9.58), Meters(100.0));  // ERROR: types don't match — bug caught!
}
```

---

## 16. Testing

Testing is built into cargo. No external framework needed for unit tests.

### 16.1 Unit tests **[I]**

By convention, unit tests live in the same file as the code they test, in a `#[cfg(test)]` module (so they compile only during `cargo test`). Test functions are marked `#[test]`.

```rust
// src/lib.rs (or any module)
pub fn add(a: i32, b: i32) -> i32 { a + b }
pub fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 { Err("divide by zero".into()) } else { Ok(a / b) }
}

#[cfg(test)]                 // only compiled for tests
mod tests {
    use super::*;            // bring the parent module's items into scope

    #[test]
    fn adds() {
        assert_eq!(add(2, 3), 5);            // equality assertion
        assert_ne!(add(2, 3), 6);            // inequality
        assert!(add(2, 2) == 4, "math broke"); // boolean + optional message
    }

    #[test]
    fn divides_ok() -> Result<(), String> {  // tests can return Result; Err = failure
        assert_eq!(divide(10, 2)?, 5);
        Ok(())
    }

    #[test]
    #[should_panic(expected = "divide by zero")]   // passes only if it panics with this msg
    fn division_panics_when_unwrapped() {
        divide(1, 0).unwrap();               // unwrap of Err panics
    }

    #[test]
    #[ignore]                                // skipped unless `cargo test -- --ignored`
    fn expensive_test() { /* ... */ }
}
```

```bash
cargo test                  # run all tests
cargo test adds             # run tests whose name contains "adds"
cargo test -- --nocapture   # show println! output from tests
cargo test -- --ignored     # run only #[ignore]d tests
```

### 16.2 Integration tests & doc tests **[I]**

- **Integration tests** live in a top-level `tests/` directory. Each file is a separate crate that uses your library through its *public* API only — exactly as a real user would.

```rust
// tests/integration.rs
use myapp::add;                  // `myapp` = your crate name from Cargo.toml

#[test]
fn public_api_works() {
    assert_eq!(add(2, 2), 4);
}
```

- **Doc tests**: code blocks in `///` doc comments are compiled and run by `cargo test`. This guarantees your documentation examples actually work — a brilliant feature.

```rust
/// Adds two numbers.
///
/// ```
/// use myapp::add;
/// assert_eq!(add(2, 3), 5);   // this runs during `cargo test`
/// ```
pub fn add(a: i32, b: i32) -> i32 { a + b }
```

---

## 17. Tooling & Ecosystem

### 17.1 Built-in tooling **[I]**

| Tool | What it does | Command |
|------|--------------|---------|
| **rustfmt** | Auto-formats code to the standard style | `cargo fmt` |
| **clippy** | Linter with 700+ lints — catches bugs & non-idiomatic code | `cargo clippy` |
| **rust-analyzer** | The LSP server powering IDE completion, go-to-def, inline errors | (used by your editor) |
| **cargo check** | Fast type-check without codegen | `cargo check` |
| **cargo doc** | Generate HTML API docs from `///` comments | `cargo doc --open` |
| **cargo bench** | Run benchmarks (stable via `criterion` crate) | `cargo bench` |

Run `cargo clippy -- -D warnings` in CI to treat lints as errors. Clippy is genuinely educational — it teaches idioms as you go.

### 17.2 Essential crates (the 2026 toolbox) **[I]**

| Crate | Purpose |
|-------|---------|
| **serde** + **serde_json** | Serialize/deserialize to/from JSON, etc. via `#[derive(Serialize, Deserialize)]` |
| **tokio** | The async runtime (networking, timers, tasks) |
| **reqwest** | Ergonomic async HTTP client |
| **clap** | CLI argument parsing (derive API) |
| **anyhow** / **thiserror** | Error handling for apps / libraries (§8.4) |
| **rayon** | Effortless data parallelism (`par_iter`) |
| **tracing** | Structured, async-aware logging/diagnostics |
| **chrono** / **time** | Dates and times |
| **regex** | Regular expressions |
| **rand** | Random number generation |
| **axum** / **actix-web** | Web frameworks (see below) |
| **sqlx** / **diesel** / **sea-orm** | Database access (sqlx = async, compile-time-checked SQL) |

A serde example, since it's ubiquitous:

```rust
// cargo add serde --features derive ; cargo add serde_json
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Debug)]
struct Config {
    name: String,
    port: u16,
    tags: Vec<String>,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let cfg = Config { name: "api".into(), port: 8080, tags: vec!["web".into()] };
    let json = serde_json::to_string_pretty(&cfg)?;     // struct -> JSON
    println!("{json}");
    let parsed: Config = serde_json::from_str(&json)?;   // JSON -> struct
    println!("{parsed:?}");
    Ok(())
}
```

### 17.3 Web frameworks **[I]**

For web services, **axum** (built by the tokio team, ergonomic, type-safe extractors) is the most popular choice in 2026; **actix-web** is a fast, mature alternative. Both are async (tokio-based). A minimal axum service:

```rust
// cargo add axum tokio --features tokio/full
use axum::{routing::get, Router};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(|| async { "Hello, axum!" }));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    println!("listening on http://127.0.0.1:3000");
    axum::serve(listener, app).await.unwrap();
}
```

---

## 18. Gotchas & Best Practices

### 18.1 Fighting the borrow checker productively **[A]**

When the borrow checker says no, it's usually pointing at a real issue (or a slightly awkward structure). Productive responses, roughly in order of preference:

1. **Narrow the scope of borrows.** Thanks to NLL, a borrow ends at its last *use*. Restructure so a `&mut` borrow finishes before you need another borrow. Often just moving a line fixes it.
2. **Borrow instead of move.** If a function only reads, take `&T`/`&str`/`&[T]` rather than `T`.
3. **Split borrows.** Borrowing two *different* fields of a struct mutably is allowed; borrowing the whole struct twice isn't. Destructure or use helper methods.
4. **Clone deliberately.** A `.clone()` is a legitimate fix when the data is small or the perf doesn't matter — just do it knowingly, not reflexively (see below).
5. **Reach for interior mutability or `Rc`/`Arc`** only when the data genuinely has shared/aliased ownership (graphs, shared caches). Don't use them to dodge learning ownership.

### 18.2 Classic beginner mistakes **[A]**

```rust
fn main() {
    // 1) CLONE-HAPPY: cloning to silence the borrow checker. Works, but wasteful.
    let data = vec![1, 2, 3];
    fn process(v: &[i32]) -> i32 { v.iter().sum() }   // GOOD: borrow
    println!("{}", process(&data));                    // no clone needed
    // BAD pattern that beginners write: process(data.clone()) with a fn taking Vec<i32>

    // 2) unwrap() / expect() IN PRODUCTION: turns a recoverable error into a crash.
    // Fine in tests/prototypes; in real code propagate with ? or handle the error.
    let n: i32 = "42".parse().unwrap();                // OK here; could panic on bad input
    let safe: i32 = "oops".parse().unwrap_or(0);       // BETTER: a default
    println!("{n} {safe}");

    // 3) &String parameters: less flexible than &str. Prefer &str.
    fn bad(s: &String) {}        // only accepts &String
    fn good(s: &str) {}          // accepts &str, &String, literals — always prefer this
    let _ = (bad, good);

    // 4) Forgetting that integer overflow PANICS in debug builds. Use checked/saturating
    //    ops when overflow is possible on user-controlled input.

    // 5) Returning references to locals — impossible (see §5.4). Return owned values.
}
```

### 18.3 Best-practice checklist **[A]**

- Run `cargo clippy` and `cargo fmt` before committing; treat clippy as a tutor.
- Prefer `?` over `unwrap()` in non-test code; reserve `panic!`/`unwrap` for true invariants.
- Take `&str`/`&[T]` parameters; return owned `String`/`Vec<T>`.
- Derive `Debug` on your types so they're printable; derive `Clone`/`PartialEq` when needed.
- Make illegal states unrepresentable with enums and the newtype pattern (§15.4).
- Use `thiserror` in libraries, `anyhow` in binaries (§8.4).
- Don't fear the compiler — error messages are exceptionally good; read them fully, they often suggest the fix.
- Keep `unsafe` rare, small, well-commented, and wrapped in a safe API. Most application code needs none.
- Commit `Cargo.lock` for binaries; for libraries, don't.

### 18.4 Naming & style conventions **[I]**

`snake_case` for functions, variables, modules; `PascalCase` for types/traits/enum variants; `SCREAMING_SNAKE_CASE` for constants/statics. Crate names are typically `kebab-case` (but referenced as `snake_case` in code). 4-space indent, ~100-char lines — but just let `cargo fmt` handle all of it.

---

## 19. Study Path & Build-to-Learn Projects

**Suggested order:** §1–3 (setup, basics, control flow) → **§4–6 (ownership, borrowing, lifetimes — slow down here; this is the whole game)** → §7–8 (structs/enums/Option/Result, error handling) → §10 (collections, strings, iterators — immediately useful) → §9 (generics & traits) → §12 (modules & cargo) → §14 (files/OS/commands — practical and fun) → §16–17 (testing & tooling) → §11, §13, §15, §18 (smart pointers, concurrency/async, macros, advanced practices).

Don't rush §4–6. Re-read them, write tiny programs that deliberately trigger borrow errors, and read the compiler's explanations. Ownership clicks suddenly, not gradually — and once it does, everything else is comparatively easy.

**Build these to cement it:**

1. **CLI tool — a `grep`/`wc`-style text utility.** Parse args with `clap`, read files and stdin (§14.1, §14.6), iterate lines, count/filter with iterators (§10.4), print results, return proper exit codes. Exercises §3, §10, §14, §17.
2. **File organizer / processor.** Walk a directory tree with `read_dir` (§14.2), classify files by extension via `Path` (§14.3), move/copy them, and shell out to `git` to commit the result (§14.5). Add good error handling with `anyhow` (§8.4). Exercises §8, §14.
3. **A small library + tests.** Model a domain (e.g. a tiny expression evaluator or a key-value store) with structs, enums, traits, and generics. Write unit, integration, and doc tests (§16). Publish-style structure with a workspace (§12.4). Exercises §7, §9, §12, §16.
4. **Concurrent download/processing pipeline.** Spawn threads or tokio tasks, pass work via channels (§13.3), aggregate results in `Arc<Mutex<T>>` (§13.4) — or go async with `reqwest` + tokio. Exercises §11, §13.
5. **A small web service with axum.** Async handlers, JSON request/response with serde, an in-memory store behind `Arc<Mutex<_>>`, a few routes (§13.5, §17). Exercises §8, §9, §13, §17 — and ties the whole guide together.

**Next steps after this guide:** dig into the async ecosystem (tokio internals, `tracing`), a database layer (`sqlx`), and `unsafe` Rust + FFI when you need to interoperate with C. For the file/OS material in another language for comparison, see the Python guide in this library (§12 there mirrors §14 here).

---
