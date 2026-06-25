# C++ — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I've never written C++" (or "I only know C-with-classes") to "I write safe, modern, RAII-driven, production-grade C++ that scales and is maintainable" — without an internet connection. This guide is **explain-first**: every concept leads with prose (what it is, *why* it works that way, when to reach for it, the safety/UB implications) and *then* shows heavily-commented, valid **C++23** code. It assumes a little C is helpful but not required — if you've never seen pointers, arrays, or `printf`, skim the [C](C_GUIDE.md) guide first; everything you actually need is re-explained here. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> The single most important idea in this guide: **modern C++ is not C-with-classes.** You will rarely write `new`/`delete`, almost never write raw owning pointers, and you will lean hard on **RAII** (Resource Acquisition Is Initialization), **value semantics**, **smart pointers**, and the **STL**. That is what makes C++ safe and maintainable rather than a minefield.
>
> **Version note:** This guide targets **C++23** (ISO/IEC 14882:2024 — the current published standard) as the baseline, and mentions **C++26** (in progress; partial support in GCC 15 / Clang 20) with items flagged **⚡ Version note**. Compilers as of 2026: **GCC 14/15**, **Clang 18/19**, **MSVC** (Visual Studio 2022). Always build with `-std=c++23 -Wall -Wextra` (or `/std:c++latest` on MSVC). Modern features used throughout: `auto`, ranges & views (C++20), concepts (C++20), modules (C++20 — compiler maturity varies), coroutines (C++20), `std::expected` (C++23), `std::print`/`<print>` (C++23), `std::format` (C++20), `std::span`, `std::string_view`, structured bindings, `constexpr`/`consteval`, designated initializers, and the three-way comparison operator `<=>`. Where a feature's library/compiler support is still uneven (notably `<print>` and modules) it is flagged with **⚡ Version note**. The author is on **Windows 11**, so cross-platform notes (path separators, `.exe`, MSVC vs MinGW) are called out. Confirm exact APIs at en.cppreference.com.
>
> Related guides in this library: [C](C_GUIDE.md) (the substrate C++ sits on), [Rust](RUST_GUIDE.md) (the other modern systems language — safety by compiler vs. by discipline), [Go](GO_GUIDE.md), [Networking](NETWORKING_GUIDE.md), [Linux Server Administration](LINUX_SERVER_ADMIN_GUIDE.md), and [Git](GIT_GUIDE.md).

---

## Table of Contents

1. [What C++ Is & the Modern-C++ Philosophy](#1-what-c-is--the-modern-c-philosophy) **[B]**
2. [Setup, Build & Your First Program (g++/clang++/MSVC, CMake)](#2-setup-build--your-first-program-gclangmsvc-cmake) **[B]**
3. [Fundamentals: Types, auto, References, const/constexpr, Namespaces](#3-fundamentals-types-auto-references-constconstexpr-namespaces) **[B]**
4. [Control Flow, Operators & Console I/O](#4-control-flow-operators--console-io) **[B]**
5. [Functions: Overloading, Defaults, Lambdas, std::function, Templates Intro](#5-functions-overloading-defaults-lambdas-stdfunction-templates-intro) **[B/I]**
6. [Classes & RAII: The Rule of 0/3/5, Copy vs Move](#6-classes--raii-the-rule-of-035-copy-vs-move) **[I]**
7. [Resource Management & Smart Pointers](#7-resource-management--smart-pointers) **[I/A]**
8. [Move Semantics & Value Categories](#8-move-semantics--value-categories) **[I/A]**
9. [STL Containers: vector, string, map, set & Friends](#9-stl-containers-vector-string-map-set--friends) **[I]**
10. [Iterators, Algorithms & Ranges (C++20)](#10-iterators-algorithms--ranges-c20) **[I/A]**
11. [Templates, Concepts & Generic Programming](#11-templates-concepts--generic-programming) **[A]**
12. [Error Handling: Exceptions, std::expected, std::optional](#12-error-handling-exceptions-stdexpected-stdoptional) **[I/A]**
13. [Inheritance, Polymorphism & std::variant](#13-inheritance-polymorphism--stdvariant) **[I/A]**
14. [File System, OS & Processes](#14-file-system-os--processes) **[I]**
15. [Concurrency: threads, mutex, atomic, async, coroutines](#15-concurrency-threads-mutex-atomic-async-coroutines) **[A]**
16. [Modern Build & Dependency Management (CMake, vcpkg, Conan, Modules)](#16-modern-build--dependency-management-cmake-vcpkg-conan-modules) **[A]**
17. [Production-Grade C++: Sanitizers, clang-tidy, Core Guidelines](#17-production-grade-c-sanitizers-clang-tidy-core-guidelines) **[A]**
18. [Testing & Maintainability (GoogleTest/Catch2, CI, Layout)](#18-testing--maintainability-googletestcatch2-ci-layout) **[A]**
19. [Performance: Cache, Allocations, constexpr, Profiling](#19-performance-cache-allocations-constexpr-profiling) **[A]**
20. [Gotchas & Best Practices](#20-gotchas--best-practices) **[I/A]**
21. [Study Path & Build-to-Learn Projects](#21-study-path--build-to-learn-projects)

---

## 1. What C++ Is & the Modern-C++ Philosophy

### 1.1 What C++ is, in one paragraph **[B]**

C++ is a **statically-typed, compiled, multi-paradigm** systems programming language. "Statically typed" means every value has a type the compiler knows and checks before the program runs. "Compiled" means your source text is translated ahead of time into native machine code for a specific CPU/OS — there is no interpreter or virtual machine at runtime (unlike Python, Java, or JavaScript). "Multi-paradigm" means it supports procedural code (functions and data, like C), object-oriented code (classes, inheritance, polymorphism), generic code (templates that work for any type), and functional touches (lambdas, immutability, ranges). It was created by Bjarne Stroustrup in the early 1980s as "C with classes," but the language of 2026 is almost unrecognizable from that origin.

### 1.2 The "zero-overhead principle" — the soul of C++ **[B]**

C++'s guiding design rule is the **zero-overhead principle**, stated by Stroustrup as: *"What you don't use, you don't pay for. And what you do use, you couldn't hand-code any better."* Concretely: abstractions in C++ (classes, templates, smart pointers, ranges, iterators) are designed to compile down to the same machine code you would have written by hand in C. A `std::unique_ptr<T>` is the same size as a raw `T*` and adds no runtime cost — it just calls `delete` for you when it goes out of scope. A `std::vector` index, with optimizations on, is the same as raw pointer arithmetic. This is why C++ powers things where every nanosecond and every byte matters: operating systems, browsers (Chrome, Firefox), game engines (Unreal), databases, high-frequency trading, embedded firmware, and the implementations of higher-level languages (CPython, the V8 JS engine, the JVM) are largely C++.

The trade is **control in exchange for responsibility**: C++ has no garbage collector and lets you touch raw memory, which means the *programmer* is responsible for correctness. Done carelessly (C-style), that responsibility produces use-after-free, double-free, buffer overflows, and data races — the source of the majority of security CVEs in shipped software. Done in **modern style** (RAII + smart pointers + STL + value semantics), the language manages those resources for you deterministically, and most of those bug classes simply stop occurring. This guide teaches the modern style relentlessly.

### 1.3 RAII and value semantics — the two ideas that make C++ safe **[B]**

Two concepts recur in every section. Learn them now, in plain language; the rest is detail.

**RAII — Resource Acquisition Is Initialization.** A clumsy name for a beautiful idea: tie the lifetime of a *resource* (heap memory, a file handle, a socket, a mutex lock, a database connection) to the lifetime of an *object on the stack*. You acquire the resource in a constructor and release it in the destructor. Because C++ guarantees that when a stack object goes out of scope its destructor runs — *automatically, deterministically, even if an exception is thrown* — the resource is always released exactly once, at exactly the right time, with no manual cleanup code and no leaks. This is how `std::unique_ptr`, `std::lock_guard`, `std::fstream`, and `std::vector` all work. RAII is the reason you don't need a garbage collector and don't need `try/finally`.

**Value semantics.** In C++ the default is that variables *are* values, not references to shared objects (unlike Python/Java/JS where `b = a` makes two names for one object). When you write `std::string b = a;` you get an independent *copy*; mutating `b` cannot affect `a`. Objects own their data, copying duplicates it, and assignment overwrites. This makes code dramatically easier to reason about — no spooky action at a distance — and combined with **move semantics** (§8) it's also efficient because copies that would be wasteful are turned into cheap moves automatically. When you *do* want sharing, you opt into it explicitly with `shared_ptr` or references.

### 1.4 C++ vs C vs Rust — when to pick what **[B]**

| Aspect | **C** | **C++** | **Rust** |
|---|---|---|---|
| Memory safety | Manual; easy to get wrong | Manual, but RAII/smart pointers make it *easy to get right* | Enforced at compile time by the borrow checker |
| Abstraction | Minimal (structs, functions) | Rich (classes, templates, ranges) at zero overhead | Rich (traits, generics) at zero overhead |
| Runtime / GC | None | None | None |
| Learning curve | Small surface, big footguns | Very large surface | Steep (ownership) up front, smooth after |
| Backward-compat baggage | Small | Large (40 years; you can write 1985-style C++) | Small, editions manage change |
| Use it when | Tiny footprint, kernels, MCUs, max portability | Huge existing ecosystem, game/HFT/desktop, need fine control + abstraction | New code where memory-safety guarantees are paramount |
| In this library | [C](C_GUIDE.md) | this guide | [Rust](RUST_GUIDE.md) |

The honest summary: **Rust** gives you memory safety by *forcing* you to prove it to the compiler; **modern C++** gives you the same safety *by convention and good tools* (RAII, smart pointers, sanitizers, Core Guidelines) but lets you bypass them, so discipline matters more. C++'s advantage is a colossal, mature ecosystem and decades of libraries. Choose C++ when you need that ecosystem or interop, Rust for greenfield safety-critical code, C for the smallest possible footprint.

### 1.5 The standards timeline — what "modern C++" means **[B]**

The ISO committee ships a new standard roughly every three years. Each adds features; compilers catch up over the following years. You select a standard with a compiler flag (`-std=c++23`). Knowing roughly what arrived when helps you read other people's code and target the right baseline.

| Standard | Year | Headline additions you'll actually use |
|---|---|---|
| **C++98/03** | 1998/2003 | The original STL, templates, exceptions — old style |
| **C++11** | 2011 | The "modern C++" reboot: `auto`, lambdas, `nullptr`, move semantics, `unique_ptr`/`shared_ptr`, range-`for`, `std::thread`, `constexpr`, `=default`/`=delete`, uniform `{}` init |
| **C++14** | 2014 | `make_unique`, generic lambdas, `constexpr` relaxed, return-type deduction |
| **C++17** | 2017 | `std::optional`, `std::variant`, `std::string_view`, `<filesystem>`, structured bindings, `if constexpr`, parallel algorithms, fold expressions |
| **C++20** | 2020 | **Concepts**, **ranges**, **modules**, **coroutines**, `std::format`, `std::span`, three-way `<=>`, `consteval`/`constinit`, designated initializers, `std::jthread` |
| **C++23** | 2024 (current) | `std::expected`, `std::print`/`<print>`, `std::mdspan`, `std::flat_map`, deducing `this`, `import std;`, ranges additions (`zip`, `enumerate`), `std::generator` |
| **C++26** | ~2026 (in progress) | ⚡ Static reflection, contracts, executors/senders, hardened std lib, more ranges — partial in GCC 15 / Clang 20 |

"Modern C++" colloquially means "C++11 and later, written in the idiomatic style" — RAII everywhere, `auto`, smart pointers, range-`for`, no raw `new`/`delete` in ordinary code. This guide is C++23-idiomatic throughout.

---

## 2. Setup, Build & Your First Program (g++/clang++/MSVC, CMake)

### 2.1 Installing a compiler **[B]**

C++ has three major production compilers, all free. You only need one, but understanding the landscape helps.

- **GCC** (`g++`) — the GNU Compiler Collection. The Linux default. On Windows you get it via **MSYS2/MinGW-w64** or **WSL**. As of 2026, GCC 14 is stable and GCC 15 adds more C++23/26.
- **Clang** (`clang++`) — the LLVM compiler. Excellent diagnostics, the sanitizers (§17), and `clang-format`/`clang-tidy`. Default on macOS (shipped as part of Xcode). Clang 18/19 in 2026.
- **MSVC** (`cl.exe`) — Microsoft's compiler, shipped with **Visual Studio 2022** (the free *Community* edition is fine). The native Windows toolchain; integrates with the Windows SDK and debugger.

⚡ **Version note (Windows):** the simplest path on Windows 11 is **Visual Studio 2022 Community** (gives you MSVC, the debugger, and CMake) *or* **MSYS2** (`pacman -S mingw-w64-ucrt-x86_64-gcc`) for a GCC toolchain. WSL2 with Ubuntu gives you a full Linux GCC/Clang environment, which is closest to production servers — recommended if you'll deploy to Linux. Verify with `g++ --version`, `clang++ --version`, or `cl` in the *Developer Command Prompt*.

### 2.2 Your first program — and why each part exists **[B]**

A C++ program is text compiled into an executable. Execution begins at the function `main`. Let's write the canonical first program two ways: the modern C++23 way (`std::print`) and the classic way (`std::cout`), and explain every token.

```cpp
// hello.cpp — compile: g++ -std=c++23 -Wall -Wextra hello.cpp -o hello
//             run (Linux/macOS): ./hello   (Windows): .\hello.exe

#include <print>   // C++23: std::print / std::println — formatted console output.
                   // ⚡ Version note: needs GCC 14+/Clang 18+ libc++ or MSVC 2022 17.10+.
                   // If your stdlib lacks <print>, use the <iostream> version below.

// main is the program entry point. `int` is the exit code returned to the OS
// (0 = success, non-zero = failure). You may write `int main()` with no args.
int main() {
    // std::println writes its arguments using a Python-like format string
    // ({} is a placeholder) and appends a newline. It is type-safe: the
    // compiler checks the format against the arguments (unlike C's printf).
    std::println("Hello, modern C++! 2 + 2 = {}", 2 + 2);
    return 0;   // optional in main — falling off the end of main implies `return 0;`
}
```

The classic form, which you will see everywhere and which works on every compiler today:

```cpp
// hello_iostream.cpp
#include <iostream>   // declares std::cout, std::cin, std::endl, etc.

int main() {
    // std::cout is the "character output" stream (the console).
    // operator<< ("put to") chains values onto the stream. std::endl writes
    // a newline AND flushes the buffer (use '\n' if you don't need the flush —
    // it's faster). Everything in the standard library lives in namespace std.
    std::cout << "Hello, classic C++! 2 + 2 = " << (2 + 2) << '\n';
    return 0;
}
```

Why `std::` everywhere? The standard library lives inside the **namespace** `std` to avoid name clashes (§3.7). You *can* write `using namespace std;` to drop the prefix, but **don't do it at file/global scope in real code** — it pulls thousands of names into your scope and causes ambiguity and silent bugs. Type `std::` or use targeted `using std::println;` declarations.

### 2.3 The compilation model — what actually happens **[B]**

Understanding the build pipeline demystifies most beginner errors ("undefined reference", "redefinition", "no such header").

1. **Preprocessing.** Lines starting with `#` are handled first by the *preprocessor*. `#include <print>` literally pastes the contents of that header into your file. `#define` does text substitution. This produces one big "translation unit."
2. **Compilation.** Each translation unit (`.cpp`) is compiled independently into an *object file* (`.o`/`.obj`) containing machine code plus a table of symbols it defines and symbols it needs.
3. **Linking.** The *linker* stitches all object files plus libraries together, resolving every "needs" against some "defines," producing the final executable. "Undefined reference to `foo`" means *no object file defined* `foo`.

This is why C++ traditionally splits code into **headers** (`.hpp` — declarations: "this function exists, here's its signature") and **source files** (`.cpp` — definitions: the actual body). Every `.cpp` that wants to call `foo` `#include`s the header to learn its signature; exactly one `.cpp` provides the body. **Modules** (§16) are the C++20 replacement for this header machinery, but headers still dominate in 2026.

### 2.4 Essential compiler flags **[B]**

Always compile with warnings on. The compiler catches a huge fraction of bugs *if you let it talk*.

```bash
# GCC / Clang — the flags you should ALWAYS use while learning and developing:
g++ -std=c++23 -Wall -Wextra -Wpedantic -g main.cpp -o app
#    │          │     │       │          │
#    │          │     │       │          └ -g: include debug info (for gdb/lldb)
#    │          │     │       └ -Wpedantic: warn on non-standard extensions
#    │          │     └ -Wextra: extra warnings beyond -Wall
#    │          └ -Wall: "all" the common warnings (despite the name, not literally all)
#    └ -std=c++23: select the language standard

# Optimized release build (turn ON optimization, turn OFF debug/asserts):
g++ -std=c++23 -O2 -DNDEBUG main.cpp -o app
#               │   └ define NDEBUG: disables assert() and many library debug checks
#               └ -O2: standard optimization level (-O3 is more aggressive; -Og for debug)

# Treat warnings as errors (do this in CI to keep the code clean):
g++ -std=c++23 -Wall -Wextra -Werror main.cpp -o app
```

MSVC equivalent (in the Developer Command Prompt): `cl /std:c++latest /W4 /EHsc main.cpp`. `/W4` is roughly `-Wall -Wextra`; `/EHsc` enables standard C++ exception handling; `/std:c++latest` selects the newest standard (use `/std:c++23` once your VS version supports it as a named option).

### 2.5 Multi-file builds and an intro to CMake **[B]**

Compiling one file by hand is fine; real projects have hundreds. **CMake** is the de-facto standard *build-system generator*: you describe your project in a `CMakeLists.txt`, and CMake generates the actual build files for your platform (Makefiles on Linux, a Visual Studio solution on Windows, Ninja files, etc.). You'll meet CMake in depth in §16; here's the minimum to build a project today.

```cmake
# CMakeLists.txt — the heart of a modern C++ project.
cmake_minimum_required(VERSION 3.28)        # 3.28+ for good C++23/modules support
project(MyApp VERSION 1.0 LANGUAGES CXX)    # CXX = C++

set(CMAKE_CXX_STANDARD 23)            # require C++23
set(CMAKE_CXX_STANDARD_REQUIRED ON)  # error if the compiler can't do C++23
set(CMAKE_CXX_EXTENSIONS OFF)        # use -std=c++23, not -std=gnu++23 (portable)

# Declare an executable named "app" built from these source files:
add_executable(app
    src/main.cpp
    src/widget.cpp
)
target_include_directories(app PRIVATE include)   # where to find headers

# Turn on warnings for our target (generator expressions pick per-compiler flags):
target_compile_options(app PRIVATE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wall;-Wextra;-Wpedantic>
    $<$<CXX_COMPILER_ID:MSVC>:/W4>
)
```

```bash
# The modern CMake workflow (out-of-source build into a "build/" directory):
cmake -S . -B build              # configure: generate build files from CMakeLists.txt
cmake --build build              # compile and link everything
./build/app                      # run (Windows: .\build\Debug\app.exe with MSVC)

# Pick a build type (CMake's optimization presets):
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release   # -O3 -DNDEBUG, single-config generators
```

---

## 3. Fundamentals: Types, auto, References, const/constexpr, Namespaces

### 3.1 The built-in types and why their sizes matter **[B]**

C++ is statically typed; every variable has a fixed type. The fundamental types map closely to hardware, which is part of why C++ is fast — but it also means integer types have *implementation-defined sizes* historically. In modern code, prefer the **fixed-width types** from `<cstdint>` when the exact size matters (file formats, network protocols, bit manipulation).

| Category | Types | Notes |
|---|---|---|
| Boolean | `bool` | `true`/`false`, 1 byte |
| Character | `char`, `char8_t` (C++20, UTF-8), `wchar_t` | `char` is 1 byte; signedness is implementation-defined |
| Signed integer | `short`, `int`, `long`, `long long` | At least 16/16/32/64 bits respectively |
| Unsigned integer | `unsigned int`, `size_t`, … | `size_t` is the type for sizes/indices |
| Fixed-width | `std::int32_t`, `std::uint64_t`, … | From `<cstdint>` — use these for portable exact sizes |
| Floating point | `float` (32-bit), `double` (64-bit), `long double` | `double` is the default; prefer it unless memory-bound |
| No value | `void` | "no type"; used for functions that return nothing |

```cpp
#include <cstdint>   // fixed-width integer types
#include <print>

int main() {
    int        count   = 42;          // platform int (≥16, usually 32 bits)
    std::int64_t big   = 9'000'000'000;   // ' is a digit separator (C++14), ignored
    double     ratio   = 3.14159;     // 64-bit float — the default for math
    bool       ok      = true;
    char       grade   = 'A';         // single quotes = a single char
    std::size_t n      = 10;          // unsigned; the type of container .size()

    // Integer division truncates toward zero; mixing signed/unsigned is dangerous (§20).
    std::println("7 / 2 = {}, 7.0 / 2 = {}", 7 / 2, 7.0 / 2);  // 3, 3.5
}
```

### 3.2 Initialization: prefer braces `{}` **[B]**

C++ has several initialization syntaxes for historical reasons. The modern recommendation is **brace (uniform) initialization** `{}`, introduced in C++11, because it works consistently everywhere *and* forbids **narrowing conversions** (silently losing data), catching a class of bugs.

```cpp
int    a = 5;       // copy-initialization — classic, fine for simple types
int    b{5};        // direct-list (brace) init — PREFERRED; no narrowing allowed
int    c(5);        // direct-init with parens
int    d{};         // brace init with no value → zero-initialized (d == 0). Useful!

// int  e{3.9};     // ERROR: narrowing double→int is forbidden by {}. Good — it caught a bug.
int    e = 3.9;     // compiles but SILENTLY truncates to 3 — the kind of bug {} prevents.

std::vector<int> v{1, 2, 3};   // braces also do "list initialization" of containers
```

A subtle gotcha (the "most vexing parse"): `Widget w();` declares a *function* named `w` returning `Widget`, not an object. `Widget w{};` is unambiguous. This is another reason to prefer braces.

### 3.3 `auto` — let the compiler deduce the type **[B]**

`auto` tells the compiler "figure out the type from the initializer." This is not dynamic typing — the type is still fixed at compile time; you're just not spelling it out. Use `auto` to avoid repeating verbose type names, to stay correct when types change, and because some types (lambdas, iterators) are unspellable. The rule of thumb (Herb Sutter's "Almost Always Auto"): use it when the type is obvious from the right-hand side or doesn't matter, write it explicitly when the exact type is load-bearing for the reader.

```cpp
auto i = 42;                       // int
auto x = 3.14;                     // double
auto name = std::string{"Ada"};    // std::string (note: "Ada" alone is const char*)
auto it = v.begin();               // some long iterator type you don't want to spell

// auto strips references and const by default. Add them back when you want them:
const auto& ref = some_big_object;  // bind a const reference — no copy (see §3.5)
auto&& fwd = make_thing();          // forwarding reference (§8)

// In modern C++, a common idiom for a typed literal:
auto count = 0uz;   // C++23: 'uz' suffix → std::size_t literal
```

### 3.4 `const`, `constexpr`, `consteval`, `constinit` — degrees of "constant" **[B]**

These keywords express *immutability* and *compile-time evaluation*, and they're central to safe, fast C++. Marking things `const` is one of the cheapest ways to make code safer: it documents intent, lets the compiler catch accidental mutation, and enables optimizations.

- **`const`** — "this value won't change after initialization." Applies to variables, references, member functions (a `const` method promises not to modify the object), and pointers. Use it *liberally*. "const-correctness" is a hallmark of good C++.
- **`constexpr`** (C++11, hugely expanded since) — "this *can* be evaluated at compile time." A `constexpr` variable is a compile-time constant; a `constexpr` function may run at compile time (if given constant inputs) or at runtime. Moves work from runtime to compile time → faster programs.
- **`consteval`** (C++20) — "this function *must* run at compile time" (an "immediate function"). Calling it with non-constant arguments is an error.
- **`constinit`** (C++20) — "this static/global must be initialized at compile time" (avoids the *static initialization order fiasco*, §20).

```cpp
const double PI = 3.141592653589793;   // runtime constant; cannot be reassigned

constexpr int square(int x) { return x * x; }   // usable at compile OR run time
constexpr int kArea = square(5);                // computed at COMPILE time → 25, a literal
int runtime = 7;
int r = square(runtime);                        // same function, computed at runtime

consteval int must_compile_time(int x) { return x + 1; }
constexpr int y = must_compile_time(10);        // OK, fully compile-time
// int z = must_compile_time(runtime);          // ERROR: arg isn't a constant expression
```

### 3.5 References vs pointers — the most important distinction in C++ **[B]**

Both let you refer to an object without copying it, but they have different semantics, and choosing correctly is core to writing clean C++.

A **reference** (`T&`) is an *alias* — another name for an existing object. It must be initialized when declared, can never be "reseated" to refer to something else, and can never be null. Think of it as the object itself, accessed through a different name. References are the normal way to pass large objects to functions without copying.

A **pointer** (`T*`) is a *variable that holds an address*. It can be null (`nullptr`), can be reassigned to point elsewhere, and you "dereference" it with `*p` to get the pointee or `p->member` to access members. Pointers are more powerful but more dangerous (null, dangling). In modern C++ you use **raw pointers only as non-owning "I'm looking at this, I don't own it" observers**; for *ownership* you use smart pointers (§7).

```cpp
int value = 10;

int& ref = value;   // ref is another name for `value`. Must init now; can't be null.
ref = 20;           // changes `value` to 20 (NOT "make ref point elsewhere")

int* ptr = &value;  // ptr holds the ADDRESS of value (& = "address of")
*ptr = 30;          // dereference: changes the pointed-to value (value is now 30)
ptr = nullptr;      // pointers can be null and reseated; references cannot

// Passing to functions — references avoid copies and express intent:
void scale(std::string& s);        // mutates the caller's string (in/out parameter)
void print(const std::string& s);  // reads it without copying (the everyday default)
void take(std::string s);          // takes a COPY (use when you need to own it)
```

**Rule of thumb:** pass small/cheap types (`int`, `double`) **by value**; pass big types you only read **by `const&`**; pass by **non-const `&`** when you intend to modify the caller's object; use raw pointers (or `std::optional<T&>`-style designs) when "no object" is a valid case. Never return a reference or pointer to a local variable (§20 — dangling).

### 3.6 Scope, lifetime & storage **[B]**

C++ objects have **automatic storage** (local variables — live on the stack, destroyed at end of scope), **static storage** (globals and `static` locals — live for the whole program), or **dynamic storage** (heap — created with `new`/smart pointers, live until freed). The crucial guarantee underpinning RAII: when a block `{ … }` ends, its automatic objects are destroyed **in reverse order of construction**, running their destructors. This determinism is what makes RAII work.

```cpp
void demo() {
    Logger log{"demo"};   // constructed here
    {
        std::vector<int> temp(1000);  // constructed
        // ... use temp ...
    }   // <- temp's destructor runs HERE, freeing its 1000 ints, exactly here
    // ... log still alive ...
}   // <- log's destructor runs HERE
```

### 3.7 Namespaces — organizing names **[B]**

A **namespace** is a named scope that groups related names and prevents collisions between libraries (two libraries can both define `Logger` if they're in different namespaces). The entire standard library is in `std`. Define your own to scope your project's code.

```cpp
namespace myapp {
    namespace net {                      // nested namespace
        int open_socket();
    }
    namespace fs = std::filesystem;      // namespace ALIAS — a short local nickname
}

// C++17 compact nested form:
namespace myapp::audio {
    void play();
}

// Using it:
int s = myapp::net::open_socket();       // fully qualified — always unambiguous
using myapp::net::open_socket;           // targeted using-declaration: now `open_socket()` works
// using namespace std;                  // AVOID at global scope; OK narrowly inside a function
```

---

## 4. Control Flow, Operators & Console I/O

Before functions and classes you need the everyday machinery every program is built from: the **operators** that compute and compare values, the **control-flow** statements that make decisions and repeat work, and **console I/O** to print output and read input. C++ inherits most of this from C, but modern C++ adds safer, clearer forms — the **range-based `for`**, **init-statements** in `if`/`switch`, and **`std::print`** (C++23) — that you should reach for first. (Types and variables came in §3; functions are next in §5.)

### 4.1 Operators **[B]**

An **operator** computes a value from one or more operands. The families you'll use constantly:

| Family | Operators | Notes |
|---|---|---|
| Arithmetic | `+ - * / %` | `/` on two `int`s does **integer** division (`7/2 == 3`); `%` is remainder (ints only) |
| Comparison | `== != < <= > >=` | yield `bool`; `<=>` (the "spaceship", §6) generates all six at once |
| Logical | `&& || !` | operate on `bool`; **short-circuit** — the right side is skipped if the left already decides it |
| Assignment | `= += -= *= /= %=` | `=` is assignment; `==` is comparison (mixing them up is a classic bug) |
| Increment/decrement | `++ --` | `++i` (pre) increments then yields; `i++` (post) yields the old value then increments — prefer `++i` |
| Bitwise | `& \| ^ ~ << >>` | operate on the bits of integers (flags, masks, low-level work) |
| Ternary (conditional) | `cond ? a : b` | an *expression* that yields `a` if `cond` is true, else `b` |

```cpp
#include <print>

int main() {
    int a = 7, b = 2;
    std::println("{} {} {} {}", a + b, a / b, a % b, a * b);   // 9 3 1 14  (note integer 7/2 == 3)

    bool ok = (a > 0) && (b != 0);     // short-circuit: if a>0 is false, b!=0 is never evaluated
    int max = (a > b) ? a : b;         // ternary: max = 7

    int flags = 0b0000;
    flags |= 0b0010;                   // set a bit with bitwise-OR  -> 0b0010
    bool isSet = flags & 0b0010;       // test a bit with bitwise-AND
    std::println("ok={} max={} isSet={}", ok, max, isSet);
}
```

> **⚡ Two precedence traps:** `&&` binds tighter than `||`, and bitwise `&`/`|` bind *looser* than comparisons — so `if (x & 1 == 0)` parses as `x & (1 == 0)`, almost never what you mean. **When in doubt, parenthesize.** And remember integer division truncates: write `double(a) / b` (or `a / 2.0`) when you want a fractional result.

### 4.2 Console I/O — `std::cout`, `std::cin` & `std::print` (C++23) **[B]**

For output, classic C++ uses **`std::cout`** with the **stream insertion operator `<<`**; modern C++23 adds **`std::print`/`std::println`** (from `<print>`), which use Python-style `{}` placeholders and are cleaner and type-safe — prefer them when your compiler supports them. For input, **`std::cin`** with the **extraction operator `>>`** reads whitespace-separated tokens; **`std::getline`** reads a whole line.

```cpp
#include <iostream>   // std::cout, std::cin
#include <print>      // std::print, std::println (C++23)
#include <string>

int main() {
    // --- Output ---
    int age = 30;
    std::cout << "Age is " << age << '\n';          // classic: chain with <<; use '\n' not std::endl
    std::println("Age is {}", age);                 // modern (C++23): placeholders, adds the newline

    // --- Input ---
    std::print("Enter your name: ");
    std::string name;
    std::getline(std::cin, name);                   // reads the WHOLE line (including spaces)

    std::print("Enter your age: ");
    int years{};
    std::cin >> years;                              // reads one integer token

    std::println("Hello {}, next year you'll be {}.", name, years + 1);
}
```

> **Gotchas:** (1) **`'\n'` vs `std::endl`** — both end a line, but `std::endl` *also flushes* the buffer, which is slow in a loop; use `'\n'` unless you specifically need a flush. (2) **Mixing `>>` and `getline`** — `std::cin >> x` leaves the trailing newline in the buffer, so a following `getline` reads an empty line; consume it with `std::cin.ignore()` or read everything with `getline` + parse. (3) `std::cin >> years` on bad input leaves the stream in a *failed* state — check `if (std::cin)` / `std::cin.fail()` and clear it before retrying. **⚡ Version note:** `std::print`/`std::println` are C++23; library support landed across GCC 14+/Clang 18+/recent MSVC but isn't universal — if unavailable, use `std::format` (C++20) into `std::cout`, or `<fmt>`.

### 4.3 Conditionals — `if`, `if` with an initializer & `switch` **[B]**

`if`/`else` branch on a `bool` condition. C++17 added the **init-statement** form `if (init; condition)`, which declares a variable scoped to the `if`/`else` — perfect for "compute something, then test it" without leaking the name. `switch` dispatches on an integral/enum value and is clearer than a long `if`/`else if` chain — but each `case` **falls through** to the next unless you `break`, so forgetting `break` is a classic bug (mark *intentional* fallthrough with `[[fallthrough]]`).

```cpp
#include <print>

int classify(int n) {
    if (n < 0) {
        return -1;
    } else if (n == 0) {
        return 0;
    }
    return 1;
}

int main() {
    // if with an initializer (C++17): `r` exists only inside this if/else
    if (int r = classify(-5); r < 0) {
        std::println("negative (code {})", r);
    } else {
        std::println("non-negative (code {})", r);
    }

    int day = 3;
    switch (day) {                       // dispatch on an integer/enum
        case 1:
        case 7:
            std::println("weekend");
            break;                       // REQUIRED, or it falls into the next case
        case 6:
            std::println("almost weekend");
            [[fallthrough]];             // explicit, intentional fall-through (silences the warning)
        case 2: case 3: case 4: case 5:
            std::println("weekday");
            break;
        default:
            std::println("invalid day");
    }
}
```

> **Best practice:** prefer the **init-statement form** to keep helper variables tightly scoped (smaller scope = fewer bugs, §3.6), always `break` each `switch` case unless you genuinely want fall-through (then say so with `[[fallthrough]]`), and always include a `default`. For booleans, write `if (ready)` not `if (ready == true)`.

### 4.4 Loops — `while`, `do`/`while`, `for` & the range-based `for` **[B]**

Four looping forms, each with a job. `while` repeats while a condition holds (zero or more times); `do`/`while` runs the body **at least once** then tests; the classic `for (init; cond; step)` is for counting; and the **range-based `for`** — the one you'll use most in modern C++ — iterates the elements of any container/array directly, with no index or bounds math to get wrong. `break` exits a loop; `continue` skips to the next iteration.

```cpp
#include <print>
#include <vector>
#include <string>

int main() {
    // while: test BEFORE each iteration (may run zero times)
    int n = 3;
    while (n > 0) { std::println("countdown {}", n); --n; }

    // do/while: body runs at least once, test AFTER
    int tries = 0;
    do { ++tries; } while (tries < 1);

    // classic counting for
    for (int i = 0; i < 5; ++i) {
        if (i == 2) continue;            // skip 2
        if (i == 4) break;               // stop before 4
        std::println("i = {}", i);       // prints 0, 1, 3
    }

    // range-based for — the modern default. `const auto&` avoids copying each element.
    std::vector<std::string> names{"Ada", "Linus", "Grace"};
    for (const auto& name : names) {     // read-only, no copy
        std::println("hello {}", name);
    }
    for (auto& name : names) {           // use a non-const ref to MODIFY in place
        name += "!";
    }

    // structured bindings in a range-for (great for maps / pairs):
    std::vector<std::pair<std::string,int>> scores{{"a", 1}, {"b", 2}};
    for (const auto& [key, value] : scores) {
        std::println("{} -> {}", key, value);
    }
}
```

> **Best practice:** reach for the **range-based `for`** by default — it can't go out of bounds and reads clearly. Iterate by **`const auto&`** to avoid copying each element, or **`auto&`** when you need to modify in place; use a plain index `for` only when you genuinely need the index or are stepping irregularly. Declare the loop variable in the smallest scope, and prefer `++i` to `i++` for non-trivial types (it avoids making a throwaway copy).

---

## 5. Functions: Overloading, Defaults, Lambdas, std::function, Templates Intro

### 5.1 Declaring and defining functions **[B]**

A function has a return type, a name, a parameter list, and a body. C++ lets you separate the **declaration** (signature, often in a header) from the **definition** (body, in a `.cpp`), which is how the compile/link model (§2.3) works.

```cpp
// Declaration (a "prototype") — tells callers the signature. Usually in a .hpp:
int add(int a, int b);

// Definition — the actual implementation. In a .cpp:
int add(int a, int b) { return a + b; }

// Trailing return type syntax (sometimes clearer, required with some templates):
auto multiply(int a, int b) -> int { return a * b; }
```

### 5.2 Overloading and default arguments **[B]**

**Overloading** lets multiple functions share a name as long as their parameter lists differ; the compiler picks the right one from the argument types. This is compile-time polymorphism — there's no runtime cost. **Default arguments** let callers omit trailing parameters.

```cpp
void log(int x);                 // overload 1
void log(double x);              // overload 2 — chosen for log(3.14)
void log(const std::string& x);  // overload 3

// Default arguments: callers may omit `base`.
int to_int(const std::string& s, int base = 10);
auto a = to_int("ff", 16);   // base = 16
auto b = to_int("42");       // base defaults to 10
```

A caution: only *trailing* parameters can have defaults, and defaults live in the declaration (header), not the definition. Overload resolution + implicit conversions can surprise you (§20) — when in doubt, make conversions explicit.

### 5.3 Lambdas and closures — the workhorses of modern C++ **[B/I]**

A **lambda** is an anonymous function you can write inline, right where you use it. Lambdas are everywhere in modern C++ because algorithms and ranges (§10) take callables. A lambda can **capture** variables from its surrounding scope, becoming a **closure** (a function bundled with some captured state). Understanding capture is essential and a common source of bugs.

The syntax is `[capture](parameters) -> return_type { body }`. The return type is usually deduced and omitted. The capture list controls *how* the lambda accesses outer variables:

- `[]` — captures nothing (a pure function of its parameters).
- `[x]` — capture `x` **by value** (a copy is stored inside the closure; later changes to the outer `x` don't affect it).
- `[&x]` — capture `x` **by reference** (the closure refers to the outer `x`; **danger**: if the outer `x` dies before the lambda runs, you have a dangling reference — see §20).
- `[=]` — capture everything used, by value. `[&]` — everything by reference.
- `[this]` — capture the enclosing object's `this` pointer (to use member variables).
- `[x = expr]` — *init capture* (C++14): create a new closure member initialized to `expr` (e.g. `[p = std::move(ptr)]` to move-capture).

```cpp
#include <algorithm>
#include <vector>
#include <print>

int main() {
    int threshold = 3;
    std::vector<int> nums{1, 2, 3, 4, 5};

    // A lambda capturing `threshold` by value, used by an STL algorithm.
    // count_if calls our lambda for each element; we return whether it qualifies.
    auto big = std::count_if(nums.begin(), nums.end(),
                             [threshold](int n) { return n > threshold; });
    std::println("{} numbers exceed {}", big, threshold);   // 2 numbers exceed 3

    // mutable lambda: by-value captures are const by default; `mutable` lets you
    // modify the COPY inside the closure (not the original outer variable).
    int counter = 0;
    auto tick = [counter]() mutable { return ++counter; };  // owns its own copy
    std::println("{} {} {}", tick(), tick(), tick());        // 1 2 3 (outer counter still 0)

    // Generic lambda (C++14): `auto` parameter → works for any type.
    auto twice = [](auto v) { return v + v; };
    std::println("{} {}", twice(21), twice(std::string{"ab"}));   // 42 abab
}
```

⚡ **Version note (C++23):** lambdas can now omit the empty `()` even when they have attributes, and you can write `[](this auto&& self){…}` with *deducing `this`* for recursive lambdas. Earlier, recursion required `std::function` or a Y-combinator trick.

### 5.4 `std::function` and passing callables around **[I]**

Each lambda has its own unique, unnameable type, which is great for performance (the compiler can inline it) but awkward when you want to *store* a callable in a variable or member, or accept "any callable with this signature" as a parameter. `std::function<R(Args...)>` is a **type-erased** wrapper that can hold any callable matching that signature — a lambda, a function pointer, a functor. It costs a small runtime indirection (and possibly a heap allocation), so prefer a template parameter (§5.5) in hot paths, but `std::function` is invaluable for callbacks, event handlers, and storing heterogeneous callables.

```cpp
#include <functional>
#include <vector>

// Store callbacks of a fixed signature — their concrete types differ, std::function unifies them:
std::vector<std::function<void(int)>> handlers;
handlers.push_back([](int x){ /* ... */ });
handlers.push_back([prefix = std::string{"id="}](int x){ /* uses captured prefix */ });
for (auto& h : handlers) h(42);

// Accept any callable (template version — zero overhead, preferred when you can use it):
template <typename Fn>
void for_each_id(const std::vector<int>& ids, Fn fn) {
    for (int id : ids) fn(id);
}
```

### 5.5 Function templates — a first taste of generics **[B/I]**

A **template** is a recipe the compiler uses to *generate* a concrete function (or class) for each type you use it with. This is how you write code once that works for `int`, `double`, `std::string`, and your own types — with zero runtime overhead, because a specialized version is compiled for each type. Templates are covered fully in §11; here's the gentle introduction.

```cpp
// `template <typename T>` introduces a type parameter T. The compiler stamps out
// a separate `max_of` for each T you call it with (int, double, ...) at compile time.
template <typename T>
T max_of(T a, T b) {
    return (a > b) ? a : b;   // requires that T supports operator>
}

auto m1 = max_of(3, 9);          // T deduced as int
auto m2 = max_of(2.5, 1.5);      // T deduced as double
auto m3 = max_of<std::string>("apple", "banana");   // explicit T

// C++20 abbreviated function template — `auto` parameters ARE a template:
auto min_of(auto a, auto b) { return (a < b) ? a : b; }
```

In §11 you'll learn to *constrain* templates with **concepts** so that misuse produces a clear error ("T must be comparable") instead of a wall of template gibberish.

---

## 6. Classes & RAII: The Rule of 0/3/5, Copy vs Move

This is the heart of safe C++. A **class** bundles data (member variables) with the functions that operate on them (member functions / methods) and controls access to that data (encapsulation). But the deeper reason classes matter in C++ is **RAII**: a class's constructor acquires a resource and its destructor releases it, so the language's automatic destruction guarantee (§3.6) gives you leak-free, exception-safe resource management for free.

### 6.1 A class, member by member **[I]**

```cpp
#include <string>
#include <print>

// `class` members are private by default; `struct` members are public by default.
// Use `struct` for plain data bundles, `class` when you maintain invariants.
class BankAccount {
public:                                   // accessible to everyone
    // Constructor: runs when an object is created. The `: balance_{...}` part is the
    // MEMBER INITIALIZER LIST — it initializes members directly (preferred over
    // assigning in the body, which would default-construct then overwrite).
    BankAccount(std::string owner, long cents)
        : owner_{std::move(owner)}, balance_{cents} {}

    // A const member function promises not to modify the object. Mark every
    // read-only method const — it's const-correctness and it enables more uses.
    long balance() const { return balance_; }

    void deposit(long cents) {
        if (cents < 0) throw std::invalid_argument{"negative deposit"};
        balance_ += cents;                // maintain the invariant: balance never set illegally
    }

private:                                  // accessible only inside the class — encapsulation
    std::string owner_;                   // trailing underscore: a common convention for members
    long        balance_;                 // money in integer cents — never use float for money
};
```

The **member initializer list** (after the `:`) is important: it constructs each member *once*, in declaration order. Initializing in the constructor body instead would default-construct the member and then assign to it — wasteful, and impossible for `const` or reference members. Always prefer the initializer list.

### 6.2 The destructor and RAII in action **[I]**

The **destructor** (`~ClassName()`) runs automatically when an object dies. This is where RAII lives: acquire in the constructor, release in the destructor. Here is the canonical example — a wrapper that owns a C-style file handle and guarantees it's closed exactly once, even if an exception is thrown:

```cpp
#include <cstdio>
#include <stdexcept>

// RAII wrapper around a C FILE*. This is what unique_ptr/fstream do internally.
class File {
public:
    // Acquire the resource in the constructor. If fopen fails we throw, and since
    // the object was never fully constructed, the destructor will NOT run — no leak.
    explicit File(const char* path, const char* mode) : fp_{std::fopen(path, mode)} {
        if (!fp_) throw std::runtime_error{"could not open file"};
    }

    // Release the resource in the destructor. Runs automatically at end of scope,
    // including during stack unwinding when an exception propagates. Guaranteed once.
    ~File() {
        if (fp_) std::fclose(fp_);
    }

    void write(std::string_view text) {
        std::fwrite(text.data(), 1, text.size(), fp_);
    }

    // Rule of 5 (next section): a class that OWNS a resource must say what copying
    // and moving mean. The safe default for a unique resource: forbid copy, allow move.
    File(const File&)            = delete;       // can't copy a file handle
    File& operator=(const File&) = delete;
    File(File&& other) noexcept : fp_{other.fp_} { other.fp_ = nullptr; }   // steal
    File& operator=(File&& other) noexcept {
        if (this != &other) { if (fp_) std::fclose(fp_); fp_ = other.fp_; other.fp_ = nullptr; }
        return *this;
    }

private:
    std::FILE* fp_;
};

void use() {
    File f{"out.txt", "w"};   // opens
    f.write("hello");
    // ... even if something here throws, f's destructor still closes the file ...
}   // <- f.~File() runs here, fclose called exactly once. No leak. No manual cleanup.
```

This pattern is the *entire game*. Every resource — memory, locks, sockets, GPU handles — gets a small RAII wrapper, and then you never leak. The standard library already provides wrappers for the common cases (`unique_ptr`, `fstream`, `lock_guard`, containers), so you'll rarely write your own.

### 6.3 The Rule of 0, 3, and 5 — the most important class-design rule **[I]**

When a class manages a resource, the compiler-generated copy/move operations are usually *wrong* (they'd copy a pointer, leading to double-free). C++ defines five "special member functions" that govern copying, moving, and destruction:

1. **Destructor** — `~T()`
2. **Copy constructor** — `T(const T&)`
3. **Copy assignment** — `T& operator=(const T&)`
4. **Move constructor** — `T(T&&) noexcept`
5. **Move assignment** — `T& operator=(T&&) noexcept`

The rules that govern them:

- **Rule of 0** (the goal — aim for this): if your class doesn't directly manage a raw resource — i.e. all its members are themselves RAII types like `std::string`, `std::vector`, `std::unique_ptr` — then **write none of the five.** The compiler-generated versions correctly delegate to the members' own correct versions. This is *by far* the most common and best case. **Most of your classes should declare none of the five.**
- **Rule of 3** (the old C++98 rule): if you need a custom destructor, copy constructor, *or* copy assignment, you almost certainly need all three, because needing one signals you manage a resource manually.
- **Rule of 5** (the modern extension): once you write any of the five, the others aren't auto-generated as you'd hope. So if you manage a resource directly, define (or `=default`/`=delete`) **all five** to be explicit and correct, including the two move operations for efficiency.

```cpp
// Rule of 0 — the IDEAL. This class owns a buffer, but via std::vector, so the
// compiler-generated copy/move/destroy are all correct. We write NONE of the five.
class Histogram {
public:
    void add(int bucket) { counts_.at(bucket)++; }
private:
    std::vector<int> counts_ = std::vector<int>(256, 0);  // RAII member does the work
};   // copyable, movable, leak-free — and we wrote zero boilerplate. Aim for this.
```

```cpp
// Rule of 5 — needed ONLY when you manage a raw resource directly. Here, raw heap.
// In real code you'd use unique_ptr and drop back to Rule of 0; this is for learning.
class Buffer {
public:
    explicit Buffer(std::size_t n) : size_{n}, data_{new int[n]{}} {}
    ~Buffer() { delete[] data_; }                                    // 1. destructor

    Buffer(const Buffer& o) : size_{o.size_}, data_{new int[o.size_]} {  // 2. copy ctor
        std::copy(o.data_, o.data_ + size_, data_);                  //    DEEP copy
    }
    Buffer& operator=(const Buffer& o) {                             // 3. copy assign
        if (this != &o) { Buffer tmp{o}; swap(tmp); }                //    copy-and-swap idiom
        return *this;
    }
    Buffer(Buffer&& o) noexcept : size_{o.size_}, data_{o.data_} {   // 4. move ctor
        o.data_ = nullptr; o.size_ = 0;                             //    STEAL, leave o empty
    }
    Buffer& operator=(Buffer&& o) noexcept {                         // 5. move assign
        if (this != &o) { delete[] data_; data_ = o.data_; size_ = o.size_;
                          o.data_ = nullptr; o.size_ = 0; }
        return *this;
    }
    void swap(Buffer& o) noexcept { std::swap(size_, o.size_); std::swap(data_, o.data_); }
private:
    std::size_t size_;
    int*        data_;
};
```

The takeaway: **don't write the Rule-of-5 boilerplate unless you must.** Use `std::vector`/`std::unique_ptr` members and live in Rule-of-0 paradise.

### 6.4 `=default`, `=delete`, and controlling copyability **[I]**

You can explicitly request the compiler's version (`= default`) or forbid an operation (`= delete`). This is how you express "this type is move-only" (like `unique_ptr`) or "this type is non-copyable and non-movable" (like a mutex).

```cpp
class NonCopyable {
public:
    NonCopyable() = default;                            // I want the default constructor
    NonCopyable(const NonCopyable&) = delete;           // forbid copying
    NonCopyable& operator=(const NonCopyable&) = delete;
    // (Move ops aren't generated once you declare copy ops, so this is move-disabled too.)
};
```

### 6.5 Operator overloading & three-way comparison `<=>` **[I]**

C++ lets a class define what operators (`+`, `==`, `<`, `[]`, `<<`) mean for it, so user types feel built-in. Overload operators *only when the meaning is obvious* (e.g. `+` for a `Vector2`); abusing them harms readability. The killer modern feature is the **three-way comparison operator** `<=>` (the "spaceship," C++20): define it once (often `= default`) and the compiler generates *all* of `<`, `<=`, `>`, `>=` consistently; pair it with a defaulted `==`.

```cpp
#include <compare>

struct Version {
    int major, minor, patch;
    // `= default` derives a lexicographic comparison from the members, in order.
    // This single line gives you <, <=, >, >=, and (with ==) full ordering.
    auto operator<=>(const Version&) const = default;
    bool operator==(const Version&) const = default;
};

bool b = (Version{1,2,0} < Version{1,3,0});   // true — generated for free
```

---

## 7. Resource Management & Smart Pointers

If you remember one rule from this guide, make it this: **in modern C++ you almost never write `new` or `delete`.** Raw owning pointers — a `T*` that you are responsible for `delete`-ing — are the single largest source of memory bugs (leaks, double-free, use-after-free). **Smart pointers** are RAII wrappers around a heap allocation: they own the object, and their destructor frees it automatically. They cost nothing extra (`unique_ptr` is the size of a raw pointer) and they make ownership *visible in the type system*.

### 7.1 Why heap allocation exists, and why to avoid it **[I]**

Most objects should live on the **stack** (as local variables) — that's cheap, automatic, and RAII handles cleanup. You only need the **heap** when an object must outlive the scope that created it, when its size isn't known until runtime (use `std::vector` for that), or for polymorphism (storing a `Derived` through a `Base` pointer). When you *do* need the heap, smart pointers manage it. The mental model: **ownership** answers "whose job is it to free this?" Smart pointers encode the answer in the type.

### 7.2 `std::unique_ptr` — exclusive ownership (your default) **[I]**

`std::unique_ptr<T>` owns its object exclusively: there is exactly one `unique_ptr` for that object, and when it's destroyed, the object is `delete`d. It is **move-only** (you can't copy it — that would mean two owners) but you can *move* ownership. It's the default smart pointer; reach for it first. Create it with `std::make_unique` (which is exception-safe and avoids writing `new`).

```cpp
#include <memory>

struct Widget {
    explicit Widget(int id) : id_{id} {}
    void draw() const { /* ... */ }
    int id_;
};

void demo() {
    // make_unique allocates a Widget on the heap and wraps it. No raw new.
    std::unique_ptr<Widget> w = std::make_unique<Widget>(42);
    w->draw();                 // use like a pointer: -> and *
    auto id = w->id_;

    // Cannot copy — this would be two owners of one object:
    // auto w2 = w;            // ERROR (deleted copy constructor)

    // CAN move — transfers ownership; afterward `w` is null:
    std::unique_ptr<Widget> w2 = std::move(w);
    // w is now nullptr; w2 owns the Widget.

}   // <- w2's destructor runs here, deletes the Widget. No leak. You wrote no delete.
```

`unique_ptr` is also how you store polymorphic objects and pass ownership across function boundaries cleanly. Pass it **by value** to transfer ownership (`void consume(std::unique_ptr<Widget> w)`), or pass the *pointee* by reference (`void use(Widget& w)`) when you just need to look at it without taking ownership.

### 7.3 `std::shared_ptr` — shared ownership (use sparingly) **[I/A]**

`std::shared_ptr<T>` allows **multiple owners**: it keeps a reference count, and the object is destroyed when the *last* `shared_ptr` to it goes away. This is convenient but not free — there's an atomic reference count (thread-safe to copy, which costs synchronization) and a separate "control block" allocation. **Default to `unique_ptr`; use `shared_ptr` only when ownership is genuinely shared** (e.g. an object referenced by several independent subsystems with no clear single owner). Create with `std::make_shared` (one allocation for both object and control block — more efficient than `shared_ptr<T>(new T)`).

```cpp
auto a = std::make_shared<Widget>(1);   // ref count = 1
{
    auto b = a;                         // COPY → both own it; ref count = 2
    b->draw();
}                                       // b destroyed → ref count back to 1
// object still alive because `a` owns it
// when `a` dies, ref count → 0, Widget deleted
```

### 7.4 `std::weak_ptr` — breaking reference cycles **[A]**

A pitfall of `shared_ptr`: if A holds a `shared_ptr` to B and B holds one back to A, their counts never reach zero — a **memory leak via reference cycle**. `std::weak_ptr<T>` is a *non-owning* observer of a `shared_ptr`-managed object: it doesn't affect the count and doesn't keep the object alive. To use it, you `lock()` it, which gives you a `shared_ptr` if the object still exists, or null if it was already destroyed. Use `weak_ptr` for back-references, caches, and observer patterns.

```cpp
struct Node {
    std::shared_ptr<Node> next;     // owns the next node
    std::weak_ptr<Node>   prev;     // observes the previous — NO ownership, breaks the cycle
};

void use(std::weak_ptr<Node> wp) {
    if (auto sp = wp.lock()) {      // lock() → shared_ptr if alive, else nullptr
        // safe to use sp here; the object is guaranteed alive while sp exists
    } else {
        // the object was already destroyed
    }
}
```

### 7.5 Ownership models — a decision table **[I]**

| You need… | Use | Why |
|---|---|---|
| A local object | a plain value on the stack | cheapest; RAII cleans up |
| A dynamic array / buffer | `std::vector<T>` / `std::string` | RAII container, not a smart pointer |
| One owner, heap object | `std::unique_ptr<T>` | zero overhead, clear ownership |
| Several genuine co-owners | `std::shared_ptr<T>` | ref-counted shared lifetime |
| A non-owning back-reference into shared data | `std::weak_ptr<T>` | breaks cycles, detects expiry |
| To just *look at* an object you don't own | `T&`, `const T&`, or raw `T*` | non-owning observer; never `delete` it |
| Optional non-owning view of a contiguous range | `std::span<T>` | see §9 |

The golden rules: **owning raw pointers: never. `new`/`delete` in application code: never** (let `make_unique`/containers do it). Raw pointers and references are fine *as non-owning observers*. If you find yourself writing `delete`, ask what RAII type should own that instead.

---

## 8. Move Semantics & Value Categories

Move semantics (C++11) is the feature that lets C++ keep clean **value semantics** (copying is the default, objects own their data) *without* the performance cost of copying large objects when a copy is unnecessary. It's also the mechanism behind `unique_ptr`'s "transfer ownership." Understanding it removes a lot of mystery from modern code.

### 8.1 The problem moves solve **[I]**

Consider returning a big `std::vector` from a function, or inserting a temporary `std::string` into a container. With only copying, you'd duplicate the entire buffer and then immediately throw the original away — pure waste. A **move** instead *steals* the internal buffer from the soon-to-die source object (just copies a pointer and a size, then nulls the source) — O(1) instead of O(n). The source is left in a valid-but-unspecified "moved-from" state (typically empty). Moving is an *optimization of copying* that the compiler applies automatically when it knows the source won't be used again.

### 8.2 Value categories: lvalue, rvalue, xvalue **[I/A]**

Every C++ expression has a **value category** that determines whether it can be moved from. The simplified model:

- **lvalue** ("locator value") — something with a name/identity you could take the address of: a variable, `obj.member`, `arr[i]`. It persists; you generally can't steal from it (it'll be used again).
- **rvalue** — a temporary with no name: the result of `a + b`, a literal, a function returning by value. It's about to disappear, so it's *safe to steal from*. (Formally split into prvalues and **xvalues** — "expiring values" — but the practical point is "temporary, safe to move from.")

`std::move(x)` doesn't move anything by itself — it's a **cast** that says "treat `x` as an rvalue; I promise I'm done with it," which lets a move constructor/assignment be selected. Misusing it (using `x` after `std::move(x)`) is a bug.

```cpp
std::string a = "hello";
std::string b = a;              // COPY: `a` is an lvalue, still needed → duplicated
std::string c = a + "!";        // MOVE: `a + "!"` is a temporary (rvalue) → buffer stolen
std::string d = std::move(a);   // MOVE: we promise we're done with `a` → buffer stolen.
                                // `a` is now in a valid-but-unspecified (empty) state.
```

### 8.3 Move constructors and `noexcept` **[I/A]**

Recall the Rule of 5 (§6.3): the move constructor `T(T&& other) noexcept` and move assignment steal `other`'s resources. Marking them `noexcept` is important: containers like `std::vector`, when they grow and must relocate their elements, will only *move* them (instead of copying) if the move is `noexcept` — otherwise they copy, to preserve the strong exception guarantee. **Always mark your moves `noexcept`.**

### 8.4 Perfect forwarding and forwarding references **[A]**

When you write a *generic* wrapper (e.g. `make_unique`, or a factory), you want to pass arguments through to a constructor *preserving their value category* — an lvalue stays an lvalue (copy), an rvalue stays an rvalue (move). This is **perfect forwarding**, achieved with a **forwarding reference** (`T&&` in a deduced template context) plus `std::forward`.

```cpp
#include <utility>

// `Args&&...` here are FORWARDING references (because Args is deduced), not rvalue refs.
// std::forward<Args>(args)... preserves each argument's original value category, so
// lvalues are copied and rvalues are moved into T's constructor — no extra copies.
template <typename T, typename... Args>
std::unique_ptr<T> my_make_unique(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}
```

The rule of thumb: use `std::move` when you have a *named* object you're done with; use `std::forward<T>` only on a *forwarding reference parameter* in a template. Don't `std::move` a return value of a local variable — the compiler already does **RVO** (next).

### 8.5 Copy elision / RVO — moves you don't even pay for **[I/A]**

The compiler is allowed (and since C++17, *required* in many cases) to **elide** copies/moves entirely via **Return Value Optimization (RVO)** and **NRVO**: when you return a local object by value, it's constructed directly in the caller's storage — zero copies, zero moves. This is why "return by value" in C++ is efficient and idiomatic; you should *not* write `return std::move(local);` (it actually *disables* NRVO and is a pessimization).

```cpp
std::vector<int> make_data() {
    std::vector<int> v(1'000'000);   // built once
    // ... fill v ...
    return v;                        // RVO: constructed directly in the caller. No copy/move.
}
auto data = make_data();             // `data` IS the vector built inside the function
```

---

## 9. STL Containers: vector, string, map, set & Friends

The **Standard Template Library (STL)** gives you battle-tested, RAII-managed, generic containers and algorithms. Using them instead of hand-rolled arrays and pointers is most of what makes C++ safe and productive. They own their memory, clean up automatically, grow as needed, and compose with algorithms and ranges (§10).

### 9.1 `std::vector` — the default container **[I]**

`std::vector<T>` is a **dynamic, contiguous array**: elements live back-to-back in one heap block (cache-friendly), it grows automatically, and indexing is O(1). It is the container you should reach for **by default** — only use something else when you have a specific reason. Because the data is contiguous, iterating it is extremely fast (great cache locality).

```cpp
#include <vector>
#include <print>

int main() {
    std::vector<int> v;            // empty
    v.push_back(10);               // append (amortized O(1))
    v.push_back(20);
    v.emplace_back(30);            // construct in place (avoids a temporary)

    std::vector<int> w{1, 2, 3, 4};     // initializer-list construction
    std::vector<int> z(100, 7);         // 100 elements, each 7

    // Access: operator[] is fast but UNCHECKED (out-of-range = undefined behavior).
    // .at() does bounds-checking and throws std::out_of_range — safer, slightly slower.
    int first = w[0];
    int safe  = w.at(2);           // throws if index invalid

    // PERFORMANCE: if you know the final size, reserve() to avoid repeated reallocations.
    std::vector<int> big;
    big.reserve(1'000'000);        // one allocation up front instead of ~20 regrowths
    for (int i = 0; i < 1'000'000; ++i) big.push_back(i);

    // Range-for is the idiomatic way to iterate. `const auto&` avoids copies.
    for (const auto& x : w) std::println("{}", x);

    std::println("size={}, capacity>={}", w.size(), w.capacity());
}
```

A key gotcha: operations that grow a vector (`push_back`, `insert`) may **reallocate**, which **invalidates all iterators, pointers, and references** into it (§20). `reserve()` mitigates this and also avoids wasted reallocations.

### 9.2 `std::array` and `std::string` **[I]**

`std::array<T, N>` is a **fixed-size** array (size known at compile time) with the safety and interface of a container — prefer it over raw C arrays (`int a[10]`), which decay to pointers and lose their size. `std::string` is a RAII-owning, growable text buffer (like a `vector<char>` with string operations).

```cpp
#include <array>
#include <string>

std::array<int, 4> a{1, 2, 3, 4};      // size fixed at compile time, lives on the stack
auto n = a.size();                     // knows its own size, unlike a raw array

std::string s = "Hello";
s += ", world";                        // grows automatically
s.append("!");
bool has = s.contains("world");        // C++23
auto sub = s.substr(0, 5);             // "Hello"
```

### 9.3 `std::string_view` and `std::span` — non-owning views **[I]**

These two C++17/20 types are *views*: they refer to data they don't own, storing just a pointer and a length. They make passing read-only data cheap and avoid copies, but you must ensure the underlying data outlives the view (a dangling view is UB — §20).

- **`std::string_view`** — a read-only view of a character sequence (a `std::string`, a string literal, a substring) with the string-query interface but no allocation. Take `std::string_view` parameters for functions that only *read* a string, so callers can pass `std::string`, `const char*`, or a literal with zero copies.
- **`std::span<T>`** — a view over a contiguous sequence (`vector`, `array`, C array). Take `std::span<const T>` parameters for functions that read a range regardless of how it's stored.

```cpp
#include <string_view>
#include <span>

// Accepts std::string, const char*, or a literal — no copy made:
std::size_t count_vowels(std::string_view text) {
    std::size_t n = 0;
    for (char c : text) if (std::string_view{"aeiou"}.find(c) != std::string_view::npos) ++n;
    return n;
}

// Works for vector, array, or C array of ints — no copy, knows its length:
int sum(std::span<const int> xs) {
    int total = 0;
    for (int x : xs) total += x;
    return total;
}
```

### 9.4 Associative containers: maps and sets **[I]**

These store keys (and optionally values) and support fast lookup. There are two flavors: **ordered** (`std::map`/`std::set`, a balanced tree — keys kept sorted, O(log n) operations) and **unordered** (`std::unordered_map`/`std::unordered_set`, a hash table — O(1) average, no ordering). Prefer the `unordered_` versions when you don't need sorted iteration and your key has a good hash; use the ordered versions when you need keys in sorted order or range queries.

```cpp
#include <map>
#include <unordered_map>
#include <set>

std::unordered_map<std::string, int> ages;     // hash map: fast lookup, no order
ages["Ada"]  = 36;                              // insert or overwrite
ages["Alan"] = 41;
if (auto it = ages.find("Ada"); it != ages.end())   // find avoids inserting on miss
    std::println("Ada is {}", it->second);
ages.contains("Bob");                           // C++20 membership test

// Structured bindings (C++17) make iterating maps clean:
for (const auto& [name, age] : ages)
    std::println("{} -> {}", name, age);

std::map<int, std::string> ordered;             // tree map: iterates in key order
std::set<int> unique_sorted{3, 1, 2, 1};        // {1,2,3} — duplicates dropped, sorted
```

⚡ **Version note (C++23):** `std::flat_map`/`std::flat_set` store their data in sorted contiguous vectors — much better cache behavior and iteration speed than `std::map` for read-heavy workloads, at the cost of slower insertion. Reach for them when you build a map once and query it often.

### 9.5 Other containers & a complexity cheat-sheet **[I]**

`std::deque` (double-ended queue — fast push/pop at both ends), `std::list` (doubly-linked list — O(1) splice/insert anywhere but poor cache behavior; rarely the right choice), `std::stack`/`std::queue`/`std::priority_queue` (adapters over other containers). Pick by your access pattern:

| Container | Lookup by key | Index access | Insert/erase | Memory layout | Reach for it when… |
|---|---|---|---|---|---|
| `vector` | — | O(1) | O(1) at end, O(n) middle | contiguous (cache-friendly) | **default**; sequence, iterate, index |
| `array` | — | O(1) | fixed size | contiguous, on stack | fixed compile-time size |
| `deque` | — | O(1) | O(1) both ends | chunked | queue with index access |
| `list` | — | O(n) | O(1) anywhere (with iterator) | nodes (cache-hostile) | frequent middle splice (rare) |
| `map` / `set` | O(log n) | — | O(log n) | tree nodes | sorted keys, range queries |
| `unordered_map`/`set` | O(1) avg | — | O(1) avg | hash table | fast lookup, order irrelevant |
| `flat_map`/`set` (C++23) | O(log n) | — | O(n) | contiguous | build once, query often |

The practical heuristic: **use `vector` unless you have a measured reason not to.** Contiguous memory beats theoretically-better Big-O on modern hardware surprisingly often (§19).

---

## 10. Iterators, Algorithms & Ranges (C++20)

The STL's containers and algorithms are connected by **iterators** — objects that generalize "a position in a sequence." An algorithm like `std::sort` doesn't know whether it's sorting a `vector` or a `deque`; it works through iterators. C++20 **ranges** then put a far more ergonomic, composable layer on top. This is where C++ starts to feel expressive.

### 10.1 Iterators and the half-open range `[begin, end)` **[I]**

A container exposes `begin()` (an iterator to the first element) and `end()` (an iterator *one past the last*). The range `[begin, end)` is **half-open**: `end` is a sentinel you never dereference. This convention makes empty ranges natural (`begin == end`) and loop bounds clean. Iterators come in **categories** of increasing power: *input/output* (single-pass), *forward*, *bidirectional* (`list`, `map` — can go `--`), and *random-access* (`vector`, `array` — can jump `it + n`). Algorithms document which category they need.

```cpp
std::vector<int> v{5, 3, 1, 4, 2};
for (auto it = v.begin(); it != v.end(); ++it) {   // classic iterator loop
    *it *= 10;                                     // dereference to access the element
}
```

### 10.2 The classic `<algorithm>` library **[I]**

`<algorithm>` and `<numeric>` provide dozens of generic algorithms that operate on iterator ranges: sorting, searching, transforming, accumulating, partitioning. Using them is clearer and less bug-prone than handwritten loops, and they're heavily optimized. They typically take a range and often a *predicate* lambda.

```cpp
#include <algorithm>
#include <numeric>
#include <vector>

std::vector<int> v{5, 3, 1, 4, 2};

std::sort(v.begin(), v.end());                           // {1,2,3,4,5}
std::sort(v.begin(), v.end(), std::greater<>{});         // {5,4,3,2,1} — custom comparator

bool any_even = std::any_of(v.begin(), v.end(), [](int x){ return x % 2 == 0; });
auto it       = std::find(v.begin(), v.end(), 3);        // search
int  total    = std::accumulate(v.begin(), v.end(), 0);  // sum, starting from 0

std::vector<int> doubled(v.size());
std::transform(v.begin(), v.end(), doubled.begin(), [](int x){ return x * 2; });

// Erase-remove idiom (pre-C++20) → simplified by std::erase_if (C++20):
std::erase_if(v, [](int x){ return x % 2 == 0; });       // remove all evens in one call
```

### 10.3 Ranges and views (C++20) — the modern way **[I/A]**

The classic algorithms require passing `begin()`/`end()` pairs and don't compose. **Ranges** (`<ranges>`, C++20) fix both: the `std::ranges::` versions take a whole container directly, and **views** (`std::views::`) are lazy, composable adaptors you chain with the pipe operator `|`. A view doesn't copy or own data — it describes a transformation that's applied lazily as you iterate. This lets you write declarative data pipelines that read top-to-bottom and cost nothing extra (no intermediate containers).

```cpp
#include <ranges>
#include <vector>
#include <print>

namespace rg = std::ranges;
namespace vw = std::views;

int main() {
    std::vector<int> nums{1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // A lazy pipeline: keep evens, square them, take the first 3. Nothing is computed
    // until we iterate; no intermediate vectors are allocated.
    auto pipeline = nums
        | vw::filter([](int n){ return n % 2 == 0; })   // 2 4 6 8 10
        | vw::transform([](int n){ return n * n; })      // 4 16 36 64 100
        | vw::take(3);                                   // 4 16 36

    for (int x : pipeline) std::println("{}", x);        // computed lazily here

    // Range algorithms take the container directly — no begin()/end():
    rg::sort(nums);
    bool found = rg::binary_search(nums, 7);

    // C++23 range adaptors: enumerate (index+value) and zip (pair up two ranges):
    std::vector<std::string> names{"Ada", "Alan", "Grace"};
    for (auto [i, name] : vw::enumerate(names))          // ⚡ C++23
        std::println("{}: {}", i, name);
}
```

⚡ **Version note:** `std::views::enumerate`, `zip`, `chunk`, `slide`, and `std::ranges::to` (materialize a view into a container) are **C++23**; you need GCC 14+/Clang 18+. `std::generator` (a coroutine-based lazy range) is also C++23. Most C++20 views (`filter`, `transform`, `take`, `drop`, `reverse`, `split`) work on any recent compiler.

### 10.4 `std::format` and `std::print` — type-safe text output **[I]**

`std::format` (C++20) is a type-safe, extensible replacement for `printf` and stream chaining, modeled on Python's `str.format`. `std::print`/`std::println` (C++23) write a formatted string straight to a stream/stdout efficiently. You can teach `std::format` how to print your own types by specializing `std::formatter`.

```cpp
#include <format>
#include <print>     // ⚡ C++23 — fall back to std::cout << std::format(...) if unavailable

int main() {
    std::string s = std::format("{} + {} = {}", 2, 3, 2 + 3);   // "2 + 3 = 5"

    // Format spec mini-language: width, alignment, precision, base, etc.
    std::println("{:>8}", "right");        // right-aligned in width 8
    std::println("{:.2f}", 3.14159);       // 3.14 — two decimals
    std::println("{:#06x}", 255);          // 0x00ff — hex, zero-padded, width 6
    std::println("{0} {1} {0}", "a", "b"); // positional args: a b a
}
```

---

## 11. Templates, Concepts & Generic Programming

Templates are how C++ does **generic programming**: writing code once that works for *any* type that supports the operations you use, with the compiler generating a specialized, fully-optimized version per type. This is the basis of the entire STL. Pre-C++20, templates were powerful but produced cryptic errors; **concepts** (C++20) tame them by letting you state requirements on type parameters clearly.

### 11.1 Class templates **[A]**

Just as functions can be templates (§5.5), so can classes. `std::vector<T>`, `std::unique_ptr<T>`, and `std::map<K,V>` are all class templates. You parameterize a class over types (or even values).

```cpp
// A minimal generic container, parameterized over its element type T.
template <typename T>
class Stack {
public:
    void push(T value) { data_.push_back(std::move(value)); }
    T pop() {
        T top = std::move(data_.back());   // move out the last element
        data_.pop_back();
        return top;                        // RVO returns it cheaply
    }
    bool empty() const { return data_.empty(); }
    std::size_t size() const { return data_.size(); }
private:
    std::vector<T> data_;                  // Rule of 0: vector handles all resource mgmt
};

Stack<int> s;          s.push(1); s.push(2);
Stack<std::string> w;  w.push("hi");
```

You can also have **non-type template parameters** (values, not types): `template <typename T, std::size_t N> class FixedArray;` — that's how `std::array<T, N>` knows its size at compile time.

### 11.2 Concepts (C++20) — constraints that make templates usable **[A]**

A **concept** is a named, compile-time predicate on types: "a type satisfies `Sortable` if it supports `<` and can be swapped," etc. Constraining a template with a concept does two things: it documents intent, and it makes misuse produce a *clear* error ("constraint not satisfied: T does not support operator<") instead of a 200-line template instantiation backtrace. The standard library ships many concepts in `<concepts>` (`std::integral`, `std::floating_point`, `std::same_as`, `std::convertible_to`, …).

```cpp
#include <concepts>

// Use a standard concept inline with `requires`, or as a constrained `auto`/typename.
template <std::integral T>                 // T must be an integer type
T gcd(T a, T b) {
    while (b) { T t = b; b = a % b; a = t; }
    return a;
}
// gcd(12, 18) → OK;   gcd(1.5, 2.0) → clear error: "1.5 is not integral"

// Define your own concept: "Addable" = supports a + a yielding the same type.
template <typename T>
concept Addable = requires(T a, T b) {
    { a + b } -> std::convertible_to<T>;   // the expression must compile and be convertible
};

template <Addable T>
T sum(const std::vector<T>& xs) {
    T total{};
    for (const auto& x : xs) total = total + x;
    return total;
}
```

Concepts replace the old **SFINAE** (`std::enable_if`) trickery for constraining overloads — far more readable. You can still see SFINAE in older code; concepts are the modern way.

### 11.3 Variadic templates **[A]**

A **variadic template** accepts any number of arguments of any types (a *parameter pack*). This is how `std::make_unique`, `std::format`, `std::tuple`, and printing helpers accept arbitrary arguments. C++17 **fold expressions** make processing the pack concise.

```cpp
#include <print>

// Accept any number of arguments and print them, space-separated, then newline.
// `Args... args` is the pack; `(expr, ...)` is a fold expression over the comma operator.
template <typename... Args>
void log_line(const Args&... args) {
    ((std::print("{} ", args)), ...);   // C++17 fold: expands to print(a) , print(b) , ...
    std::println("");
}
log_line("x =", 42, "pi =", 3.14);      // x = 42 pi = 3.14

// A sum over a pack using a fold over operator+:
template <typename... Ns>
auto add_all(Ns... ns) { return (ns + ...); }   // (n1 + n2 + ... + nk)
auto total = add_all(1, 2, 3, 4);                // 10
```

### 11.4 Type traits and `if constexpr` **[A]**

**Type traits** (`<type_traits>`) are compile-time queries and transformations on types: `std::is_integral_v<T>`, `std::is_same_v<A,B>`, `std::remove_const_t<T>`, etc. Combined with `if constexpr` (C++17) — a compile-time `if` that *discards the untaken branch entirely* — you can write one template that compiles different code for different types.

```cpp
#include <type_traits>

template <typename T>
std::string describe(T value) {
    if constexpr (std::is_integral_v<T>) {        // compiled only when T is integral
        return std::format("integer {}", value);
    } else if constexpr (std::is_floating_point_v<T>) {
        return std::format("float {:.3f}", value);
    } else {
        return std::format("other: {}", value);   // compiled only for the remaining types
    }
}
```

### 11.5 CRTP — static polymorphism **[A]**

The **Curiously Recurring Template Pattern** has a base class take the derived class as a template parameter (`class Derived : public Base<Derived>`). It enables compile-time ("static") polymorphism: the base can call the derived's methods with no virtual-call overhead. Use it for mixins and policy injection where you want polymorphic-looking code without the runtime cost of virtual functions (§13).

```cpp
template <typename Derived>
struct Printable {
    void print() const {
        // Cast to the derived type and call its method — resolved at compile time, inlined.
        std::println("{}", static_cast<const Derived&>(*this).to_string());
    }
};

struct Money : Printable<Money> {
    long cents;
    std::string to_string() const { return std::format("${}.{:02}", cents / 100, cents % 100); }
};

Money{12345}.print();   // $123.45 — no virtual dispatch
```

---

## 12. Error Handling: Exceptions, std::expected, std::optional

C++ offers several error-handling strategies, and modern code mixes them deliberately: **exceptions** for truly exceptional, hard-to-recover failures; **`std::expected`/`std::optional`** for expected, recoverable outcomes you want callers to handle explicitly. The unifying enabler is RAII: because destructors run during stack unwinding, cleanup is automatic regardless of which strategy you use.

### 12.1 Exceptions and exception safety **[I]**

An **exception** is thrown with `throw` and caught by a matching `catch` up the call stack. When thrown, the stack **unwinds**: every automatic object between the `throw` and the `catch` is destroyed (its destructor runs). *This is why RAII and exceptions are partners* — your resources are released automatically as the exception propagates, with no `finally` block needed.

```cpp
#include <stdexcept>

double safe_divide(double a, double b) {
    if (b == 0.0) throw std::invalid_argument{"division by zero"};
    return a / b;
}

void caller() {
    std::vector<int> v(1000);          // RAII: freed automatically if anything throws
    try {
        double r = safe_divide(10, 0);
    } catch (const std::invalid_argument& e) {   // catch by const reference (no slicing, no copy)
        std::println("error: {}", e.what());
    } catch (const std::exception& e) {           // base class catches all std exceptions
        std::println("other error: {}", e.what());
    }
}   // <- v is destroyed here whether or not an exception occurred
```

**Exception-safety guarantees** describe what state your code leaves things in if an exception escapes, from strongest to weakest:

- **No-throw (`noexcept`)** — the operation never throws. Destructors, swaps, and moves should aim for this.
- **Strong guarantee** — if it throws, nothing changed (the operation is all-or-nothing, like a transaction). Achieved with the *copy-and-swap* idiom.
- **Basic guarantee** — if it throws, no leaks and all invariants hold, but the state may have changed.
- **No guarantee** — avoid; this is broken code.

Mark functions that can't throw `noexcept` (it enables optimizations and is required for moves used by containers, §8.3). Never let an exception escape a destructor — if a destructor throws during unwinding, the program calls `std::terminate`.

### 12.2 `std::optional` — "a value, or nothing" **[I]**

`std::optional<T>` (C++17) represents "maybe a `T`." Use it for functions that may legitimately have *no result* (a lookup that finds nothing, parsing that yields no value) — far better than sentinel values (`-1`, null pointers) or output parameters because the "no value" case is in the type and the compiler makes you handle it.

```cpp
#include <optional>

std::optional<int> parse_int(std::string_view s) {
    try { return std::stoi(std::string{s}); }
    catch (...) { return std::nullopt; }       // no value
}

if (auto n = parse_int("42")) {                // contextually converts to bool
    std::println("got {}", *n);                // dereference to access the value
}
int port = parse_int(cfg).value_or(8080);      // supply a default when empty
```

### 12.3 `std::expected` — "a value, or an error" (C++23) **[I/A]**

⚡ **Version note (C++23):** `std::expected<T, E>` holds *either* a success value of type `T` *or* an error of type `E`. It's the modern answer to "I want explicit, exception-free error handling that carries error information" — the same idea as Rust's `Result<T, E>` (see [Rust](RUST_GUIDE.md)). It's ideal for expected failures in performance-sensitive or exception-averse code, and it forces the caller to consider the error path. Needs GCC 13+/Clang 16+/MSVC 17.6+.

```cpp
#include <expected>

enum class ParseError { Empty, NotANumber, OutOfRange };

std::expected<int, ParseError> to_int(std::string_view s) {
    if (s.empty()) return std::unexpected{ParseError::Empty};      // return an error
    try {
        return std::stoi(std::string{s});                          // return a value
    } catch (const std::out_of_range&) {
        return std::unexpected{ParseError::OutOfRange};
    } catch (...) {
        return std::unexpected{ParseError::NotANumber};
    }
}

void use() {
    auto r = to_int("123");
    if (r) {                                   // has a value?
        std::println("value: {}", *r);         // access with * or .value()
    } else {
        std::println("error code: {}", static_cast<int>(r.error()));
    }

    // Monadic operations (C++23) chain without nested ifs — transform on success,
    // short-circuit on error:
    auto doubled = to_int("21").transform([](int x){ return x * 2; });   // expected<int,...>
}
```

**When to use which:** use **exceptions** for programming errors and rare, catastrophic failures where unwinding to a high-level handler is appropriate (out of memory, corrupt state); use **`optional`** when absence is normal and you don't need an error reason; use **`expected`** when failure is expected, frequent, and you want to convey *why* without the cost/control-flow of exceptions. Don't use exceptions for ordinary control flow.

---

## 13. Inheritance, Polymorphism & std::variant

Inheritance lets one class extend another, and **runtime polymorphism** (via `virtual` functions) lets you treat different derived types uniformly through a base interface. It's powerful but easy to overuse — modern C++ often prefers *composition*, templates (§11), or `std::variant` over deep inheritance hierarchies. Know the tool; reach for it deliberately.

### 13.1 Virtual functions and abstract interfaces **[I/A]**

A **`virtual`** function can be overridden by a derived class, and a call through a base pointer/reference dispatches to the *actual* derived type at runtime (via a vtable). An **abstract class** has at least one *pure virtual* function (`= 0`) and can't be instantiated — it defines an interface. Always mark overrides with `override` (the compiler then verifies you actually override something) and give polymorphic base classes a **virtual destructor** (so `delete base_ptr` runs the derived destructor — otherwise it's undefined behavior and a leak).

```cpp
#include <memory>
#include <vector>

// Abstract interface: a Shape can compute its area and describe itself.
class Shape {
public:
    virtual ~Shape() = default;                 // VIRTUAL destructor — essential for a base
    virtual double area() const = 0;            // pure virtual → Shape is abstract
    virtual std::string name() const = 0;
};

class Circle : public Shape {
public:
    explicit Circle(double r) : r_{r} {}
    double area() const override { return 3.141592653589793 * r_ * r_; }  // override checked
    std::string name() const override { return "circle"; }
private:
    double r_;
};

class Square final : public Shape {             // `final`: no one may derive from Square
public:
    explicit Square(double s) : s_{s} {}
    double area() const override { return s_ * s_; }
    std::string name() const override { return "square"; }
private:
    double s_;
};

void report(const std::vector<std::unique_ptr<Shape>>& shapes) {
    for (const auto& shp : shapes)              // store polymorphic objects via unique_ptr<Base>
        std::println("{}: area={:.2f}", shp->name(), shp->area());   // dynamic dispatch
}

// Usage:
std::vector<std::unique_ptr<Shape>> shapes;
shapes.push_back(std::make_unique<Circle>(2.0));
shapes.push_back(std::make_unique<Square>(3.0));
report(shapes);
```

### 13.2 Object slicing — a classic trap **[I/A]**

If you store or pass a derived object **by value** as its base type, the derived part is "sliced off" — you get only the base, and virtual dispatch is lost. This is why polymorphic objects are handled through **pointers or references** (or `unique_ptr<Base>`), never by base value.

```cpp
Circle c{2.0};
Shape s = c;        // SLICING BUG: only the Shape part is copied; c's circle-ness is lost
// Always use Shape&, const Shape&, or unique_ptr<Shape> for polymorphism.
```

### 13.3 Prefer composition over inheritance **[I/A]**

Inheritance expresses an "is-a" relationship and couples the derived type tightly to the base. Often what you actually want is "has-a" — **composition** — which is more flexible and avoids fragile hierarchies. Inherit to model a genuine subtype and to share an interface; compose (hold a member) to reuse functionality. The standard guidance: *use inheritance for interfaces, composition for implementation reuse.*

### 13.4 `std::variant` and `std::visit` — closed sets without inheritance **[I/A]**

When you have a *fixed, known set* of alternative types (not an open-ended hierarchy), `std::variant<A, B, C>` (C++17) is a type-safe union: it holds exactly one of the listed types at a time. `std::visit` applies a callable to whichever type is currently held. This is value-semantic, allocation-free, and often a better fit than inheritance for things like AST nodes, parse results, or state machines — the same role *sum types/enums* play in [Rust](RUST_GUIDE.md).

```cpp
#include <variant>

struct Circle  { double r; };
struct Square  { double s; };
struct Rect    { double w, h; };

using Shape = std::variant<Circle, Square, Rect>;   // exactly one of these, by value

double area(const Shape& shp) {
    // Visit with an overload set (one lambda per alternative). The compiler checks
    // you handle EVERY alternative — exhaustiveness, like a match.
    return std::visit([](const auto& x) -> double {
        using T = std::decay_t<decltype(x)>;
        if constexpr (std::is_same_v<T, Circle>) return 3.14159 * x.r * x.r;
        else if constexpr (std::is_same_v<T, Square>) return x.s * x.s;
        else return x.w * x.h;
    }, shp);
}

Shape s = Square{3.0};
double a = area(s);     // 9.0 — no virtual calls, no heap, value semantics
```

---

## 14. File System, OS & Processes

Real programs read and write files, inspect directories, query the environment, and run other programs. C++17's `<filesystem>` gives you portable path and directory operations; `<fstream>` handles file I/O; environment access and process launching dip into the C/OS layer.

### 14.1 Paths and the filesystem library **[I]**

`std::filesystem` (namespace `std::filesystem`, commonly aliased `fs`) provides a portable `path` type and operations for files and directories that work across Windows and POSIX. The `path` type handles separators for you (`/` vs `\`) and composes with `operator/`. ⚡ It throws `std::filesystem::filesystem_error` on failure; most functions have a `std::error_code` overload if you prefer error codes over exceptions.

```cpp
#include <filesystem>
#include <print>

namespace fs = std::filesystem;

void explore() {
    fs::path dir = fs::current_path();          // where the program runs
    fs::path file = dir / "data" / "log.txt";   // operator/ joins portably (handles \ on Windows)

    std::println("exists: {}", fs::exists(file));
    if (fs::exists(file)) {
        std::println("size: {} bytes", fs::file_size(file));
        std::println("stem: {}, ext: {}", file.stem().string(), file.extension().string());
    }

    fs::create_directories(dir / "out" / "logs");   // mkdir -p, creates intermediates

    // Iterate a directory (recursively here). Skips into subdirectories automatically.
    for (const auto& entry : fs::recursive_directory_iterator{dir}) {
        if (entry.is_regular_file())
            std::println("{} ({} bytes)", entry.path().string(), entry.file_size());
    }

    fs::copy_file(file, dir / "backup.txt", fs::copy_options::overwrite_existing);
    fs::rename(dir / "backup.txt", dir / "old.txt");
    // fs::remove(dir / "old.txt");
}
```

⚡ **Version note (Windows):** paths are UTF-16 (`wchar_t`) natively on Windows and UTF-8 (`char`) on POSIX. Use `path::string()` for a narrow string (may lose non-ASCII on Windows) or `path::u8string()`/native `wstring()` when you need to preserve Unicode. The `fs::path` literals and `operator/` handle the `\` vs `/` difference so you don't hard-code separators.

### 14.2 Reading and writing files with streams **[I]**

`<fstream>` provides `std::ifstream` (input), `std::ofstream` (output), and `std::fstream` (both). They are **RAII**: opening happens in the constructor, the file is closed automatically in the destructor when the stream goes out of scope. Always check that the stream opened successfully.

```cpp
#include <fstream>
#include <sstream>
#include <string>

// Write a text file:
void write_file(const fs::path& p) {
    std::ofstream out{p};                       // opens for writing (truncates). RAII closes it.
    if (!out) throw std::runtime_error{"cannot open for writing"};
    out << "line 1\n" << "value = " << 42 << '\n';
}   // <- file flushed and closed here automatically

// Read a whole text file into a string:
std::string read_all(const fs::path& p) {
    std::ifstream in{p};
    if (!in) throw std::runtime_error{"cannot open for reading"};
    std::ostringstream ss;
    ss << in.rdbuf();                           // slurp the entire file buffer
    return ss.str();
}

// Read line by line (the idiomatic loop):
void each_line(const fs::path& p) {
    std::ifstream in{p};
    std::string line;
    while (std::getline(in, line)) {            // returns false at EOF
        std::println("{}", line);
    }
}

// Binary I/O: open with std::ios::binary and use read/write on raw bytes:
void write_binary(const fs::path& p, std::span<const std::byte> bytes) {
    std::ofstream out{p, std::ios::binary};
    out.write(reinterpret_cast<const char*>(bytes.data()), bytes.size());
}
```

### 14.3 Environment variables, arguments, and system info **[I]**

Command-line arguments arrive via `main(int argc, char* argv[])`. Environment variables come from `std::getenv` (`<cstdlib>`). There's no fully-portable standard API for OS/CPU details, but a few standard pieces are available.

```cpp
#include <cstdlib>
#include <thread>

int main(int argc, char* argv[]) {
    // Command-line arguments: argv[0] is the program name; argv[1..argc-1] are args.
    for (int i = 0; i < argc; ++i)
        std::println("arg[{}] = {}", i, argv[i]);

    // Environment variable (returns nullptr if not set — check before use):
    if (const char* home = std::getenv("HOME"))      // "USERPROFILE" on Windows
        std::println("home: {}", home);

    // Hardware concurrency (number of logical cores) — a hint, may return 0:
    std::println("cores: {}", std::thread::hardware_concurrency());
}
```

### 14.4 Running external commands / processes **[I]**

The only standard, portable way to run a command is `std::system` (`<cstdlib>`), which runs a string through the OS shell and returns its exit status. It's convenient but limited (no easy output capture, shell-injection risk if you interpolate untrusted input). For real process control (capturing stdout, pipes, async), you use the OS API (`CreateProcess` on Windows, `fork`/`exec`/`posix_spawn` on POSIX) or a library like **Boost.Process** / the proposed `std::process`.

```cpp
#include <cstdlib>

void run_commands() {
    // Portable but coarse: runs via the shell, returns the exit code.
    int rc = std::system("git status");          // ⚠ never pass untrusted input — injection risk
    if (rc != 0) std::println("command failed with code {}", rc);

    // Capture output portably with popen/_popen (POSIX/Windows), reading the pipe:
    // (popen is not ISO C++ but is widely available; wrap it in RAII.)
    std::string capture_command(const char* cmd);   // see below
}

// RAII wrapper to capture a command's stdout via popen:
#include <cstdio>
#include <array>
std::string capture_command(const char* cmd) {
#ifdef _WIN32
    FILE* pipe = _popen(cmd, "r");
#else
    FILE* pipe = popen(cmd, "r");
#endif
    if (!pipe) throw std::runtime_error{"popen failed"};
    std::string out;
    std::array<char, 4096> buf{};
    while (std::fgets(buf.data(), buf.size(), pipe)) out += buf.data();
#ifdef _WIN32
    _pclose(pipe);
#else
    pclose(pipe);
#endif
    return out;
}
```

For anything beyond toy use, prefer a maintained process library over hand-rolling — process management is full of platform-specific edge cases. See also the [Linux Server Administration](LINUX_SERVER_ADMIN_GUIDE.md) guide for the OS-level view.

---

## 15. Concurrency: threads, mutex, atomic, async, coroutines

C++ has a full memory model and threading library (since C++11) for true parallelism. Concurrency is genuinely hard: the cardinal sin is the **data race** — two threads accessing the same memory, at least one writing, without synchronization — which is *undefined behavior* in C++ (the program may do anything). The whole discipline is about preventing data races via synchronization, and RAII helps enormously by making lock management automatic.

### 15.1 Threads and `std::jthread` **[A]**

`std::thread` (C++11) runs a callable on a new OS thread. Its catch: you **must** `join()` (wait for it) or `detach()` it before it's destroyed, or the program terminates. `std::jthread` (C++20) fixes this — it **joins automatically** in its destructor (RAII!) and supports cooperative cancellation via a `std::stop_token`. **Prefer `jthread`.**

```cpp
#include <thread>
#include <print>

void worker(int id) { std::println("worker {} running", id); }

void run() {
    // jthread joins automatically when it goes out of scope — RAII for threads.
    std::jthread t1{worker, 1};
    std::jthread t2{[]{ std::println("lambda thread"); }};

    // Cooperative cancellation: the callable receives a stop_token it can poll.
    std::jthread loop{[](std::stop_token st) {
        while (!st.stop_requested()) { /* do work */ }
    }};
    loop.request_stop();      // ask it to finish

}   // <- t1, t2, loop all joined here automatically. No std::terminate, no manual join.
```

### 15.2 Mutexes and RAII locks **[A]**

A **mutex** ("mutual exclusion") ensures only one thread at a time enters a critical section. You should *never* lock and unlock a mutex by hand — an early return or exception would skip the unlock and deadlock. Instead use RAII lock guards: `std::lock_guard` (simple scope-based lock), `std::scoped_lock` (locks one *or more* mutexes deadlock-free — prefer it), and `std::unique_lock` (flexible: deferred/timed/movable, needed for condition variables).

```cpp
#include <mutex>
#include <vector>

class ThreadSafeCounter {
public:
    void increment() {
        std::scoped_lock lock{mutex_};   // RAII: locks here, unlocks when `lock` dies
        ++count_;
    }                                    // <- unlocked automatically, even if body threw
    int get() const {
        std::scoped_lock lock{mutex_};
        return count_;
    }
private:
    mutable std::mutex mutex_;           // `mutable` so const methods can lock it
    int count_ = 0;
};
```

### 15.3 `std::atomic` — lock-free synchronization **[A]**

For a single variable shared between threads, a `std::atomic<T>` provides indivisible reads/writes/updates without a mutex — operations can't be interleaved or torn. Atomics are the building block of lock-free data structures and are perfect for counters and flags. The default memory ordering (`seq_cst`, sequentially consistent) is the easiest to reason about; relaxed orderings are an expert optimization with subtle rules — don't reach for them without studying the memory model.

```cpp
#include <atomic>

std::atomic<int> hits{0};
void on_request() { hits.fetch_add(1, std::memory_order_relaxed); }   // safe concurrent ++
int total() { return hits.load(); }

std::atomic<bool> done{false};      // a flag other threads can poll safely
```

### 15.4 Futures, promises, and `std::async` **[A]**

For "run this and give me the result later," `std::async` launches a task and returns a `std::future<T>` you can `.get()` (which blocks until the result is ready, and rethrows any exception the task threw). A `std::promise`/`std::future` pair lets one thread hand a value to another. This is higher-level than raw threads and composes cleanly with exceptions.

```cpp
#include <future>

int expensive(int n) { /* ... heavy work ... */ return n * n; }

void use() {
    // std::launch::async forces a new thread; the default may run lazily on .get().
    std::future<int> fut = std::async(std::launch::async, expensive, 21);
    // ... do other work concurrently ...
    int result = fut.get();          // blocks until ready; rethrows if expensive() threw
    std::println("result: {}", result);
}
```

### 15.5 Condition variables **[A]**

A `std::condition_variable` lets threads *wait efficiently* until some condition becomes true (e.g. a queue has an item), instead of busy-polling. It pairs with a `unique_lock`; always wait with a predicate to guard against spurious wakeups.

```cpp
#include <condition_variable>
#include <queue>

std::mutex m;
std::condition_variable cv;
std::queue<int> q;

void producer(int v) {
    { std::scoped_lock lk{m}; q.push(v); }
    cv.notify_one();                 // wake one waiting consumer
}
void consumer() {
    std::unique_lock lk{m};
    cv.wait(lk, []{ return !q.empty(); });   // releases lock & sleeps until predicate true
    int v = q.front(); q.pop();
}
```

### 15.6 A word on coroutines (C++20) **[A]**

⚡ **Version note (C++20):** **coroutines** are functions that can *suspend* and *resume* — using `co_await`, `co_yield`, `co_return`. They enable elegant asynchronous code (await I/O without blocking a thread) and lazy generators without explicit state machines. The C++20 standard provides only the low-level *machinery*; the ergonomic high-level types are arriving over C++23/26 (`std::generator` is C++23; general async tasks still rely on libraries like cppcoro, or the C++26 senders/receivers). For most concurrency today, threads + `async` + a thread pool are the practical tools; reach for coroutines when you have heavy async I/O.

```cpp
#include <generator>   // ⚡ C++23

// A lazy infinite sequence — co_yield suspends and resumes on each iteration.
std::generator<int> fibonacci() {
    int a = 0, b = 1;
    while (true) {
        co_yield a;                  // produce a value, then suspend until the next request
        auto next = a + b; a = b; b = next;
    }
}
// for (int x : fibonacci() | std::views::take(10)) std::println("{}", x);
```

---

## 16. Modern Build & Dependency Management (CMake, vcpkg, Conan, Modules)

C++ historically had no standard build tool or package manager (unlike Rust's cargo or Go's toolchain). The community converged on **CMake** for builds and **vcpkg**/**Conan** for packages. Mastering these is essential for real-world C++; the language is only half the job.

### 16.1 CMake in depth **[A]**

CMake describes your project in `CMakeLists.txt` and generates native build files. **Modern CMake** (3.x, "target-based") revolves around *targets* (libraries and executables) and propagating requirements between them via `target_*` commands with `PUBLIC`/`PRIVATE`/`INTERFACE` visibility. `PRIVATE` = "I need this to build but my users don't"; `INTERFACE` = "I don't need it but my users do"; `PUBLIC` = both.

```cmake
cmake_minimum_required(VERSION 3.28)
project(MyApp VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# A reusable library target built from sources:
add_library(core
    src/engine.cpp
    src/parser.cpp
)
# Consumers of `core` get its include dir (INTERFACE/PUBLIC); the build needs it too:
target_include_directories(core PUBLIC include)
# `core` needs threads to build AND link; expose it as PUBLIC so consumers link it too:
find_package(Threads REQUIRED)
target_link_libraries(core PUBLIC Threads::Threads)

# The application links against our library — it inherits core's include dirs automatically:
add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE core)

# Enable testing and pull in a test executable (see §18):
enable_testing()
add_subdirectory(tests)
```

Use **CMake presets** (`CMakePresets.json`) to capture configure/build settings (compiler, build type, generator) so everyone builds identically:

```bash
cmake --preset debug            # uses a named preset from CMakePresets.json
cmake --build --preset debug
ctest --preset debug            # run the tests
```

### 16.2 Dependency management: vcpkg and Conan **[A]**

Rather than vendoring third-party source by hand, use a package manager:

- **vcpkg** (Microsoft) — a C/C++ package manager that builds dependencies from source and integrates with CMake via a *toolchain file*. In **manifest mode**, you list dependencies in a `vcpkg.json` and vcpkg installs them per-project. Excellent CMake integration.
- **Conan** (JFrog) — a more flexible package manager with binary caching and a Python-based recipe system; popular in larger/enterprise setups. Declares deps in `conanfile.txt`/`conanfile.py`.

```jsonc
// vcpkg.json — declare dependencies; vcpkg fetches & builds them.
{
  "name": "myapp",
  "version": "1.0.0",
  "dependencies": [ "fmt", "spdlog", "nlohmann-json", { "name": "gtest" } ]
}
```

```bash
# Configure CMake to use vcpkg's toolchain so find_package() finds the deps:
cmake -S . -B build \
  -DCMAKE_TOOLCHAIN_FILE=/path/to/vcpkg/scripts/buildsystems/vcpkg.cmake
```

Then in CMake: `find_package(fmt CONFIG REQUIRED)` and `target_link_libraries(app PRIVATE fmt::fmt)`.

### 16.3 Modules (C++20) and their state **[A]**

⚡ **Version note:** **modules** are C++20's replacement for the `#include` header model. Instead of textually pasting headers (slow, fragile, order-dependent), you `export` a module and `import` it — faster builds, real encapsulation, no macro leakage. The headline win is `import std;` (C++23), which imports the entire standard library as one fast module. **Reality check for 2026:** module support is real but still maturing across compilers and build tools — MSVC is furthest along, GCC 14/15 and Clang 18/19 have growing support, and CMake's module support (`CMAKE_CXX_MODULE_STD`) is improving but has rough edges. Use modules for new internal code if your toolchain is current; headers remain the safe, universal default for libraries that must build everywhere.

```cpp
// math.ixx (MSVC) / math.cppm (Clang/GCC convention) — a module interface unit.
export module math;             // declares the module named `math`

export int add(int a, int b) {  // `export` makes this visible to importers
    return a + b;
}
int helper() { return 0; }      // not exported → private to the module

// main.cpp — consumes the module:
import math;                    // no #include, no header file needed
// import std;                  // ⚡ C++23: the whole standard library, one import
int main() { return add(2, 3); }
```

### 16.4 Library structure & ABI **[A]**

A well-laid-out C++ library separates a public `include/` directory (headers consumers use) from private `src/`, ships a CMake *package config* so downstream `find_package()` works, and is careful about **ABI** (Application Binary Interface — the binary contract of compiled code). Changing a class's layout, virtual table, or inline function breaks ABI, meaning consumers must recompile. Header-only libraries sidestep ABI entirely (everything is recompiled with the consumer) at the cost of build time. More on layout in §18.

---

## 17. Production-Grade C++: Sanitizers, clang-tidy, Core Guidelines

Modern C++ can be as safe as garbage-collected languages *if you use the tooling*. The compiler's warnings are the first line; sanitizers, static analysis, and the Core Guidelines are the rest. Treating these as mandatory, not optional, is what separates hobby C++ from production C++.

### 17.1 The sanitizers — runtime bug detectors **[A]**

**Sanitizers** are compiler instrumentation (GCC/Clang) that catch memory and concurrency bugs *at runtime* with precise diagnostics. They're nearly magic for finding the bugs C++ is infamous for. Compile and *run your tests* with them on; the slowdown (2–20×) is fine for testing.

| Sanitizer | Flag | Catches |
|---|---|---|
| **ASan** (Address) | `-fsanitize=address` | use-after-free, buffer overflow (heap/stack/global), use-after-return, leaks |
| **UBSan** (Undefined Behavior) | `-fsanitize=undefined` | signed overflow, invalid casts, misaligned access, null deref, bad enum values |
| **TSan** (Thread) | `-fsanitize=thread` | data races, lock-order inversions |
| **LSan** (Leak) | `-fsanitize=leak` | memory leaks (often bundled with ASan) |
| **MSan** (Memory) | `-fsanitize=memory` (Clang) | reads of uninitialized memory |

```bash
# Build tests with ASan + UBSan (the everyday combo; don't mix ASan and TSan in one build):
g++ -std=c++23 -O1 -g -fsanitize=address,undefined -fno-omit-frame-pointer tests.cpp -o tests
./tests        # any UB or memory error now aborts with a detailed stack trace

# Data races (separate build):
g++ -std=c++23 -O1 -g -fsanitize=thread mt_tests.cpp -o mt_tests && ./mt_tests
```

**Valgrind** (specifically its `memcheck` tool) is an alternative leak/UB detector that needs no recompilation but is much slower; sanitizers are preferred when you can rebuild, valgrind when you can't.

### 17.2 Static analysis: clang-tidy and warnings **[A]**

**clang-tidy** is a static analyzer/linter that flags bug-prone patterns and Core-Guideline violations *without running the program*, and can auto-fix many. Configure it with a `.clang-tidy` file and run it in CI. `clang-format` (a `.clang-format` config) enforces consistent style automatically. The MSVC and Clang static analyzers (`/analyze`, `clang --analyze`) catch deeper logic bugs (null derefs, leaks on error paths).

```bash
clang-tidy src/*.cpp -- -std=c++23 -Iinclude    # analyze; add --fix to auto-apply fixes
clang-format -i src/*.cpp                        # reformat in place to your .clang-format
```

```yaml
# .clang-tidy — enable useful checks, treat them as errors in CI.
Checks: 'clang-analyzer-*,bugprone-*,modernize-*,performance-*,cppcoreguidelines-*'
WarningsAsErrors: 'bugprone-*,clang-analyzer-*'
```

### 17.3 The C++ Core Guidelines and GSL **[A]**

The **C++ Core Guidelines** (Stroustrup & Sutter) are the authoritative rules for writing safe, modern C++ — clang-tidy's `cppcoreguidelines-*` checks enforce many of them. The headline rules you should internalize:

- **R (Resources):** every resource has exactly one owner; prefer scoped objects; never `delete` a raw pointer in app code; use `unique_ptr`/`shared_ptr`.
- **F (Functions):** pass cheap types by value, big read-only ones by `const&`; return by value; avoid out-parameters.
- **ES (Expressions & Statements):** declare variables in the smallest scope, initialize on declaration, prefer `const`, avoid `reinterpret_cast` and C-casts.
- **Don't return/store dangling references; don't use `new`/`delete`; use `gsl::span` (now `std::span`) instead of pointer+length.**

The **GSL** (Guidelines Support Library) provides helpers the guidelines reference: `gsl::not_null<T*>` (a pointer that can't be null), `gsl::span` (predated `std::span`), `gsl::finally` (ad-hoc RAII cleanup), and `Expects`/`Ensures` (precondition/postcondition checks). Use `std::span` and standard facilities where they now exist; pull in GSL for the rest.

### 17.4 Secure coding: avoiding undefined behavior **[A]**

**Undefined behavior (UB)** is the central safety hazard in C++: the standard says certain operations have *no defined meaning*, so the compiler may assume they never happen and optimize accordingly — producing crashes, silent corruption, or security holes. Production C++ is largely the discipline of *never invoking UB*. The big sources:

- **Out-of-bounds access** — use `.at()`, `std::span`, range-`for`, and bounds checks; never index past the end.
- **Use-after-free / dangling** — let RAII and smart pointers own lifetimes; never return references to locals.
- **Uninitialized reads** — always initialize (`int x{};`, brace init).
- **Signed integer overflow** — is UB; use unsigned or check; UBSan catches it.
- **Invalid casts / type punning / data races / null deref** — avoid `reinterpret_cast`, use `std::bit_cast` (C++20), synchronize shared data, check pointers.

The defense in depth: compile with `-Wall -Wextra -Werror`, run tests under ASan+UBSan+TSan, lint with clang-tidy, follow the Core Guidelines, and prefer the safe standard facilities (containers, `span`, smart pointers) over raw memory operations. Do all of that and C++ is genuinely safe.

---

## 18. Testing & Maintainability (GoogleTest/Catch2, CI, Layout)

Untested C++ is dangerous C++ — the language gives you enough rope, and tests are how you keep it taut. A maintainable C++ project has automated tests, a clean directory layout, continuous integration, and a stable, documented API.

### 18.1 Unit testing with GoogleTest and Catch2 **[A]**

The two dominant frameworks are **GoogleTest** (gtest — feature-rich, includes a mocking library gmock, the de-facto industry standard) and **Catch2** (header-only or single-header, expressive `REQUIRE`/`CHECK` assertions, lightweight to adopt). Both integrate with CMake/CTest. Pick one and write tests for every non-trivial unit.

```cpp
// GoogleTest example — tests/test_math.cpp
#include <gtest/gtest.h>
#include "math.hpp"

TEST(AddTest, HandlesPositives) {
    EXPECT_EQ(add(2, 3), 5);          // non-fatal: continues on failure
    ASSERT_NE(add(2, 3), 0);          // fatal: aborts this test on failure
}
TEST(AddTest, HandlesNegatives) {
    EXPECT_EQ(add(-2, -3), -5);
}
// A test fixture shares setup across related tests:
class StackTest : public ::testing::Test {
protected:
    Stack<int> s;                     // fresh for each test
};
TEST_F(StackTest, PushPop) {
    s.push(42);
    EXPECT_EQ(s.pop(), 42);
    EXPECT_TRUE(s.empty());
}
```

```cpp
// Catch2 equivalent — note the natural-language style and BDD sections.
#include <catch2/catch_test_macros.hpp>
#include "math.hpp"

TEST_CASE("add works", "[math]") {
    REQUIRE(add(2, 3) == 5);
    SECTION("with negatives") { REQUIRE(add(-2, -3) == -5); }
}
```

Wire tests into CMake/CTest so `ctest` runs them all (and CI can gate merges on them):

```cmake
# tests/CMakeLists.txt
find_package(GTest CONFIG REQUIRED)               # provided by vcpkg/Conan
add_executable(unit_tests test_math.cpp)
target_link_libraries(unit_tests PRIVATE core GTest::gtest_main)
include(GoogleTest)
gtest_discover_tests(unit_tests)                  # registers each TEST with CTest
```

### 18.2 Project layout & continuous integration **[A]**

A conventional, scalable layout makes projects navigable and tooling-friendly:

```
myapp/
├── CMakeLists.txt          # top-level build
├── CMakePresets.json       # shared build configurations
├── vcpkg.json              # dependencies
├── include/myapp/          # PUBLIC headers (what consumers #include)
├── src/                    # implementation (.cpp) + private headers
├── tests/                  # unit/integration tests
├── benchmarks/             # performance tests (§19)
├── .clang-tidy
├── .clang-format
└── docs/                   # documentation
```

For **CI**, run the matrix that matters: multiple compilers (GCC, Clang, MSVC), Debug + Release, and a sanitizer build, with warnings-as-errors, on every push. See the [GitHub Actions CI/CD](GITHUB_ACTIONS_CICD_GUIDE.md) and [Git](GIT_GUIDE.md) guides for the pipeline mechanics; the essential C++ steps are: configure with CMake → build → `ctest` → run clang-tidy → run the ASan/UBSan test build.

### 18.3 API/ABI stability and documentation **[A]**

For a library others depend on, **API stability** (don't break source compatibility — keep function signatures, don't remove public names) and **ABI stability** (don't break binary compatibility — don't change class layouts or vtables in ways that force recompiles) matter. Techniques: prefer free functions over members where possible, use the **PIMPL idiom** (a pointer to an implementation struct, hiding members from the header so you can change them without breaking ABI), and version your API. Document public headers with **Doxygen** comments (`/** ... */`) so you can generate reference docs offline.

```cpp
// PIMPL: the public header exposes a stable interface; all data is hidden behind a pointer.
// Changing Impl's members later does NOT change Widget's size or break ABI.
class Widget {
public:
    Widget();
    ~Widget();                     // defined in .cpp where Impl is complete
    void render();
private:
    struct Impl;                   // forward-declared; defined only in the .cpp
    std::unique_ptr<Impl> pimpl_;  // the one stable member
};
```

---

## 19. Performance: Cache, Allocations, constexpr, Profiling

C++'s reason for existing is performance, but "fast" isn't automatic — it comes from understanding the machine and *measuring*. The cardinal rule: **profile before optimizing.** Intuition about what's slow is usually wrong; let a profiler tell you where the time actually goes, then optimize that.

### 19.1 Cache awareness — the dominant modern factor **[A]**

On modern CPUs, accessing main memory is ~100× slower than a cache hit, so **memory access patterns often matter more than instruction count.** This is why `std::vector` (contiguous) routinely beats `std::list` (scattered nodes) even when Big-O favors the list — the vector streams through cache while the list chases pointers across RAM. Practical implications: prefer contiguous containers, lay out data so you touch it sequentially, and favor **struct-of-arrays** over **array-of-structs** when you process one field of many objects in a hot loop (so the data you need is packed together).

### 19.2 Avoid needless allocations and copies **[A]**

Heap allocation is expensive; the fastest allocation is the one you don't do. Concrete wins:

```cpp
// 1. reserve() when you know the size — avoids repeated reallocations:
std::vector<int> v;
v.reserve(n);                 // one allocation instead of ~log2(n) regrowths

// 2. emplace_back instead of push_back(T{...}) — constructs in place, no temporary:
widgets.emplace_back(id, name);     // vs widgets.push_back(Widget{id, name})

// 3. Take string_view / span params to avoid copying when you only read (§9.3).

// 4. Move instead of copy for large objects you're done with (§8):
result.push_back(std::move(big_string));

// 5. Pass small types by value, large by const& (§3.5) — don't copy what you only read.
```

### 19.3 Compile-time computation with `constexpr` **[A]**

Work done at compile time costs *nothing* at runtime. Marking functions and data `constexpr` (and using `consteval` to *force* it) moves computation into the build. Lookup tables, parsing of compile-time-known config, and math on constants can all be precomputed. The C++23 standard library is extensively `constexpr`-enabled, so even container operations can run at compile time in `constexpr` contexts.

```cpp
// A lookup table computed entirely at compile time — zero runtime cost to build it.
constexpr std::array<int, 256> make_table() {
    std::array<int, 256> t{};
    for (int i = 0; i < 256; ++i) t[i] = i * i;   // runs during compilation
    return t;
}
constexpr auto kSquares = make_table();           // the data is baked into the binary
```

### 19.4 Profiling and benchmarking **[A]**

Measure with real tools: **perf** (Linux sampling profiler), **Visual Studio Profiler** / **VTune** (Windows/Intel), **Valgrind's callgrind** (instruction-level, slow but precise), and **google/benchmark** for microbenchmarks of specific functions. Always benchmark *optimized* (`-O2`/`-O3`) release builds — debug builds have wildly different performance. Beware microbenchmark pitfalls: the optimizer may delete code whose result you don't use (use `benchmark::DoNotOptimize`).

```cpp
#include <benchmark/benchmark.h>

static void BM_VectorSum(benchmark::State& state) {
    std::vector<int> v(state.range(0), 1);
    for (auto _ : state) {                       // the framework times this loop
        long long sum = std::accumulate(v.begin(), v.end(), 0LL);
        benchmark::DoNotOptimize(sum);           // stop the optimizer from removing the work
    }
}
BENCHMARK(BM_VectorSum)->Range(1 << 10, 1 << 20);   // sweep sizes 1K..1M
BENCHMARK_MAIN();
```

### 19.5 When zero-overhead isn't **[A]**

Abstractions are *mostly* free, but know the exceptions: **virtual calls** cost an indirection and inhibit inlining (use them for genuine polymorphism, not in tight loops — consider CRTP §11.5 or `std::variant` §13.4); `std::function` may allocate and adds indirection (prefer template callables in hot paths); `std::shared_ptr` has atomic refcount overhead (prefer `unique_ptr`); exceptions are zero-cost on the *happy path* but expensive when thrown (don't use them for control flow). The point is not to avoid abstractions — it's to know their cost and choose deliberately where it matters.

---

## 20. Gotchas & Best Practices

A consolidated list of the traps that bite C++ programmers, with the modern-C++ defense for each. Many are *undefined behavior* — meaning the program may appear to work until it catastrophically doesn't, so the fix is prevention, not debugging.

### 20.1 Dangling references, pointers, and iterators **[I/A]**

- **Returning a reference/pointer to a local** is UB — the local dies at function return. Return by value (RVO makes it cheap) or by a smart pointer.
- **A `string_view`/`span` outliving its data** dangles. Never store a view to a temporary: `std::string_view sv = get_string();` dangles if `get_string` returns by value.
- **Iterator/reference invalidation:** `push_back`/`insert` on a `vector` may reallocate, invalidating *all* iterators, pointers, and references into it. Erasing invalidates iterators at/after the erase point. Don't hold an iterator across a mutation; `reserve()` or re-fetch.

```cpp
const std::string& bad() { std::string s = "oops"; return s; }   // UB: dangling reference
std::string_view also_bad() { return std::string{"temp"}; }      // UB: view to a temporary
auto it = v.begin(); v.push_back(99); *it;                       // UB if push_back reallocated
```

### 20.2 Object slicing **[I/A]**

Assigning a derived object to a base *value* slices off the derived part and loses virtual dispatch (§13.2). Handle polymorphic objects only through references, pointers, or `unique_ptr<Base>`.

### 20.3 Integer and conversion pitfalls **[I/A]**

- **Mixing signed and unsigned** in comparisons/arithmetic causes surprising results (`-1 < 1u` is *false* because `-1` converts to a huge unsigned). Be consistent; enable `-Wsign-conversion`; prefer signed for arithmetic, `size_t` only for sizes.
- **Signed overflow is UB**; unsigned wraps. Don't rely on overflow behavior.
- **Narrowing conversions** silently lose data — use brace init `{}` to forbid them.
- **`size() - 1` on an empty container** underflows to a huge unsigned value — guard for empty.

### 20.4 Initialization traps **[I/A]**

- **Uninitialized variables** read garbage (UB). Always initialize: `int x{};`, `Widget w{};`.
- **Member initialization order** follows *declaration order in the class*, not the order in the initializer list — don't write a member initializer that depends on a later-declared member.
- **Static initialization order fiasco:** the init order of globals across different translation units is unspecified; one global depending on another may use it before it's constructed. Avoid global mutable state; use function-local statics (`static X& instance() { static X x; return x; }` — initialized on first use) or `constinit`.

### 20.5 `auto` and lambda pitfalls **[I/A]**

- **`auto` strips `const` and `&`** — `auto x = const_ref;` makes a non-const copy. Write `const auto&` to bind a reference without copying.
- **`auto` with proxy types:** `auto b = vec_of_bool[i];` captures `std::vector<bool>`'s proxy, not a `bool`. Be explicit for proxy-returning APIs.
- **Capturing by reference in a lambda that outlives the scope** dangles (§5.3). For lambdas stored or run later (callbacks, threads), capture by value or move; never `[&]` into a stored lambda.

### 20.6 The best-practices checklist **[I/A]**

| Do | Don't |
|---|---|
| RAII for every resource | manual `new`/`delete` in app code |
| `unique_ptr` by default, `shared_ptr` only when shared | raw owning pointers |
| Rule of 0 (let members manage resources) | hand-write the Rule of 5 unless you must |
| pass `const&` for big read-only params | pass big objects by value |
| return by value (RVO) | `return std::move(local)` (kills NRVO) |
| `{}` brace-init, init on declaration | leave variables uninitialized |
| `.at()`/`span`/range-`for` for bounds safety | unchecked `[]` past the end |
| `const`-correctness everywhere | mutable-by-default |
| mark moves/swaps/dtors `noexcept` | throw from destructors |
| `override`/`final`; virtual dtor on bases | forget the virtual destructor |
| compile `-Wall -Wextra -Werror`; run ASan/UBSan/TSan | ignore warnings |
| `std::expected`/`optional` for expected failures | exceptions for ordinary control flow |
| prefer the STL & ranges | hand-rolled loops and arrays |

---

## 21. Study Path & Build-to-Learn Projects

A concrete order to learn in, and projects that force you to *use* each concept until it's reflex. The fastest way to internalize modern C++ is to build small things that exercise RAII, smart pointers, value semantics, and the STL — and to run everything under the sanitizers from day one.

**Suggested learning order:**

1. **Foundations (§1–4):** install a compiler, build with `-std=c++23 -Wall -Wextra`, write programs using `auto`, references vs pointers, `const`, functions, and lambdas. Get comfortable with `std::print`/`std::format` and `std::vector`.
2. **The core of safe C++ (§6–7):** classes, RAII, the Rule of 0/3/5, smart pointers (`unique_ptr` first, then `shared_ptr`/`weak_ptr`), and move semantics. *Spend the most time here* — this is what separates modern C++ from C-with-classes.
3. **The STL (§9–9):** containers, then iterators, algorithms, and C++20 ranges. Learn to express logic as range pipelines rather than raw loops.
4. **Generics & errors (§11–11):** templates, concepts, and the trio of exceptions / `optional` / `expected`.
5. **Polymorphism, OS, concurrency (§13–14):** virtual functions vs `std::variant`, `<filesystem>` and file I/O, then threads/`jthread`/mutexes/`async`.
6. **Professional practice (§16–18):** CMake + vcpkg, sanitizers + clang-tidy + Core Guidelines, GoogleTest/Catch2 + CI, and profiling. This is what makes your C++ *production-grade*.

**Build-to-learn projects (in increasing difficulty):**

- **A RAII resource wrapper.** Write a class that owns a C resource (a `FILE*`, a socket, a `malloc`'d buffer) with a proper Rule-of-5 (move-only). Then *delete* it and replace it with `unique_ptr` + a custom deleter — feel how much boilerplate RAII saves. *Exercises:* §6, §7, §8.
- **A small `Vector<T>` container.** Implement a growable contiguous container: `push_back` with capacity doubling, `reserve`, iterators, copy/move (Rule of 5), strong exception safety. Run it under ASan/UBSan. *Exercises:* §6–§11, §17. This single project teaches more than any tutorial.
- **A thread pool.** A fixed set of `jthread`s pulling tasks (`std::function<void()>`) from a mutex+condition-variable-protected queue, with `std::future` results via `std::packaged_task`. Test it under TSan. *Exercises:* §15, §17.
- **A CLI tool.** Parse arguments, read/write files via `<filesystem>` and `<fstream>`, maybe shell out to a command, format output with `std::print`. Build it with CMake; add tests. *Exercises:* §14, §16, §18.
- **A JSON parser.** Parse JSON text into a `std::variant`-based value tree (or a recursive struct), returning `std::expected<Value, ParseError>`. Use `string_view` for zero-copy tokenizing, ranges for transforms, and write a thorough test suite. *Exercises:* §9–§13, §17, §18 — a capstone that uses nearly everything.

When these feel natural — when you reach for `unique_ptr` without thinking, return by value without worrying about copies, and run ASan reflexively — you're writing modern, production-grade C++. From here, deepen into the [C](C_GUIDE.md) guide (to understand the substrate), the [Rust](RUST_GUIDE.md) guide (to see safety enforced by the compiler instead of by discipline), and apply your C++ to systems work alongside the [Networking](NETWORKING_GUIDE.md), [Linux Server Administration](LINUX_SERVER_ADMIN_GUIDE.md), and [Git](GIT_GUIDE.md) guides.
