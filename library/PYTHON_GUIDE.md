# Python — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I've never written Python" to "I write maintainable, typed, production Python" — without an internet connection. Every concept comes with runnable, commented code. Read top-to-bottom the first time; after that, use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
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

### 1.1 Getting Python **[B]**

- **Windows:** install from python.org or `winget install Python.Python.3.13`. Windows ships the **`py` launcher** — use `py -3.13 script.py` to pick a version. On macOS/Linux the command is usually `python3`.
- Check it works:

```bash
python --version      # or: py --version   (Windows)   /   python3 --version (mac/linux)
python -c "print('hello')"   # run a one-liner
```

### 1.2 The REPL

```bash
python                 # opens the interactive shell (3.13+ has a much nicer REPL)
>>> 2 + 2
4
>>> import math; math.sqrt(16)
4.0
>>> exit()             # or Ctrl-Z then Enter (Windows) / Ctrl-D (unix)
```

### 1.3 Running scripts

```bash
python hello.py            # run a file
python -m http.server 8000 # run a module as a script (-m)
```

`-m` runs a module from the import path — prefer `python -m pip ...` over bare `pip` so you always hit the pip belonging to *this* interpreter.

### 1.4 Virtual environments — **do this for every project** **[B, important]**

A venv is an isolated set of packages so projects don't fight over dependency versions.

```bash
python -m venv .venv                 # create a venv in ./.venv
# Activate it:
.venv\Scripts\activate               # Windows (PowerShell/cmd)
source .venv/bin/activate            # macOS / Linux / Git Bash
# Now `python` and `pip` point INTO the venv:
pip install requests
pip freeze > requirements.txt        # snapshot exact versions
pip install -r requirements.txt      # reproduce elsewhere
deactivate                           # leave the venv
```

> **⚡ 2026 tooling:** **`uv`** (from Astral) is the fast, modern, all-in-one replacement for `pip`/`venv`/`pip-tools`/`pyenv`. It's the default many teams reach for now:
> ```bash
> uv venv                  # create venv
> uv pip install requests  # install (10–100x faster than pip)
> uv add requests          # add to pyproject.toml + lockfile
> uv run script.py         # run inside the project env automatically
> ```
> Other tools: **poetry** (dependency + packaging manager), **pipx** (install CLI apps in isolated envs: `pipx install ruff`).

### 1.5 `pyproject.toml`

The modern, standardized project metadata file (replaces `setup.py`/`requirements.txt` for most needs):

```toml
[project]
name = "myapp"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = ["requests>=2.32", "rich"]

[project.scripts]
myapp = "myapp.cli:main"     # creates a `myapp` command -> calls myapp/cli.py:main()

[tool.ruff]                  # ruff = the fast linter+formatter (replaces flake8/black/isort)
line-length = 100
```

---

## 2. Syntax, Values & Types

### 2.1 The basics **[B]**

Python uses **indentation** (4 spaces, no braces) to define blocks. No semicolons needed.

```python
# This is a comment.
x = 42                 # int — no type declaration; the name `x` is bound to the object 42
name = "Ada"           # str
pi = 3.14159           # float
is_ready = True        # bool (True/False are capitalized)
nothing = None         # the null/absence value (its own type, NoneType)

# Variables are just names pointing at objects; assignment never copies.
a = [1, 2, 3]
b = a                  # b points at the SAME list
b.append(4)
print(a)               # [1, 2, 3, 4]  <- a changed too! (see §17 for the trap)
```

### 2.2 Numbers **[B]**

```python
n = 10
f = 3.0
big = 1_000_000        # underscores allowed as digit separators
print(7 / 2)           # 3.5   (true division — ALWAYS a float)
print(7 // 2)          # 3     (floor division)
print(7 % 3)           # 1     (modulo)
print(2 ** 10)         # 1024  (exponent)
print(int("42"), float("3.5"))   # convert from string
print(round(3.14159, 2))         # 3.14
print(abs(-5), min(3, 9), max(3, 9), sum([1, 2, 3]))

# Integers are arbitrary precision — no overflow:
print(2 ** 200)        # a 61-digit number, exact

# Floats are IEEE-754 doubles — beware (see §17):
print(0.1 + 0.2)       # 0.30000000000000004
from decimal import Decimal
print(Decimal("0.1") + Decimal("0.2"))   # 0.3  (use Decimal for money)
```

### 2.3 Strings **[B]**

```python
s = "double"
s2 = 'single'                 # quotes are interchangeable
multi = """spans
multiple lines"""

# f-strings (formatted string literals) — the standard way to interpolate:
user, score = "Ada", 95.5
print(f"{user} scored {score:.1f}%")     # Ada scored 95.5%
print(f"{user=}")                          # user='Ada'  (debug form, 3.8+)
print(f"{score:>10}")                      # right-align in width 10
print(f"{255:#x} {255:08b}")               # 0xff 11111111  (hex / binary formatting)

# Common methods (strings are IMMUTABLE — methods return new strings):
t = "  Hello, World  "
print(t.strip())              # "Hello, World"
print(t.strip().lower())      # "hello, world"
print("a,b,c".split(","))     # ['a', 'b', 'c']
print("-".join(["a", "b"]))   # "a-b"
print("hello".replace("l", "L"))      # "heLLo"
print("hello".startswith("he"), "hello".endswith("lo"))
print("Ada".upper(), "ADA".lower(), "ada".capitalize())
print("hello world".title())          # "Hello World"
print(len("hello"))                    # 5
print("ell" in "hello")                # True (substring test)
print("hello".find("l"), "hello".index("l"))   # 2, 2 (find returns -1 if absent)

# Slicing (works on strings, lists, tuples): s[start:stop:step]  (stop is exclusive)
w = "python"
print(w[0], w[-1])      # p n      (negative = from the end)
print(w[0:3])           # pyt
print(w[::2])           # pto      (every 2nd char)
print(w[::-1])          # nohtyp   (reverse)
```

### 2.4 Booleans, None & truthiness **[B]**

```python
# Falsy values: False, None, 0, 0.0, "", [], {}, (), set()
# Everything else is truthy.
if not "":            print("empty string is falsy")
if [1]:               print("non-empty list is truthy")

x = None
print(x is None)      # True — use `is` for None, not `==`

# Ternary expression:
status = "pass" if score >= 60 else "fail"
```

### 2.5 Operators **[B]**

```python
# Comparison: ==  !=  <  <=  >  >=     (chainable!)
print(1 < 2 < 3)            # True — means (1<2) and (2<3)
# Logical: and  or  not     (short-circuit; return operands, not just bools)
print(0 or "default")       # "default"   (handy for fallbacks)
print("a" and "b")          # "b"
# Identity vs equality:
print([1] == [1])           # True  (same value)
print([1] is [1])           # False (different objects)
# Membership:
print(3 in [1, 2, 3])       # True
# Walrus := (assignment expression, 3.8+):
if (n := len([1, 2, 3])) > 2:
    print(f"length is {n}")
```

---

## 3. Built-in Data Structures

### 3.1 list — ordered, mutable **[B]**

```python
nums = [3, 1, 2]
nums.append(4)              # [3, 1, 2, 4]
nums.insert(0, 9)           # [9, 3, 1, 2, 4]
nums.extend([5, 6])         # add multiple
last = nums.pop()           # remove & return last (or pop(i))
nums.remove(9)              # remove first matching VALUE
nums.sort()                 # in place; nums.sort(reverse=True) / key=...
ordered = sorted(nums)      # returns a NEW sorted list
print(len(nums), nums[0], nums[-1])
print(2 in nums)            # membership
nums[1:3] = [10, 20]        # slice assignment
print(nums.count(2), nums.index(10))

# List comprehension — the Pythonic transform/filter:
squares = [x * x for x in range(5)]            # [0, 1, 4, 9, 16]
evens   = [x for x in range(10) if x % 2 == 0] # [0, 2, 4, 6, 8]
pairs   = [(x, y) for x in "ab" for y in [1, 2]]  # nested
```

### 3.2 tuple — ordered, immutable **[B]**

```python
point = (3, 4)
x, y = point                 # unpacking
print(point[0])              # 3
single = (1,)                # one-element tuple NEEDS the trailing comma
# Tuples are great as fixed records and dict keys (they're hashable).
a, b = b, a                  # swap — builds a tuple under the hood
first, *rest = [1, 2, 3, 4]  # first=1, rest=[2,3,4] (extended unpacking)
```

### 3.3 dict — key→value mapping **[B]**

```python
user = {"name": "Ada", "age": 36}
print(user["name"])             # Ada  (KeyError if missing)
print(user.get("email"))        # None (no error)
print(user.get("email", "n/a")) # default if missing
user["email"] = "a@x.com"       # add/update
del user["age"]                 # remove
print("name" in user)           # key membership
for key, value in user.items(): # iterate pairs
    print(key, value)
print(list(user.keys()), list(user.values()))
user.update({"age": 37, "city": "London"})
merged = {**user, "role": "eng"}   # merge via unpacking
merged = user | {"role": "eng"}    # merge operator (3.9+)
# Dict comprehension:
sq = {n: n * n for n in range(4)}  # {0:0, 1:1, 2:4, 3:9}
# Dicts preserve insertion order (guaranteed since 3.7).
```

### 3.4 set & frozenset — unique, unordered **[B]**

```python
s = {1, 2, 2, 3}            # {1, 2, 3}  (dups removed)
s.add(4); s.discard(1)     # discard = no error if absent (remove raises)
print(2 in s)              # fast membership (O(1))
a, b = {1, 2, 3}, {2, 3, 4}
print(a | b)               # union {1,2,3,4}
print(a & b)               # intersection {2,3}
print(a - b)               # difference {1}
print(a ^ b)               # symmetric difference {1,4}
empty = set()              # {} is an empty DICT, not a set!
frozen = frozenset([1, 2]) # immutable & hashable (usable as dict key / set member)
```

### 3.5 Choosing a structure

| Need | Use |
|---|---|
| Ordered, changeable sequence | `list` |
| Fixed record / hashable sequence | `tuple` |
| Lookup by key | `dict` |
| Unique items / fast membership / set math | `set` |
| Immutable set (dict key, etc.) | `frozenset` |
| Counting | `collections.Counter` (§14) |
| Default values per key | `collections.defaultdict` (§14) |
| Fast appends/pops both ends (queue) | `collections.deque` (§14) |

---

## 4. Control Flow

### 4.1 if / elif / else **[B]**

```python
def grade(score):
    if score >= 90:
        return "A"
    elif score >= 80:
        return "B"
    elif score >= 60:
        return "C"
    else:
        return "F"
```

### 4.2 for & while **[B]**

```python
for i in range(5):              # 0,1,2,3,4   range(start, stop, step)
    print(i)

for ch in "abc":                # iterate any iterable
    print(ch)

for i, name in enumerate(["a", "b"], start=1):   # index + value
    print(i, name)              # 1 a / 2 b

for name, age in zip(["Ada", "Bo"], [36, 29]):   # parallel iteration
    print(name, age)

n = 5
while n > 0:
    print(n); n -= 1

# break / continue, and the rarely-known loop `else` (runs if no break):
for x in [1, 3, 5]:
    if x % 2 == 0:
        break
else:
    print("no even number found")   # this runs
```

### 4.3 match / case — structural pattern matching **[I]** (3.10+)

```python
def describe(command):
    match command.split():
        case ["quit"]:
            return "exiting"
        case ["move", direction]:
            return f"moving {direction}"
        case ["move", *rest]:                  # capture the rest
            return f"moving multiple: {rest}"
        case _:                                # wildcard (default)
            return "unknown"

# Match on shape/type/values:
def area(shape):
    match shape:
        case {"type": "circle", "r": r}:       # dict pattern
            return 3.14159 * r * r
        case {"type": "rect", "w": w, "h": h}:
            return w * h
        case Point(x=0, y=0):                  # class pattern (matches attributes)
            return 0
        case _:
            raise ValueError("unknown shape")
```

---

## 5. Functions

### 5.1 Defining & calling **[B]**

```python
def greet(name, greeting="Hello"):     # `greeting` has a default
    """Return a greeting (this is the docstring)."""
    return f"{greeting}, {name}!"

print(greet("Ada"))                    # Hello, Ada!
print(greet("Bo", greeting="Hi"))      # keyword argument
print(greet(greeting="Hey", name="Cy"))# order doesn't matter for keywords
```

### 5.2 `*args` and `**kwargs` **[I]**

```python
def total(*args):                      # args is a tuple of positional args
    return sum(args)
print(total(1, 2, 3))                  # 6

def configure(**kwargs):               # kwargs is a dict of keyword args
    for k, v in kwargs.items():
        print(f"{k} = {v}")
configure(debug=True, level=3)

def f(a, b, *args, **kwargs): ...      # the full signature order

# Unpack when CALLING:
nums = [1, 2, 3]
print(total(*nums))                    # spread list as positional args
opts = {"debug": True}
configure(**opts)                      # spread dict as keyword args
```

### 5.3 Positional-only & keyword-only params **[A]**

```python
def f(pos_only, /, normal, *, kw_only):
    #          ^ everything before / is positional-only
    #                            ^ everything after * is keyword-only
    return (pos_only, normal, kw_only)

f(1, 2, kw_only=3)        # ok
f(1, normal=2, kw_only=3) # ok
# f(pos_only=1, ...)      # ERROR: pos_only is positional-only
```

### 5.4 Lambdas, scope & closures **[I]**

```python
# Lambda: a tiny anonymous function (one expression).
add = lambda a, b: a + b
people = [("Ada", 36), ("Bo", 29)]
people.sort(key=lambda p: p[1])        # sort by age

# Scope: LEGB — Local, Enclosing, Global, Built-in (the lookup order).
count = 0
def increment():
    global count        # without this, assignment would create a LOCAL `count`
    count += 1

# Closure: an inner function that captures variables from its enclosing scope.
def make_multiplier(factor):
    def multiply(n):
        return n * factor          # `factor` is captured
    return multiply
double = make_multiplier(2)
print(double(5))                   # 10

def counter():
    n = 0
    def inc():
        nonlocal n                 # rebind the ENCLOSING n (not global)
        n += 1
        return n
    return inc
c = counter(); print(c(), c(), c())  # 1 2 3
```

---

## 6. Object-Oriented Python

### 6.1 Classes & instances **[I]**

```python
class Account:
    interest_rate = 0.02              # CLASS attribute (shared by all instances)

    def __init__(self, owner, balance=0):
        self.owner = owner            # INSTANCE attributes (per object)
        self.balance = balance

    def deposit(self, amount):        # `self` is the instance (explicit, always first)
        self.balance += amount
        return self.balance

    def __repr__(self):               # unambiguous dev representation
        return f"Account({self.owner!r}, {self.balance})"

    def __str__(self):                # friendly representation (used by print/str)
        return f"{self.owner}: ${self.balance}"

acc = Account("Ada", 100)
acc.deposit(50)
print(acc)            # Ada: $150          (uses __str__)
print(repr(acc))      # Account('Ada', 150) (uses __repr__)
```

### 6.2 classmethod, staticmethod, property **[I]**

```python
class Temperature:
    def __init__(self, celsius):
        self._celsius = celsius

    @property                         # access like an attribute: t.fahrenheit
    def fahrenheit(self):
        return self._celsius * 9 / 5 + 32

    @fahrenheit.setter                # allow t.fahrenheit = 100
    def fahrenheit(self, value):
        self._celsius = (value - 32) * 5 / 9

    @classmethod                      # alternative constructor; gets the CLASS
    def from_fahrenheit(cls, f):
        return cls((f - 32) * 5 / 9)

    @staticmethod                     # no self/cls — just namespaced
    def is_freezing(celsius):
        return celsius <= 0

t = Temperature(25)
print(t.fahrenheit)                   # 77.0
t.fahrenheit = 212
print(t._celsius)                     # 100.0
print(Temperature.is_freezing(-5))    # True
```

### 6.3 Inheritance, MRO & super() **[I]**

```python
class Animal:
    def __init__(self, name):
        self.name = name
    def speak(self):
        raise NotImplementedError

class Dog(Animal):
    def __init__(self, name, breed):
        super().__init__(name)        # call the parent __init__
        self.breed = breed
    def speak(self):                  # override
        return "Woof"

d = Dog("Rex", "Lab")
print(d.name, d.speak())              # Rex Woof
print(isinstance(d, Animal))          # True
print(Dog.__mro__)                    # method resolution order (search path)
```

### 6.4 Dunder (magic) methods **[I]**

```python
class Vector:
    def __init__(self, x, y): self.x, self.y = x, y
    def __add__(self, other):  return Vector(self.x + other.x, self.y + other.y)
    def __eq__(self, other):   return (self.x, self.y) == (other.x, other.y)
    def __len__(self):         return 2
    def __getitem__(self, i):  return (self.x, self.y)[i]
    def __repr__(self):        return f"Vector({self.x}, {self.y})"

print(Vector(1, 2) + Vector(3, 4))    # Vector(4, 6)
print(Vector(1, 2) == Vector(1, 2))   # True
```

| Dunder | Enables |
|---|---|
| `__init__` | construction | 
| `__repr__` / `__str__` | `repr()` / `str()`/`print` |
| `__eq__`, `__lt__`, … | `==`, `<`, sorting (`@functools.total_ordering` fills the rest) |
| `__hash__` | use as dict key / set member |
| `__len__`, `__getitem__`, `__setitem__` | `len()`, indexing |
| `__iter__`, `__next__` | iteration (§9) |
| `__enter__`, `__exit__` | `with` (§9) |
| `__call__` | make instances callable like functions |

### 6.5 Dataclasses — boilerplate-free classes **[I]**

```python
from dataclasses import dataclass, field

@dataclass
class Point:
    x: float
    y: float = 0.0                    # default
    tags: list = field(default_factory=list)   # mutable default — MUST use factory

    def distance_to(self, other):
        return ((self.x - other.x) ** 2 + (self.y - other.y) ** 2) ** 0.5

p = Point(3, 4)
print(p)                              # Point(x=3, y=4, tags=[])  (auto __repr__)
print(p == Point(3, 4))               # True  (auto __eq__)

@dataclass(frozen=True)               # immutable + hashable
class Config:
    host: str
    port: int = 8080
```

### 6.6 Abstract base classes & enums **[I]**

```python
from abc import ABC, abstractmethod
from enum import Enum, auto

class Repository(ABC):
    @abstractmethod
    def get(self, id: int): ...       # subclasses MUST implement

class InMemoryRepo(Repository):
    def __init__(self): self._data = {}
    def get(self, id): return self._data.get(id)

class Color(Enum):
    RED = auto()
    GREEN = auto()
    BLUE = auto()

print(Color.RED, Color.RED.name, Color.RED.value)   # Color.RED RED 1
print(Color(1))                                       # Color.RED
for c in Color: print(c)
```

---

## 7. Modules & Packages

### 7.1 Importing **[B]**

```python
import math
from math import sqrt, pi
from math import sqrt as square_root      # alias
import json as j
print(math.sqrt(9), sqrt(9), pi)
```

### 7.2 Your own modules **[I]**

```python
# file: mathutils.py
def add(a, b):
    return a + b

PI = 3.14159

if __name__ == "__main__":      # runs ONLY when executed directly, not on import
    print("testing:", add(2, 3))
```

```python
# file: main.py
import mathutils
from mathutils import add
print(add(2, 3), mathutils.PI)
```

`if __name__ == "__main__":` is the idiom for "code that should run when this file is the entry point, but not when it's imported."

### 7.3 Packages **[I]**

A **package** is a directory of modules. With a `__init__.py` it's a "regular package"; without one it can still work as a "namespace package" (3.3+), but include `__init__.py` for normal projects.

```
myapp/
├── __init__.py          # marks the package; can re-export public API
├── core.py
└── db/
    ├── __init__.py
    └── models.py
```

```python
from myapp.db.models import User      # absolute import (preferred)
from .models import User              # relative import (inside the package)
```

---

## 8. Exceptions & Error Handling

```python
def parse_age(s):
    try:
        age = int(s)
    except ValueError as e:               # catch a specific type
        print(f"not a number: {e}")
        return None
    except (TypeError, KeyError):         # catch multiple
        return None
    else:
        print("parsed fine")              # runs only if NO exception
        return age
    finally:
        print("always runs (cleanup)")    # runs no matter what

# Raising:
def withdraw(balance, amount):
    if amount > balance:
        raise ValueError(f"insufficient funds: {amount} > {balance}")
    return balance - amount

# Custom exceptions — subclass Exception:
class InsufficientFunds(Exception):
    def __init__(self, needed, available):
        super().__init__(f"need {needed}, have {available}")
        self.needed = needed
        self.available = available

# Exception chaining — keep the original cause:
try:
    int("x")
except ValueError as e:
    raise RuntimeError("config invalid") from e   # preserves the __cause__
```

**Best practices:** catch the narrowest exception you can; never `except:` bare (it swallows `KeyboardInterrupt`/`SystemExit`); prefer EAFP ("easier to ask forgiveness than permission") — try the operation and handle the failure, rather than pre-checking everything.

---

## 9. Iterators, Generators & Context Managers

### 9.1 Iterators **[I]**

```python
nums = [1, 2, 3]
it = iter(nums)              # get an iterator
print(next(it))              # 1
print(next(it))              # 2
# A `for` loop calls iter() then next() until StopIteration.

class Countdown:
    def __init__(self, n): self.n = n
    def __iter__(self): return self
    def __next__(self):
        if self.n <= 0:
            raise StopIteration
        self.n -= 1
        return self.n + 1
print(list(Countdown(3)))    # [3, 2, 1]
```

### 9.2 Generators — lazy iterators with `yield` **[I]**

```python
def countdown(n):
    while n > 0:
        yield n              # produces a value and PAUSES here
        n -= 1
for x in countdown(3):
    print(x)                 # 3 2 1

# Generators are memory-efficient — they compute on demand:
def read_big_file(path):
    with open(path) as f:
        for line in f:       # files are iterable line-by-line
            yield line.rstrip("\n")

# Generator EXPRESSION (like a comprehension but lazy, with ()):
squares = (x * x for x in range(1_000_000))   # nothing computed yet
print(next(squares))         # 0  (computed on demand)
print(sum(x * x for x in range(5)))           # 30 (no intermediate list)
```

### 9.3 Context managers — `with` **[I]**

```python
# `with` guarantees cleanup (the resource is closed even if an error occurs):
with open("data.txt", "w", encoding="utf-8") as f:
    f.write("hello")
# f is closed automatically here.

# Multiple at once:
with open("in.txt") as src, open("out.txt", "w") as dst:
    dst.write(src.read())

# Write your own with __enter__/__exit__:
class Timer:
    def __enter__(self):
        import time
        self.start = time.perf_counter()
        return self
    def __exit__(self, exc_type, exc, tb):
        import time
        self.elapsed = time.perf_counter() - self.start
        print(f"took {self.elapsed:.3f}s")
        return False           # False = don't suppress exceptions

with Timer():
    sum(range(1_000_000))

# Or the easy way with contextlib:
from contextlib import contextmanager
@contextmanager
def timer():
    import time
    start = time.perf_counter()
    try:
        yield                  # the `with` body runs here
    finally:
        print(f"took {time.perf_counter() - start:.3f}s")
```

---

## 10. Decorators

A decorator wraps a function to add behaviour. **[I/A]**

```python
import functools

def log_calls(func):
    @functools.wraps(func)          # preserves func's name/docstring/signature
    def wrapper(*args, **kwargs):
        print(f"calling {func.__name__}({args}, {kwargs})")
        result = func(*args, **kwargs)
        print(f"-> {result}")
        return result
    return wrapper

@log_calls                          # same as: add = log_calls(add)
def add(a, b):
    return a + b
add(2, 3)

# Parameterized decorator (a decorator factory):
def retry(times):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == times - 1:
                        raise
                    print(f"retry {attempt + 1} after {e}")
        return wrapper
    return decorator

@retry(times=3)
def flaky(): ...

# Useful built-in decorators:
from functools import lru_cache, cache
@cache                              # memoize (unbounded); lru_cache(maxsize=N) for bounded
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)
```

---

## 11. Type Hints

Type hints are optional annotations checked by tools like **mypy** or **pyright** (and your editor) — they don't affect runtime. **[I/A]**

```python
def greet(name: str, times: int = 1) -> str:
    return f"hi {name} " * times

# Variables, containers (modern 3.9+/3.10+ syntax — no need to import List/Dict):
count: int = 0
names: list[str] = []
scores: dict[str, int] = {}
maybe: int | None = None            # union; `X | None` replaces Optional[X]

# Callable, tuples, etc.:
from collections.abc import Callable, Iterable
handler: Callable[[int, str], bool]
coords: tuple[float, float]

# Protocols — structural typing ("duck typing", checked):
from typing import Protocol
class SupportsClose(Protocol):
    def close(self) -> None: ...
def shutdown(x: SupportsClose) -> None:
    x.close()                        # any object with a close() method satisfies this

# TypedDict, Literal, generics:
from typing import TypedDict, Literal
class UserDict(TypedDict):
    name: str
    age: int
Mode = Literal["r", "w", "a"]
def open_file(path: str, mode: Mode) -> None: ...

# Generic functions/classes (3.12+ syntax — no TypeVar import needed):
def first[T](items: list[T]) -> T:
    return items[0]

class Box[T]:
    def __init__(self, value: T) -> None:
        self.value = value
```

Run a checker: `mypy myapp/` or `pyright`. Many teams in 2026 run **ruff** for linting/formatting and pyright/mypy for types.

---

## 12. File System, OS & Command Execution

This is the systems-programming core: read/write files, manipulate paths, inspect the OS, and shell out to other programs. **[I]**

### 12.1 Reading & writing files

```python
# Always specify encoding="utf-8" (don't rely on the platform default).
# Small files — read/write whole content:
text = open("notes.txt", encoding="utf-8").read()          # works, but doesn't close
with open("notes.txt", encoding="utf-8") as f:             # preferred — auto-closes
    text = f.read()

with open("out.txt", "w", encoding="utf-8") as f:          # "w" truncates, "a" appends
    f.write("line 1\n")
    f.writelines(["line 2\n", "line 3\n"])

# Line by line (memory-friendly for big files):
with open("big.log", encoding="utf-8") as f:
    for line in f:                 # f is a lazy iterator of lines
        process(line.rstrip("\n"))

# Modes: "r" read, "w" write/truncate, "a" append, "x" create-exclusive,
#        "b" binary (e.g. "rb"/"wb"), "+" read+write.
with open("img.png", "rb") as f:   # binary
    data = f.read()
```

### 12.2 `pathlib.Path` — the modern path API (prefer this over `os.path`)

```python
from pathlib import Path

p = Path("data") / "reports" / "2026.csv"   # `/` joins, cross-platform
print(p)                  # data\reports\2026.csv on Windows, forward slashes elsewhere
print(p.name)             # 2026.csv
print(p.stem)             # 2026
print(p.suffix)           # .csv
print(p.parent)           # data\reports
print(p.parts)            # ('data', 'reports', '2026.csv')
print(p.absolute())       # absolute path
print(p.resolve())        # absolute + symlinks resolved

# Existence / type:
print(p.exists(), p.is_file(), p.is_dir())

# Read/write shortcuts:
Path("hello.txt").write_text("hi", encoding="utf-8")
print(Path("hello.txt").read_text(encoding="utf-8"))
Path("data.bin").write_bytes(b"\x00\x01")

# Create / remove:
Path("logs/sub").mkdir(parents=True, exist_ok=True)   # like mkdir -p
Path("tmp.txt").unlink(missing_ok=True)               # delete a file

# List a directory & glob:
for child in Path(".").iterdir():
    print(child)
for csv in Path("data").glob("*.csv"):       # non-recursive
    print(csv)
for py in Path(".").rglob("*.py"):           # recursive (** under the hood)
    print(py)

# Home / cwd:
print(Path.home())        # user home dir
print(Path.cwd())         # current working dir
```

### 12.3 `os` and `shutil` for filesystem ops

```python
import os, shutil

os.getcwd()                       # current dir
os.chdir("/tmp")                  # change dir
os.listdir(".")                   # names in a dir
os.makedirs("a/b/c", exist_ok=True)
os.remove("file.txt")             # delete file
os.rename("old.txt", "new.txt")
os.replace("a.txt", "b.txt")      # atomic move/overwrite

# Walk a tree (dirpath, dirnames, filenames at each level):
for dirpath, dirnames, filenames in os.walk("project"):
    for name in filenames:
        full = os.path.join(dirpath, name)
        print(full)

# shutil — higher-level file ops:
shutil.copy("a.txt", "b.txt")        # copy file
shutil.copytree("src", "dst")        # copy a whole tree
shutil.move("a.txt", "dir/")         # move
shutil.rmtree("build")               # recursively delete a dir (careful!)
print(shutil.which("git"))           # find an executable on PATH (like `which`/`where`)
shutil.disk_usage("/")               # (total, used, free) bytes

# Archives:
shutil.make_archive("backup", "zip", "myfolder")  # -> backup.zip
shutil.unpack_archive("backup.zip", "restored")

# Temp files/dirs:
import tempfile
with tempfile.NamedTemporaryFile(mode="w", delete=False, suffix=".txt") as tf:
    tf.write("scratch")
    tmp_path = tf.name
with tempfile.TemporaryDirectory() as d:   # auto-deleted on exit
    print("temp dir:", d)
```

### 12.4 OS & system information

```python
import os, sys, platform

print(sys.argv)              # command-line args; sys.argv[0] is the script name
print(sys.version)           # Python version string
print(sys.platform)          # 'win32', 'linux', 'darwin'
print(sys.executable)        # path to the python interpreter

print(platform.system())     # 'Windows' / 'Linux' / 'Darwin'
print(platform.release(), platform.machine())   # OS release, CPU arch
print(platform.python_version())

print(os.name)               # 'nt' on Windows, 'posix' elsewhere
print(os.cpu_count())        # logical CPUs
print(os.getpid())           # current process id
print(os.getlogin())         # login name (may raise in some envs; see getpass)
import getpass
print(getpass.getuser())     # more reliable username
```

### 12.5 Environment variables

```python
import os

print(os.environ.get("PATH"))            # read (None if absent)
print(os.environ.get("API_KEY", ""))     # with default
os.environ["MY_FLAG"] = "1"              # set (for this process & children)
for key, value in os.environ.items():    # iterate all
    ...

# Loading a .env file (with the python-dotenv package):
# pip install python-dotenv
from dotenv import load_dotenv
load_dotenv()                            # reads .env into os.environ
db_url = os.environ["DATABASE_URL"]
```

### 12.6 Executing external commands — `subprocess`

```python
import subprocess

# The modern, recommended entry point is subprocess.run():
result = subprocess.run(
    ["git", "rev-parse", "--abbrev-ref", "HEAD"],   # args as a LIST (no shell)
    capture_output=True,     # capture stdout & stderr
    text=True,               # decode bytes -> str
    check=True,              # raise CalledProcessError on non-zero exit
    cwd=".",                 # working directory
    timeout=30,              # seconds
    env={**os.environ, "GIT_PAGER": "cat"},   # custom environment
)
print("branch:", result.stdout.strip())
print("exit code:", result.returncode)

# Running real tools:
subprocess.run(["pip", "install", "rich"], check=True)
subprocess.run([sys.executable, "-m", "pip", "install", "rich"])  # safest: this interpreter's pip
subprocess.run(["npm", "install"], cwd="frontend")
subprocess.run(["python", "build.py", "--release"])

# Capture & parse JSON output:
import json
out = subprocess.run(["npm", "ls", "--json"], capture_output=True, text=True).stdout
deps = json.loads(out)

# shell=True runs through the shell — convenient but a COMMAND-INJECTION risk.
# NEVER build a shell string from untrusted input.
subprocess.run("echo hello && echo world", shell=True)   # ok for trusted, fixed strings
# DANGER (do not do this with user input):
# subprocess.run(f"rm {user_supplied}", shell=True)

# Streaming output live (Popen for fine control):
proc = subprocess.Popen(["ping", "-n", "3", "127.0.0.1"],  # Windows: -n; unix: -c
                        stdout=subprocess.PIPE, text=True)
for line in proc.stdout:
    print("ping:", line.rstrip())
proc.wait()

# Piping one command into another:
p1 = subprocess.Popen(["echo", "a\nb\nc"], stdout=subprocess.PIPE, text=True)
p2 = subprocess.run(["sort", "-r"], stdin=p1.stdout, capture_output=True, text=True)
```

> **⚡ Windows note:** some commands are shell built-ins (`dir`, `copy`) and aren't real executables — run them via `subprocess.run(["cmd", "/c", "dir"])`. Tools installed by npm are often `.cmd` shims (`npm.cmd`); `shutil.which("npm")` finds the right one. On Unix use `["ls", "-la"]` directly.

### 12.7 Building a CLI with `argparse`

```python
import argparse

def main():
    parser = argparse.ArgumentParser(description="Resize images in a folder.")
    parser.add_argument("folder", help="folder to process")            # positional
    parser.add_argument("-w", "--width", type=int, default=800,        # option with value
                        help="target width in px")
    parser.add_argument("-v", "--verbose", action="store_true",        # boolean flag
                        help="print each file")
    parser.add_argument("--format", choices=["jpg", "png"], default="jpg")
    parser.add_argument("files", nargs="*", help="specific files")     # zero or more

    args = parser.parse_args()           # reads sys.argv; prints help & exits on error
    if args.verbose:
        print(f"resizing {args.folder} to width {args.width}")
    return 0

if __name__ == "__main__":
    raise SystemExit(main())             # use the return value as the exit code
```

Run: `python resize.py photos -w 1024 --verbose`. For richer CLIs, **Typer** (type-hint based) and **Click** are the popular libraries; **Rich** adds colors/tables/progress bars.

### 12.8 Reading stdin & exit codes

```python
import sys

# Read piped input (e.g. `cat file | python tool.py`):
data = sys.stdin.read()
for line in sys.stdin:        # line by line
    print(line.rstrip())

# Detect whether stdin is a terminal or a pipe:
if sys.stdin.isatty():
    print("interactive")
else:
    print("reading from a pipe/redirect")

name = input("Your name: ")  # interactive prompt (reads one line)

sys.exit(0)                  # 0 = success, non-zero = error
```

---

## 13. Concurrency & Parallelism

Three models, three jobs: **[A]**

| Model | Best for | Module |
|---|---|---|
| `threading` | I/O-bound (network, disk) — releases the GIL during I/O | `threading`, `concurrent.futures.ThreadPoolExecutor` |
| `multiprocessing` | CPU-bound — sidesteps the GIL with separate processes | `multiprocessing`, `ProcessPoolExecutor` |
| `asyncio` | high-concurrency I/O (thousands of sockets) | `asyncio` |

### 13.1 The GIL

The **Global Interpreter Lock** means only one thread executes Python bytecode at a time (in the default build). Threads still help for I/O-bound work (the GIL is released while waiting on I/O), but not for CPU-bound work — use processes for that.

> **⚡ 2026:** the **free-threaded** build (`python3.13t`, PEP 703) removes the GIL, letting threads run CPU-bound code in parallel. It's officially supported but opt-in, and some C extensions need updates. Check with `import sys; print(sys._is_gil_enabled())` on 3.13+.

### 13.2 Threads & the executor

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import urllib.request

def fetch(url):
    with urllib.request.urlopen(url) as r:
        return url, len(r.read())

urls = ["https://example.com", "https://python.org"]
with ThreadPoolExecutor(max_workers=8) as pool:
    futures = [pool.submit(fetch, u) for u in urls]
    for fut in as_completed(futures):
        url, size = fut.result()
        print(url, size)
    # Or simply: results = list(pool.map(fetch, urls))
```

### 13.3 Multiprocessing for CPU work

```python
from concurrent.futures import ProcessPoolExecutor

def heavy(n):
    return sum(i * i for i in range(n))

if __name__ == "__main__":               # REQUIRED on Windows (spawn) — guard the entry
    with ProcessPoolExecutor() as pool:
        print(list(pool.map(heavy, [10_000_00, 2_000_000])))
```

### 13.4 asyncio

```python
import asyncio

async def fetch(name, delay):
    await asyncio.sleep(delay)            # non-blocking sleep (yields control)
    return f"{name} done"

async def main():
    # Run concurrently and gather results:
    results = await asyncio.gather(
        fetch("a", 1),
        fetch("b", 2),
    )
    print(results)                        # ['a done', 'b done'] after ~2s, not 3

    # TaskGroup (3.11+) — structured concurrency:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(fetch("x", 1))
        tg.create_task(fetch("y", 1))

asyncio.run(main())
```

Use `async`/`await` only with async-aware libraries (e.g. `aiohttp`, `httpx`, async DB drivers). Mixing blocking calls into async code stalls the event loop.

---

## 14. Standard Library Highlights

```python
# datetime — dates & times
from datetime import datetime, timedelta, timezone, date
now = datetime.now(timezone.utc)         # ALWAYS prefer timezone-aware UTC
print(now.isoformat())
print(now + timedelta(days=7))
print(datetime.strptime("2026-06-21", "%Y-%m-%d"))   # parse
print(now.strftime("%Y-%m-%d %H:%M"))                # format

# json
import json
s = json.dumps({"a": 1, "b": [2, 3]}, indent=2)      # obj -> str
obj = json.loads(s)                                   # str -> obj
with open("data.json") as f: obj = json.load(f)       # from file
with open("out.json", "w") as f: json.dump(obj, f)

# re — regular expressions
import re
m = re.search(r"(\d{4})-(\d{2})-(\d{2})", "date: 2026-06-21")
if m:
    print(m.group(0), m.group(1), m.groups())   # full, year, all groups
print(re.findall(r"\w+", "a b c"))               # ['a', 'b', 'c']
print(re.sub(r"\s+", "_", "a  b   c"))           # a_b_c
pattern = re.compile(r"^\d+$")                   # precompile for reuse
print(bool(pattern.match("123")))

# collections
from collections import Counter, defaultdict, namedtuple, deque
print(Counter("mississippi"))                    # Counter({'s':4,'i':4,'p':2,'m':1})
print(Counter("mississippi").most_common(2))     # [('s',4), ('i',4)]
dd = defaultdict(list); dd["x"].append(1)        # no KeyError; auto-creates []
Pt = namedtuple("Pt", "x y"); p = Pt(1, 2); print(p.x, p.y)
q = deque([1, 2, 3]); q.appendleft(0); q.pop()   # fast both-ends queue

# itertools & functools
import itertools, functools
print(list(itertools.chain([1, 2], [3, 4])))     # [1,2,3,4]
print(list(itertools.combinations([1,2,3], 2)))  # [(1,2),(1,3),(2,3)]
print(list(itertools.islice(itertools.count(), 5)))  # [0,1,2,3,4]
print(functools.reduce(lambda a, b: a + b, [1, 2, 3, 4]))  # 10

# logging — use this instead of print() in real apps
import logging
logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
log = logging.getLogger(__name__)
log.info("started"); log.warning("careful"); log.error("oops")

# sqlite3 — a real SQL database in the stdlib, zero setup
import sqlite3
con = sqlite3.connect("app.db")
con.execute("CREATE TABLE IF NOT EXISTS users(id INTEGER PRIMARY KEY, name TEXT)")
con.execute("INSERT INTO users(name) VALUES (?)", ("Ada",))   # ? = safe param binding
con.commit()
for row in con.execute("SELECT * FROM users"):
    print(row)
con.close()
```

---

## 15. Testing

**pytest** is the de-facto standard (cleaner than `unittest`).

```python
# file: test_math.py   (pytest discovers test_*.py and test_* functions)
import pytest
from mathutils import add

def test_add():
    assert add(2, 3) == 5            # plain assert — pytest rewrites it for great messages

@pytest.mark.parametrize("a,b,expected", [(1, 1, 2), (2, 3, 5), (0, 0, 0)])
def test_add_cases(a, b, expected):
    assert add(a, b) == expected

def test_raises():
    with pytest.raises(ValueError):
        int("not a number")

@pytest.fixture
def sample_user():                   # reusable setup; teardown after `yield`
    user = {"name": "Ada"}
    yield user
    # cleanup here if needed

def test_user(sample_user):
    assert sample_user["name"] == "Ada"
```

```bash
pip install pytest
pytest                 # run all tests
pytest -v -k add       # verbose, only tests matching "add"
pytest --cov=myapp     # coverage (needs pytest-cov)
```

Mocking (replace dependencies):

```python
from unittest.mock import patch, MagicMock

def test_api(monkeypatch):
    with patch("myapp.client.requests.get") as mock_get:
        mock_get.return_value.json.return_value = {"ok": True}
        # ... call code that uses requests.get ...
        mock_get.assert_called_once()
```

---

## 16. Packaging & Distribution

```toml
# pyproject.toml — the single source of project metadata
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "mytool"
version = "1.0.0"
description = "A handy CLI"
requires-python = ">=3.12"
dependencies = ["rich>=13"]

[project.scripts]
mytool = "mytool.cli:main"     # installs a `mytool` command
```

```bash
pip install build twine
python -m build                # produces dist/*.whl and dist/*.tar.gz
twine upload dist/*            # publish to PyPI (or use `uv publish`)
pip install -e .              # editable install for local development
```

A package layout:

```
mytool/
├── pyproject.toml
├── README.md
└── src/
    └── mytool/
        ├── __init__.py
        └── cli.py
```

---

## 17. Performance, Idioms & Gotchas

### 17.1 Classic gotchas

```python
# 1) Mutable default arguments are created ONCE, shared across calls:
def bad(item, bucket=[]):       # BUG: same list every call!
    bucket.append(item)
    return bucket
print(bad(1), bad(2))           # [1] [1, 2]  <- surprise
def good(item, bucket=None):    # FIX
    bucket = [] if bucket is None else bucket
    bucket.append(item)
    return bucket

# 2) Late-binding closures in loops:
fns = [lambda: i for i in range(3)]
print([f() for f in fns])       # [2, 2, 2]  <- all see the final i
fns = [lambda i=i: i for i in range(3)]     # FIX: bind via default arg
print([f() for f in fns])       # [0, 1, 2]

# 3) `is` vs `==`: `is` checks identity, `==` checks value. Use `is` only for None.
print(1000 is 1000)             # may be False (don't rely on it)
print(1000 == 1000)             # True

# 4) Copying: assignment shares; use copy for independent data.
import copy
shallow = original[:]                  # or list(original)/dict(original)
deep = copy.deepcopy(nested)           # fully independent

# 5) Floating point: 0.1 + 0.2 != 0.3 — use math.isclose or Decimal for money.
import math
print(math.isclose(0.1 + 0.2, 0.3))    # True
```

### 17.2 Idioms & performance

```python
# Pythonic loops: iterate the thing, not indices.
for item in items: ...                 # not: for i in range(len(items))

# Truthiness over length checks:
if items: ...                          # not: if len(items) > 0

# Build strings with join, not += in a loop:
result = "".join(str(x) for x in range(1000))   # fast
# slow: s = ""; for x in ...: s += str(x)

# Use comprehensions / generators over manual loops where readable.
# Use built-ins: sum, min, max, sorted, any, all, map, filter.
print(any(x > 10 for x in nums), all(x > 0 for x in nums))

# Profiling:
#   python -m cProfile -s cumtime myscript.py
#   import timeit; timeit.timeit("expr", number=10000)

# enumerate, zip, dict.get/setdefault, and unpacking keep code clean and fast.
```

**PEP 8 essentials:** 4-space indent; `snake_case` for functions/variables, `PascalCase` for classes, `UPPER_CASE` for constants; ~79–100 char lines; two blank lines between top-level defs. Let **ruff** or **black** format for you instead of doing it by hand.

---

## 18. Study Path & Build-to-Learn Projects

**Suggested order:** §1–4 (basics) → §5–6 (functions & OOP) → §7–9 (modules, errors, generators) → §12 (file/OS — immediately useful) → §11 (type hints) → §14–15 (stdlib & testing) → §10, §13, §16–17 (advanced).

**Build these to cement it:**
1. **CLI todo manager** — argparse + JSON/sqlite persistence + `pathlib`. Exercises §12, §14.
2. **Log analyzer** — read large files with generators, count with `Counter`, regex-extract fields, print a `rich` table. Exercises §9, §14.
3. **File organizer / backup tool** — walk a tree, move files by type, zip a folder, shell out to `git`. Exercises §12.
4. **Web scraper or API client** — `httpx`/`requests`, async with `asyncio`, retry decorator, typed models with dataclasses. Exercises §10, §11, §13.
5. **A small package** — structure with `src/`, add tests with pytest, type-check with mypy/pyright, publish to a private index. Exercises §15, §16.

**Next steps after this guide:** a web framework (FastAPI or Django), an ORM (SQLAlchemy), data tooling (pandas/polars), and async I/O for real workloads. For the file/OS material in another language, compare with the Go File System / OS / CLI guide in this library.

---

*Part of the offline developer study library. Written for Python 3.13/3.14 as of 2026. Confirm fast-moving APIs against docs.python.org.*
