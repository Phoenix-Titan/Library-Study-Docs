# Kotlin — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I've never written Kotlin" to "I write idiomatic, concurrent, production Kotlin" — without an internet connection. Every concept is explained in prose first (what it is, why it exists, when to use it) and then shown with heavily-commented, runnable code. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Kotlin 2.x** (current in 2026). Things worth knowing about modern Kotlin:
> - **The K2 compiler is the default** (since Kotlin 2.0). It is a full rewrite of the frontend: faster compilation, better type inference, smarter smart-casts, and a unified analysis pipeline shared across JVM/JS/Native. You generally don't change code for it — but a few edge-case smart-casts that the old compiler missed now "just work".
> - **Kotlin Multiplatform (KMP) is stable and mature.** You can share business logic across Android, iOS, desktop JVM, web (Wasm/JS), and server. Compose Multiplatform brings the same UI toolkit to Android/desktop/iOS/web.
> - **Kotlin is the default, Google-recommended language for Android.** New Android docs, samples, and APIs (Jetpack, Compose) are Kotlin-first.
> - **Coroutines and `Flow`** are the standard concurrency and reactive-streams model and are assumed throughout.
> - Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**, so cross-platform notes (path separators, `gradlew.bat` vs `./gradlew`, shells) are called out.
>
> Kotlin runs primarily on the **JVM** (compiles to Java bytecode), but also targets **JavaScript** (Kotlin/JS), **native machine code** (Kotlin/Native via LLVM), and **WebAssembly** (Kotlin/Wasm). It interoperates seamlessly with Java — cross-references to **JAVA_GUIDE.md** are noted, but this guide is self-contained. Confirm exact APIs at kotlinlang.org.

---

## Table of Contents

1. [What Kotlin Is, Install & Running Code](#1-what-kotlin-is-install--running-code) **[B]**
2. [Basics: Values, Types & Strings](#2-basics-values-types--strings) **[B]**
3. [Null Safety — The Defining Feature](#3-null-safety--the-defining-feature) **[B]**
4. [Control Flow](#4-control-flow) **[B]**
5. [Functions](#5-functions) **[B/I]**
6. [Lambdas, Higher-Order Functions & Scope Functions](#6-lambdas-higher-order-functions--scope-functions) **[I]**
7. [Object-Oriented Kotlin](#7-object-oriented-kotlin) **[I]**
8. [Special Classes: data, enum, sealed, value, annotation](#8-special-classes-data-enum-sealed-value-annotation) **[I]**
9. [Generics & Variance](#9-generics--variance) **[I/A]**
10. [Collections & Sequences](#10-collections--sequences) **[I]**
11. [Coroutines & Flow](#11-coroutines--flow) **[A]**
12. [File System, OS Info & Command Execution](#12-file-system-os-info--command-execution) **[I]**
13. [Exceptions & Result](#13-exceptions--result) **[I]**
14. [Advanced: Delegation, DSLs, inline, Contracts](#14-advanced-delegation-dsls-inline-contracts) **[A]**
15. [Java Interop](#15-java-interop) **[I/A]**
16. [Tooling & Ecosystem](#16-tooling--ecosystem) **[I]**
17. [Gotchas & Best Practices](#17-gotchas--best-practices) **[A]**
18. [Study Path & Build-to-Learn Projects](#18-study-path--build-to-learn-projects)

---

## 1. What Kotlin Is, Install & Running Code

### 1.1 What Kotlin is and why it exists **[B]**

Kotlin is a statically-typed, general-purpose programming language created by JetBrains (the company behind IntelliJ IDEA) and first released in 2016. It was designed to be a **better Java** that runs on the same Java Virtual Machine (JVM) and interoperates perfectly with existing Java code and libraries. Google adopted it as a first-class Android language in 2017 and made it the *recommended* language in 2019.

Why does Kotlin exist when Java already runs everywhere? Java, especially the older versions Kotlin was designed against, carried a lot of friction:

- **`NullPointerException`s everywhere.** Any reference could be `null`, and the type system did nothing to warn you. Tony Hoare, who invented the null reference in 1965, later called it his "billion-dollar mistake." Kotlin bakes nullability into the type system (see §3) so the compiler stops most NPEs before your program ever runs.
- **Verbose boilerplate.** Getters, setters, `equals`, `hashCode`, `toString`, builder patterns — Java made you write all of it by hand. Kotlin generates it (`data class`, properties).
- **No expression-oriented constructs.** In Java, `if` and `switch` are statements; you can't assign their result to a variable directly. In Kotlin almost everything is an expression.
- **No first-class functions for years.** Kotlin treats functions and lambdas as values from day one, enabling concise functional code.

The design philosophy: **concise, safe, pragmatic, and interoperable**. Kotlin doesn't try to be academically pure; it tries to make everyday programming pleasant and to eliminate whole categories of bugs at compile time.

**Where Kotlin runs:**
- **Kotlin/JVM** — compiles to `.class` bytecode, runs on any JVM, uses the entire Java ecosystem. This is the most common target and what this guide assumes unless stated.
- **Kotlin/Android** — Kotlin/JVM specialized for Android (ART runtime, Jetpack libraries, Compose UI).
- **Kotlin/JS** — transpiles to JavaScript for browsers/Node.
- **Kotlin/Native** — compiles to standalone native binaries via LLVM (iOS, embedded, CLI tools with no JVM).
- **Kotlin/Wasm** — compiles to WebAssembly.
- **Kotlin Multiplatform (KMP)** — share a common module across all the above.

### 1.2 Installing Kotlin **[B]**

You need a **JDK** (Java Development Kit) first because Kotlin/JVM runs on the JVM. Install JDK 17 or 21 (LTS releases). Then pick one of these ways to get the Kotlin compiler:

- **IntelliJ IDEA** (Community Edition is free): bundles everything. The easiest path for serious work — full autocomplete, the K2 compiler, a built-in REPL, and one-click run.
- **Android Studio**: IntelliJ specialized for Android; also bundles Kotlin.
- **Command-line compiler (`kotlinc`)**:
  - Windows: `choco install kotlinc` (Chocolatey) or `scoop install kotlin`, or download the zip from GitHub and add `bin/` to `PATH`.
  - macOS: `brew install kotlin`.
  - Linux: `sdk install kotlin` (via SDKMAN!) or your package manager.
- **SDKMAN!** (great cross-platform version manager for JVM tools): `sdk install kotlin`, `sdk install gradle`, `sdk install java`.

Verify:

```bash
kotlinc -version       # prints the compiler version, e.g. "info: kotlinc-jvm 2.x"
java -version          # confirm a JDK is on PATH
```

### 1.3 Your first program — `fun main` **[B]**

Every Kotlin program starts at a top-level function named `main`. Unlike Java, you do **not** need a class wrapping it.

```kotlin
// hello.kt
// `fun` declares a function. `main` is the entry point the runtime calls.
fun main() {
    // println writes a line to standard output. String literals use double quotes.
    println("Hello, Kotlin!")
}
```

If you want command-line arguments, declare `main` with an `Array<String>` parameter:

```kotlin
// args.kt
fun main(args: Array<String>) {
    // args holds the program arguments. Indexing is zero-based.
    println("You passed ${args.size} arguments")
    for (arg in args) println(arg)
}
```

### 1.4 Compiling and running **[B]**

```bash
# Compile a single file into a runnable JAR, then run it on the JVM:
kotlinc hello.kt -include-runtime -d hello.jar   # -include-runtime bundles the Kotlin stdlib
java -jar hello.jar

# Or compile-and-run a script-like file directly (slower; compiles each time):
kotlinc hello.kt -include-runtime -d hello.jar && java -jar hello.jar

# Run a .kts script (Kotlin script) without producing a JAR:
kotlinc -script myscript.kts
```

The entry point's fully-qualified name is derived from the file: `hello.kt` produces a class `HelloKt` with a static `main`. You can override that with `@file:JvmName("App")` at the top of the file.

### 1.5 The REPL **[B]**

A REPL (Read-Eval-Print Loop) lets you type expressions and see results immediately — great for learning and experiments.

```bash
kotlinc            # starts the interactive Kotlin shell
>>> val x = 21
>>> x * 2
res1: kotlin.Int = 42
>>> "abc".uppercase()
res2: kotlin.String = ABC
>>> :quit          # exit (or Ctrl-D / Ctrl-Z then Enter on Windows)
```

IntelliJ also has **Kotlin Scratch files** (`.kts` scratch) and a "Kotlin REPL" tool window that runs against your project classpath.

### 1.6 Kotlin vs Java at a glance **[B]**

| Concern | Java | Kotlin |
|---|---|---|
| Null safety | Any reference can be null; NPEs at runtime | `String` (non-null) vs `String?` (nullable) enforced by compiler (§3) |
| Boilerplate (POJO) | Manual getters/setters/equals/hashCode | `data class` generates them (§8) |
| Variables | `final` is opt-in | `val` (immutable) vs `var` (mutable); `val` encouraged (§2) |
| `if`/`switch` | Statements | Expressions — return values (§4) |
| Semicolons | Required | Optional (rarely used) |
| Functions | Must live in a class | Top-level functions allowed |
| Checked exceptions | Forced `throws`/`try` | None — no checked exceptions (§13) |
| Coroutines | Threads / `CompletableFuture` | Lightweight coroutines + structured concurrency (§11) |
| Extension functions | Not possible | Add methods to existing types without subclassing (§5) |
| Type inference | Limited (`var` since Java 10) | Pervasive — `val x = 42` infers `Int` |

Crucially, the interop is two-way and frictionless: you can call Java from Kotlin and Kotlin from Java in the same project (§15). This means you can adopt Kotlin gradually in a Java codebase.

---

## 2. Basics: Values, Types & Strings

### 2.1 `val` vs `var` — immutability by default **[B]**

Kotlin gives you two keywords to declare a name that holds a value:

- **`val`** (from "value") declares a **read-only** reference. Once assigned, you cannot reassign it. This is the equivalent of `final` in Java, but it's the *default thing you reach for*.
- **`var`** (from "variable") declares a **mutable** reference that you can reassign.

**Why prefer `val`?** Immutable references make code easier to reason about: once you've seen where a `val` is set, you know it never changes, so you don't have to scan the whole function for reassignments. It prevents a class of bugs where a value is accidentally clobbered, and it makes concurrent code safer because immutable data can be shared without locks. The rule of thumb: **use `val` unless you have a concrete reason to reassign, then switch to `var`.** Many idiomatic Kotlin functions never use `var` at all.

Note the distinction between an immutable *reference* and an immutable *object*. `val list = mutableListOf(1, 2)` means you can't point `list` at a different list, but you can still `list.add(3)` — the object is mutable, the binding is not.

```kotlin
fun main() {
    val name = "Ada"        // read-only; type String is inferred
    // name = "Grace"       // COMPILE ERROR: val cannot be reassigned

    var count = 0           // mutable
    count = count + 1       // fine
    count += 1              // also fine

    val list = mutableListOf(1, 2)
    list.add(3)             // OK — we mutate the object, not the binding
    println("$name $count $list")  // Ada 2 [1, 2, 3]
}
```

### 2.2 Type inference and explicit types **[B]**

Kotlin is statically typed: every value has a type known at compile time. But you rarely have to write the type — the compiler **infers** it from the initializer. You can still annotate explicitly when it aids clarity or when there's no initializer.

```kotlin
val a = 42              // inferred Int
val b = 3.14            // inferred Double
val c = "hi"            // inferred String
val d: Long = 42        // explicit type annotation
val e: Double = 5.0     // explicit
var f: String           // declared without init...
f = "set later"         // ...must be assigned before use

// You MUST annotate when there's no way to infer, e.g. function parameters.
fun add(x: Int, y: Int): Int = x + y
```

### 2.3 Basic types **[B]**

Kotlin's basic types are objects (not Java's "primitives"), but the compiler optimizes them to primitives where possible, so there's no performance penalty.

| Type | Description | Example |
|---|---|---|
| `Int` | 32-bit signed integer | `42` |
| `Long` | 64-bit signed integer | `42L` |
| `Short` / `Byte` | 16-bit / 8-bit integer | `1.toShort()` |
| `Double` | 64-bit IEEE-754 float (default for decimals) | `3.14` |
| `Float` | 32-bit float | `3.14f` |
| `Boolean` | `true` / `false` | `true` |
| `Char` | a single 16-bit Unicode character | `'A'` |
| `String` | immutable sequence of characters | `"text"` |
| `UInt`/`ULong`/`UByte`/`UShort` | unsigned integers | `42u` |

```kotlin
fun main() {
    val big = 1_000_000          // underscores allowed as digit separators
    val hex = 0xFF               // 255
    val bin = 0b1010             // 10
    val longVal = 10L            // L suffix = Long
    val floatVal = 2.5f          // f suffix = Float
    val unsigned = 42u           // u suffix = UInt

    // No implicit widening: convert explicitly with toX() functions.
    val i: Int = 5
    val l: Long = i.toLong()     // must convert; `val l: Long = i` is an ERROR
    println(l + 1L)              // 6

    // Integer division truncates; mix with a Double to get a Double result.
    println(7 / 2)               // 3   (Int / Int)
    println(7.0 / 2)             // 3.5 (Double / Int -> Double)
    println(7 % 3)               // 1   (modulo)

    // Bitwise ops use named infix functions (no &, |, ^ operators on Int):
    println(0b1100 and 0b1010)   // 8  (1000)
    println(0b1100 or 0b1010)    // 14 (1110)
    println(0b1100 xor 0b1010)   // 6  (0110)
    println(1 shl 4)             // 16 (shift left)
    println(16 shr 2)            // 4  (shift right)
}
```

### 2.4 Strings, templates & multiline strings **[B]**

Strings are immutable. The standout feature is **string templates** (interpolation) using `$`.

```kotlin
fun main() {
    val user = "Ada"
    val score = 95.5

    // $name inlines a simple variable; ${expression} inlines any expression.
    println("$user scored $score%")              // Ada scored 95.5%
    println("Next year: ${score + 10}")          // Next year: 105.5
    println("Name length: ${user.length}")       // Name length: 3

    // To print a literal dollar sign, escape it:
    println("Price: \$5")                         // Price: $5

    // Multiline (raw) strings use triple quotes; no escaping, keeps newlines.
    val json = """
        {
            "name": "$user",
            "score": $score
        }
    """.trimIndent()    // trimIndent() removes the common leading whitespace
    println(json)

    // Common String operations (all return NEW strings — String is immutable):
    val t = "  Hello, World  "
    println(t.trim())                  // "Hello, World"
    println(t.trim().lowercase())      // "hello, world"
    println("a,b,c".split(","))        // [a, b, c]
    println(listOf("a", "b").joinToString("-"))  // a-b
    println("hello".replace("l", "L")) // heLLo
    println("hello".startsWith("he"))  // true
    println("hello".uppercase())       // HELLO
    println("hello".contains("ell"))   // true (substring test)
    println("hello"[0])                // h  (Char, index access)
    println("hello".reversed())        // olleh
    println("42".toInt())              // 42 (parse; throws if invalid)
    println("x".toIntOrNull())         // null (safe parse, see §3)
}
```

### 2.5 Operators & equality **[B]**

```kotlin
fun main() {
    // Arithmetic: + - * / %     Comparison: < <= > >= == !=
    // Logical (short-circuit): && || !
    println(1 < 2 && 2 < 3)            // true
    println(false || "x".isNotEmpty()) // true

    // Structural equality (==) calls .equals(); referential equality (===) checks identity.
    val a = "ab" + "c"      // could be a new object
    val b = "abc"
    println(a == b)         // true  — same VALUE (calls equals)
    println(a === b)        // may be true/false — same OBJECT identity

    // Ranges & membership (see §4): `in` and `!in`
    println(5 in 1..10)     // true
    println('c' in 'a'..'z')// true

    // Elvis and safe-call operators are about null safety — covered in §3.
}
```

> **Note:** Kotlin has **no ternary operator** (`cond ? a : b`). It doesn't need one because `if` is an expression: `val s = if (x > 0) "pos" else "neg"` (see §4).

---

## 3. Null Safety — The Defining Feature

### 3.1 The billion-dollar mistake **[B]**

In most languages any object reference can be `null`. Calling a method on a `null` reference throws a `NullPointerException` (NPE) — and the type system gives you *no warning at compile time*. These crashes are so common and so costly that Tony Hoare, who introduced the null reference in 1965, called it his "billion-dollar mistake."

Kotlin's central design decision is to make **nullability part of the type system**. By default, a type like `String` **cannot hold null**. If you want a value that might be absent, you opt in by writing `String?` (with a trailing `?`). The compiler then *forces* you to handle the null case before you can dereference it. The result: most NPEs become compile errors.

```kotlin
fun main() {
    var name: String = "Ada"
    // name = null            // COMPILE ERROR: null can't be a value of non-null type String

    var maybe: String? = "Ada" // the ? makes it nullable
    maybe = null               // now allowed

    // You can't just use a nullable directly — the compiler stops you:
    // println(maybe.length)   // COMPILE ERROR: maybe could be null
}
```

### 3.2 Safe call `?.` and the four tools **[B]**

To work with nullable types you have a small, precise toolkit. Here is each tool, why it exists, and when to use it.

**Safe call `?.`** — calls the member only if the receiver is non-null; otherwise the whole expression evaluates to `null`. Use it when "do nothing / produce null if absent" is acceptable.

```kotlin
fun main() {
    val maybe: String? = null
    println(maybe?.length)          // null  (no crash; short-circuits)

    val real: String? = "hello"
    println(real?.length)           // 5

    // Chaining: each link short-circuits to null if any is null.
    // val city = user?.address?.city
}
```

**Elvis operator `?:`** — provides a default when the left side is null. Named "Elvis" because `?:` looks like Elvis Presley's hair. Use it to supply a fallback or to early-return/throw.

```kotlin
fun greet(name: String?): String {
    // If name is null, use "Guest".
    val safe = name ?: "Guest"
    return "Hello, $safe"
}

fun lengthOrThrow(s: String?): Int {
    // Elvis can early-return or throw on the right side:
    val value = s ?: return -1          // return from the function if null
    return value.length                 // here `value` is smart-cast to non-null String
}
```

**Not-null assertion `!!`** — converts a nullable to non-null, throwing an NPE if it actually was null. This is the "trust me, it's not null" escape hatch. **Use it sparingly** — it reintroduces the very NPEs Kotlin protects you from. Prefer `?:` or `requireNotNull`/`checkNotNull`.

```kotlin
fun main() {
    val s: String? = "hi"
    println(s!!.length)     // 2 — but if s were null this LINE throws NPE
}
```

**Safe cast `as?`** — attempts a cast and yields `null` instead of throwing if the type doesn't match. Use it when a value *might* be of the target type.

```kotlin
fun main() {
    val obj: Any = "text"
    val asInt: Int? = obj as? Int       // null — "text" is not an Int (no exception)
    val asStr: String? = obj as? String // "text"
    println("$asInt / $asStr")          // null / text
}
```

### 3.3 Smart casts **[B]**

Once you've checked a nullable for null (or checked a type with `is`), the compiler **smart-casts** it to the non-null/narrower type within that scope. You don't repeat the check.

```kotlin
fun describe(x: Any?) {
    if (x == null) {
        println("nothing")
        return
    }
    // After the null check + return, x is smart-cast to non-null Any here.
    if (x is String) {
        // Inside this branch x is smart-cast to String — call String methods directly.
        println("string of length ${x.length}")
    } else {
        println("some ${x::class.simpleName}")
    }
}
```

> **⚡ Version note (K2):** The K2 compiler performs more smart-casts than the old frontend, including across some previously-unsupported control-flow patterns. Code that needed an explicit `!!` before may now compile cleanly. Smart-casts still don't apply to `var` properties that could change between the check and use (e.g. a mutable property of another object), or to `open` properties — use a local `val` capture in those cases.

### 3.4 `let` for null handling **[B/I]**

The `let` scope function (full treatment in §6) is the idiomatic way to run a block only when a value is non-null. `value?.let { ... }` executes the block only if `value` is non-null, binding it to `it` (or a name you choose) as a non-null type.

```kotlin
fun main() {
    val email: String? = "a@b.com"

    // Run the block ONLY if email is non-null. Inside, `it` is the non-null String.
    email?.let {
        println("Sending to $it")          // it: String (non-null)
    }

    // Combine with Elvis for an else branch:
    val length = email?.let { it.length } ?: 0
    println(length)                         // 7
}
```

### 3.5 Platform types from Java **[I]**

When you call Java code, the Kotlin compiler can't always know whether a returned reference is nullable (Java doesn't encode it in the type). Such a type is a **platform type**, written `String!` in error messages. Kotlin *relaxes* null checks on platform types: you can treat them as non-null, but if they actually are null you'll get an NPE at the point of use. This is the one place Kotlin trusts you.

To make Java interop safe, Kotlin honors nullability annotations on the Java side (`@Nullable`/`@NonNull` from JetBrains, JSR-305, Android, Jakarta, etc.). If the Java method is annotated `@Nullable String getName()`, Kotlin sees it as `String?` and enforces a null check.

```kotlin
// Suppose Java: public String getName() { ... }   // no annotation
// In Kotlin getName() returns String! (platform type).
fun useJava(person: JavaPerson) {
    val name = person.name        // type is String! — Kotlin won't force a null check
    println(name.length)          // compiles, but throws NPE at runtime if name is null

    // SAFER: opt into a nullable type yourself so the compiler protects you:
    val safe: String? = person.name
    println(safe?.length)         // now null-safe
}
```

**Best practice:** when wrapping Java APIs, immediately annotate the boundary by assigning to an explicit `String?` (or `String`) so the rest of your Kotlin code is null-checked normally.

---

## 4. Control Flow

### 4.1 `if` as statement and expression **[B]**

`if` works as a normal statement, but unlike Java it is also an **expression** — it returns the value of the chosen branch. The last expression in a branch block is its value. This is why Kotlin has no ternary operator.

```kotlin
fun main() {
    val x = 7

    // As a statement:
    if (x > 0) println("positive") else println("non-positive")

    // As an expression (returns a value) — replaces the ternary:
    val sign = if (x > 0) "pos" else if (x < 0) "neg" else "zero"
    println(sign)

    // Branches can be blocks; the LAST line is the block's value:
    val label = if (x % 2 == 0) {
        val note = "computed"           // local work
        "even ($note)"                  // this is the block's value
    } else {
        "odd"
    }
    println(label)                      // odd
}
```

### 4.2 `when` — the powerful switch **[B]**

`when` replaces Java's `switch` and is far more capable: it matches values, ranges, types, and arbitrary boolean conditions, and it is also an expression. There is no fall-through, so no `break` is needed.

```kotlin
fun classify(x: Any): String = when (x) {     // `when` with a subject, used as expression
    0 -> "zero"                                // exact value
    1, 2, 3 -> "small"                         // multiple values in one branch
    in 4..9 -> "single digit"                  // range check with `in`
    is String -> "a string of length ${x.length}"  // type check (smart-cast inside)
    !is Boolean -> "not a boolean"             // negated type check
    else -> "something else"                   // else is required when used as an expression
}

fun main() {
    println(classify(2))        // small
    println(classify(7))        // single digit
    println(classify("hi"))     // a string of length 2

    // `when` WITHOUT a subject acts like a chain of if/else if — clean for conditions:
    val score = 82
    val grade = when {
        score >= 90 -> "A"
        score >= 80 -> "B"
        score >= 70 -> "C"
        else -> "F"
    }
    println(grade)              // B
}
```

> **⚡ Version note:** Modern Kotlin supports `when` with a **guard** subject binding: `when (val r = compute()) { ... }` binds the result to `r` for use in the branches. Exhaustive `when` over `sealed` types and enums lets you omit `else` (see §8).

### 4.3 Loops: `for`, `while`, ranges **[B]**

Kotlin's `for` always iterates over something that provides an iterator — a range, a collection, a string, etc. There is no C-style `for (i = 0; i < n; i++)`; you iterate a range instead.

```kotlin
fun main() {
    // Ranges: `..` is inclusive on both ends.
    for (i in 1..5) print(i)          // 12345
    println()

    // `until` excludes the upper bound (great for indices 0 until size).
    for (i in 0 until 5) print(i)     // 01234
    println()

    // `..<` is the modern operator form of `until` (same meaning).
    for (i in 0..<5) print(i)         // 01234
    println()

    // downTo counts down; step changes the increment.
    for (i in 10 downTo 1 step 2) print("$i ")  // 10 8 6 4 2
    println()

    // Iterate a collection directly:
    val fruits = listOf("apple", "pear", "kiwi")
    for (f in fruits) print("$f ")    // apple pear kiwi
    println()

    // With index using withIndex():
    for ((index, f) in fruits.withIndex()) {
        println("$index: $f")
    }

    // while and do-while behave as expected:
    var n = 3
    while (n > 0) { print(n); n-- }   // 321
    println()
    do { print("once") } while (false)
}
```

### 4.4 Labels, `break`, `continue`, returns **[I]**

`break` and `continue` work on the innermost loop by default. To target an outer loop, attach a **label** (`name@`) and write `break@name`. Labels also let you return from a lambda to a specific point.

```kotlin
fun main() {
    // Labeled break: exit the OUTER loop from inside the inner loop.
    outer@ for (i in 1..3) {
        for (j in 1..3) {
            if (i * j > 4) break@outer    // jumps out of BOTH loops
            print("$i*$j ")
        }
    }
    println()   // 1*1 1*2 1*3 2*1 2*2

    // continue@label skips to the next iteration of the labeled loop.
    loop@ for (i in 1..3) {
        for (j in 1..3) {
            if (j == 2) continue@loop     // skip rest of inner, advance i
            print("$i,$j ")
        }
    }
    println()   // 1,1 2,1 3,1
}
```

---

## 5. Functions

### 5.1 Declaring functions **[B]**

Functions are declared with `fun`. Parameter types are required; the return type follows the parameter list. If a function returns nothing meaningful, its return type is `Unit` (like `void`), which can be omitted.

```kotlin
// name(params): ReturnType { body }
fun multiply(a: Int, b: Int): Int {
    return a * b
}

// Unit return (no value) — the `: Unit` is optional and usually omitted.
fun log(message: String) {
    println("[LOG] $message")
}

fun main() {
    println(multiply(6, 7))   // 42
    log("started")
}
```

### 5.2 Single-expression functions **[B]**

When a function just returns one expression, drop the braces and `return` and use `=`. The return type can be inferred. This is extremely common and idiomatic.

```kotlin
// Equivalent to: fun square(x: Int): Int { return x * x }
fun square(x: Int) = x * x            // return type Int inferred

fun fullName(first: String, last: String) = "$first $last"

fun main() {
    println(square(5))                // 25
    println(fullName("Ada", "Love"))  // Ada Love
}
```

### 5.3 Default & named arguments **[B]**

Parameters can have **default values**, so callers may omit them. Combined with **named arguments** (passing `name = value`), this eliminates the need for the telescoping/overload soup common in Java.

```kotlin
fun connect(
    host: String,
    port: Int = 5432,            // default
    useTls: Boolean = true,      // default
    timeoutMs: Int = 30_000      // default
): String = "Connecting to $host:$port tls=$useTls timeout=$timeoutMs"

fun main() {
    println(connect("db.local"))                      // uses all defaults
    println(connect("db.local", 5433))                // override port only
    // Named arguments: pass out of order and skip middle defaults clearly.
    println(connect("db.local", useTls = false))      // skip port, set useTls
    println(connect(host = "db.local", timeoutMs = 5_000))
}
```

### 5.4 `vararg` — variable arguments **[B]**

`vararg` lets a function accept any number of arguments of a type; inside, the parameter is an `Array`. Use the **spread operator** `*` to pass an existing array into a vararg.

```kotlin
fun sumAll(vararg numbers: Int): Int {
    var total = 0
    for (n in numbers) total += n     // numbers is an IntArray here
    return total
}

fun main() {
    println(sumAll(1, 2, 3))          // 6
    println(sumAll())                 // 0
    val arr = intArrayOf(4, 5, 6)
    println(sumAll(*arr))             // 15 — spread the array into varargs
}
```

### 5.5 Local functions **[I]**

You can declare a function *inside* another function. Local functions can access the enclosing function's variables (a closure), which is handy for breaking up logic without polluting the namespace or passing many parameters around.

```kotlin
fun processOrder(items: List<Int>, taxRate: Double): Double {
    // Local helper — only visible inside processOrder; closes over taxRate.
    fun withTax(amount: Double) = amount * (1 + taxRate)

    val subtotal = items.sum().toDouble()
    return withTax(subtotal)
}

fun main() {
    println(processOrder(listOf(100, 50), 0.1))   // 165.0
}
```

### 5.6 Extension functions & properties **[I]**

An **extension function** lets you add a method to an existing type — even one you don't own (like `String` or a Java class) — *without* subclassing or modifying it. You write the receiver type before the function name (`fun String.shout()`), and inside the function `this` refers to the receiver.

**How it works:** extensions are resolved *statically* at compile time. They don't actually modify the class or add to its vtable; the compiler turns `"hi".shout()` into a static call passing `"hi"` as a hidden first argument. This means extensions can't access private members and can't be overridden polymorphically — but they're perfectly safe and let you write fluent, discoverable APIs.

**When to use:** to add utility methods to types you can't edit; to keep related helpers next to where they're used; to build readable DSLs (§14). The Kotlin standard library is largely built from extensions (e.g. all the collection operations in §10).

```kotlin
// Add a method to String. `this` is the receiver (the String being called on).
fun String.shout(): String = this.uppercase() + "!"

// Extension with a parameter; works on any List<Int>.
fun List<Int>.sumOfSquares(): Int = this.sumOf { it * it }

// Extension PROPERTY — a computed property on an existing type.
// (No backing field allowed; must be computed via a getter.)
val String.firstWord: String
    get() = this.trim().substringBefore(" ")

fun main() {
    println("hello".shout())              // HELLO!
    println(listOf(1, 2, 3).sumOfSquares())  // 14
    println("hello world".firstWord)      // hello
}
```

> **Gotcha:** if a class already has a member function with the same signature, the *member* always wins over an extension — your extension is silently ignored for that call. Extensions are dispatched on the *static* (declared) type, not the runtime type.

### 5.7 Infix functions **[I]**

A function marked `infix` with a single parameter can be called without the dot and parentheses: `a to b` instead of `a.to(b)`. This reads like an operator and is used for DSL-ish constructs (the standard `to` that builds `Pair`s is infix).

```kotlin
// Define an infix extension that builds a labeled pair.
infix fun Int.repeats(text: String): String = text.repeat(this)

fun main() {
    println(3 repeats "ab")           // ababab
    val pair = "key" to "value"       // `to` is an infix stdlib function -> Pair
    println(pair)                     // (key, value)
}
```

### 5.8 Operator overloading **[I]**

Kotlin lets you give meaning to operators (`+`, `-`, `*`, `[]`, `in`, `==`, etc.) for your own types by defining functions with the `operator` modifier and a conventional name (`plus`, `minus`, `times`, `get`, `set`, `contains`, `compareTo`...). Use this for types where the operator has an *obvious* mathematical or container meaning (vectors, money, matrices) — not to be clever.

```kotlin
data class Vec(val x: Int, val y: Int) {
    // Enables: a + b
    operator fun plus(other: Vec) = Vec(x + other.x, y + other.y)
    // Enables: a * 3
    operator fun times(scalar: Int) = Vec(x * scalar, y * scalar)
    // Enables: v[0] / v[1]
    operator fun get(index: Int) = if (index == 0) x else y
}

fun main() {
    val a = Vec(1, 2)
    val b = Vec(3, 4)
    println(a + b)        // Vec(x=4, y=6)
    println(a * 3)        // Vec(x=3, y=6)
    println(a[1])         // 2
}
```

| Operator | Function name |
|---|---|
| `a + b` | `a.plus(b)` |
| `a - b` | `a.minus(b)` |
| `a * b` | `a.times(b)` |
| `a / b` | `a.div(b)` |
| `a % b` | `a.rem(b)` |
| `a[i]` | `a.get(i)` |
| `a[i] = v` | `a.set(i, v)` |
| `a in b` | `b.contains(a)` |
| `a..b` | `a.rangeTo(b)` |
| `+a` / `-a` | `a.unaryPlus()` / `a.unaryMinus()` |
| `a == b` | `a.equals(b)` |
| `a < b` etc. | `a.compareTo(b)` |
| `a()` | `a.invoke()` |

---

## 6. Lambdas, Higher-Order Functions & Scope Functions

### 6.1 Function types & lambdas **[I]**

A **lambda** is an anonymous function written as a literal — a block of code you can pass around as a value. A **higher-order function** is a function that takes or returns another function. Together they enable the concise functional style that pervades Kotlin's standard library.

A **function type** is written `(ParamTypes) -> ReturnType`. For example `(Int) -> Boolean` is "a function taking an `Int` and returning a `Boolean`."

```kotlin
fun main() {
    // A lambda literal lives in braces: { params -> body }. The last line is the result.
    val square: (Int) -> Int = { x -> x * x }
    println(square(5))                 // 25

    // If the lambda has exactly ONE parameter, you can omit it and use `it`:
    val isEven: (Int) -> Boolean = { it % 2 == 0 }
    println(isEven(4))                 // true

    // Multiple parameters:
    val add: (Int, Int) -> Int = { a, b -> a + b }
    println(add(2, 3))                 // 5
}
```

### 6.2 Higher-order functions & trailing lambdas **[I]**

When a function's *last* parameter is a lambda, you can move it outside the parentheses — the **trailing lambda** convention. If the lambda is the *only* argument, you can drop the parentheses entirely. This is why Kotlin code reads like it has built-in control structures.

```kotlin
// Higher-order function: takes a function as a parameter.
fun repeatAction(times: Int, action: (Int) -> Unit) {
    for (i in 0 until times) action(i)
}

fun main() {
    // Trailing lambda OUTSIDE the parens (because action is the last param):
    repeatAction(3) { i ->
        println("iteration $i")
    }

    // A function returning a function:
    fun multiplier(factor: Int): (Int) -> Int = { it * factor }
    val triple = multiplier(3)
    println(triple(10))               // 30

    // Pass a named function as a value with the :: reference operator:
    fun shout(s: String) = s.uppercase()
    val fn: (String) -> String = ::shout
    println(fn("hi"))                 // HI
}
```

### 6.3 The scope functions: `let`, `run`, `with`, `apply`, `also` **[I]**

The standard library provides five **scope functions** that execute a block in the *context* of an object. They differ in two ways: (1) how the object is referenced inside the block (`it` vs `this`), and (2) what the block returns (the lambda result vs the object itself). Choosing the right one makes code expressive; choosing wrong (or overusing them) makes it cryptic. Here's the decision logic.

| Function | Object referenced as | Returns | Typical use |
|---|---|---|---|
| `let` | `it` (argument) | lambda result | Null-safe ops on `?.let`; transform a value into another |
| `run` | `this` (receiver) | lambda result | Compute a result from an object; group statements |
| `with` | `this` (receiver) | lambda result | Call many members on one object (not an extension) |
| `apply` | `this` (receiver) | the object itself | Configure/build an object (set properties), then return it |
| `also` | `it` (argument) | the object itself | Side-effects like logging while keeping the value in a chain |

Rules of thumb:
- Need the **transformed result**? Use `let` or `run`.
- Need the **same object back** (chaining/config)? Use `apply` or `also`.
- Referring to the object **as `this`** reads cleaner when calling its members (`run`, `with`, `apply`); referring to it **as `it`** reads cleaner for a plain value or to avoid shadowing (`let`, `also`).

```kotlin
class Server {
    var host: String = ""
    var port: Int = 0
    fun start() = println("Starting $host:$port")
}

fun main() {
    // apply: configure an object, return the SAME object. Great for builders.
    val server = Server().apply {
        host = "localhost"   // `this` is the Server; access members directly
        port = 8080
    }
    server.start()           // Starting localhost:8080

    // also: perform a side effect (logging) while passing the value through.
    val nums = listOf(1, 2, 3)
        .also { println("about to map: $it") }   // `it` is the list; returns the list
        .map { it * 2 }
    println(nums)            // [2, 4, 6]

    // let: transform a value; commonly with ?. for null safety.
    val name: String? = "ada"
    val upper = name?.let { it.uppercase() } ?: "UNKNOWN"
    println(upper)           // ADA

    // run: compute a result using `this`; good for grouping setup + result.
    val area = server.run {
        // `this` is the Server; here we just compute something:
        host.length * port
    }
    println(area)            // 9 * 8080 ... ("localhost".length=9) = 72720

    // with: like run but called as with(obj) { ... } — for batching member calls.
    val summary = with(server) {
        "$host on $port"     // `this` is the Server
    }
    println(summary)         // localhost on 8080
}
```

> **Guidance:** Don't nest scope functions deeply — it gets unreadable fast. Use `apply` for object configuration (especially builders), `let` for null-guarding and transformation, `also` for logging/side-effects in a chain. When in doubt, a plain local `val` is clearer than a scope function.

---

## 7. Object-Oriented Kotlin

### 7.1 Classes & primary constructors **[B]**

A class is declared with `class`. The **primary constructor** is part of the class header. Parameters declared with `val`/`var` in the primary constructor automatically become properties — a huge boilerplate saver versus Java.

```kotlin
// Primary constructor in the header. `val`/`var` make these into properties.
class Person(val name: String, var age: Int) {
    // A regular method:
    fun birthday() {
        age += 1
    }
}

fun main() {
    val p = Person("Ada", 36)    // no `new` keyword in Kotlin
    println(p.name)              // Ada (read-only property)
    p.birthday()
    println(p.age)               // 37 (mutable property)
}
```

### 7.2 `init` blocks & secondary constructors **[I]**

The primary constructor has no body, so initialization logic goes in **`init` blocks**, which run in order along with property initializers. **Secondary constructors** (`constructor(...)`) are alternative entry points; they must delegate to the primary constructor with `: this(...)`.

```kotlin
class Rectangle(val width: Int, val height: Int) {
    val area: Int

    init {
        // Runs when an instance is created; can validate and compute.
        require(width > 0 && height > 0) { "dimensions must be positive" }
        area = width * height
        println("Created ${width}x${height}")
    }

    // Secondary constructor for a square; delegates to the primary one.
    constructor(side: Int) : this(side, side) {
        println("Square constructor")
    }
}

fun main() {
    val r = Rectangle(3, 4)   // Created 3x4
    println(r.area)           // 12
    val sq = Rectangle(5)     // Created 5x5 / Square constructor
    println(sq.area)          // 25
}
```

### 7.3 Properties: getters/setters, backing fields, `lateinit`, `lazy` **[I]**

A Kotlin **property** is a field plus its accessors. You can customize the getter/setter. Inside a custom accessor, `field` refers to the **backing field** — the actual storage — which exists only when you use it.

```kotlin
class Temperature {
    // Custom setter validates input; `field` is the backing storage.
    var celsius: Double = 0.0
        set(value) {
            require(value >= -273.15) { "below absolute zero" }
            field = value                 // assign to the backing field (not `celsius =`!)
        }

    // Computed property: no backing field, value derived each access.
    val fahrenheit: Double
        get() = celsius * 9 / 5 + 32
}

fun main() {
    val t = Temperature()
    t.celsius = 25.0
    println(t.fahrenheit)     // 77.0
}
```

**`lateinit`** declares a non-null `var` that you promise to initialize *before first use* — useful for dependency injection or setup-in-a-callback where you can't initialize at construction. Accessing it before initialization throws `UninitializedPropertyAccessException`. It only works for non-primitive `var`s.

**`lazy`** computes a `val` once, on first access, then caches it (thread-safe by default). Use it for expensive initialization you might not need.

```kotlin
class Service {
    lateinit var config: String          // initialized later, not at construction

    // `by lazy { ... }` — computed on first access, then cached.
    val connection: String by lazy {
        println("opening connection...")  // runs only once, only if accessed
        "CONNECTED"
    }

    fun setup() { config = "loaded" }
}

fun main() {
    val s = Service()
    s.setup()
    println(s.config)         // loaded
    println(s.connection)     // opening connection... / CONNECTED
    println(s.connection)     // CONNECTED  (cached; no re-open)
    println(s::config.isInitialized)  // true (check lateinit init state)
}
```

### 7.4 Visibility modifiers **[I]**

| Modifier | Meaning |
|---|---|
| `public` (default) | Visible everywhere |
| `private` | Visible only inside the declaring class/file |
| `protected` | Visible in the class and its subclasses (not top-level) |
| `internal` | Visible within the same compilation **module** (Gradle module) |

`internal` has no Java equivalent and is great for library code: public to your whole module but hidden from consumers.

### 7.5 Inheritance: `open`, `override`, `abstract` **[I]**

Kotlin classes are **`final` by default** — you cannot subclass them unless they're marked `open`. Similarly, methods/properties must be `open` to be overridden. This is deliberate: unintended inheritance is a common source of fragile designs, so Kotlin makes extensibility an explicit decision. Overriding members must use the `override` keyword.

```kotlin
// `open` allows subclassing.
open class Animal(val name: String) {
    open fun speak(): String = "..."       // open: can be overridden
    fun describe() = "$name says ${speak()}"  // not open: cannot be overridden
}

class Dog(name: String) : Animal(name) {     // call the superclass constructor
    override fun speak(): String = "Woof"     // override is mandatory
}

// Abstract class: cannot be instantiated; may have abstract (unimplemented) members.
abstract class Shape {
    abstract fun area(): Double               // abstract members are implicitly open
    fun printArea() = println("area = ${area()}")
}

class Circle(val r: Double) : Shape() {
    override fun area() = Math.PI * r * r
}

fun main() {
    println(Dog("Rex").describe())   // Rex says Woof
    Circle(2.0).printArea()          // area = 12.566...
}
```

### 7.6 Interfaces **[I]**

Interfaces declare a contract and may contain abstract members *and* default method implementations. A class can implement many interfaces. Interfaces can't hold state (no backing fields), but can declare abstract properties.

```kotlin
interface Greeter {
    val greeting: String                 // abstract property (no backing field)
    fun greet(name: String): String = "$greeting, $name!"   // default implementation
}

interface Logger {
    fun log(msg: String) = println("[LOG] $msg")
}

// Implement multiple interfaces.
class FriendlyService : Greeter, Logger {
    override val greeting = "Hello"      // provide the property
}

fun main() {
    val s = FriendlyService()
    println(s.greet("Ada"))   // Hello, Ada!
    s.log("done")             // [LOG] done
}
```

### 7.7 `object` — singletons & anonymous objects **[I]**

The `object` keyword creates a **singleton**: a single instance, created lazily and thread-safely. Use it for stateless utilities, registries, or the single instance of something. `object` can also create **anonymous objects** (like Java anonymous classes) inline.

```kotlin
// Object declaration = singleton. Accessed by its name.
object AppConfig {
    var debug = false
    val version = "1.0.0"
    fun describe() = "v$version debug=$debug"
}

fun main() {
    AppConfig.debug = true
    println(AppConfig.describe())    // v1.0.0 debug=true

    // Anonymous object implementing an interface on the spot:
    val handler = object : Runnable {
        override fun run() = println("running")
    }
    handler.run()
}
```

### 7.8 `companion object` — class-level members **[I]**

Kotlin has no `static` keyword. Instead, members that belong to the class itself (factories, constants) live in a **companion object** — a singleton tied to the class. You call them as `ClassName.member`.

```kotlin
class User private constructor(val id: Int) {  // private constructor
    companion object {
        const val MAX_ID = 9999                 // compile-time constant
        private var counter = 0

        // Factory method — common pattern since constructors can be private.
        fun create(): User {
            counter += 1
            return User(counter)
        }
    }
}

fun main() {
    println(User.MAX_ID)         // 9999 — accessed like a static
    val u = User.create()        // factory
    println(u.id)                // 1
}
```

> Members of a companion can be exposed to Java as real statics with `@JvmStatic` (see §15). `const val` is a true compile-time constant (only for primitives/String at top level or in objects).

### 7.9 Nested vs inner classes **[I]**

A class declared inside another is **nested** by default — it has *no* reference to the outer instance (like a Java `static` nested class). Mark it `inner` to give it access to the outer instance via `this@Outer`.

```kotlin
class Outer(val value: Int) {
    // Nested: NO access to Outer's instance members.
    class Nested {
        fun describe() = "I am nested"
    }
    // Inner: HAS a reference to the enclosing Outer instance.
    inner class Inner {
        fun describe() = "outer value is ${this@Outer.value}"
    }
}

fun main() {
    println(Outer.Nested().describe())      // I am nested (no Outer needed)
    println(Outer(42).Inner().describe())   // outer value is 42
}
```

---

## 8. Special Classes: data, enum, sealed, value, annotation

### 8.1 `data class` — value holders **[I]**

A **data class** is meant to hold data. From the properties in its primary constructor the compiler generates `equals()`, `hashCode()`, `toString()`, `copy()`, and `componentN()` functions (for destructuring). This eliminates the mountain of boilerplate Java requires for value objects.

```kotlin
data class User(val id: Int, val name: String, val email: String)

fun main() {
    val u = User(1, "Ada", "a@b.com")

    // toString() is generated and readable:
    println(u)                       // User(id=1, name=Ada, email=a@b.com)

    // equals()/hashCode() compare by VALUE, not identity:
    println(u == User(1, "Ada", "a@b.com"))   // true

    // copy() makes a modified duplicate (immutability-friendly updates):
    val renamed = u.copy(name = "Grace")
    println(renamed)                 // User(id=1, name=Grace, email=a@b.com)

    // Destructuring uses generated component functions:
    val (id, name, email) = u
    println("$id / $name / $email")  // 1 / Ada / a@b.com
}
```

> **Notes:** only properties in the *primary constructor* count toward the generated functions. A data class can't be `abstract`/`open`/`sealed`. Prefer `val` properties so the value semantics hold.

### 8.2 `enum class` — fixed sets of constants **[I]**

An enum represents a fixed set of named values. Kotlin enums can hold properties, implement interfaces, and define methods (including per-constant overrides).

```kotlin
enum class Planet(val massKg: Double, val radiusM: Double) {
    EARTH(5.976e24, 6.378e6),
    MARS(6.421e23, 3.397e6);          // semicolon needed before members

    // A method shared by all constants:
    fun surfaceGravity() = 6.67300e-11 * massKg / (radiusM * radiusM)
}

fun main() {
    val p = Planet.EARTH
    println(p.name)              // EARTH  (built-in)
    println(p.ordinal)          // 0       (built-in, position)
    println("%.2f".format(p.surfaceGravity()))  // 9.80
    println(Planet.entries)      // [EARTH, MARS]  (modern replacement for values())
    println(Planet.valueOf("MARS"))  // MARS (lookup by name)
}
```

> **⚡ Version note:** `Enum.entries` (a cached `List`) replaces `values()` (which allocated a new array each call). Prefer `entries`.

### 8.3 `sealed` classes/interfaces & exhaustive `when` **[I/A]**

A **sealed** type restricts its subclasses to a known, closed set declared in the same module/package. This is perfect for modeling a fixed number of variants (a result that's `Success` or `Error`, a UI state, an AST node). Because the compiler knows *all* the subtypes, a `when` over a sealed type is **exhaustive** — if you handle every case, you don't need an `else`, and if you later add a variant, the `when` becomes a compile error until you handle it. This turns "did I handle every case?" from a runtime worry into a compile-time guarantee.

Sealed is the idiomatic Kotlin alternative to throwing exceptions for expected outcomes and to nullable returns when you need to carry error detail.

```kotlin
// A sealed hierarchy modeling the result of a network call.
sealed interface ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>
    data class Failure(val code: Int, val message: String) : ApiResult<Nothing>
    data object Loading : ApiResult<Nothing>     // data object: singleton variant
}

fun render(result: ApiResult<String>): String = when (result) {
    // No `else` needed — the compiler knows these are ALL the cases.
    is ApiResult.Success -> "Got: ${result.data}"      // smart-cast to Success
    is ApiResult.Failure -> "Error ${result.code}: ${result.message}"
    ApiResult.Loading -> "Loading..."
}

fun main() {
    println(render(ApiResult.Success("hello")))
    println(render(ApiResult.Failure(404, "Not Found")))
    println(render(ApiResult.Loading))
}
```

### 8.4 `value` / `inline` classes — zero-cost wrappers **[A]**

A **value class** (declared `@JvmInline value class`) wraps a single value to give it a distinct type *without* the runtime cost of an extra object: where possible, the compiler inlines the underlying value and erases the wrapper. Use it to make primitive-obsessed APIs type-safe — e.g. so you can't accidentally pass a `UserId` where an `OrderId` is expected, even though both are `Int` underneath.

```kotlin
@JvmInline
value class UserId(val raw: Int)       // wraps an Int; no allocation in the common case

@JvmInline
value class Email(val value: String) {
    init { require("@" in value) { "invalid email" } }  // can validate
    fun domain() = value.substringAfter("@")
}

fun fetchUser(id: UserId) = "user-${id.raw}"

fun main() {
    val id = UserId(42)
    println(fetchUser(id))       // user-42
    // fetchUser(42)             // COMPILE ERROR: Int is not a UserId — type safety!
    println(Email("a@b.com").domain())  // b.com
}
```

### 8.5 `annotation` classes **[A]**

Annotations attach metadata to declarations, consumed by the compiler, frameworks, or reflection. You declare them with `annotation class` and configure where they apply with meta-annotations (`@Target`, `@Retention`).

```kotlin
@Target(AnnotationTarget.FUNCTION, AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)   // keep at runtime for reflection
annotation class Audited(val level: String = "INFO")

@Audited(level = "HIGH")
fun transferFunds() { /* ... */ }

fun main() {
    // Frameworks (Spring, serialization, JUnit) read annotations via reflection.
    val ann = ::transferFunds.annotations.filterIsInstance<Audited>().firstOrNull()
    println(ann?.level)          // HIGH
}
```

---

## 9. Generics & Variance

### 9.1 Generic functions & classes **[I]**

Generics let you write code parameterized over types, giving you reuse *with* type safety. A type parameter (conventionally `T`, `E`, `K`, `V`) stands for a type chosen by the caller.

```kotlin
// Generic function: works for any T, returns the same T.
fun <T> firstOrNull(list: List<T>): T? = if (list.isEmpty()) null else list[0]

// Generic class: a simple typed box.
class Box<T>(val content: T) {
    fun <R> map(transform: (T) -> R): Box<R> = Box(transform(content))
}

// Type bounds (constraints): T must be Comparable.
fun <T : Comparable<T>> maxOf2(a: T, b: T): T = if (a >= b) a else b

fun main() {
    println(firstOrNull(listOf("a", "b")))   // a
    val box = Box(5).map { it.toString() }    // Box<String>
    println(box.content)                      // "5"
    println(maxOf2(3, 9))                     // 9
}
```

### 9.2 Variance: covariance, contravariance, `in`/`out` **[A]**

Variance answers: if `Cat` is a subtype of `Animal`, is `Producer<Cat>` a subtype of `Producer<Animal>`? By default Kotlin generics are **invariant** — `Box<Cat>` and `Box<Animal>` are unrelated, because a `Box<Animal>` lets you *put* a `Dog` in, which would corrupt a `Box<Cat>`. Variance modifiers relax this safely.

- **`out` (covariance):** the type parameter is only ever *produced* (returned), never consumed. Then `Producer<Cat>` *is* a `Producer<Animal>` — reading a `Cat` where you expected an `Animal` is safe. Think "output only."
- **`in` (contravariance):** the type parameter is only ever *consumed* (passed in), never produced. Then `Consumer<Animal>` *is* a `Consumer<Cat>` — something that accepts any `Animal` can certainly accept a `Cat`. Think "input only."

Mnemonic: **PECS** — Producer `out`, Consumer `in`. You declare variance at the **declaration site** (on the class), which is cleaner than Java's per-use wildcards.

```kotlin
open class Animal(val name: String)
class Cat : Animal("cat")

// `out T`: T appears only in OUT positions (return types). Covariant.
interface Producer<out T> {
    fun produce(): T
}

// `in T`: T appears only in IN positions (parameters). Contravariant.
interface Consumer<in T> {
    fun consume(item: T)
}

fun main() {
    val catProducer: Producer<Cat> = object : Producer<Cat> {
        override fun produce() = Cat()
    }
    // Covariance: a Producer<Cat> can be used as a Producer<Animal>.
    val animalProducer: Producer<Animal> = catProducer
    println(animalProducer.produce().name)     // cat

    val animalConsumer: Consumer<Animal> = object : Consumer<Animal> {
        override fun consume(item: Animal) = println("consumed ${item.name}")
    }
    // Contravariance: a Consumer<Animal> can be used as a Consumer<Cat>.
    val catConsumer: Consumer<Cat> = animalConsumer
    catConsumer.consume(Cat())                  // consumed cat
}
```

You can also specify variance at the **use site** (Java-style projection): `fun copy(from: Array<out Any>, to: Array<Any>)`.

### 9.3 Star projection `*` **[A]**

When you don't know or care about the type argument, use the **star projection** `List<*>`. You can read elements as the upper bound (`Any?`) but can't safely write to it.

```kotlin
fun printSize(c: Collection<*>) {
    // We don't know the element type; * means "some unknown type".
    println("size = ${c.size}")
    // c.add(...) would be unsafe and is disallowed.
}

fun main() {
    printSize(listOf(1, 2, 3))
    printSize(setOf("a", "b"))
}
```

### 9.4 Reified type parameters **[A]**

Because the JVM **erases** generic types at runtime, you normally can't write `if (x is T)` or `T::class` inside a generic function. Marking the function `inline` and the parameter `reified` makes the type available at runtime — the compiler inlines the function body and substitutes the actual type. Use it for type checks, reflection, and parsing.

```kotlin
// inline + reified lets us use T as a real type at runtime.
inline fun <reified T> Any?.isA(): Boolean = this is T

inline fun <reified T> List<*>.filterByType(): List<T> =
    this.filterIsInstance<T>()        // filterIsInstance itself uses reified

fun main() {
    println("hi".isA<String>())               // true
    println(42.isA<String>())                 // false
    val mixed = listOf(1, "a", 2, "b", 3)
    println(mixed.filterByType<Int>())        // [1, 2, 3]
}
```

---

## 10. Collections & Sequences

### 10.1 Read-only vs mutable collections — and why **[I]**

Kotlin distinguishes between **read-only** interfaces (`List`, `Set`, `Map`) and **mutable** ones (`MutableList`, `MutableSet`, `MutableMap`). A `List` has no `add`/`remove`; only its mutable counterpart does. This is a *compile-time* contract that communicates intent: a function taking a `List` promises not to modify it, and you can pass collections around knowing who can change them.

Note it's a read-only *view*, not a deep immutability guarantee: a `List` can be backed by a `MutableList` someone else still holds. But for everyday code the distinction prevents accidental mutation and makes APIs honest. Prefer read-only types in signatures and `val` for collection references; reach for mutable types only where you genuinely build up or modify a collection.

```kotlin
fun main() {
    // Read-only collections (cannot add/remove):
    val nums: List<Int> = listOf(3, 1, 2)
    val unique: Set<Int> = setOf(1, 1, 2, 3)        // {1, 2, 3}
    val ages: Map<String, Int> = mapOf("Ada" to 36, "Bob" to 40)

    println(nums[0])              // 3 (indexed access)
    println(nums.size)            // 3
    println(unique)               // [1, 2, 3]
    println(ages["Ada"])          // 36

    // Mutable collections (can add/remove):
    val mutable = mutableListOf(1, 2)
    mutable.add(3)                // [1, 2, 3]
    mutable.removeAt(0)           // [2, 3]
    val map = mutableMapOf<String, Int>()
    map["x"] = 10                 // put
    map.getOrPut("y") { 20 }      // insert if absent
    println(map)                  // {x=10, y=20}
}
```

### 10.2 The functional operations **[I]**

The standard library provides a rich set of extension functions for transforming collections declaratively. These avoid manual loops and read like a pipeline. Most return a *new* collection (eager); see §10.3 for the lazy alternative.

```kotlin
fun main() {
    val nums = listOf(1, 2, 3, 4, 5, 6)

    // map: transform each element.
    println(nums.map { it * it })                 // [1, 4, 9, 16, 25, 36]

    // filter: keep elements matching a predicate.
    println(nums.filter { it % 2 == 0 })          // [2, 4, 6]

    // reduce: combine elements left-to-right (no initial value; errors if empty).
    println(nums.reduce { acc, x -> acc + x })    // 21

    // fold: like reduce but WITH an initial accumulator (and a possibly different type).
    println(nums.fold(100) { acc, x -> acc + x }) // 121

    // sumOf / count / average / max:
    println(nums.sumOf { it })                    // 21
    println(nums.count { it > 3 })                // 3
    println(nums.maxOrNull())                     // 6

    // groupBy: build a Map from a key selector.
    val words = listOf("apple", "banana", "avocado", "cherry")
    println(words.groupBy { it.first() })
    // {a=[apple, avocado], b=[banana], c=[cherry]}

    // associate / associateWith / associateBy: build maps from elements.
    println(words.associateWith { it.length })    // {apple=5, banana=6, ...}
    println(words.associateBy { it.first() })     // last-wins on key collision

    // flatMap: map each element to a collection, then flatten.
    println(listOf("a,b", "c,d").flatMap { it.split(",") })  // [a, b, c, d]

    // partition: split into (matching, not-matching) by a predicate.
    val (evens, odds) = nums.partition { it % 2 == 0 }
    println("$evens / $odds")                     // [2, 4, 6] / [1, 3, 5]

    // zip: pair up two lists element-wise.
    println(listOf("a", "b").zip(listOf(1, 2)))   // [(a, 1), (b, 2)]

    // sortedBy / sortedByDescending / sorted:
    println(words.sortedBy { it.length })         // shortest first
    println(nums.sortedByDescending { it })       // [6, 5, 4, 3, 2, 1]

    // take / drop / chunked / windowed:
    println(nums.take(2))                         // [1, 2]
    println(nums.drop(4))                         // [5, 6]
    println(nums.chunked(2))                      // [[1, 2], [3, 4], [5, 6]]

    // any / all / none / find / first:
    println(nums.any { it > 5 })                  // true
    println(nums.all { it > 0 })                  // true
    println(nums.find { it > 3 })                 // 4 (first match or null)

    // distinct / reversed / joinToString:
    println(listOf(1, 1, 2, 3).distinct())        // [1, 2, 3]
    println(nums.joinToString(", ", "[", "]"))    // [1, 2, 3, 4, 5, 6]
}
```

### 10.3 Sequences — lazy evaluation **[I/A]**

Collection operations like `map`/`filter` are **eager**: each step builds a whole intermediate list. For a long chain over a large dataset that's wasteful. A **`Sequence`** evaluates lazily and element-by-element: each element flows through the whole pipeline before the next starts, and short-circuiting operators (`first`, `take`, `find`) stop early. Use sequences when you have a large collection *and* a multi-step chain, or a potentially infinite stream. For small collections, eager is simpler and often faster (no lambda-per-element overhead).

```kotlin
fun main() {
    val nums = (1..1_000_000)

    // EAGER (List): builds a million-element intermediate list at each step.
    // val r = nums.toList().map { it * 2 }.filter { it % 3 == 0 }.first()

    // LAZY (Sequence): processes one element at a time, stops at the first match.
    val firstResult = nums.asSequence()
        .map { it * 2 }                  // not evaluated yet
        .filter { it % 3 == 0 }          // not evaluated yet
        .first()                         // pulls just enough to find the first
    println(firstResult)                 // 6

    // Infinite sequences via generateSequence — only safe because it's lazy:
    val powers = generateSequence(1) { it * 2 }    // 1, 2, 4, 8, ...
    println(powers.take(5).toList())     // [1, 2, 4, 8, 16]
}
```

### 10.4 Arrays **[I]**

`Array<T>` is a fixed-size, mutable, indexable collection backed by a JVM array. For primitives, use the specialized `IntArray`, `DoubleArray`, etc., which avoid boxing. Prefer `List` for most code; reach for arrays for performance-critical numeric work or when interoperating with Java APIs that take arrays.

```kotlin
fun main() {
    val arr = arrayOf(1, 2, 3)               // Array<Int>
    arr[0] = 10                              // mutable elements; fixed size
    println(arr.joinToString())              // 10, 2, 3

    val zeros = IntArray(5)                  // [0, 0, 0, 0, 0] — primitive, no boxing
    val squares = IntArray(5) { it * it }    // initializer lambda: [0, 1, 4, 9, 16]
    println(squares.joinToString())

    // Arrays share most functional operations with collections:
    println(arr.map { it + 1 })              // [11, 3, 4]
}
```

---

## 11. Coroutines & Flow

### 11.1 Why coroutines: the suspend model **[A]**

Threads are expensive: each one consumes significant memory and the OS schedules them. Blocking a thread on I/O (a network call, disk read) wastes it while it waits. **Coroutines** are lightweight: you can run hundreds of thousands of them on a small thread pool because a coroutine that's *waiting* simply **suspends** — it releases the underlying thread for other work and resumes later, exactly where it left off, when the awaited result is ready.

The key building block is a **`suspend` function**: a function that can pause and resume without blocking a thread. You can only call a `suspend` function from another `suspend` function or from a coroutine builder. The compiler transforms suspending code into a state machine under the hood, but you write it as if it were ordinary sequential code — no callbacks, no `.then()` chains. This is the great advantage: asynchronous code that reads top-to-bottom.

```kotlin
import kotlinx.coroutines.*

// `suspend` marks a function that may pause without blocking a thread.
suspend fun fetchUser(): String {
    delay(1000)              // delay is a SUSPENDING sleep — frees the thread, no block
    return "Ada"
}

fun main() = runBlocking {    // runBlocking bridges blocking and suspending worlds
    println("fetching...")
    val user = fetchUser()    // looks sequential; suspends here without blocking a thread
    println("got $user")
}
```

### 11.2 Structured concurrency **[A]**

**Structured concurrency** is the principle that every coroutine runs inside a **`CoroutineScope`** and a parent scope does not complete until all its children have completed. This makes concurrency safe and predictable: no orphaned background work, cancellation propagates from parent to children automatically, and an unhandled failure in a child cancels its siblings and surfaces to the parent. You never "fire and forget" into the void.

`coroutineScope { }` creates a scope that suspends until all coroutines launched inside it finish. Builders (`launch`, `async`) attach to the surrounding scope.

```kotlin
import kotlinx.coroutines.*

suspend fun loadDashboard() = coroutineScope {   // scope: waits for all children below
    // Both launches are children of this scope; the scope won't return until both finish.
    launch { delay(500); println("widget A loaded") }
    launch { delay(300); println("widget B loaded") }
    println("dashboard: launched children")
}   // <- suspends here until BOTH children complete

fun main() = runBlocking {
    loadDashboard()
    println("dashboard fully loaded")
}
```

### 11.3 `launch`, `async`, `runBlocking` **[A]**

- **`launch`** starts a coroutine that does work but returns no result; it returns a `Job` you can cancel or `join()`. Use it for fire-and-forget-*within-a-scope* side effects.
- **`async`** starts a coroutine that computes a result; it returns a `Deferred<T>` whose value you get with `await()`. Use it to run things in parallel and collect results.
- **`runBlocking`** blocks the current thread until its coroutine completes; it's the bridge from regular code into the coroutine world (use in `main` and tests, not in production async code).

```kotlin
import kotlinx.coroutines.*
import kotlin.system.measureTimeMillis

suspend fun fetchPrice(id: Int): Int { delay(500); return id * 100 }

fun main() = runBlocking {
    // SEQUENTIAL: ~1000ms total (one after another).
    val seq = measureTimeMillis {
        val a = fetchPrice(1)
        val b = fetchPrice(2)
        println("sum=${a + b}")
    }
    println("sequential: ${seq}ms")

    // PARALLEL with async/await: ~500ms (both run concurrently).
    val par = measureTimeMillis {
        val a = async { fetchPrice(1) }    // starts immediately, returns Deferred
        val b = async { fetchPrice(2) }
        println("sum=${a.await() + b.await()}")  // await suspends for the results
    }
    println("parallel: ${par}ms")

    // launch returns a Job (no result):
    val job = launch { delay(100); println("background done") }
    job.join()                              // wait for it
}
```

### 11.4 Dispatchers & context **[A]**

A coroutine runs on a **`CoroutineDispatcher`**, which decides *which thread(s)* it uses. The dispatcher is part of the `CoroutineContext`. Pick the dispatcher to match the work:

| Dispatcher | Use for |
|---|---|
| `Dispatchers.Default` | CPU-intensive work (parsing, computation); pool sized to CPU cores |
| `Dispatchers.IO` | Blocking I/O (network, disk, JDBC); large elastic pool |
| `Dispatchers.Main` | UI thread (Android/Compose/Swing); single thread |
| `Dispatchers.Unconfined` | Advanced/testing; not for general use |

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // launch on a specific dispatcher:
    launch(Dispatchers.Default) {
        // heavy computation here runs on a background thread
        println("computing on ${Thread.currentThread().name}")
    }.join()
}
```

### 11.5 `withContext` — switching dispatchers **[A]**

`withContext(dispatcher) { }` runs a block on a different dispatcher and *suspends* until it returns the block's result. This is the idiomatic way to move blocking work off the main thread and come back. Unlike `async`+`await`, it's for sequential "do this part elsewhere" steps.

```kotlin
import kotlinx.coroutines.*

suspend fun loadFromDisk(): String = withContext(Dispatchers.IO) {
    // Pretend this is a blocking file read; it runs on the IO pool.
    delay(200)
    "file contents"
}

fun main() = runBlocking {
    val data = loadFromDisk()    // suspends until the IO block returns
    println(data)
}
```

### 11.6 Cancellation **[A]**

Coroutines are **cooperatively cancellable**: cancelling a `Job` sets a flag, and suspending functions in `kotlinx.coroutines` check it and throw `CancellationException` at suspension points. Long CPU loops must check `isActive` or call `ensureActive()`/`yield()` to be cancellable. Always clean up resources in `finally` (which still runs on cancellation).

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch {
        try {
            repeat(1000) { i ->
                println("working $i")
                delay(100)            // suspension point: cancellation is checked here
            }
        } finally {
            // Runs even when cancelled — clean up here.
            println("cleaning up")
        }
    }
    delay(350)
    job.cancelAndJoin()               // request cancellation and wait for it to finish
    println("cancelled")

    // withTimeout cancels automatically after a deadline:
    val result = withTimeoutOrNull(200) {
        delay(500); "done"            // takes too long
    }
    println(result)                   // null (timed out)
}
```

### 11.7 Exception handling **[A]**

In structured concurrency, an uncaught exception in a `launch` child cancels the scope and propagates to the parent. For `async`, the exception is deferred and re-thrown at `await()`. Use a `try/catch` around `await()`, a `CoroutineExceptionHandler` for top-level `launch` coroutines, or `supervisorScope` when you want one child's failure *not* to cancel its siblings.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // CoroutineExceptionHandler catches uncaught exceptions from launch coroutines.
    val handler = CoroutineExceptionHandler { _, e ->
        println("caught: ${e.message}")
    }

    // supervisorScope: a failing child does NOT cancel its siblings.
    supervisorScope {
        launch(handler) { throw RuntimeException("child A failed") }
        launch { delay(100); println("child B still ran") }
    }

    // For async, catch at await:
    val deferred = async { throw IllegalStateException("boom") }
    try {
        deferred.await()
    } catch (e: Exception) {
        println("await caught: ${e.message}")
    }
}
```

### 11.8 `Flow` — cold asynchronous streams **[A]**

A **`Flow<T>`** is the coroutine equivalent of a sequence for *asynchronous* streams of multiple values over time. It is **cold**: nothing runs until a terminal operator like `collect` subscribes, and each collector triggers a fresh execution. Flows are built with the `flow { emit(...) }` builder and transformed with operators (`map`, `filter`, `take`, `onEach`, ...) that are themselves suspend-aware. Use `Flow` for streams: server-sent events, sensor readings, paged results, database change feeds.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// A cold flow: the block runs anew each time someone collects.
fun temperatures(): Flow<Int> = flow {
    for (t in listOf(20, 21, 19, 22)) {
        delay(100)             // can suspend between emissions
        emit(t)                // push a value to the collector
    }
}

fun main() = runBlocking {
    temperatures()
        .map { it * 9 / 5 + 32 }      // operators transform the stream
        .filter { it > 68 }
        .collect { println("$it F") } // terminal operator: starts the flow
}
```

### 11.9 `StateFlow` & `SharedFlow` — hot streams **[A]**

`flow { }` is cold and single-collector-per-run. For shared, *hot* streams that exist independently of collectors, use:

- **`StateFlow`** — always holds the latest value (like an observable state holder). New collectors immediately get the current value. Ideal for UI state. Backed by `MutableStateFlow`.
- **`SharedFlow`** — broadcasts emissions to all current collectors; configurable replay buffer. Ideal for events.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

class Counter {
    private val _count = MutableStateFlow(0)        // holds current value
    val count: StateFlow<Int> = _count.asStateFlow() // expose read-only

    fun increment() { _count.value += 1 }
}

fun main() = runBlocking {
    val counter = Counter()
    val job = launch {
        // Collectors of a StateFlow get the latest value immediately, then updates.
        counter.count.take(3).collect { println("count = $it") }
    }
    delay(50); counter.increment()
    delay(50); counter.increment()
    job.join()
}
```

---

## 12. File System, OS Info & Command Execution

Kotlin has no file API of its own — it uses Java's (`java.io.File`, `java.nio.file`) but adds a layer of concise **extension functions** that make them pleasant. This section is JVM-specific.

### 12.1 Reading & writing with `java.io.File` extensions **[I]**

The simplest path is the `File` class plus Kotlin's extensions for whole-file operations. These are great for small-to-moderate files where reading everything into memory is fine.

```kotlin
import java.io.File

fun main() {
    val f = File("notes.txt")

    // WRITE: writeText replaces the entire file (creates it if absent).
    f.writeText("first line\nsecond line\n")

    // APPEND: add to the end without truncating.
    f.appendText("third line\n")

    // READ the whole file as one String (loads it all into memory):
    val all = f.readText()
    println(all)

    // READ all lines into a List<String>:
    val lines: List<String> = f.readLines()
    println("line count: ${lines.size}")

    // Existence / metadata:
    println(f.exists())            // true
    println(f.length())            // size in bytes
    println(f.absolutePath)
    println(f.extension)           // "txt"
    println(f.nameWithoutExtension)
}
```

### 12.2 Streaming large files: `useLines` / `forEachLine` **[I]**

For large files you must *not* load everything into memory. `forEachLine` reads one line at a time. `useLines` gives you a `Sequence<String>` and **automatically closes** the underlying reader when the block finishes (the `use` pattern — Kotlin's try-with-resources, see §13). Prefer these for logs and big data files.

```kotlin
import java.io.File

fun main() {
    val log = File("big.log")

    // forEachLine: process line-by-line; the reader is closed automatically.
    var errorCount = 0
    log.forEachLine { line ->
        if ("ERROR" in line) errorCount++
    }
    println("errors: $errorCount")

    // useLines: lazy Sequence + guaranteed close. Chain functional ops cheaply.
    val firstErrors = log.useLines { lines ->
        lines.filter { "ERROR" in it }.take(5).toList()
    }
    println(firstErrors)
}
```

### 12.3 Directories: `walk`, `mkdirs`, `copyTo`, `deleteRecursively` **[I]**

```kotlin
import java.io.File

fun main() {
    val dir = File("project/src/main")
    dir.mkdirs()                       // create the whole directory chain if needed

    File("project/a.txt").writeText("A")
    File("project/b.txt").writeText("B")

    // walk: depth-first traversal yielding a FileTreeWalk (a Sequence<File>).
    File("project").walk()
        .filter { it.isFile }
        .forEach { println(it.path) }

    // walkTopDown() / walkBottomUp() give explicit ordering.
    val txtFiles = File("project").walkTopDown().filter { it.extension == "txt" }.toList()
    println("txt files: ${txtFiles.size}")

    // Copy a file (overwrite optional); copyRecursively for directories.
    File("project/a.txt").copyTo(File("project/a-copy.txt"), overwrite = true)

    // Delete a whole tree (returns true on success). Use with care!
    File("project").deleteRecursively()
}
```

### 12.4 `java.nio` Files / Path — and when to use which **[I]**

`java.nio.file` (NIO.2) is the modern, more capable file API: richer metadata, symbolic-link handling, file attributes, atomic moves, and a streaming `Files.lines`. Kotlin adds extensions (`Path.readText()`, `Path.writeText()`, `Path.div` for `/`, `path.exists()`, `path.listDirectoryEntries()`).

**When to use which:**
- Use **`java.io.File` + Kotlin extensions** for quick scripts and simple read/write — it's terse and familiar.
- Use **`java.nio.file.Path`/`Files`** for production code that needs robust error handling, file attributes, symlink control, directory streams, or atomic operations. It's the recommended modern API.

```kotlin
import java.nio.file.*
import kotlin.io.path.*       // Kotlin's Path extension functions

fun main() {
    // Build a Path. The `/` operator (Path.div extension) joins segments portably.
    val dir: Path = Path("data") / "reports"
    dir.createDirectories()                 // like mkdirs()

    val file = dir / "summary.txt"
    file.writeText("hello nio\n")           // Kotlin extension over Files.writeString
    file.appendText("more\n")

    println(file.readText())
    println(file.exists())                  // true
    println(file.fileSize())                // bytes
    println(file.isRegularFile())

    // List directory entries (auto-closes the directory stream):
    dir.listDirectoryEntries("*.txt").forEach { println(it) }

    // Stream lines lazily with NIO (use {} closes the stream):
    Files.lines(file).use { stream ->
        stream.filter { it.isNotBlank() }.forEach(::println)
    }

    // Atomic move / copy with options:
    val renamed = dir / "summary.bak"
    file.moveTo(renamed, overwrite = true)
    renamed.deleteIfExists()
    dir.toFile().deleteRecursively()        // cleanup
}
```

### 12.5 OS & system information **[I]**

`System` and `Runtime` expose JVM and OS details. `System.getProperty` reads JVM/OS properties; `System.getenv` reads environment variables; `Runtime` gives memory and processor counts.

```kotlin
fun main() {
    // System properties (set by the JVM / OS):
    println(System.getProperty("os.name"))        // e.g. "Windows 11"
    println(System.getProperty("os.arch"))        // e.g. "amd64"
    println(System.getProperty("user.name"))      // current user
    println(System.getProperty("user.home"))      // home directory
    println(System.getProperty("user.dir"))       // current working directory
    println(System.getProperty("java.version"))   // JVM version
    println(System.lineSeparator().toByteArray().toList())  // \n vs \r\n

    // Environment variables:
    println(System.getenv("PATH"))                 // a single var (null if unset)
    val home = System.getenv("HOME") ?: System.getenv("USERPROFILE")  // cross-platform home
    println(home)
    // System.getenv() with no arg returns the whole Map<String, String>.

    // Runtime: hardware/memory info:
    val rt = Runtime.getRuntime()
    println("CPUs: ${rt.availableProcessors()}")
    println("max memory MB: ${rt.maxMemory() / 1024 / 1024}")

    // Detect the OS portably:
    val isWindows = System.getProperty("os.name").startsWith("Windows", ignoreCase = true)
    println("windows? $isWindows")
}
```

### 12.6 Executing external commands with `ProcessBuilder` **[I]**

To run external programs, use `ProcessBuilder`. You provide the command and its arguments as separate list elements (no shell parsing — safer), optionally set the working directory and environment, start the process, capture its output, and wait for it to finish, inspecting the exit code.

```kotlin
import java.io.File
import java.util.concurrent.TimeUnit

fun main() {
    // Each argument is a SEPARATE list element. On Windows, many built-ins
    // (dir, echo) require running through cmd: listOf("cmd", "/c", "dir").
    val pb = ProcessBuilder("git", "status", "--short")
        .directory(File("."))                  // working directory
        .redirectErrorStream(true)             // merge stderr into stdout

    // Customize the environment for the child process:
    pb.environment()["MY_FLAG"] = "1"          // add/override an env var

    val process = pb.start()

    // Capture output by reading the process's input stream (its stdout).
    val output = process.inputStream.bufferedReader().readText()

    // Wait for completion with a timeout (avoid hanging forever).
    val finished = process.waitFor(10, TimeUnit.SECONDS)
    if (!finished) {
        process.destroyForcibly()
        println("timed out")
        return
    }

    val exitCode = process.exitValue()
    println("exit=$exitCode")
    println(output)
}
```

A reusable helper, the idiomatic Kotlin way:

```kotlin
import java.io.File
import java.util.concurrent.TimeUnit

// Returns (exitCode, combinedOutput). Throws on timeout.
fun runCommand(
    vararg command: String,
    workingDir: File = File("."),
    timeoutSec: Long = 30
): Pair<Int, String> {
    val process = ProcessBuilder(*command)         // spread the vararg into ProcessBuilder
        .directory(workingDir)
        .redirectErrorStream(true)
        .start()
    val output = process.inputStream.bufferedReader().readText()
    if (!process.waitFor(timeoutSec, TimeUnit.SECONDS)) {
        process.destroyForcibly()
        error("command timed out: ${command.joinToString(" ")}")
    }
    return process.exitValue() to output
}

fun main() {
    val (code, out) = runCommand("java", "-version")
    println("exit=$code")
    println(out)
}
```

> **Redirecting to/from files:** `pb.redirectOutput(File("out.txt"))`, `pb.redirectInput(File("in.txt"))`, `pb.redirectError(ProcessBuilder.Redirect.INHERIT)` (send child stderr to your own console). For long-running output, read the stream incrementally instead of `readText()` to avoid filling the pipe buffer (which can deadlock).

### 12.7 CLI args & reading input **[I]**

The `main(args: Array<String>)` form receives command-line arguments. For interactive input use `readlnOrNull()` (returns `null` on end-of-input) or `readln()` (throws on EOF).

```kotlin
fun main(args: Array<String>) {
    // Simple positional + flag parsing by hand:
    val verbose = "--verbose" in args
    val name = args.firstOrNull { !it.startsWith("--") } ?: "world"
    if (verbose) println("(verbose mode)")
    println("Hello, $name")

    // Read a line of input interactively (null at EOF, e.g. piped input ends):
    print("Enter your age: ")
    val age = readlnOrNull()?.toIntOrNull()
    println(if (age != null) "Next year you'll be ${age + 1}" else "no/invalid input")
}
```

> **⚡ Libraries:** For non-trivial CLIs, hand-rolled parsing gets painful. Use **clikt** (the most popular Kotlin CLI library — declarative commands, options, subcommands, help generation) or **kotlinx-cli** (official, lighter). Add `com.github.ajalt.clikt:clikt` to Gradle and you get type-safe options, validation, and auto-generated `--help`.

---

## 13. Exceptions & Result

### 13.1 Why no checked exceptions **[I]**

Java has *checked* exceptions: methods declare `throws IOException`, and callers must either catch or re-declare them. The intent was good, but in practice it led to noise — wrapping in `RuntimeException`, empty `catch` blocks that swallow errors, and `throws` clauses that bubble up through every layer and break encapsulation. Kotlin has **no checked exceptions**: nothing forces you to catch anything. This keeps signatures clean and pushes you toward handling errors where it actually makes sense — or modeling expected failures as values (`Result`, sealed types) rather than control-flow exceptions.

### 13.2 `try`/`catch`/`finally` — also an expression **[I]**

`try` is an expression: it returns the value of the `try` block or the matching `catch` block. `finally` always runs (cleanup).

```kotlin
fun parseOrDefault(s: String): Int {
    // try as an expression: its value is assigned to `result`.
    val result = try {
        s.toInt()                 // value if no exception
    } catch (e: NumberFormatException) {
        -1                        // value if this exception is thrown
    } finally {
        println("parse attempted")  // always runs (cleanup, logging)
    }
    return result
}

fun main() {
    println(parseOrDefault("42"))   // parse attempted / 42
    println(parseOrDefault("x"))    // parse attempted / -1
}
```

### 13.3 Throwing & custom exceptions **[I]**

Throw with `throw`. Define custom exceptions by extending `Exception` (or a subclass). Use the stdlib precondition helpers `require` (argument checks → `IllegalArgumentException`), `check` (state checks → `IllegalStateException`), and `error` (always throws `IllegalStateException`).

```kotlin
class InsufficientFundsException(val shortfall: Int) :
    Exception("Need $shortfall more cents")

fun withdraw(balance: Int, amount: Int): Int {
    require(amount > 0) { "amount must be positive" }     // throws IllegalArgumentException
    if (amount > balance) throw InsufficientFundsException(amount - balance)
    return balance - amount
}

fun main() {
    try {
        withdraw(100, 150)
    } catch (e: InsufficientFundsException) {
        println("declined: short by ${e.shortfall}")      // declined: short by 50
    }
}
```

### 13.4 `Result` & `runCatching` **[I]**

For *expected* failures, returning a value beats throwing. `runCatching { }` runs a block and captures success or failure in a `Result<T>`, which you transform with `map`, `getOrElse`, `getOrNull`, `onSuccess`/`onFailure`, or `fold`. This gives you exception handling as a value pipeline — no try/catch ceremony.

```kotlin
fun main() {
    // runCatching captures success OR the thrown exception into a Result.
    val result: Result<Int> = runCatching { "42".toInt() }

    // Transform and provide fallbacks without try/catch:
    val doubled = result.map { it * 2 }.getOrElse { 0 }
    println(doubled)                                  // 84

    val bad = runCatching { "x".toInt() }
    println(bad.getOrNull())                          // null
    println(bad.exceptionOrNull()?.message)           // For input string: "x"

    // fold handles both branches in one expression:
    val message = runCatching { "100".toInt() }.fold(
        onSuccess = { "ok: $it" },
        onFailure = { "failed: ${it.message}" }
    )
    println(message)                                  // ok: 100
}
```

> **Note:** prefer sealed classes (§8.3) over `Result` when you need *typed*, domain-specific error variants; use `Result`/`runCatching` for wrapping operations that throw.

---

## 14. Advanced: Delegation, DSLs, inline, Contracts

### 14.1 Class delegation with `by` **[A]**

The **delegation pattern** — "implement an interface by forwarding to another object that already implements it" — is built into the language. `class Foo(b: Bar) : Bar by b` makes `Foo` implement `Bar` by delegating every interface method to `b`, while letting you override individual methods. This is "favor composition over inheritance" with zero boilerplate.

```kotlin
interface Repository {
    fun save(item: String)
    fun load(): List<String>
}

class InMemoryRepository : Repository {
    private val items = mutableListOf<String>()
    override fun save(item: String) { items.add(item) }
    override fun load() = items.toList()
}

// LoggingRepository implements Repository BY delegating to `delegate`,
// but overrides save() to add logging. load() is forwarded automatically.
class LoggingRepository(private val delegate: Repository) : Repository by delegate {
    override fun save(item: String) {
        println("saving: $item")
        delegate.save(item)
    }
}

fun main() {
    val repo = LoggingRepository(InMemoryRepository())
    repo.save("a")              // saving: a
    repo.save("b")              // saving: b
    println(repo.load())        // [a, b]  (forwarded to the inner repo)
}
```

### 14.2 Delegated properties: `lazy`, `observable`, map-backed **[A]**

A **delegated property** routes its get/set through a delegate object that provides `getValue`/`setValue`. You've already met `by lazy` (§7.3). The standard library `Delegates` adds `observable` (callback on change) and `vetoable` (veto a change), and you can delegate a property to a `Map` (great for parsing JSON/config into typed fields).

```kotlin
import kotlin.properties.Delegates

class Settings(map: Map<String, Any?>) {
    // Map-backed delegation: the property reads from the map by its own name.
    val host: String by map
    val port: Int by map
}

class Profile {
    // observable: fires the callback whenever the value changes.
    var name: String by Delegates.observable("<unset>") { _, old, new ->
        println("name: '$old' -> '$new'")
    }
}

fun main() {
    val s = Settings(mapOf("host" to "localhost", "port" to 8080))
    println("${s.host}:${s.port}")      // localhost:8080

    val p = Profile()
    p.name = "Ada"                       // name: '<unset>' -> 'Ada'
    p.name = "Grace"                     // name: 'Ada' -> 'Grace'
}
```

### 14.3 Type-safe builders & DSLs (lambda with receiver) **[A]**

Kotlin lets you build internal **DSLs** (domain-specific languages) — fluent, declarative APIs that read like configuration. The key ingredient is a **lambda with receiver**: a function type `T.() -> Unit` where, inside the lambda, `this` is an instance of `T`. So you can call `T`'s methods directly without a qualifier, producing nested, structured code (think Gradle Kotlin DSL or Jetpack Compose). The `@DslMarker` annotation prevents accidentally calling an outer receiver's methods from an inner block.

```kotlin
// Build an HTML-like tree with a type-safe DSL.
class Tag(val name: String) {
    private val children = mutableListOf<Tag>()
    var text: String = ""

    // The parameter type `Tag.() -> Unit` is a lambda WITH RECEIVER:
    // inside `init`, `this` is the new child Tag.
    fun tag(name: String, init: Tag.() -> Unit): Tag {
        val child = Tag(name)
        child.init()                 // run the lambda with `child` as the receiver
        children.add(child)
        return child
    }

    fun render(indent: String = ""): String = buildString {
        append("$indent<$name>")
        if (text.isNotEmpty()) append(text)
        if (children.isNotEmpty()) {
            append("\n")
            children.forEach { append(it.render("$indent  ")).also { append("\n") } }
            append(indent)
        }
        append("</$name>")
    }
}

// Entry point: html { ... } where `this` inside is the root Tag.
fun html(init: Tag.() -> Unit): Tag = Tag("html").apply(init)

fun main() {
    val page = html {
        tag("body") {            // `this` is the body Tag
            tag("h1") { text = "Hello" }
            tag("p") { text = "DSL built tree" }
        }
    }
    println(page.render())
}
```

### 14.4 `inline`, `noinline`, `crossinline` **[A]**

Marking a higher-order function `inline` tells the compiler to copy the function body — *and the lambda's body* — into the call site, eliminating the runtime object allocation and call overhead of the lambda. This is why stdlib functions like `map`, `forEach`, and the scope functions are essentially free. Inlining also enables `reified` (§9.4) and lets a lambda do a **non-local return** (`return` from the enclosing function).

- **`noinline`**: opt a specific lambda parameter *out* of inlining (e.g. you need to store it in a variable).
- **`crossinline`**: keep the lambda inlined but forbid non-local returns from it (needed when the lambda is called from another execution context, like a nested object).

```kotlin
// inline: the lambda is inlined, so `return` below exits main, not just the lambda.
inline fun runIf(condition: Boolean, action: () -> Unit) {
    if (condition) action()
}

inline fun measure(block: () -> Unit): Long {
    val start = System.nanoTime()
    block()
    return System.nanoTime() - start
}

fun process(items: List<Int>) {
    items.forEach {
        runIf(it < 0) {
            println("found negative; stopping")
            return            // NON-LOCAL return: exits process(), not just the lambda
        }
        println("processing $it")
    }
}

fun main() {
    println("took ${measure { Thread.sleep(10) }} ns")
    process(listOf(1, 2, -1, 3))   // stops at -1
}
```

> Only inline small higher-order functions. Inlining a large function called in many places bloats bytecode. The compiler warns when inlining brings no benefit.

### 14.5 Type aliases **[A]**

A `typealias` gives an existing type a new, shorter, or more meaningful name. It's purely a compile-time alias (no new type), useful for taming long generic/function types.

```kotlin
typealias UserId = Int
typealias Handler = (event: String) -> Unit
typealias UserMap = Map<UserId, String>

fun register(handler: Handler) = handler("started")

fun main() {
    val users: UserMap = mapOf(1 to "Ada")
    register { event -> println("handled: $event") }   // handled: started
    println(users)
}
```

### 14.6 Destructuring **[I]**

**Destructuring** unpacks an object into multiple variables in one statement, using the object's `componentN()` functions. Data classes, `Pair`, `Map.Entry`, and collections support it.

```kotlin
data class Point(val x: Int, val y: Int)

fun main() {
    val (x, y) = Point(3, 4)              // calls component1()/component2()
    println("$x, $y")                     // 3, 4

    // In a loop over a map:
    for ((key, value) in mapOf("a" to 1, "b" to 2)) {
        println("$key=$value")
    }

    // Skip a component with underscore:
    val (_, second) = Point(10, 20)
    println(second)                       // 20

    // Common with functions returning Pair:
    val (q, r) = 17 / 5 to 17 % 5
    println("$q remainder $r")            // 3 remainder 2
}
```

### 14.7 Contracts **[A]**

Compiler **contracts** let a function tell the compiler about its behavior so smart-casts and definite-assignment analysis work across the call. For example, `require`/`check` use contracts so that after `requireNotNull(x)` the compiler knows `x` is non-null. You can write your own (an experimental but stable-enough API) for custom validation helpers.

```kotlin
import kotlin.contracts.*

// This contract tells the compiler: if the function returns true,
// then `value` is non-null in the caller afterwards.
@OptIn(ExperimentalContracts::class)
fun isNotNullString(value: String?): Boolean {
    contract {
        returns(true) implies (value != null)
    }
    return value != null
}

fun main() {
    val maybe: String? = "hi"
    if (isNotNullString(maybe)) {
        // Thanks to the contract, `maybe` is smart-cast to non-null String here.
        println(maybe.length)             // 2 — no ?. or !! needed
    }
}
```

---

## 15. Java Interop

Kotlin and Java run on the same JVM and share the same bytecode, so they interoperate two-way. You can mix `.java` and `.kt` files in one module and call freely across them. See **JAVA_GUIDE.md** for the Java side.

### 15.1 Calling Java from Kotlin **[I]**

Java classes, methods, and fields are usable directly. Java getters/setters (`getX`/`setX`) appear as Kotlin **properties** (`obj.x`). Java's `void` is Kotlin's `Unit`.

```kotlin
import java.util.ArrayList
import java.time.LocalDate

fun main() {
    val list = ArrayList<String>()        // a Java class, used like any Kotlin class
    list.add("a")
    println(list.size)                    // Java's size() — also callable as size

    val today = LocalDate.now()           // Java standard library
    println(today.year)                   // getYear() exposed as the property `year`
    println(today.plusDays(7))
}
```

Nullability and platform types from Java were covered in §3.5 — annotate the boundary to stay null-safe.

### 15.2 SAM conversions **[I]**

A **SAM** (Single Abstract Method) interface — a Java functional interface like `Runnable` or `Comparator` — can be implemented by passing a Kotlin lambda directly. The compiler converts the lambda into an instance of the interface.

```kotlin
fun main() {
    // Java's Runnable has one method run(); pass a lambda instead of an anonymous class.
    val r = Runnable { println("running via SAM") }
    Thread(r).start()

    // Comparator<String> via lambda:
    val sorted = listOf("bbb", "a", "cc").sortedWith(Comparator { x, y -> x.length - y.length })
    println(sorted)                       // [a, cc, bbb]
}
```

> Kotlin's own `fun interface` lets you declare your own SAM interfaces that accept lambdas the same way.

### 15.3 Calling Kotlin from Java: `@JvmStatic`, `@JvmOverloads`, `@JvmName` **[I/A]**

Kotlin compiles to ordinary bytecode, but some Kotlin features need annotations to look natural from Java:

- **`@JvmStatic`** — exposes a companion-object/`object` member as a real Java `static` method/field (otherwise Java must go through `Companion`).
- **`@JvmOverloads`** — generates Java overloads for a function with default arguments (Java has no defaults, so without this Java callers must supply every argument).
- **`@JvmName`** — renames the generated Java method/class (resolve name clashes or signature-erasure conflicts).
- **`@JvmField`** — exposes a property as a public Java field (no getter/setter).
- **`@Throws`** — declares checked exceptions in the bytecode so Java callers can `catch` them.

```kotlin
class Calculator {
    companion object {
        @JvmStatic                          // Java: Calculator.add(...) instead of Calculator.Companion.add(...)
        fun add(a: Int, b: Int) = a + b
    }

    // Java gets THREE overloads: greet(name), greet(name, greeting), greet(name, greeting, loud).
    @JvmOverloads
    fun greet(name: String, greeting: String = "Hello", loud: Boolean = false): String {
        val msg = "$greeting, $name"
        return if (loud) msg.uppercase() else msg
    }

    @JvmName("computeChecksum")             // Java sees this method under a different name
    fun checksum(data: String): Int = data.sumOf { it.code }
}
```

```java
// From Java (illustrative):
// int x = Calculator.add(2, 3);                  // works because of @JvmStatic
// new Calculator().greet("Ada");                 // works because of @JvmOverloads
// new Calculator().computeChecksum("hi");        // renamed by @JvmName
```

---

## 16. Tooling & Ecosystem

### 16.1 Gradle with the Kotlin DSL **[I]**

Gradle is the standard build tool for Kotlin/JVM and Android. Modern projects use the **Kotlin DSL** (`build.gradle.kts`) instead of Groovy — you get type safety and IDE autocomplete because the build script is itself Kotlin.

```kotlin
// build.gradle.kts
plugins {
    kotlin("jvm") version "2.1.0"          // the Kotlin/JVM plugin
    application                            // adds a `run` task and packaging
}

repositories {
    mavenCentral()                         // where dependencies are fetched
}

dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.9.0")
    testImplementation(kotlin("test"))     // Kotlin's test bindings (JUnit under the hood)
}

application {
    mainClass.set("com.example.MainKt")    // entry point (note the Kt suffix)
}

kotlin {
    jvmToolchain(21)                       // compile/run against JDK 21
}
```

```bash
# Windows uses the gradlew.bat wrapper; mac/linux use ./gradlew
gradlew.bat build           # compile, test, package   (Windows)
./gradlew build             # (mac/linux)
gradlew.bat run             # run the application
gradlew.bat test            # run tests only
```

### 16.2 Project structure **[I]**

```
myapp/
├── build.gradle.kts
├── settings.gradle.kts          # declares the project/modules
├── gradlew / gradlew.bat        # the Gradle wrapper (commit these)
└── src/
    ├── main/kotlin/com/example/Main.kt
    └── test/kotlin/com/example/MainTest.kt
```

### 16.3 Testing: JUnit & Kotest **[I]**

`kotlin-test` provides assertion functions that delegate to JUnit. **Kotest** is a popular Kotlin-native test framework with expressive matchers and multiple spec styles.

```kotlin
import kotlin.test.Test
import kotlin.test.assertEquals
import kotlin.test.assertTrue

class MathTest {
    @Test
    fun `addition works`() {            // backtick names allow spaces — readable test names
        assertEquals(4, 2 + 2)
        assertTrue("kotlin".isNotEmpty())
    }
}
```

```kotlin
// Kotest (StringSpec style) — fluent matchers via `shouldBe`:
// import io.kotest.core.spec.style.StringSpec
// import io.kotest.matchers.shouldBe
// class CalcSpec : StringSpec({
//     "adds numbers" { (2 + 2) shouldBe 4 }
// })
```

### 16.4 Ktor — server (and client) **[I]**

**Ktor** is JetBrains' Kotlin-native, coroutine-based framework for building asynchronous servers and HTTP clients. Routes are defined in a type-safe DSL.

```kotlin
// A minimal Ktor server (illustrative; needs ktor-server-* dependencies).
import io.ktor.server.engine.*
import io.ktor.server.netty.*
import io.ktor.server.application.*
import io.ktor.server.response.*
import io.ktor.server.routing.*

fun main() {
    embeddedServer(Netty, port = 8080) {
        routing {
            get("/") {
                call.respondText("Hello from Ktor")   // suspend handler — coroutine-backed
            }
            get("/users/{id}") {
                val id = call.parameters["id"]
                call.respondText("user $id")
            }
        }
    }.start(wait = true)
}
```

### 16.5 Jetpack Compose & Android **[I]**

For Android, Kotlin + **Jetpack Compose** is the modern declarative UI toolkit. UI is described with `@Composable` functions; state changes recompose the UI automatically. (Compose Multiplatform extends this to desktop/iOS/web.)

```kotlin
// Illustrative Compose snippet (needs Android/Compose setup):
// @Composable
// fun Greeting(name: String) {
//     var count by remember { mutableStateOf(0) }   // state survives recomposition
//     Column {
//         Text("Hello, $name! Clicked $count times")
//         Button(onClick = { count++ }) { Text("Click") }
//     }
// }
```

### 16.6 Kotlin Multiplatform (KMP) overview **[A]**

KMP lets you write business logic once and share it across platforms while keeping platform-specific UI/native code where needed. The **`commonMain`** source set holds shared code; platform source sets (`androidMain`, `iosMain`, `jvmMain`, `jsMain`) hold target-specific implementations. The **`expect`/`actual`** mechanism declares an API in common code (`expect`) and provides a per-platform implementation (`actual`).

```kotlin
// commonMain — declare the API expected on every platform:
// expect fun platformName(): String

// androidMain:
// actual fun platformName(): String = "Android ${android.os.Build.VERSION.SDK_INT}"

// iosMain:
// actual fun platformName(): String = "iOS"
```

Shared layers commonly include networking (Ktor client), serialization (`kotlinx.serialization`), coroutines, and database access (SQLDelight). **Compose Multiplatform** can additionally share the UI.

---

## 17. Gotchas & Best Practices

### 17.1 Classic gotchas

```kotlin
fun main() {
    // 1) `val` freezes the REFERENCE, not the object. The list can still mutate.
    val list = mutableListOf(1, 2)
    list.add(3)                         // allowed — the object is mutable
    println(list)                       // [1, 2, 3]

    // 2) == is structural (equals); === is referential (identity). Use == for values.
    val a: Int? = 1000
    val b: Int? = 1000
    println(a == b)                     // true  (value)
    // println(a === b)                 // may be false (boxed objects differ)

    // 3) Integer division truncates. Convert to Double for a fractional result.
    println(5 / 2)                      // 2   (NOT 2.5)
    println(5 / 2.0)                    // 2.5

    // 4) Ranges: `..` is inclusive on BOTH ends; use `until`/`..<` for exclusive upper.
    println((1..3).toList())            // [1, 2, 3]
    println((1 until 3).toList())       // [1, 2]

    // 5) Extension functions dispatch on the STATIC type, not the runtime type.
    open class Base; class Derived : Base()
    fun Base.who() = "Base"
    fun Derived.who() = "Derived"
    val x: Base = Derived()
    println(x.who())                    // "Base" — resolved by declared type, not runtime!

    // 6) `lateinit` access before init throws; check with ::prop.isInitialized.

    // 7) Don't capture `this` in long-lived lambdas in Android (memory leaks).
}
```

```kotlin
// 8) Companion object members are NOT free statics from Java without @JvmStatic.
// 9) Nullable Boolean in conditions: `if (flag)` won't compile for Boolean? —
//    use `if (flag == true)` or `flag ?: false`.
// 10) `apply`/`also` return the object; `let`/`run`/`with` return the lambda result —
//     mixing them up silently returns the wrong thing. (See §6 table.)
```

### 17.2 Idioms & best practices

- **Prefer `val` over `var`**, and read-only collections (`List`) over mutable ones in signatures. Immutability simplifies reasoning and concurrency.
- **Embrace null safety** — avoid `!!`; use `?.`, `?:`, `let`, `requireNotNull`. Annotate Java-interop boundaries.
- **Use expressions** — `if`/`when`/`try` return values; favor single-expression functions.
- **Model fixed variants with sealed types** and exhaustive `when` instead of error codes or string flags.
- **Use data classes** for value objects; **value classes** to make primitive types type-safe.
- **Prefer the functional collection operators** (`map`/`filter`/`groupBy`) over manual loops; switch to **sequences** only for large multi-step pipelines.
- **Structured concurrency**: launch coroutines inside a scope, never leak them; use the right dispatcher (`IO` for blocking, `Default` for CPU).
- **Extension functions** for utilities and readability — but remember they're statically dispatched and members win.
- **Scope functions**: `apply` to configure, `let` to null-guard/transform, `also` for side-effects. Don't nest them deeply.
- **Naming (Kotlin convention):** `camelCase` for functions/properties, `PascalCase` for classes/objects, `UPPER_SNAKE_CASE` for `const`/top-level constants. Let **ktlint** or the IDE formatter enforce style.
- **Keep functions small and pure** where possible; push side effects (I/O) to the edges.
- Run **`gradlew.bat ktlintCheck`** / detekt in CI for style and static analysis.

---

## 18. Study Path & Build-to-Learn Projects

**Suggested order:** §1–4 (basics, null safety, control flow) → §5–6 (functions, lambdas, scope functions) → §7–8 (OOP & special classes) → §10 (collections — immediately useful) → §12–13 (files/OS & errors) → §9, §11 (generics, coroutines) → §14–16 (advanced, interop, tooling) → §17 (gotchas).

**Build these to cement it:**
1. **CLI todo manager** — parse `args`, persist to a JSON/text file with `File` extensions, model items with a `data class`, status as a sealed type. Exercises §2, §5, §8, §12.
2. **Log analyzer** — stream a large file with `useLines`, extract fields with regex, aggregate with `groupBy`/`associate`, print a summary table. Exercises §10, §12.
3. **Concurrent web/price fetcher** — fetch several URLs in parallel with `async`/`await`, add timeouts and cancellation, stream results with `Flow`. Exercises §11.
4. **File organizer / backup tool** — `walk` a tree, move files by extension, zip a folder, shell out to `git` via `ProcessBuilder`, read OS info to behave correctly on Windows vs Unix. Exercises §12.
5. **Mini DSL** — build a type-safe configuration or HTML builder with lambdas-with-receiver; add delegated properties for lazy/observable fields. Exercises §14.
6. **A small Ktor service or KMP module** — expose a couple of routes (or share a model between JVM and JS), add tests with `kotlin-test`/Kotest, build with the Gradle Kotlin DSL. Exercises §15, §16.

**Next steps after this guide:** a server framework (Ktor or Spring Boot with Kotlin), `kotlinx.serialization` for JSON, Exposed or SQLDelight for databases, Jetpack Compose for Android UI, and Kotlin Multiplatform for cross-platform sharing. For the JVM/file/OS material from the Java perspective, compare with **JAVA_GUIDE.md** in this library.

---
