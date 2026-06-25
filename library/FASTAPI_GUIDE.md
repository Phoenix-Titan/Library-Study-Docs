# FastAPI — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Python developers who know the basics of the language (functions, classes, type hints, `async`/`await` at least by sight) and want to go from "I can write a script" to "I architect and ship production-grade, scalable, maintainable async APIs with FastAPI." If your Python is shaky, read the [Python](PYTHON_GUIDE.md) guide first — especially the sections on type hints, decorators, generators, context managers, and `asyncio`, because FastAPI leans on *all* of them. Every concept here is explained **prose-first**: *what it is*, *the logic / why FastAPI does it this way*, *what it's for and when you reach for it*, *how to use it*, the *key parameters and options*, *best practices*, and *security recommendations* — and only then the heavily-commented, runnable Python. Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **FastAPI 0.115+** (the current 0.11x line, June 2026) running on **Python 3.13 / 3.14**, with **Pydantic v2** (the Rust-cored rewrite — every model pattern here uses the v2 API: `model_config`, `Field`, `field_validator`/`model_validator`, and `Annotated`). Under the hood FastAPI is built on **Starlette** (the ASGI toolkit that does routing, requests, responses, WebSockets, middleware) and **Pydantic** (data validation), and it is served by an **ASGI server** — usually **Uvicorn**. The database layer uses **SQLAlchemy 2.0** (the new 2.0-style API, async engine) and/or **SQLModel**. Everything is async-capable throughout. Where an API moved recently, look for **⚡ Version note**. The author is on **Windows 11**, so OS-specific notes (paths, `uvloop` caveats) are flagged where they matter.
>
> **Cross-references:** Compare FastAPI with the batteries-included [Django](DJANGO_GUIDE.md). For the data layer see [PostgreSQL](POSTGRESQL_GUIDE.md), [SQLite3](SQLITE3_GUIDE.md), [Redis](REDIS_GUIDE.md), [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md), and [Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md). For deployment see [Networking](NETWORKING_GUIDE.md), [Nginx](NGINX_GUIDE.md), and [Docker](DOCKER_GUIDE.md). For auth concepts (sessions vs tokens, OAuth, password hashing) see [Better Auth](BETTERAUTH_GUIDE.md). Cross-links appear inline throughout.

---

## Table of Contents

1. [What FastAPI Is & Why It Exists](#1-what-fastapi-is--why-it-exists) **[B]**
2. [Setup, First App & the Auto Docs](#2-setup-first-app--the-auto-docs) **[B]**
3. [Path Operations & Routing](#3-path-operations--routing) **[B]**
4. [Pydantic v2 Models in Depth](#4-pydantic-v2-models-in-depth) **[B/I]**
5. [Request Handling — Bodies, Params, Forms & Files](#5-request-handling--bodies-params-forms--files) **[I]**
6. [Responses & Error Handling](#6-responses--error-handling) **[I]**
7. [Dependency Injection — FastAPI's Superpower](#7-dependency-injection--fastapis-superpower) **[I/A]**
8. [Async & Concurrency](#8-async--concurrency) **[I/A]**
9. [Databases — SQLAlchemy 2.0, SQLModel & Alembic](#9-databases--sqlalchemy-20-sqlmodel--alembic) **[I/A]**
10. [Authentication & Authorization](#10-authentication--authorization) **[I/A]**
11. [Validation, Settings & Config](#11-validation-settings--config) **[I]**
12. [Middleware & CORS](#12-middleware--cors) **[I]**
13. [Background Tasks & Long Jobs](#13-background-tasks--long-jobs) **[A]**
14. [WebSockets & Streaming](#14-websockets--streaming) **[I/A]**
15. [Testing](#15-testing) **[A]**
16. [Project Structure & Maintainability at Scale](#16-project-structure--maintainability-at-scale) **[A]**
17. [Production Deployment](#17-production-deployment) **[A]**
18. [Performance & Scaling](#18-performance--scaling) **[A]**
19. [Security Hardening](#19-security-hardening) **[A]**
20. [Gotchas & Best Practices](#20-gotchas--best-practices) **[I/A]**
21. [Study Path & Build-to-Learn Projects](#21-study-path--build-to-learn-projects)

---

## 1. What FastAPI Is & Why It Exists

### What it is

FastAPI is a modern, high-performance Python **web framework for building APIs** (mostly JSON/REST, but it also does WebSockets, server-sent events, GraphQL via add-ons, and file streaming). Its defining idea is that **you describe your data and parameters with ordinary Python type hints, and FastAPI turns those hints into runtime validation, serialization, and interactive documentation — automatically.** You write a function signature; FastAPI reads it and gives you parsing, type-coercion, error responses, and an OpenAPI schema for free.

FastAPI is not a monolith — it is a thin, brilliant *composition layer* over three mature libraries. Understanding this stack is the single most clarifying thing you can learn about the framework, because almost every "how does FastAPI do X?" question resolves to "which of the three layers owns X?":

- **Starlette** — the ASGI toolkit underneath. It owns the actual HTTP machinery: routing, the `Request`/`Response` objects, WebSockets, the middleware stack, background tasks, static files, and the test client. When you see `Request`, `Response`, `StreamingResponse`, `WebSocket`, or middleware, that is Starlette. FastAPI *subclasses* Starlette's `Starlette` application.
- **Pydantic** (v2) — the data layer. It owns validation, parsing, type coercion, and serialization. Your request/response models are Pydantic models. Pydantic v2's core is written in Rust (`pydantic-core`), which is why validation is fast. When you see `BaseModel`, `Field`, validators, or `model_dump()`, that is Pydantic.
- **Uvicorn** (or another ASGI server like Hypercorn/Granian) — the server. FastAPI/Starlette is an *ASGI application*; it cannot listen on a socket by itself. The ASGI server accepts TCP connections, speaks HTTP, and calls your app. Uvicorn is the usual choice.

**ASGI** is the key acronym. WSGI (the old Python web standard, used by Flask and classic Django) is *synchronous*: one request occupies one worker thread until it finishes. **ASGI** (Asynchronous Server Gateway Interface) is the async successor: a single process can juggle thousands of concurrent connections on one thread by suspending a request whenever it waits on I/O (a database query, an HTTP call) and running another in the meantime. This is why FastAPI can be dramatically more efficient than a sync framework for I/O-bound workloads — and it is the whole reason async matters (covered deeply in §8).

### The "type-hints → validation → docs" pipeline — why it's revolutionary

Before FastAPI, declaring an API endpoint meant writing the same information three times: once in code (pull `request.json["age"]`, cast to int, check it's positive), once in a separate validation schema (marshmallow, cerberus), and once in hand-maintained API documentation that immediately drifted out of date. FastAPI's bet is that **your function's type hints already contain all of that information**, so it should be the single source of truth:

```python
@app.post("/users")
async def create_user(user: UserCreate) -> UserPublic: ...
```

From that one line FastAPI derives: parse the request body as JSON; validate it against `UserCreate` (return a clean 422 with field-level errors if it's wrong); pass a typed `UserCreate` object to your function; serialize the returned value through `UserPublic` (dropping any field not in that model); *and* generate the OpenAPI schema entry describing the request body, the response shape, and the error format. One declaration, five jobs. Nothing drifts because there is only one place to change.

### The request lifecycle

Knowing the order of operations tells you *where* each concern belongs. A request flows through these stages:

```
        HTTP request (raw bytes on a socket)
                     │
                     ▼
   ┌───────────────────────────────────────────┐
   │ Uvicorn (ASGI server): parses HTTP, builds │
   │ the ASGI `scope`, calls the ASGI app       │
   └─────────────────────┬─────────────────────┘
                         ▼
   ┌───────────────────────────────────────────┐
   │ Middleware stack (outermost → innermost):  │
   │ CORS, GZip, your custom middleware, etc.   │
   └─────────────────────┬─────────────────────┘
                         ▼
   ┌───────────────────────────────────────────┐
   │ Routing: match method + path → a path op   │
   └─────────────────────┬─────────────────────┘
                         ▼
   ┌───────────────────────────────────────────┐
   │ Dependencies resolve (Depends, security):  │
   │ DB session, current user, settings, etc.   │
   └─────────────────────┬─────────────────────┘
                         ▼
   ┌───────────────────────────────────────────┐
   │ Parameter parsing + Pydantic validation    │
   │ (path/query/header/body). Fail → 422       │
   └─────────────────────┬─────────────────────┘
                         ▼
   ┌───────────────────────────────────────────┐
   │ YOUR path-operation function runs          │
   └─────────────────────┬─────────────────────┘
                         ▼
   ┌───────────────────────────────────────────┐
   │ Return value serialized via response_model │
   │ → JSON. Exceptions → exception handlers     │
   └─────────────────────┬─────────────────────┘
                         ▼
              HTTP response back out
```

### FastAPI vs Django vs Flask — when to choose what

| Dimension | FastAPI | Django (+DRF) | Flask |
|-----------|---------|---------------|-------|
| Primary use | Async APIs, microservices, ML serving | Full-stack web apps, admin-heavy products | Small apps, glue, prototypes |
| Async model | Native ASGI, async-first | Async support exists but ORM is sync-rooted | WSGI; async is bolted on |
| Validation/docs | Built-in (Pydantic + OpenAPI) | DRF serializers; docs via add-ons | Manual / extensions |
| ORM | Bring your own (SQLAlchemy/SQLModel) | Built-in Django ORM | Bring your own |
| Admin / batteries | None built-in | Famous admin, auth, forms, migrations | None |
| Learning curve | Gentle for API work | Steeper, more concepts | Gentlest |
| Best when | You want fast, typed, async JSON APIs | You want a CMS/product with an admin | You want minimal ceremony |

**Decide simply:** choose **Django** ([Django guide](DJANGO_GUIDE.md)) when you want a full product with a built-in admin, server-rendered pages, and an integrated ORM/auth/forms stack. Choose **Flask** for a tiny service where any structure is overhead. Choose **FastAPI** when the deliverable is an **API** — especially an I/O-bound one (talks to databases, other services, or queues), where async concurrency, typed validation, and free OpenAPI docs are a force multiplier. FastAPI is also the default choice for serving machine-learning models because of its speed and clean typing.

---

## 2. Setup, First App & the Auto Docs

### Environments: venv, pip, and uv

Never install packages into your system Python. Each project gets an **isolated virtual environment** so its dependency versions can't collide with another project's. The classic tool is the standard-library `venv`; the modern, much faster tool is **uv** (a Rust-based installer/resolver that has largely replaced `pip`+`venv`+`pip-tools` in 2026 workflows). Both are shown — use whichever you have.

⚡ **Version note:** `uv` is the recommended toolchain in 2026 — it creates environments and installs in a fraction of the time `pip` takes, and `uv run` executes a command inside the project env without manual activation. `pip` still works everywhere and is shown alongside.

```bash
# ---- Option A: standard library venv + pip ----
python -m venv .venv                  # create the environment in ./.venv
# Activate it (the activation command differs by shell/OS):
source .venv/bin/activate             # macOS / Linux (bash/zsh)
.venv\Scripts\Activate.ps1            # Windows PowerShell
# Install FastAPI. The [standard] extra pulls in uvicorn, the validation
# extras, python-multipart (forms/uploads), httpx (test client), etc.
pip install "fastapi[standard]"

# ---- Option B: uv (recommended, 2026) ----
uv init myapi && cd myapi             # scaffold a project with pyproject.toml
uv add "fastapi[standard]"            # adds to pyproject + installs into .venv
uv run fastapi dev                    # run a command inside the project env
```

⚡ **Version note:** `fastapi[standard]` is the batteries-included install and gives you the **FastAPI CLI** (`fastapi dev` / `fastapi run`), which wraps Uvicorn. If you want a minimal install, `pip install fastapi` gives you the framework only and you install `uvicorn` yourself.

### Your first app

```python
# main.py
from fastapi import FastAPI

# `app` is the ASGI application object. The ASGI server (uvicorn) imports it
# by the "module:attribute" string "main:app" and calls it for each request.
app = FastAPI(
    title="My API",            # shown in the docs UI and OpenAPI schema
    version="1.0.0",
    summary="A demo API",      # one-liner under the title in the docs
)

# A "path operation": the decorator binds an HTTP method (GET) + a path ("/")
# to the function below it. The function is the handler.
@app.get("/")
async def read_root() -> dict[str, str]:
    # Returning a dict → FastAPI serializes it to a JSON response automatically.
    return {"message": "Hello, FastAPI"}


@app.get("/items/{item_id}")
async def read_item(item_id: int, q: str | None = None):
    # `item_id: int` → taken from the URL path and validated/coerced to int.
    # `q: str | None = None` → an OPTIONAL query string param (?q=...).
    return {"item_id": item_id, "q": q}
```

### Running it with Uvicorn

```bash
# The FastAPI CLI (simplest):
fastapi dev main.py        # dev mode: auto-reload on file change, debug-friendly
fastapi run main.py        # production-ish: no reload, optimized defaults

# Or call uvicorn directly (more control):
uvicorn main:app --reload                       # dev: reload on save
uvicorn main:app --host 0.0.0.0 --port 8000     # bind all interfaces, port 8000
```

`--reload` watches your files and restarts the server when you save — invaluable in development, but **never use it in production** (it spawns a file-watcher and a reloader process and disables some optimizations). `main:app` means "import `app` from `main.py`." Binding to `0.0.0.0` makes the server reachable from other machines on the network (the default `127.0.0.1` is localhost-only); see the [Networking](NETWORKING_GUIDE.md) guide for what host/port binding actually does at the socket level.

### The auto-generated docs

The moment your app runs, FastAPI exposes three things derived from your type hints:

| URL | What it is |
|-----|------------|
| `/docs` | **Swagger UI** — interactive docs; you can fill in params and *execute* requests from the browser |
| `/redoc` | **ReDoc** — a clean, read-only three-pane reference rendering of the same schema |
| `/openapi.json` | The raw **OpenAPI 3.1** schema (machine-readable; feeds the two UIs and any client-codegen tool) |

This is not an add-on you configure — it is generated from your routes and Pydantic models on every startup, so it never drifts from the code. You can disable the docs in production (`FastAPI(docs_url=None, redoc_url=None)`) if you don't want them public.

---

## 3. Path Operations & Routing

### The vocabulary

A **path operation** is the pairing of an **HTTP method** (the *operation*) with a **path** (the URL). FastAPI gives you one decorator per method: `@app.get`, `@app.post`, `@app.put`, `@app.patch`, `@app.delete`, `@app.head`, `@app.options`. The function under the decorator is the handler. The methods carry REST semantics you should honor: **GET** reads (no body, safe, idempotent), **POST** creates, **PUT** replaces a whole resource (idempotent), **PATCH** partially updates, **DELETE** removes.

### Path and query parameters from type hints

FastAPI decides where each function parameter comes from by **name and type**:

- If the parameter name appears in the path string inside `{braces}`, it's a **path parameter**.
- Otherwise, if it's a scalar (int, str, float, bool, etc.), it's a **query parameter** (after `?` in the URL).
- A parameter typed as a Pydantic model is read from the **request body** (covered in §5).

A parameter with a default is **optional**; one without a default is **required**. The declared type drives validation and coercion — `item_id: int` will reject `/items/abc` with a clean 422 error.

```python
from enum import Enum
from fastapi import FastAPI

app = FastAPI()

# ---- Path parameters ----
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    # /users/42 → user_id == 42 (an int). /users/abc → 422 validation error.
    return {"user_id": user_id}

# Path converter for paths that themselves contain slashes (e.g. file paths):
@app.get("/files/{file_path:path}")
async def read_file(file_path: str):
    # :path lets file_path capture "a/b/c.txt" instead of stopping at the first /
    return {"file_path": file_path}

# ---- Enum path params: restrict to a fixed set + document the choices ----
class ModelName(str, Enum):
    alexnet = "alexnet"
    resnet = "resnet"

@app.get("/models/{name}")
async def get_model(name: ModelName):
    # Only "alexnet"/"resnet" are accepted; anything else → 422.
    return {"model": name}

# ---- Query parameters with pagination defaults ----
@app.get("/items")
async def list_items(skip: int = 0, limit: int = 10, active: bool = True):
    # /items?skip=20&limit=50&active=false
    # bool coercion is smart: true/false, 1/0, yes/no, on/off all work.
    return {"skip": skip, "limit": limit, "active": active}
```

**Order matters in routing.** FastAPI matches routes top-to-bottom, so a fixed path like `/users/me` must be declared *before* a variable path like `/users/{user_id}`, or `me` would be captured as a `user_id` and fail to parse.

### Organizing routes with `APIRouter`

Putting every route in one file does not scale. **`APIRouter`** is a mini-application you build in its own module and then mount onto the main app (or onto another router) with `app.include_router()`. It carries a `prefix`, `tags` (which group endpoints in the docs), shared `dependencies`, and default `responses`. This is the foundation of the modular project structure in §16.

```python
# routers/items.py
from fastapi import APIRouter, status

router = APIRouter(
    prefix="/items",          # every path here is implicitly under /items
    tags=["items"],           # groups these endpoints together in /docs
)

@router.get("")                                   # GET /items
async def list_items():
    return [{"id": 1}]

@router.post("", status_code=status.HTTP_201_CREATED)  # POST /items → 201 Created
async def create_item():
    return {"id": 2}

# main.py
from fastapi import FastAPI
from routers import items

app = FastAPI()
app.include_router(items.router)
# You can also add a global prefix/version: app.include_router(items.router, prefix="/api/v1")
```

### Status codes, tags, summaries and metadata

Each decorator accepts metadata that flows straight into both the response and the OpenAPI docs. Use the `status` module's named constants instead of magic numbers — they're self-documenting.

```python
from fastapi import status

@app.post(
    "/items",
    status_code=status.HTTP_201_CREATED,   # default success status for this route
    tags=["items"],                        # docs grouping
    summary="Create an item",              # short docs title
    description="Creates an item and returns it with a server-assigned id.",
    response_description="The created item",
)
async def create_item():
    return {"id": 1}
```

You can also write the long description as the function's **docstring** — FastAPI uses it (and renders Markdown) in the docs. Set `deprecated=True` to mark a route deprecated in the schema without removing it.

---

## 4. Pydantic v2 Models in Depth

### What Pydantic models are and why they're central

A **Pydantic model** is a class that declares a shape of data with type-annotated fields. When you instantiate it from untrusted input, Pydantic **validates, coerces, and (optionally) transforms** the data, raising a structured error if anything is wrong. In FastAPI, models are how you describe request bodies (parse + validate incoming JSON) and responses (serialize + *filter* outgoing data). They are the single source of truth that powers both validation and the OpenAPI schema.

⚡ **Version note — Pydantic v2:** This guide is entirely v2. The big v1→v2 changes you must know: configuration moved from an inner `class Config` to a `model_config = ConfigDict(...)` attribute; `.dict()`→`.model_dump()`, `.json()`→`.model_dump_json()`, `.parse_obj()`→`.model_validate()`; the `@validator`/`@root_validator` decorators became `@field_validator`/`@model_validator`; `orm_mode=True` became `from_attributes=True`; and the **`Annotated`** style is now the idiomatic way to attach constraints. v2's core is in Rust, so it is roughly 5–50× faster than v1.

### Declaring models and `Field` constraints

`Field` attaches validation rules and documentation metadata to a field. The modern idiom puts it inside `Annotated[type, Field(...)]` rather than as a default value — this keeps the *type* and the *metadata* cleanly separated and works correctly with static type checkers.

```python
from datetime import datetime
from typing import Annotated
from pydantic import BaseModel, Field, EmailStr, ConfigDict

class UserCreate(BaseModel):
    # The modern Annotated style: type + constraints, no default-as-Field trick.
    name: Annotated[str, Field(min_length=1, max_length=50)]
    email: EmailStr                       # validates RFC-compliant email (needs email-validator installed)
    age: Annotated[int, Field(ge=0, le=120)]            # ge/le = >=, <=
    password: Annotated[str, Field(min_length=8, repr=False)]  # repr=False hides it in reprs/logs
    bio: str | None = None                # optional with a default

    # Numeric constraints: gt, ge, lt, le, multiple_of
    # String constraints: min_length, max_length, pattern (regex)
    # `examples=[...]` and `description=...` enrich the OpenAPI docs.
    score: Annotated[float, Field(ge=0, le=100, examples=[87.5])] = 0.0
```

| `Field` argument | Meaning |
|------------------|---------|
| `min_length` / `max_length` | string/collection length bounds |
| `gt` / `ge` / `lt` / `le` | numeric `>`, `>=`, `<`, `<=` |
| `multiple_of` | numeric must be a multiple of this |
| `pattern` | regex the string must match |
| `default` / `default_factory` | default value; use `default_factory` for mutables (see §20) |
| `alias` / `validation_alias` / `serialization_alias` | accept/emit a different external field name |
| `description` / `examples` / `title` | OpenAPI documentation metadata |
| `repr=False` | exclude from `repr()` (useful for secrets) |
| `frozen=True` | make the field immutable after construction |

### `model_config` — model-wide behavior

`model_config = ConfigDict(...)` controls the whole model. The settings you'll use most:

```python
from pydantic import BaseModel, ConfigDict

class User(BaseModel):
    model_config = ConfigDict(
        from_attributes=True,    # allow building from ORM objects (was orm_mode); see §9
        str_strip_whitespace=True,   # trim leading/trailing spaces on all str fields
        extra="forbid",          # reject unexpected fields (default is "ignore")
        frozen=False,            # True → instances are immutable & hashable
        populate_by_name=True,   # accept either the field name or its alias on input
    )
    id: int
    name: str
```

`extra="forbid"` is a strong security default for request bodies: it rejects payloads with unexpected keys rather than silently dropping them, which surfaces client bugs and blocks parameter-pollution tricks.

### Validators: `field_validator` and `model_validator`

When a constraint can't be expressed declaratively, write a validator. **`@field_validator`** runs on one field; **`@model_validator`** runs on the whole model (use it for cross-field rules, like "password and confirm_password must match"). The `mode` controls *when* it runs: `"before"` sees the raw input (before type coercion), `"after"` sees the parsed, typed value (the common case).

```python
from typing import Annotated
from pydantic import BaseModel, Field, field_validator, model_validator

class SignUp(BaseModel):
    username: Annotated[str, Field(min_length=3)]
    password: str
    confirm_password: str

    @field_validator("username")
    @classmethod
    def username_lower_alnum(cls, v: str) -> str:
        # Runs "after": v is already a validated str. Transform + extra-validate.
        if not v.isalnum():
            raise ValueError("username must be alphanumeric")
        return v.lower()                 # returned value REPLACES the field value

    @model_validator(mode="after")
    def passwords_match(self) -> "SignUp":
        # Cross-field rule. In mode="after" you get the model instance (self).
        if self.password != self.confirm_password:
            raise ValueError("passwords do not match")
        return self
```

Raising `ValueError` (or `AssertionError`) inside a validator is the correct way to signal invalid data — Pydantic collects it into the structured validation error, which FastAPI turns into a clean **422** response listing the offending field(s).

### Nested models, response models & serialization filtering

Models compose: a field can be another model, a list of models, or a dict. This is how you describe rich, nested JSON. The crucial production pattern is **separate input and output models** so you never accidentally leak internal fields (hashed passwords, internal flags). `response_model` on the route tells FastAPI to *filter* the return value through that model — any field not on the response model is dropped, even if your function returns it.

```python
from pydantic import BaseModel, EmailStr

class Address(BaseModel):
    street: str
    city: str

class UserInDB(BaseModel):          # what we store internally
    id: int
    email: EmailStr
    hashed_password: str            # MUST NEVER reach the client
    address: Address                # nested model
    is_admin: bool = False

class UserPublic(BaseModel):        # what we expose
    id: int
    email: EmailStr
    address: Address
    # note: no hashed_password, no is_admin

@app.get("/users/{user_id}", response_model=UserPublic)
async def get_user(user_id: int) -> UserInDB:
    user = UserInDB(
        id=user_id, email="a@b.com",
        hashed_password="$argon2id$...", address=Address(street="1 Main", city="Lagos"),
    )
    # We return the FULL internal object, but response_model=UserPublic filters
    # it on the way out → hashed_password and is_admin are stripped from the JSON.
    return user
```

Useful `response_model` knobs: `response_model_exclude_unset=True` (omit fields the client didn't set), `response_model_exclude_none=True` (omit `null`s), and `response_model_exclude={"field"}` / `response_model_include={...}`. For serialization control on the model itself, use `model_dump(mode="json")` to get JSON-safe primitives, or `model_dump_json()` for a JSON string.

**Best practice:** define a `Base` model with shared fields, then derive `Create`, `Update`, and `Public` variants. This avoids repetition and makes the input/output split obvious.

---

## 5. Request Handling — Bodies, Params, Forms & Files

### The `Annotated` + `Query`/`Path`/`Body` pattern

FastAPI infers where a parameter comes from by type, but when you need **metadata** (validation rules, documentation, a different source) you attach a marker — `Query`, `Path`, `Header`, `Cookie`, `Body`, `Form`, `File` — using `Annotated`. The modern, recommended form is `Annotated[type, Query(...)]`. This is preferred over the old `q: str = Query(...)` default-value form because it keeps the function's real default usable, works with type checkers, and lets the same annotated type be reused across routes.

```python
from typing import Annotated
from fastapi import FastAPI, Query, Path, Header, Cookie

app = FastAPI()

@app.get("/search")
async def search(
    # Query param with validation + docs metadata, default empty string:
    q: Annotated[str, Query(min_length=3, max_length=50, description="search text")] = "",
    # A list query param: /search?tag=a&tag=b → tags == ["a", "b"]
    tags: Annotated[list[str], Query()] = [],
    # Path-style constraints work the same with Path(...):
    page: Annotated[int, Query(ge=1)] = 1,
):
    return {"q": q, "tags": tags, "page": page}

@app.get("/users/{user_id}")
async def get_user(
    user_id: Annotated[int, Path(ge=1, title="User ID")],
    # Header names map dash→underscore: User-Agent → user_agent
    user_agent: Annotated[str | None, Header()] = None,
    session: Annotated[str | None, Cookie()] = None,
):
    return {"user_id": user_id, "ua": user_agent}
```

**Reusable annotated types** are a maintainability win: define `PageQuery = Annotated[int, Query(ge=1, le=1000)]` once and use `page: PageQuery = 1` everywhere.

### Request bodies

A parameter typed as a Pydantic model is read from the JSON **request body**, validated, and handed to you as a typed object. Multiple body parameters get nested under their names; a single one is the body itself. `Body(embed=True)` forces a single model to be nested under its key.

```python
from pydantic import BaseModel
from typing import Annotated
from fastapi import Body

class Item(BaseModel):
    name: str
    price: float

class Buyer(BaseModel):
    id: int

@app.post("/orders")
async def create_order(
    item: Item,                                   # from body
    buyer: Buyer,                                 # also from body → JSON is {"item": {...}, "buyer": {...}}
    coupon: Annotated[str | None, Body()] = None, # a single extra body field
):
    return {"item": item, "buyer": buyer, "coupon": coupon}
```

### Forms and file uploads

HTML form submissions and file uploads are not JSON — they arrive as `application/x-www-form-urlencoded` or `multipart/form-data`. Use `Form(...)` for form fields and `UploadFile` for files. **`UploadFile` is the right choice for files** because it's a spooled file (kept in memory up to a threshold, then spilled to a temp file on disk), so huge uploads don't exhaust RAM, and it exposes async `.read()`/`.write()` methods. Forms and uploads require the `python-multipart` package (included in `fastapi[standard]`).

```python
from typing import Annotated
from fastapi import FastAPI, Form, File, UploadFile

app = FastAPI()

@app.post("/login")
async def login(
    username: Annotated[str, Form()],
    password: Annotated[str, Form()],
):
    return {"username": username}

@app.post("/upload")
async def upload(
    file: Annotated[UploadFile, File()],
    files: Annotated[list[UploadFile], File()] = [],   # multiple files
):
    contents = await file.read()                       # async read; returns bytes
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(contents),
        "extra_count": len(files),
    }
```

**Security:** never trust `file.filename` or `file.content_type` from the client. Validate the size (enforce a max), sniff the real type if it matters, generate your own storage name (avoid path traversal), and never write to a path derived from the client filename.

---

## 6. Responses & Error Handling

### What you can return, and how it's serialized

By default, whatever you return is run through FastAPI's JSON encoder (`jsonable_encoder`) and sent as `application/json`. That encoder knows how to turn Pydantic models, dataclasses, `datetime`, `UUID`, `Decimal`, `Enum`, and more into JSON-safe values. If you set `response_model`, the return value is first *filtered* through it (§4). Returning a Pydantic model directly is the clean, typed default.

### Custom responses

Sometimes you need a non-JSON response or full control over status/headers. FastAPI re-exports Starlette's response classes:

| Response class | Use for |
|----------------|---------|
| `JSONResponse` | explicit JSON (when you need custom status/headers) |
| `ORJSONResponse` / `UJSONResponse` | faster JSON serialization (orjson; great for large payloads) |
| `HTMLResponse` | returning HTML |
| `PlainTextResponse` | plain text |
| `RedirectResponse` | 3xx redirect to another URL |
| `StreamingResponse` | stream a generator/iterator (large files, SSE) without buffering it all |
| `FileResponse` | efficiently send a file from disk (handles range requests, content-type) |

⚡ **Version note:** Set a fast default encoder app-wide with `FastAPI(default_response_class=ORJSONResponse)` (requires `orjson`). It noticeably speeds up serialization of large responses.

```python
from fastapi import FastAPI, Response, status
from fastapi.responses import StreamingResponse, FileResponse, ORJSONResponse

app = FastAPI()

@app.get("/download")
async def download():
    # FileResponse streams the file efficiently and sets headers for you.
    return FileResponse("report.pdf", media_type="application/pdf", filename="report.pdf")

@app.get("/stream")
async def stream():
    async def gen():
        for i in range(1000):
            yield f"chunk {i}\n"          # yielded chunks are sent as they're produced
    return StreamingResponse(gen(), media_type="text/plain")

@app.post("/items", status_code=status.HTTP_201_CREATED)
async def create_item(response: Response):
    # Inject the Response to set headers/cookies on the normal (model) response:
    response.headers["X-Created-By"] = "api"
    return {"id": 1}
```

### Error handling: `HTTPException` and custom handlers

To return an error from inside a handler, **raise `HTTPException`**. FastAPI catches it and produces a JSON error response with the status code you give. Raising (rather than returning) lets you bail out from anywhere, including deep inside a dependency.

```python
from fastapi import FastAPI, HTTPException, status

app = FastAPI()

@app.get("/items/{item_id}")
async def get_item(item_id: int):
    db = {1: "sword"}
    if item_id not in db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Item not found",
            headers={"X-Error": "not-found"},   # optional custom headers
        )
    return {"item": db[item_id]}
```

For application-wide error policy, register **custom exception handlers**. This is how you map your own domain exceptions to HTTP responses in one place, or shape a consistent error envelope. You can also override the default 422 validation-error format.

```python
from fastapi import FastAPI, Request, status
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

app = FastAPI()

class InsufficientFunds(Exception):       # a domain exception, not HTTP-aware
    def __init__(self, needed: float):
        self.needed = needed

@app.exception_handler(InsufficientFunds)
async def handle_funds(request: Request, exc: InsufficientFunds):
    # Map the domain error → HTTP once, centrally.
    return JSONResponse(
        status_code=status.HTTP_402_PAYMENT_REQUIRED,
        content={"error": "insufficient_funds", "needed": exc.needed},
    )

@app.exception_handler(RequestValidationError)
async def validation_handler(request: Request, exc: RequestValidationError):
    # Customize the 422 body to your own envelope.
    return JSONResponse(status_code=422, content={"errors": exc.errors()})
```

**Best practice:** keep your **service layer free of HTTP concerns** — raise plain domain exceptions there and translate them to HTTP in exception handlers or a thin router. This keeps business logic reusable outside the web layer (in jobs, CLIs, tests).

---

## 7. Dependency Injection — FastAPI's Superpower

### What DI is and why it matters

**Dependency Injection** means a function declares *what it needs* and the framework *provides it* at call time, instead of the function constructing it itself. In FastAPI you write a function (or class) that produces a value — a database session, the current user, a settings object, a pagination tuple — and you `Depends()` on it from any path operation. FastAPI calls your dependency, caches the result for the request, and injects it. The same dependency can be reused across hundreds of routes; you write the "get the DB session and close it" logic once.

This is FastAPI's most powerful feature because dependencies are **composable** (a dependency can itself depend on other dependencies), **typed**, **testable** (you can override any dependency in tests — §15), and they participate in validation and the OpenAPI schema. Auth, DB access, rate limiting, multi-tenancy, and feature flags are all naturally expressed as dependencies.

```python
from typing import Annotated
from fastapi import FastAPI, Depends

app = FastAPI()

# A dependency is just a callable that returns a value.
def pagination(skip: int = 0, limit: int = 10) -> dict[str, int]:
    return {"skip": skip, "limit": min(limit, 100)}   # clamp the page size centrally

# Inject it with Annotated[..., Depends(...)]. Define the alias once and reuse it:
Pagination = Annotated[dict, Depends(pagination)]

@app.get("/items")
async def list_items(page: Pagination):
    return page          # ?skip=20&limit=5 was parsed by the dependency

@app.get("/users")
async def list_users(page: Pagination):    # reuse, zero repetition
    return page
```

### Dependencies with `yield` — setup and teardown

A dependency that uses **`yield`** runs code *before* the `yield` (setup), hands the yielded value to the path operation, and runs code *after* the `yield` (teardown) once the response is sent — even if the handler raised. This is the canonical pattern for resources that must be released: database sessions, file handles, locks. It's the FastAPI equivalent of a context manager scoped to one request.

```python
from typing import Annotated
from fastapi import Depends

async def get_db():
    session = AsyncSessionLocal()        # setup: open a session (see §9)
    try:
        yield session                    # hand it to the path operation
        await session.commit()           # commit if the handler succeeded
    except Exception:
        await session.rollback()         # roll back on any error
        raise
    finally:
        await session.close()            # teardown: ALWAYS close

DBSession = Annotated["AsyncSession", Depends(get_db)]
```

### Sub-dependencies, classes as dependencies, and caching

Dependencies form a tree: `get_current_user` depends on `get_db` and `oauth2_scheme`. FastAPI resolves the whole tree and, by default, **caches each dependency's result within a single request** — so if three things in one request depend on `get_settings`, it runs once. Disable that with `Depends(dep, use_cache=False)` when you genuinely need a fresh value each time.

A **class can be a dependency**: FastAPI treats its `__init__` parameters as sub-dependencies/params, and injects an instance. This is handy for grouping related query params or building stateful helpers.

```python
from typing import Annotated
from fastapi import Depends, Query

# Class-as-dependency: groups common list params + exposes them as attributes.
class ListParams:
    def __init__(
        self,
        q: Annotated[str | None, Query()] = None,
        skip: int = 0,
        limit: Annotated[int, Query(le=100)] = 10,
    ):
        self.q = q
        self.skip = skip
        self.limit = limit

@app.get("/products")
async def products(params: Annotated[ListParams, Depends()]):
    # `Depends()` with no arg infers the type from the annotation (ListParams).
    return {"q": params.q, "skip": params.skip, "limit": params.limit}
```

### App-wide and router-wide dependencies

When a dependency must run for *every* route (or every route in a router) but you don't need its **return value** — typically a guard like "require an API key" — attach it at the app or router level via the `dependencies=[...]` list. It runs and can raise `HTTPException`, but nothing is injected into your handlers.

```python
from fastapi import Depends, Header, HTTPException, status

async def verify_api_key(x_api_key: Annotated[str | None, Header()] = None):
    if x_api_key != "expected-secret":
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid API key")

# Runs before every request to the whole app:
app = FastAPI(dependencies=[Depends(verify_api_key)])
# Or per-router: APIRouter(dependencies=[Depends(verify_api_key)])
```

**Best practices:** keep dependencies small and single-purpose; type their return values; use `Annotated` aliases for reusable ones; prefer `yield` for anything that needs cleanup; and remember dependencies are the seam you'll override in tests (§15).

---

## 8. Async & Concurrency

### `async def` vs `def` — the most important performance decision

FastAPI lets you write a path operation (or dependency) as either `async def` or plain `def`, and it treats them differently:

- **`async def`** runs **directly on the event loop**. It must never block (no `time.sleep`, no synchronous DB driver, no blocking `requests.get`) — a single blocking call freezes the *entire* process and stalls every concurrent request. Inside it, you `await` non-blocking I/O.
- **`def`** (plain sync) is run by FastAPI **in an external thread pool** so it can't block the event loop. This is the safe choice when you must call blocking libraries (a sync DB driver, `requests`, heavy CPU work that releases the GIL, legacy SDKs).

The mental model: **use `async def` when your I/O is awaitable (async DB driver, `httpx` async client); use plain `def` when your code is blocking/sync.** The worst mistake is putting blocking code inside `async def` — see §20. When in doubt and the call is blocking, either use `def` or push the blocking work to a thread with `await anyio.to_thread.run_sync(blocking_fn, arg)`.

```python
import anyio
import httpx
from fastapi import FastAPI

app = FastAPI()

@app.get("/async-ok")
async def async_ok():
    # GOOD: httpx async client awaits I/O without blocking the loop.
    async with httpx.AsyncClient(timeout=10) as client:
        r = await client.get("https://example.com/api")
    return r.json()

@app.get("/sync-blocking")
def sync_blocking():
    # GOOD: a plain `def` route runs in a threadpool, so blocking is fine here.
    import time
    time.sleep(2)             # blocking, but isolated from the event loop
    return {"done": True}

@app.get("/offload")
async def offload():
    # Need to call blocking code from an async route? Push it to a thread.
    result = await anyio.to_thread.run_sync(_cpu_heavy, 1_000_000)
    return {"result": result}

def _cpu_heavy(n: int) -> int:
    return sum(i * i for i in range(n))
```

### When async actually helps

Async wins for **I/O-bound** workloads — APIs that spend most of their time *waiting* on databases, other HTTP services, caches, or queues. While one request awaits a DB round-trip, the event loop serves others. Async does **not** speed up **CPU-bound** work (parsing huge files, image processing, ML inference on CPU): that hogs the single thread regardless. For CPU-bound work, offload to a thread/process pool or a task queue (§13), or scale out with more worker processes (§17).

### Outbound HTTP with `httpx`

The `requests` library is synchronous and will block an async route. Use **`httpx`** with `AsyncClient` for outbound calls. **Create one client per app (reuse the connection pool)**, not one per request — opening a fresh connection pool every request is slow and leaks. Store it on app state via the lifespan (§16).

### Lifespan: startup and shutdown

The **lifespan** context manager runs setup code before the server accepts requests and teardown after it stops — the place to create the DB engine, the httpx client, Redis pool, and to dispose them cleanly. (The old `@app.on_event("startup")` decorators are deprecated; use lifespan.)

```python
from contextlib import asynccontextmanager
import httpx
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # ---- startup ----
    app.state.http = httpx.AsyncClient(timeout=10)   # one shared client
    yield
    # ---- shutdown ----
    await app.state.http.aclose()                    # graceful cleanup

app = FastAPI(lifespan=lifespan)
```

⚡ **Version note:** On Linux, installing `uvloop` (pulled in by `fastapi[standard]` on supported platforms) replaces the default asyncio loop with a faster one. `uvloop` is **not available on Windows** — Windows uses the standard event loop, which is fine for development; production typically runs on Linux anyway.

---

## 9. Databases — SQLAlchemy 2.0, SQLModel & Alembic

FastAPI ships no ORM — you bring one. The two dominant choices in 2026 are **SQLAlchemy 2.0** (the most powerful and standard) and **SQLModel** (a thin, ergonomic layer by FastAPI's author that fuses SQLAlchemy + Pydantic so one class is both a table and a schema). This section uses async SQLAlchemy 2.0 as the foundation, then shows SQLModel. For the database side of the story — schema design, indexing, transactions, query plans — see [PostgreSQL](POSTGRESQL_GUIDE.md), [SQLite3](SQLITE3_GUIDE.md), [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md), and [Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md).

### Async engine, session, and the session-per-request pattern

The **engine** owns a connection pool (created once, app-wide). A **session** is a short-lived unit of work that you create per request and dispose at its end — the **session-per-request** pattern, implemented with a `yield` dependency (§7). For async you need an async driver: **`asyncpg`** for PostgreSQL, **`aiosqlite`** for SQLite.

⚡ **Version note:** SQLAlchemy 2.0 unifies the modern API: `select()` queries, `Mapped[...]` / `mapped_column()` typed models, `DeclarativeBase`, and `AsyncSession`. Avoid 1.x patterns (`Query`, `Base = declarative_base()` without typing) in new code.

```python
# db.py
from collections.abc import AsyncGenerator
from typing import Annotated
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

# Engine = pooled connections, created ONCE for the whole app.
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/appdb",
    pool_size=10,          # steady-state pooled connections
    max_overflow=20,       # extra connections allowed under burst
    pool_pre_ping=True,    # check a connection is alive before using (avoids stale conns)
    echo=False,            # True logs all SQL (debug only)
)

# Session factory. expire_on_commit=False keeps objects usable after commit.
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

# Session-per-request dependency (open → use → commit/rollback → close).
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

DBSession = Annotated[AsyncSession, Depends(get_db)]
```

### Models

```python
# models.py
from datetime import datetime
from sqlalchemy import String, ForeignKey, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    hashed_password: Mapped[str] = mapped_column(String(255))
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    # One-to-many: a user has many posts. lazy="selectin" pre-loads to avoid N+1.
    posts: Mapped[list["Post"]] = relationship(back_populates="author", lazy="selectin")

class Post(Base):
    __tablename__ = "posts"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    author: Mapped["User"] = relationship(back_populates="posts")
```

### CRUD with the 2.0 `select()` API, mapped through DI

```python
from sqlalchemy import select
from fastapi import APIRouter, HTTPException

router = APIRouter(prefix="/users", tags=["users"])

@router.post("", status_code=201, response_model=UserPublic)
async def create_user(payload: UserCreate, db: DBSession):
    user = User(email=payload.email, hashed_password=hash_pw(payload.password))
    db.add(user)
    await db.flush()           # send INSERT, populate user.id, without ending the txn
    await db.refresh(user)     # reload server-defaults (created_at)
    return user                # commit happens in get_db's teardown

@router.get("/{user_id}", response_model=UserPublic)
async def get_user(user_id: int, db: DBSession):
    user = await db.get(User, user_id)            # PK lookup; returns None if absent
    if user is None:
        raise HTTPException(404, "User not found")
    return user

@router.get("", response_model=list[UserPublic])
async def list_users(db: DBSession, skip: int = 0, limit: int = 20):
    stmt = select(User).offset(skip).limit(limit).order_by(User.id)
    result = await db.execute(stmt)
    return result.scalars().all()   # .scalars() unwraps Row → User objects
```

`from_attributes=True` on the Pydantic response models (§4) lets FastAPI build them straight from ORM objects.

### Avoiding the N+1 problem

The classic ORM trap: fetch N users, then lazily load each user's posts → N+1 queries. The fix is **eager loading**: `selectinload`/`joinedload`, or set `lazy="selectin"` on the relationship. The Relational DB Design guide explains query plans and indexing that make these joins fast.

```python
from sqlalchemy.orm import selectinload

stmt = select(User).options(selectinload(User.posts))   # 2 queries total, not N+1
users = (await db.execute(stmt)).scalars().all()
```

### Migrations with Alembic

Your models define the *desired* schema; the database has the *current* schema. **Alembic** generates and applies versioned **migrations** to move from one to the other. Never edit a production schema by hand — migrations are reviewable, reversible, and replayable across environments.

```bash
alembic init -t async migrations         # scaffold an async migration env
# point migrations/env.py at your engine URL + Base.metadata, then:
alembic revision --autogenerate -m "create users and posts"   # diff models vs db
alembic upgrade head                     # apply migrations forward
alembic downgrade -1                     # roll back one revision
```

**Always review autogenerated migrations** — Alembic can't detect every change (renames, server-side defaults, some type changes) and may emit destructive operations. In production, run `alembic upgrade head` as a deploy step *before* the new app version starts.

### SQLModel — the ergonomic alternative

**SQLModel** lets one class serve as both the SQLAlchemy table and the Pydantic schema, cutting the boilerplate of parallel model hierarchies. It's built on SQLAlchemy 2.0 + Pydantic v2, so everything above still applies underneath.

```python
from sqlmodel import SQLModel, Field

class Hero(SQLModel, table=True):          # table=True → it's a DB table
    id: int | None = Field(default=None, primary_key=True)
    name: str = Field(index=True)
    secret_name: str
    age: int | None = None

# A non-table SQLModel doubles as a request/response schema (table omitted):
class HeroCreate(SQLModel):
    name: str
    secret_name: str
    age: int | None = None
```

**Choosing:** SQLModel is delightful for small/medium apps and rapid work. Reach for raw SQLAlchemy 2.0 when you need its full power (complex queries, hybrid properties, advanced relationship loading, fine-grained control) — which most large production systems eventually do.

---

## 10. Authentication & Authorization

**Authentication** = *who are you* (verify identity). **Authorization** = *what may you do* (check permissions). FastAPI provides the security plumbing (OAuth2 flows, bearer extraction, OpenAPI integration) but stays unopinionated about your user store. For the conceptual background — sessions vs tokens, OAuth2, password hashing, refresh flows — read [Better Auth](BETTERAUTH_GUIDE.md); this section is the FastAPI implementation.

### Sessions vs tokens

Two models dominate. **Session cookies**: the server stores session state (often in [Redis](REDIS_GUIDE.md)) and the client holds an opaque cookie — easy to revoke, great for browser apps, needs CSRF protection. **JWT (stateless tokens)**: the server signs a token containing claims; the client sends it as `Authorization: Bearer <token>`; the server verifies the signature without a lookup — scales horizontally with no shared session store, but is hard to revoke before expiry (mitigate with short lifetimes + refresh tokens + a deny-list). FastAPI's OAuth2 helpers target the token model, shown below.

### Password hashing

**Never store or compare plaintext passwords.** Hash with a slow, salted, memory-hard algorithm. **Argon2id** is the 2026 recommendation; **bcrypt** remains acceptable. Use `pwdlib` (the modern replacement) or `passlib`.

⚡ **Version note:** `pwdlib` is the maintained, modern hashing library recommended in 2026 (passlib's bcrypt backend hit compatibility friction with newer bcrypt releases). Argon2id is the preferred algorithm.

```python
from pwdlib import PasswordHash

password_hash = PasswordHash.recommended()   # Argon2id with sane params

def hash_pw(plain: str) -> str:
    return password_hash.hash(plain)

def verify_pw(plain: str, hashed: str) -> bool:
    return password_hash.verify(plain, hashed)
```

### OAuth2 password flow + JWT, the FastAPI way

`OAuth2PasswordBearer` declares the token URL (for the docs UI) and extracts the bearer token from the `Authorization` header, raising 401 if it's missing. You verify it in a `get_current_user` dependency that every protected route depends on.

```python
from datetime import datetime, timedelta, timezone
from typing import Annotated
import jwt                                  # PyJWT
from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm

SECRET_KEY = "load-this-from-settings"      # see §11 — NEVER hard-code in real code
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 15

app = FastAPI()
# tokenUrl is where clients POST credentials to get a token (used by /docs too).
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="auth/token")

def create_access_token(sub: str, scopes: list[str]) -> str:
    payload = {
        "sub": sub,                          # subject (user id/username)
        "scopes": scopes,
        "exp": datetime.now(timezone.utc) + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES),
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

@app.post("/auth/token")
async def login(form: Annotated[OAuth2PasswordRequestForm, Depends()], db: DBSession):
    # OAuth2PasswordRequestForm reads username/password from form data.
    user = await authenticate(db, form.username, form.password)   # your lookup + verify_pw
    if not user:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Bad credentials",
                            headers={"WWW-Authenticate": "Bearer"})
    token = create_access_token(sub=str(user.id), scopes=user.scopes)
    return {"access_token": token, "token_type": "bearer"}

async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)], db: DBSession,
):
    creds_exc = HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid token",
                              headers={"WWW-Authenticate": "Bearer"})
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])  # verifies sig + exp
    except jwt.PyJWTError:
        raise creds_exc
    user = await db.get(User, int(payload["sub"]))
    if user is None:
        raise creds_exc
    return user

CurrentUser = Annotated[User, Depends(get_current_user)]

@app.get("/me", response_model=UserPublic)
async def me(user: CurrentUser):           # any route gains auth just by adding this
    return user
```

### Scopes and role-based access (authorization)

For permissions, layer dependencies. A simple **role guard** is a dependency factory that checks the current user; OAuth2 **scopes** are FastAPI's first-class mechanism (`Security(get_current_user, scopes=[...])` + `SecurityScopes`).

```python
from fastapi import Depends

def require_role(role: str):
    async def checker(user: CurrentUser):
        if role not in user.roles:
            raise HTTPException(status.HTTP_403_FORBIDDEN, "Not enough permissions")
        return user
    return checker

@app.delete("/users/{user_id}")
async def delete_user(user_id: int, _: Annotated[User, Depends(require_role("admin"))]):
    ...
```

### API keys

For service-to-service calls, an **API key** (a header like `X-API-Key`) is simpler than full OAuth2. Validate it in a dependency (constant-time compare, look it up hashed in the DB, scope it). Use `fastapi.security.APIKeyHeader` so it appears in the docs.

**Security best practices:** short-lived access tokens (5–15 min) + refresh tokens; strong random `SECRET_KEY` from secrets (§11); always set `WWW-Authenticate: Bearer` on 401s; return identical errors for "user not found" and "bad password" (don't leak which); rate-limit the login endpoint (§19); hash API keys at rest; and prefer HttpOnly+Secure+SameSite cookies if you store tokens in the browser.

---

## 11. Validation, Settings & Config

Hard-coding configuration (database URLs, secret keys, feature flags) is a bug and a security hole. Configuration should come from the **environment** (12-factor style), be **typed and validated at startup** (so a missing or malformed value fails fast, loudly, before serving traffic), and keep secrets out of source control. **`pydantic-settings`** does exactly this: a `BaseSettings` subclass reads each field from an environment variable (or a `.env` file), validates and coerces it with Pydantic, and gives you a typed settings object.

```python
# settings.py
from functools import lru_cache
from pydantic import Field, PostgresDsn
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",            # also read a local .env (lowest precedence)
        env_file_encoding="utf-8",
        case_sensitive=False,       # APP_NAME matches app_name
        extra="ignore",
    )
    app_name: str = "My API"
    debug: bool = False
    database_url: PostgresDsn               # required + validated as a Postgres DSN
    secret_key: str = Field(min_length=32)  # required; enforce strength
    redis_url: str = "redis://localhost:6379/0"
    access_token_minutes: int = 15

@lru_cache                                  # build once, cache the singleton
def get_settings() -> Settings:
    return Settings()                       # raises at startup if anything is invalid
```

Inject settings as a dependency so they're easy to override in tests, and so the `lru_cache` ensures a single parse:

```python
from typing import Annotated
from fastapi import Depends

SettingsDep = Annotated[Settings, Depends(get_settings)]

@app.get("/info")
async def info(settings: SettingsDep):
    return {"app": settings.app_name, "debug": settings.debug}
```

**Secrets management:** keep `.env` out of git (`.gitignore` it) and provide a committed `.env.example` with dummy values. In production, inject real secrets via the orchestrator's secret store (Docker/Kubernetes secrets, a vault, or the platform's env-var mechanism) — see [Docker](DOCKER_GUIDE.md). `pydantic-settings` can also read Docker/`/run/secrets` files. Never log the settings object wholesale (it contains secrets); mark sensitive fields with `repr=False` or wrap them in `pydantic.SecretStr`.

---

## 12. Middleware & CORS

**Middleware** is code that wraps *every* request/response, running before routing on the way in and after your handler on the way out — the right place for cross-cutting concerns that aren't tied to one route: request logging/timing, adding response headers, GZip compression, CORS, trusted-host checks. It sits at the ASGI layer (Starlette), so it sees the raw request and the final response. Middleware runs in an **onion**: the first one added is the outermost (runs first on the way in, last on the way out).

⚡ **Version note:** Prefer pure-ASGI middleware classes (`add_middleware`) over the `@app.middleware("http")` decorator for performance-sensitive paths; the decorator form is convenient but slightly heavier. Both are valid.

```python
import time
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.gzip import GZipMiddleware

app = FastAPI()

# --- Built-in CORS: required for browsers to call your API from another origin ---
# CORS is enforced by the BROWSER, not your server — it asks "may JS from origin X
# read responses from origin Y?" You whitelist exactly the front-end origins.
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],   # NEVER ["*"] together with credentials
    allow_credentials=True,                       # allow cookies/Authorization
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
    max_age=600,                                  # cache the preflight for 10 min
)

# --- GZip: compress large responses to save bandwidth ---
app.add_middleware(GZipMiddleware, minimum_size=1000)   # only compress > 1 KB

# --- A custom timing/logging middleware (decorator form) ---
@app.middleware("http")
async def add_timing(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)          # let the rest of the stack handle it
    response.headers["X-Process-Time"] = f"{(time.perf_counter() - start)*1000:.1f}ms"
    return response
```

**CORS security:** `allow_origins=["*"]` is incompatible with `allow_credentials=True` (the browser rejects the combination) and is dangerous for authenticated APIs — list explicit origins. Do not reflect the request's `Origin` header back blindly. CORS is *not* an auth mechanism; it only governs browser cross-origin reads.

Other useful built-ins: `TrustedHostMiddleware` (reject requests with an unexpected `Host` header — blocks host-header attacks), and `HTTPSRedirectMiddleware`. Behind a reverse proxy ([Nginx](NGINX_GUIDE.md)), run Uvicorn with `--proxy-headers` so it trusts `X-Forwarded-For`/`X-Forwarded-Proto` and reports the real client IP and scheme (§17).

---

## 13. Background Tasks & Long Jobs

### `BackgroundTasks` — fire-and-forget *in the same process*

FastAPI's built-in **`BackgroundTasks`** runs a function *after the response is sent*, in the same process. It's perfect for quick, non-critical follow-ups — sending a confirmation email, writing an audit log, invalidating a cache — where the client shouldn't wait and you can tolerate the work being lost if the process dies.

```python
from fastapi import BackgroundTasks, FastAPI

app = FastAPI()

def send_welcome_email(address: str):
    ...                                  # runs AFTER the 200 is returned

@app.post("/signup")
async def signup(email: str, bg: BackgroundTasks):
    bg.add_task(send_welcome_email, email)    # queued; response returns immediately
    return {"status": "ok"}
```

**Its limits, and when you've outgrown it:** `BackgroundTasks` has no retries, no persistence, no scheduling, no separate scaling, and dies with the process. The work also still runs on *this* server's resources. Use it only for short, best-effort tasks.

### Real task queues — Celery, ARQ, Dramatiq

For anything durable — work that must survive restarts, retry on failure, run on dedicated worker machines, be scheduled, or take more than a moment — offload to a **task queue**. The API process enqueues a message into a broker (usually [Redis](REDIS_GUIDE.md), sometimes RabbitMQ) and returns immediately; separate **worker** processes consume and execute the jobs.

| Queue | Style | Notes |
|-------|-------|-------|
| **ARQ** | async-native, asyncio | Lightweight, Redis-based; pairs naturally with async FastAPI |
| **Celery** | mature, huge ecosystem | The industry standard; sync-rooted but battle-tested, rich scheduling (beat) |
| **Dramatiq** | simpler than Celery | Clean API, good retries/middleware, Redis or RabbitMQ |

```python
# ARQ example — async-friendly, fits FastAPI naturally.
# worker.py
from arq import create_pool
from arq.connections import RedisSettings

async def generate_report(ctx, user_id: int):   # the actual job, run by the worker
    ...                                          # heavy work here

class WorkerSettings:
    functions = [generate_report]
    redis_settings = RedisSettings(host="localhost", port=6379)

# In the API: enqueue and return a job id immediately.
from arq import create_pool
from arq.connections import RedisSettings

@app.post("/reports")
async def request_report(user_id: int):
    pool = await create_pool(RedisSettings())    # in real apps, hold this on app.state
    job = await pool.enqueue_job("generate_report", user_id)
    return {"job_id": job.job_id}                # client polls for the result later
```

**Scheduling** (cron-like recurring jobs): ARQ has cron jobs, Celery has *beat*, Dramatiq pairs with `apscheduler`. **Decide:** `BackgroundTasks` for trivial fire-and-forget; a real queue the moment durability, retries, scheduling, or independent scaling matter.

---

## 14. WebSockets & Streaming

### WebSockets — full-duplex, persistent connections

HTTP is request/response; **WebSockets** keep a single connection open for **bidirectional** real-time messaging — chat, live dashboards, notifications, collaborative editing, game state. FastAPI (via Starlette) exposes them with `@app.websocket`. You `accept()` the connection, then loop `receive_*`/`send_*` until the client disconnects (which raises `WebSocketDisconnect`).

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

class ConnectionManager:
    """Tracks active sockets so we can broadcast to all of them."""
    def __init__(self):
        self.active: list[WebSocket] = []
    async def connect(self, ws: WebSocket):
        await ws.accept()
        self.active.append(ws)
    def disconnect(self, ws: WebSocket):
        self.active.remove(ws)
    async def broadcast(self, message: str):
        for ws in self.active:
            await ws.send_text(message)

manager = ConnectionManager()

@app.websocket("/ws/{room}")
async def chat(ws: WebSocket, room: str):
    await manager.connect(ws)
    try:
        while True:
            text = await ws.receive_text()        # blocks until a message arrives
            await manager.broadcast(f"[{room}] {text}")
    except WebSocketDisconnect:
        manager.disconnect(ws)                     # clean up on disconnect
```

**Scaling WebSockets:** an in-memory `ConnectionManager` only knows about *this* process's connections. Across multiple workers/servers you need a shared backplane — typically [Redis](REDIS_GUIDE.md) pub/sub — so a message published on one node reaches subscribers connected to another. Authenticate the handshake (validate a token in the query string or first message), and apply per-connection rate limits.

### Server-Sent Events & `StreamingResponse`

When you only need **server→client** streaming (live logs, progress bars, token-by-token LLM output), **Server-Sent Events (SSE)** over plain HTTP are simpler than WebSockets — one-way, auto-reconnecting, and proxy-friendly. Implement SSE with a `StreamingResponse` yielding `data: ...\n\n` frames.

```python
import asyncio
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

@app.get("/events")
async def events():
    async def event_stream():
        for i in range(10):
            yield f"data: tick {i}\n\n"          # SSE frame format
            await asyncio.sleep(1)                # non-blocking pause
    return StreamingResponse(event_stream(), media_type="text/event-stream")
```

**Behind Nginx:** disable response buffering for streaming endpoints (`proxy_buffering off;` or the `X-Accel-Buffering: no` header) or the proxy will hold chunks back — see [Nginx](NGINX_GUIDE.md).

---

## 15. Testing

A fast, reliable test suite is what makes a FastAPI app *maintainable*. FastAPI's design — dependencies as the seam — makes testing unusually pleasant: you spin up the app in-process and **override dependencies** to swap the real DB/auth/clients for test doubles.

### `TestClient` and async `httpx.AsyncClient`

`TestClient` (Starlette, built on `httpx`) drives your app synchronously in-process — no running server needed — and is great for most route tests. For testing async code paths directly (async fixtures, concurrency), use `httpx.AsyncClient` with the ASGI transport plus `pytest-asyncio`.

```python
# test_users.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)                     # spins the app up in-process

def test_create_user():
    r = client.post("/users", json={"email": "a@b.com", "password": "longenough1"})
    assert r.status_code == 201
    assert r.json()["email"] == "a@b.com"
    assert "hashed_password" not in r.json()  # response_model must not leak it
```

```python
# Async style for async-only paths:
import pytest
from httpx import AsyncClient, ASGITransport
from main import app

@pytest.mark.asyncio
async def test_async():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        r = await ac.get("/me", headers={"Authorization": "Bearer test"})
    assert r.status_code == 200
```

### Dependency overrides — the killer feature

`app.dependency_overrides[real_dep] = fake_dep` replaces a dependency *everywhere it's used*, for the duration of a test. This is how you give tests a throwaway database, a fixed "current user", or a stub HTTP client — with no monkeypatching of internals.

```python
import pytest
from main import app, get_db, get_current_user

@pytest.fixture
def client_with_test_db(test_session):       # test_session yields a rolled-back session
    app.dependency_overrides[get_db] = lambda: test_session
    app.dependency_overrides[get_current_user] = lambda: FakeUser(id=1, roles=["admin"])
    yield TestClient(app)
    app.dependency_overrides.clear()         # always reset between tests
```

### Test database and isolation

Use a **separate test database** (or a SQLite in-memory DB, or — best — a disposable Postgres in a container, see [Docker](DOCKER_GUIDE.md)). The standard isolation trick: wrap each test in a transaction and **roll it back** afterward, so tests never see each other's writes and the DB is pristine every time. Build entities with **factory** helpers (`factory-boy` or simple functions) so each test declares only the fields it cares about. Run the suite with `pytest`, parametrize edge cases, and assert both happy paths and the 4xx/422 validation responses.

---

## 16. Project Structure & Maintainability at Scale

A single `main.py` is fine for a demo and unbearable at scale. Production FastAPI apps organize by **feature/domain** with clear layers, so that adding a feature touches one folder and the codebase stays navigable for a team.

### A scalable layout

```
app/
├── main.py                 # creates FastAPI(), lifespan, mounts routers, middleware
├── core/
│   ├── config.py           # pydantic-settings (§11)
│   ├── security.py         # hashing, JWT helpers (§10)
│   └── db.py               # engine, session factory, get_db (§9)
├── api/
│   ├── deps.py             # shared dependencies (get_current_user, pagination…)
│   └── v1/
│       ├── router.py       # aggregates all v1 routers under /api/v1
│       ├── users.py        # users APIRouter (HTTP only — thin)
│       └── posts.py
├── models/                 # SQLAlchemy ORM models (§9)
├── schemas/                # Pydantic request/response models (§4)
├── services/               # business logic (no HTTP, no FastAPI imports)
├── repositories/           # data-access functions (queries) — optional layer
└── tests/
```

### The layering rule (service / repository pattern)

Keep each layer single-purpose, and let dependencies point downward only:

- **Routers** are *thin*: parse input, call a service, shape the response. No business logic, no raw queries.
- **Services** hold business logic. They depend on repositories, raise *domain* exceptions (not `HTTPException`), and know nothing about HTTP — so the same logic is reusable from jobs, CLIs, and tests.
- **Repositories** own data access (the SQLAlchemy queries), hiding the ORM behind intention-revealing functions like `get_user_by_email`.

This separation is what keeps a large codebase testable (mock the repository to unit-test a service) and changeable (swap the data store without touching routers). It mirrors the architecture the [NestJS](NESTJS_GUIDE.md)/[Django](DJANGO_GUIDE.md) ecosystems formalize.

### Assembling the app and versioning the API

```python
# app/api/v1/router.py
from fastapi import APIRouter
from . import users, posts

api_router = APIRouter(prefix="/api/v1")     # version in the path → /api/v1/...
api_router.include_router(users.router)
api_router.include_router(posts.router)

# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.api.v1.router import api_router
from app.core.db import engine

@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    await engine.dispose()                   # close the pool on shutdown

app = FastAPI(title="My API", lifespan=lifespan)
app.include_router(api_router)
```

**Versioning** (`/api/v1`, `/api/v2`) lets you evolve the contract without breaking existing clients — keep old versions running while clients migrate. Within a version, additive changes (new optional fields, new endpoints) are safe; removals/renames are breaking and belong in a new version.

---

## 17. Production Deployment

Development (`fastapi dev` / `uvicorn --reload`) is not production. A production deployment needs **multiple worker processes** (to use all CPU cores — one async process uses one core), a **process manager**, a **reverse proxy** for TLS and static files, **containerization**, injected **secrets**, **healthchecks**, **observability**, and **graceful shutdown**.

### Workers: why and how

A single Uvicorn process runs one event loop on one core. To use an N-core box you run **N worker processes** (rule of thumb: `2 × cores` for mixed I/O, but measure). Options:

- **Uvicorn with `--workers`** — simplest: `uvicorn app.main:app --workers 4`.
- **Gunicorn managing Uvicorn workers** — the long-standing production pattern; Gunicorn is a robust process manager (restarts dead workers, handles signals): `gunicorn app.main:app -k uvicorn.workers.UvicornWorker -w 4`.
- **Granian** — a newer Rust-based ASGI server that bundles worker management and competitive performance; increasingly popular in 2026.

⚡ **Version note:** `fastapi run` (the FastAPI CLI) wraps Uvicorn with production-appropriate defaults and a `--workers` flag, and is the simplest official path. In containers, many teams run **one Uvicorn worker per container** and scale by running more containers (let Kubernetes/the orchestrator do the multiplying) — this keeps each container simple and observable.

### Behind Nginx (reverse proxy + TLS)

Don't expose Uvicorn directly to the internet. Put **[Nginx](NGINX_GUIDE.md)** (or another reverse proxy) in front to terminate **TLS** (HTTPS), serve static assets, buffer slow clients, enforce limits, and load-balance across workers/containers. Run Uvicorn with `--proxy-headers --forwarded-allow-ips="*"` (scoped appropriately) so it trusts the proxy's `X-Forwarded-*` headers and logs real client IPs/scheme. See the [Networking](NETWORKING_GUIDE.md) guide for what the proxy is doing at the TCP/TLS level.

```nginx
# nginx (see NGINX_GUIDE.md) — proxy to the app, terminate TLS upstream.
location / {
    proxy_pass http://127.0.0.1:8000;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

### Docker

Containerize for reproducible deploys. Use a slim Python base, install deps first (layer caching), copy code, run as a non-root user, and don't bake secrets into the image — inject them as env/secrets at runtime. See [Docker](DOCKER_GUIDE.md) for the full story.

```dockerfile
FROM python:3.13-slim
WORKDIR /app
# uv for fast, reproducible installs (lockfile copied first for cache hits):
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev
COPY . .
RUN useradd -m appuser && chown -R appuser /app
USER appuser                                   # never run as root
EXPOSE 8000
CMD ["uv", "run", "fastapi", "run", "app/main.py", "--host", "0.0.0.0", "--port", "8000"]
```

### Healthchecks, observability, graceful shutdown

- **Healthchecks:** expose a cheap `/health` (liveness — is the process up?) and a deeper `/ready` (readiness — can it reach the DB/Redis?). Orchestrators and load balancers poll these to route traffic and restart unhealthy pods.
- **Logging:** use **structured (JSON) logging** (`structlog`) so logs are queryable; attach a request-id per request (via middleware) for tracing a request across services.
- **Metrics:** expose **Prometheus** metrics (`prometheus-fastapi-instrumentator`) — request counts, latencies, error rates — and dashboard them.
- **Tracing:** instrument with **OpenTelemetry** to follow a request across services and into the DB; invaluable for debugging latency in distributed systems.
- **Graceful shutdown:** on SIGTERM, the server should stop accepting new connections, let in-flight requests finish, and close pools (your lifespan teardown). Give the orchestrator a sensible termination grace period so it doesn't kill mid-request.

```python
@app.get("/health")                            # liveness — cheap, no dependencies
async def health():
    return {"status": "ok"}

@app.get("/ready")                             # readiness — checks dependencies
async def ready(db: DBSession):
    await db.execute(text("SELECT 1"))         # confirm the DB is reachable
    return {"status": "ready"}
```

---

## 18. Performance & Scaling

FastAPI is fast by default, but throughput at scale comes from architecture, not micro-optimizations. The big levers:

- **Use async DB drivers end-to-end.** A sync driver inside an `async def` route blocks the loop and silently caps throughput (§8, §20). Async `asyncpg` + async SQLAlchemy keeps the loop free during DB waits.
- **Cache with [Redis](REDIS_GUIDE.md).** Put hot, expensive, read-mostly results (computed aggregates, rendered pages, third-party API responses, session lookups) in Redis with a TTL. A cache hit is microseconds vs. a multi-millisecond DB query. Cache-aside is the common pattern: check Redis → miss → query DB → store in Redis.
- **Paginate everything.** Never return an unbounded list — cap `limit`, and prefer **keyset (cursor) pagination** over `OFFSET` for large tables (OFFSET scans and discards rows, getting slower as the offset grows). See [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md).
- **Mind `response_model` cost.** Re-validating large responses through Pydantic has a price. For very large or hot payloads, consider a faster encoder (`ORJSONResponse`), or skip `response_model` validation where you fully trust the data (`response_model_exclude_*` still costs).
- **Rate limit** to protect capacity (§19) — `slowapi` or a Redis token-bucket.
- **Scale horizontally.** Because JWT-auth APIs are stateless, you add capacity by running more workers/containers behind a load balancer; keep shared state (sessions, caches, rate-limit counters) in Redis so any node can serve any request.
- **Profile before optimizing.** Measure with `py-spy` (sampling profiler, no code changes), load-test with `locust`/`k6`, and watch the metrics from §17. Optimize the actual bottleneck — usually the database, rarely Python itself.

```python
# Cache-aside with Redis (redis.asyncio): see REDIS_GUIDE.md.
import json
from redis.asyncio import Redis

redis = Redis.from_url("redis://localhost:6379/0", decode_responses=True)

async def get_dashboard(user_id: int, db: DBSession) -> dict:
    key = f"dash:{user_id}"
    if cached := await redis.get(key):
        return json.loads(cached)              # cache HIT — fast path
    data = await expensive_query(db, user_id)  # cache MISS — compute
    await redis.set(key, json.dumps(data), ex=60)   # store with 60s TTL
    return data
```

---

## 19. Security Hardening

Security is layered; no single control is enough. A production FastAPI API should address each of these, most of which appear elsewhere in this guide:

- **Input validation (you mostly get this free).** Pydantic validates types, bounds, and formats at the edge — lean on it. Set `extra="forbid"` on request models to reject unexpected fields. Cap string/collection sizes and upload sizes. Treat *every* client value as hostile.
- **SQL injection** is prevented by using the ORM / parameterized queries (never f-string SQL). SQLAlchemy parameterizes by default; the danger is only raw string SQL you build yourself.
- **Authentication & authorization** done right (§10): hashed passwords (Argon2id), short-lived signed tokens, strong secret keys, real authorization checks on *every* protected route, identical errors that don't leak which credential was wrong.
- **CORS** locked to explicit origins, never `*` with credentials (§12).
- **Rate limiting** on auth and expensive endpoints to blunt brute-force and abuse (Redis-backed so it works across workers).
- **Secrets** out of code and images, injected at runtime, never logged (§11).
- **HTTPS everywhere**, terminated at the proxy (§17); add **security headers** (HSTS, `X-Content-Type-Options: nosniff`, a `Content-Security-Policy` for any served HTML) via middleware; use `TrustedHostMiddleware`.
- **Dependency security:** pin versions with a lockfile, and scan regularly (`pip-audit`, `uv`'s audit, GitHub Dependabot) — most real-world breaches come through a vulnerable transitive dependency.
- **Don't leak internals:** disable `/docs` if the API is private; never return stack traces (set `debug=False` in production); shape errors through handlers (§6) so exception text doesn't reach clients.
- **Limit request size** at the proxy (`client_max_body_size` in Nginx) to stop memory-exhaustion uploads.

```python
# Minimal Redis-backed rate limit via slowapi.
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address, storage_uri="redis://localhost:6379")
app.state.limiter = limiter

@app.post("/auth/token")
@limiter.limit("5/minute")                     # blunt brute-force on login
async def login(request: Request, ...):
    ...
```

---

## 20. Gotchas & Best Practices

| Pitfall | Why it bites | Do this instead |
|---------|-------------|-----------------|
| **Blocking call inside `async def`** | `time.sleep`, `requests`, or a sync DB driver freezes the *whole* event loop → throughput collapses under load | Use async libs (`httpx`, `asyncpg`); or make the route plain `def`; or `await anyio.to_thread.run_sync(...)` |
| **Mutable default argument / field** | A shared `[]` or `{}` default leaks state across calls | Pydantic: `Field(default_factory=list)`; functions: default `None`, build inside |
| **`response_model` leak** | Returning the raw ORM/dict object exposes secrets (hashed_password, internal flags) | Always set a `response_model` that excludes sensitive fields (§4); test that they're absent |
| **Pydantic v1 habits** | `.dict()`, `class Config`, `@validator`, `orm_mode` are gone/renamed in v2 | Use `.model_dump()`, `model_config = ConfigDict(...)`, `@field_validator`, `from_attributes=True` |
| **One DB session per app** | Sessions aren't concurrency-safe; sharing one corrupts state | Session-per-request via a `yield` dependency (§9) |
| **N+1 queries** | Lazy-loading relationships in a loop fires a query per row | Eager-load with `selectinload`/`joinedload` or `lazy="selectin"` (§9) |
| **Forgetting to `await`** | Calling an async function without `await` returns a coroutine, not the value (often "silent") | Await every coroutine; enable a linter rule for un-awaited coroutines |
| **`--reload` / `debug=True` in prod** | Reloader overhead, leaked tracebacks, security exposure | `fastapi run` / no reload; `debug=False`; disable `/docs` if private |
| **New `httpx.AsyncClient` per request** | Rebuilds the connection pool every call → slow, leaks | One client created in lifespan, stored on `app.state`, reused (§8) |
| **CPU-bound work in a route** | Pins the event loop / a worker; latency for everyone | Offload to a thread/process pool or a task queue (§13) |
| **Trusting `UploadFile.filename`** | Path traversal, content-type spoofing | Generate your own name, validate size/type server-side (§5) |
| **`allow_origins=["*"]` with credentials** | Browser rejects it; or you open the API to any site | List explicit origins (§12) |
| **Catching nothing in `yield` deps** | A handler error leaves the session uncommitted/unclosed | `try/except/finally` around the `yield` (commit/rollback/close) |

**Top best practices, distilled:** separate input/output Pydantic models; keep routers thin and push logic into services; one session per request via DI; async drivers end-to-end; settings from the environment, validated at startup; explicit `response_model` everywhere; lock CORS down; short-lived hashed-credential auth on every protected route; structured logs + healthchecks + metrics in production; and a test suite that overrides dependencies and rolls back a real test DB.

---

## 21. Study Path & Build-to-Learn Projects

**Suggested order.** Sections 1–3 give you a running, documented API. Then §4–§6 (Pydantic, requests, responses/errors) make it correct and safe. §7 (dependencies) and §8 (async) are the conceptual heart — invest here. §9–§11 (DB, auth, settings) turn it into a real application. §12–§16 add the cross-cutting and structural pieces. §17–§19 are about shipping and surviving production. Revisit §20 whenever something behaves strangely.

If your Python fundamentals (type hints, `async`/`await`, decorators, context managers) feel shaky at any point, detour to the [Python](PYTHON_GUIDE.md) guide — FastAPI rewards a solid grasp of all of them.

**Build-to-learn projects, in increasing difficulty:**

1. **Typed CRUD micro-API (B).** A `/notes` resource with create/read/update/delete, Pydantic models, query-param pagination, proper status codes, and the auto docs. Goal: internalize path operations, models, and `response_model`.

2. **CRUD API with auth + Postgres (B/I).** Add async SQLAlchemy 2.0 + [PostgreSQL](POSTGRESQL_GUIDE.md), Alembic migrations, session-per-request DI, user signup/login with Argon2 hashing and JWTs, and route protection via `get_current_user`. Goal: the full request→DB→response loop with real auth. (Compare with how [Django](DJANGO_GUIDE.md) does the same.)

3. **Service/repository refactor (I/A).** Restructure project 2 into the layered layout of §16: thin routers, a service layer raising domain exceptions mapped by exception handlers, repositories hiding queries, `pydantic-settings` config, and a `pytest` suite using dependency overrides + a rolled-back test DB. Goal: maintainability and testability at scale.

4. **Real-time endpoint (I/A).** Add a WebSocket chat or an SSE live-progress endpoint, with [Redis](REDIS_GUIDE.md) pub/sub as the backplane so it works across multiple workers, plus handshake authentication. Goal: streaming and horizontal scale of stateful connections.

5. **Job-queue-backed API (A).** Add a "generate report / process upload" endpoint that enqueues to **ARQ/Celery** on Redis and returns a job id the client polls; add a scheduled cleanup job. Goal: offloading long work and decoupling the API from workers.

6. **Production-harden and deploy (A).** Containerize with [Docker](DOCKER_GUIDE.md), run multiple workers behind [Nginx](NGINX_GUIDE.md) with TLS, inject secrets at runtime, add `/health` + `/ready`, structured logging, Prometheus metrics, OpenTelemetry tracing, rate limiting, security headers, and graceful shutdown. Load-test with `locust`, profile with `py-spy`, and fix the real bottleneck. Goal: ship something that survives real traffic.

Build each on the previous one. By project 6 you'll have, end to end, a production-grade, scalable, maintainable async FastAPI service — which is exactly the destination this guide promised.
