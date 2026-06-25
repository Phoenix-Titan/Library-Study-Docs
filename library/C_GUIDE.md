# C — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I have never written a line of C" to "I write correct, safe, production-grade C and I can reason about memory and undefined behavior with confidence" — entirely offline. C is small but *unforgiving*: there is no garbage collector, no bounds checking, no exceptions, and the compiler will happily turn your mistake into a security hole or a crash that only appears on a customer's machine six months later. So this guide is **explain-first**: every concept is taught in prose (what it is, *why* it exists, the exact semantics, the pitfalls, the best practice, and the security angle) *before* any heavily-commented code. The two hardest topics — **pointers/memory** and **undefined behavior** — get slow, rigorous, repeated treatment, because they are where C programmers actually fail. Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **C23 (ISO/IEC 9899:2024)** — the current standard — with **C17 (the bug-fix of C11) and C11** as the widely-deployed baseline you will meet in real codebases. Compile examples with **GCC 14/15** or **Clang 18/19** using `-std=c23` (older toolchains: `-std=c2x`); **MSVC** notes are called out where its support differs (MSVC's C support has historically lagged — confirm features there). C23 conveniences used throughout — `bool`/`true`/`false`/`nullptr`/`static_assert` as real keywords, `constexpr`, `typeof`, `[[nodiscard]]`/`[[maybe_unused]]` attributes, `#embed`, binary literals (`0b1010`), `_BitInt(N)`, and `<stdckdint.h>` checked arithmetic — are each flagged with **⚡ Version note ("C23; confirm compiler support")** so you know what to avoid on an older compiler. Everything here is accurate as of **June 2026**.
>
> The author is on **Windows 11**; cross-platform notes (POSIX vs Win32, path separators, `.exe`) are called out. The POSIX-specific material (`open`/`fork`/`pthreads`) targets Linux/macOS and **WSL2** on Windows. Related guides: [C++](CPP_GUIDE.md), [Rust](RUST_GUIDE.md), [Go](GO_GUIDE.md), [Networking](NETWORKING_GUIDE.md), [Linux Server Administration](LINUX_SERVER_ADMIN_GUIDE.md), [Bash Scripting](BASH_SCRIPTING_GUIDE.md), [Git](GIT_GUIDE.md).

---

## Table of Contents

1. [What C Is & Why It Still Matters](#1-what-c-is--why-it-still-matters) **[B]**
2. [Setup & the Compile Pipeline](#2-setup--the-compile-pipeline) **[B]**
3. [Types, Integers & the Overflow Minefield](#3-types-integers--the-overflow-minefield) **[B]**
4. [Operators, Control Flow & Functions](#4-operators-control-flow--functions) **[B]**
5. [Pointers in Depth — The Heart of C](#5-pointers-in-depth--the-heart-of-c) **[B/I]**
6. [Arrays, Strings & Pointer Decay](#6-arrays-strings--pointer-decay) **[B/I]**
7. [Memory Management & the Classic Bugs](#7-memory-management--the-classic-bugs) **[I/A]**
8. [Structs, Unions, Enums, Bitfields & Alignment](#8-structs-unions-enums-bitfields--alignment) **[I]**
9. [Undefined Behavior — A Serious Treatment](#9-undefined-behavior--a-serious-treatment) **[I/A]**
10. [The Preprocessor](#10-the-preprocessor) **[I]**
11. [File System, OS Info & Command Execution](#11-file-system-os-info--command-execution) **[I]**
12. [The C Standard Library Tour](#12-the-c-standard-library-tour) **[I]**
13. [Multi-File Projects & Build Systems](#13-multi-file-projects--build-systems) **[I/A]**
14. [Concurrency: Processes, Threads, Atomics](#14-concurrency-processes-threads-atomics) **[A]**
15. [Writing Safe, Production C](#15-writing-safe-production-c) **[A]**
16. [Debugging & Tooling](#16-debugging--tooling) **[A]**
17. [Testing & Maintainability](#17-testing--maintainability) **[A]**
18. [Performance](#18-performance) **[A]**
19. [Gotchas & Best Practices](#19-gotchas--best-practices) **[I/A]**
20. [Study Path & Build-to-Learn Projects](#20-study-path--build-to-learn-projects)

---

## 1. What C Is & Why It Still Matters

### 1.1 What C is, in one paragraph **[B]**

C is a small, statically-typed, compiled, **procedural** language created by Dennis Ritchie at Bell Labs in 1972 to write the Unix operating system. Its defining trait is that it is a *thin abstraction over the machine*: a C statement maps closely and predictably to a handful of CPU instructions, you control memory layout and lifetime by hand, and the runtime is essentially nothing (no garbage collector, no virtual machine, no implicit allocations). That closeness is C's super-power and its curse. The super-power: C runs *everywhere* and *fast*, with deterministic behaviour and a tiny footprint, which is why it became the lingua franca of systems programming. The curse: the language trusts you completely. There is no bounds checking, no use-after-free protection, no null-safety. When you break a rule, the standard often does not define what happens — your program might crash, might silently corrupt data, or might appear to work until a compiler upgrade quietly weaponises the bug. Learning C well is mostly learning to *not* break those rules, deliberately and systematically.

### 1.2 Where C runs (why it is still everywhere) **[B]**

- **Operating system kernels** — Linux, the BSDs, parts of Windows, and almost every RTOS are C.
- **Embedded & firmware** — microcontrollers, automotive, medical devices, network gear. Often C is the *only* mature toolchain available.
- **Language runtimes & interpreters** — CPython, Ruby MRI, the reference Lua, PHP, and the V8/SQLite/Redis/PostgreSQL/nginx/curl/git internals are C (or C-with-some-C++).
- **Performance-critical libraries** — codecs, crypto (OpenSSL), compression (zlib), databases. Other languages bind to these C libraries through a **C ABI** (Application Binary Interface), which is the universal interop standard precisely because C is so simple.
- **Anywhere you need a stable ABI** — the C calling convention is the lowest common denominator that Python (`ctypes`/CFFI), [Rust](RUST_GUIDE.md) (`extern "C"`), [Go](GO_GUIDE.md) (cgo), Java (JNI), and every other ecosystem speak.

When *not* to choose C for a new project: if you do not need that control and can tolerate a runtime, a memory-safe language ([Rust](RUST_GUIDE.md), [Go](GO_GUIDE.md)) will eliminate whole classes of the bugs this guide spends most of its time teaching you to avoid. Choose C when you must (existing C codebase, kernel/embedded target, hard ABI requirement) or when you genuinely want maximal control and minimal dependencies.

### 1.3 The standards: C89 → C23 **[B]**

C is governed by an ISO standard, revised every few years. You should know the lineage because real codebases pin to specific versions and gate features on them:

| Standard | Year | What it gave you (highlights) |
|---|---|---|
| **C89 / C90 (ANSI C)** | 1989/1990 | The first standard. Function prototypes, `void`, the standard library. Still the portability floor for ancient toolchains. |
| **C99** | 1999 | `//` comments, `<stdint.h>` fixed-width ints, `<stdbool.h>`, `long long`, designated initializers, variable-length arrays (VLAs), `inline`, declarations mid-block, `restrict`, `snprintf`. |
| **C11** | 2011 | `<threads.h>`, `<stdatomic.h>`, `_Static_assert`, `_Generic`, anonymous structs/unions, `_Alignas`/`_Alignof`, the optional Annex K `_s` functions, `_Noreturn`. |
| **C17 / C18** | 2017/2018 | No new features — a *defect-fix* of C11. This is the safe, ubiquitous baseline in 2026. |
| **C23** | 2024 | `bool`/`true`/`false`/`static_assert`/`thread_local`/`nullptr` as keywords, `constexpr`, `typeof`/`typeof_unqual`, `[[attributes]]` (`[[nodiscard]]`, `[[maybe_unused]]`, `[[deprecated]]`, `[[fallthrough]]`), `#embed`, binary literals `0b…`, digit separators `'`, `_BitInt(N)`, `<stdckdint.h>` checked arithmetic, `auto` type inference, `unreachable()`, UTF-8 `char8_t`, `gets` finally removed. |

**Practical baseline strategy:** write to **C17** for maximum portability, *or* to **C23** if you control the toolchain (GCC 13+/Clang 16+ have broad C23 support; GCC 14/15 and Clang 18/19 are solid in 2026). MSVC's C frontend is the laggard — verify each modern feature there.

### 1.4 The toolchain at a glance **[B]**

- **Compiler:** `gcc` (GNU), `clang` (LLVM), `cl` (MSVC), plus cross-compilers for embedded. GCC and Clang are nearly drop-in compatible and accept the same flags 95% of the time.
- **Preprocessor:** runs *first*, textually expanding `#include`/`#define`/`#if`. Part of the compiler driver.
- **Assembler & linker:** turn compiled object files into an executable or library. Usually invoked transparently by `gcc`/`clang`.
- **Build tools:** `make` (the classic), **CMake** (the de-facto cross-platform meta-build), Ninja, Meson.
- **Debuggers & sanitizers:** `gdb`/`lldb`, AddressSanitizer/UBSan/TSan, `valgrind`, static analyzers (`clang-tidy`, `cppcheck`). These are not optional for production C — covered in §15–16.

---

## 2. Setup & the Compile Pipeline

### 2.1 Installing a compiler **[B]**

- **Linux:** `sudo apt install build-essential gdb` (Debian/Ubuntu) gives you GCC, `make`, headers. `sudo apt install clang` for Clang. This is the smoothest C environment.
- **macOS:** `xcode-select --install` installs Apple Clang (`gcc` is an alias for `clang`). `brew install gcc` for real GCC.
- **Windows:** You have several options. **WSL2** (Windows Subsystem for Linux) gives you a real Linux toolchain and is the recommended path for following the POSIX parts of this guide. Native options: **MSYS2** (`pacman -S mingw-w64-ucrt-x86_64-gcc`) provides GCC; **MSVC** via Visual Studio Build Tools provides `cl`; **LLVM/Clang** for Windows. Throughout, POSIX-only examples (§11, §14) assume WSL2/Linux.

Verify:

```bash
gcc --version      # e.g. gcc (Ubuntu 14.2.0) 14.2.0
clang --version
gcc -std=c23 -dM -E - < /dev/null | grep __STDC_VERSION__   # prints 202311L on a C23 compiler
```

> **⚡ Version note (C23):** `__STDC_VERSION__` is `202311L` for C23, `201710L` for C17, `201112L` for C11, `199901L` for C99. Use it in `#if` to gate features — see §10.

### 2.2 The compile pipeline — four stages, demystified **[B]**

When you run `gcc main.c -o main`, the driver silently runs **four** stages. Understanding them is the difference between fixing a build error in five seconds versus an hour:

1. **Preprocessing** (`gcc -E main.c`) — a pure text transformation. `#include` files are pasted in, `#define` macros expanded, `#if`/`#ifdef` branches resolved. Output is a single (huge) "translation unit" of pure C with no directives left. *Most "undefined symbol" confusion starts here* — if a header is missing or a macro guard is wrong, the bug is at this stage.
2. **Compilation** (`gcc -S main.c`) — translates the preprocessed C into **assembly** for the target CPU. This is where type checking, warnings, and optimisation happen. A `.s` file is produced.
3. **Assembly** (`gcc -c main.c`) — turns assembly into machine-code **object files** (`.o`). Each `.c` file becomes one `.o`. The object file has *symbols* (function/variable names) but unresolved references to symbols defined in other files.
4. **Linking** (`gcc *.o -o main`) — the **linker** stitches object files and libraries together, resolving every symbol to an address and producing the final executable. **"undefined reference to `foo`"** is the canonical *linker* error: the compiler saw a declaration but the linker found no definition.

```bash
gcc -E main.c -o main.i    # 1. preprocess only  → main.i  (pure C, no directives)
gcc -S main.i -o main.s    # 2. compile to asm   → main.s
gcc -c main.s -o main.o    # 3. assemble         → main.o  (machine code, unlinked)
gcc main.o -o main         # 4. link             → main    (executable)
# In practice you do all four at once:
gcc main.c -o main
```

### 2.3 Warning flags you must always use **[B]**

C's defaults are dangerously quiet. **Turn the warnings on and treat them as errors.** A large fraction of the bugs this guide warns about are caught for free by these flags:

| Flag | What it does |
|---|---|
| `-std=c23` | Use the C23 standard (older toolchain: `-std=c2x`). Add `-pedantic` to reject non-standard extensions. |
| `-Wall` | Enable the "all" common warnings (it is *not* actually all — a historical misnomer). |
| `-Wextra` | More warnings `-Wall` misses (unused params, sign-compare, etc.). |
| `-Werror` | Turn warnings into hard errors. Forces you to fix them. |
| `-Wpedantic` | Warn on anything not strictly ISO C. |
| `-Wshadow` | Warn when a local variable shadows an outer one (a classic bug source). |
| `-Wconversion` | Warn on implicit conversions that may change a value (integer narrowing). |
| `-O2` / `-Og` | Optimisation. `-Og` is "optimise for debugging"; `-O2` for release. Optimisation also *enables more warnings* (uninitialized-use detection needs the optimiser). |
| `-g` | Emit debug info (for gdb/lldb). |

**Recommended day-to-day command:**

```bash
gcc -std=c23 -Wall -Wextra -Werror -Wshadow -Wconversion -g -O2 main.c -o main
```

### 2.4 Your first program — explained line by line **[B]**

The classic `hello.c`. Every token matters; the comments explain *why*, not just *what*:

```c
#include <stdio.h>   // Preprocessor: paste in the declarations for printf() etc.
                     // <angle brackets> = search the system include path.

// `int main(void)` is the program entry point. The OS calls it.
// `void` in the parameter list means "explicitly takes no arguments" — in C this
// is NOT the same as empty parentheses (which historically meant "unspecified").
// Returning `int` reports the exit status to the OS: 0 = success.
int main(void)
{
    // printf writes to standard output. "\n" is a newline.
    // The return value of printf (number of chars written, or negative on error)
    // is ignored here, which is fine for a hello-world.
    printf("Hello, world!\n");

    return 0;   // 0 → success. Non-zero → failure (the shell sees it as $?).
}
```

```bash
gcc -std=c23 -Wall -Wextra hello.c -o hello
./hello            # Linux/macOS/WSL
# .\hello.exe      # Windows native
echo $?            # prints 0 (the exit status)
```

> **Why `int main(void)` and not `void main()`?** The standard defines `main` as returning `int`. `void main()` is non-standard; some compilers tolerate it, but it is wrong. Since C99 you *may* omit `return 0;` from `main` specifically (it is implied), but write it explicitly for clarity. ⚡ Pre-C23, empty parentheses `int main()` meant "unspecified args"; **C23 changed `()` to mean exactly `(void)`** — but prefer `(void)` for clarity and pre-C23 portability.

### 2.5 `make` basics — stop typing the compile command **[B]**

`make` reads a `Makefile` describing *targets*, their *dependencies*, and the *commands* to build them. It rebuilds only what changed (by comparing file modification times). A minimal Makefile — note that the command lines **must be indented with a TAB, not spaces** (the #1 Makefile beginner error):

```makefile
# Variables (convention: uppercase)
CC      = gcc
CFLAGS  = -std=c23 -Wall -Wextra -Werror -g -O2
TARGET  = myapp
OBJS    = main.o utils.o

# Default target (the first one) — built when you just type `make`.
$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) $(OBJS) -o $(TARGET)

# Pattern rule: how to turn any X.c into X.o
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@      # $< = first prerequisite, $@ = the target

# A .PHONY target isn't a file; it's a command.
.PHONY: clean
clean:
	rm -f $(TARGET) $(OBJS)
```

```bash
make          # build the default target
make clean    # run the clean recipe
```

We return to `make` and graduate to **CMake** in §13.

---

## 3. Types, Integers & the Overflow Minefield

### 3.1 The basic types and why their sizes are *not* fixed **[B]**

C's built-in types are deliberately defined in terms of *minimum* ranges, not exact sizes, so the language can map onto any hardware. This is a famous source of portability bugs. The standard guarantees only:

| Type | Guaranteed minimum range | Typical size (64-bit Linux/macOS) | Typical size (64-bit Windows) |
|---|---|---|---|
| `char` | ≥ 8 bits; signedness is **implementation-defined** | 1 byte | 1 byte |
| `short` | ≥ 16 bits | 2 bytes | 2 bytes |
| `int` | ≥ 16 bits (commonly 32) | 4 bytes | 4 bytes |
| `long` | ≥ 32 bits | **8 bytes** | **4 bytes** ⚠️ |
| `long long` | ≥ 64 bits | 8 bytes | 8 bytes |
| `float` | IEEE-754 single (usually) | 4 bytes | 4 bytes |
| `double` | IEEE-754 double (usually) | 8 bytes | 8 bytes |

Notice the trap: **`long` is 64-bit on 64-bit Linux/macOS (LP64) but 32-bit on 64-bit Windows (LLP64)**. Code that assumes `sizeof(long) == sizeof(void*)` breaks when ported. The fix is fixed-width types (§3.3).

`char` is its own surprise: **plain `char` may be signed or unsigned** depending on the platform (signed on x86, unsigned on ARM by default). If you need a guaranteed range, say `signed char` or `unsigned char` explicitly.

> **⚡ Version note (C23):** `bool`, `true`, `false` are now real keywords — no `#include <stdbool.h>` needed (though including it is still harmless). Pre-C23 you needed that header, where `bool` was a macro for `_Bool`.

### 3.2 `sizeof` — measure, never assume **[B]**

`sizeof` is a *compile-time operator* (not a function) yielding the size in bytes of a type or object, with type `size_t` (an unsigned integer type big enough to hold any object size). Always use `sizeof` rather than hard-coding sizes:

```c
#include <stdio.h>

int main(void)
{
    // %zu is the correct printf format for size_t. Using %d here would be UB.
    printf("int:       %zu bytes\n", sizeof(int));
    printf("long:      %zu bytes\n", sizeof(long));
    printf("void*:     %zu bytes\n", sizeof(void *));   // pointer size = "word" size
    printf("double:    %zu bytes\n", sizeof(double));

    int arr[10];
    // sizeof an array gives the WHOLE array's size (only here, where `arr` is a
    // real array, not a decayed pointer — see §6). Idiom for element count:
    printf("elements:  %zu\n", sizeof(arr) / sizeof(arr[0]));   // 10

    return 0;
}
```

> **Gotcha:** `sizeof(arr)/sizeof(arr[0])` ONLY works where the array has not *decayed* to a pointer (e.g. not after being passed to a function). Inside a function that received `int arr[]`, `arr` is really a pointer and `sizeof` gives the pointer size. See §6.4.

### 3.3 `<stdint.h>` — the types you should actually use **[B]**

When you care about exact width (file formats, network protocols, hardware registers, anything that must be portable), use the fixed-width types from `<stdint.h>` (C99+). They make your intent explicit and your code portable:

| Type | Meaning |
|---|---|
| `int8_t`, `int16_t`, `int32_t`, `int64_t` | Signed, *exactly* that many bits (if the platform has them). |
| `uint8_t` … `uint64_t` | Unsigned, exact width. |
| `int_least8_t`, `uint_fast16_t`, … | "At least"/"fastest" variants when exact width isn't required. |
| `intptr_t` / `uintptr_t` | An integer guaranteed wide enough to hold a pointer (for pointer↔int casts). |
| `intmax_t` / `uintmax_t` | The widest available integer. |
| `size_t` | (from `<stddef.h>`) unsigned; the type of `sizeof`, array indices, lengths. |
| `ptrdiff_t` | (from `<stddef.h>`) signed; the type of a pointer subtraction. |

Print them with the macros from `<inttypes.h>`:

```c
#include <stdint.h>
#include <inttypes.h>
#include <stdio.h>

int main(void)
{
    uint32_t crc = 0xDEADBEEF;
    int64_t  big = -9000000000;

    // PRIu32 / PRId64 expand to the right length modifier for printf.
    printf("crc = %" PRIu32 ", big = %" PRId64 "\n", crc, big);

    // C23 binary literals + digit separators make bit-twiddling readable:
    uint8_t flags = 0b1010'0001;   // ⚡ Version note (C23): 0b… and ' separators
    printf("flags = %u\n", flags);
    return 0;
}
```

> **⚡ Version note (C23):** Binary literals (`0b1010`) and the digit-separator `'` (`1'000'000`) are C23. Pre-C23, use hex (`0xA1`) and no separators.

### 3.4 The integer-promotion and overflow minefield **[B]**

This is where beginners write code that *looks* obviously correct and is silently wrong. Three rules govern almost all integer arithmetic bugs in C:

**(1) Integer promotion.** Before almost any arithmetic, operands *smaller* than `int` (`char`, `short`, bitfields, `_Bool`) are **promoted to `int`** (or `unsigned int`). So `char a = 100, b = 100; a + b` is computed as `int` (200), not wrapped at 8 bits — but if you store it back into a `char`, *then* it wraps/implementation-defines.

**(2) Usual arithmetic conversions & the signed/unsigned trap.** When you mix signed and unsigned of the same rank, the **signed operand is converted to unsigned**. This is the cause of the infamous bug below: comparing a signed value against an unsigned `size_t` flips negatives into huge positives.

```c
#include <stdio.h>
#include <string.h>

int main(void)
{
    int n = -1;
    size_t len = strlen("hi");   // size_t is UNSIGNED

    // BUG: n is converted to a HUGE unsigned value, so -1 < 2 is FALSE here!
    if (n < len)                 // -Wsign-compare warns about exactly this
        printf("n is smaller\n");
    else
        printf("WRONG: -1 looked bigger than 2\n");   // this prints

    return 0;
}
```

The fix: keep types consistent, prefer signed types for arithmetic/loop counters where you might subtract, and *heed `-Wsign-compare`* (part of `-Wextra`).

**(3) Signed overflow is Undefined Behavior; unsigned overflow wraps.** This is critical and counter-intuitive:

- **Unsigned overflow is well-defined:** it wraps modulo 2ⁿ. `UINT_MAX + 1u == 0`. Reliable, but easy to underflow below zero into a huge value.
- **Signed overflow is *undefined behavior* (UB).** `INT_MAX + 1` is not "wraps to INT_MIN" — it is UB, and the optimiser is *allowed to assume it never happens*. The classic consequence: a check like `if (x + 1 < x)` to detect overflow can be **deleted by the compiler** because signed overflow "can't happen," so the check is "always false." Your overflow guard vanishes. This is a real, frequently-exploited class of bug (§9).

```c
#include <stdio.h>
#include <limits.h>

int main(void)
{
    // CORRECT overflow check: test BEFORE doing the operation, no UB triggered.
    int a = INT_MAX, b = 1;
    if (a > INT_MAX - b)            // safe: never overflows
        printf("would overflow, refuse\n");
    else
        printf("%d\n", a + b);

    // C23 way — checked arithmetic that can't be optimised away:
    // (see §3.5)
    return 0;
}
```

### 3.5 C23 checked arithmetic — `<stdckdint.h>` **[I]**

> **⚡ Version note (C23):** `<stdckdint.h>` provides `ckd_add`, `ckd_sub`, `ckd_mul` — they perform the operation and *return `true` if it overflowed*, storing the (wrapped) result. This is the clean, portable, UB-free way to do safe integer arithmetic. Pre-C23 you used GCC/Clang builtins (`__builtin_add_overflow`) or manual range checks like §3.4.

```c
#include <stdckdint.h>   // ⚡ C23
#include <stdio.h>
#include <stdint.h>

int main(void)
{
    size_t count = 1'000'000, size = 5'000'000;   // C23 separators
    size_t total;

    // ckd_mul returns true on overflow. CRITICAL for allocation sizes:
    // malloc(count * size) with a silent overflow is a classic heap exploit.
    if (ckd_mul(&total, count, size)) {
        fprintf(stderr, "size overflow — refusing to allocate\n");
        return 1;
    }
    printf("allocate %zu bytes\n", total);
    return 0;
}
```

### 3.6 Floating point — the other minefield **[B]**

`float` (32-bit) and `double` (64-bit) are IEEE-754 binary floating point. The essential facts every C programmer must internalise:

- **You cannot represent most decimals exactly.** `0.1 + 0.2 != 0.3`. Never compare floats with `==`; compare with a tolerance (`fabs(a - b) < 1e-9`).
- **Default to `double`.** `float` literals need an `f` suffix (`3.14f`); an unsuffixed `3.14` is a `double`. Using `float` saves space but loses precision; only use it when you have a reason (memory bandwidth, SIMD, GPU).
- **Special values:** `+0.0`/`-0.0`, `INFINITY`, and `NAN` (from `<math.h>`). `NAN != NAN` is `true` — that is how you test `isnan(x)`.
- **Integer-to-float and back can lose data;** a 64-bit integer cannot always round-trip through a `double` (which has 53 bits of mantissa).

```c
#include <stdio.h>
#include <math.h>      // link with -lm on Linux: gcc ... -lm
#include <float.h>     // DBL_EPSILON etc.

int main(void)
{
    double a = 0.1 + 0.2;
    printf("%.17g\n", a);                     // 0.30000000000000004
    printf("equal? %d\n", a == 0.3);          // 0 (false!)
    printf("close? %d\n", fabs(a - 0.3) < 1e-9); // 1 (true)

    double n = NAN;
    printf("nan==nan: %d, isnan: %d\n", n == n, isnan(n));  // 0, 1
    return 0;
}
```

> **Build note:** Functions from `<math.h>` (e.g. `sqrt`, `pow`, `fabs`) require linking the math library on Linux: add **`-lm`** at the *end* of the gcc command. On Windows/macOS it is usually folded into the C runtime.

---

## 4. Operators, Control Flow & Functions

### 4.1 Operators and the precedence traps **[B]**

C has a rich operator set. Most are obvious; the ones that *bite* are worth memorising:

- **Assignment vs equality:** `=` assigns, `==` compares. `if (x = 5)` assigns 5 to `x` and is always true — a classic typo. `-Wall` warns; some people write `if (5 == x)` ("Yoda conditions") so a typo'd `=` becomes a compile error.
- **Bitwise vs logical:** `&`/`|` are bitwise; `&&`/`||` are logical *and short-circuit* (the right operand is not evaluated if the result is already known). `a && b` won't evaluate `b` if `a` is false — exploit this for null-checks: `if (p != NULL && p->x)`.
- **Precedence surprises:** `&` and `|` bind *looser* than `==`. So `if (x & MASK == 0)` parses as `x & (MASK == 0)` — almost never what you mean. **Parenthesise bitwise expressions.**
- **`++`/`--` prefix vs postfix:** `++i` increments then yields; `i++` yields then increments. Do **not** combine multiple modifications of the same variable in one expression (`i = i++ + ++i;`) — that is UB (§9, sequence points).
- **Ternary:** `cond ? a : b`. Fine, but don't nest deeply.
- **Comma operator:** `(a, b)` evaluates both, yields `b`. Rare; mostly seen in `for` headers.

```c
// Precedence trap demonstration — always parenthesise bit tests:
unsigned flags = 0b0100;
if ((flags & 0b0100) != 0) { /* correct */ }
// if (flags & 0b0100 != 0)   // WRONG: parses as flags & (0b0100 != 0) = flags & 1
```

### 4.2 Control flow **[B]**

`if/else`, `while`, `do/while`, `for`, `switch`, `break`, `continue`, `goto`. The semantics are conventional; the C-specific pitfalls:

- **Dangling `else`** binds to the nearest `if`. **Always use braces**, even for one-line bodies — it prevents the "I added a second statement and the `if` no longer guards it" bug (the Apple `goto fail;` security bug was exactly this).
- **`switch` fall-through:** cases fall through to the next unless you `break`. This is occasionally intentional but usually a bug. ⚡ **C23** gives you `[[fallthrough]];` to mark deliberate fall-through and silence the warning.
- **`goto` is not forbidden** in C: the idiomatic use is **centralised cleanup** in functions that acquire several resources (the "goto cleanup" pattern, §7.6) — it is the cleanest way to avoid deeply nested error handling without RAII.

```c
#include <stdio.h>

int classify(int n)
{
    switch (n) {
        case 0:
            return 0;
        case 1:
        case 2:                       // intentional grouping (no fallthrough hazard)
            return 1;
        case 3:
            puts("three then keep going");
            [[fallthrough]];          // ⚡ C23: explicit, silences -Wimplicit-fallthrough
        case 4:
            return 4;
        default:
            return -1;
    }
}

int main(void)
{
    for (int i = 0; i < 5; i++)       // C99+: declare the loop var in the header
        printf("%d -> %d\n", i, classify(i));
    return 0;
}
```

### 4.3 Functions: declarations, definitions, prototypes **[B]**

A **declaration** (a *prototype*) tells the compiler a function's name, return type, and parameter types so it can type-check calls — it ends in `;`. A **definition** provides the body. Prototypes live in headers; definitions in `.c` files (§13). **Always provide a full prototype** before calling a function; calling an undeclared function is an error in C23 (it was merely a warning in older C, defaulting the return type to `int` — a dangerous legacy "implicit declaration" rule that C23 finally removed).

```c
#include <stdio.h>

// Prototype (declaration): the compiler now knows how to type-check calls.
int add(int a, int b);          // names of params are optional in a prototype

int main(void)
{
    printf("%d\n", add(2, 3));
    return 0;
}

// Definition: provides the body.
int add(int a, int b) { return a + b; }
```

> **⚡ Version note (C23):** Old-style "K&R" function definitions are removed, and `f()` now means `f(void)`. Implicit `int` and implicit function declarations are gone. This is a net safety win — your old warnings are now errors.

### 4.4 Scope, lifetime & storage classes (`static`, `extern`, `auto`, `register`) **[B]**

Two orthogonal concepts that beginners conflate: **scope** (where a name is *visible*) and **lifetime/storage duration** (when the object *exists*).

- **Block scope, automatic storage:** a normal local variable. Visible only inside its `{ }` block; created on entry, destroyed on exit (lives on the stack). `auto` is the (redundant) keyword for this. ⚡ **C23 repurposes `auto` for type inference** (`auto x = 5;`), so never use the old meaning.
- **`static` local:** keeps *automatic* scope (visible only in the function) but *static* lifetime — it persists across calls, initialised once. Useful for caches/counters, but a hidden landmine in multithreaded code (shared mutable state).
- **`static` at file scope:** gives a function or global **internal linkage** — it is private to that translation unit (`.c` file). This is your primary tool for encapsulation in C: mark every helper not part of your public API `static`.
- **`extern`:** declares that a global is *defined elsewhere* (another `.c` file). Used in headers to share a global. Prefer avoiding mutable globals entirely.
- **`register`:** a now-meaningless hint (compilers ignore it); you cannot take the address of a `register` variable. Avoid.

```c
#include <stdio.h>

static int file_private(void) { return 42; }   // internal linkage: not visible to linker outside this .c

int next_id(void)
{
    static int id = 0;     // initialised once; survives between calls
    return ++id;           // 1, 2, 3, ... but NOT thread-safe!
}

int main(void)
{
    printf("%d %d %d\n", next_id(), next_id(), next_id()); // order of args is unspecified! see §9
    (void)file_private();
    return 0;
}
```

> **Pitfall above:** the order of evaluation of function arguments is *unspecified* — `next_id(), next_id(), next_id()` may print `1 2 3` or `3 2 1`. Never rely on argument evaluation order (§9.5).

---

## 5. Pointers in Depth — The Heart of C

Pointers are the concept that makes C powerful and the concept on which most beginners stall. We go slowly, because everything later — strings, dynamic memory, data structures, function callbacks, the standard library — is built on them. **A pointer is a variable whose value is a memory address.** That is the whole idea. Everything else is consequences.

### 5.1 Addresses, `&`, and `*` **[B]**

Every object lives at some address in memory. The **address-of operator `&`** gives you the address of an object. A **pointer type** `T *` holds the address of an object of type `T`. The **dereference operator `*`** (also called *indirection*) goes *from* an address *to* the object it points at — you can read or write through it.

```c
#include <stdio.h>

int main(void)
{
    int x = 10;
    int *p = &x;     // p holds the ADDRESS of x. Type of p is `int *`.

    printf("x = %d\n", x);           // 10
    printf("&x = %p\n", (void *)&x); // the address (cast to void* for %p — required!)
    printf("p  = %p\n", (void *)p);  // same address
    printf("*p = %d\n", *p);         // 10  — dereference: the int p points at

    *p = 99;         // write THROUGH the pointer: modifies x itself
    printf("x = %d\n", x);           // 99

    return 0;
}
```

> **Format-string rule:** print a pointer with `%p` and **cast it to `void *`** — `printf("%p", p)` with a non-`void*` pointer is technically UB. Small detail, real rule.

### 5.2 Why pointers exist — what they buy you **[B]**

Pointers are not an academic curiosity; they solve concrete problems that C has no other way to solve:

1. **Output parameters / pass-by-reference.** C passes everything *by value* (a copy). To let a function modify the caller's variable, pass its address. This is how `scanf("%d", &n)` works.
2. **Avoiding expensive copies.** Passing a 1 KB struct by value copies 1 KB; passing a pointer copies 8 bytes.
3. **Dynamic memory.** `malloc` returns a pointer to heap memory whose size is decided at runtime (§7).
4. **Data structures.** Linked lists, trees, graphs are *defined* by nodes pointing at other nodes.
5. **Polymorphism via function pointers** (§5.7) and **type-erasure via `void *`** (§5.6).

```c
#include <stdio.h>

// Pass-by-reference: the function receives the ADDRESS of the caller's variable.
void increment(int *p) { (*p)++; }   // modifies the caller's int

// Returning multiple values via output parameters:
void divmod(int a, int b, int *q, int *r) { *q = a / b; *r = a % b; }

int main(void)
{
    int n = 5;
    increment(&n);            // pass the address
    printf("%d\n", n);        // 6

    int q, r;
    divmod(17, 5, &q, &r);    // q and r are filled in
    printf("%d r %d\n", q, r);// 3 r 2
    return 0;
}
```

### 5.3 Pointer arithmetic **[B/I]**

Adding an integer to a pointer advances it **by whole elements, not bytes**. `p + 1` points at the *next `T`*, i.e. the address advances by `sizeof(T)`. This is why pointers are typed. Subtracting two pointers into the same array yields the number of elements between them (type `ptrdiff_t`).

```c
#include <stdio.h>

int main(void)
{
    int a[5] = {10, 20, 30, 40, 50};
    int *p = a;            // a decays to &a[0] (see §6)

    printf("%d\n", *p);        // 10
    printf("%d\n", *(p + 2));  // 30  — advanced by 2 ints, i.e. 2*sizeof(int) bytes
    printf("%d\n", p[2]);      // 30  — a[i] is literally *(a + i). Equivalent!

    int *q = &a[4];
    printf("%td elements apart\n", q - p);  // 4  (%td is ptrdiff_t)

    // CRITICAL RULE: you may form a pointer ONE PAST the end (a+5) to compare
    // against, but you may NOT dereference it, and going further is UB (§9).
    for (int *it = a; it != a + 5; ++it)   // idiomatic "begin/end" iteration
        printf("%d ", *it);
    putchar('\n');
    return 0;
}
```

### 5.4 `const` correctness **[B/I]**

`const` documents and *enforces* that something won't be modified. With pointers, *where* you put `const` changes its meaning — read declarations **right-to-left**:

| Declaration | Meaning |
|---|---|
| `const int *p` | pointer to **const int** — you can't modify `*p`, but you can repoint `p`. |
| `int *const p` | **const pointer** to int — you can modify `*p`, but can't repoint `p`. |
| `const int *const p` | const pointer to const int — neither. |

Use `const` aggressively: a function that only *reads* a buffer should take `const T *`. It documents intent, lets the compiler catch accidental writes, and enables optimisation. The standard library does this everywhere (`strlen(const char *)`).

```c
#include <stddef.h>

// "I promise not to modify the data" — the const is part of the contract.
size_t my_strlen(const char *s)
{
    const char *p = s;
    while (*p) ++p;          // reading through a const pointer: fine
    // *p = 'x';             // COMPILE ERROR: cannot write through const char *
    return (size_t)(p - s);
}
```

> **Security angle:** `const`-correctness is a cheap, compile-time guarantee that a function does not mutate data it shouldn't. Casting `const` away (`(char *)p`) and then writing is **undefined behavior** if the object was originally `const`. Never cast away `const` to write.

### 5.5 The null pointer — and C23 `nullptr` **[B/I]**

A **null pointer** is a pointer that points to *nothing*. It is the conventional "no result"/"not found"/"error" sentinel and the value pointers should hold when not pointing at a valid object. **Dereferencing a null pointer is undefined behavior** (usually a crash — a segfault — but the standard makes no promise).

Historically you wrote `NULL` (a macro, often `((void*)0)` or `0`). The `0`-based definition caused subtle bugs in type-generic contexts. ⚡ **C23 introduces `nullptr`**, a true null-pointer constant with its own type (`nullptr_t`), eliminating the ambiguity. Prefer `nullptr` on C23; `NULL` remains fine and ubiquitous.

```c
#include <stdio.h>
#include <stddef.h>     // NULL

int *find_even(int *arr, size_t n)
{
    for (size_t i = 0; i < n; ++i)
        if (arr[i] % 2 == 0)
            return &arr[i];
    return nullptr;     // ⚡ C23. Pre-C23: return NULL;
}

int main(void)
{
    int a[] = {1, 3, 5, 8, 9};
    int *p = find_even(a, 5);

    // ALWAYS null-check before dereferencing a pointer that may be null:
    if (p != nullptr)
        printf("found %d\n", *p);   // 8
    else
        printf("none\n");
    return 0;
}
```

> **Defensive habit:** after `free(p)`, set `p = nullptr;` so a later accidental use is a clean null-deref crash rather than a silent use-after-free (§7).

### 5.6 `void *` — the generic pointer **[I]**

`void *` is a pointer to *unknown type* — it can hold the address of any object. You cannot dereference it directly (the compiler doesn't know the size/type) — you must cast it back to a concrete type first. It is C's mechanism for *generic* code: `malloc` returns `void *`, `memcpy` takes `void *`, and `qsort` uses `void *` to sort arrays of anything.

```c
#include <stdio.h>
#include <string.h>

// A generic swap that works on any type, given the element size.
void swap(void *a, void *b, size_t size)
{
    unsigned char tmp[64];           // small fixed scratch (real code: bound it)
    memcpy(tmp, a, size);
    memcpy(a,   b, size);
    memcpy(b,   tmp, size);
}

int main(void)
{
    int x = 1, y = 2;
    swap(&x, &y, sizeof x);
    printf("%d %d\n", x, y);         // 2 1

    double p = 1.5, q = 9.5;
    swap(&p, &q, sizeof p);
    printf("%g %g\n", p, q);         // 9.5 1.5
    return 0;
}
```

> **Note:** In C, the cast from `void *` to another pointer type is *implicit* — `int *p = malloc(...)` needs no cast. **Do not cast the result of `malloc`** in C (it can hide a missing `<stdlib.h>` include). (This differs from [C++](CPP_GUIDE.md), where the cast is required.)

### 5.7 Function pointers **[I]**

A **function pointer** holds the address of a function, letting you store, pass, and call functions as data. This is how C does callbacks, plugin tables, state machines, and dependency injection. The syntax is notoriously ugly — `typedef` it.

```c
#include <stdio.h>

int add(int a, int b) { return a + b; }
int sub(int a, int b) { return a - b; }

// A function pointer type named BinOp: "pointer to function (int,int)->int"
typedef int (*BinOp)(int, int);

// Higher-order function: takes a callback.
int apply(BinOp op, int a, int b) { return op(a, b); }

int main(void)
{
    BinOp ops[] = { add, sub };       // a dispatch table
    printf("%d\n", apply(ops[0], 7, 3));   // 10
    printf("%d\n", apply(ops[1], 7, 3));   // 4

    // You can call directly too; &add and add both yield the function pointer.
    BinOp f = add;
    printf("%d\n", f(1, 2));          // 3 (f(...) and (*f)(...) are equivalent)
    return 0;
}
```

The standard library's `qsort` takes a function-pointer *comparator* — the canonical real-world example (§12.6).

---

## 6. Arrays, Strings & Pointer Decay

### 6.1 Arrays **[B]**

An array is a contiguous, fixed-size sequence of elements of one type. Its size is fixed at definition (compile time for a normal array). Indexing is `a[i]`, defined as `*(a + i)` — which means, amusingly, `i[a]` is also legal (don't). **C does no bounds checking.** Reading or writing `a[i]` with `i` out of range is undefined behavior — the source of the buffer-overflow vulnerabilities that have defined computer security for decades.

```c
#include <stdio.h>

int main(void)
{
    int a[5] = {1, 2, 3};   // remaining elements zero-initialised: {1,2,3,0,0}
    int b[]  = {1, 2, 3};   // size inferred as 3

    size_t n = sizeof a / sizeof a[0];   // 5
    for (size_t i = 0; i < n; ++i)
        printf("%d ", a[i]);
    putchar('\n');

    // a[5] = 1;   // ⚠️ OUT OF BOUNDS — undefined behavior. No diagnostic at runtime.
    (void)b;
    return 0;
}
```

### 6.2 Strings — NUL-terminated, and why that hurts **[B/I]**

C has **no string type**. A "string" is merely a `char` array whose end is marked by a NUL byte (`'\0'`, value 0). A string *literal* `"hi"` is a read-only array of `{'h','i','\0'}` of length 3 (the NUL is implied). Every standard string function (`strlen`, `strcpy`, `printf("%s")`) walks the bytes until it hits the NUL.

This design is the single biggest source of memory-safety bugs in computing history, because:

- **The length is not stored** — it is computed by scanning (`strlen` is O(n)).
- **Functions trust the NUL to be there.** If it isn't (because you overwrote it, or filled a buffer with `strncpy` and it truncated without terminating), the function reads off the end → out-of-bounds read, info leak, or crash.
- **The destination size is invisible** to functions like `strcpy`/`strcat` — they write until the *source* ends, overflowing a too-small destination. This is the classic stack-smashing exploit.

```c
#include <stdio.h>
#include <string.h>

int main(void)
{
    char greeting[] = "Hello";     // 6 bytes: 'H','e','l','l','o','\0'
    printf("len=%zu size=%zu\n", strlen(greeting), sizeof greeting); // 5, 6

    // String LITERALS are read-only. This pointer must be const:
    const char *msg = "world";     // do NOT write through msg — UB
    // msg[0] = 'W';               // UB: modifying a string literal

    char buf[16];
    // SAFE copy: snprintf bounds the write and always NUL-terminates.
    snprintf(buf, sizeof buf, "%s, %s!", greeting, msg);
    printf("%s\n", buf);           // Hello, world!
    return 0;
}
```

> **Rule:** prefer `snprintf` over `strcpy`/`strcat`/`sprintf`. It takes the destination size, never overflows, and always NUL-terminates (when size > 0). The dangerous functions (`gets`, `strcpy`, `strcat`, `sprintf`) are covered in the security section (§15). ⚡ `gets` was *removed* entirely in C11/C23 — it could never be used safely.

### 6.3 Array/pointer decay **[B/I]**

In most expressions, an array **"decays"** to a pointer to its first element. `a` becomes `&a[0]` of type `T *`, *losing the size information*. The exceptions where decay does **not** happen: as the operand of `sizeof`, of `&` (`&a` has type "pointer to array"), and when initialising a `char[]` from a string literal. Decay is why:

- `sizeof(arr)` gives the full array size *only in the scope where `arr` is a real array*.
- A function parameter written `int arr[]` or `int arr[10]` is **secretly a pointer** `int *arr`. Inside the function, `sizeof arr` is the pointer size (8), *not* the array size. **You must pass the length separately.**

```c
#include <stdio.h>

// These three signatures are IDENTICAL — all receive a pointer, not an array:
//   void f(int arr[10])   ==   void f(int arr[])   ==   void f(int *arr)
void print_all(const int *arr, size_t n)   // length passed explicitly — the C idiom
{
    for (size_t i = 0; i < n; ++i)
        printf("%d ", arr[i]);
    putchar('\n');
    // sizeof arr here would be 8 (pointer), NOT the array size — the decay trap.
}

int main(void)
{
    int data[] = {1, 2, 3, 4};
    print_all(data, sizeof data / sizeof data[0]);  // compute length HERE, where data is a real array
    return 0;
}
```

### 6.4 Multidimensional arrays & VLAs **[I]**

A 2-D array `int m[3][4]` is a contiguous block of 12 ints, *row-major* (the last index varies fastest). It is **not** an array of pointers — there is no indirection. When you pass it to a function, all dimensions except the first must be known so the compiler can compute `m[i][j] = *(base + i*cols + j)`.

```c
#include <stdio.h>

// All dimensions but the first must be specified:
void print_matrix(size_t rows, size_t cols, int m[rows][cols])  // VLA parameter (C99+)
{
    for (size_t i = 0; i < rows; ++i) {
        for (size_t j = 0; j < cols; ++j)
            printf("%3d ", m[i][j]);
        putchar('\n');
    }
}

int main(void)
{
    int grid[2][3] = {{1, 2, 3}, {4, 5, 6}};
    print_matrix(2, 3, grid);
    return 0;
}
```

> **⚡ VLA note:** Variable-Length Arrays (`int buf[n]` with runtime `n`) are C99, made *optional* in C11, and **discouraged in production**: a large or attacker-controlled `n` overflows the stack (a crash or exploit). Many projects ban them (`-Wvla`). Prefer `malloc` for runtime sizes. VLA *parameters* (as above) for passing 2-D arrays are more acceptable, but still confirm support (MSVC lacks VLAs).

---

## 7. Memory Management & the Classic Bugs

This is the section that separates people who *think* they know C from people who actually do. C gives you raw control over the heap and *zero* safety nets. Master the model and the discipline here and most of the rest is easy.

### 7.1 The stack vs the heap **[I]**

A C program's memory has several regions. The two you manage are:

- **The stack:** where local (automatic) variables live. Fast (just moves a pointer), automatically reclaimed when the function returns. **Limited in size** (typically 1–8 MB) — large arrays or deep recursion *overflow* it (crash). You do not free stack memory; you cannot return a pointer to a stack local (it is destroyed on return — a **dangling pointer**, §7.4).
- **The heap:** a large pool you allocate from explicitly with `malloc`/`calloc`/`realloc` and **must** return with `free`. Slower, but the memory persists until *you* free it — so its lifetime is under your control, which is exactly why you use it (dynamic size, lifetime outliving the creating function).

Other regions: **static/global** storage (lives the whole program), **read-only** data (string literals, `const`), and the **code** segment.

### 7.2 `malloc`, `calloc`, `realloc`, `free` **[I]**

| Function | What it does |
|---|---|
| `void *malloc(size_t n)` | Allocate `n` bytes, **uninitialised** (garbage contents). Returns a pointer, or `nullptr` on failure. |
| `void *calloc(size_t count, size_t size)` | Allocate `count*size` bytes, **zeroed**, with the multiply done *safely* (overflow-checked internally). Prefer it for arrays. |
| `void *realloc(void *p, size_t n)` | Resize a prior allocation to `n` bytes (may move it, copying contents). Returns new pointer or `nullptr`. |
| `void free(void *p)` | Return memory to the allocator. `free(nullptr)` is a safe no-op. |

**Iron rules:**
1. **Every `malloc`/`calloc`/`realloc` must be matched by exactly one `free`.** Not zero (leak), not two (double-free).
2. **Always check the return for `nullptr`** before using it — allocation can fail.
3. **`realloc` must be assigned to a temporary first.** `p = realloc(p, n)` leaks the original block if `realloc` fails and returns `nullptr` (you've overwritten the only pointer to it).
4. **After `free(p)`, set `p = nullptr`** to neutralise use-after-free/double-free.

```c
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    // calloc: zeroed, overflow-safe multiply. Best for arrays.
    size_t n = 10;
    int *a = calloc(n, sizeof *a);          // note: sizeof *a, not sizeof(int) — robust to type changes
    if (a == nullptr) { perror("calloc"); return 1; }   // ALWAYS check

    for (size_t i = 0; i < n; ++i) a[i] = (int)i * i;

    // Grow it safely with a temporary:
    size_t new_n = 20;
    int *tmp = realloc(a, new_n * sizeof *a);   // (real code: check new_n*sizeof overflow, §3.5)
    if (tmp == nullptr) { free(a); return 1; }  // a is still valid; free it and bail
    a = tmp;                                     // only now overwrite a

    free(a);          // exactly once
    a = nullptr;      // defensive: any later use is a clean null-deref, not UAF
    return 0;
}
```

### 7.3 Memory leaks **[I]**

A **leak** is heap memory you allocated but never freed and have lost the last pointer to — it is reserved forever (until the process exits). One-shot programs "get away with it" because the OS reclaims everything on exit, but long-running services (servers, daemons) leak until they exhaust memory and die. Leaks are caught by **valgrind** and **AddressSanitizer with leak detection** (§15–16). The discipline: decide, for every allocation, *who owns it and who frees it*, and document that ownership.

```c
void leaky(void)
{
    char *buf = malloc(100);
    // ... use buf ...
    if (some_error())
        return;          // LEAK: returned without free(buf)
    free(buf);
}
// Fix: free on every exit path, or use the "goto cleanup" pattern (§7.6).
```

### 7.4 Dangling pointers & use-after-free **[I/A]**

A **dangling pointer** points to memory that is no longer valid — either freed heap, or a stack variable whose function returned. **Using it (read or write) is undefined behavior** and a top-tier security vulnerability (use-after-free, or UAF, is the basis of countless exploits because the freed slot may be reallocated to attacker-controlled data).

```c
#include <stdlib.h>

// BUG 1: returning a pointer to a stack local — it's destroyed on return.
int *bad_stack(void)
{
    int x = 42;
    return &x;          // DANGLING: x no longer exists after return. UB to deref.
}

// BUG 2: use-after-free.
void uaf(void)
{
    int *p = malloc(sizeof *p);
    *p = 1;
    free(p);
    // *p = 2;          // USE-AFTER-FREE: p points at freed memory. UB.
    // free(p);         // DOUBLE-FREE: also UB.
}

// FIX for returning data: allocate on the heap (caller owns/frees it), and
// null out after free to defuse UAF.
int *good_heap(void)
{
    int *p = malloc(sizeof *p);
    if (p) *p = 42;
    return p;           // caller must free()
}
```

### 7.5 Buffer overflow **[I/A]**

Writing past the end of a buffer (stack or heap) is undefined behavior and *the* classic exploit. On the stack it can overwrite the saved return address (stack smashing); on the heap it can corrupt allocator metadata or adjacent objects. Avoidance: never use unbounded copies, always track sizes, use `snprintf`/`memcpy` with explicit lengths, and compile with the sanitizers and `-fstack-protector-strong` (§15).

```c
#include <string.h>
#include <stdio.h>

void unsafe(const char *input)
{
    char buf[8];
    strcpy(buf, input);   // ⚠️ OVERFLOW if input is ≥ 8 chars. Classic exploit.
}

void safe(const char *input)
{
    char buf[8];
    snprintf(buf, sizeof buf, "%s", input);   // truncates, always NUL-terminated, never overflows
    printf("%s\n", buf);
}
```

### 7.6 The `goto cleanup` ownership pattern **[I/A]**

When a function acquires several resources (allocations, files, locks), it must release *all already-acquired* ones on *any* failure path. Without RAII (which [C++](CPP_GUIDE.md)/[Rust](RUST_GUIDE.md) have and C lacks), the cleanest correct idiom is a single cleanup section reached by `goto`. Each acquired resource adds one cleanup step in reverse order; failures jump to the appropriate label.

```c
#include <stdio.h>
#include <stdlib.h>

int process(const char *path)
{
    int   rc  = -1;          // pessimistic default
    char *buf = nullptr;
    FILE *f   = nullptr;

    buf = malloc(1024);
    if (buf == nullptr) goto cleanup;          // nothing acquired yet beyond buf

    f = fopen(path, "rb");
    if (f == nullptr) goto cleanup;            // buf is freed below

    if (fread(buf, 1, 1024, f) == 0) goto cleanup;

    // ... real work ...
    rc = 0;                  // success

cleanup:                     // single exit; releases in reverse acquisition order
    if (f)   fclose(f);
    free(buf);               // free(nullptr) is safe, so no null-check needed
    return rc;
}
```

This pattern is pervasive in the Linux kernel and high-quality C libraries. It is *not* "spaghetti goto" — it is structured, forward-only, single-target cleanup, and it is the right answer in C.

---

## 8. Structs, Unions, Enums, Bitfields & Alignment

### 8.1 Structs **[I]**

A **struct** aggregates named fields of possibly different types into one object, laid out in memory in declaration order (with padding, §8.5). Structs are how you model records and build data structures. Access fields with `.` on a struct value and `->` on a pointer to a struct (`p->x` is sugar for `(*p).x`).

```c
#include <stdio.h>
#include <string.h>

struct Point { int x, y; };           // a struct type named "struct Point"

// typedef to drop the "struct" keyword at use sites (common style):
typedef struct { double re, im; } Complex;   // anonymous struct + typedef

int main(void)
{
    struct Point a = {1, 2};          // positional initialiser
    struct Point b = {.y = 9, .x = 4};// DESIGNATED initialiser (C99+): order-independent, self-documenting

    a.x = 10;                         // value access with .
    struct Point *p = &a;
    p->y = 20;                        // pointer access with ->  (== (*p).y)

    Complex c = {.re = 1.5, .im = -2.0};
    printf("%d,%d  %d,%d  %g+%gi\n", a.x, a.y, b.x, b.y, c.re, c.im);

    struct Point copy = a;            // structs are COPIED by assignment (value semantics)
    (void)copy; (void)strlen;
    return 0;
}
```

> **Designated initializers (C99+)** are strongly recommended: `{.x = 1, .y = 2}` is robust to field reordering and leaves unmentioned fields zero-initialised. Use them.

### 8.2 Passing & returning structs **[I]**

Structs are *value types*: assignment, passing to a function, and returning all **copy** the whole struct. For small structs that is fine and clean; for large ones, pass a pointer (ideally `const T *` if read-only) to avoid the copy. This is a deliberate API decision.

### 8.3 Unions **[I]**

A **union** stores *one* of several members in the *same* memory (its size is that of its largest member). You write one member and read *that same* member. **Reading a different member than the one last written is implementation-defined / often UB** (type-punning — see the strict-aliasing discussion in §9.4; `memcpy` is the portable alternative). Unions are used for tagged variants (always pair a union with an enum tag) and low-level representation tricks.

```c
#include <stdio.h>
#include <stdint.h>

// A "tagged union" (discriminated union): the enum says which member is valid.
typedef enum { TAG_INT, TAG_FLT, TAG_STR } Tag;
typedef struct {
    Tag tag;
    union { int i; double d; const char *s; } as;
} Value;

void print_value(const Value *v)
{
    switch (v->tag) {                 // ALWAYS read the member the tag says is active
        case TAG_INT: printf("int %d\n", v->as.i);    break;
        case TAG_FLT: printf("flt %g\n", v->as.d);    break;
        case TAG_STR: printf("str %s\n", v->as.s);    break;
    }
}

int main(void)
{
    Value v = {.tag = TAG_FLT, .as.d = 3.14};
    print_value(&v);
    return 0;
}
```

### 8.4 Enums & bitfields **[I]**

An **enum** is a set of named integer constants — use it for states, kinds, and option codes instead of magic numbers. ⚡ **C23** lets you specify the *underlying type* (`enum Color : uint8_t { ... }`) for ABI control. A **bitfield** packs integers into a chosen number of bits inside a struct — handy for hardware registers and compact flags, but with caveats: bit *ordering* and packing are implementation-defined, so bitfields are not portable for wire/file formats (use explicit shifts and masks there instead).

```c
#include <stdint.h>
#include <stdio.h>

enum Color : uint8_t { RED, GREEN = 5, BLUE };   // ⚡ C23 fixed underlying type; BLUE == 6

// Bitfield packing — fits flags into a single byte's worth of bits:
struct Flags {
    unsigned visible : 1;     // 1 bit
    unsigned bold    : 1;
    unsigned level   : 3;     // 0..7
};

int main(void)
{
    struct Flags f = {.visible = 1, .level = 5};
    printf("%u %u %u\n", f.visible, f.bold, f.level);   // 1 0 5
    printf("BLUE=%d\n", BLUE);                          // 6
    return 0;
}
```

### 8.5 Alignment & padding — what your struct really looks like **[I]**

The compiler inserts **padding** bytes so each field sits at an address that is a multiple of its *alignment* (typically its size: a 4-byte `int` wants a 4-byte-aligned address). This means `sizeof(struct)` can be *larger* than the sum of the fields, and **reordering fields can shrink a struct**. Misaligned access is slow or, on some architectures, a hardware fault. Don't fight the compiler; do order fields from largest to smallest to minimise padding, and never assume a struct's layout for serialization (write fields explicitly).

```c
#include <stdio.h>
#include <stddef.h>

struct Bad  { char a; int b; char c; };   // padding scattered → often 12 bytes
struct Good { int b; char a; char c; };   // packed better      → often 8 bytes

int main(void)
{
    printf("Bad=%zu Good=%zu\n", sizeof(struct Bad), sizeof(struct Good));
    printf("offset of b in Bad = %zu\n", offsetof(struct Bad, b));  // shows the padding gap
    // C11/C23: _Alignof / alignof tells you a type's alignment requirement.
    printf("alignof(int) = %zu\n", alignof(int));   // ⚡ alignof is a keyword in C23
    return 0;
}
```

> **⚡ Version note (C23):** `alignof`/`alignas` are keywords (C11 had `_Alignof`/`_Alignas`, with macros in `<stdalign.h>`). `static_assert` is also a keyword in C23 — use it to assert struct sizes at compile time: `static_assert(sizeof(struct Good) == 8, "unexpected padding");`.

---

## 9. Undefined Behavior — A Serious Treatment

If you remember one section of this guide, make it this one. **Undefined Behavior (UB) is the defining hazard of C**, and misunderstanding it is why C programmers ship security holes. We treat it with the seriousness it deserves.

### 9.1 What UB actually is **[I/A]**

The C standard classifies behaviours into three buckets:

- **Undefined behavior (UB):** the standard imposes **no requirements whatsoever**. The compiler may do *anything* — crash, produce wrong results, work today and fail tomorrow, or *delete code* around the UB. Examples: signed overflow, out-of-bounds access, use-after-free, null deref, data races.
- **Unspecified behavior:** the standard offers a *set* of allowed behaviours but doesn't say which (e.g. argument evaluation order). The program is valid but you can't rely on a particular choice.
- **Implementation-defined behavior:** the implementation must *pick and document* a behaviour (e.g. whether `char` is signed, the size of `int`). Portable but you should check the docs.

**The crucial mental model:** UB is not "behaves in some platform-specific way." It is a *promise you make to the compiler that this situation never arises*. The optimiser **believes that promise** and rewrites your code accordingly. So UB doesn't just risk a crash at the offending line — it lets the compiler make *globally* wrong assumptions, deleting checks and reordering code far from the bug. This is why "it worked on my machine / in debug mode" is no defence.

### 9.2 Why UB is catastrophic — a concrete example **[I/A]**

```c
#include <stdio.h>
#include <limits.h>

// A "safe" overflow check that signed-overflow UB lets the compiler DELETE:
int will_overflow(int a, int b)
{
    int sum = a + b;          // if this overflows, it's UB...
    return sum < a;           // ...so the compiler may assume a+b never overflows,
                              // making `sum < a` provably false → it returns 0 ALWAYS.
}

int main(void)
{
    // At -O2 this may print 0 even though INT_MAX + 1 "overflows":
    printf("%d\n", will_overflow(INT_MAX, 1));
    return 0;
}
```

The correct version never *causes* the overflow: `if (b > 0 && a > INT_MAX - b) ...` — or use `<stdckdint.h>` (§3.5) / `__builtin_add_overflow`.

### 9.3 The common UB you must know **[I/A]**

| UB | What triggers it | Avoidance |
|---|---|---|
| **Signed integer overflow** | `INT_MAX + 1`, `INT_MIN / -1`, `1 << 31` on `int` | Check before; use unsigned for wrap; `ckd_*`. |
| **Out-of-bounds access** | `a[n]` outside `[0,n)`; pointer past `end+1`; deref `end` | Track lengths; ASan; bounds-check inputs. |
| **Null / wild pointer deref** | `*p` where `p` is null/uninitialised/dangling | Initialise pointers to `nullptr`; null-check; null after free. |
| **Use-after-free / double-free** | reading/freeing freed memory | Null after free; clear ownership; ASan. |
| **Uninitialised read** | reading a local before assigning it | Always initialise; `-Wmaybe-uninitialized`; MSan. |
| **Strict-aliasing violation** | reading an object through an incompatible pointer type | Use `memcpy` to type-pun; or `-fno-strict-aliasing`. |
| **Sequence-point / unsequenced** | `i = i++;`, `a[i] = i++;` | One modification of one object per full expression. |
| **Shift errors** | shift ≥ width, or negative shift count | Mask the shift amount; only shift unsigned. |
| **`memcpy` overlap / null** | overlapping ranges (use `memmove`); null args | Use `memmove` when ranges may overlap. |
| **Division by zero** | `x / 0`, `x % 0` | Guard the divisor. |

### 9.4 Strict aliasing — the subtle one **[A]**

The compiler assumes that pointers of *different* types do **not** point at the same memory (with exceptions: `char *` may alias anything, and compatible types). This lets it cache values in registers across writes through "unrelated" pointers. If you *violate* this assumption — e.g. reinterpret a `float`'s bits by reading it through an `int *` — you have UB, and the optimiser may produce nonsense. The portable, blessed way to reinterpret bytes is **`memcpy`** (the compiler recognises and optimises it away):

```c
#include <stdint.h>
#include <string.h>
#include <stdio.h>

// WRONG: strict-aliasing violation — reading a float through a uint32_t*.
// uint32_t bits = *(uint32_t *)&f;          // UB

// RIGHT: memcpy the bytes. No aliasing, no UB; compilers optimise this to a move.
uint32_t float_bits(float f)
{
    uint32_t bits;
    memcpy(&bits, &f, sizeof bits);
    return bits;
}

int main(void)
{
    printf("0x%08x\n", float_bits(1.0f));   // 0x3f800000
    return 0;
}
```

### 9.5 Sequence points & evaluation order **[A]**

Within one expression, modifying the same object twice, or modifying it and also reading it (other than to compute the new value), without an intervening **sequence point** is UB. And the *order* in which subexpressions and function arguments are evaluated is largely **unspecified**. So:

```c
int i = 0;
// i = i++;            // UB: i modified twice (by = and by ++) — unsequenced
// a[i] = i++;         // UB: read & modify i, unsequenced
// f(i++, i++);        // UB: two unsequenced modifications of i
// printf("%d %d", next_id(), next_id());  // unspecified ORDER (not UB, but unreliable)
```

The rule of thumb: **at most one modification of any one object per full expression, and never depend on the order of side effects.** Split it into separate statements; clarity costs nothing.

### 9.6 How to defend against UB systematically **[A]**

You cannot eyeball UB out of a real codebase. Build a *system*:

1. **Compile with `-Wall -Wextra -Werror`** — catches a surprising amount statically.
2. **Run under the sanitizers** in CI and dev: `-fsanitize=address,undefined` catches OOB, UAF, signed overflow, misaligned access, and more *at runtime* (§15). UBSan literally flags UB as it happens.
3. **Run valgrind** for memory errors the compiler can't see.
4. **Fuzz** untrusted-input parsers (§15.6) — fuzzers + sanitizers find UB you never imagined.
5. **Static analysis** (`clang-tidy`, `cppcheck`, `clang --analyze`) for deeper interprocedural bugs.

UB avoidance is not heroism; it is tooling plus discipline.

---

## 10. The Preprocessor

### 10.1 What it is and why to respect it **[I]**

The preprocessor runs *before* the compiler and does pure **text substitution**: it has no understanding of C types or scope. That ignorance is exactly why macros are dangerous — they expand textually, so they can capture, double-evaluate, and surprise you. Modern C prefers `const`, `enum`, `inline`, and `static_assert` over macros wherever possible, reserving macros for what only text substitution can do (header guards, conditional compilation, `X-macros`, stringising).

### 10.2 `#include`, `#define`, and the macro pitfalls **[I]**

```c
#include <stdio.h>     // <...>  : system include path
// #include "mine.h"   // "..."  : start from the current file's directory, then system

#define PI 3.14159          // object-like macro (a named constant — but prefer `constexpr`/`const`)
#define SQUARE(x) ((x) * (x))   // function-like macro — note the parentheses!

int main(void)
{
    // Pitfall 1: missing parentheses. Define as x*x and SQUARE(1+2) → 1+2*1+2 = 5, not 9.
    printf("%d\n", SQUARE(1 + 2));   // ((1+2)*(1+2)) = 9 — correct BECAUSE of the parens

    // Pitfall 2: DOUBLE EVALUATION. The argument is substituted twice:
    int i = 3;
    // printf("%d\n", SQUARE(i++));  // ((i++)*(i++)) — modifies i twice → UB!

    (void)PI;
    return 0;
}
```

Macro hygiene rules: **wrap the whole body and every parameter in parentheses**; never pass an expression with side effects to a function-like macro; prefer `static inline` functions (type-safe, no double-evaluation) and `constexpr`/`enum` constants. ⚡ **C23 `constexpr`** gives true compile-time typed constants — `constexpr double Pi = 3.14159;` — a real improvement over `#define PI`.

### 10.3 Include guards & `#pragma once` **[I]**

A header included twice in one translation unit would redefine its types → errors. **Include guards** prevent that. Every header needs one:

```c
// in widget.h
#ifndef WIDGET_H          // if the guard macro is not yet defined...
#define WIDGET_H          // ...define it, and include the body once.

typedef struct Widget Widget;
Widget *widget_new(void);
void    widget_free(Widget *);

#endif // WIDGET_H
```

`#pragma once` is a non-standard but near-universally supported one-liner alternative (GCC, Clang, MSVC all support it). Include guards are 100% portable; many codebases use both.

### 10.4 Conditional compilation **[I]**

`#if`/`#ifdef`/`#ifndef`/`#elif`/`#else`/`#endif` select code at compile time — used for platform differences, debug builds, and feature flags. Gate version-specific features on `__STDC_VERSION__`:

```c
#include <stdio.h>

#if defined(_WIN32)
  #define PATH_SEP '\\'
#else
  #define PATH_SEP '/'
#endif

#if __STDC_VERSION__ >= 202311L
  #define HAVE_C23 1            // ⚡ gate C23 features on the version macro
#else
  #define HAVE_C23 0
#endif

int main(void)
{
    printf("sep=%c c23=%d\n", PATH_SEP, HAVE_C23);
    return 0;
}
```

### 10.5 `#embed` — C23's killer convenience **[I]**

> **⚡ Version note (C23):** `#embed` pulls a *binary file's bytes* into your source as an initializer list at compile time — no more `xxd`-generated arrays or build-time codegen for embedding shaders, icons, certificates, or lookup tables. Confirm support (GCC 15 / Clang 19 have it; older toolchains do not).

```c
// ⚡ C23: embed a file's bytes directly. Requires a recent GCC/Clang.
#include <stdio.h>

static const unsigned char logo[] = {
#embed "logo.png"          // expands to a comma-separated list of byte values
};

int main(void)
{
    printf("logo is %zu bytes\n", sizeof logo);
    return 0;
}
```

Other useful directives: `#error "msg"` (abort compilation with a message), `#warning` (C23-standard now), `#line`, and the predefined macros `__FILE__`, `__LINE__`, `__func__`, `__DATE__`.

---

## 11. File System, OS Info & Command Execution

C's portable file I/O lives in `<stdio.h>`; lower-level, more powerful (but POSIX-only) facilities live in `<fcntl.h>`/`<unistd.h>`. Process creation and environment access round out "talking to the OS." The POSIX parts below assume Linux/macOS/**WSL2**; Windows-native equivalents use the Win32 API (`CreateFile`, `CreateProcess`).

### 11.1 Buffered text & binary I/O with `<stdio.h>` **[I]**

`FILE *` is a *buffered* stream. `fopen` opens it, the `fread`/`fwrite`/`fprintf`/`fgets` family transfers data, and `fclose` flushes and closes. **Always check `fopen` for `nullptr`** and check return values — I/O fails (disk full, permissions, missing file).

```c
#include <stdio.h>

int main(void)
{
    // Write text. Modes: "r" read, "w" truncate+write, "a" append, "+" update,
    // and "b" for BINARY (matters on Windows, where text mode translates \n).
    FILE *f = fopen("data.txt", "w");
    if (!f) { perror("fopen"); return 1; }      // perror prints the errno reason
    fprintf(f, "line %d\n", 1);
    fclose(f);

    // Read it back line by line — fgets is bounded (won't overflow buf):
    f = fopen("data.txt", "r");
    if (!f) { perror("fopen"); return 1; }
    char buf[256];
    while (fgets(buf, sizeof buf, f))            // reads up to sizeof buf-1, NUL-terminates
        fputs(buf, stdout);
    fclose(f);
    return 0;
}
```

For **binary** I/O, open with `"rb"`/`"wb"` and use `fread`/`fwrite` with `sizeof`. Be wary of writing raw structs to disk (padding/endianness make them non-portable — §19):

```c
#include <stdio.h>
#include <stdint.h>

int main(void)
{
    uint32_t records[3] = {100, 200, 300};
    FILE *f = fopen("nums.bin", "wb");
    if (!f) { perror("fopen"); return 1; }
    fwrite(records, sizeof records[0], 3, f);   // returns # of elements written; check it
    fclose(f);
    return 0;
}
```

### 11.2 Low-level POSIX I/O: `open`/`read`/`write` **[I]**

Below `stdio` sit raw, *unbuffered* file descriptors (`int`s) from `<fcntl.h>`/`<unistd.h>`. You use these when you need precise control (non-blocking I/O, sockets — see [Networking](NETWORKING_GUIDE.md), exact byte counts, `O_*` flags). `read`/`write` return the number of bytes transferred (which may be *fewer* than requested — you must loop) or `-1` with `errno` set.

```c
#include <fcntl.h>      // open, O_*
#include <unistd.h>     // read, write, close
#include <string.h>
#include <stdio.h>

int main(void)
{
    // 0644 = owner rw, group/other r. The mode matters only when O_CREAT creates the file.
    int fd = open("log.txt", O_WRONLY | O_CREAT | O_APPEND, 0644);
    if (fd < 0) { perror("open"); return 1; }

    const char *msg = "event\n";
    size_t left = strlen(msg);
    const char *p = msg;
    while (left > 0) {                       // write may be partial — loop until done
        ssize_t n = write(fd, p, left);
        if (n < 0) { perror("write"); close(fd); return 1; }
        p += n; left -= (size_t)n;
    }
    close(fd);
    return 0;
}
```

### 11.3 Directories & file metadata **[I]**

POSIX `<dirent.h>` lists directories; `<sys/stat.h>` `stat()` gives metadata (size, type, mtime, permissions).

```c
#include <dirent.h>
#include <sys/stat.h>
#include <stdio.h>

int main(void)
{
    DIR *d = opendir(".");
    if (!d) { perror("opendir"); return 1; }
    struct dirent *e;
    while ((e = readdir(d)) != nullptr) {        // returns NULL at end (and on error — check errno)
        struct stat st;
        if (stat(e->d_name, &st) == 0)
            printf("%-20s %lld bytes %s\n", e->d_name, (long long)st.st_size,
                   S_ISDIR(st.st_mode) ? "[dir]" : "");
    }
    closedir(d);
    return 0;
}
```

### 11.4 `errno` and disciplined error handling **[I]**

Most C library/POSIX calls signal failure by returning a sentinel (`nullptr`, `-1`, `EOF`) and setting the global `errno` (from `<errno.h>`) to a code. Read `errno` *immediately* after a failed call (any later call may overwrite it). `perror("ctx")` prints `"ctx: <reason>"`; `strerror(errno)` gives the string. ⚡ `errno` is thread-local, so it is safe per-thread.

```c
#include <errno.h>
#include <string.h>
#include <stdio.h>

int main(void)
{
    FILE *f = fopen("/nope/missing", "r");
    if (!f) {
        // Capture errno immediately — printf below could clobber it otherwise.
        int e = errno;
        fprintf(stderr, "open failed: %s (errno %d)\n", strerror(e), e);
        return 1;
    }
    fclose(f);
    return 0;
}
```

### 11.5 Environment variables & program arguments **[I]**

Read env vars with `getenv` (returns `nullptr` if unset; **never modify the returned string**). Program arguments arrive via `int main(int argc, char **argv)` — `argv[0]` is the program name, `argv[argc]` is `nullptr`.

```c
#include <stdlib.h>
#include <stdio.h>

int main(int argc, char **argv)
{
    for (int i = 0; i < argc; ++i)
        printf("argv[%d] = %s\n", i, argv[i]);

    const char *home = getenv("HOME");       // or "USERPROFILE" on Windows
    printf("HOME = %s\n", home ? home : "(unset)");
    return 0;
}
```

### 11.6 Running other programs: `system()` vs `fork`/`exec`/`pipe` **[I]**

Two ways to run external commands, with very different safety profiles:

- **`system("cmd")`** runs a command *through the shell*. Convenient but **a security menace**: if any part of the command string comes from untrusted input, the user can inject shell metacharacters (`;`, `|`, `$(...)`) and execute arbitrary commands. **Never build a `system()` string from user input.** It is also slow (spawns a shell) and gives you no clean way to capture output.
- **`fork()` + `exec*()`** (POSIX) is the real tool: `fork` duplicates the process, the child `exec`s the target program *directly* (no shell, so **no injection** — arguments are passed as a real array, not parsed), and the parent `waitpid`s for it. Combine with `pipe()` to capture the child's output. This is what production code uses.

```c
#include <unistd.h>     // fork, execvp, pipe
#include <sys/wait.h>   // waitpid
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    pid_t pid = fork();
    if (pid < 0) { perror("fork"); return 1; }

    if (pid == 0) {
        // CHILD: replace this process image with `ls -l`.
        // execvp takes an ARGV ARRAY — no shell, so arguments can't be re-parsed
        // and there is no shell-injection surface. Last element MUST be NULL.
        char *args[] = { "ls", "-l", nullptr };
        execvp(args[0], args);
        perror("execvp");      // only reached if exec FAILED
        _exit(127);            // _exit (not exit) in a failed child: skip atexit/flush
    }

    // PARENT: wait for the child and read its exit status.
    int status;
    waitpid(pid, &status, 0);
    if (WIFEXITED(status))
        printf("child exited with %d\n", WEXITSTATUS(status));
    return 0;
}
```

> **Windows note:** `fork` does not exist natively; use `CreateProcess` (or `_spawnv`). On WSL2 the POSIX code above works as-is. For shelling out *safely* on any platform, prefer passing an argument array over a single shell string.

---

## 12. The C Standard Library Tour

The standard library is small but you must know it cold. Here are the workhorses, by header.

### 12.1 `<string.h>` — bytes & strings **[I]**

| Function | Purpose / caution |
|---|---|
| `strlen(s)` | Length up to NUL. O(n). UB if not NUL-terminated. |
| `strcmp(a,b)` / `strncmp` | Compare; returns <0/0/>0. Prefer the `n` form with a bound. |
| `memcpy(d,s,n)` | Copy `n` bytes. **UB if ranges overlap** — use `memmove`. |
| `memmove(d,s,n)` | Copy allowing overlap. |
| `memset(d,c,n)` | Fill `n` bytes with byte `c`. |
| `strcpy`/`strcat`/`sprintf` | ⚠️ Unbounded — overflow hazards. Prefer `snprintf`. |
| `strchr`/`strstr`/`strtok_r` | Search / tokenise (`strtok_r` is the reentrant, thread-safe version). |

### 12.2 `<stdlib.h>` — conversions, memory, process **[I]**

`malloc`/`calloc`/`realloc`/`free` (§7); `strtol`/`strtoul`/`strtod` (robust string→number with error reporting — **prefer over `atoi`**, which can't report errors and is UB on overflow); `qsort`/`bsearch` (§12.6); `exit`/`atexit`/`abort`; `getenv`; `rand`/`srand` (low quality — not for security; use a CSPRNG/`getrandom` for that).

```c
#include <stdlib.h>
#include <errno.h>
#include <stdio.h>

// Robust integer parsing with full error checking (atoi can't do this):
int parse_int(const char *s, long *out)
{
    char *end;
    errno = 0;
    long v = strtol(s, &end, 10);
    if (end == s || *end != '\0') return -1;      // no digits / trailing junk
    if (errno == ERANGE) return -1;               // overflow/underflow
    *out = v;
    return 0;
}

int main(void)
{
    long n;
    if (parse_int("42", &n) == 0) printf("ok %ld\n", n);
    if (parse_int("4x", &n) != 0) printf("rejected '4x'\n");
    return 0;
}
```

### 12.3 `<ctype.h>`, `<math.h>`, `<time.h>` **[I]**

- **`<ctype.h>`** — `isdigit`, `isalpha`, `isspace`, `toupper`… Pass the char *cast to `unsigned char`* (passing a plain negative `char` is UB): `isdigit((unsigned char)c)`.
- **`<math.h>`** — `sqrt`, `pow`, `floor`, `fabs`, `sin`… Link with `-lm` on Linux. Check domain errors (`sqrt(-1)` → NaN).
- **`<time.h>`** — `time`, `clock`, `difftime`, `strftime`, `localtime_r`/`gmtime_r` (use the `_r` reentrant forms).

### 12.4 `<assert.h>` **[I]**

`assert(cond)` aborts with a diagnostic if `cond` is false — for catching *programmer* bugs (violated invariants), not user errors. Compiling with `-DNDEBUG` removes all asserts, so **never put side effects inside `assert`** (they'd vanish in release builds). ⚡ C23 `static_assert(cond, "msg")` is the compile-time analogue for invariants checkable at compile time.

### 12.5 `<stdio.h>` formatting **[I]**

Master `printf`/`scanf` format specifiers — and their traps. A *mismatched* specifier (e.g. `%d` for a `long`, `%s` for an `int`) is **undefined behavior**, not a wrong number. Key specifiers: `%d`/`%u`/`%ld`/`%lld`, `%zu` (`size_t`), `%f`/`%g`/`%.3f`, `%p` (cast to `void*`), `%x`, `%s`, `%c`, `%%`. **Never use `scanf("%s", buf)` unbounded** — use a width (`%255s`) or read with `fgets` then parse.

### 12.6 `qsort` and `bsearch` — generic algorithms via `void *` **[I]**

These sort/search arrays of *any* type using a function-pointer comparator (§5.7). The comparator receives `const void *` pointers to two elements and returns negative/zero/positive.

```c
#include <stdlib.h>
#include <stdio.h>

int cmp_int(const void *a, const void *b)
{
    int x = *(const int *)a, y = *(const int *)b;
    // Return (x>y)-(x<y) — NOT x-y, which can overflow for large/negative ints!
    return (x > y) - (x < y);
}

int main(void)
{
    int a[] = {5, 2, 9, 1, 7};
    size_t n = sizeof a / sizeof a[0];
    qsort(a, n, sizeof a[0], cmp_int);                 // sort ascending
    for (size_t i = 0; i < n; ++i) printf("%d ", a[i]);
    putchar('\n');                                     // 1 2 5 7 9

    int key = 7;
    int *hit = bsearch(&key, a, n, sizeof a[0], cmp_int);  // array must be sorted!
    printf("found 7: %s\n", hit ? "yes" : "no");
    return 0;
}
```

> **Comparator gotcha (above):** never write `return x - y;` — for large or negative values that subtraction overflows (UB) and gives wrong orderings. Use `(x > y) - (x < y)`.

---

## 13. Multi-File Projects & Build Systems

### 13.1 Headers vs sources & the one-definition discipline **[I/A]**

Real programs span many files. The model: a **header** (`.h`) holds *declarations* (the interface — prototypes, type definitions, `extern` globals, macros); a **source** (`.c`) holds *definitions* (the implementation). Other `.c` files `#include` the header to use the interface, and the linker connects the call sites to the one definition. Each `.c` plus everything it includes is one **translation unit**, compiled independently to a `.o`.

**Rules that keep this sane:**
- Put in the header *only* what callers need; keep implementation details `static` in the `.c`.
- Every header needs an **include guard** (§10.3).
- Headers should be **self-contained** (include what they use) and **idempotent**.
- A function or non-`static` global may have *exactly one definition* across all translation units (the One Definition Rule); multiple definitions → "multiple definition" linker error. ⚡ C23 standardised `constexpr` and made the rules around `inline` clearer.

```c
/* === geometry.h === */
#ifndef GEOMETRY_H
#define GEOMETRY_H
typedef struct { double x, y; } Vec2;
double vec2_len(Vec2 v);          // declaration only
#endif

/* === geometry.c === */
#include "geometry.h"
#include <math.h>
double vec2_len(Vec2 v) { return sqrt(v.x*v.x + v.y*v.y); }   // the one definition

/* === main.c === */
#include "geometry.h"
#include <stdio.h>
int main(void) { printf("%g\n", vec2_len((Vec2){3, 4})); }    // 5
```

```bash
gcc -std=c23 -Wall -Wextra -c geometry.c -o geometry.o
gcc -std=c23 -Wall -Wextra -c main.c     -o main.o
gcc geometry.o main.o -lm -o app          # link both objects + math lib
```

### 13.2 Static vs shared libraries **[I/A]**

- **Static library** (`.a` on Unix, `.lib` on Windows): an archive of `.o` files; the linker copies the needed code *into* your executable. Result: a bigger, self-contained binary with no runtime dependency. Build: `ar rcs libfoo.a foo.o bar.o`; link: `gcc main.o -L. -lfoo`.
- **Shared/dynamic library** (`.so` Linux, `.dylib` macOS, `.dll` Windows): loaded at runtime; one copy shared by many programs; updatable without relinking — but you must ship it and manage **ABI compatibility** (§17.5). Build: `gcc -shared -fPIC -o libfoo.so foo.c`.

### 13.3 Intro to CMake **[I/A]**

Hand-written Makefiles don't scale or port well. **CMake** is a *meta build system*: you describe targets declaratively in `CMakeLists.txt`, and CMake generates the native build (Makefiles, Ninja, Visual Studio, Xcode). It is the de-facto standard for cross-platform C/C++.

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(myapp C)

set(CMAKE_C_STANDARD 23)            # request C23
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)         # strict ISO C (no GNU extensions)

# A library target from its sources:
add_library(geometry STATIC src/geometry.c)
target_include_directories(geometry PUBLIC include)

# The executable, linking the library and the math lib:
add_executable(app src/main.c)
target_link_libraries(app PRIVATE geometry m)

# Turn on warnings for our targets:
target_compile_options(app PRIVATE -Wall -Wextra -Werror)
```

```bash
cmake -S . -B build              # configure (generates the build system in ./build)
cmake --build build              # compile
./build/app
```

> CMake also integrates testing (`enable_testing()` + `ctest`), sanitizer builds (`-DCMAKE_C_FLAGS="-fsanitize=address"`), and dependency fetching (`FetchContent`). See §17 for testing.

---

## 14. Concurrency: Processes, Threads, Atomics

Concurrency multiplies C's hazards: now you must also avoid **data races** (two threads touching the same memory, at least one writing, without synchronisation — which is **undefined behavior** in C). Get the model right or your program will work 999 times and corrupt data on the 1000th.

### 14.1 Processes vs threads **[A]**

A **process** has its own isolated address space (created by `fork`, §11.6) — strong isolation, but communication needs IPC (pipes, sockets, shared memory). A **thread** shares the process's address space with its siblings — cheap to create and to share data, but *that shared mutable memory is exactly the hazard*. Threads need explicit synchronisation (mutexes, atomics) around shared data.

### 14.2 POSIX threads (`pthreads`) **[A]**

The mature, ubiquitous threading API on Unix. Link with `-pthread`. You create threads with `pthread_create`, join them with `pthread_join`, and protect shared data with `pthread_mutex_t`.

```c
#include <pthread.h>
#include <stdio.h>

static long counter = 0;
static pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

void *worker(void *arg)
{
    long iters = *(long *)arg;
    for (long i = 0; i < iters; ++i) {
        pthread_mutex_lock(&lock);     // critical section: only one thread at a time
        counter++;                     // WITHOUT the lock this is a data race → UB
        pthread_mutex_unlock(&lock);
    }
    return nullptr;
}

int main(void)
{
    long iters = 1'000'000;
    pthread_t t1, t2;
    pthread_create(&t1, nullptr, worker, &iters);
    pthread_create(&t2, nullptr, worker, &iters);
    pthread_join(t1, nullptr);
    pthread_join(t2, nullptr);
    printf("counter = %ld\n", counter);   // exactly 2,000,000 thanks to the mutex
    return 0;
}
```

```bash
gcc -std=c23 -Wall -Wextra -pthread threads.c -o threads
```

### 14.3 C11 `<threads.h>` — the standard threading API **[A]**

> **⚡ Version note:** C11 added a *standard* threading API in `<threads.h>` (`thrd_create`, `mtx_lock`, `cnd_wait`, `thrd_join`) — portable in principle, but support is uneven (glibc has it; some platforms and MSVC historically lacked it). In practice pthreads (Unix) and Win32 threads (Windows) remain more widely used. Know `<threads.h>` exists; reach for pthreads when you need it to *just work*.

### 14.4 Atomics & the memory model (`<stdatomic.h>`) **[A]**

For simple shared counters/flags, a full mutex is overkill. **Atomics** (C11 `<stdatomic.h>`) give you operations that are indivisible and properly synchronised across threads, defined by the C **memory model**. `atomic_int` with `atomic_fetch_add` increments without a lock and without a data race. Memory *orderings* (`memory_order_relaxed` … `seq_cst`) control how surrounding non-atomic accesses may be reordered — `seq_cst` (the default) is the easiest to reason about; weaker orderings are an expert optimisation.

```c
#include <stdatomic.h>
#include <pthread.h>
#include <stdio.h>

static atomic_long counter = 0;       // a lock-free atomic counter

void *worker(void *arg)
{
    long iters = *(long *)arg;
    for (long i = 0; i < iters; ++i)
        atomic_fetch_add(&counter, 1);   // atomic, race-free, no mutex needed
    return nullptr;
}

int main(void)
{
    long iters = 1'000'000;
    pthread_t t1, t2;
    pthread_create(&t1, nullptr, worker, &iters);
    pthread_create(&t2, nullptr, worker, &iters);
    pthread_join(t1, nullptr); pthread_join(t2, nullptr);
    printf("%ld\n", atomic_load(&counter));   // 2,000,000
    return 0;
}
```

> **Data-race rule:** *any* concurrent access to the same non-atomic object where at least one is a write, without synchronisation (mutex/atomic/etc.), is a data race and **undefined behavior**. Use **ThreadSanitizer** (`-fsanitize=thread`, §15) to find races — they are nearly impossible to find by inspection. ⚡ C23 adds `thread_local` as a keyword for per-thread storage.

---

## 15. Writing Safe, Production C

"Production C" means C that an attacker can't break and a maintainer can't trip over. It is achieved less by cleverness than by *defensive habits plus relentless tooling*. This section is the playbook.

### 15.1 Defensive coding habits **[A]**

- **Validate every input** at the boundary (length, range, format) before trusting it.
- **Check every return value** that can fail (`malloc`, `fopen`, `read`, `snprintf` truncation). ⚡ Mark functions whose result must be checked with `[[nodiscard]]` (C23) so the compiler warns if it's ignored.
- **Initialise everything** — never read an uninitialised variable. `int x = 0;`, `char buf[N] = {0};`.
- **Track sizes alongside pointers** — pass `(ptr, len)` pairs, never a bare buffer.
- **Prefer bounded APIs**: `snprintf` over `sprintf`, `fgets` over `gets`, `memcpy(_, _, n)` with a checked `n`.
- **Single ownership**: for each allocation, exactly one owner frees it; document it.
- **Fail closed**: on error, default to the *safe* outcome (deny, return error, abort), never proceed with corrupt state.

```c
#include <stdlib.h>

// ⚡ C23: [[nodiscard]] forces callers to check the return.
[[nodiscard]] void *xmalloc(size_t n);   // wrapper that aborts on OOM (a common pattern)

[[maybe_unused]] static int debug_only(void) { return 1; }  // ⚡ C23: silence unused warning
```

### 15.2 The functions to avoid, and what to use instead **[A]**

The **CERT C Secure Coding Standard** codifies these. Memorise the swaps:

| Avoid | Why | Use instead |
|---|---|---|
| `gets` | No bound — *always* overflowable. **Removed in C11/C23.** | `fgets(buf, sizeof buf, stdin)` |
| `strcpy` / `strcat` | No destination bound → overflow | `snprintf`, or `memcpy` with a checked length |
| `sprintf` | No bound | `snprintf(buf, sizeof buf, ...)` |
| `scanf("%s", ...)` | Unbounded read | `fgets` + parse, or `scanf("%255s", ...)` |
| `atoi`/`atol` | No error/overflow reporting (UB on overflow) | `strtol` with `errno`/`end` checks (§12.2) |
| `strtok` | Not reentrant (static state) | `strtok_r` |
| `rand` for secrets | Predictable, low quality | OS CSPRNG (`getrandom`, `/dev/urandom`, `BCryptGenRandom`) |

### 15.3 The sanitizers — your most valuable tools **[A]**

Sanitizers instrument your program to *catch UB and memory errors at runtime*, with a precise report. They are the single biggest leap in C safety in the last decade. Run your tests and fuzzers under them.

| Sanitizer | Flag | Catches |
|---|---|---|
| **ASan** (Address) | `-fsanitize=address` | Heap/stack/global buffer overflow, use-after-free, double-free, leaks. |
| **UBSan** (Undefined Behavior) | `-fsanitize=undefined` | Signed overflow, OOB shifts, misaligned access, null deref, more. |
| **TSan** (Thread) | `-fsanitize=thread` | Data races (use alone — incompatible with ASan). |
| **MSan** (Memory, Clang) | `-fsanitize=memory` | Reads of uninitialised memory. |

```bash
# Development/CI build: ASan + UBSan together, with line numbers.
gcc -std=c23 -Wall -Wextra -g -O1 -fsanitize=address,undefined \
    -fno-omit-frame-pointer prog.c -o prog
./prog          # any OOB/UAF/UB now prints a detailed report and aborts
```

> ASan adds ~2× slowdown and memory — fine for tests, off for production releases. TSan must run alone. These are *non-negotiable* for serious C; wire them into CI.

### 15.4 valgrind **[A]**

`valgrind --leak-check=full ./prog` runs your *unmodified* binary on a synthetic CPU, catching leaks, invalid reads/writes, and uninitialised-value use. Slower than ASan and Linux-centric, but needs no recompilation and finds some things ASan doesn't. Use both.

### 15.5 Static analyzers **[A]**

These find bugs *without running* the code, via interprocedural analysis:
- **`clang-tidy`** — lint + modernisation + bug checks; integrates with CMake.
- **`cppcheck`** — standalone, catches OOB, leaks, null derefs.
- **`clang --analyze`** / **`scan-build`** — the Clang Static Analyzer's path-sensitive checks.
- **GCC `-fanalyzer`** — GCC's built-in static analysis (good for double-free, leak, null paths).

### 15.6 Fuzzing **[A]**

A **fuzzer** feeds your code millions of mutated/random inputs to find crashes and UB — devastatingly effective on parsers and anything that touches untrusted bytes. **libFuzzer** (`-fsanitize=fuzzer`) or **AFL++**, *combined with ASan/UBSan*, is how modern C libraries (OpenSSL, curl) are hardened.

```c
// fuzz_target.c — compile: clang -fsanitize=fuzzer,address,undefined fuzz_target.c
#include <stddef.h>
#include <stdint.h>

int parse(const uint8_t *data, size_t size);   // your parser under test

// libFuzzer calls this with random inputs, under ASan/UBSan, forever.
int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size)
{
    parse(data, size);    // any crash/UB/leak is caught and minimised automatically
    return 0;
}
```

### 15.7 Hardening flags **[A]**

For release builds, add defence-in-depth: `-fstack-protector-strong` (stack-smashing canaries), `-D_FORTIFY_SOURCE=2 -O2` (compile-time/runtime checks on `memcpy`/`sprintf` sizes), `-fPIE -pie` (ASLR), `-Wl,-z,relro,-z,now` (hardened relocations), `-fcf-protection` (control-flow integrity). These cost almost nothing and stop large classes of exploits.

---

## 16. Debugging & Tooling

### 16.1 gdb / lldb **[A]**

A symbolic debugger lets you stop, step, and inspect a running program. Compile with `-g` (and `-Og` or `-O0` so the optimiser doesn't scramble the line mapping). Core skills:

```bash
gcc -std=c23 -g -O0 prog.c -o prog
gdb ./prog
```
```
(gdb) break main          # set a breakpoint (or `break file.c:42`)
(gdb) run arg1 arg2       # start with arguments
(gdb) next / step         # step over / into
(gdb) print x   / p *p    # inspect a variable / dereference a pointer
(gdb) backtrace (bt)      # the call stack — where am I?
(gdb) watch counter       # break when `counter` changes
(gdb) continue            # resume
```

`lldb` (LLVM) is equivalent with similar commands (`b`, `r`, `n`, `s`, `p`, `bt`). On Windows, the Visual Studio debugger or WinDbg fill this role.

### 16.2 Core dumps **[A]**

When a program crashes, the OS can dump its memory image to a **core file** so you can debug the crash *after the fact*. Enable with `ulimit -c unlimited` (Linux), then `gdb ./prog core` and `bt` to see exactly where it died. Essential for diagnosing crashes you can't reproduce live.

### 16.3 Profiling — measure before optimising **[A]**

- **`perf`** (Linux): `perf record ./prog` then `perf report` — sampling profiler, shows where CPU time goes with near-zero overhead. The first tool to reach for.
- **`gprof`**: compile with `-pg`, run, then `gprof ./prog gmon.out` — call-graph profiling.
- **`valgrind --tool=callgrind`** + KCachegrind: instruction-level call graphs (slow but precise).
- **Cachegrind**: cache-miss analysis (relevant to §18).

> **Golden rule:** *profile first.* Programmer intuition about hotspots is usually wrong. Optimise what the profiler proves is hot, then re-measure.

---

## 17. Testing & Maintainability

### 17.1 Why and how to unit-test C **[A]**

C has no built-in test framework, but several mature ones exist. Tests pin behaviour, catch regressions, and (run under sanitizers) surface memory bugs. Popular choices:

- **Unity** — tiny, single-file, great for embedded.
- **Check** — fork-isolated tests (a crashing test doesn't kill the suite); Unix-centric.
- **CMocka** — supports mocking and runs everywhere; integrates with CMake/CTest.
- **Criterion** — modern, auto-discovers tests.

```c
// A minimal hand-rolled test harness — no framework needed to start.
#include <stdio.h>
#include <stdlib.h>

static int failures = 0;
#define CHECK(cond) do { \
    if (!(cond)) { printf("FAIL %s:%d  %s\n", __FILE__, __LINE__, #cond); failures++; } \
} while (0)

int add(int a, int b) { return a + b; }

int main(void)
{
    CHECK(add(2, 3) == 5);
    CHECK(add(-1, 1) == 0);
    printf(failures ? "%d failure(s)\n" : "all passed\n", failures);
    return failures ? EXIT_FAILURE : EXIT_SUCCESS;   // non-zero exit fails CI
}
```

### 17.2 Continuous Integration **[A]**

Wire tests into CI ([GitHub Actions](GITHUB_ACTIONS_CICD_GUIDE.md), GitLab CI) and run them under multiple compilers (GCC *and* Clang catch different things) and with sanitizers. A good matrix: `{gcc, clang} × {release, asan+ubsan} × {-std=c17, -std=c23}`. Add static analysis and a `cppcheck`/`clang-tidy` gate. Use [Git](GIT_GUIDE.md) hooks/PR checks to block merges that fail.

### 17.3 Code organisation & documentation **[A]**

Keep public headers minimal and stable; hide implementation behind `static` and *opaque pointers* (a `typedef struct Foo Foo;` whose definition lives only in the `.c`, so callers can't see or depend on the layout — the cleanest way to do encapsulation/information-hiding in C). Document each public function's contract: ownership (who frees), preconditions, error returns, and thread-safety. Doxygen comments generate browsable docs.

### 17.4 Opaque pointers — encapsulation in C **[A]**

```c
/* === stack.h (public) — callers see only an opaque handle === */
#ifndef STACK_H
#define STACK_H
typedef struct Stack Stack;            // INCOMPLETE type: layout hidden
Stack *stack_new(void);
void   stack_push(Stack *, int);
int    stack_pop(Stack *, int *out);   // returns 0 ok / -1 empty
void   stack_free(Stack *);
#endif

/* === stack.c (private) — the layout lives here, nowhere else === */
#include "stack.h"
#include <stdlib.h>
struct Stack { int *data; size_t len, cap; };   // callers CANNOT access these fields
/* ... implementations ... */
```

Because callers only ever hold a `Stack *`, you can change the struct's internals without recompiling or breaking them — true module boundaries.

### 17.5 ABI & versioning for libraries **[A]**

A shared library's **ABI** (Application Binary Interface) is the binary contract: struct layouts, function signatures, symbol names. Changing it without bumping the version breaks every program linked against it (at *runtime*, often mysteriously). Discipline: never reorder/insert struct fields in a public struct (use opaque pointers to avoid the issue), version your symbols, use semantic versioning with an `soname`, and keep a documented stable API surface. This is why mature C libraries lean so heavily on opaque pointers — they keep the ABI free to evolve.

---

## 18. Performance

C is chosen for speed, but fast C comes from *understanding the machine*, not micro-tricks. Measure first (§16.3); then apply these principles.

### 18.1 Cache-friendliness — usually the biggest win **[A]**

Modern CPUs are starved for memory, not compute: a cache miss costs ~100× an arithmetic op. So **data layout dominates performance**. Prefer contiguous arrays over pointer-chasing linked structures; iterate in memory order (row-major for 2-D arrays); use **structure-of-arrays** instead of array-of-structures when you process one field across many records (it keeps the hot field contiguous and the cache lines full). This single idea outperforms most algorithmic micro-optimisation.

### 18.2 `inline` and `restrict` **[A]**

- **`inline`** suggests the compiler paste a small function's body at the call site, eliminating call overhead (it is a hint; the compiler decides, and `-O2` inlines aggressively anyway). Useful for tiny hot helpers in headers (`static inline`).
- **`restrict`** (C99) is a *promise* that, for the lifetime of the pointer, the object it points to is accessed *only* through that pointer (no aliasing). It lets the compiler keep values in registers and vectorise loops. Powerful for numeric kernels — but if you lie (the pointers *do* alias), it's **undefined behavior**.

```c
// `restrict` promises src and dst don't overlap → the compiler can vectorise freely.
void scale(size_t n, double *restrict dst, const double *restrict src, double k)
{
    for (size_t i = 0; i < n; ++i)
        dst[i] = src[i] * k;     // no need to reload src/dst between iterations
}
```

### 18.3 Avoiding premature optimisation **[A]**

> "Premature optimisation is the root of all evil." — Knuth.

Write clear, correct code first; profile; optimise only proven hotspots; re-measure to confirm the change actually helped (it often doesn't, or pessimises elsewhere). Let `-O2` do the routine work — it is far better at instruction selection than hand-tuning. Reserve manual effort for algorithm/data-structure choices and cache behaviour, which the compiler *can't* fix for you.

---

## 19. Gotchas & Best Practices

A consolidated, scannable checklist. Most C bugs are on this list.

### 19.1 Undefined-behavior recap **[I/A]**
- Signed overflow, OOB access, null/dangling deref, use-after-free/double-free, uninitialised reads, data races, strict-aliasing violations, multiple unsequenced modifications, out-of-range shifts. **All UB. Use ASan+UBSan+valgrind+TSan to catch them.**
- UB lets the optimiser delete your "safety checks." Never *cause* UB to *detect* a condition — check *before* the dangerous operation.

### 19.2 Integer issues **[I/A]**
- `long` is 32-bit on 64-bit Windows, 64-bit elsewhere → use `<stdint.h>` for fixed width.
- Mixing signed/unsigned converts signed → unsigned (negatives become huge). Heed `-Wsign-compare`.
- Use `size_t`/`%zu` for sizes and indices; never `int` for a length you might subtract past zero.
- Overflow-check allocation sizes (`calloc` or `ckd_mul`) — `malloc(a*b)` is a classic heap exploit.

### 19.3 String & buffer rules **[I/A]**
- Every string needs its NUL; every buffer write needs a bound. `snprintf` over `sprintf`/`strcpy`; `fgets` over `gets`; bounded `scanf`.
- `strncpy` does **not** always NUL-terminate (if the source fills the buffer) — a famous footgun. Terminate manually or avoid it.
- `printf`/`scanf` format must match the argument types exactly — mismatches are UB.

### 19.4 Memory rules **[I/A]**
- One `free` per allocation; check every `malloc`; `realloc` into a temporary; null pointers after `free`.
- Don't return pointers to stack locals; don't cast away `const` to write; don't cast `malloc`'s result.
- Decide and *document* ownership for every allocation.

### 19.5 Portability & endianness **[I/A]**
- **Endianness**: don't write raw multi-byte integers/structs to files or sockets and expect another machine to read them. Serialise field-by-field in a fixed byte order (network order for [Networking](NETWORKING_GUIDE.md): `htonl`/`ntohl`).
- **Struct layout** (padding) is implementation-defined — never `memcpy` a struct to the wire.
- **`char` signedness**, `int` size, and shift behaviour vary — use explicit types and cast for `<ctype.h>`.
- Path separators (`/` vs `\`) and line endings differ; open binary files with `"b"`.

### 19.6 Style & correctness habits **[I/A]**
- Always brace `if`/`for`/`while` bodies (defeats `goto fail` bugs).
- `-Wall -Wextra -Werror` from day one; add `-Wshadow -Wconversion`.
- Prefer `static` for internal linkage; opaque pointers for public types.
- `constexpr`/`enum`/`static inline` over `#define` for constants and tiny functions (type-safe).
- ⚡ Use C23 quality-of-life features where supported: `nullptr`, `[[nodiscard]]`, `[[fallthrough]]`, `static_assert`, `<stdckdint.h>`.

---

## 20. Study Path & Build-to-Learn Projects

Theory sticks only when you *write* C and then *break* it under the sanitizers. Follow this path; build every project with `-std=c23 -Wall -Wextra -Werror -fsanitize=address,undefined` and run valgrind on it.

**Stage 1 — Foundations (weeks 1–2).** §§1–4. Goal: comfort with the compile pipeline, types, control flow, functions. Internalise integer promotion and the signed/unsigned trap.
- *Project — CLI calculator:* parse `argv` with `strtol`, evaluate, handle every error path (bad input, overflow, divide-by-zero) without UB. Forces robust input validation.

**Stage 2 — Pointers & memory (weeks 3–5).** §§5–7. The make-or-break stage. Re-read §5 and §7 until pointer arithmetic, decay, and the malloc/free discipline are reflexive.
- *Project — dynamic array (`vector`) library:* a growable `int`/generic array with `push`/`pop`/`get`/`free`, `realloc`-based growth, overflow-checked sizing, and an opaque-pointer API. The canonical "do I understand C memory" exercise.
- *Project — your own string library:* length-prefixed strings (avoid NUL pitfalls), safe concat/format, with a full test suite. Run it under ASan and fix every report.

**Stage 3 — Aggregates, UB & the preprocessor (weeks 6–7).** §§8–10. Build mental models of layout/padding and a *gut fear* of UB.
- *Project — a tiny tagged-union interpreter* (or a JSON/CSV parser): structs, unions+enum tags, recursion, and a fuzz target (§15.6). Fuzz it until it stops crashing — this teaches UB viscerally.

**Stage 4 — Systems & the stdlib (weeks 8–10).** §§11–13. Talk to the OS; structure a real multi-file project with CMake.
- *Project — a `grep`-like CLI* or a small key-value store with file persistence: file I/O, `errno` handling, directory walking, `qsort`/`bsearch`, split across headers/sources, built with CMake and tested in CI.
- *Project — a custom memory allocator* (a bump or free-list arena over a big `malloc` block): cements alignment, pointer arithmetic, and lifetime reasoning.

**Stage 5 — Concurrency, safety & performance (weeks 11–14).** §§14–18. Production discipline.
- *Project — a tiny multithreaded HTTP server* (POSIX sockets + a pthread/thread-pool worker model): combines [Networking](NETWORKING_GUIDE.md), threads, mutexes/atomics, careful buffer handling, and TSan to prove it's race-free. Profile it with `perf` and make it cache-friendly.
- *Harden everything:* run TSan/ASan/UBSan in CI, add a libFuzzer target to the request parser, and apply the §15.7 hardening flags.

**Where to go next.** Read real C: SQLite, redis, curl, the Linux kernel coding style, and `musl` libc are masterclasses. For memory safety without C's footguns, study [Rust](RUST_GUIDE.md) (ownership replaces malloc/free discipline) and [Go](GO_GUIDE.md) (GC + goroutines); contrast their trade-offs with what you now understand about C. Keep the CERT C standard and your sanitizers close — in C, *the tooling is the safety net the language doesn't give you.*
