# Django — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** A developer who already knows **basic Python** (variables, functions, classes, decorators, comprehensions, virtual environments, `pip`) and wants to become a confident, production-grade **Django** developer — the "batteries-included" Python web framework that powers Instagram, Mozilla, and tens of thousands of businesses. If your Python is rusty, read the [Python](PYTHON_GUIDE.md) guide first; this guide *assumes* you can read a `class`, a decorator, a context manager, and a `with` block. Everything here is explained **prose-first**: for each concept you get *what it is, why it exists, when and how to use it, the key options/parameters, best practices, and the security implications* — and only **then** the heavily-commented, runnable code. Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Django 5.2 LTS** (released **April 2025**) as the safe production default, with notes on **Django 6.0** (released **December 2025**, the newest), running on **Python 3.12–3.14**, accurate as of **June 2026**. **Django REST Framework (DRF)** is used for APIs throughout. Key facts the guide is built on:
> - Django ships a **new feature release roughly every 8 months** (e.g. 5.0, 5.1, 5.2, 6.0). **Every fourth release is an LTS** (Long-Term Support, ~3 years of fixes). **5.2 is the current LTS** — it is the version you should run in production unless you have a specific reason not to. **4.2 LTS** support ends April 2026; **5.2 LTS** is supported until ~April 2028.
> - **Django 6.0** (Dec 2025) is the newest feature release but is **not** an LTS; the next LTS will be **6.2** (~April 2027). Production teams generally stay on the latest LTS (5.2) and only chase non-LTS releases for specific new features. Where 6.0 changes something, it is flagged with **⚡ Version note** — no invented 6.x-only APIs appear here.
> - **Python support:** Django 5.2 supports Python **3.10–3.13**; Django 6.0 drops 3.10/3.11 and supports **3.12–3.14**. Run the newest Python your Django version supports.
> - Modern Django emphasized throughout: the **`STORAGES`** setting (replaced `DEFAULT_FILE_STORAGE`/`STATICFILES_STORAGE` in 4.2), **async views and a growing async ORM**, the **built-in tasks framework** (new in 6.0, experimental), composite primary keys (5.2), and `django.contrib` batteries.
>
> Cross-references appear throughout to the [Python](PYTHON_GUIDE.md), [FastAPI](FASTAPI_GUIDE.md) (compared), [PostgreSQL](POSTGRESQL_GUIDE.md), [SQLite3](SQLITE3_GUIDE.md), [Redis](REDIS_GUIDE.md), [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md), [Networking](NETWORKING_GUIDE.md), [Nginx](NGINX_GUIDE.md), [Docker](DOCKER_GUIDE.md), and [Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md) guides. Authoritative source to confirm details offline-permitting: **docs.djangoproject.com/en/5.2/**.

---

## Table of Contents

1. [What Django Is & the "Batteries-Included" Philosophy — MTV, vs FastAPI/Flask, Project/App Structure, the Request Lifecycle](#1-what-django-is--the-batteries-included-philosophy--mtv-vs-fastapiflask-projectapp-structure-the-request-lifecycle) **[B]**
2. [Setup — venv/uv, startproject, manage.py, the settings.py Tour, runserver, startapp](#2-setup--venvuv-startproject-managepy-the-settingspy-tour-runserver-startapp) **[B]**
3. [URLs & Views — URLconf, path/re_path, FBVs vs CBVs, HttpRequest/HttpResponse, render, redirects](#3-urls--views--urlconf-pathre_path-fbvs-vs-cbvs-httprequesthttpresponse-render-redirects) **[B]**
4. [The Django ORM & Models — Fields, Migrations, the QuerySet API, Relationships, N+1, Aggregation, Transactions, Raw SQL](#4-the-django-orm--models--fields-migrations-the-queryset-api-relationships-n1-aggregation-transactions-raw-sql) **[B/I/A]**
5. [Migrations in Depth — How They Work, Data Migrations, Squashing, Production Strategy](#5-migrations-in-depth--how-they-work-data-migrations-squashing-production-strategy) **[I/A]**
6. [The Admin Site — Registering Models, ModelAdmin, the Killer Feature, Securing It](#6-the-admin-site--registering-models-modeladmin-the-killer-feature-securing-it) **[B/I]**
7. [Forms — Form/ModelForm, Validation, CSRF, Rendering, Formsets, File Uploads](#7-forms--formmodelform-validation-csrf-rendering-formsets-file-uploads) **[I]**
8. [Templates — the Django Template Language, Inheritance, Context, Tags/Filters, Static Files, DRF+SPA Alternative](#8-templates--the-django-template-language-inheritance-context-tagsfilters-static-files-drfspa-alternative) **[B/I]**
9. [Authentication & Authorization — Users, login_required, Permissions/Groups, Password Hashing, Custom User Model, Sessions](#9-authentication--authorization--users-login_required-permissionsgroups-password-hashing-custom-user-model-sessions) **[I/A]**
10. [Settings, Config & the 12-Factor App — Splitting Settings, env vars, Secrets, STORAGES, the Security Checklist](#10-settings-config--the-12-factor-app--splitting-settings-env-vars-secrets-storages-the-security-checklist) **[I/A]**
11. [Middleware — the Request/Response Chain, Writing Custom Middleware, Built-ins](#11-middleware--the-requestresponse-chain-writing-custom-middleware-built-ins) **[I]**
12. [Django REST Framework — Serializers, ViewSets/Routers, Generic Views, Auth (Token/JWT/Session), Permissions, Pagination, Filtering, Throttling, Versioning](#12-django-rest-framework--serializers-viewsetsrouters-generic-views-auth-tokenjwtsession-permissions-pagination-filtering-throttling-versioning) **[I/A]**
13. [Async Django — Async Views, the Async ORM, ASGI, Channels for WebSockets](#13-async-django--async-views-the-async-orm-asgi-channels-for-websockets) **[A]**
14. [Caching — the Cache Framework, Redis Backend, Per-View/Fragment/Low-Level, Invalidation](#14-caching--the-cache-framework-redis-backend-per-viewfragmentlow-level-invalidation) **[I/A]**
15. [Background Tasks — Celery (+ Redis broker), Beat Scheduling, the Built-in Tasks Framework](#15-background-tasks--celery--redis-broker-beat-scheduling-the-built-in-tasks-framework) **[A]**
16. [Testing — TestCase/pytest-django, the Test Client, factory_boy, Fixtures, Coverage](#16-testing--testcasepytest-django-the-test-client-factory_boy-fixtures-coverage) **[A]**
17. [Project Structure & Maintainability at Scale — App Boundaries, Fat Models/Thin Views/Services, Settings Layout, Reusable Apps](#17-project-structure--maintainability-at-scale--app-boundaries-fat-modelsthin-viewsservices-settings-layout-reusable-apps) **[A]**
18. [Production Deployment — Gunicorn/Uvicorn behind Nginx, collectstatic + WhiteNoise/CDN, Docker, the Deployment Checklist, Pooling, Logging](#18-production-deployment--gunicornuvicorn-behind-nginx-collectstatic--whitenoisecdn-docker-the-deployment-checklist-pooling-logging) **[A]**
19. [Security — Built-in Protections, SECURE_* Settings, HTTPS/HSTS, Secrets, check --deploy, Dependencies](#19-security--built-in-protections-secure_-settings-httpshsts-secrets-check---deploy-dependencies) **[A]**
20. [Performance & Scaling — ORM Optimization, Indexing, Caching, Pagination, Pooling, Horizontal Scaling, Profiling](#20-performance--scaling--orm-optimization-indexing-caching-pagination-pooling-horizontal-scaling-profiling) **[A]**
21. [Gotchas & Best Practices](#21-gotchas--best-practices) **[I/A]**
22. [Study Path & Build-to-Learn Projects](#22-study-path--build-to-learn-projects)

---

## 1. What Django Is & the "Batteries-Included" Philosophy — MTV, vs FastAPI/Flask, Project/App Structure, the Request Lifecycle

### 1.1 The one-paragraph definition **[B]**

**Django is a full-stack, "batteries-included" Python web framework** that gives you a complete, opinionated foundation for building web applications and APIs. Where Python's standard library gives you sockets and an HTTP server you'd have to assemble by hand, Django hands you a finished architecture: a **URL router** that maps requests to your code, a powerful **ORM** so you talk to your database in Python objects instead of SQL strings, a **migrations** system that versions your schema, a **template engine** for HTML, a **forms** layer that validates and renders user input, a **session** framework, an industrial-strength **authentication** system, and — its signature feature — a **fully automatic admin interface** generated from your models. With **Django REST Framework (DRF)** layered on top, the same models become a clean REST API in a handful of lines. The selling point is **"the web framework for perfectionists with deadlines"**: the tedious, security-sensitive parts of web development (auth, CSRF, SQL-injection-safe queries, schema migrations, an admin CRUD UI) are *already solved, correctly, in the box*.

Django was created at the **Lawrence Journal-World** newspaper in 2003–2005 and open-sourced in 2005. It is maintained by the **Django Software Foundation** and is, in 2026, one of the most-used and most-respected backend frameworks in any language — chosen specifically when teams want **maturity, security, and a complete toolkit** rather than assembling a stack from micro-libraries.

### 1.2 MTV — Django's spin on MVC **[B]**

You will hear Django described as **MTV — Model, Template, View** — which is Django's slightly-renamed take on the classic **MVC** (Model-View-Controller) pattern. The names confuse newcomers, so pin them down:

- **Model** — a Python class that represents a database table / business entity (e.g. `class Post(models.Model)`). It knows its fields, how to query and save itself, and how it relates to other models. This is your *data layer*.
- **Template** — the presentation layer: an HTML file (`templates/blog/post_detail.html`) with placeholders that get filled with data. For APIs you skip templates and emit JSON via DRF serializers instead. This is the **"V" (View)** of traditional MVC.
- **View** — in Django, a **view is a Python function or class that takes a request and returns a response.** It's the *glue / controller*: it receives the `HttpRequest`, asks models for data, and returns an `HttpResponse` (often by rendering a template or serializing to JSON). This is the **"C" (Controller)** of traditional MVC.

So the mapping is: Django **Model** = MVC Model; Django **Template** = MVC View; Django **View** = MVC Controller. Django's own answer to the naming complaint is that "the framework itself is the controller" — it's the URL dispatcher that decides which view runs.

### 1.3 The philosophy — batteries included, DRY, explicit, secure-by-default **[B]**

Four design principles explain almost every Django decision:

- **Batteries included.** Auth, admin, sessions, ORM, migrations, forms, CSRF protection, caching, i18n, an email layer, security middleware — all shipped in the box (`django.contrib.*`). You add third-party packages for *extra* needs (DRF, Celery, Channels), but the core never makes you go shopping for the basics.
- **DRY (Don't Repeat Yourself).** You define a model **once** and Django derives the database schema, the admin UI, model forms, and serializers from it. Change the model, regenerate the rest.
- **Explicit is better than implicit** (it's Python, after all). Django avoids "magic" where it can — URLs are listed explicitly, settings are a plain module, middleware is an ordered list you can see.
- **Secure by default.** This is Django's quiet superpower. The ORM parameterizes queries (no SQL injection), templates auto-escape HTML (no XSS), CSRF protection is on by default, clickjacking protection ships as middleware, passwords are hashed with a strong algorithm. You have to go out of your way to write something insecure. Section 19 details all of this.

### 1.4 Django vs FastAPI vs Flask — when to pick which **[B]**

| | **Django** | **FastAPI** | **Flask** |
|---|---|---|---|
| **Shape** | Full-stack, batteries-included | Async API microframework | Minimal microframework |
| **Best at** | Complete apps + admin + APIs; large teams; CRUD-heavy products | High-throughput async JSON APIs; type-driven; auto OpenAPI docs | Tiny services, total control, learning |
| **ORM** | Built-in Django ORM | None (bring SQLAlchemy/SQLModel) | None (bring SQLAlchemy) |
| **Admin** | **Automatic, full-featured** | None | None |
| **Auth** | Full system in the box | DIY (or third-party) | DIY (extensions) |
| **Async** | Async views + growing async ORM | Async-first, native | WSGI (async bolted on) |
| **Migrations** | Built-in | Bring Alembic | Bring Alembic |
| **Validation** | Forms / DRF serializers | Pydantic (native) | DIY |
| **Sweet spot** | Content sites, SaaS, internal tools, anything needing an admin | Pure data APIs, ML model serving, microservices | Small apps, glue services |

The honest framing: **FastAPI** (see [FastAPI](FASTAPI_GUIDE.md)) wins when you want a lean, async-first, type-hinted JSON API and you're happy to assemble the ORM/migrations/auth yourself; its automatic OpenAPI docs and Pydantic validation are excellent. **Django + DRF** wins when you want a *complete* product fast — the admin alone often saves weeks — and benefits enormously from the security-by-default posture on a large team. Many shops run **both**: Django for the main app and admin, FastAPI for a couple of hot async microservices. **Flask** is the minimalist's choice for very small services. This guide is about Django; where async throughput is the priority, Section 13 shows how far Django has come, and the FastAPI guide shows the alternative.

### 1.5 Project vs app — the structure that trips up everyone **[B]**

Django has a two-level structure that confuses every beginner, so internalize it now:

- A **project** is the whole deployable thing — your website. It holds settings, the root URL config, and the WSGI/ASGI entry points. You have **one project**.
- An **app** is a **self-contained, reusable module of functionality** *inside* the project — e.g. a `blog` app, an `accounts` app, a `payments` app. Each app has its own models, views, URLs, admin, templates, and migrations. A project is composed of **many apps**, and a well-written app can be lifted into another project. (`django.contrib.admin`, `django.contrib.auth` etc. are just apps that ship with Django.)

The mental model: **project = the assembly; apps = the components.** Section 2.7 and Section 17 go deep on drawing app boundaries well — it's the single biggest determinant of whether your codebase stays maintainable at scale.

### 1.6 The request/response lifecycle **[B]**

Understanding the path a request takes is the key to debugging Django. End to end:

1. A web server / WSGI-ASGI server (Gunicorn, Uvicorn) receives the HTTP request and hands it to Django's WSGI/ASGI application object.
2. The request passes **down** through the **middleware** chain (Section 11) — security, sessions, CSRF, auth, etc. — each layer able to inspect or short-circuit it.
3. The **URL resolver** matches `request.path` against your URLconf and selects a **view**, extracting any captured parameters.
4. The **view** runs: it reads the request, queries models via the **ORM**, and builds a response (rendering a template or serializing JSON).
5. The response travels back **up** through the middleware chain (now in reverse), where layers can modify headers, add caching, etc.
6. The server writes the `HttpResponse` back to the client.

```text
HTTP request
   │
   ▼
WSGI/ASGI server (Gunicorn / Uvicorn)
   │
   ▼
Middleware chain  ──►  (Security, Session, CSRF, Auth, ...)   [request phase, top→bottom]
   │
   ▼
URL resolver (urls.py)  ──►  picks a view + captures kwargs
   │
   ▼
View  ──►  ORM / business logic  ──►  builds HttpResponse
   │
   ▼
Middleware chain  ──►  (reverse order)                        [response phase, bottom→top]
   │
   ▼
HTTP response
```

Keep this picture in your head: nearly every "why is this happening?" in Django is answered by "where in this pipeline am I?"

---

## 2. Setup — venv/uv, startproject, manage.py, the settings.py Tour, runserver, startapp

### 2.1 Python, virtual environments, and uv **[B]**

Never install Django into your system Python. Each project gets an **isolated virtual environment** so its dependencies can't collide with another project's. The classic tool is the standard-library `venv`; the modern, dramatically faster tool is **`uv`** (a Rust-based installer/resolver that's become the 2026 default for many teams). Either is fine — `uv` is just faster and manages Python versions too.

```bash
# --- Classic approach: venv + pip ---------------------------------
python -m venv .venv                 # create an isolated environment in ./.venv
source .venv/bin/activate            # activate it (Linux/macOS)
# .venv\Scripts\activate             # Windows PowerShell equivalent
python -m pip install --upgrade pip
pip install "Django==5.2.*"          # pin to the 5.2 LTS line for production

# --- Modern approach: uv (recommended in 2026) --------------------
# uv creates the venv, resolves, and installs — far faster than pip.
uv venv                              # creates .venv using a managed Python
source .venv/bin/activate
uv pip install "Django==5.2.*"
# Or a full project workflow with a pyproject.toml + lockfile:
uv init myproject && cd myproject
uv add "Django==5.2.*"               # adds to pyproject + uv.lock, installs

# Confirm:
python -m django --version           # -> 5.2.x
```

> **Best practice:** **Pin versions** and commit a lockfile (`uv.lock`, or `pip freeze > requirements.txt`). Reproducible installs are non-negotiable for production. See the [Docker](DOCKER_GUIDE.md) guide for building deterministic images from a lockfile.

### 2.2 Creating the project **[B]**

`django-admin startproject` scaffolds a new project. A subtle but important choice is **where the settings package lives**. The default puts a nested config package inside an outer directory:

```bash
# Creates ./myproject/ containing manage.py and an inner config package "myproject/"
django-admin startproject myproject

# Cleaner, widely-preferred layout: dot means "use the current dir as the root",
# and we name the config package "config" so it's not confused with an app:
mkdir mysite && cd mysite
django-admin startproject config .
```

The resulting tree (using `config .`):

```text
mysite/
├── manage.py            # your project's command-line entry point
└── config/
    ├── __init__.py
    ├── settings.py      # all configuration (we'll split this later — Section 10)
    ├── urls.py          # the ROOT URLconf
    ├── asgi.py          # ASGI entry point (async servers)
    └── wsgi.py          # WSGI entry point (sync servers, Gunicorn)
```

### 2.3 `manage.py` — your command center **[B]**

`manage.py` is a thin wrapper around `django-admin` that knows which settings module to use (it sets `DJANGO_SETTINGS_MODULE` for you). Every administrative action goes through it.

```bash
python manage.py runserver          # start the development server (NOT for production)
python manage.py runserver 0.0.0.0:8000   # listen on all interfaces (e.g. in Docker)
python manage.py startapp blog      # scaffold a new app called "blog"
python manage.py makemigrations     # generate migration files from model changes
python manage.py migrate            # apply migrations to the database
python manage.py createsuperuser    # create an admin login
python manage.py shell              # an interactive Python shell with Django loaded
python manage.py dbshell            # open your database's native CLI (psql, sqlite3...)
python manage.py check              # run system checks (config sanity)
python manage.py check --deploy     # production-readiness audit (Section 19!)
python manage.py collectstatic      # gather static files for production (Section 18)
python manage.py test               # run the test suite (Section 16)
```

> ⚡ **Version note.** `runserver` is a **development-only** server: single-process-ish, auto-reloading, and explicitly *not* hardened for production. Never expose it to the internet. Production uses Gunicorn/Uvicorn (Section 18).

### 2.4 The `settings.py` tour **[B]**

`settings.py` is a plain Python module — Django imports it and reads module-level names as configuration. The defaults you'll see and the ones that matter most:

```python
# config/settings.py  (annotated tour of the important defaults)
from pathlib import Path

# BASE_DIR points at the project root (where manage.py lives). Build paths from it.
BASE_DIR = Path(__file__).resolve().parent.parent

# SECURITY: this key signs sessions, password-reset tokens, CSRF, etc.
# NEVER hard-code it in source for production — load from the environment (Section 10).
SECRET_KEY = "django-insecure-...replace-me..."

# DEBUG must be False in production. True leaks stack traces and disables some protections.
DEBUG = True

# Hosts/domains this site may serve. Empty is fine only while DEBUG=True.
ALLOWED_HOSTS = []

# INSTALLED_APPS: every app (yours + Django's + third-party) that is "switched on".
INSTALLED_APPS = [
    "django.contrib.admin",         # the admin site (Section 6)
    "django.contrib.auth",          # authentication system (Section 9)
    "django.contrib.contenttypes",  # generic relations / permissions plumbing
    "django.contrib.sessions",      # server-side sessions
    "django.contrib.messages",      # one-shot flash messages
    "django.contrib.staticfiles",   # static-file handling (Section 8/18)
    # --- your apps go here ---
    # "blog.apps.BlogConfig",
]

# MIDDLEWARE: the ordered request/response chain (Section 11). Order matters!
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]

ROOT_URLCONF = "config.urls"         # the module Django starts URL resolution from

# DATABASES: SQLite by default (great for dev). PostgreSQL for production (Section 18).
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": BASE_DIR / "db.sqlite3",
    }
}

# TEMPLATES: configures the template engine (Section 8).
TEMPLATES = [{
    "BACKEND": "django.template.backends.django.DjangoTemplates",
    "DIRS": [BASE_DIR / "templates"],   # project-level template dir
    "APP_DIRS": True,                   # also look in each app's templates/ folder
    "OPTIONS": {"context_processors": [
        "django.template.context_processors.request",
        "django.contrib.auth.context_processors.auth",
        "django.contrib.messages.context_processors.messages",
    ]},
}]

# Internationalization & time
LANGUAGE_CODE = "en-us"
TIME_ZONE = "UTC"
USE_I18N = True
USE_TZ = True                        # store datetimes as UTC; convert on display

# Static files (CSS/JS/images) — URL prefix and where collectstatic gathers them.
STATIC_URL = "static/"

# STORAGES: the modern (4.2+) unified file-storage config — replaces the old
# DEFAULT_FILE_STORAGE / STATICFILES_STORAGE settings (Section 10/18).
STORAGES = {
    "default": {"BACKEND": "django.core.files.storage.FileSystemStorage"},
    "staticfiles": {"BACKEND": "django.contrib.staticfiles.storage.StaticFilesStorage"},
}

# Default primary-key type for models that don't specify one.
DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"
```

> **Best practice (read this twice):** the three settings that cause the most production incidents are **`SECRET_KEY`** (must be secret + from env), **`DEBUG`** (must be `False`), and **`ALLOWED_HOSTS`** (must list your real domains). Section 10 shows the 12-factor way to handle all three.

### 2.5 Running it **[B]**

```bash
python manage.py migrate      # apply Django's built-in tables (auth, sessions, admin...)
python manage.py runserver    # http://127.0.0.1:8000/  -> the green welcome rocket
```

### 2.6 Creating an app **[B]**

```bash
python manage.py startapp blog
```

```text
blog/
├── __init__.py
├── admin.py        # register models with the admin (Section 6)
├── apps.py         # the app's config class
├── migrations/     # this app's schema migrations
│   └── __init__.py
├── models.py       # data models (Section 4)
├── tests.py        # tests (Section 16)
└── views.py        # views (Section 3)
```

Then **switch the app on** by adding it to `INSTALLED_APPS`. Use the `AppConfig` dotted path — it's the canonical form and lets you hook app startup:

```python
# config/settings.py
INSTALLED_APPS = [
    # ...django.contrib apps...
    "blog.apps.BlogConfig",   # equivalently "blog", but the AppConfig path is preferred
]
```

You'll typically also create `blog/urls.py` (Section 3) and a `blog/templates/blog/` folder (Section 8) yourself — `startapp` doesn't make those.

### 2.7 A sane starting project layout **[B]**

A layout that scales (expanded in Section 17):

```text
mysite/
├── manage.py
├── pyproject.toml / requirements.txt
├── config/                  # the project: settings, root urls, asgi/wsgi
│   ├── settings/            # split settings package (Section 10)
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── dev.py
│   │   └── prod.py
│   └── urls.py
├── apps/                    # all your apps under one package (optional but tidy)
│   ├── accounts/            # custom user model lives here — create it on day one!
│   ├── blog/
│   └── ...
├── templates/               # project-wide templates
├── static/                  # project-wide static source files
└── tests/                   # or per-app tests/
```

> **Best practice:** create your **custom user model and its `accounts` app *before your first migration*** (Section 9.6). Switching the user model later is one of the most painful migrations in Django. Do it on day one of every project.

---

## 3. URLs & Views — URLconf, path/re_path, FBVs vs CBVs, HttpRequest/HttpResponse, render, redirects

### 3.1 The URLconf — mapping URLs to views **[B]**

A **URLconf** is an ordered list of URL patterns. Django walks it top-to-bottom and runs the first pattern whose path matches `request.path`. The project has a **root URLconf** (`config/urls.py`); the convention is to keep it thin and **`include()`** each app's own `urls.py`, so apps stay self-contained.

```python
# config/urls.py — the root URLconf
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),          # the admin site
    path("blog/", include("blog.urls")),      # delegate everything under /blog/ to the app
    path("", include("pages.urls")),          # a "home" app at the root
]
```

```python
# blog/urls.py — the app's own URLconf
from django.urls import path
from . import views

# app_name enables namespaced reversing: {% url 'blog:detail' pk=1 %}
app_name = "blog"

urlpatterns = [
    path("", views.post_list, name="list"),                 # /blog/
    path("<int:pk>/", views.post_detail, name="detail"),    # /blog/42/
    path("<slug:slug>/", views.post_by_slug, name="by-slug"),
]
```

### 3.2 `path()` converters and `re_path()` **[B]**

`path()` uses readable **converters** in angle brackets. They both **match** a URL segment and **convert** it to a Python type before passing it to your view:

| Converter | Matches | Python type | Example |
|---|---|---|---|
| `str` (default) | any non-empty, no `/` | `str` | `<str:username>` |
| `int` | digits | `int` | `<int:pk>` |
| `slug` | letters/numbers/hyphens/underscores | `str` | `<slug:slug>` |
| `uuid` | a UUID | `uuid.UUID` | `<uuid:id>` |
| `path` | any, **including `/`** | `str` | `<path:filepath>` |

For anything more complex, `re_path()` uses a full regular expression with **named groups**:

```python
from django.urls import path, re_path

urlpatterns = [
    # path() with a typed converter — preferred when it suffices:
    path("articles/<int:year>/<int:month>/", views.archive),
    # re_path() for custom patterns — note the named group (?P<year>...):
    re_path(r"^reports/(?P<year>[0-9]{4})/$", views.report),
]
```

> **Best practice:** **always name your URLs** (`name="..."`) and **reverse** them rather than hard-coding paths. Use `reverse("blog:detail", kwargs={"pk": 1})` in Python and `{% url 'blog:detail' pk=1 %}` in templates. If a URL changes, you change it in one place.

### 3.3 The `HttpRequest` and `HttpResponse` **[B]**

Every view receives an `HttpRequest` as its first argument and must return an `HttpResponse` (or subclass). The request object carries everything about the incoming call:

| Attribute | What it holds |
|---|---|
| `request.method` | `"GET"`, `"POST"`, etc. |
| `request.GET` | query-string params (a `QueryDict`) |
| `request.POST` | form-encoded body params |
| `request.body` | raw request body (bytes) — use for JSON |
| `request.FILES` | uploaded files (Section 7.6) |
| `request.user` | the logged-in `User` (or `AnonymousUser`) — set by AuthenticationMiddleware |
| `request.headers` | a case-insensitive headers mapping |
| `request.COOKIES` | cookies dict |
| `request.session` | the session store (Section 9.8) |

### 3.4 Function-based views (FBVs) **[B]**

The simplest view is a function. It is explicit and easy to read — great for one-off logic.

```python
# blog/views.py
from django.http import HttpResponse, JsonResponse, HttpResponseNotFound
from django.shortcuts import render, get_object_or_404, redirect
from .models import Post

def post_list(request):
    # Query the DB via the ORM (Section 4). QuerySets are lazy — no query yet.
    posts = Post.objects.filter(published=True).order_by("-created_at")
    # render() = load template + fill context + return an HttpResponse. The shortcut
    # everyone uses for HTML views.
    return render(request, "blog/post_list.html", {"posts": posts})

def post_detail(request, pk):
    # get_object_or_404: fetch one row, or raise Http404 (clean 404 page) if missing.
    # Far better than try/except DoesNotExist everywhere.
    post = get_object_or_404(Post, pk=pk, published=True)
    return render(request, "blog/post_detail.html", {"post": post})

def api_ping(request):
    # JsonResponse serializes a dict to JSON with the right Content-Type.
    return JsonResponse({"status": "ok", "method": request.method})
```

> **Gotcha:** `redirect()` is the shortcut for HTTP redirects and accepts a model, a URL name, or a path: `redirect("blog:detail", pk=post.pk)` or `redirect(post)` (uses the model's `get_absolute_url()`). After a successful POST, **always redirect** (the Post/Redirect/Get pattern) so a browser refresh doesn't re-submit.

### 3.5 Class-based views (CBVs) and generic views **[B/I]**

**Class-based views** package view logic into classes you can subclass and configure. Their real value is the **generic views** Django ships: `ListView`, `DetailView`, `CreateView`, `UpdateView`, `DeleteView`, `FormView` implement the boilerplate of the common CRUD patterns, so you supply only the model, template, and a couple of attributes.

```python
# blog/views.py — the same two pages as CBVs
from django.views.generic import ListView, DetailView, CreateView
from django.urls import reverse_lazy
from .models import Post

class PostListView(ListView):
    model = Post                       # -> Post.objects.all() by default
    template_name = "blog/post_list.html"
    context_object_name = "posts"      # the variable name in the template
    paginate_by = 10                   # free pagination! (Section 8/20)

    def get_queryset(self):            # override to filter/optimize
        return Post.objects.filter(published=True).order_by("-created_at")

class PostDetailView(DetailView):
    model = Post                       # uses the <pk> or <slug> from the URL automatically
    template_name = "blog/post_detail.html"

class PostCreateView(CreateView):
    model = Post
    fields = ["title", "body"]         # auto-builds a ModelForm (Section 7)
    success_url = reverse_lazy("blog:list")   # where to go after a successful save
```

Wire CBVs into URLs with `.as_view()`:

```python
# blog/urls.py
path("", views.PostListView.as_view(), name="list"),
path("<int:pk>/", views.PostDetailView.as_view(), name="detail"),
path("new/", views.PostCreateView.as_view(), name="create"),
```

**When to use which:** reach for **generic CBVs** when your view is standard CRUD (huge time savings, free pagination/forms). Use **FBVs** when logic is custom, branchy, or doesn't fit a CRUD shape — they're more explicit and easier for a teammate to read. Many production codebases mix both freely. (For *APIs*, DRF's class-based ViewSets in Section 12 are the equivalent leverage.)

---

## 4. The Django ORM & Models — Fields, Migrations, the QuerySet API, Relationships, N+1, Aggregation, Transactions, Raw SQL

The ORM is the heart of Django and where you'll spend most of your time. Master it and the rest of Django falls into place.

### 4.1 Models and fields **[B]**

A **model** is a Python class subclassing `models.Model`; each class attribute is a **field** that maps to a database column. Django reads the model to generate both the schema (via migrations) and a rich Python API for querying.

```python
# blog/models.py
from django.db import models
from django.conf import settings   # to reference the (custom) user model — Section 9

class Category(models.Model):
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(unique=True)         # URL-safe short label

    class Meta:
        verbose_name_plural = "categories"       # fix admin pluralization
        ordering = ["name"]                       # default ordering for queries

    def __str__(self):                            # human-readable repr (admin, shell)
        return self.name


class Post(models.Model):
    class Status(models.TextChoices):             # enum-like choices, modern style
        DRAFT = "draft", "Draft"
        PUBLISHED = "published", "Published"

    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, unique_for_date="published_at")
    body = models.TextField()
    status = models.CharField(max_length=10, choices=Status.choices,
                              default=Status.DRAFT)
    # ForeignKey: many posts -> one author. on_delete is REQUIRED — it decides what
    # happens to posts when the author is deleted (Section 4.5).
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE,
                               related_name="posts")
    category = models.ForeignKey(Category, on_delete=models.SET_NULL,
                                 null=True, blank=True, related_name="posts")
    tags = models.ManyToManyField("Tag", blank=True, related_name="posts")
    created_at = models.DateTimeField(auto_now_add=True)   # set once on creation
    updated_at = models.DateTimeField(auto_now=True)       # updated on every save
    published_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        ordering = ["-created_at"]
        indexes = [models.Index(fields=["status", "-created_at"])]  # DB index (Section 20)

    def __str__(self):
        return self.title
```

**The most-used field types:**

| Field | Stores | Notes |
|---|---|---|
| `CharField(max_length=)` | short text | `max_length` required |
| `TextField` | long text | no length limit |
| `IntegerField` / `BigIntegerField` | integers | |
| `DecimalField(max_digits=, decimal_places=)` | exact decimals | **use for money**, never `FloatField` |
| `BooleanField` | true/false | |
| `DateField` / `DateTimeField` | dates/times | `auto_now`, `auto_now_add` |
| `EmailField`, `URLField`, `SlugField`, `UUIDField` | validated strings | |
| `JSONField` | JSON | native on PostgreSQL/SQLite |
| `FileField` / `ImageField` | uploads | stores a path; file goes to `STORAGES` (Section 7.6/18) |
| `ForeignKey`, `ManyToManyField`, `OneToOneField` | relationships | Section 4.5 |

> **`null` vs `blank`:** `null=True` is **database-level** (column allows `NULL`); `blank=True` is **validation-level** (forms allow empty). For text fields, the Django convention is to leave `null=False` and use `blank=True` (store `""` not `NULL`) to avoid two "empty" states. For non-text optional fields you usually need both.

### 4.2 Creating the schema — makemigrations & migrate **[B]**

Models don't touch the database until you migrate. **`makemigrations`** turns model changes into versioned migration files; **`migrate`** applies them.

```bash
python manage.py makemigrations blog   # writes blog/migrations/0001_initial.py
python manage.py migrate                # applies all unapplied migrations
python manage.py sqlmigrate blog 0001   # preview the SQL a migration will run (read-only)
python manage.py showmigrations         # which migrations are applied (X) or pending
```

Section 5 covers migrations in depth — for now: **change a model → `makemigrations` → `migrate`.**

### 4.3 The QuerySet API — querying without SQL **[B/I]**

You query through a model's **manager** (`Model.objects`), which returns **QuerySets**. The two facts that govern everything:

1. **QuerySets are lazy.** Building one (`.filter(...).exclude(...)`) hits the database **zero times**. The query runs only when you *evaluate* it — iterate it, `list()` it, slice with a step, call `len()`, `bool()`, `.count()`, `.first()`, etc. This is powerful (you can compose queries cheaply) and a footgun (Section 21).
2. **QuerySets are chainable and immutable** — each method returns a *new* QuerySet, so you can build queries up in pieces.

```python
from django.db.models import Q, F
from blog.models import Post

# --- Retrieval ----------------------------------------------------
Post.objects.all()                          # everything
Post.objects.filter(status="published")     # WHERE status = 'published'
Post.objects.exclude(status="draft")        # WHERE NOT ...
Post.objects.get(pk=1)                      # exactly one (raises if 0 or >1)
Post.objects.first(); Post.objects.last()   # one or None

# --- Field lookups (the __ syntax) --------------------------------
Post.objects.filter(title__icontains="django")       # case-insensitive LIKE
Post.objects.filter(created_at__year=2026)            # date parts
Post.objects.filter(author__username="ada")           # span a relationship!
Post.objects.filter(pk__in=[1, 2, 3])                 # IN (...)
Post.objects.filter(published_at__isnull=True)        # IS NULL
Post.objects.filter(views__gte=100)                   # >=  (also gt, lte, lt)

# --- Q objects: OR / NOT / complex boolean logic ------------------
Post.objects.filter(Q(status="published") | Q(author__is_staff=True))
Post.objects.filter(~Q(category__name="spam"))        # NOT

# --- F expressions: reference a column on the DB side -------------
# Atomic increment WITHOUT a race condition (no read-modify-write in Python):
Post.objects.filter(pk=1).update(views=F("views") + 1)
# Compare two columns:
Post.objects.filter(updated_at__gt=F("created_at"))

# --- Ordering, slicing, distinct, values --------------------------
Post.objects.order_by("-created_at")[:10]             # LIMIT 10 (slicing = SQL LIMIT)
Post.objects.values("id", "title")                    # dicts instead of model instances
Post.objects.values_list("id", flat=True)             # a flat list of ids
Post.objects.distinct()
```

| Common lookup | Meaning |
|---|---|
| `__exact`, `__iexact` | `=`, case-insensitive `=` |
| `__contains`, `__icontains` | `LIKE %x%` |
| `__startswith`, `__endswith` | prefix/suffix |
| `__gt`, `__gte`, `__lt`, `__lte` | comparisons |
| `__in` | `IN (...)` |
| `__range` | `BETWEEN a AND b` |
| `__isnull` | `IS [NOT] NULL` |
| `__year`, `__month`, `__day`, `__date` | datetime parts |
| `__regex`, `__iregex` | regex match |

### 4.4 Creating, updating, deleting **[B]**

```python
# Create
post = Post.objects.create(title="Hello", body="...", author=request.user)
# or: post = Post(title="Hello"); post.save()

# Update one instance
post.title = "Hello, world"
post.save(update_fields=["title"])     # only write the named columns (faster, safer)

# Bulk update (one UPDATE, no per-row save() / signals):
Post.objects.filter(status="draft").update(status="published")

# Bulk create (one INSERT for many rows):
Post.objects.bulk_create([Post(title=f"p{i}", author=u) for i in range(1000)])

# get_or_create / update_or_create: race-safe "find or make"
obj, created = Category.objects.get_or_create(slug="news", defaults={"name": "News"})

# Delete
post.delete()
Post.objects.filter(status="draft").delete()   # bulk delete
```

> **Gotcha:** `.update()` and `.delete()` on a QuerySet are **bulk SQL** operations — they do **not** call `Model.save()`/`delete()` or fire `pre_save`/`post_save` signals. If you depend on signals or custom `save()` logic, iterate and save individually (at a performance cost).

### 4.5 Relationships and related managers **[I]**

Three relationship fields cover the relational world (see [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md) for the modeling theory):

- **`ForeignKey`** — *many-to-one*. Many posts → one author. Requires **`on_delete`**.
- **`ManyToManyField`** — *many-to-many*. Posts ↔ tags. Django creates the join table.
- **`OneToOneField`** — *one-to-one*. Often a "profile" extending the user.

**`on_delete` options** (what happens to *this* row when the referenced row is deleted):

| Option | Effect |
|---|---|
| `CASCADE` | delete this row too |
| `PROTECT` | block the delete (raise `ProtectedError`) |
| `SET_NULL` | set the FK to `NULL` (requires `null=True`) |
| `SET_DEFAULT` / `SET(...)` | set to a default/computed value |
| `DO_NOTHING` | leave it (you handle integrity yourself) |

**Related managers** let you traverse relationships in both directions. The `related_name` you set becomes the reverse accessor:

```python
author = User.objects.get(username="ada")
author.posts.all()                 # reverse FK: all posts by this author (related_name)
author.posts.filter(status="published")

post = Post.objects.get(pk=1)
post.author                        # forward FK: the author (one DB query if not cached)
post.tags.all()                    # forward M2M
post.tags.add(tag1, tag2)          # manage M2M membership
post.tags.remove(tag1)
post.tags.set([tag2, tag3])        # replace the whole set

# One-to-one (e.g. a Profile):
user.profile                       # forward/reverse access by attribute name
```

### 4.6 The N+1 problem — `select_related` & `prefetch_related` **[I/A]**

This is the **single most important performance topic in Django** and the most common production-slowness cause. Because related objects load **lazily**, a loop that touches a relationship can fire one query *per row*:

```python
# THE N+1 TRAP: 1 query for the posts, then 1 MORE query per post for its author.
for post in Post.objects.all():        # 1 query
    print(post.author.username)        # +N queries (one per post!)  -> N+1 total
```

The two cures, matched to the relationship type:

- **`select_related("author")`** — for **ForeignKey / OneToOne** (the "to-one" side). Performs a **SQL JOIN**, pulling the related rows in the *same* query.
- **`prefetch_related("tags")`** — for **ManyToMany / reverse FK** (the "to-many" side). Runs **a second query** for all related objects and joins them in Python.

```python
# FIXED: one JOINed query — no per-row author lookups.
for post in Post.objects.select_related("author"):
    print(post.author.username)

# FIXED for to-many: 2 queries total regardless of how many posts/tags.
for post in Post.objects.prefetch_related("tags"):
    print([t.name for t in post.tags.all()])

# Combine and chain across relationships:
Post.objects.select_related("author", "category").prefetch_related("tags")
# Span FKs in select_related with __:
Comment.objects.select_related("post__author")
# Customize a prefetch with Prefetch():
from django.db.models import Prefetch
Post.objects.prefetch_related(
    Prefetch("comments", queryset=Comment.objects.filter(approved=True))
)
```

> **Best practice:** install **django-debug-toolbar** (Section 20) in development — it shows the query count per page so N+1 problems jump out. The discipline: any time a template or loop touches `obj.related_thing`, ask "did I `select_related`/`prefetch_related` that?"

### 4.7 Aggregation and annotation **[I/A]**

- **`aggregate()`** collapses a whole QuerySet to **summary values** (a dict).
- **`annotate()`** adds a **computed value to *each row*** (great for counts/sums per object) and works hand-in-hand with `values()` + `Count`/`Sum` for GROUP BY.

```python
from django.db.models import Count, Sum, Avg, Max, Min, F, Q

# aggregate -> one dict of summary stats over the whole queryset
Post.objects.aggregate(total=Count("id"), avg_len=Avg("views"))
# -> {"total": 412, "avg_len": 87.3}

# annotate -> attach a per-row computed field (here: number of comments per post)
posts = Post.objects.annotate(num_comments=Count("comments")).order_by("-num_comments")
for p in posts:
    print(p.title, p.num_comments)

# GROUP BY: values() before annotate() groups by those columns
Post.objects.values("author__username").annotate(n=Count("id"))
# -> [{"author__username": "ada", "n": 12}, ...]

# Conditional aggregation with filter=
Post.objects.aggregate(
    published=Count("id", filter=Q(status="published")),
    drafts=Count("id", filter=Q(status="draft")),
)
```

### 4.8 Transactions **[I/A]**

A **transaction** groups several writes so they all commit or all roll back — essential for any multi-step operation that must be all-or-nothing (e.g. transfer money: debit + credit). Django's primary tool is **`atomic()`**, usable as a decorator or context manager.

```python
from django.db import transaction

@transaction.atomic                       # the whole view runs in one transaction
def place_order(request):
    order = Order.objects.create(user=request.user)
    for item in cart:
        OrderLine.objects.create(order=order, product=item.product, qty=item.qty)
        # If anything below raises, EVERYTHING above rolls back atomically:
        item.product.decrement_stock(item.qty)

# As a context manager for a tighter scope:
def transfer(a, b, amount):
    with transaction.atomic():
        Account.objects.filter(pk=a).update(balance=F("balance") - amount)
        Account.objects.filter(pk=b).update(balance=F("balance") + amount)

# Run code only AFTER a successful commit (e.g. send email, enqueue a task):
def view(request):
    with transaction.atomic():
        order = Order.objects.create(...)
        transaction.on_commit(lambda: send_receipt_email.delay(order.id))
```

> **Best practice:** consider **`ATOMIC_REQUESTS = True`** in `DATABASES` to wrap *every* request in a transaction (simple correctness; small overhead). For lock-sensitive operations, combine `atomic()` with `select_for_update()` to take a row lock and prevent concurrent races.

### 4.9 Raw SQL — safely, when you must **[A]**

The ORM covers ~99% of needs, but for hand-tuned queries you have escape hatches. The **non-negotiable rule: never string-format user input into SQL** — always pass **parameters** so the driver escapes them (this is what protects you from SQL injection; see Section 19).

```python
# Model.objects.raw(): returns model instances from a raw SELECT.
# %s placeholders + a params list — NEVER an f-string with user input!
posts = Post.objects.raw(
    "SELECT * FROM blog_post WHERE status = %s ORDER BY created_at DESC",
    ["published"],
)

# For non-model queries, use a cursor (still parameterized):
from django.db import connection
with connection.cursor() as cur:
    cur.execute("SELECT author_id, COUNT(*) FROM blog_post GROUP BY author_id")
    rows = cur.fetchall()

# DANGER — DO NOT DO THIS (SQL injection):
#   cur.execute(f"SELECT * FROM blog_post WHERE title = '{user_input}'")
```

---

## 5. Migrations in Depth — How They Work, Data Migrations, Squashing, Production Strategy

### 5.1 What migrations are and how they work **[I]**

A **migration** is a Python file describing a **change to your schema** — create table, add column, add index, etc. Django keeps them **versioned and ordered per app** in `app/migrations/`, and records which ones have run in a `django_migrations` table. This gives you a **replayable history**: any clone of the repo can reach the same schema by running `migrate`. Migrations form a **dependency graph** (a migration can depend on another app's migration), and Django computes a safe execution order.

```bash
python manage.py makemigrations          # detect model changes -> new migration files
python manage.py migrate                 # apply pending migrations
python manage.py migrate blog 0003       # migrate a specific app to a specific version
python manage.py migrate blog 0002       # ...or migrate BACKWARD to undo (if reversible)
python manage.py showmigrations          # list applied/pending
python manage.py makemigrations --check  # CI: fail if models drift from migrations
```

A typical generated migration:

```python
# blog/migrations/0002_post_views.py
from django.db import migrations, models

class Migration(migrations.Migration):
    dependencies = [("blog", "0001_initial")]   # must run after 0001
    operations = [
        migrations.AddField(
            model_name="post",
            name="views",
            field=models.PositiveIntegerField(default=0),
        ),
    ]
```

### 5.2 Data migrations **[I/A]**

Schema migrations change *structure*; **data migrations** change or backfill *data* (e.g. populate a new column, normalize values). You write a `RunPython` operation with a forward and (ideally) reverse function. **Always use the historical models** Django provides via `apps.get_model` — not your current import — so the migration stays correct even as the model evolves.

```python
# blog/migrations/0003_backfill_slugs.py
from django.db import migrations
from django.utils.text import slugify

def populate_slugs(apps, schema_editor):
    Post = apps.get_model("blog", "Post")          # HISTORICAL model — important!
    for post in Post.objects.filter(slug=""):
        post.slug = slugify(post.title)
        post.save(update_fields=["slug"])

def reverse(apps, schema_editor):
    pass   # nothing to undo here; make reversible where you can

class Migration(migrations.Migration):
    dependencies = [("blog", "0002_post_slug")]
    operations = [migrations.RunPython(populate_slugs, reverse)]
```

### 5.3 Squashing and reset **[A]**

Over time an app accumulates dozens of migrations, slowing test setup. **Squashing** collapses a range into one optimized migration while keeping history valid:

```bash
python manage.py squashmigrations blog 0001 0042   # collapse 0001..0042 into one
```

After everyone has migrated past the squash point, the old migrations can be deleted and the squashed one marked as their replacement.

### 5.4 Production migration strategy **[A]**

Migrations are where careless deploys cause outages. The principles:

- **Make schema changes backward-compatible** so old and new code can run simultaneously during a rolling deploy. The pattern for renaming/removing a column is multi-step: (1) add the new column (nullable), deploy; (2) backfill + write to both, deploy; (3) switch reads to the new column, deploy; (4) drop the old column, deploy. Never "add NOT NULL column + use it" in one deploy with live traffic.
- **Avoid long table locks.** Adding an index or a `NOT NULL` column on a huge table can lock it. On PostgreSQL use `CREATE INDEX CONCURRENTLY` (Django offers `AddIndexConcurrently` in `django.contrib.postgres.operations`) and add columns nullable first. See [PostgreSQL](POSTGRESQL_GUIDE.md) and [Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md).
- **Run `migrate` as a discrete, gated deploy step**, before (or carefully alongside) starting new app instances — not from inside a web worker. Section 18 shows this in a Docker/entrypoint flow.
- **Test the rollback.** Keep data migrations reversible where feasible; back up the DB before risky migrations.
- **CI guard:** run `makemigrations --check --dry-run` in CI so a PR that changes models but forgets the migration fails the build.

---

## 6. The Admin Site — Registering Models, ModelAdmin, the Killer Feature, Securing It

### 6.1 Why the admin is Django's killer feature **[B]**

The **Django admin** is a **production-ready CRUD interface auto-generated from your models**. Register a model and you instantly get list views, search, filters, create/edit forms, validation, and permission checks — a back-office that would take weeks to build by hand. For internal tools, content management, and "let support edit this record" needs, it routinely saves whole sprints. It is *not* meant to be your customer-facing UI, but as a **staff/admin back office** it's unmatched among web frameworks.

```bash
python manage.py createsuperuser    # create a login, then visit /admin/
```

### 6.2 Registering models and `ModelAdmin` **[B/I]**

```python
# blog/admin.py
from django.contrib import admin
from .models import Post, Category, Tag

@admin.register(Post)                         # decorator registration (clean)
class PostAdmin(admin.ModelAdmin):
    # The change-list page:
    list_display = ("title", "author", "status", "created_at")  # columns
    list_filter = ("status", "created_at", "category")          # right-hand filters
    search_fields = ("title", "body")                           # search box (uses LIKE)
    date_hierarchy = "created_at"                               # date drill-down nav
    ordering = ("-created_at",)
    list_select_related = ("author", "category")                # avoid N+1 in the list!
    list_per_page = 50

    # The add/change form:
    prepopulated_fields = {"slug": ("title",)}                  # auto-slug from title (JS)
    autocomplete_fields = ("category",)                         # searchable FK widget
    filter_horizontal = ("tags",)                               # nicer M2M widget
    readonly_fields = ("created_at", "updated_at")
    fieldsets = (                                               # group form fields
        ("Content", {"fields": ("title", "slug", "body")}),
        ("Meta", {"fields": ("author", "category", "tags", "status")}),
    )

    # A custom bulk action:
    actions = ["make_published"]
    @admin.action(description="Mark selected posts as published")
    def make_published(self, request, queryset):
        updated = queryset.update(status="published")
        self.message_user(request, f"{updated} posts published.")

admin.site.register(Category)
admin.site.register(Tag)
```

**Inlines** let you edit related rows on the parent's page (e.g. order lines on an order):

```python
class CommentInline(admin.TabularInline):     # or StackedInline
    model = Comment
    extra = 0

class PostAdmin(admin.ModelAdmin):
    inlines = [CommentInline]
```

### 6.3 Securing the admin **[I]**

The admin is powerful, which makes it a target. Harden it:

- **Move it off `/admin/`** to a non-guessable path (`path("secret-mgmt-x7/", admin.site.urls)`).
- **Require staff + appropriate permissions** — only `is_staff` users reach it; grant per-model permissions via groups (Section 9).
- **Put it behind HTTPS** and ideally restrict by IP / VPN at the [Nginx](NGINX_GUIDE.md) layer.
- **Enforce strong auth** — consider MFA (e.g. `django-otp`), and audit logins.
- **Watch `list_display`/`list_select_related`** to avoid admin N+1 on big tables.
- In production, never leave a default `admin`/`admin` account or a weak superuser password.

---

## 7. Forms — Form/ModelForm, Validation, CSRF, Rendering, Formsets, File Uploads

### 7.1 What forms do **[I]**

Django **forms** handle the full lifecycle of user input: **rendering** the HTML widgets, **validating** and **cleaning** submitted data into Python types, **reporting errors**, and (with `ModelForm`) **saving** to a model. They're the safe, DRY way to accept input — even for APIs you'll use the same *idea* via DRF serializers (Section 12).

### 7.2 `Form` and `ModelForm` **[I]**

A plain `Form` declares fields explicitly. A **`ModelForm`** derives its fields *from a model* — the DRY default for create/edit forms.

```python
# blog/forms.py
from django import forms
from .models import Post

class ContactForm(forms.Form):                 # a plain Form (not tied to a model)
    name = forms.CharField(max_length=100)
    email = forms.EmailField()
    message = forms.CharField(widget=forms.Textarea)

    def clean_email(self):                     # field-level validation hook
        email = self.cleaned_data["email"]
        if email.endswith("@example.com"):
            raise forms.ValidationError("Example addresses are not allowed.")
        return email

    def clean(self):                           # cross-field validation hook
        cleaned = super().clean()
        # ...validate combinations of fields...
        return cleaned


class PostForm(forms.ModelForm):               # fields come from the model
    class Meta:
        model = Post
        fields = ["title", "body", "category", "tags", "status"]  # be explicit, not "__all__"
        widgets = {"body": forms.Textarea(attrs={"rows": 12})}
```

### 7.3 Using a form in a view (the canonical pattern) **[I]**

```python
# blog/views.py
from django.shortcuts import render, redirect
from .forms import PostForm

def post_create(request):
    if request.method == "POST":
        form = PostForm(request.POST)          # bind submitted data
        if form.is_valid():                    # runs all validation/cleaning
            post = form.save(commit=False)     # build instance, don't save yet
            post.author = request.user         # set fields the form doesn't include
            post.save()
            form.save_m2m()                    # needed after commit=False for M2M (tags)
            return redirect("blog:detail", pk=post.pk)   # Post/Redirect/Get
    else:
        form = PostForm()                      # unbound (empty) form for GET
    return render(request, "blog/post_form.html", {"form": form})
```

### 7.4 CSRF protection **[I]**

Django protects every state-changing POST with a **CSRF token** by default (via `CsrfViewMiddleware`). In templates you **must** include `{% csrf_token %}` inside your `<form>`, or the POST is rejected with a 403. This is on by default and you should leave it on.

```html
<form method="post">
  {% csrf_token %}            {# emits a hidden input with the per-session token #}
  {{ form.as_p }}             {# render the form fields (Section 7.5) #}
  <button type="submit">Save</button>
</form>
```

### 7.5 Rendering forms **[I]**

`{{ form }}` renders all fields with their labels, widgets, and errors. Helpers: `form.as_p` (paragraphs), `form.as_div` (the modern default, accessible), `form.as_table`, `form.as_ul`. For full control, render fields individually:

```html
{{ form.non_field_errors }}
<div class="field">
  {{ form.title.label_tag }}
  {{ form.title }}
  {{ form.title.errors }}
</div>
```

### 7.6 Formsets and file uploads **[I]**

A **formset** manages *multiple instances of the same form* on one page (e.g. several addresses). `inlineformset_factory` ties child forms to a parent.

For **file uploads**, the form needs `request.FILES`, and the HTML form needs `enctype="multipart/form-data"`. The uploaded file lands wherever the field's storage (the `STORAGES["default"]` backend, Section 10/18) puts it.

```python
# Model with an upload field:
class Document(models.Model):
    file = models.FileField(upload_to="docs/%Y/%m/")   # path under MEDIA_ROOT

# View — note request.FILES:
def upload(request):
    if request.method == "POST":
        form = DocumentForm(request.POST, request.FILES)   # <-- FILES is required
        if form.is_valid():
            form.save()
            return redirect("done")
    else:
        form = DocumentForm()
    return render(request, "upload.html", {"form": form})
```

```html
<form method="post" enctype="multipart/form-data">   {# enctype is mandatory for files #}
  {% csrf_token %}{{ form.as_div }}
  <button>Upload</button>
</form>
```

> **Security note:** validate uploads — restrict size and type, never trust the filename or content-type from the client, and serve user media from a separate, non-executable location (Section 18/19). For images, use `ImageField` (requires Pillow) which validates that the file is a real image.

---

## 8. Templates — the Django Template Language, Inheritance, Context, Tags/Filters, Static Files, DRF+SPA Alternative

### 8.1 The Django Template Language (DTL) **[B]**

DTL renders HTML by combining a **template** (HTML + placeholders) with a **context** (a dict of data the view passes). It is deliberately **limited** — you can display data, loop, branch, and call filters, but you *can't* run arbitrary Python. That restriction is a feature: it keeps logic out of templates and **auto-escapes** output to prevent XSS (Section 19).

Two syntaxes:
- **`{{ variable }}`** — print a value (auto-escaped).
- **`{% tag %}`** — logic: `{% if %}`, `{% for %}`, `{% url %}`, `{% block %}`, etc.

```html
{# blog/templates/blog/post_list.html #}
<ul>
  {% for post in posts %}                {# loop over the context variable #}
    <li>
      <a href="{% url 'blog:detail' pk=post.pk %}">{{ post.title }}</a>
      by {{ post.author.username }} — {{ post.created_at|date:"M j, Y" }}
    </li>
  {% empty %}                            {# runs if "posts" is empty #}
    <li>No posts yet.</li>
  {% endfor %}
</ul>
```

### 8.2 Template inheritance **[B]**

Inheritance is how you avoid repeating page chrome. A **base template** defines `{% block %}` placeholders; **child templates** `{% extends %}` it and fill the blocks. This is the backbone of every multi-page Django site.

```html
{# templates/base.html — the site skeleton #}
<!DOCTYPE html>
<html>
<head><title>{% block title %}My Site{% endblock %}</title></head>
<body>
  <header>…nav…</header>
  <main>{% block content %}{% endblock %}</main>   {# children fill this #}
  <footer>…</footer>
</body>
</html>
```

```html
{# blog/templates/blog/post_detail.html — a child page #}
{% extends "base.html" %}
{% block title %}{{ post.title }}{% endblock %}
{% block content %}
  <article>
    <h1>{{ post.title }}</h1>
    {{ post.body|linebreaks }}           {# a filter: turn newlines into <p>/<br> #}
  </article>
{% endblock %}
```

### 8.3 Tags, filters, and `{% include %}` **[B/I]**

**Filters** transform a value with `|`: `{{ name|upper }}`, `{{ price|floatformat:2 }}`, `{{ body|truncatewords:30 }}`, `{{ date|date:"Y-m-d" }}`, `{{ value|default:"—" }}`. **Tags** do logic. `{% include "partial.html" %}` embeds a reusable fragment. You can write **custom template tags/filters** in an app's `templatetags/` package when built-ins aren't enough.

> **Security:** output is auto-escaped. Only mark content safe with `|safe` or `{% autoescape off %}` when you *fully trust* it (or have sanitized it) — otherwise you've reintroduced XSS.

### 8.4 Static files **[B/I]**

**Static files** are your CSS/JS/images. In templates, load them via the `static` tag so the URL is correct in both dev and production (where they may be served from a CDN with hashed names).

```html
{% load static %}                                  {# load the staticfiles tag library #}
<link rel="stylesheet" href="{% static 'css/site.css' %}">
<img src="{% static 'img/logo.svg' %}" alt="logo">
```

In development, `runserver` serves static files automatically. In production you run **`collectstatic`** to gather them into `STATIC_ROOT`, then serve them via WhiteNoise or Nginx/CDN (Section 18).

### 8.5 When to skip templates: DRF + an SPA **[I]**

DTL is excellent for **server-rendered, content-driven sites** (blogs, marketing, dashboards, the admin). But if your frontend is a **single-page app** (React/Vue/Svelte) or a mobile app, you typically **don't use Django templates at all** — instead Django becomes a **JSON API via DRF** (Section 12) and the SPA renders the UI. Common 2026 architectures:

- **Server-rendered Django templates** — simplest, fast to ship, great SEO, no separate frontend build. Sprinkle in **HTMX/Alpine** for interactivity without an SPA.
- **Django (DRF) backend + separate SPA frontend** — clean separation, rich client UX, but two codebases and CORS/auth to manage.
- **Hybrid** — server-rendered shell with islands of interactivity.

Pick templates when the product is content-first and SEO matters; pick DRF+SPA when the UX is highly interactive/app-like. Either way the **models and ORM stay the same** — only the presentation layer changes.

---

## 9. Authentication & Authorization — Users, login_required, Permissions/Groups, Password Hashing, Custom User Model, Sessions

### 9.1 The auth system overview **[I]**

`django.contrib.auth` is a complete, battle-tested **authentication** (who are you?) and **authorization** (what may you do?) system: a `User` model, password hashing, login/logout, sessions, permissions, and groups — plus ready-made login/logout/password-reset views. You rarely build auth from scratch in Django, and you shouldn't.

### 9.2 Logging users in and out **[I]**

```python
from django.contrib.auth import authenticate, login, logout

def my_login(request):
    if request.method == "POST":
        # authenticate() verifies credentials against the configured backends.
        user = authenticate(request, username=request.POST["username"],
                            password=request.POST["password"])
        if user is not None:
            login(request, user)            # attach the user to the session
            return redirect("dashboard")
    return render(request, "registration/login.html")

def my_logout(request):
    logout(request)                          # flush the session
    return redirect("home")
```

Most apps just wire up Django's built-in auth URLs instead of writing the above:

```python
# config/urls.py
path("accounts/", include("django.contrib.auth.urls")),
# -> /accounts/login/, /accounts/logout/, /accounts/password_reset/, etc.
```

### 9.3 Protecting views: `login_required` & mixins **[I]**

```python
from django.contrib.auth.decorators import login_required, permission_required
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin

@login_required                              # FBV: redirect anonymous users to login
def dashboard(request):
    return render(request, "dashboard.html")

@permission_required("blog.add_post", raise_exception=True)   # require a permission
def create_post(request):
    ...

class PostCreateView(LoginRequiredMixin, PermissionRequiredMixin, CreateView):  # CBV
    permission_required = "blog.add_post"
    model = Post
    fields = ["title", "body"]
```

> ⚡ **Version note.** Django 5.1 added a **`login_required` decorator usable directly on class-based views** (and a `login_not_required` decorator). `LoginRequiredMixin` remains the explicit, widely-used CBV approach.

### 9.4 Permissions and groups **[I/A]**

Every model automatically gets four permissions: `add`, `change`, `delete`, `view` (e.g. `blog.add_post`). You can also declare **custom permissions** in a model's `Meta`. **Groups** bundle permissions so you assign roles, not individual perms.

```python
# Check a permission in code or templates:
request.user.has_perm("blog.delete_post")
# Template: {% if perms.blog.delete_post %} ... {% endif %}

# Assign via groups (typically done once, via admin or a data migration):
from django.contrib.auth.models import Group, Permission
editors = Group.objects.create(name="Editors")
editors.permissions.add(Permission.objects.get(codename="add_post"))
user.groups.add(editors)
```

For **object-level** (per-row) permissions beyond the model-level defaults, use **`django-guardian`** or implement checks in your views/DRF permissions (Section 12.6).

### 9.5 Password hashing **[I/A]**

Django **never stores plaintext passwords**. It hashes them with a strong algorithm (default **PBKDF2** with many iterations; you can switch to **Argon2** by installing `argon2-cffi` and listing it first in `PASSWORD_HASHERS`). Hashes auto-upgrade on login when you raise the work factor. Always set passwords via `user.set_password(raw)` / `create_user()` — never assign `user.password = ...` directly.

```python
# Prefer Argon2 (memory-hard, current best practice) — install argon2-cffi:
PASSWORD_HASHERS = [
    "django.contrib.auth.hashers.Argon2PasswordHasher",
    "django.contrib.auth.hashers.PBKDF2PasswordHasher",
    # ...keep older hashers below so existing hashes still verify...
]

# Enforce password strength with validators (settings.py):
AUTH_PASSWORD_VALIDATORS = [
    {"NAME": "django.contrib.auth.password_validation.UserAttributeSimilarityValidator"},
    {"NAME": "django.contrib.auth.password_validation.MinimumLengthValidator",
     "OPTIONS": {"min_length": 12}},
    {"NAME": "django.contrib.auth.password_validation.CommonPasswordValidator"},
    {"NAME": "django.contrib.auth.password_validation.NumericPasswordValidator"},
]
```

### 9.6 The custom user model — do this on day one **[I/A]**

**The single most important early decision in any Django project: define a custom user model before your first `migrate`.** Even if you don't need extra fields yet, swapping the user model after migrations exist is painful and risky. The recommended approach is to subclass `AbstractUser` (keeps username/email/etc.) — or `AbstractBaseUser` if you want full control (e.g. email-only login).

```python
# accounts/models.py — create the "accounts" app FIRST, before any migrate
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    # Add whatever your app needs now or later:
    bio = models.TextField(blank=True)
    is_verified = models.BooleanField(default=False)
    # To make EMAIL the login field instead of username, override USERNAME_FIELD
    # and REQUIRED_FIELDS and provide a custom manager (AbstractBaseUser route).
```

```python
# settings.py — point Django at your model BEFORE the first migration
AUTH_USER_MODEL = "accounts.User"
```

> **Always** reference the user via **`settings.AUTH_USER_MODEL`** (in models/FKs) and **`get_user_model()`** (in code) — never import `django.contrib.auth.models.User` directly, or your code breaks the moment you (or a reused app) swap the user model.

```python
from django.contrib.auth import get_user_model
User = get_user_model()        # the right way to get the active user model in code
```

### 9.7 Registering the custom user in the admin **[I]**

```python
# accounts/admin.py
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin
from .models import User

@admin.register(User)
class CustomUserAdmin(UserAdmin):              # reuse Django's polished user admin
    # add your extra fields to the fieldsets so they show in the admin form
    fieldsets = UserAdmin.fieldsets + (("Profile", {"fields": ("bio", "is_verified")}),)
```

### 9.8 Sessions **[I]**

A **session** is server-side per-user storage keyed by a cookie. Django uses it to remember logins and any data you stash in `request.session`. The store is pluggable — database (default), cached, or **cached-database**; for performance back sessions with **[Redis](REDIS_GUIDE.md)** (Section 14).

```python
request.session["cart_id"] = 42            # write
cart = request.session.get("cart_id")      # read
request.session.set_expiry(60 * 60 * 24)   # 1 day; 0 = expire at browser close
# settings.py: SESSION_ENGINE, SESSION_COOKIE_SECURE=True (Section 19),
#              SESSION_COOKIE_HTTPONLY=True (default), SESSION_COOKIE_SAMESITE="Lax"
```

---

## 10. Settings, Config & the 12-Factor App — Splitting Settings, env vars, Secrets, STORAGES, the Security Checklist

### 10.1 The 12-factor principle **[I]**

The **12-factor app** methodology says: **strict separation of config from code**. Anything that varies between environments (dev/staging/prod) — secrets, DB URLs, debug flag, allowed hosts — should come from the **environment**, not be hard-coded. This lets one codebase deploy anywhere and keeps secrets out of version control.

### 10.2 Splitting settings **[I/A]**

A single `settings.py` becomes unwieldy and tempts you to branch on `if DEBUG`. The clean pattern is a **settings package** with a shared `base.py` and per-environment overrides:

```text
config/settings/
├── __init__.py
├── base.py        # everything common
├── dev.py         # from .base import *; DEBUG=True; dev-only apps/tools
└── prod.py        # from .base import *; DEBUG=False; hardened settings
```

```python
# config/settings/prod.py
from .base import *          # noqa
import os

DEBUG = False
ALLOWED_HOSTS = os.environ["ALLOWED_HOSTS"].split(",")
# ...production DB, cache, security, logging overrides...
```

Select the file per environment via `DJANGO_SETTINGS_MODULE=config.settings.prod` (set in your process manager / Docker env), and point `manage.py`/`wsgi.py`/`asgi.py` at a sensible default.

### 10.3 Reading env vars and secrets **[I/A]**

Use **`django-environ`** (or plain `os.environ`) to read typed values from the environment / a local `.env` file (which you **never commit**).

```python
# config/settings/base.py
import environ
env = environ.Env(DEBUG=(bool, False))                 # declare type + default
environ.Env.read_env(BASE_DIR / ".env")                # load .env in dev (not in prod)

SECRET_KEY = env("SECRET_KEY")                          # required -> fails loudly if missing
DEBUG = env("DEBUG")
ALLOWED_HOSTS = env.list("ALLOWED_HOSTS", default=[])
DATABASES = {"default": env.db("DATABASE_URL")}        # parse postgres://user:pass@host/db
CACHES = {"default": env.cache("REDIS_URL")}           # parse redis://...
```

> **Best practice:** in real production, prefer a **secrets manager** (AWS Secrets Manager, Vault, Docker/Kubernetes secrets) over a `.env` file on disk. The `SECRET_KEY` must be high-entropy and unique per environment; rotating it invalidates sessions and password-reset tokens.

### 10.4 `STORAGES` — the modern file/static config **[I/A]**

⚡ **Version note.** Since Django **4.2**, file and static storage are configured through the single **`STORAGES`** dict — the old `DEFAULT_FILE_STORAGE` and `STATICFILES_STORAGE` settings are deprecated. This is the place to plug in S3 (via `django-storages`) for media and a hashed/manifest backend (or WhiteNoise) for static files.

```python
# Production STORAGES: media on S3, static hashed for cache-busting
STORAGES = {
    "default": {                                   # user-uploaded media
        "BACKEND": "storages.backends.s3.S3Storage",
        "OPTIONS": {"bucket_name": env("AWS_MEDIA_BUCKET")},
    },
    "staticfiles": {                               # CSS/JS — hashed filenames + manifest
        "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
    },
}
```

### 10.5 The production security settings checklist **[I/A]**

Set these in `prod.py` (each is expanded in Section 19). `python manage.py check --deploy` audits most of them:

```python
DEBUG = False
ALLOWED_HOSTS = ["example.com", "www.example.com"]
SECRET_KEY = env("SECRET_KEY")                         # from env, never in code

SECURE_SSL_REDIRECT = True                             # force HTTPS
SECURE_HSTS_SECONDS = 31536000                         # HSTS (1 year)
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SESSION_COOKIE_SECURE = True                           # cookies only over HTTPS
CSRF_COOKIE_SECURE = True
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")  # behind Nginx/LB
CSRF_TRUSTED_ORIGINS = ["https://example.com"]         # required for cross-origin POSTs
X_FRAME_OPTIONS = "DENY"                                # clickjacking
SECURE_CONTENT_TYPE_NOSNIFF = True
```

---

## 11. Middleware — the Request/Response Chain, Writing Custom Middleware, Built-ins

### 11.1 What middleware is **[I]**

**Middleware** is a stack of components that wrap every request/response. Each can inspect or modify the request on the way **in**, short-circuit it (return a response without reaching a view), and inspect or modify the response on the way **out**. It's how Django implements cross-cutting concerns — sessions, auth, CSRF, security headers, GZip — without cluttering views. The `MIDDLEWARE` list order is significant: request phase runs **top→bottom**, response phase runs **bottom→top** (Section 1.6).

### 11.2 The important built-in middleware **[I]**

| Middleware | Does |
|---|---|
| `SecurityMiddleware` | HTTPS redirect, HSTS, several `SECURE_*` headers |
| `SessionMiddleware` | loads/saves the session |
| `CommonMiddleware` | URL normalization, `APPEND_SLASH` |
| `CsrfViewMiddleware` | CSRF protection on POST |
| `AuthenticationMiddleware` | sets `request.user` |
| `MessageMiddleware` | flash messages |
| `XFrameOptionsMiddleware` | clickjacking (`X-Frame-Options`) |
| `GZipMiddleware` | gzip responses (use with care re: BREACH) |
| `WhiteNoiseMiddleware` | serve static files in production (added by you) |

### 11.3 Writing custom middleware **[I]**

The modern form is a callable factory: a function (or class with `__call__`) that takes `get_response` and returns the per-request handler.

```python
# core/middleware.py — time each request and add a header
import time

def timing_middleware(get_response):
    # one-time setup at startup goes here
    def middleware(request):
        start = time.perf_counter()
        response = get_response(request)        # call the next layer / the view
        response["X-Request-Duration-ms"] = f"{(time.perf_counter()-start)*1000:.1f}"
        return response
    return middleware
```

```python
# settings.py — add it to the chain (placement matters!)
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    # ...
    "core.middleware.timing_middleware",
]
```

You can also implement **hook methods** — `process_view`, `process_exception`, `process_template_response` — by writing a class with those methods. Keep middleware **fast and lean**: it runs on *every* request.

---

## 12. Django REST Framework — Serializers, ViewSets/Routers, Generic Views, Auth (Token/JWT/Session), Permissions, Pagination, Filtering, Throttling, Versioning

**Django REST Framework (DRF)** is the de-facto standard for building APIs in Django. It adds serializers (validation + JSON conversion), class-based API views and ViewSets, authentication/permission layers, pagination, throttling, and a browsable API. It is to JSON what forms+templates are to HTML.

```bash
uv pip install djangorestframework
```

```python
# settings.py
INSTALLED_APPS += ["rest_framework"]
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.SessionAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": ["rest_framework.permissions.IsAuthenticated"],
    "DEFAULT_PAGINATION_CLASS":
        "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 25,
}
```

### 12.1 Serializers and `ModelSerializer` **[I]**

A **serializer** converts model instances ↔ JSON and **validates** incoming data (DRF's analogue to forms). `ModelSerializer` derives fields from a model, just like `ModelForm`.

```python
# blog/serializers.py
from rest_framework import serializers
from .models import Post

class PostSerializer(serializers.ModelSerializer):
    # Read-only derived/related fields:
    author = serializers.StringRelatedField(read_only=True)   # uses author.__str__
    comment_count = serializers.IntegerField(source="comments.count", read_only=True)

    class Meta:
        model = Post
        fields = ["id", "title", "slug", "body", "status",
                  "author", "comment_count", "created_at"]
        read_only_fields = ["slug", "created_at"]

    def validate_title(self, value):          # field-level validation
        if "spam" in value.lower():
            raise serializers.ValidationError("No spam in titles.")
        return value
```

### 12.2 Views: ViewSets, routers, and generic views **[I/A]**

DRF gives you three tiers of leverage:

- **`APIView`** — class-based, you write `get`/`post` yourself (most control).
- **Generic views / mixins** — `ListCreateAPIView`, `RetrieveUpdateDestroyAPIView` — CRUD with a `queryset` + `serializer_class`.
- **`ViewSet` / `ModelViewSet`** — bundle *all* CRUD actions into one class and let a **router** generate the URLs. The highest-leverage option for standard resources.

```python
# blog/views.py
from rest_framework import viewsets, permissions
from .models import Post
from .serializers import PostSerializer

class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]

    def get_queryset(self):
        # ALWAYS optimize the queryset — DRF list endpoints are prime N+1 territory!
        return (Post.objects.filter(status="published")
                .select_related("author").prefetch_related("comments"))

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)   # set author from the request
```

```python
# blog/urls.py — a router builds /posts/, /posts/{pk}/ with all CRUD verbs
from rest_framework.routers import DefaultRouter
from .views import PostViewSet

router = DefaultRouter()
router.register(r"posts", PostViewSet, basename="post")
urlpatterns = router.urls
```

### 12.3 Authentication for APIs — Token / JWT / Session **[I/A]**

| Scheme | Best for | Notes |
|---|---|---|
| **SessionAuthentication** | a browser SPA on the **same site** | uses Django sessions + CSRF; simplest if same-origin |
| **TokenAuthentication** (DRF built-in) | simple machine/mobile clients | one static token per user; no expiry by default |
| **JWT** (`djangorestframework-simplejwt`) | mobile/SPA, microservices, stateless | short-lived access + refresh tokens; scalable |

```python
# JWT setup with simplejwt:  uv pip install djangorestframework-simplejwt
# settings.py
REST_FRAMEWORK["DEFAULT_AUTHENTICATION_CLASSES"] = [
    "rest_framework_simplejwt.authentication.JWTAuthentication",
]

# urls.py — token obtain/refresh endpoints
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView
urlpatterns += [
    path("api/token/", TokenObtainPairView.as_view()),       # POST creds -> access+refresh
    path("api/token/refresh/", TokenRefreshView.as_view()),  # POST refresh -> new access
]
```

> **Security:** with JWT, keep access tokens **short-lived** (minutes) and rotate refresh tokens; store tokens safely on the client (httpOnly cookies mitigate XSS theft, at the cost of needing CSRF handling). Never put secrets in the JWT payload — it's signed, not encrypted.

### 12.4 Permissions **[I/A]**

Permission classes gate access to a view. Built-ins: `AllowAny`, `IsAuthenticated`, `IsAdminUser`, `IsAuthenticatedOrReadOnly`. Write **custom** ones (including **object-level** `has_object_permission`) for ownership rules.

```python
from rest_framework import permissions

class IsAuthorOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:   # GET/HEAD/OPTIONS
            return True
        return obj.author == request.user                # writes: only the author
```

### 12.5 Pagination, filtering, throttling, versioning **[I/A]**

```python
# settings.py — global throttling (rate limiting) and filtering backends
REST_FRAMEWORK.update({
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.AnonRateThrottle",
        "rest_framework.throttling.UserRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {"anon": "60/min", "user": "1000/day"},
    "DEFAULT_FILTER_BACKENDS": [
        "django_filters.rest_framework.DjangoFilterBackend",  # pip django-filter
        "rest_framework.filters.SearchFilter",
        "rest_framework.filters.OrderingFilter",
    ],
})
```

```python
# Per-view filtering/search/ordering:
class PostViewSet(viewsets.ModelViewSet):
    filterset_fields = ["status", "category"]      # ?status=published&category=3
    search_fields = ["title", "body"]              # ?search=django
    ordering_fields = ["created_at", "title"]      # ?ordering=-created_at
```

- **Pagination** — `PageNumberPagination` (`?page=2`), `LimitOffsetPagination` (`?limit=&offset=`), or **`CursorPagination`** (best for large/append-heavy datasets — stable, no deep-offset cost; see Section 20).
- **Throttling** — protects against abuse and runaway clients; tune per scope.
- **Versioning** — `URLPathVersioning` (`/api/v1/...`) or header-based; set `DEFAULT_VERSIONING_CLASS`. Version from the start so you can evolve the API without breaking clients.

> **Best practice:** for OpenAPI/Swagger docs, add **`drf-spectacular`** — it generates an accurate schema and a Swagger UI, the DRF answer to FastAPI's built-in docs (see [FastAPI](FASTAPI_GUIDE.md) for the contrast).

---

## 13. Async Django — Async Views, the Async ORM, ASGI, Channels for WebSockets

### 13.1 The state of async in Django **[A]**

Django has grown real async support: you can write **`async def` views**, async middleware, and use an **async ORM** API, served over **ASGI**. The honest 2026 picture: async Django shines for **I/O-bound concurrency** — calling slow external APIs, fanning out many network requests, streaming, and WebSockets — where it lets a worker handle other requests while awaiting I/O. For ordinary CRUD that's mostly fast DB queries, **sync Django is simpler and usually just as fast**; don't go async by default. If raw async throughput on a pure JSON API is your top priority, also weigh [FastAPI](FASTAPI_GUIDE.md).

```python
# An async view — must be served by an ASGI server (Uvicorn/Daphne/Hypercorn)
import httpx
from django.http import JsonResponse

async def dashboard(request):
    # await external I/O without blocking the worker:
    async with httpx.AsyncClient() as client:
        r = await client.get("https://api.example.com/stats")
    return JsonResponse(r.json())
```

### 13.2 The async ORM **[A]**

Each blocking ORM call has an **`a`-prefixed async twin** (`aget`, `acreate`, `afirst`, `acount`, `aupdate`, `adelete`, `aexists`), and you iterate querysets with `async for`. Under the hood much DB work still runs in a threadpool, but the API is genuinely async and improving each release.

```python
async def post_detail(request, pk):
    post = await Post.objects.aget(pk=pk)             # async fetch
    comments = [c async for c in post.comments.all()]  # async iteration
    return JsonResponse({"title": post.title, "comments": len(comments)})
```

> **Gotcha:** never call a **sync** ORM method directly inside an `async def` view — it raises `SynchronousOnlyOperation`. Use the `a`-prefixed methods, or wrap blocking calls in `sync_to_async`. Conversely, in sync code don't `await` async APIs.

### 13.3 ASGI vs WSGI **[A]**

**WSGI** (Gunicorn) is the classic synchronous server interface — perfect for sync Django. **ASGI** (Uvicorn, Daphne, Hypercorn) is the async interface required for async views and WebSockets. You can run sync Django on ASGI too. Section 18 covers deploying both.

### 13.4 Channels for WebSockets **[A]**

Django's HTTP request/response model can't do **WebSockets** (persistent, bidirectional connections) on its own. **Django Channels** extends Django to ASGI consumers for WebSockets, chat, live notifications, and presence — typically with a **[Redis](REDIS_GUIDE.md)** channel layer to broadcast across worker processes.

```python
# A minimal Channels consumer (chat). Requires channels + a Redis channel layer.
from channels.generic.websocket import AsyncJsonWebsocketConsumer

class ChatConsumer(AsyncJsonWebsocketConsumer):
    async def connect(self):
        self.room = self.scope["url_route"]["kwargs"]["room"]
        await self.channel_layer.group_add(self.room, self.channel_name)  # join group
        await self.accept()

    async def receive_json(self, content):
        # broadcast to everyone in the room via the Redis-backed channel layer
        await self.channel_layer.group_send(
            self.room, {"type": "chat.message", "text": content["text"]})

    async def chat_message(self, event):
        await self.send_json({"text": event["text"]})
```

---

## 14. Caching — the Cache Framework, Redis Backend, Per-View/Fragment/Low-Level, Invalidation

### 14.1 The cache framework and Redis backend **[I/A]**

Caching stores expensive results so repeated requests are cheap. Django's **cache framework** abstracts the backend behind one API; in production the standard backend is **[Redis](REDIS_GUIDE.md)** (fast, shared across workers/servers, supports TTLs). Django 4.0+ ships a native Redis backend — no extra package needed.

```python
# settings.py
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": env("REDIS_URL", default="redis://127.0.0.1:6379/1"),
    }
}
```

### 14.2 The three levels of caching **[I/A]**

**1) Per-view caching** — cache an entire view's response for N seconds:

```python
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)                # cache this page for 15 minutes
def expensive_report(request):
    ...
```

**2) Template-fragment caching** — cache just a slow part of a page:

```html
{% load cache %}
{% cache 600 sidebar request.user.id %}    {# 600s, keyed by user so it varies per-user #}
  …expensive sidebar…
{% endcache %}
```

**3) Low-level cache API** — cache arbitrary values with full control (the most flexible):

```python
from django.core.cache import cache

def top_posts():
    key = "top_posts"
    data = cache.get(key)
    if data is None:                       # cache miss -> compute and store
        data = list(Post.objects.filter(status="published")
                    .order_by("-views")[:10].values("id", "title"))
        cache.set(key, data, timeout=300)  # TTL 5 min
    return data

# Atomic helpers:
cache.get_or_set("k", expensive_callable, timeout=60)
cache.incr("page_views")                   # atomic counter
```

### 14.3 Cache invalidation **[I/A]**

"There are only two hard things in computer science…" — invalidation is the hard part. Strategies, roughly in order of preference:

- **Short TTLs** — let stale data expire; simplest, good enough for most reads.
- **Explicit invalidation** — delete/replace the key when the underlying data changes (e.g. in a model's `save()` or via a `post_save` signal): `cache.delete("top_posts")`.
- **Key versioning / namespacing** — bump a version number to invalidate a whole group of keys at once.

> **Best practice:** also use Redis for **sessions** (Section 9.8) and as a **Celery broker** (Section 15). And cache *computed/aggregated* data, not whole querysets you then mutate — combine caching with the ORM optimizations in Section 20 rather than as a substitute for them.

---

## 15. Background Tasks — Celery (+ Redis broker), Beat Scheduling, the Built-in Tasks Framework

### 15.1 Why offload work **[A]**

A web request should return in milliseconds. Anything slow — sending email, generating PDFs/thumbnails, calling third-party APIs, heavy computation — should run **outside** the request/response cycle so the user isn't kept waiting and a worker isn't tied up. That's a **background task**: the view enqueues a job and returns immediately; a separate **worker** process picks it up.

### 15.2 Celery with a Redis broker **[A]**

**Celery** is the established distributed task queue for Python/Django. It needs a **broker** (a message queue — **[Redis](REDIS_GUIDE.md)** or RabbitMQ) to pass jobs to workers, and optionally a **result backend** to store outcomes.

```python
# config/celery.py
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.prod")
app = Celery("config")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()        # find tasks.py in each installed app
```

```python
# config/__init__.py — ensure Celery loads with Django
from .celery import app as celery_app
__all__ = ("celery_app",)
```

```python
# settings.py
CELERY_BROKER_URL = env("REDIS_URL", default="redis://localhost:6379/0")
CELERY_RESULT_BACKEND = env("REDIS_URL", default="redis://localhost:6379/0")
CELERY_TASK_ACKS_LATE = True            # re-queue if a worker dies mid-task
```

```python
# blog/tasks.py — define a task
from celery import shared_task
from django.core.mail import send_mail

@shared_task(bind=True, max_retries=3, default_retry_delay=30)
def send_receipt_email(self, user_id):
    try:
        # ...look up the user, render, send...
        send_mail("Receipt", "Thanks!", "no-reply@x.com", ["u@x.com"])
    except Exception as exc:
        raise self.retry(exc=exc)        # automatic retry with backoff
```

```python
# Enqueue from a view (returns instantly). Use on_commit so the task only runs
# if the transaction that created the data actually committed (Section 4.8):
from django.db import transaction
transaction.on_commit(lambda: send_receipt_email.delay(user.id))
```

```bash
celery -A config worker -l info          # run a worker process
celery -A config beat -l info            # run the scheduler (see below)
```

### 15.3 Beat scheduling (periodic tasks) **[A]**

**Celery Beat** runs tasks on a schedule (cron-like) — nightly reports, cache warmups, cleanup. Use `django-celery-beat` to manage schedules in the database/admin.

```python
# settings.py — a simple static schedule
from celery.schedules import crontab
CELERY_BEAT_SCHEDULE = {
    "nightly-cleanup": {
        "task": "blog.tasks.cleanup_drafts",
        "schedule": crontab(hour=3, minute=0),      # every day at 03:00
    },
}
```

### 15.4 ⚡ The built-in Tasks framework **[A]**

⚡ **Version note.** **Django 6.0 (Dec 2025) introduced a built-in Tasks framework** (`django.tasks`) — a standard, backend-agnostic API for defining and enqueuing background tasks, so simple apps can offload work **without pulling in Celery**. It ships with an immediate/in-process backend and a pluggable interface; production-grade durable workers still typically use Celery (or a Celery-backed adapter). It's new and evolving — for serious queues in 2026, **Celery remains the battle-tested default**, with the built-in framework a lightweight option for simpler needs and a likely future direction.

---

## 16. Testing — TestCase/pytest-django, the Test Client, factory_boy, Fixtures, Coverage

### 16.1 Why and how Django tests **[A]**

Django has first-class testing built in. `django.test.TestCase` wraps **each test in a transaction that rolls back**, so tests are isolated and fast and never pollute a real DB (it uses a throwaway test database). You get a **test client** that simulates requests end-to-end without a running server.

```python
# blog/tests/test_models.py — Django's built-in TestCase
from django.test import TestCase
from django.contrib.auth import get_user_model
from blog.models import Post

class PostModelTests(TestCase):
    def setUp(self):
        self.user = get_user_model().objects.create_user("ada", password="x")

    def test_str_returns_title(self):
        post = Post.objects.create(title="Hi", body="...", author=self.user)
        self.assertEqual(str(post), "Hi")
```

### 16.2 pytest-django — the popular choice **[A]**

Most teams prefer **pytest** with the `pytest-django` plugin: terser tests, powerful fixtures, better output.

```python
# conftest.py / test_views.py with pytest-django
import pytest

@pytest.mark.django_db                       # grant DB access to this test
def test_post_list(client):                  # `client` fixture = the test client
    resp = client.get("/blog/")
    assert resp.status_code == 200
```

### 16.3 The test client and API tests **[A]**

```python
from django.test import TestCase
class ViewTests(TestCase):
    def test_login_required_redirects(self):
        resp = self.client.get("/dashboard/")
        self.assertEqual(resp.status_code, 302)         # anonymous -> redirect to login
    def test_create_post(self):
        self.client.force_login(self.user)              # skip the login flow
        resp = self.client.post("/blog/new/", {"title": "T", "body": "B"})
        self.assertEqual(Post.objects.count(), 1)

# DRF APIs: use APIClient for auth + JSON helpers
from rest_framework.test import APIClient
def test_api_create(db, user):
    api = APIClient(); api.force_authenticate(user)
    resp = api.post("/api/posts/", {"title": "T", "body": "B"}, format="json")
    assert resp.status_code == 201
```

### 16.4 Factories (factory_boy) vs fixtures **[A]**

Hand-building objects in every test is tedious and brittle. **`factory_boy`** generates realistic test objects on demand — far more maintainable than static JSON **fixtures** (which drift from your models). Prefer factories.

```python
# tests/factories.py
import factory
from django.contrib.auth import get_user_model
from blog.models import Post

class UserFactory(factory.django.DjangoModelFactory):
    class Meta: model = get_user_model()
    username = factory.Sequence(lambda n: f"user{n}")

class PostFactory(factory.django.DjangoModelFactory):
    class Meta: model = Post
    title = factory.Faker("sentence")
    body = factory.Faker("paragraph")
    author = factory.SubFactory(UserFactory)      # auto-create a related user
```

### 16.5 Coverage **[A]**

```bash
pip install coverage pytest-django factory_boy
coverage run -m pytest          # run tests under coverage
coverage report -m              # show % covered + missing lines
coverage html                   # browsable HTML report
```

> **Best practice:** test **behavior, not implementation** — assert on responses, DB state, and side effects. Cover the risky stuff (auth, permissions, money, edge cases) heavily; don't chase 100% on trivial code. Run tests in CI on every push.

---

## 17. Project Structure & Maintainability at Scale — App Boundaries, Fat Models/Thin Views/Services, Settings Layout, Reusable Apps

### 17.1 Drawing app boundaries **[A]**

The defining skill in a large Django codebase is **carving the project into cohesive apps** by *domain*, not by technical layer. Good apps are **high-cohesion, low-coupling**: `accounts`, `billing`, `catalog`, `orders` — each owning its models, views, and logic, depending on others through clear interfaces. Anti-patterns: a single god app holding everything; or apps split by layer (`models_app`, `views_app`) which couples everything to everything. A healthy app could, in principle, be extracted and reused elsewhere.

### 17.2 Fat models, thin views, and a service layer **[A]**

Where does business logic live? Django wisdom:

- **Thin views** — views should orchestrate (parse request, call logic, return response), not contain business rules. A view full of branching logic and ORM gymnastics is a smell.
- **Fat models / model methods & managers** — push data-centric logic onto the model (e.g. `order.mark_paid()`) and reusable queries onto **custom managers/querysets** (`Post.objects.published()`).
- **A service layer for cross-model workflows** — when an operation spans several models or integrates external systems (e.g. "check out a cart": create order, charge card, decrement stock, email receipt), put it in a **plain-Python service function** (`services.py`) that wraps the steps in a transaction. This keeps both views and models clean and makes the workflow testable in isolation.

```python
# orders/managers.py — reusable query logic as a custom QuerySet/manager
from django.db import models

class PostQuerySet(models.QuerySet):
    def published(self):
        return self.filter(status="published")
    def by_author(self, user):
        return self.filter(author=user)

# models.py:  objects = PostQuerySet.as_manager()
# usage:      Post.objects.published().by_author(request.user)
```

```python
# orders/services.py — a service function for a multi-model workflow
from django.db import transaction

@transaction.atomic
def checkout(cart, user):
    order = Order.objects.create(user=user)
    for line in cart.lines.select_related("product"):
        line.product.reserve(line.qty)                    # model method (fat model)
        OrderLine.objects.create(order=order, product=line.product, qty=line.qty)
    transaction.on_commit(lambda: send_receipt_email.delay(order.id))
    return order
```

### 17.3 Reusable apps and big-codebase organization **[A]**

- Group apps under an `apps/` package; keep `config/` for project glue only.
- Keep each app's public surface small; avoid cross-app imports of internals.
- Use **signals sparingly** (Section 21) — explicit calls are easier to trace than implicit signal cascades.
- Split settings (Section 10), centralize constants/enums, and document app responsibilities.
- Add linting/formatting (**ruff**, **black**) and type checking (**mypy** + `django-stubs`) in CI for consistency at scale.

---

## 18. Production Deployment — Gunicorn/Uvicorn behind Nginx, collectstatic + WhiteNoise/CDN, Docker, the Deployment Checklist, Pooling, Logging

### 18.1 The production architecture **[A]**

`runserver` is for development only. In production you run an **application server** (Gunicorn for sync, or Uvicorn/Gunicorn-with-Uvicorn-workers for ASGI/async) hosting your WSGI/ASGI app, sitting **behind [Nginx](NGINX_GUIDE.md)** acting as a reverse proxy. Nginx terminates TLS, serves static/media files efficiently, buffers slow clients, and forwards dynamic requests to the app server over a socket.

```text
Internet ──HTTPS──► Nginx (TLS, static/media, reverse proxy)
                      │  (proxy_pass to a unix socket / 127.0.0.1:8000)
                      ▼
                Gunicorn / Uvicorn  ──►  Django (WSGI/ASGI)  ──►  PostgreSQL + Redis
```

```bash
# Sync (WSGI) with Gunicorn:
gunicorn config.wsgi:application --workers 4 --bind unix:/run/app.sock --timeout 60
# Async (ASGI) with Uvicorn workers under Gunicorn (for async views/Channels):
gunicorn config.asgi:application -k uvicorn.workers.UvicornWorker --workers 4 \
         --bind unix:/run/app.sock
```

A minimal Nginx server block (see the [Nginx](NGINX_GUIDE.md) guide for TLS/HSTS specifics):

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;
    # ...ssl_certificate / ssl_certificate_key + HSTS here (see Nginx guide)...

    location /static/ { alias /srv/app/staticfiles/; access_log off; expires 30d; }
    location /media/  { alias /srv/app/media/; }

    location / {
        proxy_pass http://unix:/run/app.sock;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;     # pairs with SECURE_PROXY_SSL_HEADER
    }
}
```

### 18.2 Static & media files — collectstatic, WhiteNoise, CDN **[A]**

Django doesn't serve static files in production. Two options:

- **WhiteNoise** — serve static files **from the app process** (compressed, hashed/cache-busted). Dead simple, great for containers/PaaS, no Nginx static config needed.
- **Nginx / CDN** — `collectstatic` writes everything to `STATIC_ROOT`, then Nginx (or a CDN like CloudFront) serves it. Best for high traffic.

```bash
python manage.py collectstatic --noinput     # gather all static files into STATIC_ROOT
```

```python
# WhiteNoise setup: add the middleware right after SecurityMiddleware + use its storage
MIDDLEWARE.insert(1, "whitenoise.middleware.WhiteNoiseMiddleware")
STORAGES["staticfiles"]["BACKEND"] = \
    "whitenoise.storage.CompressedManifestStaticFilesStorage"
STATIC_ROOT = BASE_DIR / "staticfiles"
```

**Media** (user uploads) should go to object storage (**S3** via `django-storages`) in any multi-server or container deploy — never the ephemeral local disk.

### 18.3 Dockerizing Django **[A]**

See the [Docker](DOCKER_GUIDE.md) guide for the full story; the Django-specific shape:

```dockerfile
# Multi-stage, slim, non-root, runs migrations + collectstatic at deploy
FROM python:3.13-slim AS base
ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1
WORKDIR /app
RUN pip install --no-cache-dir uv
COPY pyproject.toml uv.lock ./
RUN uv pip install --system -r pyproject.toml          # reproducible install from lockfile
COPY . .
RUN python manage.py collectstatic --noinput           # bake static into the image
RUN useradd -m app && chown -R app /app
USER app                                               # never run as root
EXPOSE 8000
# Run migrations at startup, then the app server:
CMD ["sh", "-c", "python manage.py migrate --noinput && \
     gunicorn config.wsgi:application -b 0.0.0.0:8000 --workers 4"]
```

> **Best practice:** run `migrate` as a **gated, one-shot deploy step** (an init container / pre-deploy job), not concurrently across many starting replicas — otherwise several instances race to migrate. See Section 5.4.

### 18.4 Database config & connection pooling **[A]**

Use **PostgreSQL** in production (see [PostgreSQL](POSTGRESQL_GUIDE.md) / [Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md)). Each Django process opening a fresh connection per request is wasteful; pool connections:

```python
DATABASES = {"default": env.db("DATABASE_URL")}      # postgres://...
DATABASES["default"]["CONN_MAX_AGE"] = 60            # persistent connections (reuse 60s)
DATABASES["default"]["CONN_HEALTH_CHECKS"] = True    # validate reused connections
```

⚡ **Version note.** Django **5.1+** can use **psycopg 3's built-in connection pool** directly (configure `"pool": True` in the backend `OPTIONS`). For heavy concurrency, **PgBouncer** in front of Postgres is the standard external pooler (use `transaction` pooling carefully with persistent connections).

### 18.5 The deployment checklist **[A]**

```bash
python manage.py check --deploy        # audits DEBUG, SECRET_KEY, SECURE_* settings, etc.
```

The non-negotiables: `DEBUG=False`; real `ALLOWED_HOSTS`; `SECRET_KEY` from env; HTTPS + HSTS + secure cookies (Section 19); `collectstatic` run; migrations applied; logging configured to stdout/stderr for your platform to collect; health-check endpoint; graceful restarts (Gunicorn `--graceful-timeout`, drain connections on deploy).

### 18.6 Logging & observability **[A]**

```python
# settings.py — log to stdout (12-factor); the platform aggregates it
LOGGING = {
    "version": 1, "disable_existing_loggers": False,
    "handlers": {"console": {"class": "logging.StreamHandler"}},
    "root": {"handlers": ["console"], "level": "INFO"},
    "loggers": {"django.request": {"level": "ERROR", "propagate": True}},
}
```

Add **Sentry** (errors), a metrics/APM tool, and request tracing for real observability. Never log secrets or full request bodies with PII.

---

## 19. Security — Built-in Protections, SECURE_* Settings, HTTPS/HSTS, Secrets, check --deploy, Dependencies

Security is Django's strongest selling point: most common web vulnerabilities are handled **by default**. Your job is to not disable them and to set the production flags.

### 19.1 The built-in protections **[A]**

| Threat | Django's defense |
|---|---|
| **SQL injection** | the ORM parameterizes all queries; raw SQL must use params (Section 4.9) |
| **XSS** | templates **auto-escape** all output; only `|safe` opts out |
| **CSRF** | `CsrfViewMiddleware` + `{% csrf_token %}` on by default |
| **Clickjacking** | `XFrameOptionsMiddleware` sets `X-Frame-Options: DENY` |
| **Password storage** | strong hashing (PBKDF2/Argon2), never plaintext |
| **Host-header attacks** | `ALLOWED_HOSTS` validation |
| **Open redirects** | `redirect()`/login `next` validate URLs |

### 19.2 The `SECURE_*` / cookie settings **[A]**

The production hardening set (also shown in Section 10.5):

```python
SECURE_SSL_REDIRECT = True                  # HTTP -> HTTPS
SECURE_HSTS_SECONDS = 31536000              # tell browsers "HTTPS only" for a year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True                  # eligible for the HSTS preload list
SECURE_CONTENT_TYPE_NOSNIFF = True          # X-Content-Type-Options: nosniff
SESSION_COOKIE_SECURE = True                # session cookie only over HTTPS
CSRF_COOKIE_SECURE = True                   # CSRF cookie only over HTTPS
SESSION_COOKIE_HTTPONLY = True              # JS can't read the session cookie (default)
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")  # trust Nginx's header
CSRF_TRUSTED_ORIGINS = ["https://example.com"]
X_FRAME_OPTIONS = "DENY"
```

> ⚡ **Caution:** only set `SECURE_PROXY_SSL_HEADER` when Django is genuinely behind a proxy you control that sets `X-Forwarded-Proto` — trusting it without that proxy lets clients spoof HTTPS.

### 19.3 Secrets, the audit, and dependencies **[A]**

- **Secrets** never in source: `SECRET_KEY`, DB passwords, API keys all come from env/secret manager (Section 10). Add a Content-Security-Policy (via `django-csp`) to further harden against XSS.
- **Audit before every deploy:** `python manage.py check --deploy` flags `DEBUG=True`, weak/missing `SECRET_KEY`, missing `SECURE_*` settings, and more.
- **Dependency security:** pin and lock versions; run `pip-audit`/`safety` (or GitHub Dependabot) to catch known CVEs; keep Django on a **supported (security-patched) release** — staying on an EOL version is itself a vulnerability.
- **File uploads:** validate type/size; store user media on a non-executable, separate domain/bucket so a malicious upload can't be served as active content (Section 7.6).

---

## 20. Performance & Scaling — ORM Optimization, Indexing, Caching, Pagination, Pooling, Horizontal Scaling, Profiling

### 20.1 ORM query optimization — the biggest wins **[A]**

90% of Django performance problems are **database** problems, and most are the **N+1** issue (Section 4.6). The toolkit:

- **`select_related` / `prefetch_related`** — kill N+1 (the #1 fix). Audit every list view and serializer.
- **`only()` / `defer()`** — fetch only the columns you need (or skip heavy ones like big `TextField`s): `Post.objects.only("id", "title")`.
- **`values()` / `values_list()`** — when you need dicts/tuples, not full model instances (lighter, no instance construction).
- **`.count()` not `len(qs)`** — let the DB count; `len()` loads every row.
- **`.exists()`** for "are there any?" — cheaper than fetching.
- **`.explain()`** — print the database's query plan to spot missing indexes or sequential scans: `print(qs.explain())`.
- **`iterator()`** — stream huge result sets without loading them all into memory.

```python
qs = (Post.objects.filter(status="published")
      .select_related("author")                 # JOIN authors (no N+1)
      .prefetch_related("tags")                 # batch-load tags
      .only("id", "title", "author__username")) # fetch minimal columns
print(qs.explain())                              # inspect the query plan
```

### 20.2 Indexing **[A]**

Add database indexes on columns you **filter, order, or join** on frequently. Declare them on the model so they live in migrations (see [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md) for which to add):

```python
class Meta:
    indexes = [
        models.Index(fields=["status", "-created_at"]),     # composite, for list queries
        models.Index(fields=["slug"]),
    ]
    constraints = [
        models.UniqueConstraint(fields=["author", "slug"], name="uniq_author_slug"),
    ]
```

### 20.3 Pagination, pooling, caching, horizontal scaling **[A]**

- **Pagination** — never return unbounded lists. Use Django's `Paginator` (templates) or DRF pagination (APIs); prefer **cursor pagination** for very large or fast-changing datasets (deep `OFFSET` is slow).
- **Connection pooling** — `CONN_MAX_AGE` / psycopg3 pool / **PgBouncer** (Section 18.4) to avoid per-request connection overhead.
- **Caching** — Redis-backed per-view/fragment/low-level caching (Section 14) for read-heavy pages.
- **Horizontal scaling** — Django is **stateless** if you keep state in shared services (DB, Redis sessions/cache). That lets you run **N identical app instances** behind a load balancer and scale out. Keep nothing in process-local memory that must persist.

### 20.4 Profiling **[A]**

- **django-debug-toolbar** — in development, shows per-request **SQL queries (and their count!)**, timings, templates, cache hits. The fastest way to catch N+1.
- **django-silk** — request/SQL profiling you can run in staging.
- **`QuerySet.explain()`**, DB slow-query logs, and APM (Sentry Performance, etc.) for production.

```python
# settings/dev.py — wire up the debug toolbar
INSTALLED_APPS += ["debug_toolbar"]
MIDDLEWARE.insert(0, "debug_toolbar.middleware.DebugToolbarMiddleware")
INTERNAL_IPS = ["127.0.0.1"]
```

---

## 21. Gotchas & Best Practices

A consolidated list of the mistakes that bite Django developers, with the fix:

| Gotcha | Why it hurts | Fix |
|---|---|---|
| **Not creating a custom user model on day one** | swapping `AUTH_USER_MODEL` after migrations is brutal | define `accounts.User(AbstractUser)` **before the first `migrate`** (Section 9.6) |
| **The N+1 query problem** | one query per row in a loop → slow pages | `select_related` (FK/1-1) + `prefetch_related` (M2M/reverse) (Section 4.6) |
| **QuerySets are lazy — and re-evaluated** | iterating the same qs twice = two queries; surprise queries in templates | cache with `list(qs)`; know when evaluation happens (Section 4.3) |
| **`DEBUG=True` in production** | leaks stack traces, settings, source; disables protections | `DEBUG=False` from env; `check --deploy` (Sections 10, 19) |
| **Secrets in source / a committed `.env`** | leaked keys = full compromise | env vars / secret manager; rotate `SECRET_KEY` (Section 10) |
| **Forgetting `ALLOWED_HOSTS`** | host-header attacks; 400s in prod | list real domains (Section 10) |
| **Risky migrations on live tables** | locks/outages during deploy | backward-compatible, multi-step, concurrent indexes (Section 5.4) |
| **`.update()`/`.delete()` skip signals & `save()`** | custom logic silently bypassed | iterate+save when you need signals; know the trade-off (Section 4.4) |
| **Signal overuse** | implicit cascades are hard to trace/test | prefer explicit service calls; use signals sparingly (Section 17) |
| **Fat views / business logic in views** | untestable, unmaintainable | thin views, fat models, a service layer (Section 17.2) |
| **Money in `FloatField`** | rounding errors | `DecimalField(max_digits, decimal_places)` (Section 4.1) |
| **`on_delete` carelessness** | accidental cascade deletes or orphans | choose `CASCADE`/`PROTECT`/`SET_NULL` deliberately (Section 4.5) |
| **Importing `auth.User` directly** | breaks with a custom user model | `settings.AUTH_USER_MODEL` / `get_user_model()` (Section 9.6) |
| **Calling sync ORM in async views** | `SynchronousOnlyOperation` | use `a`-prefixed methods or `sync_to_async` (Section 13.2) |
| **No pagination on list endpoints** | unbounded responses, OOM | paginate everything (Sections 12.5, 20.3) |
| **Marking untrusted data `|safe`** | reintroduces XSS | only `|safe` fully-trusted/sanitized content (Section 8.3) |
| **Running on EOL Django** | unpatched CVEs | stay on a supported LTS (5.2); upgrade on cadence (Section 19) |

---

## 22. Study Path & Build-to-Learn Projects

A staged path from "knows Python" to "ships production Django." Build each project, don't just read — the ORM and the request lifecycle only click by doing.

**Stage 1 — Foundations (Sections 1–3, 8).** Set up a venv, `startproject`/`startapp`, and build a **static multi-page site** with URLs, function views, templates, inheritance, and static files. Goal: internalize the request → URL → view → template flow.

**Stage 2 — The ORM & the admin (Sections 4–6).** Build a **personal blog**: `Post`/`Category`/`Tag`/`Comment` models with relationships; master the QuerySet API, `select_related`/`prefetch_related`, and migrations. Register everything in the admin and customize `ModelAdmin`. Goal: think in models and querysets; feel why the admin saves weeks.

**Stage 3 — Forms, auth, and a real app (Sections 7, 9, 10, 11).** Turn the blog into a **multi-user app**: a **custom user model from day one**, login/logout/registration, `login_required`, permissions/groups so authors edit only their own posts, forms with validation and file uploads, split settings with env vars. Goal: a secure, multi-user CRUD app.

**Stage 4 — A DRF API with JWT (Section 12).** Expose the blog as a **REST API**: `ModelSerializer`s, a `ModelViewSet` + router, **JWT auth**, object-level permissions (author-or-read-only), pagination, filtering, throttling, and `drf-spectacular` docs. Optionally build a small React/Vue SPA against it. Goal: production-grade API design.

**Stage 5 — Async, caching & background work (Sections 13–15).** Add **Redis caching** (per-view + low-level) to hot pages, a **Celery + Redis** background task (email on signup, thumbnail generation) with **Beat** scheduling, and one **async view** calling an external API. Goal: keep requests fast by offloading and caching.

**Stage 6 — Test, harden & deploy (Sections 16, 18, 19, 20).** Write a **pytest-django** suite with **factory_boy** factories covering models, views, and the API; profile with **debug-toolbar** and kill the N+1s; then **deploy**: Dockerized Gunicorn/Uvicorn behind [Nginx](NGINX_GUIDE.md) with PostgreSQL, Redis, WhiteNoise/CDN static, `DEBUG=False`, the full `SECURE_*` set, and a green `check --deploy`. Goal: a real, secured, observable production deployment.

**Capstone.** Build a **complete SaaS-style product** (e.g. a project tracker or a small store): custom user + teams/permissions, a rich admin for staff, a versioned DRF API consumed by an SPA, Celery for emails/reports/scheduled jobs, Redis cache + sessions, comprehensive tests, and a Dockerized deploy with CI running migrations and the security audit. This exercises every section of this guide end to end — and is, in miniature, exactly how production Django is built.

Keep the official docs (**docs.djangoproject.com/en/5.2/**) and the [Python](PYTHON_GUIDE.md), [PostgreSQL](POSTGRESQL_GUIDE.md), [Redis](REDIS_GUIDE.md), [Nginx](NGINX_GUIDE.md), and [Docker](DOCKER_GUIDE.md) guides at hand as you go.
