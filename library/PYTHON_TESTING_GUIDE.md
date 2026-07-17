# Testing in Python — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** People who can write a bit of Python — functions, classes, maybe a small script or web app — but have never written a *test* in their life, or have written a few and suspect they are doing it wrong. You do not need to know any testing vocabulary; this guide defines every term the first time it appears and builds from "what even *is* a test?" up to real property-based, database, and end-to-end suites that run in CI. Read it top-to-bottom the first time so the concepts stack; afterwards use the Table of Contents as a reference. Every idea is explained **prose-first** — *what* it is, *why* it exists, *when* you reach for it, *how* to use it, its key options, and the best-practice/security angle — and only then the heavily-commented, runnable code.
>
> **Version note:** This guide targets **Python 3.13 / 3.14** (which power the modern typing, `tomllib`, and improved error messages the examples assume) with **pytest 8.x** as the default runner and the stdlib **unittest** module for its roots. For coverage it uses **coverage.py 7.x** driven through **pytest-cov 6.x**; for property-based testing **Hypothesis 6.x**; for async **pytest-asyncio 0.25+** and **anyio**; for real-dependency integration **testcontainers-python 4.x** and **pytest-postgresql**; for deterministic time **freezegun 1.5**; for micro-benchmarks **pytest-benchmark 5.x**; for HTTP/API tests the **httpx**-based **Starlette / FastAPI `TestClient`** and the **Django test framework**; for end-to-end **Playwright for Python 1.5x**; for mutation testing **mutmut 3.x** / **cosmic-ray**; and for multi-environment orchestration **tox 4** / **nox**. Parallelism uses **pytest-xdist** and flaky-test retries use **pytest-rerunfailures**. Where an API changed recently, look for **⚡ Version note**. The author is on **Windows 11**, so OS-specific caveats (paths, event loops, subprocess) are flagged where they matter.
>
> **This guide's place in the library:** Testing is a cross-cutting skill, so this guide leans on several others. Language fundamentals (decorators, generators, context managers, `asyncio`, type hints) live in [Python](PYTHON_GUIDE.md). The web frameworks whose test tooling we exercise are [FastAPI](FASTAPI_GUIDE.md) and [Django](DJANGO_GUIDE.md). The database we spin up for integration tests is [PostgreSQL](POSTGRESQL_GUIDE.md). Running the whole suite automatically on every push is the job of [GitHub Actions CI/CD](GITHUB_ACTIONS_CICD_GUIDE.md), and the throwaway Postgres containers come from [Docker](DOCKER_GUIDE.md). Cross-links appear inline throughout.

---

## Table of Contents

1. [Why Test — the Pyramid and Efficient Tests](#1-why-test--the-pyramid-and-efficient-tests) **[B]**
2. [unittest — the Standard Library Roots](#2-unittest--the-standard-library-roots) **[B]**
3. [pytest Fundamentals](#3-pytest-fundamentals) **[B]**
4. [Fixtures — pytest's Core Concept](#4-fixtures--pytests-core-concept) **[B/I]**
5. [Parametrization, Markers and Conditional Tests](#5-parametrization-markers-and-conditional-tests) **[I]**
6. [Mocking and Test Doubles](#6-mocking-and-test-doubles) **[I]**
7. [Coverage — Measuring What You Test](#7-coverage--measuring-what-you-test) **[I]**
8. [Async Testing](#8-async-testing) **[I/A]**
9. [HTTP and API Integration Testing](#9-http-and-api-integration-testing) **[I/A]**
10. [Database Integration Testing](#10-database-integration-testing) **[I/A]**
11. [End-to-End Testing with Playwright](#11-end-to-end-testing-with-playwright) **[A]**
12. [Property-Based Testing with Hypothesis](#12-property-based-testing-with-hypothesis) **[A]**
13. [Benchmarking with pytest-benchmark](#13-benchmarking-with-pytest-benchmark) **[I]**
14. [doctest — Testing Your Docstrings](#14-doctest--testing-your-docstrings) **[B]**
15. [Fuzzing with Atheris](#15-fuzzing-with-atheris) **[A]**
16. [Testing Filesystem, OS and Subprocess Code](#16-testing-filesystem-os-and-subprocess-code) **[I]**
17. [Mutation Testing](#17-mutation-testing) **[A]**
18. [Multi-Env and CI](#18-multi-env-and-ci) **[I/A]**
19. [Gotchas and Best Practices](#19-gotchas-and-best-practices) **[I/A]**
20. [Study Path and Build-to-Learn Projects](#20-study-path-and-build-to-learn-projects)

---

## 1. Why Test — the Pyramid and Efficient Tests

### What a test actually is

Strip away the tooling and a **test is just a small program that runs your real program with known inputs and checks that the output is what you expected.** That is the whole idea. If you have ever added a `print()` to a function, run it, and eyeballed the output to see if it looked right, you have already done manual testing. An *automated* test replaces the eyeballing with a machine-checked assertion: instead of "*looks* like `add(2, 3)` printed 5", you write `assert add(2, 3) == 5`, and a test runner executes that line for you every time — reporting **pass** if the assertion held and **fail** (with a detailed diff) if it did not.

The value is not in running the check once. It is in running it *thousands of times, for free, forever.* The first time you write `assert add(2, 3) == 5` you confirm the function works today. Every subsequent time the suite runs — after you refactor, after a teammate edits nearby code, after you upgrade a dependency — that same line silently re-verifies the behaviour still holds. **Tests are the mechanism that lets you change code without fear.** Without them, every edit is a gamble: does this break something three modules away? With a good suite, you make the change, run the tests, and the machine tells you in seconds. This is why testing is not a chore bolted on at the end but the thing that makes fast, confident development *possible* at all.

### Why test — the concrete payoffs

- **Regression protection.** A *regression* is a bug in something that used to work. Tests are a tripwire: the moment a change breaks old behaviour, a previously-green test goes red and points at the exact broken case.
- **Design pressure.** Code that is hard to test is usually hard to *use* — tangled dependencies, hidden global state, functions that do five things. Writing the test first (or soon after) surfaces those problems while they are cheap to fix. Testable code and well-designed code are almost the same thing.
- **Executable documentation.** A test named `test_withdraw_more_than_balance_raises_insufficient_funds` tells the next reader — including future you — exactly what the system is supposed to do, and unlike a comment it *cannot* silently go out of date, because CI fails the instant it stops being true.
- **Confidence to refactor and ship.** With coverage you trust, you can restructure aggressively and deploy on Friday. Without it, code ossifies because nobody dares touch it.
- **Faster debugging.** A failing unit test localises a bug to one function; a bug reported from production could be anywhere. Tests shrink the search space.

### The test pyramid — how much of each kind to write

Tests come in a **taxonomy** ordered by *how much of the system each one exercises* and therefore how fast, isolated, and stable it is. The classic mental model is the **test pyramid**: many small tests at the bottom, fewer large ones at the top.

```text
            /\
           /  \        End-to-end (e2e)      few, slow, high-confidence,
          /----\       browser / full stack  brittle, expensive
         /      \
        /--------\      Integration           some — real DB, real HTTP,
       /          \     modules together      multiple components wired up
      /------------\
     /              \   Unit                   many, fast, isolated,
    /----------------\  one function/class     deterministic, cheap
```

- **Unit tests** (the wide base) exercise **one small piece in isolation** — a single function, method, or class — with its collaborators replaced by simple stand-ins. They are *fast* (microseconds), *deterministic*, and *precise* about where a failure is. You should have the most of these.
- **Integration tests** (the middle) exercise **several real components wired together** — your code talking to a real Postgres database, or your web handler going through the real routing and serialization stack. They catch the bugs unit tests cannot: wrong SQL, mismatched serialization, misconfigured wiring. They are slower (milliseconds to seconds) because real I/O is involved, so you write fewer.
- **End-to-end tests** (the narrow tip) drive the **entire system the way a user does** — a real browser clicking through a real deployed app hitting a real database. They give the highest confidence that the whole thing works, but they are slow (seconds each), flaky (a network blip or a timing race fails them), and expensive to maintain. Write only a handful, covering critical user journeys.

There are more kinds layered on this shape, all covered in this guide: **property-based** tests (§12) that generate hundreds of inputs to find edge cases; **doctests** (§14) embedded in docstrings; **benchmarks** (§13) that measure speed rather than correctness; **fuzz** tests (§15) that hammer parsers with malformed bytes; **mutation** tests (§17) that test your tests; and **smoke/regression/contract/snapshot** tests which are usage patterns rather than separate tools.

⚡ **Version note:** A popular counter-model is the **testing trophy** (Kent C. Dodds), which fattens the *integration* layer on the argument that integration tests give the best confidence-per-effort for modern apps. The pyramid vs trophy debate is about *proportions*, not *kinds* — both agree you want a broad fast base and a thin slow top. Use judgement: a pure library skews unit-heavy; a thin web service over a database is genuinely mostly integration.

### What makes a test *efficient* — fast, isolated, deterministic

Writing tests is easy; writing *good* tests is the skill. Three properties separate a suite you trust and run constantly from one you learn to ignore.

- **Fast.** The whole point of a suite is that you run it after every change. A suite that takes ten minutes gets run once a day; a suite that takes ten seconds gets run every save. Speed comes from having mostly unit tests, avoiding needless I/O, sleeping *never* (see §8/§19), and parallelising (§18). Slowness is not a minor annoyance — it silently destroys the habit that gives tests their value.
- **Isolated.** Each test must set up its own world and clean up after itself, sharing *nothing* mutable with other tests. If test A leaves a row in a database or mutates a module-level list that test B reads, the two are coupled: they pass or fail depending on **order**, tests become impossible to run individually, and failures become non-reproducible. Isolation is what lets you run one test, run them in parallel, or run them shuffled, and always get the same answer.
- **Deterministic.** A test must give the **same result every run** given the same code. The enemies of determinism are the *ambient* inputs your code reads without you passing them: the current time (`datetime.now()`), random numbers, network responses, filesystem state, environment variables, dict/set ordering in old code, and concurrency races. A test that reads real wall-clock time will fail at midnight or in a different timezone; a test that hits a real API will fail when the network hiccups. A test that fails *sometimes* is called **flaky**, and a flaky test is arguably worse than no test — people learn to re-run until green, which trains them to ignore real failures too. The cures — injecting the clock, freezing time (§6), seeding or mocking randomness, faking HTTP (§6), using `tmp_path` (§16) — are a recurring theme of this guide.

A useful compact acronym for isolation/speed goals is **FIRST**: **F**ast, **I**solated, **R**epeatable (deterministic), **S**elf-validating (the test asserts pass/fail itself, no human reads output), and **T**imely (written close to the code).

### AAA — the shape of every good test

Almost every well-written test, regardless of framework, has the same three-part shape, called **Arrange-Act-Assert (AAA)**. Naming these phases in your head (and often with blank lines or comments in the code) keeps tests readable and focused:

1. **Arrange** — set up the world the test needs: construct objects, prepare inputs, seed a database, configure fakes. This is the "given" precondition.
2. **Act** — perform the *single* action under test: call the one function or hit the one endpoint whose behaviour you are verifying. Keep this to one logical operation; if you find yourself acting twice, you probably have two tests.
3. **Assert** — check that the outcome is what you expected: the return value, the raised exception, the state change, the recorded side effect. This is the machine-checked "then".

```python
# A complete test in the AAA shape. This is plain pytest (covered fully in §3),
# but the SHAPE is universal — you'll see it in unittest, Django, everywhere.

def add(a, b):          # the "production" code under test (normally in another module)
    return a + b

def test_add_combines_two_numbers():
    # ---- Arrange: prepare the known inputs -------------------------------
    x = 2
    y = 3

    # ---- Act: perform the single operation being tested ------------------
    result = add(x, y)

    # ---- Assert: verify the outcome matches expectation ------------------
    assert result == 5   # if this is false, pytest prints a rich diff and fails
```

A tiny but important habit lives in the *name*: `test_add_combines_two_numbers` reads like a sentence describing a behaviour. A great test name states **the scenario and the expected outcome** (`test_withdraw_more_than_balance_raises`), so that when it fails, the failure report alone tells you what broke without opening the code. Prefer this over vague names like `test_add` or `test_1`.

### A minimal first run

You do not need a project to try this. Save the two functions above into a file named `test_add.py` and run pytest (installation is in §3):

```bash
# Install pytest into a virtual environment (details in §3).
python -m venv .venv
source .venv/bin/activate        # Windows PowerShell: .venv\Scripts\Activate.ps1
pip install pytest

# Run every test pytest can discover from the current directory.
pytest
# ── output ──────────────────────────────────────────────
# test_add.py .                                      [100%]
# 1 passed in 0.01s
```

That single green dot is the entire loop in miniature: pytest found a file starting with `test_`, found a function starting with `test_`, ran it, saw the assertion hold, and reported success. Everything else in this guide is elaboration on that loop — better ways to arrange, richer things to assert, and techniques to keep tests fast, isolated, and deterministic as the system under test grows from `add()` into a database-backed async web application.

---

## 2. unittest — the Standard Library Roots

### What unittest is and why we start here

**`unittest`** is Python's *built-in* testing framework — it ships with the interpreter, so `import unittest` works everywhere with zero installation. It is a Python port of the "xUnit" family (originally Java's JUnit / Smalltalk's SUnit), which is why it looks the way it does: tests are **methods** on a class that inherits from `unittest.TestCase`, assertions are **methods** on that class (`self.assertEqual(...)`), and shared setup lives in specially-named methods (`setUp`, `tearDown`).

We start here for two reasons even though the rest of the guide uses pytest. First, **you will read `unittest` code** — it is in the standard library, the Django test framework is built on it (§9), and countless existing codebases use it, so you must be able to recognise and maintain it. Second, understanding the xUnit vocabulary — *test case*, *fixture*, *setup/teardown*, *test suite*, *runner* — makes the leap to pytest's cleaner design meaningful rather than magical. You will *feel* why pytest's plain `assert` and fixtures are an improvement because you will have felt the friction they remove.

### The anatomy of a TestCase

The core concepts map onto the AAA shape from §1:

- A **test case** is one scenario. In `unittest` it is a *method* whose name starts with `test`, living on a subclass of `unittest.TestCase`.
- `setUp(self)` runs **before every test method** in the class — this is where the shared **Arrange** goes (build fresh objects so each test starts clean). The word "fixture" in xUnit means exactly this: the fixed baseline state a test runs against.
- `tearDown(self)` runs **after every test method**, even if it failed — this is cleanup (close files, delete temp data). It is the counterpart to `setUp`.
- **Assertions** are methods: `self.assertEqual(a, b)`, `self.assertTrue(x)`, `self.assertRaises(...)`, and many more. They are methods (not the plain `assert` statement) because `unittest` predates pytest's assertion-rewriting trick and needs each assertion to know how to format a good failure message.

```python
# file: test_account_unittest.py
import unittest

# ---- The production code we're testing (normally in its own module) -------
class BankAccount:
    """A tiny account supporting deposits and withdrawals."""
    def __init__(self, balance=0):
        self.balance = balance

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("deposit must be positive")
        self.balance += amount
        return self.balance

    def withdraw(self, amount):
        if amount > self.balance:
            raise ValueError("insufficient funds")
        self.balance -= amount
        return self.balance


# ---- The tests ------------------------------------------------------------
class TestBankAccount(unittest.TestCase):
    """Each `test_*` method is one independent scenario.

    unittest creates a BRAND-NEW TestBankAccount instance for every test
    method, then runs setUp() on it, then the test, then tearDown(). That
    fresh-instance-per-test rule is what gives you isolation (§1).
    """

    def setUp(self):
        # Runs before EACH test method → the shared "Arrange" step.
        # Because it re-runs per test, self.account is always pristine.
        self.account = BankAccount(balance=100)

    def tearDown(self):
        # Runs after EACH test method (even on failure). Nothing to clean
        # up here, but this is where you'd close files/connections.
        pass

    def test_deposit_increases_balance(self):
        new_balance = self.account.deposit(50)          # Act
        self.assertEqual(new_balance, 150)              # Assert value
        self.assertEqual(self.account.balance, 150)     # Assert state

    def test_withdraw_decreases_balance(self):
        self.account.withdraw(40)
        self.assertEqual(self.account.balance, 60)

    def test_withdraw_too_much_raises(self):
        # assertRaises used as a context manager: the block MUST raise
        # ValueError, or the test fails. This is unittest's pytest.raises.
        with self.assertRaises(ValueError):
            self.account.withdraw(1000)

    def test_deposit_negative_raises_with_message(self):
        with self.assertRaises(ValueError) as ctx:
            self.account.deposit(-5)
        # You can inspect the captured exception after the block.
        self.assertIn("positive", str(ctx.exception))


# Lets you run the file directly: `python test_account_unittest.py`
if __name__ == "__main__":
    unittest.main()
```

Run it either by executing the file (`python test_account_unittest.py`) or, better, with the module runner which discovers tests across a project:

```bash
# Discover and run every test in files matching test*.py under the cwd.
python -m unittest discover -s . -p "test_*.py" -v
# ── output ──────────────────────────────────────────────
# test_deposit_increases_balance (test_account_unittest.TestBankAccount) ... ok
# test_deposit_negative_raises_with_message (...) ... ok
# test_withdraw_decreases_balance (...) ... ok
# test_withdraw_too_much_raises (...) ... ok
# ----------------------------------------------------------------------
# Ran 4 tests in 0.001s
# OK
```

### The assertion method catalogue

Because `unittest` cannot use plain `assert`, it gives you a specific method for each kind of check, each producing a tailored failure message. The common ones:

| Method | Passes when | Typical use |
|---|---|---|
| `assertEqual(a, b)` | `a == b` | value equality (most common) |
| `assertNotEqual(a, b)` | `a != b` | values differ |
| `assertTrue(x)` / `assertFalse(x)` | `x` is truthy / falsy | boolean conditions |
| `assertIs(a, b)` / `assertIsNot(a, b)` | `a is b` (identity) | singletons like `None`, sentinels |
| `assertIsNone(x)` / `assertIsNotNone(x)` | `x is None` / not | optional return values |
| `assertIn(a, b)` / `assertNotIn(a, b)` | `a in b` | membership in list/str/dict |
| `assertIsInstance(o, cls)` | `isinstance(o, cls)` | type checks |
| `assertRaises(Exc)` | block raises `Exc` | error paths (context manager) |
| `assertRaisesRegex(Exc, r)` | raises `Exc` and message matches regex | error message content |
| `assertAlmostEqual(a, b)` | `a ≈ b` to 7 decimal places | float comparison |
| `assertGreater` / `assertLess` / `...Equal` | ordering | numeric ranges |
| `assertCountEqual(a, b)` | same elements, any order | comparing collections ignoring order |
| `assertListEqual` / `assertDictEqual` / `assertSetEqual` | typed equality with better diffs | container-specific messages |

### Class-level fixtures and skipping

Sometimes setup is *expensive* and safe to share across every test in a class (a read-only reference dataset, a compiled resource). `setUpClass`/`tearDownClass` run **once per class** instead of once per method. Because they are shared, they must be treated as read-only by tests — mutating class-level state reintroduces the coupling `setUp` was designed to avoid.

```python
import unittest

class TestWithExpensiveSetup(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        # Runs ONCE before any test in this class. Use for costly, read-only
        # resources. Note @classmethod: it receives the class, not an instance.
        cls.reference_data = list(range(1_000_000))   # pretend this is slow

    @classmethod
    def tearDownClass(cls):
        cls.reference_data = None                      # release it once, at the end

    def test_contains_zero(self):
        self.assertIn(0, self.reference_data)

    @unittest.skip("demonstrates an unconditional skip")
    def test_not_ready_yet(self):
        ...

    @unittest.skipIf(  # skip conditionally, e.g. platform-specific behaviour
        __import__("sys").platform == "win32",
        "POSIX-only behaviour",
    )
    def test_posix_only(self):
        ...

    @unittest.expectedFailure   # known-broken: fails if it UNEXPECTEDLY passes
    def test_known_bug(self):
        self.assertEqual(1, 2)
```

### Why we pivot to pytest

`unittest` works, and for a tiny project it is fine. But three frictions compound as suites grow, and they are exactly what pytest removes:

1. **Assertion verbosity.** You must remember and type the right `assertX` method for every check, and combined conditions read awkwardly (`self.assertTrue(a < b and b < c)` gives a useless "False is not True" on failure). pytest lets you write the plain Python you already know — `assert a < b < c` — and, through **assertion rewriting** (§3), *still* prints a rich breakdown of exactly which subexpression was what.
2. **Fixtures are rigid.** `setUp` runs for *every* test in a class whether that test needs it or not, there is exactly one per class, and sharing setup *across* classes means inheritance gymnastics. pytest's fixtures (§4) are composable, requested by name only where needed, scoped flexibly (function/class/module/session), and reusable across the whole suite via `conftest.py`.
3. **Boilerplate.** Every test is a method on a class that must subclass `TestCase`. pytest tests are plain functions — less ceremony, less indentation, less to explain to a newcomer.

pytest is also **backward-compatible**: it can discover and run your existing `unittest.TestCase` classes unchanged, so migration is incremental — point pytest at a `unittest` suite and it just runs. That is why the modern default, and the rest of this guide, is **pytest**, while your `unittest` knowledge remains useful for reading and maintaining the code that is already out there.

⚡ **Version note:** On Python 3.11+ `unittest` gained `enterContext`/`enterClassContext` for registering context managers as fixtures, and 3.12/3.13 continue to sharpen failure diffs. Modern `unittest` is perfectly capable — the pivot to pytest is about ergonomics and the fixture/plugin ecosystem, not because `unittest` is broken.

---

## 3. pytest Fundamentals

### What pytest is and why it won

**pytest** is a third-party test framework (`pip install pytest`) that has become the de-facto standard for Python testing. Its central design bet is **"tests should be plain functions with plain `assert` statements,"** and everything else — fixtures, parametrization, a huge plugin ecosystem — grows from keeping the common case trivially simple. You write a function named `test_something`, put `assert` statements in it, and run `pytest`. There is no class to subclass, no assertion methods to memorise, no `main` to call.

The magic that makes plain `assert` viable is **assertion rewriting**. Normally Python's `assert x == y` either passes silently or raises a bare `AssertionError` with no detail. At import time, pytest rewrites the bytecode of your test modules so that every `assert` is instrumented: when it fails, pytest **introspects the expression**, evaluates its sub-parts, and prints a detailed breakdown — the actual values, a diff for strings/lists/dicts, the works. You get `unittest`'s rich messages *and* the readability of plain Python, with none of the method-name lookup.

```python
def test_assertion_rewriting_shows_detail():
    expected = {"name": "Ada", "role": "admin", "age": 36}
    actual   = {"name": "Ada", "role": "user",  "age": 36}
    assert actual == expected
# On failure pytest prints something like:
#   E   AssertionError: assert {'age': 36, 'name': 'Ada', 'role': 'user'} == {...}
#   E     Common items: {'age': 36, 'name': 'Ada'}
#   E     Differing items: {'role': 'user'} != {'role': 'admin'}
#   E     Full diff: ...
# You instantly see it's the `role` key — no custom message needed.
```

### Installation and project layout

Install pytest into a **virtual environment** (an isolated per-project Python so test dependencies never pollute your system). Test *discovery* — how pytest finds what to run — follows simple, conventional rules, so laying the project out the standard way means everything "just works" with zero configuration.

```bash
python -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\Activate.ps1
pip install pytest
```

pytest's **default discovery rules** (memorise these — most "pytest found no tests" confusion is a naming violation):

- It searches the directories you point it at (default: current dir) recursively.
- It collects files named **`test_*.py`** or **`*_test.py`**.
- Inside those files it collects **functions** named **`test_*`**, and **methods** named `test_*` inside **classes** named **`Test*`** (the class must **not** have an `__init__` method, or pytest skips it).
- A plain `assert` failing marks the test failed; a test that finishes without an uncaught exception passes.

The recommended project layout is the **`src` layout**, which puts your package under `src/` and tests in a sibling `tests/` directory. This forces your tests to import the *installed* package (via `pip install -e .`) rather than accidentally importing loose files by relative path, which catches packaging bugs early:

```text
myproject/
├── pyproject.toml          # project + pytest config (see below)
├── src/
│   └── myapp/
│       ├── __init__.py
│       └── calculator.py
└── tests/
    ├── conftest.py         # shared fixtures (see §4)
    ├── test_calculator.py
    └── integration/
        └── test_api.py
```

### Configuration — pyproject.toml, pytest.ini, tox.ini

pytest reads its configuration from one of a few files at the project root. The modern default is a `[tool.pytest.ini_options]` table in **`pyproject.toml`** (the standard Python project file), but a dedicated **`pytest.ini`** works too and its presence unambiguously marks the project root. Configuring `testpaths`, `addopts` (default command-line options), and registering markers up front saves repetition and prevents whole classes of mistake.

```toml
# pyproject.toml
[tool.pytest.ini_options]
# Where to look for tests (speeds discovery; avoids scanning .venv etc.).
testpaths = ["tests"]

# Options applied to EVERY run, as if typed on the command line.
#   -ra           show a summary of all non-passing test reasons at the end
#   --strict-markers  error on unregistered @pytest.mark.X (catches typos)
#   --strict-config   error on unknown config keys
#   -q            quieter output
addopts = "-ra --strict-markers --strict-config -q"

# Register custom markers here so --strict-markers accepts them (see §5).
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: tests that need a real database or network",
]

# Only needed if you deviate from the default naming conventions:
python_files = ["test_*.py", "*_test.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
```

```ini
; pytest.ini — the equivalent if you prefer a dedicated file over pyproject.toml.
[pytest]
testpaths = tests
addopts = -ra --strict-markers -q
markers =
    slow: marks tests as slow
    integration: tests that need a real database or network
```

⚡ **Version note:** pytest 8.x tightened several long-deprecated behaviours (stricter marker handling, nose-style setup removal) and requires Python 3.8+. `--strict-markers` is strongly recommended on any new project — without it, a typo like `@pytest.mark.slwo` silently creates a *new* meaningless marker and the test is never deselected as intended.

### Running, filtering, and reading output

You control *which* tests run and *how loud* the output is with command-line flags. Fluency here is what makes the edit-run-fix loop fast — during development you rarely run the whole suite; you run the one test you are working on.

| Command | What it does |
|---|---|
| `pytest` | discover and run everything under `testpaths` |
| `pytest tests/test_calculator.py` | run just one file |
| `pytest tests/test_calculator.py::test_add` | run one specific test function |
| `pytest tests/test_x.py::TestClass::test_method` | run one method in a test class |
| `pytest -k "add and not negative"` | run tests whose **name** matches the keyword expression |
| `pytest -m slow` | run tests **marked** `@pytest.mark.slow` (§5) |
| `pytest -m "not slow"` | run everything *except* slow tests |
| `pytest -x` | stop at the **first** failure (fast feedback) |
| `pytest --maxfail=3` | stop after 3 failures |
| `pytest -q` / `-v` / `-vv` | quiet / verbose / extra-verbose (full diffs) |
| `pytest -s` | do **not** capture stdout — let `print()` show (debugging) |
| `pytest --lf` | run only the tests that **failed last** time |
| `pytest --ff` | run last-failed **first**, then the rest |
| `pytest --sw` | "stepwise": stop at first failure, resume there next run |
| `pytest -p no:cacheprovider` | disable the `.pytest_cache` directory |
| `pytest --collect-only` | list what *would* run without running it |
| `pytest -rA` | show extra summary info for all outcomes |

The **`-k`** expression matches against the test's *node id* (file + class + function name), supports `and`/`or`/`not`, and is substring-based — `-k add` runs `test_add`, `test_addition`, and `test_readable`. The **`--lf`/`--ff`/`--sw`** family, backed by pytest's cache, is the secret to a tight loop: fix a failure, `pytest --lf` to re-run only what broke, repeat until green, then `pytest` for the full sweep.

### Asserting on exceptions with pytest.raises

Verifying that code **raises** the right error on bad input is as important as verifying it returns the right value on good input — the error path *is* behaviour. pytest's `pytest.raises` is a context manager: the `with` block must raise the expected exception type (or a subclass), otherwise the test fails. Capture the exception object via `as excinfo` to assert on its message or attributes.

```python
import pytest

def parse_positive_int(text):
    value = int(text)                       # raises ValueError on non-numeric
    if value <= 0:
        raise ValueError(f"expected a positive integer, got {value}")
    return value

def test_rejects_non_numeric():
    # The block MUST raise ValueError; if it doesn't, the test fails.
    with pytest.raises(ValueError):
        parse_positive_int("banana")

def test_rejects_zero_with_helpful_message():
    with pytest.raises(ValueError) as excinfo:
        parse_positive_int("0")
    # `excinfo` holds the captured exception; .value is the exception object.
    assert "positive" in str(excinfo.value)

def test_message_matches_regex():
    # `match` checks the exception's str() against a regex — a common shortcut.
    with pytest.raises(ValueError, match=r"got -5"):
        parse_positive_int("-5")

def test_specific_exception_group():   # Python 3.11+ ExceptionGroups
    with pytest.raises(ExceptionGroup) as ei:
        raise ExceptionGroup("boom", [ValueError("a"), TypeError("b")])
    assert len(ei.value.exceptions) == 2
```

A subtle but important habit: **make the `with pytest.raises(...)` block as small as possible** — ideally the single call under test. If you wrap several statements, a different line could raise the same exception type and your test would pass for the wrong reason.

### Comparing floats with pytest.approx

Floating-point arithmetic is *inexact* — `0.1 + 0.2` is `0.30000000000000004`, not `0.3`, because those decimals cannot be represented exactly in binary. Asserting `== 0.3` therefore fails. `pytest.approx` compares numbers within a small tolerance, and it works element-wise on lists, tuples, dicts, and numpy arrays too.

```python
import pytest

def test_float_math_needs_approx():
    # This would FAIL: assert 0.1 + 0.2 == 0.3
    assert 0.1 + 0.2 == pytest.approx(0.3)          # passes: within default tolerance

def test_approx_tolerances():
    # rel: relative tolerance (fraction of the expected value)
    # abs: absolute tolerance (fixed epsilon), useful near zero
    assert 1000.0 == pytest.approx(1001.0, rel=1e-2)     # within 1% → passes
    assert 0.0001 == pytest.approx(0.0, abs=1e-3)        # near zero → use abs

def test_approx_on_collections():
    assert [0.1 + 0.2, 0.2 + 0.4] == pytest.approx([0.3, 0.6])
    assert {"x": 0.1 + 0.2} == pytest.approx({"x": 0.3})
```

### A realistic first test module

Putting the fundamentals together — discovery, plain asserts, `raises`, `approx`, and clear AAA structure — here is a complete module testing a small `calculator` under `src/myapp/`:

```python
# src/myapp/calculator.py
def add(a, b):
    return a + b

def divide(a, b):
    if b == 0:
        raise ZeroDivisionError("cannot divide by zero")
    return a / b

def mean(numbers):
    if not numbers:
        raise ValueError("mean() requires at least one number")
    return sum(numbers) / len(numbers)
```

```python
# tests/test_calculator.py
import pytest
from myapp.calculator import add, divide, mean   # imports the INSTALLED package

# ---- happy paths ----------------------------------------------------------
def test_add_two_positives():
    assert add(2, 3) == 5

def test_divide_produces_float():
    assert divide(7, 2) == pytest.approx(3.5)

def test_mean_of_list():
    assert mean([2, 4, 6]) == pytest.approx(4.0)

# ---- error paths ----------------------------------------------------------
def test_divide_by_zero_raises():
    with pytest.raises(ZeroDivisionError, match="cannot divide"):
        divide(1, 0)

def test_mean_of_empty_raises():
    with pytest.raises(ValueError):
        mean([])

# ---- a test class groups related tests (no TestCase subclassing needed) ----
class TestAddEdgeCases:
    def test_negatives(self):
        assert add(-2, -3) == -5

    def test_add_is_commutative(self):
        assert add(2, 9) == add(9, 2)
```

Everything from here builds on this base. §4 removes the repetition of building test inputs by hand (fixtures); §5 removes the repetition of near-identical tests (parametrization); §6 lets you replace real collaborators with controllable fakes; and the later sections apply all of it to async code, HTTP APIs, databases, and browsers.

---

## 4. Fixtures — pytest's Core Concept

### What a fixture is and the problem it solves

Look back at the `unittest` example in §2: every test needed a fresh `BankAccount`, so we built one in `setUp`. That works, but it is inflexible — `setUp` runs for *every* test in the class, gives you exactly *one* baseline, and cannot be shared with a different class without inheritance. **Fixtures are pytest's answer, and they are the single most important thing to understand about the framework.** A fixture is a **function that produces a piece of test scaffolding** — a built object, a temp directory, a database connection, a logged-in client — which any test can request simply by naming it as a **parameter**. pytest sees the parameter name, finds the matching fixture, runs it, and passes the result in. This is **dependency injection**: the test declares *what it needs*, and pytest supplies it.

The payoff over `setUp` is enormous: fixtures are **requested à la carte** (a test only gets what it names, so no wasted setup), **composable** (a fixture can request other fixtures), **reusable across the whole suite** (defined once in `conftest.py`, available everywhere below it), **scoped** (built once per function, class, module, or session as you choose), and they **handle their own teardown** cleanly via `yield`.

```python
import pytest

class BankAccount:
    def __init__(self, balance=0): self.balance = balance
    def deposit(self, amt): self.balance += amt; return self.balance
    def withdraw(self, amt):
        if amt > self.balance: raise ValueError("insufficient funds")
        self.balance -= amt; return self.balance

# ---- A fixture: a function decorated with @pytest.fixture -----------------
@pytest.fixture
def account():
    """Produce a fresh account with a known starting balance.

    The RETURN value is what gets injected. Because pytest re-runs this
    function for each test that requests it (default function scope), every
    test gets its own pristine BankAccount — isolation for free.
    """
    return BankAccount(balance=100)

# ---- Tests REQUEST the fixture by naming it as a parameter ----------------
def test_deposit(account):                 # pytest sees `account`, runs the
    assert account.deposit(50) == 150      # fixture, injects the result here

def test_withdraw(account):                # a DIFFERENT, fresh account —
    assert account.withdraw(40) == 60      # the two tests can't interfere

def test_overdraw_raises(account):
    with pytest.raises(ValueError):
        account.withdraw(1000)
```

### Setup and teardown with yield

Real scaffolding often needs **cleanup**: a file to close, a connection to drop, a temp directory to delete, a global to restore. A fixture handles both halves by using **`yield`** instead of `return`. Everything *before* the `yield` is setup (runs when the fixture is requested); the yielded value is injected into the test; everything *after* the `yield` is teardown (runs after the test finishes, **even if the test failed**, because pytest wraps it so the teardown always executes). This is the same pattern as a context manager, and it keeps a resource's acquisition and release in one readable place.

```python
import pytest

@pytest.fixture
def db_connection():
    # ---- setup: runs before the test ------------------------------------
    conn = open_connection("sqlite:///:memory:")   # pretend connect
    conn.execute("CREATE TABLE users (id INT, name TEXT)")

    yield conn          # <-- the test runs here, receiving `conn`

    # ---- teardown: runs after the test, pass OR fail --------------------
    conn.close()        # guaranteed cleanup — no leaked connections

def test_insert(db_connection):
    db_connection.execute("INSERT INTO users VALUES (1, 'Ada')")
    assert db_connection.query("SELECT count(*) FROM users") == 1
    # after this line (or on failure), db_connection's teardown runs
```

⚡ **Version note:** The old `@pytest.fixture` + separate `request.addfinalizer(...)` style still works, but `yield` is the modern, readable default. Prefer it unless you need to register several finalizers dynamically.

### Fixture scope — controlling how often setup runs

By default a fixture is rebuilt **once per test function** — maximally isolated, but wasteful if the setup is expensive and safe to reuse (starting a database container, loading a large model, building an app instance). The **`scope`** parameter trades isolation for speed by widening the lifetime:

| Scope | Fixture runs once per… | Use for |
|---|---|---|
| `function` (default) | each test function | anything mutable; the safe default |
| `class` | test class | shared read-only setup for a group of methods |
| `module` | `.py` file | expensive resource reused by a file's tests |
| `package` | test package (directory) | resource shared across a subtree |
| `session` | the entire test run | the most expensive, truly-shared things: a DB container, a browser |

The rule of thumb: **use the narrowest scope that is correct.** A wider scope is faster but only safe if tests do not mutate the shared object (or reliably reset it). A session-scoped Postgres *container* is fine because it is expensive to start; but you still give each test an isolated *transaction* inside it (§10) so the shared container never becomes shared mutable state.

```python
import pytest

@pytest.fixture(scope="session")
def expensive_resource():
    # Built ONCE for the whole test run — e.g. start a DB, load a model.
    print("\n[session setup] starting expensive resource")
    resource = {"loaded": True, "handle": object()}
    yield resource
    print("\n[session teardown] releasing expensive resource")   # once, at the end

@pytest.fixture(scope="module")
def module_data():
    # Built once per test file that uses it.
    return list(range(1000))

def test_a(expensive_resource, module_data):
    assert expensive_resource["loaded"]

def test_b(expensive_resource):          # reuses the SAME resource as test_a
    assert expensive_resource["handle"] is not None
```

### conftest.py — sharing fixtures across files

A fixture defined in a test file is only visible in that file. To share fixtures across many test files, put them in a **`conftest.py`** — a special file pytest loads automatically and whose fixtures become available to **every test in that directory and all subdirectories**, with *no import needed*. This is how you build a project-wide library of scaffolding (a database session, an HTTP client, a factory for your domain objects). You can have multiple `conftest.py` files at different levels; deeper ones add or override fixtures for their subtree. Because pytest discovers them by location, `conftest.py` is also where plugin hooks and shared markers live.

```python
# tests/conftest.py — fixtures here are available to ALL tests under tests/
import pytest

@pytest.fixture
def sample_user():
    return {"id": 1, "name": "Ada", "role": "admin"}

@pytest.fixture(scope="session")
def app_config():
    # Session-scoped config every test can read without re-parsing.
    return {"env": "test", "debug": True}
```

```python
# tests/test_users.py — no import; pytest injects conftest fixtures by name
def test_user_is_admin(sample_user):         # comes from conftest.py
    assert sample_user["role"] == "admin"
```

### Fixture composition — fixtures that use fixtures

Fixtures can **request other fixtures** exactly the way tests do — by naming them as parameters. This lets you build small, single-purpose fixtures and compose them into richer ones, keeping each piece simple and reusable. A common chain: a session-scoped `engine` → a function-scoped `connection` → a function-scoped `db_session` that begins and rolls back a transaction. Each layer depends on the one below, and pytest resolves the whole graph automatically.

```python
import pytest

@pytest.fixture(scope="session")
def engine():
    print("\n[create engine once]")
    return {"kind": "engine"}                 # pretend SQLAlchemy engine

@pytest.fixture
def connection(engine):                       # requests the session engine
    conn = {"engine": engine, "open": True}   # a fresh connection per test
    yield conn
    conn["open"] = False                      # close it after each test

@pytest.fixture
def db_session(connection):                   # builds on connection
    session = {"connection": connection, "rows": []}
    yield session
    session["rows"].clear()                    # reset per test

def test_uses_the_whole_chain(db_session):
    # Requesting db_session pulls in connection, which pulls in engine —
    # pytest wires the entire dependency graph for you.
    assert db_session["connection"]["open"] is True
```

### Factory-as-fixture — when tests need many, or parameterised, objects

Sometimes a test needs **several** objects, or objects **customised per test** (three users with different roles, an order with N line items). A fixture that returns a single fixed object cannot do that. The idiom is a **factory fixture**: the fixture returns a *function*, and the test calls that function — possibly many times, with arguments — to mint objects on demand. Combine it with the `yield` pattern to track and clean up everything the factory produced.

```python
import pytest

@pytest.fixture
def make_user():
    """Return a FACTORY function. Tests call it to build customised users,
    and we track every one so teardown can clean them all up."""
    created = []

    def _make(name="anon", role="user"):        # the factory the test calls
        user = {"id": len(created) + 1, "name": name, "role": role}
        created.append(user)
        return user

    yield _make                                  # hand the factory to the test

    for u in created:                            # teardown: clean up ALL of them
        u.clear()

def test_multiple_customised_users(make_user):
    admin = make_user(name="Ada", role="admin")   # call the factory as needed
    guest = make_user(name="Bo",  role="guest")
    assert admin["role"] == "admin"
    assert guest["id"] == 2                         # factory tracked both
```

### autouse — fixtures that apply without being requested

Occasionally you want a fixture to run for **every** test *without* each test naming it — resetting a global cache, seeding the random module deterministically, silencing a logger, ensuring the timezone is UTC. Setting `autouse=True` makes a fixture run automatically for every test in its scope. Use it *sparingly* — invisible setup is harder to reason about — but for cross-cutting determinism it is exactly right.

```python
import pytest, random

@pytest.fixture(autouse=True)
def deterministic_random():
    """Runs before EVERY test (no test needs to request it). Seeds the RNG so
    any code using `random` is reproducible — killing a source of flakiness."""
    random.seed(0)
    yield
    # (nothing to tear down here)

def test_random_is_seeded():
    # Because the autouse fixture seeded to 0, this value is deterministic.
    assert random.randint(1, 100) == random.Random(0).randint(1, 100)
```

### The essential built-in fixtures

pytest ships with several fixtures you never define but constantly use. Requesting them by name is enough. These four are the workhorses:

- **`tmp_path`** — a unique, empty **`pathlib.Path`** directory for this test, on the real filesystem, auto-cleaned by pytest (it keeps the last few runs for debugging). This is the correct way to test code that reads/writes files: never touch the real project tree or `/tmp` by hand. (Covered more in §16.)
- **`capsys`** — captures anything the code writes to **stdout/stderr**, so you can assert on printed output. `capsys.readouterr()` returns an object with `.out` and `.err`. (Use `capfd` to also capture output from C-level/subprocess file descriptors.)
- **`monkeypatch`** — safely **modifies** attributes, dict items, and environment variables *for the duration of one test*, then **automatically undoes** every change in teardown. The safe way to override `os.environ`, patch a function, or change the working directory. (Central to §6 and §16.)
- **`caplog`** — captures **log records** emitted via the `logging` module so you can assert that your code logged the right thing at the right level.

```python
import os, logging, pytest

def greet(name):
    print(f"Hello, {name}!")                  # writes to stdout
    logging.getLogger("app").warning("greeted %s", name)
    return name.upper()

def write_report(path, text):
    path.write_text(text, encoding="utf-8")   # writes a file

def read_api_key():
    return os.environ["API_KEY"]              # reads an env var

# --- capsys: assert on printed output --------------------------------------
def test_greet_prints(capsys):
    greet("Ada")
    captured = capsys.readouterr()            # grab captured stdout/stderr
    assert captured.out == "Hello, Ada!\n"

# --- tmp_path: assert on filesystem effects --------------------------------
def test_write_report_creates_file(tmp_path):
    target = tmp_path / "report.txt"          # a real path in an isolated dir
    write_report(target, "quarterly numbers")
    assert target.read_text(encoding="utf-8") == "quarterly numbers"

# --- monkeypatch: override env vars, auto-restored -------------------------
def test_read_api_key(monkeypatch):
    monkeypatch.setenv("API_KEY", "test-secret-123")   # undone after the test
    assert read_api_key() == "test-secret-123"

# --- caplog: assert on log records -----------------------------------------
def test_greet_logs_warning(caplog):
    with caplog.at_level(logging.WARNING):    # capture WARNING+ from all loggers
        greet("Bo")
    assert "greeted Bo" in caplog.text
    assert caplog.records[0].levelname == "WARNING"
```

### Inspecting and debugging fixtures

Two commands demystify what fixtures exist and how they resolve, which is invaluable in a large codebase where fixtures come from several `conftest.py` layers:

```bash
pytest --fixtures            # list every available fixture + its docstring
pytest --fixtures-per-test   # show which fixtures each test will actually use
pytest --setup-show tests/test_users.py   # trace fixture setup/teardown order
```

### Fixtures vs setUp — the summary

Everything `setUp`/`tearDown` did, fixtures do better: they are opt-in per test (no wasted work), composable (build complex scaffolding from simple parts), shareable across the whole suite (`conftest.py`), scoped for speed (session/module/class/function), self-cleaning (`yield`), and parameterisable (§5 will *parametrize* fixtures too). This is why the guideline in §19 is blunt: **prefer fixtures over `setUp`.** The one time you still see `setUp` is inside `unittest.TestCase` subclasses (including Django's), which pytest runs faithfully — but for new pytest-native tests, reach for fixtures every time.

---

## 5. Parametrization, Markers and Conditional Tests

### Why parametrize — one test, many inputs

You will constantly want to run **the same test logic against many different inputs**: `is_even` for 0, 2, 3, -4, and a huge number; a validator for a dozen malformed strings; a pricing function across tiers. The naïve approach — copy-pasting the test and changing the numbers — produces a wall of near-identical functions that is tedious to write and, worse, tends to stop at the first failure so you never learn which *other* inputs also break. **Parametrization** solves this: you write the test body **once** and give pytest a **list of input/expected pairs**, and pytest generates a **separate test case for each** — each with its own name, each reported independently, all of them run even if some fail. This is the single biggest multiplier on test coverage-per-line-of-code you have.

```python
import pytest

def is_even(n):
    return n % 2 == 0

# One test body, five generated test cases. Each tuple is (input, expected).
@pytest.mark.parametrize("number, expected", [
    (0,  True),
    (2,  True),
    (3,  False),
    (-4, True),
    (1_000_001, False),
])
def test_is_even(number, expected):
    assert is_even(number) == expected
# Reported as 5 independent tests:
#   test_is_even[0-True]  test_is_even[2-True]  test_is_even[3-False] ...
# If (3, False) fails, the OTHER four still run and report — you see the
# full picture, not just the first broken case.
```

### Readable case names with ids

By default pytest builds each generated test's name from the parameter values (`test_is_even[3-False]`). For simple scalars that is fine, but for complex inputs (objects, long strings) the auto-generated id is ugly and unstable. Supply your own **`ids`** — a list of short labels, one per case — so the test report reads like a specification and you can target a single case with `-k`.

```python
import pytest

@pytest.mark.parametrize(
    "raw, expected",
    [
        ("2026-01-15", True),
        ("15/01/2026", False),
        ("",           False),
        ("not-a-date", False),
    ],
    ids=["iso-format", "slashes", "empty-string", "garbage"],   # human-readable
)
def test_is_iso_date(raw, expected):
    def is_iso_date(s):
        import re
        return bool(re.fullmatch(r"\d{4}-\d{2}-\d{2}", s))
    assert is_iso_date(raw) == expected
# Now: test_is_iso_date[iso-format], test_is_iso_date[garbage], ...
# Run just one:  pytest -k "garbage"
```

### Stacking parametrize — the cartesian product

Applying **two** `@pytest.mark.parametrize` decorators to one test produces the **cross product** of their cases — every combination of the two lists. This is perfect for testing a function across two independent dimensions (every user role against every action, say). Be mindful that it multiplies: 4 × 5 stacked parametrizations is 20 tests, which is powerful but can explode if you stack three or four large lists.

```python
import pytest

@pytest.mark.parametrize("x", [1, 2, 3])
@pytest.mark.parametrize("y", [10, 20])
def test_sum_combinations(x, y):
    # Runs 3 × 2 = 6 times: (1,10) (1,20) (2,10) (2,20) (3,10) (3,20)
    assert x + y == y + x
```

### Indirect parametrization — feeding a fixture

Sometimes the parameter is not the raw input but a **recipe for building** one — you want to parametrize the *fixture* rather than the test. With `indirect=True`, pytest routes each parameter value **into the named fixture** (via the special `request` object) instead of straight into the test, and the test receives whatever the fixture *produced*. This is how you run the same test against, say, three differently-configured database backends or three pre-seeded object states.

```python
import pytest

@pytest.fixture
def user(request):
    # request.param holds the current parametrize value for this case.
    role = request.param
    return {"name": "test", "role": role, "is_admin": role == "admin"}

@pytest.mark.parametrize("user", ["admin", "editor", "guest"], indirect=True)
def test_admin_flag(user):
    # `user` is the DICT the fixture built, not the raw string.
    assert user["is_admin"] == (user["role"] == "admin")
```

You can also **parametrize a fixture directly** at definition time with `params=`, which reruns *every* test that uses the fixture once per param — ideal for "run the whole suite against each backend":

```python
import pytest

@pytest.fixture(params=["sqlite", "postgres", "mysql"])
def database_url(request):
    # Every test using `database_url` runs 3 times, once per backend.
    return {"sqlite": "sqlite:///:memory:",
            "postgres": "postgresql://localhost/test",
            "mysql": "mysql://localhost/test"}[request.param]

def test_connection_string_has_scheme(database_url):
    assert "://" in database_url
```

### Markers — labelling and selecting tests

A **marker** is a **label** you attach to a test with `@pytest.mark.<name>`, used to **categorise** tests so you can select or deselect groups of them at run time. The classic use is separating **fast** from **slow** or **unit** from **integration**, so that your inner-loop `pytest -m "not slow"` skips the expensive ones while CI runs everything. Register your custom markers in config (§3) and use `--strict-markers` so a typo is an error, not a silently-ignored no-op.

```python
import pytest

@pytest.mark.slow
def test_processes_large_file():
    ...                                # only runs when you ask for it

@pytest.mark.integration
def test_talks_to_real_database():
    ...
```

```bash
pytest -m slow                 # only slow tests
pytest -m "not slow"           # everything except slow (fast inner loop)
pytest -m "integration and not slow"   # boolean combinations
```

### Built-in markers — skip, skipif, and xfail

pytest ships three markers for tests that should not simply pass or fail, each expressing a different *reason*:

- **`@pytest.mark.skip(reason=...)`** — never run this test (temporarily broken, not applicable). It shows as skipped, not passed or failed.
- **`@pytest.mark.skipif(condition, reason=...)`** — skip only when a condition holds: a platform, a Python version, a missing optional dependency. This keeps a cross-platform suite green everywhere.
- **`@pytest.mark.xfail(reason=...)`** — "expected to fail": you *know* this is currently broken (an unfixed bug, an unimplemented feature). If it fails, that is reported as **xfail** (expected, not a red build); if it unexpectedly *passes*, that is reported as **xpass**, alerting you the bug is fixed and the marker can go. This is how you commit a failing test for a known bug without breaking CI.

```python
import sys
import pytest

@pytest.mark.skip(reason="rewriting this module next sprint")
def test_temporarily_disabled():
    ...

@pytest.mark.skipif(sys.platform == "win32", reason="uses POSIX signals")
def test_posix_signal_handling():
    ...

@pytest.mark.skipif(sys.version_info < (3, 14), reason="needs 3.14 syntax")
def test_new_language_feature():
    ...

@pytest.mark.xfail(reason="bug #4123: rounding is off by a cent", strict=True)
def test_known_rounding_bug():
    assert round(2.675, 2) == 2.68     # actually 2.67 due to float repr → xfail

def test_conditional_xfail():
    # You can also mark xfail at runtime, and even inside parametrize:
    ...

# xfail can decorate individual parametrize cases via pytest.param:
@pytest.mark.parametrize("value, expected", [
    (2, 4),
    (3, 9),
    pytest.param(5, 26, marks=pytest.mark.xfail(reason="typo in expected")),
])
def test_squares(value, expected):
    assert value * value == expected
```

⚡ **Version note:** Prefer **`strict=True`** on `xfail` (it is the default when you set `xfail_strict = true` in config). Strict mode turns an unexpected **pass** into a *failure*, forcing you to remove the stale marker — otherwise fixed bugs quietly keep their xfail label forever. `skipif` conditions are evaluated at collection time, so keep them cheap and side-effect-free.

### Combining it all — parametrize a marked test

These mechanisms compose freely. A realistic validator test parametrizes many inputs, gives them readable ids, marks a couple of known-bad cases xfail, and lives under an `integration` marker if it needs I/O — all at once. The result is a compact, self-documenting block that would have been dozens of copy-pasted functions in `unittest`.

---

## 6. Mocking and Test Doubles

### The problem — collaborators you cannot or should not call for real

A unit test is supposed to exercise **one** unit in **isolation**. But real code has *collaborators*: it calls a payment gateway over HTTP, reads the system clock, sends an email, queries a database, spawns a subprocess. If your test of `checkout()` really charged a credit card, it would be slow, non-deterministic, dependent on the network, and — literally — expensive. You need a way to **replace a real collaborator with a controllable stand-in** so the unit under test runs in a sealed lab where you dictate exactly what its dependencies do and observe exactly how it calls them. Those stand-ins are called **test doubles**, and creating them is **mocking**.

The vocabulary (from Gerard Meszaros's *xUnit Test Patterns*) is worth pinning down because people use "mock" loosely for all of it:

- **Dummy** — a placeholder passed only to fill a parameter list; never actually used.
- **Stub** — returns **canned answers** to calls (`get_user()` always returns a fixed user). Used to *control input* to the unit under test.
- **Fake** — a **working but simplified** implementation: an in-memory dict standing in for a database, a fake filesystem. Behaves realistically but is not production-grade.
- **Spy** — a real (or stub) object that *also* **records how it was called**, so you can assert on the interactions afterward.
- **Mock** — a pre-programmed object with **expectations**: you set up what calls it should receive and assert those calls happened. Used to *verify interaction* (that `send_email` was called once with the right address).

Python's standard library gives you all of this through **`unittest.mock`**, and it is available in pytest too — you do not need a separate library. The two workhorses are the `Mock`/`MagicMock` classes (objects that fake *anything*) and the `patch` tool (temporarily *replacing* a real name with a mock).

### Mock and MagicMock — objects that fake anything

A **`Mock`** is a shape-shifting object: **accessing any attribute** returns another auto-created `Mock`, and **calling it** returns a `Mock` and records the call. This means a single `Mock` can impersonate almost any object without you defining anything. **`MagicMock`** is the same but *also* implements the "magic" dunder methods (`__len__`, `__iter__`, `__getitem__`, `__enter__`…), so it can stand in for objects used with `len()`, iteration, `[]`, or `with`. Use `MagicMock` by default; it is a strict superset.

You control a mock's behaviour with two attributes and inspect its usage with a family of assertions:

- **`return_value`** — what the mock returns when *called*.
- **`side_effect`** — a richer alternative: set it to an **exception** (raised when called), a **function** (called with the same args, its result returned — lets a mock compute a response), or an **iterable** (successive calls return successive items — great for simulating a sequence of responses or a retry).
- **`.assert_called_once()` / `.assert_called_with(...)` / `.assert_called_once_with(...)` / `.assert_not_called()`** — verify the mock was (or was not) invoked, and with what arguments.
- **`.call_count`, `.call_args`, `.call_args_list`** — inspect the calls directly for finer assertions.

```python
from unittest.mock import Mock, MagicMock, call

# ---- return_value: canned answer -----------------------------------------
gateway = Mock()
gateway.charge.return_value = {"status": "ok", "id": "ch_123"}

result = gateway.charge(amount=1000, currency="usd")
assert result == {"status": "ok", "id": "ch_123"}

# ---- verify the interaction happened as expected -------------------------
gateway.charge.assert_called_once()
gateway.charge.assert_called_once_with(amount=1000, currency="usd")
assert gateway.charge.call_count == 1
assert gateway.charge.call_args == call(amount=1000, currency="usd")

# ---- side_effect as an EXCEPTION: simulate a failure ---------------------
flaky = Mock()
flaky.get.side_effect = ConnectionError("network down")
try:
    flaky.get()
except ConnectionError as e:
    assert str(e) == "network down"

# ---- side_effect as an ITERABLE: successive calls, successive results ----
clock = Mock()
clock.now.side_effect = [1000, 1001, 1002]     # 1st call →1000, 2nd →1001 ...
assert [clock.now(), clock.now(), clock.now()] == [1000, 1001, 1002]

# ---- side_effect as a FUNCTION: compute the response ----------------------
calc = Mock()
calc.double.side_effect = lambda n: n * 2
assert calc.double(21) == 42

# ---- MagicMock supports dunders (len, iteration, context managers) --------
container = MagicMock()
container.__len__.return_value = 3
assert len(container) == 3
```

### patch — replacing a real name with a mock

Creating a `Mock` is only half the job; you must get it *into* the code under test in place of the real collaborator. When the code constructs or imports its dependency internally (rather than receiving it as an argument), you use **`unittest.mock.patch`** to **temporarily replace a name in a module's namespace** with a mock for the duration of a test, restoring the original automatically afterward. `patch` works as a decorator, a context manager, or (via the `pytest-mock` plugin's `mocker` fixture) a fixture.

**The single most important rule of patching — and the most common mistake — is "patch where it is *looked up*, not where it is *defined*."** When module `app` does `from services import send_email`, it creates a *new name* `app.send_email` bound to that function. Patching `services.send_email` does nothing to `app`'s copy; you must patch `app.send_email` — the name at the *point of use*. Get this wrong and your "mock" silently does nothing while the real function runs.

```python
# --- production code -------------------------------------------------------
# file: notifications.py
import smtplib
def send_email(to, body):
    with smtplib.SMTP("smtp.example.com") as server:
        server.sendmail("me@example.com", to, body)
    return True

# file: signup.py
from notifications import send_email        # <-- creates name signup.send_email

def register(email):
    # ... create the user ...
    send_email(email, "Welcome!")           # looked up as signup.send_email
    return {"email": email, "notified": True}
```

```python
# --- the test --------------------------------------------------------------
from unittest.mock import patch
import signup

# CORRECT: patch the name where signup LOOKS IT UP, not where it's defined.
@patch("signup.send_email")                 # not "notifications.send_email"!
def test_register_sends_welcome(mock_send):
    # The decorator injects the created mock as an argument.
    mock_send.return_value = True

    result = signup.register("ada@example.com")

    assert result == {"email": "ada@example.com", "notified": True}
    mock_send.assert_called_once_with("ada@example.com", "Welcome!")
    # The real send_email (and real SMTP) was NEVER called — fast & isolated.

# As a context manager, when you want to scope the patch to a few lines:
def test_register_with_context_manager():
    with patch("signup.send_email") as mock_send:
        signup.register("bo@example.com")
        mock_send.assert_called_once()
```

### patch.object and patching attributes

When you want to replace a **method or attribute of a specific object or class** rather than a module-level name, **`patch.object`** is cleaner and safer than a string path: you pass the object and the attribute *name*. This is common for stubbing one method of a class while leaving the rest real.

```python
from unittest.mock import patch

class WeatherClient:
    def fetch(self, city):
        # imagine a real HTTP call here
        raise RuntimeError("real network call")

def report(client, city):
    data = client.fetch(city)
    return f"{city}: {data['temp']}°C"

def test_report_stubs_fetch():
    client = WeatherClient()
    # Replace just the .fetch method on THIS class for the duration.
    with patch.object(WeatherClient, "fetch", return_value={"temp": 21}):
        assert report(client, "Oslo") == "Oslo: 21°C"

# patch.object also patches a value on an instance/module:
import math
def test_patch_a_constant():
    with patch.object(math, "pi", 3.0):     # silly, but demonstrates the API
        assert math.pi == 3.0
```

### autospec — mocks that match the real signature

A plain `Mock` accepts *any* call — `mock.charge()`, `mock.charge(1, 2, 3, nonsense=True)`, `mock.chrage()` (typo!) all "work" and return a mock. That is dangerous: your test can pass while calling the real function *wrong*, and a typo'd method name is never caught. **`autospec`** (via `patch(..., autospec=True)` or `create_autospec`) builds a mock that **mirrors the real object's API**: it enforces the actual method names and call signatures, raising `TypeError`/`AttributeError` if you call something that does not exist or with the wrong arguments. **Prefer `autospec=True` for anything non-trivial** — it makes mocks fail when the real interface changes, which is the entire point of a test.

```python
from unittest.mock import patch, create_autospec

class PaymentGateway:
    def charge(self, amount, currency="usd"):
        raise RuntimeError("real API")

def checkout(gateway, cents):
    return gateway.charge(cents, currency="eur")

def test_checkout_with_autospec():
    # autospec mock knows charge's real signature.
    with patch("__main__.PaymentGateway", autospec=True) as MockGateway:
        instance = MockGateway.return_value
        instance.charge.return_value = {"status": "ok"}

        result = checkout(instance, 500)

        assert result == {"status": "ok"}
        instance.charge.assert_called_once_with(500, currency="eur")
        # instance.chrage(...) or instance.charge(wrong, args) would now RAISE,
        # catching signature drift — a plain Mock would silently accept them.

def test_create_autospec_standalone():
    fake = create_autospec(PaymentGateway, instance=True)
    fake.charge.return_value = {"status": "ok"}
    assert fake.charge(100) == {"status": "ok"}
```

### The monkeypatch fixture — pytest's lighter-weight patcher

For simple replacements pytest's built-in **`monkeypatch`** fixture (met in §4) is often more convenient than `unittest.mock.patch`, because it needs no decorator stack and **auto-restores** every change at test end. Its methods cover the common cases: `setattr`/`delattr` (replace an attribute), `setitem`/`delitem` (a dict entry), `setenv`/`delenv` (an environment variable), and `chdir` (working directory). Use `monkeypatch` for setting values and simple stubs; reach for `mock.patch`/`autospec` when you need call-recording assertions or signature enforcement.

```python
import os

def get_config():
    return {"debug": os.environ.get("DEBUG") == "1",
            "region": os.environ.get("REGION", "us")}

def test_config_from_env(monkeypatch):
    monkeypatch.setenv("DEBUG", "1")            # restored automatically
    monkeypatch.setenv("REGION", "eu")
    assert get_config() == {"debug": True, "region": "eu"}

def test_stub_a_function(monkeypatch):
    import random
    # Replace random.random with a deterministic stub for this test only.
    monkeypatch.setattr(random, "random", lambda: 0.42)
    assert random.random() == 0.42
```

### Freezing time with freezegun

Code that reads the **current time** — `datetime.now()`, `date.today()`, `time.time()` — is non-deterministic by nature: the value changes every run, so any assertion about "created_at is now" is impossible to pin down, and time-dependent logic (a token that expires in 1 hour, a "good morning" greeting) cannot be tested reliably. The clean fix is to **freeze time to a fixed instant** for the test. The **freezegun** library (`pip install freezegun`) does exactly this: `@freeze_time("2026-07-17")` makes *every* time function in your code return that moment, so time-dependent behaviour becomes fully deterministic. You can even advance the frozen clock to test elapsed-time logic.

```python
from datetime import datetime, timedelta
from freezegun import freeze_time

def make_token():
    return {"created": datetime.now(), "expires": datetime.now() + timedelta(hours=1)}

def is_expired(token, now=None):
    return (now or datetime.now()) >= token["expires"]

# As a decorator: the whole test sees a frozen clock.
@freeze_time("2026-07-17 12:00:00")
def test_token_creation_time():
    token = make_token()
    assert token["created"] == datetime(2026, 7, 17, 12, 0, 0)
    assert token["expires"] == datetime(2026, 7, 17, 13, 0, 0)

# As a context manager, with the ability to advance time:
def test_token_expiry():
    with freeze_time("2026-07-17 12:00:00") as frozen:
        token = make_token()
        assert not is_expired(token)          # right now: not expired
        frozen.tick(delta=timedelta(hours=2)) # jump 2 hours into the future
        assert is_expired(token)              # now it IS expired
```

⚡ **Version note:** The design-level alternative to freezing is **dependency injection of the clock** — pass a `now` callable (defaulting to `datetime.now`) into functions that need the time, then in tests pass a fixed value. This is often cleaner than freezegun because it makes the time dependency *explicit* in the signature and needs no library. freezegun shines when you cannot change the code (third-party libraries) or when time is read deep in a call stack. Python 3.13/3.14 code should use timezone-aware `datetime.now(tz=...)`; freezegun supports `tz_offset`.

### Faking HTTP with responses and respx

Code that makes **outbound HTTP calls** must not hit the real network in a unit test — it would be slow, flaky, and dependent on a third party. Instead you **intercept the HTTP layer** and return canned responses. Which tool depends on the client library: **`responses`** (`pip install responses`) intercepts the popular **`requests`** library; **`respx`** (`pip install respx`) intercepts **`httpx`** (sync and async), which is what FastAPI's TestClient and modern async code use. Both let you register "when a request matches this URL/method, return this status and body," then assert which requests were actually made.

```python
# ---- Faking `requests` calls with `responses` ----------------------------
import requests
import responses

def get_user_name(user_id):
    r = requests.get(f"https://api.example.com/users/{user_id}")
    r.raise_for_status()
    return r.json()["name"]

@responses.activate                       # activates interception for this test
def test_get_user_name():
    responses.add(                        # register a canned response
        responses.GET,
        "https://api.example.com/users/1",
        json={"id": 1, "name": "Ada"},
        status=200,
    )
    assert get_user_name(1) == "Ada"
    assert len(responses.calls) == 1      # assert the real call was intercepted
    assert responses.calls[0].request.url.endswith("/users/1")

@responses.activate
def test_get_user_name_404():
    responses.add(responses.GET, "https://api.example.com/users/999", status=404)
    import pytest
    with pytest.raises(requests.HTTPError):
        get_user_name(999)
```

```python
# ---- Faking `httpx` calls with `respx` (works for async too) -------------
import httpx
import respx

async def fetch_price(symbol):
    async with httpx.AsyncClient() as client:
        r = await client.get(f"https://api.example.com/price/{symbol}")
        return r.json()["price"]

@respx.mock
async def test_fetch_price():             # run under pytest-asyncio (§8)
    respx.get("https://api.example.com/price/ACME").mock(
        return_value=httpx.Response(200, json={"price": 42.5})
    )
    assert await fetch_price("ACME") == 42.5
```

### Fakes vs mocks — and the over-mocking warning

There is a spectrum. A **mock** verifies *interactions* ("was `save()` called with this row?"); a **fake** provides a *working simplified implementation* ("an in-memory dict that behaves like the repository"). Fakes usually produce **more robust** tests, because they assert on *observable behaviour* (the data ended up stored) rather than on the *mechanics* of how your code talked to a collaborator. Interaction assertions are brittle: they fail when you refactor the *implementation* even though the *behaviour* is unchanged — the test is coupled to internals.

```python
# A FAKE repository: a working in-memory implementation. Tests using it assert
# on RESULTS (was the user saved?) not on call mechanics — robust to refactors.
class FakeUserRepo:
    def __init__(self):
        self._store = {}
        self._next_id = 1
    def add(self, name):
        uid = self._next_id
        self._store[uid] = {"id": uid, "name": name}
        self._next_id += 1
        return uid
    def get(self, uid):
        return self._store.get(uid)

def register_user(repo, name):
    uid = repo.add(name)
    return repo.get(uid)

def test_register_with_a_fake():
    repo = FakeUserRepo()                     # no mocks, no patching
    user = register_user(repo, "Ada")
    assert user == {"id": 1, "name": "Ada"}   # assert on behaviour/state
```

**The over-mocking trap.** It is tempting to mock everything, but a test where *every* collaborator is a mock tests almost nothing real — it verifies that your code calls mocks in a certain order, which is a restatement of the implementation, not a check of behaviour. Such tests are **brittle** (break on every refactor) and give **false confidence** (they pass even when the pieces do not actually work together, because none of the real pieces ran). The guidance: mock at the **boundaries** of your system — the network, the clock, the payment provider, the email sender — and use **real objects or fakes** for your own internal collaborators. If you find yourself mocking your own domain classes constantly, that is a design smell suggesting the units are too coupled.

### Dependency injection — the design that makes mocking easy

Notice that `register_user(repo, name)` took its repository **as an argument** instead of constructing one inside. That is **dependency injection (DI)**: passing a collaborator in rather than hard-coding it. DI is the design pattern that makes testing pleasant, because swapping the real dependency for a fake/mock is trivial — you just pass a different argument, no `patch` needed. Code that reaches out and constructs its own dependencies (`self.db = Postgres()`) forces you into `patch` gymnastics; code that accepts them (`def __init__(self, db)`) lets tests hand it whatever they like. When something is hard to test, the fix is usually not a cleverer mock but a small refactor to inject the dependency. This is the practical link between "testable" and "well-designed" promised in §1, and it is why frameworks like FastAPI (§9) build DI in as a first-class feature.

---

## 7. Coverage — Measuring What You Test

### What coverage is and what it does and does not tell you

**Code coverage** measures **which lines (and branches) of your production code actually executed while the test suite ran.** If a line never runs during any test, coverage flags it as *uncovered* — a part of your program that no test exercises, and therefore could be completely broken without any test noticing. Coverage answers one specific, valuable question: **"what have I definitely *not* tested?"**

It is crucial to understand the limit of that claim. Coverage tells you a line *ran*; it does **not** tell you the line was *tested*. A test can execute a function without asserting anything meaningful about it — 100% coverage with weak assertions is a comfortable lie. Coverage is therefore a **floor, not a ceiling**: uncovered code is definitely under-tested, but covered code is not necessarily *well*-tested. Treat coverage as a tool for **finding blind spots** (which files, which branches, which error paths were never touched), not as a grade of test quality. (§17's mutation testing is the tool that actually measures assertion strength.)

### coverage.py and pytest-cov

The engine is **coverage.py** (`pip install coverage`); the convenient way to drive it from pytest is the **pytest-cov** plugin (`pip install pytest-cov`), which adds `--cov` options to the `pytest` command. Under the hood pytest-cov just runs coverage.py around your suite, but it integrates the reporting into the pytest run.

```bash
pip install pytest-cov

# Measure coverage of the `myapp` package while running the suite.
pytest --cov=myapp

# Show which specific LINES are missing in each file.
pytest --cov=myapp --cov-report=term-missing

# Generate a browsable HTML report (open htmlcov/index.html).
pytest --cov=myapp --cov-report=html

# Machine-readable XML (for CI tools like Codecov) + a terminal summary.
pytest --cov=myapp --cov-report=xml --cov-report=term
```

```text
# Example term-missing output — the "Missing" column is the gold:
Name                     Stmts   Miss Branch BrPart  Cover   Missing
--------------------------------------------------------------------
src/myapp/__init__.py        2      0      0      0   100%
src/myapp/calculator.py     18      2      6      1    88%   14, 27->29
--------------------------------------------------------------------
TOTAL                       20      2      6      1    89%
# "14" = line 14 never ran. "27->29" = the branch from line 27 to 29 (e.g. the
# `if` being false) never happened — you only ever tested the true path.
```

### Branch coverage — the important upgrade over line coverage

Plain **line coverage** only asks "did this line run?" That misses a whole class of gaps. Consider `if x: do_a()` on one line followed by more code — line coverage is satisfied once the line runs with `x` truthy, but the `x` *falsy* path (where `do_a()` is skipped) may never have been tested. **Branch coverage** fixes this by tracking, for every decision point, whether **both** outcomes occurred — the `if` taken *and* not taken, the loop entered *and* skipped. This catches the classic bug of testing only the happy path and never the "condition is false" path. **Always enable branch coverage;** line-only coverage overstates how much you have really tested.

```toml
# pyproject.toml — coverage.py configuration
[tool.coverage.run]
branch = true                       # measure branch coverage, not just lines
source = ["src/myapp"]              # what to measure (your package, not tests)
omit = ["*/__main__.py", "*/migrations/*"]   # exclude generated/entrypoint code

[tool.coverage.report]
show_missing = true                 # print the Missing column
skip_covered = true                 # hide 100%-covered files to reduce noise
# Lines matching these regexes don't count as "missed" if never run:
exclude_lines = [
    "pragma: no cover",             # explicit opt-out marker (see below)
    "if __name__ == .__main__.:",   # the script entrypoint
    "raise NotImplementedError",    # abstract stubs
    "if TYPE_CHECKING:",            # type-only imports never run at runtime
    "\\.\\.\\.",                    # `...` placeholder bodies
]

[tool.coverage.paths]
# Map coverage from different machines/venvs onto one path (for combining CI runs)
source = ["src/", "*/site-packages/"]
```

### Enforcing a threshold with --cov-fail-under

Measuring coverage is only useful if it does not silently decay. **`--cov-fail-under=N`** makes the test command **exit with a failure** if total coverage drops below `N` percent, which — wired into CI (§18) — turns coverage into a **ratchet**: a pull request that adds untested code and pushes coverage below the line fails the build and cannot merge. Pick a threshold you can *hold* rather than an aspirational one; a stable 85% that never regresses is far more valuable than a 95% target everyone routinely overrides.

```bash
# Fail the build if combined coverage is under 85%.
pytest --cov=myapp --cov-branch --cov-report=term-missing --cov-fail-under=85
```

```toml
# You can also bake the threshold into config so every run enforces it:
[tool.coverage.report]
fail_under = 85
precision = 1          # show one decimal place in the percentage
```

⚡ **Version note:** coverage.py 7.x can measure the code run by **subprocesses** and **multiple parallel workers** (essential when you use `pytest-xdist`, §18): set `parallel = true` under `[tool.coverage.run]`, let each worker write its own `.coverage.<host>.<pid>` file, then `coverage combine` them before reporting. pytest-cov handles most of this automatically. 7.x also added experimental support for measuring which *branches* of `match` statements executed.

### What to actually measure — being pragmatic

Chasing 100% is usually a poor use of time; the last few percent are often trivial getters, defensive `except` blocks for "impossible" errors, or `__repr__` methods, and forcing tests for them adds maintenance cost for near-zero bug-catching value. Sensible policy:

- **Measure your own code, not dependencies or tests.** Set `source` to your package; do not measure `site-packages` or the `tests/` directory itself.
- **Prioritise branch coverage of business logic** — the conditionals in your pricing, auth, and validation code are where real bugs hide. A missed branch in `if user.is_admin:` matters far more than an unhit `__repr__`.
- **Use `# pragma: no cover` deliberately**, not habitually, to exclude genuinely untestable lines (a `raise` for an impossible enum case, a platform-specific block you cannot run in CI). Every pragma is a documented decision that "this is intentionally not covered."
- **Watch the trend, not the absolute.** A ratchet that forbids *decreases* keeps the suite honest without demanding perfection.
- **Remember coverage measures execution, not correctness.** Pair it with strong assertions and, for critical code, mutation testing (§17) to check the tests would actually *catch* a bug.

```python
def parse_mode(mode: str) -> int:
    if mode == "r":
        return 0
    elif mode == "w":
        return 1
    else:   # pragma: no cover  — validated upstream; unreachable in practice
        raise ValueError(f"unknown mode {mode!r}")
```

---

## 8. Async Testing

### Why async needs special handling

Modern Python does a lot of I/O with **`async`/`await`** — `async def` functions (coroutines) that suspend on I/O and are driven by an **event loop** (see the asyncio material in [Python](PYTHON_GUIDE.md) and the async chapter of [FastAPI](FASTAPI_GUIDE.md)). Testing them has one wrinkle that trips up every beginner: **calling a coroutine does not run it.** `result = fetch_user(1)` where `fetch_user` is `async def` does *not* return a user — it returns a *coroutine object* that has not executed a single line. Coroutines only run when **awaited inside a running event loop**. But a pytest test function is an ordinary synchronous function; there is no event loop and you cannot `await` at its top level. So a naïve async test silently does nothing:

```python
async def fetch_user(uid):
    return {"id": uid, "name": "Ada"}

# WRONG — this "passes" but tests NOTHING. The coroutine is never awaited;
# the assertion compares a coroutine object to a dict, which... pytest actually
# warns about this now, but the point stands: the body never ran.
def test_fetch_user_broken():
    result = fetch_user(1)                # a coroutine, not a dict!
    # assert result == {...}             # would fail: coroutine != dict
```

You need a plugin that (a) lets a test itself be `async def`, and (b) spins up an event loop, runs the coroutine to completion, and reports the result. The two options are **pytest-asyncio** and **anyio**.

### pytest-asyncio

**pytest-asyncio** (`pip install pytest-asyncio`) is the most common choice. You mark async tests so the plugin runs them on an event loop. Set **`asyncio_mode = "auto"`** in config and *every* `async def test_*` is collected and run automatically — no per-test marker needed, which is the least-friction setup. (In the stricter default `"strict"` mode you decorate each with `@pytest.mark.asyncio`.)

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"          # every `async def test_*` just works
# (in "strict" mode you'd instead decorate each test with @pytest.mark.asyncio)
```

```python
import asyncio
import pytest

async def fetch_user(uid):
    await asyncio.sleep(0)               # simulate awaiting I/O
    return {"id": uid, "name": "Ada"}

# In auto mode, an async test runs on a real event loop — you can await.
async def test_fetch_user():
    result = await fetch_user(1)         # actually runs the coroutine
    assert result == {"id": 1, "name": "Ada"}

# Strict-mode style (explicit marker) — always works regardless of mode:
@pytest.mark.asyncio
async def test_fetch_user_explicit():
    assert (await fetch_user(2))["id"] == 2

# Testing that an async function raises:
async def divide(a, b):
    if b == 0:
        raise ZeroDivisionError
    return a / b

async def test_async_raises():
    with pytest.raises(ZeroDivisionError):
        await divide(1, 0)               # await INSIDE the with block
```

### Async fixtures

Fixtures can be `async def` too — necessary when setup itself does async I/O (opening an async database connection, starting an async client). pytest-asyncio runs them on the same event loop as the test. The `yield` teardown pattern works identically, just with `await`able setup/teardown.

```python
import pytest

class AsyncClient:
    async def connect(self): self.open = True
    async def close(self):  self.open = False
    async def get(self, path): return {"path": path, "ok": True}

@pytest.fixture
async def client():
    c = AsyncClient()
    await c.connect()          # async setup
    yield c                    # test runs here
    await c.close()            # async teardown

async def test_client_get(client):
    assert (await client.get("/health"))["ok"] is True
```

⚡ **Version note:** pytest-asyncio **0.25+** stabilised configurable **loop scopes** — `@pytest.mark.asyncio(loop_scope="session")` (and the `asyncio_default_fixture_loop_scope` setting) let a group of tests share one event loop, which matters when a session-scoped async resource (a connection pool) must live on the *same* loop as the tests that use it. Mismatched loop scopes are the cause of the infamous `RuntimeError: <X> is bound to a different event loop`.

### anyio — testing across asyncio and trio

**anyio** (`pip install anyio`, plus its pytest plugin) is the alternative when your code is written against anyio's abstraction so it can run on **either** asyncio **or** trio. Its `anyio_backend` fixture parametrizes tests to run on *each* backend, verifying your code is backend-agnostic. If you use plain asyncio, pytest-asyncio is simpler; if you use anyio (as newer Starlette/httpx internals do), the anyio plugin is the natural fit. FastAPI's async tests commonly run under anyio.

```python
import anyio
import pytest

# Restrict to asyncio, or list both to run twice (once per backend).
@pytest.fixture
def anyio_backend():
    return "asyncio"

@pytest.mark.anyio
async def test_with_anyio():
    async with anyio.create_task_group() as tg:
        results = []
        async def work(n): results.append(n * 2)
        for i in range(3):
            tg.start_soon(work, i)
    assert sorted(results) == [0, 2, 4]
```

### Event-loop pitfalls to avoid

- **Never `time.sleep()` in async code or tests** — it *blocks the whole event loop*, freezing every other task. Use `await asyncio.sleep(...)`. Better still, do not sleep to "wait for" something; await the actual thing.
- **One loop per test by default.** Each function-scoped async test gets a fresh loop, which is what you want for isolation. Problems arise only when a *wider-scoped* async fixture (session pool) lives on a different loop than a function-scoped test — match their loop scopes (version note above).
- **Do not create your own loop with `asyncio.run()` inside a test** that the plugin is already driving — you will get "loop already running" or nested-loop errors. Let the plugin own the loop; just `await`.
- **Async context managers and generators** need async teardown; use `async with`/`async for` and `yield` in async fixtures, not the sync equivalents.
- **Testing concurrency for real** (that two coroutines actually overlap) is subtle; often it is enough to test each coroutine's logic in isolation and trust the event loop. When you must test overlap, use `asyncio.gather` and assert on ordering/timing with a virtual clock rather than real sleeps.

---

## 9. HTTP and API Integration Testing

### What API integration testing is and why it beats unit-testing handlers

A web handler is more than its function body: a request must be **routed** to it, the body **parsed and validated**, dependencies **injected**, the response **serialised**, and errors **mapped to status codes**. Unit-testing the handler function in isolation skips all of that machinery — and that machinery is exactly where a lot of real bugs live (wrong status code, a validation rule that does not fire, a serializer that leaks a password field, a broken auth dependency). **API integration testing** exercises the endpoint *through the framework's real stack* by sending it real HTTP-shaped requests and asserting on real HTTP responses (status, headers, JSON body). Crucially, both FastAPI/Starlette and Django provide a **test client** that does this **in-process** — no real server, no real socket, no network — so these tests are fast and deterministic while still covering the whole request pipeline.

### FastAPI and Starlette TestClient

FastAPI/Starlette give you a **`TestClient`** built on **httpx**. You wrap your `app` in it and call `.get()`, `.post()`, etc., with a `requests`/`httpx`-style API; it drives the ASGI app directly and returns a response object you assert on. Because it is httpx-based, it speaks JSON, form data, files, headers, and cookies naturally.

```python
# --- the app under test ----------------------------------------------------
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()
_DB: dict[int, dict] = {}

class ItemIn(BaseModel):
    name: str
    price: float

@app.post("/items", status_code=201)
def create_item(item: ItemIn):
    item_id = len(_DB) + 1
    _DB[item_id] = {"id": item_id, **item.model_dump()}
    return _DB[item_id]

@app.get("/items/{item_id}")
def get_item(item_id: int):
    if item_id not in _DB:
        raise HTTPException(status_code=404, detail="not found")
    return _DB[item_id]
```

```python
# --- the tests -------------------------------------------------------------
import pytest
from fastapi.testclient import TestClient
from myapp.main import app, _DB

@pytest.fixture
def client():
    _DB.clear()                      # isolate: reset module state each test
    return TestClient(app)           # in-process ASGI client, no network

def test_create_item_returns_201_and_body(client):
    resp = client.post("/items", json={"name": "Widget", "price": 9.99})
    assert resp.status_code == 201
    body = resp.json()
    assert body == {"id": 1, "name": "Widget", "price": 9.99}

def test_get_missing_item_returns_404(client):
    resp = client.get("/items/999")
    assert resp.status_code == 404
    assert resp.json() == {"detail": "not found"}

def test_validation_error_returns_422(client):
    # price is not a number → FastAPI/Pydantic returns 422 automatically.
    resp = client.post("/items", json={"name": "X", "price": "free"})
    assert resp.status_code == 422
    assert resp.json()["detail"][0]["loc"] == ["body", "price"]
```

### Dependency overrides — the FastAPI testing superpower

FastAPI's dependency-injection system (see [FastAPI](FASTAPI_GUIDE.md) §7) has a killer testing feature: **`app.dependency_overrides`** lets you **swap any dependency for a test double** at the app level, no patching required. Where production injects a real database session or a real "current user," your test injects a fake or a fixed user. This is DI (§6) paying off: because the handler *asks for* its dependencies, you *give* it test ones. Always reset the overrides in teardown so tests stay isolated.

```python
from fastapi import FastAPI, Depends
from fastapi.testclient import TestClient
import pytest

app = FastAPI()

def get_db():                          # real dependency (would open a DB session)
    raise RuntimeError("real DB")

def get_current_user():                # real auth dependency
    raise RuntimeError("real auth")

@app.get("/me/orders")
def my_orders(user=Depends(get_current_user), db=Depends(get_db)):
    return {"user": user["name"], "orders": db["orders"]}

@pytest.fixture
def client():
    # Override the real dependencies with fast, deterministic fakes.
    app.dependency_overrides[get_current_user] = lambda: {"name": "Ada"}
    app.dependency_overrides[get_db] = lambda: {"orders": [1, 2, 3]}
    yield TestClient(app)
    app.dependency_overrides.clear()   # CRUCIAL: reset so tests stay isolated

def test_my_orders_uses_overrides(client):
    resp = client.get("/me/orders")
    assert resp.status_code == 200
    assert resp.json() == {"user": "Ada", "orders": [1, 2, 3]}
```

### Testing auth flows

Auth is a sequence — obtain a token, then use it — so its tests string requests together. A typical flow: POST credentials to `/login`, extract the token from the response, then send it in the `Authorization` header on a protected route and assert both the authorised (200) and unauthorised (401) cases. Testing *both* sides of the boundary (valid token works, missing/invalid token is rejected) is the point — an auth check that never rejects is worse than none.

```python
from fastapi import FastAPI, Depends, HTTPException, Header
from fastapi.testclient import TestClient

app = FastAPI()
_TOKENS = {"secret-token-abc": "ada"}

@app.post("/login")
def login(username: str, password: str):
    if username == "ada" and password == "pw":
        return {"access_token": "secret-token-abc", "token_type": "bearer"}
    raise HTTPException(401, "bad credentials")

def current_user(authorization: str = Header(default="")):
    token = authorization.removeprefix("Bearer ").strip()
    if token not in _TOKENS:
        raise HTTPException(401, "invalid token")
    return _TOKENS[token]

@app.get("/profile")
def profile(user: str = Depends(current_user)):
    return {"user": user}

client = TestClient(app)

def test_full_auth_flow():
    # 1. log in and capture the token
    login_resp = client.post("/login", params={"username": "ada", "password": "pw"})
    assert login_resp.status_code == 200
    token = login_resp.json()["access_token"]

    # 2. use it on a protected route → authorised
    ok = client.get("/profile", headers={"Authorization": f"Bearer {token}"})
    assert ok.status_code == 200 and ok.json() == {"user": "ada"}

def test_protected_route_rejects_missing_token():
    assert client.get("/profile").status_code == 401

def test_protected_route_rejects_bad_token():
    bad = client.get("/profile", headers={"Authorization": "Bearer nope"})
    assert bad.status_code == 401
```

For **async** end-to-end testing of an ASGI app (needed when you want to await inside the test, or test streaming/websockets), use httpx's `AsyncClient` with an `ASGITransport` instead of the sync `TestClient`:

```python
import httpx
import pytest
from myapp.main import app

async def test_with_async_client():                  # under pytest-asyncio (§8)
    transport = httpx.ASGITransport(app=app)
    async with httpx.AsyncClient(transport=transport, base_url="http://test") as ac:
        resp = await ac.get("/items/1")
        assert resp.status_code in (200, 404)
```

### Django test client and framework

Django ships its **own testing framework**, built on `unittest` (which is why you see `TestCase` classes and `self.assert*`), plus a **test `Client`** that makes in-process requests through Django's full middleware/URL/view stack. The standout feature is **`django.test.TestCase`**, which **wraps every test method in a database transaction and rolls it back afterward** — so tests share one migrated test database but never see each other's data, giving both speed and isolation for free (more on transactions in §10). Django also builds a throwaway test database automatically and tears it down at the end.

```python
# tests/test_views.py  (run with: pytest, using pytest-django; or manage.py test)
from django.test import TestCase, Client
from django.urls import reverse
from django.contrib.auth.models import User
from myapp.models import Article

class ArticleViewTests(TestCase):
    def setUp(self):
        # Runs per test; the surrounding transaction is rolled back after each,
        # so this data never leaks between tests.
        self.client = Client()
        self.user = User.objects.create_user("ada", password="pw")
        self.article = Article.objects.create(title="Hello", author=self.user)

    def test_article_detail_renders(self):
        resp = self.client.get(reverse("article-detail", args=[self.article.id]))
        self.assertEqual(resp.status_code, 200)
        self.assertContains(resp, "Hello")          # asserts 200 AND body contains

    def test_create_requires_login(self):
        resp = self.client.post(reverse("article-create"), {"title": "New"})
        self.assertEqual(resp.status_code, 302)     # redirected to login

    def test_create_when_logged_in(self):
        self.client.force_login(self.user)          # skip the login form
        resp = self.client.post(reverse("article-create"), {"title": "New"})
        self.assertEqual(Article.objects.filter(title="New").count(), 1)

    def test_api_returns_json(self):
        resp = self.client.get(reverse("article-json", args=[self.article.id]))
        self.assertEqual(resp.status_code, 200)
        self.assertEqual(resp.json()["title"], "Hello")
```

⚡ **Version note:** With **pytest-django** (`pip install pytest-django`) you can write Django tests as plain pytest functions using the `client`, `db`, `admin_client`, and `rf` (request factory) fixtures instead of `TestCase` subclasses — the modern, less-boilerplate way to test Django under pytest. Use `@pytest.mark.django_db` to grant a test database access; it manages the same transaction-rollback isolation. See [Django](DJANGO_GUIDE.md) for the framework details this section assumes.

---

## 10. Database Integration Testing

### The core dilemma — realism vs speed and isolation

Code that talks to a database is where the trickiest, highest-value bugs live: a subtly wrong SQL query, a migration that does not apply, a unique constraint you forgot, an ORM relationship that lazy-loads incorrectly, a transaction that does not roll back on error. **None of these show up if you mock the database** — a mocked `session.query(...)` returns whatever you told it to, so it can never reveal that your *real* query is wrong. To catch database bugs you must run against a **real database**. The problem is that a real database is *shared mutable state*, the enemy of isolation (§1): if test A inserts a user and test B counts users, they interfere. Two techniques resolve the dilemma: run a **real but disposable** database (so it is realistic), and give **each test its own transaction that rolls back** (so tests never see each other's writes). This section shows both.

⚡ **Should you test against SQLite instead of your real Postgres?** Tempting (it is fast and needs no server), but risky: SQLite and PostgreSQL differ in types, constraints, concurrency, JSON handling, and SQL dialect, so a suite green on SQLite can hide Postgres-only bugs. The rule: **test against the same engine you run in production.** Use the real thing via a container. See [PostgreSQL](POSTGRESQL_GUIDE.md) for the database itself.

### testcontainers — a real Postgres in a throwaway container

**testcontainers-python** (`pip install testcontainers[postgres]`) starts a **real database inside a Docker container** for your test session and throws it away afterward. You get genuine Postgres behaviour with zero manual setup — the library pulls the image, starts the container, waits until it is ready, hands you the connection URL, and stops it when the session ends. Make it **session-scoped** so the (relatively slow) container starts once for the whole run, then layer per-test transaction isolation on top. This needs [Docker](DOCKER_GUIDE.md) available on the machine, which is also true in CI (§18).

```python
# tests/conftest.py
import pytest
from testcontainers.postgres import PostgresContainer
from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker

@pytest.fixture(scope="session")
def pg_engine():
    """Start ONE real Postgres container for the whole test session."""
    # Pulls postgres:16, starts it, waits for readiness — all automatic.
    with PostgresContainer("postgres:16-alpine") as postgres:
        engine = create_engine(postgres.get_connection_url())
        # Create the schema once. In a real project run your migrations here
        # (Alembic/Django) instead of raw DDL.
        with engine.begin() as conn:
            conn.execute(text("""
                CREATE TABLE users (
                    id   SERIAL PRIMARY KEY,
                    name TEXT NOT NULL,
                    email TEXT UNIQUE NOT NULL
                )
            """))
        yield engine
        engine.dispose()
    # container is stopped & removed automatically on exit
```

### Transaction-rollback-per-test — perfect isolation, no cleanup

The elegant isolation trick: **wrap each test in a database transaction and roll it back at the end** instead of committing. Every insert/update the test makes is visible *to that test* (so it can read back what it wrote), but the rollback erases all of it before the next test runs — so the database returns to its pristine post-migration state after every test, with no manual `DELETE`s and no order dependence. It is also *fast* (a rollback is cheaper than re-truncating tables). The pattern: open a connection, begin a transaction, bind an ORM session to that connection, `yield` the session to the test, then roll the transaction back in teardown.

```python
# tests/conftest.py  (continued)
import pytest
from sqlalchemy.orm import sessionmaker

@pytest.fixture
def db_session(pg_engine):
    """Give each test an isolated session whose writes are rolled back."""
    connection = pg_engine.connect()
    transaction = connection.begin()          # outer transaction we will roll back
    Session = sessionmaker(bind=connection)
    session = Session()

    yield session                             # <-- the test runs, reads/writes freely

    session.close()
    transaction.rollback()                    # undo EVERYTHING the test did
    connection.close()
```

```python
# tests/test_user_repo.py — testing a repository layer against real Postgres
from sqlalchemy import text

class UserRepository:
    """The 'production' data-access layer under test."""
    def __init__(self, session):
        self.session = session
    def add(self, name, email):
        row = self.session.execute(
            text("INSERT INTO users (name, email) VALUES (:n, :e) RETURNING id"),
            {"n": name, "e": email},
        ).one()
        return row.id
    def get_by_email(self, email):
        return self.session.execute(
            text("SELECT id, name, email FROM users WHERE email = :e"),
            {"e": email},
        ).mappings().first()

def test_add_and_fetch_user(db_session):
    repo = UserRepository(db_session)
    uid = repo.add("Ada", "ada@example.com")
    fetched = repo.get_by_email("ada@example.com")
    assert fetched["id"] == uid
    assert fetched["name"] == "Ada"

def test_isolation_the_previous_user_is_gone(db_session):
    # Proof of isolation: the user inserted by the previous test was rolled
    # back, so this fresh test sees an empty table.
    repo = UserRepository(db_session)
    assert repo.get_by_email("ada@example.com") is None

def test_unique_email_constraint(db_session):
    import pytest
    from sqlalchemy.exc import IntegrityError
    repo = UserRepository(db_session)
    repo.add("Ada", "dup@example.com")
    with pytest.raises(IntegrityError):        # real Postgres enforces UNIQUE
        repo.add("Bo", "dup@example.com")
```

### pytest-postgresql — a lighter alternative

If Docker is unavailable or you want an even lighter setup, **pytest-postgresql** (`pip install pytest-postgresql`) manages a Postgres instance for tests — it can start its own `postgres` process (needs the binaries installed) or connect to an existing server, and provides a `postgresql` fixture giving a fresh, clean database per test. It is a good middle ground: more realistic than SQLite, lighter than containers. testcontainers wins when you want the *exact same* image as production and hermetic CI; pytest-postgresql wins when Docker is not an option.

```python
# Uses the `postgresql` fixture provided by the plugin (a psycopg connection).
def test_with_pytest_postgresql(postgresql):
    cur = postgresql.cursor()
    cur.execute("CREATE TABLE t (id serial PRIMARY KEY, v int)")
    cur.execute("INSERT INTO t (v) VALUES (42)")
    cur.execute("SELECT v FROM t")
    assert cur.fetchone()[0] == 42
    # The plugin drops/recreates the database between tests → isolation.
```

### Seeding data with factory_boy

Tests need *data* to act on, and building it inline (`User(name="a", email="a@x.com", role="user", created=...)`) is verbose and repetitive. **factory_boy** (`pip install factory_boy`) defines **factories** — reusable blueprints that generate model instances with sensible defaults, overridable per test, with support for sequences (unique emails), sub-factories (a `Post` that auto-creates its `Author`), and `faker`-generated realistic values. This keeps tests focused on *what matters for this case* (override just the one field under test) while the factory fills in the rest.

```python
import factory
from myapp.models import User, Post          # e.g. SQLAlchemy or Django models

class UserFactory(factory.Factory):
    class Meta:
        model = User
    name = factory.Faker("name")             # realistic fake names
    # Sequence guarantees a UNIQUE email every time → no constraint clashes.
    email = factory.Sequence(lambda n: f"user{n}@example.com")
    role = "user"                            # a plain default, overridable

class PostFactory(factory.Factory):
    class Meta:
        model = Post
    title = factory.Faker("sentence")
    author = factory.SubFactory(UserFactory) # auto-builds a User for each Post

def test_admin_can_delete_any_post():
    admin = UserFactory(role="admin")        # override just the role
    post = PostFactory()                     # author auto-created
    # ... exercise the permission logic with clearly-relevant data ...
    assert admin.role == "admin" and post.author is not None

def test_bulk():
    users = UserFactory.build_batch(10)      # ten users, all unique emails
    assert len({u.email for u in users}) == 10
```

⚡ **Version note:** For SQLAlchemy use `factory.alchemy.SQLAlchemyModelFactory` (bind its `sqlalchemy_session` to your test session so built objects are persisted); for Django use `factory.django.DjangoModelFactory`. Both integrate the factory with the ORM so `UserFactory()` actually inserts a row in your rolled-back test transaction. factory_boy pairs naturally with the transaction-rollback fixture above.

### Testing the ORM/repository layer — what to actually assert

When testing a data layer, assert on the things a mock could never verify: that a row was **really persisted and reads back correctly**, that **constraints** (unique, foreign key, not-null, check) actually fire, that a **query returns the right rows in the right order** with filtering/pagination, that a **transaction rolls back on error** leaving no partial writes, and that **relationships** load as expected without N+1 surprises. Keep these as *integration* tests (marked `@pytest.mark.integration`, §5) so your fast unit suite can skip them during the inner loop and CI runs them fully.

---

## 11. End-to-End Testing with Playwright

### What e2e testing is and when it earns its keep

**End-to-end (e2e) testing** drives your application **exactly as a real user does** — a real browser opens the real (test-deployed) app, clicks buttons, fills forms, navigates pages, and asserts on what actually renders — with the *entire* stack live underneath: frontend, backend, database, auth. It gives the **highest confidence** that the system genuinely works, because it tests the integrated whole, including the JavaScript, the templates, and the wiring no lower-level test touches. That confidence is expensive: e2e tests are **slow** (seconds each, a real browser boots and paints), **flaky-prone** (timing, animations, network), and **costly to maintain** (a UI change breaks selectors). So the pyramid guidance from §1 applies hard — write **few** e2e tests, covering only **critical user journeys** (can a user log in? can they complete a purchase?), and push everything else down to faster layers.

### Playwright for Python

**Playwright** (`pip install pytest-playwright && playwright install`) is Microsoft's modern browser-automation tool with excellent Python support. Its two standout features for reliable e2e are **auto-waiting** (before interacting with an element it automatically waits for it to be attached, visible, stable, and enabled — killing the #1 source of Selenium flakiness, the manual `sleep`) and **web-first assertions** (`expect(locator).to_be_visible()` retries until true or times out). It drives Chromium, Firefox, and WebKit from one API. The `pytest-playwright` plugin gives you a `page` fixture — a fresh browser tab per test.

```bash
pip install pytest-playwright
playwright install chromium          # download the browser binaries (once)
```

```python
# tests/e2e/test_login_flow.py
# Run against a test instance of the app already serving on localhost:8000.
import re
from playwright.sync_api import Page, expect

BASE_URL = "http://localhost:8000"

def test_admin_login_then_view_dashboard(page: Page):
    # ---- Arrange/Act: go to the login page ------------------------------
    page.goto(f"{BASE_URL}/login")

    # Fill the form. get_by_label finds inputs by their <label> — resilient,
    # accessible selectors beat brittle CSS/XPath.
    page.get_by_label("Email").fill("admin@example.com")
    page.get_by_label("Password").fill("correct-horse")
    page.get_by_role("button", name="Sign in").click()

    # ---- Assert: we landed on the dashboard -----------------------------
    # Web-first assertion: auto-waits for the navigation/render. No sleep.
    expect(page).to_have_url(re.compile(r".*/dashboard"))
    expect(page.get_by_role("heading", name="Dashboard")).to_be_visible()

    # Assert the data flow: the dashboard shows the seeded metric.
    expect(page.get_by_test_id("total-users")).to_have_text("42")

def test_login_rejects_bad_password(page: Page):
    page.goto(f"{BASE_URL}/login")
    page.get_by_label("Email").fill("admin@example.com")
    page.get_by_label("Password").fill("wrong")
    page.get_by_role("button", name="Sign in").click()
    # Stays on login and shows an error — assert the failure path too.
    expect(page.get_by_role("alert")).to_contain_text("Invalid credentials")
    expect(page).to_have_url(re.compile(r".*/login"))
```

### Fixtures, configuration, and debugging

pytest-playwright supplies `page`, `browser`, `context` (an isolated browser session — cookies/storage — great for per-test auth state), and lets you configure headed/headless and browser choice from the command line. For reliable, fast, and debuggable e2e runs, know these controls:

```bash
pytest tests/e2e --headed              # watch it run in a real window (debug)
pytest tests/e2e --browser firefox     # run against Firefox (or webkit)
pytest tests/e2e --slowmo 500          # slow each action by 500ms to watch
pytest tests/e2e --tracing on          # record a trace for post-mortem debugging
pytest tests/e2e --video on            # record video of each test
playwright show-trace trace.zip        # open the interactive trace viewer
```

```python
# A fixture that logs in ONCE and reuses the authenticated storage state,
# so every e2e test starts already-logged-in without repeating the login flow.
import pytest
from playwright.sync_api import Browser

@pytest.fixture(scope="session")
def auth_state(browser: Browser, tmp_path_factory):
    state_file = tmp_path_factory.mktemp("auth") / "state.json"
    page = browser.new_page()
    page.goto("http://localhost:8000/login")
    page.get_by_label("Email").fill("admin@example.com")
    page.get_by_label("Password").fill("correct-horse")
    page.get_by_role("button", name="Sign in").click()
    page.wait_for_url("**/dashboard")
    page.context.storage_state(path=state_file)   # save cookies/localStorage
    page.close()
    return str(state_file)

@pytest.fixture
def logged_in_page(browser: Browser, auth_state):
    context = browser.new_context(storage_state=auth_state)  # reuse the session
    page = context.new_page()
    yield page
    context.close()

def test_dashboard_loads_fast(logged_in_page):
    logged_in_page.goto("http://localhost:8000/dashboard")
    from playwright.sync_api import expect
    expect(logged_in_page.get_by_role("heading", name="Dashboard")).to_be_visible()
```

### Best practices for non-flaky e2e

- **Select by role/label/test-id, not by CSS class or text position.** `get_by_role("button", name="Save")` and `get_by_test_id("submit")` survive restyling; `.btn.btn-primary:nth-child(3)` breaks on any layout change. Add stable `data-testid` attributes to elements your tests target.
- **Never `sleep`.** Rely on auto-waiting and web-first `expect` assertions, which retry until the condition holds or a timeout fires. A hard-coded sleep is either too short (flaky) or too long (slow) and always wrong.
- **Isolate state.** Each test should start from a known server state — seed the database via an API/fixture before the run and reset between tests, so e2e tests are not order-dependent.
- **Run headless in CI, headed locally for debugging.** Use `--tracing on` / `--video on` in CI so you can inspect *why* a test failed on a machine you cannot watch.
- **Keep the count small and the journeys critical.** If you are writing an e2e test for a pure calculation or a validation rule, you are testing at the wrong layer — move it down to a unit or API test.

---

## 12. Property-Based Testing with Hypothesis

### The limitation of example-based testing

Every test you have written so far is **example-based**: *you* pick specific inputs (`add(2, 3)`, `parse("banana")`) and assert specific outputs. This has a fundamental blind spot — **you only test the cases you thought of.** Bugs live in the cases you *did not* think of: the empty string, the Unicode surrogate, the integer one past the boundary, the list with a duplicate, the negative zero, the value that overflows. Human intuition is systematically bad at generating these adversarial inputs, so example-based suites tend to test the "happy middle" and miss the edges where code actually breaks. You can add more examples, but you are always limited by your own imagination about what could go wrong.

**Property-based testing (PBT) inverts the approach.** Instead of "for *this input*, expect *this output*," you state a **property** — a rule that must hold for *all* valid inputs — and let the tool **generate hundreds of diverse inputs** (including nasty edge cases it is specifically built to try) attempting to **falsify** your property. If it finds even one input that breaks the rule, it reports it. This is how you find the bug you never would have written an example for. Python's tool for this is **Hypothesis** (`pip install hypothesis`), and it is one of the language's genuine standout testing technologies — its ideas came from Haskell's QuickCheck but its Python implementation is exceptional.

### @given and strategies — the basics

The core is the **`@given`** decorator plus **strategies**. A *strategy* (from `hypothesis.strategies`, conventionally imported as `st`) describes **how to generate a kind of value**: `st.integers()`, `st.text()`, `st.lists(st.integers())`, `st.floats()`, and so on. `@given(st.integers())` turns your test function's parameter into a *generated* value and runs the test body **many times** (100 by default) with different generated inputs. Your job is no longer to pick inputs but to state what must always be *true* about the output.

```python
from hypothesis import given, strategies as st

def encode(text: str) -> bytes:
    return text.encode("utf-8")

def decode(data: bytes) -> str:
    return data.decode("utf-8")

# PROPERTY: encoding then decoding any string returns the original string.
# Hypothesis generates ~100 wildly different strings — ASCII, emoji, control
# chars, empty, huge — and checks the round-trip holds for EVERY one.
@given(st.text())
def test_encode_decode_round_trips(s):
    assert decode(encode(s)) == s

# PROPERTY: sorting is idempotent — sorting an already-sorted list is a no-op.
@given(st.lists(st.integers()))
def test_sort_is_idempotent(xs):
    once = sorted(xs)
    assert sorted(once) == once

# PROPERTY: sorted output has the same length and elements as the input.
@given(st.lists(st.integers()))
def test_sort_preserves_multiset(xs):
    result = sorted(xs)
    assert len(result) == len(xs)
    assert sorted(result) == sorted(xs)          # same multiset
    # and it's actually ordered:
    assert all(result[i] <= result[i + 1] for i in range(len(result) - 1))
```

### Thinking in properties — the hard and valuable part

The skill of PBT is **finding properties**, since you rarely know the exact expected output for a *random* input. Several recurring patterns unlock most cases:

- **Round-trip / inverse.** Encode↔decode, serialize↔deserialize, `to_json`↔`from_json`, compress↔decompress: doing an operation and its inverse should return the original. This catches an enormous class of bugs and needs no known expected value.
- **Invariants that always hold.** Properties of the *output* regardless of input: a sort's output is ordered and a permutation of the input; a `reverse(reverse(x)) == x`; a balance is never negative; the length of a filtered list is ≤ the original.
- **Oracle / cross-check.** Compare your implementation against a **simpler, slower, obviously-correct** reference (a brute-force version, or the stdlib). If your fast `my_sort` disagrees with `sorted` on any input, one of them is wrong.
- **Metamorphic relations.** How the output *changes* when the input changes in a known way: `add(x, y) == add(y, x)` (commutativity); adding an item then removing it leaves the collection unchanged; `f(x)` and `f(x + noise)` relate predictably.
- **Never crashes.** The weakest but still useful property: for *any* valid input the function does not raise an unexpected exception. Great for parsers and input handlers.

```python
from hypothesis import given, strategies as st

# ORACLE: cross-check a custom implementation against a trusted reference.
def my_max(xs):
    m = xs[0]
    for x in xs[1:]:
        if x > m:
            m = x
    return m

@given(st.lists(st.integers(), min_size=1))     # non-empty
def test_my_max_matches_builtin(xs):
    assert my_max(xs) == max(xs)                 # `max` is the oracle

# METAMORPHIC: reversing twice is identity.
@given(st.lists(st.integers()))
def test_double_reverse_is_identity(xs):
    assert list(reversed(list(reversed(xs)))) == xs
```

### Shrinking — why the failures are actually useful

Here is the feature that makes Hypothesis *practical* rather than merely clever: **shrinking**. When a random generator finds a failing input, that input is usually messy — a 400-character string, a list of 50 random integers — and staring at it tells you nothing. Hypothesis does not report that raw failure. Instead it **automatically simplifies** the failing input, repeatedly trying "smaller" versions (shorter lists, smaller numbers, simpler strings) while the failure persists, until it finds the **minimal example** that still breaks the property. So instead of "fails on `[8823, -4, 991, 0, 17, ...]`" you get "fails on `[0, 0]`" — a tiny, comprehensible reproduction that points straight at the bug. Hypothesis also **remembers** failing examples in a local database and replays them first on the next run, so once it finds a bug it will keep failing until you fix it.

```python
from hypothesis import given, strategies as st

def buggy_average(xs):
    return sum(xs) / len(xs)      # BUG: crashes on the empty list

@given(st.lists(st.integers()))
def test_average_never_crashes(xs):
    buggy_average(xs)
# Hypothesis generates many lists, hits the empty one, and SHRINKS the failing
# input to the minimal reproduction:
#   Falsifying example: test_average_never_crashes(xs=[])
# One glance and you know exactly the edge case you missed.
```

### assume — discarding irrelevant inputs

Sometimes a generated input is not valid for the property you are stating (you want only non-empty lists, or only pairs where `a != b`). You *could* narrow the strategy, but when the constraint is awkward to express in a strategy, **`assume(condition)`** tells Hypothesis "if this condition is false, this example is irrelevant — discard it and try another." Use it sparingly: if `assume` rejects most generated inputs, generation becomes slow and Hypothesis warns you; prefer constraining the strategy itself (`min_size=1`, `st.integers(min_value=1)`) when you can.

```python
from hypothesis import given, assume, strategies as st

@given(st.integers(), st.integers())
def test_division_property(a, b):
    assume(b != 0)                     # skip examples where b is zero
    q = a // b
    # property: the quotient times b, plus remainder, reconstructs a
    assert q * b + (a % b) == a

# Better when possible: constrain the STRATEGY instead of filtering after.
@given(st.integers(), st.integers().filter(lambda n: n != 0))
def test_division_property_v2(a, b):
    assert (a // b) * b + (a % b) == a
```

### Composing and customising strategies

Real code takes structured inputs — a `User` with a name, an age in a range, an email — so you compose strategies to generate valid domain objects. Key building blocks: `st.builds(Cls, ...)` constructs objects from strategies; `st.one_of(...)` picks among alternatives; `.map(fn)` transforms generated values; `.filter(pred)` keeps only some; and `@st.composite` lets you write a function that draws several values and assembles them (for dependencies between fields). Generating *valid* domain objects and asserting properties on your business logic is where PBT delivers the most value.

```python
from dataclasses import dataclass
from hypothesis import given, strategies as st

@dataclass
class User:
    name: str
    age: int

# st.builds: generate Users from field strategies.
user_strategy = st.builds(
    User,
    name=st.text(min_size=1, max_size=50),
    age=st.integers(min_value=0, max_value=130),
)

@given(user_strategy)
def test_user_age_is_plausible(u):
    assert 0 <= u.age <= 130

# @st.composite: when fields depend on each other (start < end).
@st.composite
def date_ranges(draw):
    start = draw(st.integers(min_value=0, max_value=1000))
    length = draw(st.integers(min_value=0, max_value=1000))
    return (start, start + length)             # guarantees end >= start

@given(date_ranges())
def test_range_is_ordered(rng):
    start, end = rng
    assert start <= end
```

### settings — tuning example count, deadlines, and determinism

The **`@settings`** decorator tunes how Hypothesis runs a test: `max_examples` (how many inputs to try — raise it for critical code, lower it for speed), `deadline` (per-example time limit; raise or disable it for legitimately slow properties to avoid false flaky failures), and `derandomize` (use a fixed seed for reproducible generation in CI). You can also register profiles for different environments (a fast profile locally, a thorough one nightly).

```python
from hypothesis import given, settings, strategies as st, HealthCheck

@settings(max_examples=500)                  # try harder on important code
@given(st.text())
def test_parser_thoroughly(s):
    ...

@settings(deadline=None)                     # this property is legitimately slow
@given(st.lists(st.integers(), max_size=10_000))
def test_slow_property(xs):
    ...

# Register profiles once (e.g. in conftest.py), select with an env var / CI.
# settings.register_profile("ci", max_examples=1000, derandomize=True)
# settings.register_profile("dev", max_examples=20)
# settings.load_profile("ci")
```

### Stateful testing — finding bugs in sequences of operations

The most powerful (and advanced) Hypothesis feature is **stateful testing**: instead of generating single inputs, it generates **sequences of method calls** against a system and checks invariants hold throughout. You describe the operations as *rules* on a `RuleBasedStateMachine`, and Hypothesis explores random interleavings — then *shrinks a failing sequence* to the minimal series of steps that breaks an invariant. This finds order-dependent bugs (a cache that corrupts after a specific add/evict pattern, a state machine that reaches an illegal state) that no single-input test could. The canonical demonstration is testing your implementation against a simple *model*: run each operation on both your real object and a trivially-correct model, and assert they always agree.

```python
from hypothesis.stateful import RuleBasedStateMachine, rule, invariant
from hypothesis import strategies as st

class Stack:
    """The system under test — a stack we suspect might have a bug."""
    def __init__(self): self._items = []
    def push(self, x): self._items.append(x)
    def pop(self): return self._items.pop()
    def size(self): return len(self._items)

class StackModel(RuleBasedStateMachine):
    """Hypothesis drives random push/pop sequences and checks a MODEL
    (a plain Python list) always agrees with the real Stack."""
    def __init__(self):
        super().__init__()
        self.real = Stack()
        self.model = []                       # the obviously-correct reference

    @rule(value=st.integers())
    def push(self, value):
        self.real.push(value)
        self.model.append(value)              # mirror the op on the model

    @rule()
    def pop(self):
        if self.model:                        # only pop when non-empty
            assert self.real.pop() == self.model.pop()

    @invariant()
    def sizes_always_match(self):
        # Checked after EVERY operation in the generated sequence.
        assert self.real.size() == len(self.model)

# Expose the machine as a normal test case for pytest to collect.
TestStack = StackModel.TestCase
```

### When to use Hypothesis

PBT shines on **pure functions with clear properties**: parsers/serializers (round-trip), data transformations, encoders, math, sorting/searching, anything with an obvious invariant or a reference implementation. It complements — does not replace — example-based tests: keep a few readable example tests as living documentation of specific behaviours, and add property tests to hunt the edge cases. It is *less* natural for heavily I/O-bound or effectful code, though stateful testing extends its reach. The mental shift — from "what output for this input?" to "what must be true for *all* inputs?" — is the valuable part, and it will change how you think about your code's contracts even when you are not using Hypothesis.

---

## 13. Benchmarking with pytest-benchmark

### What benchmarking tests and why it is different

Everything so far tested **correctness** — *does the code produce the right answer?* **Benchmarking** tests **performance** — *how fast does it run?* — which matters when speed is a requirement (a hot loop, an API latency budget) and, just as importantly, when you want to catch **performance regressions**: a refactor that quietly makes a function 10× slower. **pytest-benchmark** (`pip install pytest-benchmark`) adds a `benchmark` fixture that runs your code **many times**, discards warmup noise, and reports statistics (min, max, mean, median, standard deviation, operations/second) with proper methodology — far more reliable than a hand-rolled `time.time()` around one call, which is dominated by noise.

```python
def fibonacci(n):
    if n < 2:
        return n
    a, b = 0, 1
    for _ in range(n - 1):
        a, b = b, a + b
    return b

def test_fibonacci_performance(benchmark):
    # benchmark() calls the function many times and measures it properly.
    # The RETURN value is the function's result, so you can still assert
    # correctness alongside the timing.
    result = benchmark(fibonacci, 30)
    assert result == 832040
    # pytest-benchmark prints a table: Min, Max, Mean, StdDev, Median, OPS...

def test_with_setup_per_round(benchmark):
    # When each call needs fresh input (e.g. mutating a list), use pedantic
    # mode with a setup callable so timing excludes the setup.
    def setup():
        return ([3, 1, 2] * 1000,), {}        # (args, kwargs) rebuilt each round
    benchmark.pedantic(sorted, setup=setup, rounds=100, iterations=1)
```

### Comparing runs and failing on regressions

The real power is **saving a baseline and comparing against it**, so CI can *fail* if performance degrades beyond a threshold. This turns speed into something the suite enforces, just like coverage.

```bash
# Save this run as a named baseline.
pytest tests/test_perf.py --benchmark-save=baseline

# Later, compare the current run against it and FAIL if the mean got >20% worse.
pytest tests/test_perf.py --benchmark-compare=baseline \
    --benchmark-compare-fail=mean:20%

# Only run benchmarks (skip normal tests), or skip benchmarks entirely:
pytest --benchmark-only
pytest --benchmark-skip
```

⚡ **Version note:** pytest-benchmark 5.x supports Python 3.9+. Keep benchmarks in a **separate marker or directory** and `--benchmark-skip` them in the normal fast suite — they are slow by design and their timings are noisy on shared CI runners, so treat CI benchmark *comparisons* as advisory signals (or run them on a dedicated, quiet machine) rather than hard gates that flake. For profiling *why* something is slow (as opposed to *whether* it regressed), reach for `cProfile`, `py-spy`, or `scalene` instead — benchmarking measures, profiling explains.

---

## 14. doctest — Testing Your Docstrings

### What doctest is and its niche

**doctest** is a standard-library module (`import doctest`, no install) that finds snippets in your **docstrings that look like interactive Python sessions** — lines starting with `>>>` followed by their expected output — **runs them, and checks the actual output matches.** Its purpose is narrow but valuable: it keeps the **examples in your documentation honest.** Documentation examples rot — someone changes a function's behaviour and forgets to update the docstring, so the example now lies. doctest makes those examples *executable tests*, so an out-of-date example fails CI. It is ideal for **small, illustrative examples on pure functions** in libraries, where the example doubles as both documentation and a test.

```python
def factorial(n):
    """Return n! (the product of 1..n).

    Examples double as tests — doctest runs each >>> line and checks output:

    >>> factorial(0)
    1
    >>> factorial(5)
    120
    >>> factorial(3) * factorial(2)
    12

    It also verifies that errors happen as documented:

    >>> factorial(-1)
    Traceback (most recent call last):
        ...
    ValueError: n must be non-negative
    """
    if n < 0:
        raise ValueError("n must be non-negative")
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result
```

### Running doctests

```bash
# Run a module's doctests directly:
python -m doctest mymodule.py -v

# Better: let pytest collect them alongside your real tests.
pytest --doctest-modules            # run >>> examples in your package's docstrings
pytest --doctest-glob="*.md"        # also run examples in Markdown docs/README
```

```toml
# Make it automatic in config, and enable helpful comparison flags:
[tool.pytest.ini_options]
addopts = "--doctest-modules"
doctest_optionflags = [
    "NORMALIZE_WHITESPACE",   # treat runs of whitespace as equal
    "ELLIPSIS",               # allow `...` to match arbitrary text in output
    "IGNORE_EXCEPTION_DETAIL",# match exception type without the exact message
]
```

### Strengths, limits, and gotchas

doctest's strength is that examples are **impossible to let rot** and serve double duty as docs. Its limits define when *not* to use it: output must match **exactly** by default (a trailing space, a different float repr, or **dict/set ordering** can fail a test even when the code is correct — use `NORMALIZE_WHITESPACE` and `ELLIPSIS`, or avoid printing unordered collections); it is poor for anything needing setup, fixtures, or non-trivial assertions; and object `repr`s with memory addresses (`<obj at 0x7f...>`) will never match (use `ELLIPSIS`: `<... object at 0x...>`). **Use doctest for simple, deterministic, illustrative examples; use pytest for real test logic.** The two coexist happily — `--doctest-modules` runs both in one command.

```python
def unique_sorted(items):
    """Return sorted unique items.

    Use ELLIPSIS/NORMALIZE_WHITESPACE for anything non-trivial:

    >>> unique_sorted([3, 1, 2, 3, 1])
    [1, 2, 3]
    >>> unique_sorted([])
    []
    """
    return sorted(set(items))
```

---

## 15. Fuzzing with Atheris

### What fuzzing is and how it differs from property testing

**Fuzzing** (fuzz testing) throws **large volumes of malformed, random, and adversarial input** at code — especially code that **parses untrusted data** (file formats, network protocols, deserializers, decoders) — to find inputs that make it **crash, hang, leak memory, or misbehave**. It overlaps with property-based testing (§12) — both generate inputs to break your code — but the emphasis differs: Hypothesis generates *structured* values to falsify *logical properties*; a fuzzer generates *raw bytes* at high volume, guided by **code coverage**, hunting for *crashes and security vulnerabilities*. Fuzzing is the technique that finds buffer-overflow-class and denial-of-service bugs in parsers, which is why it is a security staple.

**Atheris** (`pip install atheris`) is Google's **coverage-guided** fuzzer for Python. "Coverage-guided" is the key idea: Atheris instruments your code, watches which branches each input exercises, and **evolves inputs that reach new code paths** — so instead of blind random noise, it intelligently mutates toward inputs that dig deeper into your parser, dramatically increasing the odds of hitting the one weird branch that crashes. It integrates with libFuzzer and can catch uncaught exceptions and, with its native extensions, memory bugs in C code called from Python.

```python
# fuzz_json_parser.py — run with: python fuzz_json_parser.py
import atheris
import sys

with atheris.instrument_imports():
    import json                      # instrument the module we're fuzzing

def test_one_input(data: bytes):
    """Atheris calls this repeatedly with mutated `data`. The FuzzedDataProvider
    turns raw bytes into typed values our target function can consume."""
    fdp = atheris.FuzzedDataProvider(data)
    text = fdp.ConsumeUnicodeNoSurrogates(fdp.remaining_bytes())
    try:
        json.loads(text)             # the target: does any input crash it?
    except (json.JSONDecodeError, ValueError, RecursionError):
        pass                         # these are EXPECTED, handled errors — ignore
    # Any OTHER exception escaping here = a bug Atheris will report, with the
    # exact input that triggered it, so you can reproduce and fix.

atheris.Setup(sys.argv, test_one_input)
atheris.Fuzz()                        # runs until it finds a crash or you stop it
```

### When to fuzz and how it fits CI

Fuzzing earns its place on any code that **handles untrusted input**: your own parsers, custom deserializers, protocol decoders, image/file processors, and anything security-sensitive. The workflow: write a small **fuzz harness** (a `test_one_input(data)` like above) that feeds the fuzzed bytes into the target and lets *unexpected* exceptions escape; run it locally to shake out crashes; then run it **continuously** (Google's **OSS-Fuzz** runs Atheris harnesses against open-source projects around the clock). When Atheris finds a crashing input it saves that input to a file so you can add it as a **regression test** in your normal pytest suite — the fuzzer finds the bug once, and an ordinary example test guards against it forever after. Because a fuzzing campaign runs unbounded (minutes to days), you do **not** put open-ended fuzzing in the fast PR suite; run harnesses on a schedule or in a dedicated job with a time budget (`-max_total_time=300`).

⚡ **Version note:** Atheris requires a C/C++ toolchain (Clang) to build and works best on Linux; on **Windows** (this author's platform) it is awkward to install, so fuzz in a Linux container or WSL, or in CI. For pure-Python logic bugs prefer Hypothesis; reach for Atheris specifically when the concern is *robustness against hostile bytes* on a real parser. `sys.setrecursionlimit` and memory limits are worth setting in the harness so a deep-recursion input reports cleanly rather than hanging.

---

## 16. Testing Filesystem, OS and Subprocess Code

### Why ambient OS state is a testing hazard

Code that touches the **outside world through the operating system** — reading and writing files, reading **environment variables**, checking the working directory, spawning **subprocesses** — is a classic source of non-deterministic, non-isolated, un-reproducible tests. If a test writes to a hard-coded path like `output.txt`, it pollutes the project, collides with parallel test runs, and leaves debris that makes the *next* run behave differently. If it reads `os.environ["HOME"]` or the real current directory, it passes on your machine and fails in CI. If it actually runs `git` or `ffmpeg`, it is slow, depends on that tool being installed, and can fail for reasons unrelated to your code. The discipline is: **never let a test touch real, shared OS state.** pytest and `monkeypatch` give you isolated, disposable substitutes for all of it.

### tmp_path — isolated, disposable filesystem

The `tmp_path` fixture (introduced in §4) is the correct way to test file I/O: it hands each test a **unique, empty directory** as a `pathlib.Path`, on the real filesystem (so your code's real `open()`/`Path.write_text()` run for real), which pytest **auto-cleans**. Each test gets its own directory, so parallel and repeated runs never collide, and there is nothing to clean up manually. Use `tmp_path` for one test's files and `tmp_path_factory` (session-scoped) when several tests share a directory tree.

```python
import json
from pathlib import Path

def save_config(directory: Path, settings: dict) -> Path:
    path = directory / "config.json"
    path.write_text(json.dumps(settings, indent=2), encoding="utf-8")
    return path

def load_config(path: Path) -> dict:
    return json.loads(path.read_text(encoding="utf-8"))

def test_config_round_trips_on_disk(tmp_path):
    # tmp_path is a fresh, isolated directory unique to THIS test.
    settings = {"debug": True, "workers": 4}
    written = save_config(tmp_path, settings)

    assert written.exists()
    assert written.name == "config.json"
    assert load_config(written) == settings
    # No cleanup needed — pytest removes tmp_path automatically.

def test_missing_file_raises(tmp_path):
    import pytest
    with pytest.raises(FileNotFoundError):
        load_config(tmp_path / "does-not-exist.json")

def test_directory_creation(tmp_path):
    nested = tmp_path / "a" / "b" / "c"
    nested.mkdir(parents=True)
    (nested / "note.txt").write_text("hi", encoding="utf-8")
    assert (nested / "note.txt").read_text(encoding="utf-8") == "hi"
```

⚡ **Version note (Windows):** Because this guide's author is on **Windows 11**, be alert to OS-path differences: build paths with `pathlib` / `os.path.join`, not hard-coded `/` or `\`, so tests pass on every OS; and remember Windows may hold a file lock if you forget to close it (use `with open(...)` or `Path.write_text`, which closes for you), which can make `tmp_path` cleanup fail. `tmp_path` being a `pathlib.Path` makes cross-platform path handling natural.

### monkeypatch for environment variables and working directory

Reading configuration from **environment variables** is common and must be tested for both presence and absence, without touching the real environment (which would leak between tests and differ across machines). `monkeypatch.setenv`/`delenv` set and remove variables **only for the current test**, auto-restoring afterward. `monkeypatch.chdir` changes the working directory safely. This keeps environment-dependent code deterministic.

```python
import os
from pathlib import Path

def database_url() -> str:
    host = os.environ.get("DB_HOST", "localhost")
    port = os.environ.get("DB_PORT", "5432")
    if "DB_PASSWORD" not in os.environ:
        raise RuntimeError("DB_PASSWORD is required")
    return f"postgresql://{host}:{port}/app"

def test_database_url_from_env(monkeypatch):
    monkeypatch.setenv("DB_HOST", "db.internal")
    monkeypatch.setenv("DB_PORT", "6543")
    monkeypatch.setenv("DB_PASSWORD", "secret")
    assert database_url() == "postgresql://db.internal:6543/app"

def test_defaults_when_unset(monkeypatch):
    monkeypatch.delenv("DB_HOST", raising=False)   # ensure it's absent
    monkeypatch.delenv("DB_PORT", raising=False)
    monkeypatch.setenv("DB_PASSWORD", "x")
    assert database_url() == "postgresql://localhost:5432/app"

def test_missing_password_raises(monkeypatch):
    import pytest
    monkeypatch.delenv("DB_PASSWORD", raising=False)
    with pytest.raises(RuntimeError, match="DB_PASSWORD"):
        database_url()

def test_relative_paths_resolve_from_cwd(monkeypatch, tmp_path):
    monkeypatch.chdir(tmp_path)                     # cwd is now the temp dir
    Path("local.txt").write_text("data", encoding="utf-8")   # relative path
    assert (tmp_path / "local.txt").exists()
```

### Faking subprocess calls

Code that shells out with **`subprocess`** (running `git`, a CLI tool, a converter) should almost never *actually run the command* in a unit test — that couples your test to an external binary being installed, makes it slow, and risks real side effects. Instead, **replace the subprocess call with a mock** that returns a canned result, and assert your code (a) built the right command and (b) handled the output/return-code correctly. Patch `subprocess.run` (or `.check_output`, `.Popen`) where the code looks it up. Reserve *real* subprocess execution for a small number of integration tests that genuinely need to verify the external tool interaction.

```python
import subprocess

def current_branch() -> str:
    result = subprocess.run(
        ["git", "rev-parse", "--abbrev-ref", "HEAD"],
        capture_output=True, text=True, check=True,
    )
    return result.stdout.strip()

def deploy(env: str) -> bool:
    completed = subprocess.run(["deploy-tool", "--env", env], capture_output=True)
    return completed.returncode == 0
```

```python
from unittest.mock import patch, MagicMock
import subprocess
import pytest

def test_current_branch_parses_git_output():
    fake = MagicMock(stdout="main\n", returncode=0)
    # Patch subprocess.run where THIS module looks it up.
    with patch("subprocess.run", return_value=fake) as mock_run:
        assert current_branch() == "main"
        # Assert we built the correct command — a real bug-catcher.
        mock_run.assert_called_once_with(
            ["git", "rev-parse", "--abbrev-ref", "HEAD"],
            capture_output=True, text=True, check=True,
        )

def test_deploy_reports_failure_on_nonzero_exit():
    fake = MagicMock(returncode=1)
    with patch("subprocess.run", return_value=fake):
        assert deploy("staging") is False

def test_current_branch_propagates_git_error():
    # side_effect raises, simulating `git` failing (check=True → CalledProcessError).
    with patch("subprocess.run",
               side_effect=subprocess.CalledProcessError(128, "git")):
        with pytest.raises(subprocess.CalledProcessError):
            current_branch()
```

For heavier filesystem needs there is also **pyfakefs** (`pip install pyfakefs`), which provides a `fs` fixture replacing the *entire* filesystem with an in-memory fake — every `open`, `os`, and `pathlib` call is intercepted — useful when code touches many paths or absolute locations you cannot redirect with `tmp_path`. For most cases, though, `tmp_path` + `monkeypatch` is simpler and closer to reality.

---

## 17. Mutation Testing

### The question mutation testing answers — "who tests the tests?"

Coverage (§7) tells you which lines *ran*, but not whether your assertions would actually **catch a bug** in those lines. You can have 100% coverage with tests that assert nothing meaningful. **Mutation testing** measures the thing coverage cannot: **the ability of your test suite to detect faults.** The technique is deviously simple — the tool makes tiny, deliberate **mutations** to your production code (change `+` to `-`, `<` to `<=`, `True` to `False`, delete a line, swap `and` for `or`), each one a plausible bug, and **reruns your tests against each mutant.** If a test **fails**, the mutant is *killed* — good, your suite caught that fault. If **all tests still pass** despite the code being broken, the mutant *survived* — a gap: your tests execute that code but do not actually check its behaviour. The **mutation score** (killed ÷ total) is a far truer measure of test quality than coverage.

```python
# Suppose this is your production code:
def is_adult(age):
    return age >= 18

# And your only test:
def test_is_adult():
    assert is_adult(25) is True          # coverage: 100% — the line ran!

# A mutation testing tool will try mutants like:
#   return age > 18     (>= became >)   → test_is_adult(25) still passes → SURVIVED
#   return age <= 18    (>= became <=)  → test_is_adult(25) FAILS         → killed
# The SURVIVED mutant `>` reveals the gap: you never tested the boundary (18).
# Adding `assert is_adult(18) is True` and `assert is_adult(17) is False`
# kills it — and, not coincidentally, tests the exact edge case that matters.
```

### mutmut and cosmic-ray

Two mature Python tools. **mutmut** (`pip install mutmut`) is the easy on-ramp: point it at your source, it mutates and runs your suite, and reports survivors for you to inspect. **cosmic-ray** (`pip install cosmic-ray`) is more configurable and distributable for larger codebases. Both are **slow** — they run your (sub)suite once per mutant, and there can be thousands of mutants — so you target them at your **most critical modules** (pricing, auth, validation, anything where a silent bug is costly) rather than the whole codebase, and run them on a schedule rather than every commit.

```bash
# --- mutmut ---------------------------------------------------------------
pip install mutmut
mutmut run --paths-to-mutate src/myapp/pricing.py   # mutate one critical module
mutmut results                 # list survived/killed mutants
mutmut show 3                  # show the exact code change of mutant #3
mutmut show all                # review every survivor to decide what to test

# --- cosmic-ray (config-driven) -------------------------------------------
pip install cosmic-ray
cosmic-ray init config.toml session.sqlite   # plan the mutations
cosmic-ray exec config.toml session.sqlite   # run them
cr-report session.sqlite                      # human-readable survivor report
```

### How to use it well — and its limits

Read every **survivor** and ask: *does this mutant represent a real bug my tests should catch?* Often yes — you add the missing boundary or negative-case assertion, killing the mutant and genuinely strengthening the suite (this is the payoff: mutation testing tells you *exactly which assertion is missing*). But some survivors are **equivalent mutants** — a change that does not alter observable behaviour (e.g. mutating a value used only in a log message), which *cannot* be killed and should be ignored. Do not chase a 100% mutation score; use it as a **targeted audit** of critical code to find the specific weak assertions coverage hides. Because it is expensive, the practical pattern is: high coverage everywhere (cheap, catches "did I forget to test this at all"), plus periodic mutation testing on the crown-jewel modules (expensive, catches "are my assertions actually strong").

⚡ **Version note:** mutmut **3.x** rewrote its engine for substantially better speed and a clearer workflow; it caches results so re-runs after code changes only re-test affected mutants. Even so, scope it tightly and run it in a nightly/weekly CI job, never in the fast PR gate.

---

## 18. Multi-Env and CI

### Why test across environments — and what tox and nox do

Your suite passing on *your* machine, with *your* Python and *your* dependency versions, proves it works in exactly one place. Real software must run across **multiple Python versions** (3.13 and 3.14), sometimes **multiple dependency versions** (Django 5.1 and 5.2), on **multiple operating systems**. **`tox`** and **`nox`** automate this: they create a **fresh, isolated virtual environment for each combination**, install your package and its test dependencies into each, and run your suite in every one — so you catch "works on 3.13, breaks on 3.14" or "breaks with the new library release" *before* your users do. **tox** is configured declaratively (an ini file listing environments); **nox** is configured in Python (a `noxfile.py`), giving you loops and conditionals for complex matrices. Both also standardise the "one command to test everything" entry point that CI calls.

```ini
# tox.ini — declarative: run the suite on several Python versions + a lint env.
[tox]
env_list = py313, py314, lint, type
isolated_build = True            # build the package via pyproject.toml

[testenv]                        # the default test environment template
deps =
    pytest
    pytest-cov
    hypothesis
commands =
    pytest --cov=myapp --cov-branch --cov-fail-under=85 {posargs}
    # {posargs} forwards extra CLI args: `tox -- -k auth -x`

[testenv:lint]
deps = ruff
commands = ruff check src tests

[testenv:type]
deps = mypy
commands = mypy src
```

```python
# noxfile.py — programmatic: the same matrix, but with Python logic.
import nox

@nox.session(python=["3.13", "3.14"])
def tests(session):
    session.install(".", "pytest", "pytest-cov", "hypothesis")
    # session.posargs forwards extra args, e.g. `nox -- -k auth`
    session.run("pytest", "--cov=myapp", "--cov-branch", *session.posargs)

@nox.session
def lint(session):
    session.install("ruff")
    session.run("ruff", "check", "src", "tests")

# Parametrize across a dependency version too — nox shines at real matrices:
@nox.session(python=["3.13", "3.14"])
@nox.parametrize("django", ["5.1", "5.2"])
def django_tests(session, django):
    session.install(f"django=={django}", "pytest", "pytest-django")
    session.run("pytest", "tests/django")
```

### Running the suite in GitHub Actions with a matrix

Continuous integration (CI) runs your tests **automatically on every push and pull request**, so a regression is caught in minutes and a broken change cannot merge. **GitHub Actions** is the common choice; a **matrix** runs the same job across several Python versions (and OSes) in parallel. The pattern: check out code, set up each Python, install deps, run pytest with coverage, upload the coverage report. For integration tests needing Postgres (§10), Actions provides **service containers** (or you use testcontainers, which starts its own). See [GitHub Actions CI/CD](GITHUB_ACTIONS_CICD_GUIDE.md) for the workflow syntax in depth and [Docker](DOCKER_GUIDE.md) for the container mechanics.

```yaml
# .github/workflows/tests.yml
name: tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false                     # let all matrix legs finish, not stop at first fail
      matrix:
        python-version: ["3.13", "3.14"]
    services:
      postgres:                            # a real Postgres for integration tests
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready --health-interval 10s
          --health-timeout 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: pip
      - run: pip install -e ".[test]"
      - run: pytest --cov=myapp --cov-branch --cov-report=xml --cov-fail-under=85 -n auto
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/postgres
      - uses: codecov/codecov-action@v4    # publish coverage (optional)
        with:
          files: coverage.xml
```

### Parallelising with pytest-xdist

As a suite grows, running it serially gets slow. **pytest-xdist** (`pip install pytest-xdist`) runs tests **across multiple CPU cores** (or even multiple machines), cutting wall-clock time roughly in proportion to core count. `pytest -n auto` uses all available cores. The catch, and it is an important one: **parallelism is only safe if your tests are truly isolated** (§1) — if two tests share a file, a database row, or a global, running them simultaneously causes race conditions and flaky failures. So xdist is both a speedup *and* a stress-test of your isolation discipline: tests that fail only under `-n auto` are tests with hidden shared state you need to fix.

```bash
pytest -n auto                 # use all CPU cores
pytest -n 4                    # exactly 4 workers
pytest -n auto --dist loadscope  # group tests by module/class onto workers
                                 # (keeps a module's tests on one worker — helps
                                 #  when they share an expensive module fixture)
```

### Handling flaky tests with pytest-rerunfailures

Despite your best efforts, some tests are **flaky** — they fail intermittently for reasons *outside* the code under test (a genuinely non-deterministic external service, an unavoidable timing sensitivity in an e2e test). **pytest-rerunfailures** (`pip install pytest-rerunfailures`) can **automatically re-run a failing test** a few times before declaring it failed, papering over transient blips. **Use this as a last resort and a temporary measure, never a fix** — auto-retrying a flaky test *hides* the flakiness rather than curing it, and a test that only passes on the third try is a test you cannot trust. The correct response to flakiness is to *find and remove the source of non-determinism* (freeze time, seed randomness, fake the network, fix shared state); reserve reruns for the small residue of genuinely-external flakiness, ideally scoped to the specific tests via a marker rather than applied suite-wide.

```bash
pytest --reruns 3                          # retry any failing test up to 3 times
pytest --reruns 3 --reruns-delay 1         # wait 1s between retries
pytest --only-rerun "ConnectionError"      # only retry failures matching this
```

```python
import pytest

# Scope reruns to the genuinely-flaky test, not the whole suite:
@pytest.mark.flaky(reruns=3, reruns_delay=1)
def test_calls_a_real_external_service():
    ...
```

### Configuration recap — one place for everything

Pull the recurring configuration together in `pyproject.toml` so every developer and CI runner behaves identically: test paths, default options, registered markers, async mode, coverage settings, and xfail strictness. This single source of truth is what makes "just run `pytest`" reliable across machines.

```toml
# pyproject.toml — the consolidated testing configuration.
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra --strict-markers --strict-config --cov=myapp --cov-branch --cov-fail-under=85"
asyncio_mode = "auto"
xfail_strict = true
markers = [
    "slow: slow tests (deselect with -m 'not slow')",
    "integration: needs a real DB/network",
    "e2e: end-to-end browser tests",
]

[tool.coverage.run]
branch = true
source = ["src/myapp"]

[tool.coverage.report]
show_missing = true
fail_under = 85
exclude_lines = ["pragma: no cover", "if TYPE_CHECKING:", "raise NotImplementedError"]
```

---

## 19. Gotchas and Best Practices

This section distils the recurring failure modes and the habits that avoid them. Skim it now; return whenever a test behaves strangely.

### The high-frequency gotchas

| Gotcha | Why it bites | The fix |
|---|---|---|
| **Patching where defined, not where used** | `patch("services.send")` does nothing if the module did `from services import send` — its local name is untouched, the real function runs | Patch at the point of use: `patch("mymodule.send")` (§6) |
| **Shared mutable state between tests** | A module-level list/dict, a class attribute, or an uncleaned DB row makes tests order-dependent and flaky under `-n auto` | Build fresh state per test in a function-scoped fixture; reset globals in teardown (§4) |
| **Testing implementation, not behaviour** | Asserting *how* code works (which internal method it called) breaks on every refactor even when behaviour is unchanged | Assert observable outcomes (return value, state, side effect); prefer fakes over interaction-mocks (§6) |
| **Real time in assertions** | `datetime.now()` differs every run; "expires in 1h" logic is untestable | Freeze time (freezegun) or inject a clock (§6) |
| **Unseeded randomness** | `random`/`uuid`/`secrets` make output non-deterministic → flaky | Seed the RNG (autouse fixture) or inject/mocks the random source (§4, §6) |
| **Real network calls** | Slow, flaky, depend on a third party; can fail for reasons unrelated to your code | Fake HTTP with `responses`/`respx`; mock at the boundary (§6) |
| **Hard-coded filesystem paths** | Pollute the repo, collide in parallel, differ across OSes | Use `tmp_path`; build paths with `pathlib` (§16) |
| **Over-mocking** | Mocking every collaborator tests only that mocks were called → false confidence, brittle | Mock only boundaries (network/clock/3rd-party); use real objects/fakes internally (§6) |
| **`assert` with a side effect** | `assert delete_user(1)` — if run under `python -O`, asserts are stripped and the call vanishes | Never put required logic inside `assert`; act first, then assert on the result |
| **`pytest.raises` block too broad** | A different line in the block raises the same type → test passes for the wrong reason | Wrap only the single call under test (§3) |
| **Float `==`** | `0.1 + 0.2 != 0.3` in binary float | `pytest.approx` (§3) |
| **Un-awaited coroutine** | Calling an `async def` without `await` runs nothing; the test silently tests a coroutine object | Use pytest-asyncio and `await` the call (§8) |
| **Forgetting to reset `dependency_overrides`** | Overrides leak into later tests | `app.dependency_overrides.clear()` in fixture teardown (§9) |
| **`time.sleep` to wait for something** | Slow if too long, flaky if too short; always wrong | Await/poll the actual condition; use Playwright auto-wait (§8, §11) |
| **Mutable default arguments in code under test** | `def f(x=[])` shares one list across calls → spooky cross-test state | Fix the production bug: default `None`, build inside |
| **SQLite in tests, Postgres in prod** | Dialect/constraint/type differences hide real DB bugs | Test against the production engine via containers (§10) |
| **Unregistered markers** | A typo'd `@pytest.mark.slwo` silently no-ops; the test is never deselected | `--strict-markers` + register markers in config (§3, §5) |
| **Stale `xfail`** | A fixed bug keeps its `xfail`, so a real regression later shows as "expected fail" | `xfail_strict = true` — an unexpected pass fails the build (§5) |

### The best-practice checklist

- **Prefer fixtures over `setUp`.** Fixtures are opt-in, composable, scoped, shareable via `conftest.py`, and self-cleaning. Reserve `setUp` for `unittest`/Django `TestCase` code you are maintaining (§4).
- **Test behaviour, not implementation.** A test should still pass after any refactor that preserves observable behaviour. If refactoring breaks tests that were "green," the tests were coupled to internals. Assert on outputs and effects, not on the mechanics of collaboration.
- **One logical assertion-target per test.** A test should verify *one* behaviour, so its name can describe it and its failure pinpoints it. Several `assert`s that all check facets of one outcome are fine; testing two unrelated behaviours in one function is not.
- **Name tests as behaviour sentences.** `test_withdraw_more_than_balance_raises` tells you what broke from the failure line alone. Avoid `test_1`, `test_it_works`.
- **Keep the AAA shape.** Arrange, Act, Assert — visually separated. If "Act" is more than one operation, you probably have two tests.
- **Make tests deterministic by construction.** Freeze time, seed randomness, fake the network, isolate the filesystem, avoid order-dependence. A flaky test is worse than no test — it trains people to ignore red.
- **Keep the fast suite fast.** Mark slow/integration/e2e tests and deselect them in the inner loop (`-m "not slow"`); run the full set in CI. Parallelise with `-n auto`, which also stress-tests isolation.
- **Mock at the boundaries only.** The clock, the network, the payment gateway, the email sender — fake those. Use real objects or fakes for your own domain logic. Prefer dependency injection so swapping is trivial and patching is rare.
- **Let coverage find blind spots, not grade quality.** Enable branch coverage, ratchet with `--cov-fail-under`, but remember covered ≠ tested. Audit critical modules with mutation testing (§17).
- **Reach for the right layer.** Most tests are unit tests; some are integration; a *few* are e2e. Never write an e2e test for something a unit test could cover.
- **Property-test the pure core.** For parsers, transforms, and anything with an invariant or a reference implementation, add Hypothesis tests to find the edge cases you would never enumerate (§12).
- **Treat test code as production code.** It is read far more than written; keep it clean, DRY (via fixtures/factories/parametrize), and reviewed. Bad test code rots into a suite nobody trusts.
- **Never disable a failing test to go green.** Fix it, or mark it `xfail` with a linked reason. A skipped test is a silent liability.
- **Security angle.** Test the *rejection* paths, not just the happy path: invalid tokens are refused (§9), constraints actually fire (§10), untrusted input does not crash the parser (§12, §15), and secrets never appear in serialized output. A test suite that only proves the good case leaves the attack surface unverified.

### A note on TDD

**Test-Driven Development (TDD)** — write a failing test first, write the minimum code to pass it, then refactor ("red-green-refactor") — is a discipline, not a requirement, but it delivers two concrete benefits worth trying: it *guarantees* every line has a test (you never write untested code, because the test came first), and it applies **design pressure** early (writing the test first forces you to design a usable interface before you commit to an implementation). You need not be dogmatic — many strong engineers write tests immediately *after* the code, or interleave — but experiencing TDD on a few features teaches you what "testable design" feels like from the inside, which improves your code even when you are not doing strict TDD.

---

## 20. Study Path and Build-to-Learn Projects

**Suggested order.** Read §1 until the pyramid, isolation, determinism, and AAA feel obvious — everything else rests on them. Do §2 quickly (enough to *read* `unittest`), then live in §3–§5 (pytest, fixtures, parametrize) until they are second nature; this is 80% of day-to-day testing. §6 (mocking) and §7 (coverage) round out the core. Then branch by need: §8–§10 if you build async/web/database apps; §11 for e2e; §12 (Hypothesis) is worth learning early because it changes how you think about correctness. §13–§17 are specialist tools to reach for by name. §18 makes it all run automatically. Revisit §19 whenever a test misbehaves.

If any Python fundamental feels shaky as you go — decorators (how `@pytest.fixture` works), generators/`yield` (fixture teardown), context managers (`with pytest.raises`), `async`/`await` (§8) — detour to the [Python](PYTHON_GUIDE.md) guide; testing rewards a solid grasp of all of them.

**Build-to-learn projects, in increasing difficulty:**

1. **Test a pure library (B).** Write a small module — a temperature converter, a Roman-numeral parser, a `stats` module (mean/median/mode) — and test it thoroughly with pytest: happy paths, error paths (`pytest.raises`), floats (`approx`), and heavy `parametrize`. Add `doctest` examples to the docstrings and run them with `--doctest-modules`. Goal: fluency in the core loop, discovery, and assertions.

2. **Fixtures, mocking, and coverage (B/I).** Extend project 1 into something with collaborators — a `WeatherReport` that calls an HTTP API and formats the result. Fake the HTTP with `responses`/`respx`, freeze time with freezegun for the "as of" timestamp, inject dependencies so the client is swappable, and get branch coverage to a threshold you enforce with `--cov-fail-under`. Goal: doubles, boundaries, determinism, and coverage as a ratchet.

3. **Property-based core (I).** Add a serializer/parser pair (your own tiny CSV or query-string codec) and test it with Hypothesis: round-trip properties, an oracle cross-check against the stdlib, and `assume` where needed. Then write a `RuleBasedStateMachine` for a small stateful object (an LRU cache) checked against a model. Goal: think in properties; watch shrinking hand you minimal reproductions.

4. **API integration suite (I/A).** Build a small [FastAPI](FASTAPI_GUIDE.md) (or [Django](DJANGO_GUIDE.md)) CRUD service with auth, and test it through the `TestClient`: status codes, JSON bodies, the full login→protected-route flow, validation 422s, and dependency overrides swapping the DB/current-user. Mark these `integration`. Goal: test the whole request pipeline in-process, fast.

5. **Real-database integration (A).** Add [PostgreSQL](POSTGRESQL_GUIDE.md) via testcontainers, a session-scoped container plus a per-test transaction-rollback fixture for perfect isolation, and factory_boy for seed data. Test the repository/ORM layer against real Postgres: constraints fire, queries return the right rows, transactions roll back on error. Goal: realistic data-layer testing without cross-test contamination.

6. **End-to-end and CI (A).** Add a minimal admin dashboard and one Playwright e2e test for the login→view-data journey using role/label/test-id selectors and web-first assertions. Then wire everything into [GitHub Actions](GITHUB_ACTIONS_CICD_GUIDE.md): a Python-version matrix, a Postgres service (or testcontainers, via [Docker](DOCKER_GUIDE.md)), coverage upload with an enforced floor, `pytest -n auto` for speed, and a nightly job running mutation testing (§17) on your critical module plus an Atheris fuzz harness (§15) on any parser. Goal: a production-grade, multi-layer suite that runs itself on every push.

Build each project on the previous one. By project 6 you will have written, end to end, the full taxonomy — unit, integration, e2e, property-based, mutation, and fuzz tests — running fast, isolated, and deterministically in CI. That is exactly the destination promised at the top: from "I have never written a test" to "I write efficient, production-grade tests," across every kind of testing Python offers.
