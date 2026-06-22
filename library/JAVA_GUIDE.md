# Java — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I've never compiled a Java file" to "I write maintainable, modern, production Java" — without an internet connection. Java has a reputation for being verbose and old-fashioned; this guide shows you the *modern* language (records, sealed types, pattern matching, virtual threads) alongside the timeless fundamentals (the JVM, OOP, collections, generics). Every concept is explained in prose first — what it is, *why* it works that way, and when to reach for it — and then shown in heavily-commented, runnable code. Read top-to-bottom the first time; afterward, use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Java 21 LTS and Java 25 LTS** (current in 2026). "LTS" means Long-Term Support — the releases enterprises actually standardize on. Java now ships a new feature release every 6 months, with an LTS every 2 years. Things worth knowing about modern Java:
> - **Records** (16+) — concise immutable data carriers; replace most "data class" boilerplate.
> - **Sealed classes/interfaces** (17+) — restrict who may extend/implement, enabling exhaustive `switch`.
> - **Pattern matching** — for `instanceof` (16+), for `switch` (21), and **record patterns / deconstruction** (21). These together turn Java into a far more expressive language.
> - **Text blocks** (15+) — multi-line string literals with `"""`.
> - **Virtual threads / Project Loom** (21) — millions of cheap threads; the biggest concurrency change in Java's history.
> - **Structured concurrency** (preview through 21–24, **stable in 25** as `java.util.concurrent` API) — treat a group of concurrent subtasks as one unit.
> - **Simplified `void main()`** — JEP 445/477/512: you can now write a `main` without `public static`, without a class declaration, and without `String[] args`. Stable in Java 25 as "compact source files / instance main methods." Great for learners and scripts.
>
> Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**, so cross-platform notes (path separators, `java` vs `java.exe`, shells) are called out. Confirm exact APIs against the JDK Javadoc (`docs.oracle.com/en/java/javase/21/`).

---

## Table of Contents

1. [The Platform: JDK, JRE, JVM & How Java Runs](#1-the-platform-jdk-jre-jvm--how-java-runs) **[B]**
2. [Syntax, Types & Strings](#2-syntax-types--strings) **[B]**
3. [Control Flow](#3-control-flow) **[B]**
4. [OOP Core: Classes, Objects, Packages](#4-oop-core-classes-objects-packages) **[B/I]**
5. [OOP Advanced: Inheritance, Polymorphism, Interfaces](#5-oop-advanced-inheritance-polymorphism-interfaces) **[I]**
6. [Modern Data Modeling: Records, Enums, Sealed, Patterns](#6-modern-data-modeling-records-enums-sealed-patterns) **[I/A]**
7. [Generics](#7-generics) **[I/A]**
8. [The Collections Framework](#8-the-collections-framework) **[I]**
9. [Functional Java: Lambdas, Streams, Optional](#9-functional-java-lambdas-streams-optional) **[I/A]**
10. [Exceptions & Error Handling](#10-exceptions--error-handling) **[I]**
11. [File System, OS Info & Command Execution](#11-file-system-os-info--command-execution) **[I]**
12. [Concurrency & Parallelism](#12-concurrency--parallelism) **[A]**
13. [Standard Library Utilities](#13-standard-library-utilities) **[I]**
14. [Build Tools & Ecosystem](#14-build-tools--ecosystem) **[I/A]**
15. [Testing: JUnit 5 & Mockito](#15-testing-junit-5--mockito) **[I]**
16. [Memory & Performance](#16-memory--performance) **[A]**
17. [Gotchas & Best Practices](#17-gotchas--best-practices) **[A]**
18. [Study Path & Build-to-Learn Projects](#18-study-path--build-to-learn-projects)

---

## 1. The Platform: JDK, JRE, JVM & How Java Runs

Before writing a line of Java, understand *what Java actually is*, because almost every beginner confusion ("Why do I need to compile?", "What's a JVM?", "Which Java do I install?") comes from not having this mental model.

### 1.1 The big idea: "write once, run anywhere" **[B]**

When you write Java source code (a `.java` file), it is **not** run directly the way a Python script is. Instead it goes through two stages:

1. **Compilation** — the `javac` compiler translates your human-readable `.java` source into **bytecode**, stored in `.class` files. Bytecode is a compact, platform-independent instruction set for an imaginary computer. It is *not* machine code for your actual CPU — no Intel or ARM chip can run it directly.
2. **Execution** — the **JVM (Java Virtual Machine)** loads those `.class` files and runs the bytecode. The JVM is a real program (`java.exe` on Windows) that exists in a different build for every operating system and CPU.

The genius here is the *layer of indirection*. Because bytecode is the same everywhere, the **same** compiled `.class`/`.jar` file runs unchanged on Windows, macOS, and Linux — as long as each has a JVM. The JVM absorbs all the platform differences. This is what "write once, run anywhere" (WORA) means, and it's why Java dominated enterprise and Android development.

### 1.2 JIT compilation — why Java is fast despite the interpreter **[B/I]**

If the JVM just *interpreted* bytecode instruction-by-instruction, Java would be slow. It doesn't. The JVM uses a **JIT (Just-In-Time) compiler**:

- The JVM starts by interpreting bytecode (fast startup).
- It *profiles* your running program, noticing which methods are called most often ("hot" code).
- It then compiles those hot methods to **native machine code** at runtime, optimizing aggressively using real runtime information (e.g. which branches are actually taken, which types actually flow through). The HotSpot JVM is literally named after this "hot spot" detection.

The payoff: long-running Java programs (servers, in particular) often run as fast as C++ for the hot paths, because the JIT can make optimization decisions a static compiler can't. The trade-off is *warm-up time* — the program is slower for the first few seconds until the JIT kicks in. (Modern projects like GraalVM Native Image trade this away by ahead-of-time compiling to a native binary, sacrificing some peak throughput for instant startup.)

### 1.3 JDK vs JRE vs JVM — what each is **[B]**

These three acronyms confuse everyone. Here is the precise relationship, from innermost to outermost:

| Term | What it is | Contains | You use it to... |
|------|-----------|----------|------------------|
| **JVM** | The runtime engine that executes bytecode | The interpreter, JIT, garbage collector, memory manager | (It's embedded in the JRE — you don't install it alone) |
| **JRE** | Java Runtime Environment | JVM **+** the core class libraries (`java.lang`, `java.util`, etc.) | *Run* compiled Java programs |
| **JDK** | Java Development Kit | JRE **+** developer tools (`javac`, `jar`, `javadoc`, `jshell`, debugger) | *Develop* (compile and run) Java |

**Bottom line for you as a developer: install the JDK.** The JDK includes everything (it contains a JRE, which contains a JVM). Since Java 11, Oracle stopped shipping a standalone JRE for download — you either get the full JDK, or you create a slim custom runtime with `jlink`.

### 1.4 Choosing and installing a JDK **[B]**

Java is an open standard (the OpenJDK project) with multiple *distributions* (builds) from different vendors. They are functionally equivalent for almost all purposes; they differ in license, support, and bundled extras.

- **Eclipse Temurin** (from Adoptium) — the community default, free, no commercial strings. **Recommended for learning and most production.**
- **Oracle JDK** — Oracle's build; free for most uses under the current license but watch the terms.
- **Amazon Corretto**, **Microsoft Build of OpenJDK**, **Azul Zulk/Zulu**, **Red Hat** — all solid; pick based on your cloud/OS vendor.

Install on Windows:

```bash
# Easiest: winget (Windows Package Manager)
winget install EclipseAdoptium.Temurin.21.JDK

# Or download the .msi installer from adoptium.net
```

On macOS: `brew install temurin@21`. On Linux: use your package manager or `sdkman` (`sdk install java 21-tem`). **SDKMAN!** is the best way to manage *multiple* JDK versions across platforms (except native Windows; use it in WSL/Git Bash).

Verify the install:

```bash
java --version     # e.g. "openjdk 21.0.2 2024-01-16 LTS"
javac --version    # the compiler — "javac 21.0.2"
```

If `java` works but `javac` doesn't, you installed a JRE-only package, not a JDK. Also make sure your `PATH` (and optionally `JAVA_HOME`) points at the JDK's `bin` directory.

### 1.5 Your first program — the classic way **[B]**

Create `Hello.java`:

```java
// The file name MUST match the public class name: Hello.java -> class Hello.
public class Hello {
    // The JVM looks for this exact signature as the program's entry point.
    // public   - callable from outside (the JVM needs to call it)
    // static   - belongs to the class, not an instance (no object exists yet at startup)
    // void     - returns nothing
    // main     - the conventional name the JVM searches for
    // String[] args - command-line arguments passed to the program
    public static void main(String[] args) {
        System.out.println("Hello, world!");  // println = print line (adds a newline)
    }
}
```

Compile and run:

```bash
javac Hello.java     # produces Hello.class (bytecode)
java Hello           # runs the class — note: NO ".class" extension here
# -> Hello, world!
```

What happened: `javac` turned source into `Hello.class`; `java Hello` started a JVM, loaded `Hello.class`, found `main`, and ran it.

### 1.6 Single-file run & the simplified `main` — modern Java **[B]**

Since Java 11 you can **skip the explicit compile step** for a single source file. The `java` launcher compiles it in memory and runs it:

```bash
java Hello.java      # compiles + runs in one step (great for scripts/learning)
```

**⚡ Version note (Java 21–25):** Java has dramatically simplified the entry point for beginners and scripting. In **Java 25** (stable; preview in 21) you can write a *compact source file* with an **instance `main`** — no `public`, no `static`, no class declaration, no `String[] args` required:

```java
// File: hello.java  — that's the ENTIRE file. No class, no public static.
void main() {
    System.out.println("Hello from compact Java!");
    // IO.println(...) is also available as a simplified console helper in newer JDKs.
}
```

```bash
java hello.java      # just works on Java 25
```

This removes the famous "why do I have to type `public static void main(String[] args)` to print one line?" barrier. Under the hood the compiler still wraps it in a class — it's pure syntax sugar — but it lets newcomers focus on logic first. Use the full `public static void main(String[] args)` form for real applications and libraries; use the compact form for quick experiments.

### 1.7 JShell — the REPL **[B]**

The JDK ships a REPL (Read-Eval-Print Loop) called **JShell** for experimenting without writing a full class:

```bash
jshell
jshell> int x = 2 + 2
x ==> 4
jshell> "hello".toUpperCase()
$2 ==> "HELLO"
jshell> /exit
```

Use JShell to test a snippet, explore an API, or check what a method returns — no boilerplate required.

---

## 2. Syntax, Types & Strings

### 2.1 Statements, comments & variables **[B]**

Java is **statically typed** (every variable has a type known at compile time) and **block-structured** with curly braces `{}`. Statements end with a semicolon `;`. Whitespace and indentation are for humans only — the compiler ignores them.

```java
// Single-line comment
/* Multi-line
   comment */
/** Javadoc comment — used to generate API docs; placed above classes/methods. */

int count = 10;            // declare type, name, and value
double price = 19.99;
boolean ready = true;
char grade = 'A';          // a single character — SINGLE quotes
String name = "Ada";       // a string — DOUBLE quotes (String is a class, not a primitive)

count = count + 1;         // reassignment
final int MAX = 100;       // `final` = constant; cannot be reassigned
// MAX = 200;              // compile error
```

### 2.2 Primitives vs reference types — the core mental model **[B/I]**

This is *the* distinction that explains most of Java's behaviour. Java has exactly **two kinds of types**:

**Primitive types** hold a raw value *directly* in the variable's memory slot. There are eight, and they are lowercase. A primitive variable *is* its value.

**Reference types** (everything else — `String`, arrays, your own classes, collections) hold a **reference** (effectively a pointer/handle) to an object that lives elsewhere on the **heap**. The variable does not contain the object; it contains the *address of* the object.

Why does this matter? **Identity and copying.** When you assign or pass a primitive, you copy the *value*. When you assign or pass a reference, you copy the *reference* — both variables now point at the *same* object, so mutating through one is visible through the other. This is the source of countless beginner surprises.

```java
// PRIMITIVES: copying copies the value. Independent afterward.
int a = 5;
int b = a;        // b gets its own copy of 5
b = 99;
System.out.println(a + " " + b);   // 5 99  -> a is untouched

// REFERENCES: copying copies the pointer. Same underlying object.
int[] x = {1, 2, 3};
int[] y = x;      // y points at the SAME array as x
y[0] = 99;
System.out.println(x[0]);          // 99  -> changing y changed x's array!
```

The eight primitive types:

| Type | Size | Range / notes | Default |
|------|------|---------------|---------|
| `byte` | 8-bit | -128 to 127 | 0 |
| `short` | 16-bit | -32,768 to 32,767 | 0 |
| `int` | 32-bit | ~±2.1 billion — the default integer type | 0 |
| `long` | 64-bit | ~±9.2 quintillion — suffix literal with `L` (`10_000_000_000L`) | 0L |
| `float` | 32-bit | IEEE-754 single precision — suffix `f` (`3.14f`) | 0.0f |
| `double` | 64-bit | IEEE-754 double — the default floating type | 0.0d |
| `boolean` | (JVM-defined) | `true` / `false` | false |
| `char` | 16-bit | a single UTF-16 code unit (`'A'`, `'\n'`) | `' '` |

Each primitive has a **wrapper class** (`Integer`, `Long`, `Double`, `Boolean`, `Character`, etc.) — a reference-type object that "boxes" the value. You need these when a type parameter must be an object (collections can't hold `int`, only `Integer`). **Autoboxing** converts automatically:

```java
Integer boxed = 42;        // autoboxing: int -> Integer
int unboxed = boxed;       // auto-unboxing: Integer -> int
List<Integer> nums = new ArrayList<>();
nums.add(7);               // autoboxes int 7 into Integer
```

Beware: wrapper objects can be `null`, and unboxing a `null` throws `NullPointerException`. Autoboxing in tight loops also costs performance (see §16). And **never** compare wrappers with `==` (§17).

### 2.3 `var` — local type inference **[B/I]**

Since Java 10 you can use `var` for **local variables** and let the compiler infer the type from the initializer. This is *not* dynamic typing — the type is still fixed at compile time; you just don't write it.

```java
var message = "hello";          // inferred String
var count = 10;                 // inferred int
var list = new ArrayList<String>();  // inferred ArrayList<String>

// Rules: var requires an initializer (the compiler needs something to infer from).
// var x;          // ERROR — no initializer
// var n = null;   // ERROR — null has no type to infer
```

Use `var` to cut noise when the type is obvious from the right-hand side (`var users = new HashMap<String, User>();`). Avoid it when it *hides* the type and hurts readability (`var x = getThing();` — what is `x`?). `var` works only for local variables, not fields, parameters, or return types.

### 2.4 Operators **[B]**

```java
// Arithmetic
int sum = 7 + 3, diff = 7 - 3, prod = 7 * 3;
int quot = 7 / 2;      // 3  -> INTEGER division truncates toward zero!
int rem  = 7 % 2;      // 1  -> modulo (remainder)
double d = 7.0 / 2;    // 3.5 -> floating division if either operand is floating

// A classic trap: 5 / 2 is 2, not 2.5. Cast to get a real division:
double half = (double) 5 / 2;   // 2.5

// Increment / decrement
int i = 0;
i++;  // post-increment (use value, then add)
++i;  // pre-increment (add, then use value)

// Comparison -> boolean
boolean eq = (a == b), ne = (a != b), gt = (a > b), le = (a <= b);

// Logical (short-circuit: && stops at first false, || stops at first true)
boolean both = (a > 0) && (b > 0);
boolean either = (a > 0) || (b > 0);
boolean not = !ready;

// Ternary (the only operator with three operands): condition ? ifTrue : ifFalse
String label = (count > 0) ? "positive" : "non-positive";

// Bitwise (operate on the binary representation): & | ^ ~ << >> >>>
int flags = 0b0011 | 0b0100;   // 0b0111 = 7   (binary literals with 0b)
```

### 2.5 Strings — immutability and *why* **[B/I]**

`String` is a reference type (a class), not a primitive, but it's so common Java gives it literal syntax (`"..."`) and `+` concatenation. The single most important fact about strings: **they are immutable.** Once created, a `String`'s characters never change. Every method that "modifies" a string actually returns a *new* string.

Why make strings immutable? Several deep reasons:
- **Safety in sharing:** because a string can never change, it's safe to share one string object across threads and data structures without defensive copying or locking.
- **Hashing & caching:** the JVM caches a string's hash code; since the content never changes, the cache is always valid. This makes strings excellent `HashMap` keys.
- **The string pool:** identical string *literals* are interned (shared) in a pool, saving memory. This is only safe because they're immutable.
- **Security:** a string passed to, say, a file-open or network call can't be changed by another thread between the check and the use.

```java
String s = "Hello";
String t = s.toUpperCase();   // returns a NEW string "HELLO"
System.out.println(s);         // "Hello"  -> s is unchanged
System.out.println(t);         // "HELLO"

// Common methods (all return new strings or read-only info):
String x = "  Hello, World  ";
x.length();                    // 17
x.trim();                      // "Hello, World" (legacy); strip() is Unicode-aware (11+)
x.strip();                     // preferred over trim()
x.toLowerCase();               // "  hello, world  "
x.replace("l", "L");           // replaces all 'l'
x.substring(2, 7);             // "Hello" (start inclusive, end exclusive)
x.contains("World");           // true
x.indexOf("World");            // 9  (-1 if not found)
x.startsWith("  H");           // true
"a,b,c".split(",");            // String[] {"a", "b", "c"}
String.join("-", "a", "b");    // "a-b"
"abc".charAt(0);               // 'a'
"42".isBlank();                // false (isBlank: only whitespace?)
"  ".isBlank();                // true
"x".repeat(3);                 // "xxx" (11+)
String.format("%s is %d", "age", 30);   // "age is 30"  (printf-style)
```

**Comparing strings:** use `.equals()`, **never** `==`. `==` compares references (are these the *same object*?); `.equals()` compares *content*. Because of the string pool, `==` sometimes *appears* to work for literals, which makes the bug intermittent and nasty:

```java
String a = "hi";
String b = "hi";
String c = new String("hi");    // forces a new object, bypassing the pool
System.out.println(a == b);      // true  (both point at the pooled literal — DON'T rely on this)
System.out.println(a == c);      // false (different objects)
System.out.println(a.equals(c)); // true  (same content — ALWAYS use this)
```

### 2.6 Text blocks — multi-line strings **[B]**

**⚡ Version note (15+):** Text blocks use triple quotes `"""` for readable multi-line strings — perfect for JSON, SQL, HTML, or any embedded text. Indentation is handled intelligently: the compiler strips the common leading whitespace (the "incidental" indentation) based on the closing `"""`.

```java
String json = """
        {
            "name": "Ada",
            "active": true
        }
        """;     // the closing """ position determines how much indentation is stripped

String sql = """
        SELECT id, name
        FROM users
        WHERE active = true
        """;
// No need to escape internal double-quotes or write \n at every line end.
```

### 2.7 StringBuilder — when to use it **[B/I]**

Because strings are immutable, building one up with `+=` in a loop is *quadratic*: each `+=` allocates a brand-new string and copies all prior characters. For a loop of N appends you copy ~N²/2 characters. For one or two concatenations, `+` is fine (the compiler optimizes it). But **inside a loop, use `StringBuilder`**, a mutable buffer that appends in place.

```java
// BAD: allocates a new String every iteration (O(n^2) total work)
String result = "";
for (int i = 0; i < 10000; i++) {
    result += i + ",";        // each += copies the whole growing string
}

// GOOD: one mutable buffer, amortized O(n)
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i).append(",");  // append returns `this`, so calls chain
}
String result2 = sb.toString();   // convert to an immutable String at the end
```

(`StringBuffer` is the older, thread-safe — and slower — version. Use `StringBuilder` unless you genuinely share the buffer across threads, which is rare.)

### 2.8 Arrays **[B]**

An array is a fixed-size, ordered, indexed container of elements *of one type*. It is a reference type (lives on the heap). Its length is set at creation and never changes — if you need a growable list, use `ArrayList` (§8).

```java
int[] nums = new int[5];        // 5 ints, all default-initialized to 0
nums[0] = 10;                    // index from 0
nums[4] = 50;
System.out.println(nums.length); // 5  (a field, not a method — note: no parentheses)
// nums[5] = 1;                  // throws ArrayIndexOutOfBoundsException at runtime

int[] primes = {2, 3, 5, 7, 11}; // array literal — size inferred
String[] names = {"Ada", "Linus"};

// Iterate with a classic for or an enhanced for-each:
for (int i = 0; i < primes.length; i++) System.out.println(primes[i]);
for (int p : primes) System.out.println(p);   // for-each: read each element

// Multi-dimensional (an array of arrays):
int[][] grid = new int[3][4];   // 3 rows, 4 columns
grid[1][2] = 7;

// Helpers in java.util.Arrays:
import java.util.Arrays;
int[] a = {3, 1, 2};
Arrays.sort(a);                          // in-place sort -> {1, 2, 3}
System.out.println(Arrays.toString(a));  // "[1, 2, 3]"  (arrays' own toString is useless)
int[] copy = Arrays.copyOf(a, 5);        // {1, 2, 3, 0, 0}
boolean same = Arrays.equals(a, copy);   // element-wise comparison
```

---

## 3. Control Flow

### 3.1 if / else if / else **[B]**

```java
int score = 75;
if (score >= 90) {
    System.out.println("A");
} else if (score >= 80) {
    System.out.println("B");
} else if (score >= 70) {
    System.out.println("C");
} else {
    System.out.println("F");
}

// The condition MUST be a boolean. Unlike C, you can't write `if (x)` for an int.
// Braces are optional for single statements but ALWAYS use them — it prevents bugs
// like the infamous "goto fail" where an unbraced if silently runs only one line.
```

### 3.2 switch statements — the classic form **[B]**

A traditional `switch` compares one value against constant `case` labels. Its notorious gotcha is **fall-through**: without a `break`, execution continues into the next case. This is occasionally useful but mostly a bug magnet.

```java
int day = 3;
switch (day) {
    case 1:
        System.out.println("Monday");
        break;                 // without break, falls into case 2!
    case 2:
        System.out.println("Tuesday");
        break;
    case 6:
    case 7:                    // intentional fall-through: 6 and 7 share a body
        System.out.println("Weekend");
        break;
    default:
        System.out.println("Midweek");
}
```

### 3.3 switch expressions — the modern form **[B/I]**

**⚡ Version note (14+):** The modern `switch` is an **expression** (it produces a value) using the **arrow** `->` syntax. There is **no fall-through** (each arm is self-contained, no `break` needed), it can return a value, and the compiler enforces **exhaustiveness** (you must cover every case, often via `default`). Use `yield` to return a value from a multi-statement block arm.

```java
int day = 3;

// Arrow form returning a value — clean, safe, no break needed:
String name = switch (day) {
    case 1 -> "Monday";
    case 2 -> "Tuesday";
    case 6, 7 -> "Weekend";          // comma-separated labels share an arm
    default -> "Midweek";
};

// Block arm with yield (when you need multiple statements):
int numLetters = switch (day) {
    case 1, 3, 5 -> {
        System.out.println("computing...");
        yield 6;                     // `yield` returns the value of the block
    }
    default -> 0;
};

// Switching on strings and enums works too:
String fruit = "apple";
int calories = switch (fruit) {
    case "apple" -> 95;
    case "banana" -> 105;
    default -> 0;
};
```

This pairs powerfully with sealed types and pattern matching (§6) to write exhaustive, type-safe logic.

### 3.4 Loops **[B]**

```java
// while: check before each iteration
int i = 0;
while (i < 5) { System.out.println(i); i++; }

// do-while: body runs at least once (check after)
int j = 0;
do { System.out.println(j); j++; } while (j < 5);

// classic for: init; condition; update
for (int k = 0; k < 5; k++) System.out.println(k);

// enhanced for (for-each): iterate elements of any array or Iterable
int[] arr = {10, 20, 30};
for (int n : arr) System.out.println(n);

List<String> names = List.of("Ada", "Linus");
for (String s : names) System.out.println(s);
```

### 3.5 break, continue & labels **[B/I]**

```java
for (int i = 0; i < 10; i++) {
    if (i == 3) continue;   // skip the rest of THIS iteration
    if (i == 7) break;      // exit the loop entirely
    System.out.println(i);
}

// Labeled break/continue — to control an OUTER loop from inside a nested one.
// Use sparingly; often a sign you should extract a method instead.
outer:
for (int r = 0; r < 3; r++) {
    for (int c = 0; c < 3; c++) {
        if (r * c > 2) break outer;   // breaks the OUTER loop, not just inner
    }
}
```

---

## 4. OOP Core: Classes, Objects, Packages

Java is fundamentally object-oriented: nearly all code lives inside classes. This section builds the foundation; §5 covers the advanced relationships.

### 4.1 Classes and objects — the distinction **[B]**

A **class** is a blueprint/template that defines what *data* (fields) and *behaviour* (methods) a kind of thing has. An **object** (instance) is a concrete thing created from that blueprint at runtime with `new`. One class, many objects. The class `Car` describes "a car has a color and can drive"; each `new Car(...)` is a specific car with its own color.

```java
public class Car {
    // FIELDS (instance variables) — state each Car object carries
    String color;
    int speed;

    // METHOD — behaviour
    void accelerate(int amount) {
        speed += amount;       // operates on THIS object's speed
    }
}

// Elsewhere:
Car c1 = new Car();      // `new` allocates a Car object on the heap; c1 references it
c1.color = "red";
c1.accelerate(30);
System.out.println(c1.speed);  // 30

Car c2 = new Car();      // a separate, independent object
System.out.println(c2.speed);  // 0  (its own state)
```

### 4.2 Constructors and `this` **[B/I]**

A **constructor** is special code that runs when an object is created, used to initialize its fields. It has the same name as the class and no return type. If you write none, Java provides an invisible no-arg default constructor; the moment you write *any* constructor, the default disappears.

`this` refers to "the current object." Its most common use is to disambiguate a field from a constructor parameter with the same name.

```java
public class Car {
    String color;
    int speed;

    // Constructor: initializes a new Car
    public Car(String color, int speed) {
        this.color = color;    // this.color = the FIELD; color = the PARAMETER
        this.speed = speed;
    }

    // Constructor overloading: multiple constructors with different parameters
    public Car(String color) {
        this(color, 0);        // this(...) calls another constructor — must be first line
    }
}

Car c = new Car("blue", 50);
```

### 4.3 Access modifiers & encapsulation — the *why* **[B/I]**

Access modifiers control *who can see* a field, method, or class. The reasoning behind them is **encapsulation**: hiding an object's internal state behind a controlled interface so the class can enforce invariants and change its internals without breaking callers.

| Modifier | Visible from |
|----------|-------------|
| `private` | only within the same class |
| (none = *package-private*) | classes in the same package |
| `protected` | same package **+** subclasses (even in other packages) |
| `public` | everywhere |

The standard pattern: make fields `private`, expose controlled `public` **getters/setters**. Why bother, instead of just public fields? Because a setter can *validate* (reject a negative balance), the field can be *computed* or *renamed* later without changing callers, and the object can never be put into an illegal state from outside.

```java
public class BankAccount {
    private double balance;        // hidden — no outside code can corrupt it

    public BankAccount(double initial) {
        if (initial < 0) throw new IllegalArgumentException("negative balance");
        this.balance = initial;
    }

    public double getBalance() { return balance; }     // controlled read

    public void deposit(double amount) {                // controlled write with a rule
        if (amount <= 0) throw new IllegalArgumentException("must be positive");
        balance += amount;
    }
    // No setBalance(): the only legal ways to change balance are deposit/withdraw.
}
```

### 4.4 `static` — class members vs instance members **[B/I]**

`static` means "belongs to the *class itself*, not to any individual object." A `static` field is shared by all instances (there's exactly one copy). A `static` method can be called without creating an object (`Math.sqrt(...)`) — but it cannot access instance fields, because there's no `this`.

```java
public class Counter {
    private static int totalCreated = 0;   // ONE copy shared by all Counter objects
    private int id;                          // each object has its own

    public Counter() {
        totalCreated++;                      // increments the shared counter
        this.id = totalCreated;
    }

    public static int getTotalCreated() {    // callable without an instance
        return totalCreated;
        // return id;   // ERROR: static method can't see instance field `id`
    }
}

new Counter(); new Counter();
System.out.println(Counter.getTotalCreated());  // 2  -> called ON THE CLASS

// `static final` = a true constant:
public static final double PI = 3.14159;
```

### 4.5 Packages and imports **[B]**

A **package** is a namespace — a folder-like grouping that prevents class-name collisions and organizes code. The package declaration must be the first line of the file, and the folder structure must mirror it: `com.example.app.Car` lives in `com/example/app/Car.java`. Convention is reverse-domain-name (`com.yourcompany.project`).

```java
// File: com/example/bank/BankAccount.java
package com.example.bank;          // declares this class's package

public class BankAccount { /* ... */ }
```

```java
// File: com/example/app/Main.java
package com.example.app;

import com.example.bank.BankAccount;   // bring another package's class into scope
import java.util.List;                  // standard library import
import java.util.*;                     // wildcard: all of java.util (not recursive)

public class Main {
    public static void main(String[] args) {
        BankAccount acct = new BankAccount(100);
    }
}
```

`java.lang` (containing `String`, `System`, `Math`, `Object`, etc.) is imported automatically — you never write `import java.lang.*`. Use `import static java.lang.Math.*;` to call `sqrt(x)` instead of `Math.sqrt(x)` (static import — handy in tests).

---

## 5. OOP Advanced: Inheritance, Polymorphism, Interfaces

### 5.1 Inheritance and `super` **[I]**

**Inheritance** lets a class (subclass/child) reuse and extend another (superclass/parent) via `extends`. The child inherits the parent's accessible fields and methods and can add its own. Use it for genuine "is-a" relationships (a `Dog` *is an* `Animal`). The keyword `super` refers to the parent — used to call the parent's constructor or an overridden method.

```java
public class Animal {
    protected String name;        // protected -> subclasses can access
    public Animal(String name) { this.name = name; }
    public String speak() { return "..."; }
}

public class Dog extends Animal {  // Dog IS-A Animal
    private String breed;
    public Dog(String name, String breed) {
        super(name);               // call Animal's constructor FIRST (mandatory if no no-arg ctor)
        this.breed = breed;
    }
    @Override
    public String speak() { return "Woof"; }   // override the parent method
}
```

> **Prefer composition over inheritance** in many cases: instead of `class Stack extends ArrayList`, hold an `ArrayList` as a field. Inheritance couples you tightly to the parent's implementation; composition is more flexible. Inherit only for true "is-a" with a stable contract.

### 5.2 Overriding vs overloading **[I]**

These sound similar but are unrelated:

- **Overriding** — a subclass *replaces* an inherited method with the *same signature* (same name, same parameters). This is runtime polymorphism. Always annotate with `@Override` so the compiler verifies you actually matched a parent method (catching typos that would otherwise silently create a new method).
- **Overloading** — *several methods in the same class* share a name but differ in parameter lists. Resolved at *compile time* by the argument types.

```java
public class Printer {
    // Overloading: same name, different parameters
    void print(String s) { System.out.println(s); }
    void print(int n)    { System.out.println(n); }
    void print(int n, int m) { System.out.println(n + ", " + m); }
}
```

### 5.3 Polymorphism & dynamic dispatch — *how* it works **[I/A]**

**Polymorphism** ("many forms") means a variable of a parent type can hold any subtype object, and calling an overridden method runs the *actual object's* version — decided at *runtime*, not compile time. This runtime decision is **dynamic dispatch** (a.k.a. virtual method invocation).

How does the JVM do it? Each object carries a hidden pointer to its class's **method table** (vtable) — a lookup of which actual method implements each overridable method. When you call `animal.speak()`, the JVM looks at the *runtime* object's vtable (a `Dog`'s table points at `Dog.speak`), not the *declared* type of the variable. This is why one line of code can do different things depending on the concrete object.

```java
Animal a = new Dog("Rex", "Lab");   // declared type Animal, runtime type Dog
System.out.println(a.speak());       // "Woof"  -> Dog's version runs (dynamic dispatch)

// The power: write code against the abstraction, get behaviour of the concrete type.
List<Animal> zoo = List.of(new Dog("Rex", "Lab"), new Animal("thing"));
for (Animal animal : zoo) {
    System.out.println(animal.speak());   // each calls ITS OWN speak()
}
```

Note: *fields* and `static` methods are **not** polymorphic — they bind to the declared (compile-time) type. Only instance methods are dynamically dispatched.

### 5.4 Abstract classes vs interfaces — when to use which **[I]**

Both define contracts subtypes must fulfill, but they answer different questions.

An **abstract class** is a partial class you can't instantiate; it can have fields, constructors, implemented methods, and `abstract` methods (no body) that subclasses must implement. Use it when subtypes share *state and code* and form a clear "is-a" hierarchy. A class can extend only **one** abstract class.

An **interface** is a pure contract: a set of method signatures (plus constants and, since Java 8, default/static methods). It carries no instance state. A class can implement **many** interfaces. Use it to define a *capability* or *role* (`Comparable`, `Runnable`, `Serializable`) that unrelated classes can share.

Rule of thumb: **"is-a" with shared implementation → abstract class; "can-do" capability across unrelated types → interface.** When in doubt, prefer interfaces — they're more flexible (multiple implementation) and the basis of modern Java design.

```java
// Abstract class: shared state + some implemented, some abstract behaviour
public abstract class Shape {
    private final String name;             // state -> needs a class, not interface
    protected Shape(String name) { this.name = name; }
    public String getName() { return name; }    // shared implementation
    public abstract double area();         // each shape MUST define this
}

public class Circle extends Shape {
    private final double r;
    public Circle(double r) { super("circle"); this.r = r; }
    @Override public double area() { return Math.PI * r * r; }
}

// Interface: a capability, no state
public interface Drawable {
    void draw();                           // implicitly public abstract
}

// A class can extend one class AND implement many interfaces:
public class Sprite extends Shape implements Drawable, Comparable<Sprite> {
    public Sprite() { super("sprite"); }
    public double area() { return 0; }
    public void draw() { System.out.println("drawing"); }
    public int compareTo(Sprite o) { return 0; }
}
```

### 5.5 Default, static & private interface methods **[I]**

**⚡ Version note (8+ / 9+):** Interfaces gained the ability to carry implementations:

- **`default` methods** (Java 8) — a method with a body. Implementers inherit it but may override. This was added so the JDK could add methods to existing interfaces (like `List.sort`) *without breaking* every existing implementation — a backward-compatibility escape hatch.
- **`static` methods** (Java 8) — utility methods that belong to the interface itself (`Comparator.comparing(...)`).
- **`private` methods** (Java 9) — helper methods to share code between default methods without exposing them.

```java
public interface Greeter {
    String name();                         // abstract — must implement

    default String greet() {               // default — implementers get this free
        return "Hello, " + capitalize(name());
    }

    private String capitalize(String s) {  // private helper for the default method (9+)
        return s.substring(0, 1).toUpperCase() + s.substring(1);
    }

    static Greeter of(String n) {          // static factory on the interface
        return () -> n;                    // a lambda implementing name()
    }
}
```

### 5.6 `final` **[I]**

`final` means "cannot change," with three uses:
- **`final` variable** — cannot be reassigned (a constant, or a reference that always points at the same object — though the object itself may still mutate).
- **`final` method** — cannot be overridden by subclasses.
- **`final` class** — cannot be extended (e.g. `String` is final; this is partly why it's safely immutable).

```java
final int MAX = 100;          // MAX = 200 -> error
public final class Constants {}  // nobody can subclass
public final void critical() {}  // nobody can override
```

### 5.7 Object methods: equals, hashCode, toString — and the contract **[I/A]**

Every class implicitly extends `java.lang.Object`, inheriting `toString()`, `equals(Object)`, and `hashCode()`. The defaults are based on object *identity* (memory address), which is usually wrong for "value" objects. You override them to give *meaningful* behaviour.

- **`toString()`** — a human-readable representation, used by `System.out.println(obj)` and debuggers. Always override it for your classes; it makes debugging vastly easier.
- **`equals(Object)`** — value equality. Without overriding, `equals` is just `==` (same object).
- **`hashCode()`** — an `int` summarizing the object, used by hash-based collections (`HashMap`, `HashSet`) to bucket objects.

**The equals/hashCode contract** is critical and the source of nasty, silent bugs: *if two objects are `equals`, they MUST have the same `hashCode`.* (The reverse need not hold — collisions are allowed.) If you override `equals` but forget `hashCode`, equal objects can land in different buckets, and a `HashSet` will store "duplicates" or a `HashMap` will fail to find your key. **Always override both together.**

```java
import java.util.Objects;

public class Point {
    private final int x, y;
    public Point(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                  // same reference -> equal
        if (o == null || getClass() != o.getClass()) return false;  // type check
        Point p = (Point) o;
        return x == p.x && y == p.y;                 // value comparison
    }

    @Override
    public int hashCode() {
        return Objects.hash(x, y);                   // combine fields consistently with equals
    }

    @Override
    public String toString() {
        return "Point(" + x + ", " + y + ")";
    }
}
```

Writing all this by hand is tedious and error-prone — which is exactly why **records** (§6) exist; they generate a correct `equals`/`hashCode`/`toString` for you.

### 5.8 Nested, inner & anonymous classes **[I/A]**

Java lets you define classes inside classes. Four flavours:

- **Static nested class** — declared `static` inside another class; a plain class scoped/namespaced inside its host. No link to an instance of the outer class.
- **Inner (non-static nested) class** — each instance is tied to an instance of the outer class and can access its fields. Useful but holds a hidden reference to the outer object (a subtle memory-leak risk).
- **Local class** — a class declared inside a method.
- **Anonymous class** — a one-shot class with no name, defined and instantiated at once, typically to implement an interface inline. Largely superseded by lambdas (§9) for single-method interfaces, but still needed for multi-method ones.

```java
public class Outer {
    private int value = 10;

    static class StaticNested {           // no access to `value`
        void hello() { System.out.println("nested"); }
    }

    class Inner {                          // can access outer's `value`
        void show() { System.out.println(value); }
    }

    void demo() {
        // Anonymous class implementing Runnable inline:
        Runnable r = new Runnable() {
            @Override public void run() { System.out.println("anon"); }
        };
        r.run();
        // Modern equivalent with a lambda (since Runnable is a functional interface):
        Runnable r2 = () -> System.out.println("lambda");
    }
}
```

---

## 6. Modern Data Modeling: Records, Enums, Sealed, Patterns

This is where Java in 2026 feels like a different language than Java 8. These features work together to make data modeling concise and *safe*.

### 6.1 Records — why they exist **[I]**

**⚡ Version note (16+):** A **record** is a concise, immutable "data carrier." The problem it solves: a class that just holds a few values used to require ~40 lines of boilerplate — private final fields, a constructor, getters, `equals`, `hashCode`, `toString` — all of which is mechanical and bug-prone (forget `hashCode` and you get the §5.7 bug). A record generates all of that from a one-line declaration.

You declare the *components* in the header; the compiler synthesizes: private final fields, a canonical constructor, accessor methods (named like the component, e.g. `name()` not `getName()`), and correct `equals`/`hashCode`/`toString`.

```java
// This single line replaces ~40 lines of a hand-written class:
public record Point(int x, int y) {}

Point p = new Point(3, 4);
System.out.println(p.x());        // 3   (accessor is x(), not getX())
System.out.println(p);            // Point[x=3, y=4]   (generated toString)
System.out.println(p.equals(new Point(3, 4)));   // true (value equality, generated)

// Records are immutable: there are no setters; fields are final.

// You can add a "compact constructor" for validation (note: no parameter list):
public record Range(int lo, int hi) {
    public Range {                          // compact canonical constructor
        if (lo > hi) throw new IllegalArgumentException("lo > hi");
        // fields are assigned automatically after this block
    }
    // You can also add normal methods and static factories:
    public int span() { return hi - lo; }
}
```

Use records for DTOs, value objects, map keys, return types bundling several values, and as the carriers in sealed hierarchies (below). Don't use them when you need mutability or a deep inheritance hierarchy (records can't extend a class).

### 6.2 Enums with fields and methods **[B/I]**

An **enum** is a type with a fixed set of named constant instances. The naive view ("just named integers") undersells it: in Java an enum is a full class — each constant is a singleton object, and the enum can have fields, constructors, and methods. This makes enums ideal for modeling a closed set of options that each carry data or behaviour.

```java
public enum Planet {
    // Each constant calls the constructor with its own data:
    MERCURY(3.303e+23, 2.4397e6),
    EARTH(5.976e+24, 6.37814e6);

    private final double mass;       // each constant carries fields
    private final double radius;

    Planet(double mass, double radius) {   // enum constructor (implicitly private)
        this.mass = mass;
        this.radius = radius;
    }

    public double surfaceGravity() {        // behaviour per constant
        return 6.67300E-11 * mass / (radius * radius);
    }
}

Planet p = Planet.EARTH;
System.out.println(p.surfaceGravity());
System.out.println(p.name());           // "EARTH"
System.out.println(p.ordinal());        // 1   (position; avoid relying on it)
for (Planet pl : Planet.values()) { }    // iterate all constants
Planet.valueOf("MERCURY");               // parse from name
```

Enums work beautifully in `switch` (the compiler can check exhaustiveness) and as `EnumSet`/`EnumMap` keys (very efficient).

### 6.3 Sealed classes and interfaces — controlled hierarchies **[I/A]**

**⚡ Version note (17+):** A **sealed** type explicitly lists which classes may extend or implement it via `permits`. The problem it solves: normally an interface can be implemented by *anyone, anywhere*, so the compiler can never know the full set of subtypes — meaning a `switch` over them can never be proven exhaustive, and you always need a `default`. Sealing closes the set, enabling the compiler to verify you've handled every case (and to flag you when a new subtype is added). This models *algebraic data types* — "a Shape is exactly one of Circle, Square, or Triangle."

Each permitted subtype must itself be `final`, `sealed`, or `non-sealed` (explicitly reopening the hierarchy).

```java
public sealed interface Shape permits Circle, Square, Rectangle {}

public record Circle(double radius) implements Shape {}
public record Square(double side) implements Shape {}
public record Rectangle(double w, double h) implements Shape {}

// Because Shape is sealed, this switch is EXHAUSTIVE with no default needed —
// and if someone adds a new Shape, this switch fails to compile until updated.
public static double area(Shape s) {
    return switch (s) {
        case Circle c    -> Math.PI * c.radius() * c.radius();
        case Square sq   -> sq.side() * sq.side();
        case Rectangle r -> r.w() * r.h();
    };
}
```

### 6.4 Pattern matching — instanceof, switch, record deconstruction **[I/A]**

Pattern matching eliminates the verbose, error-prone "check the type, then cast" dance.

**Pattern matching for `instanceof` (16+):** the old way required a redundant cast; the new way binds the variable in one step if the test passes.

```java
Object obj = "hello";

// OLD: test, then cast (cast can be wrong; repeats the type)
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// NEW: type test + binding in one. `s` is in scope only where obj IS a String.
if (obj instanceof String s) {
    System.out.println(s.length());        // s is already a String, no cast
}
// You can even combine with conditions:
if (obj instanceof String s && s.length() > 3) { /* ... */ }
```

**Pattern matching for `switch` (21):** switch on the *type* of a value, binding it. Combined with sealed types, it's exhaustive and elegant. Guards (`when`) add conditions.

```java
static String describe(Object obj) {
    return switch (obj) {
        case null         -> "null";                  // switch can now match null explicitly
        case Integer i when i < 0 -> "negative int";  // guarded pattern with `when`
        case Integer i    -> "int: " + i;
        case String s     -> "string of length " + s.length();
        default           -> "something else";
    };
}
```

**Record patterns / deconstruction (21):** destructure a record directly in the pattern, pulling out its components — and nest deeply.

```java
record Point(int x, int y) {}
record Line(Point start, Point end) {}

static String classify(Object obj) {
    return switch (obj) {
        case Point(int x, int y) -> "point at " + x + "," + y;        // deconstruct
        case Line(Point(var x1, var y1), Point(var x2, var y2)) ->     // NESTED deconstruct
                "line from " + x1 + "," + y1 + " to " + x2 + "," + y2;
        default -> "unknown";
    };
}
```

Together, sealed types + records + switch patterns give you safe, exhaustive, readable "match on shape of data" code — the functional-style data modeling Java long lacked.

---

## 7. Generics

### 7.1 Why generics exist — type safety **[I]**

Before Java 5, collections held `Object`, so you could put *anything* in a list and had to cast everything coming out — with no compile-time guarantee the cast was right. A wrong cast blew up at *runtime* with `ClassCastException`. **Generics** let you parameterize a type with another type (`List<String>`), so the compiler enforces that only `String`s go in and guarantees `String`s come out — moving an entire class of bugs from runtime to compile time, and removing the casts.

```java
// Pre-generics (still legal, still dangerous — the "raw type"):
List bad = new ArrayList();
bad.add("hi");
bad.add(42);                       // compiler allows it!
String s = (String) bad.get(1);    // ClassCastException at RUNTIME

// With generics — the compiler is your safety net:
List<String> good = new ArrayList<>();   // <> = "diamond"; infers String
good.add("hi");
// good.add(42);                   // COMPILE error — caught immediately
String s2 = good.get(0);           // no cast needed
```

### 7.2 Generic classes and methods **[I/A]**

You can write your *own* generic types. `T`, `E`, `K`, `V` are conventional type-parameter names (Type, Element, Key, Value).

```java
// A generic class: Box can hold any type T
public class Box<T> {
    private T value;
    public void set(T value) { this.value = value; }
    public T get() { return value; }
}
Box<String> b = new Box<>();
b.set("hi");
String v = b.get();

// A generic method: the <T> before the return type declares the parameter.
public static <T> T firstOf(List<T> list) {
    return list.get(0);
}
String first = firstOf(List.of("a", "b"));   // T inferred as String
```

### 7.3 Bounded type parameters **[I/A]**

You can constrain a type parameter to a supertype with `extends` (works for both classes and interfaces). This lets you call that type's methods inside the generic code.

```java
// T must be Comparable so we can call compareTo:
public static <T extends Comparable<T>> T max(List<T> list) {
    T best = list.get(0);
    for (T item : list) {
        if (item.compareTo(best) > 0) best = item;
    }
    return best;
}
```

### 7.4 Wildcards and the PECS rule **[I/A]**

A wildcard `?` represents an *unknown* type, used when a method should accept a *family* of parameterizations. The subtlety: `List<Dog>` is **not** a `List<Animal>` (generics are *invariant*), even though `Dog` is an `Animal` — because if it were, you could add a `Cat` to a `List<Dog>` through the `List<Animal>` view. Wildcards reintroduce flexibility safely.

- **`? extends T`** (upper-bounded) — "some unknown subtype of T." You can **read** Ts out, but **can't add** (the compiler doesn't know the exact subtype). Use when you *produce/consume from* the structure (read).
- **`? super T`** (lower-bounded) — "some unknown supertype of T." You can **add** Ts, but reads come out as `Object`. Use when you *write into* the structure.

The mnemonic is **PECS — "Producer Extends, Consumer Super":** if the parameter is a *producer* (you read from it), use `extends`; if it's a *consumer* (you write to it), use `super`.

```java
// Producer: we READ Numbers out of `src` -> extends
static double sum(List<? extends Number> src) {
    double total = 0;
    for (Number n : src) total += n.doubleValue();   // reading is fine
    // src.add(1);   // COMPILE error — can't add to a `? extends`
    return total;
}
sum(List.of(1, 2, 3));        // List<Integer> works
sum(List.of(1.5, 2.5));       // List<Double> works too

// Consumer: we WRITE Integers into `dst` -> super
static void fill(List<? super Integer> dst) {
    dst.add(1);
    dst.add(2);               // writing Integers is fine
}
fill(new ArrayList<Number>());   // accepts List<Number>, List<Object>, List<Integer>
```

### 7.5 Type erasure and its consequences **[A]**

Generics are a *compile-time* feature. After type-checking, the compiler **erases** type parameters — `List<String>` and `List<Integer>` both become plain `List` at runtime (with casts inserted as needed). This was done to preserve backward compatibility with pre-generics bytecode. The consequences trip people up:

```java
// 1) You can't check a parameterized type at runtime:
// if (obj instanceof List<String>)   // COMPILE error — type info is gone
if (obj instanceof List<?>) { }         // OK — can only ask "is it some List?"

// 2) Two methods that differ only by generic type clash — same erasure:
// void f(List<String> x) {}
// void f(List<Integer> x) {}           // COMPILE error — both erase to f(List)

// 3) You can't create a generic array directly:
// T[] arr = new T[10];                 // ERROR — type erased, runtime can't make it

// 4) Generic type info IS retained in some places (fields, method signatures) and
//    can be read via reflection, but instances themselves carry no type argument.
```

---

## 8. The Collections Framework

The Collections Framework (`java.util`) is a set of interfaces and implementations for groups of objects. Understanding the *interface hierarchy* lets you "program to the interface" — declare variables by interface (`List`, `Map`) and choose the implementation (`ArrayList`, `HashMap`) based on performance needs, swapping freely.

### 8.1 The hierarchy **[I]**

```
Iterable
└── Collection
    ├── List   — ordered, indexed, allows duplicates      (ArrayList, LinkedList)
    ├── Set    — no duplicates                              (HashSet, LinkedHashSet, TreeSet)
    └── Queue  — ordered for processing (FIFO/priority)     (ArrayDeque, PriorityQueue, LinkedList)
        └── Deque — double-ended queue (stack + queue)      (ArrayDeque, LinkedList)

Map (NOT a Collection — it's key→value pairs)               (HashMap, LinkedHashMap, TreeMap)
```

### 8.2 List — ArrayList vs LinkedList **[I]**

A `List` is an ordered sequence with index access and duplicates allowed. Two main implementations with different performance profiles:

- **`ArrayList`** — backed by a resizable array. **O(1)** random access (`get(i)`), **O(1)** amortized append, but **O(n)** insert/remove in the middle (must shift elements). **This is your default List 95% of the time.**
- **`LinkedList`** — a doubly-linked list. **O(1)** insert/remove at the ends (and at a known node), but **O(n)** random access (must walk the chain). Rarely worth it in practice — modern CPUs make `ArrayList` faster even for many operations due to cache locality. Use only when you do lots of head/tail insertion (or as a `Deque`).

```java
import java.util.*;

List<String> list = new ArrayList<>();
list.add("a");                 // append
list.add(0, "first");          // insert at index
list.get(1);                   // random access
list.set(1, "b");              // replace
list.remove("a");              // remove by value
list.remove(0);                // remove by index
list.size();
list.contains("b");
list.indexOf("b");

// Iterate:
for (String s : list) System.out.println(s);
list.forEach(System.out::println);     // method reference (§9)
```

### 8.3 Set — HashSet, LinkedHashSet, TreeSet **[I]**

A `Set` holds unique elements (relies on `equals`/`hashCode`!).

- **`HashSet`** — fastest, **no order**, O(1) add/contains.
- **`LinkedHashSet`** — preserves *insertion* order, slightly slower.
- **`TreeSet`** — keeps elements *sorted* (by natural order or a `Comparator`), O(log n) operations.

```java
Set<String> seen = new HashSet<>();
seen.add("a"); seen.add("a");          // second add is a no-op
System.out.println(seen.size());        // 1
seen.contains("a");                     // true, O(1)

Set<Integer> sorted = new TreeSet<>(List.of(3, 1, 2));   // -> {1, 2, 3} iteration order
```

### 8.4 Map — HashMap vs TreeMap vs LinkedHashMap **[I]**

A `Map` stores key→value pairs with unique keys. Not a `Collection`.

- **`HashMap`** — O(1) get/put, **no order**. Default choice.
- **`LinkedHashMap`** — preserves insertion order (or access order — useful for LRU caches).
- **`TreeMap`** — keys kept *sorted*, O(log n), supports range queries (`firstKey`, `headMap`, `ceilingKey`).

```java
Map<String, Integer> ages = new HashMap<>();
ages.put("Ada", 36);
ages.put("Ada", 37);                    // overwrites
ages.get("Ada");                        // 37
ages.getOrDefault("Bob", 0);            // 0  (no NPE for missing keys)
ages.containsKey("Ada");
ages.remove("Ada");

// Handy modern methods:
ages.putIfAbsent("Ada", 36);
ages.computeIfAbsent("Bob", k -> 0);    // compute+store if missing
ages.merge("Ada", 1, Integer::sum);     // add 1 to Ada's value (or set to 1)

// Iterate entries:
for (Map.Entry<String, Integer> e : ages.entrySet()) {
    System.out.println(e.getKey() + " = " + e.getValue());
}
ages.forEach((k, v) -> System.out.println(k + " = " + v));
```

### 8.5 Queue & Deque **[I]**

A `Queue` processes elements in an order (usually FIFO). A `Deque` ("deck") is double-ended — it serves as both a **stack** (LIFO) and a **queue** (FIFO). **Prefer `ArrayDeque` over the legacy `Stack` class** (which is synchronized and built on `Vector`).

```java
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1); stack.push(2);          // LIFO
stack.pop();                            // 2

Deque<Integer> queue = new ArrayDeque<>();
queue.offer(1); queue.offer(2);        // add to tail
queue.poll();                           // 1  (remove from head, FIFO)

// PriorityQueue: elements come out in priority (sorted) order, not insertion order.
Queue<Integer> pq = new PriorityQueue<>();
pq.offer(3); pq.offer(1); pq.offer(2);
pq.poll();                              // 1  (smallest first)
```

### 8.6 Comparable vs Comparator **[I]**

To sort objects, Java needs to know their order. Two mechanisms:

- **`Comparable`** — the class defines its *one natural ordering* by implementing `compareTo`. `compareTo` returns negative/zero/positive for less/equal/greater.
- **`Comparator`** — an *external*, swappable ordering object. Use it when you want multiple orderings or can't modify the class.

```java
// Natural ordering via Comparable:
public class Person implements Comparable<Person> {
    String name; int age;
    public int compareTo(Person o) { return Integer.compare(this.age, o.age); }
}

// External, composable orderings via Comparator (the modern, fluent way):
List<Person> people = new ArrayList<>(/* ... */);
people.sort(Comparator.comparing((Person p) -> p.name));         // by name
people.sort(Comparator.comparingInt((Person p) -> p.age)
                      .reversed());                               // by age desc
people.sort(Comparator.comparing((Person p) -> p.age)
                      .thenComparing(p -> p.name));               // tie-break by name
```

### 8.7 Factory methods **[I]**

**⚡ Version note (9+):** `List.of`, `Set.of`, `Map.of` create compact, **immutable** collections in one line. Note these are unmodifiable — calling `add`/`put` throws `UnsupportedOperationException`. For a mutable copy, wrap: `new ArrayList<>(List.of(...))`.

```java
List<String> l = List.of("a", "b", "c");          // immutable
Set<Integer> s = Set.of(1, 2, 3);                  // immutable
Map<String, Integer> m = Map.of("a", 1, "b", 2);   // immutable
```

---

## 9. Functional Java: Lambdas, Streams, Optional

**⚡ Version note (8+):** Java 8 added a functional programming layer that transformed everyday Java. It's now idiomatic and essential.

### 9.1 Lambdas — what they compile to **[I]**

A **lambda** is a compact anonymous function: `(params) -> body`. The key insight: a lambda is an instance of a **functional interface** — any interface with exactly *one* abstract method (a "SAM" — Single Abstract Method type). The lambda *is* the implementation of that one method. The compiler infers which interface from context. (Under the hood lambdas compile to `invokedynamic` bytecode, not anonymous classes — they're cheaper and the JVM can optimize them.)

```java
// These two are equivalent — lambda vs the anonymous class it replaces:
Runnable r1 = () -> System.out.println("hi");
Runnable r2 = new Runnable() {
    public void run() { System.out.println("hi"); }
};

// Lambda forms:
Runnable noArgs = () -> System.out.println("x");
Function<Integer, Integer> square = n -> n * n;          // single param, no parens needed
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
Comparator<String> byLen = (a, b) -> {                    // block body
    return Integer.compare(a.length(), b.length());
};
```

### 9.2 Built-in functional interfaces **[I]**

`java.util.function` provides the common shapes so you rarely define your own:

| Interface | Abstract method | Meaning |
|-----------|----------------|---------|
| `Supplier<T>` | `T get()` | produces a value, takes none |
| `Consumer<T>` | `void accept(T)` | consumes a value, returns none |
| `Function<T,R>` | `R apply(T)` | transforms T → R |
| `Predicate<T>` | `boolean test(T)` | a boolean test |
| `BiFunction<T,U,R>` | `R apply(T,U)` | two args → R |
| `UnaryOperator<T>` | `T apply(T)` | T → T |

```java
Predicate<Integer> isEven = n -> n % 2 == 0;
isEven.test(4);                            // true
Function<String, Integer> len = String::length;   // method reference (below)
Supplier<Double> rnd = Math::random;
Consumer<String> printer = System.out::println;
```

### 9.3 Method references **[I]**

When a lambda just calls an existing method, a **method reference** (`::`) is shorter and clearer. Four kinds: static (`Math::abs`), bound instance (`System.out::println`), unbound instance (`String::length`), and constructor (`ArrayList::new`).

```java
list.forEach(System.out::println);        // x -> System.out.println(x)
list.stream().map(String::toUpperCase);   // s -> s.toUpperCase()
list.sort(String::compareTo);             // (a,b) -> a.compareTo(b)
Supplier<List<String>> maker = ArrayList::new;   // () -> new ArrayList<>()
```

### 9.4 The Streams API — lazy evaluation explained **[I/A]**

A **Stream** is a pipeline for processing a sequence of elements declaratively. You describe *what* to compute (filter, map, reduce) rather than *how* to loop. Critically, streams are **lazy**: intermediate operations (`filter`, `map`, `sorted`) don't run when you call them — they just *build up* a recipe. Nothing happens until a **terminal operation** (`collect`, `forEach`, `reduce`, `count`) triggers the whole pipeline. This laziness enables fusion (multiple operations run in a single pass) and short-circuiting (e.g. `findFirst` stops as soon as it has an answer).

A stream has three parts: a **source** (collection, array, range), zero or more **intermediate operations** (return a new stream — lazy), and one **terminal operation** (produces a result or side effect — eager). Streams are single-use: once consumed, you can't reuse them.

```java
import java.util.stream.*;

List<String> names = List.of("Ada", "Linus", "Grace", "Alan");

// A pipeline: source -> filter -> map -> collect (terminal)
List<String> result = names.stream()                  // source
    .filter(n -> n.length() > 3)                       // intermediate (lazy)
    .map(String::toUpperCase)                          // intermediate (lazy)
    .sorted()                                          // intermediate (lazy)
    .collect(Collectors.toList());                     // TERMINAL -> runs everything now
// -> [ALAN, GRACE, LINUS]
```

### 9.5 map, filter, reduce, collect **[I/A]**

```java
// filter: keep elements matching a predicate
Stream.of(1, 2, 3, 4).filter(n -> n % 2 == 0);        // 2, 4

// map: transform each element
Stream.of("a", "bb").map(String::length);              // 1, 2

// reduce: fold the stream into one value
int sum = Stream.of(1, 2, 3, 4).reduce(0, Integer::sum);   // 10
Optional<Integer> max = Stream.of(1, 5, 3).reduce(Integer::max);

// Numeric streams avoid boxing and add sum/average/stats:
int total = IntStream.rangeClosed(1, 100).sum();       // 5050
IntStream.range(0, 5).forEach(System.out::println);    // 0..4

// collect into various shapes:
List<Integer> list  = Stream.of(1, 2, 3).collect(Collectors.toList());
Set<Integer> set    = Stream.of(1, 1, 2).collect(Collectors.toSet());
String joined       = Stream.of("a", "b").collect(Collectors.joining(", "));  // "a, b"
// toList() (16+) is a shorter immutable terminal:
List<Integer> l2    = Stream.of(1, 2, 3).toList();
```

### 9.6 Collectors: grouping, partitioning, toMap **[I/A]**

`Collectors` provides powerful aggregation, especially `groupingBy` (SQL-style GROUP BY).

```java
record Person(String name, String city, int age) {}
List<Person> people = List.of(
    new Person("Ada", "London", 36),
    new Person("Alan", "London", 41),
    new Person("Grace", "NYC", 45));

// Group by city -> Map<String, List<Person>>
Map<String, List<Person>> byCity =
    people.stream().collect(Collectors.groupingBy(Person::city));

// Group and count:
Map<String, Long> countByCity =
    people.stream().collect(Collectors.groupingBy(Person::city, Collectors.counting()));

// Group and average a field:
Map<String, Double> avgAgeByCity =
    people.stream().collect(Collectors.groupingBy(
        Person::city, Collectors.averagingInt(Person::age)));

// Partition by a predicate -> Map<Boolean, List<Person>>
Map<Boolean, List<Person>> overForty =
    people.stream().collect(Collectors.partitioningBy(p -> p.age() > 40));

// toMap: build a Map (provide a merge function if keys can collide)
Map<String, Integer> nameToAge =
    people.stream().collect(Collectors.toMap(Person::name, Person::age));
```

### 9.7 flatMap **[I/A]**

`flatMap` flattens nested structure: each element maps to a *stream*, and all those streams are concatenated into one. Use it to flatten lists-of-lists or expand each element into many.

```java
List<List<Integer>> nested = List.of(List.of(1, 2), List.of(3, 4));
List<Integer> flat = nested.stream()
    .flatMap(List::stream)              // each inner list -> its elements
    .toList();                           // [1, 2, 3, 4]

// Split sentences into words:
List<String> words = Stream.of("hello world", "foo bar")
    .flatMap(s -> Arrays.stream(s.split(" ")))
    .toList();                           // [hello, world, foo, bar]
```

### 9.8 Parallel streams **[A]**

`.parallelStream()` (or `.parallel()`) splits work across the common `ForkJoinPool` (CPU cores). It can speed up *CPU-bound*, *large*, *stateless* pipelines — but it has real costs (splitting/merging overhead, thread coordination) and traps (don't use it with shared mutable state, ordering-sensitive logic, or small datasets). Measure before trusting it; for most workloads sequential is fine.

```java
long count = IntStream.rangeClosed(1, 1_000_000)
    .parallel()
    .filter(n -> isPrime(n))            // a CPU-heavy, independent operation
    .count();
// Rule: only parallelize big, CPU-bound, independent, side-effect-free work.
```

### 9.9 Optional — the null-avoidance rationale **[I/A]**

`Optional<T>` is a container that holds either a value or "nothing," designed to make absence *explicit in the type*. The problem it addresses: returning `null` to mean "no result" is invisible in the signature, so callers forget to check and get the dreaded `NullPointerException` ("the billion-dollar mistake"). A method returning `Optional<User>` *forces* the caller to acknowledge the empty case.

Use `Optional` primarily as a **return type** for "might not find it" methods. Do **not** use it for fields or parameters, and don't call `.get()` blindly (that just reintroduces the NPE). Use the functional methods instead.

```java
import java.util.Optional;

Optional<String> found = Optional.of("hi");      // wrap a present value
Optional<String> empty = Optional.empty();        // explicit absence
Optional<String> maybe = Optional.ofNullable(getMaybeNull());  // null -> empty

// Consume safely WITHOUT ever touching a raw value:
maybe.ifPresent(s -> System.out.println(s));       // run only if present
String val = maybe.orElse("default");              // fallback value
String val2 = maybe.orElseGet(() -> compute());    // lazy fallback (only called if empty)
String val3 = maybe.orElseThrow(() ->              // throw if empty
        new NoSuchElementException("not found"));
int len = maybe.map(String::length).orElse(0);     // transform inside the Optional

// A method that signals "may not exist" honestly:
public Optional<User> findUser(String id) {
    User u = lookup(id);
    return Optional.ofNullable(u);
}
```

---

## 10. Exceptions & Error Handling

### 10.1 The exception hierarchy & checked vs unchecked **[I]**

When something goes wrong, Java *throws* an exception object that propagates up the call stack until caught. All exceptions descend from `Throwable`, which splits into:

- **`Error`** — serious JVM problems you shouldn't catch (`OutOfMemoryError`, `StackOverflowError`).
- **`Exception`** — application problems, split again into:
  - **Checked exceptions** (subclasses of `Exception` but not `RuntimeException`) — the compiler *forces* you to handle or declare them (`IOException`, `SQLException`). The intent: recoverable conditions the caller should consciously deal with.
  - **Unchecked exceptions** (`RuntimeException` and subclasses) — *not* enforced by the compiler (`NullPointerException`, `IllegalArgumentException`, `IndexOutOfBoundsException`). The intent: programming bugs that shouldn't happen if the code is correct.

**The design debate:** checked exceptions are unique to Java and controversial. Proponents say they document failure modes and force handling. Critics (and most modern frameworks like Spring) argue they cause boilerplate, leak implementation details, and get "swallowed" with empty catch blocks — and so prefer unchecked exceptions. Pragmatic guidance: use unchecked for programming errors and most application logic; reserve checked exceptions for genuinely recoverable, expected conditions where you want to force the caller's hand. Never swallow an exception silently.

### 10.2 try / catch / finally **[I]**

```java
try {
    int x = Integer.parseInt("not a number");   // throws NumberFormatException
} catch (NumberFormatException e) {
    System.out.println("bad number: " + e.getMessage());
} finally {
    // ALWAYS runs — whether or not an exception occurred — for cleanup.
    System.out.println("done");
}
```

### 10.3 Multi-catch **[I]**

Handle several exception types with one block using `|` — reduces duplication.

```java
try {
    riskyOperation();
} catch (IOException | SQLException e) {     // one block for both
    log.error("operation failed", e);
    throw new RuntimeException(e);            // wrap & rethrow if you can't recover
}
```

### 10.4 try-with-resources & AutoCloseable **[I]**

Anything that must be *closed* (files, sockets, DB connections) should be opened in a **try-with-resources** block. Any resource implementing `AutoCloseable` declared in the parentheses is **automatically closed** at the end — even if an exception is thrown — in reverse order of opening. This replaces the error-prone `finally { stream.close(); }` dance (which itself could throw or NPE).

```java
import java.io.*;

// Resources declared in parentheses are auto-closed, even on exception:
try (BufferedReader br = new BufferedReader(new FileReader("data.txt"));
     PrintWriter pw = new PrintWriter("out.txt")) {       // multiple resources
    String line;
    while ((line = br.readLine()) != null) {
        pw.println(line.toUpperCase());
    }
}   // br.close() and pw.close() called automatically here
catch (IOException e) {
    System.err.println("I/O failed: " + e.getMessage());
}

// Make your own class auto-closeable:
class Resource implements AutoCloseable {
    public Resource() { System.out.println("open"); }
    @Override public void close() { System.out.println("close"); }
}
```

### 10.5 Custom exceptions **[I]**

Define domain-specific exceptions by extending `RuntimeException` (unchecked, usually preferred) or `Exception` (checked). Always provide a constructor that accepts a *cause* so you preserve the stack trace when wrapping.

```java
public class InsufficientFundsException extends RuntimeException {
    private final double shortfall;
    public InsufficientFundsException(double shortfall) {
        super("Short by " + shortfall);   // message
        this.shortfall = shortfall;
    }
    public InsufficientFundsException(String msg, Throwable cause) {
        super(msg, cause);                // preserve the original cause (chaining)
    }
    public double getShortfall() { return shortfall; }
}

// Throwing:
if (amount > balance) {
    throw new InsufficientFundsException(amount - balance);
}
```

### 10.6 Best practices **[I/A]**

- Catch the **most specific** exception you can handle; don't `catch (Exception e)` broadly.
- **Never** swallow exceptions with an empty `catch {}`. At minimum log them.
- When wrapping and rethrowing, pass the original as the **cause** (`new XException(msg, e)`) so the stack trace survives.
- Don't use exceptions for ordinary control flow — they're expensive (capturing a stack trace) and obscure intent.
- Validate inputs early and throw `IllegalArgumentException`/`IllegalStateException` for contract violations.

---

## 11. File System, OS Info & Command Execution

This is a heavily-used, practical area. Java has two file APIs: the modern **`java.nio.file`** (NIO.2, Java 7+) and the older **`java.io`**. Learn NIO.2 as your default; understand `java.io` streams for byte-level work and legacy code.

### 11.1 Why two APIs, and when to use each **[I]**

`java.io` (1996-era) centers on `File` and *streams*. `File` has clumsy error handling (methods return `false` on failure with no reason), weak symlink/metadata support, and no good directory-walking. `java.nio.file` (Java 7) introduced `Path` (an abstract, manipulable path), `Files` (a toolbox of static operations), proper exceptions, symbolic-link awareness, file attributes, directory streams, and a file-tree walker. **Use `java.nio.file` for almost everything.** Reach for `java.io` streams when you process raw bytes/characters incrementally (large files you can't load into memory) or interface with stream-based APIs.

### 11.2 Path and Files — the modern API **[I]**

`Path` represents a location (it doesn't touch the disk by itself). `Files` performs the actual operations.

```java
import java.nio.file.*;
import java.io.IOException;
import java.util.List;

Path p = Path.of("data", "input.txt");      // platform-independent: data/input.txt or data\input.txt
Path abs = p.toAbsolutePath();
Path home = Path.of(System.getProperty("user.home"));
Path resolved = home.resolve("notes.txt");   // join paths safely
System.out.println(p.getFileName());          // input.txt
System.out.println(p.getParent());            // data

// --- Reading ---
String text = Files.readString(p);                       // whole file -> String (11+)
List<String> lines = Files.readAllLines(p);              // each line -> List<String>
byte[] bytes = Files.readAllBytes(p);                    // raw bytes
// Stream lines lazily (good for large files — doesn't load all into memory):
try (var stream = Files.lines(p)) {
    stream.filter(l -> l.contains("ERROR")).forEach(System.out::println);
}

// --- Writing ---
Files.writeString(p, "hello\nworld");                    // write/overwrite a String (11+)
Files.write(p, lines);                                    // write a List<String>
Files.writeString(p, "appended\n", StandardOpenOption.CREATE, StandardOpenOption.APPEND);

// --- Directories & existence ---
Files.createDirectories(Path.of("a", "b", "c"));         // mkdir -p (makes all parents)
Files.exists(p);
Files.isDirectory(p);
Files.isRegularFile(p);
Files.size(p);                                            // bytes

// --- Copy / move / delete ---
Files.copy(p, Path.of("backup.txt"), StandardCopyOption.REPLACE_EXISTING);
Files.move(p, Path.of("renamed.txt"), StandardCopyOption.REPLACE_EXISTING);
Files.delete(p);                                          // throws if missing
Files.deleteIfExists(p);                                  // no throw if missing
```

### 11.3 Listing and walking directories **[I]**

```java
// list(): immediate children only (one level). Use try-with-resources — it's a stream over an OS handle.
try (var entries = Files.list(Path.of("."))) {
    entries.forEach(System.out::println);
}

// walk(): recurse the whole tree (optionally limit depth)
try (var tree = Files.walk(Path.of("src"))) {
    tree.filter(Files::isRegularFile)
        .filter(f -> f.toString().endsWith(".java"))
        .forEach(System.out::println);
}

// find(): walk + filter with a BiPredicate over path & attributes
try (var found = Files.find(Path.of("."), 5,
        (path, attrs) -> attrs.isRegularFile() && attrs.size() > 1_000_000)) {
    found.forEach(System.out::println);   // files over 1 MB, max depth 5
}
```

### 11.4 The older java.io streams **[I]**

Streams process data *incrementally*: byte streams (`InputStream`/`OutputStream`) for binary, character streams (`Reader`/`Writer`) for text. Wrap them in **buffered** variants to avoid a syscall per byte (huge performance difference). Use these for large files, network sockets, or any stream-oriented source.

```java
import java.io.*;

// Buffered character reading (line by line) — small memory footprint for huge files:
try (BufferedReader br = new BufferedReader(new FileReader("big.log"))) {
    String line;
    while ((line = br.readLine()) != null) {
        process(line);
    }
}

// Buffered writing:
try (BufferedWriter bw = new BufferedWriter(new FileWriter("out.txt"))) {
    bw.write("line one");
    bw.newLine();
    bw.write("line two");
}

// Raw byte copy with a buffer:
try (InputStream in = new BufferedInputStream(new FileInputStream("a.bin"));
     OutputStream out = new BufferedOutputStream(new FileOutputStream("b.bin"))) {
    byte[] buf = new byte[8192];
    int n;
    while ((n = in.read(buf)) != -1) {
        out.write(buf, 0, n);
    }
}
```

### 11.5 OS & system information **[I]**

```java
// System properties: facts about the JVM/OS/user (the "os.name" trick for OS detection):
System.getProperty("os.name");          // "Windows 11", "Linux", "Mac OS X"
System.getProperty("os.arch");          // "amd64", "aarch64"
System.getProperty("os.version");
System.getProperty("user.name");
System.getProperty("user.home");        // home directory
System.getProperty("user.dir");         // current working directory
System.getProperty("java.version");     // "21.0.2"
System.getProperty("file.separator");   // "\\" on Windows, "/" on Unix
System.getProperty("line.separator");   // platform newline
System.lineSeparator();                 // the same, cleaner

// Cross-platform OS check:
boolean isWindows = System.getProperty("os.name").toLowerCase().startsWith("windows");

// Environment variables (set OUTSIDE the program):
System.getenv("PATH");
System.getenv("HOME");                  // null on Windows; use "USERPROFILE" there
System.getenv().forEach((k, v) -> System.out.println(k + "=" + v));

// Runtime: memory & CPU info
Runtime rt = Runtime.getRuntime();
rt.availableProcessors();               // logical CPU cores — useful for thread pools
rt.totalMemory();                       // current heap size (bytes)
rt.maxMemory();                         // max heap the JVM will use
rt.freeMemory();                        // free within current heap

// ProcessHandle: info about THIS process and others (9+)
ProcessHandle current = ProcessHandle.current();
System.out.println(current.pid());                       // this PID
current.info().command().ifPresent(System.out::println); // the launching command
ProcessHandle.allProcesses()                             // every visible process
    .filter(ph -> ph.info().command().isPresent())
    .limit(5)
    .forEach(ph -> System.out.println(ph.pid()));
```

### 11.6 Executing external commands with ProcessBuilder **[I/A]**

To run another program (git, npm, python, ls), use **`ProcessBuilder`**. Understand the model first: when you launch a process, the OS creates a *child* with its own memory, its own working directory, its own environment, and three standard streams — **stdin** (input), **stdout** (normal output), **stderr** (errors). Your Java program is the *parent*; you wire up those streams to send input and capture output, and you can wait for the child to finish and read its **exit code** (0 = success by convention).

A crucial subtlety: **set the working directory via `.directory(...)`, do not run `cd`.** `cd` is a shell builtin, not a program — and unless you explicitly launch a shell, there's no shell to interpret it. Even with a shell, `cd` in one command wouldn't persist. `ProcessBuilder.directory()` sets the child's starting directory directly and reliably, cross-platform.

Another: `ProcessBuilder` does **not** use a shell, so it doesn't do glob expansion, pipes, or `&&`. You pass an explicit argument list (which also avoids shell-injection bugs). If you genuinely need shell features, invoke the shell explicitly: `new ProcessBuilder("bash", "-c", "ls *.txt | wc -l")` (or `cmd /c` on Windows).

```java
import java.io.*;
import java.util.concurrent.TimeUnit;

// Run `git status` in a specific directory and capture its output:
ProcessBuilder pb = new ProcessBuilder("git", "status", "--short");
pb.directory(new File("C:/projects/myrepo"));   // working dir — NOT `cd`
pb.redirectErrorStream(true);                    // merge stderr into stdout

Process process = pb.start();                    // launch the child

// Read its stdout (must consume it, or the child can block on a full pipe buffer):
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(process.getInputStream()))) {
    String line;
    while ((line = reader.readLine()) != null) {
        System.out.println(line);
    }
}

boolean finished = process.waitFor(30, TimeUnit.SECONDS);   // wait with a timeout
if (!finished) {
    process.destroyForcibly();                   // kill if it hung
} else {
    int exitCode = process.exitValue();          // 0 = success
    System.out.println("git exited with " + exitCode);
}
```

Keeping stdout and stderr separate, and setting environment variables and working directory:

```java
ProcessBuilder pb = new ProcessBuilder("python", "script.py", "--verbose");
pb.directory(new File("/work/scripts"));

// Modify the child's environment (inherits the parent's by default):
java.util.Map<String, String> env = pb.environment();
env.put("API_KEY", "secret");
env.put("PYTHONUNBUFFERED", "1");

// Redirect streams to files instead of capturing in-process:
pb.redirectOutput(new File("out.log"));
pb.redirectError(new File("err.log"));
// Or inherit the parent's console directly:
// pb.inheritIO();

Process proc = pb.start();
int code = proc.waitFor();   // blocks until done, returns exit code
```

Common real-world commands (note the explicit, shell-free argument lists):

```java
new ProcessBuilder("npm", "install").directory(new File("./frontend")).inheritIO().start();
new ProcessBuilder("git", "clone", "https://example.com/r.git").start();
// List a directory cross-platform — choose the right native command:
List<String> cmd = isWindows
    ? List.of("cmd", "/c", "dir")
    : List.of("ls", "-la");
new ProcessBuilder(cmd).inheritIO().start().waitFor();
```

### 11.7 Command-line arguments & reading input **[I]**

`String[] args` in `main` holds the command-line arguments (already split by the shell, no program name at index 0 — unlike C).

```java
public static void main(String[] args) {
    // java App.java hello 42  ->  args = ["hello", "42"]
    if (args.length < 2) {
        System.err.println("usage: App <name> <count>");
        System.exit(1);              // nonzero exit code = failure
    }
    String name = args[0];
    int count = Integer.parseInt(args[1]);   // parse strings to numbers
}
```

Reading interactive input — `Scanner` is convenient; `BufferedReader` is faster for bulk input:

```java
import java.util.Scanner;

Scanner sc = new Scanner(System.in);
System.out.print("Enter your name: ");
String name = sc.nextLine();                 // a whole line
System.out.print("Enter age: ");
int age = sc.nextInt();                       // a token parsed as int
sc.close();

// Faster bulk reading:
import java.io.*;
BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
String line = br.readLine();
```

> **picocli** is the de-facto library for real CLIs (subcommands, typed options, `--help`, auto-completion) via annotations. For anything beyond a couple of positional args, reach for it instead of hand-parsing `args`.

```java
// Sketch of picocli usage (requires the dependency):
// @Command(name = "greet", mixinStandardHelpOptions = true)
// class Greet implements Runnable {
//     @Option(names = "--name") String name;
//     @Parameters int count;
//     public void run() { for (int i=0;i<count;i++) System.out.println("Hi " + name); }
// }
// public static void main(String[] a){ System.exit(new CommandLine(new Greet()).execute(a)); }
```

---

## 12. Concurrency & Parallelism

Concurrency lets a program make progress on multiple tasks. It's one of Java's deepest topics; this section builds from threads up to virtual threads and structured concurrency.

### 12.1 Threads, Runnable, Callable **[I/A]**

A **thread** is an independent path of execution within your process. Define the work as a `Runnable` (returns nothing) or `Callable` (returns a value / may throw). Creating threads manually is educational but you'll almost always use an `ExecutorService` (§12.4) in real code.

```java
// Runnable via lambda:
Thread t = new Thread(() -> System.out.println("running in " + Thread.currentThread().getName()));
t.start();          // start() spawns a new thread and calls run(); DON'T call run() directly
t.join();           // wait for t to finish

// Callable returns a value (used with executors):
import java.util.concurrent.Callable;
Callable<Integer> task = () -> { Thread.sleep(100); return 42; };
```

### 12.2 The memory-visibility problem **[A]**

Here is the heart of why concurrency is hard. Threads may run on different CPU cores, each with its own caches, and both the compiler and CPU may *reorder* instructions for speed. The consequence: a write by one thread is **not guaranteed to be visible** to another thread *when* and *in the order* you'd expect, unless you use a synchronization mechanism. This isn't theoretical — the classic "loop that never sees the flag set" bug:

```java
boolean running = true;          // BUG: another thread setting this may NEVER be seen
while (running) { }              // the JIT may cache `running` in a register forever
```

Java's solutions, each establishing a "happens-before" relationship that forces visibility and ordering:

- **`volatile`** — guarantees reads/writes of a field go to/from main memory and aren't reordered. Use for simple flags. Does **not** make compound operations (`count++`) atomic.
- **`synchronized`** — a mutual-exclusion lock. Only one thread holds a given object's monitor at a time; entering/exiting also flushes memory. Use to protect a *critical section* (a block touching shared mutable state).
- **`java.util.concurrent.locks.Lock`** — explicit locks with more control (try-lock, timed, fair, read/write).

```java
// volatile flag — the loop now reliably sees the change:
private volatile boolean running = true;

// synchronized makes compound updates atomic AND visible:
private int count = 0;
public synchronized void increment() { count++; }   // method-level lock on `this`

public void blockLevel() {
    synchronized (lockObject) {                       // block-level: lock a specific object
        // critical section — only one thread at a time
    }
}
```

### 12.3 Atomics & concurrent collections **[A]**

For single-variable updates, **atomic classes** (`AtomicInteger`, `AtomicLong`, `AtomicReference`) provide lock-free, thread-safe operations using CPU compare-and-swap — faster than `synchronized` for counters. For shared maps/queues, use the **concurrent collections** instead of wrapping a `HashMap` in locks.

```java
import java.util.concurrent.atomic.AtomicInteger;
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();          // atomic ++  (thread-safe, lock-free)
counter.addAndGet(5);

import java.util.concurrent.*;
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.merge("hits", 1, Integer::sum);   // atomic update of one key
BlockingQueue<String> queue = new LinkedBlockingQueue<>();  // producer/consumer-safe
```

### 12.4 ExecutorService & Future **[A]**

Manually creating threads doesn't scale (threads are expensive; unbounded creation crashes). An **`ExecutorService`** is a managed *thread pool*: you submit tasks, it reuses a bounded set of threads. `submit` returns a **`Future`** — a handle to a result that may not exist yet; call `.get()` to block until it's ready.

```java
import java.util.concurrent.*;

ExecutorService pool = Executors.newFixedThreadPool(4);   // reuse 4 threads

Future<Integer> future = pool.submit(() -> {
    Thread.sleep(100);
    return 42;
});
Integer result = future.get();        // blocks until the task completes -> 42

// Submit many, gather results:
List<Future<Integer>> futures = pool.invokeAll(List.of(
    () -> 1, () -> 2, () -> 3));

pool.shutdown();                       // stop accepting new tasks; let running ones finish
pool.awaitTermination(10, TimeUnit.SECONDS);
```

### 12.5 CompletableFuture **[A]**

`CompletableFuture` is for *asynchronous pipelines*: chain non-blocking steps, combine results, and handle errors without blocking threads. It's Java's answer to promises/futures with composition.

```java
import java.util.concurrent.CompletableFuture;

CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> fetchUser(42))         // run async, produce a value
    .thenApply(user -> user.toUpperCase())     // transform the result (still async)
    .thenCompose(name -> CompletableFuture     // chain another async call
        .supplyAsync(() -> lookupProfile(name)))
    .exceptionally(ex -> "fallback");          // handle any failure in the chain

String value = cf.join();                       // get the final result (unchecked get)

// Combine two independent async results:
CompletableFuture<Integer> a = CompletableFuture.supplyAsync(() -> 10);
CompletableFuture<Integer> b = CompletableFuture.supplyAsync(() -> 20);
a.thenCombine(b, Integer::sum).thenAccept(System.out::println);   // 30
```

### 12.6 Virtual threads — what problem they solve **[A]**

**⚡ Version note (21):** **Virtual threads** (Project Loom) are the biggest concurrency change in Java's history. The problem: traditional ("platform") threads map 1:1 to OS threads, which are *heavy* — each consumes ~1 MB of stack and the OS struggles past a few thousand. So servers handling many concurrent connections couldn't just "use a thread per request"; they had to write complex asynchronous/reactive code (callbacks, `CompletableFuture` chains) that's hard to read and debug.

Virtual threads are *lightweight* threads managed by the JVM, not the OS. You can have **millions**. When a virtual thread blocks on I/O, the JVM *unmounts* it from its carrier OS thread (which goes off to run another virtual thread) and remounts it when the I/O completes — all transparently. The result: you write **simple, blocking, thread-per-task code** and get the scalability of async code, with readable stack traces. This essentially obsoletes reactive frameworks for most I/O-bound server work.

```java
// Just write blocking code — but spawn a virtual thread per task. Scales to millions.
Thread.startVirtualThread(() -> {
    var response = blockingHttpCall();   // blocking is FINE — JVM unmounts the vthread
    process(response);
});

// An executor that creates a NEW virtual thread per task (the idiomatic server pattern):
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1_000_000; i++) {        // a million concurrent tasks — no problem
        executor.submit(() -> {
            Thread.sleep(1000);                   // blocks this vthread only, not an OS thread
            return null;
        });
    }
}   // try-with-resources waits for all tasks
```

### 12.7 Structured concurrency **[A]**

**⚡ Version note (preview through 24, stabilizing in 25):** Structured concurrency treats a group of concurrent subtasks as a single unit of work with a clear lifetime — mirroring how structured *control flow* (blocks, loops) tamed `goto`. The problem it solves: with raw executors, if you fork three subtasks and one fails, the others keep running (leaked work), error handling and cancellation are manual and bug-prone, and the relationship between tasks is invisible. `StructuredTaskScope` scopes subtasks to a block: they all start inside it, the scope waits for them, and if one fails the rest are *automatically cancelled*. Errors and cancellation compose cleanly.

```java
import java.util.concurrent.StructuredTaskScope;   // API package stabilized in 25

// Fork two subtasks; if EITHER fails, the other is cancelled; we join both.
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var userTask    = scope.fork(() -> fetchUser(id));      // runs concurrently
    var ordersTask  = scope.fork(() -> fetchOrders(id));    // runs concurrently

    scope.join();             // wait for both to finish (or one to fail)
    scope.throwIfFailed();    // propagate the first failure, cancelling the rest

    // Both succeeded — safe to read results:
    var result = combine(userTask.get(), ordersTask.get());
}
```

---

## 13. Standard Library Utilities

### 13.1 java.time — the modern date/time API **[I]**

**⚡ Version note (8+):** The old `Date`/`Calendar` classes are mutable, confusing (months 0-indexed!), and not thread-safe. `java.time` (JSR-310) replaced them with an *immutable*, well-designed API. Use it exclusively. Key types: `LocalDate` (date only), `LocalTime`, `LocalDateTime` (no zone), `ZonedDateTime` (with zone), `Instant` (a machine timestamp — UTC), `Duration` (time-based amount), `Period` (date-based amount).

```java
import java.time.*;
import java.time.format.DateTimeFormatter;

LocalDate today = LocalDate.now();
LocalDate d = LocalDate.of(2026, 6, 21);
LocalDate next = today.plusDays(7).minusMonths(1);   // immutable -> returns new objects
boolean before = d.isBefore(today);

LocalDateTime now = LocalDateTime.now();
Instant ts = Instant.now();                  // a UTC timestamp for logging/storage

Duration dur = Duration.between(now, now.plusHours(2));   // 2 hours
dur.toMinutes();                             // 120
Period age = Period.between(LocalDate.of(1990, 1, 1), today);   // years/months/days

ZonedDateTime ny = ZonedDateTime.now(ZoneId.of("America/New_York"));

// Formatting & parsing:
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");
String s = now.format(fmt);
LocalDate parsed = LocalDate.parse("2026-06-21");   // ISO format by default
```

### 13.2 Regular expressions — Pattern & Matcher **[I]**

`Pattern` compiles a regex (reuse it — compilation is expensive); `Matcher` applies it to input. Note: in Java string literals, backslashes are doubled (`\\d` for the regex `\d`).

```java
import java.util.regex.*;

Pattern p = Pattern.compile("(\\d{4})-(\\d{2})-(\\d{2})");   // a date with capture groups
Matcher m = p.matcher("today is 2026-06-21 ok");

if (m.find()) {                       // search for the pattern anywhere
    System.out.println(m.group());     // "2026-06-21"  (whole match)
    System.out.println(m.group(1));    // "2026"        (first capture group)
}

"2026-06-21".matches("\\d{4}-\\d{2}-\\d{2}");   // true — full-string match (String convenience)
"a1b2c3".replaceAll("\\d", "#");                 // "a#b#c#"
Pattern.compile(",\\s*").split("a, b,c");        // ["a", "b", "c"]
```

### 13.3 Scanner, Random, UUID **[I]**

```java
import java.util.*;

// Random numbers:
Random rnd = new Random();
rnd.nextInt(100);            // 0..99
rnd.nextDouble();            // 0.0..1.0
rnd.nextBoolean();
// Modern source (17+) with selectable algorithms:
RandomGenerator g = RandomGenerator.getDefault();

// ThreadLocalRandom for concurrent code (avoid sharing one Random across threads):
int n = java.util.concurrent.ThreadLocalRandom.current().nextInt(1, 7);   // a die roll

// UUID — a unique 128-bit identifier (great for IDs):
UUID id = UUID.randomUUID();                 // e.g. "550e8400-e29b-41d4-a716-446655440000"
String idStr = id.toString();
UUID back = UUID.fromString(idStr);

// Math utilities:
Math.max(3, 7); Math.min(3, 7); Math.abs(-5);
Math.pow(2, 10); Math.sqrt(144); Math.round(3.7);
Math.floorDiv(7, 2); Math.floorMod(-1, 3);   // proper modulo for negatives
```

### 13.4 Objects utility & null handling **[I]**

```java
import java.util.Objects;
Objects.requireNonNull(arg, "arg must not be null");   // fail fast with a clear message
Objects.equals(a, b);          // null-safe equals (handles either being null)
Objects.hashCode(x);
Objects.requireNonNullElse(maybeNull, "default");
Objects.toString(maybeNull, "none");
```

---

## 14. Build Tools & Ecosystem

Real projects aren't compiled by hand with `javac` — they use a *build tool* to manage dependencies, compile, test, and package. Two dominate: **Maven** and **Gradle**.

### 14.1 The standard project layout **[I]**

Both tools use the same convention:

```
my-app/
├── pom.xml  (Maven)  or  build.gradle (Gradle)
└── src/
    ├── main/
    │   ├── java/          # production source (packages here)
    │   │   └── com/example/App.java
    │   └── resources/     # config files, etc. (bundled into the jar)
    └── test/
        ├── java/          # test source
        └── resources/
```

### 14.2 Maven **[I]**

Maven is declarative and convention-driven, configured by an XML `pom.xml` (Project Object Model). You declare dependencies by coordinates (groupId:artifactId:version); Maven downloads them (and their transitive dependencies) from Maven Central into a local cache. It has a fixed **lifecycle** of phases: `validate → compile → test → package → verify → install → deploy`. Running a later phase runs all earlier ones.

```xml
<!-- pom.xml -->
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>my-app</artifactId>
  <version>1.0.0</version>
  <properties>
    <maven.compiler.release>21</maven.compiler.release>   <!-- target Java 21 -->
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>
  <dependencies>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <version>5.11.0</version>
      <scope>test</scope>                     <!-- only on the test classpath -->
    </dependency>
  </dependencies>
</project>
```

```bash
mvn compile          # compile main source
mvn test             # compile + run tests
mvn package          # produce the jar in target/
mvn clean install    # clean, build, test, install to local repo
mvn dependency:tree  # see the full dependency graph (great for conflicts)
```

### 14.3 Gradle **[I/A]**

Gradle uses a programmable build script (Groovy or, preferably, **Kotlin DSL** in `build.gradle.kts`). It's faster (incremental builds, build cache, daemon) and more flexible than Maven, at the cost of more complexity. Popular for Android and large polyglot builds.

```kotlin
// build.gradle.kts
plugins {
    application
    java
}
java {
    toolchain { languageVersion.set(JavaLanguageVersion.of(21)) }
}
repositories { mavenCentral() }
dependencies {
    implementation("com.google.guava:guava:33.0.0-jre")
    testImplementation("org.junit.jupiter:junit-jupiter:5.11.0")
}
application { mainClass.set("com.example.App") }
tasks.test { useJUnitPlatform() }
```

```bash
./gradlew build      # compile, test, assemble (the wrapper pins the Gradle version)
./gradlew run        # run the application
./gradlew test
```

> Use the **wrapper** (`gradlew`/`mvnw`) checked into the repo so every developer and CI uses the exact same tool version — no "works on my machine."

### 14.4 JARs **[I]**

A **JAR** (Java ARchive) is a ZIP file bundling `.class` files (plus resources and a `MANIFEST.MF`). It's how Java code is distributed and run. An *executable* (fat/uber) jar bundles dependencies and declares a `Main-Class`, so `java -jar app.jar` just works.

```bash
jar cf app.jar -C target/classes .          # create a jar
jar tf app.jar                               # list contents
java -jar app.jar                            # run an executable jar
java -cp app.jar com.example.App             # run a class from the classpath
```

### 14.5 The module system (JPMS) — overview **[A]**

**⚡ Version note (9+):** The Java Platform Module System adds a layer above packages: a *module* (declared in `module-info.java`) explicitly states what packages it **exports** (its public API) and what other modules it **requires**. This gives strong encapsulation (you can hide internal packages even if they're `public`), reliable dependency configuration (no more classpath chaos), and lets `jlink` build a minimal custom runtime with only the modules you use. The JDK itself is modularized (`java.base`, `java.sql`, etc.). Many applications still run on the classpath rather than as modules; JPMS matters most for libraries and slim runtimes.

```java
// module-info.java at the source root
module com.example.app {
    requires java.net.http;           // depend on the HTTP client module
    requires com.example.util;        // depend on another module
    exports com.example.app.api;      // expose ONLY this package to others
    // com.example.app.internal stays hidden even though its classes are public
}
```

### 14.6 Spring Boot — pointer **[A]**

For real-world server applications, **Spring Boot** is the dominant framework: it provides dependency injection, auto-configuration, an embedded web server, data access, security, and "starter" dependencies that wire everything together. A typical app:

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) { SpringApplication.run(App.class, args); }
}

@RestController
class HelloController {
    @GetMapping("/hello")
    String hello() { return "Hello, world"; }
}
```

Learning Spring Boot (plus an ORM like Spring Data JPA/Hibernate) is the standard next step after this guide for backend work. Note virtual threads (§12.6) now let Spring handle huge concurrency with simple blocking code.

---

## 15. Testing: JUnit 5 & Mockito

### 15.1 JUnit 5 basics **[I]**

**JUnit 5** (a.k.a. Jupiter) is the standard testing framework. A test is a method annotated `@Test`; you make *assertions* about expected behaviour. Tests are isolated and should be deterministic.

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {

    Calculator calc;

    @BeforeEach                              // runs before EACH test -> fresh state
    void setUp() { calc = new Calculator(); }

    @Test
    void addsTwoNumbers() {
        assertEquals(5, calc.add(2, 3));      // expected, actual
    }

    @Test
    void throwsOnDivideByZero() {
        // assert that the code throws a specific exception:
        assertThrows(ArithmeticException.class, () -> calc.divide(1, 0));
    }

    @Test
    void multipleAssertions() {
        assertAll(                             // report ALL failures, not just the first
            () -> assertTrue(calc.isPositive(5)),
            () -> assertFalse(calc.isPositive(-1)),
            () -> assertNotNull(calc));
    }

    @Disabled("flaky — fix later")
    @Test
    void skipped() { }
}
```

### 15.2 Lifecycle annotations **[I]**

| Annotation | When it runs |
|-----------|--------------|
| `@BeforeAll` | once, before all tests (must be `static`) — expensive shared setup |
| `@BeforeEach` | before every test method — fresh per-test state |
| `@AfterEach` | after every test method — cleanup |
| `@AfterAll` | once, after all tests (`static`) |

### 15.3 Parameterized tests **[I]**

Run the same test logic over many inputs without duplication.

```java
import org.junit.jupiter.params.*;
import org.junit.jupiter.params.provider.*;

class ParamTest {
    @ParameterizedTest
    @ValueSource(ints = {2, 4, 6, 8})
    void allEven(int n) { assertEquals(0, n % 2); }

    @ParameterizedTest
    @CsvSource({"2, 3, 5", "10, 20, 30", "-1, 1, 0"})   // each row -> a, b, expected
    void adds(int a, int b, int expected) {
        assertEquals(expected, a + b);
    }
}
```

### 15.4 Mockito basics **[I]**

A **mock** is a fake implementation of a dependency, so you can test a class in isolation (without hitting a real database or network) and verify how it interacts with that dependency. **Mockito** is the standard mocking library: you stub return values (`when(...).thenReturn(...)`) and verify calls (`verify(...)`).

```java
import static org.mockito.Mockito.*;

@Test
void usesRepository() {
    // 1) Create a mock of a dependency:
    UserRepository repo = mock(UserRepository.class);

    // 2) Stub its behaviour:
    when(repo.findById(42)).thenReturn(new User("Ada"));

    // 3) Inject the mock and exercise the code under test:
    UserService service = new UserService(repo);
    String name = service.getUserName(42);

    // 4) Assert results AND verify interactions:
    assertEquals("Ada", name);
    verify(repo).findById(42);             // confirm the method was called
    verify(repo, never()).deleteAll();     // confirm something did NOT happen
}
```

---

## 16. Memory & Performance

### 16.1 Heap vs stack **[A]**

The JVM divides memory into two key regions:

- **The stack** — per-thread, holds *method call frames*: local variables, parameters, and *references* (not the objects). It's small, fast (LIFO push/pop), and automatically reclaimed when a method returns. Primitives declared locally and reference *variables* live here. Deep/infinite recursion overflows it → `StackOverflowError`.
- **The heap** — shared by all threads, holds *every object* created with `new` (and arrays). It's large, managed by the garbage collector. The reference on the stack points into the heap. Running out of heap → `OutOfMemoryError`.

```java
void example() {
    int x = 5;                  // primitive x -> on the STACK
    Person p = new Person();    // the Person OBJECT is on the HEAP;
                                // the reference `p` is on the STACK
}   // frame popped: x and the reference p vanish; the Person object becomes
    // eligible for GC once nothing references it
```

### 16.2 Garbage collection overview **[A]**

Java has *automatic* memory management: you never `free()` objects. The **garbage collector (GC)** periodically finds objects no longer *reachable* from any live reference (roots: stack variables, statics) and reclaims their memory. The dominant model is *generational*: most objects die young, so the heap is split into a **young generation** (cheap, frequent "minor" GCs) and an **old generation** (promoted survivors, rarer "major" GCs).

Modern collectors are excellent and mostly self-tuning:
- **G1** — the default; balances throughput and pause times, good general choice.
- **ZGC** / **Shenandoah** — ultra-low-pause (sub-millisecond) collectors for latency-sensitive apps with large heaps. ZGC is now production-ready and *generational*.

You tune with flags: `-Xms`/`-Xmx` (initial/max heap), `-XX:+UseZGC`, etc. Don't prematurely tune; measure first.

### 16.3 Memory leaks in a GC language **[A]**

GC doesn't prevent leaks — it prevents leaks of *unreachable* objects. A **leak** here means objects you no longer need but are still *reachable*, so GC won't collect them. Common causes:
- A long-lived collection (static `List`/`Map`, cache) that you keep adding to and never clear.
- Listeners/callbacks registered but never unregistered.
- Inner classes (and lambdas capturing `this`) holding a hidden reference to a large enclosing object.

```java
// LEAK: a static cache that grows forever
static final Map<String, byte[]> CACHE = new HashMap<>();
void store(String k, byte[] v) { CACHE.put(k, v); }   // never evicted -> grows unbounded
// FIX: use a bounded cache (e.g. Caffeine), WeakHashMap, or evict explicitly.
```

### 16.4 Autoboxing pitfalls **[A]**

Autoboxing (§2.2) is convenient but allocates a wrapper object and costs cache misses. In hot loops it can dominate runtime; prefer primitives and primitive streams (`IntStream`) on hot paths.

```java
// SLOW: each += autoboxes/unboxes; creates ~a billion Long objects
Long sum = 0L;
for (long i = 0; i < 1_000_000_000L; i++) sum += i;   // boxing on every iteration!

// FAST: use the primitive
long sum2 = 0L;
for (long i = 0; i < 1_000_000_000L; i++) sum2 += i;
```

---

## 17. Gotchas & Best Practices

### 17.1 == vs .equals() **[B/I]**

`==` compares *references* (identity) for objects, *values* for primitives. `.equals()` compares *content*. Always use `.equals()` for objects (and beware the wrapper-cache trap):

```java
Integer a = 127, b = 127;
Integer c = 128, d = 128;
System.out.println(a == b);        // true  — the Integer cache holds -128..127
System.out.println(c == d);        // false — outside the cache, different objects!
System.out.println(c.equals(d));   // true  — ALWAYS use equals for wrappers
```

### 17.2 Integer overflow **[I]**

`int`/`long` are fixed-width and **wrap around silently** on overflow — no exception. This causes subtle, dangerous bugs (security, finance).

```java
int max = Integer.MAX_VALUE;       // 2147483647
System.out.println(max + 1);        // -2147483648  (wraps to negative!)

// Use Math.addExact etc. to throw on overflow instead:
Math.addExact(max, 1);              // throws ArithmeticException
// Or use long / BigInteger for large values:
long big = (long) max + 1;          // 2147483648  (cast BEFORE adding)
import java.math.BigInteger;
BigInteger.valueOf(max).add(BigInteger.ONE);   // arbitrary precision
```

### 17.3 Mutable static state **[I/A]**

Mutable `static` fields are global shared state — a source of thread-safety bugs, hidden coupling, and test flakiness. Prefer instance fields, dependency injection, and immutability. If a static must be mutable, make it thread-safe (atomic/concurrent type) and document it.

### 17.4 null and Optional **[I]**

- Return `Optional<T>` (not `null`) from methods that may not find a result (§9.9).
- Validate parameters with `Objects.requireNonNull` at the top of methods (*fail fast*).
- Don't use `Optional` for fields or method parameters.
- Initialize collections to empty (`List.of()`), not `null`, to avoid NPEs in callers.

### 17.5 Other essentials **[I/A]**

```java
// Floating point isn't exact — never use double/float for money:
System.out.println(0.1 + 0.2);      // 0.30000000000000004
import java.math.BigDecimal;
new BigDecimal("0.1").add(new BigDecimal("0.2"));   // exactly 0.3 — use for currency

// Don't modify a collection while iterating it with a for-each -> ConcurrentModificationException:
// for (String s : list) if (cond) list.remove(s);   // BUG
list.removeIf(s -> cond(s));         // correct: use removeIf / an explicit Iterator

// Always close resources -> use try-with-resources (§10.4), never manual close() in finally.

// Prefer immutability: final fields, records, unmodifiable collections. Immutable objects
// are automatically thread-safe and easier to reason about.
```

**Code style essentials:** `PascalCase` for classes/interfaces, `camelCase` for methods/variables/fields, `UPPER_SNAKE_CASE` for `static final` constants, lowercase dotted package names. Keep methods small and single-purpose. Program to interfaces (`List` not `ArrayList` in declarations). Let your IDE (IntelliJ IDEA / Eclipse / VS Code) and a formatter (Spotless, google-java-format) handle layout.

---

## 18. Study Path & Build-to-Learn Projects

**Suggested order:** §1–3 (platform, syntax, control flow) → §4–5 (OOP core and advanced — the heart of Java) → §6 (modern data modeling — records/sealed/patterns; learn early, they change how you write everything) → §8 (collections — daily bread) → §9 (functional/streams — daily bread) → §7, §10 (generics, exceptions) → §11 (file/OS/process — immediately useful) → §13 (stdlib) → §14–15 (build tools & testing — needed for any real project) → §12, §16–17 (concurrency, memory, gotchas — advanced).

**Build these to cement it:**
1. **CLI todo / notes manager** — parse `args`, persist to a JSON or plain-text file with `java.nio.file`, model items as records. Exercises §2, §6, §11. Upgrade it with **picocli**.
2. **Log analyzer** — stream a large log file with `Files.lines`, extract fields with regex, group/count with Streams `Collectors.groupingBy`, print a summary. Exercises §9, §11, §13.
3. **File organizer / backup tool** — `Files.walk` a directory, move/copy files by extension, zip a folder, shell out to `git`/`7z` with `ProcessBuilder`. Exercises §11.
4. **Bank/inventory domain model** — design with sealed interfaces + records + pattern-matching `switch`, enforce invariants with encapsulation, write a full JUnit 5 + Mockito test suite. Exercises §4–6, §10, §15.
5. **Concurrent web fetcher** — fetch many URLs with `java.net.http.HttpClient`, run them on a *virtual-thread-per-task* executor, aggregate with `CompletableFuture` or structured concurrency, handle timeouts and failures. Exercises §12.
6. **A small Maven/Gradle library** — proper `src/` layout, dependencies, tests, an executable jar, optionally a `module-info.java`. Then build a tiny **Spring Boot** REST API on top. Exercises §14, §15.

**Next steps after this guide:** Spring Boot (web/DI/data), an ORM (Hibernate / Spring Data JPA), a database (see the PostgreSQL guide in this library), reactive vs virtual-thread concurrency, and JVM tuning/profiling (JFR, async-profiler). For the file/OS/process material in other languages, compare with the Python, Go, and Bash guides in this library.

---

*Part of the offline developer study library. Written for Java 21 & 25 LTS as of 2026. Confirm fast-moving APIs (preview features, structured concurrency, virtual-thread behaviour) against the JDK Javadoc and JEP index.*
