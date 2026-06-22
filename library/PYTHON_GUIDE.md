# Python — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I've never written Python" to "I write maintainable, typed, production Python" — without an internet connection. Every concept comes with prose that explains *what it is*, *why Python works this way*, *when you reach for it*, and *how to use it* — followed by runnable, commented code. Read top-to-bottom the first time; after that, use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Python 3.13 / 3.14** (current in 2026). Things worth knowing about modern CPython:
> - **Free-threaded CPython** (the "no-GIL" build, PEP 703) is shipping as an officially supported build in 3.13+ and maturing in 3.14. Most code still runs on the default GIL build; the free-threaded build is opt-in (`python3.13t`). Covered in §13.
> - **Experimental JIT** (PEP 744) is being rolled in; it's transparent — you don't change code for it.
> - **Greatly improved error messages & tracebacks** (colorized in 3.13+), and a new interactive REPL.
> - `match`/`case` structural pattern matching (3.10+), the `type` statement and `X | Y` union syntax (3.12+ for generics) are all assumed available.
>
> Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**, so cross-platform notes (path separators, `python` vs `py`, shells) are called out. Confirm exact APIs at docs.python.org.

---

## Table of Contents

1. [Install, Tooling & Running Code](#1-install-tooling--running-code) **[B]**
2. [Syntax, Values & Types](#2-syntax-values--types) **[B]**
3. [Built-in Data Structures](#3-built-in-data-structures) **[B]**
4. [Control Flow](#4-control-flow) **[B]**
5. [Functions](#5-functions) **[B/I]**
6. [Object-Oriented Python](#6-object-oriented-python) **[I]**
7. [Modules & Packages](#7-modules--packages) **[I]**
8. [Exceptions & Error Handling](#8-exceptions--error-handling) **[I]**
9. [Iterators, Generators & Context Managers](#9-iterators-generators--context-managers) **[I]**
10. [Decorators](#10-decorators) **[I/A]**
11. [Type Hints](#11-type-hints) **[I/A]**
12. [File System, OS & Command Execution](#12-file-system-os--command-execution) **[I]**
13. [Concurrency & Parallelism](#13-concurrency--parallelism) **[A]**
14. [Standard Library Highlights](#14-standard-library-highlights) **[I]**
15. [Testing](#15-testing) **[I]**
16. [Packaging & Distribution](#16-packaging--distribution) **[A]**
17. [Performance, Idioms & Gotchas](#17-performance-idioms--gotchas) **[A]**
18. [Study Path & Build-to-Learn Projects](#18-study-path--build-to-learn-projects)

---

## 1. Install, Tooling & Running Code

Before you can write Python you need an interpreter, and before you can work productively you need to understand how Python finds, isolates, and runs code. This section gives you the mental model: Python is an *interpreted* language, meaning a program called the **interpreter** (CPython, the reference implementation) reads your `.py` source and executes it. There is no separate compile step you manage by hand — though under the hood CPython compiles to bytecode (`.pyc` files cached in `__pycache__/`) and then runs that on its virtual machine.

### 1.1 Getting Python **[B]**

Python comes in many versions, and they are *not* fully interchangeable — a 3.10 feature like `match`/`case` will not run on 3.9. So the first skill is knowing *which* interpreter you're invoking.

- **Windows:** install from python.org or `winget install Python.Python.3.13`. Windows ships the **`py` launcher** — a small program that finds and dispatches to the right installed version — so `py -3.13 script.py` picks a specific version even when several are installed. On macOS/Linux the command is usually `python3` (plain `python` historically meant Python 2 and may be missing).
- Always verify *what* you actually have before assuming, because "it ran" and "it ran on the version I expected" are different claims:

```bash
python --version      # or: py --version   (Windows)   /   python3 --version (mac/linux)
python -c "print('hello')"   # run a one-liner without creating a file
where python          # Windows: show every python on PATH (Unix: `which python`)
```

### 1.2 The REPL

The **REPL** (Read-Eval-Print Loop) is an interactive prompt. You type an expression, it evaluates and prints the result immediately. This is Python's killer feature for learning: you never have to guess what a snippet does — you ask. Use it constantly to probe behaviour, inspect objects with `dir(obj)` and `help(obj)`, and test ideas before committing them to a file.

```bash
python                 # opens the interactive shell (3.13+ has a much nicer, colorized REPL)
>>> 2 + 2
4
>>> import math; math.sqrt(16)
4.0
>>> help(str.strip)    # read built-in documentation, offline, no internet needed
>>> dir([])            # list every method a list has
>>> exit()             # or Ctrl-Z then Enter (Windows) / Ctrl-D (unix)
```

### 1.3 Running scripts

A "script" is just a `.py` file you run from start to finish. The interpreter executes top-level statements in order. There are two ways to invoke code, and the difference matters more than beginners expect.

```bash
python hello.py            # run a file by PATH
python -m http.server 8000 # run a module from the import path as a script (-m)
```

`-m` runs a module *from the import path* rather than from a filename. This matters for two reasons. First, it makes packages runnable (`python -m mypackage`). Second, and crucially, prefer `python -m pip ...` over bare `pip`: `-m` guarantees you hit the pip belonging to *this exact interpreter*, sidestepping the classic confusion where `pip` on your PATH installs into a different Python than the one you run.

### 1.4 Virtual environments — **do this for every project** **[B, important]**

Here is the problem a venv solves. Python installs packages into a shared, system-wide location by default. Project A needs `requests==2.28`; project B needs `requests==2.32`. Install one and you break the other. A **virtual environment** (venv) is a self-contained folder with its own copy of the interpreter link and its own `site-packages` directory, so each project gets an isolated, reproducible set of dependencies. This is not optional discipline — it is the baseline for sane Python work.

The logic: when you "activate" a venv, your shell's `PATH` is rewritten so that `python` and `pip` point *into* the venv folder. Anything you install lands there and nowhere else. Deactivating restores your normal PATH.

```bash
python -m venv .venv                 # create a venv in ./.venv (a hidden folder by convention)
# Activate it:
.venv\Scripts\activate               # Windows (PowerShell/cmd)
source .venv/bin/activate            # macOS / Linux / Git Bash
# Now `python` and `pip` point INTO the venv:
pip install requests
pip freeze > requirements.txt        # snapshot the EXACT installed versions (pinned)
pip install -r requirements.txt      # reproduce that exact set elsewhere
deactivate                           # leave the venv (restores your shell PATH)
```

> **Why `pip freeze`?** `requirements.txt` is your reproducibility contract. `pip freeze` writes every package and its exact version, so a teammate (or your CI server, or future-you) gets byte-identical dependencies. Loose specs (`requests`) are fine in `pyproject.toml`; pinned specs (`requests==2.32.3`) belong in a lockfile or frozen requirements.

> **⚡ 2026 tooling:** **`uv`** (from Astral) is the fast, modern, all-in-one replacement for `pip`/`venv`/`pip-tools`/`pyenv`. It's the default many teams reach for now because it is 10–100× faster and manages the lockfile for you:
> ```bash
> uv venv                  # create venv
> uv pip install requests  # install (drop-in pip replacement)
> uv add requests          # add to pyproject.toml + update the lockfile (uv.lock)
> uv run script.py         # run inside the project env automatically (no manual activate)
> ```
> Other tools: **poetry** (dependency + packaging manager), **pipx** (install CLI apps in isolated envs so they don't pollute your projects: `pipx install ruff`).

### 1.5 `pyproject.toml`

`pyproject.toml` is the modern, standardized (PEP 518/621) project metadata file. It replaces the older `setup.py`/`setup.cfg`/`requirements.txt` sprawl with one declarative file that describes your project's name, dependencies, entry-point commands, and tool configuration. Think of it as the single source of truth for "what is this project and what does it need."

```toml
[project]
name = "myapp"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = ["requests>=2.32", "rich"]   # runtime deps, loose lower bounds

[project.scripts]
myapp = "myapp.cli:main"     # creates a `myapp` terminal command -> calls myapp/cli.py:main()

[tool.ruff]                  # ruff = the fast linter+formatter (replaces flake8/black/isort)
line-length = 100
```

**The everyday toolbelt:** **ruff** lints and formats (one tool, very fast); **mypy** or **pyright** checks types; **pytest** runs tests. Configuring all three in `pyproject.toml` keeps your project's standards in one place.

---

## 2. Syntax, Values & Types

### 2.1 The basics **[B]**

Python's syntax is built around two ideas that make it readable: **significant indentation** and **everything is an object**. Where C-family languages use `{ }` to mark blocks, Python uses indentation — a block is a run of lines at the same indent level. Use **4 spaces** (never tabs mixed with spaces; that's a syntax error waiting to happen). This isn't just style; the parser uses it to understand structure, which is why consistent indentation is mandatory rather than cosmetic.

The second idea is the **name/object model**, and getting it right early prevents a whole category of bugs. In Python a variable is not a box that holds a value — it is a **name bound to an object**. The object lives in memory; the name is a label pointing at it. Assignment (`=`) never copies the object; it just points a name at it. This is why two names can refer to the same mutable object and "spookily" affect each other.

```python
# This is a comment. Python has no block comments; use # on each line (or a string).
x = 42                 # int — no type declaration needed; the name `x` is bound to the object 42
name = "Ada"           # str
pi = 3.14159           # float
is_ready = True        # bool (True/False are capitalized)
nothing = None         # the null/absence value (its own type, NoneType)

# Variables are just names pointing at objects; assignment never copies.
a = [1, 2, 3]
b = a                  # b is a SECOND NAME for the SAME list object — no copy happened
b.append(4)
print(a)               # [1, 2, 3, 4]  <- a "changed" because a and b name one object (see §17)
```

Python is **dynamically typed** (a name can be bound to an int now and a str later) but **strongly typed** (it won't silently coerce `"3" + 4`; that's a `TypeError`). Type hints (§11) let you document and tool-check intended types without changing this runtime behaviour.

### 2.2 Numbers **[B]**

Python has three core numeric types, and the design choices behind them are worth understanding. `int` is **arbitrary precision** — there is no 32/64-bit ceiling, so integers never silently overflow; they just use more memory as they grow. `float` is a standard IEEE-754 double, which means it's fast and hardware-backed but *cannot represent most decimals exactly* (a recurring source of "why is 0.1 + 0.2 not 0.3?" confusion). For exact decimal arithmetic — money, especially — reach for `Decimal`.

```python
n = 10
f = 3.0
big = 1_000_000        # underscores allowed as digit separators (purely visual, ignored)
print(7 / 2)           # 3.5   (true division — ALWAYS returns a float, even 4/2 -> 2.0)
print(7 // 2)          # 3     (floor division — rounds toward negative infinity)
print(-7 // 2)         # -4    (NOT -3 — floors, doesn't truncate toward zero)
print(7 % 3)           # 1     (modulo — result takes the sign of the divisor in Python)
print(2 ** 10)         # 1024  (exponent)
print(int("42"), float("3.5"))   # convert from string (raises ValueError on garbage)
print(round(3.14159, 2))         # 3.14  (banker's rounding: round(0.5)==0, round(2.5)==2)
print(abs(-5), min(3, 9), max(3, 9), sum([1, 2, 3]))

# Integers are arbitrary precision — no overflow, ever:
print(2 ** 200)        # a 61-digit number, computed exactly

# Floats are IEEE-754 doubles — beware (see §17):
print(0.1 + 0.2)       # 0.30000000000000004  (binary can't represent 0.1 exactly)
from decimal import Decimal
print(Decimal("0.1") + Decimal("0.2"))   # 0.3  (use Decimal for money / exact decimals)
# Note: pass Decimal a STRING ("0.1"), not a float (0.1) — the float is already imprecise.
```

### 2.3 Strings **[B]**

A `str` in Python 3 is an **immutable sequence of Unicode code points**. Immutable means once created it can never change — every "modifying" method returns a *new* string and leaves the original untouched. This immutability is what makes strings hashable (usable as dict keys) and safe to share, but it's also why building a big string by repeated `+=` in a loop is slow (each step creates a fresh string); use `"".join(...)` instead (§17).

The everyday way to build strings with embedded values is the **f-string** (formatted string literal). The `f` prefix lets you put expressions directly inside `{ }`, and a format spec after a colon controls width, precision, alignment, and number bases.

```python
s = "double"
s2 = 'single'                 # quotes are interchangeable; pick one and be consistent
multi = """spans
multiple lines"""             # triple quotes preserve newlines; also used for docstrings

# f-strings (formatted string literals) — the standard way to interpolate:
user, score = "Ada", 95.5
print(f"{user} scored {score:.1f}%")     # Ada scored 95.5%   (.1f = 1 decimal place)
print(f"{user=}")                          # user='Ada'  (self-documenting debug form, 3.8+)
print(f"{score:>10}")                      # right-align in a field of width 10
print(f"{score:<10}|")                     # left-align;  ^ for center
print(f"{255:#x} {255:08b}")               # 0xff 11111111  (hex / zero-padded binary)
print(f"{1234567:,}")                      # 1,234,567  (thousands separator)

# Common methods (strings are IMMUTABLE — every method returns a NEW string):
t = "  Hello, World  "
print(t.strip())              # "Hello, World"  (removes leading/trailing whitespace)
print(t.strip().lower())      # "hello, world"  (methods chain because each returns a str)
print("a,b,c".split(","))     # ['a', 'b', 'c']  (split on a delimiter)
print("a b  c".split())       # ['a', 'b', 'c']  (no arg -> split on any whitespace run)
print("-".join(["a", "b"]))   # "a-b"            (join is the inverse of split)
print("hello".replace("l", "L"))      # "heLLo"
print("hello".startswith("he"), "hello".endswith("lo"))   # True True
print("Ada".upper(), "ADA".lower(), "ada".capitalize())
print("hello world".title())          # "Hello World"
print(len("hello"))                    # 5  (number of characters)
print("ell" in "hello")                # True (substring test)
print("hello".find("l"), "hello".index("l"))   # 2, 2 (find returns -1 if absent; index raises)
print("42".isdigit(), "abc".isalpha())  # validation helpers

# Slicing (works on strings, lists, tuples): s[start:stop:step]  (stop is EXCLUSIVE)
w = "python"
print(w[0], w[-1])      # p n      (negative index counts from the end; -1 is last)
print(w[0:3])           # pyt      (indices 0,1,2 — stop 3 is excluded)
print(w[2:])            # thon     (omit stop -> to the end)
print(w[::2])           # pto      (every 2nd char)
print(w[::-1])          # nohtyp   (reverse — a famous Python idiom)
```

### 2.4 Booleans, None & truthiness **[B]**

Python lets you use *any* object in a boolean context (an `if`, a `while`, `and`/`or`). The rule of **truthiness** is: certain values are "falsy" (treated as False), and *everything else* is "truthy." This is enormously convenient — `if items:` reads as "if items is non-empty" — but you must memorize the falsy set so you don't get surprised by `0` or empty containers being treated as False.

```python
# Falsy values: False, None, 0, 0.0, 0j, "", [], {}, (), set(), range(0)
# EVERYTHING else is truthy (including "0", "False", [0], and any custom object by default).
if not "":            print("empty string is falsy")
if [1]:               print("non-empty list is truthy")

x = None
print(x is None)      # True — use `is` (identity) for None, never `== None`
# Why `is`? None is a singleton — there's exactly one None object — so identity is correct
# and faster, and it can't be fooled by a class that overrides __eq__.

# Ternary expression (one-line if/else that PRODUCES a value):
score = 72
status = "pass" if score >= 60 else "fail"   # value if condition else other_value
```

### 2.5 Operators **[B]**

Two Python operator behaviours surprise newcomers and are worth calling out. First, **comparison chaining**: `1 < x < 10` is real and means `(1 < x) and (x < 10)`, evaluating `x` once. Second, **`and`/`or` return operands, not booleans**, and short-circuit (stop evaluating as soon as the result is known). `a or b` returns `a` if `a` is truthy, else `b` — which is why `value or default` is the idiomatic fallback.

```python
# Comparison: ==  !=  <  <=  >  >=     (chainable!)
print(1 < 2 < 3)            # True — means (1<2) and (2<3), with the middle evaluated once
# Logical: and  or  not     (short-circuit; return an OPERAND, not just a bool)
print(0 or "default")       # "default"   (0 is falsy, so `or` yields the second operand)
print("a" and "b")          # "b"         (both truthy, `and` yields the last)
print("" and crash())       # ""          ("" is falsy, so crash() is never called)
# Identity vs equality:
print([1] == [1])           # True  (same VALUE — __eq__ compares contents)
print([1] is [1])           # False (different OBJECTS in memory)
# Membership:
print(3 in [1, 2, 3])       # True
print("x" not in "abc")     # True
# Walrus := (assignment expression, 3.8+) — assign AND use the value in one expression:
if (n := len([1, 2, 3])) > 2:
    print(f"length is {n}")   # avoids computing len() twice
```

| Category | Operators | Note |
|---|---|---|
| Arithmetic | `+ - * / // % ** ` | `/` always float, `//` floors |
| Comparison | `== != < <= > >=` | chainable |
| Logical | `and or not` | short-circuit, return operands |
| Bitwise | `& \| ^ ~ << >>` | also set ops on `set` |
| Identity / membership | `is`, `is not`, `in`, `not in` | `is` only for `None`/singletons |
| Assignment | `= += -= *= //= **=` etc. | augmented assignment |

---

## 3. Built-in Data Structures

Python ships four workhorse collections — `list`, `tuple`, `dict`, `set` — and choosing the right one is one of the highest-leverage decisions you make. They differ along two axes: **ordered vs unordered**, and **mutable vs immutable**. Understanding *why* each exists (and its performance characteristics) lets you pick correctly instead of defaulting to `list` for everything.

### 3.1 list — ordered, mutable **[B]**

A `list` is a dynamic, ordered, mutable sequence — the closest thing Python has to an "array," but it grows and shrinks freely and can hold mixed types. Internally it's a resizable array of object references, so indexing is O(1) and appending is amortized O(1), but inserting/removing at the front is O(n) (everything shifts). Reach for a list whenever you have an ordered collection you'll modify.

```python
nums = [3, 1, 2]
nums.append(4)              # [3, 1, 2, 4]   add one item to the end (O(1) amortized)
nums.insert(0, 9)           # [9, 3, 1, 2, 4]  insert at index (O(n) — shifts the rest)
nums.extend([5, 6])         # add multiple items (extend ≠ append: append([5,6]) nests it)
last = nums.pop()           # remove & RETURN last (or pop(i) for a specific index)
nums.remove(9)              # remove the first matching VALUE (raises if absent)
nums.sort()                 # sort IN PLACE (mutates); nums.sort(reverse=True, key=...)
ordered = sorted(nums)      # returns a NEW sorted list, original untouched
print(len(nums), nums[0], nums[-1])
print(2 in nums)            # membership test — O(n) for lists (use a set for fast lookup)
nums[1:3] = [10, 20]        # slice assignment can even change the list's length
print(nums.count(2), nums.index(10))   # count occurrences / find first index

# List comprehension — the Pythonic, fast way to transform/filter a sequence.
# Read it as: "give me EXPR for each ITEM in ITERABLE [if CONDITION]".
squares = [x * x for x in range(5)]            # [0, 1, 4, 9, 16]   (transform)
evens   = [x for x in range(10) if x % 2 == 0] # [0, 2, 4, 6, 8]    (filter)
pairs   = [(x, y) for x in "ab" for y in [1, 2]]  # nested loops, left-to-right
# Comprehensions are faster than the equivalent for-loop + append AND clearer once learned.
```

### 3.2 tuple — ordered, immutable **[B]**

A `tuple` is an ordered, **immutable** sequence. "Immutable" is the whole point: once built it can't change, which makes it **hashable** (usable as a dict key or set member, unlike a list) and signals to readers "this is a fixed record." Use tuples for heterogeneous fixed-size records (a coordinate, a database row) and lists for homogeneous, growable collections — though this is a guideline, not a law.

```python
point = (3, 4)
x, y = point                 # UNPACKING — bind each element to a name (very common)
print(point[0])              # 3   (indexing/slicing work like lists; mutation does not)
single = (1,)                # a ONE-element tuple NEEDS the trailing comma; (1) is just int 1
empty = ()
# Tuples are great as fixed records and as dict keys (because they're hashable):
locations = {(0, 0): "origin", (1, 2): "point A"}

a, b = 1, 2
a, b = b, a                  # swap with no temp var — the right side builds a tuple first
first, *rest = [1, 2, 3, 4]  # extended unpacking: first=1, rest=[2,3,4] (rest is a LIST)
*init, last = [1, 2, 3, 4]   # init=[1,2,3], last=4
```

### 3.3 dict — key→value mapping **[B]**

A `dict` maps **keys to values** with O(1) average lookup, insertion, and deletion — it's a hash table. This is the structure you use whenever you need to look something up *by name* rather than by position. Keys must be **hashable** (immutable: str, int, tuple of immutables — not lists). Since Python 3.7, dicts **guarantee insertion order**, so iterating a dict yields keys in the order they were added.

```python
user = {"name": "Ada", "age": 36}
print(user["name"])             # Ada  (raises KeyError if the key is missing)
print(user.get("email"))        # None (get never raises — returns None for missing)
print(user.get("email", "n/a")) # "n/a"  (supply a default)
user["email"] = "a@x.com"       # add OR update (same syntax)
del user["age"]                 # remove a key (raises KeyError if absent)
print("name" in user)           # membership tests KEYS, and it's O(1)
for key, value in user.items(): # iterate key/value PAIRS — the usual loop
    print(key, value)
print(list(user.keys()), list(user.values()))
user.update({"age": 37, "city": "London"})   # bulk add/update from another dict
merged = {**user, "role": "eng"}   # merge via unpacking (right wins on conflicts)
merged = user | {"role": "eng"}    # merge operator (3.9+) — cleaner equivalent
# setdefault: get a key, inserting a default if it's missing (handy for grouping):
groups = {}
groups.setdefault("a", []).append(1)
# Dict comprehension:
sq = {n: n * n for n in range(4)}  # {0:0, 1:1, 2:4, 3:9}
```

### 3.4 set & frozenset — unique, unordered **[B]**

A `set` is an unordered collection of **unique, hashable** elements, backed by a hash table like dict. Its superpowers are O(1) membership testing and mathematical set operations (union, intersection, difference). Use a set when you need to deduplicate, test "is X in this collection?" fast, or do set algebra. `frozenset` is the immutable (hence hashable) version, so it can live inside another set or be a dict key.

```python
s = {1, 2, 2, 3}            # {1, 2, 3}  (duplicates collapse automatically)
s.add(4); s.discard(1)     # discard = no error if absent; remove() raises KeyError
print(2 in s)              # fast membership — O(1) average (vs O(n) for a list)
a, b = {1, 2, 3}, {2, 3, 4}
print(a | b)               # union           {1,2,3,4}
print(a & b)               # intersection    {2,3}
print(a - b)               # difference      {1}
print(a ^ b)               # symmetric diff  {1,4}  (in one or the other, not both)
print(a <= b)              # subset test
empty = set()              # IMPORTANT: {} is an empty DICT, not a set!
frozen = frozenset([1, 2]) # immutable & hashable -> usable as a dict key / set member
unique = list(set([3, 1, 3, 2]))   # dedupe a list (loses order; use dict.fromkeys to keep it)
```

### 3.5 Choosing a structure

| Need | Use | Why |
|---|---|---|
| Ordered, changeable sequence | `list` | O(1) index/append, mutable |
| Fixed record / hashable sequence | `tuple` | immutable, hashable, signals "fixed" |
| Lookup by key | `dict` | O(1) keyed access, ordered |
| Unique items / fast membership / set math | `set` | O(1) membership, set algebra |
| Immutable set (dict key, etc.) | `frozenset` | hashable set |
| Counting occurrences | `collections.Counter` (§14) | tallies + `most_common` |
| Default values per key | `collections.defaultdict` (§14) | no KeyError on first access |
| Fast appends/pops at BOTH ends (queue) | `collections.deque` (§14) | O(1) both ends (list is O(n) at front) |

---

## 4. Control Flow

Control flow is how you make decisions and repeat work. Python keeps it minimal: `if`/`elif`/`else` for branching, `for`/`while` for looping, plus the modern `match`/`case` for matching the *shape* of data. The recurring theme is that Python prefers iterating over *things* rather than over *index numbers*.

### 4.1 if / elif / else **[B]**

Branches are evaluated top to bottom; the first truthy condition wins and the rest are skipped. `elif` ("else if") avoids deep nesting. There is no `switch` statement in the C sense — historically you chained `elif`s, and now you also have `match` (§4.3).

```python
def grade(score):
    if score >= 90:
        return "A"
    elif score >= 80:        # only checked if the first was False
        return "B"
    elif score >= 60:
        return "C"
    else:                    # the fallback when nothing above matched
        return "F"
```

### 4.2 for & while **[B]**

`for` iterates over any **iterable** (anything you can step through: lists, strings, files, ranges, generators). You almost never manage an index counter by hand — instead use `enumerate` when you need the index, and `zip` to walk several sequences in lockstep. `while` repeats as long as a condition stays truthy and is the right tool when you don't know the iteration count ahead of time.

```python
for i in range(5):              # 0,1,2,3,4   range(start, stop, step), stop EXCLUSIVE
    print(i)
for i in range(2, 10, 2):       # 2,4,6,8
    print(i)

for ch in "abc":                # strings are iterable -> one character at a time
    print(ch)

# enumerate: when you genuinely need the index alongside the value
for i, name in enumerate(["a", "b"], start=1):   # start controls the first index
    print(i, name)              # 1 a / 2 b

# zip: iterate multiple sequences in parallel (stops at the shortest)
for name, age in zip(["Ada", "Bo"], [36, 29]):
    print(name, age)

n = 5
while n > 0:                    # repeat until the condition becomes falsy
    print(n)
    n -= 1                      # MUST change n, or you loop forever

# break stops the loop entirely; continue skips to the next iteration.
# The rarely-known loop `else` runs ONLY IF the loop finished without `break`:
for x in [1, 3, 5]:
    if x % 2 == 0:
        break
else:
    print("no even number found")   # this runs (no break happened) — useful for search loops
```

### 4.3 match / case — structural pattern matching **[I]** (3.10+)

`match`/`case` is far more than a `switch`. It performs **structural pattern matching**: it inspects the *shape* of a value — its type, its length, its keys, its attributes — and can simultaneously *destructure* it, binding pieces to names. This is what makes it powerful for parsing commands, walking ASTs, or handling tagged data. Patterns are tried top to bottom; `_` is the wildcard (default). A bare name in a pattern is a **capture** (it binds), so use dotted names or guards when you mean "match this specific value."

```python
def describe(command):
    match command.split():               # match against a list pattern
        case ["quit"]:
            return "exiting"
        case ["move", direction]:        # binds the 2nd word to `direction`
            return f"moving {direction}"
        case ["move", *rest]:            # capture the remaining elements into `rest`
            return f"moving multiple: {rest}"
        case _:                          # wildcard (default) — matches anything
            return "unknown"

# Match on type, mapping shape, and attribute values:
def area(shape):
    match shape:
        case {"type": "circle", "r": r}:       # dict/mapping pattern, binds r
            return 3.14159 * r * r
        case {"type": "rect", "w": w, "h": h}:
            return w * h
        case Point(x=0, y=0):                  # class pattern (matches by attributes)
            return 0
        case int() | float() as n if n < 0:    # type pattern + OR + guard (the `if`)
            raise ValueError(f"negative size {n}")
        case _:
            raise ValueError("unknown shape")
```

---

## 5. Functions

A function packages reusable behaviour behind a name. In Python functions are **first-class objects**: you can pass them as arguments, return them from other functions, store them in lists, and attach attributes. That property is the foundation for closures, decorators (§10), and the functional-style tools in the standard library. Understanding how arguments are passed and how names are resolved (scope) is what separates "I can write a function" from "I understand what my function does."

### 5.1 Defining & calling **[B]**

You define with `def`, document with a **docstring** (the first string in the body — accessible via `help()` and `__doc__`), and return a value with `return` (a function with no `return` yields `None`). Parameters can have defaults, making them optional, and callers can pass arguments **positionally** or by **keyword** (by name), which improves readability at call sites.

```python
def greet(name, greeting="Hello"):     # `greeting` has a default -> it's optional
    """Return a greeting (this first string is the docstring)."""
    return f"{greeting}, {name}!"

print(greet("Ada"))                    # Hello, Ada!           (positional)
print(greet("Bo", greeting="Hi"))      # Hi, Bo!               (keyword for the second)
print(greet(greeting="Hey", name="Cy"))# Hey, Cy!  (keyword args can be in any order)
print(greet.__doc__)                   # access the docstring programmatically
```

> **Pass-by-object-reference.** Python doesn't "pass by value" or "pass by reference" in the classic sense — it passes the *object reference by value*. The practical upshot: if you mutate a mutable argument (e.g. `lst.append(x)`), the caller sees it; if you rebind the parameter (`lst = [...]`), the caller does not.

### 5.2 `*args` and `**kwargs` **[I]**

Sometimes you don't know in advance how many arguments a function will receive. `*args` collects extra **positional** arguments into a tuple; `**kwargs` collects extra **keyword** arguments into a dict. (The names are convention; the `*` and `**` do the work.) The same `*`/`**` syntax also works at the *call* site to "spread" a sequence/dict into arguments — this symmetry is the key insight.

```python
def total(*args):                      # args is a TUPLE of all positional args
    return sum(args)
print(total(1, 2, 3))                  # 6
print(total())                         # 0  (works with zero args)

def configure(**kwargs):               # kwargs is a DICT of all keyword args
    for k, v in kwargs.items():
        print(f"{k} = {v}")
configure(debug=True, level=3)

def f(a, b, *args, **kwargs): ...      # the full canonical order of parameter kinds

# Unpacking when CALLING (the mirror image):
nums = [1, 2, 3]
print(total(*nums))                    # spread a list as positional args -> total(1,2,3)
opts = {"debug": True}
configure(**opts)                      # spread a dict as keyword args -> configure(debug=True)
```

### 5.3 Positional-only & keyword-only params **[A]**

You can constrain *how* callers pass each argument. Everything before a `/` is **positional-only** (callers can't use its name — useful when the parameter name is an implementation detail you might rename). Everything after a `*` is **keyword-only** (callers *must* name it — useful for flags whose meaning isn't obvious positionally, like `sorted(data, reverse=True)`). This makes APIs both safer and clearer.

```python
def f(pos_only, /, normal, *, kw_only):
    #            ^ everything BEFORE / is positional-only
    #                             ^ everything AFTER * is keyword-only
    return (pos_only, normal, kw_only)

f(1, 2, kw_only=3)        # ok: pos_only & normal positional, kw_only by name
f(1, normal=2, kw_only=3) # ok: normal can be either
# f(pos_only=1, ...)      # ERROR: pos_only cannot be passed by name
# f(1, 2, 3)              # ERROR: kw_only must be passed by name
```

### 5.4 Lambdas, scope & closures **[I]**

A **lambda** is a tiny anonymous function limited to a single expression — handy as a throwaway `key=` for sorting. **Scope** is the set of rules (called **LEGB**) Python uses to resolve a name: it looks in the **L**ocal scope, then any **E**nclosing function scopes, then the **G**lobal (module) scope, then **B**uilt-ins. Crucially, *assigning* to a name inside a function makes it local by default — to modify an outer name you must declare `global` or `nonlocal`. A **closure** is an inner function that "captures" and remembers variables from its enclosing scope even after the outer function has returned; this is how you create function factories and stateful functions without classes.

```python
# Lambda: a one-expression anonymous function. Great for sort keys, not for logic.
add = lambda a, b: a + b
people = [("Ada", 36), ("Bo", 29)]
people.sort(key=lambda p: p[1])        # sort by the second element (age)

# Scope: LEGB lookup order. Assignment creates a LOCAL name unless told otherwise.
count = 0
def increment():
    global count        # without this, `count += 1` would raise UnboundLocalError
    count += 1          # because the assignment marks `count` local, but it's read first

# Closure: an inner function captures variables from the enclosing scope.
def make_multiplier(factor):
    def multiply(n):
        return n * factor          # `factor` is captured and remembered after make_* returns
    return multiply                # return the inner function itself (functions are objects)
double = make_multiplier(2)
print(double(5))                   # 10  — `double` still knows factor == 2

# nonlocal: rebind a variable in the ENCLOSING (not global) scope — for stateful closures.
def counter():
    n = 0
    def inc():
        nonlocal n                 # without this, n += 1 would try to make a new local n
        n += 1
        return n
    return inc
c = counter()
print(c(), c(), c())               # 1 2 3  — the closure keeps state between calls
```

---

## 6. Object-Oriented Python

Object-oriented programming bundles **data** (attributes) with the **behaviour** that operates on it (methods) into objects, created from blueprints called **classes**. The reason to use OOP is *modeling*: when your problem has things with state and identity that change over time (a bank account, a game character, an HTTP session), a class captures that cleanly. Python's object model is unusually transparent — almost everything is an object, attribute access is customizable, and the "magic" you can hook into (dunder methods) is explicit and learnable. That said, Python doesn't force OOP on you; for plain data, a `dict`, `dataclass`, or function is often better than a heavyweight class.

### 6.1 Classes & instances **[I]**

A **class** is a blueprint; an **instance** is a concrete object built from it. `__init__` is the **initializer** — it runs right after a new instance is created and sets up its per-object state. The first parameter of every instance method is `self`, the instance being operated on; Python passes it automatically when you call `obj.method()`. The distinction between **class attributes** (defined in the class body, shared by all instances) and **instance attributes** (assigned via `self.x = ...`, unique per object) is fundamental and a common source of bugs when a mutable class attribute is shared accidentally.

```python
class Account:
    interest_rate = 0.02              # CLASS attribute — shared by ALL instances

    def __init__(self, owner, balance=0):
        # __init__ runs on each new instance; `self` is that fresh instance.
        self.owner = owner            # INSTANCE attributes — unique to each object
        self.balance = balance

    def deposit(self, amount):        # `self` is the instance; Python passes it for you
        if amount <= 0:
            raise ValueError("amount must be positive")
        self.balance += amount
        return self.balance

    def __repr__(self):               # UNAMBIGUOUS developer representation (for debugging)
        return f"Account({self.owner!r}, {self.balance})"   # !r calls repr() on owner

    def __str__(self):                # FRIENDLY representation (used by print/str)
        return f"{self.owner}: ${self.balance}"

acc = Account("Ada", 100)             # calls __init__ with owner="Ada", balance=100
acc.deposit(50)
print(acc)            # Ada: $150            (print uses __str__)
print(repr(acc))      # Account('Ada', 150)  (repr uses __repr__)
print(Account.interest_rate)          # 0.02 — accessible on the class itself
```

> **Why two string methods?** `__str__` is for humans (end-user display); `__repr__` is for developers (should ideally be valid Python that recreates the object, and is what you see in the REPL and in container displays). If you write only one, write `__repr__` — `str()` falls back to it.

### 6.2 classmethod, staticmethod, property **[I]**

These three decorators reshape how a method relates to its class and instances, and each solves a specific design need.

- **`@property`** turns a method into a *computed attribute*. You call it without parentheses (`t.fahrenheit`), so callers can't tell whether it's stored or computed — this lets you add validation or computation later *without breaking the public interface*. It's Python's answer to getters/setters: write plain attribute access until you need logic, then promote it to a property.
- **`@classmethod`** receives the *class* (`cls`) instead of an instance. Its main use is **alternative constructors** — factory methods like `from_string` that build and return an instance, while still working correctly for subclasses (because `cls` is the actual subclass).
- **`@staticmethod`** receives neither `self` nor `cls`. It's just a plain function living in the class's namespace for organization — use it for helpers that logically belong to the class but don't touch instance or class state.

```python
class Temperature:
    def __init__(self, celsius):
        self._celsius = celsius        # leading _ signals "internal, don't touch directly"

    @property                          # access like an attribute: t.fahrenheit (no parens)
    def fahrenheit(self):
        return self._celsius * 9 / 5 + 32

    @fahrenheit.setter                 # enables: t.fahrenheit = 100 (with validation room)
    def fahrenheit(self, value):
        self._celsius = (value - 32) * 5 / 9

    @classmethod                       # alternative constructor — receives the CLASS as cls
    def from_fahrenheit(cls, f):
        return cls((f - 32) * 5 / 9)   # cls(...) builds the right type, even for subclasses

    @staticmethod                      # no self/cls — just a namespaced helper
    def is_freezing(celsius):
        return celsius <= 0

t = Temperature(25)
print(t.fahrenheit)                    # 77.0  (computed on access)
t.fahrenheit = 212                     # calls the setter
print(t._celsius)                      # 100.0
boiling = Temperature.from_fahrenheit(212)   # use the alternative constructor
print(Temperature.is_freezing(-5))     # True
```

### 6.3 Inheritance, MRO & super() **[I]**

**Inheritance** lets a class reuse and specialize another class's behaviour: a subclass gets the parent's attributes/methods and can **override** them. Use it for genuine "is-a" relationships (a `Dog` *is an* `Animal`). `super()` calls the parent implementation — essential in `__init__` so the parent can set up its part of the object before you add yours. When multiple inheritance is involved, Python uses the **MRO** (Method Resolution Order, computed by the C3 algorithm) to decide, deterministically, which class's method runs; `super()` follows that order, which is why cooperative multiple inheritance works. Favor **composition** (holding other objects) over deep inheritance hierarchies — it's usually more flexible.

```python
class Animal:
    def __init__(self, name):
        self.name = name
    def speak(self):
        raise NotImplementedError("subclasses must implement speak()")
    def describe(self):
        return f"{self.name} says {self.speak()}"

class Dog(Animal):
    def __init__(self, name, breed):
        super().__init__(name)        # run the parent's __init__ FIRST (sets self.name)
        self.breed = breed            # then add subclass-specific state
    def speak(self):                  # override the abstract method
        return "Woof"

d = Dog("Rex", "Lab")
print(d.name, d.speak())              # Rex Woof
print(d.describe())                   # Rex says Woof  (parent method calls overridden speak)
print(isinstance(d, Animal))          # True  — a Dog IS an Animal
print(issubclass(Dog, Animal))        # True
print(Dog.__mro__)                    # the search order: (Dog, Animal, object)

# Mixins: small classes that add a slice of behaviour via multiple inheritance.
class JsonMixin:
    def to_json(self):
        import json
        return json.dumps(self.__dict__)   # __dict__ holds instance attributes

class User(JsonMixin):
    def __init__(self, name): self.name = name
print(User("Ada").to_json())          # {"name": "Ada"}
```

### 6.4 Dunder (magic) methods **[I]**

"Dunder" (double-underscore) methods are the hooks that integrate your objects with Python's syntax and built-ins. When you write `a + b`, Python actually calls `a.__add__(b)`; `len(x)` calls `x.__len__()`; `x[i]` calls `x.__getitem__(i)`. By implementing the right dunders, your custom objects become indistinguishable from built-ins at the syntax level — they can be added, compared, indexed, iterated, printed, and used in `with` blocks. This is **operator overloading** done in a principled, opt-in way.

```python
class Vector:
    def __init__(self, x, y):
        self.x, self.y = x, y
    def __add__(self, other):                       # enables  v1 + v2
        return Vector(self.x + other.x, self.y + other.y)
    def __mul__(self, scalar):                      # enables  v * 3
        return Vector(self.x * scalar, self.y * scalar)
    def __eq__(self, other):                        # enables  ==  (compare by value)
        return (self.x, self.y) == (other.x, other.y)
    def __hash__(self):                             # needed to use Vector in a set/dict key
        return hash((self.x, self.y))               # (define when you define __eq__)
    def __len__(self):                              # enables  len(v)
        return 2
    def __getitem__(self, i):                       # enables  v[0], and iteration
        return (self.x, self.y)[i]
    def __repr__(self):
        return f"Vector({self.x}, {self.y})"

print(Vector(1, 2) + Vector(3, 4))     # Vector(4, 6)
print(Vector(1, 2) * 3)                # Vector(3, 6)
print(Vector(1, 2) == Vector(1, 2))    # True
print({Vector(0, 0)})                  # works as a set member because __hash__ is defined
```

| Dunder | Enables |
|---|---|
| `__init__` | construction / initialization |
| `__new__` | low-level instance creation (rarely needed; for immutables/singletons) |
| `__repr__` / `__str__` | `repr()` / `str()` & `print` |
| `__eq__`, `__lt__`, `__le__`, … | `==`, `<`, sorting (`@functools.total_ordering` fills the rest) |
| `__hash__` | use as dict key / set member |
| `__len__`, `__getitem__`, `__setitem__` | `len()`, `obj[i]`, `obj[i] = v` |
| `__contains__` | `x in obj` |
| `__iter__`, `__next__` | iteration (§9) |
| `__enter__`, `__exit__` | `with` blocks (§9) |
| `__call__` | make instances callable like functions: `obj()` |
| `__add__`, `__mul__`, `__sub__`, … | arithmetic operators |

### 6.5 Dataclasses — boilerplate-free classes **[I]**

Most classes that just hold data force you to write a repetitive `__init__`, `__repr__`, and `__eq__` by hand. The `@dataclass` decorator (3.7+) generates all of that from your type-annotated attributes. The logic: you *declare* the fields and their types, and the decorator *writes* the plumbing. Use dataclasses for any "struct-like" class — configuration objects, records, value objects. There's one critical rule: a **mutable default** (a list/dict) must use `field(default_factory=...)`, because a plain default would be shared across all instances (the same mutable-default trap as §17).

```python
from dataclasses import dataclass, field

@dataclass                            # generates __init__, __repr__, __eq__ from the fields
class Point:
    x: float                          # required field (type annotation is mandatory here)
    y: float = 0.0                    # field with a default
    tags: list = field(default_factory=list)   # MUTABLE default -> MUST use a factory

    def distance_to(self, other):     # you can still add normal methods
        return ((self.x - other.x) ** 2 + (self.y - other.y) ** 2) ** 0.5

p = Point(3, 4)
print(p)                              # Point(x=3, y=4, tags=[])   (auto-generated __repr__)
print(p == Point(3, 4))               # True  (auto-generated __eq__ compares field by field)

@dataclass(frozen=True)               # frozen -> immutable AND hashable (usable as dict key)
class Config:
    host: str
    port: int = 8080
# c = Config("x"); c.port = 9  -> raises FrozenInstanceError

@dataclass(slots=True)                # slots=True (3.10+) saves memory, blocks new attributes
class Pixel:
    r: int; g: int; b: int
```

### 6.6 Abstract base classes & enums **[I]**

An **abstract base class (ABC)** defines an interface — methods that subclasses *must* implement — and refuses to be instantiated directly. Use it to enforce a contract across a family of classes (every `Repository` must provide `get`). An **enum** defines a fixed set of named constant values, which is far safer and more readable than passing around bare strings or magic numbers (`Color.RED` instead of `1` or `"red"`); the type system and the reader both know the full set of valid values.

```python
from abc import ABC, abstractmethod
from enum import Enum, auto

class Repository(ABC):                 # ABC -> cannot be instantiated directly
    @abstractmethod
    def get(self, id: int): ...        # subclasses MUST implement this or stay abstract

    def get_or_raise(self, id):        # ABCs can also provide concrete shared behaviour
        value = self.get(id)
        if value is None:
            raise KeyError(id)
        return value

class InMemoryRepo(Repository):
    def __init__(self): self._data = {}
    def get(self, id): return self._data.get(id)   # satisfies the abstract method

class Color(Enum):                     # a fixed, named set of constants
    RED = auto()                       # auto() assigns 1, 2, 3, ... automatically
    GREEN = auto()
    BLUE = auto()

print(Color.RED, Color.RED.name, Color.RED.value)   # Color.RED RED 1
print(Color(1))                                       # Color.RED  (look up by value)
print(Color["GREEN"])                                 # Color.GREEN (look up by name)
for c in Color:                                       # enums are iterable
    print(c)

from enum import StrEnum                # 3.11+: members ARE strings (great for APIs/JSON)
class Status(StrEnum):
    ACTIVE = "active"
    CLOSED = "closed"
print(Status.ACTIVE == "active")        # True
```

---

## 7. Modules & Packages

As programs grow you split them into multiple files so each has a clear responsibility. A **module** is simply a `.py` file; a **package** is a directory of modules. The `import` system is how Python finds and loads them, executing the module's top-level code exactly once and caching the result in `sys.modules`. Understanding imports — absolute vs relative, what `__name__` means, how the search path works — is what lets you structure a real project instead of cramming everything into one file.

### 7.1 Importing **[B]**

Importing runs another module and binds names from it into your namespace. There are several forms, and the choice affects readability and the risk of name collisions. Prefer `import module` (keeps names qualified, so it's clear where things come from) or selective `from module import name`. Avoid `from module import *` outside the REPL — it dumps unknown names into your scope and hides their origin.

```python
import math                               # bind the module; use math.sqrt
from math import sqrt, pi                  # bind specific names directly
from math import sqrt as square_root       # alias to avoid a clash or shorten
import json as j                           # alias a whole module
print(math.sqrt(9), sqrt(9), pi)
# import order convention (PEP 8): stdlib, then third-party, then your own — each grouped.
```

### 7.2 Your own modules **[I]**

Any `.py` file you write is importable by its name (no extension). When a module is imported, *all* its top-level code runs. That raises a question: how do you write a file that can be *both* imported as a library *and* run as a script? The answer is the `if __name__ == "__main__":` idiom. Python sets the special variable `__name__` to `"__main__"` when a file is run directly, but to the module's name when it's imported — so code guarded by that check runs only on direct execution.

```python
# file: mathutils.py
def add(a, b):
    return a + b

PI = 3.14159

if __name__ == "__main__":      # True only when run as `python mathutils.py`
    print("testing:", add(2, 3))   # NOT executed when another file imports mathutils
```

```python
# file: main.py
import mathutils                       # runs mathutils' top level, but NOT its __main__ block
from mathutils import add
print(add(2, 3), mathutils.PI)
```

### 7.3 Packages **[I]**

A **package** groups related modules under a directory, giving you a namespace like `myapp.db.models`. An `__init__.py` file marks the directory as a "regular package" and runs when the package is first imported — a good place to re-export your public API so users can write `from myapp import User` instead of digging into submodules. Without `__init__.py` the directory can still act as a "namespace package" (3.3+), but include `__init__.py` for normal applications; it removes ambiguity and gives you that re-export hook.

```
myapp/
├── __init__.py          # marks the package; can re-export the public API
├── core.py
└── db/
    ├── __init__.py
    └── models.py
```

```python
from myapp.db.models import User      # ABSOLUTE import — preferred; unambiguous
from .models import User              # RELATIVE import (the leading . = "this package")
from ..core import setup              # .. = parent package (only valid inside a package)
# Run a package as a program by adding myapp/__main__.py, then: python -m myapp
```

> **The import search path.** Python looks for modules in the directories listed in `sys.path`: the script's directory (or cwd), then `PYTHONPATH` entries, then the standard library and `site-packages`. If an import fails with `ModuleNotFoundError`, the cause is almost always that the package isn't on `sys.path` — installing your project (`pip install -e .`) or running from the right directory fixes it.

---

## 8. Exceptions & Error Handling

Errors are not exceptional in real software — files go missing, networks fail, users type garbage. Python's strategy is **exceptions**: when something goes wrong, the normal flow stops and an exception object propagates up the call stack until some `try`/`except` handles it (or it reaches the top and crashes the program with a traceback). The Pythonic philosophy is **EAFP** ("Easier to Ask Forgiveness than Permission"): rather than checking every precondition up front (LBYL, "Look Before You Leap"), you attempt the operation and handle the failure. This is cleaner and avoids race conditions (the file could vanish between your check and your use).

The four-part `try` block has distinct roles you should keep straight: `try` holds the risky code; `except` handles specific failures; `else` runs only if *no* exception occurred (keep it small — just the code that depends on success); `finally` *always* runs (use it for cleanup that must happen win or lose).

```python
def parse_age(s):
    try:
        age = int(s)                      # the operation that might fail
    except ValueError as e:               # catch a SPECIFIC exception type, bind it to e
        print(f"not a number: {e}")
        return None
    except (TypeError, KeyError):         # catch several types with a tuple
        return None
    else:
        print("parsed fine")              # runs ONLY if the try block raised nothing
        return age
    finally:
        print("always runs (cleanup)")    # runs no matter what — even on return/raise

# RAISING an exception when a precondition is violated:
def withdraw(balance, amount):
    if amount > balance:
        raise ValueError(f"insufficient funds: {amount} > {balance}")
    return balance - amount

# CUSTOM exceptions — subclass Exception (never BaseException) to model your domain errors.
# Custom types let callers catch precisely (`except InsufficientFunds`) and carry data.
class InsufficientFunds(Exception):
    def __init__(self, needed, available):
        super().__init__(f"need {needed}, have {available}")
        self.needed = needed              # attach structured data for the handler
        self.available = available

# EXCEPTION CHAINING — preserve the original cause so the traceback tells the full story:
try:
    int("x")
except ValueError as e:
    raise RuntimeError("config invalid") from e   # sets __cause__; shows BOTH in the traceback
```

> **ExceptionGroup (3.11+).** When several operations can fail at once (e.g. concurrent tasks), an `ExceptionGroup` bundles multiple exceptions, and `except*` handles them selectively. This pairs with `asyncio.TaskGroup` (§13).

**Best practices, and the reasoning behind each:**
- **Catch the narrowest type you can.** `except Exception` (let alone bare `except:`) hides bugs and swallows `KeyboardInterrupt`/`SystemExit`, making your program impossible to Ctrl-C. Catch `ValueError`, not "everything."
- **Don't silence exceptions you can't handle.** An empty `except: pass` turns a loud failure into a silent wrong result. If you must, at least log it.
- **Use `finally` or context managers for cleanup** so resources are released even when an error fires (§9).
- **Add context when re-raising** with `raise ... from e` so the root cause survives.

---

## 9. Iterators, Generators & Context Managers

These three concepts share a theme: **controlling *how* you traverse data and *when* work happens**. Iterators give a uniform protocol for stepping through anything. Generators let you produce values lazily — computing each on demand instead of building everything up front, which is the key to handling huge or infinite streams in constant memory. Context managers guarantee setup/teardown pairs (open/close, acquire/release) run correctly even when errors strike. Master these and you write code that is both memory-efficient and leak-free.

### 9.1 Iterators **[I]**

The **iterator protocol** is the contract behind every `for` loop. An *iterable* is anything with `__iter__()` that returns an *iterator*; an iterator has `__next__()` that returns the next value or raises `StopIteration` when exhausted. A `for` loop is just sugar: it calls `iter()` once to get an iterator, then `next()` repeatedly until `StopIteration`. Knowing this demystifies loops and lets you build your own iterable objects.

```python
nums = [1, 2, 3]
it = iter(nums)             # iter() -> an iterator over the list
print(next(it))             # 1   (next() advances and returns)
print(next(it))             # 2
# next(it) once more gives 3; the call after that raises StopIteration.
# This is EXACTLY the machinery a `for` loop drives for you.

# A custom iterator implements __iter__ (returns self) and __next__:
class Countdown:
    def __init__(self, n):
        self.n = n
    def __iter__(self):
        return self                 # the object IS its own iterator
    def __next__(self):
        if self.n <= 0:
            raise StopIteration     # signal "done" — for loops catch this silently
        self.n -= 1
        return self.n + 1
print(list(Countdown(3)))           # [3, 2, 1]
```

### 9.2 Generators — lazy iterators with `yield` **[I]**

Writing iterators by hand is tedious. A **generator** is the easy way: any function containing `yield` becomes a generator function, and calling it returns a generator object (an iterator) *without running the body yet*. Each time you ask for the next value, the body runs up to the next `yield`, hands back that value, and **pauses with its entire local state frozen** — then resumes from exactly there on the next call. This pause/resume is what makes generators *lazy*: they compute one value at a time, on demand, so you can process a 50 GB file or an infinite sequence without ever holding it all in memory.

```python
def countdown(n):
    while n > 0:
        yield n              # produce a value and PAUSE here, remembering n
        n -= 1               # resumes here on the next next() call
for x in countdown(3):
    print(x)                 # 3 2 1

# The classic win: stream a huge file line-by-line, constant memory, never loading it all:
def read_big_file(path):
    with open(path) as f:
        for line in f:       # files are themselves lazy iterators of lines
            yield line.rstrip("\n")

# Infinite generators are fine because they're lazy — just don't list() them:
def naturals():
    n = 0
    while True:
        yield n
        n += 1

# Generator EXPRESSION — like a list comprehension but lazy, written with ( ):
squares = (x * x for x in range(1_000_000))   # builds NOTHING yet; no million-item list
print(next(squares))         # 0  (computed only when asked)
print(sum(x * x for x in range(5)))           # 30 — sum consumes lazily, no temp list
# Rule of thumb: use a generator expression when you'll consume once (e.g. feed to sum/any/max).

# yield from delegates to a sub-iterable (flattens generator composition):
def chain(a, b):
    yield from a
    yield from b
print(list(chain([1, 2], [3, 4])))   # [1, 2, 3, 4]
```

> **Generators are single-use.** Once exhausted, a generator is empty — iterate it again and you get nothing. If you need to traverse data multiple times, materialize it (`list(...)`) or recreate the generator.

### 9.3 Context managers — `with` **[I]**

A **context manager** guarantees that a paired setup and teardown both happen, *even if the code between them raises an exception*. The canonical case is files: `with open(...) as f:` opens the file, runs your block, and **closes it no matter what** — no leaked file handles. Under the hood, `with` calls `__enter__()` on entry (its return value is bound by `as`) and `__exit__()` on exit (always, including on exceptions). You'll use `with` constantly for files, locks, database connections, and network sessions.

```python
# `with` guarantees cleanup — the file is closed even if write() raises midway:
with open("data.txt", "w", encoding="utf-8") as f:
    f.write("hello")
# f is closed HERE, automatically, whether or not an error occurred.

# Multiple managers in one statement (or parenthesized, 3.10+, for long lists):
with open("in.txt") as src, open("out.txt", "w") as dst:
    dst.write(src.read())

# Write your own with __enter__ / __exit__:
import time
class Timer:
    def __enter__(self):
        self.start = time.perf_counter()
        return self                       # value bound to `as` (here: the Timer itself)
    def __exit__(self, exc_type, exc, tb):
        self.elapsed = time.perf_counter() - self.start
        print(f"took {self.elapsed:.3f}s")
        return False                      # False -> do NOT suppress exceptions (re-raise them)

with Timer():
    sum(range(1_000_000))

# The EASY way for simple cases — contextlib turns a generator into a context manager.
# Code before `yield` is setup; code in `finally` after `yield` is teardown.
from contextlib import contextmanager
@contextmanager
def timer():
    start = time.perf_counter()
    try:
        yield                             # the body of the `with` runs at this point
    finally:
        print(f"took {time.perf_counter() - start:.3f}s")   # runs even on exception

# Handy contextlib helpers:
from contextlib import suppress
with suppress(FileNotFoundError):         # cleaner than try/except/pass for "ignore if absent"
    Path("maybe.txt").unlink()
```

---

## 10. Decorators

A **decorator** is a function that takes a function (or class) and returns a *replacement* — almost always a wrapper that adds behaviour around the original. The `@decorator` syntax above a `def` is pure sugar: `@log_calls` over `def add` simply does `add = log_calls(add)` after defining `add`. Decorators exist because Python functions are first-class objects, so you can pass them around and rewrap them. The payoff is **separation of concerns**: cross-cutting behaviour — logging, timing, caching, retrying, access control, registration — lives in one reusable decorator instead of being copy-pasted into every function. **[I/A]**

The mechanics to internalize: the wrapper must accept `*args, **kwargs` so it can forward *any* call to the original, and you should apply `@functools.wraps(func)` to the wrapper so it inherits the original's name, docstring, and signature (otherwise introspection and `help()` show the meaningless wrapper instead).

```python
import functools

def log_calls(func):                       # a decorator: takes a function, returns one
    @functools.wraps(func)                 # copy func's __name__/__doc__ onto wrapper
    def wrapper(*args, **kwargs):          # accept ANY arguments so we can forward them
        print(f"calling {func.__name__}({args}, {kwargs})")
        result = func(*args, **kwargs)     # call the ORIGINAL, capturing its result
        print(f"-> {result}")
        return result                      # return it so the caller sees the real value
    return wrapper                         # this replaces the original function

@log_calls                                 # equivalent to: add = log_calls(add)
def add(a, b):
    return a + b
add(2, 3)                                  # prints the call, the result, returns 5

# PARAMETERIZED decorator (a decorator FACTORY): one extra layer of nesting.
# retry(times=3) is CALLED first and RETURNS the actual decorator.
def retry(times):
    def decorator(func):                   # the real decorator, with `times` captured
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == times - 1:
                        raise              # out of retries -> let it propagate
                    print(f"retry {attempt + 1} after {e}")
        return wrapper
    return decorator                       # retry(3) -> decorator -> wrapper

@retry(times=3)
def flaky():
    ...

# Stacking decorators: applied BOTTOM-UP. Here: add = log_calls(retry(3)(add)).
@log_calls
@retry(times=3)
def fetch_data():
    ...

# Extremely useful BUILT-IN decorators from functools:
from functools import lru_cache, cache, reduce
@cache                                     # memoize results forever (3.9+); huge for recursion
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)   # exponential -> linear with caching
# Use @lru_cache(maxsize=128) when you want a bounded cache that evicts least-recently-used.

# A class-based decorator (when the wrapper needs its own state, e.g. a call counter):
class CountCalls:
    def __init__(self, func):
        functools.update_wrapper(self, func)
        self.func = func
        self.count = 0
    def __call__(self, *args, **kwargs):   # __call__ makes the INSTANCE callable
        self.count += 1
        print(f"call #{self.count}")
        return self.func(*args, **kwargs)

@CountCalls
def greet(): print("hi")
greet(); greet()                           # call #1 / call #2
```

> **When NOT to decorate.** If the behaviour is specific to one call site, just write it inline. Decorators shine for genuinely cross-cutting concerns reused across many functions. Also remember a decorator can change a function's effective signature for type checkers — `functools.wraps` mitigates most of this, and `ParamSpec` (§11) preserves it precisely.

---

## 11. Type Hints

Python is dynamically typed, but you can **annotate** the types you intend, and that's hugely valuable on any codebase that outlives a weekend. Type hints are *optional* and, importantly, **not enforced at runtime** — the interpreter ignores them for execution. Their value comes from *tools*: a static type checker (**mypy** or **pyright**) reads them and flags mismatches *before* you run the code, and your editor uses them for autocomplete, inline docs, and refactoring. Think of type hints as machine-checked documentation that catches a whole class of bugs (passing a `str` where an `int` is expected, forgetting a `None` case) without any runtime cost. **[I/A]**

Modern Python (3.9+/3.10+/3.12+) made the syntax pleasant: use built-in generics (`list[int]`, not `List[int]`), use `X | Y` for unions and `X | None` instead of `Optional[X]`, and use the `[T]` syntax for generics with no `TypeVar` import.

```python
def greet(name: str, times: int = 1) -> str:    # param types + return type after ->
    return f"hi {name} " * times

# Variables and containers (modern syntax — no need to import List/Dict from typing):
count: int = 0
names: list[str] = []                  # a list of strings
scores: dict[str, int] = {}            # str keys -> int values
matrix: list[list[float]] = []         # nested generics compose naturally
maybe: int | None = None               # union; `X | None` replaces the old Optional[X]
either: int | str = 0                  # a value that's an int OR a str

# Callables and tuples:
from collections.abc import Callable, Iterable, Sequence, Mapping
handler: Callable[[int, str], bool]    # takes (int, str), returns bool
coords: tuple[float, float]            # a FIXED 2-tuple of floats
row: tuple[int, ...]                   # a variable-length tuple of ints
# Prefer abstract types for PARAMETERS (accept more): Iterable/Sequence/Mapping over list/dict.
def total(values: Iterable[int]) -> int:   # accepts lists, tuples, generators, ...
    return sum(values)

# Protocols — STRUCTURAL typing ("duck typing", but statically checked).
# Any object with the right METHODS matches, regardless of inheritance.
from typing import Protocol
class SupportsClose(Protocol):
    def close(self) -> None: ...
def shutdown(x: SupportsClose) -> None:
    x.close()                          # any object with a close() satisfies this — no base class

# TypedDict (shape of a dict), Literal (a fixed set of allowed values):
from typing import TypedDict, Literal, Final
class UserDict(TypedDict):
    name: str
    age: int
Mode = Literal["r", "w", "a"]          # only these three strings are valid
def open_file(path: str, mode: Mode) -> None: ...
MAX_RETRIES: Final = 3                  # Final -> checkers flag any reassignment

# Generics (3.12+ syntax — declare the type parameter inline, no TypeVar import):
def first[T](items: list[T]) -> T:     # T is inferred from the argument; return matches
    return items[0]
reveal = first([1, 2, 3])              # a checker infers reveal: int

class Box[T]:                          # a generic class
    def __init__(self, value: T) -> None:
        self.value = value
    def get(self) -> T:
        return self.value
b: Box[str] = Box("hi")

# NewType for distinct semantic types; cast / Any as escape hatches (use sparingly):
from typing import NewType, Any, cast
UserId = NewType("UserId", int)        # a distinct int the checker won't confuse with others
raw: Any = get_json()                  # Any disables checking — a deliberate opt-out
n = cast(int, raw["count"])            # assert a type to the checker (no runtime effect)
```

**Gradual typing.** You don't have to annotate everything at once — add hints where they pay off (public functions, tricky data) and leave the rest. Checkers treat un-annotated code permissively.

**Running a checker:** `mypy myapp/` or `pyright`. Most teams in 2026 run **ruff** for linting/formatting plus **pyright** or **mypy** for types, often in CI so type errors block merges.

| Hint | Means |
|---|---|
| `list[int]`, `dict[str, int]` | concrete container of element types |
| `X \| Y`, `X \| None` | union / optional |
| `Iterable[T]`, `Sequence[T]`, `Mapping[K, V]` | accept-broad abstract types (good for params) |
| `Callable[[A, B], R]` | function taking A, B and returning R |
| `tuple[int, str]` vs `tuple[int, ...]` | fixed-shape vs variable-length |
| `Literal["a", "b"]` | one of an exact set of values |
| `Protocol` | structural ("duck") typing |
| `TypedDict` | a dict with a known key/value shape |
| `Final` | must not be reassigned |
| `Any` | turn checking off for this value (escape hatch) |

---

## 12. File System, OS & Command Execution

This is the systems-programming core of Python: reading and writing files, manipulating paths portably, inspecting the operating system and environment, and shelling out to run other programs. This is where Python shines as a "glue" and automation language — and where cross-platform care matters most, because Windows and Unix differ on path separators, line endings, available commands, and how executables are found. **[I]**

### 12.1 Reading & writing files

File I/O follows a simple lifecycle: **open** a file (getting a file object), **read or write** through it, then **close** it to flush buffers and release the OS handle. Always use `with` so closing is automatic even on error (§9), and always pass `encoding="utf-8"` explicitly — relying on the platform default is a classic source of "works on my machine" bugs (Windows historically defaulted to a non-UTF-8 codepage). For large files, *iterate* line by line instead of `.read()`-ing the whole thing into memory.

```python
# Small files — read the whole content (with `with`, the file auto-closes):
with open("notes.txt", encoding="utf-8") as f:   # default mode is "r" (read text)
    text = f.read()                              # entire contents as one string
with open("notes.txt", encoding="utf-8") as f:
    lines = f.readlines()                        # list of lines (each keeps its \n)

# Writing — "w" TRUNCATES (erases first), "a" APPENDS, "x" fails if the file exists:
with open("out.txt", "w", encoding="utf-8") as f:
    f.write("line 1\n")                          # write() does NOT add a newline for you
    f.writelines(["line 2\n", "line 3\n"])       # writes each string as-is (add \n yourself)

# Line by line — memory-friendly for huge files (f is a LAZY iterator of lines):
with open("big.log", encoding="utf-8") as f:
    for line in f:
        process(line.rstrip("\n"))               # strip the trailing newline

# Binary mode ("b") — no encoding, you get/give bytes (images, archives, protocols):
with open("img.png", "rb") as f:
    data = f.read()                              # bytes, not str
with open("copy.png", "wb") as f:
    f.write(data)
```

| Mode | Meaning |
|---|---|
| `"r"` | read text (default), error if missing |
| `"w"` | write text, **truncate** (or create) |
| `"a"` | append text (or create) |
| `"x"` | create exclusively, error if it exists |
| `"r+"` | read **and** write (no truncate) |
| add `"b"` | binary, e.g. `"rb"`, `"wb"` — read/write `bytes` |

### 12.2 `pathlib.Path` — the modern path API (prefer this over `os.path`)

`pathlib` models a filesystem path as an **object** with methods and properties, which is far cleaner than the old string-based `os.path` functions. Its standout feature is the `/` operator for joining path segments — it builds correct paths on any OS without you ever typing a separator, sidestepping the Windows-backslash vs Unix-forward-slash mess entirely. A `Path` also bundles existence checks, reading/writing shortcuts, directory creation, and globbing. Make `pathlib` your default for anything path-related.

```python
from pathlib import Path

p = Path("data") / "reports" / "2026.csv"   # `/` joins segments — correct on every OS
print(p)                  # data\reports\2026.csv on Windows; data/reports/2026.csv elsewhere
# Parts of a path (no filesystem access needed — pure string logic):
print(p.name)             # 2026.csv   (final component)
print(p.stem)             # 2026       (name without the suffix)
print(p.suffix)           # .csv       (the extension, with the dot)
print(p.parent)           # data\reports   (everything but the last part)
print(p.parts)            # ('data', 'reports', '2026.csv')
print(p.with_suffix(".json"))   # data\reports\2026.json   (swap the extension)
print(p.absolute())       # make absolute relative to cwd
print(p.resolve())        # absolute + resolve symlinks and `..`

# Filesystem queries:
print(p.exists(), p.is_file(), p.is_dir())

# Read/write shortcuts (open + read/write + close, in one call):
Path("hello.txt").write_text("hi", encoding="utf-8")
print(Path("hello.txt").read_text(encoding="utf-8"))
Path("data.bin").write_bytes(b"\x00\x01")

# Create and remove:
Path("logs/sub").mkdir(parents=True, exist_ok=True)   # like `mkdir -p`; no error if it exists
Path("tmp.txt").unlink(missing_ok=True)               # delete a file; ignore if absent
Path("emptydir").rmdir()                              # remove an EMPTY directory

# List a directory and match patterns with glob:
for child in Path(".").iterdir():        # immediate children (files and dirs)
    print(child)
for csv in Path("data").glob("*.csv"):   # non-recursive pattern match
    print(csv)
for py in Path(".").rglob("*.py"):       # RECURSIVE (searches all subdirectories)
    print(py)

# Useful well-known locations:
print(Path.home())        # the user's home directory
print(Path.cwd())         # the current working directory
print(Path(__file__).parent)   # the directory containing the current script
```

### 12.3 `os` and `shutil` for filesystem ops

`os` exposes low-level OS services (process info, environment, directory walking), and `shutil` ("shell utilities") provides high-level file operations that mirror common shell commands: copy, move, recursive delete, archive. Use `pathlib` for path *manipulation* and individual file ops; reach into `os`/`shutil` for things `pathlib` doesn't cover — recursively walking a tree, copying whole directories, making zip archives, or finding an executable on `PATH`.

```python
import os, shutil

os.getcwd()                       # current working directory (string)
os.chdir("/tmp")                  # change the process's cwd
os.listdir(".")                   # bare names in a directory (no path prefix)
os.makedirs("a/b/c", exist_ok=True)   # create nested dirs (pathlib's mkdir(parents=True))
os.remove("file.txt")             # delete a file (== Path.unlink)
os.rename("old.txt", "new.txt")   # rename/move (fails across filesystems / if dest exists)
os.replace("a.txt", "b.txt")      # like rename but ATOMIC and overwrites the destination

# Walk an entire directory tree. At each directory you get its path, its subdir NAMES,
# and its file NAMES. Edit `dirnames` IN PLACE to prune which subdirs get visited.
for dirpath, dirnames, filenames in os.walk("project"):
    if ".git" in dirnames:
        dirnames.remove(".git")       # skip descending into .git
    for name in filenames:
        full = os.path.join(dirpath, name)   # build the full path
        print(full)

# shutil — higher-level operations resembling shell commands:
shutil.copy("a.txt", "b.txt")        # copy a file's contents (copy2 also preserves metadata)
shutil.copytree("src", "dst")        # copy an entire directory tree
shutil.move("a.txt", "dir/")         # move a file or directory
shutil.rmtree("build")               # recursively delete a directory — DANGEROUS, no undo
print(shutil.which("git"))           # locate an executable on PATH (like `which`/`where`)
print(shutil.disk_usage("/"))        # (total, used, free) in bytes

# Archives — zip/tar a folder or unpack one, in one call:
shutil.make_archive("backup", "zip", "myfolder")   # -> backup.zip
shutil.unpack_archive("backup.zip", "restored")    # extract into restored/

# Temporary files and directories — auto-cleaned, race-free, OS-appropriate location:
import tempfile
with tempfile.NamedTemporaryFile(mode="w", delete=False, suffix=".txt") as tf:
    tf.write("scratch")
    tmp_path = tf.name               # keep the path; delete=False so it survives the block
with tempfile.TemporaryDirectory() as d:   # the whole directory is removed on exit
    print("temp dir:", d)
```

### 12.4 OS & system information

When a script must adapt to its environment — branch on the OS, read the Python version, find the interpreter path, count CPUs — use `sys` (interpreter/runtime info) and `platform` (OS/hardware info). `sys.argv` is your raw command-line; `sys.executable` is the exact Python running your code (use it to re-invoke pip safely, see §12.6).

```python
import os, sys, platform

print(sys.argv)              # list of command-line args; sys.argv[0] is the script path
print(sys.version)           # full Python version string
print(sys.version_info[:2])  # (3, 13) — easy to compare: if sys.version_info >= (3, 12)
print(sys.platform)          # 'win32', 'linux', or 'darwin' (macOS)
print(sys.executable)        # absolute path to THIS python interpreter
print(sys.path[:3])          # the module search path (see §7.3)

print(platform.system())     # 'Windows' / 'Linux' / 'Darwin'
print(platform.release(), platform.machine())   # OS release, CPU architecture (e.g. AMD64)
print(platform.python_version())

print(os.name)               # 'nt' on Windows, 'posix' on Unix-likes
print(os.cpu_count())        # number of logical CPUs (useful for sizing pools, §13)
print(os.getpid())           # this process's id

import getpass
print(getpass.getuser())     # current username — more portable than os.getlogin()
```

### 12.5 Environment variables

**Environment variables** are key/value strings the OS hands to every process — the standard channel for configuration and secrets (API keys, database URLs) that shouldn't be hard-coded. Read them via `os.environ` (a dict-like mapping). Always use `.get()` with a default for optional config so a missing variable doesn't crash with `KeyError`. For local development, a `.env` file loaded by `python-dotenv` is the common pattern (never commit real secrets to version control).

```python
import os

print(os.environ.get("PATH"))            # read; returns None if the variable is absent
print(os.environ.get("API_KEY", ""))     # supply a default for optional config
os.environ["MY_FLAG"] = "1"              # set for THIS process and any children it spawns
for key, value in os.environ.items():    # iterate everything (don't print secrets!)
    ...

# Loading a .env file in development (requires: pip install python-dotenv):
from dotenv import load_dotenv
load_dotenv()                            # parses ./.env and injects pairs into os.environ
db_url = os.environ["DATABASE_URL"]      # now available; use [] when it's truly required
```

### 12.6 Executing external commands — `subprocess`

`subprocess` runs *other programs* from Python — git, npm, ffmpeg, your own scripts — capturing their output and exit codes. The modern, recommended entry point is `subprocess.run()`. Two rules keep you safe and portable: **pass the command as a list of arguments** (not a single string) so the OS doesn't reinterpret spaces or special characters, and **avoid `shell=True` with any untrusted input** because it invites command injection. Use `check=True` to turn a non-zero exit code into a Python exception, `capture_output=True` + `text=True` to collect stdout/stderr as strings, and `timeout=` to avoid hanging forever.

```python
import subprocess, sys, os

# The standard call: args as a LIST, no shell involved.
result = subprocess.run(
    ["git", "rev-parse", "--abbrev-ref", "HEAD"],   # program + args, each its own list item
    capture_output=True,     # collect stdout & stderr (else they go to the terminal)
    text=True,               # decode bytes -> str using the locale encoding
    check=True,              # raise CalledProcessError if the exit code is non-zero
    cwd=".",                 # run in this working directory
    timeout=30,              # kill it and raise TimeoutExpired after 30s
    env={**os.environ, "GIT_PAGER": "cat"},   # custom environment (copy + override)
)
print("branch:", result.stdout.strip())
print("exit code:", result.returncode)        # 0 on success (check=True guarantees it here)

# Running real tools:
subprocess.run(["pip", "install", "rich"], check=True)
subprocess.run([sys.executable, "-m", "pip", "install", "rich"])  # SAFEST: this interpreter's pip
subprocess.run(["npm", "install"], cwd="frontend")
subprocess.run(["python", "build.py", "--release"])

# Capture output and parse it (e.g. JSON):
import json
out = subprocess.run(["npm", "ls", "--json"], capture_output=True, text=True).stdout
deps = json.loads(out)

# Handling failure explicitly (without check=True):
proc = subprocess.run(["grep", "TODO", "missing.py"], capture_output=True, text=True)
if proc.returncode != 0:
    print("grep failed or found nothing:", proc.stderr)

# shell=True runs the command THROUGH the shell — convenient (pipes, globs) but a
# COMMAND-INJECTION risk. Only ever use it with a fixed, trusted, constant string.
subprocess.run("echo hello && echo world", shell=True)   # fine: fully literal
# DANGER — never interpolate user input into a shell string:
# subprocess.run(f"rm {user_supplied}", shell=True)   # user_supplied="x; rm -rf /" = disaster

# Streaming output LIVE with Popen (when you want to react as lines arrive, not wait):
proc = subprocess.Popen(
    ["ping", "-n", "3", "127.0.0.1"],     # Windows uses -n; Unix uses -c
    stdout=subprocess.PIPE, text=True,
)
for line in proc.stdout:                  # read lines as the child prints them
    print("ping:", line.rstrip())
proc.wait()                               # wait for it to finish; sets returncode

# Piping one process into another (cmd1 | cmd2):
p1 = subprocess.Popen(["echo", "c\nb\na"], stdout=subprocess.PIPE, text=True)
p2 = subprocess.run(["sort"], stdin=p1.stdout, capture_output=True, text=True)
print(p2.stdout)                          # a\nb\nc
```

> **⚡ Windows note:** some commands are shell *built-ins* (`dir`, `copy`, `del`), not real executables, so they must run via `subprocess.run(["cmd", "/c", "dir"])`. Tools installed by npm are often `.cmd` shims (e.g. `npm.cmd`); `shutil.which("npm")` finds the right one to pass to `run`. On Unix you can call `["ls", "-la"]` directly. When in doubt, resolve the program with `shutil.which()` first.

### 12.7 Building a CLI with `argparse`

A command-line interface needs to parse arguments, validate them, generate `--help`, and report usage errors. `argparse` (in the stdlib) does all of this declaratively: you describe your **positional** arguments (required, order-based) and **options** (`-w`/`--width`, often with values or as boolean flags), and `argparse` parses `sys.argv`, converts types, enforces `choices`, and auto-builds the help text. Returning an integer from `main()` and feeding it to `SystemExit` gives you proper Unix exit codes.

```python
import argparse

def main():
    parser = argparse.ArgumentParser(description="Resize images in a folder.")
    parser.add_argument("folder", help="folder to process")            # POSITIONAL (required)
    parser.add_argument("-w", "--width", type=int, default=800,        # OPTION with a value
                        help="target width in px")
    parser.add_argument("-v", "--verbose", action="store_true",        # boolean FLAG (no value)
                        help="print each file")
    parser.add_argument("--format", choices=["jpg", "png"], default="jpg")  # restricted choices
    parser.add_argument("files", nargs="*", help="specific files")     # zero or more values

    args = parser.parse_args()           # parses sys.argv; on error prints usage and exits(2)
    if args.verbose:                     # access by the long name (or dest)
        print(f"resizing {args.folder} to width {args.width} as {args.format}")
    return 0                             # 0 = success

if __name__ == "__main__":
    raise SystemExit(main())             # use main()'s return value as the process exit code
```

Run: `python resize.py photos -w 1024 --verbose`. For richer CLIs, **Typer** (build commands from type-hinted functions) and **Click** are popular libraries; **Rich** adds colors, tables, and progress bars for polished terminal output.

### 12.8 Reading stdin & exit codes

Well-behaved CLI tools participate in the Unix pipeline: they can read from **standard input** (so `cat file | tool` works), and they signal success or failure through their **exit code** (0 = success, non-zero = error). Read piped data via `sys.stdin`; detect whether you're attached to a terminal or a pipe with `isatty()` (so you can prompt interactively only when it makes sense). Set the exit code with `sys.exit()` or by returning from `main()` into `SystemExit`.

```python
import sys

# Read everything piped in (e.g. `cat file | python tool.py`):
data = sys.stdin.read()
# ...or line by line, lazily, for large input:
for line in sys.stdin:
    print(line.rstrip())

# Adapt behaviour to interactive vs piped usage:
if sys.stdin.isatty():
    name = input("Your name: ")          # safe to prompt — a human is at the keyboard
else:
    name = sys.stdin.readline().strip()  # reading from a pipe/redirect

# Exit codes communicate success/failure to the caller and to shell `&&`/`||`:
sys.exit(0)                              # 0 = success
# sys.exit("error message")             # prints to stderr and exits with code 1
```

---

## 13. Concurrency & Parallelism

"Doing many things at once" splits into two distinct goals, and confusing them is the #1 source of concurrency mistakes in Python. **Concurrency** is *structuring* a program so multiple tasks make progress by interleaving (great for waiting on I/O); **parallelism** is *literally* running computations at the same instant on multiple CPU cores (needed for CPU-bound work). Python gives you three tools, each suited to a different job — and the right choice hinges almost entirely on whether your bottleneck is **I/O-bound** (waiting on network/disk) or **CPU-bound** (crunching numbers). **[A]**

| Model | Best for | Why it works | Module |
|---|---|---|---|
| `threading` | I/O-bound (network, disk) | the GIL is released while a thread waits on I/O, so others run | `threading`, `concurrent.futures.ThreadPoolExecutor` |
| `multiprocessing` | CPU-bound | separate processes = separate interpreters = no shared GIL | `multiprocessing`, `ProcessPoolExecutor` |
| `asyncio` | very high-concurrency I/O (thousands of sockets) | one thread, cooperative scheduling, tiny per-task overhead | `asyncio` |

### 13.1 The GIL — why it exists and what it actually does

The **Global Interpreter Lock** is a single mutex inside CPython that allows only **one thread to execute Python bytecode at a time**. It exists because CPython's memory management (reference counting) isn't thread-safe; the GIL makes the interpreter simple, fast for single-threaded code, and safe for C extensions — at the cost of true multi-core parallelism for pure-Python code.

The practical consequences are precise and worth memorizing:
- **CPU-bound Python in threads gets no speedup** (often a slight slowdown from lock contention), because only one thread runs bytecode at any moment. For CPU work, use **processes** (each has its own GIL) or libraries that release the GIL in C (NumPy).
- **I/O-bound work in threads scales well**, because a thread *releases the GIL while blocked on I/O* (reading a socket, waiting on disk). While one thread waits, another runs. This is why `ThreadPoolExecutor` is excellent for fetching many URLs or files concurrently.

```python
import sys
# On 3.13+ you can check whether the GIL is active in the current build:
print(sys._is_gil_enabled())   # True on the standard build; False on the free-threaded build
```

> **⚡ 2026:** the **free-threaded** build (`python3.13t`, PEP 703) removes the GIL, letting threads run CPU-bound Python in genuine parallel. It is officially supported but **opt-in**, and some C extensions still need updates to be safe and fast without the GIL. Until it's the default, the rule of thumb above (threads for I/O, processes for CPU) remains the safe default.

### 13.2 Threads & the executor

A **thread** runs concurrently within your process, sharing memory with other threads. Raw `threading.Thread` is available, but the high-level `concurrent.futures.ThreadPoolExecutor` is almost always better: it manages a pool of worker threads, you `submit` callables and get back **`Future`** objects (handles to results that will exist later), and `as_completed` lets you process results in finish order. Because threads share memory, any shared *mutable* state needs a `Lock` to avoid races.

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import urllib.request

def fetch(url):
    with urllib.request.urlopen(url) as r:
        return url, len(r.read())       # this BLOCKS on the network -> GIL released here

urls = ["https://example.com", "https://python.org"]
with ThreadPoolExecutor(max_workers=8) as pool:        # pool of 8 worker threads
    futures = [pool.submit(fetch, u) for u in urls]    # submit returns Future objects
    for fut in as_completed(futures):                  # yields each Future as it finishes
        url, size = fut.result()                       # .result() returns it (or re-raises)
        print(url, size)
    # Simpler when you want results in INPUT order and don't need streaming:
    # results = list(pool.map(fetch, urls))

# Protecting shared state with a Lock (only one thread in the block at a time):
import threading
counter = 0
lock = threading.Lock()
def bump():
    global counter
    with lock:                          # acquire on enter, release on exit (even on error)
        counter += 1                    # without the lock this is a race (read-modify-write)
```

### 13.3 Multiprocessing for CPU work

For CPU-bound work, sidestep the GIL by using **multiple processes** instead of threads — each process is a separate Python interpreter with its own GIL and memory, so they truly run in parallel across cores. `ProcessPoolExecutor` mirrors the thread API. The trade-offs: arguments and return values must be **picklable** (they're serialized to cross the process boundary), there's startup and communication overhead (so it pays off only for substantial work), and on Windows/macOS the default `spawn` start method **re-imports your module in each child**, which is exactly why you must guard the entry point with `if __name__ == "__main__":` — otherwise children re-run your top-level code and you get an infinite spawn loop.

```python
from concurrent.futures import ProcessPoolExecutor

def heavy(n):                            # a real CPU-bound task (no I/O)
    return sum(i * i for i in range(n))

if __name__ == "__main__":               # REQUIRED on Windows/macOS (spawn) — guard the entry!
    with ProcessPoolExecutor() as pool:  # defaults to os.cpu_count() workers
        results = list(pool.map(heavy, [1_000_000, 2_000_000, 3_000_000]))
        print(results)
# Rule: use processes for CPU-bound parallelism; the data you pass must be picklable.
```

### 13.4 asyncio — cooperative concurrency in one thread

`asyncio` achieves massive I/O concurrency with a *single* thread by **cooperative multitasking**: you mark functions `async def` (coroutines) and use `await` at points where they'd otherwise block; at each `await`, the coroutine voluntarily yields control to an **event loop**, which runs other ready coroutines meanwhile. There are no threads and no locks for this model, and the per-task overhead is tiny, so you can have *tens of thousands* of concurrent connections. The catch — and it's a sharp one — is that everything must be non-blocking: calling a *blocking* function (a synchronous `requests.get`, `time.sleep`, heavy CPU) inside a coroutine freezes the entire event loop. You need async-aware libraries (`aiohttp`, `httpx`, async DB drivers).

```python
import asyncio

async def fetch(name, delay):
    await asyncio.sleep(delay)            # NON-blocking sleep — yields control to the loop
    return f"{name} done"                 # (a blocking time.sleep here would stall everything)

async def main():
    # gather: run coroutines concurrently and collect results in input order.
    results = await asyncio.gather(
        fetch("a", 1),
        fetch("b", 2),
    )
    print(results)                        # ['a done', 'b done'] after ~2s total, not 3

    # TaskGroup (3.11+) — structured concurrency: the `async with` block won't exit until
    # all child tasks finish, and if any raises, the rest are cancelled (errors come as a group).
    async with asyncio.TaskGroup() as tg:
        tg.create_task(fetch("x", 1))
        tg.create_task(fetch("y", 1))

asyncio.run(main())                       # create the event loop, run main() to completion

# Bridging blocking code into async without stalling the loop — offload to a thread:
async def use_blocking():
    result = await asyncio.to_thread(some_blocking_function, arg)   # runs it in a thread pool
    return result
```

**Choosing in practice:** fetching 500 URLs? `asyncio` (or threads). Resizing 500 images / hashing files / numeric crunching? `multiprocessing`. A handful of blocking I/O calls? `ThreadPoolExecutor` is the least-fuss option. Don't mix blocking calls into `asyncio` without `to_thread`.

---

## 14. Standard Library Highlights

Python's tagline is "batteries included," and it's earned: the standard library ships modules for dates, JSON, regex, specialized collections, functional tools, logging, and even a full SQL database — all offline, no `pip install`. Knowing what's already there saves you from reinventing wheels and from pulling in dependencies you don't need. This section tours the modules you'll reach for most. **[I]**

```python
# datetime — dates, times, and arithmetic. ALWAYS prefer timezone-AWARE datetimes in UTC;
# "naive" datetimes (no tzinfo) cause subtle, painful bugs across timezones/DST.
from datetime import datetime, timedelta, timezone, date
now = datetime.now(timezone.utc)         # aware, in UTC — the safe default for stored times
print(now.isoformat())                   # 2026-06-21T... +00:00  (machine-friendly)
print(now + timedelta(days=7, hours=3))  # arithmetic with timedelta
print(datetime.strptime("2026-06-21", "%Y-%m-%d"))   # PARSE a string -> datetime
print(now.strftime("%Y-%m-%d %H:%M"))                # FORMAT a datetime -> string
print((date(2026, 12, 25) - date.today()).days)      # day count between dates

# json — serialize/deserialize between Python objects and JSON text.
import json
s = json.dumps({"a": 1, "b": [2, 3]}, indent=2)      # object -> pretty JSON string
obj = json.loads(s)                                   # JSON string -> Python object
with open("data.json") as f: obj = json.load(f)       # read+parse from a file
with open("out.json", "w") as f: json.dump(obj, f)    # serialize to a file
# Tip: json.dumps(data, default=str) lets you serialize dates/Decimals as strings.

# re — regular expressions for pattern matching/extraction in text.
import re
m = re.search(r"(\d{4})-(\d{2})-(\d{2})", "date: 2026-06-21")   # find first match anywhere
if m:
    print(m.group(0), m.group(1), m.groups())   # whole match, first group, all groups tuple
print(re.findall(r"\w+", "a b c"))               # ['a', 'b', 'c'] — every match
print(re.sub(r"\s+", "_", "a  b   c"))           # 'a_b_c' — search-and-replace
pattern = re.compile(r"^\d+$")                   # PRECOMPILE when reusing a pattern in a loop
print(bool(pattern.match("123")))                # match() anchors at the START of the string

# collections — specialized containers that solve common problems elegantly.
from collections import Counter, defaultdict, namedtuple, deque
print(Counter("mississippi"))                    # Counter({'s':4,'i':4,'p':2,'m':1}) — tally
print(Counter("mississippi").most_common(2))     # [('s',4), ('i',4)] — top-N
dd = defaultdict(list); dd["x"].append(1)        # no KeyError — missing keys auto-create []
Pt = namedtuple("Pt", "x y"); p = Pt(1, 2)       # a lightweight, named, immutable record
print(p.x, p.y)                                  # 1 2 — access by name, not just index
q = deque([1, 2, 3], maxlen=5); q.appendleft(0)  # O(1) at BOTH ends; maxlen makes a ring buffer

# itertools & functools — building blocks for iteration and functional style.
import itertools, functools
print(list(itertools.chain([1, 2], [3, 4])))         # [1,2,3,4] — concatenate iterables
print(list(itertools.combinations([1,2,3], 2)))      # [(1,2),(1,3),(2,3)]
print(list(itertools.islice(itertools.count(), 5)))  # [0,1,2,3,4] — slice an infinite stream
print(list(itertools.groupby("aabbbc")))             # group consecutive equal items
print(functools.reduce(lambda a, b: a + b, [1,2,3,4]))  # 10 — fold a sequence to one value

# logging — structured, level-based output. Use this, NOT print(), in real applications:
# it gives you levels, timestamps, module names, and configurable destinations.
import logging
logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(name)s: %(message)s")
log = logging.getLogger(__name__)        # a logger named after the current module
log.debug("won't show (below INFO)"); log.info("started"); log.warning("careful")
log.error("oops"); log.exception("with traceback")   # call .exception() inside an except block

# sqlite3 — a complete, file-based SQL database built into Python. Zero setup, great for
# local apps, caches, and prototypes. Use ? placeholders for params to prevent SQL injection.
import sqlite3
con = sqlite3.connect("app.db")          # creates the file if it doesn't exist
con.execute("CREATE TABLE IF NOT EXISTS users(id INTEGER PRIMARY KEY, name TEXT)")
con.execute("INSERT INTO users(name) VALUES (?)", ("Ada",))   # ? = SAFE parameter binding
con.commit()                             # changes aren't persisted until you commit
for row in con.execute("SELECT * FROM users"):
    print(row)                           # (1, 'Ada')
con.close()
```

**Other gems worth knowing exist:** `csv` (read/write CSV safely), `configparser` (INI files), `hashlib` (SHA/MD5 digests), `secrets` (cryptographically secure tokens — use over `random` for security), `random` (general randomness), `math`/`statistics`, `urllib.request` (basic HTTP without dependencies), `dataclasses` (§6), `enum` (§6), `textwrap`, `pprint`, and `zoneinfo` (IANA timezones).

---

## 15. Testing

Tests are executable specifications: they state what your code *should* do and verify it still does after every change. The payoff is **confidence to refactor** — without tests, every edit is a gamble. **pytest** is the de-facto standard because it's far less ceremonious than the stdlib `unittest`: you write plain functions named `test_*` and use plain `assert` statements, and pytest's assertion rewriting produces rich failure messages showing exactly what differed. **[I]**

The core ideas: pytest **discovers** `test_*.py` files and `test_*` functions automatically; **fixtures** provide reusable setup/teardown (and dependency injection by parameter name); **parametrize** runs one test across many inputs; and `pytest.raises` asserts that code *fails* the way you expect.

```python
# file: test_math.py   (pytest auto-discovers test_*.py files and test_* functions)
import pytest
from mathutils import add

def test_add():
    assert add(2, 3) == 5            # plain assert — pytest rewrites it for a detailed message

# parametrize: run the SAME test logic across many input/expected pairs (each is a separate
# test case in the report, so you see exactly which input failed).
@pytest.mark.parametrize("a,b,expected", [(1, 1, 2), (2, 3, 5), (0, 0, 0), (-1, 1, 0)])
def test_add_cases(a, b, expected):
    assert add(a, b) == expected

# Asserting that code RAISES — testing the failure path is as important as the happy path.
def test_raises():
    with pytest.raises(ValueError):
        int("not a number")

# Fixtures: reusable setup, injected by matching the parameter NAME to the fixture name.
# Code before `yield` is setup; code after `yield` is teardown (runs even if the test fails).
@pytest.fixture
def sample_user():
    user = {"name": "Ada"}           # setup
    yield user                       # the test receives this value
    # teardown here (close files, delete temp data, etc.)

def test_user(sample_user):          # pytest sees the name and passes the fixture's value
    assert sample_user["name"] == "Ada"

# Built-in fixtures worth knowing: tmp_path (a fresh temp dir Path), monkeypatch (patch
# attributes/env vars and auto-undo), capsys (capture printed output).
def test_writes_file(tmp_path):
    target = tmp_path / "out.txt"
    target.write_text("hi")
    assert target.read_text() == "hi"
```

```bash
pip install pytest pytest-cov
pytest                 # discover and run every test
pytest -v              # verbose: list each test and its outcome
pytest -k add          # run only tests whose name matches "add"
pytest -x              # stop at the first failure
pytest --cov=myapp     # measure code coverage (needs pytest-cov)
```

**Mocking** replaces a real dependency (network, time, randomness, a database) with a controllable stand-in, so tests stay fast, deterministic, and isolated from the outside world. Patch *where the name is looked up*, not where it's defined — a common pitfall.

```python
from unittest.mock import patch

def test_api():
    # Patch the requests.get that myapp.client actually uses, and script its return value.
    with patch("myapp.client.requests.get") as mock_get:
        mock_get.return_value.json.return_value = {"ok": True}
        # ... call the code under test that calls requests.get ...
        mock_get.assert_called_once()        # verify the interaction happened exactly once
```

> **What to test (and what not).** Test behaviour and edge cases (empty input, boundaries, error paths), not trivial getters or the standard library. Aim for fast, independent tests; a slow or flaky suite is one people stop running.

---

## 16. Packaging & Distribution

Eventually you'll want to *share* code — install it across projects, hand it to teammates, or publish it. Packaging turns a directory of `.py` files into an installable, versioned **distribution** (a wheel `.whl` and a source `.tar.gz`). Modern packaging centers on `pyproject.toml`: it declares your metadata, dependencies, build backend, and any console-script entry points, and standard tools (`build`, `twine`/`uv`) produce and publish the artifacts. **[A]**

```toml
# pyproject.toml — the single source of truth for building and distributing your project.
[build-system]
requires = ["hatchling"]              # the build backend (others: setuptools, flit, pdm)
build-backend = "hatchling.build"

[project]
name = "mytool"
version = "1.0.0"
description = "A handy CLI"
readme = "README.md"
requires-python = ">=3.12"
dependencies = ["rich>=13"]           # what users need installed alongside your package
authors = [{name = "Ada", email = "ada@example.com"}]

[project.optional-dependencies]
dev = ["pytest", "mypy", "ruff"]      # install with: pip install mytool[dev]

[project.scripts]
mytool = "mytool.cli:main"            # installs a `mytool` terminal command -> cli.py:main()
```

```bash
pip install build twine
python -m build                # produce dist/*.whl (wheel) and dist/*.tar.gz (sdist)
twine upload dist/*            # publish to PyPI (or `uv publish`); test first on TestPyPI
pip install -e .              # EDITABLE install: your source is used live, no reinstall on edit
```

The recommended layout is the **`src/` layout**, which prevents a subtle bug where tests accidentally import your package from the working directory instead of the installed copy:

```
mytool/
├── pyproject.toml
├── README.md
├── tests/
│   └── test_cli.py
└── src/
    └── mytool/
        ├── __init__.py      # often defines __version__ and re-exports the public API
        └── cli.py
```

> **Versioning.** Follow **SemVer** (`MAJOR.MINOR.PATCH`): bump PATCH for fixes, MINOR for backward-compatible features, MAJOR for breaking changes. Your version is a promise to users about compatibility.

---

## 17. Performance, Idioms & Gotchas

This section is the "wisdom" layer: the recurring traps that bite everyone once, the idioms that mark code as fluent Python, and the performance principles that matter. The meta-rule on performance is **measure, don't guess** — profile to find the real hot spot before optimizing, because intuition about what's slow is usually wrong. **[A]**

### 17.1 Classic gotchas

Each of these stems directly from Python's object model and evaluation rules covered earlier — they're not arbitrary, and understanding the *why* prevents the bug.

```python
# 1) MUTABLE DEFAULT ARGUMENTS are evaluated ONCE, at definition time, and SHARED across calls.
def bad(item, bucket=[]):       # BUG: every call reuses the SAME list object
    bucket.append(item)
    return bucket
print(bad(1), bad(2))           # [1] [1, 2]  <- the surprise: state leaks between calls
def good(item, bucket=None):    # FIX: use None as the sentinel, create a fresh list inside
    bucket = [] if bucket is None else bucket
    bucket.append(item)
    return bucket

# 2) LATE-BINDING CLOSURES in loops: a closure captures the VARIABLE, not its value at the time.
fns = [lambda: i for i in range(3)]
print([f() for f in fns])       # [2, 2, 2]  <- all see the FINAL value of i (the loop is done)
fns = [lambda i=i: i for i in range(3)]     # FIX: bind the current value via a default arg
print([f() for f in fns])       # [0, 1, 2]

# 3) `is` vs `==`: `is` compares IDENTITY (same object), `==` compares VALUE. Use `is` only
#    for None/True/False (singletons). Small-int and string caching is an implementation detail
#    you must NEVER rely on:
print(1000 is 1000)             # may be True or False depending on context — don't depend on it
print(1000 == 1000)             # True — always correct for value comparison

# 4) COPYING: assignment shares the object; mutating one name affects all names for it.
import copy
original = [1, 2, 3]
shallow = original[:]                  # or list(original) — independent TOP level only
nested = [[1], [2]]
deep = copy.deepcopy(nested)          # fully independent, all the way down

# 5) FLOATING POINT: 0.1 + 0.2 != 0.3 exactly. Compare with tolerance, or use Decimal for money.
import math
print(0.1 + 0.2 == 0.3)               # False — never == two floats
print(math.isclose(0.1 + 0.2, 0.3))   # True  — compare with a tolerance instead

# 6) MODIFYING A LIST WHILE ITERATING it skips elements / corrupts the loop. Iterate a copy
#    or build a new list:
nums = [1, 2, 3, 4]
nums = [n for n in nums if n % 2 == 0]   # FIX: filter into a new list, don't remove in place
```

### 17.2 Idioms & performance

Idiomatic Python is usually *also* the fast Python, because the readable construct (a comprehension, a built-in, `join`) is implemented in optimized C.

```python
items = [3, 1, 2]
nums = [1, 2, 3]

# Iterate the THING, not indices — clearer and faster:
for item in items: ...                 # not: for i in range(len(items)): items[i]

# Truthiness over explicit length checks:
if items: ...                          # not: if len(items) > 0

# Build strings with join, never += in a loop (each += creates a whole new string -> O(n^2)):
result = "".join(str(x) for x in range(1000))   # fast, single allocation
# slow/quadratic: s = ""; for x in ...: s += str(x)

# Reach for built-ins and generator expressions — they're C-fast and lazy:
print(any(x > 10 for x in nums))       # short-circuits at the first True
print(all(x > 0 for x in nums))        # short-circuits at the first False
print(sum(x * x for x in nums))        # no intermediate list

# Membership: use a set, not a list, for repeated `in` tests (O(1) vs O(n)):
allowed = {"a", "b", "c"}              # if x in allowed:  -> hash lookup

# enumerate / zip / dict.get / dict.setdefault / unpacking keep code clean AND fast.

# PROFILING — measure before optimizing:
#   python -m cProfile -s cumtime myscript.py     # find the real hot functions
#   import timeit; timeit.timeit("expr", number=10000)   # micro-benchmark a snippet
```

**PEP 8 essentials (Python's style guide):** 4-space indentation; `snake_case` for functions/variables/modules, `PascalCase` for classes, `UPPER_CASE` for constants; lines ~79–100 chars; two blank lines between top-level definitions, one between methods; imports grouped (stdlib, third-party, local). Don't format by hand — let **ruff** or **black** do it automatically and consistently, and let **ruff** lint for the rest.

| Want | Prefer | Avoid |
|---|---|---|
| Loop with values | `for x in seq` | `for i in range(len(seq))` |
| Index + value | `enumerate(seq)` | manual counter |
| Parallel sequences | `zip(a, b)` | index gymnastics |
| Transform/filter | comprehension / generator | manual append loop |
| Concatenate strings | `"".join(parts)` | `+=` in a loop |
| Repeated membership | `set` | `list` |
| Missing-key default | `dict.get` / `defaultdict` | try/except KeyError everywhere |
| Exact decimals (money) | `Decimal` | `float` |

---

## 18. Study Path & Build-to-Learn Projects

Knowledge sticks when you *build*. Read for understanding, then immediately apply each cluster of concepts in a small project. The suggested order front-loads the universally useful material (basics, then file/OS work) and saves the genuinely advanced topics (decorators, concurrency, packaging) for once the fundamentals are automatic.

**Suggested order:** §1–4 (basics) → §5–6 (functions & OOP) → §7–9 (modules, errors, generators) → §12 (file/OS — immediately useful and motivating) → §11 (type hints) → §14–15 (stdlib & testing) → §10, §13, §16–17 (advanced).

**Build these to cement it (each targets specific sections):**
1. **CLI todo manager** — `argparse` for the interface, JSON or `sqlite3` for persistence, `pathlib` for the data file. Exercises §12, §14. *Stretch:* add type hints and pytest tests.
2. **Log analyzer** — read large files lazily with generators, tally with `Counter`, extract fields with `re`, render a `rich` table. Exercises §9, §14, §17.
3. **File organizer / backup tool** — walk a tree with `os.walk`/`rglob`, move files by extension, zip a folder with `shutil`, shell out to `git` with `subprocess`. Exercises §12.
4. **Web scraper or API client** — `httpx`/`requests`, concurrency with `asyncio` (or a `ThreadPoolExecutor`), a retry decorator, typed response models with dataclasses. Exercises §10, §11, §13.
5. **A small installable package** — `src/` layout, pytest tests, type-checked with mypy/pyright, built with `python -m build`, published to TestPyPI. Exercises §15, §16.

**Next steps after this guide:** a web framework (FastAPI or Django), an ORM (SQLAlchemy), data tooling (pandas/polars), and real-world async I/O. For comparison, the file/OS material maps closely to the Go File System / OS / CLI guide elsewhere in this library — reading both deepens your understanding of *why* each language made its design choices.

---

*Part of the offline developer study library. Written for Python 3.13/3.14 as of 2026. Confirm fast-moving APIs against docs.python.org.*
