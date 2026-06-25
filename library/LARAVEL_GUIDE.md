# Laravel — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** A developer who already knows **basic PHP** (variables, arrays, functions, classes, namespaces, Composer) and wants to become a confident, production-grade **Laravel** developer — the dominant PHP web framework. If your PHP is rusty, read the [PHP](PHP_GUIDE.md) guide first; this guide *assumes* you can read a `class`, a `namespace`, a closure, and a Composer autoload. Everything here is explained **prose-first**: for each concept you get *what it is, why it exists, when and how to use it, the key options/parameters, best practices, and the security implications* — and only **then** the heavily-commented, runnable code. Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **Laravel 12** (released **24 February 2025**) on **PHP 8.2+**, the established current version as of **June 2026**. Key facts the guide is built on:
> - Laravel does **yearly major releases (~Q1)**. **Laravel 11** (March 2024) introduced the **slimmed application skeleton**: there is **no `app/Http/Kernel.php`** and **no `app/Console/Kernel.php`** anymore — middleware, exceptions, and routing are configured in **`bootstrap/app.php`**; the `config/` files are minimal; `app/Providers/AppServiceProvider.php` is the one provider you get by default. **Laravel 12 keeps this exact structure** with minimal breaking changes (it was largely a maintenance release focused on dependency updates).
> - ⚡ **Laravel 13** is expected **~Q1 2026**, so by June 2026 it is the newest major. This guide's worked content stays on **Laravel 12**, which 13 is broadly compatible with. Where 13 matters it is flagged with **⚡ Version note** — no invented 13-only APIs appear here.
> - **Starter kits:** the old **Breeze/Jetstream** are superseded by the **new official starter kits (React, Vue, Livewire)** shipped with Laravel 12 — React/Vue are built on **Inertia 2**, with optional **WorkOS AuthKit** for hosted auth. (Breeze/Jetstream still work; they're just no longer the headline.)
> - **First-party packages (2026):** **Reverb** (first-party WebSocket server), **Folio** (page-based routing), **Volt** (functional Livewire), **Pennant** (feature flags), **Pulse** (app monitoring), **Sanctum** (API/SPA tokens), **Horizon** (Redis queues), **Octane** (high-perf via Swoole/FrankenPHP), **Pint** (formatter), **Sail** (Docker), **Herd** (local env). **Pest** is the now-popular test runner alongside **PHPUnit**.
>
> Cross-references appear throughout to the [PHP](PHP_GUIDE.md), [Networking](NETWORKING_GUIDE.md), [Nginx](NGINX_GUIDE.md), [Docker](DOCKER_GUIDE.md), [PostgreSQL](POSTGRESQL_GUIDE.md), [SQLite3](SQLITE3_GUIDE.md), [Redis](REDIS_GUIDE.md), [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md), [Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md), [React 19](REACT_19_GUIDE.md), and [Git](GIT_GUIDE.md) guides. Authoritative source to confirm details offline-permitting: **laravel.com/docs/12.x**.

---

## Table of Contents

1. [What Laravel Is & the 2026 Ecosystem — MVC, Philosophy, the Request Lifecycle](#1-what-laravel-is--the-2026-ecosystem--mvc-philosophy-the-request-lifecycle) **[B]**
2. [Setup — Installer, Herd, Sail/Docker, .env, the Slimmed Structure, Artisan](#2-setup--installer-herd-saildocker-env-the-slimmed-structure-artisan) **[B]**
3. [Routing — Web vs API, Params, Named Routes, Model Binding, Groups](#3-routing--web-vs-api-params-named-routes-model-binding-groups) **[B/I]**
4. [Controllers, Requests & Responses](#4-controllers-requests--responses) **[B/I]**
5. [Blade Templating & the Frontend Story (Vite, Starter Kits, Livewire)](#5-blade-templating--the-frontend-story-vite-starter-kits-livewire) **[B/I]**
6. [The Service Container & Dependency Injection — the Heart of Laravel](#6-the-service-container--dependency-injection--the-heart-of-laravel) **[I/A]**
7. [Service Providers & Bootstrapping — bootstrap/app.php](#7-service-providers--bootstrapping--bootstrapappphp) **[I/A]**
8. [Eloquent ORM — Models, Migrations, Relationships, Eager Loading, Casts](#8-eloquent-orm--models-migrations-relationships-eager-loading-casts) **[I/A]**
9. [The Query Builder & Raw DB Access](#9-the-query-builder--raw-db-access) **[I]**
10. [Validation — validate(), Form Requests, Custom Rules](#10-validation--validate-form-requests-custom-rules) **[I]**
11. [Middleware in Depth](#11-middleware-in-depth) **[I]**
12. [Authentication & Authorization — Guards, Gates, Policies, Sanctum](#12-authentication--authorization--guards-gates-policies-sanctum) **[I/A]**
13. [Building REST APIs — Resources, Pagination, Versioning, Rate Limiting](#13-building-rest-apis--resources-pagination-versioning-rate-limiting) **[I/A]**
14. [Queues & Jobs — Workers, Drivers, Batching, Horizon](#14-queues--jobs--workers-drivers-batching-horizon) **[A]**
15. [Events, Listeners & Broadcasting with Reverb](#15-events-listeners--broadcasting-with-reverb) **[A]**
16. [Artisan & Task Scheduling](#16-artisan--task-scheduling) **[I/A]**
17. [Caching, Sessions & Redis](#17-caching-sessions--redis) **[I]**
18. [Mail & Notifications](#18-mail--notifications) **[I]**
19. [File Storage — the Filesystem Abstraction](#19-file-storage--the-filesystem-abstraction) **[I]**
20. [Testing — Pest & PHPUnit](#20-testing--pest--phpunit) **[A]**
21. [Production & Performance — Caching, Octane, Deployment, Hardening](#21-production--performance--caching-octane-deployment-hardening) **[A]**
22. [Gotchas & Best Practices](#22-gotchas--best-practices) **[I/A]**
23. [Study Path & Build-to-Learn Projects](#23-study-path--build-to-learn-projects)

---

## 1. What Laravel Is & the 2026 Ecosystem — MVC, Philosophy, the Request Lifecycle

### 1.1 The one-paragraph definition **[B]**

**Laravel is a full-stack PHP web framework that gives you a complete, batteries-included foundation for building web applications and APIs.** Where raw PHP gives you a language and a pile of `superglobals` (`$_GET`, `$_POST`, `$_SESSION`), Laravel gives you an opinionated *architecture*: a router that maps URLs to code, a powerful ORM (Eloquent) so you talk to your database in objects instead of SQL strings, a templating engine (Blade), a dependency-injection container, a queue system, a mail layer, an auth system, a test harness, and dozens of other subsystems — all designed to fit together. The selling point is **developer happiness through expressive, readable code**: things that are tedious in plain PHP (validation, auth, sending email, background jobs, WebSockets) become a few lines.

Laravel was created by **Taylor Otwell** in 2011 and is, in 2026, by a wide margin the **dominant PHP framework** and one of the most-used backend frameworks of any language. It is open-source (MIT), with a large commercial ecosystem (Forge, Vapor, Nova, Envoyer, Cloud) funding ongoing development.

### 1.2 MVC — and "and more" **[B]**

Laravel is usually described as **MVC** (Model-View-Controller), and that's the right starting mental model:

- **Model** — a PHP class representing a database table / business entity (an Eloquent model like `App\Models\Post`). It knows how to load, save, and relate data.
- **View** — what the user sees: a Blade template (`resources/views/posts/show.blade.php`) that renders HTML, or — for an API — a JSON resource.
- **Controller** — the glue: a class whose methods receive a request, ask models for data, and return a view or JSON response.

But "MVC" undersells it. The pieces that make Laravel *Laravel* sit **around** MVC: the **service container** (Section 6) that wires everything together, **middleware** (Section 11) that filters requests, **service providers** (Section 7) that bootstrap features, **the event system** (Section 15), **queues** (Section 14), **the validation layer** (Section 10), and so on. A good Laravel developer thinks less "MVC" and more "a request flows through a pipeline of well-defined stages."

### 1.3 The philosophy — convention over configuration **[B]**

Laravel leans hard on **convention over configuration**: if you name and place things the Laravel way, everything "just works" with zero wiring. A model named `Post` automatically maps to a `posts` table; a controller method type-hinting `Post $post` automatically loads the right row (route-model binding); a migration in `database/migrations` is found automatically. You *can* override every convention, but you rarely need to. This is why Laravel apps from different teams look broadly similar — and why this guide can teach you "the Laravel way" rather than "one team's way."

The other pillar is **expressiveness**: APIs are designed to read like English. `User::where('active', true)->orderBy('name')->get()` says what it does. `Mail::to($user)->send(new WelcomeEmail())` says what it does. You'll see this everywhere — fluent, chainable, readable.

### 1.4 Release cadence & versions **[B]**

⚡ **Version note.** Laravel ships **one major version per year, around Q1**, each supported with bug fixes for ~18 months and security fixes for ~2 years. The relevant timeline:

| Version | Released | Notable |
|---|---|---|
| Laravel 9 | Feb 2022 | First to require PHP 8.0; moved to Symfony 6 |
| Laravel 10 | Feb 2023 | Native types throughout the skeleton |
| **Laravel 11** | **March 2024** | **Slimmed skeleton** — no `Http/Kernel.php`, config in `bootstrap/app.php`; per-second rate limiting; health route |
| **Laravel 12** | **24 Feb 2025** | **Current.** Mostly maintenance + dependency updates; **new official starter kits** (React/Vue/Livewire) replace Breeze/Jetstream as the headline |
| **Laravel 13** | **~Q1 2026 (expected)** | The newest major by June 2026; broadly compatible with 12. Confirm specifics against the official docs — this guide does not invent 13-only APIs. |

**Requirement:** Laravel 12 needs **PHP 8.2 or newer** (8.3/8.4 fully supported). The biggest *practical* thing to internalize is the **slimmed skeleton** introduced in 11 and kept in 12 — Section 2.5 and Section 7 cover it in detail, because confusing it with the old (Laravel 8–10) layout is the #1 way to look out of date.

### 1.5 The first-party package universe (2026) **[B]**

Part of "Laravel" is a constellation of **official packages** you install when you need them. Knowing they exist saves you from reinventing or reaching for inferior third-party options:

| Package | What it does | Section |
|---|---|---|
| **Sanctum** | API tokens **and** SPA cookie auth | 12 |
| **Fortify** | Headless auth backend (login/register/2FA logic, no UI) | 12 |
| **Socialite** | OAuth login (Google, GitHub, …) | 12 |
| **Horizon** | Dashboard + supervisor for **Redis** queues | 14 |
| **Reverb** | First-party **WebSocket server** for broadcasting | 15 |
| **Echo** | JS client library for broadcasting | 15 |
| **Pennant** | Feature flags | — |
| **Pulse** | Real-time app performance/monitoring dashboard | 21 |
| **Telescope** | Local debugging dashboard (requests, queries, jobs) | 20/21 |
| **Octane** | Run the app in a long-lived worker (Swoole/FrankenPHP) for huge throughput | 21 |
| **Folio** | Page-based routing (file = route) | 5 |
| **Volt** | Functional, single-file **Livewire** components | 5 |
| **Livewire** | Build dynamic UIs in PHP (no/little JS) | 5 |
| **Pint** | Opinionated code formatter (PHP-CS-Fixer wrapper) | 2/21 |
| **Sail** | Docker dev environment | 2 |
| **Herd** | Native macOS/Windows local PHP environment (no Docker) | 2 |
| **Dusk** | Browser (end-to-end) testing | 20 |

### 1.6 The request lifecycle — the single most important diagram **[B]**

To debug Laravel, you must understand **what happens between a browser hitting your URL and a response coming back.** Here is the full journey for a normal web request, with the modern (Laravel 11/12) file names:

```
                       ┌─────────────────────────────────────────────┐
  Browser ──HTTP──▶    │ Nginx / Apache  (web server)                 │
                       │  - serves static files directly              │
                       │  - sends PHP requests to PHP-FPM             │  ← see NGINX_GUIDE.md
                       └───────────────────────┬─────────────────────┘
                                               ▼
                         public/index.php   ← the ONE entry point
                                               │  (the web root is public/, not the project root)
                                               ▼
                         require bootstrap/app.php
                                               │  builds & configures the Application (the container)
                                               │  registers middleware, exceptions, routing  ← Laravel 11+!
                                               ▼
                         HTTP Kernel (Illuminate\Foundation\Http\Kernel)
                                               │  (framework-internal now; you no longer edit it)
                                               ▼
                         Global middleware pipeline
                                               │  (maintenance mode, trim strings, CORS, …)
                                               ▼
                         Router  (routes/web.php or routes/api.php)
                                               │  matches the URL + method to a route
                                               ▼
                         Route middleware (auth, throttle, your own…)
                                               ▼
                         Controller / route closure  ──▶  Models (Eloquent) ──▶ Database
                                               │
                                               ▼  returns a View / JSON / Redirect
                         Response travels back UP through the middleware (after-stage)
                                               ▼
                         public/index.php sends the HTTP response ──▶ Browser
```

Three things to burn into memory:

1. **`public/index.php` is the only entry point.** Every request — every page, every API call, every asset that isn't a literal static file — goes through it. Your web server's document root must be the `public/` directory, never the project root (Section 21 covers why this matters for security: it keeps `.env`, your code, and `vendor/` out of web reach).
2. **`bootstrap/app.php` is where the app is configured** in Laravel 11/12. This replaced the old `app/Http/Kernel.php` + `app/Console/Kernel.php` + a pile of `config/*.php`. If a tutorial tells you to "edit `Kernel.php` to add middleware," it's pre-Laravel-11 — translate it to `bootstrap/app.php` (Section 7).
3. **Middleware wraps the controller** like an onion: requests pass *down* through middleware to the controller, and the response passes back *up* through them. This is the **pipeline pattern** and it's how auth, CSRF, sessions, and rate limiting all hook in.

---

## 2. Setup — Installer, Herd, Sail/Docker, .env, the Slimmed Structure, Artisan

### 2.1 What you need first **[B]**

Laravel needs **PHP 8.2+**, **Composer** (PHP's package manager), and a database (SQLite, MySQL/MariaDB, or [PostgreSQL](POSTGRESQL_GUIDE.md)). On a fresh machine you don't install these by hand anymore — you use one of the environments below.

### 2.2 Three ways to get a local environment **[B]**

| Tool | What it is | Best when |
|---|---|---|
| **Herd** | A free native app (macOS **and Windows**) that bundles PHP, Nginx, and a DNS for `*.test` domains. Zero config; near-instant. | Solo/local dev, the fastest path. The author is on **Windows 11** — Herd works here. |
| **Sail** | A thin Docker wrapper Laravel ships. `docker-compose` with PHP, MySQL/Postgres, Redis, Mailpit, etc. | You want your dev env to mirror production containers — see [Docker](DOCKER_GUIDE.md). |
| **Manual** | Install PHP + Composer + a DB yourself; run `php artisan serve`. | CI, servers, or when you want full control. |

### 2.3 Creating a new project **[B]**

The modern way is the **`laravel` installer** (a global Composer tool). It can scaffold a starter kit interactively:

```bash
# 1. Install the global installer once (skip if you already have it):
composer global require laravel/installer

# 2. Create a new app. The installer ASKS which starter kit (none / React / Vue / Livewire),
#    which testing framework (Pest or PHPUnit), and which database.
laravel new blog

# Equivalent without the global installer:
composer create-project laravel/laravel blog

cd blog
```

⚡ **Version note.** With Laravel 12, choosing **React** or **Vue** scaffolds an **Inertia 2** app (optionally with **WorkOS AuthKit**); choosing **Livewire** scaffolds a Livewire + Volt app. The old `laravel new blog --breeze` / `--jet` flags are legacy — the interactive starter-kit picker replaced them. Choose **"none"** for a clean API or to learn Blade from scratch (recommended while studying).

Run it:

```bash
# The simplest dev server (uses PHP's built-in server, good enough for local dev):
php artisan serve            # serves http://127.0.0.1:8000

# Or with Herd, the folder is auto-served at http://blog.test — nothing to run.
```

### 2.4 The `.env` file — configuration & secrets **[B]**

Laravel reads environment-specific configuration from a **`.env`** file in the project root. This is where database credentials, app keys, mail settings, and any secret live. **`.env` is git-ignored** (a `.env.example` template is committed instead) — never commit real secrets.

```env
APP_NAME=Blog
APP_ENV=local                 # local | production | testing
APP_KEY=base64:...            # generated by `php artisan key:generate`; used for encryption/sessions
APP_DEBUG=true                # MUST be false in production (it leaks stack traces — Section 21)
APP_URL=http://localhost

# Laravel 11+ defaults to SQLITE for zero-setup local dev:
DB_CONNECTION=sqlite
# For MySQL/Postgres instead, set these (and comment the sqlite line):
# DB_CONNECTION=pgsql
# DB_HOST=127.0.0.1
# DB_PORT=5432
# DB_DATABASE=blog
# DB_USERNAME=blog
# DB_PASSWORD=secret

# Laravel 11+ defaults many subsystems to the database driver so a fresh app runs with no Redis:
SESSION_DRIVER=database
QUEUE_CONNECTION=database
CACHE_STORE=database
MAIL_MAILER=log               # writes emails to the log instead of sending, in local dev
```

How you *read* these in code is via the **`config()` helper**, never `env()` directly outside config files (this is a real gotcha — see 2.4.1).

#### 2.4.1 `env()` vs `config()` — the trap **[B]**

`env('FOO')` reads the `.env` file. **You must only call `env()` inside `config/*.php` files.** Everywhere else, read configuration with `config('app.name')`. Why? Because in production you run `php artisan config:cache`, which compiles all config into one file and **stops loading `.env` entirely** — so any `env()` call in a controller/model returns `null`. This breaks apps in production in a way that works fine locally. (Gotcha repeated in Section 22.)

```php
// config/services.php — env() is fine HERE:
return [
    'stripe' => ['key' => env('STRIPE_KEY')],
];

// A controller — read it through config(), NOT env():
$key = config('services.stripe.key');   // ✅ survives config:cache
```

### 2.5 The Laravel 11/12 slimmed directory structure **[B]**

⚡ **Version note — read this carefully; it is the thing most likely to make older tutorials confusing.** Since **Laravel 11**, the skeleton is deliberately minimal. Here is what you actually get:

```
blog/
├── app/
│   ├── Http/
│   │   ├── Controllers/        # your controllers
│   │   └── Middleware/         # ONLY custom middleware you create (none ship by default now)
│   ├── Models/                 # Eloquent models (User.php ships here)
│   └── Providers/
│       └── AppServiceProvider.php   # THE service provider — register/boot your services here
├── bootstrap/
│   ├── app.php                 # ⭐ THE config hub: middleware, routing, exceptions
│   ├── providers.php           # the list of your service providers (replaces config/app.php's array)
│   └── cache/                  # framework cache files
├── config/                     # minimal; run `php artisan config:publish <name>` to materialize one
├── database/
│   ├── factories/  migrations/  seeders/
├── public/
│   └── index.php               # the single entry point (web root)
├── resources/
│   ├── views/                  # Blade templates
│   ├── css/  js/               # Vite-compiled assets
├── routes/
│   ├── web.php                 # browser routes (session, CSRF)
│   ├── console.php             # Artisan commands + the SCHEDULER lives here now
│   └── api.php                 # ONLY after `php artisan install:api` (Section 3.7)
├── storage/                    # logs, compiled views, file uploads, caches
├── tests/                      # Pest/PHPUnit tests
├── .env                        # config & secrets (git-ignored)
├── artisan                     # the CLI entry point
├── composer.json
└── vite.config.js
```

**What is gone vs Laravel 8–10:**

| Old (≤ L10) | New (L11/L12) |
|---|---|
| `app/Http/Kernel.php` (global + route middleware lists) | **Gone.** Configure middleware in `bootstrap/app.php` (Section 7.3) |
| `app/Console/Kernel.php` (commands + schedule) | **Gone.** Commands auto-register from `app/Console/Commands`; the **schedule lives in `routes/console.php`** (Section 16) |
| `app/Exceptions/Handler.php` | **Gone.** Configure exception handling in `bootstrap/app.php` (Section 7.4) |
| `app/Http/Middleware/*` (8 default files) | **Gone** as files; the behavior is built-in and tuned in `bootstrap/app.php` |
| Many `config/*.php` files | **Minimal** — most use sensible defaults; publish one only when you need to change it |
| `config/app.php` providers array | `bootstrap/providers.php` |
| `App\Providers\RouteServiceProvider` | **Gone** — routing configured in `bootstrap/app.php` |

If a tutorial says "open `Kernel.php`," mentally translate to `bootstrap/app.php`. That single substitution fixes 90% of "this doesn't match my project" confusion.

### 2.6 Artisan basics **[B]**

**Artisan** is Laravel's command-line tool (`php artisan`). You'll use it constantly — to generate files, run migrations, clear caches, start workers, and inspect the app. The most-used commands:

```bash
php artisan list                 # every available command
php artisan help make:model      # help for one command

# --- Generators (scaffolding) ---
php artisan make:model Post -mcr  # model + migration + controller (resource) in one shot
php artisan make:controller PostController --resource
php artisan make:migration create_posts_table
php artisan make:request StorePostRequest
php artisan make:middleware EnsureTokenIsValid
php artisan make:job ProcessPodcast
php artisan make:command SendReports

# --- Database ---
php artisan migrate              # run pending migrations
php artisan migrate:fresh --seed # drop all tables, re-migrate, run seeders (DESTRUCTIVE — dev only)
php artisan db:seed

# --- Inspection / maintenance ---
php artisan route:list           # every registered route (super useful)
php artisan tinker               # an interactive REPL with your app booted (try `User::count()`)
php artisan about                # environment + versions summary

# --- Caches (mostly for production; see Section 21) ---
php artisan optimize             # cache config + routes + views + events at once
php artisan optimize:clear       # clear them all
```

`-mcr` flags on `make:model` are worth memorizing: **m**igration, **c**ontroller, **r**esource. There's also `-f` (factory) and `-s` (seeder). `php artisan make:model Post -a` makes *all* of them.

---

## 3. Routing — Web vs API, Params, Named Routes, Model Binding, Groups

### 3.1 What routing is and why it exists **[B]**

**Routing maps an incoming HTTP request (a method + a URL) to the code that handles it.** Instead of one giant PHP file with `if ($_SERVER['REQUEST_URI'] === ...)`, you declare clean, readable routes. In Laravel, web routes live in `routes/web.php` and (after you opt in) API routes in `routes/api.php`. The difference matters:

- **`routes/web.php`** routes get the **web middleware group**: sessions, cookies, and **CSRF protection** (Section 11). Use these for anything a browser navigates to or submits forms to.
- **`routes/api.php`** routes get the **api middleware group**: **stateless**, no session, no CSRF, and they're automatically prefixed with `/api`. Use these for JSON APIs consumed by JavaScript, mobile apps, or other services. (See Section 3.7 — this file only exists after `php artisan install:api`.)

### 3.2 Basic routes **[B]**

The router exposes a method per HTTP verb. The handler can be a **closure** (fine for tiny things) or, far more commonly, a **`[Controller::class, 'method']` array** (Section 4):

```php
// routes/web.php
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\PostController;

// Closure handler — quick, but don't build a real app out of these:
Route::get('/', function () {
    return view('welcome');          // render resources/views/welcome.blade.php
});

// Controller handler — the production pattern:
Route::get('/posts', [PostController::class, 'index']);
Route::post('/posts', [PostController::class, 'store']);

// One verb per method; match() and any() exist for multiple:
Route::match(['get', 'post'], '/search', [SearchController::class, 'search']);
```

The verbs: `Route::get`, `post`, `put`, `patch`, `delete`, `options`. Browsers only send GET/POST from forms, so to send PUT/PATCH/DELETE from a Blade form you add `@method('PUT')` (Section 5.4) — Laravel reads a hidden `_method` field.

### 3.3 Route parameters **[B]**

Capture parts of the URL with `{braces}`. They're passed to your handler **in order**:

```php
// Required parameter:
Route::get('/posts/{id}', function (string $id) {
    return "Post #$id";
});

// Optional parameter (note the ? and the default):
Route::get('/users/{name?}', function (?string $name = 'guest') {
    return "Hello $name";
});

// Constrain a parameter with a regex (here: id must be digits):
Route::get('/posts/{id}', [PostController::class, 'show'])
     ->whereNumber('id');                       // helper for ->where('id', '[0-9]+')
// Other helpers: ->whereAlpha('slug'), ->whereUuid('id'), ->whereIn('cat', ['a','b'])
```

### 3.4 Named routes — never hard-code URLs **[B]**

Give a route a **name** and refer to it by that name everywhere (in redirects, in Blade links). If the URL changes later, you change it in one place:

```php
Route::get('/posts/{post}', [PostController::class, 'show'])->name('posts.show');
```

```php
// Generate the URL by name — pass parameters as an array:
$url = route('posts.show', ['post' => 42]);   // → /posts/42
return redirect()->route('posts.show', $post); // redirect to it
```

```blade
{{-- In Blade: --}}
<a href="{{ route('posts.show', $post) }}">Read more</a>
```

**Best practice:** name *every* route, using the `resource.action` convention (`posts.index`, `posts.show`, `posts.store`). Resource controllers (Section 4.4) do this for you.

### 3.5 Route model binding — the convention that saves you a query line **[B/I]**

This is one of Laravel's most loved conveniences. If a route parameter's **name matches a type-hinted Eloquent model variable**, Laravel automatically fetches that model by primary key — and returns a **404** if it doesn't exist. You never write `Post::findOrFail($id)`.

```php
// The {post} parameter + a Post $post type-hint = implicit binding:
Route::get('/posts/{post}', function (App\Models\Post $post) {
    return $post;   // already loaded by ID; 404 automatically if not found
});
```

You can bind by a **different column** (e.g. a slug) — useful for pretty URLs:

```php
// Match {post} against the `slug` column instead of `id`:
Route::get('/posts/{post:slug}', [PostController::class, 'show']);
```

**Scoped bindings** ensure a child belongs to its parent (security: stops users guessing IDs across owners):

```php
// Only find the comment if it actually belongs to this post:
Route::get('/posts/{post}/comments/{comment}', function (Post $post, Comment $comment) {
    return $comment;  // scoped: comment must belong to $post, else 404
})->scopeBindings();
```

### 3.6 Groups, prefixes & middleware **[B/I]**

When many routes share a prefix, a middleware, a name-prefix, or a controller, **group** them to avoid repetition:

```php
// Everything here is /admin/*, protected by auth, named admin.*:
Route::middleware('auth')
    ->prefix('admin')
    ->name('admin.')
    ->group(function () {
        Route::get('/dashboard', [DashboardController::class, 'index'])->name('dashboard');
        // → URL: /admin/dashboard, name: admin.dashboard, requires auth
    });

// Share one controller across routes:
Route::controller(PostController::class)->group(function () {
    Route::get('/posts', 'index');
    Route::post('/posts', 'store');
});
```

### 3.7 API routes — `php artisan install:api` **[B/I]**

⚡ **Version note.** In Laravel 11/12 the `routes/api.php` file **does not exist by default** — APIs are opt-in. Run:

```bash
php artisan install:api
```

This creates `routes/api.php`, registers the api routes in `bootstrap/app.php`, and installs **Sanctum** for token auth (Section 12.6). API routes are automatically prefixed with `/api` and use the stateless `api` middleware group:

```php
// routes/api.php
Route::get('/posts', [PostController::class, 'index']);   // → GET /api/posts
Route::middleware('auth:sanctum')->get('/user', fn (Request $r) => $r->user());
```

**Reference: the route helpers cheat sheet**

| Need | Code |
|---|---|
| Name a route | `->name('posts.show')` |
| Constrain a param | `->whereNumber('id')` / `->whereUuid('id')` |
| Bind by column | `{post:slug}` |
| Require auth | `->middleware('auth')` |
| Rate limit | `->middleware('throttle:60,1')` (60/min) |
| Group by prefix | `Route::prefix('admin')->group(...)` |
| Full CRUD set | `Route::resource('posts', PostController::class)` (Section 4.4) |
| List all routes | `php artisan route:list` |

---

## 4. Controllers, Requests & Responses

### 4.1 Why controllers **[B]**

A **controller** is a class that groups related request handlers. Putting logic in controllers (instead of route closures) keeps `routes/web.php` a readable table of contents and makes your handlers testable, injectable, and reusable. They live in `app/Http/Controllers`.

```bash
php artisan make:controller PostController
```

```php
// app/Http/Controllers/PostController.php
namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\Request;

class PostController extends Controller
{
    // GET /posts — list
    public function index()
    {
        $posts = Post::latest()->paginate(15);   // newest first, 15 per page
        return view('posts.index', ['posts' => $posts]);
    }

    // GET /posts/{post} — show one (route-model binding loads it)
    public function show(Post $post)
    {
        return view('posts.show', compact('post'));
    }
}
```

### 4.2 The Request object **[B]**

Type-hint `Illuminate\Http\Request` on any controller method and the container injects the current request. It's your typed, safe replacement for `$_GET`/`$_POST`/`$_FILES`:

```php
use Illuminate\Http\Request;

public function store(Request $request)
{
    $title  = $request->input('title');         // a single field (any method)
    $title  = $request->title;                  // shorthand property access
    $all    = $request->all();                  // every input as an array
    $only   = $request->only(['title', 'body']);
    $has    = $request->has('title');           // bool
    $page   = $request->query('page', 1);       // query-string only, with default
    $file   = $request->file('avatar');         // an UploadedFile (Section 19)
    $isJson = $request->expectsJson();          // content negotiation
    $user   = $request->user();                 // the authenticated user (or null)
    return back();
}
```

### 4.3 Single-action controllers **[B/I]**

When a controller does exactly one thing, give it an `__invoke()` method and reference it without a method name. It reads cleanly for things like `ShowDashboard` or `GenerateReport`:

```bash
php artisan make:controller GenerateReport --invokable
```

```php
class GenerateReport extends Controller
{
    public function __invoke(Request $request) { /* ... */ }
}

// Route — note: no method, just the class:
Route::get('/report', GenerateReport::class);
```

### 4.4 Resource controllers — the CRUD convention **[B/I]**

REST/CRUD has seven standard actions. `Route::resource()` wires all seven to conventionally-named controller methods in one line, and **names them** for you:

```bash
php artisan make:controller PostController --resource --model=Post
```

```php
Route::resource('posts', PostController::class);
// Only some actions:
Route::resource('posts', PostController::class)->only(['index', 'show']);
// API version: no create/edit form routes (they make no sense in JSON):
Route::apiResource('posts', PostController::class);
```

| Verb | URI | Method | Route name |
|---|---|---|---|
| GET | `/posts` | `index` | `posts.index` |
| GET | `/posts/create` | `create` | `posts.create` |
| POST | `/posts` | `store` | `posts.store` |
| GET | `/posts/{post}` | `show` | `posts.show` |
| GET | `/posts/{post}/edit` | `edit` | `posts.edit` |
| PUT/PATCH | `/posts/{post}` | `update` | `posts.update` |
| DELETE | `/posts/{post}` | `destroy` | `posts.destroy` |

### 4.5 Responses & redirects **[B/I]**

What you `return` becomes the HTTP response. Laravel is smart about conversion:

```php
return view('posts.index');                       // HTML (a Blade view)
return 'Hello';                                    // plain text, 200
return ['ok' => true];                             // auto-converted to JSON!
return response()->json(['ok' => true], 201);      // explicit JSON + status
return redirect('/home');                           // 302 redirect
return redirect()->route('posts.show', $post);      // redirect to a named route
return redirect()->back()->withInput();             // back to the form, keeping old input
return redirect()->route('posts.index')
    ->with('status', 'Post created!');              // flash a one-request session message

// Custom headers, status, downloads:
return response('Body', 200)->header('X-Foo', 'bar');
return response()->download($pathToFile, 'invoice.pdf');
return response()->noContent();                     // 204
```

**Returning an array or an Eloquent model auto-JSONs** — handy for quick APIs, but for real APIs use **API Resources** (Section 13) so you control the shape and don't leak columns.

---

## 5. Blade Templating & the Frontend Story (Vite, Starter Kits, Livewire)

### 5.1 What Blade is **[B]**

**Blade is Laravel's templating engine.** A `.blade.php` file is mostly HTML with sprinkles of Blade directives (`@if`, `@foreach`) and `{{ }}` echo tags. Blade compiles to plain PHP and caches it, so it's fast. Its killer feature is **automatic HTML-escaping**: `{{ $var }}` runs the value through `htmlspecialchars`, which prevents **XSS** (Section 21) by default. Views live in `resources/views`.

### 5.2 Echoing & control structures **[B]**

```blade
{{-- resources/views/posts/show.blade.php --}}

{{ $post->title }}              {{-- ESCAPED — safe against XSS. Use this 99% of the time. --}}
{!! $post->html_body !!}        {{-- UNescaped — only for content you KNOW is safe HTML --}}

@if ($post->published)
    <span>Published</span>
@elseif ($post->draft)
    <span>Draft</span>
@else
    <span>Unknown</span>
@endif

@foreach ($comments as $comment)
    <p>{{ $comment->body }}</p>
@endforeach

@forelse ($comments as $comment)      {{-- foreach + an empty-case in one --}}
    <p>{{ $comment->body }}</p>
@empty
    <p>No comments yet.</p>
@endforelse

@auth  ...shown only if logged in...  @endauth
@guest ...shown only if a guest...    @endguest
@can('update', $post) <a href="...">Edit</a> @endcan   {{-- authorization, Section 12 --}}
```

⚠️ **Security:** `{!! !!}` disables escaping — every use is a potential XSS hole. Only use it for HTML you generated or sanitized yourself, never for raw user input.

### 5.3 Layouts with template inheritance & components **[B/I]**

You don't repeat your `<html>` boilerplate on every page. Two approaches exist; **components** are the modern, preferred one.

**Classic inheritance** (`@extends`/`@section`/`@yield`):

```blade
{{-- resources/views/layouts/app.blade.php --}}
<!DOCTYPE html>
<html>
<head><title>@yield('title', 'Blog')</title></head>
<body>
    @yield('content')
</body>
</html>
```

```blade
{{-- resources/views/posts/index.blade.php --}}
@extends('layouts.app')
@section('title', 'All Posts')
@section('content')
    <h1>Posts</h1>
    @foreach ($posts as $post) ... @endforeach
@endsection
```

**Components (preferred, Section 5.3.1).** A **component** is a reusable Blade snippet with its own data. The modern app layout uses an `<x-app-layout>` component with a **slot** for the page body:

```blade
{{-- resources/views/components/app-layout.blade.php --}}
<!DOCTYPE html>
<html>
<head><title>{{ $title ?? 'Blog' }}</title></head>
<body>
    {{ $slot }}          {{-- the page content goes here --}}
</body>
</html>
```

```blade
{{-- a page that uses it --}}
<x-app-layout title="All Posts">
    <h1>Posts</h1>
</x-app-layout>
```

#### 5.3.1 Components & slots in depth **[I]**

Components come in two flavors. **Anonymous components** are just a Blade file in `resources/views/components/` (no class). **Class components** (`php artisan make:component Alert`) pair a Blade view with a PHP class for logic. Data flows in via attributes; `{{ $slot }}` receives the body; **named slots** let you pass multiple blocks:

```blade
{{-- resources/views/components/alert.blade.php --}}
@props(['type' => 'info'])                {{-- declare props with defaults --}}
<div {{ $attributes->merge(['class' => "alert alert-$type"]) }}>
    <strong>{{ $title }}</strong>          {{-- a named slot --}}
    {{ $slot }}                            {{-- the default slot --}}
</div>
```

```blade
{{-- using it: --}}
<x-alert type="danger" class="mt-4">
    <x-slot:title>Heads up!</x-slot:title>
    Something went wrong.
</x-alert>
```

`@props` declares accepted attributes (with defaults); `$attributes->merge()` merges caller-supplied HTML attributes (like `class`) onto the root element — this is how you build a reusable, well-behaved component library.

### 5.4 Forms, CSRF & method spoofing **[B]**

Every POST/PUT/PATCH/DELETE form in a `web` route **must** include a CSRF token, or Laravel rejects it (419 error). Use `@csrf`. To send a non-GET/POST verb, use `@method`:

```blade
<form method="POST" action="{{ route('posts.update', $post) }}">
    @csrf                {{-- hidden _token; prevents Cross-Site Request Forgery --}}
    @method('PUT')       {{-- spoof the HTTP verb to PUT --}}
    <input name="title" value="{{ old('title', $post->title) }}">
    @error('title') <span>{{ $message }}</span> @enderror   {{-- validation error --}}
    <button>Save</button>
</form>
```

`old('title', ...)` repopulates the field with the previously-submitted value after a validation failure (Section 10). `@error` shows the field's validation message.

### 5.5 The frontend story — Vite, starter kits, Inertia, Livewire **[B/I]**

Laravel ships **Vite** for compiling CSS/JS. Your Blade includes assets with the `@vite` directive; in dev Vite hot-reloads, in prod you run `npm run build`:

```blade
{{-- in your <head> --}}
@vite(['resources/css/app.css', 'resources/js/app.js'])
```

```bash
npm install
npm run dev      # dev server with hot reload
npm run build    # production build (run on deploy)
```

For interactive UIs you have three modern paths, and the **new starter kits** (Section 1.5) scaffold each:

| Approach | What it is | When |
|---|---|---|
| **Blade + Alpine.js** | Server-rendered HTML with light JS sprinkles | Simple sites, content sites |
| **Livewire (+ Volt)** | Build dynamic, reactive UIs **in PHP** — server renders, a tiny JS runtime swaps HTML over AJAX. **Volt** makes components single-file & functional. | You want SPA-like interactivity without writing a JS frontend |
| **Inertia 2 + React/Vue** | A real [React 19](REACT_19_GUIDE.md)/Vue SPA, but routing & data come from Laravel controllers (no separate API to build). The React/Vue starter kits use this. | You want a full JS frontend with Laravel as the backend |

⚡ **Version note.** The **React** and **Vue** starter kits are built on **Inertia 2**; the **Livewire** starter kit uses **Volt**. **Folio** (page-based routing) lets a Blade file in `resources/views/pages/` *be* a route with no `routes/web.php` entry. These are the 2026-current frontends; Breeze/Jetstream still exist but are no longer the recommended starting point.

A minimal Livewire/Volt counter, to show the flavor (PHP that re-renders on interaction):

```php
// A Volt single-file component (resources/views/livewire/counter.blade.php)
use function Livewire\Volt\{state};
state(['count' => 0]);
$increment = fn () => $this->count++;
?>
<div>
    <button wire:click="increment">+</button>
    <span>{{ $count }}</span>     {{-- updates without a full page reload --}}
</div>
```

---

## 6. The Service Container & Dependency Injection — the Heart of Laravel

### 6.1 What the container is and why it's everything **[I]**

The **service container** (a.k.a. the IoC container) is the object that **builds and wires together all the other objects in your app.** It's the single most important concept for understanding *how* Laravel works under the hood — middleware, controllers, jobs, events, and facades all flow through it.

Here's the problem it solves. Suppose a controller needs a `PaymentGateway`, which needs an HTTP `Client`, which needs an API key from config. Without a container, every place that creates a controller must also know how to construct that whole tree (`new Controller(new PaymentGateway(new Client(config('...')))))`). That's brittle. **Dependency Injection (DI)** flips it: a class *declares* what it needs (via constructor or method type-hints) and the **container provides it**. You ask for the thing you want; the container figures out how to build it and everything it depends on, recursively.

### 6.2 Automatic resolution — you're already using it **[I]**

The reason you can type-hint `Request $request` in a controller and it just appears is the container. For any class with no constructor dependencies (or whose dependencies are themselves resolvable), the container **auto-resolves** it via reflection — no configuration needed:

```php
// app/Services/Invoicer.php
namespace App\Services;
class Invoicer
{
    public function __construct(private \App\Services\PdfRenderer $pdf) {}
    public function generate(int $orderId): string { /* ... */ return 'invoice.pdf'; }
}

// A controller just type-hints it — the container builds Invoicer AND its PdfRenderer:
class InvoiceController extends Controller
{
    public function show(int $id, \App\Services\Invoicer $invoicer)  // ← injected
    {
        return response()->download($invoicer->generate($id));
    }
}
```

You can also resolve manually when you're outside the injection flow:

```php
$invoicer = app(\App\Services\Invoicer::class);   // ask the container directly
$invoicer = app()->make(Invoicer::class);          // same thing
```

### 6.3 Binding — teaching the container how to build something **[I/A]**

Auto-resolution fails when a class needs something the container *can't* guess — most often an **interface** (you can't `new` an interface) or a value (an API key, a config). You **bind** the recipe in a service provider's `register()` method (Section 7).

```php
// In a service provider's register() method:

// (a) Bind an interface to a concrete implementation.
//     Now type-hinting the INTERFACE anywhere resolves to the concrete class.
$this->app->bind(
    \App\Contracts\PaymentGateway::class,        // when someone asks for this…
    \App\Services\StripeGateway::class           // …give them this
);

// (b) Bind with a closure for full control over construction:
$this->app->bind(\App\Services\Weather::class, function ($app) {
    return new \App\Services\Weather(config('services.weather.key'));
});
```

```php
// Elsewhere, code depends on the ABSTRACTION, not the concrete class —
// swap Stripe for PayPal by changing ONE bind() line. This is the whole point.
class CheckoutController extends Controller
{
    public function __construct(private \App\Contracts\PaymentGateway $gateway) {}
    public function pay() { $this->gateway->charge(/* ... */); }
}
```

### 6.4 Singletons vs transient bindings **[I/A]**

- **`bind()`** creates a **new instance every time** it's resolved (transient).
- **`singleton()`** creates the instance **once** and returns that same instance for the rest of the request (or worker lifetime under Octane — be careful with state there, Section 21).

```php
// One shared instance per request — good for stateful clients, caches, connections:
$this->app->singleton(\App\Services\AnalyticsClient::class, function ($app) {
    return new \App\Services\AnalyticsClient(config('services.analytics.token'));
});

// Bind an already-built object:
$this->app->instance('settings', $loadedSettings);

// scoped() = singleton, but reset between Octane requests (use for request-state):
$this->app->scoped(\App\Services\RequestContext::class);
```

### 6.5 Contextual binding **[A]**

Sometimes two classes need the *same* interface resolved to *different* implementations — e.g. a `PhotoController` should get S3 storage while a `VideoController` gets local. **Contextual binding** expresses exactly that:

```php
use Illuminate\Contracts\Filesystem\Filesystem;

$this->app->when(\App\Http\Controllers\PhotoController::class)
          ->needs(Filesystem::class)
          ->give(fn () => \Illuminate\Support\Facades\Storage::disk('s3'));

$this->app->when(\App\Http\Controllers\VideoController::class)
          ->needs(Filesystem::class)
          ->give(fn () => \Illuminate\Support\Facades\Storage::disk('local'));
```

### 6.6 Facades — the friendly face of the container **[I]**

You've seen `Route::get()`, `DB::table()`, `Cache::get()`. These **facades** are static-looking proxies to objects **resolved from the container**. `Cache::get()` is really "resolve the `cache` service from the container and call `get()` on it." They give terse, readable syntax. The trade-off: they hide their dependencies (a class using `Cache::` doesn't *declare* needing a cache). For testability and clarity, constructor injection is "more correct," but facades are idiomatic and fine — and every facade has a real-time alias and is mockable in tests (`Cache::shouldReceive(...)`). Know that `Str::`, `Arr::`, `Log::`, `Auth::`, `Storage::` are all facades.

---

## 7. Service Providers & Bootstrapping — bootstrap/app.php

### 7.1 What service providers are **[I]**

**Service providers are the central place where your application is bootstrapped** — where you register container bindings, event listeners, middleware, routes, view composers, and configuration. Every feature in Laravel (the database, the queue, the mailer) is registered by a service provider. In the slimmed skeleton you usually have just **`App\Providers\AppServiceProvider`**, and you list any others you add in **`bootstrap/providers.php`**.

A provider has two methods, and the **distinction between them is a classic exam question:**

- **`register()`** — bind things into the container. **Only** put binding code here. You must **not** use other services here, because not all providers have registered yet (order isn't guaranteed).
- **`boot()`** — runs **after all providers have registered**, so here it's safe to *use* services: define routes, register event listeners, share view data, define gates, etc.

```php
// app/Providers/AppServiceProvider.php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Illuminate\Pagination\Paginator;
use App\Contracts\PaymentGateway;
use App\Services\StripeGateway;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // ONLY bindings here:
        $this->app->bind(PaymentGateway::class, StripeGateway::class);
    }

    public function boot(): void
    {
        // Safe to USE services here:
        Paginator::useBootstrapFive();                 // pagination styling
        \Illuminate\Database\Eloquent\Model::shouldBeStrict();  // dev: catch N+1/missing attrs
        \Illuminate\Support\Facades\Gate::define('admin', fn ($user) => $user->is_admin);
    }
}
```

### 7.2 `bootstrap/app.php` — the configuration hub **[I/A]**

⚡ **Version note — the centerpiece of the modern skeleton.** In Laravel 11/12, `bootstrap/app.php` is where you configure routing, middleware, and exception handling. It returns a fluent `Application` builder. Here is a realistic, annotated example:

```php
// bootstrap/app.php
<?php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',          // load web routes
        api: __DIR__.'/../routes/api.php',          // (present after install:api)
        commands: __DIR__.'/../routes/console.php', // console + scheduler
        health: '/up',                               // built-in health-check endpoint
    )
    ->withMiddleware(function (Middleware $middleware) {
        // Configure middleware HERE (this replaces app/Http/Kernel.php):
        $middleware->alias([
            'admin' => \App\Http\Middleware\EnsureUserIsAdmin::class,
        ]);
        $middleware->append(\App\Http\Middleware\TrackVisits::class); // global, runs last
        // Make API stateful for an SPA (Sanctum cookie auth, Section 12.6):
        $middleware->statefulApi();
        // Exclude a URI from CSRF:
        $middleware->validateCsrfTokens(except: ['stripe/webhook']);
    })
    ->withExceptions(function (Exceptions $exceptions) {
        // Customize exception handling HERE (replaces app/Exceptions/Handler.php):
        $exceptions->dontReport(\App\Exceptions\PaymentSkippedException::class);
        $exceptions->render(function (\App\Exceptions\PaymentFailed $e, $request) {
            return response()->json(['error' => 'Payment failed'], 402);
        });
    })
    ->create();
```

### 7.3 Configuring middleware in `bootstrap/app.php` **[I/A]**

The `->withMiddleware()` closure is the **only** place to register middleware in Laravel 11/12. The key methods:

| Method | Purpose |
|---|---|
| `$middleware->append($class)` | Add to the **global** stack, at the **end** |
| `$middleware->prepend($class)` | Add to the global stack, at the **start** |
| `$middleware->alias(['name' => Class::class])` | Give a route middleware a short name for `->middleware('name')` |
| `$middleware->web(append: ...)` / `api(append: ...)` | Add to the web/api **group** |
| `$middleware->group('custom', [...])` | Define a new named group |
| `$middleware->statefulApi()` | Enable Sanctum SPA cookie auth |
| `$middleware->throttleApi()` | Apply rate limiting to the api group |
| `$middleware->trustProxies(at: '*')` | Trust load-balancer proxy headers (Section 21) |

### 7.4 Configuring exceptions **[I/A]**

`->withExceptions()` replaces the old `Handler.php`. Common uses: stop reporting a noisy exception (`dontReport`), customize how an exception renders (`render`), or add context to logs:

```php
->withExceptions(function (Exceptions $exceptions) {
    $exceptions->report(function (\App\Exceptions\OrderFailed $e) {
        // custom logging/notification when this exception occurs
    });
    // Return JSON for API clients automatically:
    $exceptions->shouldRenderJsonWhen(fn ($req, $e) => $req->is('api/*'));
})
```

---

## 8. Eloquent ORM — Models, Migrations, Relationships, Eager Loading, Casts

### 8.1 What Eloquent is **[I]**

**Eloquent is Laravel's own ORM** (Object-Relational Mapper) — the layer that lets you work with database rows as PHP objects. Each database table gets a **model** class (`Post` ↔ `posts`); each row is an instance; columns are properties; and queries are expressed as fluent method chains that read like English. It implements the **Active Record** pattern: a model object both *represents* a row and *knows how to save itself*.

> **Note on the rest of the library:** the Node/Go guides use **Prisma**/**ent** as their ORMs — Eloquent is Laravel's native equivalent and is unrelated to those. For schema design, indexing, and SQL depth, lean on the [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md), [PostgreSQL](POSTGRESQL_GUIDE.md), and [SQLite3](SQLITE3_GUIDE.md) guides — Eloquent generates the SQL, but the database principles are the same.

### 8.2 Migrations — version control for your schema **[I]**

A **migration** is a PHP class describing a schema change (create a table, add a column). Migrations are the *source of truth* for your database structure, checked into Git so every developer and server builds the same schema. You never run `CREATE TABLE` by hand.

```bash
php artisan make:migration create_posts_table
```

```php
// database/migrations/2026_06_25_000000_create_posts_table.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('posts', function (Blueprint $table) {
            $table->id();                                    // BIGINT auto-increment PK
            $table->foreignId('user_id')                     // FK column
                  ->constrained()                            // references users.id
                  ->cascadeOnDelete();                       // delete posts when user is deleted
            $table->string('title');
            $table->string('slug')->unique();
            $table->text('body');
            $table->boolean('published')->default(false);
            $table->timestamp('published_at')->nullable();
            $table->timestamps();                            // created_at + updated_at
            $table->softDeletes();                           // deleted_at (Section 8.10)

            $table->index(['published', 'published_at']);    // composite index for queries
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('posts');                       // how to roll back
    }
};
```

```bash
php artisan migrate            # apply pending migrations
php artisan migrate:rollback   # undo the last batch (runs down())
php artisan migrate:fresh --seed   # drop everything, re-run all, seed (DEV ONLY)
```

**Common column types:** `id`, `foreignId`, `string`, `text`, `longText`, `integer`, `bigInteger`, `boolean`, `decimal('price', 8, 2)`, `json`, `uuid`, `enum('status', ['a','b'])`, `timestamp`, `date`. **Modifiers:** `->nullable()`, `->default(x)`, `->unique()`, `->index()`, `->after('col')`.

### 8.3 The model **[I]**

```bash
php artisan make:model Post
```

```php
// app/Models/Post.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\SoftDeletes;

class Post extends Model
{
    use HasFactory, SoftDeletes;

    // MASS-ASSIGNMENT protection (Section 8.4) — declare what's fillable:
    protected $fillable = ['title', 'slug', 'body', 'published', 'published_at'];

    // Attribute casting (Section 8.8):
    protected function casts(): array
    {
        return [
            'published'    => 'boolean',
            'published_at' => 'datetime',
        ];
    }
}
```

Conventions Eloquent assumes (all overridable): table = plural snake_case of the model (`Post`→`posts`); primary key = `id`; timestamp columns = `created_at`/`updated_at`.

### 8.4 Mass assignment & `$fillable` — a security feature **[I]**

**Mass assignment** is creating/updating a model from an array of input: `Post::create($request->all())`. Without protection, a malicious user could POST an extra field like `is_admin=1` or `user_id=999` and have it silently saved. Eloquent **blocks this by default**: only attributes listed in **`$fillable`** can be mass-assigned. (The inverse, `$guarded`, lists what's *not* fillable — `$fillable` is the safer, explicit choice.)

```php
// Only $fillable columns are accepted; extras are silently ignored:
$post = Post::create($request->only(['title', 'body']));   // best: also whitelist at the request

// CRUD basics:
$post = Post::find(1);                  // by PK (null if missing)
$post = Post::findOrFail(1);            // 404 if missing
$post = Post::where('slug', $slug)->first();
$post->update(['title' => 'New']);      // mass-assignment-protected update
$post->title = 'New'; $post->save();    // direct property set is NOT mass assignment (always allowed)
$post->delete();
Post::destroy([1, 2, 3]);
```

⚠️ **Security:** never `$model->forceFill($request->all())` or `Model::unguard()` with raw request data. And even with `$fillable`, validate input (Section 10) — `$fillable` controls *which* columns, not whether the *values* are valid.

### 8.5 Relationships — the heart of Eloquent **[I/A]**

Relationships let you traverse foreign keys as object properties. You declare them as methods on the model; accessing them as a **property** runs the query and caches the result.

```php
// app/Models/User.php
class User extends Model
{
    // One user HAS MANY posts:
    public function posts() { return $this->hasMany(Post::class); }

    // One user HAS ONE profile:
    public function profile() { return $this->hasOne(Profile::class); }

    // MANY-TO-MANY through a pivot table (role_user):
    public function roles() { return $this->belongsToMany(Role::class); }
}

// app/Models/Post.php
class Post extends Model
{
    // The INVERSE of hasMany — a post BELONGS TO a user:
    public function author() { return $this->belongsTo(User::class, 'user_id'); }

    // A post HAS MANY comments:
    public function comments() { return $this->hasMany(Comment::class); }

    // HAS-MANY-THROUGH: a country's posts via its users (Country→User→Post):
    // (declared on Country) public function posts(){ return $this->hasManyThrough(Post::class, User::class); }

    // POLYMORPHIC: comments/tags that can attach to many model types:
    public function tags() { return $this->morphToMany(Tag::class, 'taggable'); }
}
```

```php
// Using them (note: METHOD () returns a query; PROPERTY runs & caches it):
$user->posts;                       // Collection of Post (cached after first access)
$user->posts()->where('published', true)->count();  // refine the relationship query
$post->author->name;                // traverse the inverse
$user->roles()->attach($roleId);    // many-to-many: add a pivot row
$user->roles()->sync([1, 2, 3]);    // set the exact set of related ids
$post->comments()->create([...]);   // create a related row with the FK pre-filled
```

| Type | Method | Example |
|---|---|---|
| One-to-many | `hasMany` | User → Posts |
| Inverse | `belongsTo` | Post → User |
| One-to-one | `hasOne` / `belongsTo` | User → Profile |
| Many-to-many | `belongsToMany` | User ↔ Roles (pivot) |
| Has-many-through | `hasManyThrough` | Country → Posts (via Users) |
| Polymorphic 1-many | `morphMany`/`morphTo` | Comments on Posts *and* Videos |
| Polymorphic m-many | `morphToMany` | Tags on many models |

### 8.6 The N+1 problem & eager loading — the performance bug you WILL hit **[I/A]**

This is the most common Eloquent performance trap, and it's worth understanding precisely. Consider listing 50 posts with their authors:

```php
$posts = Post::all();                 // 1 query
foreach ($posts as $post) {
    echo $post->author->name;         // 1 query EACH → 50 more queries!
}
// Total: 1 + 50 = 51 queries. This is "N+1".
```

The fix is **eager loading** with `with()`: load all the related rows up front in a second query (using a `WHERE IN`):

```php
$posts = Post::with('author')->get();   // 2 queries total, regardless of count
foreach ($posts as $post) {
    echo $post->author->name;            // no extra queries — already loaded
}

// Eager-load nested & multiple relations:
$posts = Post::with(['author', 'comments.user'])->get();
// Constrain an eager load:
$posts = Post::with(['comments' => fn ($q) => $q->latest()->limit(5)])->get();
// Count related without loading them:
$posts = Post::withCount('comments')->get();   // each post gets ->comments_count
```

⚡ **Best practice / dev aid:** call `Model::preventLazyLoading()` (inside `AppServiceProvider::boot()`, guarded by `!app()->isProduction()`) so that any accidental lazy load **throws in development** — turning silent N+1 bugs into loud errors. This is one of the highest-value lines you can add to a project.

### 8.7 Query scopes — reusable query fragments **[I]**

A **scope** packages a common query constraint into a readable method. **Local scopes** are methods prefixed `scope` you call on the model; **global scopes** apply automatically to *every* query for a model (e.g. soft deletes is a global scope).

```php
class Post extends Model
{
    // Local scope — call as ->published()
    public function scopePublished($query)
    {
        return $query->where('published', true)->whereNotNull('published_at');
    }
    // Parameterized local scope:
    public function scopeOfAuthor($query, User $user)
    {
        return $query->where('user_id', $user->id);
    }
}

// Reads beautifully:
$posts = Post::published()->ofAuthor($user)->latest()->paginate(10);
```

### 8.8 Casts — typing your columns (enum, JSON, encrypted, dates) **[I/A]**

Databases store strings and numbers; **casts** turn them into proper PHP types when you read them and back when you write. Declare them in the `casts()` method:

```php
protected function casts(): array
{
    return [
        'published'    => 'boolean',
        'published_at' => 'datetime',          // → a Carbon date object
        'meta'         => 'array',             // JSON column ↔ PHP array
        'options'      => AsCollection::class,  // JSON ↔ a Collection
        'status'       => PostStatus::class,    // a PHP enum! (Section 8.8.1)
        'secret'       => 'encrypted',          // transparently encrypted at rest
    ];
}
```

#### 8.8.1 Enum casts **[I/A]**

PHP 8.1+ enums + Eloquent casts give you type-safe status columns. Define a backed enum and cast to it:

```php
// app/Enums/PostStatus.php
namespace App\Enums;
enum PostStatus: string {
    case Draft = 'draft';
    case Published = 'published';
    case Archived = 'archived';
}
```

```php
// In the model casts(): 'status' => PostStatus::class
$post->status = PostStatus::Published;     // assign the enum, stored as 'published'
if ($post->status === PostStatus::Draft) { /* ... */ }   // type-safe comparison
```

### 8.9 Accessors & mutators — the modern `Attribute` syntax **[I]**

**Accessors** transform a value when you read it; **mutators** transform it when you set it. ⚡ Since Laravel 9 the modern way is a single method returning an `Attribute` object (the old `getXAttribute`/`setXAttribute` pair still works but is legacy):

```php
use Illuminate\Database\Eloquent\Casts\Attribute;

class User extends Model
{
    // A computed, read-only attribute: $user->full_name
    protected function fullName(): Attribute
    {
        return Attribute::make(
            get: fn ($value, array $attributes) =>
                "{$attributes['first_name']} {$attributes['last_name']}",
        );
    }

    // Both get and set: normalize email on write, lowercase on read
    protected function email(): Attribute
    {
        return Attribute::make(
            get: fn ($value) => strtolower($value),
            set: fn ($value) => trim(strtolower($value)),
        );
    }
}
```

### 8.10 Soft deletes **[I]**

A **soft delete** marks a row deleted (sets `deleted_at`) instead of removing it — so you can restore it and keep referential history. Add the `SoftDeletes` trait + a `softDeletes()` column. Eloquent then auto-excludes soft-deleted rows from queries (a global scope):

```php
$post->delete();             // sets deleted_at; row stays in the DB
Post::withTrashed()->get();  // include soft-deleted
Post::onlyTrashed()->get();  // only soft-deleted
$post->restore();            // un-delete
$post->forceDelete();        // permanently remove
```

### 8.11 Factories & seeders — fake data for dev & tests **[I/A]**

**Factories** define how to build a fake model instance (using Faker). **Seeders** populate the database. Together they give you realistic dev data and are essential for testing (Section 20).

```php
// database/factories/PostFactory.php
namespace Database\Factories;
use Illuminate\Database\Eloquent\Factories\Factory;

class PostFactory extends Factory
{
    public function definition(): array
    {
        return [
            'title'     => fake()->sentence(),
            'slug'      => fake()->unique()->slug(),
            'body'      => fake()->paragraphs(3, true),
            'published' => fake()->boolean(80),     // 80% published
            'user_id'   => \App\Models\User::factory(),  // auto-creates a related user
        ];
    }
    // A "state" — a named variation:
    public function draft(): static
    {
        return $this->state(fn () => ['published' => false]);
    }
}
```

```php
// Using factories:
Post::factory()->count(50)->create();            // 50 posts in the DB
Post::factory()->draft()->create();              // one draft
User::factory()->has(Post::factory()->count(3))->create();  // a user with 3 posts
$post = Post::factory()->make();                 // in-memory, NOT saved

// database/seeders/DatabaseSeeder.php
public function run(): void
{
    User::factory(10)->create();
    Post::factory(50)->create();
}
// run with: php artisan db:seed   (or migrate:fresh --seed)
```

### 8.12 Events & observers **[I/A]**

Models fire **events** at lifecycle points (`creating`, `created`, `updating`, `saved`, `deleting`, `deleted`, …). An **observer** is a class that listens to them — ideal for cross-cutting concerns like auto-generating a slug, clearing a cache, or logging:

```bash
php artisan make:observer PostObserver --model=Post
```

```php
// app/Observers/PostObserver.php
class PostObserver
{
    public function creating(Post $post): void
    {
        $post->slug ??= \Illuminate\Support\Str::slug($post->title);  // auto slug
    }
    public function deleted(Post $post): void
    {
        \Illuminate\Support\Facades\Cache::forget("post.{$post->id}");
    }
}
```

⚡ **Version note.** In Laravel 11/12 you can register an observer with a PHP **attribute** on the model — no manual wiring:

```php
use Illuminate\Database\Eloquent\Attributes\ObservedBy;

#[ObservedBy(PostObserver::class)]
class Post extends Model { /* ... */ }
```

---

## 9. The Query Builder & Raw DB Access

### 9.1 When to use the Query Builder instead of Eloquent **[I]**

Eloquent is great for working with model objects, but sometimes you want a **fluent, lower-level query** without the overhead of hydrating models — reports, aggregates, bulk operations, complex joins, or tables that don't have a model. That's the **Query Builder**, accessed via the `DB` facade. It produces parameterized SQL (so it's safe against injection) and returns plain `stdClass`/array results.

```php
use Illuminate\Support\Facades\DB;

$users = DB::table('users')
    ->where('active', true)
    ->where('age', '>=', 18)
    ->orderBy('name')
    ->select('id', 'name', 'email')
    ->get();                                  // Collection of stdClass

$count = DB::table('orders')->where('paid', true)->count();
$total = DB::table('orders')->sum('amount');
```

### 9.2 Joins, grouping, aggregates **[I]**

```php
$report = DB::table('orders')
    ->join('users', 'users.id', '=', 'orders.user_id')   // INNER JOIN
    ->leftJoin('coupons', 'coupons.id', '=', 'orders.coupon_id')
    ->where('orders.created_at', '>=', now()->subMonth())
    ->groupBy('users.id', 'users.name')
    ->select('users.name', DB::raw('SUM(orders.amount) as revenue'))
    ->havingRaw('SUM(orders.amount) > ?', [1000])
    ->orderByDesc('revenue')
    ->get();
```

### 9.3 Raw expressions — safely **[I]**

Sometimes you need raw SQL fragments. The rule: **never concatenate user input into a raw string** — pass it as **bindings** (`?` placeholders), which the driver escapes. `DB::raw()` is for *trusted* SQL (column expressions), not for values.

```php
// ✅ Safe — value comes through a binding, not string interpolation:
$rows = DB::select('SELECT * FROM posts WHERE views > ? AND status = ?', [100, 'published']);

// ✅ whereRaw with bindings:
DB::table('posts')->whereRaw('LOWER(title) LIKE ?', ['%'.strtolower($q).'%'])->get();

// ❌ NEVER do this — SQL injection:
// DB::select("SELECT * FROM posts WHERE id = $userInput");
```

### 9.4 Transactions — all-or-nothing **[I]**

A **transaction** groups multiple writes so they either all succeed or all roll back — essential whenever one logical operation touches several tables (e.g. transfer money, place an order with line items). The closure form auto-commits on success and auto-rolls-back on any exception:

```php
use Illuminate\Support\Facades\DB;

DB::transaction(function () use ($from, $to, $amount) {
    $from->decrement('balance', $amount);    // both succeed…
    $to->increment('balance', $amount);      // …or both are rolled back
    Transfer::create([...]);
});

// Manual control when you need it:
DB::beginTransaction();
try {
    // ... writes ...
    DB::commit();
} catch (\Throwable $e) {
    DB::rollBack();
    throw $e;
}
```

⚠️ **Gotcha:** dispatching a queued job (Section 14) or sending mail *inside* a transaction can fire **before** the data is committed (the worker reads a row that isn't there yet). Use `Bus::dispatch()->afterCommit()` or set `after_commit` on the queue connection. Eloquent also supports `DB::afterCommit(fn () => ...)`.

---

## 10. Validation — validate(), Form Requests, Custom Rules

### 10.1 Why validation is its own layer **[I]**

Never trust input. **Validation** checks incoming data against rules *before* it touches your models or database, and — for web requests — automatically redirects back with errors and the old input on failure. Doing this in a consistent, declarative way (instead of hand-rolled `if` checks) is one of Laravel's biggest quality-of-life wins, and it's a **security** layer (it's your first defense against malformed/malicious input).

### 10.2 The quick path — `$request->validate()` **[I]**

For simple cases, validate right in the controller. On failure it **throws**, Laravel catches it, and (for web) redirects back with errors in the session; (for JSON/API) returns a `422` with a JSON error body — automatically.

```php
public function store(Request $request)
{
    // Returns ONLY the validated data (a clean array) — use this for create():
    $validated = $request->validate([
        'title'        => ['required', 'string', 'max:255'],
        'slug'         => ['required', 'alpha_dash', 'unique:posts,slug'],
        'body'         => ['required', 'string'],
        'published_at' => ['nullable', 'date', 'after:now'],
        'tags'         => ['array'],                 // tags must be an array
        'tags.*'       => ['integer', 'exists:tags,id'],  // each tag id must exist
        'email'        => ['required', 'email:rfc,dns'],
    ]);

    $post = Post::create($validated);   // only validated keys — safe
    return redirect()->route('posts.show', $post)->with('status', 'Created!');
}
```

Displaying errors in Blade was shown in Section 5.4 (`@error`, `old()`). The `$errors` variable is **always** available in every view.

### 10.3 Form Request classes — the production pattern **[I]**

When rules grow, move them into a **Form Request**: a dedicated class encapsulating both **authorization** (can this user even do this?) and **validation**. Type-hint it in the controller and Laravel runs it automatically *before* your method body — if it fails, your method never executes. This keeps controllers thin.

```bash
php artisan make:request StorePostRequest
```

```php
// app/Http/Requests/StorePostRequest.php
namespace App\Http\Requests;
use Illuminate\Foundation\Http\FormRequest;

class StorePostRequest extends FormRequest
{
    // Authorization gate for this action (return false → 403):
    public function authorize(): bool
    {
        return $this->user()->can('create', \App\Models\Post::class);
    }

    // The rules:
    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'body'  => ['required', 'string'],
        ];
    }

    // Customize messages / attribute names (optional):
    public function messages(): array
    {
        return ['title.required' => 'Every post needs a title.'];
    }

    // Mutate input before validation (e.g. auto-slug):
    protected function prepareForValidation(): void
    {
        $this->merge(['slug' => \Illuminate\Support\Str::slug($this->title)]);
    }
}
```

```php
// The controller stays tiny — validation & authz already happened:
public function store(StorePostRequest $request)
{
    $post = Post::create($request->validated());
    return redirect()->route('posts.show', $post);
}
```

### 10.4 The rule cheat sheet **[I]**

| Rule | Meaning |
|---|---|
| `required` / `nullable` | must be present / may be null |
| `string` `integer` `numeric` `boolean` `array` | type |
| `email:rfc,dns` | valid email (optionally DNS-checked) |
| `min:n` `max:n` `between:a,b` `size:n` | length/value bounds |
| `unique:table,column` | not already in the table |
| `exists:table,column` | must already exist |
| `confirmed` | needs a matching `field_confirmation` |
| `in:a,b,c` / `enum(StatusEnum::class)` | allowed set |
| `date` `after:now` `before:tomorrow` | date constraints |
| `regex:/.../ ` | pattern |
| `image` `mimes:jpg,png` `max:2048` | file uploads (KB) |
| `password` (the `Password` rule object) | strong-password policy |

### 10.5 Custom rules **[I]**

When built-ins don't fit, write a rule object (`php artisan make:rule Uppercase`) or use an inline closure:

```php
// Inline closure rule:
$request->validate([
    'title' => [function ($attribute, $value, $fail) {
        if (str_contains($value, 'spam')) {
            $fail("The {$attribute} contains a banned word.");
        }
    }],
]);
```

---

## 11. Middleware in Depth

### 11.1 What middleware is and why **[I]**

**Middleware is a layer of code that wraps your route handlers** — it inspects or transforms every request on the way *in* and the response on the way *out*. It's how cross-cutting concerns are applied uniformly: authentication, CSRF protection, session loading, rate limiting, CORS, logging, forcing HTTPS, locale detection. Conceptually it's a series of nested functions (the **pipeline**): each middleware can pass the request to the next (`$next($request)`) or short-circuit (redirect, throw, deny).

### 11.2 Before vs after middleware **[I]**

```bash
php artisan make:middleware EnsureUserIsAdmin
```

```php
// app/Http/Middleware/EnsureUserIsAdmin.php
namespace App\Http\Middleware;
use Closure;
use Illuminate\Http\Request;

class EnsureUserIsAdmin
{
    public function handle(Request $request, Closure $next)
    {
        // --- BEFORE logic: runs before the controller ---
        if (! $request->user()?->is_admin) {
            abort(403, 'Admins only.');     // short-circuit: controller never runs
        }

        $response = $next($request);        // hand off to the next layer / controller

        // --- AFTER logic: runs after the controller, can modify the response ---
        $response->headers->set('X-Admin-Area', 'true');

        return $response;
    }
}
```

### 11.3 Registering & applying middleware **[I]**

⚡ As covered in Section 7.3, you register middleware in **`bootstrap/app.php`**, not a Kernel file. Give it an alias, then apply it to routes:

```php
// bootstrap/app.php → withMiddleware:
$middleware->alias(['admin' => \App\Http\Middleware\EnsureUserIsAdmin::class]);
```

```php
// Apply by alias on a route or group:
Route::get('/admin', DashboardController::class)->middleware('admin');
Route::middleware(['auth', 'admin'])->group(fn () => /* ... */);
```

### 11.4 Middleware parameters **[I]**

Middleware can take arguments after the request/closure — passed in the route string after a `:`:

```php
public function handle(Request $request, Closure $next, string $role)
{
    if (! $request->user()->hasRole($role)) abort(403);
    return $next($request);
}
```

```php
Route::put('/post/{post}', ...)->middleware('role:editor');  // $role = 'editor'
```

### 11.5 Throttling / rate limiting **[I]**

The built-in `throttle` middleware caps requests per window — essential for login endpoints and APIs (DoS / brute-force defense). ⚡ Laravel 11/12 supports **per-second** limits and named rate limiters defined in a provider:

```php
Route::middleware('throttle:60,1')->group(...);   // 60 requests / 1 minute

// Named, dynamic limiter (define in AppServiceProvider::boot()):
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Cache\RateLimiting\Limit;

RateLimiter::for('api', function ($request) {
    return $request->user()
        ? Limit::perMinute(120)->by($request->user()->id)   // per-user
        : Limit::perMinute(30)->by($request->ip());          // per-IP for guests
});
// then: ->middleware('throttle:api')
```

---

## 12. Authentication & Authorization — Guards, Gates, Policies, Sanctum

### 12.1 Authentication vs authorization **[I]**

Two distinct ideas you must keep straight:

- **Authentication** = *who are you?* (login, sessions, tokens). 
- **Authorization** = *are you allowed to do this?* (gates, policies, roles).

Laravel has first-class support for both. For full apps you almost never hand-roll auth — you use a **starter kit** (Section 1.5) or **Fortify** for the backend logic. But understanding the pieces underneath matters.

### 12.2 Guards & providers — the mental model **[I]**

- A **guard** defines *how users are authenticated for each request*. The `web` guard uses **sessions + cookies** (stateful); the `sanctum`/`token` guard uses **API tokens** (stateless). Config lives in `config/auth.php`.
- A **provider** defines *how users are retrieved from storage* (the default `eloquent` provider uses the `User` model). 

You rarely edit these, but knowing `auth:web` vs `auth:sanctum` exist explains why API routes use a different middleware than web routes.

### 12.3 Using the starter kits **[I]**

The fastest correct path: scaffold auth with a starter kit at project creation (Section 2.3). You get registration, login, password reset, email verification, profile management, and (for the React/Vue kits) an Inertia frontend — production-quality, customizable. ⚡ The React/Vue kits can use **WorkOS AuthKit** for hosted auth (SSO, social login) if you choose it.

### 12.4 Manual authentication **[I]**

When you need control, the `Auth` facade gives you the primitives:

```php
use Illuminate\Support\Facades\Auth;

// Attempt login with credentials (hashes & compares password automatically):
if (Auth::attempt(['email' => $email, 'password' => $password], $remember)) {
    $request->session()->regenerate();          // prevent session fixation (security!)
    return redirect()->intended('/dashboard');  // back to where they were headed
}
// failed:
return back()->withErrors(['email' => 'Invalid credentials.']);

Auth::user();        // the current user (or null)
Auth::id();          // their id
Auth::check();       // bool: logged in?
Auth::logout();      // log out
```

### 12.5 Authorization: Gates & Policies **[I/A]**

**Gates** are simple closures for one-off permission checks; **Policies** are classes that group all the authorization rules for a model (the preferred, organized approach).

```php
// A Gate (define in AppServiceProvider::boot()):
use Illuminate\Support\Facades\Gate;
Gate::define('view-admin', fn ($user) => $user->is_admin);
// Use it:
if (Gate::allows('view-admin')) { /* ... */ }
```

```bash
php artisan make:policy PostPolicy --model=Post
```

```php
// app/Policies/PostPolicy.php — one method per action:
class PostPolicy
{
    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;     // only the author can edit
    }
    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->user_id || $user->is_admin;
    }
}
```

⚡ In Laravel 11/12 policies are **auto-discovered** by naming convention (`Post` → `PostPolicy`) — no manual registration. Use them everywhere:

```php
// In a controller — throws 403 if denied:
$this->authorize('update', $post);
// On the user:
if ($request->user()->can('update', $post)) { /* ... */ }
// In Blade:
@can('update', $post) <a href="...">Edit</a> @endcan
// In a Form Request authorize() (Section 10.3):
return $this->user()->can('update', $this->route('post'));
```

### 12.6 Sanctum — the two modes (the part people get wrong) **[I/A]**

**Sanctum** is Laravel's lightweight auth package for APIs and SPAs. The crucial thing to understand is that it does **two different jobs**, and you pick one:

**Mode 1 — API tokens (for mobile apps, third-party API access, server-to-server).** A user generates a token; the client sends it as a `Bearer` header. Tokens can carry **abilities** (scopes).

```php
// Issue a token (e.g. in a login endpoint):
$token = $user->createToken('mobile-app', ['posts:read', 'posts:write'])->plainTextToken;
return ['token' => $token];   // client stores it, sends: Authorization: Bearer <token>

// Protect routes with the sanctum guard:
Route::middleware('auth:sanctum')->get('/user', fn (Request $r) => $r->user());

// Check an ability:
if ($request->user()->tokenCan('posts:write')) { /* ... */ }
```

**Mode 2 — SPA cookie authentication (for a first-party JavaScript SPA on the same top-level domain).** Here there are **no tokens** — Sanctum uses the normal Laravel **session cookie** + CSRF, so it's stateful but feels like an API. The SPA first hits `/sanctum/csrf-cookie`, then logs in via the normal `/login` route; subsequent XHR requests are authenticated by the cookie. You enable it with `$middleware->statefulApi()` in `bootstrap/app.php` (Section 7.3) and set `SANCTUM_STATEFUL_DOMAINS`.

| | Mode 1: API tokens | Mode 2: SPA cookies |
|---|---|---|
| Client | Mobile, CLI, other servers | Your own JS SPA (Inertia/React/Vue) |
| Credential | `Bearer` token in header | Session cookie + CSRF |
| State | Stateless | Stateful (session) |
| Enable | `auth:sanctum` middleware | `statefulApi()` + stateful domains |

**Rule of thumb:** same-domain SPA → cookies (Mode 2, more secure: no token to steal from JS). Mobile/third-party → tokens (Mode 1).

### 12.7 Fortify & Socialite **[I]**

- **Fortify** is the **headless** auth backend (the controllers/logic for login, registration, 2FA, password reset) with **no UI** — the starter kits and Jetstream build on it. Use it directly when you want Laravel's battle-tested auth logic but your own frontend.
- **Socialite** handles **OAuth login** with providers (Google, GitHub, Facebook, …): you redirect the user to the provider, they approve, and you get their profile back to find-or-create a local user.

```php
// routes + controller with Socialite:
return Socialite::driver('github')->redirect();          // send to GitHub
$ghUser = Socialite::driver('github')->user();           // callback: get profile
$user = User::updateOrCreate(
    ['email' => $ghUser->getEmail()],
    ['name' => $ghUser->getName(), 'github_id' => $ghUser->getId()]
);
Auth::login($user);
```

### 12.8 Password hashing **[I]**

Laravel **hashes passwords automatically** (bcrypt by default; argon2id available) via the `hashed` cast on the `User` model's password — never store plaintext. To hash manually use the `Hash` facade; **never** use `md5`/`sha1`:

```php
use Illuminate\Support\Facades\Hash;
$hash = Hash::make($plain);            // bcrypt by default
Hash::check($plain, $user->password);  // verify (constant-time)
```

---

## 13. Building REST APIs — Resources, Pagination, Versioning, Rate Limiting

### 13.1 The API mindset **[I/A]**

An API returns **JSON**, is **stateless**, authenticates with **Sanctum tokens** (or SPA cookies), and should return **consistent shapes** and **proper HTTP status codes**. The single most important tool for shape consistency is **API Resources** — don't just `return $model`, which leaks every column (including ones you renamed or want hidden) and couples your JSON to your DB schema.

### 13.2 Eloquent API Resources — transformers **[I/A]**

An **API Resource** is a class that transforms a model (or collection) into the exact JSON you want to expose. It decouples your public API from your internal schema and lets you include relationships conditionally.

```bash
php artisan make:resource PostResource
```

```php
// app/Http/Resources/PostResource.php
namespace App\Http\Resources;
use Illuminate\Http\Resources\Json\JsonResource;

class PostResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id'        => $this->id,
            'title'     => $this->title,
            'body'      => $this->body,
            'published' => $this->published,
            // Only include the author if it was eager-loaded (no extra query / N+1):
            'author'    => UserResource::make($this->whenLoaded('author')),
            'comments_count' => $this->whenCounted('comments'),
            'created_at' => $this->created_at->toIso8601String(),
            // Never leak internal columns like user_id or soft-delete timestamps unless intended
        ];
    }
}
```

```php
// In the controller:
public function show(Post $post)
{
    return new PostResource($post->load('author'));
}
public function index()
{
    // collection() keeps pagination metadata intact:
    return PostResource::collection(Post::with('author')->paginate(15));
}
```

### 13.3 Pagination **[I/A]**

Never return unbounded lists. `paginate()` returns a paginator that, when JSON-serialized (or via a Resource collection), includes `data`, `links`, and `meta` (current page, total, per-page):

```php
$posts = Post::latest()->paginate(20);          // ?page=2 handled automatically
$posts = Post::latest()->cursorPaginate(20);    // cursor pagination — faster for huge tables
$posts = Post::latest()->simplePaginate(20);    // just next/prev (no total count query)
```

### 13.4 Versioning **[I/A]**

APIs evolve; version them so you don't break existing clients. The common approach is a URL prefix via route groups:

```php
// routes/api.php
Route::prefix('v1')->group(function () {
    Route::apiResource('posts', \App\Http\Controllers\Api\V1\PostController::class);
});
Route::prefix('v2')->group(function () { /* ... */ });
```

### 13.5 Consistent errors & status codes **[I/A]**

Return the right status code and a predictable error body. Laravel does a lot automatically: validation failures → `422` with `{message, errors}`; `findOrFail`/model binding misses → `404`; failed authorization → `403`; missing auth → `401`. Customize global JSON error rendering in `bootstrap/app.php` (Section 7.4).

```php
// Throwing the right thing:
abort(404, 'Post not found');
abort_if($post->locked, 423, 'Post is locked');
throw new \Illuminate\Auth\Access\AuthorizationException();  // → 403
return response()->json(['message' => 'Created', 'data' => new PostResource($post)], 201);
```

| Situation | Status |
|---|---|
| Success (GET/PUT/PATCH) | 200 |
| Created | 201 |
| No content (DELETE) | 204 |
| Validation failed | 422 |
| Unauthenticated | 401 |
| Forbidden | 403 |
| Not found | 404 |
| Rate limited | 429 |

### 13.6 Sanctum-protected, rate-limited API **[I/A]**

Pulling it together — a protected, versioned, throttled API resource:

```php
// routes/api.php
Route::middleware(['auth:sanctum', 'throttle:api'])
    ->prefix('v1')
    ->group(function () {
        Route::apiResource('posts', PostController::class);
    });
```

---

## 14. Queues & Jobs — Workers, Drivers, Batching, Horizon

### 14.1 Why async — the problem queues solve **[A]**

Some work is slow: sending email, processing an image, calling a third-party API, generating a PDF. If you do it *during* the web request, the user waits. **Queues** let you push that work onto a background queue and respond to the user immediately; a separate **worker** process picks the job up and runs it. This keeps requests fast and makes slow/flaky work **retryable**.

### 14.2 Jobs **[A]**

A **job** is a class encapsulating a unit of background work. Generate one, put the work in `handle()`, and **dispatch** it.

```bash
php artisan make:job SendWelcomeEmail
```

```php
// app/Jobs/SendWelcomeEmail.php
namespace App\Jobs;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use App\Models\User;

class SendWelcomeEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;          // retry up to 3 times on failure
    public int $backoff = 30;       // wait 30s between retries
    public int $timeout = 120;      // kill the job if it runs > 120s

    // SerializesModels stores only the model's ID and re-fetches it in the worker:
    public function __construct(public User $user) {}

    public function handle(): void
    {
        \Illuminate\Support\Facades\Mail::to($this->user)
            ->send(new \App\Mail\WelcomeMail($this->user));
    }

    // Called when the job ultimately fails (after all retries):
    public function failed(\Throwable $e): void { /* notify, log */ }
}
```

```php
// Dispatch it (from a controller, anywhere):
SendWelcomeEmail::dispatch($user);                       // queued
SendWelcomeEmail::dispatch($user)->onQueue('emails');    // a specific queue
SendWelcomeEmail::dispatch($user)->delay(now()->addMinutes(10));  // run later
SendWelcomeEmail::dispatchSync($user);                   // run NOW, no queue (for testing)
SendWelcomeEmail::dispatch($user)->afterCommit();        // wait for the DB transaction (Section 9.4!)
```

### 14.3 Drivers & running workers **[A]**

The **queue driver** (set by `QUEUE_CONNECTION`) decides where jobs are stored. Laravel 11/12 defaults to **`database`** (zero setup); **`redis`** is the production choice (faster, supports Horizon — see [Redis](REDIS_GUIDE.md)). `sync` runs jobs immediately (no real queue — for local dev/tests).

```bash
# Create the jobs/failed_jobs tables (database driver):
php artisan make:queue-table && php artisan migrate

# Run a worker (this process must stay running in production — Section 21):
php artisan queue:work redis --queue=high,default --tries=3 --max-time=3600

# After deploying new code, tell long-running workers to restart:
php artisan queue:restart

# Inspect & retry failures:
php artisan queue:failed
php artisan queue:retry all
```

⚠️ **Gotcha:** `queue:work` loads your code once and keeps running — it **won't see code changes** until restarted. In dev use `queue:listen` (slower but reloads each job) or remember to restart.

### 14.4 Batching **[A]**

**Job batches** run a group of jobs and fire callbacks when they all finish (or any fail) — useful for "process these 10,000 records, then notify me." Requires a `job_batches` table.

```php
use Illuminate\Support\Facades\Bus;

$batch = Bus::batch([
    new ImportCsvChunk($chunk1),
    new ImportCsvChunk($chunk2),
])->then(fn ($batch) => /* all done */)
  ->catch(fn ($batch, $e) => /* a job failed */)
  ->finally(fn ($batch) => /* always */)
  ->name('CSV import')
  ->dispatch();
```

### 14.5 Horizon **[A]**

**Horizon** is a first-party dashboard + supervisor for **Redis** queues. It gives you a real-time UI (throughput, runtimes, failures), tag-based monitoring, and config-as-code for worker processes/auto-scaling. In production with Redis, Horizon replaces raw `queue:work`:

```bash
composer require laravel/horizon
php artisan horizon:install
php artisan horizon            # runs the supervised workers (keep alive via systemd/supervisor)
```

---

## 15. Events, Listeners & Broadcasting with Reverb

### 15.1 The event system — decoupling **[A]**

**Events** let you decouple "something happened" from "the things that should react to it." A controller dispatches `OrderShipped`; multiple **listeners** (send email, update analytics, notify a webhook) react — without the controller knowing about any of them. This keeps code modular: to add a reaction, add a listener; you never touch the dispatch site.

```bash
php artisan make:event OrderShipped
php artisan make:listener SendShipmentNotification --event=OrderShipped
```

```php
// app/Events/OrderShipped.php
class OrderShipped
{
    use Dispatchable, SerializesModels;
    public function __construct(public Order $order) {}
}

// app/Listeners/SendShipmentNotification.php  (implements ShouldQueue to run async)
class SendShipmentNotification implements ShouldQueue
{
    public function handle(OrderShipped $event): void
    {
        // react to the event…
    }
}
```

```php
// Dispatch it:
OrderShipped::dispatch($order);
event(new OrderShipped($order));   // equivalent
```

⚡ In Laravel 11/12, listeners are **auto-discovered** by their type-hinted event parameter — no manual `EventServiceProvider` mapping needed (though you can register explicitly if you prefer).

### 15.2 Broadcasting — events to the browser in real time **[A]**

**Broadcasting** pushes server-side events to connected clients over **WebSockets**, so the browser updates live (new chat message, notification, live dashboard) without polling. An event implementing `ShouldBroadcast` is sent to a channel; the browser, subscribed via **Echo**, receives it.

```bash
php artisan install:broadcasting    # sets up channels, config, and (offers) Reverb
```

```php
// A broadcastable event:
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Broadcasting\PrivateChannel;

class MessageSent implements ShouldBroadcast
{
    use Dispatchable, SerializesModels;
    public function __construct(public Message $message) {}

    public function broadcastOn(): array
    {
        // a PRIVATE channel — only authorized users can listen (auth in routes/channels.php):
        return [new PrivateChannel("chat.{$this->message->room_id}")];
    }
    public function broadcastAs(): string { return 'message.sent'; }
}
```

### 15.3 Reverb — the first-party WebSocket server **[A]**

⚡ **Version note.** **Reverb** is Laravel's own, high-performance WebSocket server (introduced 2024), now the recommended broadcasting backend — it replaces having to use the paid Pusher service or third-party `laravel-websockets`. It speaks the Pusher protocol, so **Echo** works against it unchanged.

```bash
composer require laravel/reverb
php artisan reverb:install
php artisan reverb:start        # runs the WS server (keep alive in production)
```

```env
BROADCAST_CONNECTION=reverb
REVERB_APP_ID=...
REVERB_APP_KEY=...
REVERB_APP_SECRET=...
REVERB_HOST=localhost
REVERB_PORT=8080
```

On the client, **Laravel Echo** subscribes and listens:

```js
// resources/js/echo.js — configured for Reverb
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';
window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'reverb',
    key: import.meta.env.VITE_REVERB_APP_KEY,
    wsHost: import.meta.env.VITE_REVERB_HOST,
    wsPort: import.meta.env.VITE_REVERB_PORT ?? 8080,
    forceTLS: false,
    enabledTransports: ['ws', 'wss'],
});

// Listen on a private channel (Echo handles the auth handshake):
window.Echo.private(`chat.${roomId}`)
    .listen('.message.sent', (e) => console.log('New message:', e.message));
```

For the JS side broadly, see the [React 19](REACT_19_GUIDE.md) guide; for the protocol underneath, the [Networking](NETWORKING_GUIDE.md) guide's WebSockets section.

---

## 16. Artisan & Task Scheduling

### 16.1 Writing console commands **[I/A]**

Beyond the built-in commands, you write your own **Artisan commands** for maintenance tasks, imports, report generation, or anything you want to run from the CLI or on a schedule. ⚡ In Laravel 11/12, commands in `app/Console/Commands` are **auto-registered** — no Kernel to edit.

```bash
php artisan make:command SendDailyReport
```

```php
// app/Console/Commands/SendDailyReport.php
namespace App\Console\Commands;
use Illuminate\Console\Command;

class SendDailyReport extends Command
{
    // name + arguments/options, then a description:
    protected $signature = 'report:daily {--email= : Override recipient}';
    protected $description = 'Email the daily sales report';

    public function handle(): int
    {
        $to = $this->option('email') ?? config('mail.reports_to');
        $this->info("Generating report for {$to}…");      // colored CLI output
        // …do the work, with a progress bar if long:
        $this->withProgressBar(Order::all(), fn ($o) => /* ... */);
        $this->newLine();
        return Command::SUCCESS;                           // exit code 0
    }
}
```

```bash
php artisan report:daily --email=boss@example.com
```

Helpful I/O methods: `$this->argument()`, `$this->option()`, `$this->ask()`, `$this->confirm()`, `$this->choice()`, `$this->info()/warn()/error()`, `$this->table()`.

### 16.2 The scheduler — cron without crontab sprawl **[I/A]**

⚡ **Version note.** Instead of managing many crontab lines, you define **all** scheduled tasks in code. In Laravel 11/12 the schedule lives in **`routes/console.php`** (it used to live in `app/Console/Kernel.php`). You then add **one** cron entry that runs `schedule:run` every minute, and Laravel figures out what's actually due.

```php
// routes/console.php
use Illuminate\Support\Facades\Schedule;

Schedule::command('report:daily')->dailyAt('07:00');
Schedule::command('telescope:prune')->daily();
Schedule::job(new \App\Jobs\PruneStaleCarts)->hourly();
Schedule::call(fn () => Cache::forget('stats'))->everyFiveMinutes();

// Real-world hardening:
Schedule::command('backup:run')
    ->dailyAt('02:00')
    ->onOneServer()                 // multi-server: run on only ONE node
    ->withoutOverlapping()          // don't start if the last run is still going
    ->runInBackground()
    ->emailOutputOnFailure('ops@example.com');
```

The **single** cron line you add on the server (see [Linux Server Administration]):

```cron
* * * * * cd /path/to/app && php artisan schedule:run >> /dev/null 2>&1
```

Frequency helpers: `everyMinute()`, `everyFiveMinutes()`, `hourly()`, `daily()`, `dailyAt('13:00')`, `weekly()`, `monthly()`, `cron('* * * * *')`, plus constraints `weekdays()`, `between('9:00','17:00')`.

---

## 17. Caching, Sessions & Redis

### 17.1 The cache — why and the API **[I]**

**Caching** stores the result of expensive work (a heavy query, an API call, a rendered fragment) so you don't redo it on every request. Laravel gives one unified API over many backends (the `CACHE_STORE` driver): `array` (per-request, tests), `file`, `database`, and — for production — **`redis`** (see [Redis](REDIS_GUIDE.md)). ⚡ Laravel 11/12 defaults `CACHE_STORE` to `database` for a setup-free start.

```php
use Illuminate\Support\Facades\Cache;

Cache::put('key', $value, now()->addHour());   // store with a TTL
$value = Cache::get('key', 'default');         // read with a fallback
Cache::forget('key');
Cache::has('key');

// The pattern you'll use most — "remember": return cached, or compute, cache, and return:
$stats = Cache::remember('dashboard.stats', 600, function () {
    return ['users' => User::count(), 'revenue' => Order::sum('amount')];  // runs only on a miss
});

// Forever + manual invalidation (e.g. from a model observer, Section 8.12):
Cache::rememberForever("post.{$id}", fn () => Post::with('author')->findOrFail($id));

// Atomic increments, locks (prevent race conditions):
Cache::increment('page.views');
Cache::lock('import')->block(5, function () { /* only one process at a time */ });
```

**Cache tags** (Redis/Memcached only) let you invalidate a *group* of keys at once: `Cache::tags(['posts'])->put(...)` then `Cache::tags(['posts'])->flush()`.

### 17.2 Sessions **[I]**

A **session** stores per-user data across requests (the logged-in user id, flash messages, the CSRF token). The `SESSION_DRIVER` (default `database` in L11/12; `redis` in production) decides where it's stored. You usually interact with it indirectly (auth, `->with()` flash messages), but the API is there:

```php
$request->session()->put('cart_id', $id);
$id = $request->session()->get('cart_id');
session(['theme' => 'dark']);              // helper shorthand
$request->session()->flash('status', 'Saved!');   // available for the NEXT request only
$request->session()->forget('cart_id');
```

### 17.3 Redis integration **[I]**

[Redis](REDIS_GUIDE.md) is the workhorse behind production Laravel: it backs the **cache**, **sessions**, **queues** (with Horizon), and **broadcasting** scaling. Point `CACHE_STORE`, `SESSION_DRIVER`, and `QUEUE_CONNECTION` at `redis`, install the client (`predis/predis` or the faster PhpRedis extension), and you're done. You can also use Redis directly:

```php
use Illuminate\Support\Facades\Redis;
Redis::set('user:1:visits', 0);
Redis::incr('user:1:visits');
Redis::zadd('leaderboard', 100, 'alice');   // sorted set for a leaderboard
```

---

## 18. Mail & Notifications

### 18.1 Mailables **[I]**

A **Mailable** is a class representing one type of email. You build its content (usually a Markdown or Blade view), pass it data, and send it. In local dev set `MAIL_MAILER=log` (writes to the log) or use **Mailpit** (Sail ships it) to view emails in a browser.

```bash
php artisan make:mail OrderShipped --markdown=mail.orders.shipped
```

```php
// app/Mail/OrderShipped.php
class OrderShipped extends \Illuminate\Mail\Mailable
{
    use Queueable, SerializesModels;
    public function __construct(public Order $order) {}

    public function envelope(): \Illuminate\Mail\Mailables\Envelope
    {
        return new \Illuminate\Mail\Mailables\Envelope(subject: 'Your order shipped!');
    }
    public function content(): \Illuminate\Mail\Mailables\Content
    {
        return new \Illuminate\Mail\Mailables\Content(markdown: 'mail.orders.shipped');
    }
    public function attachments(): array
    {
        return [\Illuminate\Mail\Mailables\Attachment::fromStorage("invoices/{$this->order->id}.pdf")];
    }
}
```

```php
use Illuminate\Support\Facades\Mail;
Mail::to($user)->cc($manager)->send(new OrderShipped($order));
Mail::to($user)->queue(new OrderShipped($order));   // send via the queue (don't block the request)
```

**Markdown mail** gives you pre-styled, responsive components:

```blade
{{-- resources/views/mail/orders/shipped.blade.php --}}
<x-mail::message>
# Your order shipped!
Order #{{ $order->id }} is on its way.

<x-mail::button :url="route('orders.show', $order)">View Order</x-mail::button>

Thanks,<br>{{ config('app.name') }}
</x-mail::message>
```

### 18.2 The notification system — many channels, one message **[I]**

**Notifications** are like mailables but channel-agnostic: one notification class can deliver over **mail**, **database** (an in-app notification bell), **broadcast** (real-time), **Slack**, SMS (Vonage), and more — you choose channels per notification. This is the right tool for "tell the user something happened."

```bash
php artisan make:notification InvoicePaid
```

```php
class InvoicePaid extends \Illuminate\Notifications\Notification implements ShouldQueue
{
    use Queueable;
    public function __construct(public Invoice $invoice) {}

    // Which channels to use for a given notifiable:
    public function via(object $notifiable): array
    {
        return ['mail', 'database', 'broadcast'];
    }
    public function toMail($notifiable): \Illuminate\Notifications\Messages\MailMessage
    {
        return (new \Illuminate\Notifications\Messages\MailMessage)
            ->subject('Invoice paid')
            ->line("Invoice #{$this->invoice->id} was paid.")
            ->action('View Invoice', route('invoices.show', $this->invoice));
    }
    // Stored in the notifications table (run `php artisan make:notifications-table`):
    public function toArray($notifiable): array
    {
        return ['invoice_id' => $this->invoice->id, 'amount' => $this->invoice->amount];
    }
}
```

```php
// The User model uses the Notifiable trait, so:
$user->notify(new InvoicePaid($invoice));
// Read in-app notifications:
$user->unreadNotifications;       // collection from the notifications table
$user->unreadNotifications->markAsRead();
```

Because the class implements `ShouldQueue`, delivery is queued automatically — the request doesn't wait on SMTP.

---

## 19. File Storage — the Filesystem Abstraction

### 19.1 The abstraction and why it matters **[I]**

Laravel's **filesystem abstraction** (built on Flysystem) gives you one API to read/write files regardless of where they live — **local disk**, the public folder, **Amazon S3**, or other cloud storage. You configure named **disks** in `config/filesystems.php` and switch from local to S3 in production by changing config, **not code**.

| Disk | Where | Web-accessible? |
|---|---|---|
| `local` | `storage/app/private` | No (private) |
| `public` | `storage/app/public` | Yes, after `storage:link` |
| `s3` | Amazon S3 / compatible | Via signed/public URLs |

### 19.2 Uploads & the Storage facade **[I]**

```php
use Illuminate\Support\Facades\Storage;

public function store(Request $request)
{
    $request->validate(['avatar' => ['required', 'image', 'max:2048']]);  // KB

    // store() generates a unique name; returns the path:
    $path = $request->file('avatar')->store('avatars', 'public');
    // or keep a specific name:
    // $path = $request->file('avatar')->storeAs('avatars', $user->id.'.jpg', 'public');

    $request->user()->update(['avatar_path' => $path]);
    return back();
}
```

```php
// Reading / writing / checking:
Storage::disk('public')->put('notes/hello.txt', 'content');
$contents = Storage::disk('public')->get('notes/hello.txt');
Storage::disk('public')->exists($path);
Storage::disk('public')->delete($path);
$url = Storage::disk('public')->url($path);            // public URL
return Storage::download($path);                        // force a download response
```

For the `public` disk to be reachable over HTTP you must create the symlink **once**:

```bash
php artisan storage:link    # links public/storage → storage/app/public
```

### 19.3 Signed / temporary URLs — sharing private files securely **[I]**

To let a user download a **private** file without making it public, generate a **temporary signed URL** that expires — the signature prevents tampering with the path or expiry:

```php
// Expiring URL to a private S3 object:
$url = Storage::disk('s3')->temporaryUrl($path, now()->addMinutes(15));

// Signed route URLs (e.g. one-click unsubscribe / email-confirm links):
$url = \Illuminate\Support\Facades\URL::temporarySignedRoute(
    'unsubscribe', now()->addDay(), ['user' => $user->id]
);
// protect the route with the 'signed' middleware so a tampered URL → 403
```

---

## 20. Testing — Pest & PHPUnit

### 20.1 Why test, and the two frameworks **[A]**

Tests let you change code confidently — they prove your routes, validation, auth, and business logic still work. Laravel has **first-class testing** with a full app booted, a test database, factories, and HTTP/assertion helpers. ⚡ **Two runners** ship: classic **PHPUnit** and **Pest** (a modern, expressive layer on top of PHPUnit that's now the popular default — the installer offers it). Both run via `php artisan test`.

- **Feature tests** exercise the app through HTTP (a route → controller → DB → response). This is where most value is.
- **Unit tests** test a single class/method in isolation.

### 20.2 A feature test, in both styles **[A]**

**Pest** (concise, function-style):

```php
// tests/Feature/PostTest.php
use App\Models\User;
use function Pest\Laravel\{actingAs, post, get};
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);   // migrate a fresh DB per test, rolled back after

it('lets an authed user create a post', function () {
    $user = User::factory()->create();

    $response = actingAs($user)->post('/posts', [
        'title' => 'Hello',
        'body'  => 'World',
    ]);

    $response->assertRedirect();
    expect(\App\Models\Post::count())->toBe(1);
    $this->assertDatabaseHas('posts', ['title' => 'Hello', 'user_id' => $user->id]);
});

it('rejects an empty title', function () {
    actingAs(User::factory()->create())
        ->post('/posts', ['body' => 'x'])
        ->assertSessionHasErrors('title');
});
```

**PHPUnit** (class-style — same behavior):

```php
class PostTest extends \Tests\TestCase
{
    use \Illuminate\Foundation\Testing\RefreshDatabase;

    public function test_user_can_create_post(): void
    {
        $user = \App\Models\User::factory()->create();
        $this->actingAs($user)
             ->post('/posts', ['title' => 'Hi', 'body' => 'Yo'])
             ->assertRedirect();
        $this->assertDatabaseHas('posts', ['title' => 'Hi']);
    }
}
```

### 20.3 Database testing & the key helpers **[A]**

- **`RefreshDatabase`** wraps each test in a transaction (or migrates a fresh schema) so tests are isolated and the DB is clean every time. Point your test DB at SQLite in-memory for speed (`DB_CONNECTION=sqlite`, `DB_DATABASE=:memory:` in `phpunit.xml`).
- **Factories** create the data each test needs (Section 8.11).

| Assertion | Checks |
|---|---|
| `assertDatabaseHas('table', [...])` | a matching row exists |
| `assertDatabaseMissing(...)` | no matching row |
| `assertDatabaseCount('posts', 3)` | exact row count |
| `assertModelExists($post)` / `assertModelMissing($post)` | model in/not in DB |
| `assertStatus(201)` / `assertOk()` / `assertForbidden()` | HTTP status |
| `assertRedirect('/x')` | redirect target |
| `assertJson([...])` / `assertJsonPath('data.0.title', 'X')` | JSON body |
| `assertSessionHasErrors('field')` | validation failed for a field |

### 20.4 HTTP, JSON & mocking **[A]**

```php
// Test a JSON API endpoint:
$this->getJson('/api/v1/posts')
     ->assertOk()
     ->assertJsonCount(15, 'data')
     ->assertJsonStructure(['data' => [['id', 'title']], 'meta', 'links']);

// Sanctum auth in tests:
\Laravel\Sanctum\Sanctum::actingAs(User::factory()->create(), ['posts:write']);

// Fake side-effects so nothing real happens AND you can assert on them:
\Illuminate\Support\Facades\Mail::fake();
\Illuminate\Support\Facades\Queue::fake();
\Illuminate\Support\Facades\Event::fake();
\Illuminate\Support\Facades\Storage::fake('public');

// …run the code…
Mail::assertSent(\App\Mail\OrderShipped::class);
Queue::assertPushed(\App\Jobs\SendWelcomeEmail::class);
```

```bash
php artisan test                       # run everything
php artisan test --filter=PostTest     # one test/file
php artisan test --parallel            # run across cores (big speedup)
```

⚡ **Telescope** (a local debugging dashboard) and **Pulse** complement tests by showing real requests/queries/jobs while you develop — install Telescope in dev for visibility.

---

## 21. Production & Performance — Caching, Octane, Deployment, Hardening

### 21.1 The production optimization commands **[A]**

Local dev re-reads config, routes, and views on every request for convenience. In **production** you compile them into fast cached files. The umbrella command:

```bash
php artisan optimize         # caches config + routes + events + views in one go
# individually:
php artisan config:cache     # merge all config into one file (⚠️ now env() returns null — Section 2.4.1)
php artisan route:cache      # serialize routes (⚠️ closures in routes break this — use controllers)
php artisan view:cache       # precompile Blade
php artisan event:cache
```

⚠️ **Run these on every deploy, AFTER pulling new code** — and `php artisan optimize:clear` if you change `.env` or config, otherwise stale cached config silently wins. Also run `composer install --no-dev --optimize-autoloader` and `npm run build`.

### 21.2 Octane — long-lived workers for huge throughput **[A]**

Normally PHP **boots the whole framework on every request** (the classic share-nothing model). **Octane** boots the app **once** and keeps it resident in worker processes powered by **Swoole** or **FrankenPHP**, serving many requests from memory. This can multiply throughput several-fold.

```bash
composer require laravel/octane
php artisan octane:install        # choose FrankenPHP or Swoole
php artisan octane:start --workers=4 --max-requests=500
```

⚠️ **The Octane caveat:** because the app stays in memory, **state leaks between requests** if you're careless. Avoid storing request-specific data in singletons or static properties; be wary of global state. Use `scoped()` bindings (Section 6.4) for per-request services. Code that's fine under classic PHP can leak under Octane — test before adopting.

### 21.3 Deployment options **[A]**

| Path | What | When |
|---|---|---|
| **Forge** | Laravel's server-provisioning SaaS — sets up Nginx + PHP-FPM + queue workers + scheduler + TLS on your VPS (DigitalOcean/AWS/…) | Most apps; you want a real server without ops drudgery |
| **Vapor** | Serverless on AWS Lambda (auto-scaling, no servers) | Spiky traffic, want serverless |
| **Cloud** | Laravel's managed hosting platform | Fully managed, minimal config |
| **Manual VPS** | You configure **Nginx + PHP-FPM** yourself | Full control / learning |

For a manual VPS, the stack is **[Nginx](NGINX_GUIDE.md) → PHP-FPM → your app**, with the web root set to `public/`. A minimal Nginx server block:

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/blog/public;          # ← public/, NOT the project root (security!)
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;   # send everything to Laravel
    }
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;       # PHP-FPM
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }
    location ~ /\.(?!well-known).* { deny all; }          # block dotfiles like .env
}
```

See the [Nginx](NGINX_GUIDE.md), [Docker](DOCKER_GUIDE.md), [Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md), and [Linux Server Administration] guides for the surrounding infrastructure, and [Git](GIT_GUIDE.md) for deploy workflows.

### 21.4 Running queues & the scheduler in production **[A]**

Two long-running processes must be **supervised** (auto-restart on crash/boot) — use **systemd** or **Supervisor**:

1. **The queue worker** (`php artisan queue:work redis ...`) — or **Horizon** (`php artisan horizon`).
2. **The scheduler** — a single cron line running `php artisan schedule:run` every minute (Section 16.2).

Always run `php artisan queue:restart` on deploy so workers pick up new code.

### 21.5 Security hardening checklist **[A]**

Laravel is secure by default, but you must keep it that way. The essentials:

| Concern | Defense |
|---|---|
| **Debug leaks** | `APP_DEBUG=false` and `APP_ENV=production` — debug pages expose secrets & stack traces |
| **CSRF** | On by default for `web` routes via `@csrf`; don't disable it blindly |
| **XSS** | Blade `{{ }}` auto-escapes — use `{!! !!}` only for sanitized HTML |
| **SQL injection** | Eloquent/Query Builder parameterize automatically — never interpolate input into raw SQL (Section 9.3) |
| **Mass assignment** | Use `$fillable` + validate; never `unguard()` with raw input (Section 8.4) |
| **Secrets** | Keep them in `.env` (git-ignored); web root is `public/` so `.env` is unreachable |
| **HTTPS** | Force it: `URL::forceScheme('https')` and `trustProxies()` behind a load balancer |
| **Auth throttling** | Rate-limit login & API routes (Section 11.5) |
| **Dependencies** | `composer audit`; keep Laravel & packages patched |
| **Headers** | Set HSTS, CSP, X-Frame-Options (at Nginx or via middleware) |
| **File permissions** | `storage/` and `bootstrap/cache/` writable by the web user; nothing else |

⚠️ The single most common production mistake: **leaving `APP_DEBUG=true`**. It turns every error page into a full stack trace with environment variables visible. Always `false` in production.

---

## 22. Gotchas & Best Practices

A field guide to the mistakes Laravel developers actually make. Several were flagged inline above; here they are collected.

### 22.1 The N+1 query problem **[I/A]**
**Symptom:** a page issues hundreds of queries; it's slow. **Cause:** accessing a relationship inside a loop without eager loading. **Fix:** `with()` (Section 8.6). **Prevention:** `Model::preventLazyLoading()` in dev so accidental lazy loads throw. Use **Telescope**/**Pulse** or `php artisan db:` debug tools to spot query counts.

### 22.2 `env()` outside config breaks under `config:cache` **[B/I]**
Calling `env()` anywhere but `config/*.php` returns `null` in production after `config:cache` (Section 2.4.1). **Always** read via `config()`. If config seems wrong after a deploy, run `php artisan config:clear`.

### 22.3 The slimmed-skeleton confusion (coming from L8–L10) **[I]**
There is **no `app/Http/Kernel.php`**, no `Console/Kernel.php`, no `Exceptions/Handler.php` in Laravel 11/12. Middleware, exceptions, and the scheduler moved (Sections 2.5, 7, 16). When a tutorial references those files, translate: middleware/exceptions → `bootstrap/app.php`; schedule → `routes/console.php`; commands auto-register. This is the #1 source of "my project doesn't match the guide."

### 22.4 Mass-assignment surprises **[I]**
Forgetting `$fillable` → `MassAssignmentException`, or being too permissive → users set fields they shouldn't. Be explicit; validate; pass `$request->validated()` (a whitelist) into `create()`, not `$request->all()`.

### 22.5 Eloquent vs Query Builder — pick the right tool **[I]**
Use **Eloquent** for working with model objects, relationships, and business logic. Use the **Query Builder/`DB`** for reports, heavy aggregates, and bulk updates where hydrating thousands of models would waste memory. Don't hydrate 100k models to compute a `SUM` — `DB::table()->sum()` instead.

### 22.6 Fat controllers **[I/A]**
Controllers that validate, authorize, run business logic, send mail, and format output become untestable monsters. Push validation/authz into **Form Requests** (Section 10.3), business logic into **service classes** or **model methods/actions**, output into **API Resources** (Section 13), and slow work into **jobs** (Section 14). A controller should read like a short summary.

### 22.7 Dispatching jobs/mail inside a transaction **[I/A]**
A queued job can run before the transaction commits and find missing data. Use `->afterCommit()` or configure `after_commit` on the connection (Section 9.4).

### 22.8 `queue:work` doesn't reload code **[A]**
A running worker holds old code in memory until restarted. Run `php artisan queue:restart` on every deploy (Section 14.3).

### 22.9 Forgetting `storage:link` **[I]**
Uploaded files on the `public` disk 404 because the symlink doesn't exist. Run `php artisan storage:link` (Section 19.2) — and remember it on each fresh deploy.

### 22.10 General best practices **[I/A]**
- **Name every route**; never hard-code URLs (Section 3.4).
- **Type everything** — type-hints power the container and your IDE.
- **Run Pint** (`./vendor/bin/pint`) so formatting is consistent and never debated.
- **Keep migrations forward-only in prod**; never edit a shipped migration — add a new one.
- **Don't put logic in Blade** beyond simple presentation.
- **Use `config()` and the DI container** rather than reaching for globals/singletons by hand.
- **Read `php artisan about` and `route:list`** when onboarding to any app.

---

## 23. Study Path & Build-to-Learn Projects

A concrete order to learn Laravel and the projects that cement each layer. Build, don't just read — Laravel rewards hands-on practice in `tinker` and small apps.

### Phase 1 — Foundations (Week 1) **[B]**
1. Confirm your [PHP](PHP_GUIDE.md) is solid (classes, namespaces, Composer, closures).
2. Install **Herd** (or Sail/Docker); `laravel new blog` with **no starter kit** and **SQLite**.
3. Tour the slimmed structure (Section 2.5); run `php artisan about`, `route:list`, `tinker`.
4. Routing + controllers + Blade: build static pages, a layout component, and a contact form (with `@csrf`).
5. **Build:** a multi-page personal site with a working contact form (validation + flash message).

### Phase 2 — Data with Eloquent (Week 2) **[I]**
6. Migrations, models, `$fillable`, factories, seeders. Seed fake data.
7. Relationships (hasMany/belongsTo/belongsToMany), eager loading, and **fix an N+1** you deliberately create.
8. Resource controller + full CRUD with Form Request validation.
9. **Build:** a blog — posts, categories (belongsToMany), comments (hasMany), pagination, search.

### Phase 3 — Auth & authorization (Week 3) **[I]**
10. Scaffold a **starter kit** (Livewire or React) to see real auth; then study manual `Auth::attempt`.
11. Add **Policies** so only an author can edit/delete their post; gate the UI with `@can`.
12. **Build:** turn the blog multi-user — registration, ownership, an admin role via a gate.

### Phase 4 — APIs (Week 4) **[I/A]**
13. `php artisan install:api`; build a versioned JSON API with **API Resources** + pagination.
14. Protect it with **Sanctum** (both modes — understand tokens vs SPA cookies, Section 12.6) and rate limiting.
15. **Build:** a REST API for the blog consumed by a small [React 19](REACT_19_GUIDE.md) frontend (or Inertia).

### Phase 5 — The async stack (Weeks 5–6) **[A]**
16. **Queues & jobs:** move email-sending to a job; run `queue:work`; trigger and inspect a failure.
17. **Events & notifications:** notify users (mail + database channel) when something happens.
18. **Broadcasting with Reverb + Echo:** push a real-time notification/comment to the browser.
19. **Scheduler & a custom Artisan command:** a nightly cleanup + a daily report email.
20. **Build:** add to the blog — queued welcome emails, real-time comments via Reverb, a daily-digest command.

### Phase 6 — Testing & quality (Week 7) **[A]**
21. Write **Pest** feature tests for the CRUD and auth flows (`RefreshDatabase`, factories, `assertDatabaseHas`).
22. Fake mail/queue/events in tests; add a unit test for a service class. Run with `--parallel`.
23. Add **Pint** and a **GitHub Actions** CI run (tests + Pint) — see the GitHub Actions guide.

### Phase 7 — Production (Week 8) **[A]**
24. Switch cache/session/queue to **Redis**; add **Horizon**.
25. Deploy to a VPS behind **[Nginx](NGINX_GUIDE.md) + PHP-FPM** (or use **Forge**), web root = `public/`.
26. Run `php artisan optimize`, supervise the worker + scheduler, set `APP_DEBUG=false`, force HTTPS, run `composer audit`.
27. **Final project:** ship a production-ready app end to end.

### Build-to-learn projects (ordered by complexity)

| # | Project | Concepts practiced |
|---|---------|--------------------|
| 1 | Personal site + contact form | Routing, Blade components, validation, flash messages |
| 2 | Blog (single-user) | Eloquent, migrations, relationships, eager loading, CRUD, Form Requests |
| 3 | Multi-user blog | Auth, policies, gates, ownership, starter kits |
| 4 | Blog REST API + SPA | API Resources, Sanctum (both modes), pagination, versioning, [React 19](REACT_19_GUIDE.md) |
| 5 | Job board with email queue | Jobs, queues, notifications, scheduler, custom commands |
| 6 | Real-time chat / live comments | Events, broadcasting, **Reverb**, Echo, private channels |
| 7 | E-commerce backend | Transactions, complex relations, Stripe webhooks, inventory, caching, Horizon |
| 8 | SaaS dashboard | Multi-tenancy, billing, Pennant feature flags, Pulse monitoring, Octane |

---

*Official docs: **https://laravel.com/docs/12.x** — exceptionally good; bookmark it. All code here targets **Laravel 12 on PHP 8.2+ (current as of June 2026)**; ⚡ **Laravel 13** (~Q1 2026) is broadly compatible — confirm version-specific details against the official docs. Cross-reference the foundation ([PHP](PHP_GUIDE.md)), the data layer ([Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md), [PostgreSQL](POSTGRESQL_GUIDE.md), [SQLite3](SQLITE3_GUIDE.md), [Redis](REDIS_GUIDE.md), [Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md)), the frontend ([React 19](REACT_19_GUIDE.md)), the infrastructure ([Nginx](NGINX_GUIDE.md), [Docker](DOCKER_GUIDE.md), [Networking](NETWORKING_GUIDE.md)), and your workflow ([Git](GIT_GUIDE.md)).*
