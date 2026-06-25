# PHP — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** A developer who can already write a little code in *some* language and wants to learn **modern, professional PHP** — the PHP of 2026, which is a strict, statically-analyzable, fast, fully object-oriented language, not the loose PHP 5 of the 2000s. We start from "what is `<?php`?" and end at Fibers, the JIT, static analysis, and security hardening. Every concept is explained in **prose first** — *what it is, why it exists, when and how to use it, the key syntax and parameters, best practices, and the security implications* — and only then shown as **heavily-commented, runnable code**. Read it top to bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** Everything here is current **as of June 2026**.
> - **PHP 8.4** (released **November 2024**) is the established **current** version and the baseline for this guide. We use its headline features confidently: **property hooks**, **asymmetric visibility** (`public private(set)`), calling a method directly on `new` without wrapping parentheses (`new Foo()->bar()`), the `#[\Deprecated]` attribute, the new array functions `array_find` / `array_any` / `array_all` / `array_find_key`, and **lazy objects**.
> - **PHP 8.5** shipped **November 2025** and is the **newest** release. Its highlights — the **pipe operator `|>`**, the **`#[\NoDiscard]`** attribute, `clone`-with named-argument improvements, the `#[\Override]`-style ergonomics, and persistent CLI/`get_error_handler()` additions — are flagged with **⚡ Version note** throughout. Treat fine syntactic details of 8.5 as "confirm against the official docs," not gospel.
> - **Support timeline:** PHP **8.1** and **8.2** are end-of-life or security-only; **8.3** is **security-only**; **8.4** is in **active support**; **8.5** is the **newest**. PHP ships **one minor release every November** with ~2 years of active support and ~1 further year of security fixes. Always develop against a supported version.
> - Authoritative sources to confirm anything: **php.net** (the manual), **php-fig.org** (PSRs), **getcomposer.org**, and **phpstan.org**. This guide is offline-first and self-contained.
>
> Cross-references to other guides in this library: [Laravel](LARAVEL_GUIDE.md) (the dominant PHP framework), [Networking](NETWORKING_GUIDE.md), [Nginx](NGINX_GUIDE.md) (PHP-FPM lives behind it), [Docker](DOCKER_GUIDE.md), [PostgreSQL](POSTGRESQL_GUIDE.md), [SQLite3](SQLITE3_GUIDE.md), [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md), [Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md), [Git](GIT_GUIDE.md), and [JavaScript](JAVASCRIPT_GUIDE.md) appear where relevant.

---

## Table of Contents

1. [What PHP Is & the 2026 Landscape](#1-what-php-is--the-2026-landscape) **[B]**
2. [Setup & Running PHP — CLI, Built-in Server, php.ini, FPM, Composer](#2-setup--running-php--cli-built-in-server-phpini-fpm-composer) **[B]**
3. [Syntax Fundamentals — Tags, Variables, Types, Strings](#3-syntax-fundamentals--tags-variables-types-strings) **[B]**
4. [Control Flow — if/else, match, Loops, Template Syntax](#4-control-flow--ifelse-match-loops-template-syntax) **[B]**
5. [Functions — Typed Params, Named Args, Closures, First-Class Callables](#5-functions--typed-params-named-args-closures-first-class-callables) **[B/I]**
6. [Arrays in Depth — PHP's Workhorse](#6-arrays-in-depth--phps-workhorse) **[B/I]**
7. [OOP Part 1 — Classes, Properties, Promotion, Static](#7-oop-part-1--classes-properties-promotion-static) **[I]**
8. [OOP Part 2 — Interfaces, Traits, Enums, Inheritance, Magic Methods](#8-oop-part-2--interfaces-traits-enums-inheritance-magic-methods) **[I/A]**
9. [The Modern Type System — Unions, Hooks, Asymmetric Visibility, Generics](#9-the-modern-type-system--unions-hooks-asymmetric-visibility-generics) **[I/A]**
10. [Namespaces & Autoloading — PSR-4 & Composer](#10-namespaces--autoloading--psr-4--composer) **[I]**
11. [Errors & Exceptions](#11-errors--exceptions) **[I]**
12. [File System, OS Info & Command Execution](#12-file-system-os-info--command-execution) **[I]**
13. [Working with Data — JSON, Dates, Regex, Strings](#13-working-with-data--json-dates-regex-strings) **[I]**
14. [Databases with PDO — Prepared Statements & Transactions](#14-databases-with-pdo--prepared-statements--transactions) **[I/A]**
15. [PHP for the Web — Requests, Sessions, Forms, Security](#15-php-for-the-web--requests-sessions-forms-security) **[I/A]**
16. [The Ecosystem — Composer, PSRs, SemVer, Frameworks](#16-the-ecosystem--composer-psrs-semver-frameworks) **[I]**
17. [Advanced Features — Attributes, Generators, SPL, Fibers, FFI](#17-advanced-features--attributes-generators-spl-fibers-ffi) **[A]**
18. [Tooling & Quality — PHPUnit/Pest, PHPStan, CS-Fixer, Xdebug](#18-tooling--quality--phpunitpest-phpstan-cs-fixer-xdebug) **[A]**
19. [Performance — OPcache, JIT, Preloading, Profiling](#19-performance--opcache-jit-preloading-profiling) **[A]**
20. [Security Best Practices — Consolidated](#20-security-best-practices--consolidated) **[A]**
21. [Gotchas & Best Practices](#21-gotchas--best-practices) **[I/A]**
22. [Study Path & Build-to-Learn Projects](#22-study-path--build-to-learn-projects)

---

## 1. What PHP Is & the 2026 Landscape

### 1.1 The one-paragraph definition **[B]**

**PHP** (a recursive acronym for *PHP: Hypertext Preprocessor*) is a general-purpose, dynamically-typed, server-side scripting language whose original superpower is that it can be **embedded directly inside HTML**: anything outside `<?php ... ?>` tags is emitted verbatim, anything inside is executed. That origin made it the default language of the web in the 2000s — and also gave it a reputation for sloppy, insecure code, because the language let you get away with almost anything. **Modern PHP is a different language wearing the same name.** Since 8.0 (2020) it has had: scalar and union type declarations, JIT compilation, attributes, enums, readonly properties, named arguments, the `match` expression, fibers, and — in 8.4 — property hooks and asymmetric visibility. Written with `declare(strict_types=1);` and analyzed with PHPStan at high levels, today's PHP is **strict, typed, and refactor-safe**. This guide teaches *that* PHP. We will not teach the PHP 5 footguns except to warn you off them in §21.

### 1.2 Where PHP runs **[B]**

PHP has three main execution contexts, and understanding them prevents a lot of confusion:

| Context | What it is | Typical use |
|---|---|---|
| **CLI** (`php script.php`) | The command-line interpreter. A normal program: reads `argv`, has a `STDIN`/`STDOUT`/`STDERR`, runs once and exits. | Scripts, queue workers, cron jobs, build tools, Composer itself. |
| **PHP-FPM** (FastCGI Process Manager) | A pool of long-lived PHP worker processes that a web server (usually [Nginx](NGINX_GUIDE.md)) hands requests to over the FastCGI protocol. | The standard way to serve PHP web apps in production. |
| **Built-in dev server** (`php -S`) | A tiny single-process web server bundled with PHP. **Development only** — never production. | Local development and quick demos. |

Historically PHP also ran as an Apache module (`mod_php`); in 2026 the **Nginx + PHP-FPM** pairing is the mainstream production stack — see the [Nginx](NGINX_GUIDE.md) guide for the `fastcgi_pass` wiring, and [Docker](DOCKER_GUIDE.md) for containerizing the FPM pool.

The crucial mental model for web PHP is **shared-nothing**: each HTTP request gets a *fresh* PHP process state — superglobals populated, your code run from scratch, then everything torn down. There is no long-lived application object holding state between requests by default (unlike Node or Go). This makes PHP wonderfully simple to reason about and crash-resistant (one request blowing up doesn't poison the next), at the cost of needing an external store (a database, [Redis](REDIS_GUIDE.md), sessions) for anything that must persist. Async runtimes like Swoole and ReactPHP (§17) break this model deliberately for performance.

### 1.3 The 2026 renaissance & the release cadence **[B]**

PHP today is fast (the 8.x branch is several times faster than 5.6), tooled (Composer, PHPStan, Pest, modern IDEs), and standardized (PSRs). It powers a huge share of the web — WordPress alone is on a large fraction of all sites — plus modern application frameworks ([Laravel](LARAVEL_GUIDE.md) and Symfony) that are genuinely pleasant. The release cadence is predictable: **one minor version each November**, with roughly **two years of active (bug-fix) support** followed by **one year of security-only** support.

```text
Version   Released        Status (June 2026)
8.1       Nov 2021        End of active; security only (ending soon)
8.2       Dec 2022        Security only
8.3       Nov 2023        Security only
8.4       Nov 2024        ACTIVE support  ← baseline for this guide
8.5       Nov 2025        NEWEST (active)  ← pipe operator, #[\NoDiscard]
8.6       Nov 2026        (expected)
```

**Rule of thumb:** target the oldest *actively supported* version your hosting allows (8.4 here), and test against the newest. Never start a new project on an EOL version.

---

## 2. Setup & Running PHP — CLI, Built-in Server, php.ini, FPM, Composer

### 2.1 Installing PHP **[B]**

You want PHP **8.4+** with the common extensions (`mbstring`, `pdo`, `pdo_mysql`/`pdo_pgsql`, `curl`, `intl`, `opcache`, `zip`). How you install depends on your OS:

```bash
# macOS (Homebrew)
brew install php            # installs the current stable (8.4+)

# Debian / Ubuntu (via the widely-used Ondřej Surý PPA for recent versions)
sudo add-apt-repository ppa:ondrej/php
sudo apt update
sudo apt install php8.4-cli php8.4-fpm php8.4-mbstring php8.4-pdo \
                 php8.4-pgsql php8.4-curl php8.4-intl php8.4-zip

# Windows: download the VS-built ZIP from windows.php.net, unzip,
# add the folder to PATH. Or use the official Docker image (recommended):
#   docker run --rm -it php:8.4-cli php -v
```

Verify and inspect:

```bash
php -v                      # version banner + whether OPcache/JIT is loaded
php -m                      # list compiled/loaded extensions (modules)
php --ini                   # which php.ini file(s) are actually being read
php -i | less               # full phpinfo() for the CLI SAPI
php -r 'echo PHP_VERSION, PHP_EOL;'   # run a one-liner from the shell
```

> **Tip:** PHP has *separate* `php.ini` files per SAPI (Server API). The CLI uses one ini; PHP-FPM uses another. A setting working on the command line but not on the web (or vice-versa) is almost always because you edited the wrong ini. `php --ini` tells you the CLI path; for FPM check `/etc/php/8.4/fpm/php.ini` (Linux) or the path in `phpinfo()` from a web request.

### 2.2 The CLI & the built-in server **[B]**

```bash
php hello.php               # run a script
php -a                      # interactive REPL (basic; consider `psysh` for a good one)
php -l script.php           # lint: check syntax only, don't run
php -r 'var_dump(PHP_INT_MAX);'   # eval an expression

# The built-in development web server (DEV ONLY — single-threaded, not for prod):
php -S localhost:8000               # serve current dir; index.php is the default
php -S localhost:8000 -t public/    # set the document root to public/
php -S localhost:8000 router.php    # route every request through router.php (front controller)
```

A minimal `router.php` front controller — every modern framework uses this pattern (one entry point that dispatches):

```php
<?php
declare(strict_types=1);

// If the request maps to a real static file (css, js, images), let the
// built-in server serve it directly by returning false.
$path = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
if ($path !== '/' && file_exists(__DIR__ . '/public' . $path)) {
    return false;
}

// Otherwise, this is an application route — handle it ourselves.
echo "You requested: " . htmlspecialchars($path, ENT_QUOTES);
```

### 2.3 php.ini — the settings you actually touch **[B]**

`php.ini` is the global configuration. You will not memorize it, but these are the directives that matter in practice. Note the sharp difference between **development** and **production** values, especially for error display (showing errors to users is a security leak).

```ini
; ---- Error handling ----
; DEVELOPMENT: see everything, on screen.
display_errors = On
error_reporting = E_ALL
; PRODUCTION: never show errors to users; log them instead.
;   display_errors = Off
;   log_errors = On
;   error_log = /var/log/php/error.log

; ---- Resource limits (per request) ----
memory_limit = 256M         ; max memory one script may use
max_execution_time = 30     ; seconds before a script is killed (CLI is unlimited)
upload_max_filesize = 16M   ; biggest single uploaded file...
post_max_size = 20M         ; ...must be >= upload_max_filesize + form overhead

; ---- Locale / encoding ----
default_charset = "UTF-8"
mbstring.internal_encoding = "UTF-8"
date.timezone = "UTC"       ; ALWAYS set this; default to UTC, convert on display

; ---- Performance (see §19) ----
opcache.enable = 1
opcache.enable_cli = 0      ; usually off for CLI; on if you preload for workers
opcache.jit = tracing       ; enable the JIT
opcache.jit_buffer_size = 64M

; ---- Security ----
expose_php = Off            ; don't leak the PHP version in the X-Powered-By header
session.cookie_httponly = 1 ; JS can't read the session cookie (anti-XSS)
session.cookie_secure = 1   ; only send the cookie over HTTPS
session.cookie_samesite = "Lax"
```

You can also set ini values per-invocation (`php -d memory_limit=512M script.php`) or at runtime for the few that allow it (`ini_set('display_errors', '0');`).

### 2.4 PHP-FPM in one breath **[B]**

**PHP-FPM** is a process manager that keeps a *pool* of PHP worker processes ready and feeds them requests from a web server over FastCGI. You rarely run it by hand — your web server talks to it via a Unix socket or TCP port. The key tuning lives in the pool config (`/etc/php/8.4/fpm/pool.d/www.conf`):

```ini
[www]
user = www-data
group = www-data
; Listen on a Unix socket (fastest, same-host) — Nginx points fastcgi_pass here.
listen = /run/php/php8.4-fpm.sock

; Process management strategy.
pm = dynamic           ; spawn/kill workers based on load
pm.max_children = 20   ; HARD cap: max concurrent requests = this number
pm.start_servers = 4
pm.min_spare_servers = 2
pm.max_spare_servers = 6
pm.max_requests = 500  ; recycle a worker after N requests (guards against leaks)
```

The single most important number is `pm.max_children`: it is your **maximum concurrency**. Each worker holds memory; `max_children × memory_limit` must fit in RAM. See the [Nginx](NGINX_GUIDE.md) guide for the matching `fastcgi_pass unix:/run/php/php8.4-fpm.sock;` block, and [Docker](DOCKER_GUIDE.md) for running `php:8.4-fpm` as a container alongside an `nginx` container.

### 2.5 Composer & version management **[B]**

**Composer** is PHP's dependency manager (analogous to npm for [JavaScript](JAVASCRIPT_GUIDE.md), or `go mod`). It reads `composer.json`, resolves a compatible dependency graph, writes the exact resolved versions to `composer.lock`, and installs everything under `vendor/` with a PSR-4 autoloader. You will use it in every real project.

```bash
# Install Composer globally (see getcomposer.org for the verified installer script).
# Then:
composer init                 # interactive: create composer.json
composer require monolog/monolog          # add a runtime dependency
composer require --dev phpunit/phpunit     # add a dev-only dependency
composer install              # install exactly what composer.lock pins (CI/production)
composer update               # re-resolve to newest allowed; rewrite the lock file
composer dump-autoload -o     # regenerate an optimized classmap autoloader
```

For managing multiple PHP **versions** on one machine, use your OS package manager's versioned packages (`php8.3`, `php8.4`) and switch the default, or tools like `phpenv`. In containers, you simply pick the base image tag (`php:8.4-fpm`). We cover Composer in depth in §16.

---

## 3. Syntax Fundamentals — Tags, Variables, Types, Strings

### 3.1 Tags, statements, and `declare(strict_types=1)` **[B]**

PHP code lives between `<?php` and `?>`. Anything outside those tags is output literally — that is the "embedded in HTML" heritage. In a **pure-PHP file** (a class, a library, anything that isn't a template), you write the opening `<?php` and **omit the closing `?>`**: leaving it off prevents accidental whitespace after it from being sent to the browser (a classic "headers already sent" bug).

The single most important line in modern PHP is **`declare(strict_types=1);`**. It must be the **very first statement** in a file. Without it, PHP is in *coercive* mode: pass `"5"` (a string) where an `int` is declared and PHP silently converts it. In **strict** mode, the type must match (with the one sane exception that an `int` is accepted where a `float` is wanted). Strict types turn a category of silent bugs into immediate, loud `TypeError`s. **Use it in every file you write.**

```php
<?php
declare(strict_types=1);   // FIRST line, every file. Non-negotiable in modern PHP.

// Statements end with a semicolon.
echo "Hello, world\n";

// Three comment styles:
// single line
# also single line (shell style)
/* block
   comment */
```

### 3.2 Variables & the type system **[B]**

Variables start with `$`, are case-sensitive, and need no declaration. PHP is dynamically typed: a variable's *value* has a type; the *variable* does not. PHP has these built-in types:

| Category | Types | Notes |
|---|---|---|
| **Scalars** | `int`, `float`, `string`, `bool` | `int` is 64-bit on 64-bit platforms (`PHP_INT_MAX`). |
| **Special** | `null`, `array`, `object`, `callable`, `iterable`, `resource` | `resource` is a legacy handle (file, db) being phased out for objects. |
| **Compound** | `array`, `object` | Arrays are PHP's super-structure (§6). |

```php
<?php
declare(strict_types=1);

$count   = 42;            // int
$price   = 19.99;         // float
$name    = "Ada";         // string
$active  = true;          // bool
$nothing = null;          // null — the absence of a value

// Inspect a value's type and contents — your two best debugging friends:
var_dump($count);         // int(42)        — type + value, recursive for arrays/objects
var_dump($price);         // float(19.99)
echo gettype($name), "\n"; // string
echo get_debug_type($active), "\n";  // bool  — modern, accurate type name (use this)

// Explicit casts when you really mean to convert:
$n = (int) "123abc";      // 123  (parses leading digits)
$f = (float) "3.14";      // 3.14
$s = (string) 42;         // "42"
$b = (bool) 0;            // false

// Null coalescing: "$x if it's set and not null, else default".
$page = $_GET['page'] ?? 1;          // no notice if 'page' is missing
$config['timeout'] ??= 30;           // assign only if currently null/unset
```

**Falsy values** (things that evaluate to `false` in a boolean context) are a common source of bugs: `false`, `0`, `0.0`, `""`, `"0"`, `[]` (empty array), and `null`. Note that `"0"` is falsy but `"0.0"` and `"false"` are truthy. Prefer explicit checks (`$x === ''`, `$x === null`) over relying on truthiness.

### 3.3 Constants **[B]**

Constants are immutable named values. Use `const` at file/class scope (resolved at compile time, the modern default) and reserve `define()` for the rare runtime-computed case. Constant *names* are case-sensitive and conventionally `UPPER_SNAKE_CASE`.

```php
<?php
declare(strict_types=1);

const MAX_RETRIES = 3;                 // compile-time constant (preferred)
const SUPPORTED = ['en', 'fr', 'de'];  // arrays are allowed
define('APP_ENV', getenv('APP_ENV') ?: 'production');  // runtime value

echo MAX_RETRIES;                      // 3
echo PHP_VERSION;                      // a built-in magic constant
echo __DIR__, ' ', __FILE__, ' ', __LINE__;  // magic constants: paths & location
```

### 3.4 Strings — quoting, interpolation, heredoc/nowdoc **[B]**

PHP has four string syntaxes, and the difference is **whether variables and escapes are processed**:

- **Single quotes** `'...'` — *literal*. No variable interpolation; only `\\` and `\'` are special. Fastest, safest for literals.
- **Double quotes** `"..."` — *interpolated*. `$var` and `{$expr}` are substituted; escapes like `\n`, `\t`, `\u{1F600}` work.
- **Heredoc** `<<<EOT ... EOT;` — like double quotes but multi-line, no surrounding quotes needed.
- **Nowdoc** `<<<'EOT' ... EOT;` — like single quotes (literal) but multi-line.

```php
<?php
declare(strict_types=1);

$user = 'Ada';
$obj  = (object) ['name' => 'Grace'];

echo 'Hello $user';          // Hello $user   (literal — no interpolation)
echo "Hello $user\n";        // Hello Ada     (interpolated, newline)

// Use {braces} for anything non-trivial (array access, properties, method calls):
echo "Name: {$obj->name}\n";
$arr = ['k' => 'v'];
echo "Value: {$arr['k']}\n"; // braces required for quoted array keys

// Heredoc: great for multi-line templates / SQL / emails.
$body = <<<HTML
    <p>Hello {$user},</p>
    <p>Welcome aboard.</p>
    HTML;   // closing marker may be indented (PHP 7.3+); that indentation is stripped

// Nowdoc: literal multi-line, no interpolation (e.g. code samples, regexes).
$raw = <<<'TXT'
    Literal $user, no escapes processed here.
    TXT;

// Concatenation uses the DOT operator (not +), and .= to append:
$greeting = 'Hello, ' . $user . '!';
$greeting .= ' Welcome.';
```

> **Security note:** building HTML or SQL by interpolating variables into strings is the root of XSS and SQL-injection. **Never** interpolate untrusted data into HTML without `htmlspecialchars()` (§15) or into SQL without prepared statements (§14). String interpolation is for trusted, structural text only.

PHP strings are **byte strings**, not Unicode strings. `strlen("é")` is 2 (UTF-8 bytes), not 1. For human-correct length, case, and substringing of UTF-8 text, use the **`mbstring`** functions: `mb_strlen`, `mb_strtoupper`, `mb_substr`, etc. (§13).

---

## 4. Control Flow — if/else, match, Loops, Template Syntax

### 4.1 Conditionals & the comparison operators **[B]**

```php
<?php
declare(strict_types=1);

$score = 72;

if ($score >= 90) {
    $grade = 'A';
} elseif ($score >= 70) {       // note: elseif is one word
    $grade = 'B';
} else {
    $grade = 'C';
}

// Ternary and the short ("Elvis") ternary:
$label = $score >= 60 ? 'pass' : 'fail';
$name  = $input ?: 'anonymous';   // $input if truthy, else 'anonymous'
```

**Always use the identity operators `===` / `!==`** (compare value *and* type) rather than `==` / `!=` (loose, with type juggling). Loose comparison is one of PHP's historical footguns — `0 == "a"` was true before PHP 8 — and is covered as a gotcha in §21. The full operator set:

| Operator | Meaning |
|---|---|
| `===`, `!==` | Identical / not identical (type-strict) — **prefer these** |
| `==`, `!=` | Equal / not equal (loose, type-juggling) — avoid |
| `<`, `<=`, `>`, `>=` | Ordering |
| `<=>` | Spaceship: returns -1/0/1; ideal for sort callbacks |
| `??`, `??=` | Null coalescing / coalescing assignment |
| `?:` | Elvis (short ternary) |
| `and`, `or`, `xor` | Logical, but **lower precedence** than `=` — prefer `&&`/`||` |

### 4.2 `match` — the modern, safe switch **[I]**

The **`match` expression** (PHP 8.0+) is the modern replacement for most `switch` statements, and you should reach for it first. Four reasons it is better:

1. It is an **expression** — it *returns* a value you can assign.
2. It uses **strict `===` comparison** (no type juggling like `switch`).
3. There is **no fall-through** — no `break` needed, no accidental cascade.
4. It is **exhaustive**: an unmatched value throws `\UnhandledMatchError` unless you provide a `default`, so adding a new enum case without handling it fails loudly.

```php
<?php
declare(strict_types=1);

$status = 404;

// match RETURNS a value; arms use === ; commas group conditions.
$message = match ($status) {
    200, 201, 204 => 'Success',
    301, 302      => 'Redirect',
    404           => 'Not Found',
    500           => 'Server Error',
    default       => 'Unknown',     // omit -> UnhandledMatchError on no match
};

// match(true) is a clean way to express ranges / arbitrary conditions:
$tier = match (true) {
    $score >= 90 => 'gold',
    $score >= 70 => 'silver',
    default      => 'bronze',
};
```

Reach for the old `switch` only when you genuinely want fall-through or loose comparison (rare). When you do, remember every case needs `break`.

### 4.3 Loops **[B]**

```php
<?php
declare(strict_types=1);

// Classic for
for ($i = 0; $i < 3; $i++) {
    echo $i;                 // 012
}

// while / do-while
$n = 0;
while ($n < 3) { $n++; }
do { $n--; } while ($n > 0);   // body runs at least once

// foreach — THE loop for arrays/iterables. Use it 95% of the time.
$fruits = ['apple', 'banana', 'cherry'];
foreach ($fruits as $fruit) {
    echo $fruit, "\n";
}
foreach (['a' => 1, 'b' => 2] as $key => $value) {
    echo "$key=$value\n";
}

// Modify in place with a reference (&) — then ALWAYS unset() the reference,
// or the dangling $v will clobber the last element on the next foreach.
foreach ($fruits as &$v) {
    $v = strtoupper($v);
}
unset($v);   // critical cleanup

// continue / break work as elsewhere; break N exits N nested loops.
foreach ($fruits as $f) {
    if ($f === 'BANANA') continue;
    if ($f === 'CHERRY') break;
    echo $f;
}
```

### 4.4 Alternative syntax for templates **[B]**

When PHP is embedded in HTML, the curly-brace style reads badly. PHP offers an **alternative syntax** — `if:`/`endif;`, `foreach:`/`endforeach;` — that interleaves cleanly with markup. This is the basis of every PHP templating approach (and of Blade/Twig under the hood).

```php
<ul>
<?php foreach ($users as $user): ?>
    <li><?= htmlspecialchars($user->name, ENT_QUOTES) ?></li>
<?php endforeach; ?>
</ul>

<?php if ($isAdmin): ?>
    <a href="/admin">Admin panel</a>
<?php else: ?>
    <p>Welcome, member.</p>
<?php endif; ?>
```

`<?= ... ?>` is short-hand for `<?php echo ... ?>` and is always available. **Every value echoed into HTML must be escaped** with `htmlspecialchars()` (§15) — this is the front line against XSS.

---

## 5. Functions — Typed Params, Named Args, Closures, First-Class Callables

### 5.1 Declaring functions with types **[B]**

A modern PHP function **declares the type of every parameter and the return type.** Types are documentation the engine enforces (especially under `strict_types=1`), they enable IDE autocompletion and static analysis, and they catch whole classes of bugs at the boundary.

```php
<?php
declare(strict_types=1);

// int $a, int $b are typed params; ": int" is the return type.
function add(int $a, int $b): int {
    return $a + $b;
}

// ": void" means "returns nothing"; ": never" (§9) means "never returns".
function logMessage(string $msg): void {
    error_log($msg);
}

// Nullable type with ?  -> "string or null". Default values come last.
function greet(?string $name = null): string {
    return 'Hello, ' . ($name ?? 'stranger');
}

echo add(2, 3);              // 5
// add(2, "3");             // TypeError under strict_types — caught immediately
```

### 5.2 Named arguments, variadics, defaults **[B/I]**

**Named arguments** (PHP 8.0) let callers pass arguments by parameter name in any order, which is invaluable when a function has several optional/boolean parameters — `setPosition(y: 10)` is self-documenting where `setPosition(0, 10, false, true)` is a riddle. **Variadics** (`...$args`) collect any number of trailing arguments into an array; the **spread** operator unpacks an array back into arguments.

```php
<?php
declare(strict_types=1);

function createUser(
    string $name,
    string $role = 'member',
    bool $active = true,
    bool $verified = false,
): array {
    return compact('name', 'role', 'active', 'verified');
}

// Named args: skip defaults, name only what you change, in any order.
$u = createUser(name: 'Ada', verified: true);
//   => ['name'=>'Ada','role'=>'member','active'=>true,'verified'=>true]

// Variadic: collect trailing args into an array.
function sum(int ...$numbers): int {
    return array_sum($numbers);
}
echo sum(1, 2, 3, 4);        // 10

// Spread: unpack an array into the call.
$vals = [1, 2, 3];
echo sum(...$vals);          // 6
echo sum(...$vals, ...[10]); // 16  (multiple spreads allowed)
```

### 5.3 Closures, arrow functions & `use` **[B/I]**

A **closure** is an anonymous function stored in a variable. By default a closure does **not** see the surrounding scope — you must import variables explicitly with `use` (`use (&$x)` to capture by reference). **Arrow functions** (`fn`) are concise single-expression closures that *automatically* capture the enclosing scope by value — perfect for array callbacks.

```php
<?php
declare(strict_types=1);

$tax = 0.2;

// Classic closure: explicit capture with use.
$withTax = function (float $net) use ($tax): float {
    return $net * (1 + $tax);
};
echo $withTax(100.0);        // 120

// Arrow function: auto-captures $tax by value, single expression, implicit return.
$withTax2 = fn(float $net): float => $net * (1 + $tax);

// Closures shine as array callbacks:
$prices = [10, 20, 30];
$gross  = array_map(fn($p) => $p * 1.2, $prices);   // [12, 24, 36]
$dear   = array_filter($prices, fn($p) => $p >= 20); // [1=>20, 2=>30]

// Capture by reference to mutate outer state (use sparingly):
$total = 0;
array_walk($prices, function ($p) use (&$total) { $total += $p; });
echo $total;                 // 60
```

### 5.4 First-class callable syntax `strlen(...)` **[I]**

PHP 8.1 added a clean way to turn any function or method into a `Closure` object: append `(...)`. This replaced the old, error-prone string/array callable forms (`'strlen'`, `[$obj, 'method']`) with something the IDE and static analyzer fully understand.

```php
<?php
declare(strict_types=1);

$len = strlen(...);                 // a Closure wrapping strlen
echo $len('hello');                 // 5

$upper = array_map(strtoupper(...), ['a', 'b']);  // ['A', 'B']

class Math {
    public function square(int $n): int { return $n * $n; }
    public static function cube(int $n): int { return $n ** 3; }
}
$m = new Math();
$sq = $m->square(...);              // bound instance method as a closure
$cb = Math::cube(...);              // static method as a closure
echo $sq(4), ' ', $cb(3);           // 16 27
```

> **⚡ Version note (PHP 8.5):** the new **pipe operator `|>`** chains callables left-to-right, reading like a Unix pipeline: `$result = $input |> trim(...) |> strtoupper(...);` passes `$input` through `trim`, then its result through `strtoupper`. It pairs naturally with first-class callables. Confirm the exact precedence and semantics against the php.net 8.5 docs before relying on edge cases.

---

## 6. Arrays in Depth — PHP's Workhorse

### 6.1 What a PHP array really is **[B]**

A PHP "array" is not a contiguous list like in C or [Go](GO_GUIDE.md) — it is an **ordered hash map**. It maps integer *or* string keys to values, preserves insertion order, and serves as list, dictionary, stack, queue, set, and tuple all at once. This versatility is why arrays are everywhere in PHP. The cost is that they are not the right tool for everything (a typed object or a `\SplStack` is sometimes clearer), and that "is this a list or a map?" is genuinely ambiguous — hence the helper `array_is_list()`.

```php
<?php
declare(strict_types=1);

// Indexed (list): keys 0,1,2 assigned automatically.
$colors = ['red', 'green', 'blue'];

// Associative (map): explicit string keys.
$user = ['name' => 'Ada', 'age' => 36, 'admin' => true];

// Mixed and nested freely.
$data = [
    'tags'  => ['php', 'web'],
    'meta'  => ['views' => 10],
];

echo $colors[0];             // red
echo $user['name'];          // Ada
echo $data['meta']['views']; // 10
$colors[] = 'yellow';        // append (next integer key)

var_dump(array_is_list($colors)); // true — keys are 0..n-1 with no gaps
var_dump(array_is_list($user));   // false
```

### 6.2 Destructuring & spread **[B/I]**

```php
<?php
declare(strict_types=1);

// List destructuring (positional).
[$r, $g, $b] = ['red', 'green', 'blue'];
[, , $third] = [1, 2, 3];          // skip elements with a blank slot -> $third = 3

// Associative destructuring (by key, any order).
['name' => $name, 'age' => $age] = ['age' => 36, 'name' => 'Ada'];

// Swap without a temp.
[$a, $b] = [$b, $a];

// foreach with destructuring.
$points = [[1, 2], [3, 4]];
foreach ($points as [$x, $y]) {
    echo "($x,$y) ";
}

// Spread to merge (string keys: later wins; integer keys: re-indexed).
$base   = ['a' => 1, 'b' => 2];
$merged = [...$base, 'b' => 20, 'c' => 3];   // ['a'=>1,'b'=>20,'c'=>3]
$list   = [...[1, 2], ...[3, 4]];            // [1,2,3,4]
```

### 6.3 The essential `array_*` functions **[B/I]**

PHP has ~80 array functions; you use perhaps 20 daily. The cornerstones are the functional trio — **`array_map`** (transform each element), **`array_filter`** (keep elements matching a predicate), and **`array_reduce`** (fold to a single value). A persistent gotcha: **`array_filter` preserves keys**, so the result of filtering a list is often a sparse array — re-index with `array_values()` when you need a clean list.

| Function | Purpose |
|---|---|
| `array_map(fn, $a)` | Transform each element → new array |
| `array_filter($a, fn)` | Keep elements where `fn` is truthy (keeps keys!) |
| `array_reduce($a, fn, $init)` | Fold to a single accumulated value |
| `array_values($a)` / `array_keys($a)` | Extract values (re-index) / keys |
| `in_array($v, $a, true)` | Is value present? (pass `true` for strict ===) |
| `array_search($v, $a, true)` | First key of a value, or `false` |
| `array_column($rows, 'col')` | Pluck a column out of rows of records |
| `array_merge` / `+` | Merge (re-index ints) / union (left keys win) |
| `array_slice` / `array_splice` | Sub-array / remove-and-replace in place |
| `sort` / `usort` / `ksort` / `uasort` | Sort by value / custom / key / custom keeping keys |
| `array_key_exists($k, $a)` | Key present even if its value is `null` |
| `implode($glue, $a)` / `explode($d, $s)` | Array→string / string→array |

```php
<?php
declare(strict_types=1);

$nums = [1, 2, 3, 4, 5, 6];

$doubled = array_map(fn($n) => $n * 2, $nums);       // [2,4,6,8,10,12]
$evens   = array_filter($nums, fn($n) => $n % 2 === 0); // [1=>2,3=>4,5=>6] (keys kept!)
$evens   = array_values($evens);                      // [2,4,6] (re-indexed)
$total   = array_reduce($nums, fn($acc, $n) => $acc + $n, 0); // 21

// Pluck a column from rows (e.g. DB results), optionally keyed by id:
$rows = [['id' => 1, 'name' => 'Ada'], ['id' => 2, 'name' => 'Grace']];
$names = array_column($rows, 'name');           // ['Ada','Grace']
$byId  = array_column($rows, 'name', 'id');     // [1=>'Ada', 2=>'Grace']

// Custom sort with the spaceship operator:
$people = [['age' => 40], ['age' => 25], ['age' => 33]];
usort($people, fn($a, $b) => $a['age'] <=> $b['age']);  // ascending by age
```

### 6.4 The PHP 8.4 array find functions **[B/I]**

PHP 8.4 finally added the search functions every other language had. They take the array and a predicate (`fn(value, key)`):

- **`array_find($a, $fn)`** — the *first value* matching the predicate, or `null`.
- **`array_find_key($a, $fn)`** — the *key* of the first match, or `null`.
- **`array_any($a, $fn)`** — `true` if *any* element matches (short-circuits).
- **`array_all($a, $fn)`** — `true` if *every* element matches.

```php
<?php
declare(strict_types=1);

$users = [
    ['name' => 'Ada',   'age' => 36, 'active' => true],
    ['name' => 'Grace', 'age' => 41, 'active' => false],
    ['name' => 'Linus', 'age' => 29, 'active' => true],
];

// First user over 40 — returns the value (the row), or null if none.
$senior = array_find($users, fn($u) => $u['age'] > 40);   // Grace's row

// Does anyone match? Is everyone active?
$hasInactive = array_any($users, fn($u) => !$u['active']); // true
$allAdults   = array_all($users, fn($u) => $u['age'] >= 18); // true

// Key of the first inactive user:
$idx = array_find_key($users, fn($u) => !$u['active']);    // 1
```

Before 8.4 you wrote these by hand with `foreach` + `break` or abused `array_filter(...)[0]`; now there is one clear, short-circuiting function for each.

---

## 7. OOP Part 1 — Classes, Properties, Promotion, Static

### 7.1 Classes, typed & readonly properties, visibility **[I]**

A **class** is a blueprint pairing data (properties) with behavior (methods). Modern PHP classes declare a **type** on every property and an explicit **visibility** (`public`, `protected`, `private`). Visibility is the core of encapsulation: keep state `private`, expose behavior through methods, so the object's invariants can't be violated from outside. A **`readonly`** property (PHP 8.1) can be written exactly once — typically in the constructor — and is immutable thereafter, which makes value objects and DTOs trivially safe to share.

```php
<?php
declare(strict_types=1);

class BankAccount
{
    // Typed properties. private = only this class touches them.
    private int $balanceCents = 0;

    // readonly: set once (in the constructor), never changed again.
    public readonly string $owner;

    public function __construct(string $owner)
    {
        $this->owner = $owner;   // allowed: first write to a readonly prop
    }

    // Public methods are the controlled API to the private state.
    public function deposit(int $cents): void
    {
        if ($cents <= 0) {
            throw new \InvalidArgumentException('Deposit must be positive.');
        }
        $this->balanceCents += $cents;
    }

    public function balance(): float
    {
        return $this->balanceCents / 100;
    }
}

$acct = new BankAccount('Ada');
$acct->deposit(5000);
echo $acct->balance();       // 50
echo $acct->owner;           // Ada
// $acct->owner = 'X';       // Error: Cannot modify readonly property
// $acct->balanceCents;      // Error: private — encapsulation enforced
```

> **⚡ Version note (PHP 8.4):** you can now call a method directly on a `new` expression **without** wrapping parentheses: `new BankAccount('Ada')->balance()`. Pre-8.4 you needed `(new BankAccount('Ada'))->balance()`.

### 7.2 Constructor property promotion **[I]**

Declaring a property, adding a constructor parameter, and assigning `$this->x = $x` for each field is boilerplate. **Constructor property promotion** (PHP 8.0) collapses all three into one: put the visibility (and `readonly`) right on the constructor parameter and PHP declares and assigns the property for you. This is the standard modern style for value objects and dependency injection.

```php
<?php
declare(strict_types=1);

// All three lines (declare, param, assign) per field collapse into the signature.
final class Money
{
    public function __construct(
        public readonly int $amountCents,
        public readonly string $currency = 'USD',
    ) {}

    public function format(): string
    {
        return sprintf('%s %.2f', $this->currency, $this->amountCents / 100);
    }
}

$price = new Money(1999);
echo $price->format();       // USD 19.99
echo $price->currency;       // USD
```

### 7.3 Static members, constants, and `::class` **[I]**

**Static** properties and methods belong to the *class*, not to instances — accessed with `::` (the scope-resolution operator) and `self::`/`static::` from inside. Use static for things that are genuinely class-level: counters, factory methods, named constructors. Overusing static creates global mutable state that is hard to test, so prefer instances by default. **Class constants** (`const`) are immutable, class-scoped values, and may have visibility and types. The **`::class`** constant resolves to a class's fully-qualified name as a string — invaluable, refactor-safe, and what you pass to DI containers and `instanceof`-style logic.

```php
<?php
declare(strict_types=1);

final class Temperature
{
    public const float ABSOLUTE_ZERO_C = -273.15;  // typed constant (8.3+)
    private static int $instances = 0;

    private function __construct(public readonly float $celsius) {}

    // Named constructors (static factory methods) read better than new + setters.
    public static function fromCelsius(float $c): self
    {
        self::$instances++;
        return new self($c);
    }
    public static function fromFahrenheit(float $f): self
    {
        return self::fromCelsius(($f - 32) * 5 / 9);
    }

    public static function count(): int { return self::$instances; }
}

$t = Temperature::fromFahrenheit(98.6);
echo round($t->celsius, 1);              // 37
echo Temperature::ABSOLUTE_ZERO_C;       // -273.15
echo Temperature::count();               // 1
echo Temperature::class;                 // "Temperature" (FQCN as a string)
```

---

## 8. OOP Part 2 — Interfaces, Traits, Enums, Inheritance, Magic Methods

### 8.1 Interfaces — programming to a contract **[I]**

An **interface** is a pure contract: a list of method signatures with no implementation. A class `implements` an interface and *promises* to provide those methods. The value is **decoupling**: your code depends on the interface (the *what*), not on a concrete class (the *how*), so you can swap implementations freely — a real `DatabaseLogger` in production, an in-memory `FakeLogger` in tests — without changing the consumer. This is the foundation of dependency injection and testability. A class may implement *many* interfaces.

```php
<?php
declare(strict_types=1);

interface Logger
{
    public function log(string $level, string $message): void;
}

final class FileLogger implements Logger
{
    public function __construct(private string $path) {}

    public function log(string $level, string $message): void
    {
        file_put_contents(
            $this->path,
            sprintf("[%s] %s: %s\n", date('c'), strtoupper($level), $message),
            FILE_APPEND
        );
    }
}

// The consumer depends on the INTERFACE, so any Logger works.
function processOrder(int $id, Logger $logger): void
{
    $logger->log('info', "Processing order $id");
}
processOrder(42, new FileLogger('/tmp/app.log'));
```

### 8.2 Abstract classes & inheritance **[I]**

An **abstract class** sits between an interface and a concrete class: it can provide shared implementation *and* declare `abstract` methods that subclasses must fill in. Use inheritance (`extends`) for genuine "is-a" relationships and shared behavior; **favor composition and interfaces over deep inheritance trees** (deep hierarchies are rigid and hard to follow). Mark classes `final` when they aren't designed to be extended — it's a clear signal and prevents surprise subclassing.

```php
<?php
declare(strict_types=1);

abstract class Shape
{
    abstract public function area(): float;   // subclasses MUST implement

    // Concrete shared method, available to all subclasses.
    public function describe(): string
    {
        return sprintf('%s with area %.2f', static::class, $this->area());
    }
}

final class Circle extends Shape
{
    public function __construct(private float $radius) {}
    public function area(): float { return M_PI * $this->radius ** 2; }
}

final class Rectangle extends Shape
{
    public function __construct(private float $w, private float $h) {}
    public function area(): float { return $this->w * $this->h; }
}

echo (new Circle(2))->describe();        // Circle with area 12.57
echo (new Rectangle(3, 4))->describe();  // Rectangle with area 12.00
```

### 8.3 Traits — horizontal reuse **[I]**

PHP has single inheritance, so a class can extend only one parent. A **trait** is a reusable bundle of methods (and properties) you can `use` in *any* class — horizontal code reuse without inheritance. Good for cross-cutting helpers (timestamps, an `id` field, a comparison helper) shared by unrelated classes. Don't overuse traits as a dumping ground; they hide where behavior comes from and can collide.

```php
<?php
declare(strict_types=1);

trait HasTimestamps
{
    public ?\DateTimeImmutable $createdAt = null;
    public ?\DateTimeImmutable $updatedAt = null;

    public function touch(): void
    {
        $now = new \DateTimeImmutable();
        $this->createdAt ??= $now;
        $this->updatedAt = $now;
    }
}

final class Article
{
    use HasTimestamps;   // mixes the trait's members into this class
    public function __construct(public string $title) {}
}

$a = new Article('Hello');
$a->touch();
echo $a->updatedAt->format('Y-m-d');
```

### 8.4 Enums — first-class enumerations **[I/A]**

**Enums** (PHP 8.1) give you a fixed, type-safe set of named values — replacing the old anti-pattern of loose string/int constants. A **pure** enum has cases with no scalar value; a **backed** enum assigns each case a `string` or `int` (ideal for storing in a database or serializing to JSON). Enums are objects: they can implement interfaces, have **methods**, and define constants. Because an enum *type* can only ever be one of its cases, using one as a parameter type makes whole categories of invalid input impossible.

```php
<?php
declare(strict_types=1);

// Backed enum: each case has a stable string value for DB/JSON storage.
enum Status: string
{
    case Draft     = 'draft';
    case Published = 'published';
    case Archived  = 'archived';

    // Enums can have methods — behavior lives with the data.
    public function label(): string
    {
        return match ($this) {
            Status::Draft     => 'Draft (not visible)',
            Status::Published => 'Live',
            Status::Archived  => 'Archived',
        };
    }

    // A common helper: cases that count as "public".
    public function isVisible(): bool
    {
        return $this === Status::Published;
    }
}

$s = Status::Published;
echo $s->value;                  // "published"  (the backing value)
echo $s->name;                   // "Published"  (the case name)
echo $s->label();                // "Live"

// from() / tryFrom() convert a scalar (e.g. from the DB) back to a case.
$s2 = Status::from('draft');        // Status::Draft (throws if invalid)
$s3 = Status::tryFrom('bogus');     // null instead of throwing
$all = Status::cases();             // [Draft, Published, Archived]

// As a parameter type, only a valid Status can ever be passed:
function publish(Status $status): void { /* ... */ }
```

### 8.5 Late static binding & magic methods **[A]**

**Late static binding** is the difference between `self::` and `static::`. `self::` is resolved at *compile time* to the class where it's written; `static::` is resolved at *runtime* to the actual called class — so a static factory in a base class can return an instance of the *subclass* that called it. **Magic methods** are special `__`-prefixed methods PHP calls automatically: `__construct`, `__toString` (string conversion), `__get`/`__set` (intercept access to inaccessible properties), `__call`/`__callStatic` (intercept missing method calls), `__invoke` (make an object callable). They are powerful but obscure control flow — use them deliberately (frameworks lean on them; application code usually shouldn't).

```php
<?php
declare(strict_types=1);

abstract class Model
{
    // static:: binds to the CALLED class at runtime (late static binding),
    // so Model::create() actually returns a User when called as User::create().
    public static function create(): static
    {
        return new static();
    }
}
final class User extends Model {}
var_dump(User::create() instanceof User);   // true

final class Temperature
{
    public function __construct(private float $c) {}

    // Called automatically when the object is used as a string.
    public function __toString(): string
    {
        return sprintf('%.1f°C', $this->c);
    }
}
echo new Temperature(21.567);   // 21.6°C
```

---

## 9. The Modern Type System — Unions, Hooks, Asymmetric Visibility, Generics

### 9.1 Union, intersection & nullable types **[I/A]**

PHP's type system has grown teeth. A **union type** (`int|string`) accepts any of several types. A **nullable** type (`?T`) is shorthand for `T|null`. An **intersection type** (`Countable&Traversable`) requires a value to satisfy *all* listed types simultaneously — useful for "must implement these two interfaces at once." Special return types: **`void`** (returns nothing), **`never`** (the function always throws or exits — it never returns control, which lets analyzers prune dead code), and **`mixed`** (any type — the escape hatch, use sparingly).

```php
<?php
declare(strict_types=1);

// Union: accept an int id OR a string slug.
function find(int|string $idOrSlug): ?string
{
    return is_int($idOrSlug) ? "by-id-$idOrSlug" : "by-slug-$idOrSlug";
}

// Intersection: the argument must be BOTH Countable AND Traversable.
function describe(\Countable&\Traversable $collection): string
{
    return 'has ' . count($collection) . ' items';
}

// never: this function never returns normally (it always throws).
function abort(string $why): never
{
    throw new \RuntimeException($why);
}
```

### 9.2 Property hooks (PHP 8.4) **[I/A]**

**Property hooks** are one of 8.4's headline features. They let a property have computed `get` and validated `set` logic *without* writing separate getter/setter methods and *without* a backing field — the property *is* the interface. This brings PHP in line with C#-style properties: callers write `$user->name` (a clean property access) while you keep full control over reading and writing. Use them for computed/derived values and for validation-on-assignment, replacing reams of `getX()`/`setX()` boilerplate.

```php
<?php
declare(strict_types=1);

final class User
{
    // A "virtual" property computed on read — no stored field for it.
    public string $fullName {
        get => trim("$this->first $this->last");
    }

    // A property with a validating set hook. $value is the incoming value;
    // the assigned result is stored in the (implicit) backing field.
    public string $email {
        set (string $value) {
            $value = strtolower(trim($value));
            if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
                throw new \InvalidArgumentException("Invalid email: $value");
            }
            $this->email = $value;   // write the validated value
        }
    }

    public function __construct(
        public string $first,
        public string $last,
        string $email,
    ) {
        $this->email = $email;       // runs the set hook
    }
}

$u = new User('Ada', 'Lovelace', '  ADA@EXAMPLE.COM ');
echo $u->fullName;   // "Ada Lovelace"  (computed via the get hook)
echo $u->email;      // "ada@example.com" (normalized by the set hook)
// $u->email = 'nope';   // throws InvalidArgumentException
```

### 9.3 Asymmetric visibility (PHP 8.4) **[I/A]**

**Asymmetric visibility** lets a property be **readable** from one scope but **writable** only from a narrower one. The syntax `public private(set)` means "anyone can read this; only this class may write it." This is the common case that `readonly` *almost* covers — you want public reads and protected writes, but you still need to mutate the value internally after construction (which `readonly` forbids). It removes a huge amount of getter boilerplate.

```php
<?php
declare(strict_types=1);

final class Counter
{
    // Read publicly, write only from inside this class.
    public private(set) int $value = 0;

    // protected(set): readable everywhere, writable by this class + subclasses.
    public protected(set) string $label = 'count';

    public function increment(): void
    {
        $this->value++;          // allowed — we're inside the class
    }
}

$c = new Counter();
$c->increment();
echo $c->value;       // 1  (public read — fine)
// $c->value = 99;    // Error: Cannot modify private(set) property from outside
```

### 9.4 Variance — covariance & contravariance **[A]**

When overriding methods, PHP enforces **variance** rules so substitutability (Liskov) holds. **Return types are covariant**: an overriding method may return a *more specific* type than the parent (return a `Cat` where the parent returned `Animal`). **Parameter types are contravariant**: an override may accept a *more general* type. In practice you mostly benefit from covariant returns — e.g. a base `repository()->find(): ?Entity` overridden to `: ?User`. Violating variance is a fatal error caught at class-load time.

### 9.5 Generics via docblocks + PHPStan **[I/A]**

PHP has **no native generics at runtime** (as of 8.4/8.5). You cannot write `class Collection<T>` and have the engine enforce `T`. Instead, the ecosystem expresses generics in **docblock annotations** that **PHPStan** and **Psalm** (§18) understand and check *statically*. This gives you most of the safety of generics — wrong-type insertions are flagged by the analyzer and your IDE — with zero runtime cost. This is the standard modern approach; learn it.

```php
<?php
declare(strict_types=1);

/**
 * A type-safe collection. The @template line declares the generic parameter T;
 * PHPStan enforces it across these methods even though PHP itself does not.
 *
 * @template T
 */
final class TypedCollection
{
    /** @var list<T> */
    private array $items = [];

    /** @param T $item */
    public function add(mixed $item): void
    {
        $this->items[] = $item;
    }

    /** @return T|null */
    public function first(): mixed
    {
        return $this->items[0] ?? null;
    }
}

/** @var TypedCollection<\DateTimeImmutable> $dates */
$dates = new TypedCollection();
$dates->add(new \DateTimeImmutable());
// $dates->add('not a date');  // PHPStan flags this; runtime would not
```

> **⚡ Version note:** native generics remain an open RFC topic and are **not** in 8.4 or 8.5. Until they land, docblock generics + PHPStan are the answer; don't expect `<T>` syntax to work at runtime.

---

## 10. Namespaces & Autoloading — PSR-4 & Composer

### 10.1 Namespaces **[I]**

A **namespace** is a prefix that prevents class-name collisions across packages — without it, two libraries each defining a `Logger` class couldn't coexist. Namespaces are declared with `namespace App\Service;` at the top of a file and mirror your directory structure by convention. The leading `\` denotes the global namespace (so `\strlen`, `\DateTimeImmutable` refer to the built-ins even inside a namespace). `use` imports a name so you can refer to it short.

```php
<?php
declare(strict_types=1);

namespace App\Service;          // this file's classes live under App\Service

use App\Model\User;             // import a class so we can write "User"
use App\Repository\UserRepository as Repo;  // import with an alias
use function App\Support\slugify;            // import a function
use const App\Config\VERSION;                // import a constant

final class Registration
{
    public function __construct(private Repo $repo) {}

    public function register(string $name): User
    {
        // \DateTimeImmutable: leading backslash = the GLOBAL namespace.
        return new User($name, new \DateTimeImmutable());
    }
}
```

### 10.2 PSR-4 autoloading via Composer **[I]**

Manually `require`-ing files is unmaintainable. **Autoloading** loads a class file automatically the first time the class is used. The standard is **PSR-4**, which maps a namespace prefix to a directory: `App\` → `src/`, so the class `App\Service\Registration` *must* live in `src/Service/Registration.php`. Composer generates the autoloader; you include it once and everything else loads on demand.

```json
{
    "name": "acme/app",
    "require": { "php": ">=8.4" },
    "autoload": {
        "psr-4": { "App\\": "src/" }
    },
    "autoload-dev": {
        "psr-4": { "Tests\\": "tests/" }
    }
}
```

```bash
composer dump-autoload   # (re)generate vendor/autoload.php after changing the map
```

```php
<?php
declare(strict_types=1);

// The ONE require you need — Composer's autoloader. After this, every
// class under App\ (and every Composer package) loads automatically.
require __DIR__ . '/vendor/autoload.php';

use App\Service\Registration;

$reg = new Registration(/* ... */);   // src/Service/Registration.php loaded on demand
```

For production, run `composer dump-autoload -o` (or `composer install --no-dev --optimize-autoloader`) to build a static classmap, which is faster than filesystem lookups (§19).

---

## 11. Errors & Exceptions

### 11.1 The model: errors vs exceptions **[I]**

Modern PHP unifies failures under a class hierarchy rooted at **`\Throwable`**, which has two branches:

- **`\Error`** — engine-level problems: `TypeError`, `ValueError`, `DivisionByZeroError`, `ParseError`. These signal *bugs in your code* (wrong type passed, calling a method on `null`). You generally let them surface in dev and convert them to a generic 500 in production — you don't routinely catch them.
- **`\Exception`** — application-level conditions you anticipate and handle: a file not found, a validation failure, an HTTP error. You `throw` and `catch` these.

```text
Throwable (interface)
├── Error
│   ├── TypeError
│   ├── ValueError
│   ├── ArithmeticError → DivisionByZeroError
│   └── ParseError
└── Exception
    ├── LogicException        → InvalidArgumentException, OutOfRangeException, ...
    └── RuntimeException      → RuntimeException subclasses, your own app exceptions
```

The old C-style "warnings/notices" still exist for some legacy functions, but in modern code you escalate them to exceptions (e.g. PDO's `ERRMODE_EXCEPTION`, §14) so failures are never silently ignored.

### 11.2 try / catch / finally & custom exceptions **[I]**

`try` runs code that may fail; `catch` handles specific exception types (you can catch a union: `catch (TypeError | ValueError $e)`); `finally` *always* runs (success or failure) and is the place to release resources. Define **custom exception classes** to model your domain's failures meaningfully — a `PaymentDeclinedException` is far more useful to catch than a generic `Exception`. Always preserve the original error as the **`$previous`** argument when re-throwing, so the stack trace chain isn't lost.

```php
<?php
declare(strict_types=1);

// A domain-specific exception. Extending RuntimeException signals "runtime
// condition", and lets callers catch THIS precisely.
final class InsufficientFundsException extends \RuntimeException {}

function withdraw(int $balance, int $amount): int
{
    if ($amount > $balance) {
        throw new InsufficientFundsException(
            "Cannot withdraw $amount from $balance"
        );
    }
    return $balance - $amount;
}

$handle = null;
try {
    $handle = fopen('php://temp', 'r+');
    $newBalance = withdraw(100, 250);     // throws
} catch (InsufficientFundsException $e) {
    // Handle the EXPECTED business condition specifically.
    error_log('Declined: ' . $e->getMessage());
} catch (\Throwable $e) {
    // Catch-all safety net; re-wrap to add context, preserving the cause.
    throw new \RuntimeException('Withdrawal failed', previous: $e);
} finally {
    // ALWAYS runs — clean up regardless of success/failure.
    if (is_resource($handle)) {
        fclose($handle);
    }
}
```

> **⚡ Version note (PHP 8.5):** the **`#[\NoDiscard]`** attribute marks a function/method whose return value must not be ignored (e.g. a `Result` object you must inspect) — the engine warns if the caller discards it. Pair it with explicit `(void)` casts to acknowledge intentional discards. Confirm exact semantics against the 8.5 docs.

---

## 12. File System, OS Info & Command Execution

This section mirrors the dedicated FS/OS section every language guide in this library has. PHP grew up as a web language, but its CLI file and process APIs are complete and frequently used in scripts, build tools, and deploy automation.

### 12.1 Reading & writing files **[I]**

PHP offers three tiers. For **whole-file** work, `file_get_contents`/`file_put_contents` are one-liners. For **streaming** large files (so you don't load gigabytes into memory), use `fopen`/`fgets`/`fread`/`fclose`. The object-oriented **`SplFileObject`** wraps a file as an iterable. Always check for failure (these return `false` on error) and prefer the `LOCK_EX` flag when concurrent writers are possible.

```php
<?php
declare(strict_types=1);

// --- Whole-file (simple, for reasonably-sized files) ---
$contents = file_get_contents('data.txt');          // entire file as a string, or false
$lines    = file('data.txt', FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES); // array of lines

file_put_contents('out.txt', "hello\n");                       // overwrite
file_put_contents('log.txt', "entry\n", FILE_APPEND | LOCK_EX); // append, exclusively locked

// --- Streaming (memory-safe for huge files) ---
$fh = fopen('big.csv', 'rb');                        // 'rb' = read binary
if ($fh === false) {
    throw new \RuntimeException('Cannot open big.csv');
}
try {
    while (($line = fgets($fh)) !== false) {         // one line at a time
        // process $line without holding the whole file in memory
    }
} finally {
    fclose($fh);
}

// --- SplFileObject: iterate a CSV row-by-row, object style ---
$csv = new \SplFileObject('people.csv');
$csv->setFlags(\SplFileObject::READ_CSV | \SplFileObject::SKIP_EMPTY);
foreach ($csv as $row) {
    // $row is an array of fields, parsed as CSV
}
```

### 12.2 Paths, directories & globbing **[I]**

```php
<?php
declare(strict_types=1);

// Path components — prefer the cross-platform DIRECTORY_SEPARATOR / these helpers.
echo dirname('/var/www/app/index.php');    // /var/www/app
echo basename('/var/www/app/index.php');   // index.php
echo pathinfo('archive.tar.gz', PATHINFO_EXTENSION); // gz
echo realpath('../config');                // resolve to absolute, canonical path

// Existence & metadata.
var_dump(file_exists('data.txt'), is_dir('cache'), is_writable('logs'));
echo filesize('data.txt'), ' ', date('c', filemtime('data.txt'));

// Create / remove directories.
mkdir('storage/cache', 0775, recursive: true);   // recursive: make parents too
rename('old.txt', 'new.txt');
unlink('temp.txt');                              // delete a file
// rmdir('emptydir');                            // only removes EMPTY dirs

// glob(): shell-style pattern matching → array of paths.
foreach (glob('logs/*.log') as $log) {
    echo $log, "\n";
}

// Recursive traversal with the SPL iterators (no manual recursion):
$it = new \RecursiveIteratorIterator(
    new \RecursiveDirectoryIterator('src', \FilesystemIterator::SKIP_DOTS)
);
foreach ($it as $file) {
    if ($file->getExtension() === 'php') {
        echo $file->getPathname(), "\n";
    }
}
```

### 12.3 Running external commands — safely **[I]**

PHP can shell out, and this is **the single most dangerous area for command injection**. The cardinal rule: **never** interpolate user input into a command string. Always pass arguments through **`escapeshellarg()`** (quotes/escapes a single argument) or, better, use **`proc_open`** with explicit pipes so you control stdin/stdout/stderr. `exec` returns the last line (and fills an array of all lines), `shell_exec` returns the whole output, and `proc_open` gives full bidirectional control.

```php
<?php
declare(strict_types=1);

// exec(): get output lines + exit code. ESCAPE every interpolated argument.
$userInput = 'my file.txt';
$cmd = 'wc -l ' . escapeshellarg($userInput);   // escapeshellarg => safe single arg
exec($cmd, $output, $exitCode);
// $output is array of lines; $exitCode is the process exit status (0 = success)

if ($exitCode !== 0) {
    throw new \RuntimeException("Command failed (exit $exitCode)");
}

// shell_exec(): whole output as one string (null on failure / no output).
$who = shell_exec('whoami');

// proc_open(): full control — feed stdin, capture stdout/stderr separately.
$descriptors = [
    0 => ['pipe', 'r'],   // child's STDIN
    1 => ['pipe', 'w'],   // child's STDOUT
    2 => ['pipe', 'w'],   // child's STDERR
];
$proc = proc_open(['grep', 'error'], $descriptors, $pipes);  // ARRAY form: no shell, no injection
if (is_resource($proc)) {
    fwrite($pipes[0], "all good\nan error here\nfine\n");
    fclose($pipes[0]);
    $matched = stream_get_contents($pipes[1]);   // "an error here\n"
    fclose($pipes[1]);
    fclose($pipes[2]);
    $code = proc_close($proc);
}
```

> **Security note:** the safest form is `proc_open(['program', 'arg1', 'arg2'], ...)` with an **array** command — PHP executes the program directly with no shell involved, so there is *nothing to inject into*. Prefer it whenever you don't need shell features (pipes, globbing). For [Symfony/Laravel](LARAVEL_GUIDE.md) apps, the `symfony/process` component wraps this safely.

### 12.4 Environment variables & system info **[I]**

Configuration and secrets belong in **environment variables**, never in code (§20). Read them with `getenv()` or the `$_ENV` superglobal. `php_uname()` and the `PHP_OS_FAMILY` constant give OS details for platform-specific behavior.

```php
<?php
declare(strict_types=1);

// Env vars: the standard place for config/secrets. Provide a default.
$dbHost = getenv('DB_HOST') ?: '127.0.0.1';
$apiKey = $_ENV['API_KEY'] ?? throw new \RuntimeException('API_KEY not set');

// System & runtime info.
echo PHP_OS_FAMILY;          // "Linux" | "Windows" | "Darwin" | "BSD" — branch on this
echo php_uname();            // full OS string
echo PHP_VERSION;            // "8.4.x"
echo gethostname();          // machine hostname
echo PHP_INT_SIZE * 8;       // 64  (bit-width of int)
echo getmypid();             // current process id

// CLI script arguments (when run as `php script.php a b c`):
//   $argv = ['script.php', 'a', 'b', 'c'];  $argc = 4;
// Use getopt() for real flag parsing:
$opts = getopt('v', ['name:', 'verbose']);   // -v, --name=X, --verbose
```

### 12.5 Streams **[I]**

PHP's **stream wrappers** unify files, network sockets, compression, and in-memory buffers behind one API. `php://stdin`, `php://stdout`, `php://stderr` are the CLI standard streams; `php://memory` and `php://temp` are scratch buffers; `php://input` is the raw request body (vital for JSON APIs, §15). `http://`/`https://` URLs can even be `fopen`-ed (with `allow_url_fopen`), though for real HTTP you'll use cURL or Guzzle.

```php
<?php
declare(strict_types=1);

// Write to STDERR (so it doesn't pollute STDOUT in a pipeline):
fwrite(STDERR, "warning: low disk\n");

// In-memory stream as a scratch file:
$mem = fopen('php://temp', 'r+');
fwrite($mem, "buffered data");
rewind($mem);
echo stream_get_contents($mem);

// Read the raw HTTP request body (e.g. a JSON POST):
$rawBody = file_get_contents('php://input');
```

---

## 13. Working with Data — JSON, Dates, Regex, Strings

### 13.1 JSON **[I]**

JSON is the lingua franca of web APIs. `json_encode` turns PHP values into JSON; `json_decode` parses JSON back. The two **must-know flags**: pass **`JSON_THROW_ON_ERROR`** so malformed JSON throws a `\JsonException` instead of silently returning `null` (always use it), and pass **`true`** (or `JSON_OBJECT_AS_ARRAY`) as the second `json_decode` argument to get associative arrays instead of `stdClass` objects. Other handy encode flags: `JSON_PRETTY_PRINT`, `JSON_UNESCAPED_SLASHES`, `JSON_UNESCAPED_UNICODE`.

```php
<?php
declare(strict_types=1);

$data = ['name' => 'Ada', 'tags' => ['php', 'web'], 'active' => true];

// Encode. JSON_THROW_ON_ERROR makes failures loud.
$json = json_encode($data, JSON_THROW_ON_ERROR | JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);

// Decode to an associative array (the `true` argument) — and throw on bad input.
try {
    $back = json_decode($json, associative: true, flags: JSON_THROW_ON_ERROR);
    echo $back['name'];      // Ada
} catch (\JsonException $e) {
    error_log('Bad JSON: ' . $e->getMessage());
}
```

### 13.2 Dates & times — use `DateTimeImmutable` **[I]**

**Always use `DateTimeImmutable`, never the mutable `DateTime`.** With the mutable class, calling `$d->modify('+1 day')` changes `$d` *in place*, so any other code holding a reference to that object is silently affected — a classic source of bugs. `DateTimeImmutable`'s methods *return a new object* and leave the original untouched, exactly like a value object should. Store and compute in **UTC**; convert to the user's timezone only for display. `DateInterval` represents durations; `DateTimeImmutable::diff()` gives the difference between two instants.

```php
<?php
declare(strict_types=1);

// Construct: now, from a string, or from a specific format.
$now   = new \DateTimeImmutable('now', new \DateTimeZone('UTC'));
$xmas  = new \DateTimeImmutable('2026-12-25 09:00:00', new \DateTimeZone('UTC'));
$parsed = \DateTimeImmutable::createFromFormat('Y-m-d', '2026-06-25');

// Arithmetic RETURNS A NEW object (the original is untouched — that's the point).
$tomorrow = $now->modify('+1 day');
$inTwoWeeks = $now->add(new \DateInterval('P2W'));   // ISO 8601 duration: P2W = 2 weeks

// Difference between two instants.
$diff = $now->diff($xmas);
echo $diff->days, ' days until Christmas';

// Format for output (note: format codes, not strftime). 'c' = ISO 8601, 'U' = unix ts.
echo $now->format('Y-m-d H:i:s');     // 2026-06-25 14:30:00
echo $now->format(\DateTimeInterface::ATOM);   // 2026-06-25T14:30:00+00:00

// Display in a user's timezone WITHOUT changing the underlying instant:
$lagos = $now->setTimezone(new \DateTimeZone('Africa/Lagos'));
```

### 13.3 Regular expressions (PCRE) **[I]**

PHP's regex engine is **PCRE** (Perl-Compatible). Patterns are strings wrapped in delimiters (commonly `/.../` or `~...~`) with optional trailing flags (`i` case-insensitive, `m` multiline, `s` dot-matches-newline, `u` UTF-8). The core functions: `preg_match` (first match, returns 0/1/false), `preg_match_all` (all matches), `preg_replace` (substitute), `preg_replace_callback` (substitute via a function), `preg_split`. **Always pass the `u` flag for UTF-8 text**, and check for `false` return (a malformed pattern or backtrack-limit hit).

```php
<?php
declare(strict_types=1);

$text = 'Contact: ada@example.com or grace@dev.io';

// preg_match with a named capture group; result lands in $m.
if (preg_match('/(?<user>[\w.]+)@(?<domain>[\w.]+)/', $text, $m)) {
    echo $m['user'], ' @ ', $m['domain'];   // ada @ example.com (first match)
}

// All matches:
preg_match_all('/[\w.]+@[\w.]+/', $text, $all);
// $all[0] => ['ada@example.com', 'grace@dev.io']

// Replace with a callback (e.g. mask emails):
$masked = preg_replace_callback(
    '/([\w.]+)@([\w.]+)/',
    fn($m) => '***@' . $m[2],
    $text
);

// Split on one-or-more whitespace:
$words = preg_split('/\s+/', 'one  two   three');   // ['one','two','three']
```

> For *validation* of well-known formats (email, URL, IP), prefer **`filter_var($x, FILTER_VALIDATE_EMAIL)`** over a hand-rolled regex — it's correct and battle-tested (§15).

### 13.4 String functions you'll use constantly **[I]**

Remember PHP strings are **bytes**: for ASCII the plain `str*` functions are fine, but for UTF-8 human text use the **`mb_*`** variants so multi-byte characters count and case-fold correctly.

| Need | Function |
|---|---|
| Length | `strlen` (bytes) / `mb_strlen` (chars) |
| Substring | `substr` / `mb_substr` |
| Find position | `strpos` / `mb_strpos` (returns `false` if absent — use `!==`) |
| Contains / starts / ends | `str_contains`, `str_starts_with`, `str_ends_with` (8.0+) |
| Replace | `str_replace`, `str_ireplace` |
| Case | `strtolower`/`strtoupper` (ASCII) vs `mb_strtolower`/`mb_strtoupper` |
| Trim | `trim`, `ltrim`, `rtrim` |
| Split / join | `explode`, `implode`, `str_split` |
| Format | `sprintf`, `number_format`, `str_pad`, `wordwrap` |
| Template | `str_replace` / `strtr` / `sprintf` named args via `vsprintf` |

```php
<?php
declare(strict_types=1);

$s = '  Hello, World  ';
echo trim($s);                              // "Hello, World"
var_dump(str_contains($s, 'World'));        // true  (no more strpos !== false dance)
var_dump(str_starts_with(trim($s), 'Hello')); // true
echo sprintf('%05.2f', 3.1);                // "03.10"
echo number_format(1234567.891, 2);         // "1,234,567.89"
echo mb_strtoupper('élan');                 // "ÉLAN"  (mb_ handles the accent)
```

---

## 14. Databases with PDO — Prepared Statements & Transactions

### 14.1 Why PDO, and connecting **[I/A]**

**PDO** (PHP Data Objects) is PHP's unified, object-oriented database API. The same code works against [PostgreSQL](POSTGRESQL_GUIDE.md), MySQL/MariaDB, [SQLite](SQLITE3_GUIDE.md), and more — you change only the **DSN** (connection string). Prefer PDO over the older `mysqli` because it's database-agnostic and has clean prepared-statement support. Two configuration choices you should *always* make on connect: set the **error mode to exceptions** (so failures throw instead of being ignored) and turn **emulated prepares off** for real server-side prepared statements.

```php
<?php
declare(strict_types=1);

// DSN encodes driver + host + db + charset. Swap the driver to change databases.
$dsn = 'pgsql:host=127.0.0.1;port=5432;dbname=app';      // PostgreSQL
// $dsn = 'mysql:host=127.0.0.1;dbname=app;charset=utf8mb4'; // MySQL/MariaDB
// $dsn = 'sqlite:' . __DIR__ . '/app.db';                   // SQLite

$pdo = new \PDO($dsn, getenv('DB_USER'), getenv('DB_PASS'), [
    // Throw exceptions on error — the single most important PDO setting.
    \PDO::ATTR_ERRMODE            => \PDO::ERRMODE_EXCEPTION,
    // Fetch rows as associative arrays by default.
    \PDO::ATTR_DEFAULT_FETCH_MODE => \PDO::FETCH_ASSOC,
    // Use REAL server-side prepared statements (true SQLi protection).
    \PDO::ATTR_EMULATE_PREPARES   => false,
]);
```

### 14.2 Prepared statements — the SQL-injection defense **[I/A]**

This is the most important section in the database chapter. **SQL injection** happens when untrusted data is concatenated into a query string, letting an attacker change the query's meaning (`'; DROP TABLE users; --`). **Prepared statements** eliminate it structurally: you send the SQL with **placeholders** first, the database compiles it, *then* you send the data separately — the data can never be interpreted as SQL. **Never** build a query by string concatenation/interpolation with user data. There is no excuse, ever.

```php
<?php
declare(strict_types=1);

// NEVER do this — classic SQL injection:
//   $pdo->query("SELECT * FROM users WHERE email = '$email'");  // CATASTROPHIC

// DO this — named placeholders, data bound separately:
$stmt = $pdo->prepare('SELECT id, name FROM users WHERE email = :email AND active = :active');
$stmt->execute(['email' => $email, 'active' => true]);   // data sent apart from SQL
$user = $stmt->fetch();                                  // one row (assoc array) or false

// Positional placeholders (?) work too:
$stmt = $pdo->prepare('SELECT id, name FROM users WHERE age > ? ORDER BY name LIMIT ?');
$stmt->execute([18, 10]);
$rows = $stmt->fetchAll();                               // all rows

// INSERT and get the new id:
$stmt = $pdo->prepare('INSERT INTO users (name, email) VALUES (:name, :email)');
$stmt->execute(['name' => 'Ada', 'email' => 'ada@example.com']);
$newId = $pdo->lastInsertId();

// Fetch into typed objects:
$stmt = $pdo->query('SELECT id, name FROM users');
$users = $stmt->fetchAll(\PDO::FETCH_CLASS, User::class);
```

> **Note:** placeholders bind **values**, not identifiers. You cannot parameterize a table or column *name* — if you must build those dynamically, validate them against an explicit allow-list of known-good names, never against raw user input.

### 14.3 Transactions **[I/A]**

A **transaction** groups multiple statements so they either *all* succeed or *all* roll back — the atomicity you need for operations like "debit one account, credit another." Wrap the work in `beginTransaction()` / `commit()`, and in a `catch` block call `rollBack()` so a mid-operation failure leaves the database consistent. See [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md) for the deeper ACID model and [Database Server Administration](DATABASE_SERVER_ADMIN_GUIDE.md) for isolation-level tuning.

```php
<?php
declare(strict_types=1);

function transfer(\PDO $pdo, int $from, int $to, int $cents): void
{
    $pdo->beginTransaction();
    try {
        $debit = $pdo->prepare('UPDATE accounts SET cents = cents - :c WHERE id = :id');
        $debit->execute(['c' => $cents, 'id' => $from]);

        $credit = $pdo->prepare('UPDATE accounts SET cents = cents + :c WHERE id = :id');
        $credit->execute(['c' => $cents, 'id' => $to]);

        $pdo->commit();           // both succeeded — make it permanent
    } catch (\Throwable $e) {
        $pdo->rollBack();         // anything failed — undo everything
        throw $e;                 // re-throw so the caller knows
    }
}
```

A note on **MySQLi**: it's the MySQL-only alternative to PDO, with a similar prepared-statement API (`mysqli_stmt::bind_param`). It's fine, but PDO's database-agnosticism and cleaner binding make it the default recommendation. Whichever you use: **prepared statements, always.** For ORMs, see [Laravel](LARAVEL_GUIDE.md)'s Eloquent.

---

## 15. PHP for the Web — Requests, Sessions, Forms, Security

### 15.1 The request/response model & superglobals **[I/A]**

When a request hits PHP-FPM, PHP populates **superglobals** — arrays available everywhere — with the request data, runs your script, and sends whatever you `echo` (plus headers) as the response. The key superglobals:

| Superglobal | Contents |
|---|---|
| `$_GET` | Query-string parameters (`?page=2`) |
| `$_POST` | Form-encoded request body fields |
| `$_REQUEST` | `$_GET` + `$_POST` + `$_COOKIE` merged — **avoid** (ambiguous source) |
| `$_SERVER` | Request metadata: method, URI, headers, client IP |
| `$_COOKIE` | Cookies sent by the browser |
| `$_SESSION` | Server-side per-user session store (after `session_start()`) |
| `$_FILES` | Uploaded files |
| `$_ENV` | Environment variables |

**Treat every superglobal except `$_SESSION`/`$_ENV` as untrusted user input.** Validate and sanitize before use.

```php
<?php
declare(strict_types=1);

$method = $_SERVER['REQUEST_METHOD'];        // 'GET' | 'POST' | ...
$page   = (int) ($_GET['page'] ?? 1);        // cast — never trust the type

// Sending a response: set status + headers BEFORE any output, then echo the body.
http_response_code(200);
header('Content-Type: application/json; charset=utf-8');
echo json_encode(['page' => $page], JSON_THROW_ON_ERROR);
```

### 15.2 Sessions & cookies **[I]**

A **session** is per-user server-side storage keyed by a cookie. `session_start()` reads (or issues) the session-id cookie and loads `$_SESSION`. Use sessions for login state and CSRF tokens. **Security hardening is mandatory:** configure the cookie as `HttpOnly` (JS can't read it → mitigates XSS-based theft), `Secure` (HTTPS only), and `SameSite=Lax/Strict` (anti-CSRF), and **regenerate the session id on privilege changes** (login) to prevent session fixation.

```php
<?php
declare(strict_types=1);

session_start([
    'cookie_httponly' => true,
    'cookie_secure'   => true,      // requires HTTPS
    'cookie_samesite' => 'Lax',
    'use_strict_mode' => true,      // reject attacker-supplied session ids
]);

// On successful login: regenerate the id to block session fixation.
function login(int $userId): void
{
    session_regenerate_id(delete_old_session: true);
    $_SESSION['user_id'] = $userId;
}

$loggedIn = isset($_SESSION['user_id']);

// A plain cookie (separate from the session): always set sane flags.
setcookie('theme', 'dark', [
    'expires'  => time() + 86400 * 30,
    'httponly' => true,
    'secure'   => true,
    'samesite' => 'Lax',
]);
```

### 15.3 Handling forms — validation & the big four web threats **[I/A]**

Web security is mostly four defenses, and you apply them at specific points:

1. **SQL injection** → prepared statements (§14). At the database boundary.
2. **XSS (Cross-Site Scripting)** → escape *output* with `htmlspecialchars()` (or `htmlentities`) every time you put data into HTML. At the view boundary.
3. **CSRF (Cross-Site Request Forgery)** → a per-session token embedded in every state-changing form and verified on submit, plus `SameSite` cookies.
4. **Password handling** → never store plaintext; hash with `password_hash()` (Argon2/bcrypt) and verify with `password_verify()`.

```php
<?php
declare(strict_types=1);
session_start();

// --- CSRF: issue a token, embed it in the form, verify on POST ---
$_SESSION['csrf'] ??= bin2hex(random_bytes(32));   // CSPRNG token

function verifyCsrf(string $sent): void
{
    // hash_equals: constant-time compare (no timing side-channel).
    if (!hash_equals($_SESSION['csrf'] ?? '', $sent)) {
        http_response_code(419);
        exit('CSRF validation failed');
    }
}

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    verifyCsrf($_POST['csrf'] ?? '');

    // --- Validate & sanitize input ---
    $email = filter_var($_POST['email'] ?? '', FILTER_VALIDATE_EMAIL);
    if ($email === false) {
        exit('Invalid email');
    }
    $name = trim((string) ($_POST['name'] ?? ''));
    if ($name === '' || mb_strlen($name) > 100) {
        exit('Invalid name');
    }
    // ... store with a prepared statement (§14) ...
}
?>
<form method="post">
    <!-- Embed the CSRF token; escape it for HTML output. -->
    <input type="hidden" name="csrf" value="<?= htmlspecialchars($_SESSION['csrf'], ENT_QUOTES) ?>">
    <input type="email" name="email" required>
    <input type="text" name="name" required>
    <button>Save</button>
</form>
```

### 15.4 Password hashing — `password_hash` / `password_verify` **[I/A]**

**Never** store passwords as plaintext, MD5, or SHA-1. Use **`password_hash()`**, which by default uses bcrypt and supports **Argon2id** (`PASSWORD_ARGON2ID`) — slow, salted, memory-hard hashes designed precisely to resist password cracking. Verify with **`password_verify()`** (constant-time), and use **`password_needs_rehash()`** to transparently upgrade old hashes when you raise the cost. See the [Go JWT/Argon2 guide](GO_JWT_ARGON2_GUIDE.md) for the same concepts in Go.

```php
<?php
declare(strict_types=1);

// Hashing on signup. Argon2id is the current recommendation where available.
$hash = password_hash($plainPassword, PASSWORD_ARGON2ID, [
    'memory_cost' => 65536,   // 64 MB
    'time_cost'   => 4,
    'threads'     => 2,
]);
// Store $hash (it includes the algorithm, cost, and salt — no separate salt column).

// Verifying on login:
if (password_verify($submittedPassword, $hash)) {
    // Opportunistically upgrade the hash if parameters have since strengthened.
    if (password_needs_rehash($hash, PASSWORD_ARGON2ID)) {
        $hash = password_hash($submittedPassword, PASSWORD_ARGON2ID);
        // ... persist the new $hash ...
    }
    // ... log the user in ...
}
```

### 15.5 File uploads **[I]**

Uploaded files land in `$_FILES` and a temp path. They are **dangerous**: never trust the client-supplied name or MIME type. Verify with **`is_uploaded_file()`**, check the size, determine the real type yourself (`finfo`), generate your own filename, store **outside the web root** (or with execution disabled), and move it with **`move_uploaded_file()`**.

```php
<?php
declare(strict_types=1);

$file = $_FILES['avatar'] ?? null;
if ($file && $file['error'] === UPLOAD_ERR_OK && is_uploaded_file($file['tmp_name'])) {
    if ($file['size'] > 2 * 1024 * 1024) {        // 2 MB cap
        exit('File too large');
    }
    // Detect the REAL mime type from content — never trust $file['type'].
    $mime = (new \finfo(FILEINFO_MIME_TYPE))->file($file['tmp_name']);
    $allowed = ['image/jpeg' => 'jpg', 'image/png' => 'png', 'image/webp' => 'webp'];
    if (!isset($allowed[$mime])) {
        exit('Unsupported file type');
    }
    // Generate our own safe filename; store OUTSIDE the web root.
    $name = bin2hex(random_bytes(16)) . '.' . $allowed[$mime];
    move_uploaded_file($file['tmp_name'], "/var/app/uploads/$name");
}
```

### 15.6 PSR-7 / PSR-15 — the modern HTTP abstraction **[I/A]**

Working with raw superglobals doesn't scale. The PHP-FIG standards **PSR-7** (HTTP message interfaces: `ServerRequestInterface`, `ResponseInterface` — immutable value objects) and **PSR-15** (middleware: `process(request, handler): response`) define a framework-agnostic way to model HTTP. Frameworks (Laravel, Symfony, Slim) and middleware libraries interoperate through them. You won't implement PSR-7 by hand — you'll consume it via a framework — but recognizing immutable request/response objects and the middleware "onion" is essential modern PHP literacy. See [Networking](NETWORKING_GUIDE.md) for the HTTP protocol underneath and [Nginx](NGINX_GUIDE.md) for the reverse proxy in front.

---

## 16. The Ecosystem — Composer, PSRs, SemVer, Frameworks

### 16.1 Composer in depth **[I]**

Composer is more than `require`. Key concepts: **`composer.json`** declares your *intent* (acceptable version ranges); **`composer.lock`** records the *exact* resolved versions — commit it, and in CI/production run **`composer install`** (which obeys the lock) rather than `composer update` (which re-resolves). **Scripts** let you alias common tasks. **Platform requirements** (`config.platform`) pin the PHP version Composer resolves against.

```json
{
    "require": {
        "php": ">=8.4",
        "monolog/monolog": "^3.6",
        "ramsey/uuid": "^4.7"
    },
    "require-dev": {
        "phpunit/phpunit": "^11.0",
        "phpstan/phpstan": "^2.0",
        "friendsofphp/php-cs-fixer": "^3.64"
    },
    "autoload": { "psr-4": { "App\\": "src/" } },
    "scripts": {
        "test":    "phpunit",
        "analyse": "phpstan analyse src --level=max",
        "fix":     "php-cs-fixer fix",
        "check":   ["@analyse", "@test"]
    }
}
```

```bash
composer install            # reproduce the locked dependency set (CI/prod)
composer update monolog/monolog   # update ONE package within its allowed range
composer require ramsey/uuid:^4.7 # add a package with a constraint
composer run check          # run the custom "check" script (analyse + test)
composer audit              # report known security advisories in your dependencies
composer outdated --direct  # which of your direct deps have newer versions
```

### 16.2 Semantic versioning & constraints **[I]**

Composer relies on **SemVer**: `MAJOR.MINOR.PATCH`, where MAJOR = breaking changes, MINOR = backward-compatible features, PATCH = backward-compatible fixes. The constraint operators decide how much Composer may upgrade:

| Constraint | Allows | Meaning |
|---|---|---|
| `^3.6` | `>=3.6.0 <4.0.0` | Caret: up to (not incl.) next major. **The default — use this.** |
| `~3.6` | `>=3.6.0 <3.7.0` | Tilde: up to next minor (more conservative) |
| `3.6.*` | `>=3.6.0 <3.7.0` | Wildcard patch |
| `>=3.6 <4.0` | explicit range | Manual bounds |
| `3.6.2` | exactly that | Pin (rarely; loses fixes) |

### 16.3 PSR standards & PHP-FIG **[I]**

**PHP-FIG** (the Framework Interop Group) publishes **PSRs** — recommendations that let independent libraries work together. The ones you'll meet:

| PSR | Topic |
|---|---|
| **PSR-1 / PSR-12** | Coding style (PSR-12 supersedes PSR-2; the de-facto formatting standard) |
| **PSR-4** | Autoloading (namespace→directory mapping) |
| **PSR-3** | Logger interface (Monolog implements it) |
| **PSR-7 / 15 / 17 / 18** | HTTP messages / middleware / message factories / HTTP client |
| **PSR-11** | Container (dependency injection) interface |
| **PSR-6 / 16** | Caching (pool / simple) |

Coding to PSR interfaces (a PSR-3 `LoggerInterface`, a PSR-11 container) keeps your code swappable and framework-neutral.

### 16.4 Frameworks overview **[I]**

You rarely build a serious app on raw PHP; you build on a framework that supplies routing, DI, ORM, templating, validation, and security defaults:

- **[Laravel](LARAVEL_GUIDE.md)** — the most popular full-stack PHP framework: expressive, batteries-included (Eloquent ORM, Blade templates, queues, auth, Artisan CLI). Best default for most new apps; covered in its own guide.
- **Symfony** — a robust, component-based framework favored for large/enterprise apps; many of its components (Console, HttpFoundation, Process) are used standalone and *under* Laravel.
- **Slim / Mezzio** — micro-frameworks built around PSR-7/15 middleware, for lean APIs.

Whatever you choose, the language fundamentals in this guide are what you actually program in — the framework just organizes them.

---

## 17. Advanced Features — Attributes, Generators, SPL, Fibers, FFI

### 17.1 Attributes — structured metadata **[A]**

**Attributes** (PHP 8.0) attach structured, machine-readable metadata to classes, methods, properties, and parameters using `#[...]` syntax. They replace the old practice of parsing magic docblock comments. Frameworks read them via **Reflection** to wire routes, validation rules, ORM mappings, and DI — e.g. Symfony's `#[Route]`, ORM `#[Column]`. You can define your own. Attributes do nothing on their own; some code must reflect over them.

```php
<?php
declare(strict_types=1);

// Declare an attribute by marking a class with #[Attribute].
#[\Attribute(\Attribute::TARGET_METHOD)]
final class Route
{
    public function __construct(
        public string $path,
        public string $method = 'GET',
    ) {}
}

final class UserController
{
    #[Route('/users', method: 'GET')]      // metadata attached to this method
    public function index(): array { return []; }

    #[Route('/users/{id}', method: 'GET')]
    public function show(int $id): array { return ['id' => $id]; }
}

// A router reads them via Reflection at boot:
$ref = new \ReflectionMethod(UserController::class, 'index');
foreach ($ref->getAttributes(Route::class) as $attr) {
    $route = $attr->newInstance();         // instantiate the Route object
    echo "$route->method $route->path";    // GET /users
}
```

> **⚡ Version note (PHP 8.4/8.5):** the built-in **`#[\Deprecated]`** attribute (8.4) marks functions/methods/constants as deprecated so the engine emits a deprecation notice when they're used. The **`#[\NoDiscard]`** attribute (8.5) flags return values that must not be ignored. Confirm 8.5 specifics against the docs.

### 17.2 Generators & iterators **[A]**

A **generator** is a function that uses **`yield`** to produce values lazily, one at a time, instead of building and returning a whole array. It is the memory-efficient way to iterate over huge or infinite sequences: you only ever hold one item at a time. Use generators to stream a multi-gigabyte file line-by-line, paginate an API, or model an unbounded sequence. The **`Iterator`** and **`IteratorAggregate`** interfaces are the manual, object-based equivalent; generators are the easy way to satisfy them.

```php
<?php
declare(strict_types=1);

// Reads a huge file lazily — memory stays flat regardless of file size.
function readLines(string $path): \Generator
{
    $fh = fopen($path, 'rb');
    try {
        while (($line = fgets($fh)) !== false) {
            yield rtrim($line, "\n");   // produce one line, then PAUSE here
        }
    } finally {
        fclose($fh);
    }
}

foreach (readLines('huge.log') as $i => $line) {   // pulls one line per iteration
    if (str_contains($line, 'ERROR')) echo "$i: $line\n";
}

// Generators can yield key=>value and even receive values via ->send().
function naturals(): \Generator
{
    $n = 1;
    while (true) { yield $n++; }     // infinite — only safe because it's lazy
}
```

### 17.3 The SPL **[A]**

The **Standard PHP Library (SPL)** bundles efficient data structures and iterators you'd otherwise re-implement: `SplStack`, `SplQueue`, `SplDoublyLinkedList`, `SplPriorityQueue`, `SplFixedArray` (a true fixed-size, memory-light array), `SplObjectStorage` (an object-keyed map/set), and the directory/file iterators from §12. Reach for these when a plain array's semantics are wrong (you want strict LIFO/FIFO, priority ordering, or object identity keys).

```php
<?php
declare(strict_types=1);

$stack = new \SplStack();
$stack->push('a'); $stack->push('b');
echo $stack->pop();              // 'b'  (LIFO)

$pq = new \SplPriorityQueue();
$pq->insert('low', 1);
$pq->insert('high', 10);
echo $pq->extract();             // 'high'  (highest priority first)

// SplObjectStorage: use objects themselves as keys (identity-based set/map).
$seen = new \SplObjectStorage();
$obj  = new \stdClass();
$seen->attach($obj, 'metadata');
var_dump($seen->contains($obj)); // true
```

### 17.4 Fibers & the async story **[A]**

**Fibers** (PHP 8.1) are a low-level concurrency primitive: a fiber is a block of code whose execution you can **suspend** and later **resume**, with its own stack — cooperative, single-threaded coroutines. Fibers are *plumbing*, not an application API: you almost never write raw `Fiber` code. Their purpose is to let **event-loop libraries** implement non-blocking, async-looking code *without* the `Promise`/callback gymnastics of older PHP.

The practical async landscape in 2026:

- **AMPHP / ReactPHP** — userland event loops; with Fibers, AMPHP v3 lets you write straight-line async code that yields under the hood.
- **Swoole / OpenSwoole** — a C extension providing coroutines, an HTTP server, and connection pooling; it makes PHP a long-running, high-concurrency server (breaking the shared-nothing model deliberately).
- **Default PHP-FPM** — remember that for most web apps you *don't need* async: FPM gives you concurrency by running many worker processes. Reach for async only for I/O-bound workloads (many simultaneous outbound calls, websockets, long-lived connections).

```php
<?php
declare(strict_types=1);

// Raw Fiber — shown for understanding; in practice a library drives this.
$fiber = new \Fiber(function (): void {
    echo "start\n";
    $resumeValue = \Fiber::suspend('paused');   // hand control back to the caller
    echo "resumed with: $resumeValue\n";
});

$suspendValue = $fiber->start();    // runs until the first suspend(); returns 'paused'
echo "fiber yielded: $suspendValue\n";
$fiber->resume('go');               // continue from where it suspended
```

### 17.5 FFI & weak references **[A]**

**FFI** (Foreign Function Interface) lets PHP call functions in C shared libraries directly from PHP, without writing a C extension — useful for binding to an existing native library. It's a power tool with sharp edges (you're responsible for memory/types) and is typically restricted in production. **Weak references** (`WeakReference`, `WeakMap`) let you reference an object *without* preventing it from being garbage-collected — ideal for caches and metadata keyed by object identity that mustn't cause memory leaks.

```php
<?php
declare(strict_types=1);

// WeakMap: associate data with an object without keeping it alive.
$cache = new \WeakMap();
$user  = new \stdClass();
$cache[$user] = 'expensive computed value';
// When $user is unset and GC'd, its entry vanishes automatically — no leak.
```

---

## 18. Tooling & Quality — PHPUnit/Pest, PHPStan, CS-Fixer, Xdebug

### 18.1 Testing — PHPUnit & Pest **[A]**

Automated tests are non-negotiable for professional code. **PHPUnit** is the venerable, xUnit-style standard; **Pest** is a newer, expressive layer built on top of PHPUnit with a clean closure-based syntax that many teams now prefer. Both run the same underlying engine. Write **unit tests** for isolated logic and **feature/integration tests** for behavior across layers; aim to test behavior, not implementation details.

```php
<?php
declare(strict_types=1);

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\Test;

final class MoneyTest extends TestCase
{
    #[Test]                                   // attribute replaces the test* naming
    public function it_formats_with_currency(): void
    {
        $money = new Money(1999, 'EUR');
        $this->assertSame('EUR 19.99', $money->format());
    }

    #[Test]
    public function it_rejects_unknown_currency(): void
    {
        $this->expectException(\InvalidArgumentException::class);
        new Money(100, 'XXX');
    }
}
```

```php
<?php
// The same test in Pest — concise, closure-based:
it('formats with currency', function () {
    expect(new Money(1999, 'EUR')->format())->toBe('EUR 19.99');
});
```

```bash
vendor/bin/phpunit                  # run the PHPUnit suite
vendor/bin/pest                     # run via Pest
vendor/bin/phpunit --coverage-text  # coverage report (needs Xdebug or PCOV)
```

### 18.2 Static analysis — PHPStan & Psalm **[A]**

This is the defining tool of modern PHP. **PHPStan** (and **Psalm**) analyze your code *without running it*, catching type errors, undefined methods/properties, dead code, impossible conditions, and null-safety bugs — the things a compiler catches in statically-typed languages. PHPStan has **levels 0–9 (plus "max")**; you start low on a legacy codebase and ratchet up. **Run it in CI and treat failures as build-breaking.** Combined with `declare(strict_types=1)` and docblock generics (§9.5), PHPStan at a high level makes PHP feel genuinely type-safe.

```bash
composer require --dev phpstan/phpstan
vendor/bin/phpstan analyse src --level=max
```

```neon
# phpstan.neon — project config
parameters:
    level: 9
    paths:
        - src
        - tests
    # Treat phpdoc types as authoritative for generics checking.
    treatPhpDocTypesAsCertain: true
```

### 18.3 Code style — PHP-CS-Fixer & PHP_CodeSniffer **[A]**

Consistent formatting removes bikeshedding from reviews. **PHP-CS-Fixer** and **PHP_CodeSniffer** (`phpcs`/`phpcbf`) enforce a style — usually **PSR-12** — and can auto-fix violations. Wire them into a pre-commit hook ([Git](GIT_GUIDE.md)) and CI so style is never debated by humans.

```bash
vendor/bin/php-cs-fixer fix          # auto-format to the configured ruleset
vendor/bin/phpcs --standard=PSR12 src  # report PSR-12 violations
vendor/bin/phpcbf --standard=PSR12 src # auto-fix them
```

### 18.4 Debugging — Xdebug **[A]**

**Xdebug** is the step-debugger and profiler. With it and an IDE you set breakpoints and inspect a running request line-by-line — vastly better than `var_dump` archaeology. It also produces code-coverage data for PHPUnit and profiling output for performance work (§19). It's a *development* extension; **never enable Xdebug in production** (it carries a large performance penalty and can leak internals).

```ini
; php.ini (development only)
zend_extension=xdebug
xdebug.mode=develop,debug,coverage   ; nicer var_dump + step debugging + coverage
xdebug.start_with_request=trigger     ; only debug when a trigger cookie/var is present
xdebug.client_host=127.0.0.1
xdebug.client_port=9003
```

---

## 19. Performance — OPcache, JIT, Preloading, Profiling

### 19.1 OPcache — the single biggest win **[A]**

PHP normally recompiles every `.php` file to bytecode (opcodes) on *every request* — wasteful, since the code rarely changes. **OPcache** compiles each file once and caches the bytecode in shared memory, so subsequent requests skip parsing and compilation entirely. It is the single most important performance setting and should be **on in every production deployment.** In production, also disable timestamp validation so OPcache doesn't `stat()` files on each request (deploy a fresh process to pick up new code).

```ini
; php.ini — production OPcache
opcache.enable = 1
opcache.memory_consumption = 256       ; MB of shared memory for opcodes
opcache.max_accelerated_files = 20000  ; raise above your file count
opcache.validate_timestamps = 0        ; PROD: don't re-check files (restart on deploy)
; opcache.validate_timestamps = 1      ; DEV: pick up edits automatically
```

### 19.2 The JIT **[A]**

The **JIT** (Just-In-Time compiler, PHP 8.0) goes a step further than OPcache: it compiles hot opcodes to native machine code at runtime. The honest truth: for typical **web request** workloads (I/O-bound, short-lived), the JIT's benefit is **modest** — OPcache already removed the big cost, and most time is spent waiting on the database. Where the JIT genuinely shines is **CPU-bound** work: image processing, math, simulations, long-running CLI computation. Enable it, but don't expect it to speed up a database-bound web app.

```ini
opcache.jit = tracing          ; the recommended JIT mode
opcache.jit_buffer_size = 64M  ; 0 disables the JIT even if a mode is set
```

### 19.3 Preloading **[A]**

**Preloading** (PHP 7.4+) loads a set of classes/functions into OPcache **once at server startup** and keeps them resident in memory for *all* requests, skipping even the autoload-and-link step. For a framework with a large, stable class set this shaves real time off every request. You point `opcache.preload` at a script that `require`s (or uses the framework's helper to compile) the hot files.

```ini
opcache.preload = /var/www/app/preload.php
opcache.preload_user = www-data
```

### 19.4 Profiling & common pitfalls **[A]**

To make code faster, **measure first** — profile with Xdebug, **Blackfire**, or **SPX** to find the real hotspot rather than guessing. The recurring PHP performance pitfalls:

- **N+1 queries** — a loop that runs a query per row. Batch with a single `IN (...)` query or an ORM eager-load. Usually the #1 real-world bottleneck.
- **Loading huge result sets/files into memory** — stream with generators (§17.2) or PDO cursors.
- **Repeated work in loops** — hoist invariant computation (compile a regex once, fetch config once).
- **Missing OPcache** — covered above; verify it's actually enabled in prod.
- **Unbounded autoloading** — run `composer dump-autoload -o` for an optimized classmap.

Cache expensive results in [Redis](REDIS_GUIDE.md) or a PSR-6/16 cache. And remember: a well-indexed database (see [PostgreSQL](POSTGRESQL_GUIDE.md) and [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md)) beats almost any PHP-level micro-optimization.

---

## 20. Security Best Practices — Consolidated

Security is woven through this guide; here is the consolidated checklist. Internalize it — most real-world PHP breaches come from ignoring these basics, not from exotic attacks.

| Threat | Defense |
|---|---|
| **SQL injection** | **Prepared statements**, always (§14). Never concatenate user data into SQL. Allow-list dynamic identifiers. |
| **XSS** | Escape *output* with `htmlspecialchars($x, ENT_QUOTES)` at the view boundary (§15). Set a Content-Security-Policy header. |
| **CSRF** | Per-session token on every state-changing request + `SameSite` cookies (§15.3). |
| **Broken auth** | `password_hash`/`password_verify` (Argon2id), `session_regenerate_id` on login, secure cookie flags (§15). |
| **Command injection** | `escapeshellarg()` or array-form `proc_open` (§12.3). Avoid shelling out at all if you can. |
| **Path traversal** | `realpath()` + verify the result stays under an allowed base dir; never use raw user input in file paths. |
| **Sensitive data exposure** | HTTPS everywhere; `display_errors=Off` in prod; never log secrets; secrets in env vars, not code (§12.4). |
| **Vulnerable dependencies** | `composer audit` in CI; keep packages and PHP patched; pin with `composer.lock`. |
| **Mass assignment / over-posting** | Explicitly allow-list which fields a request may set; never blindly hydrate an entity from `$_POST`. |
| **Insecure deserialization** | Avoid `unserialize()` on untrusted input — use `json_decode` instead. If you must, pass `allowed_classes`. |

Two cross-cutting principles: **validate input, escape output** (the input/output asymmetry is the heart of web security), and **never trust the client** — every byte from `$_GET`/`$_POST`/headers/cookies/uploads is attacker-controlled until proven otherwise. Generate randomness for tokens/ids with the **CSPRNG** functions `random_bytes()`/`random_int()`, never `rand()`/`mt_rand()`. See [Networking](NETWORKING_GUIDE.md) for TLS and the [Better Auth](BETTERAUTH_GUIDE.md)/[Go JWT](GO_JWT_ARGON2_GUIDE.md) guides for auth patterns.

---

## 21. Gotchas & Best Practices

These are the sharp edges — many are PHP 5-era footguns that strict, modern code mostly avoids, but you must recognize them.

### 21.1 Loose `==` vs strict `===` & type juggling **[I/A]**

The classic PHP trap. `==` performs **type juggling** (coercing operands to a common type) before comparing, which produces surprising results. **Always use `===`/`!==`.** PHP 8 fixed the worst case (`0 == "abc"` is now `false`, was `true` pre-8.0), but loose comparison is still treacherous.

```php
<?php
declare(strict_types=1);

var_dump(0 == "0");        // true
var_dump("1" == "01");     // true   (both juggled to int 1)
var_dump("10" == "1e1");   // true   (numeric string equivalence)
var_dump(null == false);   // true
var_dump(0 === "0");       // false  ← use === and these surprises vanish
var_dump("1" === "01");    // false
```

### 21.2 The `false`/`0` return ambiguity **[I]**

Functions like `strpos`, `array_search`, and `preg_match` can legitimately return `0` (a valid index/position) or `false` (not found). Under `==`, `0 == false` is true, so a naive check misfires. **Always compare the result with `===`/`!==`.** Better: prefer `str_contains`/`str_starts_with` (8.0+) which return clean booleans.

```php
<?php
declare(strict_types=1);

$pos = strpos('hello', 'h');     // 0 — found at the start!
if ($pos == false)  { /* WRONG: 0 == false is true → "not found" misfire */ }
if ($pos === false) { /* CORRECT */ }
if (str_contains('hello', 'h')) { /* CLEAREST */ }
```

### 21.3 Arrays are copy-on-write & passed by value **[I/A]**

PHP arrays have **value semantics**: assigning or passing an array *conceptually copies* it (efficiently, via copy-on-write — the copy is deferred until one side is modified). So a function that mutates an array parameter does **not** affect the caller's array unless the parameter is taken by reference (`&$arr`) or returned. Objects, by contrast, are passed by **handle** (reference-like) — mutating an object inside a function *does* affect the caller. Mixing these mental models causes bugs.

```php
<?php
declare(strict_types=1);

function addItem(array $a): void { $a[] = 'x'; }   // mutates the COPY
$list = [1, 2];
addItem($list);
var_dump($list);              // [1, 2] — caller's array unchanged

function addItemRef(array &$a): void { $a[] = 'x'; } // by reference
addItemRef($list);
var_dump($list);              // [1, 2, 'x'] — now it changed

// Objects are handles: mutation leaks out without &.
function rename(object $o): void { $o->name = 'changed'; }
$obj = (object) ['name' => 'orig'];
rename($obj);
echo $obj->name;             // "changed"
```

### 21.4 Null handling & the nullsafe operator **[I]**

Calling a method on `null` is a fatal `Error`. Guard with the **null coalescing** `??` for values and the **nullsafe** `?->` operator for chains — `?->` short-circuits the whole chain to `null` if any link is `null`, avoiding nested `if` checks.

```php
<?php
declare(strict_types=1);

$country = $user?->address?->country ?? 'Unknown';
//          ^ if $user is null, OR address is null, the whole expression is null,
//            and ?? supplies the fallback. No "Trying to access property of null".
```

### 21.5 Float math is not exact **[I]**

Floats are IEEE-754 binary and cannot represent many decimals exactly, so `0.1 + 0.2 !== 0.3`. **Never use floats for money** — store integer minor units (cents) or use the **BCMath**/**GMP** arbitrary-precision extensions. Compare floats with a small epsilon, not `===`.

```php
<?php
declare(strict_types=1);

var_dump(0.1 + 0.2 === 0.3);                    // false!
var_dump(abs((0.1 + 0.2) - 0.3) < PHP_FLOAT_EPSILON); // true — epsilon compare

// Money: integer cents (see the Money class in §7.2), or BCMath:
echo bcadd('0.1', '0.2', 1);                    // "0.3" exactly
```

### 21.6 Best-practice summary **[I/A]**

- **`declare(strict_types=1);`** in every file; type everything; run **PHPStan at a high level**.
- Prefer **`===`**, **`match`**, **enums**, **readonly**, **immutable value objects**, and **`DateTimeImmutable`**.
- **Prepared statements** for all SQL; **escape all HTML output**; **CSRF tokens**; **`password_hash`**.
- Keep functions and classes small; **program to interfaces**; **inject dependencies** rather than `new`-ing them inside.
- Use **Composer + PSR-4**, commit `composer.lock`, run `composer audit`.
- Don't reach for async/Swoole until you have an I/O-bound reason; FPM concurrency is enough for most apps.
- **OPcache on** in production; profile before optimizing; fix N+1 queries first.
- Never `unserialize()` untrusted input; use `random_bytes`/`random_int` for security tokens; set `expose_php=Off`.

---

## 22. Study Path & Build-to-Learn Projects

PHP rewards writing code as you read. Follow this order and *type out every example* — reading alone won't stick.

1. **§1–§4** — get PHP installed, run scripts and the built-in server, internalize `declare(strict_types=1)`, variables/types/strings, and control flow (especially **`match`** over `switch`). Write tiny scripts until the syntax is automatic.
2. **§5–§6** — typed functions, named args, closures/arrow functions, first-class callables, and **arrays** with the functional trio (`map`/`filter`/`reduce`) plus the 8.4 find functions. Arrays are 80% of day-to-day PHP — get fluent.
3. **§7–§8** — OOP: classes, **constructor promotion**, `readonly`, visibility, then interfaces, traits, and especially **enums**. This is where modern PHP architecture lives.
4. **§9–§11** — the modern type system (**property hooks**, **asymmetric visibility**, docblock generics), namespaces/**PSR-4 autoloading**, and errors/exceptions. These three make your code robust and refactor-safe.
5. **§12–§13** — the FS/OS/exec section and data handling (JSON, **`DateTimeImmutable`**, regex). The everyday toolkit.
6. **§14–§15** — **PDO + prepared statements** and web fundamentals (superglobals, sessions, the four security defenses, password hashing). Do not skip the security material — it's the difference between a toy and a deployable app.
7. **§16–§18** — Composer/PSRs/frameworks, then advanced features (attributes, generators, Fibers), then the quality toolchain (**PHPUnit/Pest, PHPStan, CS-Fixer, Xdebug**). Make PHPStan-at-a-high-level and a test suite habits, not afterthoughts.
8. **§19–§21** — performance (OPcache/JIT), the consolidated security checklist, and the gotchas. Re-read §21 after you've been bitten by a few — it'll mean more.

Then pick up the **[Laravel](LARAVEL_GUIDE.md)** guide to learn the dominant framework, and the database guides ([PostgreSQL](POSTGRESQL_GUIDE.md), [SQLite3](SQLITE3_GUIDE.md), [Relational DB Design](RELATIONAL_DB_DESIGN_GUIDE.md)) for the data layer.

### Build-to-learn projects (increasing depth)

Build the simplest version that works, get it under test, then refactor toward the idioms — that's how you learn *why* each rule exists.

- **CLI to-do / notes tool** — argument parsing (`getopt`), JSON file persistence, typed functions, enums for status, exceptions for bad input. Exercises §3–§6, §11, §12–§13. *Stretch:* a `--export` flag streaming output via a generator.
- **A small, well-tested value-object library** — e.g. `Money`, `EmailAddress`, `DateRange`: immutable, `readonly`, validated in constructors, with a full PHPUnit/Pest suite and PHPStan at max. Exercises §7–§9, §18. *Stretch:* docblock generics on a typed collection.
- **A no-framework web app** (a guestbook or link shortener) — a `public/index.php` front controller, a tiny router, PDO with prepared statements, sessions, **CSRF + XSS + password hashing** done by hand. Exercises §14–§15 end-to-end. Doing security manually once makes you appreciate what frameworks give you.
- **A REST/JSON API** — request parsing from `php://input`, JSON responses with `JSON_THROW_ON_ERROR`, PSR-7-style request/response objects, PDO repositories, transactions, and integration tests. Exercises §13–§16. *Stretch:* run it behind [Nginx](NGINX_GUIDE.md) + PHP-FPM in [Docker](DOCKER_GUIDE.md).
- **Capstone: a small blog/CMS** — auth (Argon2id + sessions), CRUD with prepared statements and transactions, file uploads done safely, an admin area, OPcache tuned, PHPStan at a high level in CI, a real test suite, and `composer audit` passing. Pulls in almost every section. *Stretch:* rebuild it on [Laravel](LARAVEL_GUIDE.md) and feel the difference.

For each project: write the types, run PHPStan, write the tests, and treat security as a feature — not a follow-up. That habit is what separates a hobbyist from a professional PHP developer.
